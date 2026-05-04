# Interview Walkthrough — The Full Flow

> Read this the night before. Walk through it once out loud. This is the entire 90 minutes, in order.

---

## Phase 1: Read & Scope `[0-5 min]`

Read the brief carefully. Then ask the **one question that frames everything**:

> "Are we designing the PSP platform itself — the acquiring infrastructure, tokenisation vault, merchant API — or a merchant's payment system that integrates with a PSP like Checkout.com?"

Then state your assumptions and ask for confirmation:

> "I'm assuming the Payment API handles payment creation with auth and capture, refunds, status queries, and webhook notifications. The dashboard surfaces transaction data, success rates, and chargeback metrics. For compliance — PCI DSS, PSD2/SCA, UK GDPR, FCA. Does that match?"

**Pause. Let them respond. This is a conversation.**

---

## Phase 2: Requirements `[5-15 min]`

### Functional — What It Does

Walk through these naturally, not as a list:

- **Payments:** create, authorise, capture, void, refund (full + partial)
- **Status:** polling via GET + push via webhooks
- **Idempotency:** client sends UUID in HTTP header — exactly-once guarantee
- **Dashboard:** transaction list, detail view, aggregations, chargeback rate, export
- **Users:** merchants (API + dashboard), internal ops (dashboard)

### Non-Functional — How Well It Does It

State **concrete numbers** — this is what separates senior from mid:

| Requirement | Target | One-Liner |
|---|---|---|
| **Availability** | 99.99% API, 99.9% dashboard | "52 min downtime/year for payments — downtime = lost merchant revenue" |
| **Latency** | p99 < 1s payment, p95 < 1s dashboard | "Accept fast, process async — the PSP call alone takes 1-3 seconds" |
| **Durability** | Zero data loss (RPO = 0) | "Event persisted to Kafka before we consider it complete" |
| **Consistency** | Strong for payments, eventual for dashboard | "Money must be correct now; dashboard can lag by seconds" |
| **Scale** | 1K-10K TPS, handle 10x peak | "Black Friday can 10x steady state" |

### UK Compliance — Drop These Early

> "Since we're UK-based: PCI DSS via PSP hosted page, PSD2/SCA via 3DS, UK GDPR with pseudonymisation for right-to-erasure, FCA compliance via append-only double-entry ledger."

**End with:** "Does this align with what you'd expect, or should I adjust any targets?"

---

## Phase 3: High-Level Architecture `[15-30 min]`

### What to Draw (In This Order)

Draw left to right, top to bottom. Name each box as you draw it. Say **why** it's there.

```
                        SYNCHRONOUS PATH (accept fast)
                        ==============================

┌──────────┐      ┌───────────────┐      ┌──────────────────┐
│ Customer  │─────►│  API Gateway  │─────►│ Payment Service   │
│           │      │               │      │ (orchestrator)    │
└──────────┘      │ • TLS termination    └──────┬───────────┘
                  │ • OAuth/JWT          ┌──────┼──────────────┐
                  │ • Rate limiting      │      │              │
                  │ • WAF               ▼      ▼              ▼
                  └───────────────┘ ┌────────┐┌──────┐   ┌──────────┐
                                   │Postgres││Kafka │   │  Redis   │
                                   │ (ACID) ││(events)   │(SETNX,  │
                                   │ source ││99.999%│   │ cache)  │
                                   │of truth│└──┬───┘   └──────────┘
                                   └────────┘   │
                                                │
              ════════════════ ASYNC BOUNDARY ════════════════
                                                │
                        ASYNCHRONOUS PATH (process reliably)
                        ====================================
                                                │
                                   ┌────────────┴────────────┐
                                   ▼                         ▼
                            ┌────────────┐            ┌────────────┐
                            │  Payment   │            │  Webhook   │
                            │ Processor  │            │ Dispatcher │
                            │ (worker)   │            │(retry+DLQ) │
                            └──────┬─────┘            └──────┬─────┘
                                   │                         │
                                   ▼                         ▼
                            ┌────────────┐            Merchant's
                            │    PSP     │            endpoint
                            │• hosted page│
                            │• 3DS / SCA │
                            │• tokenisation│
                            └──────┬─────┘
                                   │
                          ┌────────┴────────┐
                          ▼                 ▼
                   ┌────────────┐    ┌────────────┐
                   │ Acquiring  │───►│  Issuing   │
                   │   Bank     │card│   Bank     │
                   │(merchant's)│net │(customer's)│
                   └────────────┘    └────────────┘

                        POST-PROCESSING (on success)
                        ============================

                            Kafka ──────┬──────────────────┐
                                        │                  │
                                        ▼                  ▼
                                 ┌────────────┐    ┌────────────┐
                                 │  Wallet    │    │  Ledger    │
                                 │ (merchant  │    │ (append-   │
                                 │  balance)  │    │  only, DR  │
                                 └────────────┘    │  + CR)     │
                                                   └────────────┘
                                        │
                                        ▼
                                 ┌────────────┐
                                 │   Data     │◄── CDC from PostgreSQL
                                 │ Warehouse  │
                                 │(reconcile, │
                                 │ analytics) │
                                 └──────┬─────┘
                                        │
                                        ▼
                                 ┌────────────┐
                                 │ Dashboard  │  ← reads from here,
                                 │ (merchant  │    never from primary DB
                                 │  + ops)    │
                                 └────────────┘
```

