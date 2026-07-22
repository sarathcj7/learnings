# Token Bucket Algorithm

The most popular rate-limiting algorithm in production systems. Simple, elegant, and handles burst traffic well. Used by AWS, Google Cloud, and most API gateways.

---

## TL;DR

- **Token bucket**: A bucket with capacity C; tokens added at rate R per second. Request costs 1 token. No token → reject (rate-limited).
- **Handles bursts**: Up to C requests can be served immediately if tokens available; then limited to R req/s after.
- **Implementation**: Maintain bucket state (tokens, last_refill_time). On each request, refill based on elapsed time.
- **Distributed**: Use Redis/Memcached to maintain shared bucket state across multiple servers.
- **Advantages**: Predictable throughput, handles bursts, easy to implement, tunable (capacity, rate).
- **Disadvantages**: Tokens lost on overflow (if bucket full, new tokens discarded); requires centralized state for multi-server.

---

## Core Concept

### Bucket Mechanics

```
Capacity: 10 tokens
Refill rate: 2 tokens per second

Initial state: 10 tokens (full)

Timeline:
  T=0s:   10 tokens. Request 1 → 9 tokens
  T=0.5s: 9 + (0.5 * 2) = 10 tokens (full). Request 1 → 9 tokens
  T=1s:   9 + (0.5 * 2) = 10 tokens. Request 10 → 0 tokens (all consumed)
  T=1.1s: 0 + (0.1 * 2) = 0.2 tokens. Request 1 → REJECTED (0.2 < 1)
  T=1.5s: 0.2 + (0.4 * 2) = 1 token. Request 1 → 0 tokens
```

### Visual Representation

```
Capacity = 10 tokens
Rate = 2 tokens/sec

[████████████████████] 10 tokens (full)
 Request arrives (1 token)
[███████████████████] 9 tokens
 1 second passes (2 new tokens generated)
[████████████████████] 10 tokens (full, overflow capped)
 10 requests arrive (each needs 1 token)
[] 0 tokens (empty)
 0.5 seconds pass (1 new token generated)
[██] 1 token
 Request needs 1 token
[] 0 tokens (empty, request served)
 Next request → REJECTED (no tokens)
```

---

## Implementation

### Simple Single-Server Implementation

```python
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.tokens = capacity
        self.last_refill_time = time.time()
    
    def allow_request(self):
        now = time.time()
        elapsed = now - self.last_refill_time
        
        # Refill tokens based on elapsed time
        tokens_to_add = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill_time = now
        
        # Check if request can proceed
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        else:
            return False
```

### Example Usage

```python
bucket = TokenBucket(capacity=10, refill_rate=2)

for i in range(15):
    if bucket.allow_request():
        print(f"Request {i}: ALLOWED")
    else:
        print(f"Request {i}: REJECTED (rate-limited)")
    
    # Assume request takes 0.1 seconds to process
    time.sleep(0.1)

Output:
Request 0: ALLOWED     (9 tokens left)
Request 1: ALLOWED     (8 tokens left)
...
Request 9: ALLOWED     (0 tokens left)
Request 10: REJECTED   (no tokens, 0.2 sec until next token)
Request 11: REJECTED   (still no tokens)
...
Request 15: ALLOWED    (after ~1.5 seconds, tokens refilled)
```

### Redis-Based Distributed Implementation

```python
import redis
import time

class DistributedTokenBucket:
    def __init__(self, redis_client, key, capacity, refill_rate):
        self.redis = redis_client
        self.key = key
        self.capacity = capacity
        self.refill_rate = refill_rate
        
        # Initialize bucket in Redis
        self.redis.hset(key, mapping={
            'tokens': capacity,
            'last_refill': time.time()
        })
    
    def allow_request(self):
        # Lua script for atomic operation (avoid race conditions)
        lua_script = """
        local tokens = tonumber(redis.call('HGET', KEYS[1], 'tokens'))
        local last_refill = tonumber(redis.call('HGET', KEYS[1], 'last_refill'))
        local now = ARGV[1]
        local capacity = tonumber(ARGV[2])
        local rate = tonumber(ARGV[3])
        
        local elapsed = now - last_refill
        local new_tokens = math.min(capacity, tokens + elapsed * rate)
        
        if new_tokens >= 1 then
            redis.call('HSET', KEYS[1], 'tokens', new_tokens - 1)
            redis.call('HSET', KEYS[1], 'last_refill', now)
            return 1
        else
            redis.call('HSET', KEYS[1], 'tokens', new_tokens)
            redis.call('HSET', KEYS[1], 'last_refill', now)
            return 0
        end
        """
        
        result = self.redis.eval(
            lua_script, 1, self.key,
            time.time(), self.capacity, self.refill_rate
        )
        return result == 1

# Usage
redis_client = redis.Redis()
bucket = DistributedTokenBucket(redis_client, 'api:user:123', capacity=10, refill_rate=2)

if bucket.allow_request():
    print("Request allowed")
else:
    print("Request rejected (rate-limited)")
```

---

## Handling Bursts

### Burst Capacity

Token bucket is ideal for bursts: If clients send 10 requests at once, the first 10 (up to capacity) are served immediately.

```
Scenario: API with capacity=10, rate=2 req/s

Steady-state load: 2 requests per second
Bursty load: 10 requests at once

With token bucket:
  - 10 requests arrive simultaneously
  - All 10 are served (tokens = 10)
  - Tokens now = 0
  - Next requests are delayed until tokens refill
  - Throughput returns to 2 req/s

Without rate limiting:
  - 10 requests processed immediately (spike)
  - Backend server overloaded, tail latency explodes
```

