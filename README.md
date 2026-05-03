# AgentHire

> **AgentHire**: Avalanche-side marketplace for agents—Flask routes and SQLAlchemy, x402-priced calls settled with EIP-3009 through `web3.py` (no mandatory Node facilitator), on-chain escrow/registry/reputation, optional ICM/Teleporter, and OpenAI-compatible inference—usually vLLM or Ollama on Akash.

---

## Table of Contents

1. [Overview](#overview)
2. [Design Philosophy](#design-philosophy)
3. [Architecture](#architecture)
4. [Application Layer and Persistence](#application-layer-and-persistence)
5. [On-Chain Protocol Composition](#on-chain-protocol-composition)
6. [Payments: HTTP 402, EIP-3009, and Escrow](#payments-http-402-eip-3009-and-escrow)
7. [Reputation, Disputes, and the Gatekeeper](#reputation-disputes-and-the-gatekeeper)
8. [Market Mechanisms: Auctions and Staking](#market-mechanisms-auctions-and-staking)
9. [Listing Verification and Risk Controls](#listing-verification-and-risk-controls)
10. [Cross-Subnet Messaging (ICM / Teleporter)](#cross-subnet-messaging-icm--teleporter)
11. [Inference Service Design](#inference-service-design)
12. [Simulation Methodology and Index Reconciliation](#simulation-methodology-and-index-reconciliation)
13. [Configuration and Hardening](#configuration-and-hardening)
14. [Testing and Release Discipline](#testing-and-release-discipline)
15. [Technical Stack](#technical-stack)
16. [Cloud Infrastructure (Akash Network)](#cloud-infrastructure-akash-network)
17. [Repository Layout](#repository-layout)
18. [Deployed Contracts (Avalanche Fuji Testnet)](#deployed-contracts-avalanche-fuji-testnet)
19. [HTTP API Surface](#http-api-surface)
20. [User Flows](#user-flows)
21. [Roadmap](#roadmap)

---

## Overview

AgentHire is a marketplace in which **agents are economic actors**: on-chain identity, listed pricing, escrow-backed buyer spend, reputation tied to completed work and incidents, and (for sellers) stake that can be slashed when the gatekeeper path fires. The implementation is a **monolithic Flask** service with **SQLAlchemy**, a **`web3.py`** execution path for payments (no mandatory Node facilitator), and an **HTTP client** to an OpenAI-compatible inference base URL.

The reference deployment targets **Avalanche Fuji** with fixed contract addresses; each can be overridden via environment variables.

This document focuses on **architecture, trust boundaries, and measured inference hosting**. Fuji contract addresses are public. Signing keys, non-public moderation parameters, and provider-specific lease metadata are out of scope.

---

## Design Philosophy

- **Trust boundaries:** The protocol states **rules**, not blanket safety: how USDC moves, how escrow settles, how disputes become gatekeeper-signed incidents, how stake and reputation update.

- **Machine-native payments:** Priced routes use **HTTP 402** with an `x402/eip-3009` challenge. Settlement uses **EIP-3009** so the buyer signs authorization and a **facilitator** pays gas.

- **Standards adapters:** ERC-8004 is not static. `erc8004.py` maps a stable integration surface onto `AgentRegistry` / `ReputationContract`.

- **Compute isolation:** Inference is **out of process** (`LLM_URL` → vLLM-compatible `POST /v1/chat/completions`). The app tier holds DB and signing keys; the inference host sees only HTTP and an optional API key. The hackathon reference used a **single GPU-backed vLLM host** ([Cloud Infrastructure (Akash Network)](#cloud-infrastructure-akash-network)); any OpenAI-compatible endpoint is interchangeable.

---

## Architecture

Five runtime bands:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Experience: Jinja2, JavaScript, ethers.js (wallets, contracts)               │
├──────────────────────────────────────────────────────────────────────────────┤
│  Application: Flask, SQLAlchemy, rate limits, admin API-key gate              │
├──────────────────────────────────────────────────────────────────────────────┤
│  Adapters: onchain (web3), x402, erc8004, icm, llm (HTTP)                     │
├──────────────────────────────────────────────────────────────────────────────┤
│  Persistence: SQLite or Postgres; `chain_transactions` index                  │
├──────────────────────────────────────────────────────────────────────────────┤
│  Settlement: Avalanche Fuji — registry, reputation, stake, escrow, auctions,  │
│  Teleporter (ICM)                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

Swapping RPC, contract addresses, or `LLM_URL` does not require rewriting marketplace logic if ABIs and config contracts stay aligned.

---

## Application Layer and Persistence

Cross-cutting: **`X-Request-ID`**, structured logs, **CORS** on `/api/*`, **Flask-Limiter** (defaults tolerant of demo polling).

| Table | Role |
|-------|------|
| `agents` | Listings, pricing, category, verification tier, seller fields |
| `orders` | Buyer orders ↔ escrow sessions |
| `verification_entries` | Review queue |
| `auction_bids` | Open bids (UI/API mirror) |
| `chain_transactions` | Economically meaningful events (dashboard index) |
| `payouts`, `moderation_reports`, `reviews` | Ops and ratings |

`seed_db()` hydrates dev fixtures; production runs with **`AUTO_SEED_DATA` off**.

**`chain_transactions`:** use as the first source for dashboard aggregates; reconcile to `web3` when keys and RPC are live.

---

## On-Chain Protocol Composition

`onchain.py` exposes **`OnChain`** for routes and `x402` settlement.

| Contract | Role |
|----------|------|
| **MockUSDC** | EIP-3009 `transferWithAuthorization`; `approve` / `balanceOf` |
| **AgentRegistry** | Identity, listing bounds, availability |
| **ReputationContract** | Credit profile; `submitIncident` + gatekeeper sig |
| **StakingSlashing** | Stake, unstake, slash ladder |
| **EscrowPayment** | `depositFunds`, `getSession`, settlement / cancel |
| **AuctionMarket** | Bids, cancel, claim ordering (per bytecode) |

**Teleporter (`icm.py`):** `sendCrossChainMessage` on the canonical messenger. Fees, `requiredGasLimit`, and relayers are defined by Avalanche ICM specs.

---

## Payments: HTTP 402, EIP-3009, and Escrow

**x402:** Unpaid → **402** + `X-Payment-Challenge` (permit template). Client signs EIP-3009, retries with **`X-Payment`**. Server executes **`transferWithAuthorization`** via facilitator.

**Checkout:** Same permit pattern → approve **EscrowPayment** → **`depositFunds`** → on-chain **session id** → `/order/<id>`.

**Settlement:** `settleSession` and guards encode refunds, seller payout, fees in Solidity.

**Facilitator:** `FACILITATOR_PRIVATE_KEY` (Python path) vs `FACILITATOR_URL` (proxy). If both set, **Python path wins** in the reference tree.

---

## Reputation, Disputes, and the Gatekeeper

`getCreditProfile` returns score, tier, tasks, incidents, and decay fields. The app **renders** them and does not re-score off-chain.

**`POST /api/dispute/submit`** → `GATEKEEPER_URL` or local **`GATEKEEPER_PRIVATE_KEY`**. Signer must equal **`GATEKEEPER_ADDRESS()`** or the tx reverts.

---

## Market Mechanisms: Auctions and Staking

**AuctionMarket:** bids with deposit, token budget, max price per token, category, min seller tier; claims per contract rules.

**StakingSlashing:** slash schedule and bans are on-chain; simulation mirrors them for chain-off demos.

---

## Listing Verification and Risk Controls

**Basic:** automated gates (static checks, sandboxed runs, policy passes, fingerprints). **Thorough:** automated path + human review + registry alignment for a verified badge.

Thresholds and internal prompts are operational; they are not specified here.

---

## Cross-Subnet Messaging (ICM / Teleporter)

ICM moves messages between Avalanche deployments. AgentHire used it to try **demand origination** off the primary Fuji C-chain footprint while keeping registry/escrow reference on Fuji for the milestone. Fees, relayers, and finality follow Avalanche documentation.

---

## Inference Service Design

- **Transport:** `POST {LLM_URL}/v1/chat/completions`, `GET {LLM_URL}/v1/models`.
- **Prompting:** per-listing system text (name, category, bio) plus a short protocol context block; one shared model serves many listings.
- **Defaults (application env):** `LLM_MODEL=Qwen/Qwen2.5-Coder-7B-Instruct` (see application `llm.py`); optional `LLM_API_KEY`.

Resource allocation and measured spend for the reference GPU host are documented under [Cloud Infrastructure (Akash Network)](#cloud-infrastructure-akash-network).

---

## Simulation Methodology and Index Reconciliation

Without keys, `simulation.py` / `sim_engine.py` drive deterministic stake, reputation, auctions, and surge-style pricing so the UI stays coherent. Output is **testnet/demo grade**, not a production pricing oracle.

---

## Configuration and Hardening

| Concern | Development | Production |
|---------|-------------|------------|
| `AUTO_SEED_DATA` | on | off |
| `ENABLE_SIM_ENGINE` | on | off |
| `STRICT_PROD_VALIDATION` | configurable | enforce |
| Database | SQLite | Postgres |
| Limiter | memory | Redis URI when scaled |

`validate_runtime_config` blocks default `SECRET_KEY`, wildcard production CORS, and similar misconfigurations.

---

## Testing and Release Discipline

**pytest** (routes and state transitions), **PowerShell** smoke/predeploy scripts against a live app, **`fullstack_check.py`** with `CHECK_MODE` / `BASE_URL`, and **Docker** image builds before demos.

---

## Technical Stack

| Layer | Technology |
|-------|------------|
| Runtime | Python 3.12 |
| Web | Flask 3, Gunicorn (`wsgi.py`) |
| ORM / DB | SQLAlchemy; SQLite or Postgres |
| Chain | `web3.py`, `eth-account` |
| Frontend | Jinja2, JavaScript, ethers.js, Chart.js |
| Payments | x402, EIP-3009, USDC-compatible token |
| Cross-chain | Avalanche ICM / Teleporter |
| Inference client | OpenAI-compatible HTTP → vLLM |
| App container | Docker (non-root) |
| Compute infrastructure | Akash Network (GPU inference for vLLM) |

---

## Cloud Infrastructure (Akash Network)

The hackathon **vLLM** inference tier — OpenAI-compatible chat completions for agent listings — was hosted on [Akash Network](https://akash.network), a decentralized cloud compute marketplace. Akash was chosen for its permissionless GPU access, competitive pricing relative to centralized cloud providers, and alignment with running inference as decentralized compute next to the Avalanche Fuji protocol deployment.

The Flask application, database, and signing keys were deployed **outside** the lease. The Akash deployment ran **only vLLM** (OpenAI-compatible API) and **Qwen2.5-Coder-7B-Instruct** model weights, defined by an Akash **v2** stack manifest (SDL) with provider bids and settlement in **AKT**.

### Deployment Architecture

| Phase | Replicas | Workload | Purpose |
|-------|----------|-----------|---------|
| Inference | **1** | vLLM (OpenAI API) + Qwen2.5-Coder-7B-Instruct | Chat completions for marketplace agents; `/v1/models` health |

### Resource Profile (Accepted Lease)

The table lists **resource blocks** from the deployed SDL compute profile accepted by the provider for the Qwen/vLLM service:

| Resource | Declared value |
|----------|----------------|
| **CPU** | **8.0** `cpu.units` → **8** threads (SDL v2 `cpu` resource model) |
| **Memory** | **32 GiB** |
| **Storage** | **50 GiB** (weights + image layers + scratch) |
| **GPU** | **1× NVIDIA A40** (`vendor.nvidia.model: a40`) |
| **Service replicas** | **1** |
| **Container image** | `vllm/vllm-openai:latest` (lock to digest in production; hackathon tracked `latest` at pull time) |
| **Model weights** | `Qwen/Qwen2.5-Coder-7B-Instruct` (HF id; loaded by vLLM) |
| **Process listen port** | **8000** (manifest maps **8000 → 80**, `global: true` for HTTPS fronting) |
| **Placement bid cap** | **5000 uakt** per pricing block in SDL (ceiling bid; **settled rate** is what the accepting provider closed at on-chain) |

**Application wiring:** `LLM_URL` = HTTPS base of the leased service (no path suffix; client appends `/v1/chat/completions`). Optional `LLM_API_KEY` if vLLM was started with `--api-key`.

Per-listing behavior is **prompt-only** on the app side; **one** vLLM replica backed all demo listings.

### Cost

| Component | USD | Notes |
|-----------|-----|-------|
| GPU inference lease (Qwen, vLLM) | **62** | Lease-related **AKT** for this workload, converted to USD at the hackathon close-out rate (nearest dollar). Wallet or block explorer totals are the audit trail. |

All pricing settled in **AKT** via the Akash on-chain marketplace. The figure reflects lease-related outflows for this workload during the hackathon window (excludes Avalanche gas from facilitator keys, DNS, and non-inference hosts).

### SDL Definitions

#### vLLM inference service

The following SDL matches the hackathon manifest **shape and numbers** (service, image, env, `expose`, `profiles.compute`, `placement` pricing, `deployment` counts). Region/provider `attributes` from the live file are omitted.

```yaml
---
version: "2.0"

services:
  vllm:
    image: vllm/vllm-openai:latest
    env:
      - MODEL_NAME=Qwen/Qwen2.5-Coder-7B-Instruct
    expose:
      - port: 8000
        as: 80
        to:
          - global: true

profiles:
  compute:
    vllm:
      resources:
        cpu:
          units: 8.0
        memory:
          size: 32Gi
        storage:
          - size: 50Gi
        gpu:
          units: 1
          attributes:
            vendor:
              nvidia:
                - model: a40
  placement:
    akash:
      pricing:
        vllm:
          denom: uakt
          amount: 5000

deployment:
  vllm:
    akash:
      profile: vllm
      count: 1
```

**Notes:** Pin `image` to a digest for reproducibility beyond demos; add vLLM flags (tensor parallel, max model len) via `env` / `command` when scaling. Do not put wallet or database secrets in the SDL.

---

## Repository Layout

```
agenthire/
├── app.py
├── config.py
├── extensions.py
├── models.py
├── auth.py
├── onchain.py
├── x402.py
├── erc8004.py
├── icm.py
├── llm.py
├── simulation.py
├── sim_engine.py
├── wsgi.py
├── Dockerfile
├── requirements.txt
├── .env.example
├── scripts/
├── static/
├── templates/
└── tests/
```

---

## Deployed Contracts (Avalanche Fuji Testnet)

| Contract | Address |
|----------|---------|
| MockUSDC | `0x9C49D730Dfb82B7663aBE6069B5bFe867fa34c9f` |
| AgentRegistry | `0x6B71b84Fa3C313ccC43D63A400Ab47e6A0d4BCbB` |
| ReputationContract | `0x40ef89Ce1E248Df00AF6Dc37f96BBf92A9Bf603A` |
| StakingSlashing | `0xfc942b4d1Eb363F25886b3F5935394BD4932B896` |
| EscrowPayment | `0xD19990C7CB8C386fa865135Ce9706A5A37A3f2f2` |
| AuctionMarket | `0xa7AEEca5a76bd5Cd38B15dfcC2c288d3645E53E3` |

- **Explorer:** [testnet.snowtrace.io](https://testnet.snowtrace.io)
- **Chain ID:** `43113`
- **RPC:** [Avalanche Fuji C-Chain](https://api.avax-test.network/ext/C/rpc)

---

## HTTP API Surface

### Public

| Method | Path |
|--------|------|
| GET | `/api/agents`, `/api/agents/<id>`, `/api/price/<id>`, `/api/search` |
| GET | `/api/onchain/info`, `/api/health`, `/api/ready` |

### Settlement and Chain

| Method | Path |
|--------|------|
| POST | `/api/x402/pay`, `/api/dispute/submit` |
| GET | `/api/session/<id>` |
| POST | `/api/session/<id>/cancel` |
| GET | `/api/agents/<id>/reputation`, `/api/agents/<id>/stake` |

### Auctions, Seller, and Admin

| Method | Path |
|--------|------|
| GET | `/api/auctions`, `/api/auctions/<id>` |
| POST | `/api/auctions/bid`, `/api/auctions/<id>/cancel` |
| POST | `/seller/create`, `/seller/agents/<id>`, `/api/agents/<id>/rate` |
| POST | `/admin/verification-queue/<id>/approve` |
| POST | `/admin/verification-queue/<id>/reject` |
| POST | `/admin/review/<id>`, `/admin/payouts/<id>/release`, `/admin/moderation/<id>/resolve`, `/api/orders/<id>/complete` |

**Auction POST relays:** `/api/auctions/bid` and `/api/auctions/<id>/cancel` return **410** in the reference tree; bids and cancels are expected to be **wallet-signed** against `AuctionMarket`.

Admin mutations use **`X-Api-Key`** when `API_KEY` is set.

---

## User Flows

- **Buyer:** browse → agent → checkout → EIP-3009 → order track → complete → rate.
- **Seller:** listing wizard → verification tier → dashboard → manage → earnings.
- **Admin:** verification queue → gate results → human review → moderation → payouts.

**Tiers:** Basic ($10 USDC, automated only) · Thorough ($50 USDC, automated + human + registry).

---

## Roadmap

- Mainnet USDC, audited escrow, hardened facilitator
- 24/7 gatekeeper ops
- Migrations, Redis limiter, end-user auth beyond API keys
- Webhooks / WebSockets for live orders

---

*Reference deployment: Avalanche Fuji testnet.*
