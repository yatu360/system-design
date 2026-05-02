# Architectural Trade-Offs — Stage by Stage

> A principal engineer doesn't just pick technologies — they **argue for choices, acknowledge what they're giving up, and explain when they'd change their mind.** This document walks through every stage of the payment system design with the trade-offs you should be ready to discuss.

---

## Stage 1: Requirement Gathering

> Not a "design" trade-off per se, but the recruiter stressed this twice. The trade-off here is **time spent asking vs time spent building** in a 90-minute interview.

### Functional Requirements (What It Does)

| Requirement | Questions to Validate |
|-------------|----------------------|
| **Create payment** | One-time only or recurring/subscriptions? |
| **Refund** | Full and partial? Who initiates — merchant or customer? |
| **Payment status** | Polling or push notification (webhooks)? |
| **Dashboard** | Real-time or near-real-time? Who uses it — merchants, internal ops, both? |
| **Transaction history** | How far back? Searchable? Exportable? |
| **Multi-currency** | Single currency or cross-border with FX conversion? |

### Non-Functional Requirements (How Well It Does It)

| Requirement | Typical Target | Why This Number |
|-------------|---------------|-----------------|
| **Throughput** | ~1,000–10,000 TPS (depending on scale) | Black Friday / peak events can 10x steady state |
| **Latency** | p99 < 500ms for payment initiation | Customer is waiting at checkout — every second = drop-off |
| **Availability** | 99.99% (4 nines = ~52 min downtime/year) | Downtime = lost revenue for every merchant on the platform |
| **Durability** | Zero data loss for financial transactions | Regulatory and legal requirement — you cannot lose payment records |
| **Consistency** | Strong for payments, eventual acceptable for dashboard | Money must be correct NOW; dashboard can lag by seconds |

### The Trade-Off: Depth vs Breadth in Requirements

| Approach | Risk |
|----------|------|
| Spend 15 min on requirements | Less time to design, but system matches what's expected |
| Spend 3 min on requirements | More time to design, but you might build the wrong thing |

> **Principal call:** 8–10 minutes on requirements. Enough to validate scope, not so much you lose design time. The interviewer will redirect you if you're going too deep.

---

## Stage 2: API Design

### REST vs Async — When Each Applies

| Flow | Choice | Why | Trade-Off |
|------|--------|-----|-----------|
| **Create payment** | **Sync REST** (returns immediately with `PENDING`) | Customer is at checkout, needs confirmation the request was received | Client must poll or listen for final status |
| **Payment processing** | **Async** (background via Kafka) | PSP + bank calls take seconds, can't hold HTTP connection | Need webhook or polling for completion status |
| **Get payment status** | **Sync REST** | Simple read, sub-100ms from cache | Stale data if cache TTL too long |
| **Refund** | **Sync REST** (accepts) → **Async** (processes) | Same pattern as payment — accept fast, process in background | Same webhook/polling requirement |
| **Dashboard data** | **Sync REST** with pagination | Read-heavy, cacheable, standard CRUD | Large result sets need pagination + cursor strategy |
| **Payment notifications** | **Webhooks** (async push) | Merchant shouldn't poll — we push status changes to their endpoint | Need retry logic, signature verification, DLQ for failed deliveries |

### RESTful API Endpoints

```
POST   /v1/payments              → Create payment (idempotency key in header)
GET    /v1/payments/{id}         → Get payment status
POST   /v1/payments/{id}/refund  → Initiate refund
GET    /v1/payments?merchant={id}&from={date}&to={date}&cursor={token}
                                 → List transactions (paginated)

POST   /v1/webhooks              → Register webhook endpoint
DELETE /v1/webhooks/{id}         → Remove webhook

GET    /v1/dashboard/summary     → Aggregated metrics for dashboard
GET    /v1/dashboard/transactions → Paginated transaction list for dashboard
```

### API Design Trade-Offs

