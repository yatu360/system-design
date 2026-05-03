# Bank-to-Inventory Payment System — Saga Pattern

> What changes when the interview asks you to design a payment system between a bank and an inventory system, instead of a PSP-based merchant payment API.

---

## How This Differs From Our PSP-Based Design

Our main design routes payments through a PSP to card networks:

```
Customer → API Gateway → Payment Service → PSP → Acquiring Bank → Card Network → Issuing Bank
```

A bank-to-inventory design talks directly to a bank and must coordinate with an inventory system:

```
Customer → API Gateway → Order Service → Payment Service → Bank API
                              │
                              └──────────► Inventory Service
```

**The core problem shifts.** In the PSP design, the hard problem is "don't lose money, don't charge twice." In the bank-to-inventory design, the hard problem is **"don't sell what you don't have, don't charge for what you can't ship"** — a distributed transaction across two independent systems.

---

## What Changes, What Stays

| Aspect | PSP-Based (Our Main Design) | Bank-to-Inventory |
|---|---|---|
| **PSP** | Central — handles card data, 3DS, tokenization | No PSP — direct bank API (A2A transfer, direct debit, or Open Banking) |
| **Card networks** | Visa/Mastercard routing | Not involved — bank transfer or direct debit |
| **PCI DSS** | Offloaded to PSP hosted page | Likely not relevant — no card data |
| **3DS / SCA** | Mandatory via PSP | May apply via Open Banking consent flow, but different mechanism |
| **Payment lifecycle** | PENDING → AUTHORIZED → CAPTURED → SETTLED | PENDING → RESERVED → DEBITED → CONFIRMED |
| **Inventory** | Not our concern — merchant handles stock | Critical — must coordinate stock and payment atomically |
| **Core challenge** | Idempotency + reliability | Distributed transaction (Saga pattern) |

### What Stays the Same

These patterns carry over directly:

- **Kafka** — event backbone for all state transitions
- **Idempotency** — Redis SETNX + DB constraint (don't charge twice, don't reserve twice)
- **Retry + exponential backoff + jitter** — bank APIs fail too
- **Circuit breaker** — if the bank is down, stop calling it
- **DLQ** — for steps that fail permanently
- **Append-only ledger** — audit trail of every financial movement
- **PostgreSQL** — ACID for payment and order state
- **Observability** — metrics, traces, alerting

---

## The Core Problem: Distributed Transaction

Two separate services must agree on the outcome:

```
Inventory Service: "I reserved 1x Widget for order_123"
Payment Service:   "I charged £50 for order_123"

Both must succeed, or BOTH must roll back.
```

You can't wrap this in a single database transaction because they're separate services with separate databases. This is where the **Saga pattern** comes in.

---

## The Saga Pattern — Explained Simply

A saga is a sequence of steps where each step has a **compensating action** that undoes it if a later step fails.

Think of it like booking a holiday:
1. Book the flight
2. Book the hotel
3. Book the car

If the car rental fails, you cancel the hotel, then cancel the flight. Each cancellation is a "compensating action."

### Two Approaches

| Approach | How It Works | When to Use |
|---|---|---|
| **Orchestration** (recommended) | A central coordinator tells each service what to do and what to undo | When you need clear visibility, control, and debugging — payment systems |
| **Choreography** | Each service emits events, the next service reacts on its own | When services are truly independent and loosely coupled |

**For payments, always use orchestration.** Money requires clear control flow and easy debugging. With choreography, tracing "why was this customer charged but didn't get their item?" becomes a nightmare.

---

## The Full Saga Flow

### Happy Path

```
Order Service (Saga Orchestrator)
    │
    │  Step 1: RESERVE INVENTORY
    │  ──────────────────────────
    ├──► Inventory Service: "Reserve 1x Widget for order_123"
    │    ← "Reserved. Reservation expires in 15 minutes."
    │
    │  Step 2: CHARGE CUSTOMER
    │  ────────────────────────
    ├──► Payment Service: "Charge £50 for order_123"
    │    ├──► Bank API: initiate debit
    │    ← "Payment successful. Reference: bank_ref_456"
    │
    │  Step 3: CONFIRM ORDER
    │  ──────────────────────
    ├──► Inventory Service: "Confirm reservation for order_123"
    │    ← "Confirmed. Stock decremented."
    │
    │  Step 4: TRIGGER SHIPMENT
    │  ────────────────────────
    └──► Fulfilment Service: "Ship order_123 to customer"
         ← "Shipment scheduled."

    Order status: PENDING → RESERVED → PAID → CONFIRMED → SHIPPED
```

