# Kappa Architecture

Stream-only alternative to Lambda architecture. Process all data through a unified streaming layer with event log replay for recomputation.

---

## TL;DR

- **Kappa**: Single streaming layer (no batch layer separation)
- **Event log replay**: Recompute from scratch by replaying events from beginning
- **Simpler**: No dual pipelines, unified codebase
- **Trade-off**: Requires replayable events and immutable event log
- **When to use**: Real-time analytics, when data fits stream model
- **vs Lambda**: Simpler code but need robust event log

---

## Problem with Lambda Architecture

```
Lambda complexity:
  - Batch layer: Hadoop/Spark jobs nightly
  - Speed layer: Kafka/Flink real-time
  - Serving layer: Merge results
  
Challenges:
  1. Duplicate code: Same logic in batch and stream
  2. Operational burden: Two systems to maintain
  3. Complexity: Merging results at serving layer
  4. Bugs: Subtle differences between layers cause discrepancies
```

---

## Kappa Architecture Model

```
Single streaming pipeline:

Events → Kafka (immutable log) → Stream processor → Results DB
                    ↑
                    └─ Replay for recomputation
                    
Benefits:
  - Single codebase for all processing
  - No Lambda complexity
  - Easier testing and debugging
  - Clear data lineage
```

---

## How Recomputation Works

### Scenario: Bug in computation logic

```
Original: Calculate user total spend (has bug - counts refunds as revenue)

Events log:
  Event 0: Purchase $100 (user_1)
  Event 1: Refund -$20 (user_1)
  Event 2: Purchase $50 (user_1)
  
Buggy result: user_1 = $100 + (-$20) + $50 = $130 (wrong, treats refund as +$20)

Fix applied: Change logic to subtract refunds properly

Recompute:
  1. Rewind consumer offset to 0 (start of log)
  2. Re-process all events with fixed logic
  3. user_1 = $100 - $20 + $50 = $130 (correct)
  4. Write corrected results to fresh state store
  5. Atomically switch results DB

No data loss, complete correctness
```

---

## Event Log as Source of Truth

### Immutability Requirement

```
Event log (Kafka):
  Offset 0: { timestamp: 2026-01-01T08:00:00, user_id: 1, action: "purchase", amount: 100 }
  Offset 1: { timestamp: 2026-01-01T08:05:00, user_id: 2, action: "login" }
  Offset 2: { timestamp: 2026-01-01T08:10:00, user_id: 1, action: "refund", amount: 20 }
  ...
  
Properties required:
  1. Append-only (no updates)
  2. Immutable events (no changes to past events)
  3. Timestamped (capture causality)
  4. Complete (all important state changes as events)
  
This ensures:
  - Replaying always produces same result
  - Deterministic computation
  - No surprises from changed data
```

---

## Use Case: Real-time Analytics Platform

### Architecture

```
Events generated:
  User page views, clicks, searches, purchases
  
Event Stream:
  Kafka topic: "user-events"
  Retention: 30 days (3 billion events)
  Partitions: 100
  
Stream Processor:
  Flink/Kafka Streams application
  Stateful computation:
    - Track user session (30 min inactivity timeout)
    - Aggregate events per session
    - Calculate metrics (page views, clicks, revenue)
  State store: RocksDB (on disk)
  
Output:
  Store results in: Elasticsearch/TimescaleDB
  Dashboard queries latest metrics
  
Recomputation flow:
  Alert: Metrics don't match expected revenue for day 5
  Investigate: Found bug in purchase amount parsing
  Fix code: Deploy new version
  Recompute: Restart Flink from offset 0
  Parallel replay: 100 partitions × 50 events/sec = 5000 events/sec
  30 days of data: 2.5B events ÷ 5000 = ~6 days
  After 6 days: All metrics recomputed correctly
  Switch results: Point dashboard to new results
```

---

## Kappa vs Lambda: Detailed Comparison

| Aspect | Kappa | Lambda |
|---|---|---|
| **Layers** | 1 (stream only) | 3 (batch + stream + serving) |
| **Code duplication** | None | High (batch + stream logic) |
| **Recomputation** | Event log replay | Batch job rerun |
| **Time to recompute** | Streaming speed | Batch window (slow) |
| **Complexity** | Low | High |
| **Operational overhead** | Low | High |
| **Data consistency** | Strong (single source) | Eventual (merge delay) |
| **Requirements** | Immutable event log | Batch storage + stream |
| **Best for** | Real-time, low latency | Mixed latency requirements |

---

## Design Patterns in Kappa

### Pattern 1: Windowed Aggregations

