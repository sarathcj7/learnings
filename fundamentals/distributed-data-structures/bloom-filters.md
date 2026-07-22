# Bloom Filters

Probabilistic data structure for fast membership testing with controlled false positives. Trade small false positive rate for constant memory and O(1) lookup time.

---

## TL;DR

- **Bloom filter**: Space-efficient membership test (is X in set?)
- **False positives**: Possible (say yes when should say no)
- **False negatives**: Impossible (never miss valid members)
- **Memory**: O(n) bits (not O(n) items), ~5-10 bits per item
- **Lookup**: O(k) hash functions, typically k=3-5 (still O(1) effectively)
- **Use cases**: Deduplication, cache filters, URL blacklists, spell checkers

---

## How Bloom Filters Work

### Basic Concept

A probabilistic hash table trading accuracy for space.

```
Bit array (initialize all zeros):
  [0][0][0][0][0][0][0][0][0][0][0][0]
   0  1  2  3  4  5  6  7  8  9 10 11

Add "apple":
  hash1("apple") = 3  → Set bit[3] = 1
  hash2("apple") = 7  → Set bit[7] = 1
  hash3("apple") = 11 → Set bit[11] = 1
  
After adding "apple":
  [0][0][0][1][0][0][0][1][0][0][0][1]
   0  1  2  3  4  5  6  7  8  9 10 11

Test membership ("apple"):
  hash1("apple") = 3  → bit[3] = 1 ✓
  hash2("apple") = 7  → bit[7] = 1 ✓
  hash3("apple") = 11 → bit[11] = 1 ✓
  Result: LIKELY in set (all bits set)

Test membership ("banana"):
  hash1("banana") = 2  → bit[2] = 0 ✗
  hash2("banana") = 5  → bit[5] = 0 ✗
  hash3("banana") = 8  → bit[8] = 0 ✗
  Result: DEFINITELY NOT in set (at least one bit unset)

Test membership ("grape"):
  hash1("grape") = 3  → bit[3] = 1 ✓
  hash2("grape") = 5  → bit[5] = 0 ✗
  Result: DEFINITELY NOT in set
  
Test membership ("pear"):
  hash1("pear") = 3  → bit[3] = 1 ✓ (was set by "apple")
  hash2("pear") = 7  → bit[7] = 1 ✓ (was set by "apple")
  hash3("pear") = 9  → bit[9] = 0 ✗
  Result: DEFINITELY NOT in set
  
False positive example:
  If by coincidence hash1("mango")=3, hash2("mango")=7, hash3("mango")=11
  → All bits set → FALSE POSITIVE (say "in set" but it's not)
```

---

## False Positive Rate

### Trade-off

Larger filter → Lower false positive rate.

```
n = 10,000 items
k = 3 hash functions

Filter size: 50,000 bits (5 bits per item)
  False positive rate: ~2%
  
Filter size: 100,000 bits (10 bits per item)
  False positive rate: ~0.1%
  
Filter size: 200,000 bits (20 bits per item)
  False positive rate: ~0.001%
```

### Formula

```
Theoretical false positive rate:
  f = (1 - e^(-k*n/m))^k
  
Where:
  m = bit array size (bits)
  n = number of items
  k = number of hash functions
  
Example:
  m = 100,000 bits
  n = 10,000 items
  k = 3
  f = (1 - e^(-3*10000/100000))^3 = (1 - e^(-0.3))^3 ≈ 0.0005 (0.05%)
```

### Optimal Hash Functions

```
Optimal k = (m / n) * ln(2) ≈ 0.693 * (m / n)

Example:
  m = 100,000 bits
  n = 10,000 items
  optimal k = (100000 / 10000) * ln(2) = 10 * 0.693 ≈ 7 hash functions

Too few k: Higher false positive rate
Too many k: Worse hash independence, still higher FP rate
```

---

## Bloom Filter Variants

### Counting Bloom Filter

