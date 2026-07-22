# CDN Caching

## TL;DR

- **CDN = Content Delivery Network**: Distributed caches at edge (geographically close to users).
- **Cache Headers**: `Cache-Control`, `ETag`, `Last-Modified` tell CDN how long to cache.
- **Hit Ratio**: % of requests served from CDN (not origin). 80%+ is good.
- **Purge**: Manually remove content from CDN when update is urgent.
- **TLS termination**: CDN decrypts HTTPS, caches plaintext, re-encrypts to client.

---

## CDN Architecture (from app's perspective)

### How It Works

```
User in Tokyo requests image.jpg

1. DNS resolves cdn.example.com → closest PoP (Point of Presence) in Tokyo
2. Request hits Tokyo PoP edge cache
3. If hit: Serve from cache (sub-10ms latency)
4. If miss: Edge cache fetches from origin (e.g., S3 in us-west-2)
   - Might fetch from origin shield (intermediate cache closer to origin)
   - Cache result at edge for next user
5. Response: ~50ms for cache miss (origin latency) vs ~10ms for cache hit
```

### PoPs (Points of Presence)

Major CDNs (Cloudflare, Akamai, AWS CloudFront) operate hundreds of PoPs globally. Each PoP is a small data center with cache servers.

```
PoP Tokyo
├── Cache servers (hot URLs)
├── TLS termination
└── Load balancer

User in Tokyo → DNS → 123.45.67.89 (PoP Tokyo) → cache hit → 10ms
User in London → DNS → 234.56.78.90 (PoP London) → cache hit → 10ms
```

---

## Cache Headers

Tell CDN (and browsers) how long to cache content.

### Cache-Control

```
Cache-Control: public, max-age=3600

Meaning:
- public: Can be cached by any cache (CDN, browser, proxies)
- max-age=3600: Cache for 3600 seconds (1 hour)

Effect: CDN caches this URL for 1 hour. After 1 hour, CDN re-fetches from origin.
```

### Common Header Values

| Header | Meaning | Example |
|---|---|---|
| `max-age=3600` | Cache for 3600 seconds | Static images, CSS, JS |
| `max-age=0, must-revalidate` | Don't cache, validate every time | Dynamic pages (no-cache-but-check) |
| `no-cache` | CDN can store, but must check freshness before serving | API responses |
| `no-store` | Never cache | Sensitive data, auth tokens |
| `private` | Only browser cache, not CDN | User-specific data |
| `s-maxage=7200` | Override max-age for shared caches (CDN), use max-age for browser | CDN caches 2h, browser caches less |

### ETag (Entity Tag)

Hash of content. If content hasn't changed, server returns 304 Not Modified (no re-download).

```
Request 1:
  GET /image.jpg
  Response: 200 OK
  ETag: "abc123"
  [body: 5MB image]

Request 2 (after browser cache expired):
  GET /image.jpg
  If-None-Match: "abc123"
  Response: 304 Not Modified
  [no body, saves 5MB bandwidth]
```

**CDN Use**: On cache miss, CDN sends `If-None-Match: <old-ETag>` to origin. If origin says 304, CDN reuses cached copy. Saves origin bandwidth.

### Last-Modified

Timestamp of content. Similar to ETag but time-based.

```
Response: Last-Modified: Mon, 23 Jul 2026 10:00:00 GMT

Next request:
  If-Modified-Since: Mon, 23 Jul 2026 10:00:00 GMT
  Response: 304 Not Modified (if not modified since that time)
```

---

## Cache Hit Ratio

**Hit Ratio = Hits / (Hits + Misses)** = % of requests served from CDN edge.

### Factors Affecting Hit Ratio

1. **Cache-Control headers**: Too short TTL → frequent misses.
2. **Content type**: Static (images, JS, CSS) = 95%+ hit ratio. Dynamic (HTML) = 50-80%.
3. **Geographic distribution**: If users are in many countries, hit ratio might be lower initially (CDN needs to warm each PoP).
4. **Cache key**: URL-based caching means `example.com/image.jpg?v=1` and `example.com/image.jpg?v=2` are different cache entries (waste).

### Improving Hit Ratio

