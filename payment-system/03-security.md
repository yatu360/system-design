# Security — Protecting the Money

---

## The Principle

> Payment systems handle the **most sensitive data** on the internet — card numbers, bank details, personal identity. Security is **non-negotiable**.

---

## Six Pillars of Payment Security

### 1. Encryption

**Two layers — data at rest AND in transit:**

| Layer | What | How |
|-------|------|-----|
| **Data-at-Rest** | Data stored in databases/disks | **Disk encryption** and **database encryption** tools — data is converted to a secure format requiring a **decryption key** to read |
| **Data-in-Transit** | Data moving between services | **TLS** for confidentiality, integrity, and authentication + **VPN** for secure encrypted connections |

```
[Client] ──TLS──► [Payment Service] ──VPN──► [PSP] ──TLS──► [Bank]
                         │
                    Encrypted DB
                  (data-at-rest)
```

> **TLS provides three things:** confidentiality, data integrity, and authentication between two parties.

---

### 2. Access Controls

- Restrict data access to **authorized users only**
- **Two-factor authentication (2FA)** to verify user identities
- **Principle of least privilege** — each service/user gets minimum required access

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

> **Rainbow table:** An attacker precomputes hashes of common passwords. If your hash matches, they know your password. Defense: **strong passwords + salting**.

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
                                PSP handles everything:
                                ├── PCI DSS compliance
                                ├── Risk management
                                ├── Fraud prevention
                                └── Card data transmission to banks
```

> **Key insight:** By using a PSP hosted page, the merchant **offloads PCI DSS compliance** to the PSP. The merchant never touches card numbers.

---

## Payment Gateway's Security Role

| Function | Purpose |
|----------|---------|
| **Validates financial credentials** | Ensures payment info is legitimate |
| **Enforces PCI DSS** | Card data security standards |
| **Enforces GDPR** | Data privacy regulations |
| **Forwards to fraud verification** | Catches suspicious transactions |

---

## Security Quick Reference for Interviews

| Question | Answer |
|----------|--------|
| How do you protect data at rest? | **Disk/DB encryption** with decryption keys |
| How do you protect data in transit? | **TLS** (confidentiality + integrity + auth) + **VPN** |
| How do you avoid handling card data? | **PSP hosted payment page** — merchant never sees card numbers |
| How do you prevent double-charging? | **Idempotency keys** with DB unique constraints |
| How do you detect breaches? | **Data integrity monitoring** on credentials and key stores |
| How do you prevent rainbow table attacks? | **Strong passwords + salting** |
| What compliance standards matter? | **PCI DSS** (card data) + **GDPR** (privacy) |

---

**Next:** [04-interview-cheat-sheet.md](04-interview-cheat-sheet.md) — Everything on one page
