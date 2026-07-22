# DNS Resolution

Understanding how domain names map to IP addresses, DNS caching, TTL, and common DNS patterns in system design.

---

## TL;DR

- **DNS**: Translates example.com → 1.2.3.4
- **Recursive resolver**: Client's local DNS server (ISP or 8.8.8.8)
- **TTL**: Time-to-live (how long to cache, typically 300s-3600s)
- **Short TTL**: Fast failover (~10s), but more DNS queries
- **Long TTL**: Fewer queries, but slower failover (~1 hour)
- **GSLB**: Uses DNS to route based on geography/health

---

## DNS Query Path

### Step 1: Client Sends Query

```
User types: example.com
Browser checks local cache (none)
Browser sends: Recursive query to resolver
  Query: "What's the IP for example.com?"
  Resolver: 8.8.8.8 (Google) or local ISP resolver
```

---

### Step 2: Resolver Queries Authoritative Servers

Resolver performs iterative queries:

```
Resolver: "What's the IP for example.com?"

Step 1: Query root nameserver (.com)
  Root: "Ask the .com TLD server"
  
Step 2: Query .com TLD server
  TLD: "Ask ns1.example.com (authoritative)"
  
Step 3: Query authoritative nameserver
  Authoritative: "example.com = 1.2.3.4"
  
Resolver caches result, returns to client
```

---

### Step 3: Caching

```
Client caches result:
  example.com → 1.2.3.4
  TTL: 3600 (cache for 1 hour)

Resolver caches result:
  example.com → 1.2.3.4
  TTL: 3600
  
Result: Next queries to example.com are instant (no network)
```

---

## TTL Impact on Failover

### Short TTL (10 seconds)

```
Primary server: 1.2.3.4 (healthy)
Client caches: 1.2.3.4 with TTL 10s

Primary fails at T=5s:
  Client still has cached IP (5s left in TTL)
  Client sends requests to dead server
  Requests fail
  
At T=10s:
  Cache expires
  Client queries DNS again
  Gets new IP: 1.2.3.5 (failover server)
  Traffic resumes to healthy server
  
Downtime: 5 seconds (max)
```

---

### Long TTL (3600 seconds)

```
Primary server: 1.2.3.4
Client caches: 1.2.3.4 with TTL 3600s

Primary fails at T=100s:
  Client still cached (3500s left)
  Requests fail for up to 3500s
  
At T=3600s:
  Cache expires
  Gets failover IP: 1.2.3.5
  
Downtime: Up to 1 hour! (unacceptable)
```

---

## DNS Record Types

### A Record (IPv4)

```
example.com A 1.2.3.4

Resolution: example.com → 1.2.3.4
```

---

### AAAA Record (IPv6)

```
example.com AAAA 2001:db8::1

Resolution: example.com → 2001:db8::1
```

---

### CNAME Record (Alias)

```
api.example.com CNAME api-load-balancer.example.com
api-load-balancer.example.com A 1.2.3.4

Resolution:
  1. Query api.example.com
  2. Get CNAME → api-load-balancer.example.com
  3. Query api-load-balancer.example.com
  4. Get A → 1.2.3.4
```

---

### MX Record (Mail)

```
example.com MX 10 mail.example.com
example.com MX 20 mail2.example.com

Resolution (for email):
  Priority 10: mail.example.com (primary)
  Priority 20: mail2.example.com (backup)
```

---

### NS Record (Nameserver)

```
example.com NS ns1.example.com
example.com NS ns2.example.com

Result: Queries delegated to ns1 and ns2
```

---

## DNS Patterns in System Design

### Pattern 1: Round-Robin DNS

```
example.com A 1.2.3.4
example.com A 1.2.3.5
example.com A 1.2.3.6

Resolution:
  Query 1: 1.2.3.4
  Query 2: 1.2.3.5
  Query 3: 1.2.3.6
  Query 4: 1.2.3.4 (cycle)
  
Result: Simple load balancing at DNS level
Downside: No health checks, no control over distribution
```

---

### Pattern 2: GSLB (Route by Geography)