### Burst Size vs. Steady-State Rate

Different buckets for different burst tolerances:

```
High burst tolerance:
  Capacity = 100, Rate = 10 req/s
  Allows bursts of 100, steady rate 10 req/s

Low burst tolerance:
  Capacity = 10, Rate = 10 req/s
  Allows bursts of 10, steady rate 10 req/s
  Nearly equivalent to leaky bucket (smooth output)

Unlimited burst:
  Capacity = infinity, Rate = 10 req/s (not recommended, defeats limiting)
```

---

## Multi-Token Requests

Some operations cost more than 1 token (e.g., bulk operations, large file uploads).

```python
def allow_request(tokens_needed=1):
    if self.tokens >= tokens_needed:
        self.tokens -= tokens_needed
        return True
    else:
        return False

# Usage
if bucket.allow_request(tokens_needed=5):  # Bulk operation needs 5 tokens
    print("Bulk operation allowed")
else:
    print("Bulk operation rate-limited; retry with single items")
```

---

## Distributed Considerations

### Token Bucket State Management

In a distributed system, multiple servers must share the same bucket:

**Approach 1: Centralized (Redis)**
- Single Redis instance maintains bucket state
- Pro: Strong consistency
- Con: Redis is single point of failure (mitigate with Redis replication)

**Approach 2: Per-Server Bucket**
- Each server has its own bucket (capacity / number_of_servers)
- Pro: No coordination, fault-tolerant
- Con: Uneven distribution if servers have different load

**Approach 3: Hybrid**
- Each server has local bucket (capacity / N)
- If local bucket depleted, request forwarded to central bucket (Redis)
- Pro: Reduces Redis load, still fair
- Con: Complexity

### Synchronization Issues

```
Without proper sync:
  Server A thinks user has 5 tokens
  Server B thinks user has 5 tokens
  Both serve 5 requests to same user
  Actual limit violated (10 requests instead of 5)

Solution: Use Redis Lua script (atomic operation)
  All servers execute same script on Redis
  State updated atomically
  No race conditions
```

---

## Advantages & Disadvantages

### Advantages

- **Predictable throughput**: Rate R is guaranteed over long window
- **Burst handling**: Capacity C allows temporary spikes without rejection
- **Simple to implement**: Straightforward logic, easy to debug
- **Tunable**: Capacity and rate are independent, flexible configuration
- **No message loss tracking needed**: Unlike sliding window, no need to store request timestamps

### Disadvantages

- **Overflow loss**: Tokens generated after bucket is full are discarded (capacity acts as ceiling)
- **Requires state management**: Need to track tokens and refill time (single-server state or Redis for distributed)
- **Delayed feedback**: User doesn't know rate-limit status until request arrives and tokens are checked
- **Doesn't account for time windows**: If user sends 5 requests in 1st second, then waits 10 seconds, all tokens refilled — no memory of past

### When Tokens Are Discarded

```
Capacity = 10, Rate = 1 token/sec

T=0: Bucket full (10 tokens)
User inactive for 100 seconds (100 new tokens generated, but capacity = 10, so all discarded)
T=100: Bucket still has 10 tokens

Result: Inactive users get same burst capacity as active users (fair or unfair?)
```

---

## Real-World Examples

### AWS API Gateway

Rate limit: 10,000 requests per second per API, with burst capacity of 5,000 requests

```
Capacity = 5,000
Rate = 10,000 req/sec

Handles sudden spikes (flash sale), then smooths to steady rate
```

### GitHub API

Rate limits per token (API key):

```
Requests per hour: 5,000 (rate)
Burst: Can be consumed quickly if requests are spaced closely
```

---

## Comparison with Alternatives

| Aspect | Token Bucket | Leaky Bucket | Sliding Window |
|--------|--------------|--------------|-----------------|
| **Burst handling** | Good | Poor (smooths all) | Excellent |
| **Memory usage** | Low | Low | Medium (tracks requests) |
| **Fairness** | Per-user burst | Smooth rate | Per time window |
| **Implementation** | Simple | Simple | Complex (tracking) |

---

## Related Fundamentals

- **[Sliding Window Counter](sliding-window.md)**: Alternative approach with better time-window precision
- **[Rate Limiting Strategies](rate-limiting-strategies.md)**: Per-user, per-endpoint, global limits
- **[Reliability & Resiliency](../reliability-and-resiliency/)**: Rate limiting as resilience mechanism
- **[Scalability & Load Balancing](../scalability-and-load-balancing/)**: Traffic shaping and load shedding

---

## Key Takeaways

1. **Token bucket is the industry standard**: Simple, practical, handles bursts.
2. **Capacity and rate are independent levers**: Tune separately for burst tolerance and throughput.
3. **Distributed implementation requires centralized state**: Use Redis with atomic Lua scripts.
4. **Tokens overflow silently**: Inactive users don't "accumulate" tokens over time beyond capacity.
5. **Trade-off**: Simplicity vs. precision. Sliding window is more precise per time window; token bucket is simpler and handles bursts naturally.

---

**Study Tips**

- Implement token bucket from scratch in Python. Step through by hand for a few requests.
- Compare token bucket vs. leaky bucket conceptually. Draw the flow.
- Design a distributed rate limiter using Redis. Think through race conditions.
- Calculate capacity and rate for your API's expected burst and steady-state load.

---

**Status**: ✅ Complete
