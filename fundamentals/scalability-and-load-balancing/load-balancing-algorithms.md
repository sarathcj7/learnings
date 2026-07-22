# Load Balancing Algorithms

How to distribute traffic across multiple servers, from simple round-robin to advanced algorithms that handle heterogeneous infrastructure.

---

## TL;DR

- **Round-robin**: Simplest, worst for heterogeneous servers
- **Least connections**: Better for persistent connections, stateful apps
- **Weighted load balancing**: Accounts for different server capacities
- **IP hash / Consistent hash**: Sticky sessions, cache affinity
- **Least response time**: Dynamically measures performance
- **Production reality**: Layered approach (DNS → LB → consistent hash)

---

## Core Algorithms

### 1. Round-Robin (RR)

Send requests in circular order: Server 1, Server 2, Server 3, Server 1, ...

```
Pseudocode:
  counter = 0
  On request:
    server = servers[counter % servers.length]
    counter += 1
    return server
```

**Pros**: 
- Trivial to implement
- Stateless (no coordination needed)

**Cons**: 
- Ignores server load
- Ignores network latency
- Bad if servers have different capacities

**Use case**: Stateless services with identical servers

---

### 2. Least Connections (LC)

Route to server with fewest active connections.

```
On request:
  server = servers with minimum active_connections
  server.active_connections += 1
  forward request
  
On response:
  server.active_connections -= 1
```

**Pros**:
- Adapts to server load
- Better distribution than RR
- Good for long-lived connections (WebSocket, gRPC)

**Cons**:
- Requires tracking connection state (on LB)
- Not ideal for quick requests (request count better than connection count)

**Use case**: Chat systems, WebSocket services, database connection pools

---

### 3. Weighted Round-Robin (WRR)

Servers have weights. Server with weight 3 gets 3x more traffic.

```
Pseudocode:
  counter = 0
  On request:
    server = servers[counter % total_weight]
    counter += 1
    return server
```

**Example**:
```
Server A: weight 3 (fast hardware)
Server B: weight 1 (slower hardware)

Sequence: A, A, A, B, A, A, A, B, ...
A gets 75%, B gets 25%
```

**Pros**:
- Accounts for heterogeneous hardware
- Static (no monitoring needed)

**Cons**:
- Weights are manual configuration
- Ignores dynamic load changes

**Use case**: Mixed infrastructure (spot + reserved instances)

---

### 4. IP Hash (Source IP Affinity)

Hash client IP → same server every time.

```
Pseudocode:
  server_index = hash(client_ip) % servers.length
  return servers[server_index]
```

**Pros**:
- Sticky sessions (client → same server always)
- Cache-friendly (if server has local cache)
- Stateless on LB (hash is deterministic)

**Cons**:
- Bad distribution if hash has hotspots
- Load imbalance (some IPs route to same server)
- When servers added/removed, all hashes change (cache misses)

**Use case**: Session affinity, but use consistent hashing instead

---

### 5. Consistent Hashing

Hash client/request → server on ring, minimize reshuffling when servers added/removed.

```
Ring: 0 to 2^32-1

Each server: Hashed to position on ring
Each request: Hashed to position on ring
  Next server clockwise = destination

Example:
  Server A at position 1000
  Server B at position 5000
  Server C at position 9000
  
  Request hash 3000 → Server B (next clockwise)
  Request hash 7000 → Server C
  Request hash 500 → Server A
```

**Pros**:
- When server added/removed, only ~1/N of requests rehashed
- Cache-affinity preserved
- Highly used in distributed systems (Memcached, Redis)

**Cons**:
- More complex than IP hash
- Still doesn't account for load (use load-aware variants)

**Use case**: Caching, distributed systems, sticky routing

---

### 6. Least Response Time (LRT)

Route to server with best response time (monitored dynamically).

```
On request:
  server = servers with minimum average_response_time
  start_time = now()
  forward request
  
On response:
  response_time = now() - start_time
  server.average_response_time = weighted_avg(response_time)
  return response
```

**Pros**:
- Adapts to actual performance
- No manual tuning (weights)
- Good for heterogeneous hardware

**Cons**:
- Monitoring overhead
- Can be volatile (spikes in response time cause shifts)
- Slower to converge on good distribution

**Use case**: APIs with variable latency, microservices mesh

---

## Advanced Considerations

### Sticky Sessions + Consistency

Problem: User's session on Server A, LB sends next request to Server B.

Solutions:
1. **IP Hash**: Route same IP to same server (simple, cache-friendly)
2. **Consistent Hash**: Same as above, better reshuffling
3. **Session Store**: Store session in Redis, all servers access (removes stickiness requirement)

**Production choice**: Session store (more flexible, enables horizontal scaling)

---

### Connection Draining (Graceful Shutdown)

```
Step 1: Mark server as "drain"
  LB stops sending NEW requests
  Existing connections continue

Step 2: Wait for requests to finish
  Timeout = 30s, if still have connections after 30s, force close

Step 3: Remove server from pool
  
Result: No dropped requests during graceful shutdown
```

---

### Health Checks

Load balancer periodically checks if server is healthy:

```
Every 5 seconds:
  GET /health on each server
  If 2 checks fail in row: Mark as UNHEALTHY
  Stop routing to unhealthy server
  Resume after 2 successful checks
```

**Failure modes**:
- False positives (server briefly slow, marked unhealthy)
- Thundering herd (all traffic shifts when server comes back online)

---

## Comparison Table

| Algorithm | Load-Aware | Sticky | Scalable | Complexity |
|---|---|---|---|---|
| **Round-robin** | ❌ | ❌ | ✅ | Low |
| **Least connections** | ✅ | ❌ | ✅ | Low |
| **Weighted RR** | ❌ (manual) | ❌ | ✅ | Low |
| **IP hash** | ❌ | ✅ | ❌ (reshuffling) | Low |
| **Consistent hash** | ❌ | ✅ | ✅ | Medium |
| **Least response time** | ✅ | ❌ | ✅ | Medium |

---

## Production Patterns

### Layered Load Balancing

```
1. DNS Load Balancing (GSLB)
   example.com → Route to closest regional LB
   Clients around world → regional LBs
   
2. L7 Load Balancer (regional)
   HTTP routing, path-based, host-based
   Route /api/users → user-service
   Route /api/posts → post-service
   
3. L4 Load Balancer (per service)
   TCP/UDP routing, simple
   Distribute within service

4. Consistent Hash in Service
   Client-side or mesh sidecar
   Route to specific replica
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Consistent hash** | IP hash | Handles server additions gracefully |
| **Least response time** | Round-robin | Adapts to heterogeneous hardware, dynamic load |
| **Session store** | IP hash | Enables distributed deployments, no stickiness bottleneck |
| **Health checks** | No monitoring | Catch failures quickly, minimal false positives with good tuning |

---

## Related Fundamentals

- [Scalability & Load Balancing/GSLB](gslb.md) – Global load balancing across regions
- [Consensus & Coordination](../consensus-and-coordination/) – Coordination of LB state
- [Networking](../networking-and-protocols/) – TCP/UDP, DNS, SSL termination

---

**Status**: ✅ Complete. Covers all major algorithms with production patterns.

