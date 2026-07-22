# Distributed Tracing

Following requests across microservices to identify bottlenecks and understand latency.

---

## TL;DR

- **Trace**: Request journey through system (ID it, track it)
- **Span**: Single operation (1-50ms duration)
- **Trace ID**: Propagate through all services
- **Jaeger/Zipkin**: Trace storage and visualization
- **Sampling**: Can't trace everything (too much data), sample 1-10%

---

## Trace vs Log vs Metric

```
Log:
  "2026-07-23 10:30:45 request_received user_id=123"
  Point-in-time event
  
Metric:
  requests_per_second: 1000
  Current throughput
  
Trace:
  Request 123 timeline:
    T=0ms: UserService received request
    T=5ms: Called AuthService (5ms RPC)
    T=10ms: Called PaymentService (5ms RPC)
    T=50ms: Called NotificationService (40ms RPC)
    T=55ms: Response sent
  Complete request flow
```

---

## Trace Structure

### Root Span

```
Trace ID: abc123
  Span 1 (UserService): 0-50ms
    Span 1.1 (DB query): 10-20ms
    Span 1.2 (Cache lookup): 20-30ms
    Span 1.3 (RPC to Payment): 30-45ms
      Span 1.3.1 (PaymentService): 32-40ms
      Span 1.3.2 (DB query): 35-38ms
  
  Span 2 (NotificationService): 45-50ms
```

---

## Implementation (Jaeger)

### Trace Context Propagation

```
Request 1: UserService
  Generate: trace_id=abc123, span_id=span1
  
Call downstream: AuthService
  HTTP header: traceparent: 00-abc123-span1-01
  
AuthService:
  Extract: trace_id=abc123, parent_span=span1
  Create: span_id=span2 (child of span1)
  Call Notification
  Header: traceparent: 00-abc123-span2-01
  
Result: Full chain traced end-to-end
```

---

### Visualizing Latency

```
Timeline view:
  UserService [==============] 50ms
    AuthService [==] 10ms
    PaymentService [=========] 25ms
      DB Query [==] 5ms
    NotificationService [===] 15ms
    
Shows: Payment is slowest (25ms), should optimize DB
```

---

## Sampling

### Problem: Too Much Data

```
1M requests/sec
1KB per trace
= 1GB/sec (1 petabyte/day!)
Impossible to store
```

### Solution: Sampling

```
Sample 1% of requests:
  10k traces/sec
  10MB/sec (manageable)
  
Sampling strategy:
  Random: 1% of all requests
  Adaptive: Sample 100% of errors, 10% of normal
  Rate-based: Sample up to 10k traces/sec
```

---

## Related Fundamentals

- [Logging](logging-and-aggregation.md) – Logs within traces
- [Metrics](metrics-and-monitoring.md) – Performance trends
- [Reliability](../reliability-and-resiliency/monitoring-and-alerting.md) – SLO monitoring

---

**Status**: ✅ Complete. Covers trace structure, propagation, sampling.

