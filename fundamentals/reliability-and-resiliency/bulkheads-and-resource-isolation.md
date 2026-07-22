# Bulkheads & Resource Isolation

Preventing one failure from taking down entire system by isolating resource usage. Inspired by ship bulkheads that compartmentalize to prevent total sinking.

---

## TL;DR

- **Bulkhead pattern**: Isolate resources so one overloaded component doesn't starve others
- **Connection pool isolation**: Different pools for critical vs non-critical queries
- **Thread pool isolation**: Fast requests don't wait behind slow requests
- **Memory limits**: Container limits prevent one service consuming all RAM
- **Prevents cascade**: If search is slow, users can still check email

---

## Problem: Shared Resources

### Without Isolation

```
Database connection pool (20 connections):
  - Critical queries: login, payment
  - Non-critical: search, recommendations
  
Scenario: Search query is slow (takes 5 minutes)
  Query 1 (search): Acquires connection, holds for 5 min
  Query 2 (search): Acquires connection, holds for 5 min
  ...
  Query 20 (search): Acquires connection, holds for 5 min
  
  Connection pool exhausted (all 20 held by search)
  
Query 21 (payment): Tries to acquire connection
  No connections available!
  Payment request FAILS (times out after 30s)
  
Result: Slow search destroyed critical payment feature
```

---

## Bulkhead Pattern: Resource Isolation

Separate pools for different request types:

```
Database connection pools:
  - Critical pool (10 connections): login, payment, core features
  - Non-critical pool (10 connections): search, recommendations, analytics
  
Scenario: Search query is slow (5 minutes)
  Non-critical pool exhausted (10 connections)
  BUT critical pool still has 10 connections available
  
Payment query: Acquires from critical pool (success!)
  Completes in 50ms
  
Result: Search slowness doesn't affect critical features
```

---

## Thread Pool Bulkheading

### Problem: Single Thread Pool

```
Web server thread pool (100 threads):
  - Handle all requests (fast and slow)
  
Scenario:
  Request 1 (payment): Arrives, takes 1 thread
  Request 2-100 (payment): Take 99 threads
  Request 101 (health check): No threads available!
  Health check times out
  Load balancer thinks server is down
  Server removed from rotation
  Cascade failure
```

---

### Solution: Multiple Thread Pools

```
Web server:
  - Critical pool (20 threads): Health, status, core
  - Standard pool (50 threads): Normal requests
  - Background pool (30 threads): Async work
  
Scenario: 100 payment requests arrive
  20 run immediately (critical pool has priority)
  50 run on standard pool
  30 queue up (wait)
  
Health check request:
  Doesn't even try standard pool
  Uses critical pool
  1 thread available (19 other critical, 1 reserved)
  Health check succeeds
  Server stays in rotation
```

---

## Process-Level Bulkheads (Microservices)

### Old Monolith Approach

```
Single service process:
  Handle users, payments, search, recommendations, analytics
  
If search uses 90% CPU:
  All features slow down (users, payments affected)
  Cascade failure
```

---

### Microservices Bulkhead

```
Separate services:
  User Service (process 1): 2 CPU cores
  Payment Service (process 2): 4 CPU cores
  Search Service (process 3): 8 CPU cores
  Recommendation Service (process 4): 4 CPU cores
  Analytics Service (process 5): 2 CPU cores
  
If Search uses 90% CPU:
  Only Search affected (8 cores maxed)
  Other services unaffected (own CPU resources)
  
Result: Graceful degradation (search slow, everything else works)
```

---

## Kubernetes Resource Limits (Container Bulkheads)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: search-service
spec:
  containers:
  - name: search
    resources:
      requests:  # Guaranteed minimum
        cpu: "2"
        memory: "2Gi"
      limits:    # Hard cap
        cpu: "4"
        memory: "4Gi"