### How to Talk Through the Diagram

As you draw each section, say:

**Sync path:** "Customer hits the API Gateway — that's our front door. TLS termination, OAuth, rate limiting, WAF. The Payment Service orchestrates: checks idempotency in Redis (sub-ms), persists the payment order in PostgreSQL, publishes the event to Kafka. Returns 202 Accepted. Fast — under 200ms."

**Async boundary:** "Everything below this line is async. The payment isn't complete until the event is in Kafka — that's our durability guarantee."

**Async path:** "Payment Processor consumes from Kafka, calls the PSP. The PSP handles the hosted payment page, 3DS authentication, and talks to the acquiring bank, which routes through the card network to the issuing bank. Separately, the Webhook Dispatcher pushes status updates to the merchant — retry with backoff + jitter, DLQ after max retries."

**Post-processing:** "On success, Kafka feeds three services: Wallet updates the merchant balance, Ledger appends balanced debit + credit entries, and the Data Warehouse receives data via CDC for reconciliation and the dashboard."

### The Key Insight to Say Out Loud

> "The architecture is async-first. We accept the payment synchronously — return 202 Accepted — then process everything else through Kafka. The only synchronous path is customer → API Gateway → Payment Service → persist to Kafka. Everything after that is async."

**Trade-off:** "The trade-off is the client doesn't get an immediate success/failure — they need webhooks or polling. But blocking a HTTP connection for 2-10 seconds while we talk to the bank is worse — timeouts, wasted connections, poor UX."

---

## Phase 4: Deep Dives `[30-75 min]`

Go through each area. For every decision, use the formula:

```
State the choice → Explain why → Acknowledge trade-off → Mitigate
```

---

### 4.1 API Design

**Endpoints** — draw or list these:

```
POST   /v1/payments              (idempotency key in header)
GET    /v1/payments/{id}
POST   /v1/payments/{id}/refund
GET    /v1/payments?cursor={token}
POST   /v1/webhooks
```

**Key talking points:**
- **Sync accept, async process:** "Return 202 Accepted immediately. PSP + bank call happens async via Kafka. Merchant gets result via webhook."
- **Cursor pagination, not offset:** "Offset breaks when new transactions insert mid-page. Cursor is stable for append-heavy payment data."
- **Webhooks + polling:** "Webhooks as primary — HMAC-signed, retry with exponential backoff + jitter, DLQ after max retries. Polling as fallback for merchants with unreliable endpoints."

**Trade-off:** "Supporting both webhooks and polling is more implementation work, but merchant reliability varies — we can't depend on every merchant having a reliable endpoint."

---

### 4.2 Data Layer

**Four stores, each with a reason:**

