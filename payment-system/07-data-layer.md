# Data Layer — Databases for Payment Systems

---

## The One-Liner

> A payment system needs **ACID guarantees** for money movement, **low-latency reads** for real-time lookups, and **append-only storage** for audit trails. No single database does all three well — you pick the right tool for each job.

---

## SQL vs NoSQL — The Core Trade-Off

| | **SQL (Relational)** | **NoSQL (Non-Relational)** |
|---|---|---|
| **Data model** | Tables, rows, columns with **rigid schema** | Documents, key-value, wide-column, graph — **flexible schema** |
| **Consistency** | **Strong consistency** by default (ACID) | **Eventual consistency** by default (BASE) — some offer strong consistency at cost |
| **Scaling** | **Vertical** primarily (scale up). Horizontal is possible but complex (sharding) | **Horizontal** natively (scale out). Built for distributed clusters |
| **Joins** | Native, efficient **multi-table joins** | No joins or expensive — data is **denormalized** |
| **Schema changes** | Migrations required — **rigid but safe** | Schema-less — **flexible but risky** |
| **Transactions** | Multi-row, multi-table **ACID transactions** built-in | Limited transaction support (some offer single-document atomicity) |
| **Query language** | Standardized **SQL** | Proprietary APIs, varies per database |
| **Best for** | Complex relationships, transactions, consistency | High throughput, flexible data, massive scale |

### BASE vs ACID

| **ACID** (SQL) | **BASE** (NoSQL) |
|---|---|
| **A**tomicity — all or nothing | **B**asically **A**vailable — system always responds |
| **C**onsistency — data is always valid | **S**oft state — state may change over time without input |
| **I**solation — concurrent transactions don't interfere | **E**ventual consistency — will become consistent... eventually |
| **D**urability — committed data survives crashes | |

> **For payments:** ACID is non-negotiable for the transactional core. You cannot have a state where money left account A but never arrived at account B.

---

## CAP Theorem — The Fundamental Constraint

### What It Says

> In a **distributed system**, when a **network partition** (P) occurs, you must choose between **Consistency** (C) and **Availability** (A). You cannot have all three simultaneously.

```
                    C
                 Consistency
                 "Every read gets the
                  most recent write"
                   /          \
                  /            \
                 /              \
                /    PICK TWO    \
               /    (when P hits) \
              /                    \
             A ──────────────────── P
        Availability           Partition Tolerance
        "Every request          "System works despite
         gets a response"        network splits"
```

### The Three Guarantees

