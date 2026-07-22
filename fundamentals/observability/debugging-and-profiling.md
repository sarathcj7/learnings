# Debugging & Profiling

Production debugging techniques: CPU profiling, memory profiling, flame graphs, and finding bottlenecks.

---

## TL;DR

- **CPU profiling**: Find functions consuming CPU (flame graphs)
- **Memory profiling**: Detect memory leaks, allocation hotspots
- **Flame graphs**: Visualize call stacks (where is time spent?)
- **pprof**: Go profiling tool
- **Sampling**: Low overhead profiling in production

---

## CPU Profiling

### Problem: Slow Endpoint

```
GET /expensive endpoint takes 500ms
Users complaining
Need to find bottleneck
```

### Solution: CPU Profile

```
Collect profile (sample function calls every 10ms):
  Function A: 150ms
  Function B: 100ms
  Function C: 200ms
  Function D: 50ms
  
Hottest function: C (200ms = 40%)
  Likely: Inefficient algorithm or N+1 query
  Optimization: Fix C → Total 300ms (40% improvement)
```

---

## Memory Profiling

### Heap Allocations

```
Sample: 1GB allocated in test
Breakdown:
  User cache: 400MB
  Post cache: 300MB
  Temporary buffers: 250MB
  Other: 50MB
  
Finding: User cache growing unbounded
  Likely: LRU eviction broken
  Fix: Re-enable eviction → Memory stable
```

---

## Flame Graphs

### Visualization

```
Top: main()
├─ ServerLoop (400ms)
│  ├─ HandleRequest (380ms)
│  │  ├─ QueryDatabase (250ms)
│  │  │  ├─ ParseSQL (50ms)
│  │  │  └─ FetchRows (200ms)
│  │  └─ ProcessData (130ms)
│  │     └─ FilterResults (120ms)
│  └─ SendResponse (20ms)
└─ Other (100ms)

Width = time spent
Height = stack depth

Finding: FetchRows is bottleneck (200ms)
```

---

## pprof (Go Example)

### Usage

```go
import _ "net/http/pprof"

// Endpoint available at:
// GET /debug/pprof/profile?seconds=30
// GET /debug/pprof/heap
// GET /debug/pprof/mutex

// Collect profile:
// go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// top -cum  (show cumulative time)
// web       (graph visualization)
```

---

## Production Profiling

### Always-On (Low Overhead)

```
Continuous sampling:
  CPU: Sample every 10ms (small overhead)
  Memory: Sample every allocation (2% overhead)
  
Benefits:
  Detect performance regressions
  Find slow endpoints before users complain
  
Tools:
  Datadog profiling
  Parca (continuous profiling)
```

---

## Related Fundamentals

- [Metrics](metrics-and-monitoring.md) – Monitor before profiling
- [Logging](logging-and-aggregation.md) – Trace errors to bottleneck
- [Reliability](../reliability-and-resiliency/) – Cascade detection

---

**Status**: ✅ Complete. Covers CPU/memory profiling, flame graphs, tools.