| Store | Why | Trade-Off |
|---|---|---|
| **PostgreSQL** | "ACID — debit and credit must be atomic. Money can't be wrong." | "Harder to scale horizontally. Mitigate: shard by merchant_id with Citus, read replicas for dashboard." |
| **Redis** | "Sub-ms idempotency via SETNX. Also rate limiting, cache, distributed locks." | "Extra infra. But the speed layer catches 99% of duplicates before they hit the DB." |
| **Kafka** | "99.999% available. Payment event isn't complete until persisted here. Replay capability if a consumer goes down." | "Operationally complex. Use managed (MSK/Confluent) — don't self-host Kafka." |
| **Data Warehouse** | "CDC from PostgreSQL via Kafka. Powers dashboard aggregations, reconciliation, fraud analytics." | "Seconds of lag. Acceptable for dashboard — not for payment state." |

**If challenged — "Why not DynamoDB?":**
> "DynamoDB scales better horizontally but doesn't give us multi-table ACID transactions. Payment processing needs atomic debit + credit across accounts. If we hit 100K+ TPS, I'd look at CockroachDB or TiDB — distributed SQL that preserves ACID."

**CAP per operation, not per system:**
> "Payment writes are CP — reject if inconsistent. Dashboard reads are AP — showing data from 5 seconds ago is better than showing nothing."

---

### 4.3 Kafka & Event Flow

**One topic, one producer, four consumers:**

The Payment Service is the only producer. It publishes JSON events to a single `payment-events` topic, partitioned by `payment_id`:

```json
{
  "event_type": "PaymentCaptured",
  "payment_id": "pay_abc",
  "merchant_id": "merchant_123",
  "amount": 5000,
  "currency": "GBP",
  "timestamp": "2026-05-04T10:30:00Z"
}
```

**Four independent consumer groups** read from the same topic. Each checks `event_type` and acts or ignores:

| Consumer Group | Reacts To | Action |
|---|---|---|
| **Payment Processor** | `PaymentCreated` | Calls PSP → acquiring bank → card network → issuing bank |
| **Webhook Dispatcher** | All status changes | POSTs to merchant's endpoint (retry + DLQ) |
| **Wallet Service** | `PaymentCaptured` | Credits merchant balance |
| **Ledger Service** | `PaymentCaptured` | Appends balanced debit + credit entries |

**Key talking points:**
- **Why one topic:** "Same partition key, same retention, same ordering needs. I'd split only if retention policies or partitioning needs diverge."
- **Why Kafka over direct calls:** "Consumers pull at their own pace — if the Ledger is slow, it doesn't block webhooks or wallet updates. That's the decoupling."
- **Ordering guarantee:** "Partition key is `payment_id` — all events for one payment land in the same partition, processed in order. PENDING before CAPTURED, guaranteed."
- **DLQ:** "Messages that fail after max retries go to `payment-events-dlq` — isolate bad messages without blocking healthy traffic."

**Trade-off:** "One topic is simpler to operate but means all consumers see all events. At our scale (1K-10K TPS) this is fine — consumers filter cheaply. If we hit 100K+ TPS with very different retention needs, I'd split into separate topics."

---

### 4.4 Redis

**What it does in our system — three jobs, all sub-millisecond:**

| Job | Key Pattern | Value | TTL |
|---|---|---|---|
| **Idempotency** | `idempotency:{uuid}` | `{"status":"PENDING","payment_id":"pay_abc"}` | 24 hours |
| **Rate limiting** | `ratelimit:{merchant_id}:{window}` | `count (integer)` | 1 second |
| **Cache** | `payment:{payment_id}` | Full payment object (JSON) | 5 minutes |

**Example — idempotency check:**

```
Client sends: POST /v1/payments
              Header: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Redis command:
  SETNX idempotency:550e8400-e29b-41d4-a716-446655440000
        '{"status":"PENDING","payment_id":"pay_abc"}'

  Key exists?  → Return cached result (duplicate request, no processing)
  Key is new?  → Set with 24h TTL, continue to PostgreSQL
```

**Example — rate limiting (sliding window):**