```
DNS server receives query from Tokyo:
  Resolves example.com → 5.6.7.8 (Tokyo datacenter)
  
Same query from London:
  Resolves example.com → 3.4.5.6 (London datacenter)
  
Result: Geographic routing, minimizes latency
Implementation: Route53, Cloudflare, etc.
```

---

### Pattern 3: Weighted Routing

```
example.com A 1.2.3.4 (weight 70%)
example.com A 1.2.3.5 (weight 30%)

Result:
  70% of queries → 1.2.3.4
  30% of queries → 1.2.3.5
  
Use: Gradual rollout (send 10% to new server)
```

---

## DNS Failures & Mitigation

### Failure 1: DNS Server Down

```
If authoritative nameserver for example.com goes down:
  All queries for example.com fail
  Domain becomes unreachable
  
Mitigation:
  Multiple nameservers (ns1, ns2, ns3)
  Geographically distributed
  Different providers (no single point of failure)
```

---

### Failure 2: Long Propagation Time

```
Update: Change example.com A record from 1.2.3.4 to 1.2.3.5

Propagation:
  T=0s: Update authoritative nameserver
  T=30s: Resolvers start seeing new IP
  T=300s: Most resolvers updated (TTL dependent)
  T=3600s: All resolvers updated
  
During propagation: Clients split between old and new IP
  50% hit old server (might be scaling down)
  50% hit new server (might not be ready)
  
Mitigation: Set short TTL before change, then revert after
```

---

### Failure 3: DNS Poisoning

```
Attacker hijacks DNS query:
  Client: "What's IP for example.com?"
  Attacker intercepts: "It's attacker.com's IP"
  Client connects to attacker site
  Attacker steals credentials
  
Mitigation:
  DNSSEC (cryptographic signatures)
  Use trusted resolvers (8.8.8.8, 1.1.1.1)
  VPN/TLS to prevent interception
```

---

## TTL Best Practices

```
Dynamic services (elastic, auto-scaling):
  TTL: 10-60 seconds
  Allows fast failover when instances change
  Example: Kubernetes services

Static infrastructure:
  TTL: 3600-86400 seconds (1 hour to 1 day)
  Reduces DNS queries significantly
  Example: CDN endpoints that rarely change

Failover scenarios:
  Normal TTL: 300-3600s
  Before planned maintenance: Reduce to 10s
  After maintenance: Restore to normal
```

---

## DNS Query Optimization

### Problem: DNS Query Overhead

```
Scenario: Mobile app makes 1000 requests/day

Each request involves DNS query:
  Round-trip: 50ms average (might be slow on mobile)
  1000 × 50ms = 50 seconds wasted!
  
Actual network time: 50% of total request time!
```

---

### Solutions

**1. DNS Caching** (application-level):
```
First query: 50ms (network)
Subsequent queries: 0.1ms (cache)

Cache TTL: Match DNS TTL (e.g., 300s)
Result: 1 DNS query every 300s (instead of every request)
```

**2. Connection Reuse**:
```
Instead of:
  DNS query → Connect → HTTP request → Disconnect

Do:
  DNS query → Connect → HTTP request (keep-alive)
  → HTTP request (same connection)
  → HTTP request (same connection)
  
Result: DNS query amortized over many requests
```

**3. Pre-resolution**:
```
Application startup: Resolve all known domains
  example.com → 1.2.3.4 (cached)
  api.example.com → 1.2.3.5 (cached)
  cdn.example.com → 5.6.7.8 (cached)
  
First user request: Use cached IPs, no DNS query
```

---

## Production Checklist

- [ ] Multiple authoritative nameservers (at least 2)
- [ ] TTL appropriate for change frequency
- [ ] Failover DNS records configured
- [ ] Health checks on DNS records (if supported)
- [ ] DNS query monitoring (response time, failure rate)
- [ ] DNSSEC enabled for security
- [ ] DNS caching in application
- [ ] Connection reuse (keep-alive) configured

---

## Related Fundamentals

- [GSLB](../scalability-and-load-balancing/gslb.md) – Geographic DNS routing
- [Load Balancing](../scalability-and-load-balancing/load-balancing-algorithms.md) – Complements DNS LB
- [Reliability](../reliability-and-resiliency/) – DNS failure detection

---

**Status**: ✅ Complete. Covers resolution, TTL, records, patterns, failures, optimization.

