# PSP Architecture — Full System Diagram

> We ARE the PSP (Checkout.com). We own tokenization, 3DS, acquiring, and card network connections.

---

## Full Architecture

```
┌──────────────┐
│   Merchant   │  (our customer — e.g., ASOS, Uber)
│   Backend    │
└──────┬───────┘
       │  POST /payments (amount, currency, card_token)
       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        OUR PLATFORM (the PSP)                        │
│                                                                      │
│  ┌────────────────┐                                                  │
│  │  API Gateway   │  OAuth/JWT, rate limiting, TLS termination       │
│  └───────┬────────┘                                                  │
│          │                                                           │
│          ▼                                                           │
│  ┌────────────────┐     ┌───────────┐     ┌────────────────┐        │
│  │Payment Service │────►│  Redis    │     │  PostgreSQL    │        │
│  │(orchestrator)  │────────────────────────►• payment_orders│        │
│  │                │     │• Idempotency    │• ledger_entries│        │
│  │ 1. Save PENDING│     │• Rate limits    │• wallet_accounts│       │
│  │ 2. Publish     │     │• Cache          │                │        │
│  │ 3. Return 202  │     └───────────┘     └────────────────┘        │
│  └───────┬────────┘                                                  │
│          │                                                           │
│          ▼                                                           │
│  ┌─────────────────┐                                                 │
│  │ Kafka           │  payment-events (12 partitions)                 │
│  └──┬──┬──┬──┬─────┘                                                 │
│     │  │  │  │                                                       │
│     │  │  │  └──► Webhook Dispatcher ──► Merchant endpoint           │
│     │  │  └─────► Wallet Service (update merchant balance)           │
│     │  └────────► Ledger Service (double-entry bookkeeping)          │
│     │                                                                │
│     ▼                                                                │
│  ┌─────────────────────────────────────────────────────────┐         │
│  │ Payment Processor (consumer)                            │         │
│  │                                                         │         │
│  │  STEP 1: Decrypt                                        │         │
│  │  ┌────────────────┐                                     │         │
│  │  │ Token Vault    │  tok_xxx in → raw PAN out           │         │
│  │  │ (PCI Level 1)  │  AES-256, HSM key management       │         │
│  │  └───────┬────────┘                                     │         │
│  │          │ raw PAN (BIN identifies issuing bank)        │         │
│  │          ▼                                              │         │
│  │  STEP 2: Authenticate (3DS)                             │         │
│  │  ┌────────────────┐                                     │         │
│  │  │  3DS Engine    │──── challenge ──►┐                  │         │
│  │  │  (SCA/PSD2)    │◄─── result ─────┐│                  │         │
│  │  └───────┬────────┘                 ││                  │         │
│  │          │ PAN + 3DS proof           ││                  │         │
│  │          ▼                           ││                  │         │
│  │  STEP 3: Authorize                   ││                  │         │
│  │  ┌─────────────────────────────┐     ││                  │         │
│  │  │  Acquiring Gateway          │     ││                  │         │
│  │  │  (principal member Visa/MC) │     ││                  │         │
│  │  │  Sends: PAN + 3DS + amount  │     ││                  │         │
│  │  └─────────────┬───────────────┘     ││                  │         │
│  │                │                     ││                  │         │
│  │  STEP 4: Publish result event        ││                  │         │
│  └────────────────┼─────────────────────┼┼──────────────────┘         │
│                   │                     ││                            │
└───────────────────┼─────────────────────┼┼────────────────────────────┘
                    │                     ││
                    ▼                     ││
          ┌──────────────────┐            ││
          │  Card Networks   │ Visa, MC   ││
          └────────┬─────────┘            ││
                   │                      ││
                   ▼                      ▼│
          ┌──────────────────────────────────┐
          │  Issuing Bank (customer's bank)  │
          │                                  │
          │  • ACS: handles 3DS challenge    │ ◄── 3DS Engine talks here
          │  • Auth: approves/declines       │ ◄── Card Network routes here
          └──────────────────────────────────┘
```

---

## Payment Flow (Step by Step)

