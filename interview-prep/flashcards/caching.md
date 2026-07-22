# Caching — Flashcards

**Format**: Question · Answer (collapsed). Review, self-grade, repeat until automatic.

---

## Caching Strategies

### Q: What's cache-aside?
<details><summary>Answer</summary>

Client checks cache on every request. On miss, fetches from source and populates cache. Simple, resilient (cache failure doesn't break app), but requires dual writes. Most common pattern.

</details>

### Q: When is write-through better than cache-aside?
<details><summary>Answer</summary>

When strong consistency is critical (financial transactions, inventory). Write-through guarantees cache and source are always in sync. Trade-off: slower writes.

</details>

### Q: What's the difference between read-through and cache-aside?
<details><summary>Answer</summary>

Cache-aside: Client manages cache logic. Read-through: Cache layer manages fetching from source. Read-through is cleaner separation but more complex.

</details>

### Q: When would you use write-behind caching?
<details><summary>Answer</summary>

High-throughput write systems where eventual consistency is OK (metrics, analytics, social likes). Writes return immediately (in-memory), asynchronously sync to DB. Risk: data loss on crash.

</details>

---

## Eviction Policies

### Q: What's LRU and why is it common?
<details><summary>Answer</summary>

Least Recently Used. Assumes recently accessed data will be accessed again (temporal locality). Simple to implement, 80%+ hit rates in practice. Used in Redis, memcached.

</details>

### Q: What's the "scan attack" on LRU?
<details><summary>Answer</summary>

A full-scan query accessing all data sequentially evicts the entire hot set (used once, never again). Mitigation: probabilistic insertion, or prioritize hot data separately.

</details>

### Q: LRU vs LFU: when would you choose LFU?
<details><summary>Answer</summary>

LFU for skewed distributions where 10% of keys are 90% of traffic. LFU protects hot keys even if not accessed recently. Trade-off: more complex, O(log N) eviction.

</details>

### Q: What's the cold-start problem with LFU?
<details><summary>Answer</summary>

New keys start at frequency=1, vulnerable to immediate eviction. After time, frequency counts inflate (old keys have frequency=1M). TTL-based eviction avoids this.

</details>

### Q: How long should TTL be for cached user profiles?
<details><summary>Answer</summary>

10-30 minutes is typical. Too long (1h) = stale data frustrates users. Too short (1min) = frequent misses, origin overloaded. Tune based on write frequency.

</details>

---

## Distributed Caching

### Q: What's consistent hashing and why use it?
<details><summary>Answer</summary>

Maps keys and nodes to a ring (0 to 2^32-1). Key maps to next node clockwise. On node add/remove, only 1/N of keys rehash (vs all in naive modulo hashing).

</details>

### Q: What are virtual nodes in consistent hashing?
<details><summary>Answer</summary>

Each physical node has multiple positions on the ring (e.g., 100+ virtual nodes per physical). Balances load across heterogeneous hardware and smooths resharding.

</details>

### Q: Would you replicate cache (full copy on each node) or partition (shard)?
<details><summary>Answer</summary>

Partition (shard) for scale, replicate for fault tolerance. In production: partitioned + replicated (each shard has primary + 2 replicas). Scales writes, fault-tolerant.

</details>

### Q: What happens when a Redis cluster node fails?
<details><summary>Answer</summary>

If primary fails, cluster promotes a replica to primary (automatic failover, ~30s). Clients are redirected. Data is safe if replication was fast enough (RPO = replication lag).

</details>

### Q: What's a "hot spot" in a distributed cache?
<details><summary>Answer</summary>

One partition gets disproportionate traffic (e.g., celebrity profile). Mitigation: replicate hot key across all nodes, or use local (in-process) cache for known hot keys.

</details>

---

## CDN Caching

### Q: What does `Cache-Control: max-age=3600` do?
<details><summary>Answer</summary>

Tells CDN/browser to cache for 3600 seconds. After that, CDN refetches from origin. Longer TTL = fewer origin requests but higher staleness risk.

</details>

### Q: What's an ETag and when is it useful?
<details><summary>Answer</summary>

Hash of content. If client/CDN sends `If-None-Match: <old-ETag>` and content hasn't changed, server returns 304 Not Modified (saves re-downloading). Reduces bandwidth.

</details>

### Q: What's a good CDN cache hit ratio?
<details><summary>Answer</summary>

80%+ is excellent. 50-80% is good. <50% suggests short TTL or dynamic content. Track hit ratio and alert on drops (might indicate misconfiguration or attack).

</details>

### Q: How does CDN handle HTTPS?
<details><summary>Answer</summary>

CDN terminates TLS (decrypts), caches plaintext, re-encrypts to client. Saves origin CPU, but CDN sees plaintext. Alternative: end-to-end TLS (more secure, higher cost).

</details>

### Q: How do you invalidate a CDN cache entry?
<details><summary>Answer</summary>

Purge API: send URL list to CDN to remove. Or version URLs (app.js?v=1 → app.js?v=2). Or rely on TTL. Versioning is best (no purge needed).

</details>

---

## Cache Invalidation

### Q: Why is cache invalidation hard?
<details><summary>Answer</summary>

Must keep multiple copies (cache, source, multiple cache layers) consistent. Failures can leave cache permanently stale. Ordering matters (cascade of updates). Phil Karlton: "only two hard things in CS: cache invalidation and naming things."

</details>

### Q: What's the thundering herd problem?
<details><summary>Answer</summary>

Multiple requests miss cache simultaneously (e.g., key expires), all query origin at once → origin overloaded. Mitigation: probabilistic early refresh, distributed lock, or async recompute.

</details>

### Q: Design a hybrid invalidation strategy.
<details><summary>Answer</summary>

Explicit invalidation (DELETE on update) + TTL (5-30 min fallback). Most updates trigger explicit delete (immediate freshness). If delete fails, TTL prevents permanent staleness.

</details>

### Q: How does versioning avoid cache invalidation?
<details><summary>Answer</summary>

Instead of mutating `user:123`, create `user:123:v2`. Immutable, never cached before, no invalidation. Trade-off: more cache entries, need garbage collection.

</details>

### Q: What's probabilistic early refresh?
<details><summary>Answer</summary>

With probability P, trigger early cache refresh (e.g., 100s before TTL). By the time TTL actually expires, key is fresh. Prevents thundering herd.

</details>

---

## Advanced

### Q: How do you handle multi-layer cache invalidation?
<details><summary>Answer</summary>

Broadcast updates to all layers (app cache, Redis, CDN) simultaneously. TTL is safety net if one layer doesn't hear about update. Measure end-to-end invalidation latency.

</details>

### Q: In-process cache (Guava) vs Redis: when use each?
<details><summary>Answer</summary>

In-process (Guava): Fast, no network latency, per-JVM. Use for hot local data. Redis: Shared across services, larger capacity, survives restarts. Use for global hot data.

</details>

### Q: What metrics would you monitor for cache health?
<details><summary>Answer</summary>

Hit rate, eviction rate, staleness (time since update), memory usage, latency. Alert on hit rate drop, eviction spike, or staleness increase.

</details>

### Q: How do you prevent cache poisoning?
<details><summary>Answer</summary>

Don't cache error pages (5xx) long. Validate content before caching. Use cache versioning/ETags. Monitor for anomalies (sudden spike in reads to error cached pages).

</details>

---

**Study Tip**: Review these 2-3x/week. Move known ones to "mastered" and focus on weak spots.
