# Recruiter Briefing — System Design Interview

> Recorded voice memo from recruiter. Interview is **Wednesday next week**. 
> Values interview prep is in [06-values-interview.md](06-values-interview.md).

---

## Part 1: System Design Interview

### Format
- **90 minutes**, whiteboard exercise
- Topic: Designing a system for a **Payment API + Dashboard**
- You'll receive **written instructions** — take ~5 minutes to read carefully before starting

---

### Requirement Gathering (Recruiter Stressed This Twice)

> The recruiter called this out in **both** briefings. This is clearly a make-or-break signal. Don't skip it, don't rush it.

**What to ask / validate before drawing anything:**

| Category | Questions to Ask |
|----------|-----------------|
| **Functional** | What operations? (create payment, refund, status check, dashboard views?) Who are the users? (merchants, customers, internal ops?) What does the dashboard show? (real-time transactions, historical reports, analytics?) |
| **Non-Functional** | What **load** are we designing for? (TPS, concurrent users, peak vs steady) What **response time** targets? (< 200ms for API? < 1s for dashboard?) |
| **Sync vs Async** | Which flows need **real-time** responses? (payment initiation) Which can be **asynchronous**? (notifications, reconciliation, reports) |
| **Scale & Region** | Single region or **multi-regional**? Where are the users? What's the growth trajectory? |
| **Hosting** | Cloud provider preference? (AWS / GCP / Azure) On-prem constraints? Managed services or self-hosted? |
| **Compliance** | PCI DSS scope? UK GDPR requirements? Data residency laws for multi-region? |

**How to do it:**
- Frame each question as **"I'm assuming X — is that correct?"** — validates without sounding lost
- It's a **two-way conversation**, not a Q&A — let the interviewer steer you
- **Think out loud** as you gather — "I'm asking about load because it affects whether we need async processing and how we partition data"

---

### How to Approach It (Process)

| Phase | What To Do | Why It Matters |
|-------|-----------|----------------|
| **1. Read Instructions** | Take 5 min to carefully read the brief | Don't rush — understand what's expected |
| **2. Requirement Gathering** | Ask questions to **validate your assumptions** | Top signal: candidates who engage interviewers. Skipping this = bad signal |
| **3. Think Out Loud** | Narrate your reasoning as you design | Lets interviewers follow your logic, challenge you, and help guide you |
| **4. Justify Every Decision** | Explain the **why** behind every architectural choice | **This is what separates senior/principal from mid-level** |
| **5. Be Open to Challenge** | When challenged, defend or concede gracefully | "Yeah, actually this could be better because..." shows maturity |

> **CRITICAL:** The recruiter explicitly warned — many candidates just start designing without engaging interviewers. This is a **bad signal**. Make it feel like a **conversation**, not a presentation.

---

### Topics to Cover (Checklist)

#### 1. Requirement Gathering
- [ ] Clarify **functional requirements** (what the system does)
- [ ] Clarify **non-functional requirements** (how well it does it)
- [ ] Establish **load expectations** (requests/sec, concurrent users)
- [ ] Establish **response time** targets (latency SLAs)
- [ ] Validate assumptions before designing

#### 2. API Design
- [ ] **RESTful API** design (endpoints, methods, payloads)
- [ ] **Asynchronous flows** (when and why to use async over sync)
- [ ] API versioning, pagination, error responses

#### 3. Data Storage
- [ ] **Dedicated databases** (relational — PostgreSQL, MySQL)
- [ ] **Key-value stores** (Redis, DynamoDB — for caching, sessions)
- [ ] **Data warehousing** (analytics, reporting, historical data)
- [ ] Justify **why** each storage technology was chosen

#### 4. Security
- [ ] **Firewalls** (network-level protection)
- [ ] **HTTPS / TLS** (data-in-transit encryption)
- [ ] **Rate limiting** (abuse prevention)
- [ ] Authentication & authorization
- [ ] PCI DSS compliance (for payment data)

#### 5. Observability (Technical + Business Metrics)
- [ ] **Technical metrics:** API failure rate, latency percentiles, error rates, **CPU usage**, memory
- [ ] **Business metrics:** Per-user failure rates, conversion rates, transaction volume
- [ ] **Tracing systems:** Distributed tracing (e.g., OpenTelemetry, Jaeger)
- [ ] **Alerting** — thresholds, paging, escalation
- [ ] Dashboards (Grafana / Datadog)

#### 6. Event-Driven Architecture
- [ ] **Message queues** (Kafka, SQS, RabbitMQ)
- [ ] **Webhooks** (async notifications to external systems)
- [ ] Event sourcing patterns
- [ ] Why event-driven over request-response

#### 7. Fault Tolerance & Resiliency
- [ ] **Asynchronous communication** to handle failures
- [ ] Retry strategies (exponential backoff + jitter)
- [ ] Circuit breakers
- [ ] Dead letter queues
- [ ] Idempotency (prevent double-processing)
- [ ] Fallback mechanisms

#### 8. Infrastructure & Scalability
- [ ] **Containerization** (Docker, Kubernetes)
- [ ] **Auto-scaling** (horizontal scaling based on load)
- [ ] **Multi-regional deployment** — latency, data residency, failover
- [ ] **Hosting** — cloud provider choice (AWS/GCP/Azure), managed vs self-hosted trade-offs
- [ ] **Database technology choices** and scaling strategies (sharding, read replicas)
- [ ] Deployment strategies (blue-green, canary)
- [ ] Load balancing

