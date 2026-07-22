# Logging & Log Aggregation

Structured logging, log levels, centralized log management, and debugging production systems.

---

## TL;DR

- **Structured logging**: JSON logs (searchable) vs plaintext (human-readable)
- **Log levels**: DEBUG, INFO, WARN, ERROR, CRITICAL (choose what to log)
- **Log aggregation**: Centralize logs (ELK, Datadog, Splunk)
- **Retention**: Keep 7-30 days (storage cost vs audit trail)
- **Security**: Never log passwords/PII, redact sensitive data

---

## Structured Logging

### Plaintext (Bad for Search)

```
2026-07-23 10:30:45 User login attempt
2026-07-23 10:30:46 Invalid password for user admin@example.com
2026-07-23 10:30:47 User john@example.com logged in from 192.168.1.1
```

**Problem**: Hard to search/parse programmatically

---

### Structured (JSON)

```json
{
  "timestamp": "2026-07-23T10:30:45Z",
  "level": "INFO",
  "message": "user_login",
  "user_id": "user_123",
  "email": "john@example.com",
  "ip": "192.168.1.1",
  "status": "success"
}
```

**Benefit**: Can query: `level=ERROR AND user_id=user_123`

---

## Log Levels

```
DEBUG: Development only
  logger.debug("Processing request", {"path": "/api/users", "params": {...}})
  
INFO: Important events
  logger.info("user_login", {"user_id": 123})
  
WARN: Unexpected but recoverable
  logger.warn("slow_query", {"duration_ms": 5000, "query": "SELECT..."})
  
ERROR: Error occurred
  logger.error("request_failed", {"error": "connection timeout", "retry_count": 3})
  
CRITICAL: System-level failure
  logger.critical("database_unavailable", {"service": "payment"})
```

---

## Log Aggregation

### Problem: Multiple Servers

```
100 servers, 10GB logs each = 1TB storage
Finding error: Grep across 100 servers (slow!)
```

### Solution: Centralized Logging

```
Server 1 → Logs → Central storage (ELK, Datadog)
Server 2 → Logs → 
Server 3 → Logs → 

Query: Find all errors in last hour
Result: 5 minutes (vs hours across 100 servers)

Visualization: Dashboard of errors, latency, traffic
Alerting: Alert on error rate spike
```

---

## Implementation

### ELK Stack (Elasticsearch, Logstash, Kibana)

```
Logstash: Collect logs from servers
  - Parse JSON
  - Add fields (environment, version)
  - Filter sensitive data
  
Elasticsearch: Index & search
  - Store logs as JSON documents
  - Searchable (full-text, field queries)
  - Retention: Delete logs > 30 days (space)
  
Kibana: Dashboard & visualization
  - Graph error rate over time
  - Heat map of latency
  - List top errors
  
Cost: Self-hosted (cheap, ops overhead)
```

---

### Managed Services

```
Datadog:
  - Ingest logs from anywhere
  - Automatic parsing
  - Built-in dashboards
  - Cost: ~$1/GB ingested (expensive for high volume)
  
Splunk:
  - Similar to Datadog
  - Enterprise-grade
  - Cost: ~$5/GB (more expensive)
  
CloudWatch (AWS):
  - Native to AWS
  - Integration with Lambda, EC2
  - Cost: ~$0.50/GB (cheaper)
```

---

## Security in Logging

### Never Log Sensitive Data

```
❌ Bad:
  logger.info("payment", {"card": "4111-1111-1111-1111", "amount": "$100"})

✅ Good:
  logger.info("payment", {"card_last4": "1111", "amount": "$100"})
```

---

### Redaction

```
PII that slips in automatically redacted:
  email@example.com → redacted
  192.168.1.100 → redacted (potential internal IP)
  password=xyz → password=redacted
```

---

## Performance Considerations

### Async Logging

```
Synchronous:
  logger.info(...) → Block until written to disk
  Latency: +5-10ms per log
  
Asynchronous:
  logger.info(...) → Queue to background thread
  Latency: <1ms
  Risk: Logs lost if crash before queue drained
  
Production: Always async (queue provides safety)
```

---

## Related Fundamentals

- [Metrics & Monitoring](metrics-and-monitoring.md) – Logs + metrics + traces
- [Distributed Tracing](distributed-tracing.md) – Trace ID correlates logs
- [Security](../security/) – PII redaction

---

**Status**: ✅ Complete. Covers structured logging, levels, aggregation, security.

