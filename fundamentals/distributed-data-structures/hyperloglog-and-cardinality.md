# HyperLogLog and Cardinality Estimation

Probabilistic algorithm for estimating distinct element count with constant memory. Trades small error for 1000x+ memory savings versus exact sets.

---

## TL;DR

- **Cardinality**: Count of distinct elements (unique visitors, unique IPs)
- **HyperLogLog**: Probabilistic cardinality estimation
- **Memory**: O(log log N) bits per element (incredibly small)
- **Accuracy**: ~2% error configurable
- **Speed**: O(1) insertion and query
- **Use cases**: Unique visitors, unique IPs, distinct values in stream

---

## The Problem

### Exact Cardinality (Expensive)

```
Count unique visitors to website:

Approach 1: Set
  Store all visitor IPs: {1.1.1.1, 2.2.2.2, 3.3.3.3, ...}
  
Results:
  1 billion visitors
  IP addresses: 4 bytes each
  Memory: 1B × 4 bytes = 4 GB
  
Disadvantage: Linear memory (4 GB for exact count)
```

### HyperLogLog (Efficient)

```
Estimate unique visitors:

HyperLogLog:
  Store only hash patterns
  Estimate count from patterns
  
Results:
  1 billion visitors
  HyperLogLog: 14 KB (configurable precision)
  Error: ~2%
  
Benefit: 4GB vs 14KB = 300,000x smaller
Trade-off: 2% error (acceptable for analytics)
```

---

## How HyperLogLog Works

### Core Insight

```
Observation: When hashing random data with k bits,
the maximum leading zeros follow a pattern.

Hash of random element: 0b100101...
  Leading zeros: 2

Many hashes: 0b1, 0b0001, 0b01, 0b100, 0b010000
  Maximum leading zeros so far: 4

More elements hashed:
  Keep tracking maximum leading zeros
  Distribution of max leading zeros indicates cardinality

High cardinality → Larger max leading zeros likely
Low cardinality → Small max leading zeros likely
```

### Algorithm

```
1. Initialize array M of size m = 2^b
   Example: b = 4, m = 16 registers (each stores max leading zeros)
   M = [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

2. For each element E:
   a. Hash E to get h (64-bit hash)
   b. Split hash into:
      - First b bits: j (register index)
      - Remaining bits: w (leading zero count + 1)
   c. M[j] = max(M[j], leading_zeros(w) + 1)

3. Estimate cardinality:
   E = α_m * m^2 / sum(2^(-M[j]))
   
   Where α_m is calibration constant
   m = number of registers (2^b)
```

### Example with Small Values

```
b = 4 (16 registers, 4-bit precision)
m = 16

Hash element "alice":
  Hash: 0x1A2B3C4D (binary: 00011010001010110011110001001101)
  First 4 bits (j): 0001 = 1
  Remaining: 010001010110011110001001101
  Leading zeros: 1 (bit pattern starts with 0, then 1)
  M[1] = max(0, 1+1) = 2

Hash element "bob":
  Hash: 0x5E6F7A8B (binary: 01011110011011110111101010001011)
  First 4 bits (j): 0101 = 5
  Remaining: 110011011110111101010001011
  Leading zeros: 0 (bit pattern starts with 1)
  M[5] = max(0, 0+1) = 1

Hash element "charlie":
  Hash: 0x0000FFFF
  First 4 bits (j): 0000 = 0
  Remaining: 1111111111111111
  Leading zeros: 0
  M[0] = 1

After processing:
  M = [1, 2, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
  
Estimate cardinality from M:
  E = α_16 * 256 / (2^(-1) + 2^(-2) + 2^(-1) + ...)
  Result: ~3 (approximately correct for 3 elements)
```

---

## Accuracy and Configuration

### Precision Parameter

Precision (b) determines accuracy:

```
b = 4  (16 registers):   ~20% error,  2 bytes
b = 8  (256 registers):  ~6.5% error, 32 bytes
b = 12 (4k registers):   ~2% error,   512 bytes
b = 14 (16k registers):  ~1% error,   2 KB
b = 16 (64k registers):  ~0.5% error, 8 KB

Typical choice: b = 12-14 (2% error, KB-scale memory)
```

### Error Characteristics

```
Standard error: 1.04 / sqrt(m)

m = 16 registers: error = 1.04 / sqrt(16) = 26%
m = 256 registers: error = 1.04 / sqrt(256) = 6.5%
m = 4096 registers: error = 1.04 / sqrt(4096) = 1.6%

Higher cardinality: Error stays ~constant percentage
  1k distinct: ~1.6% error (estimate 1016)
  1M distinct: ~1.6% error (estimate 1.016M)
```

---

## Use Cases

### Use Case 1: Unique Visitors

```
Website analytics:
  Track unique visitors per day

Exact approach (HashSet):
  Store IP of every visitor
  1M unique visitors × 4-8 bytes = 4-8 MB per day
  1 year: 1.4-2.8 GB
  
HyperLogLog approach:
  One HyperLogLog per day (2 KB)
  1 year: 730 KB
  
Result: 4000x smaller
Error: ~2% (1M visitors ± 20k acceptable)
```

### Use Case 2: Duplicate Detection

