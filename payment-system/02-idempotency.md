# Idempotency — Never Charge Twice

---

## The One-Liner

> An **idempotent** operation produces **no additional effect** if called more than once with the **same input parameters**.

**Translation:** Charge a customer once. If the same request arrives again (retry, duplicate, timeout-resend), **nothing extra happens**.

---

## Why It's Critical

```
Customer clicks "Pay" ──► Request times out ──► System retries ──► DANGER: two charges?
```

Without idempotency, **timeouts + retries = double payments**. This is the most common payment bug.

---

## How It Works — Two-Layer Idempotency

> Two layers: **Redis** catches 99% of duplicates in sub-millisecond. **DB unique constraint** is the ultimate safety net.

```
┌──────────┐         ┌──────────────────┐      ┌─────────┐      ┌────────────┐
│  Client   │         │  Payment Service  │      │  Redis   │      │  Database   │
└─────┬────┘         └────────┬─────────┘      └────┬────┘      └──────┬─────┘
      │                       │                     │                  │
      │  1. Generate UUID     │                     │                  │
      │     (idempotency key) │                     │                  │
      │                       │                     │                  │
      │  2. Send request      │                     │                  │
      │  ┌─────────────────►  │                     │                  │
      │  │ HTTP Header:       │                     │                  │
      │  │ Idempotency-Key:   │                     │                  │
      │  │ "abc-123-def"      │                     │                  │
      │                       │  3. SETNX check     │                  │
      │                       │  ──────────────────► │                  │
      │                       │                     │                  │
      │                       │  ┌─ Key exists?      │                  │
      │                       │  │  DUPLICATE ──────►│  Return cached   │
      │                       │  │  (sub-ms reject)  │  result          │
      │                       │  │                   │                  │
      │                       │  └─ Key new?         │                  │
      │                       │     SET with TTL     │                  │
      │                       │                     │                  │
      │                       │  4. INSERT payment   │                  │
      │                       │  ──────────────────────────────────►   │
      │                       │                     │                  │
      │                       │     ┌─ New key? INSERT succeeds        │
      │                       │     │                │                  │
      │                       │     └─ Duplicate?    │                  │
      │                       │        Constraint    │                  │
      │                       │        violation     │                  │
      │                       │        (safety net)  │                  │
      │                       │                     │                  │
      │  5. Response          │                     │                  │
      │  ◄─────────────────   │                     │                  │
```

### Layer 1: Redis SETNX (Speed)

```
SETNX idempotency:{key} {payment_id}    ← atomic "set if not exists"
EXPIRE idempotency:{key} 86400           ← TTL: 24 hours
```

- **Sub-millisecond** duplicate detection
- Catches **99% of duplicates** — click-click-click, network retries, timeout resends
- If Redis is down, **fall through to Layer 2** — system still works

### Layer 2: DB Unique Constraint (Safety Net)

```sql
CREATE TABLE payment_orders (
    id UUID PRIMARY KEY,          -- ← This IS the idempotency key
    amount DECIMAL NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP
);

-- Second INSERT with same UUID → constraint violation → no double charge
```

- **Ultimate guarantee** — even if Redis fails, evicts the key, or has a split-brain
- No distributed locks needed — just a database unique constraint
- The idempotency key **IS** the payment order ID (same UUID)

### Why Two Layers?

| Scenario | Redis | DB | Result |
|----------|-------|-----|--------|
| Normal duplicate (click-click) | Catches it in **<1ms** | Never reached | Fast reject |
| Redis down | Misses it | **Catches it** via constraint | Still safe |
| Redis key evicted (TTL) | Misses it | **Catches it** via constraint | Still safe |
| Redis split-brain | Might miss it | **Catches it** via constraint | Still safe |
| First request | Sets key | INSERT succeeds | Payment processed |

> **Redis is an optimisation. The DB is the guarantee.** If you can only have one, keep the DB constraint.

---

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| **Who generates the key?** | **Client** | Client controls retry behavior and knows which requests are duplicates |
| **What format?** | **UUID** | Universally unique, no coordination needed |
| **Where is it sent?** | **HTTP Header** | Clean separation from business payload |
| **Layer 1 (speed)?** | **Redis SETNX** | Sub-ms duplicate detection, catches 99% of duplicates |
| **Layer 2 (safety)?** | **DB unique constraint** | Ultimate guarantee — works even if Redis fails |
| **Key = Payment ID?** | **Yes** | Same UUID serves as both idempotency key and order ID |

---

## The Timeout + Idempotency Combo

```
Request ──► Times out ──► Client retries with SAME idempotency key
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            Original request              Original request
            actually failed               actually succeeded
                    │                               │
                    ▼                               ▼
            New INSERT succeeds           Duplicate key blocked
            (processes normally)          (returns original result)
                    │                               │
                    ▼                               ▼
              Correct: charged once          Correct: charged once
```

> **Either way, the customer is charged exactly once.** This is the magic.

---

## Interview Keywords to Remember

- **Idempotency key** = UUID generated by client
- **HTTP header** = where the key travels
- **Two-layer idempotency** = Redis SETNX (speed) + DB unique constraint (safety)
- **Redis SETNX** = "Set if Not eXists" — atomic, sub-millisecond duplicate check
- **DB unique constraint** = ultimate safety net — works even if Redis fails
- **Exactly-once guarantee** = the outcome
- **Same key = same payment order ID** = the implementation trick
- **Timeout + retry + idempotency** = the trifecta that prevents double-charging

---

**Next:** [03-security.md](03-security.md) — Protecting the money
