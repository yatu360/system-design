# Kafka Fundamentals — Explained Simply

> Every concept is explained from scratch, then tied to our payment system. No prior Kafka knowledge needed.

---

## What Is Kafka?

Think of Kafka as a **shared notebook** that everyone writes to and reads from independently.

- The **Payment Service** writes a new line every time something happens: "payment pay_abc was created", "payment pay_abc was authorized", etc.
- The **Ledger Service** reads the notebook and records debits and credits.
- The **Wallet Service** reads the same notebook and updates merchant balances.
- The **Webhook Dispatcher** reads the same notebook and notifies merchants.

Each reader has their own bookmark. They read at their own pace. If one falls behind, the others aren't affected. If one crashes, it picks up where its bookmark was — the notebook still has everything.

> **One-sentence definition:** Kafka is a distributed, append-only log that stores ordered events so multiple independent services can read them at their own pace, with replay capability.

---

## Core Concepts — One at a Time

### 1. Topic — "Which notebook?"

A **topic** is a named stream of events. Think of it as a separate notebook for each type of event.

**Our payment system has 3 topics:**

| Topic | What Goes In It | Who Reads It |
|---|---|---|
| `payment-events` | Every payment lifecycle event (created, authorized, captured, settled, refunded) | payment-processor, ledger-service, wallet-service, webhook-dispatcher |
| `payment-status-changes` | Only status transitions (e.g. AUTHORIZED → CAPTURED) | webhook-dispatcher, data-warehouse |
| `webhook-events` | Outbound webhook delivery attempts and results | monitoring, retry-handler |

---

### 2. Partition — "Which page of the notebook?"

A topic is split into **partitions**. Each partition is an independent, ordered sequence of events.

**Why split?** So multiple consumers can read in parallel — each consumer reads from different partitions at the same time.

```
Topic: payment-events (12 partitions)

Partition 0:  [pay_abc: PENDING] [pay_abc: AUTHORIZED] [pay_abc: CAPTURED]
Partition 1:  [pay_def: PENDING] [pay_def: AUTHORIZED]
Partition 2:  [pay_ghi: PENDING] [pay_ghi: AUTHORIZED] [pay_ghi: CAPTURED] [pay_ghi: SETTLED]
...
Partition 11: [pay_xyz: PENDING]
```

**How does Kafka decide which partition?** You provide a **partition key**. Kafka hashes it and always sends the same key to the same partition.

**Our partition key is `payment_id`.** This means ALL events for `pay_abc` always go to the same partition — guaranteed.

> **Our setup:** 12 partitions per topic. Enough parallelism for peak traffic, not so many that broker management becomes painful.

---

### 3. Offset — "Which line am I on?"

Every message in a partition has a number called an **offset**. It's just the position — message 0, message 1, message 2, etc.

```
Partition 0:

Offset:   0              1                2
          [pay_abc:      [pay_abc:        [pay_abc:
           PENDING]       AUTHORIZED]      CAPTURED]
                                              ▲
                                              │
                                      Consumer is here
                                      (committed offset = 2)
```

Each consumer tracks its own offset — "I've read up to message 2 in partition 0." If it crashes and restarts, it picks up from where it left off.

**Important:** Offsets are per-partition, not per-topic. Partition 0 has its own offsets. Partition 1 has completely separate offsets.

---

### 4. Broker — "Which server stores the data?"

A **broker** is a Kafka server. Our cluster has **3 brokers**. Each broker stores some of the partitions.

```
Broker 1: Partition 0, Partition 3, Partition 6, Partition 9
Broker 2: Partition 1, Partition 4, Partition 7, Partition 10
Broker 3: Partition 2, Partition 5, Partition 8, Partition 11
```

If one broker dies, the others still have their partitions. And because of replication (explained below), the dead broker's data isn't lost.

---

### 5. Producer — "Who writes to the notebook?"

A **producer** is any service that writes events to Kafka.

**In our system, the Payment Service is the main producer.** When a payment is created, authorized, or captured, the Payment Service writes an event to the `payment-events` topic.

