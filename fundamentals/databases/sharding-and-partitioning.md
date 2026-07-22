# Sharding & Partitioning

## TL;DR

- **Sharding**: Split data across multiple databases. Scales writes and storage.
- **Hash sharding**: hash(key) % N shards. Even distribution, hard resharding.
- **Range sharding**: Ranges of key values per shard. Easy resharding, risk of hot spots.
- **Consistent hashing**: Minimize reshuffling when shards added/removed.
- **Hot spot**: One shard gets 90% of traffic. Mitigations: micro-shard, replicate, cache.

## Why Shard?

Single database can't handle:
- Write throughput (1000s QPS)
- Storage (100s GB)
- Latency (lock contention)

Solution: Split data across N shards, distribute load.

```
Single DB: 10k QPS → ~1000 QPS per shard
Storage: 1 TB → ~100 GB per shard (manageable)
Latency: Fewer locks, faster (contention reduced)
```

---

## Sharding Strategies

### Hash Sharding

```
Shard number = hash(key) % num_shards

User 1: hash(1) % 10 = 3 → Shard 3
User 2: hash(2) % 10 = 7 → Shard 7
User 100: hash(100) % 10 = 3 → Shard 3

Advantage: Even distribution (each shard gets ~equal load)
Disadvantage: Resharding is painful (add shard 11, all keys remap: hash % 11 ≠ hash % 10)
```

### Range Sharding

```
Shard 1: User IDs 1-1000000
Shard 2: User IDs 1000001-2000000
Shard 3: User IDs 2000001-3000000

Advantage: Easy resharding (add new range, no remapping)
Disadvantage: Hot spots (new users → Shard 3, old users → Shard 1, uneven load)
```

### Directory-Based Sharding

```
Lookup table: userId → ShardId
User 1 → Shard 3
User 2 → Shard 7

Advantage: Can rebalance by changing lookup table (no remapping)
Disadvantage: Lookup table is single point of failure, requires caching/replication
```

---

## Consistent Hashing

Minimize resharding by using a ring.

```
Ring (0 to 2^32-1):
    Shard B (100°)
        |
Shard C (250°) --- Shard A (20°)

Key "user:123" hashes to 50° → Maps to Shard A (next clockwise)

Add Shard D at 180°:
  Key at 50° → Still Shard A (no change)
  Key at 200° → Now Shard D (was C, moved)
  
Result: Only 1/N of keys move (not all)
```

With virtual nodes (each shard has many positions on ring), resharding is even smoother.

---

## Resharding

Adding a new shard:

```
1. Double-write: New writes go to old + new shard
2. Backfill: Scan old shard, copy data to new shard
3. Verify: Check both match
4. Cutover: Redirect reads to new shard
5. Cleanup: Drop old shard (after verification)

Duration: Hours to days (depending on data size)
Downtime: 0 (if careful)
```

---

## Hot Spots

One shard gets 90% of traffic.

### Causes

- **Celebrity users**: One user creates 10x more data
- **Temporal skew**: New data concentrated in latest shard (range sharding)
- **Bad hash function**: Some keys cluster together

### Mitigations

```
1. Micro-shard: Split hot shard into 2-10 sub-shards
   Shard 7 → Shard 7-1, 7-2, 7-3
   
2. Replicate hot key: Copy to all shards (cache strategy)

3. Local cache: Application-level cache for hot keys

4. Better hash function: If clustering, use better hash (but requires resharding)
```

---

## Cross-Shard Queries

Queries spanning multiple shards are expensive.

```
❌ Expensive:
  SELECT * FROM users WHERE status = 'premium'
  (Must query all 10 shards, merge results)

✅ Cheap:
  SELECT * FROM users WHERE userId = 123
  (Route to shard containing userId=123, single shard)
```

**Strategy**:
- Shard on frequently queried column (userId, tenantId)
- Accept that other queries are expensive (or use secondary index)

---

## Distributed Transactions Across Shards

Updating data on multiple shards:

```
Transfer $100: User A (Shard 1) → User B (Shard 3)
  UPDATE balance = balance - 100 WHERE userId = A [Shard 1]
  UPDATE balance = balance + 100 WHERE userId = B [Shard 3]

Problem: What if first succeeds, second fails?
Solution: 2-phase commit (2PC) or Saga pattern
```

See Distributed Transactions fundamentals for details.

---

## Production Considerations

1. **Shard key choice is critical**: Can't easily change later
2. **Plan for growth**: Start with 4 shards, shard limit = 2^N reshards
3. **Uneven shards**: Monitor shard sizes, rebalance if diverge
4. **Geo-sharding**: Can shard by geography (users in EU → Shard EU-1)
5. **Snowflake IDs**: Distributed ID generation that encodes shard ID

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- "Data Management at Scale" — Beyer et al.

---

## Related Fundamentals

- [Replication](replication.md) – Combine replication + sharding for scale + durability
- [Distributed Transactions](../distributed-transactions/) – Cross-shard consistency
- [Distributed Data Structures](../distributed-data-structures/consistent-hashing.md) – Consistent hashing deep dive
