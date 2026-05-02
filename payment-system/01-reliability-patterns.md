# Reliability Patterns — Keeping Payments Alive

---

## The Golden Rule

> A payment system must **never lose money** and **never charge twice**. Every pattern below exists to enforce this.

---

## Async vs Sync Communication

### Default: **Asynchronous** (use this unless forced otherwise)

| Benefit | Why It Matters |
|---------|----------------|
| **Scalability** | Handles massive request volume without **blocking the main thread** |
| **Performance** | Processes transactions without **waiting for external responses** |
| **Fault Tolerance** | Failed external call? **Retry or reroute** — no crash |
| **Loose Coupling** | Swap/modify components **without breaking the system** |
| **Background Processing** | Frees resources, **reduces transaction completion time** |

### Exception: **Synchronous** (only when real-time is mandatory)

> **Physical store payments** require **real-time authorization** — you must know immediately if it's success or failure.

**Rule of thumb:** Use sync **only if there is no other way**. Prefer async in all other cases.

---

## Three Categories of Errors

| Type | Symbol | Example | Strategy |
|------|--------|---------|----------|
| **System Failures** | :boom: | Network down, server crash | **Retry + Timeout + Fallback** |
| **Poison Pill Errors** | :pill: | Malformed message that can't be consumed | **Dead Letter Queue** |
| **Functional Bugs** | :bug: | No technical error but results are invalid | **Monitoring + Alerts** |

---

## Pattern 1: Retry Strategies

### Four Levels (worst → best)

```
Immediate Retry      ──► Wastes resources, system likely still broken
    │                     "unlikely the issue has been solved in such a short time"
    ▼
Fixed Intervals      ──► Consistent wait time, gives system breathing room
    │
    ▼
Incremental Intervals ──► Increasing wait times
    │
    ▼
Exponential Backoff  ──► BEST: doubles wait time each retry (2^n)
                          1s → 2s → 4s → 8s → 16s...
```

### Why Add Jitter?

> Exponential backoff alone isn't enough. If 1,000 requests fail at the same time, they all retry at 2s, then 4s, then 8s — same time, every time. This is the **thundering herd**.

**Jitter** adds a **random offset** to each retry interval, spreading retries across time:

```
Without jitter:  1,000 requests all retry at exactly 2s, 4s, 8s  ← thundering herd
With jitter:     1,000 requests retry between 1-3s, 2-6s, 4-12s  ← distributed load

Formula: delay = min(cap, base * 2^attempt) + random(0, base * 2^attempt)
```

| Strategy | Formula | When |
|----------|---------|------|
| **Full jitter** | `random(0, base * 2^n)` | Default — best spread |
| **Equal jitter** | `(base * 2^n / 2) + random(0, base * 2^n / 2)` | When you want a guaranteed minimum wait |
| **Decorrelated jitter** | `random(base, previous_delay * 3)` | When retries are independent |

### Key Takeaway

> **Exponential backoff + jitter** is the gold standard — backoff prevents overwhelming a recovering service, jitter prevents the **thundering herd** when many clients retry simultaneously.

---

## Pattern 2: Timeout

**Problem:** Without timeouts, a hung service means **indefinite waiting**.

**Solution:** Set timeouts that **balance allowing slower responses** with **not waiting forever**.

### The Dangerous Edge Case

> A request **times out** → your system treats it as failed → **but the payment actually went through** on the other side.

**Result:** Customer gets **double-charged** or sees incorrect status.

**Fix:** Combine **timeout + idempotency + retry** (see [02-idempotency.md](02-idempotency.md))

---

## Pattern 3: Fallback

**What:** If a service call fails, **fill in a fallback value** and continue execution.

**When to use:** Only when the **fallback value is acceptable** — balances **risk vs. customer satisfaction**.

**Example:** If a fraud-check service is down, you might temporarily accept low-risk transactions while queueing high-risk ones.

> **Warning:** Fallbacks are a trade-off — never use them for critical financial validations.

---

## Pattern 4: Circuit Breaker

**Problem:** A downstream service is failing. Retries keep hitting it, wasting resources and making recovery harder.

**Solution:** Stop calling the failing service entirely. Give it time to recover.

### Three States

```
         success                                  success
           │                                        │
    ┌──────▼──────┐   failures > threshold   ┌──────▼──────┐
    │   CLOSED    │ ─────────────────────►    │    OPEN     │
    │ (normal —   │                           │ (fail fast, │
    │  all calls  │   ◄─────────────────────  │  don't call │
    │  pass thru) │    probe succeeds         │  downstream)│
    └─────────────┘                           └──────┬──────┘
                                                     │
                                              timer expires
                                                     │
                                              ┌──────▼──────┐
                                              │  HALF-OPEN  │
                                              │ (let ONE    │
                                              │  request    │
                                              │  through)   │
                                              └─────────────┘
                                              success → CLOSED
                                              failure → OPEN
```

| State | Behaviour | When |
|-------|-----------|------|
| **Closed** | All requests pass through normally | Service is healthy |
| **Open** | Requests **fail immediately** — no call to downstream | Failure threshold exceeded (e.g., 5 failures in 30s) |
| **Half-Open** | Let **one probe request** through to test recovery | After a cooldown timer expires |

### Per-Merchant Circuit Breakers

> For webhook delivery, use **per-merchant circuit breakers** — one merchant's failing endpoint shouldn't block notifications to all other merchants.

### When to Use

- **Between services**: Payment Service → PSP (if PSP is down, stop hammering it)
- **Webhook delivery**: per-merchant breakers prevent one bad endpoint from affecting others
- **Combined with retry**: retry handles transient errors, circuit breaker handles sustained failures

> **Key insight:** Retries help when a service **might recover per-request**. Circuit breakers help when a service is **down for a while**. Use both together.

---

## Pattern 5: Message Queue (Kafka)

```
Payment Event ───► Kafka (99.999% availability)
                       │
                       ▼
              Payment NOT complete
              until event is persisted here
```

### Why Kafka?

- Its **only job is storing messages** — simple = reliable
- **99.999% availability** (far higher than business services)
- Payment events are **never lost**, even during downstream failures
- For every order placed or paid → **an event is created in Kafka**

### Critical Rule

> The payment operation is **NOT considered complete** until the event is safely stored in the message queue.

---

## Pattern 6: Dead Letter Queue (DLQ)

**For messages that repeatedly fail processing:**

| Failure Type | What Happens |
|-------------|--------------|
| **Incompatible/malformed data** | Saved to DLQ for **later debugging** |
| **Service is down** | Stored in **persistent queue** until service recovers |

```
Normal Queue ───► Consumer fails ───► Retry N times ───► Still failing?
                                                              │
                                                              ▼
                                                     Dead Letter Queue
                                                     (isolate & debug later)
```

> DLQ **isolates** problematic messages so they **don't block** healthy traffic.

---

## The Reliability Toolbox (Summary)

| Tool | Purpose |
|------|---------|
| **Redundancy** | Resilience during internal system failures |
| **Kafka** | Persist messages — payment guarantee |
| **Retry + Backoff + Jitter** | Handle transient failures without thundering herd |
| **Timeout** | Prevent indefinite waiting on hung services |
| **Fallback** | Graceful degradation when acceptable |
| **Circuit Breaker** | Stop calling failing services, let them recover |
| **Message Queues** | Avoid **overloading** services |
| **Idempotency** | Exactly-once processing guarantee |
| **Dead Letter Queue** | Isolate and debug persistent failures |

---

**Next:** [02-idempotency.md](02-idempotency.md) — The key to "never charge twice"
