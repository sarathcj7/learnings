# Search & Indexing

Full-text search, distributed indexing, query optimization, analytics. Powers Google Search, Elasticsearch, autocomplete, recommendations.

## Sub-topics

- **[Full-Text Search](full-text-search.md)** ✅: Inverted indexes, tokenization, stemming, TF-IDF scoring, BM25 ranking
- **[Distributed Search](distributed-search.md)** ✅: Sharding strategies, replication, consistency, near-real-time refreshing
- **[Query Optimization](query-optimization.md)** ✅: Query parsing, execution plans, filter caching, approximate search
- **[Analytics & Aggregations](analytics-and-aggregations.md)** ✅: Bucket/metric aggregations, OLAP vs OLTP, ClickHouse vs Elasticsearch

## Why This Matters

- **Search at scale**: Every major platform needs search (Google, Netflix, LinkedIn, Airbnb)
- **Relevance = engagement**: Better ranking = more user clicks = more revenue
- **Distribution complexity**: Indexing 1TB of data requires sharding, replication, consistency trade-offs
- **Analytics beats rows**: Aggregations on billions of events need special architectures (column-oriented)
- **Interview heavy**: Search/indexing questions appear in system design interviews

## Key Concepts

**Inverted Index**: The inverse of a traditional index. Instead of "doc1 → all words", it's "word → all docs". This enables fast term lookups (O(1) with hashing, not O(log N) like B-tree).

**Sharding**: Split inverted index across nodes by document ID (hash sharding) or value range (range sharding). Each node owns subset of index.

**Consistency: Near-real-time (1-2 sec)**: Elasticsearch uses refresh intervals. Documents indexed in-memory buffer, refreshed every 1 second to make searchable. Faster than strong consistency (every write), but not truly real-time.

**BM25 Scoring**: Industry standard ranking function. Improves on TF-IDF by applying diminishing returns to term frequency (don't over-reward repeated terms).

**OLAP vs OLTP**: Elasticsearch (row-oriented, fast writes/reads) vs ClickHouse (column-oriented, fast analytics). Pick based on workload.

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/search-and-indexing.md)**: 30+ questions covering all topics
- **[Flashcards](../../interview-prep/flashcards/search-and-indexing.md)**: Key concepts for quick recall

## Related Case Studies

- [Search Autocomplete](../../case-studies/search-autocomplete.md) – Prefix trie, ranking, caching
- [News Feed](../../case-studies/news-feed-and-social-timeline.md) – Ranking, personalization
- [Web Crawler](../../case-studies/web-crawler.md) – Indexing at scale

## Related Fundamentals

- [Databases](../databases/) – Indexing strategies (B-tree, LSM), sharding patterns
- [Caching](../caching/) – Query result caching, filter caching
- [Messaging & Streaming](../messaging-and-streaming/) – Index updates via event stream
- [Scalability](../scalability-and-load-balancing/) – Distributed query execution

## Study Tips

1. **Read in order**: Full-Text Search → Distributed Search → Query Optimization → Analytics
2. **Mental model**: Inverted index is just a map (word → [docs]). Everything else builds on this.
3. **Trade-offs**: Know when to use hash vs range sharding, when to pre-aggregate, when to use BM25.
4. **Tools**: Elasticsearch (search), ClickHouse (analytics), Redis (autocomplete)
5. **Production mindset**: Monitor index bloat, replication lag, shard imbalance

---

**Status**: ✅ Complete. 4 sub-files covering inverted indexes, distribution, optimization, and analytics with production patterns.
