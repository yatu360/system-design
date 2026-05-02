# Functional & Non-Functional Requirements — Payment System

> This is the **first thing you do** in the interview. The recruiter stressed it twice. Walk through these requirements out loud, validate with the interviewer, and only then start designing.

---

## Why This Matters

> **"Many candidates just start designing without engaging interviewers. This is a bad signal."** — Recruiter

Requirements gathering is not a formality — it's where you prove you think before you build. A principal engineer scopes the problem before solving it. Getting this wrong means your entire design misses the mark.

---

## Functional Requirements (FR)

> **What the system does.** These are the features and capabilities.

### Payment API

| # | Requirement | Details to Clarify |
|---|------------|-------------------|
| **FR1** | **Create Payment** | One-time or recurring/subscriptions? What payment methods — card only, or bank transfer, wallets, BNPL? |
| **FR2** | **Payment Status** | Can the client poll (`GET /payments/{id}`)? Do we push updates via webhooks? Both? |
| **FR3** | **Refunds** | Full refunds only, or partial? Who initiates — merchant, customer, or both? Time limit on refunds? |
| **FR4** | **Payment Cancellation** | Can a payment be cancelled while `PENDING`? What about after `PROCESSING` has started? |
| **FR5** | **Multi-Currency** | Single currency (GBP) or multi-currency? If multi, do we handle FX conversion or does the PSP? |
| **FR6** | **Webhook Notifications** | Push payment status changes to merchant endpoints? What events — `payment.completed`, `payment.failed`, `refund.completed`? |
| **FR7** | **Retry / Re-attempt** | If a payment fails (insufficient funds), can the customer retry with the same order? Different card? |
| **FR8** | **Idempotency** | Client sends idempotency key — duplicate requests return the original result, never double-charge |

### Dashboard

| # | Requirement | Details to Clarify |
|---|------------|-------------------|
| **FR9** | **Transaction List** | Paginated list of transactions. Filterable by date, status, amount, merchant? Searchable by payment ID? |
| **FR10** | **Transaction Detail** | Drill into a single payment — full lifecycle (created → processing → success/failed), PSP reference, timestamps |
| **FR11** | **Summary / Aggregations** | Total revenue, success rate, failure rate, refund rate — by day, week, month? |
| **FR12** | **Real-Time vs Historical** | Does the dashboard need live-updating data (WebSocket/SSE) or is periodic refresh acceptable? |
| **FR13** | **Export** | Can merchants export transaction data as CSV/PDF for their own reconciliation? |
| **FR14** | **User Roles** | Who accesses the dashboard? Merchants only? Internal ops? Different permission levels? |

### How to Present FRs in the Interview

> Don't read a list. **State your assumption and ask for confirmation:**

```
"I'm assuming the payment API needs to support creating payments, checking status, 
and processing refunds. Merchants will receive webhook notifications for status changes, 
and there's a dashboard for viewing transaction history and aggregated metrics. 
Does that match the scope, or should I include anything else?"
```

---

## Non-Functional Requirements (NFR)

> **How well the system does it.** These define quality, performance, and operational constraints.

### Performance

| # | Requirement | Target | Why This Number |
|---|------------|--------|-----------------|
| **NFR1** | **Throughput** | 1,000–10,000 TPS steady state | Varies by company size. Ask: "What scale are we designing for?" Peak (Black Friday) can be **10x** steady state |
| **NFR2** | **Latency — Payment API** | p50 < 200ms, p95 < 500ms, p99 < 1s | Customer is at checkout — every extra second increases cart abandonment. The PSP + bank call dominates latency (1-3s), so "accept fast, process async" |
| **NFR3** | **Latency — Dashboard** | p95 < 1s for list views, < 3s for heavy aggregations | Internal tool — slightly more tolerance than customer-facing API |
| **NFR4** | **Latency — Webhooks** | Delivered within 30s of status change | Near real-time but not instant — async is fine |

### Availability

| # | Requirement | Target | Why This Number |
|---|------------|--------|-----------------|
| **NFR5** | **API Availability** | **99.99%** (4 nines) = ~52 min downtime/year | Downtime = lost revenue for every merchant on the platform. 3 nines (99.9% = 8.7 hrs/year) is too much for payments |
| **NFR6** | **Dashboard Availability** | **99.9%** (3 nines) = ~8.7 hrs downtime/year | Internal tool — lower bar acceptable. Merchants can tolerate brief dashboard outages if payments still work |
| **NFR7** | **Kafka Availability** | **99.999%** (5 nines) | Payment events must never be lost. Kafka's simple job (store messages) enables this |