```
SYNCHRONOUS (fast — returns 202 to merchant):
1. Merchant calls POST /payments with tok_xxx
2. API Gateway → auth, rate limit, TLS termination
3. Payment Service → idempotency check (Redis SETNX + DB PK)
4. Payment Service → saves payment as PENDING in PostgreSQL
5. Payment Service → publishes PaymentCreated to Kafka
6. Payment Service → returns 202 Accepted to merchant

ASYNCHRONOUS (Payment Processor consumer picks up event):
7. Token Vault → decrypt tok_xxx → returns raw PAN
8. 3DS Engine → challenges customer via Issuing Bank ACS → returns 3DS proof
9. Acquiring Gateway → sends PAN + 3DS proof + amount to card network
10. Card Network → routes to Issuing Bank → approves/declines
11. Payment Processor → publishes PaymentAuthorized or PaymentDeclined to Kafka

DOWNSTREAM CONSUMERS react to result:
12. Ledger Service → debit + credit entries
13. Wallet Service → updates merchant balance (on capture)
14. Webhook Dispatcher → notifies merchant of outcome
```

---

## What We Own (vs Merchant-Side Design)

| Component | We Own It | What It Does |
|---|---|---|
| **API Gateway** | Yes | Auth, rate limiting, TLS, WAF |
| **Payment Service** | Yes | Orchestrates the payment lifecycle |
| **Token Vault** | Yes | Stores raw card PANs encrypted (AES-256). PCI DSS Level 1 |
| **3DS Engine** | Yes | SCA compliance — triggers OTP/biometric with issuing bank |
| **Acquiring Gateway** | Yes | Direct connection to Visa/Mastercard as principal member |
| **Kafka** | Yes | Event backbone — payment-events topic |
| **PostgreSQL** | Yes | Source of truth — payment state, ledger, wallets |
| **Redis** | Yes | Idempotency, rate limiting, caching |
| **Webhook Dispatcher** | Yes | Notify merchants of payment status changes |
| **Settlement Engine** | Yes | Daily batch — aggregate, generate files, transfer to merchant bank |
| **Fraud Detection** | Yes | Inline scoring before authorization |

---

## Key Differences: We Are the PSP

| Aspect | Merchant-side design | We ARE the PSP |
|---|---|---|
| **Card data** | Never touches us | We store raw PANs in our vault |
| **PCI scope** | SAQ-A (minimal) | Level 1 (full — we handle card data) |
| **3DS** | PSP handles | We build and run the 3DS engine |
| **Card networks** | PSP/acquirer contacts | We contact directly (principal member) |
| **Our customers** | End consumers | Merchants (ASOS, Uber, etc.) |
| **Tokenization** | PSP does it | We do it — vault + hosted page |
| **Settlement** | PSP settles to us | We settle to merchants |
| **Fraud** | PSP scores | We score inline before auth |

---

## Components Explained

### Token Vault
- Stores raw card numbers (PANs) encrypted with AES-256
- Returns opaque token (`tok_xxx`) to merchant — they never see the real card
- HSM (Hardware Security Module) for key management
- PCI DSS Level 1 — annual audit, penetration testing, network segmentation

### 3DS Engine
- Implements 3DS2 protocol (EMVCo standard)
- Triggers SCA challenge: biometric or OTP
- Communicates with issuing bank's Access Control Server (ACS)
- Manages exemptions: low-value (<£30), recurring, trusted merchants, TRA
- Shifts fraud liability from merchant to issuer on successful 3DS

### Acquiring Gateway
- Principal member of Visa and Mastercard (not via third-party acquirer)
- Sends authorization requests in ISO 8583 format
- Handles clearing and settlement messages
- Direct connection = lower fees, faster settlement, more control

### Settlement Engine
- Daily batch: collect captured payments → aggregate per merchant → transfer funds
- T+1 to T+3 depending on merchant agreement
- Reconciles against card network settlement files
- See [17-settlement-batch-processing.md](17-settlement-batch-processing.md) for full design

---

## Interview Framing

> "Since we're designing the PSP, we own the full stack. The merchant calls our API — they never touch card data. Our hosted payment page captures card details into our PCI Level 1 token vault. We run the 3DS engine for SCA compliance. As a principal member of Visa and Mastercard, our acquiring gateway connects directly to the card networks — no third-party acquirer needed. This gives us lower interchange fees, faster settlement, and full control over the authorization flow."
