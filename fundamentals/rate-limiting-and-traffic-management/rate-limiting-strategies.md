# Rate Limiting Strategies

Beyond the algorithm (token bucket, sliding window), rate limiting has many dimensions: per-user, per-endpoint, per-IP, global limits, and adaptive strategies. This file explores strategies to apply rate limiting in real systems.

---

## TL;DR

- **Per-user limits**: Individual quotas per API key/user (most common)
- **Per-endpoint limits**: Different limits for expensive operations (image upload vs. read)
- **Per-IP limits**: Prevent single IP from overwhelming system
- **Global limits**: System-wide throughput cap
- **Adaptive rate limiting**: Adjust limits based on server load/latency
- **Backpressure & load shedding**: Reject/defer requests when overwhelmed
- **Choice depends on**: System criticality (API vs. internal), traffic patterns (bursty vs. smooth), business requirements (quotas for monetization)

---

## Per-User Rate Limiting

The most common strategy. Each user/API key has independent quota.

### Mechanics

```
User A (free tier): 100 requests/day
User B (paid tier): 10,000 requests/day
User C (enterprise): 1,000,000 requests/day

Each user's quota is tracked separately and independently.
One user hitting their limit doesn't affect others.
```

### Implementation

```python
def rate_limit_per_user(user_id, request):
    bucket = redis.hgetall(f"user:{user_id}:rate_limit")
    
    if bucket.is_available():
        bucket.consume_token()
        return forward_to_backend(request)
    else:
        return reject_with_429(
            "Rate limit exceeded",
            retry_after=bucket.time_until_next_token()
        )
```

### Advantages

- **Fair to users**: Each gets their quota regardless of others
- **Monetization**: Can offer tiered quotas (free vs. paid)
- **Isolation**: One heavy user doesn't affect others
- **Flexible**: Easy to adjust per-user quotas

### Disadvantages

- **Shared abuse**: If one user's API key is leaked, all quota consumed
- **Quota fairness**: Fair allocation needs monitoring (users may game system)
- **Storage**: Need to track quota for every user (millions of users = millions of buckets)

### Database Schema

```sql
CREATE TABLE user_rate_limits (
    user_id INT PRIMARY KEY,
    tier VARCHAR(10),  -- 'free', 'paid', 'enterprise'
    daily_limit INT,   -- requests per day
    current_day_requests INT,
    day_started_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Indexes
CREATE INDEX idx_user_day ON user_rate_limits(user_id, day_started_at);
```

---

## Per-Endpoint Rate Limiting

Different endpoints have different costs. Expensive operations have lower limits.

### Example

```
GET /users/{id}          → 1 token per request (cheap, read-only)
POST /images/upload      → 50 tokens per request (expensive, storage)
GET /search              → 5 tokens per request (compute-intensive, indexed)
DELETE /account          → 1000 tokens per request (dangerous, prevent accidents)

User quota: 10,000 tokens/day
Can make:
  - 10,000 GET /users requests, OR
  - 200 POST /images/upload requests, OR
  - Mix of operations up to 10,000 tokens
```

### Implementation

```python
ENDPOINT_COSTS = {
    'GET /users/{id}': 1,
    'POST /images/upload': 50,
    'GET /search': 5,
    'DELETE /account': 1000,
}

def rate_limit_per_endpoint(user_id, endpoint, request):
    cost = ENDPOINT_COSTS.get(endpoint, 1)  # default 1 token
    bucket = get_user_bucket(user_id)
    
    if bucket.tokens >= cost:
        bucket.consume_tokens(cost)
        return forward_to_backend(request)
    else:
        return reject_with_429(f"Insufficient tokens ({cost} required)")
```

### Advantages

- **Cost-aware**: Prevents users from abusing expensive endpoints
- **Flexibility**: Can adjust costs dynamically
- **Resource protection**: Expensive operations are naturally limited

### Disadvantages

- **Complexity**: Need to define costs for every endpoint (requires estimation)
- **Unpredictable quotas**: Users don't know how many operations they can perform (1 cheap vs. 1 expensive)
- **Gaming**: Users may batch expensive operations with many cheap ones

---

## Per-IP Rate Limiting

Limit requests from a single IP address (regardless of user).

### Use Cases

- **DDoS mitigation**: Reject requests from IPs sending too many requests
- **Abuse prevention**: Block IPs doing suspicious activity (brute force, scraping)
- **Public APIs**: Throttle unauthenticated requests by IP

### Implementation

```python
def rate_limit_per_ip(client_ip, request):
    bucket = redis.get(f"ip:{client_ip}:rate_limit")
    limit = 1000  # 1000 req/hour per IP
    
    if bucket.allow_request():
        return forward_to_backend(request)
    else:
        return reject_with_429(f"Too many requests from {client_ip}")
```

### Challenges