```
Redis command:
  INCR ratelimit:merchant_123:1714830000
  EXPIRE ratelimit:merchant_123:1714830000 1

  Count > 1000? → Return 429 Too Many Requests
  Count ≤ 1000? → Allow through
```

**Key talking points:**
- **Why Redis over DB for idempotency:** "Sub-ms response. We check idempotency on every request — hitting PostgreSQL for that adds 5-10ms per request under load. Redis catches 99% of duplicates before they touch the DB."
- **Why Redis AND DB:** "Redis is the speed layer, DB unique constraint is the safety net. Redis can lose data on restart — the DB constraint guarantees correctness even if Redis fails."
- **Why not Redis as primary store:** "Redis is in-memory, not durable by default. Payment state lives in PostgreSQL. Redis is a speed layer, not a source of truth."

**Trade-off:** "Extra infrastructure to operate. But for a payment system at 1K-10K TPS, the alternative — hitting PostgreSQL on every idempotency check — adds unacceptable latency. Redis is the right tool here."

---

### 4.5 Idempotency

**The problem:** A customer clicks "Pay" twice, or a network timeout triggers an automatic retry. Without idempotency, you charge them twice.

**The solution — two layers:**

```
Request arrives with Idempotency-Key: 550e8400...

Layer 1: REDIS (speed)
─────────────────────
  SETNX idempotency:550e8400... → key exists? return cached result
                                → key new? set TTL 24h, continue
  Speed: < 1ms
  Catches: 99% of duplicates

Layer 2: POSTGRESQL (safety)
────────────────────────────
  INSERT INTO payment_orders (id, ...) VALUES ('550e8400...', ...)
  → unique constraint violation? return existing payment
  → success? payment created
  Speed: 5-10ms
  Catches: the 1% that slip through (Redis restart, network partition)
```

**Why two layers — say this:**
> "Redis is fast but not durable — if it restarts, keys are gone. The DB unique constraint is slow but guaranteed. Together: speed for the happy path, correctness for edge cases. Belt and suspenders."

**Full flow with timeline:**

```
T=0ms   Client sends POST /v1/payments (Idempotency-Key: 550e8400)
T=1ms   Redis SETNX → key is new → set with 24h TTL
T=6ms   PostgreSQL INSERT → succeeds (unique constraint passes)
T=8ms   Kafka publish PaymentCreated event
T=10ms  Return 202 Accepted {payment_id: "pay_abc", status: "PENDING"}

T=50ms  Client retries same request (timeout, didn't see response)
T=51ms  Redis SETNX → KEY EXISTS → return cached {payment_id: "pay_abc"}
        ← No DB hit, no Kafka event, no duplicate processing
```

**If Redis is down:**

```
T=0ms   Client sends POST /v1/payments (Idempotency-Key: 550e8400)
T=1ms   Redis SETNX → connection refused (Redis down)
T=6ms   PostgreSQL INSERT → succeeds (first time)
T=10ms  Return 202 Accepted

T=50ms  Client retries
T=51ms  Redis SETNX → connection refused (still down)
T=56ms  PostgreSQL INSERT → UNIQUE CONSTRAINT VIOLATION → return existing payment
        ← DB catches the duplicate. Slower, but correct.
```

**Key talking points:**
- **Client generates the key:** "UUID generated client-side, sent in HTTP header. This is critical — the server can't generate idempotency keys because it doesn't know if the client's previous request succeeded."
- **TTL choice:** "24 hours. Long enough to cover retries and reconciliation. Short enough to not fill Redis memory. Configurable per merchant if needed."
- **What we store in Redis:** "Not just a flag — we store the response. So on duplicate, we can return the exact same response without any processing."

**Trade-off:** "Two systems to maintain for one guarantee. But the cost of double-charging a customer — refund, support ticket, trust damage, potential chargeback — far exceeds the operational cost of Redis."

---

### 4.6 Payment Lifecycle

**This is not a single "pay" step — walk through the real card flow:**

```
Authorization ──► Capture ──► Settlement
    │                            │
  VOIDED                   REFUNDED or DISPUTED
(release hold)                   │
                          CB_WON or CB_LOST
```