| Decision | Option A | Option B | Payment System Choice | Why |
|----------|----------|----------|----------------------|-----|
| **Versioning** | URL path (`/v1/`) | Header (`Accept-Version: 1`) | **URL path** | Simpler to route, debug, and cache. Header versioning is cleaner in theory but harder to operate |
| **Pagination** | Offset-based (`?page=3`) | Cursor-based (`?cursor=abc123`) | **Cursor-based** | Offset breaks when new transactions insert mid-page. Cursor is stable for append-heavy data |
| **Error format** | HTTP status only | Structured body with error code + message | **Structured body** | `400 Bad Request` isn't enough — client needs `{ "code": "DUPLICATE_IDEMPOTENCY_KEY", "message": "..." }` |
| **Sync accept → async process** | Fully sync (block until done) | Accept → process → notify | **Accept → process → notify** | Bank calls take 2-10s. Blocking the HTTP connection that long = timeouts, poor UX, wasted connections |

### Webhooks — The Missing Piece

```
Payment completes/fails
    │
    ▼
Payment Service ──► Webhook Dispatcher
                         │
                         ▼
                    POST to merchant's URL
                    {
                      "event": "payment.completed",
                      "payment_id": "abc-123",
                      "status": "SUCCESS",
                      "amount": 99.99,
                      "currency": "GBP",
                      "timestamp": "2026-05-02T10:30:00Z",
                      "signature": "HMAC-SHA256..."    ← merchant verifies authenticity
                    }
                         │
                    ┌────┴────┐
                    │         │
                 200 OK    Failure
                 (done)       │
                              ▼
                         Retry with exponential backoff
                         (max 5 attempts)
                              │
                         Still failing?
                              │
                              ▼
                         Dead Letter Queue
                         (manual investigation)
```

**Trade-off:** Webhooks vs polling

| | Webhooks (push) | Polling (pull) |
|---|---|---|
| **Latency** | Near real-time | Depends on poll interval |
| **Efficiency** | Only fires when state changes | Wastes requests when nothing changed |
| **Complexity** | Harder — retries, signatures, DLQ | Simpler — just GET on a timer |
| **Reliability** | Merchant's server must be up | Works even if merchant has intermittent downtime |
| **Principal answer** | **Support both.** Webhooks as primary, polling as fallback. Merchants with unreliable infra can poll. |

---

## Stage 3: Data Storage

### The Polyglot Persistence Trade-Off

| Decision | Option A | Option B | Choice | Why | What You Give Up |
|----------|----------|----------|--------|-----|-----------------|
| **Payment data** | PostgreSQL (SQL) | DynamoDB (NoSQL) | **PostgreSQL** | ACID transactions — debit + credit must be atomic | Harder to scale horizontally |
| **Caching layer** | Redis | Memcached | **Redis** | Data structures (sorted sets, SETNX), TTL, persistence options | Higher memory per key than Memcached |
| **Analytics** | Same PostgreSQL | Separate warehouse | **Separate warehouse** | Heavy aggregation kills transactional DB performance | Operational complexity of CDC pipeline |
| **Ledger storage** | Mutable rows | Append-only | **Append-only** | Tamper-evident audit trail, regulatory requirement | More storage, no in-place corrections |
| **Money type** | `FLOAT` | `DECIMAL(19,4)` | **DECIMAL** | Float has rounding errors (`0.1 + 0.2 ≠ 0.3`) | Slightly more storage, slower arithmetic |

### SQL vs NoSQL — The Real Trade-Off

> It's not about which is "better." It's about **what guarantees you need**.

| If You Need... | Use | Because |
|-----------------|-----|---------|
| Money can never be wrong | **SQL** | ACID transactions, strong consistency |
| 100K+ reads/sec with flexible schema | **NoSQL** | Horizontal scale, schema flexibility |
| Both (payment system) | **SQL primary + NoSQL cache** | Correctness from SQL, speed from NoSQL |

