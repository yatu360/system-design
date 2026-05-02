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

### Key Takeaway

> **Exponential backoff** is the gold standard — it **prevents thundering herd** and gives downstream services time to recover.

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

## Pattern 4: Message Queue (Kafka)

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

## Pattern 5: Dead Letter Queue (DLQ)

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
| **Retry + Timeout + Fallback** | Handle transient failures gracefully |
| **Message Queues** | Avoid **overloading** services |
| **Idempotency** | Exactly-once processing guarantee |
| **Dead Letter Queue** | Isolate and debug persistent failures |

---

**Next:** [02-idempotency.md](02-idempotency.md) — The key to "never charge twice"
