# Payment System — Interview Cheat Sheet

> Everything you need on one page. Scan this 10 minutes before your interview.

---

## Architecture (Draw This)

```
┌──────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Customer  │────►│ Payment Service   │────►│     PSP      │────►│Acquiring Bank│────►│ Issuing Bank │
│           │     │ (orchestrator)    │     │ (hosted page)│     │(merchant's)  │     │(customer's)  │
└──────────┘     └──────┬───────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                        │
                 ┌──────┴──────┐
                 │             │
                 ▼             ▼
           ┌──────────┐ ┌──────────┐
           │  Wallet   │ │  Ledger  │
           │ (balance) │ │ (audit)  │
           └──────────┘ └──────────┘
                 │
                 ▼
              Kafka
           (event store)
```

---

## Top 10 Keywords to Drop

| # | Keyword | Context |
|---|---------|---------|
| 1 | **Idempotency Key (UUID)** | "Client generates a UUID, sends it in the HTTP header. DB unique constraint guarantees exactly-once." |
| 2 | **Exponential Backoff** | "Best retry strategy — doubles wait time each attempt (2^n). Prevents thundering herd." |
| 3 | **Exactly-Once Guarantee** | "Achieved via idempotency key + DB unique constraint. Not distributed locks." |
| 4 | **Kafka (99.999%)** | "Payment event persisted to Kafka before operation is considered complete." |
| 5 | **Dead Letter Queue** | "Isolates poison pill messages — malformed data saved for debugging without blocking traffic." |
| 6 | **PSP Hosted Page** | "Merchant never touches card data. Offloads PCI DSS compliance to PSP." |
| 7 | **TLS + VPN** | "Data-in-transit: TLS for confidentiality/integrity/auth. VPN for secure connections." |
| 8 | **Async-First** | "Default to async for scalability and fault tolerance. Sync only for physical store real-time auth." |
| 9 | **Ledger (Record-by-Record)** | "Append-only log of every transaction. Revenue analysis + auditing." |
| 10 | **PCI DSS / GDPR** | "Payment Gateway enforces compliance. PSP hosted page keeps merchant out of scope." |

---

## The "How Do You Handle X?" Quick Answers

### "How do you prevent double payments?"
> **Idempotency key** (UUID) in HTTP header → same key used as payment order ID → **DB unique constraint** rejects duplicates → **exactly-once guarantee**.

### "What happens when a service goes down?"
> **Kafka persists events** (99.999% uptime). Retry with **exponential backoff**. If retries exhausted → **dead letter queue** for debugging. **Fallback values** if acceptable.

### "How do you handle timeouts?"
> Set balanced timeouts. On timeout, **retry with the same idempotency key**. If the original succeeded → duplicate rejected. If it failed → retry succeeds. **Either way, charged once.**

### "How do you ensure data consistency?"
> **Ledger service** records every transaction **record-by-record** (append-only). **Kafka events** guarantee no payment is lost. **Reconciliation tools** from PSP catch discrepancies.

### "How do you scale?"
> **Async communication** prevents bottlenecks. **Kafka** decouples producers/consumers. **Loose coupling** lets services scale independently. **Background processing** frees resources.

### "How do you secure card data?"
> Merchant **never handles card data** — PSP's **hosted payment page** collects it. **TLS** in transit, **encryption at rest**. Payment Gateway enforces **PCI DSS / GDPR**.

### "Sync or async?"
> **Async by default** — scalability, fault tolerance, loose coupling. **Sync only** for physical store payments requiring **real-time authorization**.

---

## Error Handling Matrix

| Error Type | Example | Pattern | Outcome |
|------------|---------|---------|---------|
| **Transient** (network blip) | Connection reset | **Exponential backoff** + **timeout** | Auto-recovers |
| **Timeout** (ambiguous) | No response | **Retry + idempotency key** | Charged exactly once |
| **Poison pill** (bad data) | Malformed message | **Dead letter queue** | Isolated for debugging |
| **Service down** | PSP unreachable | **Persistent queue** + **fallback** | Retries when service returns |
| **Double request** (user spam) | Click-click-click | **Idempotency key** + **DB constraint** | Only first processed |

---

## The Reliability Stack (Order Matters)

```
Layer 1: KAFKA           ──► "No event is ever lost"
Layer 2: IDEMPOTENCY     ──► "No payment happens twice"
Layer 3: RETRY           ──► "Transient failures recover automatically"
Layer 4: TIMEOUT         ──► "Nothing waits forever"
Layer 5: FALLBACK        ──► "Graceful degradation when acceptable"
Layer 6: DEAD LETTER Q   ──► "Bad messages don't block good ones"
Layer 7: MONITORING      ──► "We know when something is wrong"
```

---

## Non-Functional Requirements Checklist

- [ ] **Reliability** — Kafka persistence, retry strategies, DLQ
- [ ] **Idempotency** — UUID keys, DB unique constraints, exactly-once
- [ ] **Security** — TLS, encryption at rest, PSP hosted page, PCI DSS
- [ ] **Scalability** — Async-first, Kafka decoupling, loose coupling
- [ ] **Auditability** — Ledger service, append-only records
- [ ] **Fault Tolerance** — Exponential backoff, fallback, dead letter queue
- [ ] **Compliance** — PCI DSS (card data), GDPR (privacy)

---

## File Index

| File | Topic |
|------|-------|
| [00-payment-system-overview.md](00-payment-system-overview.md) | Architecture, components, payment flow |
| [01-reliability-patterns.md](01-reliability-patterns.md) | Retry, timeout, fallback, Kafka, DLQ |
| [02-idempotency.md](02-idempotency.md) | Exactly-once guarantee, UUID keys, DB constraints |
| [03-security.md](03-security.md) | Encryption, TLS, PCI DSS, access controls |
| [04-interview-cheat-sheet.md](./04-interview-cheat-sheet.md) | This file — quick reference for interviews |
