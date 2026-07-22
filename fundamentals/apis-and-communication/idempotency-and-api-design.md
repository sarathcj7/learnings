# Idempotency & Safe API Design

Designing APIs that can be retried safely, preventing duplicate charges and state inconsistencies. Critical for distributed systems.

---

## TL;DR

- **Idempotent operation**: Can be executed multiple times without side effects
- **Idempotency key**: Unique ID tracking which operations were already processed
- **Safe**: GET, HEAD, OPTIONS (read-only, safe to retry)
- **Unsafe**: POST, PUT, PATCH, DELETE (modify state, need idempotency keys)
- **Importance**: Enable retries without duplicating actions (charge twice, create twice, etc.)

---

## The Problem: Network Failures

### Transfer Money Example

```
Client: POST /transfer { from: 100, to: 200, amount: 50 }

Server:
  1. Debit account 100: $500 → $450
  2. Credit account 200: $100 → $150
  3. Return response
  
Network fails after step 2 (before response sent):
  Client: Received no response (timeout)
  Client: Retries request
  
Server receives retry:
  1. Debit account 100: $450 → $400 (OOPS! Second debit!)
  2. Credit account 200: $150 → $200
  3. Return response
  
Result: $50 transferred twice! Account 100 lost $100!
```

---

## Solution 1: Idempotency Keys

Unique ID for each operation, prevents duplicate execution:

```
Request 1:
  POST /transfer
  Idempotency-Key: "txn_abc123"
  Body: { from: 100, to: 200, amount: 50 }
  
Server:
  1. Check if key "txn_abc123" already processed
     → No, continue
  2. Debit account 100
  3. Credit account 200
  4. Store result: txn_abc123 → { from: 100, to: 200, result: success }
  5. Return response

Request 2 (retry):
  Same Idempotency-Key: "txn_abc123"
  Same body
  
Server:
  1. Check if key "txn_abc123" already processed
     → Yes! Return cached result
  2. Don't re-execute transfer
  3. Return stored response
  
Result: Second request returns same response, no duplicate charge!
```

---

## Idempotency Key Implementation

### Storage

```
Store mapping:
  Idempotency-Key → Operation result

Example:
  txn_abc123 → {
    operation: "transfer",
    status: "success",
    result: { txn_id: 999, amount: 50 },
    timestamp: 2026-07-23T10:00:00Z
  }

Retention:
  Keep for 24 hours minimum (covers retry window)
  Clean up after 7 days (save storage)
```

---

### API Design

```
Request header:
  Idempotency-Key: "txn_abc123"
  (client generates, typically UUID)

Response header:
  Idempotency-Key: "txn_abc123"
  (server echoes, confirms received)
  Idempotency-Replay: false (first time) or true (replay)

Status codes:
  200 OK: Success
  202 Accepted: Still processing (async)
  409 Conflict: Different request body with same key
```

---

## Safe API Methods

### GET (Idempotent by Default)

```
GET /users/123

First request: Returns user data
Second request: Returns same user data
Thousandth request: Returns same user data

Always safe, no side effects, can retry freely
```

---

### POST (Unsafe Without Idempotency Key)

```
POST /users { name: "Alice" }

First request: Creates user, returns 201
Second request (retry): Creates ANOTHER user → Two users!

Solution: Idempotency key
POST /users 
Idempotency-Key: user_alice_123
{ name: "Alice" }

First request: Creates user
Second request: Returns existing user (no duplicate)
```

---

### PUT (Idempotent if Implemented Correctly)

```
PUT /users/123 { name: "Bob" }

Implementation 1 (replace):
  First request: User 123 → name: "Bob"
  Second request: User 123 → name: "Bob" (same)
  ✓ Idempotent (same result both times)

Implementation 2 (buggy, increment age):
  PUT /users/123/age { increment: 1 }
  First request: age 30 → age 31
  Second request: age 31 → age 32 (different!)
  ✗ Not idempotent (different result)
```

---

### DELETE (Edge Cases)

```
DELETE /users/123

First request: User deleted, return 204 No Content
Second request (retry): User already deleted
  Option A: Return 404 (user not found)
  Option B: Return 204 (pretend success)
  
Better: Option B (404 + 204 = client confused)

Implementation:
  DELETE /users/123 with Idempotency-Key
  First: Delete, store result
  Retry: Return 204 (same response as first)
  Client sees: Always 204, retries work fine
```

