# E-AVCFP — Electronic Agricultural Value Chain Financing Platform

> A Bank-operated digital financing platform connecting Anchors, Farmers, Suppliers, Aggregators, and Buyers across agricultural value chains in Uganda.



## Overview

E-AVCFP digitises the full lifecycle of agricultural value chain financing — from Anchor credit scoring and ecosystem creation, through input supply and harvest collection, to automated waterfall settlement and farmer Mobile Money payouts. The Bank maintains full control of funds at all times through a Master Ecosystem Wallet (MEW); no money moves without a valid approval chain.

**Stack:** Python (FastAPI) · PostgreSQL · Next.js · Mobile Money APi · CBS ISO 8583



## Key Actors

| Actor              | Role                                                                                   |
| ------------------ | -------------------------------------------------------------------------------------- |
| **Bank Officer**   | Creates ecosystems, onboards Anchors, sets financing limits, configures rules          |
| **Anchor Officer** | Onboards all participants, approves invoices, confirms deliveries, triggers settlement |
| **Farmer**         | Signs contracts, receives inputs, records harvest, withdraws residual payout           |
| **Supplier**       | Delivers inputs, raises invoices, receives 80/20 disbursement                          |
| **Aggregator**     | Collects and grades harvest, delivers to Anchor, invoices for services                 |
| **Buyer**          | Signs offtake contracts, pays for crop, triggers settlement waterfall                  |



## How It Works

```
Bank onboards Anchor → CRB check → Credit score → Financing limit set
    → Ecosystem created → Master Wallet funded → Rules configured
        → Anchor onboards Farmers, Suppliers, Aggregators, Buyers
            → All sign contracts linked to Ecosystem
                → Supplier delivers inputs → Invoice raised → 80/20 disbursement
                    → Supplier + Farmer both confirm delivery → 20% released after harvest
                        → Farmer harvests → Aggregator collects → Anchor → Buyer
                            → Buyer pays → Waterfall settlement (tiered)
                                → Farmer residual split by contribution % → MoMo / Bank
```



## Settlement Waterfall

When the Buyer pays, proceeds flow through five mandatory tiers in order:

1. **Insurance** — premium deducted first
2. **Supplier Recovery** — credit line replenished
3. **Bank Principal** — loan principal repaid to MEW
4. **Bank Interest** — interest and fees repaid
5. **Farmer Residual** — remainder split proportionally by each farmer's harvest contribution, net of loan deductions, paid to individual Farmer Sub-Wallets then disbursed via MoMo or bank transfer

> The waterfall is atomic — all tiers complete or the entire settlement rolls back within 30 seconds.



## Project Structure

```
eavcfp/
├── backend/                     # Python / FastAPI
│   ├── app/
│   │   ├── api/                 # Route handlers per domain
│   │   │   ├── ecosystem.py
│   │   │   ├── anchor.py
│   │   │   ├── farmer.py
│   │   │   ├── supplier.py
│   │   │   ├── aggregator.py
│   │   │   ├── buyer.py
│   │   │   ├── disbursement.py
│   │   │   ├── settlement.py
│   │   │   └── wallet.py
│   │   ├── models/              # SQLAlchemy ORM models (33 tables)
│   │   ├── schemas/             # Pydantic request / response schemas
│   │   ├── services/            # Business logic layer
│   │   │   ├── waterfall.py     # 5-tier atomic settlement engine
│   │   │   ├── disbursement.py  # 80/20 rule engine
│   │   │   ├── wallet.py        # MEW / sub-wallet operations
│   │   │   └── momo.py          # MTN / Airtel MoMo integration
│   │   ├── core/
│   │   │   ├── config.py        # Settings and environment
│   │   │   ├── security.py      # JWT RS256, RBAC, MFA
│   │   │   └── audit.py         # Immutable audit log writer
│   │   └── main.py
│   ├── alembic/                 # Database migrations
│   ├── tests/
│   ├── requirements.txt
│   └── Dockerfile
│
├── frontend/                    # Next.js (App Router)
│   ├── app/
│   │   ├── (bank)/              # Bank Officer dashboard
│   │   ├── (anchor)/            # Anchor Officer dashboard
│   │   ├── (farmer)/            # Farmer portal (mobile-first)
│   │   ├── (supplier)/          # Supplier portal
│   │   ├── (aggregator)/        # Aggregator portal
│   │   └── (buyer)/             # Buyer portal
│   ├── components/
│   ├── lib/                     # API client, auth helpers
│   └── Dockerfile
│
├── docker-compose.yml
├── .env.example
└── README.md
```