**Key talking points:**
- **Auth:** "Reserves funds. No money moves. Common for hotels, car rentals."
- **Capture:** "Commits the charge. Auth + immediate capture is standard for e-commerce."
- **Settlement:** "Bank transfer, T+1 to T+3 business days."
- **Void:** "Release the hold before capture — cheaper than refund, no interchange fees."
- **Chargebacks:** "Customer disputes via issuing bank. Funds reversed. Merchant submits evidence. Rate above 1% triggers card network penalties — dashboard surfaces this per merchant."

**3DS / SCA (mandatory in UK):**
> "PSD2 mandates Strong Customer Authentication. PSP handles 3DS2 — customer authenticates with their bank via biometric or OTP. Two of three factors: know, have, are. Shifts fraud liability from merchant to issuer. Exemptions for low-value under £30, recurring, trusted merchants."

---

### 4.7 Reliability

**The stack, in order — each layer protects against something different:**

| Layer | What It Prevents |
|---|---|
| **Idempotency** (Redis SETNX + DB constraint) | Double-charging — "two layers: Redis catches 99% in sub-ms, DB constraint is the safety net" |
| **Retry + exponential backoff + jitter** | Transient failures — "backoff gives time to recover, jitter prevents thundering herd" |
| **Timeout** | Infinite waiting — "combined with idempotency key so retries after timeout are safe" |
| **Circuit breaker** | Cascading failures — "closed → open → half-open. Per-merchant breakers for webhooks" |
| **Kafka** | Data loss — "event persisted before processing. 99.999% available" |
| **Dead letter queue** | Poison pills blocking healthy traffic — "isolate, debug later" |

**The idempotency answer they'll definitely ask:**
> "Client generates UUID, sends it in HTTP header. Redis SETNX checks in sub-millisecond — if key exists, return cached result. If new, set with TTL and proceed. DB unique constraint on the payment order ID is the ultimate safety net — even if Redis fails, the DB catches the duplicate. Two layers: speed and safety."

---

### 4.8 Security & Compliance

**Four regulations, four solutions — keep it tight:**

| Regulation | Solution |
|---|---|
| **PCI DSS** | "PSP hosted page collects card details. We store tokens, never card numbers. Reduces PCI scope to SAQ-A." |
| **PSD2 / SCA** | "3DS2 via PSP. Mandatory in UK. Biometric/OTP. Liability shifts to issuer." |
| **UK GDPR** | "Pseudonymisation — PII in separate table linked by opaque token. On erasure: delete the mapping. Ledger stays intact but unidentifiable." |
| **FCA** | "Append-only double-entry ledger. Daily reconciliation. Full audit trails." |

**Encryption:**
> "TLS 1.2+ for all external traffic. mTLS for internal service-to-service — zero trust. AES-256 at rest."

**Auth model:**
> "OAuth 2.0 + JWT for merchant API and dashboard. mTLS for internal services. API keys for identification, JWTs for authorisation."

---

### 4.9 Observability

**Three pillars — say this:**
> "Metrics tell us something is wrong. Logs tell us what happened. Traces tell us where."

**What to monitor:**

| Type | Examples |
|---|---|
| **Technical** | API error rate, p99 latency, CPU, Kafka consumer lag |
| **Business** | Payment success rate, per-merchant failure rate, chargeback rate, conversion |

**Alerting — P1 to P4:**
> "Payment success rate below 90% = page on-call immediately. p99 above 1 second = page within 15 minutes. Single merchant failing = Slack alert. Refund rate trending up = ticket."

**Tool choice:**
> "OpenTelemetry for instrumentation — vendor-neutral. Datadog in production for the unified metrics + logs + traces view."

---

### 4.10 Infrastructure

**Compute:**
> "Kubernetes for payment core — predictable latency, no cold starts, sidecar mTLS, horizontal pod autoscaling. Lambda for webhook dispatcher — spiky, stateless, cold starts acceptable for async."

