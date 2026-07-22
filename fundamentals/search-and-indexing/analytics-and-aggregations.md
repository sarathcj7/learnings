# Analytics & Aggregations

## TL;DR

- **Bucket aggregations**: Group documents by field value (e.g., GROUP BY category)
- **Metric aggregations**: Calculate min/max/sum/avg/percentile over groups
- **Pre-aggregation**: Pre-compute aggregations at index time (vs query time)
- **ClickHouse**: Column-oriented DB, optimized for analytics (vs Elasticsearch's row-oriented)
- **OLAP vs OLTP**: Analytics (read-heavy, slow) vs Transactional (read/write, fast)

## Aggregations Fundamentals

SQL vs Elasticsearch aggregations (conceptually similar, different names).

```
SQL query:
  SELECT category, COUNT(*), AVG(price)
  FROM products
  GROUP BY category
  HAVING COUNT(*) > 100

Elasticsearch query:
  {
    "aggs": {
      "by_category": {
        "terms": {"field": "category", "size": 10},
        "aggs": {
          "avg_price": {"avg": {"field": "price"}},
          "count": {"value_count": {"field": "_id"}}
        }
      }
    },
    "query": {"match_all": {}}
  }

Result:
  category: "shoes", count: 500, avg_price: $45
  category: "shirts", count: 300, avg_price: $25
  category: "hats", count: 150, avg_price: $15
```

---

## Bucket Aggregations

Grouping documents by field value.

### Terms Aggregation (GROUP BY)

```
Aggregation: "Group by brand"

Index documents:
  doc1: {brand: "nike", price: 80}
  doc2: {brand: "adidas", price: 70}
  doc3: {brand: "nike", price: 90}
  doc4: {brand: "nike", price: 100}
  doc5: {brand: "adidas", price: 60}

Result:
  nike: 3 documents
  adidas: 2 documents

Elasticsearch query:
  {
    "aggs": {
      "brands": {
        "terms": {"field": "brand", "size": 10}
      }
    }
  }
```

### Date Histogram (Time-series Grouping)

```
Aggregation: "Group by day"

Documents:
  doc1: {timestamp: "2026-07-23 10:00", sales: 100}
  doc2: {timestamp: "2026-07-23 15:00", sales: 200}
  doc3: {timestamp: "2026-07-24 09:00", sales: 150}
  doc4: {timestamp: "2026-07-24 16:00", sales: 250}

Result:
  2026-07-23: 2 documents, total_sales: 300
  2026-07-24: 2 documents, total_sales: 400

Query:
  {
    "aggs": {
      "by_day": {
        "date_histogram": {
          "field": "timestamp",
          "calendar_interval": "day"
        }
      }
    }
  }

Use cases: Traffic over time, sales trends, error rates by hour
```

### Range Aggregation

```
Aggregation: "Group by price range"

Query:
  {
    "aggs": {
      "price_ranges": {
        "range": {
          "field": "price",
          "ranges": [
            {"to": 50},
            {"from": 50, "to": 100},
            {"from": 100}
          ]
        }
      }
    }
  }

Result:
  < $50: 1000 documents
  $50-$100: 2000 documents
  > $100: 500 documents
```

---

## Metric Aggregations

Calculate statistics over grouped data.

### Avg / Sum / Min / Max

```
Query: "Average price by brand"

Aggregation:
  {
    "aggs": {
      "by_brand": {
        "terms": {"field": "brand"},
        "aggs": {
          "avg_price": {"avg": {"field": "price"}},
          "min_price": {"min": {"field": "price"}},
          "max_price": {"max": {"field": "price"}}
        }
      }
    }
  }

Result:
  nike:
    avg: $85
    min: $50
    max: $200
  adidas:
    avg: $75
    min: $40
    max: $150
```

### Percentile Aggregation

Useful for latency/response time analysis.

```
Query: "P50, P95, P99 latency by endpoint"

Aggregation:
  {
    "aggs": {
      "by_endpoint": {
        "terms": {"field": "endpoint"},
        "aggs": {
          "latency_percentiles": {
            "percentiles": {
              "field": "latency_ms",
              "percents": [50, 95, 99]
            }
          }
        }
      }
    }
  }

Result:
  /search:
    P50: 10ms
    P95: 50ms
    P99: 200ms
  /checkout:
    P50: 30ms
    P95: 100ms
    P99: 500ms

Interpretation:
  - 50% of searches are < 10ms (median)
  - 95% are < 50ms (most users OK)
  - 1% > 200ms (tail latency, needs optimization)
```

### Cardinality Aggregation

Count unique values (approximately).

```
Query: "Unique users per day"

Aggregation:
  {
    "aggs": {
      "by_day": {
        "date_histogram": {"field": "timestamp", "interval": "day"},
        "aggs": {
          "unique_users": {
            "cardinality": {"field": "user_id"}
          }
        }
      }
    }
  }

Result:
  2026-07-23: 50,000 unique users
  2026-07-24: 52,000 unique users

Implementation: HyperLogLog (probabilistic counting)
  Accuracy: ~2% error
  Memory: Constant O(1) vs O(N) for exact count
```

---

## Pre-aggregation & Bucketing

Pre-compute aggregations at index time instead of query time.

### Pre-aggregated Metrics

```
Without pre-aggregation (query time):
  Query: "Total sales by brand"
  Execution: Scan all 1M documents, sum by brand
  Latency: 500ms

With pre-aggregation (index time):
  Index pipeline:
    1. Document arrives: {brand: "nike", sales: 100}
    2. Pipeline: Extract (brand, sales)
    3. Store pre-aggregated: {brand: "nike", daily_sales: 100}
    4. Periodic aggregate: {brand: "nike", total_sales: 1000}
  
  Query: "Total sales by brand"
  Execution: Lookup pre-computed bucket
  Latency: 10ms

Cost trade-off:
  Storage: Extra ~10-20% for aggregation buckets
  Indexing: Slower (aggregate during index)
  Query: Much faster (trade indexing cost for query speed)
```

### Metrics Aggregation Framework (MAF)

```
Elasticsearch Metric Aggregation Framework:

At index time, compute:
  1. Sum of values
  2. Count of values
  3. Min/max values
  4. Higher moments (variance, skewness for percentiles)

Query time uses pre-computed stats:
  avg = sum / count
  P95 = approximate from moments
  
Benefit: Fast aggregations, not exact percentiles
```

---

## OLAP vs OLTP: Different Tools for Different Jobs

### OLTP (Online Transactional Processing)

Elasticsearch, MySQL, PostgreSQL.

```
Characteristics:
  - Row-oriented storage
  - Fast single-row reads/writes
  - Transactions (ACID)
  - Optimized for: INSERT, UPDATE, DELETE, SELECT by ID
  
Example: E-commerce product database
  Query: "Get product 123" → 1ms (fast)
  Query: "Sum sales by brand" → 1000ms (slow, scans all rows)
```

### OLAP (Online Analytical Processing)

ClickHouse, BigQuery, Redshift.

```
Characteristics:
  - Column-oriented storage (each column separate)
  - Slow single-row access
  - No transactions
  - Optimized for: Aggregations, range scans
  
Example: Analytics on sales data
  Query: "Sum sales by brand" → 100ms (fast, columns scanned in parallel)
  Query: "Get product 123" → 100ms (slow, must reconstruct row)
```

### Storage Comparison

```
Row-oriented (OLTP):
  doc1: {id: 1, brand: "nike", price: 80, category: "shoes"}
  doc2: {id: 2, brand: "adidas", price: 70, category: "shirts"}
  
  Layout in memory:
    [1, nike, 80, shoes][2, adidas, 70, shirts]
  
  Advantages:
    - Fast row access (all columns together)
    - Fast INSERT (write one row)
  
  Disadvantages:
    - Slow aggregations (scan all rows)

Column-oriented (OLAP):
  id column:     [1, 2, 3, ...]
  brand column:  [nike, adidas, nike, ...]
  price column:  [80, 70, 90, ...]
  category:      [shoes, shirts, shoes, ...]
  
  Advantages:
    - Fast aggregations (scan one column)
    - Compression (same type values compress better)
  
  Disadvantages:
    - Slow row access (must reconstruct from columns)
    - Slow INSERT (insert into multiple columns)
```

---

## ClickHouse: Column-Oriented Search & Analytics

Purpose-built for analytics, not general search.

```
Elasticsearch:
  Index structure: Inverted index (term → documents)
  Use case: Full-text search, term-based queries
  Aggregation: Scan inverted index (slower)
  
ClickHouse:
  Index structure: Column indexes (column range, min/max)
  Use case: Numeric analytics, time-series
  Aggregation: Scan columns in parallel (very fast)

Comparison:
Query: "Count events by country where event_type='click'"

Elasticsearch:
  1. Lookup inverted index for event_type='click' → 10M document IDs
  2. For each doc ID, fetch country → 10M lookups
  3. Count by country
  Latency: 1000ms

ClickHouse:
  1. Column event_type = scan, filter (1000ms per column)
  2. Column country = scan filtered rows, count by group (100ms)
  Latency: 100ms (10x faster, parallel column scans)
```

### When to Use Which

| Use Case | Tool |
|---|---|
| **Full-text search** | Elasticsearch, Solr |
| **Autocomplete** | Redis Trie, Elasticsearch |
| **Numeric analytics** | ClickHouse, BigQuery |
| **Transactional** | PostgreSQL, MySQL |
| **Time-series** | ClickHouse, InfluxDB, Prometheus |
| **Hybrid (search + analytics)** | Elasticsearch (search) + ClickHouse (analytics) |

---

## Architecture: Dual-System Setup

Most production systems combine both.

```
User actions:
  Click, search, add-to-cart

Dual write:
  1. → Elasticsearch: For search results (inverted index)
  2. → ClickHouse: For analytics (column store)

Queries:
  "Search running shoes": → Elasticsearch
  "Sessions per hour": → ClickHouse

Complexity: Two systems to operate, synchronization
Benefit: Each system optimized for its workload
```

---

## Production Considerations

1. **Aggregation shard size**: Don't aggregate across too many shards (overhead)
2. **Cardinality estimation**: HyperLogLog accuracy depends on cardinality level
3. **Monitor aggregation latency**: Slow aggregations block queries
4. **Cache aggregation results**: Pre-compute daily/hourly aggregations
5. **Time-based archiving**: Move old data to ClickHouse, keep recent in Elasticsearch

---

## References

- Elasticsearch documentation: "Aggregations"
- ClickHouse documentation: "Query Language"
- "OLAP Cube Technology" — Gray et al.

---

## Related Fundamentals

- [Full-Text Search](full-text-search.md) – Scoring for search (vs aggregations)
- [Query Optimization](query-optimization.md) – Aggregation execution planning
- [Databases](../databases/) – Sharding strategies affect aggregation performance
