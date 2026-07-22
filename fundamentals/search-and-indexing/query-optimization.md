# Query Optimization

## TL;DR

- **Query parsing**: Convert text query into AST (AND, OR, NOT terms)
- **Execution plan**: Determine order of operations (filters before scoring = less work)
- **Filter caching**: Cache term posting lists in memory for common filters
- **Approximate search**: Trade accuracy for speed (e.g., search first 10k results only)
- **Query rewriting**: Simplify query structure to reduce operations

## Query Parsing

Converting user query into searchable form.

```
User query: "nike running shoes under $100"

Parsing:
  1. Tokenize: ["nike", "running", "shoes", "under", "$100"]
  2. Parse: brand=nike AND category=running AND category=shoes AND price<100
  3. AST (Abstract Syntax Tree):
     
     AND
     ├── term: nike
     ├── term: running
     ├── term: shoes
     └── range: price < 100

Execution would blindly search all 4 clauses:
  Find all docs with "nike" (10M docs)
  Find all docs with "running" (5M docs)
  Find all docs with "shoes" (8M docs)
  Find all docs with price < 100 (3M docs)
  Intersect: 10M ∩ 5M ∩ 8M ∩ 3M = ?
```

---

## Execution Planning

Query planner optimizes execution order.

### Filter-First Execution

Execute most selective filters first (reduce candidate set).

```
Query: "nike AND running AND shoes AND price < 100"

Naive execution (left to right):
  1. Find nike: 10M documents
  2. Find running in 10M: 3M documents
  3. Find shoes in 3M: 1M documents
  4. Filter price < 100 in 1M: 50k documents
  
  Comparisons: 10M + 3M + 1M = 14M document checks

Smart execution (filter first):
  1. Find price < 100: 3M documents (most selective)
  2. Find nike in 3M: 1M documents
  3. Find running in 1M: 300k documents
  4. Find shoes in 300k: 50k documents
  
  Comparisons: 3M + 1M + 300k = 4.3M document checks
  
Speedup: 3.2x fewer comparisons
```

### Index for Filters

Use different index structures for different query types.

```
Column A (numeric price):
  Query: price < 100
  Use: B-tree or sorted array (range query efficient)
  
Column B (text search):
  Query: text="running"
  Use: Inverted index (term lookup efficient)
  
Don't use inverted index for range queries:
  Inverted index of "price=50": Not useful for price < 100
  Must scan all price=1, price=2, ..., price=99 entries (slow)
```

---

## Filter Caching

Cache filter results in memory (frequent filters).

### Elasticsearch Filter Cache

```
Query 1: "nike AND price < 100"
  Execute price < 100 filter
  Cache result: bitset [1,1,0,1,1,0,...] (1 = matches, 0 = doesn't match)
  Memory: 1 bit per document (~1GB per 1B documents)
  
Query 2: "adidas AND price < 100"
  Reuse cached "price < 100" result
  Only search for "adidas"
  Much faster (filter already cached)

Cache effectiveness:
  Filter used frequently: Cache pays off
  Filter rarely used: Cache wastes memory
```

### Cache Invalidation

```
When document added/deleted:
  1. Index updated
  2. All filter caches invalidated
  3. Filters re-executed and cached

Typical: Refresh every 1-2 seconds
  Caches valid for 1-2 seconds
  Then invalidated, rebuilt
```

---

## Approximate Search: Trade Speed for Accuracy

### Early Termination

Stop searching after collecting enough results.

```
Elasticsearch default:
  Query all N shards
  Collect top K results from each shard
  Merge, re-rank, return top K globally
  
Time complexity: O(N log K) per shard (N = documents per shard, K = result size)

Early termination (optional):
  Query shards one at a time
  Stop when top result quality plateaus
  Risk: Might miss better results from later shards
  Benefit: 10-100x faster for large result sets
  
Use case: Search engines (first 10 results matter, result 100+ rarely viewed)
```

### Approximate Nearest Neighbor (ANN)

For vector search (embeddings).

