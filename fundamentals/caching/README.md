# Caching

Caching is the most impactful optimization in system design. It's the difference between serving requests in milliseconds vs seconds. This topic covers caching strategies, eviction policies, distributed caching, CDNs, and invalidation — everything you need to design high-performance systems.

## Sub-topics

- **[Strategies](strategies.md)**: Cache-aside, read-through, write-through, write-behind — when each applies
- **[Eviction Policies](eviction-policies.md)**: LRU, LFU, ARC, FIFO, TTL — memory management under load
- **[CDN Caching](cdn.md)**: From the app's perspective: cache headers, hit ratios, edge serving
- **[Distributed Caching](distributed-caching.md)**: Consistent hashing, Redis/Memcached, replicated vs partitioned
- **[Cache Invalidation](invalidation.md)**: TTL vs explicit invalidation, thundering herd, multi-tier coherence

## Why This Matters

- **Performance**: Caching reduces latency by 100–1000x (disk I/O → memory I/O).
- **Scale**: Caching is often the first step to scale a monolith to millions of QPS.
- **Cost**: A smart cache layer can eliminate 90% of database queries, saving infra costs.
- **Tradeoffs**: Caching introduces eventual consistency, staleness, and invalidation complexity.

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/caching.md)**: Self-test recall on caching concepts
- **[Flashcards](../../interview-prep/flashcards/caching.md)**: Spaced-repetition drills

## Related Fundamentals

- [Databases](../databases/): Indexing and query optimization are complementary to caching
- [Distributed Data Structures](../distributed-data-structures/): Consistent hashing powers distributed caches
- [Scalability & Load Balancing](../scalability-and-load-balancing/): Caching is the scaling enabler
- [Reliability & Resiliency](../reliability-and-resiliency/): Cache failures and graceful degradation

## Related Case Studies

- [URL Shortener](../../case-studies/url-shortener.md) – Redis caching for hot URLs
- [Distributed Cache](../../case-studies/distributed-cache.md) – Building a cache from scratch
- [News Feed](../../case-studies/news-feed-and-social-timeline.md) – Caching personalized feeds
- [Search Autocomplete](../../case-studies/search-autocomplete.md) – Prefix-based caching

---

**Study Tips**

1. **Read in order**: Strategies → Eviction → Distributed → Invalidation. Each builds on the last.
2. **Mentally apply**: As you read each sub-file, think of a system you use (Instagram, Twitter, YouTube) and how it likely uses caching.
3. **Test yourself**: Answer the question bank without looking at the sub-files. Then check.
4. **Design practice**: Open a case study (like URL Shortener or News Feed) and design the cache layer before reading the solution.

---

**Status**: ⭕ Not started · 📖 Reviewed · 💪 Practiced · 🎯 Mastered
