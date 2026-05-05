# Database Sharding — Payment System

> When one PostgreSQL instance isn't enough, how do you split payment data across multiple databases without breaking queries or losing ACID guarantees?

---

## When to Shard

A single PostgreSQL instance handles **~5K-10K TPS writes** before it struggles. For our payment system:

| Scale | Solution | When |
|---|---|---|
| < 5K TPS | Single PostgreSQL + read replicas | Startup to mid-scale |
| 5K-50K TPS | **Shard** (split data across multiple databases) | Growing merchant base |
| 50K+ TPS | Shard + consider distributed SQL (CockroachDB, TiDB) | Global scale |

**Signs you need sharding:**
- Write latency increasing despite query optimisation
- Single DB CPU consistently above 70%
- Table size causing vacuum issues (billions of rows)
- Read replicas can't keep up with replication lag

**Don't shard prematurely.** Sharding adds complexity — routing logic, cross-shard queries, operational overhead. Exhaust vertical scaling and read replicas first.

---

## Shard Key: Why `merchant_id`

The shard key determines which database a row lives in. Choose wrong, and every query hits every shard.

**Our choice: `merchant_id`**

Why? Because almost **every query in our system is scoped to one merchant:**

```
-- Dashboard: "Show me my transactions"
SELECT * FROM payments WHERE merchant_id = 'merchant_asos' AND created_at > '2026-01-01';

-- Refund lookup
SELECT * FROM payments WHERE merchant_id = 'merchant_asos' AND payment_id = 'pay_abc';

-- Chargeback rate
SELECT COUNT(*) FROM payments WHERE merchant_id = 'merchant_asos' AND status = 'DISPUTED';

-- Merchant balance
SELECT balance FROM wallets WHERE merchant_id = 'merchant_asos';
```

All of these hit **one shard**. No scatter-gather across multiple databases.

**Three reasons `merchant_id` wins:**

| Reason | Explanation |
|---|---|
| **Query locality** | Dashboard, refunds, chargebacks — all per-merchant queries hit one shard |
| **Transaction locality** | Payment + ledger entry + wallet update for one merchant on same shard = local ACID transaction |
| **Tenant isolation** | One merchant's traffic spike doesn't slow down another merchant's queries |

---

## Why NOT Other Shard Keys

### `payment_id`

```
Shard 1: pay_abc, pay_ghi, pay_mno  (random merchants mixed)
Shard 2: pay_def, pay_jkl, pay_pqr  (random merchants mixed)
```

- Distributes evenly (UUIDs hash uniformly)
- But **dashboard query scatters** — "show me all ASOS transactions" hits every shard
- **Cross-shard joins** — payment + ledger + wallet are on different shards
- Fine for write-heavy, bad for read-heavy (and dashboards are read-heavy)

### `location` / `region`

```
Shard 1 (UK): UK customer payments
Shard 2 (EU): EU customer payments
Shard 3 (US): US customer payments
```

- ASOS has customers in UK, France, Germany, US — their data is split across 3+ shards
- Dashboard query for ASOS now cross-shard — slow and expensive
- Confuses two concerns: data residency (legal) vs sharding (performance)

### `date` / `time`

```
Shard 1: January payments
Shard 2: February payments
Shard 3: March payments (HOT — all current traffic!)
```

- Current month shard gets ALL writes — massive hot spot
- Historical shards are cold (wasted resources)
- Good for archiving, terrible for live traffic distribution

---

## Sharding Strategies

### 1. Hash-Based

```
shard = hash(merchant_id) % number_of_shards

hash("merchant_asos") % 4 = 2  → shard_2
hash("merchant_uber") % 4 = 0  → shard_0
hash("merchant_amazon") % 4 = 1 → shard_1
```

| Pros | Cons |
|---|---|
| Simple, deterministic, no lookup needed | Can't move a hot merchant to its own shard |
| Even distribution (if hash is good) | Adding shards = rehash everything (data migration) |
| No extra infrastructure | No flexibility for uneven merchants |

### 2. Range-Based

