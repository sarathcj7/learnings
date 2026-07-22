# Indexing

## TL;DR

- **B-tree**: Default index, O(log N) lookup, good for range queries
- **Hash index**: O(1) exact match, doesn't support range queries
- **LSM tree**: Log-Structured Merge, write-optimized, used in RocksDB, LevelDB
- **Composite index**: Index on multiple columns, order matters (col1 then col2)
- **Covering index**: All columns needed by query in the index (never touch main table)

## B-tree Index

Most common, used by PostgreSQL, MySQL, Oracle.

### Structure

```
Root level:
        [M-Z]
       /     \
    [A-L]   [N-Z]
    /  |  \
  [A-C] [D-G] [H-L]

Search for "Charlie":
  Root: C < M, go left
  [A-L]: C > A and C < L, go middle child
  [D-G]: C < D, go left
  [A-C]: Found C → Return data pointer
```

### Properties

- **O(log N) access**: 1 billion rows = ~30 levels
- **Range queries efficient**: Find "Z", then scan forward for all Z-*
- **Sorted**: Data kept in order (helps sequential scans)

### Composite Index

Order matters!

```
Index on (userId, createdAt):

SELECT * FROM posts WHERE userId = 1 AND createdAt > '2026-07-01'
  → Uses index fully (filter by userId, then createdAt range)

SELECT * FROM posts WHERE createdAt > '2026-07-01' AND userId = 1
  → Also uses full index (query planner reorders)

SELECT * FROM posts WHERE createdAt > '2026-07-01'
  → Can't use index (userId is first, must be filtered first in B-tree order)
```

**Rule**: Put frequently filtered columns first.

---

## Hash Index

O(1) exact match, used for primary keys, doesn't support ranges.

```
Hash index on userId:
  hash(1) → bucket 1 → [ptr to row 1]
  hash(2) → bucket 5 → [ptr to row 2]

Query: WHERE userId = 1
  → hash(1) → direct lookup → O(1)

Query: WHERE userId > 1
  → Hash can't help (need to scan all buckets, defeats purpose)
  → Don't use hash index for this
```

---

## LSM Tree (Log-Structured Merge)

Write-optimized, used in RocksDB, Cassandra, HBase.

### How It Works

```
Write path:
  1. Write to in-memory buffer (MemTable)
  2. When full, flush to disk (Level 0 SSTable)
  3. Periodically merge levels (Level 0 → Level 1, etc.)

Read path:
  1. Check MemTable (fast)
  2. Check Level 0 SSTables (newest)
  3. Check Level 1 SSTables
  4. ...

Advantage: Writes are sequential disk I/O (fast)
Disadvantage: Reads require checking multiple levels (slower)
```

### Trade-offs

| Aspect | B-tree | LSM |
|---|---|---|
| **Write speed** | O(log N) random I/O (slow) | O(1) sequential I/O (fast) |
| **Read speed** | O(log N) (fast) | O(log N) but multiple levels (slower) |
| **Write amplification** | Low | High (rewrites data during compaction) |
| **Use case** | Traditional OLTP (balanced) | Write-heavy (fast ingestion) |

---

## Covering Index

All columns needed by query are in the index (never touch main table).

```
Query: SELECT name, email FROM users WHERE userId = 1

❌ Without covering index:
  Index on (userId): Finds userId=1 → Gets row pointer
  → Go to main table → Fetch name, email
  → 2 disk accesses

✅ Covering index on (userId, name, email):
  Index contains all three columns
  → Find userId=1 → name, email right in index
  → 1 disk access (index only)
```

**Cost**: Index is larger (stores extra columns), but read is faster (no table lookup).

---

## Index Maintenance

### Write Cost

Every INSERT/UPDATE/DELETE must update all indexes.

```
100 indexes on table → 100 index updates per write
Write latency: 100x slower (if index writes not parallelized)

Sweet spot: 3-5 indexes per table
Too many: Writes become bottleneck
```

### Index Bloat

Over time, dead rows accumulate (delete doesn't immediately free space).

**Mitigation**: VACUUM (PostgreSQL), OPTIMIZE (MySQL) periodically.

---

## When to Index

### Do Index

- WHERE clause columns (frequent filters)
- JOIN ON columns (foreign keys)
- ORDER BY columns
- Composite: Common filter combinations (name, age, city)

### Don't Index

- Low-cardinality (only 2 values: gender, isActive)
  - Index lookup slower than sequential scan
- Rarely queried columns
- Large columns (strings, BLOBs)
  - Index storage overhead huge

---

## Production Considerations

1. **Monitor index usage**: Some indexes never used, delete them
2. **Plan index strategy before scale**: Retroactively indexing 1B row table is painful
3. **Index selectivity**: If index returns 80% of rows anyway, sequential scan is faster
4. **Statistics**: DB needs accurate row counts to pick right index (ANALYZE in PostgreSQL)
5. **Partial indexes**: Index only rows matching condition, save space

---

## References

- "Designing Data-Intensive Applications" — Kleppmann, Chapter 3
- PostgreSQL documentation: "Indexes"

---

## Related Fundamentals

- [Relational Fundamentals](relational-fundamentals.md) – Primary/foreign keys, normalization
- [Sharding & Partitioning](sharding-and-partitioning.md) – Index strategy changes with sharding