**When the interviewer challenges:** "What if you need to handle 100K TPS?"
> "At that scale I'd shard PostgreSQL by merchant_id using Citus, keeping ACID within each shard. Cross-shard transactions are rare in payments — a merchant's data is self-contained. If we genuinely hit a point where relational sharding isn't enough, I'd consider CockroachDB or TiDB — distributed SQL that preserves ACID across nodes."

### Dashboard Data — The Read Pattern Trade-Off

| Approach | Pros | Cons |
|----------|------|------|
| Query PostgreSQL directly | Fresh data, simple | Competes with payment writes, slow for aggregations |
| Read replica | Doesn't impact primary | Replication lag (seconds), still slow for heavy analytics |
| Materialized views | Pre-computed, fast reads | Refresh overhead, stale between refreshes |
| **CQRS — separate read model** | Optimized for dashboard queries, no impact on writes | Eventual consistency, more infrastructure |
| **CDC → Data Warehouse** | Best for heavy analytics, reconciliation, reporting | Highest latency (seconds to minutes), highest complexity |

> **Principal answer:** "For the real-time dashboard, I'd use a read replica with materialized views for common aggregations. For historical reporting and reconciliation, CDC into a data warehouse. Two different read patterns, two different solutions."

---

## Stage 4: Security

### Authentication & Authorization Trade-Offs

| Method | Pros | Cons | When to Use |
|--------|------|------|-------------|
| **API Keys** | Simple, stateless, easy to rotate | No user identity, easy to leak in logs | Service-to-service, merchant API access |
| **JWT (JSON Web Tokens)** | Stateless, contains claims, widely supported | Can't revoke until expiry (without blocklist), size grows with claims | Dashboard user sessions, short-lived tokens |
| **OAuth 2.0** | Industry standard, delegated access, scoped permissions | Complex flow, requires auth server | Third-party integrations, merchant dashboard |
| **mTLS** | Strongest — mutual certificate verification | Certificate management is painful, harder to debug | Internal service-to-service in payment core |

> **Principal approach for this system:**
> - **Merchant API:** OAuth 2.0 with API keys for identification + JWT for authorization
> - **Dashboard:** OAuth 2.0 with short-lived JWTs + refresh tokens
> - **Internal services:** mTLS between payment-critical services
> - **PSP communication:** TLS + signed payloads (HMAC)
> - **Card data:** PSP hosted page + tokenization — merchant never handles card numbers (**PCI DSS** scope reduction)
> - **Customer authentication:** 3DS2 via PSP — mandatory under **PSD2/SCA** in the UK. Two-factor (biometric/OTP). Shifts fraud liability to issuer

### Firewall & Network Security

```
Internet ──► WAF (Web Application Firewall) ──► Load Balancer ──► API Gateway ──► Services
                   │                                                    │
                   ├── DDoS protection                                  ├── Rate limiting
                   ├── IP allowlisting                                  ├── Auth/authz
                   ├── SQL injection filtering                          ├── Request validation
                   └── Geo-blocking                                     └── Throttling
                   
Internal network:
Services ◄──► Services    (mTLS, private subnet, no public internet)
Services ──► Database     (private subnet, security groups, no public access)
```

**Trade-off:** WAF + strict rules vs. open access with monitoring

| | Strict firewall rules | Permissive + monitoring |
|---|---|---|
| **Security** | Higher — blocks threats at the edge | Lower — relies on detection after the fact |
| **False positives** | Risk of blocking legitimate traffic | No blocking risk |
| **Complexity** | Rule management, tuning, maintenance | Simpler ingress, complex detection |
| **Payment systems** | **This one.** Financial data demands defense in depth | Unacceptable for PCI DSS scope |

### Rate Limiting Strategies

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Fixed window** | N requests per minute, resets at minute boundary | Simple | Burst at window edges (2x allowed at boundary) |
| **Sliding window** | N requests in any rolling 60s period | Smooth, no boundary bursts | More memory (track timestamps) |
| **Token bucket** | Bucket fills at rate R, each request takes a token | Allows controlled bursts | Slightly more complex |
| **Leaky bucket** | Requests processed at fixed rate, excess queued | Smoothest output rate | Queued requests add latency |

