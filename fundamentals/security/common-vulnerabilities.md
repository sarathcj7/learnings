# Common Vulnerabilities & Prevention

OWASP Top 10 vulnerabilities and how to avoid them. Every developer should know these.

---

## TL;DR

- **SQL Injection**: Parameterized queries prevent code execution
- **XSS**: Sanitize/escape user input, Content Security Policy
- **CSRF**: SameSite cookies, CSRF tokens
- **Authentication**: Secure password storage (bcrypt), rate limiting
- **Sensitive data**: Encrypt, don't log, use HTTPS
- **XML External Entities (XXE)**: Disable external entities
- **Broken Access Control**: Check permissions on every operation

---

## 1. SQL Injection

### Vulnerable Code

```python
❌ Bad:
  query = f"SELECT * FROM users WHERE email = '{email}'"
  result = db.execute(query)
  
Attack:
  email = "admin@example.com' OR '1'='1"
  Query becomes: SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
  Returns: All users (broken logic!)
  
Worse attack:
  email = "admin'; DROP TABLE users; --"
  Query becomes: SELECT * FROM users WHERE email = 'admin'; DROP TABLE users; --'
  Deletes entire table!
```

---

### Secure Code

```python
✅ Good (Parameterized query):
  query = "SELECT * FROM users WHERE email = ?"
  result = db.execute(query, [email])
  
  Database driver:
    1. Parses query template
    2. Keeps placeholders separate from parameters
    3. Prevents SQL injection (no string concatenation)
  
Benefit:
  email = "admin@example.com' OR '1'='1"
  Treated as single literal string
  Returns: Only user with that exact email
```

---

## 2. Cross-Site Scripting (XSS)

### Reflected XSS

```html
❌ Bad:
  HTML: <h1>Search results for {{ query }}</h1>
  Template: Renders query directly
  
  URL: /search?query=<script>alert('hacked')</script>
  Rendered: <h1>Search results for <script>alert('hacked')</script></h1>
  Browser executes script!
  
  Attack: Steal cookies, redirect to phishing, change DOM
```

---

### Stored XSS

```javascript
❌ Bad:
  User posts comment: "<img src=x onerror='stealCookies()'>"
  Comment stored in database
  Other users load comment
  Script executes for all users!
```

---

### Prevention

```javascript
✅ Good (Escape output):
  query = "<script>alert('xss')</script>"
  escaped = htmlEscape(query)  // "&lt;script&gt;alert('xss')&lt;/script&gt;"
  
  HTML: <h1>Search results for {{ escaped }}</h1>
  Rendered: <h1>Search results for &lt;script&gt;alert('xss')&lt;/script&gt;</h1>
  Browser displays: Search results for <script>alert('xss')</script>
  
  Script tag not executed (rendered as text)

✅ Good (Content Security Policy):
  Response header: Content-Security-Policy: script-src 'self'
  
  Effect:
    Only scripts from same origin allowed
    Inline scripts blocked (unless whitelisted)
    Prevents most XSS attacks
```

---

## 3. CSRF (Cross-Site Request Forgery)

### Attack

```
User logged into bank.com
User visits evil.com
evil.com has: <img src="https://bank.com/transfer?amount=1000&to=attacker">
Browser sends request with bank.com cookies!
Transfer happens!
```

---

### Prevention 1: SameSite Cookies

```
Set-Cookie: sessionid=abc; SameSite=Strict

Levels:
  Strict: Cookie never sent to cross-site requests
  Lax: Cookie sent only for safe methods (GET), not POST
  None: Cookie sent always (requires Secure flag)

Result:
  evil.com image request to bank.com: Cookie not sent
  Bank requests user to login (no valid session)
  Transfer blocked
```

---

### Prevention 2: CSRF Token

```
Form:
  <form method="POST" action="/transfer">
    <input type="hidden" name="csrf_token" value="random_value_abc123">
    <input type="text" name="amount">
    <button>Transfer</button>
  </form>

Server validation:
  1. User views form: Server generates token, stores in session
  2. Form submitted: Server checks token in body matches session token
  3. If token missing or wrong: Reject request

Result:
  evil.com can't submit form (doesn't have token)
  Requires server to have generated token first
```

---

## 4. Weak Authentication

### Vulnerable

```
✖ Passwords:
  ✗ Weak policy (no minimum length)
  ✗ No rate limiting (10000 attempts/sec)
  ✗ Stored plaintext in database
  ✗ Reused across services
  ✗ No multi-factor authentication
  
Result:
  Brute force succeeds
  Plaintext leaked → All systems compromised
```

---

### Secure

```
✓ Passwords:
  ✓ Minimum 12 characters
  ✓ Rate limiting (5 attempts, then 15-min lockout)
  ✓ Bcrypt hashed (cost=12, auto-salted)
  ✓ Unique per service
  ✓ Multi-factor auth (TOTP, hardware key)
  ✓ Compromised password detection (PWned DB check)
  
Result:
  Brute force fails (rate limited)
  Hash leaked → Attacker can't recover passwords (slow hash)
  Leaked creds detected → Force password reset
```