```
Merchants A-H → shard_1
Merchants I-P → shard_2
Merchants Q-Z → shard_3
```

| Pros | Cons |
|---|---|
| Simple to understand | Unpredictable distribution (what if A-H has 80% of volume?) |
| Easy range scans | Hot spots likely |
| | Rebalancing = moving huge ranges |

### 3. Lookup Table (Our Choice)

```
┌──────────────────┬──────────┐
│ merchant_id      │ shard_id │
├──────────────────┼──────────┤
│ merchant_asos    │ shard_1  │
│ merchant_uber    │ shard_1  │
│ merchant_amazon  │ shard_3  │  ← dedicated shard (hot merchant)
│ merchant_nike    │ shard_2  │
│ merchant_tesco   │ shard_2  │
└──────────────────┴──────────┘
```

| Pros | Cons |
|---|---|
| **Flexible** — move any merchant anytime | Extra hop (lookup before query) |
| Hot merchants get dedicated shards | Routing table is a dependency |
| Add shards without rehashing | Must cache aggressively (Redis) |
| Rebalance without downtime | |

---

## Lookup Table — How It Works

### The Routing Flow

```
Request: POST /v1/payments (merchant_id: "merchant_asos")
    │
    │  Step 1: Which shard?
    ▼
┌──────────────────────────────────────────────┐
│  Redis: GET shard_routing:merchant_asos      │
│                                              │
│  Cache HIT → "shard_1"  (sub-ms, 99% of     │
│              time — mapping rarely changes)   │
│                                              │
│  Cache MISS → PostgreSQL routing table       │
│              → cache in Redis (TTL: 1 hour)  │
│              → "shard_1"                     │
└──────────────────────┬───────────────────────┘
                       │
    │  Step 2: Route to shard
    ▼
┌──────────────────────────────────────────────┐
│  Connect to shard_1 connection pool          │
│  Execute query on shard_1                    │
└──────────────────────────────────────────────┘
```

### The Routing Table (PostgreSQL)

```sql
shard_routing (
    merchant_id     VARCHAR(255) PRIMARY KEY,
    shard_id        VARCHAR(50) NOT NULL,     -- "shard_1", "shard_2", etc.
    assigned_at     TIMESTAMP NOT NULL,
    reason          VARCHAR(255)              -- "initial", "rebalanced", "hot_merchant"
);
```

### New Merchant Assignment

When a new merchant signs up:
1. Query current shard load (transaction count per shard)
2. Assign to the least-loaded shard
3. Insert into `shard_routing`
4. Cache in Redis

### Rebalancing a Hot Merchant

```
Step 1: Create new shard (shard_3) or identify under-loaded shard
Step 2: Begin background copy — replicate merchant_amazon data to shard_3
Step 3: Keep writing to old shard (shard_1) during migration
Step 4: When copy is caught up — brief pause, final sync
Step 5: Update routing table: merchant_amazon → shard_3
Step 6: Invalidate Redis cache for merchant_amazon
Step 7: New requests route to shard_3
Step 8: Drop old data from shard_1 (after verification)
```

No downtime. Other merchants on shard_1 are unaffected.

---

## Hot Merchant Problem

**The problem:** Amazon-scale merchant does 10x the volume of average merchants. Their shard is overloaded.

**Detection:**
- Monitor queries/second per shard
- Monitor p99 latency per shard
- Alert if one shard consistently above 70% capacity

**Solutions (in order of complexity):**

| Solution | When | Trade-off |
|---|---|---|
| **Move to dedicated shard** | One merchant is disproportionately large | Simple — just update routing table |
| **Read replicas for that shard** | Read-heavy (dashboard queries) | Doesn't help write throughput |
| **Sub-shard by `payment_id`** | Single merchant exceeds one shard's write capacity | Cross-shard queries for that merchant's dashboard |

### Sub-Sharding (Last Resort)

```
Routing table for extreme scale:

merchant_amazon → sub-shard routing:
    hash(payment_id) % 4 = 0 → shard_3a
    hash(payment_id) % 4 = 1 → shard_3b
    hash(payment_id) % 4 = 2 → shard_3c
    hash(payment_id) % 4 = 3 → shard_3d
```

