# Authentication & Authorization

Who are you (auth) vs what can you do (authz). Essential for secure systems.

---

## TL;DR

- **Authentication**: Verify identity (who are you?)
- **Authorization**: Check permissions (what can you do?)
- **Session**: Track authenticated user across requests
- **OAuth 2.0**: Delegate auth to trusted provider (Google, GitHub)
- **JWT**: Stateless token (no server-side session storage)
- **mTLS**: Mutual TLS for service-to-service

---

## Authentication Methods

### Basic Auth (Insecure)

```
HTTP request:
  Authorization: Basic base64(username:password)
  Example: Authorization: Basic dXNlcjpwYXNz

Problems:
  ❌ Password in every request (credential reuse risk)
  ❌ No encryption (must use HTTPS, but often doesn't)
  ❌ Password stored in client memory (vulnerable to theft)
  
Use only:
  Internal tools (behind firewall)
  Automated scripts (no better option)
```

---

### Session-Based (Traditional)

```
1. User POST /login { username, password }
2. Server validates
3. Server creates session (stores in memory or database)
4. Server sends session ID in cookie
5. Client stores cookie
6. On future requests: Cookie sent automatically
7. Server looks up session, verifies

Pros:
  ✅ Simple
  ✅ Can revoke instantly (delete session)
  
Cons:
  ❌ Server must store sessions (not stateless)
  ❌ Doesn't scale across multiple servers (need shared session store)
  ❌ CSRF risk if not careful (cookie sent with any request)
```

---

### JWT (JSON Web Token)

```
Token structure:
  header.payload.signature
  
Example:
  eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
  eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
  SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
  
Decoded payload:
  {
    sub: "1234567890",  // Subject (user ID)
    name: "John Doe",
    iat: 1516239022     // Issued at (timestamp)
  }

Flow:
  1. User POST /login { username, password }
  2. Server validates
  3. Server generates JWT (signed with secret key)
  4. Server sends JWT to client
  5. Client stores JWT (localStorage, sessionStorage)
  6. On future requests: JWT in Authorization header
  7. Server verifies signature (no database lookup needed!)
  
Pros:
  ✅ Stateless (no session storage on server)
  ✅ Scales (every server can verify same JWT)
  ✅ Mobile-friendly (no cookies)
  
Cons:
  ❌ Can't instantly revoke (token valid until expiry)
  ❌ Large token size (security info encoded)
  ❌ Must secure secret key (if leaked, all JWTs compromised)
```

---

### OAuth 2.0 (Delegated Auth)

```
User clicks: "Login with Google"

Flow:
  1. App redirects to: google.com/oauth/authorize?client_id=...
  2. User logs in with Google (not with your app!)
  3. Google redirects back: your-app.com/callback?code=abc123
  4. Your app (backend) exchanges code for access token
  5. Your app uses token to get user info from Google
  6. Your app creates user in system
  7. Your app issues JWT or session
  
Advantage:
  ✅ User never gives password to your app
  ✅ Google handles password security
  ✅ User can manage permissions (revoke access)
  ✅ Works with multiple providers (Google, GitHub, Facebook)
```

---

## Authorization (What Can You Do?)

### Role-Based Access Control (RBAC)

```
User roles:
  Admin: Can do everything
  User: Can read own data, write own data
  Guest: Can only read public data

Implementation:
  User record:
    id: 123
    username: "alice"
    role: "admin"
  
  Request:
    GET /users/456
    
  Check:
    Is requester.role == "admin"? → Allow
    Is requester.id == 456? → Allow (own data)
    Else → Deny 403 Forbidden
```

---

### Attribute-Based Access Control (ABAC)

```
More flexible than RBAC:

Policy:
  "Allow if user.department == 'finance' AND resource.type == 'budget'"
  
Evaluation:
  User attributes: { department: "finance", level: 5 }
  Resource attributes: { type: "budget", confidentiality: "high" }
  Action: read
  
  Match: department matches, so allow (but check confidentiality too)
  Result: Access denied (because confidentiality: high requires level >= 10)
```

---

## Secure Password Storage

### Hashing

