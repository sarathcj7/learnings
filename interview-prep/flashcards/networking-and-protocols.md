# Networking and Protocols — Flashcards

**Format**: Question · Answer (collapsed). Review, self-grade, repeat until automatic.

---

## HTTP Versions & Performance

### Q: What problem does HTTP/2 solve that HTTP/1.1 had?
<details><summary>Answer</summary>

Head-of-line blocking: In HTTP/1.1, if one request is slow, it blocks subsequent requests. HTTP/2 uses multiplexing over a single connection to send multiple requests concurrently.

</details>

### Q: What's multiplexing in HTTP/2?
<details><summary>Answer</summary>

Multiple requests and responses share a single TCP connection. Each stream has its own ID and priority. Server can interleave responses (unlike HTTP/1.1 where responses must be in order).

</details>

### Q: When would you migrate from HTTP/1.1 to HTTP/2?
<details><summary>Answer</summary>

When page load time is limited by number of parallel connections or head-of-line blocking (typical for 100+ assets). Check: are users opening 6+ connections? Is latency due to ordering? Then migrate.

</details>

### Q: What's HTTP/3 and why use QUIC instead of TCP?
<details><summary>Answer</summary>

HTTP/3 uses QUIC (UDP-based protocol). Benefit: 1-RTT connection setup (vs 3-RTT in TCP). Better for mobile/lossy networks. QUIC has built-in encryption, stream multiplexing, and connection migration.

</details>

### Q: What's server push in HTTP/2?
<details><summary>Answer</summary>

Server proactively sends resources to client before client requests them. Example: server sends CSS when client requests HTML. Benefit: reduce round trips. Trade-off: risks wasting bandwidth if client caches.

</details>

---

## WebSockets & Real-Time Communication

### Q: When would you use WebSockets vs HTTP polling?
<details><summary>Answer</summary>

WebSocket for true bidirectional real-time (chat, notifications). HTTP polling for periodic updates (status checks every 30s). WebSocket: lower latency, lower overhead. Polling: simpler, stateless.

</details>

### Q: What's the difference between WebSocket and HTTP/2 server push?
<details><summary>Answer</summary>

WebSocket: full bidirectional channel, client initiates upgrade, persistent connection. Server push: server sends data after client request, one-directional (though HTTP/2 allows responses after).

</details>

### Q: How do you handle WebSocket reconnection?
<details><summary>Answer</summary>

Exponential backoff: retry after 1s, 2s, 4s, etc. Maintain sequence numbers to detect missed messages on reconnect. Resync state on reconnect (ask server for latest data). Avoid immediate retry (thundering herd).

</details>

---

## TLS & SSL Security

### Q: How many round trips are in TLS 1.3 handshake?
<details><summary>Answer</summary>

1-RTT: Client sends ClientHello with key share, server responds with ServerHello + finished. Encryption starts immediately. Improvement over TLS 1.2 (2-RTT).

</details>

### Q: What's a certificate chain and why do you need a root CA?
<details><summary>Answer</summary>

Your server has certificate signed by intermediate CA. Intermediate CA has certificate signed by root CA. Root CA is self-signed and pre-installed in browsers. Chain proves your cert is legitimate.

</details>

### Q: How does a client verify a server certificate?
<details><summary>Answer</summary>

