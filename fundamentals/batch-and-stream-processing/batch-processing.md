# Batch Processing

Processing large datasets in discrete batches. Hadoop, Spark, MapReduce.

---

## TL;DR

- **Batch**: Process all data periodically (hourly, daily)
- **MapReduce**: Map (transform) → Shuffle → Reduce (aggregate)
- **Spark**: In-memory processing (10x faster than Hadoop)
- **Latency**: Minutes to hours (not real-time)
- **Use**: Analytics, reports, ETL jobs

---

## MapReduce Model

```
Input: 1TB log file

Map phase:
  Parse each line
  Extract: (user_id, revenue)
  Emit: (user_id, revenue)

Shuffle:
  Group by user_id
  user_1: [100, 50, 200]
  user_2: [150, 75]

Reduce phase:
  Sum per user
  user_1: 350
  user_2: 225

Output: User revenue totals
```

---

## Hadoop vs Spark

| Aspect | Hadoop | Spark |
|---|---|---|
| **Speed** | Disk I/O (slow) | In-memory (10x faster) |
| **Cost** | Cheaper | More expensive (RAM) |
| **Latency** | Hours | Minutes |
| **Learning** | Complex | Simpler API |

---

## Related Fundamentals

- [Stream Processing](../messaging-and-streaming/stream-processing.md) – Real-time alternative
- [Capacity Planning](../capacity-planning-and-estimation/) – Sizing clusters

---

**Status**: ✅ Complete.

