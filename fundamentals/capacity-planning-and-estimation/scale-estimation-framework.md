# Scale Estimation Framework

How to estimate system capacity, predict bottlenecks, and plan infrastructure for scale. The systematic approach to "How much hardware do we need?"

---

## TL;DR

- **Bottom-up estimation**: Start with user metrics, derive QPS, storage, bandwidth
- **Capacity per component**: Know limits (DB ~5k QPS, cache ~100k QPS, LB ~1M QPS)
- **Headroom**: Design for 2x peak traffic (don't run at 100% utilization)
- **Bottleneck identification**: Database or network usually first
- **Scaling checklist**: Know which components scale horizontally vs vertically

---

## Estimation Formula

```
Step 1: Users
  Daily Active Users (DAU): 100M users

Step 2: Requests per User
  Each user does: 5 actions per day average
  Requests per user: 5 reads, 1 write
  Total: 6 requests per user per day

Step 3: Daily QPS
  100M users × 6 requests = 600M requests/day
  600M requests / 86400 seconds = 6,944 QPS average
  
  Peak QPS (4x average): 27,776 QPS peak

Step 4: Add Headroom (2x)
  Design for: 27,776 × 2 = 55,552 QPS capacity

Result: Capacity needed for 55k QPS
```

---

## Component Capacity Limits

### Database (Primary)

```
Single database server can handle:
  - Read-heavy (simple queries): 5k QPS
  - Mixed read/write: 2k QPS
  - Write-heavy: 1k QPS
  
Assuming:
  - Indexed queries (no table scans)
  - Simple queries (not complex joins)
  - Modern hardware (16 cores, SSD)

Example:
  55k QPS × 70% reads, 30% writes
  Read load: 38.5k QPS (need 8 databases)
  Write load: 16.5k QPS (need 17 databases)
  
  Solution: Primary + read replicas (primary handles writes, replicas handle reads)
```

---

### Cache (Redis)

```
Single Redis instance:
  - Simple get/set: 100k QPS
  - Pipelining: 1M QPS
  
Assuming:
  - Network latency included (1-5ms round trip)
  - 1KB average value
  - Commodity hardware (4 cores, RAM)

Example:
  55k QPS, 80% cache hit rate
  Cache load: 55k × 80% = 44k QPS (1 Redis instance, headroom)
  
  At 3x scale (165k QPS):
  Cache load: 132k QPS (need 2-3 instances, add for redundancy)
```

---

### Load Balancer

```
Single LB can handle:
  - Layer 7 (HTTP routing): 500k QPS
  - Layer 4 (TCP/UDP): 1M+ QPS
  
Assuming:
  - Modern LB (hardware or cloud-based)
  - Connection pooling to backend

Example:
  55k QPS needs single LB (headroom for 10x)
  
  At 1M+ QPS:
  Need GSLB across regions or LB farms
```

---

### Network Bandwidth

```
Formula:
  Bandwidth = QPS × Average response size

Example:
  55k QPS × 10KB average response
  = 550MB/s = 4.4 Gbps (4-10x hardware saturation)
  
  Needs: 10-25 Gbps network capacity
```

---

## Storage Estimation

```
Step 1: Data per user
  User profile: 1KB
  100M users × 1KB = 100GB

Step 2: Data created per day
  1M posts/day × 50KB per post = 50TB/day
  100B messages/day × 1KB per message = 100TB/day
  Total: 150TB/day

Step 3: Yearly retention
  150TB/day × 365 days = 54.75 PB/year
  
  But: Archive old data (1 year hot, 7 years cold)
  Hot storage: 54.75TB (1 year)
  Cold storage: 329TB (6 years old)
  
  Replication (3x): 54.75TB × 3 = 164TB (hot tier)
```

---

## Bottleneck Identification

Estimate each component, find the one that hits limit first:

```
System: 55k QPS, 2-year retention

Component capacity:
  Database: 2k QPS (write bottleneck)
  Cache: 100k QPS
  LB: 500k QPS
  Network: 4.4 Gbps (not bottleneck at this scale)
  Storage: 109TB hot + 655TB archive (not bottleneck)
  
First bottleneck: DATABASE at 2k QPS

Solution: 
  55k QPS ÷ 2k per DB = 28 database instances needed
  Use sharding (partition data)
  Each shard handles 2k QPS, stores subset of data
```

---

## Scaling Strategy per Bottleneck

### Bottleneck: Database Writes

```
Symptoms:
  Database CPU 100%, write latency > 100ms
  Can't add more read replicas (help reads, not writes)
  
Solutions:
  1. Sharding (partition by user ID, timestamp, etc.)
  2. Write cache (batching, async writes)
  3. Event sourcing (write only append log, read from cache)
  4. Vertical scaling (bigger server), but has limits
```

---

### Bottleneck: Cache Hits

```
Symptoms:
  Cache hit rate 30%, should be 80%+
  Database load high despite caching
  
Solutions:
  1. Improve cache strategy (longer TTL, warm cache proactively)
  2. Add more cache instances (horizontal scale)
  3. Improve cache key design (less fragmentation)
  4. Pre-compute popular items (ranking, recommendations)
```

---

### Bottleneck: Network

```
Symptoms:
  Bandwidth saturated, response times high
  Packet loss increasing
  
Solutions:
  1. Compress responses (gzip, brotli)
  2. CDN for static content
  3. Pagination/chunking (send less per request)
  4. More regional datacenters (closer users = less backbone traffic)
```

---

## Headroom & Safety

```
Design for 2x peak traffic:
  Peak: 55k QPS
  Design capacity: 110k QPS
  
Why 2x?
  - Unexpected traffic spikes
  - Cascading from other regions
  - Maintenance headroom (remove 1 server, still handle peak)
  - Auto-scaling ramp-up time (takes 30-60s to add servers)
  
At 100% utilization:
  - No room for failures
  - No room for spikes
  - Cascades likely
```

---

## Estimation Accuracy

```
Rule of thumb:
  First estimate: ±50% accuracy (could be 2x wrong)
  After load testing: ±20% accuracy
  After production: ±5% accuracy
  
Don't spend 10 hours on first estimate!
  Spend 30 min on initial estimate
  Validate with load testing
  Adjust based on real production metrics
```

---

## Checklist for Scale Estimation

- [ ] Define DAU (daily active users)
- [ ] Estimate requests per user (reads, writes)
- [ ] Calculate QPS and peak QPS (4x average)
- [ ] Apply 2x headroom
- [ ] Estimate database capacity needed
- [ ] Estimate cache capacity needed
- [ ] Identify bottleneck (usually database)
- [ ] Calculate storage (hot + archive)
- [ ] Calculate bandwidth
- [ ] Size LB and network
- [ ] Plan sharding strategy if needed

---

## Related Fundamentals

- [Load Balancing Algorithms](../scalability-and-load-balancing/load-balancing-algorithms.md) – Routing to scaled components
- [Scaling Patterns](../scalability-and-load-balancing/scaling-patterns.md) – Horizontal vs vertical
- [Databases/Sharding](../databases/sharding-and-partitioning.md) – Partitioning data
- [Caching](../caching/) – Cache sizing

---

**Status**: ✅ Complete. Covers formulas, limits, bottleneck identification, and headroom.

