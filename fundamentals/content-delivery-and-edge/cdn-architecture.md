# CDN Architecture

Content Delivery Networks cache content at edge locations worldwide to reduce latency and origin load.

---

## TL;DR

- **CDN**: Distributed servers at geographic locations serving cached content
- **Edge locations**: Data centers physically close to users (reduced latency)
- **Origin**: Main server holding source content
- **Cache headers**: `Cache-Control`, `ETag`, `Last-Modified` control what/how long to cache
- **Hit ratio**: % of requests served from cache; 95%+ is excellent
- **Origin shield**: Extra cache layer between edge and origin (protects origin from thundering herd)

---

## CDN Basics

### Why CDNs Matter

```
Without CDN:
  User in Tokyo → Request → Server in New York
  Latency: 300ms (network + server processing)
  
With CDN:
  User in Tokyo → Request → Edge server in Tokyo
  Latency: 50ms (network only, server nearby)
  Origin server traffic: Reduced 99% (edge caches most requests)
```

### How CDN Works

```
1. User requests content: GET /image.png
2. DNS resolves to nearest CDN edge location
3. Edge server checks cache:
   ✅ Cache hit → Return immediately (cached)
   ❌ Cache miss → Fetch from origin, cache, return
4. Next user in same region → Cache hit → Fast response
```

---

## Cache Layers

### L1: Browser Cache
```
Browser stores files locally (CSS, images, JS)
Expires per HTTP headers
User refreshes: Fetches from browser cache (instant)
```

### L2: CDN Edge Cache
```
CDN edge server caches popular content
Shared by all users in geographic region
User A requests → CDN caches it
User B requests → Gets from CDN (milliseconds)
Hit ratio: Usually 80-95% (popular content dominates)
```

### L3: CDN Origin Shield
```
Optional layer between edge and origin
Deduplicates requests to origin during spike
Example: 100 edge servers all requesting same uncached object
  Without shield: 100 requests to origin
  With shield: 1 request to origin, shield caches, serves 99 edges
```

### L4: Origin

Main server with authoritative content.

```
Cache hit → Edge returns (fast)
Cache miss → Request goes to origin
  Origin processing adds latency
  High-traffic events: Origin gets hammered (must be protected)
```

---

## Cache Headers

### Cache-Control (Main Control)

```
Cache-Control: max-age=3600
  → Cache for 3600 seconds (1 hour)

Cache-Control: max-age=3600, public
  → Cache for 1 hour, shareable by all caches (edge, proxy, browser)

Cache-Control: max-age=3600, private
  → Cache for 1 hour, only in browser (not CDN)

Cache-Control: no-cache
  → Must revalidate before serving (not "don't cache")

Cache-Control: no-store
  → Never cache (sensitive data)

Cache-Control: max-age=0, must-revalidate
  → Cache expired immediately, must check origin (rarely used)

Practical example:
  Profile page: max-age=60, private (user-specific, browser only)
  Homepage: max-age=3600, public (shared, cacheable by CDN)
  User uploads: max-age=31536000, public (immutable, cache forever)
```

### ETag (Entity Tag)

```
Response header: ETag: "abc123"
  → Content identifier (hash of file)

Client subsequent request:
  If-None-Match: "abc123"
  → If unchanged, server returns 304 Not Modified
  → Browser uses cached version (no re-download)

Advantage over Last-Modified:
  Detects ANY change (even clock-skewed servers)
  
CDN use:
  Small responses (304 is tiny, saves bandwidth)
  Validation doesn't re-download content
```

### Last-Modified

```
Response header: Last-Modified: Wed, 01 Jan 2026 00:00:00 GMT
  → File modification time

Client subsequent request:
  If-Modified-Since: Wed, 01 Jan 2026 00:00:00 GMT
  → If unchanged, server returns 304 Not Modified

Less reliable than ETag (clock skew, timezones)
Usually combined with ETag
```

---

## Cache Hit Ratio

Hit ratio: (cache hits) / (total requests) × 100%

### Measuring

```
CDN logs show:
  Request 1: Cache miss → Origin fetch
  Request 2-100: Cache hit (same resource)
  
Hit ratio = 99/100 = 99%

Excellent range: 85-95%+
  Most traffic is repeat content (homepage, images, CSS)

Poor hit ratio < 70%:
  Personalized content (can't cache)
  Short cache TTL
  Insufficient edge capacity
```

### Improving Hit Ratio

| Strategy | Tradeoff |
|---|---|
| Increase TTL (max-age) | Content becomes stale, users see old version |
| Cache more aggressively | Use cache keys wisely (avoid personalizations) |
| Add origin shield | Extra hop, adds ~5-10ms, protects origin |
| Smart cache keys | Don't include user ID in cache key if possible |
| Prefetch popular content | Populate cache before demand spike |

### Cache Invalidation

```
When to purge cache:
  1. Site deploy (code changed)
  2. User uploads new image
  3. Price update
  4. Data change

Strategies:
  1. TTL expiry: Wait for max-age to expire (simplest, passive)
  2. Purge API: CDN provider purges immediately (fast)
  3. Versioned URLs: /image-v2.png instead of /image.png (immutable)
```

---

## Origin Load Protection

### Thundering Herd

