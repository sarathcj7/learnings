# Data Protection & Encryption

Protecting data at rest (storage) and in transit (network). Compliance with regulations (GDPR, PCI-DSS).

---

## TL;DR

- **Encryption at rest**: Database, backups, stored files
- **Encryption in transit**: HTTPS, TLS, mTLS
- **Key management**: Secure key storage, rotation, separation of duties
- **PII handling**: Personally identifiable information needs special protection
- **Compliance**: GDPR (right to be forgotten), HIPAA, PCI-DSS

---

## Encryption at Rest

### Database Encryption

```
SQL Server configured with Transparent Data Encryption (TDE):

Without encryption:
  Database file on disk: readable plaintext
  If disk stolen: Data compromised

With TDE:
  Database file on disk: encrypted
  SQL Server decrypts automatically (transparent to app)
  If disk stolen: Data protected

Performance:
  Minimal overhead (CPU cost ~5%)
  Automatically decrypts blocks as needed
  
Key storage:
  Master key stored separately (AWS KMS, Azure Key Vault)
  If master key lost: Data unrecoverable
```

---

### File Storage Encryption

```
S3 bucket with encryption:

Without encryption:
  File uploaded: s3://bucket/sensitive.csv
  On disk: plaintext
  
With encryption:
  File uploaded: s3://bucket/sensitive.csv
  On disk: encrypted (AWS S3-managed key)
  S3 decrypts on GET (transparent to app)
  
Client-side encryption (more secure):
  Client encrypts before uploading
  S3 stores encrypted blob
  Only client can decrypt (S3 can't read data)
```

---

### Key Derivation

```
Scenario: Encrypt 1M database records

Naive approach:
  1 master key for entire database
  If key exposed: All data compromised
  
Better: Key per user (derived from user ID)
  master_key = "secret"
  user_key = HKDF(master_key, user_id)
  encrypted_record = AES(record, user_key)
  
Benefit:
  Compromise of one user key only exposes that user's data
  Can revoke user key without re-encrypting all data
```

---

## Encryption in Transit

### TLS 1.3 (HTTPS)

```
Browser → Server connection:

1. ClientHello: Browser sends supported ciphers, version
2. ServerHello: Server picks cipher, sends certificate
3. Certificate verification: Browser verifies server's certificate
4. Key exchange: Both generate shared secret (ECDHE)
5. Finished: Both verify handshake integrity
6. Data transmission: All traffic encrypted with shared key

Encryption algorithm (modern):
  ChaCha20-Poly1305 or AES-256-GCM
  Forward secrecy: Even if private key stolen, past traffic safe
  
Configuration:
  TLS 1.2+ only (no SSLv3, TLS 1.0, 1.1)
  Strong cipher suites only
  Certificate from trusted CA
```

---

## Key Management

### Key Rotation

```
Current key: key_v1 (used for 1 year)
New key: key_v2

Rotation strategy:
  Day 1: Start using key_v2 for encryption
  Day 1-90: Keep key_v1 for decryption (support old encrypted data)
  Day 90: Can decrypt old data, but not encrypt with it
  Day 365: Retire key_v1 (if no old data needs reading)

Why:
  If key_v1 compromised (leaked to press, attacker): Limits exposure
  All data encrypted with key_v1 only exposed during that 1-year period
  Not retroactive
```

---

### Key Separation

```
Principle: Different keys for different purposes

Bad:
  1 key for everything (encryption + signing + auth)
  If key compromised: All systems affected
  
Good:
  Data encryption key (DEK): Encrypts actual data
  Key encryption key (KEK): Encrypts DEK
  Signing key: Signs messages
  Auth key: Authenticates users
  
Benefit:
  Compromise of auth key doesn't compromise data
  Can rotate individual keys independently
  Better security posture
```

---

### Secure Key Storage