**Trade-off:** Amazon's dashboard query ("show all my transactions") now hits 4 sub-shards. Scatter-gather adds latency. Only do this when the write throughput problem is severe enough to justify it.

---

## Data Residency vs Sharding

These are **two independent decisions**. Don't confuse them.

| Concern | Question | Answer | Mechanism |
|---|---|---|---|
| **Data residency** | Which *country* is the data in? | UK (London) | Region selection |
| **Sharding** | How is data *split* for performance? | By merchant_id | Lookup table |

```
┌─────────────────────────────────────────────────────────────┐
│  AWS eu-west-2 (London)  ← All shards physically here       │
│                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │  │ Shard 4 │      │
│  │ ASOS    │  │ Nike    │  │ Amazon  │  │ Tesco   │      │
│  │ Uber    │  │ Zara    │  │(dedicated│  │ Boots   │      │
│  │ JD      │  │ H&M     │  │ hot)    │  │ Argos   │      │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
│                                                              │
│  Data residency ✓ — ALL data is in UK                        │
│  UK GDPR ✓ — PII stored in UK                               │
│  FCA ✓ — financial records accessible to UK regulator        │
└─────────────────────────────────────────────────────────────┘
```

**ASOS has a French customer who pays?** That payment is in London (shard_1). Not in France.

**Why this is legal:**
- UK has EU adequacy decision — EU citizen data can be processed in UK
- US has no federal data residency law
- All merchants' payments from all customers worldwide → London

**When would you go multi-region?**

Only at Checkout.com global scale — and it's **separate deployments**, not shards:

```
UK Deployment (London)           US Deployment (Virginia)
├── UK/EU merchants              ├── US merchants
├── Own PostgreSQL (sharded)     ├── Own PostgreSQL (sharded)
├── Own Kafka                    ├── Own Kafka
└── UK acquiring banks           └── US acquiring banks
```

This is for **latency** (US merchant → US acquiring bank) and **regulatory simplicity** at global scale. Not database sharding.

---

## Citus — PostgreSQL Sharding Extension

Citus is a PostgreSQL extension that makes sharding transparent. Instead of managing routing logic yourself, Citus handles it:

```sql
-- Tell Citus to shard by merchant_id
SELECT create_distributed_table('payments', 'merchant_id');
SELECT create_distributed_table('ledger_entries', 'merchant_id');
SELECT create_distributed_table('wallets', 'merchant_id');

-- Queries work as normal — Citus routes automatically
SELECT * FROM payments WHERE merchant_id = 'merchant_asos';
-- ↑ Citus knows this is on shard_2, routes there directly
```

**Co-location:** Tables sharded by the same key are placed together:

```
Shard 1:
├── payments      WHERE merchant_id IN (asos, uber, jd)
├── ledger_entries WHERE merchant_id IN (asos, uber, jd)
└── wallets       WHERE merchant_id IN (asos, uber, jd)

→ JOIN between payments and ledger_entries for ASOS = local (one shard)
→ ACID transaction across payments + wallet for ASOS = local (one shard)
```

**For the interview:** "At our scale, Citus gives us merchant-based sharding out of the box. Co-located tables mean JOINs and transactions stay local within a shard. If we outgrow Citus, the next step is CockroachDB or TiDB — distributed SQL that preserves ACID across shards."

---

## What About Cross-Shard Queries?

Some queries need data from all merchants (platform-level):

```sql
-- Total platform revenue today (cross-shard)
SELECT SUM(amount) FROM payments WHERE status = 'CAPTURED' AND created_at > TODAY;

-- Top merchants by volume (cross-shard)
SELECT merchant_id, COUNT(*) FROM payments GROUP BY merchant_id ORDER BY 2 DESC LIMIT 10;
```

**Solution: Don't query the sharded database for this.**

