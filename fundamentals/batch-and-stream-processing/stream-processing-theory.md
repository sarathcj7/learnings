# Stream Processing Theory

Fundamentals of event processing: stateful operations, windowing, watermarks, triggering, triggering modes.

---

## TL;DR

- **Event processing**: Process unbounded streams of events (vs finite batches)
- **Stateless**: Map, filter operations (embarrassingly parallel)
- **Stateful**: Aggregation, joins require coordination and state management
- **Windowing**: Divide unbounded stream into finite chunks (tumbling, sliding, session)
- **Watermarks**: Marker indicating "all events before time X have arrived" (handles late data)
- **Triggering**: When to emit results (early, on-time, late)
- **Time semantics**: Event time (when happened) vs processing time (when processed)

---

## Unbounded vs Bounded Data

### Bounded Data (Batch Processing)

```
Dataset: 1TB of historical log files

Characteristics:
  ✓ Known size (exactly 1TB)
  ✓ Complete (all data available upfront)
  ✓ Finite (will finish processing)
  ✓ Random access (can read any part)
  
Processing:
  Read entire 1TB → Process all records → Write results
  Duration: Known upfront (1 hour)
  Memory: Can plan buffer sizes (know total data)
  
Example: Daily analytics report (process all events from midnight-1am)
```

### Unbounded Data (Stream Processing)

```
Dataset: Credit card transaction stream

Characteristics:
  ✗ Unknown size (events arrive forever)
  ✗ Incomplete (always more events coming)
  ✗ Infinite (processing never finishes)
  ✗ Sequential access only (can't rewind to past)
  
Processing:
  Events arrive continuously → Process immediately → Results updated continuously
  Duration: Unknown (stream runs forever)
  Memory: Must be bounded (can't buffer unlimited events)
  
Example: Real-time fraud detection (process transactions as they arrive)
```

---

## Time Semantics

Two notions of time in stream processing.

### Event Time

Time the event actually occurred (when transaction happened).

```
Transaction details:
  eventId: 12345
  timestamp: 2026-07-23T10:05:30  ← Event time
  amount: $99.99
  
Event time determines membership in time windows:
  Window [10:00-10:05]: Events with timestamp 10:00-10:05
  Event with timestamp 10:03 belongs to [10:00-10:05] window
  Event with timestamp 10:07 belongs to [10:05-10:10] window
  
Challenges:
  - Out-of-order: Event with timestamp 10:03 arrives after 10:07
  - Late arrivals: Event with timestamp 09:50 arrives at 10:20
  - Clock skew: Different servers have different clocks
```

### Processing Time

Time the event is actually processed by system.

```
Event with timestamp 10:05:30 arrives at processor

Processing time:
  Arrival time: 10:05:35 (5 seconds after event)
  Processing time: 10:05:37 (2 seconds to process)
  
Window assignment (processing time):
  Window [10:05-10:10]: Events processed during 10:05-10:10
  Event processed at 10:05:37 goes to [10:05-10:10] window
  
Problem: Unreliable for business logic
  "Count transactions per 5-minute period"
  Using processing time: Arbitrary (depends on network latency)
  Using event time: Correct (reflects actual business periods)
  
Use case: Monitoring systems (when did alert trigger?)
```

---

## Windowing Strategies

Divide unbounded stream into bounded chunks.

### Tumbling Window

Non-overlapping fixed-size windows.

```
Window size: 5 minutes
Window boundaries: [0-5), [5-10), [10-15), ...

Events by event time:
  [0:00] Event A
  [1:30] Event B
  [3:45] Event C
  [5:15] Event D
  [7:20] Event E
  
Windowing:
  Window [0:00-5:00): Events A, B, C
  Window [5:00-10:00): Events D, E
  
Query: "Count events per 5-minute period"
  Result: [0-5)=3, [5-10)=2
```

### Sliding Window

Overlapping windows with fixed step.

```
Window size: 10 minutes
Step size: 5 minutes (slide every 5 minutes)
Windows: [0-10), [5-15), [10-20), [15-25), ...

Events by event time:
  [2:00] Event A
  [7:00] Event B
  [12:00] Event C
  [17:00] Event D
  
Windowing:
  Window [0-10): Events A (counted once)
  Window [5-15): Events A, B, C
  Window [10-20): Events B, C, D
  Window [15-25): Events D
  
Query: "Rolling 10-minute average latency"
  Compute at each step: Average of all events in window
  Emit result every 5 minutes
  
Overhead: Windows overlap (more computation)
```

### Session Window

Variable-size windows based on activity.

```
Session timeout: 30 minutes (inactivity ends session)

User events:
  [10:00] Click
  [10:05] View page (5 min since last event)
  [10:10] Click
  (30 minutes gap - session ends)
  [10:41] Click (new session starts)
  [10:45] Purchase
  
Sessions:
  Session 1: [10:00-10:10] (3 events)
  Session 2: [10:41-10:45] (2 events)
  
Query: "Count events per user session"
  User 123 Session 1: 3 events
  User 123 Session 2: 2 events
  
Use case: User behavior analysis (understand user activity patterns)
```

---

## Watermarks

Signal indicating "all data before this time has arrived."

### The Problem: Late Data

```
Stream of events with event time:

Ideal scenario (in-order arrival):
  t=10:00 Event with timestamp 10:00 arrives
  t=10:01 Event with timestamp 10:01 arrives
  t=10:02 Event with timestamp 10:02 arrives
  
Window [10:00-10:02]: Complete, emit result

Reality (out-of-order arrival):
  t=10:00 Event with timestamp 10:00 arrives
  t=10:01 Event with timestamp 10:02 arrives
  t=10:02 Event with timestamp 10:01 arrives (LATE!)
  t=10:03 Event with timestamp 10:05 arrives
  
Problem:
  At t=10:02, did window [10:00-10:02] have all events?
  No! Event 10:01 arrived late at t=10:02
  Should we wait forever? (indecision → no results)
```