Supports deletion (standard Bloom filter doesn't).

```
Instead of 1 bit per position, use counter (8 bits):
  [1][2][0][3][1][0][1][2][0][1][0][1]

Add "apple":
  Increment counters at hash positions
  counter[3]++, counter[7]++, counter[11]++

Delete "apple":
  Decrement counters at hash positions
  counter[3]--, counter[7]--, counter[11]--

Problem: More memory (8 bits per position vs 1 bit)
Advantage: Supports deletion
```

---

## Use Cases

### Use Case 1: Cache Filter

```
Scenario: Distributed cache miss problem

Large website with millions of users
Cache backend (Redis) stores popular data
Problem: Users request rarely-used items
  → Cache misses for non-existent items
  → Origin database hit (slow)
  
Solution: Bloom filter at cache layer
  Before querying database, check Bloom filter
  If filter says "not in database" → Return 404 (no DB hit)
  If filter says "maybe in database" → Query DB

Result:
  Database hits: Only for items likely in DB
  Saves 90%+ of unnecessary DB queries
  Small memory overhead (few MB for Bloom filter)
```

### Use Case 2: Deduplication

```
Scenario: Process log entries, remove duplicates

High-volume logging system
Logs: 100M entries per hour
Need to deduplicate within batch

Approach:
  1. Create Bloom filter for batch
  2. For each log entry:
     - Check Bloom filter
     - If seen: Skip (duplicate)
     - If not seen: Add to Bloom, process entry, add to output
  3. Output deduplicated logs

Memory: 100M entries × 10 bits = 125 MB
vs Set/HashSet: 100M entries × 100 bytes = 10 GB
  
Trade-off: ~0.1% false positive rate (rare)
Benefit: 80x less memory
```

### Use Case 3: URL Blacklist

```
Scenario: Browser needs to check if URL is malicious

Millions of known malware URLs
Can't store all in memory locally (browser)

Solution: Compact Bloom filter
  1. Google maintains Bloom filter of malicious URLs
  2. Distributes to all browsers (small file, few MB)
  3. Browser checks Bloom filter locally (instant)
  4. False positives: Query Google server (rare)
  
Benefit:
  - Instant local checking
  - Compact distribution (Bloom filter < URL list)
  - Rare server queries (most URLs safe, don't need confirmation)
```

### Use Case 4: Spell Checker

```
Scenario: Mobile spell checker, offline mode

Dictionary: 500k words
Can't store all dictionary data on phone (space limited)

Solution: Bloom filter of dictionary
  1. Create Bloom filter from dictionary (5 MB)
  2. Distribute with app
  3. Check spelling: Bloom filter + small exception list
  
False positives: Accept rare misspellings (~0.1%)
Benefit: Small dictionary file, instant checking
```

---

## Implementation Considerations

### Hashing

```
Multiple independent hashes:
  hash1 = MurmurHash(x, seed1)
  hash2 = MurmurHash(x, seed2)
  hash3 = MurmurHash(x, seed3)

OR double hashing (faster):
  g_i(x) = (hash1(x) + i * hash2(x)) mod m
  
Requirement: Hash outputs must be independent
  (not correlated, spread evenly)
```

### Resizing

```
Standard Bloom filters can't be resized
(changing m requires rehashing all elements)

Two solutions:
  1. Estimate size upfront, never resize
  2. Partitioned Bloom filters:
     - Multiple smaller filters
     - Add new filter when growing
     - Check all filters during lookup
```

### Thread Safety

```
Concurrent adds/lookups:
  Thread A: Add "item1"
  Thread B: Lookup "item2"
  
Not a problem:
  Only setting bits (never unsetting)
  Bit operations are atomic (on modern CPUs)
  
Counting Bloom (with deletes):
  Requires locks (decrement not atomic with reads)
```

---

## Comparison with Alternatives

| Data Structure | Memory | Lookup | Insert | False Positive |
|---|---|---|---|---|
| **Bloom Filter** | O(n) bits | O(k) | O(k) | Yes (configurable) |
| **HashSet** | O(n) items | O(1) avg | O(1) avg | No |
| **B-tree** | O(n) items | O(log n) | O(log n) | No |
| **Sorted array + binary search** | O(n) items | O(log n) | O(n) | No |

**Bloom filter wins** when:
- Space is critical (order of magnitude savings)
- False positives acceptable
- Inserts/lookups vastly exceed deletes

---

## Real-World Examples

### Google Chrome Safe Browsing

```
Protects against malware URLs
Bloom filter of dangerous URLs: 2.4 MB
Full URL list: 2.4 GB

User checks URL:
  1. Local Bloom filter check (instant) ✓
  2. If Bloom says "possible", query Google (rare) ✓
  
Result: Instant for 99%+ of URLs, verified for unsafe URLs
```

### Cassandra Database

```
Bloom filters in LSM tree compaction
When compacting SSTables:
  Check Bloom filter before reading SSTable
  If key not in filter → Skip entire SSTable
  Saves disk I/O
```

### Apache HBase

```
Each HBase region server maintains Bloom filters
When looking up key:
  Check Bloom before disk read
  Reduces unnecessary I/O significantly
```

---

## Production Considerations

### 1. Calibration

```
Estimate maximum set size (n)
Choose acceptable false positive rate (typically 1-2%)
Calculate required bit array size (m)

Too small: False positive rate unacceptable (>5%)
Too large: Memory waste (unnecessary precision)

Use online calculator or formula
```

### 2. Monitoring

```
Track false positive rate in production:
  Count lookups returning "maybe"
  Count database/server queries confirming false positives
  FP rate = confirmed FP / total "maybe" lookups
  
If rate > expected:
  Might need larger filter
  Or size estimate was wrong
```

### 3. Serialization

```
Bloom filters must be persistent:
  Serialize to bytes (bit array)
  Deserialize from disk/network
  
Ensure bit array format stable across versions
  (Endianness, padding, format)
```

---

## References

- "Designing Data-Intensive Applications" — Kleppmann, Chapter 3
- Bloom, B. H. (1970) "Space/time trade-offs in hash coding with allowable errors"
- Apache Cassandra Bloom filters documentation

---

## Related Fundamentals

- [Indexing](../databases/indexing.md) – Data lookup structures
- [Caching](../caching/) – Complementary cache filtering strategy
- [HyperLogLog](hyperloglog-and-cardinality.md) – Another probabilistic structure
- [Skip Lists](skip-lists.md) – Alternative probabilistic structure

---

**Status**: ✅ Complete. Covers mechanics, false positive rates, use cases, and production patterns.
