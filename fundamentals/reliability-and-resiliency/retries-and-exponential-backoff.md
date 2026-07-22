# Retries & Exponential Backoff

When to retry failed requests, how to avoid overwhelming a struggling service, and the math behind backoff algorithms.

---

## TL;DR

- **Retries**: Transient failures (network hiccup) often succeed on retry
- **Exponential backoff**: Wait longer between retries (1s, 2s, 4s, 8s...) to give service time to recover
- **Jitter**: Add randomness to stagger retries, prevent thundering herd
- **Idempotency**: Essential—retrying non-idempotent operations can duplicate actions (charge twice!)
- **Max retries**: Typically 3-5, cap the wait time at 30-60 seconds

---

## Transient vs Permanent Failures

### Transient (Retry)

Temporary failure, likely to succeed on retry:

```
Network hiccup: 1s connection dropped
  → Retry after 1s: Success

Service overloaded: 10s spike in CPU
  → Request times out
  → Retry after 2s: Service recovered, Success

Database connection pool exhausted momentarily
  → Connection timeout
  → Retry after 1s: Pool drained, Success
```

**HTTP status codes**:
- 429 Too Many Requests (rate limit)
- 503 Service Unavailable (temporary)
- 504 Gateway Timeout

---

### Permanent (Don't Retry)

Failure won't be fixed by waiting:

```
Invalid request: malformed JSON
  → Retry 100 times: Still malformed
  
Authentication failed: wrong API key
  → Retry 100 times: Still wrong key
  
Service moved: endpoint doesn't exist (404)
  → Retry 100 times: Still doesn't exist
```

**HTTP status codes**:
- 400 Bad Request (malformed)
- 401 Unauthorized (auth failed)
- 403 Forbidden (permission denied)
- 404 Not Found (resource missing)

---

## Linear Retry (Bad)

```
Attempt 1: Fail @ 0s
Attempt 2: Fail @ 1s (wait 1s)
Attempt 3: Fail @ 2s (wait 1s)
Attempt 4: Fail @ 3s (wait 1s)
Attempt 5: Fail @ 4s (wait 1s)

Total time: 4 seconds

Problem: Service under high load
  Receiving 1000 retries/sec continuously
  Never gets time to recover!
```

---

## Exponential Backoff (Better)

```
Attempt 1: Fail @ 0s
Attempt 2: Fail @ 1s (wait 1s = 2^0)
Attempt 3: Fail @ 3s (wait 2s = 2^1)
Attempt 4: Fail @ 7s (wait 4s = 2^2)
Attempt 5: Fail @ 15s (wait 8s = 2^3)

Total time: 15 seconds

Formula:
  wait_time = base * 2^attempt
  base = 1 second
  attempt = 0, 1, 2, 3, ...
```

**Effect**: Requests spread out, service gets breathing room to recover.

---

## Exponential Backoff with Jitter

Problem with pure exponential:

```
Scenario: 1000 clients all hit service, all get 503
  All retry @ 1 second: 1000 retries hit at SAME time
  → Thundering herd
  
Service was 503, gets hammered by 1000 retries, stays down
```

**Solution: Add jitter** (randomness):

```
wait_time = base * 2^attempt + random(0, jitter)

Example:
  Attempt 1: wait = 1 * 2^0 + rand(0-1000ms) = 1-2 seconds
  Attempt 2: wait = 1 * 2^1 + rand(0-1000ms) = 2-3 seconds
  Attempt 3: wait = 1 * 2^2 + rand(0-1000ms) = 4-5 seconds
  Attempt 4: wait = 1 * 2^3 + rand(0-1000ms) = 8-9 seconds

Result: All 1000 retries spread across time, not at same second
Service recovers, fewer are wasted
```

---

## Capping Maximum Wait

Problem with unbounded exponential:

```
Attempt 1: 1s
Attempt 2: 2s
Attempt 3: 4s
Attempt 4: 8s
Attempt 5: 16s
Attempt 6: 32s
Attempt 7: 64s (over 1 minute!)

Total wait: 127 seconds

User sees: "Service is slow" (from their perspective)
System is: Actually healthy, user is just waiting for backoff
```

