# Skip Lists

Probabilistic data structure providing O(log N) search, insert, and delete with simpler implementation than balanced trees. Used in Redis, LevelDB, and other systems.

---

## TL;DR

- **Skip list**: Linked list with probabilistic skip pointers (express lanes)
- **Levels**: Multiple layers, each skipping nodes (higher levels skip more)
- **Complexity**: O(log N) search, insert, delete (probabilistic)
- **Comparison to B-tree**: Similar performance, simpler code
- **Rebalancing**: None needed (probability handles it)
- **Use cases**: Sorted sets, ordered indexes, in-memory databases

---

## How Skip Lists Work

### Single Linked List

```
Sorted linked list (no speedup):
  1 → 3 → 5 → 7 → 9 → 11

Search for 7:
  1 → 3 → 5 → 7 ✓ (4 steps)

Search for 8 (not in list):
  1 → 3 → 5 → 7 → 9 (STOP, too big) (5 steps)

Worst case: O(N) (search from head)
```

### Skip List with Express Lanes

```
Level 2 (express):
  1 ────→ 5 ────→ 9
  ↓       ↓       ↓
Level 1 (normal):
  1 → 3 → 5 → 7 → 9 → 11

Search for 7:
  Level 2: 1 → 5 (stop, 9 is too big)
  Level 1: 5 → 7 ✓ (2 + 2 = 4 steps, same)
  
  But with more levels...

Level 3 (express fast):
  1 ────────────→ 9
  ↓              ↓
Level 2 (express):
  1 ────→ 5 ────→ 9
  ↓       ↓       ↓
Level 1 (normal):
  1 → 3 → 5 → 7 → 9 → 11

Search for 7:
  Level 3: 1 → 9 (stop, too big)
  Level 2: 5 → 9 (stop, too big)
  Level 1: 5 → 7 ✓ (3 comparisons)
  
Complexity: O(log N) instead of O(N)
```

---

## Skip List Structure

### Construction

```
Each node has random height (number of forward pointers):
  Node value: 5
  Level 1: Pointer to 7
  Level 2: Pointer to 9
  Level 3: Pointer to 13

Random height chosen probabilistically:
  50% chance of height 1
  25% chance of height 2
  12.5% chance of height 3
  6.25% chance of height 4
  ...
  
Probability: P(height ≥ k) = (1/2)^k
```

### Visual Example

```
Insert: 1, 3, 5, 7, 9, 11, 13

Level 3:     1 ─────────────────────→ 13
             ↓                        ↓
Level 2:     1 ───→ 5 ───→ 9 ────→ 13
             ↓      ↓      ↓        ↓
Level 1:  1 → 3 → 5 → 7 → 9 → 11 → 13
          ↓   ↓   ↓   ↓   ↓   ↓    ↓
Level 0:  1 → 3 → 5 → 7 → 9 → 11 → 13 → 15

Node 1: height 3
Node 3: height 1
Node 5: height 2
Node 7: height 1
Node 9: height 2
Node 11: height 1
Node 13: height 3
Node 15: height 1
```

---

## Search Operation

### Algorithm

```
Search for key K:
  1. Start at top level
  2. Move forward while node < K
     (stop when next node >= K)
  3. Move down one level
  4. Repeat until Level 0
  5. Check if current node == K

Complexity: O(log N)
  - Log N levels (each level skips ~50% of nodes)
  - Each level traverses ~2 nodes (geometric distribution)
  - Total: log N levels × 2 nodes = O(log N)
```

### Search Example

```
Skip list:
  Level 3:     1 ─────────────────→ 13
  Level 2:     1 ───→ 5 ───→ 9 ───→ 13
  Level 1:  1 → 3 → 5 → 7 → 9 → 11 → 13

Search for 7:
  1. Start at Level 3, node 1
     1 < 7, try next (13)
     13 > 7, stop, go down
  
  2. Level 2, node 1
     1 < 7, try next (5)
     5 < 7, try next (9)
     9 > 7, stop, go down
  
  3. Level 1, node 5
     5 < 7, try next (7)
     7 == 7, FOUND ✓
     
  Steps: 1 + 2 + 2 = 5 operations
```

---

## Insert and Delete

### Insert

```
Insert 6:
  1. Search for 6 (find insertion point between 5 and 7)
  2. Create new node with random height (say height 2)
  3. Update pointers:
     Level 1: 5.next = new_node; new_node.next = 7
     Level 2: 5.next = new_node; new_node.next = 9
  4. Done

Before:
  Level 2:  1 ───→ 5 ───→ 9 ───→ 13
  Level 1:  1 → 3 → 5 → 7 → 9 → 11 → 13

After (insert 6, height 2):
  Level 2:  1 ───→ 5 ───→ 6 ───→ 9 ───→ 13
  Level 1:  1 → 3 → 5 → 6 → 7 → 9 → 11 → 13

Complexity: O(log N) to find position + O(1) pointer updates = O(log N)
```

### Delete

```
Delete 6:
  1. Search for 6
  2. Remove from all levels
     Level 1: 5.next = 7
     Level 2: 5.next = 9
  3. Done

Before:
  Level 2:  1 ───→ 5 ───→ 6 ───→ 9 ───→ 13
  Level 1:  1 → 3 → 5 → 6 → 7 → 9 → 11 → 13

After (delete 6):
  Level 2:  1 ───→ 5 ───→ 9 ───→ 13
  Level 1:  1 → 3 → 5 → 7 → 9 → 11 → 13

Complexity: O(log N)
```

