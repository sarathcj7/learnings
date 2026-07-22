# Database Sizing & Optimization

Calculating database hardware needs, identifying bottlenecks, and optimizing to get maximum performance before scaling out.

---

## TL;DR

- **Row/table size**: Calculate bytes per record, multiply by count
- **QPS vs latency**: Different hardware tuning for different workloads
- **Working set**: Keep hot data in RAM (most critical factor)
- **Query optimization**: Often 100x better than hardware upgrade
- **Index strategy**: Right indexes = 10x performance, wrong indexes = disaster

---

## Storage Sizing

### Simple Calculation

```
Table: users
  Columns: id (8B), email (50B), name (50B), created_at (8B), ...
  Row size: 200 bytes

100M users:
  Storage: 100M × 200B = 20GB
  With replication (3x): 60GB
  With backup (2 copies): 120GB
```

---

### Accounting for Growth

```
Current: 20GB
Growth rate: 10% per year
5-year projection:
  Year 0: 20GB
  Year 1: 22GB
  Year 2: 24GB
  Year 3: 27GB
  Year 4: 30GB
  Year 5: 33GB

Capacity plan: Provision 50GB (headroom for 50% growth)
```

---

## RAM Sizing (Most Critical)

### Working Set

The dataset size that fits in RAM:

```
Database server: 256GB RAM

Queries mostly access:
  Hot data: Recent 1000 posts, 10000 active users
  Hot data size: 5GB

Working set: 5GB (fits in RAM)
Result: Queries serve from RAM (~1ms latency)

Growth: Working set grows to 100GB
Result: Data falls out of RAM, hits disk (~10ms latency)
Problem: 10x slower!
```

---

### Rule of Thumb

```
Working set should be ≤ 80% of RAM

Example:
  Database server: 256GB RAM
  Safe working set: 200GB
  
  If working set > 200GB:
    Either:
      1. Add more RAM (expensive)
      2. Shard data (split into multiple DBs)
      3. Archive old data (cold storage)
```

---

## CPU Sizing

### Per-Query CPU Cost

```
Simple query (indexed lookup): 0.1ms CPU
  100 concurrent queries: 0.1 × 100 = 10ms total (1 core utilization)
  
Complex query (join, aggregation): 10ms CPU
  100 concurrent queries: 10 × 100 = 1000ms (8 cores at 100%)
  
Result: Complex queries consume much more CPU
```

---

### Database Workload

```
Read-heavy (simple lookups):
  1000 QPS × 0.1ms = 100ms CPU utilization across cores
  Need: 2-4 cores
  
Write-heavy (INSERT with indexes):
  100 QPS × 5ms = 500ms CPU + lock management
  Need: 8-16 cores
  
Mixed complex (joins, aggregations):
  50 QPS × 50ms = 2500ms = 2.5 seconds per core
  Need: 32-64 cores (handle 50 QPS with headroom)
```

---

## Disk I/O Sizing

### IOPS (Input/Output Operations Per Second)

```
SSD:
  Random read: 100k IOPS
  Sequential read: 500MB/s
  
HDD (traditional spinning disk):
  Random read: 200 IOPS
  Sequential read: 100MB/s
  
Example:
  Database handles 5000 random reads/sec
  SSD: 5000 / 100k = 5% utilization (plenty of headroom)
  HDD: 5000 / 200 = 2500% utilization (!!! OVERLOADED)
  
  Result: Must use SSD for random I/O workloads
```

---

### I/O Size

```
Query execution:
  1. Fetch index from disk (random I/O, 4KB page)
  2. Fetch table row from disk (random I/O, 4-8KB pages)
  3. Execute join to related table (more random I/O)
  
Total I/O per query: 3-5 disk operations

At 5000 QPS:
  5000 × 3 I/O per query = 15k IOPS
  SSD can handle easily
  
At 50k QPS:
  50k × 3 = 150k IOPS
  SSD still OK (max 100k IOPS, need 1.5x more SSDs)
```

---

## Query Optimization

### Before Hardware

```
Slow query:
  SELECT * FROM users 
  INNER JOIN orders ON users.id = orders.user_id
  INNER JOIN products ON products.id = orders.product_id
  WHERE users.created_at > ?
  
Execution: 5 seconds
  
Optimization:
  1. Add index on users.created_at
  2. Reorder joins (products first, smaller result set)
  3. Fetch only needed columns (not *)
  
New execution: 50ms (100x improvement!)
```

---

### Index Strategy