## Tech Stack

| Layer            | Technology                                       |
| ---------------- | ------------------------------------------------ |
| **API**          | Python 3.12 · FastAPI · Uvicorn                  |
| **ORM**          | SQLAlchemy 2.0 · Alembic migrations              |
| **Database**     | PostgreSQL 16 · TimescaleDB (time-series)        |
| **Cache**        | Redis 7                                          |
| **Frontend**     | Next.js 15 · TypeScript · Tailwind CSS           |
| **Auth**         | JWT RS256 · OAuth2 · Role-based access (6 roles) |
| **Payments**     | MTN MoMo API v3 · Airtel Money API v2            |
| **Core Banking** | ISO 8583 CBS integration                         |
| **Infra**        | Docker · Kubernetes · nginx                      |

---

## Getting Started

### Prerequisites

- Python 3.12+
- Node.js 20+
- PostgreSQL 16
- Redis 7
- Docker (recommended)

### Quick Start

```bash
# Clone the repo
git clone https://github.com/your-org/eavcfp.git
cd eavcfp

# Copy environment variables
cp .env.example .env

# Start all services
docker-compose up -d

# Run backend migrations
docker-compose exec backend alembic upgrade head

# Seed initial roles and system user
docker-compose exec backend python -m app.scripts.seed
```

- API → `http://localhost:8000`
- Frontend → `http://localhost:3000`
- Swagger docs → `http://localhost:8000/docs`

### Environment Variables

```env
# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/eavcfp
REDIS_URL=redis://localhost:6379

# Auth
JWT_PRIVATE_KEY=...
JWT_PUBLIC_KEY=...
JWT_TTL_MINUTES=15

# Mobile Money
MTN_MOMO_API_KEY=...
MTN_MOMO_API_SECRET=...
AIRTEL_API_KEY=...

# Core Banking
CBS_HOST=...
CBS_PORT=8583
```

---

## API Overview

All endpoints are prefixed `/api/v1/`. Authentication via `Authorization: Bearer <JWT>`.

| Method | Endpoint                 | Description                       |
| ------ | ------------------------ | --------------------------------- |
| `POST` | `/ecosystem`             | Create ecosystem                  |
| `POST` | `/anchor`                | Onboard anchor                    |
| `POST` | `/farmer`                | Register farmer                   |
| `POST` | `/supplier/invoice`      | Submit supplier invoice           |
| `POST` | `/invoice/{id}/approve`  | Anchor approves invoice           |
| `POST` | `/delivery/{id}/confirm` | Confirm input delivery            |
| `POST` | `/settlement/trigger`    | Trigger waterfall settlement      |
| `GET`  | `/wallet/{id}/balance`   | Get wallet balance                |
| `POST` | `/farmer/{id}/withdraw`  | Farmer withdrawal to MoMo or bank |



---

## Roles

| Role                | Access                                                                      |
| ------------------- | --------------------------------------------------------------------------- |
| `BANK_SYSTEM_ADMIN` | Full access — ecosystem creation, rule configuration, anchor onboarding     |
| `BANK_RISK_OFFICER` | Read-only — audit, reporting, early warning alerts                          |
| `ANCHOR_OFFICER`    | Participant onboarding, invoice approval, delivery confirmation, settlement |
| `SUPPLIER_WORKER`   | Invoice submission, delivery recording                                      |
| `AGGREGATOR_WORKER` | Collection recording, aggregator invoices                                   |
| `BUYER_WORKER`      | Payment initiation, contract management                                     |

---

## Database

33 tables across 10 domains. All monetary values stored as `BIGINT` in minor currency units (UGX). All PKs and FKs are UUIDs. PII fields (NIN, MSISDN) are AES-256 encrypted at field level.

See https://github.com/OkelloEmtech/E-Agric-Value-Chain-Finance-Platform/blob/main/DB_Schema_EAVCFP.docx for full schema documentation.

---

## Compliance

- Bank of Uganda FIA 2004 · Agricultural Finance Strategy 2023
- AML Act 2013 — STR filing within 24 hours of trigger
- Uganda Data Protection & Privacy Act 2019 — PII encryption, consent management
- Audit log retained 7 years — immutable, cryptographically signed

---


