# Stream Processing

Processing unbounded data streams (events) in real-time. Apache Flink, Spark Streaming, Kafka Streams.

---

## TL;DR

- **Stream processing**: Process events as they arrive (vs batch = process all at once)
- **Stateless**: Map, filter operations (trivial to scale)
- **Stateful**: Aggregate, join operations (requires coordination)
- **Windowing**: Time-based (tumbling, sliding, session) to handle unbounded data
- **Exactly-once**: Difficult but achievable with frameworks (Flink)

---

## Stream vs Batch

### Batch Processing

```
Process data in intervals:

Data arrives continuously
T=0-60: Accumulate 1000 events
T=60: Run batch job (20 minute latency!)
  Process all 1000 events
  Compute aggregates
  Write results
T=80: Results available
  
Use: Daily reports, analytics, data science
Latency: Minutes to hours
```

---

### Stream Processing

```
Process as events arrive:

Data arrives continuously
T=0: Event 1 arrives → Process immediately
T=5: Event 2 arrives → Process immediately
T=10: Event 3 arrives → Process immediately
...
T=55: Results available (55s latency for all events!)

Use: Real-time dashboards, alerts, recommendations
Latency: Seconds to subseconds
```

---

## Windowing

### Tumbling Window

```
Fixed-size non-overlapping windows

Time: 0----10----20----30----40----50
      [1][2][3]
         [2][3]
            [3]
            
Window 1 (0-10): Events 1,2
Window 2 (10-20): Events 2,3
Window 3 (20-30): Events 3

Use: Daily aggregates, hourly analytics
```

---

### Sliding Window

```
Overlapping windows with step size

Time: 0----10----20----30----40----50
      [1][2][3][4][5]
         [2][3][4][5][6]
            [3][4][5][6][7]

Window size: 50s, slide every 10s

Use: Moving average, running totals
```

---

### Session Window

```
Variable-size windows based on inactivity

Events: User click @ 10:00:05
        User click @ 10:00:20
        (30s gap - session ends)
        User click @ 10:01:10
        User click @ 10:01:15

Session 1: 10:00:05 - 10:00:20 (15s activity, 30s gap ends session)
Session 2: 10:01:10 - 10:01:15 (5s activity)

Use: User session analytics
```

---

## Stateful Operations

### Aggregation (Sum, Count, Average)

```
Requirement: "Count messages per user per minute"

Stream of events:
  user_id, timestamp, message_id
  
Processing:
  Window (1 minute)
  Group by user_id
  Count messages per user
  
State required:
  Keep running count per user per minute
  "user_123 has 45 messages so far (this minute)"
```

---

### Join Streams

```
Two streams:
  Stream A: User clicks (id, timestamp, url)
  Stream B: User profile updates (id, timestamp, name)
  
Join requirement:
  "For each click, add user's current name"
  
Challenge:
  Click @ 10:00:05, Profile update @ 10:00:10
  Must wait or keep profile in state
  
State required:
  Maintain current user profiles
  "user_123's name is Alice (last updated 10:00:10)"
  
Latency tradeoff:
  Wait for join: 10 seconds latency
  Assume state: 1 second latency (might be stale)
```

---

## Exactly-Once Semantics

### Challenge

```
Event arrives, processed, result written, crash

Result not committed to state store
Restart: Event reprocessed (duplicate!)

Solution:
  Transactional write: Process + Commit atomic
  If crash during commit: Rollback, reprocess
```

---

### Implementation (Flink)

```
Flink provides exactly-once via:

1. Checkpoint: Pause processing
   Save all operator state to durable storage
   
2. Resume: Restart from last checkpoint
   Re-process from that point
   Framework deduplicates

Result:
  Each event processed exactly once
  No duplicates, no loss
  Small overhead (~100ms checkpoints)
```

---

## Stream Processing Frameworks

### Kafka Streams

```
Pros:
  ✓ Runs in application (no separate infrastructure)
  ✓ Lightweight (millions of streams per instance)
  ✓ Local state stores (fast)
  
Cons:
  ✗ Limited to Kafka source
  ✗ No automatic recovery (app manages)
  
Use: Kafka-to-Kafka transformations
Example: Enrich events with cached data
```

---

### Apache Flink

```
Pros:
  ✓ Distributed (scales to 1000s of nodes)
  ✓ Exactly-once guarantee
  ✓ Complex windowing
  ✓ State management (remote state stores)
  
Cons:
  ✗ Operational overhead (cluster to manage)
  ✗ More complex deployment
  
Use: Large-scale streaming, strict guarantees
Example: High-volume event processing
```

---

### Apache Spark Streaming

```
Pros:
  ✓ Unified with batch processing (same code)
  ✓ Good performance
  ✓ SQL support
  
Cons:
  ✗ Micro-batching (not truly streaming, 500ms+ latency)
  ✗ Overhead for very low latency needs
  
Use: Stream + batch workloads
Example: Real-time analytics with batch backfill
```

---

## Real-World Example: Fraud Detection

```
Event stream: Credit card transactions

Processing:
  1. For each transaction:
     - Check: Has user made 10+ transactions in past 1 hour?
     - Check: Is amount > user's avg by 3x?
     - Check: Is location different from previous by 1000 km?
  
  2. If 2+ checks triggered: Flag for review

  3. Count: How many transactions from merchant in past 5 min?
     - If > threshold: Block merchant

State required:
  - User's transaction history (last 1 hour)
  - User's transaction amounts (for average)
  - User's last known location
  - Merchant's transaction rate (5 min window)

Latency: <100ms (decision before transaction completes)
Accuracy: 99%+ (minimize false positives)
```

---

## Production Considerations

### Backpressure

```
Producer sending faster than processor can handle

Kafka Streams:
  Automatically handles (consumes slower)
  Latency increases but no data loss

Flink:
  Can handle backpressure
  Adjusts processing rate automatically

Risk: If processor falls behind, delay grows
Monitoring: Track lag (producer vs consumer offset)
```

---

### Exactly-Once Cost

```
At-least-once:
  No checkpointing overhead
  Latency: ~10ms
  Risk: Duplicates possible
  
Exactly-once:
  Periodic checkpoints (~100ms)
  Latency: ~100-500ms
  Safety: Guaranteed no duplicates
  
Trade-off:
  Most use at-least-once + idempotency
  Avoid exactly-once overhead
```

---

## Related Fundamentals

- [Kafka Architecture](kafka-architecture.md) – Event source
- [Batch Processing](../batch-and-stream-processing/) – Related pattern
- [Exactly-Once](exactly-once-semantics.md) – Delivery guarantees

---

**Status**: ✅ Complete. Covers windowing, stateful ops, frameworks, exactly-once.