1. Check signature (verify with issuer's public key). 2. Verify issuer's cert (chain up to root). 3. Check validity dates. 4. Verify hostname matches certificate CN or SAN. 5. Check cert revocation (OCSP/CRL).

</details>

### Q: What's certificate pinning?
<details><summary>Answer</summary>

Client hardcodes the server's certificate or public key. On connect, verify certificate matches pinned value. Prevents MITM even if CA is compromised. Trade-off: complex to rotate certificates.

</details>

### Q: What's OCSP stapling?
<details><summary>Answer</summary>

Server proactively gets certificate revocation status (OCSP) and includes it with certificate. Avoids client needing to contact OCSP responder (which is slow/expensive). Better privacy and performance.

</details>

---

## DNS & Resolution

### Q: How many DNS queries are needed to resolve www.example.com?
<details><summary>Answer</summary>

Typically 4: Client queries recursive resolver. Resolver queries root nameserver (where is .com?). Root responds with TLD nameserver. Resolver queries TLD (where is example.com?). TLD responds with authoritative nameserver. Resolver queries authoritative (what's IP of www.example.com?). Authoritative responds with IP.

</details>

### Q: What's DNS TTL and why does it matter?
<details><summary>Answer</summary>

TTL (Time To Live) is how long a DNS record can be cached. High TTL (3600s): fewer queries, slower updates. Low TTL (60s): faster updates, more queries. For load balancing: low TTL. For static records: high TTL.

</details>

### Q: Why do users sometimes see old IPs after DNS change?
<details><summary>Answer</summary>

Multiple cache layers: browser cache, OS cache, ISP resolver, intermediate resolvers all cache separately. Each has its own TTL. Some caches ignore TTL (browser cache ~1 hour). Result: propagation takes hours to days.

</details>

### Q: What's GeoDNS?
<details><summary>Answer</summary>

DNS resolves queries based on client's geography/IP. Different users get different answers (route to nearest datacenter). Benefit: low latency. Trade-off: inconsistent results, complex failover.

</details>

---

## Load Balancing & Connection Management

### Q: What's connection pooling and why is it important?
<details><summary>Answer</summary>

Reuse TCP connections instead of opening new ones per request. Benefit: avoid 3-way handshake overhead (50-100ms), SSL handshake (200ms+). Trade-off: manage connection lifecycle, avoid connection leaks.

</details>

### Q: What's keep-alive in HTTP?
<details><summary>Answer</summary>

HTTP Keep-Alive: reuse TCP connection for multiple requests. Client sends "Connection: keep-alive", server keeps connection open. Reduces latency from multiple round trips of connection setup.

</details>

### Q: How do you handle sticky sessions (load imbalance)?
<details><summary>Answer</summary>

Sticky sessions route same client to same backend (using cookies). Problem: if backend goes down, sticky clients are disconnected. Solution: store session in shared cache (Redis) so any backend can serve.

</details>

### Q: What's a hot shard in load balancing?
<details><summary>Answer</summary>

One partition/server gets disproportionate traffic (e.g., popular user in partition 0, but only partition 0 is loaded). Causes that server to be bottleneck. Solution: re-partition by different key, or use consistent hashing with virtual nodes.

</details>

---

## Protocol Selection & Trade-offs

### Q: When would you use UDP instead of TCP?
<details><summary>Answer</summary>

UDP for low-latency, best-effort delivery (gaming, video streaming, DNS). No connection overhead, no retries. Accept packet loss. TCP for reliability (web, email, file transfer).

</details>

### Q: What's TCP congestion control?
<details><summary>Answer</summary>

Algorithm to avoid network congestion. Sender gradually increases throughput (slow start), detects packet loss as signal of congestion, backs off. Balances utilization vs latency.

</details>

### Q: What's Nagle's algorithm?
<details><summary>Answer</summary>

Wait for ACK before sending next small packet (or accumulate until full MSS). Reduces tiny packets on network. Problem: adds latency (undesirable for interactive apps). Solution: disable with TCP_NODELAY.

</details>

### Q: What's TCP fast open (TFO)?
<details><summary>Answer</summary>

Client can send data before 3-way handshake completes. Server issues TFO cookie on first connection. Client uses cookie on next connection to skip 1 RTT. Benefit: 1-RTT reduction (especially for short connections).

</details>

---

## Self-Scoring Guide

- Mark each flashcard: immediate answer (✓) or struggled (✗)
- Review ✗ cards daily until they become automatic
- Interview readiness: 15+ correct on first pass