---

## Comparison with B-Tree

### Skip List Advantages

```
1. Simpler to implement
   B-tree: Complex rotations, rebalancing logic
   Skip list: Random height, straightforward pointer updates
   
2. Range queries easy
   B-tree: Complex tree traversal
   Skip list: Just follow level 0 pointers
   
3. Concurrent inserts easier
   B-tree: Rebalancing affects multiple nodes
   Skip list: Lock only affected nodes
```

### B-Tree Advantages

```
1. Better cache locality
   B-tree: Multiple keys per node (better CPU cache)
   Skip list: Pointers scattered across memory
   
2. Deterministic O(log N)
   B-tree: Guaranteed balanced (no variance)
   Skip list: Probabilistic (rare worst case)
   
3. Space predictable
   B-tree: Exact space usage calculable
   Skip list: Variable (depends on random heights)
```

### Complexity Comparison

| Operation | Skip List | B-Tree |
|---|---|---|
| **Search** | O(log N) prob | O(log N) |
| **Insert** | O(log N) prob | O(log N) |
| **Delete** | O(log N) prob | O(log N) |
| **Range query** | O(log N + K) | O(log N + K) |
| **Space** | O(N) avg | O(N) |
| **Rebalancing** | None | Complex |

---

## Probabilistic Balancing

### Why It Works

```
Intuition: Probability keeps skip list balanced

Each level skips ~50% of nodes
  If you have 1M nodes:
    Level 0: 1M nodes (all)
    Level 1: ~500k nodes (50% randomly chosen)
    Level 2: ~250k nodes (50% of level 1)
    Level 3: ~125k nodes (50% of level 2)
    ...
    Level 20: ~1 node (1M / 2^20)

Height needed: ~log2(N) = log2(1M) ≈ 20

Search traverses:
  ~log N levels
  ~2 nodes per level
  Total: ~2 * log N = O(log N)

Probability ensures natural distribution
(no explicit rebalancing needed)
```

### Variance Analysis

```
In practice, skip lists show:
  - Very consistent O(log N) performance
  - Rare worst-case deviations
  - No need for rebalancing operations
  
With N=1,000,000:
  Expected max height: ~25
  Actual max height: Usually 20-25
  Never hits theoretical worst case O(N)
```

---

## Real-World Usage

### Redis Sorted Sets

```
Redis uses skip lists for sorted sets:
  ZADD myzset 1 "one"
  ZADD myzset 2 "two"
  ZADD myzset 3 "three"
  ZRANGE myzset 0 -1  # Returns members in sorted order
  ZRANK myzset "two"  # Returns rank (fast, O(log N))

Why skip list?
  - Fast insertion while maintaining order
  - Range queries efficient
  - Concurrent operations simple
```

### LevelDB (Google)

```
LevelDB uses skip list for memtable (in-memory buffer)
  New writes go to memtable (skip list)
  Sorted by key (skip list maintains order)
  When full, flush to disk (creates SSTable)
  
Why?
  - Fast inserts while sorted
  - Easy range queries
  - Memory efficient
```

---

## Production Considerations

### 1. Choice of P (Probability Parameter)

```
Standard choice: P = 0.5
  50% chance of promoting each level
  Expected height: log_2(N)
  
Alternative: P = 0.25 (25% promotion)
  Expected height: log_4(N)
  Pros: Fewer levels, faster search
  Cons: More complex probabilistic analysis
  
P = 0.5 is most common (balance of simplicity and performance)
```

### 2. Max Height

```
Pre-compute max height:
  maxHeight = ceil(log(N))
  
Why? Avoid unbounded random heights
  Very unlikely to exceed (1 - P)^maxHeight
  
Example: N = 1M, P = 0.5, maxHeight = 20
  Probability of exceeding: ~0.000001% (negligible)
```

### 3. Memory Overhead

```
Storage per node:
  Value: 8 bytes (int) or more (string)
  Pointers: height × 8 bytes
  
Average pointers per node (P=0.5):
  Level 1: 1 pointer
  Level 2: 0.5 pointers
  Level 3: 0.25 pointers
  ...
  Total: 1 + 0.5 + 0.25 + ... = 2 pointers average
  
Memory per node: 8 + 16 = 24 bytes (for 64-bit pointers)
For 1M nodes: 24 MB (acceptable)
```

### 4. Concurrent Access

```
Skip list advantages for concurrency:
  - Lock only affected levels during insert/delete
  - Not entire tree (unlike some B-tree implementations)
  - Multiple readers can scan simultaneously
  
Typical approach:
  - Read-write lock per node
  - Or optimistic locking (check version on write)
```

---

## References

- Pugh, W. (1990) "Skip Lists: A Probabilistic Alternative to Balanced Trees"
- Redis sorted set implementation
- LevelDB memtable implementation

---

## Related Fundamentals

- [Indexing](../databases/indexing.md) – B-tree, LSM tree alternatives
- [Bloom Filters](bloom-filters.md) – Another probabilistic structure
- [Databases/Transactions](../databases/transactions-and-isolation-levels.md) – Concurrent access patterns
- [Caching](../caching/) – In-memory index structures

---

**Status**: ✅ Complete. Covers mechanics, search/insert/delete, probability analysis, and production patterns.
