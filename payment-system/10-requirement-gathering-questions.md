# Requirement Gathering Questions — Checkout.com Interview

> These are the questions you ask in the **first 8-10 minutes** before drawing anything. The recruiter stressed this twice — requirement gathering is the make-or-break signal.
>
> Tailor every question to show you understand **Checkout.com's business** — they are a **full-stack PSP and acquirer**, FCA-regulated, with direct card network connections. Not a merchant, not a bank, not an aggregator.

---

## The Scope Question (Ask First)

> This is the single most important question. It frames the entire 90 minutes.

**Ask:**
> "Before I start — are we designing **Checkout.com's own platform** — the PSP/acquirer infrastructure that processes payments on behalf of merchants? Or are we designing a **merchant's payment system** that integrates with a PSP like Checkout.com?"

**Why this matters:**

| Scope | You're Designing | Complexity |
|-------|-----------------|------------|
| **Checkout.com itself** | The PSP — tokenization vault, direct card network connections, acquiring, 3DS engine, merchant API, settlement engine | Much harder — you own the full stack, PCI DSS Level 1, FCA-regulated |
| **Merchant using Checkout.com** | A merchant's backend that calls Checkout.com's API — payment orchestration, idempotency, webhooks, dashboard | Easier — Checkout.com handles card data, 3DS, acquiring, settlement |
| **Hybrid / specific component** | A single component — e.g., "design the webhook delivery system" or "design the reconciliation pipeline" | Focused — go deep on one area |

> **Follow-up:** "And is the dashboard a merchant-facing product, an internal ops tool, or both?"

---

## Functional Requirements — What Are We Building?

### Payment API Scope

| # | Question | Why You're Asking | Checkout.com Context |
|---|----------|-------------------|---------------------|
| 1 | "What payment methods are in scope — **cards only**, or also digital wallets (Apple Pay, Google Pay), bank transfers, and alternative methods (Klarna, iDEAL)?" | Determines API surface area and integration complexity | Checkout.com supports 20+ payment methods — they'll likely scope it to cards + 1-2 APMs |
| 2 | "Are we handling **authorization and capture separately**, or as a single charge? Do we need to support **voids**?" | Shows you understand the real card lifecycle, not just "create payment" | Checkout.com's API supports auth-only, auth+capture, and voids — this is core to their product |
| 3 | "Do we need to support **recurring payments** and card-on-file, or just one-time transactions?" | Recurring adds tokenization, subscription management, retry logic for failed renewals | Checkout.com has a Retry & Recovery product specifically for this |
| 4 | "Are **refunds** full only, or do we need **partial refunds**? Who initiates — the merchant via API, or also via the dashboard?" | Scopes the refund flow and who the actors are | Both full and partial refunds are core Checkout.com features |
| 5 | "For the **3DS / SCA flow** — are we building the 3DS engine, or treating it as a dependency we call?" | If designing Checkout.com itself, you might own the 3DS engine. If merchant-side, the PSP handles it | Checkout.com has a built-in 3DS2 engine — they own this |
| 6 | "Do merchants receive payment updates via **webhooks**, **polling**, or both?" | Determines async notification architecture | Checkout.com uses webhooks as primary, polling via GET as fallback |
| 7 | "Is there a **smart routing / payment orchestration** layer — routing to different acquirers or processors based on cost, success rate, or geography?" | Shows awareness of Checkout.com's Flow product — their orchestration layer | Checkout.com's Flow product does exactly this — intelligent routing across multiple processors |

### Dashboard Scope

| # | Question | Why You're Asking | Checkout.com Context |
|---|----------|-------------------|---------------------|
| 8 | "Who uses the dashboard — **merchants**, **internal ops**, or both? Different permission levels?" | Determines auth model, data access patterns, role-based views | Checkout.com's Dashboard serves merchants (transaction views) and internal teams (ops, risk, support) |
| 9 | "Does the dashboard need **real-time data** (live transaction feed) or is **near real-time** (30s–1min lag) acceptable?" | Drives architecture — WebSocket/SSE vs polling, read replica vs CDC | Real-time is impressive but expensive. Most dashboards are near real-time |
| 10 | "What aggregations matter most — **transaction volume**, **success/failure rates**, **revenue**, **chargeback rate**? Per merchant?" | Shapes the data warehouse queries and materialized views | Checkout.com's dashboard surfaces per-merchant metrics, chargeback rates (critical for card network compliance) |
| 11 | "Does the dashboard include **dispute/chargeback management** — viewing disputes, submitting evidence, tracking outcomes?" | Chargebacks are a major operational concern for a PSP | Checkout.com provides dispute management tooling in their dashboard |

---

## Non-Functional Requirements — How Well Does It Need to Work?

