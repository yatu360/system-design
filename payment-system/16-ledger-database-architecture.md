# Ledger & Database Architecture

> Design a system to reliably store transaction records, balances, and payment states over time.

---

## Requirements (State in First 5 Minutes)

**Functional:**
- Append-only transaction ledger (double-entry bookkeeping)
- Real-time balance queries per merchant/account
- Payment state machine tracking (PENDING → AUTHORIZED → CAPTURED → SETTLED)
- Full audit trail — who changed what, when

**Non-Functional:**
- Zero data loss (RPO = 0)
- Strong consistency for balances and payment state
- DECIMAL precision (never float for money)
- UK GDPR erasure without breaking audit trail
- Scale: millions of transactions/day, thousands of merchants

---

## Core Schema

```sql
payment_orders (
    id              UUID PRIMARY KEY,        -- idempotency key
    merchant_id     UUID NOT NULL,
    amount          DECIMAL(19,4) NOT NULL,  -- never FLOAT
    currency        VARCHAR(3) NOT NULL,     -- ISO 4217
    status          VARCHAR(20) NOT NULL,    -- state machine
    created_at      TIMESTAMP NOT NULL
);

ledger_entries (
    id              UUID PRIMARY KEY,
    payment_id      UUID REFERENCES payment_orders(id),
    account_id      UUID NOT NULL,
    entry_type      VARCHAR(10) NOT NULL,    -- DEBIT / CREDIT
    amount          DECIMAL(19,4) NOT NULL,
    balance_after   DECIMAL(19,4) NOT NULL,
    created_at      TIMESTAMP NOT NULL       -- append-only
);

wallet_accounts (
    id              UUID PRIMARY KEY,
    merchant_id     UUID NOT NULL UNIQUE,
    balance         DECIMAL(19,4) NOT NULL DEFAULT 0,
    currency        VARCHAR(3) NOT NULL
);
```

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| **Storage engine** | PostgreSQL | ACID, row-level locking, DECIMAL support, mature for finance |
| **Money type** | DECIMAL(19,4) | Float gives `0.1 + 0.2 = 0.30000004`. Unacceptable for money |
| **Ledger mutability** | Append-only | Never UPDATE/DELETE. Corrections = new reversal entries. FCA compliance |
| **Bookkeeping** | Double-entry | Every payment = DEBIT + CREDIT. Sum must always = 0. Catches bugs |
| **Balance tracking** | Materialized in `wallet_accounts` | Avoids scanning entire ledger for current balance |
| **State machine** | Enforced in application layer | DB stores state, app enforces valid transitions |
| **Idempotency** | UUID PK = idempotency key | Duplicate INSERT → PK violation → return existing record |

---

## Double-Entry Bookkeeping

```
Payment of £100: Customer pays Merchant

  ledger_entries:
  ┌─────────────────────────────────────────────────┐
  │ entry_type: DEBIT   | account: customer  | £100 │  ← money leaves
  │ entry_type: CREDIT  | account: merchant  | £100 │  ← money arrives
  └─────────────────────────────────────────────────┘

  Rule: SUM(debits) = SUM(credits) — ALWAYS
  If they don't match → reconciliation catches it immediately
```

---

## Scaling Strategy

```
Single PostgreSQL (up to ~10K TPS writes)
         │
         ▼ (when you outgrow single instance)
Shard by merchant_id (lookup table in Redis)
         │
         ▼
┌────────┐  ┌────────┐  ┌────────┐
│Shard 1 │  │Shard 2 │  │Shard 3 │  ... (all in UK region)
│Merch A │  │Merch B │  │Merch C │
└────────┘  └────────┘  └────────┘
```

- **Shard key:** `merchant_id` — queries are per-merchant (dashboard, refunds, reconciliation)
- **Lookup table:** `merchant_id → shard_id` cached in Redis
- **Hot merchants:** Move to dedicated shard via lookup table update
- **Cross-shard queries:** Data warehouse (CDC pipeline), not production DB

---

## GDPR vs Append-Only Conflict

| Problem | Solution |
|---|---|
| Ledger is append-only (FCA) | Never delete financial records |
| GDPR requires right to erasure | Must delete PII on request |
| **Resolution: Pseudonymization** | Ledger stores opaque tokens, mapping table links to PII. Delete mapping = erasure. Ledger intact. |

---

## Read Path (Dashboard/Reporting)

```
PostgreSQL (primary) → CDC → Data Warehouse / Read Replica
                                      │
                                      ▼
                              Dashboard queries here
                              (eventual consistency, <5s lag)
```

- Never query production DB for analytics
- CQRS: write path (strong consistency) separate from read path (eventual)
- Materialized views for aggregations (daily revenue, success rates)

---

## Interview Q&A

**"Why PostgreSQL over NoSQL?"**
> "Financial data needs ACID transactions, JOINs across orders/ledger/wallet, DECIMAL precision, and unique constraints for idempotency. DynamoDB can't do cross-item transactions or JOINs — we'd reinvent half of PostgreSQL in application code."

**"Why append-only?"**
> "FCA requires tamper-evident audit trails. If you UPDATE a ledger row, you lose the history. Corrections are new reversal entries — the original record always exists. Same principle banks use."

**"How do you prevent double-crediting?"**
> "Double-entry bookkeeping. Every payment creates exactly one DEBIT and one CREDIT. Reconciliation job checks SUM(debits) = SUM(credits) daily. Any mismatch triggers an alert."

**"How do you handle balance queries at scale?"**
> "Materialized balance in `wallet_accounts` — updated transactionally with each ledger entry. We don't scan the entire ledger to compute current balance. The ledger is the source of truth; the wallet is a cached projection."

**"What about time-travel queries?"**
> "`balance_after` on each ledger entry gives you the balance at any point in time. Since it's append-only, the full history is preserved. Query: WHERE account_id = X AND created_at <= timestamp."

---

## One-Page Summary

```
PostgreSQL + DECIMAL(19,4) + append-only ledger + double-entry
     │
     ├── payment_orders: state machine, UUID = idempotency key
     ├── ledger_entries: DEBIT + CREDIT per payment, never mutated
     └── wallet_accounts: materialized balance, updated transactionally

Scale: shard by merchant_id with lookup table
GDPR: pseudonymization (delete mapping, keep ledger)
Reads: CDC → data warehouse (CQRS)
```