---

## 5. Sensitive Data Exposure

### What Not To Do

```
❌ Log passwords:
  logger.info(f"User login: {username} / {password}")
  If logs leaked: Passwords compromised

❌ Log credit cards:
  logger.info(f"Payment: {card_number}")
  PCI-DSS violation
  Thousands of dollars in fines

❌ Send PII unencrypted:
  Send API call: GET /api/users/123 → { ssn: "123-45-6789" }
  If intercepted: PII compromised

❌ Store secrets in code:
  api_key = "sk_live_1234567890"  (in source code)
  GitHub scans code → Key found → Attacker uses it
```

---

### What To Do

```
✅ Never log sensitive data:
  logger.info(f"User login: {username}")  (no password)

✅ Mask in logs:
  logger.info(f"Card: ****-****-****-{card[-4:]}")

✅ Encrypt PII at rest and in transit:
  GET /api/users/123 (over HTTPS)
  Response encrypted in database
  Key stored in KMS

✅ Use environment variables for secrets:
  export API_KEY="sk_live_1234567890"
  code: api_key = os.getenv("API_KEY")
  
  OR use secrets manager:
  AWS Secrets Manager, Vault, etc.
```

---

## 6. XML External Entities (XXE)

### Attack

```xml
❌ Bad XML parser (external entities enabled):

Input:
  <?xml version="1.0"?>
  <!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
  <root>&xxe;</root>

Parser:
  1. Sees external entity reference
  2. Loads file:///etc/passwd
  3. Returns file contents
  
Result:
  Attacker reads /etc/passwd (sensitive file!)
  Can read AWS credentials, database passwords, etc.
```

---

### Prevention

```python
✅ Good (Disable external entities):
  import xml.etree.ElementTree as ET
  
  # Disable external entities
  parser = ET.XMLParser()
  parser.entity['file'] = None  # Disable file:// URIs
  
  tree = ET.parse('input.xml', parser=parser)
  
  Result:
    Parser refuses to load external entities
    Attack fails
```

---

## 7. Broken Access Control

### Vulnerable

```python
❌ Bad:
  GET /api/users/456
  
  Server (trusts client):
    User 123 requesting: Return user 456's data
    (No check: Does user 123 have access to user 456?)
    
  Response: User 456's email, phone, address (PII!)
  
  Attack: Change URL to 789, 790, 791, ... → Access all users!
```

---

### Secure

```python
✅ Good (Check permissions):
  GET /api/users/456
  
  Server:
    1. Get current user: 123 (from session/token)
    2. Get requested user: 456
    3. Check: Does 123 have access to 456?
       If 123 == 456: Yes (own data)
       If 123.role == 'admin': Yes (admins see all)
       Else: No
    4. If no access: Return 403 Forbidden
    5. If access: Return data
  
  Result:
    User 123 can only access own data
    User 123 can't enumerate other users
    Privilege escalation prevented
```

---

## OWASP Top 10 Quick Reference

| Rank | Vulnerability | Prevention |
|---|---|---|
| 1 | Broken Access Control | Check permissions on every operation |
| 2 | Cryptographic Failures | Encrypt at rest/transit, secure key storage |
| 3 | Injection | Parameterized queries, escaping |
| 4 | Insecure Design | Security by design, threat modeling |
| 5 | Security Misconfiguration | Secure defaults, minimal software |
| 6 | Vulnerable Components | Keep dependencies updated |
| 7 | Auth Failures | Strong passwords, MFA, rate limiting |
| 8 | Data Integrity Issues | Verify data authenticity (cryptographic signatures) |
| 9 | Logging Failures | Log security events, audit trails |
| 10 | SSRF | Validate URLs, block internal IPs |

---

## Security Checklist

- [ ] Parameterized queries (no SQL injection)
- [ ] Input validation/escaping (no XSS)
- [ ] CSRF tokens or SameSite cookies
- [ ] Strong password policy (min 12 chars, complexity)
- [ ] Rate limiting on login (prevent brute force)
- [ ] Bcrypt for password hashing
- [ ] No sensitive data in logs
- [ ] HTTPS everywhere (TLS 1.2+)
- [ ] XXE disabled in XML parsers
- [ ] Access control checked on every operation
- [ ] Dependencies up-to-date (security patches)
- [ ] Security headers (CSP, X-Frame-Options, etc.)

---

## Related Fundamentals

- [Authentication](authentication-and-authorization.md) – Secure login
- [Data Protection](data-protection-and-encryption.md) – Encryption, compliance
- [Observability/Audit](../observability/) – Security event logging

---

**Status**: ✅ Complete. Covers top vulnerabilities, OWASP Top 10, prevention patterns.