```
❌ Bad: Store plaintext password
  database.users.password = "password123"
  If database leaked: All passwords compromised

❌ Bad: Simple hash
  password_hash = SHA256("password123")
  Problem: Rainbow tables (pre-computed hashes)

✅ Good: Salted hash
  salt = random_bytes(16)
  hash = PBKDF2("password123", salt, iterations=100000)
  Store: { salt, hash }
  
  On login:
    Retrieve salt from database
    Compute hash = PBKDF2(entered_password, salt, iterations=100000)
    Compare with stored hash
    
  Benefit: Rainbow tables useless (each password salted differently)
```

---

### Best Practice: Bcrypt

```
Use bcrypt (or Argon2):
  bcrypt("password123", cost=12)
  → $2b$12$R9h/cIPz0gi.URNNGNVB2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUW
  
Advantages:
  ✅ Automatically salts
  ✅ Key-stretching (slow by design)
  ✅ Cost parameter (can increase as computers get faster)
  ✅ Hard to crack (intentionally slow)
```

---

## Session Management

### Session Fixation Prevention

```
❌ Vulnerable:
  1. User visits login page
  2. Server assigns session: session_id=abc123
  3. User enters credentials
  4. Server validates, stores user in same session
  5. User has session_id=abc123

Attacker could:
  1. Create session_id=abc123 on their browser
  2. Trick user into using that session
  3. User logs in, attacker's browser now authenticated!

✅ Secure:
  1. User visits login page (no session created)
  2. User submits credentials
  3. Server creates NEW session_id=xyz789
  4. Server invalidates any old session
  5. Result: Previous session unusable
```

---

### CSRF (Cross-Site Request Forgery) Prevention

```
❌ Vulnerable:
  User logged into facebook.com
  User visits evil.com
  evil.com has: <img src="https://facebook.com/transfer?amount=1000&to=attacker">
  Browser sends request with facebook cookie!
  Money transferred!

✅ Prevention 1: SameSite Cookie
  Set-Cookie: sessionid=abc; SameSite=Strict
  Cookie not sent to cross-site requests
  
✅ Prevention 2: CSRF Token
  Form includes: <input type="hidden" name="csrf_token" value="random_token">
  Server validates: Token must match stored value
  evil.com doesn't have token, can't submit form
```

---

## mTLS (Mutual TLS)

For service-to-service authentication:

```
Service A (client) → Service B (server)

Traditional TLS:
  Service A verifies: Is Service B who it claims?
  Service A doesn't authenticate to Service B
  
mTLS:
  Service A: Presents certificate (proves "I'm Service A")
  Service B: Verifies certificate (checks signature chain)
  Service B: Presents certificate (proves "I'm Service B")
  Service A: Verifies certificate
  
Result: Mutual authentication (both sides know who they're talking to)

Implementation:
  Each service has:
    Private key (secret.key)
    Certificate (cert.pem) signed by trusted CA
  
  Service A config:
    client_cert: /etc/certs/service-a.pem
    client_key: /etc/certs/service-a-key.pem
    ca_cert: /etc/certs/ca.pem (to verify Service B)
```

---

## Authentication Checklist

- [ ] Never send passwords in query parameters (use POST + HTTPS)
- [ ] Use HTTPS everywhere (no plaintext auth)
- [ ] Hash passwords with bcrypt/Argon2 (never plaintext)
- [ ] Implement token expiration (JWT: max 1 hour, refresh: 7 days)
- [ ] Revoke sessions on logout
- [ ] Prevent session fixation (regenerate session on login)
- [ ] CSRF protection (SameSite cookies or CSRF tokens)
- [ ] Rate limit login attempts (prevent brute force)
- [ ] Log authentication events (audit trail)
- [ ] mTLS for service-to-service communication

---

## Related Fundamentals

- [TLS/PKI](tls-and-certificates.md) – Cryptographic foundation
- [API Design](../apis-and-communication/idempotency-and-api-design.md) – Secure API patterns
- [Observability](../observability/) – Audit logging

---

**Status**: ✅ Complete. Covers auth methods, session, JWT, OAuth, authorization, password security.

