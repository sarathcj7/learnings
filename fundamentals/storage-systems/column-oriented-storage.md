# Column-Oriented Storage

Store data by column instead of row. Used in ClickHouse, Apache Parquet, Vertica.

---

## TL;DR

- **Row-oriented**: Store row 1 (all columns), then row 2 (all columns). For OLTP (fast updates)
- **Column-oriented**: Store column 1 (all rows), then column 2 (all rows). For OLAP (fast analytics)
- **ClickHouse**: Columnar database, sub-second queries on billions of rows
- **Parquet**: Columnar file format for data warehouses
- **Compression**: Columns compress 10-100x better (same type, repetitive data)
- **Query selectivity**: Avoid reading unused columns (massive speedup)

---

## Row-Oriented vs Column-Oriented

### Row-Oriented Storage (Traditional)

```
Table: Events
  eventId, userId, timestamp, action, amount

Storage layout (row-major):
  Row 1: [1, 100, 2026-07-23T10:00, "click", 0]
  Row 2: [2, 101, 2026-07-23T10:05, "view", 0]
  Row 3: [3, 100, 2026-07-23T10:10, "purchase", 99.99]
  Row 4: [4, 102, 2026-07-23T10:15, "click", 0]
  ...
  
Disk layout:
  [1][100][2026-07-23T10:00]["click"][0]
  [2][101][2026-07-23T10:05]["view"][0]
  [3][100][2026-07-23T10:10]["purchase"][99.99]
  [4][102][2026-07-23T10:15]["click"][0]

Query: SELECT SUM(amount) WHERE action = "purchase"
  Read all 1B rows (want only 1 column for each)
  I/O: 1B rows × 5 columns × 8 bytes = 40GB read
  (Wasted: Read userId, timestamp, action when only need amount)
  
Latency: 40GB / 500MB/s = 80 seconds
```

### Column-Oriented Storage

```
Storage layout (column-major):
  Column eventId:   [1, 2, 3, 4, ...]
  Column userId:    [100, 101, 100, 102, ...]
  Column timestamp: [2026-07-23T10:00, 2026-07-23T10:05, ...]
  Column action:    ["click", "view", "purchase", "click", ...]
  Column amount:    [0, 0, 99.99, 0, ...]

Disk layout (compressed):
  eventId column: [Compressed binary]
  userId column:  [Compressed binary]
  action column:  [Compressed binary]
  amount column:  [Compressed binary]
  
Query: SELECT SUM(amount) WHERE action = "purchase"
  1. Read action column (scan for "purchase" rows)
  2. Read amount column (only rows matching action)
  
I/O: ~2 columns instead of 5
I/O: 2 × 1B rows × 8 bytes = 16GB uncompressed → 0.5GB compressed
  
Latency: 0.5GB / 500MB/s = 1 second (80x faster!)
```

---

## Compression Benefits

Column-oriented achieves much better compression:

```
Row-oriented compression:
  [eventId: varies, userId: varies, action: varies, amount: varies]
  Data highly heterogeneous → Limited compression
  Compression ratio: 2-5x
  
Column-oriented compression:
  amount column: [0, 0, 99.99, 0, 0, 50, 0, 0, 49.99, ...]
  Highly repetitive (lots of 0s) → Excellent compression
  Techniques:
    - Dictionary encoding: [0→0x01, 99.99→0x02, 50→0x03, ...]
      Store: 1B values → 1B bytes (vs 8B each)
    - Run-length encoding: [0x01, 0x01, 0x01, 0x02, 0x01, 0x01]
      Store: 100 zeros as "0x01×100" (vs 100 bytes)
  
Compression ratio: 10-100x
```

### Real Numbers

```
Table: 1 billion events, 10 columns, 8 bytes each

Row-oriented (PostgreSQL):
  Raw size: 1B rows × 10 columns × 8 bytes = 80GB
  Compressed: 80GB / 3 = 27GB
  
Column-oriented (ClickHouse):
  Raw size: 80GB
  Compressed: 80GB / 30 = 2.7GB
  Savings: 10x vs row-oriented!
  Query time: 0.5 seconds vs 40 seconds
```

---

## ClickHouse

High-performance column database by Yandex.

### Architecture