### Failure at Each Step (Compensating Actions)

```
Step 1 fails (can't reserve inventory):
    → Abort. Nothing to compensate. Tell customer "out of stock."

Step 2 fails (bank declines payment):
    → Compensate Step 1: release inventory reservation
    → Tell customer "payment failed."

Step 3 fails (inventory confirmation crashes):
    → Compensate Step 2: refund payment
    → Compensate Step 1: release inventory reservation
    → Tell customer "order failed, refund initiated."

Step 4 fails (fulfilment unavailable):
    → Compensate Step 3: restore inventory
    → Compensate Step 2: refund payment
    → Tell customer "order cancelled, refund initiated."
```

**Each step has exactly one compensating action:**

| Step | Action | Compensating Action |
|---|---|---|
| 1. Reserve inventory | `POST /inventory/reserve` | `POST /inventory/release` |
| 2. Charge customer | `POST /payments/charge` | `POST /payments/refund` |
| 3. Confirm order | `POST /inventory/confirm` | `POST /inventory/restore` |
| 4. Ship | `POST /fulfilment/ship` | `POST /fulfilment/cancel` |

---

## Saga State Machine

The orchestrator tracks the saga's progress in PostgreSQL:

```
┌─────────┐     reserve      ┌──────────┐     charge      ┌────────┐
│ PENDING │ ───inventory────► │ RESERVED │ ───customer────► │  PAID  │
└─────────┘                   └──────────┘                  └────────┘
                                   │                            │
                              reserve fails                charge fails
                                   │                            │
                                   ▼                            ▼
                              ┌─────────┐               ┌────────────┐
                              │ FAILED  │               │COMPENSATING│
                              │(nothing │               │(releasing  │
                              │ to undo)│               │ inventory) │
                              └─────────┘               └─────┬──────┘
                                                              │
     ┌──────────┐    confirm     ┌───────────┐                ▼
     │ SHIPPED  │◄──shipment──── │ CONFIRMED │          ┌─────────┐
     └──────────┘                └───────────┘          │ FAILED  │
                                      │                 │(refunded│
                                 confirm fails          │+released)│
                                      │                 └─────────┘
                                      ▼
                                ┌────────────┐
                                │COMPENSATING│
                                │(refunding  │
                                │ + releasing)│
                                └─────┬──────┘
                                      │
                                      ▼
                                ┌─────────┐
                                │ FAILED  │
                                └─────────┘
```

### Saga State Table (PostgreSQL)

```sql
sagas (
    id              UUID PRIMARY KEY,
    order_id        UUID NOT NULL UNIQUE,     -- idempotency: one saga per order
    status          VARCHAR(20) NOT NULL,      -- PENDING, RESERVED, PAID, CONFIRMED, SHIPPED, COMPENSATING, FAILED
    current_step    INTEGER NOT NULL,          -- which step we're on (1-4)
    inventory_ref   VARCHAR(255),              -- reservation ID from inventory service
    payment_ref     VARCHAR(255),              -- payment reference from bank
    failure_reason  TEXT,                      -- why it failed
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);

saga_steps (
    id              UUID PRIMARY KEY,
    saga_id         UUID NOT NULL REFERENCES sagas(id),
    step_number     INTEGER NOT NULL,
    action          VARCHAR(50) NOT NULL,      -- RESERVE, CHARGE, CONFIRM, SHIP
    status          VARCHAR(20) NOT NULL,      -- PENDING, COMPLETED, FAILED, COMPENSATED
    request_payload JSONB,
    response_payload JSONB,
    created_at      TIMESTAMP NOT NULL
);
```

---

## How Kafka Fits