```
Sharded PostgreSQL                  Data Warehouse (unsharded)
├── Shard 1                         ├── ALL payments
├── Shard 2         ──── CDC ────►  ├── ALL merchants
├── Shard 3          (Kafka)        ├── ALL ledger entries
└── Shard 4                         └── Optimised for aggregation

Cross-shard query? → Query the data warehouse
Per-merchant query? → Query the shard directly
```

CDC (Change Data Capture) streams all shard data into the warehouse via Kafka. Dashboard aggregations, reconciliation, analytics — all hit the warehouse, never the shards.

**Trade-off:** Data warehouse is seconds behind (eventual consistency). Acceptable for dashboards and reports. Not acceptable for "is this payment captured?" — that queries the shard directly.

---

## Interview Questions & Answers

### "Why shard by merchant_id?"
> "Query patterns. Every dashboard query, refund lookup, and chargeback check is per-merchant. Sharding by merchant means these queries hit one shard — no scatter-gather. It also gives us transaction locality — payment, ledger, and wallet for one merchant are on the same shard, so ACID transactions stay local."

### "Why not shard by payment_id?"
> "Even distribution but wrong query patterns. Payment_id distributes uniformly, but the dashboard — 'show me all my transactions' — would hit every shard. We'd trade write distribution for expensive reads. Since dashboards are read-heavy and merchant-scoped, merchant_id is the better key."

### "What about hot merchants (like Amazon)?"
> "The lookup table handles this. If one merchant does disproportionate volume, I update the routing table to give them a dedicated shard — one config change, no impact on other merchants. If they outgrow one shard, I sub-shard within that merchant by payment_id, but that's a last resort because it complicates their dashboard queries."

### "Why lookup table over hash-based sharding?"
> "Flexibility. With hash-based, if Amazon is hot, I can't move them — they go wherever the hash puts them. Adding shards means rehashing everything. Lookup table lets me move any merchant to any shard anytime. The extra Redis hop is sub-ms and the mapping rarely changes, so cache hit rate is effectively 100%."

### "How do you handle data residency?"
> "Data residency is a region decision, not a shard decision. All shards physically sit in London — UK GDPR and FCA satisfied. EU customer data in UK is legal under adequacy. We don't shard by location because merchants have global customers. Multi-region only comes in at Checkout.com global scale, and that's separate deployments, not database shards."

### "What about cross-shard queries?"
> "Platform-level analytics — total revenue, top merchants, global chargeback rate — don't hit the shards. CDC streams all data to a data warehouse via Kafka. Dashboards and reports query the warehouse. Per-merchant queries go to the shard directly. Different read paths for different consistency needs."

### "When would you NOT shard?"
> "Under 5K-10K TPS writes. Sharding adds routing logic, operational complexity, and cross-shard query limitations. Single PostgreSQL with read replicas handles most payment systems fine. I'd exhaust vertical scaling, connection pooling (PgBouncer), and read replicas before sharding."

### "How do you migrate from single DB to sharded?"
> "Incrementally. Start with the lookup table pointing all merchants to the single DB (shard_1). Add a second shard. Migrate a few merchants at a time — copy data, flip the routing, verify. No big-bang migration. The lookup table makes this safe — each merchant migrates independently."

### "What's Citus?"
> "PostgreSQL extension for transparent sharding. You declare a distribution key — merchant_id — and Citus handles routing, co-location of related tables, and distributed queries. At our scale, it gives us sharding out of the box without custom routing logic. If we outgrow it, CockroachDB or TiDB offer distributed SQL with cross-shard ACID."

### "What about the shard routing table itself — isn't that a single point of failure?"
> "The routing table is cached in Redis with a 1-hour TTL. Even if the routing PostgreSQL goes down, cached mappings serve requests. The table is tiny (one row per merchant — maybe 10K rows) and rarely changes. We could also replicate it to every application instance on startup as an in-memory map — it's small enough."

---

## Related Docs

- [07-data-layer.md](07-data-layer.md) — PostgreSQL as source of truth, CAP per operation
- [11-interview-walkthrough.md](11-interview-walkthrough.md) — Section 4.2 mentions "shard by merchant_id with Citus"
- [00-payment-system-overview.md](00-payment-system-overview.md) — Overall architecture
