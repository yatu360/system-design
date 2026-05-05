# Daily Batch Processing / Settlement Systems

> Design a fault-tolerant system that processes millions of rows daily to settle funds with external processors and banks.

---

## Requirements (State in First 5 Minutes)

**Functional:**
- Aggregate captured payments per merchant per day
- Calculate net settlement (payments minus refunds minus fees)
- Generate settlement files for banks/processors
- Transfer funds to merchant bank accounts
- Reconcile: our records vs bank confirmation

**Non-Functional:**
- Process millions of transactions in a bounded time window (e.g., 2-4 hours)
- Exactly-once settlement — never pay a merchant twice, never miss a payment
- Fault-tolerant — resume from failure point, don't restart from scratch
- Idempotent — re-running the same batch produces the same result
- Audit trail — full record of what was settled, when, to whom

---

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │         SETTLEMENT PIPELINE             │
                    │                                         │
 Midnight trigger   │  1. COLLECT    Gather today's captured  │
 (cron / scheduler) │               payments per merchant    │
         │          │                                         │
         ▼          │  2. AGGREGATE  Net amount per merchant  │
  ┌─────────────┐   │               (payments - refunds -    │
  │  Scheduler  │──▶│                fees)                   │
  │  (Airflow / │   │                                         │
  │   Step Fn)  │   │  3. GENERATE  Settlement file per bank │
  └─────────────┘   │               (CSV/ISO 20022 XML)      │
                    │                                         │
                    │  4. SUBMIT    Send to bank via SFTP /   │
                    │               API                       │
                    │                                         │
                    │  5. RECONCILE Match bank confirmation   │
                    │               against our records       │
                    └─────────────────────────────────────────┘
```

---

## Settlement Flow (Detail)

### Step 1: Collect

```sql
SELECT merchant_id, SUM(amount) as gross, COUNT(*) as tx_count
FROM payment_orders
WHERE status = 'CAPTURED'
  AND settlement_batch_id IS NULL
  AND captured_at >= '2024-01-15 00:00:00'
  AND captured_at <  '2024-01-16 00:00:00'
GROUP BY merchant_id;
```

- Query runs against **read replica** or data warehouse — never production
- Mark collected payments with `settlement_batch_id` to prevent double-processing
- Batch ID is idempotent: same date + same merchant = same batch

### Step 2: Aggregate

```
Merchant A:
  Gross payments:    £10,000 (150 transactions)
  Refunds:          -£500   (3 refunds)
  Platform fee (2%): -£190
  ─────────────────────────
  Net settlement:    £9,310
```

### Step 3: Generate File

```
settlement_batch (
    id              UUID PRIMARY KEY,
    merchant_id     UUID NOT NULL,
    settlement_date DATE NOT NULL,
    gross_amount    DECIMAL(19,4),
    refund_amount   DECIMAL(19,4),
    fee_amount      DECIMAL(19,4),
    net_amount      DECIMAL(19,4),
    status          VARCHAR(20),  -- PENDING → SUBMITTED → CONFIRMED → FAILED
    file_reference  VARCHAR(255),
    created_at      TIMESTAMP
);

UNIQUE (merchant_id, settlement_date)  -- idempotent: one settlement per merchant per day
```

### Step 4: Submit to Bank

- SFTP upload (traditional) or API call (modern)
- File format: ISO 20022 XML or bank-specific CSV
- Retry with exponential backoff on failure
- Store bank acknowledgement reference

### Step 5: Reconcile

```
Next day: Bank sends confirmation file
  → Match each line to our settlement_batch records
  → Status: CONFIRMED (amounts match) or DISCREPANCY (investigate)

