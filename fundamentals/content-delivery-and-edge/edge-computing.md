# Edge Computing

Execute compute at network edge (geographically distributed servers) instead of centralizing in origin data center. Reduces latency and origin load by processing requests closer to users.

---

## TL;DR

- **Edge computing**: Code execution at edge locations (not origin)
- **Lambda@Edge**: AWS compute at CloudFront edges (~200 locations)
- **Cloudflare Workers**: Serverless functions at Cloudflare edge (~250 locations)
- **Use cases**: Request routing, personalization, authentication, transformations
- **Latency**: 5-50ms vs 100-500ms to origin
- **Cost**: Pay per compute unit, cheaper than origin processing if it reduces misses

---

## Why Edge Compute Matters

### Latency Reduction

```
Centralized (Traditional):
  User → Network (100ms) → Origin Server (50ms) → Response
  Total: 150ms+ (network dominates)

With Edge Compute:
  User → Edge (20ms) → Process locally → Response
  Total: 20-50ms (no round-trip to origin)
  
Difference: 3-10x faster for latency-sensitive operations
```

### Origin Load Reduction

```
Traditional request flow:
  10,000 requests → All hit origin
  Origin processes, caches, returns
  Origin CPU: Overloaded

Edge compute flow:
  10,000 requests → Distributed to 100 edges
  Edges process (aggregate, filter, transform)
  Only 100 requests to origin (aggregated)
  Origin CPU: 100x less load
```

---

## Lambda@Edge (AWS CloudFront)

AWS's serverless compute at CloudFront edge locations.

### Architecture

```
CloudFront edge location (220+ worldwide):
  ├─ Lambda@Edge function (your code)
  ├─ Viewers (incoming requests from users)
  ├─ Origins (upstream servers)
  └─ Cache

Execution triggers:
  1. Viewer Request (user → edge, before cache)
  2. Origin Request (edge → origin, cache miss)
  3. Origin Response (origin → edge, after origin fetch)
  4. Viewer Response (edge → user, before sending user)
```

### Execution Flow

```
GET /api/user/123?city=NYC from User in Tokyo

1. Viewer Request:
   Lambda runs BEFORE cache lookup
   Can modify request (add headers, redirect)
   Example: Check auth token
   
2. Cache lookup:
   If cache HIT: Skip to Viewer Response
   If cache MISS: Proceed to Origin Request
   
3. Origin Request:
   Lambda runs BEFORE fetching origin
   Can modify request sent to origin
   Example: Add X-Forwarded-For header, rewrite URL
   
4. Origin Response:
   Lambda runs AFTER origin returns
   Can modify response (add headers, compress)
   Example: Add security headers, modify Cache-Control
   
5. Viewer Response:
   Lambda runs BEFORE sending to user
   Can modify response headers
   Example: Customize content-type, add CORS headers
   
6. Response sent to user (Tokyo) in ~20ms
```

### Example: Request Authentication

```javascript
// Viewer Request trigger
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  
  // Check auth token
  const token = headers.authorization?.[0]?.value;
  if (!token || !isValidToken(token)) {
    return {
      status: 401,
      body: 'Unauthorized'
    };
  }
  
  // Token valid, continue to origin
  return request;
};

Result: Request validated at edge (Tokyo)
  No need to send to US origin
  Rejected requests stop at edge (save bandwidth)
```

### Example: Request Routing

```javascript
// Viewer Request trigger
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  
  // Route based on user device
  const userAgent = headers['cloudfront-is-mobile-viewer']?.[0]?.value;
  if (userAgent === 'true') {
    request.uri = request.uri + '?device=mobile';
  } else {
    request.uri = request.uri + '?device=desktop';
  }
  
  return request;
};

Result: Same URL serves different content
  No round-trip to origin
  Different cache for mobile/desktop
```

### Lambda@Edge Limits

```
Memory: 128 MB - 3 GB
Timeout: 5 seconds (viewer), 30 seconds (origin)
Code size: 1 MB (uncompressed)
Concurrent executions: Limited per region

Not suitable for:
  Heavy computations (ML, crypto)
  Large data processing
  Anything needing > 30 seconds
```

---

## Cloudflare Workers

Cloudflare's serverless platform at network edge (~250 locations).

### Architecture

```
Cloudflare edge network:
  ├─ Worker (your JavaScript/WebAssembly code)
  ├─ Request interceptor (intercepts all traffic)
  ├─ Cache (built-in KV store)
  ├─ Origins (upstream servers)
  └─ Analytics (built-in logging)

Key difference from Lambda@Edge:
  - Single execution point (not tied to cache events)
  - Can access distributed KV store (key-value cache across edges)
  - Easier to deploy and modify
```

### Execution Flow

```
GET /api/data from user in London

1. Cloudflare receives request
2. Worker executes immediately
3. Worker decides:
   ✅ Serve from cache KV
   ✅ Fetch from origin and cache
   ✅ Transform and return custom response
4. Response sent in 5-50ms
```

### Example: API Gateway at Edge

```javascript
// Cloudflare Worker
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  const path = url.pathname;
  
  // Route to different backends
  if (path.startsWith('/api/users')) {
    return fetch('https://users-api.example.com' + path);
  } else if (path.startsWith('/api/products')) {
    return fetch('https://products-api.example.com' + path);
  } else {
    return fetch('https://main-origin.example.com' + path);
  }
}

Result: Intelligent routing at edge
  Requests routed to specific backend
  Reduced origin latency (direct backend access)
  Load distribution across multiple origins
```

