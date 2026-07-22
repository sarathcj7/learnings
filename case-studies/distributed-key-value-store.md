# Distributed Key-Value Store (Dynamo-Style)

*Design a distributed KV store like DynamoDB, Cassandra. Billions of keys, high availability, eventual consistency, geo-distributed.*

## Problem Statement

Build a KV store (like DynamoDB/Cassandra) that can store billions of keys across multiple data centers. Must provide high availability even with node/region failures. Write/read consistency tunable. Think of it as building infrastructure that powers the database layer.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Keys stored** | 1B+ | Terabytes of data |
| **QPS** | 10M writes/sec | Extreme throughput |
| **Replication factor** | 3 (3 copies) | Availability |
| **Regions** | 5 global | Disaster recovery |

## Architecture

```
Consistent Hashing Ring (partitioned by key):
  Node A: Keys 0-200M
  Node B: Keys 200M-400M
  Node C: Keys 400M-600M
  ...

Each node replicates to N-1 successors:
  Node A → Replicas: B, C (3 copies total)
  Node B → Replicas: C, D
  
Client:
  Write: hash(key) → Node A
         Replicate to B, C (async)
         Return after 1+ acks (tunable)
  
  Read: hash(key) → Query any of {A, B, C}
        If stale: Read repair (sync back to all)
```

## Key Components

### 1. Consistent Hashing (With Replicas)

```
Write: hash("user:123") % ring = 5000 → Node A primary
       Also replicate to Node B (first successor), Node C (second successor)
       W=2 means: Wait for A and B to ack before returning
       
Read: hash("user:123") → Could read from A, B, or C
      R=2 means: Read from 2 nodes, return latest version
      W+R > N ensures read sees latest write (quorum reads)
```

### 2. Versioning (Vector Clocks)

```
Node A: SET key="v1" (timestamp, vector clock)
Node B: SET key="v2" (concurrent write from different client)

On read from both:
  [A:1, B:0] vs [A:0, B:1] → Conflicting versions
  Return both to client (application resolves)
  Or use LWW (last-write-wins): pick by timestamp
```

### 3. Anti-Entropy (Merkle Trees)

```
Background process:
  Compare hash of data ranges with replicas
  If mismatch: Sync that range
  
Example:
  Node A range [0-100k]: hash = "abc123"
  Node B range [0-100k]: hash = "def456" (out of date!)
  Merkle tree difference → Sync range [0-100k] from A to B
```

## Bottlenecks & Scaling

**At 10M+ QPS**: Network bandwidth saturated.

**Solution**:
- Multiple regions (write locally, replicate async)
- Compression
- Batching (write amplification)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Eventual consistency** | Strong consistency | Higher availability, lower latency |
| **Quorum reads** | Single-node reads | Read consistency without locking |
| **Vector clocks** | Last-write-wins | Detect true conflicts vs concurrent writes |

---

**Status**: ✅ Complete. Shows distributed KV fundamentals (Dynamo/Cassandra).
