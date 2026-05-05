# Fraud Detection System

> Design an event-driven system that flags high-risk transactions in real-time based on streaming data.

---

## Requirements (State in First 5 Minutes)

**Functional:**
- Score every transaction in real-time before authorization
- Flag/block high-risk transactions (score > threshold)
- Rules engine (velocity checks, geo anomalies, amount thresholds)
- ML model scoring (learned patterns)
- Analyst dashboard for reviewing flagged transactions
- Feedback loop: analyst decisions improve the model

**Non-Functional:**
- Latency: < 100ms decision (inline with payment flow — can't slow checkout)
- Availability: 99.99% (if fraud system is down, must decide: block all or allow all)
- Scale: score every transaction (thousands TPS)
- False positive rate: < 1% (blocking legitimate customers = lost revenue)
- Eventual consistency acceptable for model updates

---

## Architecture

```
                         < 100ms
Customer ──▶ Payment Service ──▶ Fraud Service ──▶ Decision
                   │                    │
                   │              ┌─────┴──────┐
                   │              │             │
                   │         Rules Engine    ML Model
                   │         (deterministic) (probabilistic)
                   │              │             │
                   │              └─────┬──────┘
                   │                    │
                   │              Combined Score
                   │              ┌─────┴──────┐
                   │              │             │
                   │           APPROVE       REVIEW / BLOCK
                   │
                   ▼
             Kafka: payment-events
                   │
                   ▼
         ┌─────────────────┐
         │  Feature Store  │  ← async enrichment
         │  (Redis)        │     (update velocity counters,
         └─────────────────┘      geo history, device fingerprints)
```

---

## Two-Layer Scoring

| Layer | Type | Latency | Examples |
|---|---|---|---|
| **Rules Engine** | Deterministic | < 10ms | Velocity: > 5 transactions in 1 minute. Amount: > £5,000 single transaction. Geo: card used in UK then US within 1 hour |
| **ML Model** | Probabilistic | < 50ms | Learned patterns from historical fraud. Device fingerprint anomalies. Spending behavior deviation |

### Combined Decision

```
rules_score  = evaluate_rules(transaction, features)    -- 0 to 100
ml_score     = model.predict(transaction, features)     -- 0.0 to 1.0

combined = (rules_score * 0.4) + (ml_score * 100 * 0.6)

if combined > 80:  BLOCK   (auto-decline)
if combined > 50:  REVIEW  (queue for analyst)
if combined <= 50: APPROVE (proceed with payment)
```

---

## Feature Store (Redis)

Real-time features computed from streaming data:

```
Key                                    Value              TTL
─────────────────────────────────────────────────────────────
fraud:velocity:card:4242...            7 (tx last hour)   1h
fraud:velocity:customer:cust_123      12 (tx last day)   24h
fraud:geo:last_country:card:4242      "GB"               7d
fraud:device:fingerprint:cust_123     "fp_abc123"        30d
fraud:amount:avg:card:4242            "45.50"            30d
fraud:amount:max:card:4242            "250.00"           30d
```

- Updated **asynchronously** after each transaction (Kafka consumer)
- Read **synchronously** during scoring (< 5ms Redis lookup)
- If Redis is down: degrade to rules-only scoring (no velocity checks)

---

## Rules Engine

```yaml
rules:
  - name: high_velocity
    condition: tx_count_last_hour > 5
    score: 40
    
  - name: large_amount
    condition: amount > 5000
    score: 30
    
  - name: geo_anomaly
    condition: country != last_country AND time_since_last < 2h
    score: 60
    
  - name: new_card
    condition: card_age < 1h AND amount > 500
    score: 35
    
  - name: velocity_spike
    condition: tx_count_last_hour > 3 * avg_hourly_rate
    score: 50
```

- Rules are **hot-reloadable** — update without deployment
- Stored in config service or database, cached in memory
- New rules can be added by fraud analysts, not just engineers

---

## ML Model

| Aspect | Choice | Why |
|---|---|---|
| **Model type** | Gradient boosted trees (XGBoost/LightGBM) | Fast inference (< 50ms), handles tabular data well, interpretable |
| **NOT deep learning** | Neural nets are slower and harder to explain to regulators | FCA requires explainable decisions |
| **Serving** | Pre-loaded in memory (sidecar or embedded) | No network hop for inference = low latency |
| **Training** | Offline batch (daily/weekly retrain on labelled data) | New fraud patterns incorporated regularly |
| **Features** | ~50-100 features (velocity, amount, geo, device, time, history) | More features = better signal |

### Model Update Flow

```
Labelled data (analyst decisions + chargebacks)
        │
        ▼
  Offline Training (daily)
        │
        ▼
  Model Registry (versioned)
        │
        ▼
  Canary Deployment (5% traffic)
        │
        ▼
  Full Rollout (if metrics improve)
```

---

## Failure Modes

| Failure | Strategy | Justification |
|---|---|---|
| **Fraud service down** | Default APPROVE (with logging) | Blocking all payments = revenue loss > fraud loss. Flag for manual review later |
| **Redis down (features unavailable)** | Rules-only scoring (no velocity) | Partial protection better than none |
| **ML model timeout** | Rules-only score | Rules are deterministic and fast |
| **High latency (> 100ms)** | Circuit breaker → skip scoring | Payment latency SLA > fraud detection perfection |
| **False positive spike** | Auto-adjust thresholds down | Sudden block spike usually = bad model, not fraud wave |

> **Key trade-off:** When fraud system fails, we **allow** payments through (and log for review) rather than block all payments. Revenue protection > fraud prevention in degraded mode.

---

## Async Enrichment Pipeline

```
Kafka: payment-events
        │
        ▼
┌──────────────────┐
│ Fraud Enrichment │
│ Consumer         │
│                  │
│ • Update velocity counters in Redis
│ • Update geo history
│ • Update device fingerprint
│ • Feed labelled outcomes to training pipeline
│ • Generate alerts for analyst dashboard
└──────────────────┘
```

- This runs AFTER the payment decision (async, not blocking)
- Keeps feature store current for the NEXT transaction
- Feeds the analyst review queue

---

## Analyst Feedback Loop

```
Transaction flagged (score > 50)
        │
        ▼
  Analyst Review Queue
        │
        ├── Confirm fraud    → label as FRAUD   → retrain model
        └── Clear as legit   → label as LEGIT   → retrain model
                                                    (reduces false positives)
```

- Analyst decisions become **training data**
- Chargebacks (reported 30-90 days later) are the strongest fraud signal
- Model improves over time as labelled dataset grows

---

## Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| **Inline vs async** | Inline scoring (sync, pre-auth) | Must block fraud BEFORE money moves |
| **Rules + ML** | Both | Rules catch known patterns instantly. ML catches novel patterns. Neither alone is sufficient |
| **Feature storage** | Redis | Sub-millisecond reads, TTL for auto-expiry, perfect for counters |
| **Default on failure** | Allow (not block) | Revenue loss from blocking all > fraud loss from allowing some |
| **Model type** | Gradient boosted trees | Fast, explainable (FCA requirement), good on tabular data |
| **Threshold tuning** | Per-merchant configurable | High-risk merchants (gambling) need lower thresholds than low-risk (SaaS) |

---

## Interview Q&A

**"Why inline and not async?"**
> "Fraud detection must happen before authorization. Once money moves, recovering it is expensive (chargebacks, disputes). We score in < 100ms inline with the payment flow — the customer doesn't notice."

**"Why both rules and ML?"**
> "Rules catch known, deterministic patterns instantly — velocity spikes, geo impossibilities. ML catches novel patterns humans haven't written rules for. Rules are your safety net on day one; ML improves over time with data."

**"What if the fraud system is slow or down?"**
> "Circuit breaker. If latency exceeds 100ms or the service is unreachable, we allow the payment through and log it for async review. Revenue loss from blocking all payments far exceeds fraud loss from allowing a few bad ones during a brief outage."

**"How do you handle false positives?"**
> "Analyst feedback loop. Every flagged-but-legitimate transaction trains the model to be less aggressive on that pattern. We track false positive rate as a key metric — target < 1%. Too many false positives = customers abandoning checkout."

**"How does the model stay current?"**
> "Daily retrain on new labelled data — analyst decisions plus chargeback outcomes. Canary deployment: new model sees 5% of traffic first. If fraud catch rate improves and false positives don't spike, full rollout."

**"How do you explain decisions to regulators?"**
> "Gradient boosted trees are interpretable — we can output feature importance per decision. 'This transaction was flagged because: velocity score 40 (7 transactions in 1 hour), geo anomaly score 60 (UK to US in 30 minutes).' Deep learning can't do this easily."

---

## One-Page Summary

```
Inline scoring (< 100ms) → Rules + ML → APPROVE / REVIEW / BLOCK

Features:     Redis (velocity counters, geo, device) — updated async via Kafka
Rules:        Deterministic, hot-reloadable, analyst-configurable
ML Model:     XGBoost, retrained daily, canary deployed
Failure mode: Allow payments through (revenue > fraud loss)
Feedback:     Analyst decisions + chargebacks → retrain model

Key numbers: < 100ms latency, < 1% false positive, 99.99% available
```
