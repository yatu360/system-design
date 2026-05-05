# Kafka Cheat Sheet — Quick Scan Before the Interview

> Every answer uses our payment system. Read this in 5 minutes.

---

## How Kafka Works in Our System (One Picture)

```
Payment Service (PRODUCER)
    │
    │ writes: { payment_id: "pay_abc", status: "AUTHORIZED", amount: 5000 }
    │ partition key: "pay_abc"
    │ acks: all (waits for 2+ brokers to confirm)
    │
    ▼
┌────────────────────────────────────────────────────────┐
│  Topic: payment-events  (12 partitions, 3 replicas)    │
│                                                        │
│  Partition 0: [pay_abc: PENDING] [pay_abc: AUTHORIZED]  │ ← all pay_abc events here
│  Partition 1: [pay_def: PENDING]                       │ ← all pay_def events here
│  Partition 2: [pay_ghi: PENDING] [pay_ghi: AUTHORIZED] │
│  ...                                                   │
│  Partition 11: [pay_xyz: PENDING]                      │
│                                                        │
│  Each partition = ordered. Across partitions = no order │
└───────┬──────────────┬──────────────┬─────────────┬────┘
        │              │              │             │
        ▼              ▼              ▼             ▼
  ┌───────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────┐
  │ payment-  │  │ ledger-   │  │ wallet-  │  │ webhook-     │
  │ processor │  │ service   │  │ service  │  │ dispatcher   │
  │           │  │           │  │          │  │              │
  │ calls PSP │  │ appends   │  │ updates  │  │ notifies     │
  │ to charge │  │ debit +   │  │ merchant │  │ merchant via │
  │ the card  │  │ credit    │  │ balance  │  │ webhook POST │
  └───────────┘  └───────────┘  └──────────┘  └──────────────┘
  
  Each group reads ALL events independently.
  They don't interfere with each other.
```

---

## 10 Key Concepts (One Line Each)

| # | Concept | Plain English (Using Our System) |
|---|---|---|
| 1 | **Topic** | A named stream of events. `payment-events` is where all payment lifecycle events go. |
| 2 | **Partition** | A topic split into ordered chunks. `payment-events` has 12. Each partition is an independent ordered log. |
| 3 | **Partition key** | Decides which partition a message goes to. We use `payment_id` — so all events for pay_abc always land in the same partition. |
| 4 | **Ordering** | Guaranteed within a partition ONLY. Same `payment_id` = same partition = events arrive in order. |
| 5 | **Consumer group** | A team of consumers that share the work. `ledger-service` is one group. Each group reads every event independently. |
| 6 | **Offset** | A message's position number within a partition (0, 1, 2...). Each consumer tracks where it's up to. |
| 7 | **Replication** | Every partition is copied to 3 brokers. If one dies, no data is lost. |
| 8 | **acks=all** | Producer waits until 2+ brokers confirm they stored the event. Payment events can't be lost. |
| 9 | **Commit after processing** | Consumer saves its position AFTER processing, not before. If it crashes, the message is redelivered — idempotency handles the duplicate. |
| 10 | **Replay** | Rewind a consumer's offset to reprocess past events. If ledger-service was down, it catches up. SQS can't do this. |

---

## Our Payment System Kafka Setup

| Setting | Value | Why |
|---|---|---|
| **Topics** | `payment-events` + `payment-events-dlq` | One main topic, one DLQ for failed messages |
| **Partition key** | `payment_id` | All events for one payment stay in order |
| **Partitions per topic** | 12 | Enough parallelism for peak, room to grow |
| **Replication factor** | 3 | Survive loss of 2 brokers |
| **min.insync.replicas** | 2 | Write only succeeds if 2 of 3 copies confirm |
| **acks** | all | No payment event is ever lost |
| **Retention** | 30 days (`payment-events`), 90 days (DLQ) | 30 days for replay and reconciliation; DLQ kept longer for investigation |
| **Offset commit** | Manual, after processing | Accept redelivery over data loss |
| **Consumer groups** | payment-processor, ledger-service, wallet-service, webhook-dispatcher | Each reads all events independently, filters by event_type |

---

## Interview Questions & Answers

### The Basics

**"What is Kafka?"**
> "A distributed append-only log. Producers write events, consumers read them independently. It gives us ordering within partitions, replay by rewinding offsets, and multiple consumer groups reading the same stream without interfering."

**"Why Kafka and not a database for events?"**
> "A database is pull-based and not designed for streaming. Kafka gives us high-throughput writes, consumer groups for parallel processing, offset-based replay, and decoupled services — I don't need to build polling, position tracking, or fan-out from scratch."

**"What's a partition?"**
> "An ordered sequence within a topic. Our `payment-events` topic has 12 partitions. Each is an independent log. Kafka only guarantees order within a partition — that's why the partition key matters."

---

### Design Decisions

**"Why Kafka over SQS?"**
> "Replay. If ledger-service goes down, I rewind the offset and reprocess. SQS deletes after consumption — can't replay. Also, four consumer groups read `payment-events` independently. With SQS I'd need separate queues and fan-out."

