# Scaling Patterns

Strategies for scaling applications horizontally (add more servers) and vertically (make servers more powerful), and when to use each.

---

## TL;DR

- **Horizontal scaling**: Add more servers (preferred, handles growth better)
- **Vertical scaling**: Bigger hardware (limits: cost, single point of failure)
- **Stateless services**: Scale horizontally easily
- **Stateful services**: Harder to scale (need sticky routing or distributed state)
- **Database scaling**: Sharding (horizontal), replicas (read scaling), vertical (limited)
- **Cache layer**: Reduces load on database, enables high horizontal scaling

---

## Horizontal vs Vertical Scaling

### Vertical Scaling (Scale Up)

Buy bigger hardware:

```
Single server: 32 GB RAM, 8 CPU cores
  Handles: 10k req/sec
  Cost: $5k/month
  
Bigger server: 256 GB RAM, 64 CPU cores
  Handles: 80k req/sec (8x)
  Cost: $40k/month (8x)
```

**Limits**:
- Amazon's biggest instance: 24 TB RAM, 448 CPUs (can't buy bigger)
- Cost grows faster than capacity (logarithmic return)
- Single point of failure (if hardware fails, entire service down)

**Use case**: Justified only if horizontal scaling is impossible (e.g., single-threaded legacy code)

---

### Horizontal Scaling (Scale Out)

Add more servers:

```
10 servers × 10k req/sec each = 100k req/sec
  Cost: 10 × $5k = $50k/month
  
Add 10 more servers = 200k req/sec
  Cost: 20 × $5k = $100k/month
  
Add 100 servers = 1M req/sec
  Cost: 100 × $5k = $500k/month
```

**Advantages**:
- Linear cost scaling (more servers = proportionally more capacity)
- Fault tolerant (if 1 of 100 servers fails, 99% capacity remains)
- Can handle growth to extreme scale

**Challenges**:
- Requires load balancing
- Stateless code (sessions, caches managed centrally)
- Operational complexity

**Use case**: All modern high-scale systems

---

## Stateless vs Stateful

### Stateless Services (Easy to Scale)

Service holds no session data:

```
Server A: Process request → No memory of user
Server B: Process request from same user → No problem
Server C: Process request from same user → No problem

Each request routed independently via LB (round-robin, least connections)
```

**Example**: API server, video transcoding, image resizing

**Horizontal scaling**: Trivial (add servers, LB routes to any)

---

### Stateful Services (Hard to Scale)

Service holds session state:

```
Server A: User logs in → Session stored in memory
Server B: Same user requests → Session NOT in memory → FAIL

Solution required:
  1. Sticky routing (same user → same server always)
  2. Distributed session store (Redis) → removed statefulness
```

**Example**: Game server, WebSocket chat (before distributed queues)

**Option 1 - Sticky Routing**:
```
LB uses IP hash → user always routed to Server A
Problem: If Server A fails, user's session lost
Problem: Uneven load (some users' IPs hash to busy server)
```

**Option 2 - Distributed State**:
```
Session stored in Redis (separate service)
Any server can serve user (fetch session from Redis)
Scaling: Simply add more servers, all access same Redis
```

**Recommendation**: Always use distributed state (more flexible)

---

## Service Scaling Patterns

### Pattern 1: Shared Database

```
Load Balancer
  → Server A ─┐
  → Server B ─┼─→ Single Database
  → Server C ─┘
```

**Pros**: 
- Simple to understand
- All servers see same data

**Cons**:
- Database is bottleneck (5k connections max, typical)
- Doesn't scale beyond ~50 servers

**Use case**: Startups, small scale (< 50k req/sec)

---

### Pattern 2: Database Replication (Read Replicas)

```
Load Balancer (writes)
  → Server A ─→ Primary Database (writes)
                    ↓
              Replication stream
                    ↓
            Replica DB 1 (read-only)
            Replica DB 2 (read-only)
            
Load Balancer (reads)
  → Server B ─→ Replica 1
  → Server C ─→ Replica 2
```

**Pros**:
- Read load distributed to replicas
- Primary handles only writes