> **Payment system choice:** **Token bucket** per API key. Allows small bursts (legitimate checkout spikes) while enforcing overall rate. Return `429 Too Many Requests` with `Retry-After` header.

---

## Stage 5: Observability

### The Three Pillars

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   METRICS    │     │    LOGS      │     │   TRACES     │
│              │     │              │     │              │
│ "What is     │     │ "What        │     │ "What path   │
│  happening?" │     │  happened?"  │     │  did it take?"|
│              │     │              │     │              │
│ Prometheus   │     │ ELK Stack    │     │ OpenTelemetry│
│ Datadog      │     │ CloudWatch   │     │ Jaeger       │
│ Grafana      │     │ Splunk       │     │ Zipkin       │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Metrics — What to Track

| Category | Metric | Why | Alert Threshold |
|----------|--------|-----|-----------------|
| **Technical** | API failure rate (5xx) | Service health | > 1% of requests |
| **Technical** | p50 / p95 / p99 latency | Performance degradation | p99 > 500ms |
| **Technical** | CPU usage | Capacity planning | > 80% sustained |
| **Technical** | Memory usage | OOM risk | > 85% |
| **Technical** | Kafka consumer lag | Processing falling behind | > 10K messages |
| **Technical** | DB connection pool utilization | Connection exhaustion | > 80% |
| **Business** | Payment success rate | Revenue impact | < 95% |
| **Business** | Conversion rate (initiated → completed) | Checkout health | Drop > 10% from baseline |
| **Business** | Per-merchant failure rate | Merchant-specific issues | > 5% for any single merchant |
| **Business** | Transaction volume per minute | Traffic anomaly detection | > 3 std dev from baseline |
| **Business** | Refund rate | Fraud or quality signal | > 2% of transactions |

### Distributed Tracing — Why It's Critical for Payments

```
Payment Request ─── trace_id: abc-123 ───────────────────────────►
    │
    ├── span: api-gateway        (2ms)    auth, rate limit, validation
    ├── span: payment-service    (5ms)    create order, persist to DB
    ├── span: kafka-produce      (3ms)    publish payment event
    ├── span: payment-processor  (1200ms) call PSP, wait for bank response
    ├── span: wallet-service     (8ms)    update merchant balance
    ├── span: ledger-service     (4ms)    append ledger entry
    └── span: webhook-dispatcher (15ms)   notify merchant
                                 ─────
                                 ~1237ms total
```

> Without tracing, when a payment takes 5 seconds, you have **no idea which hop is slow** — is it the PSP? The DB? Kafka? With tracing, you see the exact bottleneck.

**Tool choice trade-off:**

| Tool | Pros | Cons |
|------|------|------|
| **OpenTelemetry + Jaeger** | Open standard, vendor-neutral, free | Self-host Jaeger, operational burden |
| **Datadog** | All-in-one (metrics + logs + traces), managed | Expensive at scale, vendor lock-in |
| **AWS X-Ray** | Tight AWS integration, cheap | AWS-only, less flexible |

> **Principal answer:** "OpenTelemetry as the instrumentation standard — it's vendor-neutral so we can switch backends. Jaeger for traces in dev/staging, Datadog in production for the unified view. Pragmatic: don't fight vendor lock-in where the ops benefit is clear."

### Alerting — Thresholds, Paging, Escalation

| Severity | Condition | Action | Responder |
|----------|-----------|--------|-----------|
| **P1 — Critical** | Payment success rate < 90%, or zero transactions for 5 min | Page on-call immediately (PagerDuty) | On-call engineer |
| **P2 — High** | p99 latency > 2s, or Kafka consumer lag > 50K | Page on-call within 15 min | On-call engineer |
| **P3 — Medium** | Single merchant failure rate > 10%, CPU > 85% | Slack alert, next business day | Team |
| **P4 — Low** | Refund rate trending up, dashboard slow | Ticket created | Sprint backlog |

