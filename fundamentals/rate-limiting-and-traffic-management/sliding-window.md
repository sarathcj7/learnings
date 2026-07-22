# Sliding Window Counter (Sliding Log)

An alternative to token bucket that tracks individual request timestamps. Provides precise rate-limiting per fixed time window at the cost of higher memory overhead.

---

## TL;DR

- **Sliding window**: Track timestamps of requests in a window (e.g., last 1 minute). On new request, count requests in window; reject if exceeds limit.
- **Precise per window**: Exactly enforces limit for each 1-minute (or 1-second) window, no token overflow.
- **Trade-off**: Accuracy (tracks actual requests in window) vs. memory (stores all request timestamps).
- **Implementation**: Store request timestamps in sorted set (Redis), purge old entries, count current window.
- **Better for**: Strict rate limiting where precision matters (API quotas, billing).
- **Worse for**: Systems with high request volume (memory overhead) or needing burst handling.

---

## Core Concept

### Window Example

Limit: 5 requests per 1-minute window

```
Time Window: [T - 60s, T]

Timeline:
  T=0s:   Request 1 → Count=1 in [0, 60]. ALLOWED
  T=10s:  Request 2 → Count=2 in [10, 70]. ALLOWED
  T=20s:  Request 3 → Count=3 in [20, 80]. ALLOWED
  T=30s:  Request 4 → Count=4 in [30, 90]. ALLOWED
  T=40s:  Request 5 → Count=5 in [40, 100]. ALLOWED
  T=50s:  Request 6 → Count=5 in [50, 110]. REJECTED (at limit)
  T=61s:  Request 7 → Count=2 in [61, 121] (R1,R2 outside window). ALLOWED
```

### Visual Representation

```
Limit: 5 requests per minute

Window: [0s, 60s]
[R1=5s] [R2=15s] [R3=25s] [R4=35s] [R5=45s] ✗ [R6=55s]
|-------|-------|-------|-------|-------|
0s                                       60s

R6 at 55s → Count requests in [55-60, 55] = 5 (R1, R2, R3, R4, R5) → REJECTED

Window: [1s, 61s]
        [R2=15s] [R3=25s] [R4=35s] [R5=45s] ✓ [R6=55s]
|-------|-------|-------|-------|-------|
1s                                       61s

R7 at 61s → Count requests in [61-60, 61] = [1s, 61s] = 2 (R2, R3 outside; R5, R6, R7 inside) → ALLOWED
```

---

## Implementation

### Single-Server Implementation (In-Memory)

```python
from collections import deque
import time

class SlidingWindowCounter:
    def __init__(self, window_size, limit):
        self.window_size = window_size  # seconds
        self.limit = limit  # max requests per window
        self.requests = deque()  # timestamps of requests
    
    def allow_request(self):
        now = time.time()
        
        # Remove requests outside the window
        while self.requests and self.requests[0] < now - self.window_size:
            self.requests.popleft()
        
        # Check if request is allowed
        if len(self.requests) < self.limit:
            self.requests.append(now)
            return True
        else:
            return False

# Usage
limiter = SlidingWindowCounter(window_size=60, limit=5)

for i in range(10):
    if limiter.allow_request():
        print(f"Request {i}: ALLOWED")
    else:
        print(f"Request {i}: REJECTED")
    time.sleep(5)  # 5 seconds between requests
```

### Redis Sorted Set Implementation

```python
import redis
import time

class DistributedSlidingWindow:
    def __init__(self, redis_client, key, window_size, limit):
        self.redis = redis_client
        self.key = key
        self.window_size = window_size
        self.limit = limit
    
    def allow_request(self):
        now = time.time()
        
        # Use Lua script for atomic operation
        lua_script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])
        
        -- Remove old requests outside the window
        redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
        
        -- Count requests in the current window
        local count = redis.call('ZCARD', key)
        
        if count < limit then
            -- Add current request
            redis.call('ZADD', key, now, now)
            -- Set expiry to window_size
            redis.call('EXPIRE', key, window)
            return 1
        else
            return 0
        end
        """
        
        result = self.redis.eval(lua_script, 1, self.key, now, self.window_size, self.limit)
        return result == 1

# Usage
redis_client = redis.Redis()
limiter = DistributedSlidingWindow(redis_client, 'api:user:123', window_size=60, limit=5)

if limiter.allow_request():
    print("Request allowed")
else:
    print("Request rejected (rate-limited)")
```

### Sorted Set Details

```
Redis ZSET (sorted set) stores (score, member) pairs, sorted by score.

For sliding window:
  Score = timestamp of request
  Member = unique ID (could be timestamp, or "timestamp_counter")

Commands:
  ZADD key score member  → Add request timestamp
  ZCARD key              → Count requests
  ZREMRANGEBYSCORE key min max → Remove requests outside window

Example:
  ZADD "user:123" 1000 "req_1000_1"
  ZADD "user:123" 1010 "req_1010_1"
  ZADD "user:123" 1020 "req_1020_1"
  
  ZCARD "user:123" → 3 requests
  
  ZREMRANGEBYSCORE "user:123" "-inf" 950 → Remove all before 950s (purge)
  
  ZCARD "user:123" → 3 requests still (all within window)
```

---

## Advantages

- **Precise per window**: Exactly limits N requests per time window (no overflow like token bucket)
- **No burst loss**: Every burst up to the limit is allowed; no tokens discarded
- **Clear semantics**: "Limit 10 requests per minute" is exactly what it does
- **Fair across users**: Each user gets full quota per window
- **Easy to understand**: Timestamps are intuitive; no refill rates or capacity

---

## Disadvantages

