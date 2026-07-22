# Full-Text Search

## TL;DR

- **Inverted Index**: Maps words → documents (reverse of document → words). Core of search engines.
- **Tokenization**: Break text into tokens (words). "running shoes" → ["running", "shoes"]
- **Stemming/Lemmatization**: Reduce words to root form. "running", "runs", "ran" → "run"
- **TF-IDF Scoring**: Relevance = Term Frequency × Inverse Document Frequency
- **Analyzer**: Tokenizer + filters (lowercase, remove stopwords, stem). Elasticsearch uses analyzers.

## Inverted Index

The foundation of full-text search. Maps each word to all documents containing it.

```
Normal (forward) index:
  doc1: "running shoes are fast"
  doc2: "running is fun"
  doc3: "fast cars run"

Inverted index:
  "running" → [doc1, doc2]
  "shoes" → [doc1]
  "are" → [doc1]
  "fast" → [doc1, doc3]
  "is" → [doc2]
  "fun" → [doc2]
  "cars" → [doc3]
  "run" → [doc3]

Search "running shoes":
  1. Look up "running" → [doc1, doc2]
  2. Look up "shoes" → [doc1]
  3. Intersection → [doc1]
  4. Return doc1 (both words present)
```

### Index Structure

Real search engines store more info per term:

```
Inverted index entry:
  Term: "running"
  Posting list: [
    {docID: 1, positions: [0], freq: 1},
    {docID: 2, positions: [0], freq: 1},
  ]
  
  docID: Document ID
  positions: Word positions in document (for phrase queries)
  freq: Term frequency (how many times appears in doc)
```

**Benefit**: Search for "running" is O(P) where P = posting list size, not O(D) where D = total documents.

---

## Tokenization

Breaking text into searchable tokens.

```
Raw text: "I'm running quickly!"

Simple tokenization (split on space/punctuation):
  → ["I'm", "running", "quickly"]

Better tokenization (handle contractions):
  → ["I", "am", "running", "quickly"]

Problem: "I'm" could mean "I am" or "I am" (ambiguous)
Solution: Use language-aware tokenizer (handles English rules)
```

### Tokenizer Types

| Type | Example Input | Output | Use Case |
|---|---|---|---|
| **Whitespace** | "hello world" | ["hello", "world"] | Simple, fast, but naive |
| **Standard** | "hello, world!" | ["hello", "world"] | Default, handles punctuation |
| **Keyword** | "hello world" | ["hello world"] | No tokenization (treat as single token) |
| **N-gram** | "hello" | ["he", "el", "ll", "lo"] | Typo-tolerant search |

---

## Stemming & Lemmatization

Reduce words to root form for better matching.

```
Search: "running shoes"

Without stemming:
  Doc1: "running shoes" → Match ✓
  Doc2: "run shoes" → No match (different word form) ✗
  Doc3: "runs better shoes" → No match ✗

With stemming (convert to root):
  "running" → "run"
  "run" → "run"
  "runs" → "run"
  
  Doc1: "run shoes" → Match ✓
  Doc2: "run shoes" → Match ✓
  Doc3: "run better shoes" → Match ✓
```

### Stemming vs Lemmatization

```
Word: "better"

Stemming (rule-based, aggressive):
  "better" → "bet" (remove suffix "er")
  Problem: Over-stemming loses meaning

Lemmatization (dictionary-based, accurate):
  "better" → "good" (actual root)
  Slower but more accurate
  
Typical choice: Stemming (faster, good enough)
```

---

## Analyzers: Combining Tokenization & Filtering

An analyzer = tokenizer + token filters (lowercase, remove stopwords, stem).

### Elasticsearch Analyzer Example

```
Analyzer: "text" analyzer (default for full-text search)
  
Stages:
  1. Tokenizer: Standard (split on punctuation)
  2. Filter 1: Lowercase (convert to lowercase)
  3. Filter 2: Stop (remove common words)
  4. Filter 3: Snowball stemmer (stem words)

Input: "The quick brown foxes jumping"

After tokenizer: ["The", "quick", "brown", "foxes", "jumping"]
After lowercase: ["the", "quick", "brown", "foxes", "jumping"]
After stopword filter: ["quick", "brown", "foxes", "jumping"]
                       (removed "the")
After stemmer: ["quick", "brown", "fox", "jump"]
               (stemmed "foxes" → "fox", "jumping" → "jump")

Indexed tokens: ["quick", "brown", "fox", "jump"]
```

### Custom Analyzer

```json
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filters": [
            "lowercase",
            "stop",
            "snowball"
          ]
        }
      }
    }
  }
}
```

---

## TF-IDF Scoring: Relevance Ranking

Relevance = how well a document matches the query.

### Term Frequency (TF)

How often term appears in document. Normalized to 0-1.

