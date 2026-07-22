# Security

Protecting systems from attacks: who can access what (access control), securing data (encryption), and avoiding common vulnerabilities. Non-negotiable for production systems.

## Sub-topics

- **[Authentication & Authorization](authentication-and-authorization.md)** ✅: Session-based, JWT, OAuth 2.0, RBAC/ABAC, password security
- **[Data Protection & Encryption](data-protection-and-encryption.md)** ✅: Encryption at rest/transit, TLS, key management, PII, compliance (GDPR/HIPAA/PCI)
- **[Common Vulnerabilities](common-vulnerabilities.md)** ✅: SQL injection, XSS, CSRF, broken access control, OWASP Top 10
- **[TLS & PKI](tls-and-pki.md)** ✅: CAs, certificate chains, pinning, key management, mTLS

## Why This Matters

- **Regulatory**: GDPR fines up to $20M, PCI-DSS required for payments
- **Reputation**: Security breach destroys trust (Equifax, OPM)
- **Operational**: Every developer should know these vulnerabilities
- **Interview**: Security appears in design interviews, shows thoughtfulness
- **Scale**: Security patterns don't degrade performance (modern crypto is fast)

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/security.md)**: 30+ security questions
- **[Flashcards](../../interview-prep/flashcards/security.md)**: Vulnerabilities, prevention patterns

## Related Case Studies

- [Payment System](../../case-studies/payment-and-ledger-system.md) – ACID, audit trails, PCI compliance
- [API Gateway](../../case-studies/api-gateway-design.md) – Auth, rate limiting, security headers
- [Chat System](../../case-studies/chat-and-messaging-system.md) – Message encryption, access control

## Related Fundamentals

- [APIs & Communication](../apis-and-communication/idempotency-and-api-design.md) – Secure API design
- [Reliability](../reliability-and-resiliency/) – Audit logging
- [Observability](../observability/) – Security event detection

## Study Tips

1. **Start with OWASP Top 10**: Know SQL injection, XSS, CSRF, broken access control
2. **Password security**: Always use bcrypt/Argon2, never plaintext
3. **Encryption matters**: At rest (database), in transit (TLS), in logs (redact PII)
4. **Key management**: Separate keys for different purposes, rotate regularly
5. **Access control**: Check permissions on EVERY operation (not just login)

---

**Status**: ✅ **COMPLETE**. 4/4 files written.
