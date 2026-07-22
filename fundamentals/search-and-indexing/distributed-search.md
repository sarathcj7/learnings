# Distributed Search

## TL;DR

- **Sharding**: Split inverted index across N nodes (each node owns subset of terms or documents)
- **Replication**: Each shard replicated for redundancy + read scaling
- **Near-real-time**: Elasticsearch refreshes index every 1-2 seconds (not truly real-time)
- **Consistency**: Eventual consistency (replicas catch up asynchronously)
- **Query routing**: Query goes to all shards, results merged, ranked

## Why Distribute?

Single-node limitations:

```
Single Elasticsearch node:
  Max heap: 32GB
  Index size: ~100GB (3-4x documents)
  Problem: Can't fit large indexes in-memory buffer

Example: 1TB data
  Index size: 3-4TB
  Need to split across 4-5 nodes
```

---

## Sharding Strategies

### 1. Hash-based Sharding

Hash document ID, assign to shard.

```
Shard assignment:
  shard_id = hash(docID) % N

Document 1001: hash(1001) % 5 = 2 → Shard 2
Document 1002: hash(1002) % 5 = 4 → Shard 4
Document 1003: hash(1003) % 5 = 0 → Shard 0

Advantage:
  - Even distribution (random hash)
  
Disadvantage:
  - Resharding painful when N changes
  - hash(1001) % 5 ≠ hash(1001) % 6 (all keys move)
```

### 2. Range-based Sharding

Partition documents by a key range.

```
Sharding by user_id:
  Shard 0: user_id [1 - 1M]
  Shard 1: user_id [1M+1 - 2M]
  Shard 2: user_id [2M+1 - 3M]
  Shard 3: user_id [3M+1 - 5M]

Advantage:
  - Easy resharding (rebalance ranges)
  - Co-locate user data (user 1's docs → Shard 0)
  
Disadvantage:
  - Hot spot (if user 5M has lots of data, Shard 3 overloaded)
```

### 3. Consistent Hashing

Hash-based with resharding resilience.

```
Ring (0 - 2^32):
  hash(doc1) = 1000 → Shard A (owns 900-1100)
  hash(doc2) = 5000 → Shard C (owns 4500-5500)
  hash(doc3) = 3000 → Shard B (owns 2500-3500)

Add Shard D:
  Takes over range (owns 1100-2000)
  Only docs in 1100-2000 moved
  Docs outside range unaffected
  
Resharding: Only 1/N fraction of data moves (vs all with simple hash)
```

---

## Elasticsearch Sharding Model

Index = collection of shards + replicas.

```
Index: "products" with 3 shards, 2 replicas

Elasticsearch distributes:
  Shard 0: Holds 1/3 of inverted index
    - Primary Shard 0
    - Replica Shard 0
  Shard 1: Holds 1/3 of inverted index
    - Primary Shard 1
    - Replica Shard 1
  Shard 2: Holds 1/3 of inverted index
    - Primary Shard 2
    - Replica Shard 2

Total: 6 shard copies (3 primary + 3 replica)
Across 3-5 nodes (depends on replication factor)
```

### Document Routing

Which shard gets a document?

```
Default routing (hash document ID):
  shard_id = hash(docID) % num_shards

Custom routing (route by custom field):
  shard_id = hash(user_id) % num_shards
  
PUT /products/doc/123?routing=user_456
  → Document 123 goes to shard for user_456
  → All docs for user_456 co-located
```

---

## Replication & Consistency

### Write Path (Indexing)

```
Client writes document:
  1. Route to primary shard
  2. Primary indexing: Add to inverted index
  3. Replicate: Send update to all replica shards
  4. Wait for N replicas to ACK
  5. Return to client

Consistency levels:
  - ALL: Wait for all replicas (slow, safe)
  - QUORUM: Wait for majority (balanced)
  - ONE: Return after primary only (fast, risky)
  
Default: QUORUM (1 primary + 1 replica = 2 total, need 2 ACKs)
```

### Read Path

```
Client searches:
  1. Query sent to all N shards
  2. Each shard does local search (inverted index lookup)
  3. Collect results from each shard
  4. Merge and rank results globally
  5. Return top K to client

Replication benefit:
  - Read from primary or replica (load balance reads)
  - If primary fails, read from replica
```

### Replication Lag