| Letter | Guarantee | In Plain English |
|--------|-----------|-----------------|
| **C** — Consistency | Every read receives the **most recent write** or an error | If you just paid £50, your balance shows £50 deducted — not the old balance |
| **A** — Availability | Every request receives a **non-error response** (no guarantee it's the latest) | The system always responds, even if the data might be slightly stale |
| **P** — Partition Tolerance | System continues to operate despite **network failures** between nodes | If the connection between EU and US data centres drops, the system doesn't crash |

### The Reality: It's Not a Choice of Two — It's a Spectrum

> **Common misconception:** "Pick two out of three."
>
> **Reality:** Network partitions **will happen** in any distributed system. P is not optional. So the real choice is: **when a partition occurs, do you sacrifice consistency or availability?**

```
Partition happens ──► You MUST choose:

    Option 1: CP (Consistency + Partition Tolerance)
    ──► Reject requests until partition heals
    ──► "I'd rather return an error than give you wrong data"
    ──► Example: Payment balance check → MUST be correct

    Option 2: AP (Availability + Partition Tolerance)  
    ──► Serve requests with potentially stale data
    ──► "I'd rather give you slightly old data than nothing"
    ──► Example: Dashboard showing transaction count → a few seconds stale is fine
```

### CAP Applied to Our Payment System

| Component | CAP Choice | Why | What Happens During Partition |
|-----------|-----------|-----|------------------------------|
| **Payment Service (writes)** | **CP** | A wrong balance or double charge is unacceptable. Correctness over availability | Returns error → client retries with idempotency key. Better to reject than corrupt |
| **Wallet Balance** | **CP** | Balance must be accurate NOW. Serving a stale balance could allow overspending | Read fails or blocks until consistent state is confirmed |
| **Ledger** | **CP** | Financial records must be sequentially consistent. Out-of-order entries break auditing | Writes queue until partition heals, preserving order |
| **Redis Cache** | **AP** | Cache is a convenience, not source of truth. Stale cache → fall through to PostgreSQL | Serves potentially stale data. Worst case: idempotency check misses → PostgreSQL catches it |
| **Dashboard** | **AP** | Showing transactions from 5 seconds ago is fine. Showing nothing is worse | Serves slightly stale aggregations. User sees data, just not real-time |
| **Webhook Dispatcher** | **AP** | Better to deliver a notification late than not at all. DLQ catches failures | Continues dispatching from queue. May deliver out of order — merchants handle with timestamps |
| **Dashboard (read model)** | **AP** | Showing transactions from 5 seconds ago is fine. An error page is not | Serves last-known aggregations. Users get stale data rather than errors |

### CP vs AP — Database Mapping

| Database | CAP Classification | Why |
|----------|-------------------|-----|
| **PostgreSQL** (single node) | **CA** (no partition in single node) | Single node = no network partition. Full consistency + availability. But no horizontal distribution |
| **PostgreSQL** (with sync replicas) | **CP** | Sync replication blocks writes during partition → consistent but may become unavailable |
| **PostgreSQL** (with async replicas) | **AP** for reads, **CP** for writes | Reads from replica may be stale (AP). Writes to primary are consistent (CP) |
| **Redis** | **AP** | Designed for speed. Redis Cluster uses async replication — during partition, reads may be stale |
| **Redis Sentinel** | **CP** | Sentinel promotes replica to master during failure, but writes rejected during failover window |
| **MongoDB** | **CP** by default | Writes go to primary. During partition, minority side can't accept writes |
| **DynamoDB** | **AP** by default, **CP** optional | Eventually consistent reads by default. Can request strongly consistent reads (CP) at higher cost/latency |
| **Cassandra** | **AP** | Tunable consistency, but designed availability-first. Great for write-heavy, geo-distributed workloads |
| **CockroachDB** | **CP** | Distributed SQL — ACID across nodes but sacrifices availability during partition |

### The Principal-Level Nuance: CAP Per Operation, Not Per System

> A mid-level engineer says: "Our system is CP."
>
> A principal engineer says: "Our system makes **different CAP trade-offs per operation**."

```
Payment System
│
├── Payment creation     → CP  (reject if inconsistent)
├── Balance lookup       → CP  (must be accurate)
├── Idempotency check    → AP at Redis layer, CP at PostgreSQL layer
├── Dashboard reads      → AP  (eventual consistency acceptable)
├── Webhook delivery     → AP  (deliver late rather than not at all)
└── Ledger append        → CP  (ordering and consistency mandatory)
```

> **This is the key interview insight.** You don't pick CP or AP for the whole system. You pick it **per data path** based on the cost of being wrong.

### The PACELC Extension (Beyond CAP)

> CAP only talks about what happens **during a partition**. But what about normal operation?

**PACELC:** If **P**artition → choose **A** or **C**. **E**lse (normal operation) → choose **L**atency or **C**onsistency.

| Component | During Partition (PAC) | Normal Operation (ELC) |
|-----------|----------------------|----------------------|
| **Payment writes** | Choose **C** (reject if inconsistent) | Choose **C** (strong consistency, accept higher latency) |
| **Dashboard reads** | Choose **A** (serve stale data) | Choose **L** (eventual consistency, serve fast from replica) |
| **Redis cache** | Choose **A** (serve cached value) | Choose **L** (sub-ms reads, may be stale) |

> **Why mention PACELC?** It shows the interviewer you understand that consistency vs latency trade-offs exist **even when everything is healthy** — not just during failures. This is a daily operational reality, not a theoretical edge case.

### CAP Quick Reference for Interview

| Question | Answer |
|----------|--------|
| "What is CAP theorem?" | In a distributed system during a network partition, you must choose between consistency and availability. Partition tolerance is non-negotiable. |
| "Is your payment system CP or AP?" | **It's both — per operation.** Payment writes and balances are CP (correctness over availability). Dashboard and cache are AP (availability over freshness). |
| "Why not just make everything CP?" | Because users would see errors on the dashboard when a minor network blip occurs. The dashboard doesn't need real-time accuracy — showing a slightly stale view is better than showing nothing. |
| "Why not just make everything AP?" | Because showing a stale wallet balance could allow a merchant to withdraw money they don't have. Financial data must be correct NOW. |
| "How does this affect your database choice?" | PostgreSQL (CP) for the payment core — I need ACID and strong consistency. Redis (AP) for caching — speed matters more than freshness. Data warehouse (AP) for dashboard reads — eventual consistency is fine for analytics. |

---

## Which Database Where? (Payment System Data Map)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PAYMENT SYSTEM DATA LAYER                     │
├──────────────────┬──────────────────┬───────────────────────────────┤
│   TRANSACTIONAL  │   OPERATIONAL    │         ANALYTICAL            │
│                  │                  │                               │
│  ┌────────────┐  │  ┌────────────┐  │  ┌─────────────────────────┐  │
│  │ PostgreSQL │  │  │   Redis    │  │  │   Data Warehouse        │  │
│  │            │  │  │            │  │  │   (BigQuery/Redshift/    │  │
│  │ • Orders   │  │  │ • Sessions │  │  │    Snowflake)           │  │
│  │ • Payments │  │  │ • Cache    │  │  │                         │  │
│  │ • Accounts │  │  │ • Rate     │  │  │ • Revenue reports       │  │
│  │ • Ledger   │  │  │   limits   │  │  │ • Reconciliation        │  │
│  │            │  │  │ • Idempot. │  │  │ • Fraud analytics       │  │
│  │            │  │  │   keys     │  │  │ • Business metrics      │  │
│  └────────────┘  │  └────────────┘  │  └─────────────────────────┘  │
│                  │                  │                               │
│  ACID required   │  Speed required  │  Aggregation required         │
│  Strong consist. │  Sub-ms latency  │  Columnar storage             │
│  Complex joins   │  TTL expiry      │  Batch + stream processing    │
└──────────────────┴──────────────────┴───────────────────────────────┘
```

---

## Layer 1: Transactional Database (SQL — PostgreSQL)

### Why PostgreSQL for Payments?

| Reason | Detail |
|--------|--------|
| **ACID transactions** | Transfer money atomically — debit + credit in one transaction, both succeed or both fail |
| **Strong consistency** | Read-after-write guarantee — no stale balances |
| **Referential integrity** | Foreign keys enforce that every payment references a valid order, customer, and merchant |
| **Complex queries** | JOINs across orders, payments, refunds, customers in a single query |
| **Unique constraints** | **Idempotency enforcement** — DB rejects duplicate payment IDs natively |
| **Mature ecosystem** | Battle-tested in financial systems for decades, excellent tooling |
| **Row-level locking** | Concurrent transactions on different rows don't block each other |

### What Lives Here

```sql
-- Core payment tables

payment_orders (
    id              UUID PRIMARY KEY,     -- idempotency key from client
    customer_id     UUID NOT NULL,
    merchant_id     UUID NOT NULL,
    amount          DECIMAL(19,4) NOT NULL,  -- DECIMAL, never FLOAT for money
    currency        VARCHAR(3) NOT NULL,     -- ISO 4217 (USD, GBP, EUR)
    status          VARCHAR(20) NOT NULL,    -- PENDING → PROCESSING → SUCCESS / FAILED
    psp_reference   VARCHAR(255),
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);

wallet_accounts (
    id              UUID PRIMARY KEY,
    merchant_id     UUID NOT NULL UNIQUE,
    balance         DECIMAL(19,4) NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL,
    updated_at      TIMESTAMP NOT NULL
);

ledger_entries (
    id              UUID PRIMARY KEY,
    payment_id      UUID NOT NULL REFERENCES payment_orders(id),
    entry_type      VARCHAR(20) NOT NULL,    -- DEBIT / CREDIT
    amount          DECIMAL(19,4) NOT NULL,
    balance_after   DECIMAL(19,4) NOT NULL,
    created_at      TIMESTAMP NOT NULL       -- append-only, never updated
);
```

### Critical Design Decisions

**Use `DECIMAL(19,4)`, NEVER `FLOAT` for money.**
> Floating point has rounding errors. `0.1 + 0.2 = 0.30000000000000004` in float. In a payment system, being off by a fraction of a cent across millions of transactions means real financial discrepancies.

**Ledger is append-only.**
> Never UPDATE or DELETE a ledger row. Corrections are new entries (reversals/adjustments). This gives you a complete, tamper-evident audit trail.

**Status is a state machine.**
> `PENDING → AUTHORIZED → CAPTURED → SETTLED → REFUNDED`. Plus dispute states: `DISPUTED → CB_WON | CB_LOST`. Never skip states. Each transition is a new ledger entry. See [00-payment-system-overview.md](00-payment-system-overview.md) for full state machine.

### Double-Entry Bookkeeping

> Every financial transaction creates **two ledger entries** that must balance: total debits = total credits. This is the foundation of financial accounting and a **principal-level expectation** in a payments interview.

**The rule:** For every payment, money leaves one account and enters another. The ledger records **both sides**.

```
Payment of £100 from Customer → Merchant:

    Entry 1 (DEBIT):  Customer's account   -£100
    Entry 2 (CREDIT): Merchant's account    +£100
    ─────────────────────────────────────────────
    Net:                                     £0    ← MUST always balance

Refund of £100 from Merchant → Customer:

    Entry 3 (DEBIT):  Merchant's account    -£100
    Entry 4 (CREDIT): Customer's account    +£100
    ─────────────────────────────────────────────
    Net:                                     £0    ← Still balanced
```

**Schema that enforces this:**

```sql
ledger_entries (
    id              UUID PRIMARY KEY,
    payment_id      UUID NOT NULL REFERENCES payment_orders(id),
    account_id      UUID NOT NULL,           -- which account (customer, merchant, platform)
    entry_type      VARCHAR(20) NOT NULL,    -- DEBIT / CREDIT
    amount          DECIMAL(19,4) NOT NULL,  -- always positive
    balance_after   DECIMAL(19,4) NOT NULL,  -- running balance for this account
    created_at      TIMESTAMP NOT NULL       -- append-only, never updated
);

-- For every payment_id, SUM of DEBITs MUST equal SUM of CREDITs
-- This is validated at application level and audited in reconciliation
```

**Why double-entry matters:**
- **Self-auditing** — if debits ≠ credits for any transaction, something is wrong
- **Reconciliation** — match internal ledger against PSP/bank statements
- **Regulatory requirement** — FCA expects proper financial record-keeping
- **Debugging** — trace exactly where money went when something looks wrong

**Platform fees example (3 entries):**
```
Customer pays £100, platform takes 2.5% fee:

    Entry 1 (DEBIT):  Customer's account    -£100.00
    Entry 2 (CREDIT): Merchant's account    +£97.50
    Entry 3 (CREDIT): Platform revenue      +£2.50
    ────────────────────────────────────────────────
    Net:                                     £0.00
```

### Scaling PostgreSQL

| Strategy | When | How |
|----------|------|-----|
| **Read replicas** | Read-heavy workloads (dashboards, reports) | Async replication to replica nodes, route reads there |
| **Connection pooling** | High concurrent connections | PgBouncer in front of PostgreSQL |
| **Partitioning** | Table grows past ~100M rows | Partition `ledger_entries` by `created_at` (time-range partitioning) |
| **Sharding** | Multi-region or extreme scale | Shard by `merchant_id` — all of a merchant's data on one shard (Citus, Vitess) |

> **Why shard by merchant_id?** Payments, wallets, and ledger entries for a merchant are always queried together. Co-locating them avoids cross-shard joins.

---

## Layer 2: Operational Cache (NoSQL — Redis)

### Why Redis for Payments?

| Reason | Detail |
|--------|--------|
| **Sub-millisecond latency** | In-memory storage, ~0.1ms reads |
| **TTL (Time-To-Live)** | Keys auto-expire — perfect for idempotency keys, sessions, rate limits |
| **Atomic operations** | `INCR`, `SETNX` — thread-safe counters and locks without DB round-trips |
| **Data structures** | Sorted sets, hash maps, lists — more than just key-value |

### What Lives Here

| Data | Why Redis | TTL |
|------|-----------|-----|
| **Idempotency keys** | Check for duplicate payment requests in < 1ms before hitting the DB | 24-48 hours |
| **Rate limiting** | Track requests per API key per time window using `INCR` + `EXPIRE` | Sliding window |
| **Session tokens** | Authenticate API calls without DB lookup on every request | 30 min - 24 hrs |
| **Payment status cache** | Customer polling "is my payment done?" — serve from cache, not DB | 5-15 min |
| **Distributed locks** | Prevent concurrent processing of the same payment | 30-60 sec |

### Idempotency Check — Redis Before PostgreSQL

```
Client sends payment request with idempotency key
    │
    ▼
Redis: SETNX idempotency:{key} "processing"    ← atomic, sub-ms
    │
    ├── Key didn't exist (new request)
    │       → SET succeeds
    │       → Proceed to PostgreSQL for actual payment
    │       → On completion: update Redis value to result
    │
    └── Key already exists (duplicate)
            → SET fails (SETNX = "set if not exists")
            → Return cached result immediately
            → PostgreSQL never touched
```

> **Two layers of idempotency:** Redis for speed (catch 99% of duplicates instantly), PostgreSQL unique constraint as the ultimate safety net.

### Redis Is NOT Your Database

| Redis does | Redis does NOT |
|------------|----------------|
| Cache hot data | Store source of truth |
| Speed up reads | Replace PostgreSQL for payments |
| Handle ephemeral state | Guarantee durability (even with AOF, it's not ACID) |
| Rate limit and lock | Handle complex queries or joins |

> **If Redis goes down**, the system is slower but **not broken**. Requests fall through to PostgreSQL. If PostgreSQL goes down, the system **stops processing payments**.

---

## Layer 3: Analytical / Data Warehouse

### Why a Separate Analytical Layer?

Running heavy analytical queries on your transactional database **kills payment processing performance**. Lock contention, full table scans, and CPU-heavy aggregations compete with real-time payment transactions.

### What Lives Here

| Data | Purpose |
|------|---------|
| **Revenue reports** | Total revenue by merchant, day, week, month |
| **Reconciliation** | Match internal records against PSP/bank statements |
| **Fraud analytics** | Pattern detection across millions of transactions |
| **Business metrics** | Conversion rates, failure rates, average transaction value |
| **Compliance reporting** | Audit trails for regulators |

### Technology Choices

| Database | When to Use |
|----------|-------------|
| **BigQuery** (GCP) | Already on GCP, serverless, pay-per-query |
| **Redshift** (AWS) | Already on AWS, need reserved capacity |
| **Snowflake** | Multi-cloud, separation of storage and compute |
| **ClickHouse** | Self-hosted, real-time analytics, extremely fast aggregations |

### How Data Gets There

```
PostgreSQL ──► Change Data Capture (CDC) ──► Kafka ──► Data Warehouse
                  (Debezium)                              │
                                                          ▼
                                                   Dashboards
                                                   (Grafana / Looker)
```

> **CDC (Change Data Capture):** Streams every INSERT/UPDATE from PostgreSQL to Kafka without impacting DB performance. No batch ETL jobs — data arrives in near real-time.

### Reconciliation — What It Actually Means

> Mentioned in many docs but never explained. This is a **daily operational process** at any payment company.

**Reconciliation** = matching your **internal ledger** against **external records** (PSP settlement reports, bank statements) to find discrepancies.

```
Your Ledger (source of truth)          PSP Settlement Report (external)
─────────────────────────────          ──────────────────────────────
Payment abc-123:  £50.00 SUCCESS       Payment abc-123:  £50.00 SETTLED
Payment def-456:  £30.00 SUCCESS       Payment def-456:  £30.00 SETTLED
Payment ghi-789:  £20.00 SUCCESS       Payment ghi-789:  ??? MISSING     ← discrepancy!
Payment ???:      ??? MISSING          Payment jkl-012:  £75.00 SETTLED   ← discrepancy!
```

**Types of discrepancy:**
- **Missing from PSP** — we think it succeeded, PSP has no record → investigate
- **Missing from us** — PSP settled a payment we don't have → possible bug or race condition
- **Amount mismatch** — we say £50, PSP says £49.50 → FX conversion or fee issue
- **Status mismatch** — we say SUCCESS, PSP says FAILED → dangerous, needs immediate resolution

**Where it runs:** Data warehouse, not the production database. Reconciliation queries scan millions of rows and must never compete with live payments.

### Tokenization

> Replace sensitive card data with an opaque **token**. The real card number is stored **only at the PSP**.

```
Customer's card: 4242 4242 4242 4242
                         │
                         ▼
PSP tokenizes:  tok_8a3f2b1c9d4e     ← stored in our DB
                         │
Real card number:        │ stored ONLY at PSP (in PCI-compliant vault)
                         │
Next payment:   send tok_8a3f2b1c9d4e to PSP → PSP looks up real card → processes
```

**Why tokenization matters:**
- Our system **never stores card numbers** → reduces PCI DSS scope dramatically
- Enables **card-on-file** for returning customers (one-click payments)
- Tokens are **merchant-specific** — stolen token is useless to another merchant
- PSP handles token lifecycle (expiry, card updates)

---

## When Would You Use NoSQL as Primary? (And When You Wouldn't)

### NoSQL Fits Better For...

| Use Case | Why | Example DB |
|----------|-----|------------|
| **Dashboard / analytics read model** | Read-heavy, flexible schema, high throughput, eventual consistency is fine | **MongoDB** or **DynamoDB** |
| **Event log / activity stream** | Append-only, massive write throughput, no joins needed | **Cassandra** or **Kafka** |
| **User profiles / preferences** | Schema varies per user, read-heavy | **MongoDB** or **DynamoDB** |
| **Real-time leaderboards** | Sorted sets, counters, sub-ms reads | **Redis** |

### NoSQL Does NOT Fit For...

| Use Case | Why Not |
|----------|---------|
| **Payment transactions** | Need ACID, multi-table transactions, strong consistency |
| **Wallet balances** | Cannot risk eventual consistency — balance must be accurate NOW |
| **Ledger / audit trail** | Need referential integrity, complex queries for reconciliation |
| **Refund processing** | Requires atomic debit + credit across accounts |

> **Interview insight:** If the interviewer asks about a **read-heavy, non-financial** component (like a dashboard read model or a public-facing data API), that's where you can reach for NoSQL. High read throughput, flexible schema, and tolerance for eventual consistency make it a natural fit. **Payments are the opposite.**

---

## The Interview Answer — Putting It All Together

### "How would you design the data layer for a payment system?"

> "I'd use a **polyglot persistence** approach — different databases for different concerns:
>
> **PostgreSQL** as the transactional backbone. Payments demand **ACID guarantees** — when money moves, the debit and credit must succeed or fail atomically. I'd use `DECIMAL` types for monetary values, never floats. The ledger is **append-only** for auditability. **Idempotency** is enforced via a unique constraint on the payment order ID, which doubles as the client-generated UUID.
>
> **Redis** as the operational layer. It sits in front of PostgreSQL to handle **idempotency checks** in sub-millisecond time, **rate limiting** per API key, **session management**, and **distributed locks** to prevent concurrent processing of the same payment. If Redis fails, we fall through to PostgreSQL — slower but correct.
>
> **Data warehouse** (BigQuery/Redshift/Snowflake) for analytics. We stream data from PostgreSQL via **CDC through Kafka** so we never run heavy aggregation queries against the live transaction database. This powers reconciliation, fraud detection, revenue reporting, and business metrics.
>
> For scaling PostgreSQL, I'd start with **read replicas** and **connection pooling**, then move to **time-based partitioning** on the ledger table, and eventually **shard by merchant_id** if we hit extreme scale — keeping all of a merchant's data co-located to avoid cross-shard joins."

---

## Quick Reference — Database Decision Matrix

| Question | Answer | Database |
|----------|--------|----------|
| Does it involve money moving? | Yes | **PostgreSQL** (ACID) |
| Does it need sub-ms reads? | Yes | **Redis** (cache) |
| Is it an audit/compliance query? | Yes | **PostgreSQL** (source of truth) |
| Is it a heavy analytical report? | Yes | **Data Warehouse** |
| Is the schema flexible & read-heavy? | Yes | **NoSQL** (MongoDB/DynamoDB) |
| Does it need strong consistency? | Yes | **PostgreSQL** |
| Is eventual consistency acceptable? | Yes | **NoSQL** or **Redis** |

---

**Next:** [08-trade-offs.md](08-trade-offs.md) — Stage-by-stage trade-offs for every decision