1. **Normalize query parameters**: Strip unused params from cache key.
2. **Long TTLs for static content**: Use max-age=31536000 (1 year) for versioned assets (app.js?v=abc123).
3. **Version your assets**: Include hash in filename so cache never needs invalidation (v1/app.js, v2/app.js).
4. **Combine small files**: Reduce number of cache entries, increase hit ratio (fewer unique URLs).

### Typical Hit Ratios

- **Static site** (no dynamic content): 95%+
- **E-commerce** (mix of static + dynamic): 60-80%
- **SaaS** (mostly dynamic): 30-50%
- **News site** (mix, but often stale OK): 70-85%

---

## Cache Invalidation

How to refresh content when source updates.

### TTL-Based (Lazy)

Set short TTL, wait for expiration. Simple, no active invalidation.

```
Cache-Control: max-age=300  # 5 minutes

Effect: After any update, users see new version within 5 min.
Trade-off: Staleness up to TTL, but simple.
```

### Purge (Active)

Manually tell CDN to remove content immediately.

```
CDN API:
  POST /purge
  { urls: ["/image.jpg", "/api/posts"] }

Effect: Removed from all PoPs within seconds.
When to use: Urgent updates (security patch, bug fix).
Cost: Manual action, not scalable for frequent invalidation.
```

### Versioning (Cache-Busting)

Include version in URL. On update, change version.

```
Old: /app.js (cached)
Update: /app.js?v=2 (new URL, not in cache, forces fetch from origin)

Benefit: No purge needed, automatic invalidation on version change.
Deploy: Update HTML to reference /app.js?v=2, old cache remains (no harm).
```

### Surrogate-Key Invalidation

Tag multiple URLs with a key. Purge by key, invalidates all tagged URLs.

```
Cache header: Surrogate-Key: user-profile, user-123

Later, update user 123's profile:
  POST /purge
  { surrogate-key: "user-123" }

Effect: All URLs tagged with "user-123" are invalidated.
Benefit: Invalidate related content with one API call.
```

---

## Performance Implications

### Cache Hit: ~10-50ms

```
Request → GeoDNS → PoP Tokyo → Cache hit → TLS decrypt → serve → Client (10ms)
```

### Cache Miss: ~50-200ms

```
Request → GeoDNS → PoP Tokyo → Cache miss → Origin Fetch (100-150ms) → Cache → serve → Client (150ms)
```

### Origin Overload

If hit ratio drops (many misses), origin traffic spikes. Protect with rate limiting or auto-scaling.

```
Normal: 10k QPS, 80% CDN hit rate → 2k origin requests
Degraded: 10k QPS, 20% hit rate (origin down?) → 8k origin requests (80x traffic spike)
Solution: Rate limit CDN→origin requests, queue excess, degrade gracefully.
```

---

## TLS & Encryption

CDN terminates TLS connection to client, re-encrypts to origin.

```
Browser → (HTTPS) → PoP → (HTTP or HTTPS) → Origin

PoP decrypts HTTPS, caches plaintext, re-encrypts to client.

Pros: CDN CPU handles TLS, Origin can use cheaper HTTP.
Cons: CDN sees plaintext content (privacy trade-off), requires trust in CDN provider.

Alternative: End-to-end TLS
Browser → (HTTPS) → PoP → (HTTPS) → Origin
CDN can't read content (more secure, but higher cost).
```

---

## Production Considerations

1. **Monitor hit ratio**: Alert if drops below baseline (might indicate misconfiguration or attack).
2. **Purge strategy**: Document when/how to purge. Manual purges should be rare.
3. **Cost**: Egress from CDN is cheaper than origin egress, but not free. Monitor bandwidth.
4. **Cache poisoning**: Don't cache error pages (5xx) for long. Someone might deliberately trigger error to pollute cache.
5. **Bypass for origin-only requests**: Some requests should skip CDN (e.g., `/admin` paths). Use cache policy or origin headers.

---

## References

- "HTTP Caching" — MDN Web Docs
- CloudFront documentation (AWS)
- Cloudflare Cache documentation

---

## Related Fundamentals

- [Strategies](strategies.md) – Read-through strategy relevant to CDN
- [Cache Invalidation](invalidation.md) – Detailed invalidation strategies
- [Content Delivery & Edge](../content-delivery-and-edge/cdn-architecture.md) – CDN's internal distributed architecture