```
On server:

❌ Bad: Store key in config file
  config.yaml: encryption_key: "secret"
  If config leaked: Key compromised

✅ Good: Use key management service
  AWS KMS, Azure Key Vault, HashiCorp Vault
  Server requests key from KMS (with audit logging)
  KMS returns key
  Never disk-stored
  
Architecture:
  App → KMS API → "Give me key for database"
  KMS: "Verify you're authorized, check audit log"
  KMS: "Return key" (in memory, not disk)
  App: Uses key, discards when done
```

---

## PII (Personally Identifiable Information)

### What is PII?

```
Personally identifiable information:
  ✅ Name
  ✅ Email
  ✅ Phone number
  ✅ Social security number
  ✅ IP address (sometimes)
  ✅ Location data
  ✅ Credit card number
  
Sensitive:
  Requires encryption at rest
  Requires secure deletion
  Requires access controls
```

---

### Handling PII Securely

```
User registers: name="John", email="john@example.com"

Storage:
  Encrypted in database
  Key rotation annually
  Backup encrypted
  
Access control:
  Only backend service can read
  Frontend never sees plaintext
  Admin team: Audit log on access
  
Retention:
  Store while user active
  On deletion: Securely wipe
  GDPR: User can request deletion → purge within 30 days
  
Sharing:
  Never send via email
  Never log in plaintext
  Only transmit over TLS
```

---

### GDPR Right to be Forgotten

```
User: "Delete my data"

Requirement:
  Within 30 days: All personal data must be deleted
  
Implementation challenges:
  Database: Easy (DELETE FROM users WHERE id=123)
  Backups: Need encryption key rotation + purge old backups
  Analytics: Aggregated data OK (can't identify user)
  Audit logs: Can keep (necessary for compliance)
  
Solution:
  Anonymize instead of delete (if possible)
  Purge backups older than 30 days
  Keep only non-PII audit trail
```

---

## Compliance Frameworks

### PCI-DSS (Payment Card Industry)

```
Requirement: Secure credit card data

Rules:
  ✅ Encrypt card data in transit (TLS 1.2+)
  ✅ Encrypt at rest (AES-256)
  ✅ Never store full card number (store last 4 digits only)
  ✅ No plaintext in logs
  ✅ Regular penetration testing
  ✅ Access controls (least privilege)

Audit:
  Annual compliance assessment
  Quarterly vulnerability scans
  Maintain audit trail
```

---

### HIPAA (Healthcare)

```
Requirement: Protect patient health information (PHI)

Rules:
  ✅ Encrypt patient data
  ✅ Role-based access (doctor sees only assigned patients)
  ✅ Audit logging (who accessed what, when)
  ✅ Data breach notification (within 60 days)
  ✅ Business associate agreements (vendors must comply)
  
Penalties for breach:
  Up to $1.5M per year per violation
  Criminal charges possible
```

---

### GDPR (General Data Protection Regulation)

```
Requirement: Protect EU citizen data

Key principles:
  ✅ Explicit consent before collecting data
  ✅ Transparency (users know what data you have)
  ✅ Right to access (users can download their data)
  ✅ Right to be forgotten (users can request deletion)
  ✅ Data portability (users can export to another service)
  ✅ Breach notification (inform within 72 hours)
  
Fines:
  Up to 20 million EUR or 4% of annual revenue (whichever higher)
  
Applies to:
  Any company with EU users (even if company outside EU)
```

---

## Security Checklist

- [ ] TLS 1.2+ enforced everywhere
- [ ] Database encrypted at rest
- [ ] Backups encrypted
- [ ] Key management service (KMS) in use
- [ ] Key rotation automated (annually minimum)
- [ ] PII encrypted
- [ ] Access controls logged (audit trail)
- [ ] No secrets in code/logs
- [ ] Compliance framework identified (PCI, HIPAA, GDPR)
- [ ] Incident response plan (breach scenario)

---

## Related Fundamentals

- [Authentication](authentication-and-authorization.md) – Secure access
- [TLS/Certificates](tls-and-certificates.md) – Cryptographic foundation
- [Observability/Audit Logging](../observability/) – Track access

---

**Status**: ✅ Complete. Covers encryption at rest/transit, key management, PII, compliance.

