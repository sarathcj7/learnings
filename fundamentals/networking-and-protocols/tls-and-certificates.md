# TLS & Certificates

Securing communication with HTTPS. Certificate management, handshakes, and production considerations.

---

## TL;DR

- **TLS 1.3**: Modern standard (0-RTT resumption, faster)
- **Certificate**: Public key + identity proof (signed by CA)
- **Chain of trust**: Browser verifies cert → intermediate → root CA
- **ACME**: Automated certificate issuance (Let's Encrypt)
- **Pinning**: Pin certificate to prevent MITM attacks

---

## TLS Handshake

### Full Handshake (First Connection)

```
Client → Server: ClientHello
  - TLS version (1.3)
  - Supported ciphers
  - Extensions (SNI, supported groups)

Server → Client: ServerHello + Certificate + CertificateVerify
  - Chosen cipher
  - Server certificate
  - Signed handshake proof

Client → Server: Finished
  - Client sends MAC over all handshake messages
  - Server verifies
  
Encrypted connection established
Time: ~100ms (1 RTT) + TLS processing
```

---

### TLS 1.3 Improvements

```
TLS 1.2: 2 RTTs (ClientHello → ServerHello, then key exchange)
TLS 1.3: 1 RTT (key exchange in ClientHello itself)

TLS 1.3 0-RTT:
  Client has previous session key
  Sends encrypted data immediately
  Server decrypts with remembered key
  
Result: Connection + data in same packet (0 RTT)
Risk: Replay attacks (send same request twice)
Mitigation: Stateless server, sequence numbers
```

---

## Certificates

### Structure

```
Certificate:
  Subject: CN=api.example.com
           OU=Engineering
           O=Example Corp
           C=US
  
  Issuer: CN=Let's Encrypt Authority X3
          O=Let's Encrypt
          C=US
  
  Public Key: RSA-2048 (or ECDSA-256)
  
  Valid From: 2026-01-01
  Valid Until: 2027-01-01
  
  Extensions:
    Subject Alternative Name (SAN):
      - api.example.com
      - *.example.com
      - other.example.com
  
  Signature: (signed by issuer's private key)
```

---

### Certificate Chain

```
Browser trusts: Root CA (embedded in OS/browser)

Server certificate:
  Signed by: Intermediate CA
  
Intermediate CA certificate:
  Signed by: Root CA (in browser trust store)

Verification:
  1. Browser has Root CA public key
  2. Use Root's key to verify Intermediate's signature ✓
  3. Use Intermediate's key to verify Server's signature ✓
  4. Server certificate is valid!

Why chain?
  - Root CA kept offline (security)
  - Intermediate can be revoked without affecting root
  - Delegation of signing authority
```

---

## Certificate Types

### Domain Validation (DV)

```
Cheapest, fastest (minutes to hours)

Process:
  1. Provide domain name
  2. CA verifies you own domain (DNS TXT record or HTTP challenge)
  3. Certificate issued
  
Security: Proves domain ownership only, not organization identity
Use: Most websites, internal services
Cost: Free (Let's Encrypt) to $10/year
```

---

### Organization Validation (OV)

```
More expensive ($50-200/year)

Process:
  1. Provide domain + company info
  2. CA verifies domain + organization details
  3. Certificate issued
  
Security: Proves domain + organization ownership
Use: e-commerce, financial services
Appearance: Browser shows company name on cert
```

---

### Extended Validation (EV)

```
Most expensive ($200-500/year)

Process:
  1. Heavy verification: domain, company, legal status
  2. Phone calls, document verification
  3. Takes weeks

Security: Highest assurance
Use: Banks, high-security services
Appearance: Browser shows green bar (newer browsers removed this)
```

---

## Certificate Management

### Let's Encrypt (Automated)

```
Free, automated, 90-day certificates

Process:
  1. Install certbot
  2. Run: certbot certonly -d example.com
  3. Complete ACME challenge (DNS or HTTP)
  4. Certificate issued + auto-renewed
  
Renewal: Automatic (before expiration)
Cost: Free
Use: Production standard for most websites
```

---

### Self-Signed (Development Only)

```
NOT FOR PRODUCTION

Process:
  openssl req -x509 -newkey rsa:2048 -out cert.pem -keyout key.pem

No external validation (signed by yourself)
Browsers warn "untrusted"
Use: Local testing, internal tools only
```

---

## Common Issues

### Certificate Expiration

```
❌ Expired certificate:
  Browsers show: "Your connection is not private"
  Users can't connect

Solution:
  Monitor expiration date
  Automate renewal (Let's Encrypt does this)
  Alert on 30 days before expiry
```

---

### SAN (Subject Alternative Name)

```
Certificate for: example.com
SAN missing: www.example.com

Result: www.example.com rejected (different domain)

Correct:
  Main: example.com
  SAN: www.example.com, *.example.com, api.example.com
  
One cert works for all domains!
```

---

### Wildcard Certificates

```
Standard:
  example.com
  www.example.com
  api.example.com
  (3 separate certs if not SAN)

Wildcard:
  *.example.com covers:
  - www.example.com ✓
  - api.example.com ✓
  - mail.example.com ✓
  - anything.example.com ✓
  
NOT: subdomain.sub.example.com (multi-level)
```

---

## Production Checklist

- [ ] TLS 1.2+ only (no SSLv3, TLS 1.0, 1.1)
- [ ] Strong ciphers (AEAD: AES-256-GCM, ChaCha20)
- [ ] HTTPS everywhere (redirect HTTP → HTTPS)
- [ ] HSTS header (force HTTPS for future visits)
- [ ] Certificate monitoring (expiration alerts)
- [ ] Automated renewal (Let's Encrypt + certbot)
- [ ] Certificate pinning (if high-security needed)
- [ ] Proper certificate chain (all intermediate certs included)
- [ ] OCSP stapling (revocation checking)
- [ ] SNI enabled (multiple certs per IP)

---

## Related Fundamentals

- [Authentication](../security/authentication-and-authorization.md) – Secure identity
- [HTTP/2 & HTTP/3](http-and-versions.md) – ALPN negotiation
- [Security](../security/) – Encryption & compliance

---

**Status**: ✅ Complete. Covers handshakes, certificates, chains, management.

