# Security — Protecting the Money

---

## The Principle

> Payment systems handle the **most sensitive data** on the internet — card numbers, bank details, personal identity. Security is **non-negotiable**. In the UK, you're regulated by the **FCA** and must comply with **PCI DSS**, **PSD2/SCA**, and **UK GDPR**.

---

## Six Pillars of Payment Security

### 1. Encryption

**Two layers — data at rest AND in transit:**

| Layer | What | How |
|-------|------|-----|
| **Data-at-Rest** | Data stored in databases/disks | **Disk encryption** and **database encryption** tools — data is converted to a secure format requiring a **decryption key** to read |
| **Data-in-Transit** | Data moving between services | **TLS 1.2+** for external traffic, **mTLS** for internal service-to-service (zero-trust) |

```
[Client] ──TLS──► [API Gateway] ──mTLS──► [Payment Service] ──mTLS──► [Kafka / DB]
                                                   │
                                              TLS + signed payloads
                                                   │
                                                   ▼
                                                 [PSP] ──TLS──► [Bank]

All databases: encrypted at rest (AES-256)
```

> **TLS** provides confidentiality, data integrity, and authentication between two parties.
> **mTLS** adds mutual verification — both sides present certificates. Standard for zero-trust internal networks.

---

### 2. Access Controls

- Restrict data access to **authorized users only**
- **Two-factor authentication (2FA)** to verify user identities
- **Principle of least privilege** — each service/user gets minimum required access
- **OAuth 2.0 + JWT** for merchant API and dashboard access
- **API keys** for service identification, **mTLS** for internal service-to-service auth

---

### 3. Software Maintenance

- **Regularly update** software, libraries, and operating systems
- Prevents exploitation of **known vulnerabilities**
- Patch management is continuous, not one-time

---

### 4. Data Backup

- **Regular backups** to ensure recovery from loss or damage
- Critical for **disaster recovery** scenarios
- Backups themselves must be **encrypted**

---

### 5. Password Security

- Use **long, complex passwords** that are difficult to guess or crack
- Avoid common passwords like "password"
- Defend against **rainbow table attacks** (precomputed tables of reversed password hashes)

> **Rainbow table:** An attacker precomputes hashes of common passwords. If your hash matches, they know your password. Defense: **strong passwords + salting + bcrypt/argon2**.

---

### 6. Data Integrity Monitoring

- **Regularly check for changes** in vulnerable data
- Generate **security alerts** when anomalies detected
- Detects **malware and threats** early

**Trade-off:** Resource-intensive — focus monitoring on the **most vulnerable and confidential data**:
- **User credentials**
- **Encryption key stores**

---

## PSP's Role in Security

The PSP provides a **hosted payment page** — this is a critical security design:

```
Customer ──► Merchant site ──► REDIRECT to PSP hosted page
                                       │
                                Customer enters card details HERE
                                (merchant NEVER sees raw card data)
                                       │
                                PSP handles:
                                ├── PCI DSS compliance (card data vault)
                                ├── Tokenization (returns token, stores real card)
                                ├── 3DS / SCA authentication (PSD2 mandatory in UK)
                                ├── Risk management & fraud scoring
                                └── Card data transmission to banks
```

> **Key insight:** By using a PSP hosted page, the merchant **offloads PCI DSS compliance** to the PSP. The merchant stores **tokens**, never card numbers.

---

## Tokenization

> The merchant never stores card numbers. The PSP stores the real card and returns an opaque **token**.

| What | Where Stored | Who Can Use It |
|------|-------------|----------------|
| Real card number (`4242 4242 4242 4242`) | **PSP vault only** (PCI-compliant) | PSP |
| Token (`tok_8a3f2b1c9d4e`) | **Our database** | Our system — sent to PSP on next payment |

**Benefits:**
- Dramatically **reduces PCI DSS scope** — our systems never handle card data
- Enables **card-on-file** for returning customers (one-click payments)
- Tokens are **merchant-specific** — useless if stolen
- PSP handles card expiry updates, network token migration

---

## UK Compliance Landscape

| Regulation | Authority | What It Requires | How We Comply |
|-----------|-----------|-----------------|---------------|
| **PCI DSS** | PCI SSC (global) | Secure handling of card data | PSP hosted page + tokenization — we never touch card numbers |
| **PSD2 / SCA** | FCA (UK) | Strong Customer Authentication on payments | 3DS via PSP — two-factor (biometric/OTP). Exemptions for low-value, recurring, trusted merchants |
| **UK GDPR** | ICO | Data privacy, right to erasure, data residency | Pseudonymization — delete PII mapping on erasure request. Ledger entries remain but unidentifiable |
| **FCA Regulations** | FCA | Proper financial record-keeping, consumer protection | Double-entry append-only ledger, daily reconciliation, audit trails |
| **Open Banking (PSD2)** | CMA / FCA | API access to bank accounts | Relevant if supporting bank-to-bank payments (A2A) as alternative to card payments |
| **Electronic Money Regs 2011** | FCA | If holding customer funds | Safeguarding requirements — customer funds held in segregated accounts |

### 3DS / Strong Customer Authentication (SCA)

> **Mandatory in the UK.** Failure to implement = transaction declines from issuing banks.

- **What:** Two of three factors required — something you **know** (PIN), **have** (phone), **are** (biometric)
- **How:** PSP handles 3DS2 flow embedded in checkout. Customer authenticates with their bank
- **Exemptions:** Low-value (<£30 individual, <£100 cumulative), recurring after first payment, merchant-initiated, low-risk TRA (Transaction Risk Analysis)
- **Liability shift:** With 3DS, fraud liability moves from **merchant to issuing bank**
- **3DS2 vs 3DS1:** 3DS2 is embedded (less friction), 3DS1 was full-page redirect (deprecated)

---

## Security Quick Reference for Interviews

| Question | Answer |
|----------|--------|
| How do you protect data at rest? | **AES-256 encryption** on disk and database level |
| How do you protect data in transit? | **TLS 1.2+** external, **mTLS** internal (zero-trust) |
| How do you avoid handling card data? | **PSP hosted page** + **tokenization** — we store tokens, not card numbers |
| How do you handle authentication? | **OAuth 2.0 + JWT** for merchants/dashboard, **mTLS** for internal services, **API keys** for identification |
| How do you prevent double-charging? | **Idempotency keys** — Redis SETNX + DB unique constraints |
| How do you handle SCA in the UK? | **3DS2** via PSP, mandatory under **PSD2**. Exemptions for low-value and recurring |
| How do you handle UK GDPR right-to-erasure? | **Pseudonymization** — delete PII mapping, ledger entries remain anonymized |
| What compliance standards matter in the UK? | **PCI DSS** (card data) + **PSD2/SCA** (authentication) + **UK GDPR** (privacy) + **FCA** (financial conduct) |

---

**Next:** [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) — Everything on one page
