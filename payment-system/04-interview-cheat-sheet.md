# Payment System — Interview Cheat Sheet

> Scan this 10 minutes before your interview. Updated architecture, UK compliance, full payment lifecycle.

---

## Architecture (Draw This)

```
┌──────────┐     ┌───────────────┐     ┌──────────────────┐
│ Customer  │────►│  API Gateway  │────►│ Payment Service   │
│           │     │ • OAuth/JWT   │     │ (orchestrator)    │
└──────────┘     │ • Rate limit  │     └──────┬───────────┘
                 │ • TLS termination│          │
                 │ • WAF         │     ┌──────┼──────────────┐
                 └───────────────┘     │      │              │
                                       ▼      ▼              ▼
┌──────────┐                     ┌────────┐┌──────┐   ┌──────────┐
│Dashboard │◄── Read Replica / ──│Postgres││Kafka │   │  Redis   │
│  (Web)   │    Data Warehouse   │ (ACID) ││(events)   │ (cache)  │
└──────────┘                     └────────┘└──┬───┘   └──────────┘
                                              │
                                   ┌──────────┴──────────┐
                                   ▼                     ▼
                            ┌────────────┐        ┌────────────┐
                            │  Payment   │        │  Webhook   │
                            │ Processor  │        │ Dispatcher │
                            └──────┬─────┘        └──────┬─────┘
                                   │                     │
                                   ▼                     ▼
                            ┌────────────┐        Merchant's
                            │  PSP       │        endpoint
                            │(3DS + hosted page)
                            └──────┬─────┘
                          ┌────────┴────────┐
                          ▼                 ▼
                   ┌────────────┐    ┌────────────┐
                   │ Acquiring  │───►│  Issuing   │
                   │   Bank     │card│   Bank     │
                   │(merchant's)│net │(customer's)│
                   └────────────┘    └────────────┘

Post-Processing:  Kafka ──► Wallet + Ledger + Data Warehouse
```

---

## Payment Lifecycle (Know This Cold)

```
PENDING → AUTHORIZED → CAPTURED → SETTLED → (REFUNDED or DISPUTED)
               │            │
               ▼            │   Auth = reserve funds (no money moves)
            VOIDED          │   Capture = charge the card
          (release hold)    │   Settlement = bank transfer (T+1 to T+3)
                            │   Void = release hold (cheaper than refund)
                            │
                       DISPUTED → CB_WON or CB_LOST
```

---

## Top 12 Keywords to Drop

| # | Keyword | Context |
|---|---------|---------|
| 1 | **Idempotency Key (UUID)** | "Client generates UUID in HTTP header. Redis SETNX for speed, DB unique constraint as safety net. Exactly-once." |
| 2 | **Auth → Capture → Settle** | "Authorization reserves funds, capture charges, settlement transfers. Not a single step." |
| 3 | **Exponential Backoff + Jitter** | "Doubles wait time (2^n) with random jitter to prevent thundering herd." |
| 4 | **Kafka (99.999%)** | "Payment event persisted to Kafka before operation is considered complete." |
| 5 | **3DS / SCA** | "Mandatory under UK PSD2. Two-factor auth on card payments. PSP handles the flow. Shifts fraud liability to issuer." |
| 6 | **Double-Entry Ledger** | "Every transaction: debit + credit. Total debits = total credits. Append-only. Self-auditing." |
| 7 | **PSP Hosted Page + Tokenization** | "Merchant never touches card data. PSP stores real card, returns token. Reduces PCI DSS scope." |
| 8 | **TLS + mTLS** | "TLS for external traffic. mTLS for internal service-to-service. Zero-trust network." |
| 9 | **Dead Letter Queue** | "Isolates poison pill messages for debugging without blocking healthy traffic." |
| 10 | **Circuit Breaker** | "Closed → Open → Half-Open. Prevents cascading failures. Per-merchant breakers for webhooks." |
| 11 | **CAP Per Operation** | "Payment writes = CP. Dashboard reads = AP. Different trade-offs per data path, not per system." |
| 12 | **PCI DSS + PSD2/SCA + UK GDPR** | "PSP hosted page for PCI. 3DS for SCA. Pseudonymization for GDPR right-to-erasure vs append-only ledger." |

