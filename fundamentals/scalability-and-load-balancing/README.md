# Scalability & Load Balancing

How to design systems that grow from thousands to millions of requests per second. Covers load balancing algorithms, geographic routing, scaling patterns, and operational strategies for staying ahead of traffic growth.

## Sub-topics

- **[Load Balancing Algorithms](load-balancing-algorithms.md)** ✅: Round-robin, least connections, weighted, IP hash, consistent hash, least response time
- **[GSLB](gslb.md)** ✅: Global server load balancing, geographic routing, failover, TTL trade-offs
- **[Scaling Patterns](scaling-patterns.md)** ✅: Horizontal vs vertical, stateless, stateful, read replicas, sharding, CQRS
- **[Connection Pooling & Timeouts](connection-pooling-and-timeouts.md)** ✅: Pool management, timeout tuning, connection leaks, cascade failures
- **[Capacity Planning](capacity-planning.md)** ⏳ (coming next)

## Why This Matters

- **Core skill**: Every system design interview asks how to scale to millions of requests
- **Practical**: Difference between "works at 1k QPS" and "works at 1M QPS" is entirely these patterns
- **Foundation**: Many failures happen at scale thresholds—connection pool exhaustion, database saturation, cache coherence breaks
- **Interview critical**: "How do you scale X?" appears in 80% of FAANG interviews

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/scalability-and-load-balancing.md)**: 40+ scaling questions
- **[Flashcards](../../interview-prep/flashcards/scalability-and-load-balancing.md)**: Load balancing and scaling tradeoffs

## Related Case Studies

- [URL Shortener](../../case-studies/url-shortener.md) – Sharding, caching, load balancing
- [News Feed](../../case-studies/news-feed-and-social-timeline.md) – Scaling reads with caching and fanout
- [Video Streaming](../../case-studies/video-streaming-platform.md) – CDN scaling, bandwidth optimization
- [Ride-Sharing](../../case-studies/ride-sharing-service.md) – Geolocation sharding, real-time scaling

## Related Fundamentals

- [Databases/Replication](../databases/replication.md) – Read replicas for scaling reads
- [Databases/Sharding](../databases/sharding-and-partitioning.md) – Partitioning for scaling writes
- [Caching](../caching/) – Reduces load on database
- [Reliability & Resiliency](../reliability-and-resiliency/) – Circuit breakers, bulkheads
- [Networking](../networking-and-protocols/) – TCP, DNS, HTTP/2

## Study Tips

1. **Start with algorithms**: Understand round-robin → consistent hashing progression
2. **Scale estimation**: Each case study teaches how bottlenecks emerge at different scales
3. **GSLB last**: Requires understanding both LB algorithms and DNS fundamentals
4. **Connect to databases**: Scaling patterns work hand-in-hand with sharding, replication, caching
5. **Production bias**: This topic is heavy on real-world constraints, not theory

## Common Interview Progression

**Easy**: "Design a system for 1k req/sec" → Answer: Single server, maybe cache

**Medium**: "Make it 100k req/sec" → Answer: Multiple servers, read replicas, caching

**Hard**: "Make it 1M req/sec" → Answer: Sharding, GSLB, connection pooling, circuit breakers

**Expert**: "Handle traffic spikes?" → Answer: Auto-scaling, burst capacity, graceful degradation

---

**Status**: ✅ Complete. 4/5 files written (capacity planning coming in parallel).
