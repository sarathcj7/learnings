# Monitoring & Alerting for Reliability

What to measure, how to alert on problems, and catching failures before users notice.

---

## TL;DR

- **RED metrics**: Request rate, error rate, duration (latency)
- **SLI/SLO/SLA**: Define reliability targets and measure against them
- **Alerting**: Alert on symptoms (not causes), make actionable
- **Dashboards**: For investigation, not "all metrics at once"
- **Logs**: For debugging, traces for distributed context

---

## RED Method (Best Practices)

### Request Rate

```
Metric: Requests per second (RPS)

What to measure:
  Total RPS (overall traffic)
  Per-endpoint (POST /users: 500 RPS)
  Per-client (mobile: 40%, web: 60%)

Purpose: Detect traffic spikes, validate scaling

Alert: 
  If RPS > 2x normal → Scale capacity
  If RPS drops 50% → Possible issue upstream
```

---

### Error Rate

```
Metric: Errors per second (or % of requests)

What to measure:
  HTTP 500: Server errors (our problem!)
  HTTP 429: Rate limited (capacity issue)
  HTTP 404: Not found (might be expected)
  Timeout errors: Network issues
  
Calculation:
  error_rate = errors / total_requests
  Example: 10 errors / 1000 requests = 1% error rate

Alert:
  If error_rate > 1% → Investigate immediately
  If error_rate > 5% → Page on-call engineer
  If error_rate > 10% → Declare incident
```

---

### Duration (Latency)

```
Metric: Response time percentiles

What to measure:
  p50 (median): 50% of requests faster than this
  p95: 95% of requests faster than this
  p99: 99% of requests faster than this
  p99.9: 99.9% of requests faster than this
  
Example:
  p50 latency: 50ms (typical request)
  p99 latency: 500ms (slow request, but acceptable)
  p99.9 latency: 5s (rare, very slow request)

Alert:
  If p99 latency > 1s → Investigate
  If p99.9 latency > 10s → Cascade risk
```

---

## SLI, SLO, SLA

### SLI (Service Level Indicator)

What you actually measure:

```
Examples:
  "API responds within 100ms": Measured as p99 latency < 100ms
  "API error rate < 0.1%": Measured as error_count / total_requests
  "Database uptime 99.9%": Measured as minutes_up / total_minutes
  
How to measure:
  Application metrics (internal measurements)
  Synthetic tests (external, simulate user)
  Real user monitoring (actual traffic)
```

---

### SLO (Service Level Objective)

What you want to achieve:

```
Examples:
  "Maintain p99 latency < 100ms"
  "Maintain error rate < 0.1%"
  "Maintain uptime 99.9% (30s downtime/month)"
  "Maintain database backup < 1 hour old"

SLO is a TARGET, not a guarantee.
```

---

### SLA (Service Level Agreement)

Contract with users:

```
SLA: "99.9% uptime, or customer gets $100 credit"

Relationship:
  SLI < SLO < SLA
  
Example:
  SLI: Measured uptime 99.87%
  SLO: Target 99.9%
  SLA: Promise 99.9%
  
  SLI (99.87%) < SLO (99.9%) = Missed target
  Might owe credits
```

---

## Alerting Philosophy

### Good Alert vs Bad Alert

**Bad Alert** (Too noisy):
```
Alert: "CPU > 50%"
  Fires constantly (normal operation)
  On-call ignores it (alert fatigue)
  When real issue happens, missed in noise
  
Result: Unreliable alerting
```

---

**Good Alert** (Symptom-based):
```
Alert: "Error rate > 1% for 5 minutes"
  Fires only on actual problems
  On-call investigates (high signal)
  Directly actionable (check error logs)
  
Result: Reliable, trusted alerting
```

---

## Alert Types

### Type 1: Threshold Alert

```
Condition: Metric > value for duration

Examples:
  Error rate > 1% for 1 minute
  p99 latency > 1 second for 2 minutes
  Disk usage > 80% for 5 minutes
  
Implementation:
  if error_rate > 0.01 for last 60s:
    send_alert()
```

---

### Type 2: Anomaly Detection

