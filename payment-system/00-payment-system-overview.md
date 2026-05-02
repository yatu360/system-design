# Payment System Design — Overview & Architecture

> Source: [lucianoprea.com/blog/payment-system-design](https://lucianoprea.com/blog/payment-system-design)

---

## The One-Liner

A payment system **coordinates money movement** from a customer's bank to a merchant's bank, while guaranteeing **exactly-once processing**, **auditability**, and **fault tolerance**.

---

## High-Level Architecture

```
┌─────────┐       ┌───────────────┐       ┌────────────┐
│ Customer │──────►│  API Gateway  │──────►│  Payment   │
│          │       │               │       │  Service   │
└─────────┘       │ • Auth (OAuth)│       │(orchestrator)│
                  │ • Rate limit  │       └──────┬─────┘
                  │ • TLS termination│            │
                  │ • Validation  │       ┌──────┴──────────────────────┐
                  └───────────────┘       │              │              │
                                          ▼              ▼              ▼
┌─────────┐                         ┌──────────┐  ┌──────────┐  ┌──────────┐
│Dashboard│◄── Read Replica / ──────│PostgreSQL│  │  Kafka   │  │  Redis   │
│  (Web)  │    Data Warehouse       │  (ACID)  │  │(events)  │  │ (cache)  │
└─────────┘                         └──────────┘  └────┬─────┘  └──────────┘
                                                       │
                                          ┌────────────┴────────────┐
                                          ▼                         ▼
                                   ┌────────────┐           ┌────────────┐
                                   │  Payment   │           │  Webhook   │
                                   │ Processor  │           │ Dispatcher │
                                   └──────┬─────┘           └──────┬─────┘
                                          │                        │
                                          ▼                        ▼
                                   ┌────────────┐           Merchant's
                                   │    PSP     │           endpoint
                                   │(hosted page)│
                                   └──────┬─────┘
                                          │
                                ┌─────────┴─────────┐
                                ▼                   ▼
                         ┌────────────┐      ┌────────────┐
                         │ Acquiring  │      │  Issuing   │
                         │   Bank     │─────►│   Bank     │
                         │(merchant's)│ card │(customer's)│
                         └────────────┘ network└──────────┘

Post-Processing (on success):
                                   ┌────────────┐
                         ┌────────►│  Wallet    │  (merchant balance)
                         │         └────────────┘
              Kafka ─────┤
                         │         ┌────────────┐
                         └────────►│  Ledger    │  (audit trail)
                                   └────────────┘
                         │
                         │         ┌────────────┐
                         └────────►│  Data      │  (analytics / reconciliation)
                                   │ Warehouse  │
                                   └────────────┘
```

---

## Core Components

### Request Path (Synchronous)

| Component | Role | Think of it as... |
|-----------|------|-------------------|
| **API Gateway** | Entry point — **TLS termination**, **authentication** (OAuth/JWT), **rate limiting**, request validation, routing | The **front door** |
| **Payment Service** | Orchestrates the entire flow — stores payment events, calls PSP, updates wallet & ledger | The **conductor** |
| **Redis** | **Idempotency checks** (sub-ms), **rate limiting**, **session cache**, **distributed locks** | The **speed layer** |
| **PostgreSQL** | Source of truth — payment orders, wallet balances, ledger entries. **ACID transactions**, **strong consistency** | The **vault** |

### External Partners

> **Note:** "Payment Gateway" is a **bank-side** component (validates card credentials, routes to card networks). "API Gateway" is **our infrastructure** (auth, rate limiting, TLS termination). Different things, similar names.

| Component | Role | Think of it as... |
|-----------|------|-------------------|
| **PSP** (Payment Service Provider) | Processes card payments via **hosted payment page**, handles **3DS/SCA**, risk management, **reconciliation** tools | The **middleman** to banks |
| **Payment Gateway** | Bank-side: validates financial credentials, routes transactions through **card networks** (Visa/Mastercard), enforces **PCI DSS** compliance | The **bank's front door** |
| **Acquiring Bank** | Merchant's bank — captures transaction info, validates, routes through **card networks** to issuing bank | Merchant's **money receiver** |
| **Issuing Bank** | Customer's bank — approves/declines, checks balance & account status, handles **3DS authentication** | Customer's **money sender** |

### Async Processing Path

| Component | Role | Think of it as... |
|-----------|------|-------------------|
| **Kafka** | Persists payment events so nothing is lost — **99.999% availability**. Payment NOT complete until event is persisted | The **safety net** |
| **Payment Processor** | Consumes Kafka events, calls PSP, handles bank communication | The **worker** |
| **Webhook Dispatcher** | Pushes status changes to merchant endpoints with **retry + DLQ** | The **notifier** |

### Post-Payment Services

| Component | Role | Think of it as... |
|-----------|------|-------------------|
| **Wallet Service** | Tracks merchant **account balance** — updated after successful PSP processing | The **balance sheet** |
| **Ledger Service** | Appends **every financial transaction record-by-record** — **append-only**, never update/delete | The **audit trail** |
| **Data Warehouse** | Receives data via **CDC from PostgreSQL through Kafka**. Powers analytics, reconciliation, fraud detection | The **analyst** |

---

## The Payment Flow (6 Steps)

```
Customer ──► Merchant ──► PSP (hosted page) ──► Acquiring Bank ──► Card Network ──► Issuing Bank
                                                                                        │
Customer ◄── Merchant ◄── PSP ◄────────────── Acquiring Bank ◄── Card Network ◄────────┘
                                                                              (approve/decline)
```

### Step-by-Step

1. **Customer places order** and provides payment info to merchant
2. **Merchant redirects** customer to **PSP-hosted payment form** (merchant never sees raw card data)
3. **PSP transmits** cardholder info to the **acquiring bank**
4. **Acquiring bank** captures, validates, routes through **card network** to **issuing bank**
5. **Issuing bank** approves or declines — response flows back through the chain
6. **Merchant receives status** and displays result to customer

### Key Insight

> The merchant **never directly handles card details** — the PSP's hosted page collects them. This is critical for **PCI DSS compliance**.

---

## Internal System Flow (Detailed)

```
Client (with idempotency key in HTTP header)
    │
    ▼
API Gateway ───► TLS termination, OAuth/JWT auth, rate limiting, validation
    │
    ▼
Payment Service
    │
    ├──► Redis: SETNX idempotency:{key}
    │       ├── Key exists? → return cached result (duplicate request)
    │       └── Key new? → continue
    │
    ├──► PostgreSQL: INSERT payment_order (UUID = idempotency key)
    │       └── Unique constraint = ultimate safety net against duplicates
    │
    ├──► Kafka: publish PaymentCreated event
    │       └── Operation NOT complete until event is persisted
    │
    └──► Return 202 Accepted { payment_id, status: "PENDING" }

                    ─── async boundary ───

Kafka consumer (Payment Processor)
    │
    ├──► Call PSP with amount + currency
    │       └── PSP → Acquiring Bank → Card Network → Issuing Bank → approve/decline
    │
    ├──► On SUCCESS:
    │       ├── PostgreSQL: UPDATE payment_order SET status = 'SUCCESS'
    │       ├── Wallet Service: credit merchant balance
    │       ├── Ledger Service: append DEBIT + CREDIT entries
    │       └── Kafka: publish PaymentCompleted event
    │
    ├──► On FAILURE:
    │       ├── PostgreSQL: UPDATE payment_order SET status = 'FAILED'
    │       └── Kafka: publish PaymentFailed event
    │
    └──► Webhook Dispatcher picks up completion/failure event
            └── POST to merchant's webhook URL (with HMAC signature)
                ├── 200 OK → done
                └── Failure → retry (exponential backoff + jitter) → DLQ after max retries
```

### Payment Status State Machine

```
                         ┌────────────┐
                         │  PENDING   │  (order created, not yet sent to PSP)
                         └─────┬──────┘
                               │ PSP called
                               ▼
                         ┌────────────┐
                         │ AUTHORIZED │  (funds RESERVED on card — no money moved yet)
                         └─────┬──────┘
                               │
                    ┌──────────┼──────────┐
                    │          │          │
                    ▼          ▼          ▼
             ┌──────────┐ ┌────────┐ ┌────────────┐
             │ CAPTURED  │ │ VOIDED │ │AUTH_FAILED │
             │(funds     │ │(release│ │(bank       │
             │ charged)  │ │ hold)  │ │ declined)  │
             └─────┬─────┘ └────────┘ └────────────┘
                   │
                   │ T+1 to T+3 days
                   ▼
             ┌──────────┐
             │ SETTLED  │  (funds transferred between banks)
             └─────┬─────┘
                   │
          ┌────────┼────────┐
          ▼                 ▼
    ┌──────────┐     ┌────────────┐
    │ REFUNDED │     │ DISPUTED / │
    │(full or  │     │ CHARGEBACK │
    │ partial) │     └──────┬─────┘
    └──────────┘            │
                    ┌───────┼───────┐
                    ▼               ▼
             ┌────────────┐  ┌────────────┐
             │ CB_WON     │  │ CB_LOST    │
             │(merchant   │  │(funds      │
             │ keeps funds│  │ returned + │
             │            │  │ fee charged│
             └────────────┘  └────────────┘
```

> Each state transition creates a **new ledger entry**. States are never skipped.

### Authorization vs Capture vs Settlement

> This is the real card payment lifecycle — not a single "pay" step.

| Phase | What Happens | Money Moves? | Typical Timing |
|-------|-------------|-------------|----------------|
| **Authorization** | Issuing bank **reserves** funds on the cardholder's account. Card limit reduced but no money moves | **No** — hold only | Instant (1-3s) |
| **Capture** | Merchant confirms the charge. Issuing bank **commits** the reserved funds | **Yes** — debit initiated | Same day or within days of auth |
| **Settlement** | Acquiring bank transfers funds to merchant's account via card network | **Yes** — money arrives | **T+1 to T+3** business days |

```
Customer taps "Pay"
    │
    ▼
Authorization ──► "Is there £50 available?" ──► Issuing bank says YES ──► £50 reserved (not charged)
    │
    │  (merchant can VOID here to release the hold)
    ▼
Capture ──► "Charge the £50" ──► Funds committed
    │
    │  (T+1 to T+3)
    ▼
Settlement ──► Acquiring bank pays merchant ──► Funds in merchant's account
```

**Why this matters:**
- **Auth-only** is common for hotels, car rentals (reserve now, charge later)
- **Auth + immediate capture** is standard for e-commerce
- **Void** releases the hold without charging — cheaper than refund (no interchange fees)
- The time gap between auth and settlement is where **chargebacks** can occur

### Chargebacks & Disputes

> A chargeback is when a customer disputes a charge through their **issuing bank**, bypassing the merchant entirely.

```
Customer contacts issuing bank: "I didn't make this purchase"
    │
    ▼
Issuing bank opens dispute ──► Funds REVERSED from merchant's account
    │
    ▼
Merchant receives chargeback notification
    │
    ├──► Merchant submits evidence (receipts, delivery proof, logs)
    │
    ├──► Issuing bank reviews
    │        │
    │        ├──► CB_WON: Merchant keeps funds
    │        └──► CB_LOST: Funds stay with customer + merchant pays chargeback FEE (£15-25)
    │
    └──► If chargeback rate > 1%: card networks (Visa/Mastercard) impose fines or terminate merchant
```

**System design implications:**
- State machine needs `DISPUTED`, `CB_WON`, `CB_LOST` states
- Webhook events: `payment.disputed`, `chargeback.won`, `chargeback.lost`
- Dashboard must surface chargeback rate per merchant (**card networks penalise >1%**)
- All payment data must be **retained for evidence** — supports append-only ledger design
- Chargeback notifications arrive via PSP webhooks (async, can be days later)

### 3D Secure (3DS) & Strong Customer Authentication

> **Mandatory in the UK under PSD2/SCA regulation.** If you skip this in a UK payments interview, it's a red flag.

```
Customer enters card details on PSP hosted page
    │
    ▼
PSP checks if 3DS is required (SCA regulation)
    │
    ├──► Low-risk / exemption applies → skip 3DS → proceed to authorization
    │
    └──► 3DS required → redirect customer to issuing bank's auth page
              │
              ├──► Customer authenticates (biometric / SMS OTP / banking app)
              │         │
              │         ├──► Success → proceed to authorization
              │         └──► Failure → payment declined
              │
              └──► 3DS shifts fraud liability from merchant to issuing bank
```

**Key points for interview:**
- **PSD2** (Payment Services Directive 2) mandates **SCA** (Strong Customer Authentication) for EU/UK
- SCA requires **two of three factors**: something you **know** (PIN), **have** (phone), **are** (biometric)
- **3DS2** is the current version — embedded in checkout flow, less friction than 3DS1 redirects
- **Exemptions exist**: low-value (<£30), recurring payments, trusted merchants, low-risk transactions
- **Liability shift**: with 3DS, fraud liability moves from merchant to issuing bank — major incentive
- The PSP handles 3DS flow — another reason to use their **hosted payment page**

---

## API Endpoints

```
POST   /v1/payments                  → Create payment (idempotency key in header)
GET    /v1/payments/{id}             → Get payment status
POST   /v1/payments/{id}/refund      → Initiate refund
GET    /v1/payments?merchant={id}&from={date}&to={date}&cursor={token}
                                     → List transactions (cursor-paginated)

POST   /v1/webhooks                  → Register merchant webhook endpoint
DELETE /v1/webhooks/{id}             → Remove webhook

GET    /v1/dashboard/summary         → Aggregated metrics
GET    /v1/dashboard/transactions    → Paginated transaction list
```

---

## Data Layer Summary

```
PostgreSQL (CP)  ── Source of truth, ACID, strong consistency
Redis (AP)       ── Cache, idempotency fast-path, rate limiting
Kafka            ── Event persistence, async processing, 99.999% available
Data Warehouse   ── Analytics, reconciliation, reporting (via CDC)
```

> **CAP per operation:** Payment writes = **CP** (reject if inconsistent). Dashboard reads = **AP** (serve stale over serving nothing). See [07-data-layer.md](07-data-layer.md) for full CAP analysis.

---

## Why This Design Matters

| Concern | How It's Addressed |
|---------|--------------------|
| **Money can't be lost** | Kafka persists events before processing continues |
| **Double-charging** | Idempotency keys (Redis SETNX + DB unique constraint) |
| **Auditing** | Append-only ledger, never update or delete |
| **Bank failures** | Retry (exponential backoff + jitter) + timeout + circuit breaker + DLQ |
| **Compliance** | PSP hosted page (PCI DSS), TLS/mTLS (encryption), UK GDPR (data residency) |
| **Observability** | Metrics (Prometheus/Datadog), tracing (OpenTelemetry), alerting (P1–P4 severity) |
| **Scalability** | Kafka decouples producers/consumers, K8s auto-scaling, DB read replicas + sharding |
| **Security** | API Gateway (auth, rate limiting, WAF), mTLS internal, encryption at rest |

---

## Document Index

| File | What It Covers |
|------|---------------|
| **00-payment-system-overview.md** | **This file** — architecture, components, flows |
| [01-reliability-patterns.md](01-reliability-patterns.md) | Retry + jitter, timeout, fallback, circuit breaker, Kafka, DLQ |
| [02-idempotency.md](02-idempotency.md) | Two-layer idempotency (Redis SETNX + DB constraint), exactly-once guarantee |
| [03-security.md](03-security.md) | Encryption, TLS/mTLS, PCI DSS, tokenization, UK compliance |
| [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) | Quick-reference for interview day |
| [05-recruiter-briefing.md](05-recruiter-briefing.md) | Recruiter's guidance, checklist, study priorities |
| [06-values-interview.md](06-values-interview.md) | Values interview (STAR framework) |
| [07-data-layer.md](07-data-layer.md) | SQL vs NoSQL, CAP theorem, PostgreSQL, Redis, data warehouse |
| [08-trade-offs.md](08-trade-offs.md) | Stage-by-stage trade-offs, hosting, observability |
| [09-requirements-fr-nfr.md](09-requirements-fr-nfr.md) | Functional & non-functional requirements |
| [10-requirement-gathering-questions.md](10-requirement-gathering-questions.md) | Checkout.com-specific requirement gathering questions |