**"How do you guarantee ordering?"**
> "Partition by `payment_id`. All events for one payment go to the same partition. Kafka guarantees order within a partition. So PENDING → AUTHORIZED → CAPTURED always arrives in that order."

**"How many partitions and why?"**
> "Target throughput divided by per-consumer throughput, plus headroom for peak. 12 partitions gives us enough parallelism for 12 parallel consumers per group. Not so many that rebalancing is painful."

**"How did you choose the partition key?"**
> "`payment_id` because the state machine — PENDING, AUTHORIZED, CAPTURED, SETTLED — must be strictly ordered per payment. UUIDs distribute evenly so no hot partitions. If we needed merchant-level ordering, we'd use `merchant_id`."

**"Why not RabbitMQ?"**
> "RabbitMQ is better for complex routing but lacks log retention and replay. For payment events where we need durability and reprocessing, Kafka wins."

---

### Failure Scenarios

**"What if a broker dies?"**
> "Leader election in seconds. Replication factor 3 means the data is already on other brokers. Producers and consumers reconnect to the new leader. No data loss, no manual intervention."

**"What if a consumer is slow?"**
> "Consumer lag increases. We auto-scale consumers with KEDA based on the lag metric. Alert if lag exceeds 10K messages. Max parallelism is capped at the partition count."

**"What if a message can't be processed?"**
> "Retry N times. If it still fails, move it to a Dead Letter Queue. The poison pill is isolated for investigation. Healthy messages continue flowing."

**"What if Kafka can't accept writes?"**
> "Happens when 2 of 3 brokers are down — min.insync.replicas not met. Payment Service returns 503. Client retries with their idempotency key — safe because nothing was processed yet."

**"What if the ledger-service was down for an hour?"**
> "No data lost. Kafka retained all events for 30 days. Ledger-service resumes from its last committed offset and processes everything it missed. This is exactly why we chose Kafka over SQS."

---

### Go Deeper

**"How do you get exactly-once?"**
> "At-least-once from Kafka — commit offset after processing, never before. If a message is redelivered, our two-layer idempotency (Redis SETNX + DB unique constraint) catches the duplicate. Effectively exactly-once without Kafka transactions — which only work within Kafka, not for external PSP calls."

**"How do you prevent data loss?"**
> "Replication factor 3, min.insync.replicas 2, acks=all. The event is confirmed on at least 2 brokers before the producer gets an acknowledgement. One broker can die immediately — the event survives."

**"What's log compaction vs retention?"**
> "Retention deletes messages after N days — we use 30-day retention on `payment-events` for replay. Compaction keeps only the latest value per key — useful if you had a topic where services only need the current state of each payment rather than the full history. We don't use compaction in our design — we use retention-based cleanup."

**"What if you need more parallelism than partitions allow?"**
> "Increase the partition count. Can't decrease it without recreating the topic, so we start with headroom. If a single partition is hot, it's usually a partition key issue — `payment_id` distributes evenly because UUIDs hash uniformly."

**"How does consumer rebalancing work?"**
> "When a consumer joins, leaves, or crashes, Kafka redistributes partitions across remaining consumers. Brief pause during rebalance. We use sticky assignment so consumers keep existing partitions where possible, and reasonable session timeouts to avoid unnecessary rebalances."

---

## When Things Break — Quick Reference

| Failure | What Happens | Recovery |
|---|---|---|
| **Broker dies** | Leader election for affected partitions (seconds) | Replicas take over automatically. No data loss. |
| **Consumer crashes** | Its partitions get reassigned to remaining consumers | Redelivery from last committed offset + idempotency handles duplicates |
| **Consumer too slow** | Events pile up, lag increases | Auto-scale consumers with KEDA. Alert at 10K+ lag. |
| **Can't write** (2 of 3 brokers down) | Producer gets error | Payment Service returns 503. Client retries with idempotency key. |
| **Poison pill** (bad message) | Consumer fails repeatedly on one message | Retry N times → Dead Letter Queue → alert → manual investigation |
| **Network partition** | Some replicas fall out of sync | acks=all + min.insync.replicas prevents writes to under-replicated partitions |

---

## Config Quick Reference

### Producer (Payment Service)

| Config | Value | Why |
|---|---|---|
| `acks` | `all` | Wait for all in-sync replicas — no data loss |
| `enable.idempotence` | `true` | Producer-side retries don't create duplicate events |
| `retries` | `MAX_VALUE` | Keep retrying transient failures — don't give up |
| `compression.type` | `lz4` | Compress before sending. Fast, saves bandwidth. |

### Consumer (All groups)

| Config | Value | Why |
|---|---|---|
| `enable.auto.commit` | `false` | Save position manually after processing — never lose events |
| `auto.offset.reset` | `earliest` | New consumer starts from the beginning — don't miss anything |
| `max.poll.records` | `100-500` | Messages per read batch. Balance throughput vs latency. |
| `session.timeout.ms` | `30000` | 30s without heartbeat = consumer considered dead |

---

**Full explanations:** [12-kafka-fundamentals.md](12-kafka-fundamentals.md)
