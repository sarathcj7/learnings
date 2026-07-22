# Caching — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## Caching Strategies

1. Describe the cache-aside pattern. When would you use it?
2. What's the difference between cache-aside and read-through? Which is simpler?
3. Explain write-through caching. What's the downside?
4. What's write-behind caching and why is it risky?
5. You're designing a system with high write throughput. Which caching strategy would you pick and why?
6. Your social media feed needs strong consistency for user updates. Which strategy would you use?
7. What's the "thundering herd" problem in cache-aside? How would you mitigate it?
8. Explain the difference between cache-aside for reads vs writes.

---

## Eviction Policies

9. What's LRU caching? How would you implement it efficiently?
10. When is LRU worse than other eviction policies? Give an example.
11. What's LFU caching? How is it different from LRU?
12. Describe a "scan attack" on LRU cache. How would you defend against it?
13. What's TTL-based eviction? What are the pros and cons vs LRU?
14. If you're caching user sessions, would you use LRU or TTL? Why?
15. You have a cache with 1000 entries. 90% of traffic hits 10 of them. Which eviction policy would you use?
16. Explain the "cold start" problem with LFU. How does TTL-based eviction avoid it?

---

## Distributed Caching

17. You have a single Redis instance handling all caches. It's becoming a bottleneck. How would you scale it?
18. What's consistent hashing and why is it used in distributed caches?
19. Explain what happens when a cache node fails in a Redis cluster.
20. If a node joins a Redis cluster, what percentage of keys need to rehash? (Assuming consistent hashing with virtual nodes)
21. What's a "hot spot" in a distributed cache? How would you handle it?
22. Would you use replicated caching (full copy on each node) or partitioned caching (shards)? Explain the trade-off.
23. How would you handle read-after-write consistency in a distributed cache?
24. Describe how Redis replicates data from primary to replicas.

---

## CDN Caching

25. You're designing an image delivery system. How would you use a CDN?
26. What does the `Cache-Control: max-age=3600` header do?
27. What's the difference between `Cache-Control: no-cache` and `no-store`?
28. Explain ETags and how they reduce bandwidth.
29. Your website has 80% cache hit ratio at the CDN. Is that good or bad? What would you do?
30. How does a CDN handle HTTPS? (What is TLS termination?)
31. A user updates their profile picture. How do you ensure the CDN serves the new version quickly?
32. What's the difference between cache invalidation by TTL vs by purge request?

---

## Cache Invalidation

33. Why is cache invalidation considered "one of the hardest problems in CS"?
34. You have a comment that gets deleted. How do you invalidate it across a CDN, Redis, and application cache?
35. Your cache TTL is 1 hour. A user updates their profile. Users see old data for up to 1 hour. How would you fix this?
36. Explain the "thundering herd" problem when cache entries expire simultaneously.
37. What's probabilistic early refresh? How does it help?
38. You're using explicit cache invalidation (DELETE on updates). What's the failure mode if the DELETE message is lost?
39. How would you version cache keys to avoid invalidation logic entirely?
40. Design a hybrid invalidation strategy combining TTL and explicit invalidation.

---

## Synthesis & Cross-Cutting

41. You're designing Instagram's feed cache. Users follow thousands of accounts, and the feed updates constantly. Which caching strategy and eviction policy would you use? How would you handle invalidation?
42. Design a distributed cache for a real-time leaderboard (top 100 users by score, updated constantly).
43. Your API has 10k QPS. You add a cache layer and hit ratio becomes 90%. How much load does the origin database see now?
44. Cache miss at a CDN PoP causes it to fetch from origin. Origin is 200ms away. But your SLA is 100ms P99. How do you meet it?
45. You have 3 layers of cache: application (in-process), Redis, CDN. A data item updates. Design a failsafe invalidation strategy.

---

## Advanced & Tricky Questions

46. What are the failure scenarios when you have multiple replicas of a cache? Can both be stale?
47. How would you detect and prevent cache poisoning (serving stale/malicious content)?
48. In a write-behind cache, how do you ensure data doesn't get lost if the cache crashes before writing to the source?
49. If you use consistent hashing with virtual nodes, why do you need virtual nodes? What happens without them?
50. Design a caching layer that survives the failure of an entire AWS region.
51. You have a key with 10k queries/sec. The cache entry expires, and all 10k concurrent requests miss. What happens? How do you prevent it?
52. Compare in-process caching (Guava, caffeine) vs distributed caching (Redis). When would you choose each?
53. How would you measure cache effectiveness (hit rate, eviction rate, staleness)? What metrics would you alert on?

---

## Answers & Guidance

These are prompts only. Refer to the fundamentals sub-files for detailed answers:

- **Strategies**: See [strategies.md](../../fundamentals/caching/strategies.md)
- **Eviction Policies**: See [eviction-policies.md](../../fundamentals/caching/eviction-policies.md)
- **Distributed Caching**: See [distributed-caching.md](../../fundamentals/caching/distributed-caching.md)
- **CDN**: See [cdn.md](../../fundamentals/caching/cdn.md)
- **Invalidation**: See [invalidation.md](../../fundamentals/caching/invalidation.md)

---

## Self-Scoring Guide

- **0-20 answered well**: Not ready for interviews on caching. Study fundamentals first.
- **21-40 answered well**: You know the basics. Study weak areas.
- **41-53 answered well**: You're interview-ready on caching.
