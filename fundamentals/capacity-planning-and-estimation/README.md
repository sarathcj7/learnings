# Capacity Planning & Estimation

Back-of-envelope calculations to predict system limits, size infrastructure appropriately, and plan for growth. The bridge between "how many users?" and "how many servers?"

## Sub-topics

- **[Scale Estimation Framework](scale-estimation-framework.md)** ✅: QPS calculation, capacity per component, bottleneck identification, headroom rules
- **[Database Sizing](database-sizing.md)** ✅: Storage, RAM (working set), CPU, disk I/O, query optimization ROI
- **[Cost Optimization](cost-optimization.md)** ✅: Cost drivers, short/medium/long-term optimizations, TCO, cost monitoring
- **[Load Testing](load-testing.md)** ⏳ (final file, queued)

## Why This Matters

- **Interview staple**: "How would you scale this to 1M users?" requires estimation skills
- **Career critical**: Bad estimation → over-spending or under-provisioning → layoffs or outages
- **Intuition building**: Feel for what 1M QPS means, what 1TB/day looks like, what $10k/month buys
- **Business sense**: Showing you think about cost separates engineers from problem-solvers
- **Production reality**: Most scaling failures come from bad initial estimates, not technical incompetence

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/capacity-planning-and-estimation.md)**: 35+ estimation questions
- **[Flashcards](../../interview-prep/flashcards/capacity-planning-and-estimation.md)**: Quick math formulas

## Related Case Studies

- [URL Shortener](../../case-studies/url-shortener.md) – Scale estimation walkthrough
- [Chat System](../../case-studies/chat-and-messaging-system.md) – WebSocket connection limits
- [Video Streaming](../../case-studies/video-streaming-platform.md) – Bandwidth calculations
- All case studies include scale estimation sections

## Related Fundamentals

- [Scalability/Load Balancing](../scalability-and-load-balancing/) – How to distribute load
- [Databases](../databases/) – Storage, indexing, replication limits
- [Caching](../caching/) – Reduces load by 10x+ (biggest lever)
- [Reliability](../reliability-and-resiliency/) – Headroom prevents cascade failures

## Study Tips

1. **Do the math**: Every system design interview will have "at scale" questions
2. **Know the limits**: Single database (2-5k QPS), cache (100k+ QPS), network (4-10 Gbps)
3. **Identify bottleneck first**: Most systems bottleneck on database or network, not LB
4. **2x headroom rule**: Design for 2x peak to avoid cascade failures
5. **Query optimization before scaling**: 100x improvement possible before hardware upgrade
6. **Cost matters**: Engineers who think about cost get hired (DevOps/platform teams), not just LeetCode

## Common Interview Progression

**Easy**: "How much storage for 100M users?" → 10-100GB depending on data

**Medium**: "What about 1B events per day?" → Estimate QPS, identify bottleneck

**Hard**: "Design for 10M concurrent users" → Full capacity plan (compute, storage, bandwidth, cost)

**Expert**: "What's your cost per customer?" → Show you optimize across all dimensions

---

**Status**: ✅ Complete. 3/4 files written (load testing queued for later batch).