---

## The "How Do You Handle X?" Quick Answers

### "How do you prevent double payments?"
> Two-layer idempotency: **Redis SETNX** catches 99% of duplicates in sub-ms. **DB unique constraint** on the payment order ID (= idempotency key) is the ultimate safety net. **Exactly-once guarantee.**

### "Walk me through the payment lifecycle."
> **Authorization** reserves funds on the card — no money moves. **Capture** commits the charge. **Settlement** transfers funds between banks (T+1 to T+3). Merchant can **void** before capture to release the hold (cheaper than refund — no interchange fees).

### "What about chargebacks?"
> Customer disputes through issuing bank → funds reversed from merchant → merchant submits evidence → bank decides. System needs `DISPUTED`, `CB_WON`, `CB_LOST` states. **Chargeback rate >1% triggers card network penalties.** Dashboard surfaces this per merchant.

### "How do you handle 3DS / SCA?"
> **Mandatory under PSD2 in the UK.** PSP handles the 3DS flow — customer authenticates via biometric/OTP with their issuing bank. Exemptions exist for low-value (<£30), recurring, and trusted merchants. **3DS shifts fraud liability from merchant to issuer.**

### "What happens when a service goes down?"
> **Kafka persists events** (99.999%). Retry with **exponential backoff + jitter**. **Circuit breaker** prevents cascading. If retries exhausted → **DLQ** for investigation. Non-critical services (webhooks, dashboard) can fail without stopping payments.

### "How do you ensure data consistency?"
> **Double-entry ledger** — every transaction has balanced debit + credit entries. **Append-only** — never update or delete. **Reconciliation** runs daily in the data warehouse, matching internal ledger against PSP settlement reports and bank statements.

### "How do you secure card data?"
> Merchant **never handles card data** — PSP's hosted page collects it and returns a **token**. Tokens stored in our DB, real card numbers only at PSP. **TLS** external, **mTLS** internal. **PCI DSS** compliance offloaded to PSP.

### "How do you handle UK compliance?"
> **PCI DSS** via PSP hosted page + tokenization. **PSD2/SCA** via 3DS (PSP handles). **UK GDPR** via pseudonymization for right-to-erasure vs append-only ledger conflict. **FCA** regulated — proper financial record-keeping via double-entry ledger.

---

## Error Handling Matrix

| Error Type | Example | Pattern | Outcome |
|------------|---------|---------|---------|
| **Transient** (network blip) | Connection reset | **Exponential backoff + jitter** | Auto-recovers |
| **Timeout** (ambiguous) | No response | **Retry + idempotency key** | Charged exactly once |
| **Poison pill** (bad data) | Malformed message | **Dead letter queue** | Isolated for debugging |
| **Service down** | PSP unreachable | **Circuit breaker** + **DLQ** | Queued, processed when recovered |
| **Double request** (user spam) | Click-click-click | **Redis SETNX + DB constraint** | Only first processed |
| **Chargeback** | Customer disputes | **Webhook from PSP** → state machine | Evidence submitted, outcome tracked |

---

## The Reliability Stack (Order Matters)

```
Layer 1: KAFKA             ──► "No event is ever lost"
Layer 2: IDEMPOTENCY       ──► "No payment happens twice" (Redis + DB)
Layer 3: CIRCUIT BREAKER   ──► "Failing services don't cascade"
Layer 4: RETRY + JITTER    ──► "Transient failures recover without thundering herd"
Layer 5: TIMEOUT           ──► "Nothing waits forever"
Layer 6: FALLBACK          ──► "Graceful degradation when acceptable"
Layer 7: DEAD LETTER Q     ──► "Bad messages don't block good ones"
Layer 8: MONITORING        ──► "We know when something is wrong"
```

---

## UK Compliance Quick Reference

