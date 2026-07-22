# Content Replication

Distribute content across multiple geographic regions to reduce latency, improve availability, and meet data residency requirements.

---

## TL;DR

- **Replication**: Copy data from origin to multiple regions
- **Pull-based**: Edge caches on-demand (lazy, cheaper, higher miss latency)
- **Push-based**: Proactively distribute before demand (eager, higher cost, instant)
- **Consistency**: Eventually consistent (replica lag acceptable for most use cases)
- **Multi-region**: Separate origins per region for sovereignty/compliance
- **Trade-off**: Cost vs latency; consistency vs availability

---

## Replication Strategies

### Strategy 1: Pull-Based Caching (On-Demand)

Content fetched from origin only when requested.

```
Scenario: Website with images

Day 1:
  User in Tokyo requests image.png
  → Image not at Tokyo CDN edge
  → Fetch from origin (US)
  → Cache at Tokyo edge (takes 300ms)
  → Return to user

Day 2:
  User in Tokyo requests same image
  → Image cached at Tokyo edge
  → Return immediately (10ms)
  
User in London requests image.png
  → Image not at London CDN edge
  → Fetch from origin (US)
  → Cache at London edge
  → Return to user
```

**Pros**:
- Cheap (no bandwidth until needed)
- Storage efficient (only popular content cached)
- Simple (no coordination needed)

**Cons**:
- First user in region gets high latency (cache miss)
- Origin traffic spikes during popular events

**Use case**: Long-tail content, variable demand, cost-sensitive

---

### Strategy 2: Push-Based Replication (Pre-Population)

Content proactively distributed to all regions before demand.

```
Scenario: Product launch announcement (known high-traffic event)

Before launch:
  1. Upload content to origin
  2. Trigger distribution job
  3. Content pushed to all CDN edges (20+ locations)
  4. All edges cache in parallel (5-10 minutes)
  
Launch time:
  Millions of users worldwide
  All request from their nearest edge
  All edges have content cached
  Zero origin load (all served from edge)
  All users get instant response (<50ms)
```

**Pros**:
- No cache misses at launch (prepared)
- Origin protected from spike
- Predictable latency

**Cons**:
- Higher cost (bandwidth for all regions upfront)
- Wastes resources if demand low
- Requires advance planning

**Use case**: Predictable high-traffic events, sensitive content, compliance requirements

---

### Strategy 3: Hybrid (Tiered)

Combine push and pull strategies.

```
Tier 1 - Popular content (top 1% by traffic):
  Push to all 200 edges
  Cost: High, but justified by traffic volume

Tier 2 - Semi-popular (top 1-10%):
  Push to 20 regional hubs
  On-demand to local edges

Tier 3 - Long-tail (bottom 90%):
  On-demand pull only
  Never pushed

Result: Balanced cost/performance
  Popular content instant everywhere
  Rare content still available (slower for first user)
```

---

## Multi-Region Content Distribution

### Architecture: Separate Origins Per Region

```
Global request: GET /products from user in Australia

Global routing:
  1. Check location (geolocation API)
  2. Route to regional origin

Example infrastructure:
  North America origin: origin-us.example.com
  Europe origin: origin-eu.example.com
  Asia-Pacific origin: origin-ap.example.com

Australia user:
  → origin-ap.example.com
  → Latency: 50ms (local)
  → vs origin-us: 300ms (intercontinental)
```

**Advantages**:
- Data residency (data stays in region)
- Compliance (GDPR, data localization laws)
- Reduced latency (regional origin)
- Reduced bandwidth (local traffic stays local)

**Disadvantages**:
- Complexity (manage multiple origins)
- Consistency challenges (sync between regions)
- Higher operational cost (multiple databases)

---

## Consistency Models

### Eventual Consistency

Replicas lag behind origin, but eventually converge.

```
Timeline:
  T=0: Write to origin (US): price = $100
  T=0: Origin acknowledges write
  
  T=1ms: User in US reads: $100 ✅
  T=100ms: Replica in EU still has $99 ❌
  T=200ms: Replica in AP still has $99 ❌
  
  T=1000ms: Replication catches up
  T=1000ms: EU replica: $100 ✅
  T=1000ms: AP replica: $100 ✅
```

**Acceptable for**:
- Content (blog posts, articles)
- Pricing (minor staleness acceptable)
- Analytics (eventual aggregation)
- Social media (likes, follows)

**Not acceptable for**:
- Financial transactions (must be consistent)
- Inventory (oversell risk)
- User authentication (must be current)

---

### Strong Consistency

Replicas always synchronized with origin.

```
Write to origin: price = $100
  → Wait for all replicas to acknowledge
  → Expensive (network round-trip to all regions)
  → Slow (synchronous)
  
All replicas updated:
  US: $100 ✅
  EU: $100 ✅
  AP: $100 ✅
  
All users see latest value

Cost: Extra latency (wait for slowest replica)
  Typical: 100-500ms extra per write
```

**Acceptable for**:
- Financial transactions
- Inventory updates
- Critical configuration

**Trade-off**: Consistency vs Availability (CAP theorem)
- Consistent → Slow (wait for all)
- Available → Eventually consistent (accept lag)

---

## Replication Patterns

### Pattern 1: Write-Through

```
Application:
  1. Write to origin
  2. Wait for response
  3. Return to user
  
Behind scenes:
  Origin accepts write
  Asynchronously replicates to all regions
  Regions acknowledge replication
  
Latency: Only origin write time (fast)
Consistency: Strong at origin, eventual elsewhere
```

