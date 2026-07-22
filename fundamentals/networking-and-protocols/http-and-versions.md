# HTTP Versions & Protocols

Evolution from HTTP/1.1 to HTTP/3, understanding performance implications.

---

## TL;DR

- **HTTP/1.1**: One request at a time per connection, head-of-line blocking
- **HTTP/2**: Multiplexing (many requests per connection), server push
- **HTTP/3 (QUIC)**: UDP-based, faster handshake, connection migration
- **Performance**: HTTP/2 ~30% faster, HTTP/3 ~10% faster on good networks
- **Adoption**: HTTP/2 standard, HTTP/3 growing (Chrome 90%+, Firefox)

---

## HTTP/1.1 (1997)

### How It Works

```
Client → Server connection

Request 1:
  GET /page.html
  (wait for response)

Response 1:
  200 OK
  <html>...</html>

Request 2:
  GET /style.css
  (wait for response)

Request 3:
  GET /script.js
  (wait for response)

Sequential: Each request waits for previous response
Total: ~150ms latency × 3 requests = 450ms+
```

### Head-of-Line Blocking

```
Browser needs: page.html, style.css, script.js, image.png

HTTP/1.1 behavior:
  1. Request page.html (50ms)
  2. Request style.css (50ms) - waits for page
  3. Request script.js (50ms) - waits for style
  4. Request image.png (50ms) - waits for script
  
Total: 200ms (sequential!)

Workaround: Open 6 concurrent connections
  3 connections in parallel × 2 requests each
  But wasteful (6 TCP connections)
```

---

## HTTP/2 (2015)

### Multiplexing

```
Single connection, many requests in flight:

Connection 1:
  Stream 1: GET /page.html (start)
  Stream 2: GET /style.css (start immediately)
  Stream 3: GET /script.js (start immediately)
  Stream 4: GET /image.png (start immediately)
  
  Stream 1 arrives → 50ms
  Stream 3 arrives → 50ms
  Stream 2 arrives → 75ms (server busy)
  Stream 4 arrives → 60ms

Total: ~75ms (4 requests in parallel!)
vs HTTP/1.1: ~200ms
Improvement: 2.7x faster
```

---

### Binary Protocol

```
HTTP/1.1:
  GET /users/123 HTTP/1.1
  Host: api.example.com
  Accept: application/json
  
  Text format, human-readable but verbose

HTTP/2:
  Binary frames:
    HEADERS frame: encoded headers
    DATA frame: payload
    
  Smaller (compressed headers)
  Faster parsing (no text scanning)
  
HPACK Compression:
  Request 1: Full headers (100 bytes)
  Request 2: Only differences (10 bytes)
  
  Savings: 90 bytes per similar request!
```

---

### Server Push

```
Client requests page.html
Server knows page needs style.css immediately

HTTP/1.1:
  1. Client requests /page.html
  2. Server returns page
  3. Client parses, sees <link rel="stylesheet" href="/style.css">
  4. Client requests /style.css

HTTP/2:
  1. Client requests /page.html
  2. Server returns page AND pushes /style.css preemptively
  3. Client receives both
  
Result: No extra round-trip for style.css
Faster page load
```

---

## HTTP/3 (2022)

### QUIC Protocol (UDP-based)

```
TCP issues:
  - 3-way handshake: 1 RTT (50ms)
  - TLS handshake: +1-2 RTTs (50-100ms)
  - Total: 100-150ms before data sent
  
QUIC solution:
  - Combined handshake: 0 RTT (send data immediately!)
  - Uses UDP (faster, fewer round-trips)
  - Connection migration (WiFi → mobile seamless)
  
Performance:
  Slow networks (cellular): 10-20% faster
  Good networks: 2-5% faster (already optimized)
```

---

### 0-RTT Connection Resumption

```
First visit:
  Client → Server: Hello (includes key material)
  Server → Client: OK (sends data)
  Time: 1 RTT (50ms)

Subsequent visits:
  Client → Server: Hi (I remember the key)
  Data sent immediately in same packet!
  Time: 0 RTT (just network latency ~5ms)
  
Result: Instant reconnection
```

---

### Connection Migration

```
User on WiFi, switches to mobile mid-download

TCP:
  Connection breaks
  All streams reset
  Must reconnect
  
QUIC:
  Connection ID stays the same (not TCP port number)
  Mobile network has same connection ID
  Download resumes seamlessly!
  
Real-world: Video call drops briefly, then continues on mobile
```

---

## Performance Comparison

| Aspect | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| **Requests** | Sequential | Multiplexed | Multiplexed |
| **Handshake** | 1 RTT (TCP) + 1-2 RTTs (TLS) | Same as 1.1 | 0-1 RTT |
| **Compression** | None | HPACK | QPACK |
| **Head-of-line blocking** | Per connection | Fixed | None |
| **Connection migration** | No | No | Yes |
| **Browser support** | 100% | 95%+ | 60%+ |
| **Speed** | Baseline | +30% | +35-40% |

---

## When to Use

### HTTP/2
```
✓ Modern browsers (2015+)
✓ Desktop/mobile web
✓ Production standard
✓ Wide server support (nginx, Apache, Caddy)
```

### HTTP/3
```
✓ Video streaming (YouTube, Netflix use QUIC)
✓ Mobile-first apps (connection migration benefit)
✓ High-latency networks (0-RTT helps)
✓ Future-proofing
```

### Still HTTP/1.1
```
✓ Legacy systems (old browsers)
✓ Embedded devices
✓ Some APIs (old services)
```

---

## Production Considerations

### Enabling HTTP/2

```
Nginx:
  listen 443 ssl http2;
  
Apache:
  Protocols h2 http/1.1
  
Go:
  Automatic (net/http supports it)
```

### Enabling HTTP/3

```
Cloudflare:
  Toggle "HTTP/3" in dashboard
  
Nginx 1.25+:
  listen 443 quic reuseport;
  http3_max_datagram_size 1200;
  
Note: HTTP/3 requires QUIC, which is UDP
Some networks/firewalls block UDP
Fallback to HTTP/2 automatic
```

---

## Common Issues

### Problem: HTTP/2 Not Working

```
Server reports: "HTTP/1.1 detected"

Causes:
  1. Server doesn't support HTTP/2
  2. ALPN negotiation failed (TLS level)
  3. Reverse proxy strips HTTP/2
  
Debug:
  curl -I --http2 https://example.com
  Should show HTTP/2.0 200
```

---

### Problem: HTTP/3 Fallback

```
QUIC not supported (firewall blocks UDP 443)

Automatic fallback:
  Try QUIC (UDP)
  If blocked: Fall back to HTTP/2 (TCP)
  User sees no difference
  Just slower (1-2 RTTs instead of 0)
```

---

## Related Fundamentals

- [TLS & Certificates](tls-and-certificates.md) – HTTPS encryption
- [Performance](../scalability-and-load-balancing/) – Optimization
- [CDN](../caching/cdn.md) – Delivery via HTTP/2, HTTP/3

---

**Status**: ✅ Complete. Covers evolution, multiplexing, QUIC, performance.