```
Order Service (Saga Orchestrator)
    │
    │  produces saga events
    ▼
┌─────────────────────────────────────────────────────┐
│  Topic: order-events                                │
│  Partition key: order_id                            │
│                                                     │
│  [order_123: CREATED]                               │
│  [order_123: INVENTORY_RESERVED]                    │
│  [order_123: PAYMENT_CHARGED]                       │
│  [order_123: ORDER_CONFIRMED]                       │
│  [order_123: SHIPPED]                               │
└──────────┬──────────────┬───────────────────────────┘
           │              │
           ▼              ▼
    ┌────────────┐  ┌────────────┐
    │ ledger-    │  │ webhook-   │
    │ service    │  │ dispatcher │
    └────────────┘  └────────────┘

┌─────────────────────────────────────────────────────┐
│  Topic: saga-commands                               │
│  Partition key: order_id                            │
│                                                     │
│  [order_123: RESERVE_INVENTORY]                     │
│  [order_123: CHARGE_CUSTOMER]                       │
│  [order_123: CONFIRM_ORDER]                         │
│  [order_123: SHIP_ORDER]                            │
│                                                     │
│  On failure:                                        │
│  [order_123: RELEASE_INVENTORY]   ← compensating    │
│  [order_123: REFUND_PAYMENT]      ← compensating    │
└──────────┬──────────────┬───────────────────────────┘
           │              │
           ▼              ▼
    ┌────────────┐  ┌────────────┐
    │ inventory- │  │ payment-   │
    │ service    │  │ service    │
    └────────────┘  └────────────┘
```

**Partition key = `order_id`** — all saga events for one order stay in the same partition, processed in order. Same principle as `payment_id` in the PSP design.

---

## Inventory-Specific Patterns

### Reservation with TTL

Don't hold inventory forever. If the payment doesn't complete, auto-release:

```
Reserve:   "Hold 1x Widget for order_123. Expires in 15 minutes."
    │
    ├── Payment completes within 15 min → confirm reservation
    │
    └── Payment fails OR 15 min passes → reservation auto-expires
                                         stock becomes available again
```

This prevents "phantom stock" — items reserved but never purchased, appearing sold out to other customers.

### Stock Consistency

```
Optimistic locking (recommended for low contention):
    UPDATE inventory SET quantity = quantity - 1, version = version + 1
    WHERE product_id = 'widget' AND quantity > 0 AND version = 5
    
    If version changed (someone else reserved first) → retry

Pessimistic locking (for high contention / flash sales):
    SELECT quantity FROM inventory WHERE product_id = 'widget' FOR UPDATE
    -- row is locked until transaction commits
```

| Approach | When | Trade-off |
|---|---|---|
| **Optimistic** | Normal traffic — most reservations don't conflict | Retry on conflict, but no lock contention |
| **Pessimistic** | Flash sales — many customers competing for same item | Guaranteed consistency, but locks slow down throughput |

---

## Failure Scenarios

| Failure | What Happens | Recovery |
|---|---|---|
| **Inventory service down** | Saga can't start (Step 1 fails) | Return error to customer. Retry via Kafka if async. |
| **Bank API down** | Step 2 fails after inventory reserved | Compensate: release reservation. Circuit breaker on bank API. |
| **Bank timeout (ambiguous)** | Don't know if charge succeeded | Check bank status API with idempotency key. If charged → continue. If not → release inventory. |
| **Orchestrator crashes mid-saga** | Saga state is in PostgreSQL | On restart, read saga state, resume from current step or compensate. |
| **Inventory confirmation fails** | Step 3 fails after payment charged | Compensate: refund payment + release reservation. |
| **Reservation expires** | Customer took too long | Payment hasn't happened yet → no compensation needed. If payment was in-flight → refund. |
| **Double reservation** | Same order submitted twice | Idempotency: `order_id` unique constraint on saga table. Second request returns existing saga. |

---

## Architecture Diagram