**Solution: Cap at reasonable maximum**:

```
max_wait = 60 seconds
Attempt 7: min(2^6 + jitter, 60) = 60 seconds
Attempt 8: 60 seconds (capped)
Attempt 9: 60 seconds (capped)

Total backoff cap: 60 seconds
If still failing after 60s, likely won't recover
```

---

## Idempotency: Critical for Retries

### The Problem

```
Request 1: POST /transfer (Transfer $100 from account A to B)
  Network fails before response received
  Client doesn't know if succeeded or failed

Client retries: POST /transfer (same request)
  Request 2: $100 transferred AGAIN!
  Account B now has +$200 instead of +$100
  
Result: Money duplicated!
```

---

### Solution: Idempotency Keys

```
Request 1: POST /transfer
  Headers: Idempotency-Key: "12345"
  Body: {fromId: 100, toId: 200, amount: 100}
  
  Server:
    Check if key "12345" already processed
    No → Execute transfer, store mapping: 12345 → txn_id_999
    Return: {status: "success", txn_id: 999}

Request 2 (retry): POST /transfer
  Headers: Idempotency-Key: "12345"
  Same body
  
  Server:
    Check if key "12345" already processed
    Yes → Return stored result: {status: "success", txn_id: 999}
    Don't re-execute transfer
    
Result: Same result both times, no duplicate charge!
```

---

### Safe vs Unsafe Methods

**Safe to retry** (GET, HEAD):
```
GET /user/123
  Doesn't modify anything
  Safe to retry 100 times
  Same response each time
```

**Unsafe without idempotency** (POST, PUT, DELETE):
```
POST /transfer → Might charge twice
PUT /update → Might overwrite with race condition
DELETE /user → Might fail second time (already deleted), but maybe OK

Rule: Never retry unsafe operations without idempotency key
```

---

## Retry Budget

Prevent retries from overwhelming system:

```
Service SLA: Respond within 5 seconds
  Assume 1% error rate
  
Retry budget: 1% of requests can be retried
  1% × 1M req/sec = 10k retry req/sec
  
If retries exceed budget:
  Circuit breaker open, reject new requests
  Prevent retry amplification
```

---

## Full Algorithm Example

```python
def call_with_retry(func, max_attempts=5, base_wait=1, max_wait=60):
  for attempt in range(max_attempts):
    try:
      return func()
    except TransientError as e:
      if attempt == max_attempts - 1:
        raise  # Last attempt, give up
      
      # Exponential backoff + jitter
      wait = base_wait * (2 ** attempt)
      wait = min(wait, max_wait)  # Cap at max
      jitter = random.uniform(0, wait * 0.1)  # 10% jitter
      total_wait = wait + jitter
      
      sleep(total_wait)
    except PermanentError as e:
      raise  # Don't retry permanent errors
```

---

## Distributed Retries: Watch for Amplification

Problem: Multiple layers of retries:

```
Client → API Gateway → Service A → Service B

Client retries (1s, 2s, 4s, 8s)
API Gateway retries (1s, 2s, 4s, 8s)  ← Now 2x retries!
Service A retries (1s, 2s, 4s, 8s)    ← Now 3x retries!

Result: One failed request multiplied to 3^5 = 243 retries
Service B gets hammered
```

**Solution**:
- Only outermost layer retries (client or gateway)
- Inner services fail immediately, don't retry
- OR track retries across layers, cap total

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Exponential backoff** | Linear or no backoff | Gives service time to recover, reduces load |
| **Jitter** | Pure exponential | Prevents thundering herd, better distribution |
| **Max cap** | Unbounded | Prevents user-facing timeouts from backoff |
| **Idempotency keys** | Hope for best | Prevents duplicate charges, safe retries |
| **Retry budget** | Unlimited retries | Prevents retry amplification cascade |

---

## Related Fundamentals

- [Circuit Breakers](circuit-breakers.md) – Stop retrying when service is down
- [Timeouts](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Detect failures quickly
- [APIs & Communication](../apis-and-communication/) – Idempotency API design

---

**Status**: ✅ Complete. Covers backoff algorithms, idempotency, retry budgets, and amplification risks.