### Durability & Consistency

| # | Requirement | Target | Why |
|---|------------|--------|-----|
| **NFR8** | **Zero data loss** for payment records | RPO = 0 (Recovery Point Objective) | Regulatory and legal — you cannot lose a financial transaction record. Ever. |
| **NFR9** | **Strong consistency** for payment state | Read-after-write guarantee | Customer pays → status must immediately reflect the charge. Stale reads = double charges or incorrect balances |
| **NFR10** | **Eventual consistency** acceptable for dashboard | Lag < 5 seconds | Dashboard aggregations can trail real-time by seconds. Freshness is nice, correctness-right-now is not required |
| **NFR11** | **Append-only ledger** | No UPDATE, no DELETE on financial records | Audit trail must be tamper-evident. Corrections are new entries (reversals), not overwrites |

### Scalability

| # | Requirement | Target | Why |
|---|------------|--------|-----|
| **NFR12** | **Horizontal scaling** | Add capacity without downtime | Traffic spikes (flash sales, events, seasonal). Cannot scale vertically forever |
| **NFR13** | **Peak handling** | 10x steady state without degradation | Black Friday, major sporting events. System must absorb spikes gracefully |
| **NFR14** | **Multi-region ready** | Design should support multi-region even if starting single-region | Data residency (**UK GDPR**), latency reduction, disaster recovery. Primary region in UK for **FCA regulatory proximity** |

### Security & Compliance

| # | Requirement | Target | Why |
|---|------------|--------|-----|
| **NFR15** | **PCI DSS compliance** | Merchant never handles raw card data | PSP hosted payment page collects card details. Keeps merchant out of PCI scope |
| **NFR16** | **Encryption at rest** | All payment data encrypted in DB | Regulatory requirement + defense against data breach |
| **NFR17** | **Encryption in transit** | TLS 1.2+ for all external, mTLS for internal | Data integrity and confidentiality. mTLS for zero-trust internal network |
| **NFR18** | **UK GDPR compliance** | Data residency, right to erasure, consent | UK customer data resides in UK data centres. Right to erasure conflicts with append-only ledger — resolved via pseudonymization (see below) |
| **NFR19** | **Rate limiting** | Per API key, per merchant | Abuse prevention, DDoS mitigation, fair usage |
| **NFR20** | **PSD2 / SCA compliance** | 3DS mandatory for UK card payments | PSP handles 3DS2 flow. Two-factor auth (biometric/OTP). Exemptions for low-value (<£30), recurring, trusted merchants. Shifts fraud liability to issuer |
| **NFR21** | **FCA regulatory compliance** | Financial record-keeping, consumer protection | Double-entry append-only ledger, daily reconciliation, full audit trails. FCA is the UK's financial conduct authority |

### Observability

| # | Requirement | Target | Why |
|---|------------|--------|-----|
| **NFR22** | **Monitoring** | Technical metrics (CPU, memory, error rate) + business metrics (success rate, conversion) | Can't fix what you can't see. Business metrics catch revenue-impacting issues that technical metrics miss |
| **NFR23** | **Distributed tracing** | Every payment request traceable end-to-end | When a payment takes 10s, you need to know which hop is slow — PSP? DB? Kafka? |
| **NFR24** | **Alerting** | P1–P4 severity levels with escalation paths | Right person, right urgency. Payment success rate < 90% = page immediately |
| **NFR25** | **Audit logging** | Every state change logged with actor, timestamp, before/after | Compliance, debugging, dispute resolution |

### Reliability

| # | Requirement | Target | Why |
|---|------------|--------|-----|
| **NFR26** | **Idempotency** | Every write operation is idempotent (two-layer: Redis SETNX + DB constraint) | Retries and duplicates must never cause double-charges |
| **NFR27** | **Graceful degradation** | Non-critical services can fail without stopping payments | Webhook dispatcher down? Payments still process. Dashboard down? Payments still process |
| **NFR28** | **Disaster recovery** | RTO < 15 min (Recovery Time Objective) | How fast can we restore service after a region failure? |

---

## FR vs NFR — How They Shape the Design

> Every NFR constrains your design choices. Here's how they connect:

