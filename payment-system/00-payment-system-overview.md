# Payment System Design — Overview & Architecture

> Source: [lucianoprea.com/blog/payment-system-design](https://lucianoprea.com/blog/payment-system-design)

---

## The One-Liner

A payment system **coordinates money movement** from a customer's bank to a merchant's bank, while guaranteeing **exactly-once processing**, **auditability**, and **fault tolerance**.

---

## Core Components (6 Players)

| Component | Role | Think of it as... |
|-----------|------|-------------------|
| **Payment Service** | Orchestrates the entire flow — stores payment events, calls PSP, updates wallet & ledger | The **conductor** |
| **PSP** (Payment Service Provider) | Third-party that processes card payments, provides hosted payment page, risk management, **reconciliation** tools | The **middleman** to banks |
| **Payment Gateway** | Validates credentials, transfers funds, enforces **PCI DSS / GDPR** compliance, forwards to fraud verification | The **security guard** |
| **Acquiring Bank** | Merchant's bank — captures transaction info, validates, routes through **card networks** to issuing bank | Merchant's **money receiver** |
| **Issuing Bank** | Customer's bank — approves/declines, checks balance & account status | Customer's **money sender** |
| **Kafka (Message Queue)** | Persists payment events so nothing is lost — **99.999% availability** | The **safety net** |

### Post-Payment Services

| Component | Role |
|-----------|------|
| **Wallet Service** | Tracks merchant **account balance** — updated after successful PSP processing |
| **Ledger Service** | Appends **every financial transaction record-by-record** — used for revenue analysis & **auditing** |

---

## The Payment Flow (6 Steps)

```
Customer ──► Merchant ──► PSP (hosted page) ──► Acquiring Bank ──► Card Network ──► Issuing Bank
                                                                                        │
Customer ◄── Merchant ◄── PSP ◄────────────── Acquiring Bank ◄── Card Network ◄────────┘
                                                                              (approve/decline)
```

### Step-by-Step

1. **Customer places order** and provides payment info to merchant
2. **Merchant redirects** customer to **PSP-hosted payment form** (merchant never sees raw card data)
3. **PSP transmits** cardholder info to the **acquiring bank**
4. **Acquiring bank** captures, validates, routes through **card network** to **issuing bank**
5. **Issuing bank** approves or declines — response flows back through the chain
6. **Merchant receives status** and displays result to customer

### Key Insight

> The merchant **never directly handles card details** — the PSP's hosted page collects them. This is critical for **PCI DSS compliance**.

---

## Internal System Flow

```
Order Placed
    │
    ▼
Payment Service ───► stores payment event in DB
    │
    ▼
Payment Service ───► creates event in Kafka (operation NOT complete until event is persisted)
    │
    ▼
PSP processes card payment (amount + currency)
    │
    ▼
On success:
    ├──► Wallet Service updates merchant balance
    └──► Ledger Service appends transaction record
```

---

## Why This Design Matters

| Concern | How It's Addressed |
|---------|--------------------|
| **Money can't be lost** | Kafka persists events before processing continues |
| **Double-charging** | Idempotency keys + DB unique constraints |
| **Auditing** | Ledger records every transaction record-by-record |
| **Bank failures** | Retry + timeout + fallback patterns |
| **Compliance** | PSP hosted page, Payment Gateway enforces PCI DSS / GDPR |

---

**Next:** [01-reliability-patterns.md](01-reliability-patterns.md) — How the system stays alive when things break