**Best case with index**:
```
Query: WHERE user_id = 123
Index on user_id:
  B-tree lookup: O(log N) = 15 seeks (10M rows)
  Fetch matching rows: O(K) = 100 rows
  Total: 115 row fetches
  Time: 115ms (1ms per row)
```

**Worst case without index**:
```
Query: WHERE user_id = 123
No index (table scan):
  Must read all 10M rows
  Find matching rows: 100 rows
  Time: 10 seconds (1ms per row × 10M rows)
  
Improvement with index: 100x faster!
```

---

### Wrong Indexes

```
Table: users (100M rows)
  Indexes: First name (useless), last name, email
  
Query: WHERE user_id = 123 AND created_at > ?
  No index on user_id (scans entire table!)
  No index on created_at (scans entire table!)
  
Time: 10 seconds (table scan)

Solution:
  Add composite index: (user_id, created_at)
  Time: 1ms (B-tree lookup)
  
Improvement: 10,000x faster! (one good index worth more than 10 bad ones)
```

---

## Database-Specific Sizing

### MySQL

```
Small deployment: 1 primary + 1 replica
  Primary: 16 cores, 128GB RAM, NVMe SSD
  Replica: 8 cores, 64GB RAM, SATA SSD
  
Medium: 1 primary + 2-3 replicas
  Primary: 32 cores, 256GB RAM, NVMe SSD
  Replicas: 16 cores, 128GB RAM, SATA SSD
  
Large: Sharded (10+ shards)
  Each shard: 32 cores, 512GB RAM, NVMe SSD
  Max: ~5k QPS per shard (50k+ total)
```

---

### PostgreSQL

```
Similar to MySQL, but typically:
  More CPU-efficient queries (better query planner)
  Slightly higher RAM usage (MVCC overhead)
  
Small: Same as MySQL
Medium: Slightly smaller hardware (better efficiency)
Large: Sharded, ~8k QPS per shard (better than MySQL)
```

---

### NoSQL (DynamoDB, MongoDB)

```
DynamoDB (serverless):
  Pay per RCU/WCU provisioned
  1 RCU = 1 consistent read per second of 4KB
  1 WCU = 1 write per second of 1KB
  
  Example:
    5000 reads/sec × 1 RCU = 5000 RCU
    500 writes/sec × 1 WCU = 500 WCU
    Cost: ~$2.5k/month
    
MongoDB (self-hosted):
  Similar to MySQL in hardware needs
  But scales more easily horizontally (sharding built-in)
```

---

## Capacity Planning Timeline

```
Current load: 5000 QPS
Hardware: 32-core, 256GB RAM server
Utilization: 40% CPU, 50% RAM

Growth rate: 10% per quarter
Timeline to capacity:
  Q1: 5500 QPS (44% CPU, 55% RAM) — OK
  Q2: 6050 QPS (48% CPU, 60% RAM) — OK
  Q3: 6655 QPS (53% CPU, 66% RAM) — Getting close
  Q4: 7320 QPS (59% CPU, 72% RAM) — ACTION: Plan scaling

Action options:
  1. Add read replicas (for read-heavy, now)
  2. Add more RAM (cheaper than new server)
  3. Begin sharding planning (longer timeline)
  4. Optimize queries (cheapest, but only 10-20% improvement typical)
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Query optimization** | Vertical scale | 100x improvement, much cheaper |
| **Right indexes** | More servers | Fix root cause, not symptom |
| **SSD** | HDD for random I/O | 500x more IOPS, worth the cost |
| **Sharding** | Vertical scaling | Enables horizontal growth beyond hardware limits |
| **Replication** | Single server | Data safety, read scaling |

---

## Database Capacity Checklist

- [ ] Row/table size calculated
- [ ] Storage needs for 2-3 year projection
- [ ] RAM: Working set ≤ 80% of total
- [ ] CPU cores: Right-sized for workload (4-64)
- [ ] Disk: SSD for random I/O workloads
- [ ] Key queries: Have appropriate indexes
- [ ] Replication: Primary + at least 1 replica
- [ ] Monitoring: Track CPU, RAM, IOPS usage
- [ ] Sharding: Plan when single instance hits QPS limit

---

## Related Fundamentals

- [Databases/Indexing](../databases/indexing.md) – Index design strategy
- [Databases/Sharding](../databases/sharding-and-partitioning.md) – Horizontal scaling
- [Capacity Planning/Scale Estimation](scale-estimation-framework.md) – Top-down sizing

---

**Status**: ✅ Complete. Covers storage, RAM, CPU, disk, query optimization, and sizing.

