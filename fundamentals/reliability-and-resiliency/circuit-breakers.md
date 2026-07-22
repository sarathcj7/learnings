# Circuit Breakers & Fault Tolerance

Preventing cascade failures by detecting broken services and stopping requests before they fail. The circuit breaker pattern is essential for microservices reliability.

---

## TL;DR

- **Circuit breaker**: Monitor failures, trip when threshold exceeded, stop sending requests
- **States**: Closed (normal) → Open (failing) → Half-open (testing recovery)
- **Prevents cascade**: One failing service doesn't bring down dependent services
- **Fast fail**: Reject requests immediately instead of waiting for timeout
- **Production critical**: Every microservice interaction should have circuit breaker

---

## Problem: Cascade Failures

### Without Circuit Breaker

```
Service A calls Service B (happy path):
  Request → Service B → Success (50ms)

Service B starts failing (500 errors):
  Request 1 → B → Timeout (30s) → Return error
  Request 2 → B → Timeout (30s) → Return error
  ...
  Request 1000 → B → Timeout (30s) → Return error
  
Total time wasted: 1000 × 30s = 30,000 seconds of timeouts!

Meanwhile:
  A's connection pool exhausted (100 connections × 30s = 3000s wait)
  A becomes unresponsive (appears to be broken)
  Clients of A get errors
  Cascade: B fails → A fails → C fails (if C calls A)
```

### With Circuit Breaker

```
Service B fails:
  Request 1 → Fail (5ms)
  Request 2 → Fail (5ms)
  Request 3 → Fail (5ms)
  Request 4 → Fail (5ms)
  Request 5 → Fail (5ms)
  
After 5 failures: Circuit OPENS (stop sending requests)
  
Request 6+ → Circuit breaker rejects immediately (1ms)
  No timeout wait, no connection pool exhaustion
  Service A stays responsive
  
Result: A survives while waiting for B to recover
```

---

## Circuit Breaker States

### State Machine

```
    ┌─────────────────────────────────────┐
    │          CLOSED (Normal)            │
    │  Requests pass through normally     │
    │  Monitor failure rate               │
    │  success_count = 0                  │
    │  failure_count = 0                  │
    └──────────────┬──────────────────────┘
                   │
                   │ [failure_rate > threshold]
                   │ [failures > min_failures]
                   ▼
    ┌─────────────────────────────────────┐
    │         OPEN (Failing)              │
    │  Reject all requests (fail fast)    │
    │  Don't send to downstream service   │
    │  status = "Circuit Open"            │
    │  transition_time = now              │
    └──────────────┬──────────────────────┘
                   │
                   │ [timeout (30s) reached]
                   │ Try recovery
                   ▼
    ┌─────────────────────────────────────┐
    │       HALF-OPEN (Testing)           │
    │  Allow limited requests through     │
    │  Test if downstream recovered       │
    │  max_requests_in_half_open = 5      │
    └──────────────┬──────────────────────┘
                   │
          ┌────────┴────────┐
          │                 │
     [All succeed]      [Any fails]
          │                 │
          ▼                 ▼
    ┌──────────────┐   ┌──────────────┐
    │ CLOSED       │   │ OPEN         │
    │ Resume flow  │   │ Back to open │
    │              │   │ Reset timer  │
    └──────────────┘   └──────────────┘
```

---

## Closed State: Normal Operation

Requests pass through, failures counted:

```python
class CircuitBreaker:
  def call(self, func, *args):
    if self.state == CLOSED:
      try:
        result = func(*args)
        self.success_count += 1
        self.failure_count = 0  # Reset on success
        return result
      except Exception as e:
        self.failure_count += 1
        
        # Check if should trip
        if self.failure_count >= self.min_failures:
          if self.failure_count / (self.success_count + self.failure_count) > self.threshold:
            self.trip()  # → OPEN
        raise e
```

**Config example**:
```
min_failures: 5
threshold: 50% (5 failures out of 10 attempts)

If 5 failures out of 10 attempts fail, trip circuit
```

---

## Open State: Failing Service

Reject requests immediately:

```python
def call(self, func, *args):
  if self.state == OPEN:
    # Check if timeout passed
    if now - self.trip_time > self.timeout:
      self.state = HALF_OPEN
      self.success_count = 0
      self.failure_count = 0
      # Fall through to try one request
    else:
      # Still in timeout, reject immediately
      raise CircuitBreakerOpen("Circuit breaker is open")
```