Discrepancy types:
  - Amount mismatch (FX, fees)
  - Missing settlement (bank didn't process)
  - Extra settlement (we don't have a record)
```

---

## Fault Tolerance

| Failure | How We Handle It |
|---|---|
| **Crash mid-batch** | Checkpoint after each step. Resume from last checkpoint, not from scratch |
| **Duplicate run** | `UNIQUE (merchant_id, settlement_date)` — second run is a no-op |
| **Bank rejects file** | Alert → fix → regenerate → resubmit. Original batch marked FAILED |
| **Partial bank success** | Per-merchant status tracking. Retry only failed merchants |
| **Data inconsistency** | Reconciliation catches mismatches next day. Alert on any discrepancy |

### Checkpointing

```
Batch status progression:
  COLLECTING → AGGREGATED → FILE_GENERATED → SUBMITTED → CONFIRMED

On restart:
  - Status = COLLECTING? → Re-collect (idempotent query)
  - Status = AGGREGATED? → Skip to file generation
  - Status = SUBMITTED? → Check with bank before resubmitting
```

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| **Orchestrator** | AWS Step Functions / Airflow | Visual pipeline, built-in retry, state tracking, dead-letter |
| **Processing** | Batch (not streaming) | Settlement is inherently daily — no need for real-time |
| **Idempotency** | Unique constraint on (merchant_id, date) | Re-running same day = safe no-op |
| **Data source** | Read replica / warehouse | Never compete with live payment traffic |
| **File format** | ISO 20022 | Industry standard for bank communication |
| **Time window** | Run at 02:00, complete by 06:00 | Off-peak, before business hours |
| **Partitioning** | Process by merchant in parallel | Independent merchants = embarrassingly parallel |

---

## Scaling for Millions of Rows

```
Sequential (slow):
  FOR each merchant: collect → aggregate → generate → submit

Parallel (fast):
  ┌─ Merchant A: collect → aggregate → generate ─┐
  ├─ Merchant B: collect → aggregate → generate ─┼─→ Batch submit to bank
  ├─ Merchant C: collect → aggregate → generate ─┤
  └─ Merchant D: collect → aggregate → generate ─┘
```

- **Fan-out by merchant** — each merchant's settlement is independent
- **Limit concurrency** — don't overwhelm bank APIs (rate limit: 100 concurrent)
- **Chunking** — for huge merchants (Amazon-scale), chunk by time window within the day
- **Database indexing** — `(status, captured_at, merchant_id)` composite index for the collection query

---

## Interview Q&A

**"How do you ensure exactly-once settlement?"**
> "Unique constraint on (merchant_id, settlement_date). If the batch crashes and restarts, it can't create a duplicate settlement. Each batch has a status — we check status before re-executing any step. The pipeline is idempotent end-to-end."

**"What if the bank is down?"**
> "Retry with exponential backoff. If still failing after N attempts, alert ops and mark batch as PENDING_RETRY. Settlements for other banks proceed independently. The merchant gets paid next cycle with a combined amount."

**"How do you handle millions of rows efficiently?"**
> "Fan-out by merchant — each settlement is independent and can run in parallel. Collection query runs on a read replica with a composite index. We chunk the work and cap concurrency to avoid overwhelming downstream systems."

**"Why not real-time settlement?"**
> "Banks operate on batch cycles — they don't support real-time fund transfers for settlement. Even if they did, batching gives us reconciliation windows, error correction before submission, and cost efficiency (one transfer per merchant per day vs thousands)."

**"How do you reconcile?"**
> "Next-day: bank sends a confirmation file. We match each line to our settlement records. Any mismatch — amount, missing record, extra record — triggers an alert. Reconciliation runs as a separate pipeline that reads both our DB and the bank's file."

**"What about timezone issues?"**
> "Settlement date is defined in UTC. Cut-off is midnight UTC. All timestamps stored in UTC. Merchant-facing display converts to local timezone."

---

## One-Page Summary

```
Nightly batch: Collect → Aggregate → Generate file → Submit to bank → Reconcile

Fault-tolerant:  Checkpoints per step, resume on failure
Idempotent:      UNIQUE (merchant_id, date), re-run = no-op
Parallel:        Fan-out by merchant, cap concurrency
Reconciliation:  Next-day match against bank confirmation

Tech: Step Functions/Airflow + Read Replica + SFTP/API to bank
```