- **Memory overhead**: Stores all request timestamps (10k requests/sec * 60s = 600k entries in memory)
- **Storage costs**: In distributed systems, Redis can become expensive with high request volume
- **Slower than token bucket**: Each request requires timestamp lookup + deque/ZSET operations
- **No smooth traffic**: User can send all N requests in first second, then nothing; doesn't smooth to average rate
- **Clock skew risk**: If servers have different clocks, entries might be purged incorrectly or duplicated

---

## Memory Overhead Analysis

### Memory Cost

```
Scenario: 100,000 users, 1,000 requests/sec, 60-second window

Worst case (all users equally active):
  Time window = 60s
  Requests per user = 1,000 * 60 / 100,000 = 0.6 requests per user per minute
  Total entries = ~60,000 per minute (100k users * 60 entries per minute)
  
Per entry (in Redis sorted set):
  Score (timestamp): 8 bytes
  Member (ID): ~20 bytes
  Redis overhead: ~60 bytes per entry
  Total per entry: ~100 bytes
  
Total memory = 60,000 entries * 100 bytes = 6 MB per minute
Over 60 minutes: 360 MB (sustainable)

But if request rate increases to 10,000 req/sec:
  Total entries = 600,000 per minute
  Total memory = 60 MB per minute
  Over 60 minutes: 3.6 GB (expensive, requires larger Redis instance)
```

### Token Bucket (by comparison)

```
Token bucket stores:
  - Current token count (1 float): 8 bytes
  - Last refill time (1 timestamp): 8 bytes
  - Total: ~16 bytes per user

For 100,000 users: 1.6 MB (negligible)
```

---

## Comparison with Token Bucket

| Aspect | Sliding Window | Token Bucket |
|--------|---|---|
| **Memory** | O(rate * window) | O(1) per bucket |
| **Precision** | Exact per window | Approximate (with bursts) |
| **Burst handling** | Allows full burst in 1st second | Smooth over time, up to capacity |
| **Fairness** | Fair (each window same quota) | Unfair (inactive users get burst capacity) |
| **Performance** | Slower (timestamp lookup) | Faster (simple arithmetic) |
| **Real-world use** | API billing, strict quotas | General rate limiting, traffic shaping |

---

## Hybrid Approach: Sliding Window + Token Bucket

Combine both for best of both worlds:

```python
class HybridRateLimiter:
    def __init__(self, redis_client, key, window_size, limit):
        self.token_bucket = TokenBucket(capacity=limit, refill_rate=limit/window_size)
        self.sliding_window = SlidingWindowCounter(redis_client, key, window_size, limit)
    
    def allow_request(self):
        # Check both: token bucket (fast, local) AND sliding window (accurate, distributed)
        if not self.token_bucket.allow_request():
            return False  # Token bucket rejected
        
        # Token bucket allowed, but double-check with sliding window for accuracy
        if not self.sliding_window.allow_request():
            return False  # Sliding window rejected (e.g., clock skew or many servers)
        
        return True  # Both allow

# Benefit: Token bucket provides fast local checks; sliding window ensures distributed accuracy
```

---

## Clock Skew Handling

### Problem

If servers have unsynchronized clocks, sliding window breaks:

```
Server A (fast clock): T=1010, records request as timestamp 1010
Server B (slow clock): T=1000, purges entries before 950
Result: Server B thinks it's in an earlier time window, allows requests Server A rejected
```

### Solution

```
1. Use NTP (Network Time Protocol) to sync server clocks
2. Use centralized time source (Redis server time, not client time)
3. Add clock skew tolerance (allow +/- 5 seconds)
4. Log clock skew anomalies for ops investigation
```

---

## Real-World Examples

### GitHub API Rate Limits

- Limit: 5,000 requests per hour per token
- Implementation: Likely sliding window (tracks per-hour usage exactly)
- Precision: Needed for billing accuracy

### AWS API Gateway

- Limit: 10,000 requests per second per API
- Implementation: Likely token bucket (handles burst, high throughput)
- Precision: Less critical (burst handling preferred)

---

## When to Use

**Sliding Window for**:
- **Billing accuracy**: Each customer gets exactly N requests per month
- **Strict quotas**: API tiers (free: 100 req/day, paid: 10k req/day)
- **Compliance**: Regulatory requirements (logging all requests per window)
- **Low to medium traffic**: <1k req/sec per limiter

**Token Bucket for**:
- **Traffic shaping**: Smooth traffic, handle bursts gracefully
- **High throughput**: 10k+ req/sec
- **General APIs**: Fairness secondary to availability
- **Simplicity**: Token bucket is easier to reason about and debug

---

## Related Fundamentals

- **[Token Bucket](token-bucket.md)**: Primary alternative to sliding window
- **[Rate Limiting Strategies](rate-limiting-strategies.md)**: Per-user, per-endpoint, global limits
- **[Scalability & Load Balancing](../scalability-and-load-balancing/)**: Traffic shaping and load shedding
- **[Reliability & Resiliency](../reliability-and-resiliency/)**: Rate limiting for system protection

---

## Key Takeaways

1. **Sliding window tracks actual requests** in a time window for precise limiting.
2. **Memory overhead scales with request rate**: High-traffic systems need careful capacity planning.
3. **Hybrid approach** (token bucket local + sliding window remote) combines speed and accuracy.
4. **Clock skew is a real issue** in distributed systems; use centralized time source.
5. **Trade-off**: Accuracy vs. performance. Choose based on requirements (billing vs. traffic shaping).

---

**Study Tips**

- Implement both token bucket and sliding window. Compare latency and memory usage.
- Trace a sliding window by hand for 10 requests over 60 seconds.
- Calculate memory overhead for your system's request rate.
- Debate: GitHub API (sliding window) vs. AWS API Gateway (token bucket). Why each?

---

**Status**: ✅ Complete