```
                     SYNCHRONOUS PATH
                     =================

┌──────────┐      ┌───────────────┐      ┌──────────────────┐
│ Customer  │─────►│  API Gateway  │─────►│  Order Service    │
│           │      │ • TLS         │      │ (saga orchestrator)│
└──────────┘      │ • OAuth/JWT   │      └──────┬───────────┘
                  │ • Rate limit  │      ┌──────┼──────────────┐
                  └───────────────┘      │      │              │
                                          ▼      ▼              ▼
                                    ┌────────┐┌──────┐   ┌──────────┐
                                    │Postgres││Kafka │   │  Redis   │
                                    │(saga   ││(events)   │(idempot. │
                                    │ state) │└──┬───┘   │ + cache) │
                                    └────────┘   │       └──────────┘
                                                 │
           ════════════════ ASYNC BOUNDARY ════════════════
                                                 │
                     SAGA STEPS (via Kafka)
                     ======================
                                                 │
                          ┌──────────────────────┼───────────────┐
                          ▼                      ▼               ▼
                   ┌────────────┐         ┌────────────┐  ┌────────────┐
                   │ Inventory  │         │  Payment   │  │ Fulfilment │
                   │ Service    │         │  Service   │  │ Service    │
                   │            │         │            │  │            │
                   │ • reserve  │         │ • charge   │  │ • ship     │
                   │ • confirm  │         │ • refund   │  │ • cancel   │
                   │ • release  │         │            │  │            │
                   └──────┬─────┘         └──────┬─────┘  └────────────┘
                          │                      │
                          ▼                      ▼
                   ┌────────────┐         ┌────────────┐
                   │ Inventory  │         │   Bank     │
                   │ Database   │         │   API      │
                   │ (stock)    │         │(direct debit│
                   └────────────┘         │ or A2A)    │
                                          └────────────┘

                     POST-PROCESSING
                     ================

                          Kafka ──────┬──────────────┐
                                      │              │
                                      ▼              ▼
                               ┌────────────┐ ┌────────────┐
                               │  Ledger    │ │  Webhook   │
                               │ (append-   │ │ Dispatcher │
                               │  only)     │ │(notify     │
                               └────────────┘ │ customer)  │
                                              └────────────┘
```

---

## Interview Questions & Answers

**"Why not use a distributed transaction (2PC)?"**
> "Two-phase commit locks all participating services until the coordinator decides commit or abort. In a payment system, that means the bank API and inventory database are both locked during the entire transaction — terrible for throughput and availability. If the coordinator crashes, everything stays locked. Sagas avoid this by using compensating actions instead of global locks."

**"Why orchestration over choreography?"**
> "With money involved, I need a single place that knows the full saga state — which steps completed, which failed, what needs compensating. Choreography spreads this across services via events, making it hard to answer 'why was this customer charged but didn't get their item?' Orchestration gives me a saga table I can query directly."

**"What if the compensating action fails?"**
> "Retry with idempotency. If the refund call to the bank fails, retry with the same idempotency key. If it keeps failing, send to DLQ and alert. The saga stays in COMPENSATING state until resolved — either automatically or by ops."

**"How do you handle the bank timeout?"**
> "Same as PSP timeout in our main design. Don't assume failure — check the bank's status API with the idempotency key. If charged, continue the saga. If not charged, compensate. The ambiguous state is the most dangerous — never guess, always verify."

**"What about inventory overselling?"**
> "Reservation with TTL. Reserve stock for 15 minutes. If payment doesn't complete, reservation expires and stock returns to available. Combined with optimistic locking on the inventory table — UPDATE only if version matches. Under extreme contention like flash sales, switch to pessimistic locking (SELECT FOR UPDATE)."

**"How is this different from your PSP design?"**
> "The core reliability patterns are identical — Kafka, idempotency, retry, circuit breaker, DLQ. The difference is the distributed transaction problem. In the PSP design, the Payment Service is a single orchestrator calling one external system. Here, I'm coordinating two independent services — inventory and payment — that must agree on the outcome. The Saga pattern solves this with compensating actions instead of distributed locks."

---

**Related docs:**
- [00-payment-system-overview.md](00-payment-system-overview.md) — PSP-based architecture (compare)
- [01-reliability-patterns.md](01-reliability-patterns.md) — Retry, circuit breaker, DLQ (reused here)
- [02-idempotency.md](02-idempotency.md) — Two-layer idempotency (reused here)
- [12-kafka-fundamentals.md](12-kafka-fundamentals.md) — Kafka partitioning and consumer groups (reused here)