**Trade-off:** Aggressive alerting vs alert fatigue

| | Too many alerts | Too few alerts |
|---|---|---|
| Risk | Team ignores pages, alert fatigue | Silent failures, customers discover issues |
| **Balance** | Alert on **symptoms** (success rate, latency), not causes (CPU, memory). Investigate causes after paging. |

### Dashboards — What to Visualise

> Grafana connected to **Prometheus** (metrics) and **Loki** (logs). Or Datadog as all-in-one.

| Dashboard | Key Panels | Refresh |
|-----------|-----------|---------|
| **Payments — Live** | Success rate (%), p50/p95/p99 latency, TPS, active payment count | 10s |
| **Payments — Per Merchant** | Failure rate by merchant, chargeback rate (flag >1%), volume | 1 min |
| **Infrastructure** | CPU, memory, Kafka consumer lag, DB connection pool, Redis hit rate | 30s |
| **Business** | Revenue (daily/weekly), conversion funnel, refund rate, top error codes | 5 min |

> Keep it simple — **four dashboards** covering live ops, merchant health, infra, and business. Grafana pulls from Prometheus via PromQL queries.

---

## Stage 6: Event-Driven Architecture

### Message Queue Trade-Offs

| Queue | Ordering | Delivery | Throughput | Complexity | Best For |
|-------|----------|----------|------------|------------|----------|
| **Kafka** | Per-partition ordering | At-least-once (exactly-once with idempotency) | Millions/sec | High — partitions, consumer groups, offset management | **Payment events** — need ordering, durability, replay |
| **SQS** | No strict ordering (FIFO variant available) | At-least-once | 3,000/sec (FIFO) | Low — fully managed, no infra | Webhook dispatch, notifications |
| **RabbitMQ** | Per-queue ordering | At-least-once | 10K-50K/sec | Medium — clustering, but simpler than Kafka | Task queues, routing-heavy workloads |

**When the interviewer asks "Why not just SQS?"**
> "For payment events, I need **log compaction** and **replay capability**. If the ledger service goes down for an hour, I need to replay all missed events from Kafka's log. SQS deletes messages after consumption — I can't replay. For webhook dispatch, SQS is perfect — simpler, managed, and I don't need replay for notifications."

### Event Sourcing — Trade-Off

| | Traditional CRUD | Event Sourcing |
|---|---|---|
| **State** | Store current state, overwrite on update | Store every event, derive current state |
| **Audit** | Need separate audit log | Audit trail is built-in |
| **Debugging** | "Why is the balance wrong?" — hard to trace | Replay events to see exactly how state evolved |
| **Complexity** | Simple — CRUD is well-understood | Complex — event versioning, snapshotting, projections |
| **Payment system** | Ledger is already append-only → **naturally event-sourced** | Kafka events + append-only ledger = event sourcing without the full framework overhead |

> **Principal nuance:** "I wouldn't introduce a full event sourcing framework. But our design is already event-sourced in spirit — Kafka persists every payment event, the ledger is append-only, and we can rebuild state from the event log. I get 80% of the benefit without the CQRS/ES framework complexity."

---

## Stage 7: Fault Tolerance & Resiliency

### Circuit Breaker Pattern

```
         ┌───────────────────────────────────┐
         │         CIRCUIT BREAKER           │
         │                                   │
         │  ┌────────┐  failures  ┌───────┐  │
         │  │ CLOSED  │─────────►│  OPEN  │  │
         │  │(normal) │ exceed    │(reject │  │
         │  │         │ threshold │ all)   │  │
         │  └────┬───┘           └───┬────┘  │
         │       ▲                    │       │
         │       │success         timeout     │
         │       │                    │       │
         │  ┌────┴─────────────┐     │       │
         │  │    HALF-OPEN     │◄────┘       │
         │  │(allow one probe) │              │
         │  └──────────────────┘              │
         └───────────────────────────────────┘
```