**Timeout**: Typically 30-60 seconds.

---

## Half-Open State: Testing Recovery

Allow limited requests to test if service recovered:

```python
def call(self, func, *args):
  if self.state == HALF_OPEN:
    # Allow only N requests to test
    if self.active_requests >= self.max_half_open_requests:
      raise CircuitBreakerOpen("Too many half-open requests")
    
    try:
      result = func(*args)
      self.success_count += 1
      
      # All tests succeeded, resume normal
      if self.success_count >= self.max_half_open_requests:
        self.state = CLOSED
        self.failure_count = 0
      return result
    except Exception as e:
      # Test failed, back to open
      self.failure_count += 1
      self.state = OPEN
      self.trip_time = now
      raise e
```

---

## Production Configurations

### Aggressive (Fast Failover)

```
min_failures: 3
threshold: 33%  (3 failures out of 9)
timeout: 10 seconds
half_open_max_requests: 3

Result: Trip circuit quickly, recover fast
Use for: Non-critical APIs, user can retry
```

---

### Conservative (Avoid False Positives)

```
min_failures: 10
threshold: 50%  (10 failures out of 20)
timeout: 60 seconds
half_open_max_requests: 5

Result: Only trip on sustained failures
Use for: Critical payment APIs, money involved
```

---

### Balanced (Recommended Default)

```
min_failures: 5
threshold: 50%
timeout: 30 seconds
half_open_max_requests: 5

Result: Trip on moderate failure rate, 30s recovery test
Use for: Most services
```

---

## Monitoring & Alerting

### Metrics to Track

```
circuit_breaker_state (gauge)
  0 = CLOSED (normal)
  1 = OPEN (failing)
  2 = HALF_OPEN (testing)

circuit_breaker_transitions (counter)
  How often circuit trips (shouldn't happen often)

circuit_breaker_success_rate (gauge)
  % of successful requests in closed state

circuit_breaker_rejections (counter)
  How many requests rejected while open
  (Successful fast-fail count)
```

### Alerting Rules

```
Alert if:
  circuit_breaker_state == OPEN for > 5 minutes
    → Downstream service likely down

Alert if:
  circuit_breaker_transitions > 10 in 10 minutes
    → Service flapping (unreliable)
```

---

## Circuit Breaker Per Endpoint

### Issue: All-Or-Nothing

If entire service is up but single endpoint fails:

```
Service B:
  /api/users → Health check passes
  /api/posts → Returns 500 errors
  
Without per-endpoint CB:
  Circuit trips for entire service

With per-endpoint CB:
  Only /api/posts circuit trips
  /api/users still works
  Better availability
```

### Implementation

```
Map: {
  "POST /api/users": CircuitBreaker(),
  "GET /api/posts": CircuitBreaker(),
  "PUT /api/posts/:id": CircuitBreaker(),
}
```

---

## Patterns: Fallback vs Reject

### Pattern 1: Fail Fast (Reject)

```
Service A calls Service B
Service B circuit opens
Response: 503 Service Unavailable (immediately)

Pros: Clear error, client knows to retry
Cons: User sees error

Use: APIs where fallback doesn't make sense
```

---

### Pattern 2: Fallback

```
Service A calls Service B (get user profile)
Service B circuit opens
Fallback: Return cached profile (30 min old)

Pros: User still sees something
Cons: Data is stale, incorrect in rare cases

Use: Non-critical data, caching available
```

---

### Pattern 3: Graceful Degradation

```
Service A calls Service B (generate recommendations)
Service B circuit opens
Fallback: Return popular items (generic, not personalized)

Pros: Service still functions, better UX than error
Cons: Less relevant recommendations

Use: Ranking, suggestions, non-core features
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Circuit breaker** | Retries only | Prevents cascade, enables fast fail |
| **Per-endpoint** | Service-level | More granular, handles partial failures |
| **Fast trip** | Conservative | Catches failures early, prevents overload |
| **Fallback** | Reject | Preserves service function, acceptable staleness |

---

## Related Fundamentals

- [Retries & Exponential Backoff](retries-and-exponential-backoff.md) – Complementary to circuit breakers
- [Bulkheads & Resource Isolation](bulkheads-and-resource-isolation.md) – Separate failure domains
- [Timeouts](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Detect failures quickly

---

**Status**: ✅ Complete. Covers state machine, configurations, monitoring, and patterns.