**Cons**:
- Eventual consistency (replicas lag behind primary)
- Replication can be bottleneck at extreme scale

**Scaling**: Works up to ~500k req/sec read, limited writes

---

### Pattern 3: Database Sharding

```
Load Balancer
  → Server A ─→ Shard 1 (users 0-100k)
  → Server B ─→ Shard 2 (users 100k-200k)
  → Server C ─→ Shard 3 (users 200k-300k)
```

**How it works**:
```
User ID 150k → hash(150k) % 3 = 1 → Shard 2
Server B queries Shard 2 (only relevant data)
```

**Pros**:
- Data distributed across multiple databases
- Each shard smaller, faster queries
- Linear scaling (add shard → add capacity)

**Cons**:
- Complex (cross-shard queries hard)
- Resharding is expensive (moving data between shards)
- Distributed transactions difficult

**Scaling**: Handles millions of req/sec (Netflix: 100k req/sec per shard × 500 shards)

---

### Pattern 4: CQRS (Command Query Responsibility Segregation)

```
Writes: 
  → Server → Primary Database
  → Publish event to Kafka
  
Reads:
  → Server → Materialized View (denormalized data in separate DB)
  → View updated by Kafka subscribers
```

**Pros**:
- Read path can be heavily optimized (denormalized)
- Different DB for writes vs reads
- Extreme scaling of read path

**Cons**:
- Complexity (eventual consistency between write and read DBs)
- More operational overhead

**Use case**: High-read systems (news feeds, leaderboards)

---

## Caching Strategy for Scaling

### Without Caching

```
1M users × 1 read per hour / 3600 sec = 278 req/sec from DB
  Seems fine...
  
But spike: 100k users online at same time, each refreshes every 5 sec
  100k / 5 = 20k req/sec to DB
  Database can't handle (typically 1-5k req/sec max)
  System becomes slow or crashes
```

### With Caching

```
Same 20k req/sec, but 95% hit rate on cache
  DB load: 20k × 5% = 1k req/sec
  Database easily handles
```

**Scaling with caching**:
```
Each server has local cache (L1): 1ms latency
  → Distributed cache (Redis, L2): 5ms latency
    → Database (L3): 10ms latency
      
Result: 95% of requests served from L1/L2, DB gets only 5%
Database remains bottleneck-free even at 1M+ req/sec
```

---

## Scaling Tiers Summary

### Tier 1: Single Server
- Max: 1k req/sec
- One machine, single database
- Good for prototypes, internal tools

### Tier 2: Multiple Servers + Shared DB
- Max: 10k req/sec
- LB routes to multiple stateless servers
- Single database shared
- Typical at Series A startup

### Tier 3: Read Replicas + Caching
- Max: 100k req/sec
- Replicas handle reads
- Redis caches hot data
- Typical at Series B

### Tier 4: Sharded Database
- Max: 1M+ req/sec
- Database partitioned by shard key
- Complex joins avoided
- Typical at scale (Uber, Airbnb, Netflix)

### Tier 5: Polyglot Persistence + CQRS
- Max: 10M+ req/sec
- Multiple databases for different data
- Read path optimized separately
- Typical at hyperscale (Google, Facebook)

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Horizontal** | Vertical | Scales better, fault tolerant, cost-effective |
| **Distributed state** | Sticky routing | Handles failures, allows any server |
| **Read replicas** | Sharding for reads | Simpler, good for read-heavy workloads |
| **Sharding** | Replicas | Required when writes are bottleneck |
| **Caching** | Scale database | Removes database bottleneck, enables 10x+ scale |
| **CQRS** | Single write DB | Complex but necessary at extreme scale |

---

## Related Fundamentals

- [Load Balancing Algorithms](load-balancing-algorithms.md) – Distributing to servers
- [Databases/Replication](../databases/replication.md) – Read replicas, failover
- [Databases/Sharding](../databases/sharding-and-partitioning.md) – Partitioning data
- [Caching](../caching/) – Reducing database load

---

**Status**: ✅ Complete. Covers horizontal, vertical, stateless, stateful, and caching patterns.