```

**Effect**:
```
Search service allocated:
  CPU limit: 4 cores (can't exceed)
  Memory limit: 4GB (can't exceed)

If search tries to allocate > 4GB:
  Kubernetes kills the pod (OOM killer)
  Restarts in healthy state
  Service recovered, no cascade
```

---

## Queue Bulkheading

Separate request queues prevent slow consumers from blocking fast ones:

```
Single queue:
  Slow batch job: Acquires, holds for 10 minutes
  Fast API requests: Wait behind slow batch job
  
Separate queues:
  Batch queue: Slow job runs (isolated)
  API queue: Fast requests served immediately (isolated)
  
Result: API responsiveness unaffected by batch job
```

---

## Memory Bulkheads

Prevent one component from consuming all system memory:

```
Service with multiple caches:
  User cache (could grow unbounded)
  Post cache (could grow unbounded)
  Recommendation cache (could grow unbounded)
  
Without bulkheads:
  User cache grows to 50GB
  OS memory exhausted
  Other caches evicted
  Service crashes
  
With bulkheads:
  User cache capped at 10GB (LRU eviction)
  Post cache capped at 15GB (LRU eviction)
  Recommendation cache capped at 5GB (LRU eviction)
  
  User cache full? Evict oldest
  Other caches unaffected
  Service keeps running
```

---

## Fan-Out Bulkheading (Parallel Requests)

### Problem: Slow Dependency

```
Service A needs data from 5 dependencies:
  B, C, D, E, F (each has timeout 30s)
  
A calls all 5 in parallel:
  B: 10ms (success)
  C: 20ms (success)
  D: 30s (timeout!)
  E: 15ms (success)
  F: 12ms (success)
  
A's response time: 30s (due to D's timeout)
```

---

### Solution: Bulkheaded Timeouts

```
A calls dependencies with individual timeouts:
  B: 30s timeout (acceptable if slow)
  C: 30s timeout
  D: 3s timeout (bulkhead: D is known to be slow/optional)
  E: 30s timeout
  F: 30s timeout
  
D times out after 3s:
  A returns partial response (B, C, E, F data)
  D missing (expected)
  
Result: A's response time: max(10, 20, 3, 15, 12) = 20ms
Instead of 30s + 3s = 33s
```

---

## Bulkhead Configuration

### Conservative (Safe)

```
Connection pools:
  Critical: 10 connections
  Standard: 10 connections
  Low-priority: 5 connections
  
Timeouts:
  Critical: 60s
  Standard: 30s
  Low-priority: 5s

Result: Slow requests don't affect fast path
Downside: Might use too much memory (many pools)
```

---

### Aggressive (Tight Resource Limits)

```
Connection pools:
  Shared: 20 connections
  No isolation (fallback to circuit breaker to prevent cascade)
  
Timeouts:
  All: 10s (uniform)

Result: Lower resource overhead
Downside: One slow query can still affect others
```

---

## Monitoring Bulkheads

Track utilization per bulkhead:

```
Critical pool utilization: 60%
  Alert if > 80% (critical features might be starved)
  
Standard pool utilization: 40%
  Alert if > 90% (cascade risk to non-critical)
  
Low-priority queue size: 1000
  Alert if > 5000 (background work piling up)
  
Result: Can proactively add capacity before cascade
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Multiple pools** | Single shared pool | Isolates failures, prevents cascade |
| **Microservices** | Monolith | Process-level isolation, independent scaling |
| **Bulkheaded timeouts** | Shared timeout | Fast requests don't wait for slow dependencies |
| **Memory limits** | Unbounded | Prevents OOM, ensures graceful degradation |

---

## Related Fundamentals

- [Circuit Breakers](circuit-breakers.md) – Stop trying when service down
- [Connection Pooling](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Pool management
- [Retries & Backoff](retries-and-exponential-backoff.md) – Coordinated with bulkheads

---

**Status**: ✅ Complete. Covers connection, thread, process, memory, and queue isolation patterns.