```
Exact search (brute force):
  Query vector: [0.5, 0.3, ...]
  Calculate distance to all 1M vectors
  Sort, return top 10
  Time: O(M × D) where M = documents, D = dimensions
  For 1M docs × 768 dims = 768M operations

Approximate search (HNSW - Hierarchical Navigable Small World):
  Build hierarchical graph during indexing
  Start from layer 0 (sparse)
  Navigate greedily to nearest neighbors
  Return top K
  Time: O(log M × K) = ~20 operations
  
Trade-off: Might miss 0.1% of true nearest neighbors, but 10000x faster
Used by: Elasticsearch, Pinecone, Weaviate
```

---

## Query Rewriting

Simplifying queries before execution.

### Combining Adjacent Filters

```
Query: "(price > 50 AND price < 100) AND color = 'red'"

Parser creates:
  AND
  ├── AND
  │   ├── price > 50
  │   └── price < 100
  └── color = 'red'

Rewriter combines:
  AND
  ├── price ∈ (50, 100)  ← Combined range
  └── color = 'red'

Benefit: One range filter instead of two separate filters
```

### Removing Redundant Terms

```
Query: "(brand = nike OR brand = adidas) AND brand = nike"

Parser creates:
  AND
  ├── OR
  │   ├── brand = nike
  │   └── brand = adidas
  └── brand = nike

Rewriter: Intersecting with "brand = nike" means OR becomes meaningless
  Simplified to: brand = nike

Benefit: Fewer operations
```

### NOT to NOT exists

```
Query: "NOT brand = 'unknown'"

Rewriter: NOT brand = 'unknown' is equivalent to brand IS NOT NULL

Benefit: Use index to find not-null values (faster than NOT operation)
```

---

## Caching Query Results

For repeated queries, cache results.

### Query Cache

```
Query 1: "nike running shoes" → Results A (cost: 1000ms)
Query 2: "nike running shoes" → Return cached result A (cost: 1ms)
Query 3: "nike running shoes" → Return cached result A (cost: 1ms)

Cache hit rate: 2/3 = 66%
Average latency: (1000 + 1 + 1) / 3 = 334ms vs 1000ms baseline

Cache invalidation:
  When index refreshes (new documents added)
  When filter cache invalidates
  TTL: Usually 1-5 minutes
```

### Bloom Filters for Cache Misses

Quickly determine if result might exist.

```
Query cache miss (not in memory):
  Check bloom filter (0.1ms)
  
Bloom filter says "definitely not in cache":
  Execute query (1000ms)
  
Bloom filter says "might be in cache":
  Fall through to cache lookup (fast)
  
Benefit: Avoid wasting lookup time on obvious misses
```

---

## Profiling Slow Queries

Elasticsearch provides query profiling.

### Profile Output

```json
{
  "profile": {
    "shards": [
      {
        "id": "shard0",
        "searches": [
          {
            "query": [
              {
                "type": "BooleanQuery",
                "description": "brand:nike price:[* TO 100]",
                "time_in_nanos": 5000000,  ← 5ms
                "breakdown": {
                  "create_weight": 100000,
                  "build_scorer": 500000,
                  "score": 4400000
                },
                "children": [...]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

### Common Bottlenecks

| Issue | Cause | Fix |
|---|---|---|
| **High scorer time** | Calculating BM25 for many docs | Use filter instead of query |
| **High build_scorer time** | Complex boolean queries | Simplify query structure |
| **Slow shard queries** | Unbalanced sharding | Rebalance shards |
| **High merge time** | Many small segments | Force merge before query |

---

## Production Checklist

1. **Enable query cache**: cache_size = 10-15% of heap
2. **Set filter cache**: Use bitset filters for common filters
3. **Monitor slow queries**: Log queries > 100ms
4. **Profile expensive queries**: Use _profile API before optimizing
5. **Batch queries**: Combine multiple queries per request
6. **Index warming**: Pre-load filters by querying before traffic spike
7. **Query timeouts**: Set timeout=1s to prevent runaway queries

---

## References

- Elasticsearch documentation: "Query Profiling"
- Lucene in Action: Query parsing and execution
- "Approximate Nearest Neighbor Algorithms" — Dong et al.

---

## Related Fundamentals

- [Full-Text Search](full-text-search.md) – TF-IDF scoring optimization
- [Distributed Search](distributed-search.md) – Query execution across shards
- [Analytics & Aggregations](analytics-and-aggregations.md) – Aggregation execution planning