- **Shared IPs**: Proxy servers, corporate networks share single IP (NAT, VPN)
  ```
  100 users behind corporate proxy → appear as single IP
  Each user hits shared limit, legitimately throttled
  ```
- **Spoofing**: Attacker can send requests from multiple IPs (botnets do this)
- **Legitimate traffic**: CDNs, proxies have high request rates (need whitelist)

### Solutions

```
1. Use X-Forwarded-For header (real client IP from proxy)
   Risk: Header can be spoofed; trust only proxies you control
   
2. Distinguish between authenticated and unauthenticated:
   - Authenticated: Use per-user limit
   - Unauthenticated: Use per-IP limit (lower)
   
3. Whitelist known proxies (CDNs, monitoring services)
   
4. Graduated limits:
   - Per-IP burst limit (1000 req/sec)
   - If sustained, escalate to CAPTCHA or manual review
```

---

## Global Rate Limiting

Limit total system throughput (across all users).

### Use Cases

- **Capacity protection**: Prevent backend overload
- **Cost control**: Limit API costs (e.g., AWS API calls)
- **SLA enforcement**: Maintain p99 latency below 100ms

### Implementation

```python
GLOBAL_LIMIT = 100000  # 100k requests/sec across all users

def global_rate_limit(request):
    global_bucket = redis.get("global:rate_limit")
    
    if global_bucket.allow_request():
        return forward_to_backend(request)
    else:
        # Drop request or queue for retry
        return reject_with_503("Service overloaded, try again later")
```

### Coordination in Distributed System

```
Problem: Multiple servers need to share global limit

Option 1: Centralized (Redis)
  All servers query Redis for global bucket
  Pro: Accurate
  Con: Redis is bottleneck (query every request)

Option 2: Local + Sync
  Each server has local quota (global_limit / num_servers)
  Periodically sync across servers
  Pro: Distributes load
  Con: Requires synchronization, risk of over-allocation

Option 3: Sampling
  Monitor request rate on each server
  If rate > threshold, apply backpressure
  Pro: Low overhead
  Con: Approximate (not exact)
```

### Example: Sampling-Based Global Limit

```python
def sampling_based_global_limit(request, sampling_rate=0.1):
    # Check every 10th request (1/10 sampling rate)
    if random.random() > sampling_rate:
        return forward_to_backend(request)  # Assume OK
    
    # Every 10th request, check global health
    current_load = get_server_load()
    if current_load > THRESHOLD:
        return reject_with_503("Overloaded")
    else:
        return forward_to_backend(request)
```

---

## Adaptive Rate Limiting

Adjust limits dynamically based on system health.

### Mechanism

```
Monitor: CPU, memory, latency, error rate
If healthy: Allow up to normal limit
If degraded: Reduce limit gradually
If critical: Reduce limit aggressively or reject
```

### Example

```python
class AdaptiveRateLimiter:
    def __init__(self, base_limit=10000):
        self.base_limit = base_limit
        self.current_limit = base_limit
    
    def update_health(self, cpu, memory, p99_latency):
        if cpu > 90 and memory > 85:
            # Critical: Reduce limit by 50%
            self.current_limit = self.base_limit * 0.5
        elif cpu > 75 or memory > 70:
            # Degraded: Reduce limit by 20%
            self.current_limit = self.base_limit * 0.8
        elif p99_latency > 500ms:
            # Slow: Reduce limit by 10%
            self.current_limit = self.base_limit * 0.9
        else:
            # Healthy: Full limit
            self.current_limit = self.base_limit
    
    def allow_request(self):
        bucket = self.get_bucket()
        return bucket.tokens >= 1 and bucket.tokens <= self.current_limit

limiter = AdaptiveRateLimiter(base_limit=10000)
while True:
    metrics = collect_server_metrics()
    limiter.update_health(metrics.cpu, metrics.memory, metrics.p99)
    time.sleep(1)
```

### Advantages

- **Resilience**: Protects system from overload
- **Automatic**: No manual intervention needed
- **Graceful degradation**: Smooth reduction, not abrupt cutoff

### Disadvantages

- **Latency**: Detecting degradation takes time (lag in reaction)
- **Cascading**: If limit reduced too aggressively, causes more errors (vicious cycle)
- **Fairness**: Reducing limits affects all users equally (may prefer dropping low-priority users)

---

## Backpressure & Load Shedding

When rate limiter rejects requests, it's informing client to back off (backpressure). Load shedding is aggressive rejection of low-priority requests.

### Backpressure Signals

```http
HTTP 429 Too Many Requests
Retry-After: 60

X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 5234
X-RateLimit-Reset: 1625097600
```

Client should:
1. **Exponential backoff**: Wait and retry with increasing delay
2. **Circuit breaker**: Stop sending requests if repeatedly rate-limited (prevent cascade)

### Load Shedding Strategies

When system is overloaded, drop requests strategically:

