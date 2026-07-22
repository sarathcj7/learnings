# Eviction Policies

## TL;DR

- **LRU**: Remove least recently used. Simple, effective for temporal locality. Default in most caches.
- **LFU**: Remove least frequently used. Better than LRU for temporal patterns, but more complex.
- **TTL**: Remove by time (absolute or sliding expiration). Ensures freshness, doesn't require eviction algorithm.
- **FIFO**: Remove oldest first. Simple but naive.
- **ARC** (Adaptive Replacement Cache): Adapts between LRU and LFU based on workload. State-of-art but complex.

## LRU (Least Recently Used)

Most common eviction policy. Remove the entry not accessed for the longest time.

### How It Works

Maintain a doubly-linked list ordered by access time. On access, move entry to head (most recent). On eviction, remove tail (least recent).

### Implementation

```
LRU Cache (capacity=3):

Access pattern: A, B, C, A, D

Step 1: A accessed
  [A]

Step 2: B accessed
  [B, A]

Step 3: C accessed
  [C, B, A]

Step 4: A accessed (move to head)
  [A, C, B]

Step 5: D accessed (capacity full, evict LRU which is B)
  [D, A, C]
```

### Pros

- **Temporal locality**: Assumes recently accessed data is likely accessed again soon (true for 80% of workloads).
- **Simple**: O(1) access, O(1) eviction with doubly-linked list + hash table.
- **Effective**: 80-90% hit rates in practice for typical workloads.

### Cons

- **Scan attacks**: A full-scan query (accessing all data sequentially) can evict the entire hot set. Example: monthly analytics scan through 100GB logs evicts the 1GB hot cache.
  - **Mitigation**: Probabilistic insertion (insert with P=0.1 instead of always), or multi-level caching.
- **Recency bias**: A once-hot key accessed again late doesn't get re-promoted enough.

### Real-World Example

Redis uses **LRU approximation**: Instead of maintaining exact order (expensive), it randomly samples N entries on eviction and removes the LRU among them (default N=5). 85% accuracy, much faster.

---

## LFU (Least Frequently Used)

Remove the entry accessed least often. Better than LRU for repeated bursts.

### How It Works

Track frequency count for each entry. On eviction, remove entry with lowest frequency. Ties broken by LRU (least recently used among same frequency).

### Example

```
Access pattern: A, B, C, A, A, D

After accesses:
  A: frequency=3, lastAccess=t5
  B: frequency=1, lastAccess=t2
  C: frequency=1, lastAccess=t3
  D: frequency=1, lastAccess=t6

On eviction (capacity full), remove B (frequency=1, oldest).
```

### Pros

- **Frequency locality**: Captures popular keys (even if not recently accessed).
- **Prevents scan attacks**: A full-scan operation has frequency=1, so it's first to evict.
- **Better for skewed distributions**: If 10% of keys are 90% of traffic, LFU protects those hot keys.

### Cons

- **O(log N) eviction**: Must maintain a priority queue of frequencies, slower than LRU.
- **Cold start**: New keys start at frequency=1, so they're vulnerable to eviction immediately.
- **Frequency inflation**: Long-running systems have frequency counts → 1M for hot keys, complexity.

### Variant: TinyLFU

Combines LFU benefits with efficiency: Maintain a Bloom filter of recent accesses. On insertion, check filter to estimate frequency. Cheaper than full LFU.

---

## TTL (Time-To-Live)

Entries expire after a specified duration. No active eviction policy; expired entries are lazily removed on access.

### How It Works

```
SET key value EX 3600  # Expire in 3600 seconds

Clock: t=0     → SET created, expiration set to t=3600
Clock: t=1800  → GET returns value (not expired)
Clock: t=3601  → GET finds entry expired, returns nil, removes entry
```

### Pros

- **Simplicity**: No complex eviction logic, just a timer.
- **Freshness guarantee**: Data never served older than TTL.
- **Operational simplicity**: Easy to reason about staleness (at most X seconds old).

