# Global Server Load Balancing (GSLB)

Routing traffic across geographically distributed data centers to minimize latency, improve availability, and balance global load.

---

## TL;DR

- **GSLB routes DNS queries** based on geographic location, server health, load
- **TTL trade-off**: Short TTL = fast failover but more DNS queries; long TTL = fewer queries but stale routing
- **Latency-based routing**: Route to geographically closest datacenter
- **Failover**: GSLB can redirect to backup region on disaster
- **Production**: Implemented via DNS providers (Route53, Cloudflare) or dedicated GSLB appliances

---

## Problem & Context

### Without GSLB

```
User in Tokyo: example.com → 1.2.3.4 (US East datacenter)
  Latency: ~150ms (crosses Pacific)
  
User in London: example.com → 1.2.3.4 (US East)
  Latency: ~100ms (crosses Atlantic)
  
User in São Paulo: example.com → 1.2.3.4 (US East)
  Latency: ~200ms (long way around)
```

### With GSLB

```
User in Tokyo: example.com → 5.6.7.8 (Asia Pacific datacenter)
  Latency: ~5ms (local)
  
User in London: example.com → 3.4.5.6 (Europe datacenter)
  Latency: ~2ms (local)
  
User in São Paulo: example.com → 7.8.9.10 (South America datacenter)
  Latency: ~8ms (local)
```

---

## How GSLB Works

### Step 1: User Query

```
User in Tokyo: resolve example.com
  Query → Local DNS resolver
  Resolver → GSLB system
```

### Step 2: GSLB Decision

GSLB evaluates:
- Client IP geolocation (approximately where user is)
- Server health (which datacenters are up)
- Server load (which datacenters have capacity)
- Latency measurements (fastest path to user)

### Step 3: Return IP

```
GSLB response: "Tokyo user → Route to Tokyo datacenter IP (5.6.7.8)"
  Client queries 5.6.7.8
  Gets response from Tokyo datacenter
  Low latency achieved
```

---

## Key Mechanisms

### 1. Geolocation Routing

IP geolocation database maps IP ranges to countries/regions:

```
Query from IP 210.x.x.x (Japan): Route to JP datacenter
Query from IP 77.x.x.x (UK): Route to EU datacenter
Query from IP 200.x.x.x (Brazil): Route to SA datacenter
```

**Accuracy**: ~85% country-level, ~50% city-level (corporate proxies, VPNs skew accuracy)

**Fallback**: If geolocation unknown, route to closest by latency

---

### 2. Health Checks

GSLB monitors all datacenters:

```
Every 30 seconds:
  GSLB → Send health check to each datacenter
  Example: GET /health endpoint
  
If datacenter is down:
  GSLB removes from rotation
  All new queries route to healthy DCs
  
When datacenter recovers:
  GSLB re-adds to rotation
```

**Failure detection latency**: 30-60 seconds (acceptable for most cases)

---

### 3. TTL (Time-To-Live)

Critical for GSLB:

```
SHORT TTL (10 seconds):
  Pros: Quick failover (if DC goes down, 10s until users see new IP)
  Cons: Many DNS queries, increases DNS load
  
LONG TTL (3600 seconds / 1 hour):
  Pros: Fewer DNS queries, less DNS overhead
  Cons: Slow failover (up to 1 hour before users reach new DC)
  
Typical production: 60 seconds (good balance)
```

---

### 4. Load-Based Routing

GSLB can consider datacenter load:

```
Datacenter A: 50% CPU (healthy)
Datacenter B: 90% CPU (near capacity)

Query from user: Route to DC A (has more capacity)
Result: Better distribution of load
```

**Challenge**: Requires real-time metrics feed from all datacenters

---

## Failure Scenarios

### Datacenter Down

```
Before failure:
  Tokyo DC → Healthy → Receives 30% of traffic
  Singapore DC → Healthy → Receives 70% of traffic
  
Tokyo DC fails:
  GSLB detects failure (after 30-60 seconds)
  Removes Tokyo DC from rotation
  
New queries:
  All users → Route to Singapore DC
  Result: Singapore DC capacity exceeded, may be slow

Recovery:
  Tokyo DC comes back online
  GSLB re-adds to rotation
  Traffic gradually shifts back
```