### Watermark Solution

```
Watermark = "All events with timestamp < X have arrived (with high confidence)"

Watermark progression:
  t=10:00: Emit watermark(10:00) → All events < 10:00 arrived
  t=10:01: Emit watermark(10:01) → All events < 10:01 arrived
  t=10:02: Emit watermark(10:02) → All events < 10:02 arrived
  t=10:05: Emit watermark(10:05) → All events < 10:05 arrived

Window completion:
  Window [10:00-10:02]: When watermark reaches 10:02, emit result
  Guarantee: All events with timestamp [10:00-10:02] have been processed
  
Late event handling:
  Event with timestamp 10:01 arrives at t=10:03 (after watermark 10:02)
  Action: Drop (too late) or update result (allowed grace period)
  Grace period: Allow 30 seconds of late data
  
Watermark with grace period:
  Window [10:00-10:02]: Emit at watermark 10:02, re-emit if late data within 30s
```

---

## Triggering Modes

When to emit results as data arrives and updates.

### Trigger: On Watermark

Emit result when watermark passes window end.

```
Window [10:00-10:02]

Result emission:
  t=10:00: Watermark 10:00 → Window not complete
  t=10:01: Watermark 10:01 → Window not complete
  t=10:02: Watermark 10:02 → Emit result (all data received)
  t=10:02.5: Late event arrives → Drop (result already emitted)
  
Characteristics:
  ✓ Correct (all data considered)
  ✗ Late result (wait until watermark, adds latency)
  
Example: Daily analytics (wait until end of day to emit report)
```

### Trigger: On Processing Time

Emit result after X seconds of processing time.

```
Window [10:00-10:02]

Result emission:
  Window opens at t=10:00
  Every 10 seconds: Emit intermediate result
  
  t=10:00-10:10: Emit result (1 day left)
  t=10:10-10:20: Emit result (3 events added)
  t=10:20-10:30: Emit result (2 events added)
  t=10:30: Watermark 10:02 → Emit final result
  
Characteristics:
  ✓ Quick results (emit every 10 seconds, early feedback)
  ✗ Inaccurate (may change when late data arrives)
  ✗ Variable latency (depends on processing time, not event time)
  
Use case: Dashboards (show best-guess results, update as data arrives)
```

### Trigger: Early + On-Time + Late (Composite)

Combine multiple triggers for flexibility.

```
Window [10:00-10:02]

1. Early trigger: Every 5 seconds (processing time)
   t=10:05: Emit early result (5 events so far)
   t=10:10: Emit early result (8 events)
   t=10:15: Emit early result (12 events)
   
2. On-time trigger: When watermark passes 10:02
   t=10:22: Watermark 10:02 → Emit on-time result (15 events final)
   
3. Late trigger: After watermark, if late data arrives
   t=10:25: Event 10:01 arrives (late, but within grace period)
   t=10:25: Re-emit late result (16 events)

Result stream:
  5 early results (updated every 5 seconds)
  1 on-time result (final, watermark passed)
  1 late result (unexpected late event, grace period triggered)
  Total: 7 updates for single window
```

---

## Stateful Operations

Operations requiring state management.

### Aggregation (COUNT, SUM)

```
Stream of purchase events:

Event: (userId, amount, timestamp)
Operation: "Sum purchases per user per minute"

State required:
  userAggregates = Map<userId, Map<minute, sum>>
  
  Event (user=123, amount=50, time=10:05):
    1. Determine window: 10:05 → [10:05-10:06) minute
    2. Lookup state: userAggregates[123][10:05-10:06]
    3. Accumulate: previous_sum + 50
    4. Update state: userAggregates[123][10:05-10:06] = new_sum
    5. On watermark: Emit final sum
    
Memory requirement: O(distinct_users × minutes_buffered)
  100k users × 24 hours = 2.4M state entries = ~500MB
```

### Join Streams

```
Stream A: Clicks (userId, timestamp, url)
Stream B: User profiles (userId, timestamp, name, country)

Operation: "Enrich clicks with user profile"

State required:
  profiles = Map<userId, (name, country, timestamp)>
  
  Event from Stream A (userId=123, url="/login"):
    1. Lookup state: profiles[123]
    2. Found: (name="Alice", country="USA", timestamp=10:05)
    3. Emit enriched: (userId=123, url="/login", name="Alice", country="USA")
    
  Event from Stream B (userId=123, name="Alice!", timestamp=10:10):
    1. Update state: profiles[123] = ("Alice!", "USA", 10:10)
    2. Any future clicks from user 123 use new name
    
State management:
  Keep recent profiles in state
  Evict old profiles (no longer relevant)
  Join latency: <100ms (local state lookup)
```

---

## Production Considerations

1. **State size**: Monitor memory usage, implement state eviction policies
2. **Exactly-once semantics**: Use transactional writes (Flink checkpoints)
3. **Watermark accuracy**: Model expected lateness, set grace period appropriately
4. **Backpressure**: Monitor lag (producer vs consumer offset difference)
5. **Late data policy**: Decide: drop, update, or store separately
6. **Monitoring**: Track watermark lag, state size, processing latency

---

## Related Fundamentals

- [Stream Processing (Frameworks)](../messaging-and-streaming/stream-processing.md) – Implementation details
- [Batch Processing](batch-processing.md) – Alternative to streaming
- [Kafka Architecture](../messaging-and-streaming/kafka-architecture.md) – Event transport layer

---

**Status**: ✅ Complete. Covers event/processing time, windowing, watermarks, triggering, stateful operations.