```
Stream: User purchase events
Processor: Calculate revenue per user per hour

Kafka Streams topology:
  Source: Purchase events
  Windowed aggregation: TimeWindows.of(1 hour)
  Result: (user_id, hour, revenue)
  Sink: Output topic

Example:
  Events:
    12:15 user_1 buys $50
    12:45 user_1 buys $30
    13:05 user_1 buys $20
  
  Window [12:00-13:00]: user_1 = $80
  Window [13:00-14:00]: user_1 = $20
```

---

### Pattern 2: Stream Joins

```
Scenario: Enrich purchase events with user profile

Topic 1: Purchase events (stream)
  { purchase_id: 123, user_id: 5, amount: $50, timestamp: T }
  
Topic 2: User profiles (stream or global store)
  { user_id: 5, name: "Alice", country: "US", tier: "premium", timestamp: T }
  
Kstream-Ktable join:
  Purchase event arrives
  Look up user_id in user profile table
  Emit joined result:
    { purchase_id: 123, user: Alice, amount: $50, tier: premium }
```

---

### Pattern 3: Stateful Processing

```
Use case: Detect fraud (multiple failed logins)

State: Map of user_id -> failed_login_count

Processing:
  Event 1: user_5 login failed
    State[user_5] = 1
    
  Event 2: user_5 login failed
    State[user_5] = 2
    
  Event 3: user_5 login failed
    State[user_5] = 3
    Emit alert: Fraud detected for user_5 (3 failures)
    Reset: State[user_5] = 0
    
State management:
  Persisted to local RocksDB
  Backed up to changelog topic
  On restart: Replayed from changelog to restore state
```

---

## Challenges and Solutions

### Challenge 1: Event Log Growth

```
Problem:
  30 days of events = 2.5 billion events
  Retention grows with time
  Recomputation from start gets slower
  
Solutions:
  1. Snapshots: Save state periodically
     Every 7 days: Take snapshot of current state
     Recompute from last snapshot (faster)
     
  2. Compacted topics: Keep only latest version per key
     User profile topic (compacted)
     Only latest profile for each user stored
     Reduces replay time
```

---

### Challenge 2: Schema Evolution

```
Problem:
  v1: { user_id, amount }
  v2: { user_id, amount, currency }
  
Old events missing "currency" field
Replaying fails due to schema mismatch

Solution: Schema versioning
  Explicitly version schema in event
  { version: 2, user_id, amount, currency }
  
  On replay:
    v1 events: Use defaults (currency = "USD")
    v2 events: Use as-is
```

---

### Challenge 3: Stateful Processor Recovery

```
Problem:
  State store has 1TB of data (sessions, aggregations)
  Processor crashes
  Restoring from changelog topic takes hours
  Downtime unacceptable
  
Solution: State store snapshots
  Every 10 minutes: Snapshot state to S3
  On recovery:
    1. Restore snapshot (10min)
    2. Replay changelog from snapshot point (5min)
    3. Total: 15min downtime vs hours
```

---

## Kappa Architecture Pitfalls

### Pitfall 1: Non-deterministic Processing

```
Broken code:
  result = current_timestamp()  // Uses NOW()
  
Problem:
  First run: T=2026-01-01 12:00:00
  Recompute: T=2026-07-23 02:00:00
  Different results from same events!
  
Solution:
  Use event timestamp (from events)
  result = event.timestamp
  
This ensures: Same events always produce same result
```

---

### Pitfall 2: Mutable Writes

```
Broken approach:
  Receive event: Purchase $100
  UPDATE users SET balance = balance + 100 WHERE user_id = 1
  
Problem on replay:
  First run: balance += $100 (balance = $100)
  Replay: balance += $100 again (balance = $200)
  Incorrect!
  
Solution:
  Write immutable results, not mutations
  INSERT INTO user_balance_by_hour VALUES (user_1, 2026-01-01T12:00, +$100)
  
  Query: SUM(amount) for time period
  Always consistent regardless of replay count
```

---

## When NOT to Use Kappa

```
Use Lambda if:
  1. You have heavy batch jobs (ML model training)
     Kappa not designed for model retraining
     
  2. You need point-in-time queries on historical state
     Event log doesn't store all historical states
     
  3. Data doesn't fit stream processing model
     E.g., image processing, large file analysis
     
  4. Recomputation latency is critical
     Kappa replay can be slow for 1 year of events
     
Otherwise: Kappa is simpler and preferred
```

---

## Related Fundamentals

- [Lambda Architecture](lambda-architecture.md) – Batch + stream hybrid approach
- [Stream Processing](../messaging-and-streaming/stream-processing.md) – Kappa processing basics
- [Exactly-Once Semantics](../messaging-and-streaming/exactly-once-semantics.md) – Ensuring correctness
- [Event Sourcing](../messaging-and-streaming/event-sourcing.md) – Event log as source of truth

---

**Status**: ✅ Complete. Covers Kappa model, recomputation, comparison with Lambda, patterns, challenges.
