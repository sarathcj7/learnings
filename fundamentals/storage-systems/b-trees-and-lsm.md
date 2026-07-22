# B-trees vs LSM Trees

Two fundamental indexing structures with opposite trade-offs: B-trees optimize reads, LSM trees optimize writes.

---

## TL;DR

- **B-tree**: Balanced tree, O(log N) reads and writes, read-optimized (PostgreSQL, MySQL)
- **LSM**: Log-Structured Merge, sequential writes, read cost higher (RocksDB, Cassandra)
- **Read/Write latency**: B-tree symmetric, LSM asymmetric (fast writes, slower reads)
- **Compaction**: LSM merges levels periodically, CPU-intensive
- **Bloom filters**: LSM optimization to skip unnecessary level checks
- **Write amplification**: LSM rewrites data during compaction (overhead)

---

## B-tree Structure

Balanced tree with sorted keys at each level.

### How It Works

```
B-tree of order 3 (max 2 keys per node):

                 [40, 80]
              /    |    \
          [10,20] [50,60] [90]
          /  |  \  /  |  \ 
        [5] [15][35][45][55] [85][95]

Search for 55:
  Root: 55 > 40, 55 < 80 → go right
  [50,60]: 55 > 50, 55 < 60 → go to child [55]
  Found!

O(log N) comparisons for N rows
1B rows = ~log₃(1B) ≈ 20 levels = 20 disk seeks
```

### Properties

```
Balanced:
  All leaves at same depth
  → Guarantees O(log N) even for worst case
  
Sorted:
  Keys in order at each level
  → Range queries efficient (find 55, scan forward for 55-60)
  
Multiple keys per node:
  Reduces tree height (fewer disk seeks)
  
In-place updates:
  Modify existing keys directly
  → No compaction needed
```

---

## B-tree Write Performance

```
Insert 100:
  1. Search for position (20 disk seeks)
  2. Insert into leaf node
  3. If leaf full → Split and propagate up (1-5 node updates)
  4. Fsync to disk (1-2 disk I/O)
  
Total: ~25 disk I/O operations
  All random I/O (slow on HDD, medium on SSD)
  
Latency: ~10-50ms on HDD, ~100μs on SSD
Throughput: ~100-1000 writes/sec on HDD
```

---

## LSM Tree (Log-Structured Merge)

Sequential writes to log, periodic merging of sorted files.

### How It Works

```
Write path:
  1. Append to in-memory buffer (MemTable)
  2. When full (64MB) → Flush to disk (Level 0 SSTable)
     (Sequential write, very fast)
  
Read path:
  1. Check MemTable (in-memory, fast)
  2. Check Level 0 SSTables (newest, smallest)
  3. Check Level 1 SSTables (older, medium)
  4. ...
  5. Bloom filter: Skip levels if key not present
  
Compaction (background):
  Level 0 has 4 SSTables → Merge with Level 1
  Level 1 becomes overlapping → Merge to Level 2
  ...
  (Reorganizes for faster reads later)
```

### Levels Example

```
Level 0: 4 SSTables (recent data, all keys possible)
  SSTable_0: [1-100]
  SSTable_1: [50-150]
  SSTable_2: [200-300]
  SSTable_3: [250-350]

Level 1: 10 SSTables (older, non-overlapping)
  SSTable_1_1: [1-50]
  SSTable_1_2: [51-100]
  SSTable_1_3: [101-150]
  ...

Read for key 75:
  Not in MemTable
  Check L0 SSTable_0: [1-100] → Bloom filter says maybe
  Check L0 SSTable_1: [50-150] → Bloom filter says maybe
  If not found → Check L1 (Bloom filter skips levels without key)
```

---

## LSM Write Performance

```
Insert 100:
  1. Append to MemTable (in-memory write, ~1μs)
  2. Return success immediately
  
When MemTable full:
  3. Flush to Level 0 SSTable (sequential disk write, ~1ms for 64MB)
  4. Fsync (1 disk I/O)
  
Total for single write: ~1μs + cost amortized
Effective throughput: ~1M writes/sec
Latency: ~100μs per write (write is amortized with flush)
```

---

## Compaction Process

LSM periodically reorganizes to maintain performance.

```
Compaction triggered when L0 has too many SSTables:

Before:
  L0: [1-100], [50-150], [200-300] (overlapping)
  L1: [1-50], [51-100], [101-150], ...
  
During compaction (merge L0 → L1):
  1. Read all keys from L0 and relevant L1 SSTables
  2. Merge-sort into new L1 SSTables
  3. Remove old SSTables
  
After:
  L0: (empty)
  L1: [1-50], [51-100], [101-150], [200-300], ...
      (Non-overlapping, sorted)

Time: ~100ms-1s for Level 0 compaction
CPU: High (merging is CPU-intensive)
I/O: 2-3x (reads old data + writes new data)
```

### Write Amplification