```
Popular event: Celebrity tweet, viral video
  100k users request simultaneously
  
Without CDN edge caching:
  100k requests hit origin
  Origin becomes bottleneck (crashes)

With CDN edge caching:
  100k requests → distributed to 10+ edge locations
  Each edge caches → next 10k requests served from edge
  Origin hit only 1-5% (handles easily)

Origin shield protects further:
  Multiple edges requesting same uncached object → shield deduplicates
  Origin sees 1 request instead of 100
```

### Cache Stampede

```
When does it happen:
  Popular content cache expires (e.g., max-age=3600 expires)
  Simultaneous requests before revalidation
  
Without protection:
  1000 requests arrive at exact expiry time
  All miss cache
  All hit origin simultaneously
  Origin can crash

Solution 1: Cache-Control: stale-while-revalidate
  Serve stale copy while revalidating in background
  
  Cache-Control: max-age=3600, stale-while-revalidate=86400
  → Use cache up to 1 hour (fresh)
  → Use cache up to 24 hours while fetching new copy (stale but available)

Solution 2: Probabilistic early expiration
  Don't expire at exact time, randomize by 5-10%

Solution 3: Origin shield with deduplication
  Shield caches, forwards one request to origin
```

---

## Advanced CDN Concepts

### Geo-Routing

```
CDN routes users to nearest edge:
  User in London → London edge server
  User in Sydney → Sydney edge server
  User in San Francisco → San Francisco edge server

Improves:
  Latency (geographic proximity)
  Local compliance (data residency)
  Bandwidth costs (local traffic stays local)
```

### Tiered Caching

```
Global CDN hierarchy:
  User browser → Nearest edge (Level 1)
    ↓ miss
  Regional hub (Level 2)
    ↓ miss
  Origin shield (Level 3)
    ↓ miss
  Origin

Each layer caches, reducing origin load
Popular content spreads up from origin automatically
```

### Smart Cache Expiry

```
Adaptive TTL based on request patterns:
  High-traffic content → Keep longer (worth caching)
  Low-traffic content → Expire sooner (save space)
  
CDN analyzes patterns:
  "users.html" downloaded 10k times/hour → cache 24 hours
  "admin-report.csv" downloaded 2 times/hour → cache 1 hour
```

---

## Production Considerations

### 1. Cache Key Construction

```
❌ Bad: cache key = URL only
  GET /products?sort=price&user=alice
  GET /products?sort=price&user=bob
  → Different users, same product list, separate cache entries (wasted)

✅ Good: cache key = normalized query params
  Normalize: sort by param name, ignore user-specific params
  Both URLs → Same cache key → Shared across users
  
Rule: Cache key should NOT include user ID unless needed
```

### 2. HTTPS and CDN

```
Protocol choice:
  HTTP → CDN over public internet (cheaper, riskier)
  HTTPS → CDN over secure tunnel (required for auth, payments)

TLS termination at edge:
  User → HTTPS → CDN edge (TLS handshake)
  CDN → origin (reuse or new TLS)
  Advantage: Origin doesn't handle TLS overhead
  Disadvantage: Trust management complex
```

### 3. Monitoring Cache Health

```
Key metrics:
  1. Hit ratio: Target 85%+
  2. Origin requests/sec: Should be 1-5% of total traffic
  3. Cache eviction: Objects aging out (space full)
  4. Stale serving: How often stale-while-revalidate triggered
  5. Edge latency: p95 < 100ms ideally
```

### 4. Cost Optimization

```
Bandwidth cost: $0.02-0.10 per GB (varies by region)
Cache storage: $0.01-0.05 per GB/month

To reduce costs:
  1. Increase cache TTL (cache longer, fewer misses)
  2. Compress responses (gzip, brotli)
  3. Delete unused edge locations
  4. Use origin shield selectively (only for high-traffic)
  5. Lazy purge (wait for TTL, don't force purge)
```

---

## Real-World Examples

### Scenario 1: Website Spike

```
Event: Product launch announcement
Expected traffic: 10x normal

Without CDN: Origin must handle 10x (buy servers, hard)
With CDN: 95% cached → Origin handles only 0.5x normal (trivial)

Result: Smooth experience, no origin scaling needed
```

### Scenario 2: Video Streaming

```
Video file: 500 MB
Users: 1M
Storage at origin: 1M × 500 MB = 500 TB (impossible)

CDN solution:
  Store original in origin (500 MB)
  Each edge caches only popular videos
  Distribution: 50 edges, average 100 popular videos each
  Total cache: 50 × 100 × 500 MB = 2.5 TB (feasible)
  
Unpopular videos: Cached on-demand per region
Popular videos: Automatically cached everywhere
```

---

## References

- "High Performance Browser Networking" — Ilya Grigorik, Chapter 7
- HTTP Caching specification: RFC 7234
- Cloudflare CDN guide
- AWS CloudFront best practices

---

## Related Fundamentals

- [Edge Computing](edge-computing.md) – Compute at edge, not just caching
- [Content Replication](content-replication.md) – Multi-region distribution strategies
- [APIs & Communication](../apis-and-communication/) – HTTP headers, compression
- [Scalability/Load Balancing](../scalability-and-load-balancing/) – Geographic load balancing

---

**Status**: ✅ Complete. Covers CDN architecture, cache layers, headers, hit ratio, and production patterns.
