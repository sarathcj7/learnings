# E-Commerce Inventory & Flash Sale

*Design Alibaba/Amazon flash sale: millions of concurrent requests, inventory accuracy, no overselling.*

## Problem Statement

Design e-commerce inventory for a flash sale. 1M users simultaneously trying to buy limited stock (100 units). Must prevent overselling. Handle stock updates atomically.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Concurrent requests** | 1M | Peak traffic |
| **Stock available** | 100 | Very limited |
| **Overstock ratio** | ~1000:1 | Most fail, few succeed |
| **QPS (add-to-cart)** | 100k | High throughput |

## Challenges

### Preventing Overselling

```
Naive approach:
  IF stock > 0:
    Sell item
    Update stock
  
Problem: Race condition
  Thread 1: stock=1, check 1>0 ✓
  Thread 2: stock=1, check 1>0 ✓
  Thread 1: stock = 0
  Thread 2: stock = -1 (oversold!)
  
Solution: Atomic database transaction
  BEGIN
    SELECT stock WHERE item_id = 123 FOR UPDATE (lock)
    IF stock > 0:
      stock -= 1
      INSERT order
  COMMIT
  (prevents race)
```

### Handling Massive Load

```
1M concurrent add-to-cart requests
Most will fail (only 100 succeed)

Solution: Early rejection
  Lottery: Accept 10k requests, reject 990k immediately
  Process accepted 10k (attempt purchases, 100 succeed, 9.9k fail with "sold out")
  vs. Process all 1M (server overload)
```

### Inventory Cache

```
Cache: stock = 100
User 1 buys 1: cache = 99
User 2 buys 1: cache = 98
...

After 100 sales: cache = 0, REJECT all further
But: What if cache is stale? (actual DB = 50)

Solution:
  Cache-only for fast rejection path
  DB always source of truth
  Periodic sync (every 10s) to catch drift
```

## Architecture

```
API Gateway (rate limiting):
  10k requests/sec from 1M users
  Others get 429 Too Many Requests

Cache Layer (Redis):
  Quick stock check: cache.get("stock:item-123")
  
Order Service (transactional):
  Acquire lock on inventory row
  Check actual stock in DB
  If available: Sell + insert order
  Release lock

Inventory Service:
  Replica DB in each region
  Async sync to primary
  Trade: Availability over consistency
```

## Bottlenecks & Scaling

**Bottleneck**: Inventory lock contention (100 successful attempts, 100k lock attempts).

**Solution**:
- Pessimistic locking: Lock early, hold briefly
- Or: Optimistic locking (retry on conflict)
- Or: Queue model (queue orders, process sequentially)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Transactional DB** | Distributed queue | Simplicity + strong consistency |
| **Early rejection** | Process all requests | Scalability (reduce load) |
| **Cache + periodic sync** | Always consistent | Performance (cache hits) over consistency |

---

**Status**: ✅ Complete. Shows inventory, transactions, flash sale design.