| # | Question | Why You're Asking | What to Listen For |
|---|----------|-------------------|--------------------|
| 12 | "What **scale** are we designing for — transactions per second? Are we talking hundreds, thousands, or Checkout.com-scale (**tens of thousands TPS**)?" | Checkout.com processes $250B+ annually. Scale drives every architectural decision | If they say "Checkout.com scale" — you need Kafka, sharding, multi-region |
| 13 | "What's the **latency target** for payment API responses? I'm assuming sub-second for the synchronous accept, with async processing behind it?" | Validates the accept-fast-process-async pattern | Most PSPs target <500ms for the API response, 1-5s for full processing |
| 14 | "What **availability** target — **four nines** (99.99%) for the payment API? Higher?" | Payment downtime = lost revenue for every merchant. Drives redundancy decisions | Checkout.com likely targets 99.99%+ for their core API |
| 15 | "**Zero data loss** for payment records — is that a given, or should I validate?" | Shows you understand financial data durability is non-negotiable | Always yes — RPO = 0 for payment events |
| 16 | "For consistency — **strong consistency** for payment state, **eventual consistency** acceptable for dashboard reads?" | Shows CAP awareness per operation, not per system | Standard pattern — CP for payments, AP for dashboards |

---

## Compliance & Regulation

| # | Question | Why You're Asking | Checkout.com Context |
|---|----------|-------------------|---------------------|
| 17 | "Since we're UK-based and **FCA-regulated** — are there specific compliance constraints I should design around? PCI DSS Level 1, PSD2/SCA, UK GDPR data residency?" | Shows you know Checkout.com's regulatory environment | Checkout.com is FCA-authorised, PCI DSS Level 1, PSD2-compliant. All three matter |
| 18 | "For **data residency** — does UK customer data need to stay in UK data centres, or is EEA acceptable post-Brexit?" | Post-Brexit UK GDPR has specific data adequacy rules | Checkout.com has UK and EU infrastructure — this is a real concern for them |
| 19 | "If we're the PSP — are we in **full PCI DSS scope** (Level 1, handling raw card data), or can we reduce scope with tokenization?" | If designing Checkout.com itself, they're Level 1. If merchant-side, scope is reduced | Checkout.com is PCI DSS Level 1 — they handle raw card data in their vault |

---

## Infrastructure & Technical Constraints

| # | Question | Why You're Asking |
|---|----------|-------------------|
| 20 | "Any **cloud provider** preference — AWS, GCP, Azure? Or multi-cloud?" | Shapes which managed services you reference |
| 21 | "**Single region** (UK) to start, or **multi-region** from day one?" | Multi-region adds replication, routing, data residency complexity |
| 22 | "Are there existing systems or **legacy constraints** I should design around, or is this greenfield?" | In an interview, usually greenfield — but asking shows maturity |

---

## How to Ask These (The Technique)

> Don't read a list. State your assumption and ask for confirmation.

### The Formula

```
"I'm assuming [X] — does that match what you have in mind, or should I adjust?"
```

### Example Opening (60 seconds)

```
"Before I start designing — I want to validate the scope. Checkout.com is a full-stack 
PSP and acquirer with direct card network connections. Are we designing the PSP platform 
itself — the acquiring infrastructure, tokenization vault, merchant API — or are we 
designing a merchant's system that integrates with a PSP like Checkout.com?

I'm assuming the Payment API needs to handle payment creation with auth and capture, 
refunds, status queries, and webhook notifications to merchants. The dashboard surfaces 
transaction data, success rates, and chargeback metrics — used by merchants and 
potentially internal ops.

For non-functionals, I'm targeting 99.99% availability, sub-second API latency, 
zero data loss, and UK compliance — PCI DSS, PSD2/SCA, UK GDPR, FCA.

Does that align with what you're expecting, or should I adjust the scope?"
```

> Then **pause**. Let the interviewer respond. This is a **conversation**, not a presentation.

---

## Questions That Show You Know Checkout.com

These are the questions that signal "this person has done their homework":

| Question | What It Signals |
|----------|----------------|
| "Are we designing the **acquiring side** — direct card network connections — or abstracting over third-party acquirers?" | You know Checkout.com is a principal member of Visa/Mastercard, not a reseller |
| "Should the system support **intelligent routing** across multiple processors or acquirers?" | You know about their Flow/orchestration product |
| "Is **network tokenization** in scope — replacing card PANs with network-level tokens for higher auth rates?" | You know about their Vault and Network Tokens products |
| "For failed payments — do we need a **retry and recovery** engine that automatically retries with optimised parameters?" | You know about their Retry & Recovery product |
| "Are we building the **fraud scoring** engine inline, or is that a separate service we call during authorization?" | You know Checkout.com has Fraud Detection Pro as a product |

---

## File Index

| File | Content |
|------|---------|
| [00-payment-system-overview.md](00-payment-system-overview.md) | Architecture, components, payment flow |
| [01-reliability-patterns.md](01-reliability-patterns.md) | Retry, timeout, fallback, circuit breaker, Kafka, DLQ |
| [02-idempotency.md](02-idempotency.md) | Two-layer idempotency (Redis + DB) |
| [03-security.md](03-security.md) | Encryption, TLS/mTLS, PCI DSS, UK compliance |
| [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) | Quick-reference cheat sheet |
| [05-recruiter-briefing.md](05-recruiter-briefing.md) | Recruiter's guidance and checklist |
| [06-values-interview.md](06-values-interview.md) | Values interview (STAR framework) |
| [07-data-layer.md](07-data-layer.md) | SQL vs NoSQL, CAP, ledger, reconciliation |
| [08-trade-offs.md](08-trade-offs.md) | Stage-by-stage trade-offs, hosting, observability |
| [09-requirements-fr-nfr.md](09-requirements-fr-nfr.md) | Functional & non-functional requirements |
| **10-requirement-gathering-questions.md** | **This file — Checkout.com-specific questions** |
