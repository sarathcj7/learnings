# Databases

Databases are the foundation of system design. This topic covers relational fundamentals, indexing strategies, replication patterns, sharding at scale, ACID guarantees, and the NoSQL/NewSQL landscape.

## Sub-topics

- **[Relational Fundamentals](relational-fundamentals.md)** ✅: Entities, ACID, normalization, denormalization, indexes
- **[Indexing](indexing.md)** ✅: B-tree, hash, LSM, composite indexes, when to index
- **[Replication](replication.md)** ✅: Leader-follower, multi-leader, leaderless (Dynamo-style), failover
- **[Sharding & Partitioning](sharding-and-partitioning.md)** ✅: Hash, range, consistent hashing, hot spots, resharding
- **[Transactions & Isolation Levels](transactions-and-isolation-levels.md)** ✅: ACID, isolation levels, anomalies, MVCC
- **[NoSQL & NewSQL Landscape](nosql-and-newsql-landscape.md)** ✅: Key-value, document, wide-column, graph, Spanner, CockroachDB

## Why This Matters

- **Core to every system**: Every system needs persistent data storage
- **Scale foundation**: Sharding, replication, indexing are how you scale to billions of rows
- **Consistency guarantees**: ACID/BASE trade-off is critical architectural decision
- **Interview heavy**: Databases questions appear in every system design interview

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/databases.md)**: 40+ questions covering all topics
- **[Flashcards](../../interview-prep/flashcards/databases.md)**: Key concepts for quick recall

## Related Case Studies

- [URL Shortener](../../case-studies/url-shortener.md) – Sharding by user ID, read replicas
- [Ride-Sharing Service](../../case-studies/ride-sharing-service.md) – Geographic sharding, consistency requirements
- [News Feed](../../case-studies/news-feed-and-social-timeline.md) – Denormalization for speed, write optimization

## Related Fundamentals

- [Caching](../caching/) – Databases + cache layer = fast scale
- [Distributed Transactions](../distributed-transactions/) – ACID across shards
- [Consensus & Coordination](../consensus-and-coordination/) – Leader election for failover
- [Reliability & Resiliency](../reliability-and-resiliency/) – Replication strategy + failover

---

## Study Tips

1. **Read in order**: Fundamentals → Indexing → Replication → Sharding → Transactions → NoSQL
2. **Mental model**: Internalize trade-offs (consistency vs latency, redundancy vs storage)
3. **Design patterns**: Know when to normalize, when to denormalize, when to shard
4. **Production mindset**: Think about monitoring, backups, failover for each pattern

---

**Status**: ✅ Complete. 6 sub-files with production patterns and trade-off analysis.