```
Condition: Metric deviates from baseline

Examples:
  RPS dropped 50% (unusual, check cache hit ratio)
  Latency spiked 2x normal (might indicate GC pause)
  Memory growing 1MB/s (memory leak)
  
Implementation:
  baseline = average over last 24 hours
  current = average over last 5 min
  if abs(current - baseline) > 2 * stddev:
    send_alert()
```

---

### Type 3: Composite Alert

```
Condition: Multiple metrics together

Examples:
  Error rate > 1% AND p99 latency > 500ms
    (not just any error, but slow errors)
    
  CPU > 90% AND memory > 80%
    (resource exhaustion, not just one metric)
    
  Circuit breaker OPEN for > 5 min
    (sustained downstream failure)
```

---

## Dashboard Design

### Bad Dashboard (Overwhelming)

```
"System Status Dashboard"
  100 metrics displayed simultaneously
  Graphs of: CPU, Memory, Disk, Network, Latency, QPS, 
             Cache hits, GC, Connections, Threads, ...
             
User stares at dashboard:
  Which metric do I care about?
  What does this tell me?
  What should I do?
```

---

### Good Dashboard (Focused)

```
"API Health Dashboard"
  RED metrics prominently:
    Request rate: Graph
    Error rate: Graph (RED zone if > 1%)
    p99 latency: Graph (RED zone if > 1s)
  
  Secondary:
    Request breakdown (by endpoint)
    Error breakdown (by error type)
  
  Incident mode (when alert fires):
    Zoom to last hour
    Highlight anomaly
    Show related metrics

User sees at a glance: Is API healthy? Yes/No
```

---

## Structured Logging

```
Bad log:
  "Error processing request"
  
Good log:
  {
    timestamp: "2026-07-23T10:30:45.123Z",
    level: "ERROR",
    message: "Database query timeout",
    service: "user-service",
    endpoint: "GET /user/123",
    userId: 456,
    duration_ms: 30123,
    error: "TIMEOUT",
    database: "user-db-replica-2",
    query: "SELECT * FROM users WHERE id = ?",
    stacktrace: "..."
  }
```

**Advantage**: Can search logs efficiently:
```
Find all timeouts: error = "TIMEOUT"
Find user 456 issues: userId = 456
Find database replica issues: database = "*replica*"
```

---

## Distributed Tracing

For understanding latency across microservices:

```
User request → API Gateway (10ms)
  ├─ Call User Service (50ms)
  │   ├─ Query database (30ms)
  │   └─ Call cache (1ms)
  └─ Call Recommendation Service (200ms)
      ├─ Call ML model (180ms)
      └─ Call database (15ms)
      
Total: 260ms

Without trace: "Request took 260ms, but why?"
With trace: Can see Recommendation Service is bottleneck (200ms)
```

---

## Monitoring Reliability Patterns

### Circuit Breaker Health

```
Alert: circuit_breaker_state == OPEN for > 5 minutes
  Indicates downstream service is broken
  Need to investigate and fix upstream service
  Or increase circuit breaker timeout
```

---

### Connection Pool Health

```
Alert: pool.available < 1 for 10 seconds
  Connection pool exhausted
  Risk of cascade failure
  Investigate slow queries, increase pool size, or reduce timeout
```

---

### Queue Depth

```
Alert: job_queue_size > 10000 for 5 minutes
  Processing slower than intake
  Queue will grow until system crashes
  Scale up worker count or optimize job processing
```

---

## Production Checklist

- [ ] RED metrics dashboards set up
- [ ] Error rate alert (> 1% for 1 min)
- [ ] Latency alert (p99 > SLO for 2 min)
- [ ] Resource alerts (CPU > 80%, Memory > 85%, Disk > 90%)
- [ ] Structured logging configured (JSON format)
- [ ] Distributed tracing for microservices
- [ ] On-call runbook for each alert
- [ ] SLO defined and tracked

---

## Related Fundamentals

- [Circuit Breakers](circuit-breakers.md) – Monitor trip rate
- [Connection Pooling](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Monitor pool utilization
- [Observability/Logging](../observability/) – Deeper logging patterns

---

**Status**: ✅ Complete. Covers RED method, SLI/SLO, alerting, dashboards, logging.

