# Distributed Cache

*Design a distributed cache system like Redis or Memcached that can serve millions of requests per second.*

## Problem Statement

Build a cache service that can be shared across multiple application servers. Must handle high throughput (1M+ QPS), high availability, and provide strong consistency guarantees. Think of it as building the infrastructure behind Redis cluster or Memcached.

---

## Clarifying Questions & Requirements

### Functional Requirements
- GET/SET operations (cache key-value pairs)
- DEL (delete key)
- TTL support (automatic expiration)
- Eviction policies (LRU, LFU)
- Replication across nodes
- Consistent hashing for scaling

### Non-Functional Requirements
- **Scale**: 1M+ QPS, 100GB+ data
- **Latency**: P99 < 5ms
- **Availability**: 99.99% uptime
- **Consistency**: Strong consistency (latest write visible immediately)
- **Durability**: Data survives restarts (optional, for Redis persistence)

---

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **QPS** | 1M peak QPS | 1,000,000 requests/sec |
| **Nodes needed** | 1M QPS / 100k QPS per node | 10 nodes |
| **Storage** | 100GB total data | 10GB per node |
| **Network** | 1M QPS × 1KB avg = 1GB/s | High bandwidth |
| **Connections** | 1000 clients × 100 connections | 100k concurrent connections |

---

## API Design

```
GET key
  Response: { value: string, found: bool }
  Latency: P99 < 2ms
  
SET key value [EX seconds]
  Response: { success: bool }
  Latency: P99 < 5ms
  
DEL key
  Response: { deleted: bool }
  
MGET key1 key2 ... keyN
  Response: { values: [v1, v2, ...] }
  (batching reduces round trips)
```

---

## High-Level Architecture

```mermaid
graph TB
    Client["Client<br/>(Application Server)"]
    LB["Load Balancer<br/>(Consistent Hashing)"]
    
    Node1["Cache Node 1<br/>(Primary for shard 1)"]
    Node2["Cache Node 2<br/>(Primary for shard 2)"]
    Node3["Cache Node 3<br/>(Primary for shard 3)"]
    
    Replica1["Replica Node 1-A<br/>(Backup shard 1)"]
    Replica2["Replica Node 2-A<br/>(Backup shard 2)"]
    Replica3["Replica Node 3-A<br/>(Backup shard 3)"]
    
    Client -->|hash(key)| LB
    LB -->|shard 1| Node1
    LB -->|shard 2| Node2
    LB -->|shard 3| Node3
    
    Node1 -->|replicate| Replica1
    Node2 -->|replicate| Replica2
    Node3 -->|replicate| Replica3
```

---

## Deep Dive: Core Components

### 1. In-Memory Data Structure

Each cache node stores data in memory with hash tables for O(1) access:

```
HashMap: {
  key1 → { value: "...", expiredAt: 1234567, accessCount: 100 }
  key2 → { value: "...", expiredAt: 1234568, accessCount: 50 }
  ...
}

On access: Update accessCount (for LFU), update accessTime (for LRU)
```

### 2. Eviction Engine

When node is full (reaches maxmemory), evict using LRU/LFU:

```
LRU Eviction Pseudocode:
  If memory_used > maxmemory:
    candidates = sample(keys, 100)  // Randomly sample 100 keys
    lru_key = candidates with oldest accessTime
    DELETE lru_key
    Repeat until memory_used < maxmemory
```

**Why sampling**: Scanning all keys is O(N), too slow. Random sampling is O(1) and "good enough".

### 3. Replication (Primary + Replica)

Primary node accepts writes and replicates asynchronously to replicas:

```
Client write: SET key value
  Primary: Store in memory immediately
  Primary → Replica: Send update async
  Replica: Store in memory
  
On primary failure:
  Cluster detects (heartbeat miss)
  Promotes replica to primary
  No data loss if replication was fast
```

### 4. Consistent Hashing

Map keys to nodes minimizing reshuffling when nodes added/removed:

```
Ring: 0 to 2^32-1

key "user:123" → hash() % ring = 5000 → Node A (next clockwise)
key "post:456" → hash() % ring = 15000 → Node B

Add Node D:
  Re-hash only ~1/N of keys (not all)
```

---

## Bottlenecks & Scaling

### At Current (1M QPS, 10 nodes)

**Bottleneck**: Network bandwidth (1GB/s = saturated at 10G Ethernet).

**Solution**: 
- Use 100G Ethernet
- Batch operations (MGET/MSET)
- Compression for large values

### At 10x (10M QPS)

**Bottleneck**: CPU (hashing, eviction sampling).

**Solution**:
- Add more nodes (25-30 total)
- Use faster CPUs (custom cache hardware)
- Shard hot keys separately

### At 100x (100M QPS)

**Bottleneck**: Single region capacity limit.

**Solution**:
- Multi-region replication (read-only replicas in other regions)
- Trade consistency for availability (local consistency)

---

## Failure Scenarios

### Node Failure

**Scenario**: Primary node for shard 1 crashes.

**Mitigation**:
- Cluster detects failure (3 missed heartbeats, ~3 sec)
- Promotes replica to primary
- Clients redirected (load balancer reroutes)

**RTO**: ~5 seconds  
**RPO**: ~100ms (replication lag)

### Network Partition

**Scenario**: Partition between primaries and replicas.

**Mitigation**:
- If primary isolated (minority): Stops accepting writes (consistency)
- If primary has majority: Continues (availability)

---

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Async replication** | Sync replication | Faster writes, acceptable data loss risk |
| **LRU approximation** | Exact LRU | O(1) sampling vs O(N) full scan |
| **Consistent hashing** | Naive modulo | Resharding doesn't affect all keys |
| **Strong consistency** | Eventual consistency | Cache should be immediately fresh |

---

## Related Fundamentals

- [Caching](../fundamentals/caching/distributed-caching.md) – Distributed caching patterns
- [Consistent Hashing](../fundamentals/distributed-data-structures/consistent-hashing.md) – Hashing algorithm
- [Replication](../fundamentals/databases/replication.md) – Primary-replica pattern

---

**Status**: ✅ Complete. Shows distributed system design for caching infrastructure.