---

## Patterns for Making Operations Idempotent

### Pattern 1: State Tracking

```
Operation: "Activate user account"

Query user status:
  If status == "ACTIVE" → Already activated, return success
  If status == "INACTIVE" → Activate now, update status
  
Result: No matter how many times called, end state is ACTIVE
```

---

### Pattern 2: Unique Constraints

```
Database constraint:
  CREATE UNIQUE INDEX email ON users(email)

Operation: "Create user with email alice@example.com"

First request: Insert → Success, user created
Retry: Insert → Constraint violation, reject
  But: Return 409 (conflict) + existing user data
  Client gets: "User already exists, here's the user"

Better: Add idempotency key to avoid 409 conflict
```

---

### Pattern 3: Transactional Writes

```
Operation: Multiple related updates

Transaction:
  BEGIN
  UPDATE account A: -$50
  UPDATE account B: +$50
  DELETE pending_transfer id 123
  COMMIT

If any step fails: Entire transaction rolls back
  Retry: All or nothing, no partial states
  
Result: Idempotent by nature (ACID guarantees)
```

---

## Distributed Idempotency

### Problem: Multiple Servers

```
Request: POST /transfer with Idempotency-Key: "txn_abc123"

Server 1 gets request:
  Checks idempotency key store → Not found
  Executes transfer
  Stores result in Redis
  Returns response
  
Retry reaches Server 2:
  Checks idempotency key store → Also Redis (shared)
  Finds key "txn_abc123" → Already processed
  Returns cached result
  
Result: Idempotency works across servers!
```

---

### Idempotency Key Storage

```
Redis:
  SET idempotency:txn_abc123 { status: "success", result: {...} } EX 86400
  Fast, distributed, TTL auto-cleanup

Database:
  INSERT INTO idempotency_keys (key, result) VALUES (...)
  UNIQUE constraint prevents duplicates
  Durable, but slower

Recommendation:
  Redis for fast path
  Database for durability
  (Both if critical operations)
```

---

## When Idempotency Keys Not Needed

```
Safe operations (don't modify state):
  ✓ GET /users/123
  ✓ GET /posts/456
  ✓ HEAD /files/789
  
Async operations (tracked externally):
  ✓ POST /batch_jobs (job ID returned, idempotency tracked by job system)
  
Read-only (client-side deduplication OK):
  ✓ GET /search?q=...
```

---

## API Version Evolution

### Problem: Changing APIs

```
Version 1:
  POST /users { name, email }
  
Version 2 (added phone):
  POST /users { name, email, phone }
  
Old client sends v1 request → Server might reject (missing phone)
New client sends v2 request → Old server might fail (unknown field)
```

---

### Solution 1: URL Versioning

```
V1: POST /v1/users { name, email }
V2: POST /v2/users { name, email, phone }

Pros: Clear versions, easy to support multiple
Cons: More code duplication
```

---

### Solution 2: Backward Compatibility

```
Server always accepts both:
  POST /users { name, email, phone? }
  
If phone provided: Use it
If phone omitted: Use default or skip

Result: One endpoint, both versions work
```

---

## API Design Checklist

- [ ] Non-idempotent operations have Idempotency-Key header support
- [ ] Idempotency keys stored (Redis + DB for critical operations)
- [ ] Retry-safe implementations (no side effects on retry)
- [ ] Proper HTTP status codes (409 for conflicts, 202 for async)
- [ ] API versioning strategy documented
- [ ] Backward compatibility maintained
- [ ] Client can generate unique idempotency keys (UUID)
- [ ] Idempotency storage retention policy (24h minimum)

---

## Related Fundamentals

- [Retries & Backoff](../reliability-and-resiliency/retries-and-exponential-backoff.md) – Idempotency enables safe retries
- [Distributed Transactions](../databases/transactions-and-isolation-levels.md) – ACID makes idempotent
- [APIs/REST vs GraphQL](rest-vs-graphql.md) – Idempotency at API level

---

**Status**: ✅ Complete. Covers idempotency keys, patterns, storage, distributed systems.

