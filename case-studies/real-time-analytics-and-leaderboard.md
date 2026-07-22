# Real-Time Analytics & Leaderboard

*Design a real-time leaderboard (top 100 users by score). Updated constantly, low latency, billions of events.*

## Problem Statement

Build a real-time leaderboard. 500M users, 1M score updates/sec. Top 100 users must be accurate within 1 second. Billions of historical events to compute on.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Score updates/sec** | 1M | High volume |
| **Leaderboard size** | Top 100 | Small but hot |
| **Historical events** | 1B+ | Batch processing |
| **Query latency** | P99 < 100ms | Real-time |

## Architecture

```
Event Stream (Kafka):
  User plays game → Emit {userId, score_change, timestamp}
  
Stream Processor (Flink, Spark Streaming):
  Consume events
  Aggregate per user: sum(score_change)
  Update leaderboard in real-time
  
Leaderboard Store (Redis):
  Sorted set: {userId: score}
  ZADD leaderboard userId score (atomic)
  ZREVRANGE leaderboard 0 99 (top 100)
  
Batch Recompute (Spark):
  Hourly/daily: Recompute from event log
  Validates streaming aggregations (catch bugs)
  Historical analytics
```

## Challenges

### Aggregation State

```
Stream processor maintains in-memory state:
  userId → current_score
  
On new event:
  current_score[userId] += event.score_change
  ZADD leaderboard userId current_score[userId]
  
On processor crash:
  Restore state from checkpoint (Kafka offset + saved state)
```

### Exactly-Once Semantics

```
Event: Player scores 100 points
  Processor: Increment score, update leaderboard
  Crash before commit

On restart:
  Kafka replays event (at-least-once)
  Score incremented twice if not idempotent
  
Solution: Deduplication by event_id + source_offset
  If seen this event_id before, skip (idempotent)
```

### Leaderboard Accuracy

```
Real-time leaderboard: Updated every 1 sec (from stream)
Batch recompute: Every hour (from full event log)

If they diverge:
  Stream bug (e.g., lost event)
  Alert + fallback to batch result
```

## Bottlenecks & Scaling

**Bottleneck**: Stream processor state management (1M users).

**Solution**:
- Partition stream by userId (each processor handles subset)
- Distributed state backend (external store for failover)
- Checkpointing to durable storage

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Stream + batch** | Stream only | Batch validates stream, catches bugs |
| **Redis sorted set** | Database | O(1) leaderboard access, fast updates |
| **At-least-once + dedup** | Exactly-once | Simpler, acceptable overhead |

---

**Status**: ✅ Complete. Shows stream processing, aggregation, fault tolerance.
