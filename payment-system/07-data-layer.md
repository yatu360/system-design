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
> `PENDING → PROCESSING → SUCCESS | FAILED | REFUNDED`. Never skip states. Each transition is a new ledger entry.

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

---

## When Would You Use NoSQL as Primary? (And When You Wouldn't)

### NoSQL Fits Better For...

| Use Case | Why | Example DB |
|----------|-----|------------|
| **Sports API** (from recruiter brief) | Read-heavy, flexible schema, high throughput, eventual consistency is fine | **MongoDB** or **DynamoDB** |
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

> **Interview insight:** If the interviewer asks about the **Sports API**, that's where you can reach for NoSQL. Real-time scores, match data, and player stats are read-heavy, schema-flexible, and tolerance for eventual consistency is higher. **Payments are the opposite.**

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

**Next:** [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) — Everything on one page