```
ClickHouse cluster:
  Node 1: Store replicas of columns
  Node 2: Store replicas of columns
  ...
  
Write path:
  INSERT events VALUES (1, 100, 2026-07-23T10:00, "click", 0)
  ↓
  Append to column buffers in memory
  ↓
  When buffer full → Flush to disk (one file per column)
  
Read path:
  SELECT SUM(amount) WHERE action = "purchase"
  ↓
  Load action column
  ↓
  Filter for "purchase"
  ↓
  Load amount column (only matching rows)
  ↓
  Sum and return result
```

### Performance

```
Query: Aggregate 1B rows, 10 columns, sum 1 column, filter on 1 column

Row-oriented (PostgreSQL):
  - Must load all 10 columns
  - Latency: 20-40 seconds
  
ClickHouse:
  - Load 2 columns (action, amount)
  - Compression: 30x reduction
  - Latency: <1 second
  - Throughput: 1B rows/sec

Real-world: Analytics on 100B+ events in seconds
```

### Limitations

```
❌ Not suitable for:
  - High-frequency updates (column-oriented is write-heavy)
  - OLTP (optimized for batch inserts, not random updates)
  - Few rows, many columns (column format overhead)
  
✅ Great for:
  - Time-series data (events, metrics, logs)
  - Aggregation queries (SUM, COUNT, AVG)
  - Range queries (WHERE timestamp > X)
  - Immutable data (append-only workloads)
```

---

## Parquet Format

Columnar file format for data warehouses (Hadoop, Spark, Dremio).

### Structure

```
Parquet file format:

[Magic: "PAR1"]
[Column Group 1]
  [Column 1: values, stats, metadata]
  [Column 2: values, stats, metadata]
  [Column 3: values, stats, metadata]
[Column Group 2]
  [Column 1: values, stats, metadata]
  ...
[Metadata]
  [Schema]
  [Statistics (min, max, count per column)]
[Magic: "PAR1"]

Key features:
  - Columnar: Each column stored separately
  - Compressed: Dictionary, RLE, Snappy/GZIP compression
  - Statistics: Min/max per column → Skip unnecessary column groups
```

### Parquet for Analytics

```
Parquet file: events.parquet (1GB, contains 100M rows)

Query 1: SELECT SUM(amount)
  Read metadata → See amount column
  Read only amount column → 100M values × 8 bytes = 800MB
  Uncompressed → ~80MB after compression
  Latency: 80MB read = 0.16 seconds

Query 2: SELECT SUM(amount) WHERE timestamp >= "2026-07-20"
  Read metadata → timestamp has min/max per column group
  Skip column groups where max(timestamp) < "2026-07-20"
  Read only relevant column groups (~50% of file)
  Latency: 40MB read = 0.08 seconds

Query 3: SELECT SUM(amount) WHERE userId = 100
  Read metadata → Check statistics
  No min/max for userId (not in metadata)
  Must read all rows, scan for userId = 100
  Latency: 80MB read = 0.16 seconds
```

---

## Column-Oriented vs Row-Oriented Trade-offs

| Aspect | Row-Oriented | Column-Oriented |
|---|---|---|
| **Read (selective columns)** | Slow (read all) | Fast (read needed only) |
| **Write** | Fast (single write) | Slower (update multiple columns) |
| **Compression** | 2-5x | 10-100x |
| **Query latency** | 10-100 seconds | <1 second |
| **Transactional** | ✓ ACID | ✗ Append-only |
| **Updates** | ✓ Fast | ✗ Slow |
| **Deletes** | ✓ Fast | ✗ Requires tombstones |
| **Use case** | OLTP (transactional) | OLAP (analytics) |

---

## Production Considerations

1. **Parquet partitioning**: Partition by date/region to enable pruning
2. **Column statistics**: Ensure database computes stats for pruning
3. **Compression tuning**: Balance CPU cost vs I/O savings
4. **Cache**: Column caches (Memcached) for hot columns
5. **Avoid small queries**: Overhead only worth it for large scans (1M+ rows)

---

## Related Fundamentals

- [B-trees and LSM Trees](b-trees-and-lsm.md) – Index structures for row stores
- [Databases](../databases/) – Comparison with row-oriented systems
- [Search and Indexing](../search-and-indexing/) – Optimize column queries

---

**Status**: ✅ Complete. Covers column storage, compression, ClickHouse, Parquet, and trade-offs.