**Tradeoff**: Fast writes, eventual consistency for reads

---

### Pattern 2: Write-Back (Delayed)

```
Application:
  1. Write to local cache (memory)
  2. Return to user immediately (fast!)
  3. Asynchronously flush to origin and replicas
  
Latency: 1-10ms (just cache write)
Consistency: Eventual (risky if server crashes)
  
Disadvantage: Data loss if crashes before flush
  Solution: Write-ahead log (log write before cache)
```

**Tradeoff**: Very fast writes, risk of data loss

---

### Pattern 3: Active-Active Replication

```
Multiple origins accept writes:
  User in US → Write to US origin
  User in EU → Write to EU origin
  User in AP → Write to AP origin
  
Each origin replicates to others:
  US → EU, AP
  EU → US, AP
  AP → US, EU

Advantage: Write latency low (any origin nearby)
Disadvantage: Conflict resolution needed
  User A updates price to $100 (US)
  User B updates price to $50 (EU)
  Regions conflict: Which is correct?
```

**Conflict resolution**:
1. Last-write-wins: Later timestamp wins
2. Vector clocks: Detect causality
3. Application logic: Merge intelligently

---

## Real-World Distribution Patterns

### Pattern: Content Versioning

```
Immutable content with versions:
  /images/logo-v1.png (old)
  /images/logo-v2.png (new)

Deployment:
  1. Push logo-v2.png to all edges (new version)
  2. Update HTML to reference logo-v2.png
  3. Users request logo-v2.png from edge (cache hit)
  
Advantage: No consistency issues (versioned URLs)
  v1 and v2 coexist indefinitely
  Users see old or new based on when they load
  No stale content issues
```

### Pattern: Cache Invalidation on Deploy

```
Website deploy: CSS changed

Problem: Users cached old CSS
  Browser caches: index.html (old reference)
  Edge caches: styles.css (old version)
  User sees broken layout

Solution: Version CSS filenames
  Old: /styles.css
  New: /styles-v2.abc123.css (hash in filename)
  
  Old HTML: <link href="/styles.css">
  New HTML: <link href="/styles-v2.abc123.css">
  
Deploy: Release new HTML
  Browsers fetch new HTML
  New HTML references new CSS
  All browsers get new CSS
  Zero stale CSS issues
```

---

## Production Considerations

### 1. Replication Lag Monitoring

```
Keep track of replica freshness:
  Freshness = time since last update
  
Monitor:
  Max freshness: 100ms? 1s? 10s?
  Alert if exceeds threshold
  
High lag indicates:
  Network issues
  Replica overload
  Replication bottleneck (fix before outage)
```

### 2. Failover Strategy

```
Origin fails (origin-us.example.com down)

Automated failover:
  1. Detect failure (health checks fail)
  2. Promote replica to primary (origin-eu becomes writable)
  3. Redirect writes to new primary
  4. Resume from backup origin-us
  
Time to failover: < 30 seconds (automated)
Data loss: Minimal (replicated before failure)
```

### 3. Bandwidth Costs

```
Replication bandwidth costs:
  Origin → Replica 1: $0.02/GB
  Origin → Replica 2: $0.02/GB
  ...
  
1 TB dataset × 20 regions = 20 TB egress
  Cost: $400-1000/month (depending on region)

Strategies to reduce:
  1. Compression (gzip reduces 50-80%)
  2. Deduplication (send deltas, not full copies)
  3. Selective replication (only popular content)
  4. Lazy replication (on-demand pull)
```

### 4. Quota Management

```
Regional quotas:
  North America: 500 GB storage, 100 Gbps bandwidth
  Europe: 300 GB storage, 50 Gbps bandwidth
  Asia: 300 GB storage, 50 Gbps bandwidth

Monitor usage:
  Approaching limit → Cache eviction starts
  Exceeding limit → Requests might fail
  
Capacity planning:
  Growth rate: +20% YoY?
  Plan ahead: Add capacity 3-6 months early
```

---

## Comparison: Pull vs Push

| Aspect | Pull | Push | Hybrid |
|---|---|---|---|
| **First-user latency** | High (cache miss) | Low (cached) | Medium (tier-based) |
| **Origin load** | High (misses) | Low (pre-cached) | Medium (tiered) |
| **Cost** | Low (on-demand) | High (upfront) | Medium |
| **Bandwidth waste** | None (only used) | Possible (unused data) | Minimal |
| **Scalability** | Unlimited (on-demand) | Limited (pre-plan) | Good |
| **Setup complexity** | Simple | Complex (planning) | Medium |
| **Best for** | Variable demand | Predictable events | Mixed workloads |

---

## References

- "Designing Data-Intensive Applications" — Kleppmann, Chapter 5 (Replication)
- AWS S3 Transfer Acceleration documentation
- Cloudflare cache purge/prefetch API

---

## Related Fundamentals

- [CDN Architecture](cdn-architecture.md) – Edge caching, cache headers
- [Edge Computing](edge-computing.md) – Compute at edge
- [Databases/Replication](../databases/replication.md) – Database replication strategies
- [Scalability/Patterns](../scalability-and-load-balancing/scaling-patterns.md) – Scaling architectures

---

**Status**: ✅ Complete. Covers pull-based, push-based, multi-region, and consistency strategies.