```
Strategy 1: Strict priority
  Drop: Background jobs, analytics, cron tasks
  Keep: User-facing requests, payments

Strategy 2: Cost-based
  Drop: Expensive operations (image processing)
  Keep: Cheap operations (read queries)

Strategy 3: User-based
  Drop: Free-tier users
  Keep: Paid users, premium customers

Strategy 4: Random
  Drop: Random 10% of requests
  Pro: Fair, prevents thundering herd
  Con: Can surprise users with failures
```

### Implementation Example

```python
def load_shed_if_needed(request):
    current_load = get_system_load()
    
    if current_load < 70:
        return forward_to_backend(request)
    elif current_load < 85:
        # Shed 20% of low-priority requests
        if request.priority == "low" and random.random() < 0.2:
            return reject_with_503("Overloaded")
        else:
            return forward_to_backend(request)
    else:
        # Shed 50% of non-critical requests
        if request.priority in ["low", "medium"] and random.random() < 0.5:
            return reject_with_503("Overloaded")
        else:
            return forward_to_backend(request)
```

---

## Combined Strategy: Multi-Level Rate Limiting

Real systems use all strategies together:

```
1. Per-IP limit (coarse, block abusers)
   ↓ [IP not abusing]
2. Per-user limit (fine, quota enforcement)
   ↓ [User within quota]
3. Per-endpoint limit (cost-aware)
   ↓ [Endpoint not overloaded]
4. Global limit (capacity protection)
   ↓ [System healthy]
5. Adaptive limit (dynamic adjustment)
   ↓ [Load OK]
6. Forward to backend
```

### Configuration Example

```yaml
rate_limits:
  per_ip:
    burst: 1000/sec
    sustained: 100/sec
  per_user:
    free_tier: 1000/day
    paid_tier: 100000/day
    enterprise_tier: unlimited
  per_endpoint:
    GET /users: 1 token
    POST /images: 50 tokens
    GET /search: 5 tokens
  global:
    limit: 100000/sec
    breach_action: reject
  adaptive:
    enabled: true
    cpu_threshold: 80
    latency_threshold: 200ms
    reduction: 0.8  # 20% reduction
```

---

## Comparison of Strategies

| Strategy | Pros | Cons | Best For |
|----------|------|------|----------|
| **Per-user** | Fair, monetizable | Requires auth | Commercial APIs |
| **Per-endpoint** | Cost-aware, protects expensive ops | Complex to configure | Mixed-cost APIs |
| **Per-IP** | Prevents DDoS, no auth needed | Shared IPs, spoofing | Public APIs, DDoS prevention |
| **Global** | Simple, protects system | Unfair (all users affected) | Internal services, capacity management |
| **Adaptive** | Resilient, automatic | Latency in reaction, cascades | Large-scale systems |

---

## Real-World Examples

### GitHub API

```
Per-user limits (per hour):
  Unauthenticated: 60 requests/hour
  Authenticated: 5,000 requests/hour
  Specific endpoints (search): 30 requests/minute

Per-IP and global limits (DDoS protection):
  Aggressive brute-force attempts → temporary block
```

### AWS API Gateway

```
Per-API limits:
  10,000 requests/sec per account (soft limit)
  Burst capacity: 5,000 requests
  
Per-IP limits:
  Prevent single IP from overwhelming
  
Load shedding:
  If backend latency > threshold, shed low-priority requests
```

### Netflix (internal)

```
Adaptive rate limiting based on:
  Playback quality (reduce if network poor)
  Server load (reduce if under stress)
  User tier (priority to premium members)
```

---

## Related Fundamentals

- **[Token Bucket](token-bucket.md)**: Primary algorithm for rate limiting
- **[Sliding Window](sliding-window.md)**: Alternative algorithm for precise quota tracking
- **[Reliability & Resiliency](../reliability-and-resiliency/)**: Rate limiting as resilience mechanism
- **[Scalability & Load Balancing](../scalability-and-load-balancing/)**: Coordinating limits across servers
- **[Security](../security/)**: DDoS mitigation, abuse prevention

---

## Key Takeaways

1. **Multi-level rate limiting**: Combine per-user, per-endpoint, per-IP, and global limits.
2. **Adaptive rate limiting**: Adjust limits based on system health (CPU, latency, errors).
3. **Backpressure is key**: Return clear signals (429, retry headers) so clients can back off.
4. **Load shedding**: Strategically drop low-priority requests when overloaded.
5. **Trade-off**: Fairness vs. flexibility. Strict quotas (fair) vs. adaptive limits (flexible).

---

**Study Tips**

- Design rate-limiting strategy for a real API (GitHub, Twitter, Stripe). What limits would you implement?
- Trace adaptive rate limiting: system healthy → degraded → critical. How do limits change?
- Calculate per-IP limits for a public API with 1 million users.
- Debate: Should rate-limiting be per-user (fair) or per-IP (simple)? Why?

---

**Status**: ✅ Complete