```
Data deduplication in pipeline:

Stream of events: pageviews, clicks, impressions
Need to detect: New event vs already seen

Approach:
  1. Maintain HyperLogLog of seen event IDs
  2. For each new event:
     - Check HyperLogLog: "Seen before?"
     - If yes (95% confidence): Skip (duplicate)
     - If no (95% confidence): Process (new)
  3. False positives/negatives: ~5% (acceptable)

Result: Fast dedup without storing all seen IDs
Memory: Constant regardless of event count
```

### Use Case 3: Cardinality-Based Alerting

```
Infrastructure monitoring:

Alert: "Too many unique error codes in system"

Metric:
  Unique error codes per minute
  If > 1000 unique codes → Potential issue
  
Implementation:
  Maintain HyperLogLog of error codes per minute
  At minute boundary, read cardinality
  Compare to threshold
  
Benefit:
  Detect anomalies without storing all errors
  Memory constant (single HyperLogLog per metric)
  Real-time (O(1) update and query)
```

### Use Case 4: Set Cardinality Union

```
Combine cardinalities from multiple sources:

Social network: Followers union
  User A followers: HyperLogLog (1M followers, 2KB)
  User B followers: HyperLogLog (500k followers, 2KB)
  Union cardinality: Merge HyperLogLogs → ~1.2M (with overlap)

Implementation:
  1. Merge register arrays (take max of each)
  2. Estimate cardinality from merged array
  
Result: Estimate of union without storing all followers
Accuracy: ~2% (same as individual HyperLogLog)
```

---

## Advanced Variants

### HyperLogLog++

Refinement with better accuracy:

```
Improvements:
  1. Sparse representation (only non-zero registers)
     Saves space for small cardinalities
  2. Improved bias correction
     Better accuracy for very small/large cardinalities
  3. Explicit counts for small cardinalities
     Switch to exact count when |S| < 2.5m
  
Result: Better accuracy across full range
Used in: Google BigQuery, Redis
```

### Adaptive Counting

```
T-Digest-like approach:
  Start with exact set (HashSet)
  When memory exceeds threshold:
    - Switch to HyperLogLog
    - Discard exact set
  
Result: Exact answers for small sets, HyperLogLog for large
Benefit: No error for small cardinalities
```

---

## Memory Analysis

### HyperLogLog Memory

```
Standard HyperLogLog:
  b = 12 (4k registers)
  Each register: 5-6 bits (stores leading zero count, 0-64)
  Total: 4096 × 5 bits / 8 = 2.5 KB
  
Per-element added: ~0 KB (constant)
  Adding 1 element vs 1M elements: Same memory
  
Comparison with HashSet:
  1M elements × 64 bits = 8 MB
  HyperLogLog: 2.5 KB
  Ratio: 3200x smaller
```

### Space Complexity

```
Theoretical: O(log log N) bits per element
  For N = 1 billion:
  log log(1B) = log(30) ≈ 5 bits per element
  1B × 5 bits = 625 MB (theoretical minimum)
  
Practical: O(1) bits (doesn't grow with N)
  HyperLogLog size fixed (doesn't depend on N)
  
Unlike Bloom filter: O(log N) bits per element
  Larger for very large sets
```

---

## Production Considerations

### 1. Merging Multiple Streams

```
Combine counts from multiple servers:

Server A: HyperLogLog of unique users today
Server B: HyperLogLog of unique users today
Server C: HyperLogLog of unique users today

Merge (element-wise max of registers):
  HyperLogLog merged = combine(A, B, C)
  Cardinality = estimate(merged)
  
Result: Total unique users across all servers
Error: Still ~2% (same as individual)
```

### 2. Time Series Cardinality

```
Track unique visitors over time:

Per-hour HyperLogLog: 24 HyperLogLogs (2 KB each = 48 KB/day)
Per-day HyperLogLog: 365 HyperLogLogs (730 KB/year)

Query: "Unique users in Jan-Feb?"
  Merge 59 daily HyperLogLogs → Estimate
  Result in O(59 × 4k) = quick
  
vs Exact set: 59 × (millions of bytes) = GB scale
```

### 3. False Positives/Negatives

```
HyperLogLog estimates cardinality, not membership

Cannot ask: "Is user 123 in the set?"
  (Bloom filter answers this)

Can ask: "Approximately how many users?"
  (HyperLogLog answers this)

Combined approach:
  HyperLogLog: "~1M unique users"
  Set (for recent data): "Which of last 1000 users"
  Bloom filter: "Has this user ever logged in?"
```

### 4. Serialization

```
Serialize HyperLogLog to persistent storage:
  Save register array (4KB for b=12)
  On load, deserialize and estimate
  
Version stability:
  Ensure b stays constant (changing breaks estimates)
  Format stable across platforms (byte order)
```

---

## References

- Flajolet, P., et al. (2007) "HyperLogLog: The analysis of a near-optimal cardinality estimation algorithm"
- Redis HyperLogLog implementation
- Google Bigtable/BigQuery cardinality estimation

---

## Related Fundamentals

- [Bloom Filters](bloom-filters.md) – Membership testing (different problem)
- [Skip Lists](skip-lists.md) – Another probabilistic structure
- [Caching](../caching/) – Set cardinality for cache statistics
- [Databases/Indexing](../databases/indexing.md) – Cardinality for query optimization

---

**Status**: ✅ Complete. Covers algorithm, accuracy, use cases, and production patterns.