```
Payment Service (Producer)
    │
    │  "pay_abc was AUTHORIZED"
    │  partition key = "pay_abc"
    │
    ▼
Kafka: payment-events, Partition 0
    [...] [pay_abc: AUTHORIZED]  ← appended here
```

---

### 6. Consumer — "Who reads the notebook?"

A **consumer** is any service that reads events from Kafka.

**In our system, the payment-processor is a consumer.** It reads events from `payment-events` and calls the PSP to process them.

The consumer reads events in order within each partition:

```
payment-processor reads Partition 0:
    → pay_abc: PENDING     (processes it)
    → pay_abc: AUTHORIZED  (processes it)
    → pay_abc: CAPTURED    (processes it)
    
    Always in order. Never sees CAPTURED before AUTHORIZED.
```

---

### 7. Consumer Group — "Teams of readers"

A **consumer group** is a team of consumers that work together to read a topic. Within a group, each partition is assigned to exactly one consumer — no two consumers in the same group read the same partition.

**Our payment system has 4 consumer groups, each doing a different job:**

```
Topic: payment-events (12 partitions)

Consumer Group: payment-processor (3 instances)
┌──────────────────────────────────────────────┐
│  Instance A ← Partitions 0, 1, 2, 3         │  → calls the PSP
│  Instance B ← Partitions 4, 5, 6, 7         │
│  Instance C ← Partitions 8, 9, 10, 11       │
└──────────────────────────────────────────────┘

Consumer Group: ledger-service (2 instances)
┌──────────────────────────────────────────────┐
│  Instance X ← Partitions 0-5                 │  → appends debit + credit
│  Instance Y ← Partitions 6-11               │
└──────────────────────────────────────────────┘

Consumer Group: wallet-service (2 instances)
┌──────────────────────────────────────────────┐
│  Instance P ← Partitions 0-5                 │  → updates merchant balance
│  Instance Q ← Partitions 6-11               │
└──────────────────────────────────────────────┘

Consumer Group: webhook-dispatcher (3 instances)
┌──────────────────────────────────────────────┐
│  Instance M ← Partitions 0, 1, 2, 3         │  → notifies merchants
│  Instance N ← Partitions 4, 5, 6, 7         │
│  Instance O ← Partitions 8, 9, 10, 11       │
└──────────────────────────────────────────────┘
```

**Key rules:**
- Each group reads ALL events independently — the ledger-service and wallet-service both see every payment event
- Within a group, one partition = one consumer. No sharing.
- More consumers than partitions = idle consumers (with 12 partitions, adding a 13th consumer in a group does nothing)
- **Max parallelism = number of partitions.** Want 12 parallel consumers? Need at least 12 partitions.

---

## Why Order Matters — The Most Important Concept

Kafka guarantees order **within a partition only** — NOT across partitions.

### What goes right (same partition):

```
Payment pay_abc — partition key = "pay_abc" → all go to Partition 0

Partition 0:
    [pay_abc: PENDING] → [pay_abc: AUTHORIZED] → [pay_abc: CAPTURED]
    
Consumer reads in this exact order ✓
    1. Pending    → save to DB
    2. Authorized → reserve funds
    3. Captured   → charge the card
```

### What goes wrong (different partitions):

```
If pay_abc events somehow landed on different partitions:

Partition 0: [pay_abc: CAPTURED]     ← consumer reads this first
Partition 1: [pay_abc: PENDING]      ← consumer reads this second

Consumer tries to capture a payment that doesn't exist yet ✗
```

### How we prevent this:

**Partition key = `payment_id`.** Kafka hashes the payment ID and always routes it to the same partition. Every event for `pay_abc` goes to the same partition, so they're always in order.

| What We Partition By | What It Guarantees | When To Use |
|---|---|---|
| `payment_id` | All events for one payment are ordered (PENDING → AUTHORIZED → CAPTURED) | **Our default** — the state machine must be ordered |
| `merchant_id` | All events for one merchant are ordered | When you need merchant-level ordering (webhook delivery) |