```
Compaction rewrites data multiple times:

Insert 1 key:
  1. Written to MemTable (1x)
  2. Flushed to L0 SSTable (1x)
  3. Compacted to L1 SSTable (1x)
  4. Compacted to L2 SSTable (1x)
  5. Final storage (1x)
  
Total disk I/O: 5x for single key!
  (Called "write amplification factor" = 5)

RocksDB typical: 5-10x amplification
  → 100M keys inserted = 500M-1B keys written to disk
  → SSD lifetime concern (limited writes)
```

---

## Bloom Filters

Probabilistic data structure to skip unnecessary level checks.

### How Bloom Filters Work

```
Bloom filter for keys [10, 50, 100]:
  Hash functions: h1, h2, h3
  Bit array: [0,0,0,0,0,0,0,0] (size 8)
  
Insert 10:
  h1(10) = 2 → set bit[2] = 1
  h2(10) = 5 → set bit[5] = 1
  h3(10) = 7 → set bit[7] = 1
  Result: [0,0,1,0,0,1,0,1]
  
Insert 50: (set bits 1,3,6)
Insert 100: (set bits 0,4,6)
Result: [1,1,1,1,1,1,1,1]

Query for 30:
  h1(30) = 2 → check bit[2] = 1
  h2(30) = 4 → check bit[4] = 1
  h3(30) = 6 → check bit[6] = 1
  All bits set → Maybe present (false positive possible)
  
Query for 75:
  h1(75) = 1 → check bit[1] = 1
  h2(75) = 3 → check bit[3] = 1
  h3(75) = 5 → check bit[5] = 1
  All bits set → Maybe present (false positive possible)
  
Query for 25:
  h1(25) = 0 → check bit[0] = 1
  h2(25) = 6 → check bit[6] = 1
  h3(25) = 2 → check bit[2] = 1
  All bits set → Maybe present
  
Actually: False positive rate ~1-5% with good parameters
Performance: Query in O(k) time (k hash functions, ~3-5)
```

### LSM Bloom Filter Optimization

```
LSM Levels:
  L0: 4 SSTables (may be missing key)
  L1: 100 SSTables (organized, key in at most 1)
  L2: 1000 SSTables (organized, key in at most 1)
  
Read for key 500:
  Check MemTable → Miss
  Check L0 Bloom filters → 3 maybe, 1 no → Check 3 SSTables
  Check L1 Bloom filters → 95 no, 5 maybe → Check 5 SSTables
  Check L2 Bloom filters → 999 no, 1 maybe → Check 1 SSTable
  
Without Bloom filters:
  Check all levels exhaustively
  → 1000+ disk seeks
  
With Bloom filters:
  Check ~9 SSTables
  → 9 disk seeks
  
Savings: 100x reduction in disk I/O!
```

---

## B-tree vs LSM Comparison

| Aspect | B-tree | LSM |
|---|---|---|
| **Read latency** | O(log N), ~10-100μs | O(log N + k), ~100-1000μs |
| **Write latency** | O(log N), ~10-100μs | ~1-10μs (amortized) |
| **Write throughput** | 1k-10k/sec | 1M+/sec |
| **Read throughput** | 10k-100k/sec | 10k-100k/sec |
| **Space overhead** | Low | Medium (compaction buffers) |
| **Write amplification** | 1x (no rewrites) | 5-10x (compaction) |
| **Predictability** | Consistent latency | Occasional spikes (compaction) |
| **Range queries** | Excellent (sorted) | Good (but slower) |
| **Cache efficiency** | Good (in-place updates) | Worse (compaction moves data) |

---

## When to Use Each

### B-tree
- **PostgreSQL, MySQL**: Balanced read/write workloads
- **Traditional OLTP**: Need low-latency reads and writes
- **Indexes**: Database indexes almost always B-trees
- **When**: Balanced workload (not write-heavy)

### LSM
- **RocksDB, LevelDB**: Write-heavy workloads (logging, time-series)
- **Cassandra, HBase**: High-throughput data ingestion
- **Kafka**: Store millions of log messages per second
- **Time-series databases**: Billions of events per day
- **When**: Throughput > latency

---

## Production Considerations

1. **Compaction tuning**: Monitor CPU spikes from LSM compaction
2. **SSD wear**: LSM's write amplification shortens SSD lifespan (rate-limit writes)
3. **Read latency spikes**: LSM can spike during compaction (pause traffic if needed)
4. **Cache size**: LSM benefits from larger block caches (to offset multiple level checks)
5. **Bloom filter tuning**: Adjust false positive rate (safety vs memory)

---

## Related Fundamentals

- [Embedded Databases](embedded-databases.md) – RocksDB/LevelDB use LSM
- [Indexing](../databases/indexing.md) – B-trees in databases
- [Databases](../databases/) – Storage layer comparison

---

**Status**: ✅ Complete. Covers structure, performance, compaction, bloom filters, and trade-offs.