```
t=0: Client writes "price=100" to primary shard 0
     Primary has price=100
     Replicas still have price=95

t=50ms: Replica 0 replicates the write
        Now has price=100

Problem: If client reads from replica during t=0-50ms, sees stale price=95
Solution: Most searches read stale replicas (OK for products), critical writes use primary
```

---

## Consistency: Near-Real-Time

Elasticsearch uses refresh intervals for index freshness.

```
Write to in-memory buffer:
  t=0: Document indexed in-memory buffer (not yet searchable)
  t=1000ms: Refresh triggers, buffer → filesystem cache (now searchable)
  t=5000ms: Fsync, cache → disk (durable)

Freshness: ~1 second latency before document searchable
Cost: Must wait for refresh interval

Tuning:
  refresh_interval = 1s (default, near-real-time)
  refresh_interval = 30s (fewer refreshes, higher throughput)
```

---

## Query Execution in Distributed Setting

### Scatter-Gather Pattern

```
Query: "nike running shoes"

1. Scatter (send to all shards):
   Shard 0: Search "nike running shoes" → top 100 results
   Shard 1: Search "nike running shoes" → top 100 results
   Shard 2: Search "nike running shoes" → top 100 results

2. Gather (merge results):
   Merge: 300 results from shards
   Rank by relevance (BM25)
   Return top 10

Problem: Each shard might return different top 100
Solution: Return top K from each, merge, re-rank globally (K >> final result size)
```

### Two-Phase Query (Especially for Aggregations)

```
Phase 1 (per shard):
  Shard 0: Aggregate user clicks in shard 0 → {user1: 10, user2: 5}
  Shard 1: Aggregate user clicks in shard 1 → {user1: 8, user3: 7}
  Shard 2: Aggregate user clicks in shard 2 → {user2: 6, user3: 9}

Phase 2 (merge):
  Merge results: {user1: 18, user2: 11, user3: 16}
  Return top users
```

---

## Scaling: Adding Shards

### Resharding Challenge

Adding shards requires redistributing data.

```
Original: 5 shards
New: 10 shards

Option 1: Re-index all data
  - Stop writes
  - Re-index all documents with new shard mapping
  - Resume writes
  - Downtime: Risky in production

Option 2: Progressive rebalancing
  - Run new shards in parallel
  - Gradually move documents (using consistent hashing)
  - Validate data integrity
  - Switch over after verification
  - No downtime, but complex
```

### Elasticsearch Splitting

```
Original index: 3 shards
Goal: 6 shards (double for growth)

API: POST /products/_split/products-large
  split_index.number_of_shards: 6

Process:
  1. Make index read-only (force merge)
  2. Split each shard to 2 shards
  3. Re-create index with 6 shards
  4. Reindex documents
  
Outage: Brief read-only window
```

---

## Hot Spots

Uneven load across shards.

```
Range sharding by user_id:
  Shard 0: [1 - 1M]
  Shard 1: [1M - 2M]
  Shard 2: [2M - 5M]  ← Hot shard (celebrity user 3M has 90% of posts)

Problem: Shard 2 CPU/memory high, other shards idle
Solution: 
  - Re-shard by hash (randomize)
  - Extract hot entity (celebrity posts) to separate index
  - Time-based sharding (by date, older data less queried)
```

---

## Production Considerations

1. **Replication factor**: 2-3 (1 primary + 1-2 replicas)
   - 1 replica: Lose one node, data still there
   - 2 replicas: Lose two nodes, data still there

2. **Shard size**: 10-50GB per shard
   - Too small: Overhead (many shards to manage)
   - Too large: Slow recovery when node fails
   
3. **Shard count**: Balance between parallelism and overhead
   - 3 shards: Good for distributed search
   - 10+ shards: Fine if cluster is large enough

4. **Replica placement**: Different nodes/racks/regions
   - Avoid placing primary + all replicas on same node
   - Geographic replication for disaster recovery

5. **Monitoring**: Track per-shard metrics
   - Shard size imbalance
   - Query latency per shard
   - Merge activity (segment compaction)

---

## References

- Elasticsearch documentation: "Shards and Replicas"
- "Designing Data-Intensive Applications" — Kleppmann, Chapters 6-7
- Google Bigtable paper: "Tablets and Sharding"

---

## Related Fundamentals

- [Full-Text Search](full-text-search.md) – Index structure within a shard
- [Query Optimization](query-optimization.md) – How queries execute across shards
- [Databases: Sharding](../databases/sharding-and-partitioning.md) – General sharding patterns
