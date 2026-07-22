# TLS & Public Key Infrastructure (PKI)

Certificate authorities, trust chains, and key management for secure communications.

---

## TL;DR

- **PKI**: System for managing digital certificates and public key cryptography
- **Root CA**: Trusted authority (embedded in OS/browser)
- **Certificate chain**: Root → Intermediate → End-entity
- **Certificate pinning**: Pin public key to prevent MITM
- **Key rotation**: Replace keys regularly
- **OCSP**: Check certificate revocation status

---

## Public Key Cryptography

### Asymmetric Encryption

```
Private key: Secret (server keeps safe)
Public key: Shareable (client has copy)

Encrypt with public key: Only private key can decrypt
Encrypt with private key (sign): Anyone with public key can verify

Example:
  Server generates: private_key + public_key
  Server shares: public_key to all clients
  Client encrypts message with public_key
  Server decrypts with private_key (only server can read)
  
  Server signs response with private_key
  Client verifies with public_key (proves it's from server)
```

---

## PKI Components

### Root Certificate Authority (Root CA)

```
Master key (kept offline in secure vault):
  private_key_root (secret!)
  public_key_root (in browser/OS trust store)

Root CA signs:
  "I certify that Intermediate CA X is trustworthy"
  Sign certificate with private_key_root

Browser trusts: Root CAs (built-in list ~100)
  Mozilla maintains list
  Apple maintains list (iOS/macOS)
  Microsoft maintains list (Windows)
```

---

### Intermediate Certificates

```
Why needed?
  Root key too valuable to use frequently
  Risk of compromise if used often
  
Solution:
  Root signs Intermediate certificate
  Intermediate signs End-Entity certificates
  
Example:
  Root: "DigiCert Root"
  ├─ Intermediate: "DigiCert TLS RSA"
  │  ├─ End-Entity: "api.example.com"
  │  ├─ End-Entity: "cdn.example.com"
  │  └─ End-Entity: "mail.example.com"
  └─ Intermediate: "DigiCert ECC"
     └─ End-Entity: "app.example.com"
```

---

### Certificate Revocation

```
Issue: Private key compromised or certificate should be invalidated

Solution: Certificate revocation (two methods)

CRL (Certificate Revocation List):
  List of revoked serial numbers
  Publish periodically
  Browsers check against list
  Problem: List grows large, update latency
  
OCSP (Online Certificate Status Protocol):
  Query: "Is certificate XYZ still valid?"
  Response: "Yes" or "Revoked"
  Real-time, smaller data
  Problem: Extra network request (latency)
  
OCSP Stapling:
  Server pre-fetches OCSP response
  Server sends response with certificate
  No extra request from client
  Best solution
```

---

## Certificate Pinning

### Problem: Compromised CA

```
Scenario:
  Attacker compromises DigiCert (imaginary)
  Attacker generates fake certificate for example.com
  Signed by legitimate root
  Browsers trust it (path to root valid!)
  Attacker intercepts traffic
  
Risk: Any CA compromise = all sites vulnerable
```

---

### Solution: Public Key Pinning

```
App hardcodes:
  "example.com must use public key: ABC123..."

Even if fake certificate signed by legitimate CA:
  Public key is different
  App rejects (public key mismatch!)
  Attacker blocked

Implementation:
  Store hash of public key in app
  Verify received certificate's public key matches
  
Trade-off:
  Security: Better (CA compromise doesn't matter)
  Maintenance: Need to update app when key rotates
  Expiration: Must update before key expires!
```

---

## Key Management

### Key Rotation

```
Annual rotation:

Year 1:
  Generate key_v1 + certificate_v1
  Use for HTTPS

Year 2:
  Generate key_v2 + certificate_v2
  Start using key_v2
  Keep key_v1 for decryption (might need to decrypt old sessions)

Year 3:
  Can retire key_v1 (most sessions expired)

Year 4:
  key_v1 completely retired

Benefit:
  If key_v1 compromised: Only affects Year 1 data
  Not retroactive (Year 2+ unaffected)
```

---

### Key Separation

```
One private key = one purpose

Bad:
  server_key: Used for TLS, signing, auth, encryption
  If compromised: All systems affected

Good:
  tls_key: For HTTPS handshakes
  signing_key: For signing software/code
  auth_key: For API authentication
  encryption_key: For data encryption
  
Rotation independent:
  Compromise of tls_key doesn't affect auth_key
  Can rotate one without rotating all
```

---

## mTLS (Mutual TLS)

### Verification Both Directions

```
Standard TLS:
  Client verifies: "Is this really the server?"
  Server doesn't verify client
  
mTLS:
  Client verifies: "Is this really the server?"
  Server verifies: "Is this really the authorized client?"

Implementation:
  Client provides certificate (proving identity)
  Server checks: Is certificate in trusted list?
  
Use:
  Microservices (service-to-service auth)
  Internal APIs
  High-security systems
```

---

## Compliance & Standards

### FIPS 140-2

```
US government standard for cryptographic modules

Requirements:
  Approved algorithms (AES, RSA, ECDSA)
  Key generation in secure environment
  Secure key storage (hardware security modules)
  
When needed: Government contracts, financial institutions
```

---

### CAA Records

```
DNS record that specifies: Which CAs can issue certificates for this domain

Example:
  example.com CAA 0 issue "letsencrypt.org"
  
Effect:
  Only Let's Encrypt can issue certificates for example.com
  Other CAs blocked (even if they wanted to)
  
Protection:
  If attacker gets access to other CA
  Can't issue fake certificates for your domain
  Extra validation layer
```

---

## Production Checklist

- [ ] Use TLS 1.3 minimum
- [ ] Strong key: RSA-2048+ or ECDSA-256+
- [ ] Certificate chain complete (all intermediates included)
- [ ] OCSP stapling enabled
- [ ] CAA records configured (lock down CAs)
- [ ] Annual key rotation
- [ ] Keep private keys in hardware security modules (HSM)
- [ ] Monitor certificate expiration
- [ ] Pin public keys (if high security needed)
- [ ] mTLS for service-to-service communication

---

## Related Fundamentals

- [TLS & Certificates](../networking-and-protocols/tls-and-certificates.md) – Practical TLS setup
- [Authentication](authentication-and-authorization.md) – Identity verification
- [Data Protection](data-protection-and-encryption.md) – Encryption at rest

---

**Status**: ✅ Complete. Covers PKI, CAs, pinning, mTLS, key management.

