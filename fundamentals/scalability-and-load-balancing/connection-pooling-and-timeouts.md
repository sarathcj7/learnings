# Connection Pooling & Timeouts

Managing connections efficiently is critical for scaling. Learn pooling strategies, timeout tuning, and connection leak prevention.

---

## TL;DR

- **Connection pool**: Reuse DB connections, don't create new for each request
- **Pool size**: Typically = (core count × 2) + pending I/O connections
- **Timeouts**: Connection (establish time) vs Read (data arrival) vs Idle (unused connection)
- **Connection leak**: Forgetting to close connection = grows until OOM
- **Cascade failures**: One slow query can exhaust connection pool → all subsequent requests fail

---

## Problem: Why Connection Pooling Matters

### Without Pooling

```
Request 1:
  1. Establish TCP connection to DB (10ms)
  2. Handshake + auth (20ms)
  3. Execute query (50ms)
  4. Close connection (5ms)
  Total: 85ms per request

1000 req/sec: 85 seconds just in connection overhead!
```

### With Pooling

```
Pre-created pool: 20 connections maintained
Request 1:
  1. Get connection from pool (1μs)
  2. Execute query (50ms)
  3. Return connection to pool (1μs)
  Total: 50ms per request

1000 req/sec: No connection overhead!
```

---

## Connection Pool Mechanics

### Creating the Pool

```
Initialize pool:
  Create 20 connections to DB
  Each connection is idle, ready to use
  
Pool state: {
  available: 20,  // Connections ready to use
  active: 0,      // Connections currently executing
  total: 20       // Total size
}
```

### Acquiring Connection

```
Request arrives:
  if pool.available > 0:
    connection = pool.acquire()  // O(1)
    pool.available -= 1
    pool.active += 1
  else:
    wait in queue until connection available (timeout ~5s)
```

### Returning Connection

```
Query complete:
  pool.release(connection)
  pool.available += 1
  pool.active -= 1
  
  If queue not empty:
    Waiting request wakes up, acquires connection
```

---

## Pool Sizing

### Formula

```
pool_size = (core_count × 2) + pending_i_o_count

Example:
  8-core server × 2 = 16 base connections
  + 4 pending I/O = 20 total
  
Rationale:
  (core_count × 2): Thread pool size + headroom
  pending_i_o: Requests that might block (network I/O, locks)
```

### Too Small

```
pool_size = 5, but 20 requests arrive simultaneously

Pool state:
  5 connections acquired, 15 waiting
  Average wait: 100ms (if query = 50ms)
  
System appears slow (excessive queueing)
```

### Too Large

```
pool_size = 1000, but server has 8 cores

Result:
  998 connections idle, using RAM
  8 cores can't serve more than 8-10 requests concurrently anyway
  Context switching overhead
  
Memory wasted, CPU thrashing
```

---

## Timeouts: Three Levels

### 1. Connection Timeout (Establish)

Time to establish TCP connection and authenticate:

```
socket.connect(host, port)
  TCP handshake: ~10ms
  SSL handshake: ~50ms
  Auth: ~20ms
  
Typical: 5-10 seconds
Too short (1s): Legitimate slow connections fail
Too long (60s): Slow network issues hang for a minute
```

---

### 2. Read Timeout (Query Execution)

Time waiting for query results:

```
execute_query("SELECT * FROM huge_table")
  If query takes > timeout: Kill query, return error
  
Typical: 30 seconds
Too short (1s): Complex queries killed prematurely
Too long (300s): Slow queries block connections for 5 minutes
```

---

### 3. Idle Timeout (Lifetime)

How long connection can sit unused before being closed:

```
Connection acquired @ 10:00:00
  No queries sent
Connection idle @ 10:05:00 (5 minutes later)

If idle_timeout = 5 min:
  Connection closed, removed from pool
  
If idle_timeout = 1 hour:
  Connection kept in pool (wastes resources)
```

**Typical**: 15-30 minutes

---

## Connection Leak Detection

### What is a Leak?

