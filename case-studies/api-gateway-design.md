# API Gateway Design

*Design Kong, AWS API Gateway: routing, authentication, rate limiting, caching, monitoring for microservices.*

## Problem Statement

Build an API gateway. Entry point for all microservices. 100M+ requests/sec. Must handle auth, rate limiting, caching, logging, monitoring.

## Architecture

```
Request Flow:
  Client → API Gateway → [Auth] → [Rate limit] → [Route] → Service
  
  Gateway responsibilities:
    1. TLS termination
    2. Authentication (OAuth, API key, JWT)
    3. Rate limiting (per-user, per-IP)
    4. Request routing (path-based to services)
    5. Caching (cache response if applicable)
    6. Logging (structured logs)
    7. Monitoring (metrics, traces)
    8. Request/response transformation
```

## Key Components

### 1. Routing

```
GET /users/{id} → User Service
GET /orders/{id} → Order Service
POST /orders → Order Service

Use trie or path-based routing for O(log N) lookup
```

### 2. Rate Limiting

```
Per-user: 1000 requests/minute
Per-IP: 10k requests/minute

Distributed counter (Redis):
  INCR "rate-limit:user:123"
  EXPIRE "rate-limit:user:123" 60 (reset every minute)
```

### 3. Caching

```
GET /products/{id} (idempotent):
  Cache response for 1 hour
  Serve from cache if hit

Cache key: method:path:user_id
  Different users see different cached content (personalized)
```

## Bottlenecks & Scaling

**Bottleneck**: Gateway becomes single point of failure/throughput.

**Solution**:
- Horizontal scaling (multiple gateway instances)
- Load balancer in front (distribute traffic)
- Caching (reduce upstream load)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Centralized gateway** | Service-to-service mesh | Simpler operations, single control point |
| **Gateway caching** | No caching | Performance (cache hits reduce upstream QPS) |
| **Distributed rate limiting** | In-process | Accurate across gateways (not per-gateway quotas) |

---

**Status**: ✅ Complete. Shows gateway, routing, auth, monitoring.
