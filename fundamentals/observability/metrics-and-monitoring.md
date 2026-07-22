# Metrics & Monitoring

Collecting metrics (CPU, latency, throughput), dashboards, and alerting on anomalies.

---

## TL;DR

- **Metrics**: Numerical measurements over time (counters, gauges, histograms)
- **Prometheus**: Time-series database, scrapes metrics from apps
- **Grafana**: Dashboards + visualization
- **RED method**: Rate, Errors, Duration (what to measure)
- **Alerting**: Alert on thresholds or anomalies

---

## Metric Types

### Counter

```
Continuously increasing number

Example: Total requests received
  T=0: 1000 requests
  T=1: 1005 requests (increased by 5)
  T=2: 1010 requests (increased by 5)
  
Use: Total requests, errors, bytes processed
Query: Derivative (requests per second)
```

---

### Gauge

```
Value that can go up or down

Example: Current CPU usage
  T=0: 45% CPU
  T=1: 60% CPU (went up)
  T=2: 30% CPU (went down)
  
Use: Current memory, open connections, CPU usage
Query: Alert if > 80%
```

---

### Histogram

```
Distribution of values

Example: Request latencies
  100ms: 50 requests
  500ms: 30 requests
  1s: 15 requests
  5s: 4 requests
  
Percentiles:
  p50: 100ms (50% of requests faster)
  p95: 500ms
  p99: 1s
  p99.9: 5s
```

---

## Prometheus

### Pull Model

```
App exposes metrics at /metrics:
  requests_total{endpoint="/api/users"} 1000
  request_duration_seconds{endpoint="/api/users"} 0.05
  
Prometheus scrapes every 15 seconds:
  GET /metrics
  Parse, store in time-series DB
  
Query: request_duration_seconds{endpoint="/api/users"}
Result: [0.05, 0.055, 0.048, ...] (time-series)
```

---

## Dashboards

### Key Metrics (RED)

```
Rate (throughput):
  Requests per second graph
  Alert if drops 50% (possible outage)
  
Errors:
  Error rate (%) over time
  Alert if > 1%
  
Duration (latency):
  P95, P99 latency graph
  Alert if p99 > 1s
```

---

## Alerting

### Threshold Alert

```
CPU > 80% for 5 minutes → Page on-call engineer

Configuration:
  metric: node_cpu_percent
  condition: > 80
  duration: 5 min
  action: send_pagerduty
```

---

### Anomaly Detection

```
Error rate spike:
  Baseline: 0.1%
  Detected: 2% (20x spike!)
  Alert: "Error rate spiked 20x"
  
Configuration:
  metric: error_rate
  condition: > baseline * 3  (anomaly)
  action: send_slack
```

---

## Related Fundamentals

- [Logging](logging-and-aggregation.md) – Log context
- [Distributed Tracing](distributed-tracing.md) – Trace latency details
- [Reliability](../reliability-and-resiliency/monitoring-and-alerting.md) – RED metrics

---

**Status**: ✅ Complete. Covers metric types, Prometheus, dashboards, alerting.

