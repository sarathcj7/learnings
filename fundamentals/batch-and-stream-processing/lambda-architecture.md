# Lambda Architecture

Batch + stream layers combined for accuracy and speed.

---

## TL;DR

- **Lambda**: Batch (accurate) + Stream (fast) + Serving (merged results)
- **Speed layer**: Stream processing (minutes)
- **Batch layer**: Batch jobs (nightly, accurate)
- **Serving**: Combine both for best of both
- **Trade-off**: Complexity for accuracy + speed

---

## Architecture

```
Incoming data
├─ Speed layer: Kafka → Flink (minute-level latency)
│  Results: Approximate, real-time
├─ Batch layer: Spark job nightly (hour-level latency)
│  Results: Accurate, complete
└─ Serving layer: Merge both
   If < 1 hour old: Use stream results (fast)
   If > 1 hour old: Wait for batch (accurate)

Result:
  Recent: Fast + approximate
  Old: Accurate (batch recomputed)
```

---

## Example: Leaderboard

```
Speed layer:
  Stream recent scores
  Update every 1 minute
  Fast but might miss late submissions

Batch layer:
  Recompute daily (after cutoff)
  Include all submissions
  Accurate and complete

Serving:
  Until midnight: Use speed layer (fast)
  After midnight: Switch to batch results (accurate)
```

---

**Status**: ✅ Complete.