**Where to apply in a payment system:**

| Service Call | Why Circuit Breaker | Threshold |
|-------------|---------------------|-----------|
| **Payment Service → PSP** | PSP outage shouldn't cascade to all services | 5 failures in 30s → open for 60s |
| **Payment Service → Wallet** | Wallet DB lock contention shouldn't block new payments | 10 failures in 60s → open for 30s |
| **Webhook Dispatcher → Merchant** | One merchant's broken endpoint shouldn't delay others | Per-merchant circuit: 3 failures → open |

**Trade-off:** Circuit breaker vs unlimited retries

| | Circuit Breaker | Unlimited Retries |
|---|---|---|
| **Cascade protection** | Stops hammering a failing service | Keeps adding load to an already-failing service |
| **Recovery time** | Gives downstream time to recover | No breathing room |
| **False positives** | May reject requests when service recovers mid-window | No false rejections |
| **Payments** | **Prefer circuit breaker + DLQ.** Open circuit → queue payments → process when circuit closes |

### The Full Resiliency Stack — Layered

```
Request arrives
    │
    ▼
Rate Limiter ── too many? → 429 Too Many Requests
    │
    ▼
Circuit Breaker ── open? → return fallback or queue to DLQ
    │
    ▼
Timeout ── took too long? → cancel + retry with same idempotency key
    │
    ▼
Retry (exponential backoff + jitter)
    │
    ├── Success → continue
    │
    └── Max retries exceeded → Dead Letter Queue
                                     │
                                     ▼
                               Manual investigation / alert
```

