# Embedded Databases

Lightweight key-value stores embedded in applications. RocksDB, LevelDB, SQLite, PebbleDB.

---

## TL;DR

- **Embedded**: No separate server, runs in application process
- **Use cases**: Local caches, state management, mobile apps, browser storage
- **RocksDB**: Write-optimized (LSM), used by CockroachDB, Kafka brokers
- **LevelDB**: Simpler LSM alternative, foundation for RocksDB
- **SQLite**: Full SQL support, ideal for single-writer scenarios
- **WAL**: Write-Ahead Logging ensures durability on crashes

---

## What Are Embedded Databases?

Unlike PostgreSQL or MySQL (client-server), embedded databases run inside the application process.

```
Traditional Database:
  App → Network → DB Server → Disk
  Latency: 1-5ms (network + server processing)

Embedded Database:
  App → In-process function → Disk
  Latency: <1ms (direct function call)
  
Trade-off:
  Speed: Faster (no network)
  Isolation: No isolation between processes (single app owns data)
  Scalability: Single machine only
```

---

## RocksDB

Write-optimized key-value store by Facebook, uses Log-Structured Merge (LSM) tree.

### Architecture

```
Write path (fast):
  1. Write to in-memory buffer (MemTable, usually 64MB)
  2. When full → Flush to disk as Level 0 SSTable
  3. Compaction: Merge overlapping SSTables (Levels 0→1→2...)
  
Read path:
  1. Check MemTable (hot data)
  2. Check Level 0 SSTables (newest, possibly multiple)
  3. Check Level 1+ SSTables (older, fewer)
  4. If key not found → Return nil

Write speed: ~100k-1M writes/sec
Read speed: ~10k-100k reads/sec (depends on level)
Latency: Write 10μs, Read 100-1000μs
```

### Compaction Strategy

RocksDB compacts levels to maintain performance:

```
Level 0: 4 SSTables (recently flushed)
  Compaction triggered → Merge with Level 1
  
Level 1: 10 SSTables (10x size of Level 0)
Level 2: 100 SSTables (10x size of Level 1)
...

Trade-off:
  ✓ Writes fast (just flush to Level 0)
  ✗ Read latency varies (might need to check multiple levels)
  ✗ Compaction is CPU-intensive (happens in background)
```

### Use Cases

- **Kafka brokers**: Store partition segments (millions of msgs)
- **CockroachDB**: Store local key-value data
- **Blockchain**: Store account balances, smart contract state
- **Time-series databases**: InfluxDB uses similar design

---

## LevelDB

Simpler LSM implementation by Google, inspired RocksDB.

```
Differences from RocksDB:

LevelDB:
  ✓ Simpler codebase (easier to understand/modify)
  ✓ Proven stability
  ✗ Fewer tuning options
  ✗ No multi-threading support (originally)
  
RocksDB:
  ✓ Multi-threaded compaction
  ✓ Better for high-throughput
  ✗ More complexity
```

### Use Cases

- **Bitcoin**: Store blockchain data
- **Chrome**: IndexedDB backed by LevelDB
- **Ethereum**: Early version used LevelDB

---

## SQLite

Full-featured SQL database in a single file, no server.

### When to Use SQLite

```
✅ Use SQLite:
  - Single-writer, multiple readers (desktop apps, mobile)
  - <100GB data (single file limit)
  - <1000 concurrent connections (in-process)
  - Offline-first apps (data syncs when online)
  - Browser-based (via WebAssembly)

❌ Don't use SQLite:
  - Multiple writers (locks entire DB during write)
  - Distributed systems (no replication)
  - High concurrency (OLTP with 1000s of connections)
  - Needs sharding (single file can't split)
```

### Architecture

```
SQLite stores data in a single file:
  [SQLite Database File (sqlite.db)]
    ├── Page 1: Table schema (CREATE TABLE statements)
    ├── Page 2-100: Table data (rowid, columns)
    ├── Page 101-150: Index data (B-tree for indexes)
    └── Page 151-200: Metadata (database version, etc.)

Transactions:
  BEGIN TRANSACTION
    Write to rollback journal (old data)
    Modify pages in memory
  COMMIT
    Mark journal complete
    On crash: Journal replayed to recover
```

### Concurrency Limitations

```
Multiple readers:
  Reader 1: SELECT (acquires read lock)
  Reader 2: SELECT (acquires read lock, can run concurrently)
  ✓ Works fine
  
Writer + Reader:
  Writer: INSERT (acquires write lock)
  Reader: SELECT (waits for write lock)
  Latency: Reader blocked until writer completes
  
Multiple writers:
  Writer 1: INSERT
  Writer 2: INSERT (blocked, must wait for Writer 1)
  Latency: Sequential writes only (~100-1000 writes/sec)
```

---

## Write-Ahead Logging (WAL)

Technique to ensure durability: log change before applying it.

### Standard Approach

```
Standard write (no WAL):
  1. Modify page in memory
  2. Fsync to disk
  3. Return success
  
Risk: Crash between step 1-2 → Data loss

WAL approach:
  1. Append operation to log (fsync)
  2. Apply operation to page
  3. Return success
  
On crash:
  Restart: Replay log to recover uncommitted operations
  ✓ Atomicity guaranteed
```

### Performance Impact

```
RocksDB with WAL:
  ✓ Durability: Crash-safe
  ✗ Write latency: 2x slower (log write + data write)
  ✗ Disk I/O: 2x (two fsync calls)

Tuning options:
  - Batch writes: Accumulate 100 operations, log once
  - Async WAL: Don't fsync log (faster, less durable)
  - Compression: Compress log entries (less I/O)
```

---

## Trade-offs: Embedded vs Server Databases

| Aspect | Embedded (RocksDB/SQLite) | Server (PostgreSQL) |
|---|---|---|
| **Latency** | <1ms | 1-5ms |
| **Throughput** | 1M ops/sec | 10k-100k ops/sec |
| **Concurrency** | Single writer (SQLite) or limited | Thousands of connections |
| **Data consistency** | Single machine | ACID across network |
| **Backup** | Copy file | Continuous replication |
| **Scaling** | Single machine only | Scale horizontally (replicas) |
| **Operational** | None (in-process) | Major (server, monitoring) |

---

## Choosing the Right Embedded Database

### SQLite
- Desktop/mobile apps
- Offline-first apps
- Query flexibility (full SQL)
- Constraint: Single writer

### RocksDB
- High-throughput ingestion
- NoSQL workloads (key-value)
- Embedded state stores (Kafka, KStream)
- No SQL queries needed

### LevelDB
- Simpler alternative to RocksDB
- When you need basics, not advanced features
- Proven stability over new features

---

## Production Considerations

1. **Backup strategy**: Copy DB file periodically to durable storage
2. **Compaction tuning**: Monitor CPU usage, adjust level sizes
3. **Cache sizing**: Tune MemTable size based on workload
4. **Monitoring**: Track read latency (indicates compaction stalls)
5. **WAL configuration**: Balance between durability and performance

---

## Related Fundamentals

- [B-trees and LSM Trees](b-trees-and-lsm.md) – Storage structure internals
- [Transactions & Isolation Levels](../databases/transactions-and-isolation-levels.md) – ACID guarantees
- [Indexing](../databases/indexing.md) – Query optimization with indexes

---

**Status**: ✅ Complete. Covers RocksDB, LevelDB, SQLite, WAL, and use cases.
