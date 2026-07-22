# Rate Limiting & Traffic Management

Protecting systems from overload and abuse. From token bucket and sliding window algorithms to multi-level strategies, adaptive limits, and load shedding — the fundamentals of controlling traffic flow.

## Sub-topics

- **[Token Bucket](token-bucket.md)**: The most popular algorithm. Simple, handles bursts well, low memory overhead. Tokens refill at fixed rate; request costs 1 token.
- **[Sliding Window Counter](sliding-window.md)**: Precise per-window rate limiting. Tracks request timestamps; more accurate than token bucket but higher memory overhead.
- **[Rate Limiting Strategies](rate-limiting-strategies.md)**: Beyond the algorithm — per-user limits, per-endpoint costs, per-IP limits, global caps, adaptive limiting, backpressure, load shedding.

## Why This Matters

- **System protection**: Rate limiting is the first line of defense against overload and DDoS.
- **Fairness**: Prevent one user/IP from hogging resources.
- **Monetization**: Quota-based pricing (free tier: 100 req/day, paid: unlimited).
- **Resilience**: Graceful degradation under load (adaptive limits, load shedding).
- **User experience**: Backpressure signals let clients back off intelligently (exponential retry).

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/rate-limiting-and-traffic-management.md)**: Self-test recall on algorithms, strategies, and trade-offs
- **[Flashcards](../../interview-prep/flashcards/rate-limiting-and-traffic-management.md)**: Spaced-repetition drills

## Related Fundamentals

- [Scalability & Load Balancing](../scalability-and-load-balancing/): Coordinating rate limits across load balancers
- [Reliability & Resiliency](../reliability-and-resiliency/): Rate limiting as resilience mechanism
- [Security](../security/): DDoS mitigation, abuse prevention
- [Networking & Protocols](../networking-and-protocols/): TCP congestion control (AIMD), similar ideas at transport layer

## Related Case Studies

- [URL Shortener](../../case-studies/url-shortener.md) – Handling traffic spikes
- [Distributed Cache](../../case-studies/distributed-cache.md) – Coordinating limits across caches
- [Messaging Queue](../../case-studies/messaging-queue.md) – Per-consumer rate limiting

---

**Study Tips**

1. **Start with algorithms**: Understand token bucket (simple, practical) before sliding window (precise, expensive).
2. **Apply strategies**: Layer per-user, per-endpoint, per-IP limits like a pyramid.
3. **Think about trade-offs**: Algorithm choice (token bucket vs. sliding window), level choice (per-user vs. global), and resilience (adaptive vs. static).
4. **Design practice**: For each case study (GitHub, AWS), identify which rate limits they likely use and why.
5. **Backpressure intuition**: A rejected request is feedback, not failure. Good client code exponentially backs off.

---

**Status**: ✅ Complete