### Cons

- **Cold data persists**: Unpopular keys stay in cache until TTL expires, wasting memory.
- **Thundering herd on expiration**: If 1000 keys expire simultaneously at t=3600, all clients miss at once.
  - **Mitigation**: Add randomness to TTL (±10%), or use sliding windows (refresh TTL on access).
- **Staleness**: Data can be exactly TTL old before refresh.

### Hybrid: Sliding Window TTL

On access, extend TTL by another window. Keeps hot data fresh, cold data expires.

```
SET key value EX 3600 KEEPTTL
# On every access, TTL restarts

Effect: Hot keys stay forever (kept alive by traffic), cold keys expire after 3600s of inactivity.
```

---

## FIFO (First-In-First-Out)

Remove oldest entry regardless of access pattern. Rarely used in practice.

### Pros

- **O(1) eviction**: Simple queue, remove from front.
- **Predictable**: Entries last exactly as long as insertion order suggests.

### Cons

- **Poor locality**: Doesn't account for access patterns. An old key accessed 10k times is evicted before a new key accessed once.
- **Scan attacks**: Same as LRU.

### When Used

- Bounded queues (buffer size limit).
- CDC (Change Data Capture) pipelines.
- Rarely in caches.

---

## ARC (Adaptive Replacement Cache)

State-of-the-art but complex. Dynamically adapts between LRU and LFU based on workload.

### How It Works

Maintain two lists: **T1** (recently used once) and **T2** (recently used multiple times). Adaptively shift cache size between T1 and T2 based on hit rates. If T2 hits are high, grow T2. If T1 hits are high, grow T1.

### Pros

- **Adapts to workload**: Self-tunes between recency and frequency.
- **Near-optimal**: Theoretically optimal within 3x of optimal offline algorithm.

### Cons

- **Complexity**: Hard to implement correctly, non-trivial to debug.
- **Patents**: Originally patented by IBM, licensing issues.

### Real-World

Used in PostgreSQL, Enterprise Storage systems. Rarely in web-scale caches (overhead not worth it vs simpler LRU approximation).

---

## Comparison Table

| Policy | Hit Rate | Speed | Memory | Best For | Worst For |
|---|---|---|---|---|---|
| **FIFO** | 60% | O(1) | Low | None (rarely used) | Temporal patterns |
| **LRU** | 80% | O(1) | Low | General purpose | Scan attacks, frequency skew |
| **LFU** | 85% | O(log N) | Medium | Skewed distribution, scan resistance | Recency bias, cold start |
| **TTL** | 70-80% | O(1) | Low | Freshness guarantee | Cold data, expiration storms |
| **ARC** | 85-90% | O(log N) | Medium | Optimal (when tuned) | Complexity, licensing |

---

## Production Considerations

1. **Measurement**: Track **hit rate** (hits / (hits + misses)). 80%+ is excellent. Below 50% means cache too small or TTL too short.
2. **Policy choice**: Default to LRU with approximation (like Redis) unless you have specific frequency-based workloads (e.g., popular product recommendations).
3. **Hybrid**: Many systems use TTL + LRU together: TTL ensures freshness upper bound, LRU evicts cold data.
4. **Size**: Cache size should be ~1-5% of working set size. If working set is 100GB and you need 80% hit rate, allocate 5-10GB cache.
5. **Warmup**: On restart, cache is cold. Use pre-warming (load hot keys on startup) to avoid initial slow-down.

---

## References

- "ARC: A Self-Tuning Low Overhead Replacement Cache" — Megiddo & Modha (IBM Research)
- Redis documentation: "Eviction Policies"
- Memcached documentation: "LRU Eviction"

---

## Related Fundamentals

- [Caching Strategies](strategies.md) – Which strategy pairs with which eviction policy?
- [Distributed Caching](distributed-caching.md) – How do distributed caches handle eviction?
- [Reliability & Resiliency](../reliability-and-resiliency/) – Graceful degradation when cache fails