### Example: Distributed Cache (KV Store)

```javascript
// Cloudflare Worker with KV
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  const key = url.pathname + url.search;
  
  // Check KV cache (distributed across all edges)
  let response = await CACHE.get(key);
  if (response) {
    return new Response(response, { status: 200 });
  }
  
  // Cache miss, fetch from origin
  response = await fetch('https://origin.example.com' + url.pathname);
  const data = await response.text();
  
  // Store in KV for 1 hour (propagates to all edges)
  await CACHE.put(key, data, { expirationTtl: 3600 });
  
  return new Response(data);
}

Advantage: True distributed cache
  Set in edge A → Available in edge B, C, D instantly
  No need to fetch from origin when already cached elsewhere
```

### Example: Rate Limiting at Edge

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const ip = request.headers.get('cf-connecting-ip');
  const key = `rate-limit:${ip}`;
  
  // Check rate limit counter in KV
  let count = parseInt(await CACHE.get(key) || '0');
  
  if (count > 100) {
    return new Response('Rate limit exceeded', { status: 429 });
  }
  
  // Increment counter
  await CACHE.put(key, (count + 1).toString(), { expirationTtl: 60 });
  
  // Proceed with request
  return fetch(request);
}

Result: Rate limiting at edge
  Malicious requests blocked before reaching origin
  No bandwidth wasted on rejected requests
  Distributed enforcement (each edge tracks independently)
```

---

## Edge vs Origin Compute: When to Use

| Scenario | Edge | Origin | Reason |
|---|---|---|---|
| **Simple routing** | ✅ | ❌ | Latency sensitive, lightweight |
| **Authentication** | ✅ | ❌ | Block unauthorized early |
| **Personalization** | ✅ | ⚠️ | If simple rules; origin if complex |
| **Database queries** | ❌ | ✅ | Edge can't connect reliably to DB |
| **Aggregation** | ✅ | ⚠️ | Multiple small requests → one aggregated request |
| **Heavy computation** | ❌ | ✅ | ML, large data processing |
| **Real-time analytics** | ✅ | ⚠️ | Edge can log immediately; origin aggregates |
| **Security filtering** | ✅ | ❌ | DDoS, bot detection before origin |

---

## Real-World Patterns

### Pattern 1: Edge-Based API Composition

```
User request: GET /dashboard

Edge Worker:
  1. Request from /api/users (50ms)
  2. Request from /api/posts (50ms)
  3. Request from /api/notifications (50ms)
  Parallel → Total: 50ms
  Combine responses → Return to user (60ms total)

Without edge:
  User requests each separately
  User request → Origin (100ms)
  User request → Origin (100ms)
  User request → Origin (100ms)
  Total: 300ms+ (sequential)

Benefit: 5x faster with edge composition
```

### Pattern 2: Geo-Specific Content

```
Cloudflare Worker:
  1. Detect country from request headers
  2. Fetch region-specific pricing
  3. Fetch region-specific legal terms
  4. Inject into response
  5. Return to user

Result: Custom experience per region
  No additional round-trip to origin
  Compliance (data stays in region)
```

### Pattern 3: Request Deduplication

```
Lambda@Edge Origin Request trigger:

Problem: Multiple users request same uncached object
  User A → Origin (first request, slow)
  User B → Origin (also uncached, slow)
  User C → Origin (also uncached, slow)

Solution: Deduplication lock at edge
  User A: Lock key "resource-123", fetch from origin
  User B: Detects lock, wait for User A's result
  User C: Detects lock, wait for User A's result
  
  User A finishes → Cache populated
  Users B, C → Return from cache

Result: Origin hit only once, users B/C served from cache
```

---

## Production Considerations

### 1. Cold Starts

```
Lambda@Edge cold start: 1-5 seconds (container initialization)
Cloudflare Workers cold start: 5-50ms (lightweight)

Problem: First request slow
Solution: Pre-warm functions before peak traffic
  Periodic healthchecks from multiple regions
  Keeps containers alive
```

### 2. Observability

```
Edge functions distributed across 200+ locations
Logging everything → massive data volume

Best practice:
  1. Sample logs (10-1% depending on volume)
  2. Aggregate metrics (latency, error rate, cache hits)
  3. Alert on anomalies
  4. Store detailed logs only for errors
```

### 3. Testing

```
Local testing:
  Deploy to edge → Test in production regions
  Use Cloudflare local testing or AWS test behavior

Staging:
  Deploy to subset of edges (traffic splitting)
  Monitor errors, latency before full rollout
```

### 4. Cost Analysis

```
Edge execution cost: $0.50 per million requests (Cloudflare)
                    $0.60 per million requests (Lambda@Edge)

Benefits:
  Reduced origin traffic → Lower database load
  Reduced origin bandwidth → Lower egress costs
  Faster responses → Better user experience → Retention

ROI: Payoff in weeks if it reduces origin load significantly
```

---

## References

- AWS Lambda@Edge documentation
- Cloudflare Workers documentation
- "Designing Distributed Systems" — Brendan Burns, Chapter 5

---

## Related Fundamentals

- [CDN Architecture](cdn-architecture.md) – Caching at edge, complementary to compute
- [Content Replication](content-replication.md) – Edge data placement strategies
- [Microservices Architecture](../microservices-architecture/) – Distributed services patterns
- [APIs & Communication](../apis-and-communication/) – Request/response patterns

---

**Status**: ✅ Complete. Covers Lambda@Edge, Cloudflare Workers, use cases, and production patterns.