Connection acquired but never returned to pool:

```python
connection = pool.acquire()
execute_query(connection, "SELECT ...")
// Oops! Forgot to pool.release(connection)

Result:
  pool.available decreases
  After 20 requests: pool.available = 0
  All subsequent requests wait forever
  Service hangs
```

---

### Prevention

**Pattern 1: Try-Finally** (old style):

```java
Connection conn = pool.acquire();
try {
  executeQuery(conn);
} finally {
  pool.release(conn);  // Always called
}
```

**Pattern 2: Try-With-Resources** (modern):

```java
try (Connection conn = pool.acquire()) {
  executeQuery(conn);
}  // Connection auto-released on exit
```

**Pattern 3: Language Feature (Go, Rust)**:

```go
defer conn.Close()  // Go defer
conn := pool.Acquire()
executeQuery(conn)
```

---

### Detection & Monitoring

```
Alert if:
  pool.active > threshold (e.g., 90% utilized constantly)
  pool.available < threshold (e.g., frequently 0)
  queue_wait_time increasing
  
Debug: Print stack traces of connections held > 10 seconds
  Identify which queries leak connections
```

---

## Cascade Failure via Connection Pool

### Scenario

```
Service A: Database connection pool (20 connections)
  
Request 1: Slow query takes 5 minutes
  Acquires 1 connection (now 19 available)
  
Requests 2-20: All acquire connections
  19 available (now 0)
  
Requests 21+: Queue up, wait for connection
  Average wait: 5 min = timeout triggers
  Requests fail
  
All clients see "Service unavailable" even though issue is ONE slow query!
```

### Prevention

**Strategy 1: Bulkhead Pattern** (isolation):

```
Database connection pools:
  - Fast queries pool (20 connections)
  - Slow queries pool (5 connections)
  
Slow query doesn't drain fast query pool
Latency-sensitive requests unaffected
```

**Strategy 2: Adaptive Timeout**:

```
If queue_wait_time > 500ms:
  Reject new requests immediately (fail fast)
  Don't let them wait, exhaust pool
  
Result: Graceful degradation instead of cascade
```

**Strategy 3: Circuit Breaker**:

```
If pool.available < 1 for 10 seconds:
  Open circuit (reject all requests immediately)
  Wait 30 seconds, retry single request
  If successful: Close circuit, resume
  
Result: Prevent thrashing, allow recovery
```

---

## Connection Pooling in Microservices

### Sidecar Pattern

```
Service A → Sidecar LB (Envoy)
  Sidecar maintains connection pools to Service B
  Service A code makes requests to sidecar
  
Advantage:
  Service code doesn't manage pools
  Sidecar enforces pooling, timeouts centrally
  
Example: Istio, Linkerd
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Pooling** | Create per-request | Eliminates connection overhead, enables scaling |
| **Moderate pool size** | Too large or small | Balanced resource usage, no queuing backlog |
| **Short read timeout** | Long timeout | Fail fast, prevent cascade failures |
| **Bulkhead pattern** | Shared pool | Isolate fast/slow paths, prevent cascade |
| **Explicit close** | Auto-cleanup | Deterministic resource management |

---

## Production Checklist

- [ ] Connection pool size tuned to core count + I/O
- [ ] Connection timeout configured (5-10s)
- [ ] Read timeout configured (30-60s, query-dependent)
- [ ] Idle timeout configured (15-30 min)
- [ ] Connection leak monitoring (pool utilization alerts)
- [ ] Cascade failure prevention (bulkheads or circuit breakers)
- [ ] Graceful degradation on pool exhaustion

---

## Related Fundamentals

- [Reliability & Resiliency](../reliability-and-resiliency/) – Circuit breakers, bulkheads, cascade prevention
- [Load Balancing](load-balancing-algorithms.md) – Distributing connections across servers
- [Databases/Replication](../databases/replication.md) – Connection pooling across replicas

---

**Status**: ✅ Complete. Covers pooling mechanics, timeouts, leaks, and cascade failures.