| NFR | Design Decision It Forces |
|-----|--------------------------|
| **p99 < 1s latency** | Accept payment synchronously → process asynchronously. Can't block on PSP + bank call |
| **99.99% availability** | Redundancy, multi-AZ deployment, health checks, circuit breakers |
| **Zero data loss** | Kafka event persistence before processing. Synchronous DB replication for primary |
| **Strong consistency (payments)** | PostgreSQL with ACID transactions, not eventually-consistent NoSQL |
| **Eventual consistency (dashboard)** | Separate read model, CDC pipeline, materialized views — don't query the primary |
| **10x peak handling** | Horizontal auto-scaling, Kafka buffering, rate limiting to protect backend |
| **PCI DSS** | PSP hosted payment page, tokenization, never store raw card numbers |
| **UK GDPR + data residency** | Pseudonymization for right-to-erasure, UK data centres, active-passive with region-aware routing |
| **PSD2 / SCA** | 3DS2 via PSP, mandatory for UK card payments, exemptions managed by PSP |
| **Idempotency** | UUID idempotency key from client, DB unique constraint, Redis SETNX for fast duplicate detection |
| **Graceful degradation** | Circuit breakers, fallbacks, DLQ — non-critical paths can fail independently |

---

## The UK GDPR vs Append-Only Ledger Conflict

> This is a **principal-level** talking point that impresses interviewers.

**The problem:**
- NFR11 says the ledger is **append-only** — never delete financial records
- NFR18 says **UK GDPR** requires **right to erasure** — customers can request their data be deleted

**These directly conflict.** Here's how you resolve it:

| Approach | How | Trade-Off |
|----------|-----|-----------|
| **Pseudonymization** | Replace customer PII with a token. Delete the mapping table on erasure request. Ledger entries remain but can't be linked to a person | Ledger intact for auditing, PII removed. Regulators generally accept this |
| **Crypto-shredding** | Encrypt PII with a per-customer key. On erasure, delete the key. Data exists but is unreadable | Strongest guarantee. Key management adds complexity |
| **Soft delete with retention** | Mark as deleted, purge after legal retention period expires | Simplest. Risk: data exists during retention window |

> **Principal answer:** "I'd use pseudonymization as the default — replace PII with opaque tokens in the ledger, store the mapping separately. On a GDPR erasure request, delete the mapping. The ledger remains intact for financial auditing, but the data is no longer personally identifiable. For the most sensitive cases, crypto-shredding gives us the strongest guarantee."

---

## How to Present NFRs in the Interview

> State concrete numbers, not vague goals:

```
"For non-functional requirements, I'm targeting:
 - 99.99% availability for the payment API — downtime directly impacts merchant revenue
 - p99 latency under 1 second for payment creation — we'll accept synchronously 
   and process asynchronously since the PSP call alone takes 1-3 seconds
 - Zero data loss — payment events are persisted to Kafka before we consider 
   the operation complete
 - Strong consistency for payment state, eventual consistency for the dashboard 
   with a lag target under 5 seconds
 - Horizontal scaling to handle 10x peak traffic without degradation
 - UK compliance: PCI DSS via PSP hosted page, PSD2/SCA via 3DS, UK GDPR via 
   pseudonymization, FCA regulations via append-only double-entry ledger

Does this align with what you'd expect, or should I adjust any of these targets?"
```

> End with a question — keep it a **two-way conversation**.

---

## Quick Reference — FR and NFR at a Glance

### Functional (What)

| Area | Core Features |
|------|--------------|
| **Payment API** | Create, status, refund, cancel, idempotency, webhooks |
| **Dashboard** | Transaction list, detail view, aggregations, export, roles |

### Non-Functional (How Well)

| Category | Key Target |
|----------|-----------|
| **Performance** | p99 < 1s payment, 1K–10K TPS, 10x peak |
| **Availability** | 99.99% API, 99.9% dashboard, 99.999% Kafka |
| **Consistency** | Strong for payments, eventual for dashboard |
| **Durability** | Zero data loss, append-only ledger, RPO = 0 |
| **Security** | PCI DSS, TLS/mTLS, UK GDPR, PSD2/SCA, FCA, rate limiting |
| **Observability** | Metrics + tracing + alerting (3 pillars) |
| **Reliability** | Idempotency, graceful degradation, RTO < 15 min |

---

**Next:** [08-trade-offs.md](./08-trade-offs.md) — Trade-offs for every architectural decision