**Hosting:**
> "AWS — broadest catalogue, London region for UK data residency, managed services: MSK for Kafka, RDS for PostgreSQL, ElastiCache for Redis. Default to managed — the cost of self-hosted Kafka going down at 3 AM exceeds the MSK bill."

**Multi-region:**
> "Active-passive. Primary in UK for FCA regulatory proximity and UK GDPR data residency. Async replication to a standby region. Write conflicts in financial data are unacceptable — active-active is for systems where stale data is tolerable."

**Deployments:**
> "Canary for the payment service — 5% traffic, watch success rate and latency, auto-rollback if success drops by 1%. For database migrations — blue-green with backward-compatible schema."

---

## Phase 5: Wrap Up `[75-85 min]`

Summarise the key design decisions in one sweep:

> "To recap — async-first architecture with Kafka as the backbone. PostgreSQL for ACID guarantees on payment data, Redis for sub-ms idempotency, data warehouse for analytics and reconciliation. PSP hosted page keeps us out of PCI scope. 3DS via PSP for PSD2/SCA. Two-layer idempotency for exactly-once. Circuit breakers and DLQ for fault tolerance. Canary deployments, active-passive multi-region, UK-first for compliance."

Then:

> "Is there any area you'd like me to go deeper on?"

---

## The 12 Phrases to Memorise

These are the sentences that signal principal-level thinking. Drop them naturally:

| # | Phrase |
|---|---|
| 1 | "Accept synchronously, process asynchronously" |
| 2 | "The trade-off is..." |
| 3 | "Two-layer idempotency — Redis for speed, DB for safety" |
| 4 | "CP for payments, AP for dashboard — per operation, not per system" |
| 5 | "Exponential backoff with jitter to prevent thundering herd" |
| 6 | "PSP hosted page — merchant never touches card data" |
| 7 | "Append-only double-entry ledger — every transaction balances" |
| 8 | "Mandatory under PSD2 in the UK — shifts liability to issuer" |
| 9 | "Pseudonymisation — delete the mapping, ledger stays intact" |
| 10 | "I'm choosing complexity only where the cost of failure justifies it" |
| 11 | "Circuit breaker — stop hammering a failing service, let it recover" |
| 12 | "Default to managed services — self-host only when you need control they don't give you" |

---

## Quick Navigation

| Phase | Time | What Happens |
|---|---|---|
| **1. Read & Scope** | 0-5 min | Read brief, ask scope question, state assumptions |
| **2. Requirements** | 5-15 min | FRs, NFRs with numbers, UK compliance |
| **3. Architecture** | 15-30 min | Draw the diagram, explain each component, highlight async boundary |
| **4. Deep Dives** | 30-75 min | API, data, payment lifecycle, reliability, security, observability, infra |
| **5. Wrap Up** | 75-85 min | Recap decisions, invite deeper questions |

---

## Detailed Reference

| Topic | Deep-Dive Doc |
|---|---|
| Architecture & payment flow | [00-payment-system-overview.md](00-payment-system-overview.md) |
| Reliability patterns | [01-reliability-patterns.md](01-reliability-patterns.md) |
| Idempotency | [02-idempotency.md](02-idempotency.md) |
| Security & UK compliance | [03-security.md](03-security.md) |
| Quick reference cheat sheet | [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) |
| Recruiter's guidance | [05-recruiter-briefing.md](05-recruiter-briefing.md) |
| Values interview | [06-values-interview.md](06-values-interview.md) |
| Data layer & CAP | [07-data-layer.md](07-data-layer.md) |
| Trade-offs & hosting | [08-trade-offs.md](08-trade-offs.md) |
| Requirements (FR & NFR) | [09-requirements-fr-nfr.md](09-requirements-fr-nfr.md) |
| Requirement gathering questions | [10-requirement-gathering-questions.md](10-requirement-gathering-questions.md) |
| Kafka fundamentals | [12-kafka-fundamentals.md](12-kafka-fundamentals.md) |
| Kafka cheat sheet | [13-kafka-cheat-sheet.md](13-kafka-cheat-sheet.md) |
| Bank-to-inventory saga | [14-bank-inventory-saga.md](14-bank-inventory-saga.md) |