> **The rule:** Same partition key = same partition = ordered. Different keys = different partitions = no ordering between them. Cross-payment ordering doesn't matter — pay_abc and pay_def are independent.

---

## How Kafka Stays Safe — Replication

What if the server storing Partition 0 dies? Without copies, that data is gone.

Kafka solves this with **replication** — every partition is copied to multiple brokers.

```
Partition 0, Replication Factor = 3:

Broker 1: [LEADER]     ← all reads and writes go here
Broker 2: [FOLLOWER]   ← keeps a copy, stays in sync
Broker 3: [FOLLOWER]   ← keeps a copy, stays in sync

If Broker 1 dies:
    → Broker 2 becomes the new LEADER (automatic, takes seconds)
    → No data is lost because Broker 2 has the same copy
```

### Our replication settings (and what each one means):

| Setting | Our Value | What It Means in Plain English |
|---|---|---|
| `replication.factor` | `3` | Every partition exists on 3 different brokers. Even if 2 brokers die, one copy survives. |
| `min.insync.replicas` | `2` | A write only succeeds if at least 2 of the 3 copies have it. If only 1 broker is alive, Kafka refuses writes rather than risking data loss. |
| `acks` | `all` | The producer waits until ALL in-sync replicas confirm they've stored the event before considering it "written." |

**What this means together:** When the Payment Service writes "pay_abc was AUTHORIZED", that event is stored on at least 2 brokers before the Payment Service gets a success response. One broker can die immediately after — the event is safe.