**Prevention**: Over-provisioning or auto-scaling in remaining DCs

---

### DNS Cache Stale

```
User queries @ 10:00:00 → Gets IP of Tokyo DC
DNS resolver caches result (TTL = 1 hour)

Tokyo DC fails @ 10:05:00
GSLB updated immediately, removes Tokyo DC

User queries @ 10:30:00 → DNS resolver serves STALE CACHE
User routed to failed Tokyo DC
Connection fails!

User retries @ 11:00:01 → Cache expired
Gets fresh GSLB response → Routes correctly
```

**Mitigation**: Short TTLs, application-level retries

---

## Implementation Approaches

### 1. DNS-Based GSLB

GSLB logic runs on authoritative DNS server:

```
Query: example.com?

DNS server:
  Check geolocation of query source IP
  Check health of all datacenters
  Return A record of appropriate datacenter IP

Pros: 
  - Transparent to client (standard DNS)
  - Stateless (no connection state)
  
Cons: 
  - DNS has security concerns (DNS poisoning)
  - Limited to DNS (can't consider HTTP-level metrics)
  - TTL limits responsiveness
```

**Examples**: Route53, Cloudflare, Akamai

---

### 2. Anycast + BGP

All datacenters advertise same IP, BGP routes to closest:

```
All DCs: Advertise 10.1.1.1 via BGP
User sends packet to 10.1.1.1

BGP routing:
  Network routes to CLOSEST datacenter by BGP AS-path length
  Result: Automatic geographic routing

Pros: 
  - No DNS involvement (works for any protocol)
  - Fast failover (BGP reconverges in seconds)
  
Cons:
  - Requires BGP expertise
  - Network-level (limited application awareness)
  - Not all traffic patterns supported
```

**Use case**: DDoS mitigation services, CDNs

---

### 3. Dedicated GSLB Appliance

Standalone GSLB device handles all routing logic:

```
User → GSLB appliance (f5.com, Citrix NetScaler)
  Appliance evaluates:
    - Geolocation
    - Health
    - Load
    - Latency
  Returns: Datacenter IP via DNS
```

**Pros**:
  - Full control over logic
  - Health checks, metrics integration
  
**Cons**:
  - Expensive (hardware + licensing)
  - Operational complexity
  - Becomes critical single point of failure

---

## Production Example: Netflix Architecture

```
User queries: example.com

Route 53 (AWS GSLB):
  Checks geolocation
  Checks health of regional LBs
  Returns IP of regional LB

Regional LB (e.g., US-East):
  Distributes to individual service LBs
  
Service LB (e.g., video-streaming-service):
  Distributes to app instances
  Uses consistent hashing for cache affinity
```

**Result**: 3-level hierarchy, geographic + load-aware routing

---

## Performance Implications

### Latency Savings

```
Without GSLB:
  User in Tokyo → US East
  Latency: ~150ms
  
With GSLB:
  User in Tokyo → Tokyo DC
  Latency: ~5ms
  
Improvement: 30x faster!
```

---

### DNS Query Overhead

```
With short TTL (10s):
  100M users × 1 query per 10s = 10M QPS to DNS
  
With long TTL (3600s):
  100M users × 1 query per hour = 27k QPS to DNS
  
Solution: Use CDN for DNS caching + recursive resolvers
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Short TTL** | Long TTL | Fast failover, but higher DNS load |
| **Geolocation** | Latency-based | Geolocation fast, latency more accurate |
| **DNS-based** | Dedicated appliance | Cheaper, simpler, sufficient for most use cases |
| **Anycast** | DNS GSLB | Anycast lower latency, but limited availability features |

---

## Related Fundamentals

- [Load Balancing Algorithms](load-balancing-algorithms.md) – LB strategies within datacenter
- [Networking/DNS](../networking-and-protocols/dns.md) – DNS protocol, TTL, caching
- [Reliability & Resiliency](../reliability-and-resiliency/) – Failover, disaster recovery

---

**Status**: ✅ Complete. Covers geographic routing, health checks, TTL trade-offs.