| Regulation | What | How We Handle It | Technical Implementation |
|-----------|------|-----------------|--------------------------|
| **PCI DSS** | Card data security | PSP hosted page + tokenization — merchant never touches card numbers | PSP iframe/redirect collects card details → returns `tok_xxx` token → stored in PostgreSQL. Real card in PSP vault only. Reduces our PCI scope to SAQ-A |
| **PSD2 / SCA** | Strong Customer Authentication | 3DS via PSP — biometric/OTP. Mandatory for UK/EU card payments | PSP triggers 3DS2 challenge during checkout → customer authenticates with issuing bank (biometric/SMS OTP) → PSP returns auth result. Exemptions (low-value <£30, recurring, TRA) managed by PSP's risk engine |
| **UK GDPR** | Data privacy, right to erasure | Pseudonymization — delete PII mapping, ledger entries remain anonymized | PII stored in separate `customer_pii` table linked by opaque `customer_token`. On erasure request: DELETE from `customer_pii` → ledger entries remain but are unidentifiable. Alternative: crypto-shredding (per-customer encryption key, destroy key on erasure) |
| **FCA** | Financial Conduct Authority regulation | Double-entry ledger, reconciliation, proper financial record-keeping | Append-only `ledger_entries` table — every transaction = balanced debit + credit rows. No UPDATE/DELETE. Daily reconciliation job in data warehouse cross-references internal ledger vs PSP settlement reports vs bank statements. Discrepancies trigger P2 alerts |
| **Open Banking** | API access to bank accounts | Relevant if supporting bank-to-bank (A2A) payments | Account Information Service (AIS) and Payment Initiation Service (PIS) APIs via Open Banking aggregator. Alternative rail to card networks — lower fees, no interchange |

---

## Non-Functional Requirements Checklist

- [ ] **Reliability** — Kafka persistence, retry + jitter, circuit breakers, DLQ
- [ ] **Idempotency** — Redis SETNX + DB unique constraints, exactly-once
- [ ] **Security** — TLS/mTLS, encryption at rest, PSP hosted page, tokenization
- [ ] **Scalability** — Async-first, Kafka decoupling, K8s auto-scaling
- [ ] **Auditability** — Double-entry append-only ledger, reconciliation
- [ ] **Fault Tolerance** — Circuit breakers, exponential backoff + jitter, fallback, DLQ
- [ ] **Compliance** — PCI DSS, PSD2/SCA (3DS), UK GDPR, FCA
- [ ] **Observability** — Metrics + tracing + alerting (three pillars)
- [ ] **Payment Lifecycle** — Auth → capture → settle, void, refund, chargeback

---

## File Index

| File | Topic |
|------|-------|
| [00-payment-system-overview.md](00-payment-system-overview.md) | Architecture, components, payment lifecycle, 3DS, chargebacks |
| [01-reliability-patterns.md](01-reliability-patterns.md) | Retry + jitter, timeout, fallback, circuit breakers, Kafka, DLQ |
| [02-idempotency.md](02-idempotency.md) | Exactly-once guarantee, Redis + DB two-layer approach |
| [03-security.md](03-security.md) | Encryption, TLS/mTLS, PCI DSS, tokenization, UK compliance |
| [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) | This file — quick reference for interviews |
| [05-recruiter-briefing.md](05-recruiter-briefing.md) | Recruiter's guidance, checklist, study priorities |
| [06-values-interview.md](06-values-interview.md) | Values interview (STAR framework) |
| [07-data-layer.md](07-data-layer.md) | SQL vs NoSQL, CAP theorem, double-entry ledger, reconciliation, tokenization |
| [08-trade-offs.md](08-trade-offs.md) | Stage-by-stage trade-offs, circuit breakers, observability, infra |
| [09-requirements-fr-nfr.md](09-requirements-fr-nfr.md) | Functional & non-functional requirements with targets |
| [10-requirement-gathering-questions.md](10-requirement-gathering-questions.md) | Checkout.com-specific requirement gathering questions |
| [11-interview-walkthrough.md](11-interview-walkthrough.md) | Full interview flow — what to say, in order, with trade-offs |
| [12-kafka-fundamentals.md](12-kafka-fundamentals.md) | Kafka core concepts, partition ordering, replication, delivery semantics |
| [13-kafka-cheat-sheet.md](13-kafka-cheat-sheet.md) | Kafka quick reference — 10 key concepts, config, failure scenarios |
| [14-bank-inventory-saga.md](14-bank-inventory-saga.md) | Bank-to-inventory design — Saga pattern, compensating actions |