> **Why this matters for payments:** Losing a payment event means losing money. These three settings together mean: no payment event is ever lost unless 2 out of 3 brokers die simultaneously (which is why they're in different availability zones).

---

## Delivery Guarantees — What Can Go Wrong?

There are two things that can go wrong when delivering messages:

1. **Message is lost** — the consumer never processes it (customer paid but we never recorded it)
2. **Message is processed twice** — the consumer processes it twice (customer is charged twice)

### The three guarantees:

| Guarantee | Risk | How It Happens |
|---|---|---|
| **At-most-once** | Message can be **lost** | Consumer saves its position BEFORE processing. If it crashes during processing, the message is skipped. |
| **At-least-once** | Message can be **duplicated** | Consumer saves its position AFTER processing. If it crashes after processing but before saving position, the message is reprocessed. |
| **Exactly-once** | Neither lost nor duplicated | Requires either Kafka transactions (only works within Kafka) or at-least-once + idempotent consumer. |

### Our choice: At-least-once + idempotent consumer

```
1. Consumer reads event: "pay_abc AUTHORIZED"
2. Consumer processes it (calls PSP, updates DB)
3. Consumer saves its position (commits offset)

What if it crashes between step 2 and step 3?
    → Kafka redelivers the event (because position wasn't saved)
    → Consumer processes "pay_abc AUTHORIZED" again
    → Our idempotency layer catches it:
        - Redis SETNX says "already processed" → skip
        - Even if Redis missed it, DB unique constraint blocks the duplicate
    → Customer is NOT charged twice ✓
```

**Why not use Kafka's built-in exactly-once?**

Kafka's exactly-once (transactional API) only works within Kafka — from one topic to another. Our payment processing goes outside Kafka to a PSP and a bank. Kafka can't guarantee exactly-once for external calls. So we handle it at the application layer with idempotency keys.

> **The principle:** Accept redelivery over data loss — always. A duplicate we can detect is safer than a payment we lose.

---

## How It All Fits — Walking Through One Payment

Let's follow `pay_abc` (a customer paying a merchant £50) through the entire system:

```
Step 1: PRODUCE
─────────────────
Payment Service creates pay_abc
    → Writes to Kafka: { payment_id: "pay_abc", amount: 5000, currency: "GBP", 
                          status: "PENDING", merchant_id: "merch_42" }
    → Topic: payment-events
    → Partition key: "pay_abc" → hashes to Partition 3
    → acks=all → event is on 2+ brokers before acknowledged ✓

Step 2: CONSUME (all 4 groups read this independently)
──────────────────────────────────────────────────────

payment-processor (Instance B, assigned to Partition 3):
    → Reads "pay_abc: PENDING"
    → Calls PSP to authorize the £50 charge
    → PSP returns success
    → Writes new event: { payment_id: "pay_abc", status: "AUTHORIZED" }
    → Commits offset (saves position)

ledger-service (Instance X, assigned to Partition 3):
    → Reads "pay_abc: PENDING"
    → No action yet — payment not authorized
    → Commits offset
    (Later, reads "pay_abc: CAPTURED" → appends ledger entries:
        DEBIT  customer_account  £50.00
        CREDIT merchant_account  £50.00)

wallet-service (Instance P, assigned to Partition 3):
    → Reads "pay_abc: PENDING"
    → No action yet — payment not captured
    → Commits offset
    (Later, reads "pay_abc: CAPTURED" → updates merchant merch_42 balance: +£50.00)

webhook-dispatcher (Instance N, assigned to Partition 3):
    → Reads "pay_abc: PENDING"
    → Sends POST to merchant's webhook URL with payment status
    → Commits offset

Step 3: NEXT EVENTS
───────────────────
Payment Service writes "pay_abc: AUTHORIZED" → same partition (Partition 3)
    → All 4 consumer groups read it next
    → Same process repeats for CAPTURED, SETTLED, etc.
```

---

## What Happens When Things Break

| What Breaks | What Actually Happens | How We Recover |
|---|---|---|
| **A broker dies** | The partitions it led get new leaders from the followers. Takes a few seconds. | Automatic. Replication means no data loss. Producers and consumers reconnect to new leaders. |
| **A consumer crashes** | Its partitions get reassigned to the remaining consumers in the group. This is called a "rebalance." | Automatic. Events on those partitions are redelivered from the last committed offset. Idempotency handles duplicates. |
| **Consumers are too slow** | Events pile up in the partition. The gap between the latest event and where the consumer is reading grows — this is called "consumer lag." | We auto-scale consumers with KEDA (watches the lag metric). Alert if lag exceeds 10K messages. |
| **Can't write to Kafka** | Happens when 2 of 3 brokers are down (min.insync.replicas not met). Producer gets an error. | Payment Service returns 503 to the client. Client retries with their idempotency key — safe because we haven't processed anything yet. |
| **Bad message that can't be processed** | One malformed event causes the consumer to fail repeatedly. This is called a "poison pill." | Retry N times. If still failing, send to a Dead Letter Queue (DLQ). The bad message is isolated — healthy messages behind it continue flowing. |
| **Network split between brokers** | Some replicas fall out of the "in-sync" set. Kafka won't accept writes unless min.insync.replicas are available. | By design — we'd rather refuse a write than write to too few replicas and risk data loss. |

---

## Our Kafka Configuration

### Producer Config (Payment Service)

| Setting | Value | Why (Plain English) |
|---|---|---|
| `acks` | `all` | Don't tell me the write succeeded until ALL in-sync replicas have it. No data loss. |
| `enable.idempotence` | `true` | If the producer retries due to a network blip, Kafka detects the duplicate and doesn't store it twice. |
| `retries` | `MAX_VALUE` | Keep retrying on transient failures (network blips, broker switching). Don't give up. |
| `compression.type` | `lz4` | Compress messages before sending. Saves network bandwidth. lz4 is fastest. |
| `max.in.flight.requests.per.connection` | `5` | Send up to 5 batches at once without waiting. With idempotence enabled, Kafka still guarantees order. |

### Consumer Config (All consumer groups)

| Setting | Value | Why (Plain English) |
|---|---|---|
| `enable.auto.commit` | `false` | Do NOT let Kafka automatically save my position. I will save it manually AFTER I finish processing each message. |
| `auto.offset.reset` | `earliest` | If a brand new consumer group starts for the first time, start from the very beginning. Don't skip any events. |
| `max.poll.records` | `100-500` | How many messages to grab per read. Higher = more throughput but slower per-batch processing. |
| `session.timeout.ms` | `30000` | If a consumer doesn't send a heartbeat for 30 seconds, Kafka considers it dead and reassigns its partitions. |
| `max.poll.interval.ms` | `300000` | If a consumer doesn't read new messages for 5 minutes, Kafka kicks it from the group. Prevents stuck consumers from holding partitions. |

---

## Kafka vs Alternatives — Quick Comparison

| If You Need... | Use | Not Kafka Because... |
|---|---|---|
| Simple task queue, no replay | **SQS** | Fully managed, no partitions to tune, cheaper for fire-and-forget |
| Complex routing by headers/patterns | **RabbitMQ** | Exchange/binding routing is more flexible than topic-based |
| Sub-millisecond pub/sub | **Redis Pub/Sub** | Kafka has millisecond-level latency. But Redis has no persistence — if no one is listening, messages are lost |
| Log retention + replay + multiple consumers | **Kafka** | This is exactly what Kafka was built for |

> **Our choice:** Kafka for `payment-events` because we need replay, ordering, and multiple consumer groups. If the ledger-service goes down for an hour, we rewind the offset and reprocess everything it missed. SQS deletes messages after consumption — can't replay.

---

## Log Compaction vs Retention

Two ways to manage old data in Kafka:

**Time-based retention** — delete messages older than N days.
```
Day 1: [pay_abc: PENDING] [pay_abc: AUTHORIZED] [pay_abc: CAPTURED]
Day 31: (all deleted — past 30-day retention)
```

**Log compaction** — keep only the latest value per key.
```
Before compaction:
  [pay_abc: PENDING] [pay_def: PENDING] [pay_abc: AUTHORIZED] [pay_abc: CAPTURED] [pay_def: AUTHORIZED]

After compaction:
  [pay_abc: CAPTURED] [pay_def: AUTHORIZED]
  
  Only the latest status per payment survives.
```

| Our Topic | Strategy | Why |
|---|---|---|
| `payment-events` | Time-based, 30 days | Need full history for replay and reconciliation |
| `payment-status-changes` | Log compaction | Other services only care about the current status, not the history |

---

## Interview Questions & Answers

### "Explain the Basics"

**Q: What is Kafka, in simple terms?**
> "It's a distributed append-only log. Producers write events, consumers read them independently at their own pace. Unlike a database, it's optimized for streaming — high throughput, ordered within partitions, and consumers can replay by rewinding their position."

**Q: Why not just use a database for events?**
> "A database is pull-based — consumers poll. Kafka is push/pull with consumer groups, built for high throughput. It also gives us replay: if the ledger-service crashes, it rewinds its offset and reprocesses. With a database, I'd need to build all of that — polling, offset tracking, replay — from scratch."

**Q: What's a partition and why does it matter?**
> "A partition is an ordered sequence within a topic. Kafka only guarantees order within a partition. I partition by `payment_id` so all events for one payment — PENDING, AUTHORIZED, CAPTURED — are always in order. Different payments can be on different partitions and processed in parallel."

**Q: What's the difference between a topic and a partition?**
> "A topic is the logical name — like `payment-events`. A partition is a physical split within that topic. Our `payment-events` topic has 12 partitions. The topic is what producers write to and consumers subscribe to. Partitions are how Kafka distributes and orders the data."

---

### "Design Decisions"

**Q: Why Kafka over SQS for payment events?**
> "Replay capability. If the ledger-service goes down for an hour, I rewind the consumer offset and reprocess everything it missed. SQS deletes messages after consumption — can't replay. Also, Kafka supports multiple consumer groups reading the same events independently — payment-processor, ledger-service, wallet-service, and webhook-dispatcher all read from `payment-events` without interfering. With SQS, I'd need separate queues and fan-out logic."

**Q: How did you choose the partition key?**
> "I use `payment_id` because the payment state machine — PENDING → AUTHORIZED → CAPTURED → SETTLED — must be ordered. If CAPTURED arrives before PENDING, the system breaks. Since Kafka only guarantees order within a partition, putting all events for one payment on the same partition solves this. Distribution is naturally even because payment IDs are UUIDs."

**Q: How many partitions do you need?**
> "Target throughput divided by per-consumer throughput. If I need 10,000 events per second and each consumer handles 1,000, that's 10 partitions minimum. I add headroom for peak traffic and land at 12-24. Too few = bottleneck. Too many = slower rebalancing and more memory on brokers."

**Q: Why not RabbitMQ?**
> "RabbitMQ is better for complex routing — headers, patterns, exchange bindings. But it doesn't have Kafka's log retention and replay. For payment events where I need durability, reprocessing, and multiple independent consumers, Kafka wins."

---

### "What If" — Failure Scenarios

**Q: What if a broker dies?**
> "Leader election — a follower becomes the new leader in seconds. With replication factor 3 and min.insync.replicas 2, the data is already on other brokers. Producers and consumers reconnect automatically. No data loss, no manual intervention."

**Q: What if a consumer is too slow?**
> "Consumer lag increases — the gap between the latest event and where the consumer has read up to. We monitor this metric and auto-scale consumers with KEDA. Alert fires if lag exceeds 10K messages. The fix is more consumer instances — up to the number of partitions."

**Q: What if a message can't be processed (poison pill)?**
> "Retry it a fixed number of times. If it still fails, move it to a Dead Letter Queue. The bad message is isolated for manual investigation, and healthy messages behind it continue processing. Never let one bad event block the entire stream."

**Q: What if the ledger-service was down for an hour?**
> "That's the beauty of Kafka. The events are still in the topic — Kafka retains them for 30 days. When the ledger-service comes back, it resumes from its last committed offset and processes everything it missed. No data is lost. This is something SQS can't do."

**Q: What if Kafka itself can't accept writes?**
> "This happens when min.insync.replicas isn't met — meaning 2 of our 3 brokers are down. The producer gets an error. The Payment Service returns 503 to the client. The client retries with their idempotency key, which is safe because nothing was processed yet."

---

### "Go Deeper" — Advanced Questions

**Q: How do you achieve exactly-once processing?**
> "At-least-once from Kafka — commit offset after processing, so we never lose a message. If a message is redelivered, our two-layer idempotency catches the duplicate: Redis SETNX checks in sub-millisecond, DB unique constraint is the safety net. Effectively exactly-once without Kafka transactions. Kafka's built-in exactly-once only works within Kafka — our processing goes outside to PSPs and banks, so we handle it at the application layer."

**Q: What's the difference between log compaction and retention?**
> "Retention deletes messages older than N days — we use 30-day retention on `payment-events` for replay and reconciliation. Compaction keeps only the latest value per key — useful for a `payment-status-changes` topic where other services only care about the current state of each payment, not the full history."

**Q: How do you handle consumer rebalancing?**
> "When a consumer joins, leaves, or crashes, Kafka reassigns partitions across the remaining consumers in the group. During rebalancing, the group briefly pauses consumption. We minimize disruption by using sticky partition assignment (consumers keep their existing partitions where possible) and setting reasonable session timeouts to avoid unnecessary rebalances from slow consumers."

**Q: How do you scale consumers beyond the partition count?**
> "You can't — partitions are the upper bound on parallelism within a consumer group. If 12 partitions isn't enough, we increase the partition count. But you can't decrease it later without recreating the topic, so we start with enough headroom. If a single partition is a bottleneck, the issue is usually a hot key — one partition getting disproportionate traffic — which we solve by choosing a well-distributed partition key like UUID-based `payment_id`."

**Q: What happens to ordering if you increase partitions?**
> "New messages are distributed across the new partition count, but existing messages stay where they are. This means a payment that already had events on Partition 3 might get future events on a different partition. In practice, this is fine for us — active payments complete quickly, and new payments get correctly routed from the start. We'd increase partitions during low-traffic periods to minimize in-flight payments."

---

**Deep reference:** [13-kafka-cheat-sheet.md](13-kafka-cheat-sheet.md) — Quick-scan version of everything above