---

### The Senior/Principal Differentiator

> The recruiter said this **twice across both briefings**: **"The distinction between mid-level and senior is the level of reasoning behind decision-making."** And in the second briefing: **"Reasoning behind the architecture choosing — go in depth."**

This is clearly what they're scoring hardest on. Don't just name a technology — **argue for it like a principal engineer defending an RFC.**

| Level | What They Say | What the Interviewer Hears |
|-------|---------------|---------------------------|
| **Mid-Level** | "I'd use Kafka" | Knows the tool, doesn't know why |
| **Senior** | "Kafka because we need guaranteed delivery and decoupled consumers. Trade-off is operational complexity." | Understands trade-offs |
| **Principal** | "Kafka for payment events because losing a payment is unacceptable — 99.999% availability vs our business services at ~99.9%. For webhook dispatch where delivery order is less critical, SQS is simpler and sufficient. I'm choosing complexity only where the cost of failure justifies it." | **Reasons per context, doesn't apply one-size-fits-all** |

**The formula for every architectural choice:**

```
1. STATE the choice        → "I'd use PostgreSQL for the payment data"
2. EXPLAIN why             → "Because we need ACID transactions — money movement must be atomic"
3. ACKNOWLEDGE trade-offs  → "The trade-off is horizontal scaling is harder than NoSQL"
4. EXPLAIN mitigation      → "We can handle that with read replicas, partitioning, and sharding by merchant_id"
5. CONTRAST alternatives   → "DynamoDB would scale easier but doesn't give us multi-table transactions"
6. DEFEND when challenged  → OR concede gracefully: "Good point — if we expect >100K TPS, we'd need to revisit"
```

---

## Study Priority (What to Prepare This Weekend)

### Must-Know (Non-Negotiable)

| Priority | Topic | Reference |
|----------|-------|-----------|
| **P0** | Requirement gathering questions for Payment API + Dashboard | Practice out loud |
| **P0** | API design (REST + async flows) | [00-payment-system-overview.md](00-payment-system-overview.md) |
| **P0** | Idempotency & exactly-once delivery | [02-idempotency.md](02-idempotency.md) |
| **P0** | Fault tolerance patterns (retry, timeout, DLQ) | [01-reliability-patterns.md](01-reliability-patterns.md) |
| **P0** | Security (HTTPS, firewalls, PCI DSS) | [03-security.md](03-security.md) |

### Should-Know (Strong Senior Signal)

| Priority | Topic |
|----------|-------|
| **P1** | Observability — technical metrics, business metrics, distributed tracing, alerting |
| **P1** | Event-driven architecture — Kafka, webhooks, async patterns |
| **P1** | Data storage trade-offs — relational vs key-value vs warehouse |
| **P1** | Infrastructure — containerization, auto-scaling, deployment strategies |
| **P1** | Multi-regional deployment — latency, data residency, failover |
| **P1** | Hosting — cloud choices and managed vs self-hosted trade-offs |

### Nice-to-Know (Principal-Level Polish)

| Priority | Topic |
|----------|-------|
| **P2** | CAP theorem and how it applies to payment systems |
| **P2** | CQRS / Event sourcing for payment data |
| **P2** | Cost optimization and capacity planning |

---

## Quick Reference — File Index

| File | Content |
|------|---------|
| [00-payment-system-overview.md](00-payment-system-overview.md) | Architecture, components, payment flow |
| [01-reliability-patterns.md](01-reliability-patterns.md) | Retry + jitter, timeout, fallback, circuit breaker, Kafka, DLQ |
| [02-idempotency.md](02-idempotency.md) | Two-layer idempotency (Redis SETNX + DB constraint) |
| [03-security.md](03-security.md) | Encryption, TLS/mTLS, PCI DSS, tokenization, UK compliance |
| [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) | Quick-reference cheat sheet |
| **05-recruiter-briefing.md** | **This file — system design focus** |
| [06-values-interview.md](06-values-interview.md) | Values interview (STAR framework) |
| [07-data-layer.md](07-data-layer.md) | SQL vs NoSQL, CAP theorem, PostgreSQL, Redis, data warehouse |
| [08-trade-offs.md](08-trade-offs.md) | Stage-by-stage trade-offs, hosting, observability, infra |
| [09-requirements-fr-nfr.md](09-requirements-fr-nfr.md) | Functional & non-functional requirements with targets |
| [10-requirement-gathering-questions.md](10-requirement-gathering-questions.md) | Checkout.com-specific requirement gathering questions |
| [11-interview-walkthrough.md](11-interview-walkthrough.md) | Full interview flow — what to say, in order, with trade-offs |
| [12-kafka-fundamentals.md](12-kafka-fundamentals.md) | Kafka core concepts, partition ordering, replication, delivery semantics |
| [13-kafka-cheat-sheet.md](13-kafka-cheat-sheet.md) | Kafka quick reference — 10 key concepts, config, failure scenarios |
| [14-bank-inventory-saga.md](14-bank-inventory-saga.md) | Bank-to-inventory design — Saga pattern, compensating actions |
| [15-database-sharding.md](15-database-sharding.md) | Sharding by merchant_id, lookup table, hot merchants, data residency |