> **Principal insight:** These patterns are **layered, not alternatives**. Rate limiter first (protect yourself), circuit breaker second (protect downstream), timeout + retry third (handle transient failures), DLQ last (don't lose data).

---

## Stage 8: Infrastructure & Scalability

### Hosting — Cloud Provider Choice

| | **AWS** | **GCP** | **Azure** |
|---|---|---|---|
| **Market share** | Largest — most mature, widest service catalogue | Strong in data/ML, growing in general compute | Strong enterprise, Microsoft ecosystem |
| **Payments-relevant services** | SQS, MSK (managed Kafka), RDS, ElastiCache, Lambda, ALB, Route 53, WAF | Pub/Sub, Cloud SQL, Memorystore, Cloud Run, Cloud Armor | Service Bus, Azure SQL, Azure Cache, Functions, Front Door |
| **Managed Kafka** | MSK (fully managed) | No native — use Confluent Cloud | Event Hubs (Kafka-compatible API) |
| **UK region** | `eu-west-2` (London) | `europe-west2` (London) | `UK South` (London) |
| **PCI DSS certified** | Yes | Yes | Yes |
| **Ecosystem** | Most third-party tooling, largest talent pool | Strongest BigQuery/analytics | Best if existing Microsoft/Azure AD estate |

> **Principal answer:** "I'd default to **AWS** — broadest service catalogue, London region for UK data residency, mature managed services (MSK for Kafka, RDS for PostgreSQL, ElastiCache for Redis). If the company already runs on GCP or Azure, I'd stay — the migration cost outweighs the marginal AWS advantage. The cloud provider choice matters less than using **managed services** consistently."

### Managed vs Self-Hosted — The Real Decision

| Component | Managed | Self-Hosted | Choice | Why |
|-----------|---------|-------------|--------|-----|
| **PostgreSQL** | RDS / Cloud SQL | PostgreSQL on EC2/GKE | **Managed** | Automated backups, failover, patching. Team focuses on schema, not OS patches |
| **Kafka** | MSK / Confluent Cloud | Kafka on K8s (Strimzi) | **Managed** | Kafka operations are notoriously painful — broker rebalancing, partition management. Let the cloud handle it |
| **Redis** | ElastiCache / Memorystore | Redis on K8s | **Managed** | Failover, patching, cluster scaling. Self-hosted Redis adds on-call burden for minimal control benefit |
| **Kubernetes** | EKS / GKE / AKS | Self-managed K8s | **Managed** | Control plane management is undifferentiated work. EKS handles upgrades, node management |
| **Monitoring** | Datadog / CloudWatch | Prometheus + Grafana on K8s | **Depends** | Self-hosted Prometheus is viable and avoids Datadog cost. But Datadog's unified view (metrics + logs + traces) saves engineering time |

> **Trade-off:** Managed services cost more per unit but **reduce operational burden**. For a payment system, the cost of a self-hosted Kafka broker going down at 3 AM is far higher than the MSK bill. **Default to managed, self-host only when you need control the managed service doesn't give you** (custom Kafka configs, specific PostgreSQL extensions).

### Containerization — Trade-Off

| | Containers (K8s) | VMs / Bare Metal | Serverless (Lambda) |
|---|---|---|---|
| **Scaling speed** | Seconds (pod autoscaler) | Minutes | Milliseconds (cold start aside) |
| **Cost** | Pay for cluster overhead | Pay for idle capacity | Pay per invocation |
| **Control** | High — custom networking, sidecars | Highest | Low — vendor constraints |
| **Complexity** | High — K8s learning curve, YAML | Medium | Low |
| **Payment system** | **Best fit.** Need consistent latency, custom networking, sidecar for mTLS | Viable but slower to scale | Cold starts unacceptable for payments |

> **Principal reasoning:** "Kubernetes for the payment core — we need predictable latency (no cold starts), sidecar injection for mTLS, and horizontal pod autoscaling. For the webhook dispatcher, I'd consider Lambda — spiky workload, stateless, and occasional cold starts are acceptable for async notifications."

### Auto-Scaling Strategy

| Component | Scaling Metric | Scale Strategy |
|-----------|---------------|----------------|
| **API Gateway** | Requests per second | Horizontal Pod Autoscaler (HPA) on RPS |
| **Payment Service** | CPU + request queue depth | HPA on CPU (target 70%) |
| **Payment Processor** | Kafka consumer lag | KEDA — scale consumers based on lag |
| **Webhook Dispatcher** | Queue depth (SQS) | Lambda concurrency / KEDA on queue length |
| **Database** | Connection count + CPU | Read replicas (manual or auto for cloud-managed) |
| **Redis** | Memory usage + connections | Cluster mode — add shards |

### Multi-Regional Deployment

```
                    ┌─────────────────────┐
                    │   Global DNS / CDN  │
                    │  (Route 53 / CF)    │
                    └──────────┬──────────┘
                               │ latency-based routing
                    ┌──────────┴──────────┐
                    │                     │
              ┌─────▼─────┐         ┌─────▼─────┐
              │  Region A  │         │  Region B  │
              │  (EU-West) │         │ (US-East)  │
              │            │         │            │
              │ ┌────────┐ │         │ ┌────────┐ │
              │ │Services│ │         │ │Services│ │
              │ └────────┘ │         │ └────────┘ │
              │ ┌────────┐ │         │ ┌────────┐ │
              │ │  DB     │◄────────►│ │  DB     │ │
              │ │(primary)│ │  async  │ │(replica)│ │
              │ └────────┘ │  replic. │ └────────┘ │
              └────────────┘         └────────────┘
```

**Trade-offs:**

| Decision | Option A | Option B | Trade-Off |
|----------|----------|----------|-----------|
| **Active-Active** vs **Active-Passive** | Both regions serve traffic | One region is standby | Active-active = lower latency globally but write conflicts. Active-passive = simpler but failover takes minutes |
| **DB replication** | Synchronous | Asynchronous | Sync = zero data loss but higher write latency. Async = fast writes but risk data loss on failover |
| **Data residency** | All data in one region | Data stays in user's region | One region = simpler. Per-region = **UK GDPR** compliance — UK customer data should reside in UK/EEA data centres |

> **Principal call:** "Active-passive for payments. Write conflicts in financial data are unacceptable — I'd rather have one primary region (UK, for **FCA regulatory proximity and UK GDPR data residency**) with async replicas for reads and failover. Active-active for the sports API is fine — stale scores for 2 seconds are acceptable, stale balances are not."

### Deployment Strategies

| Strategy | How | Risk | Rollback Speed | Best For |
|----------|-----|------|----------------|----------|
| **Blue-Green** | Two identical envs, switch traffic at once | All-or-nothing — if green is bad, everyone is affected until switch back | Fast — switch DNS/LB back to blue | Database migrations, major releases |
| **Canary** | Route 5% → new version, watch metrics, gradually increase | Low — only canary % affected | Fast — route canary back | Payment service changes — **catch issues before full rollout** |
| **Rolling** | Replace pods one at a time | Medium — mixed versions during rollout | Slow — must roll back one by one | Stateless services, minor updates |

> **Payment system:** "Canary deployments for the payment service. I want to see real traffic hitting the new version while monitoring payment success rate and latency. If the canary's success rate drops by even 1%, auto-rollback. For database migrations, blue-green with a backward-compatible schema change — the old code must work with the new schema during the transition."

### Load Balancing

| Layer | Tool | Algorithm | Why |
|-------|------|-----------|-----|
| **DNS level** | Route 53 / CloudFlare | Latency-based routing | Route users to nearest region |
| **L7 (Application)** | ALB / Nginx / Envoy | Least connections | Distribute evenly across payment service pods — avoid sending to a pod already handling slow bank calls |
| **gRPC / internal** | Envoy (service mesh) | Round-robin with health checks | Internal service-to-service, integrated with K8s |

**Trade-off:** L4 vs L7 load balancing

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| **Speed** | Faster — no payload inspection | Slightly slower — reads HTTP headers |
| **Routing** | IP + port only | URL path, headers, cookies |
| **TLS termination** | At backend | At load balancer |
| **Payment system** | Internal, high-throughput paths | **API Gateway layer** — need header inspection for routing, auth, rate limiting |

---

## The Trade-Off Cheat Sheet (One Table)

| Stage | Key Decision | Choice | Trade-Off Accepted | Would Reconsider If... |
|-------|-------------|--------|-------------------|----------------------|
| **API** | Sync accept → async process | Accept fast, process in background | Client needs webhook/polling for result | Latency budget allows blocking (rare) |
| **API** | Webhooks + polling | Support both | More implementation work | Merchant reliability is guaranteed |
| **Data** | PostgreSQL over DynamoDB | ACID, strong consistency | Harder horizontal scaling | TPS exceeds 100K sustained |
| **Data** | Redis cache layer | Sub-ms idempotency checks | Extra infra, cache invalidation | Redis operational cost unjustified at low scale |
| **Data** | Separate data warehouse | No analytics on prod DB | CDC pipeline complexity | Query volume is low enough to use read replica |
| **Security** | mTLS internal + OAuth external | Defense in depth | Certificate management overhead | Team size too small to manage cert rotation |
| **Observability** | OpenTelemetry + Datadog | Vendor-neutral instrumentation | Datadog cost at scale | Budget constrained → self-host Jaeger |
| **Events** | Kafka for payments, SQS for webhooks | Right tool per job | Two queue systems to manage | Team prefers single system → Kafka for both |
| **Resiliency** | Circuit breaker + DLQ | Protect downstream + no data loss | Delayed processing during outages | Real-time requirement can't tolerate delays |
| **Infra** | Kubernetes | Predictable latency, sidecar support | K8s operational complexity | Small team → managed ECS or even VMs |
| **Infra** | Canary deployments | Low-risk rollouts | Slower full rollout | Emergency hotfix → blue-green for speed |
| **Infra** | Active-passive multi-region | No write conflicts for payments | Higher latency for distant users | Sports API → active-active is fine |

---

**Next:** [04-interview-cheat-sheet.md](./04-interview-cheat-sheet.md) — Quick reference for the interview