```
Document: "running shoes for running athletes are running"

TF("running") = 3 / 8 = 0.375  (appears 3 times, 8 words total)
TF("shoes") = 1 / 8 = 0.125
TF("athletes") = 1 / 8 = 0.125

High TF: Important term for this document
```

### Inverse Document Frequency (IDF)

How rare is the term? Rare terms are more informative.

```
Index: 1,000,000 documents

"running" appears in 50,000 docs → IDF = log(1M / 50k) ≈ 2.3  (common)
"shoes" appears in 100,000 docs → IDF = log(1M / 100k) ≈ 2.0  (common)
"orthopedic" appears in 1,000 docs → IDF = log(1M / 1k) ≈ 6.9  (rare, specific)

High IDF: Specific, discriminating term
```

### TF-IDF Score

```
Query: "orthopedic running shoes"

Document A: "orthopedic running shoes review"
  TF("orthopedic") = 1/4 = 0.25,  IDF = 6.9  → Score = 1.725
  TF("running") = 1/4 = 0.25,     IDF = 2.3  → Score = 0.575
  TF("shoes") = 1/4 = 0.25,       IDF = 2.0  → Score = 0.5
  Total Score: 2.8

Document B: "I run in running shoes, running is fun"
  TF("orthopedic") = 0,           IDF = 6.9  → Score = 0
  TF("running") = 2/8 = 0.25,     IDF = 2.3  → Score = 0.575
  TF("shoes") = 1/8 = 0.125,      IDF = 2.0  → Score = 0.25
  Total Score: 0.825

Rank: Doc A (2.8) > Doc B (0.825)
```

**Intuition**: Document A uses specific term ("orthopedic") → higher relevance.

---

## BM25: Industry Standard Ranking

TF-IDF is simple but naive. BM25 (Best Matching 25) improves it:

```
TF-IDF issue: More term frequencies = higher score (unbounded)
  "running shoes running shoes" = higher score than "running shoes"
  Problem: Query stuffing, redundancy penalized poorly

BM25 solution: Diminishing returns on term frequency

BM25 Score = Σ IDF(term) × (TF(term) × (k1 + 1)) / (TF(term) + k1 × (1 - b + b × (docLen / avgLen)))

Parameters:
  k1 = 1.2 (controls term frequency saturation)
  b = 0.75 (controls doc length normalization)
  
With k1=1.2:
  TF=1: Score = IDF × (1.2 × 2) / (1 + 1.2) = IDF × 1.09
  TF=2: Score = IDF × (1.2 × 3) / (2 + 1.2) = IDF × 1.34
  TF=3: Score = IDF × (1.2 × 4) / (3 + 1.2) = IDF × 1.45
  TF=10: Score = IDF × (1.2 × 11) / (10 + 1.2) = IDF × 1.65
  
  Diminishing returns: +1 TF at start is bigger boost than +1 TF at end
```

Most search engines (Elasticsearch, Solr) use BM25 as default.

---

## Trade-offs & Production Considerations

### Index Size vs Search Speed

```
Minimal index:
  Store only document IDs per term
  Index size: Small
  Search: Slow (still need to process large posting lists)

Full index:
  Store term freq, positions, field info per term
  Index size: Large (3-5x original document size)
  Search: Fast (use positions for phrase queries, tf for ranking)
  
Typical: Full index (space worth the speed)
```

### Indexing Latency vs Freshness

```
Real-time indexing:
  Update inverted index immediately on write
  Freshness: Instant (1ms latency)
  Cost: Write latency high, index modification overhead
  
Batch indexing (hourly):
  Batch writes, index every hour
  Freshness: 1 hour delay
  Cost: Writes fast, index stable
  
Typical choice: Near-real-time (1-2 sec batches, Elasticsearch)
```

### Relevance vs Performance

```
Simple ranking (document frequency only):
  TF only: Fast, reasonable relevance
  
Advanced ranking (full BM25):
  BM25 + boosting + custom scoring: Better relevance, slower
  
Production: BM25 default, optimize if relevance poor
```

---

## Production Checklist

1. **Choose analyzer**: Match language (English vs Chinese tokenization differs)
2. **Set up reranking**: Elasticsearch BM25 is good start, but A/B test custom models
3. **Monitor query latency**: Inverted index lookups should be <100ms
4. **Tune index refresh**: Set refresh_interval to balance freshness vs write cost
5. **Index storage**: Monitor index bloat, segment merge settings

---

## References

- "Lucene in Action" — Hatcher, Gospodnetic, McCandless
- Elasticsearch documentation: "Analyzers"
- Okapi BM25 paper: "Probabilistic Relevance Framework"

---

## Related Fundamentals

- [Distributed Search](distributed-search.md) – Sharding inverted indexes across nodes
- [Query Optimization](query-optimization.md) – Execution planning for queries
- [Analytics & Aggregations](analytics-and-aggregations.md) – Complementary to search (aggregates vs terms)
