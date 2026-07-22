# Time, Clocks & Ordering

## TL;DR

- **Problem**: Distributed systems have no global clock. How do we order events?
- **Lamport timestamps**: Logical clock, simple, works for causal order
- **Vector clocks**: Better than Lamport, detects causality, higher overhead
- **Hybrid logical clocks (HLC)**: Best of both, physical + logical component

## Lamport Timestamps

Simple logical clock. Each process increments counter on every operation, includes in message.

```
Process A: op1 (ts=1), send msg to B (ts=2)
Process B: receive msg (ts=2), op2 (ts=3)

Order: A.op1 (ts=1) → A.send (ts=2) → B.receive (ts=2) → B.op2 (ts=3)

Property: If A → B (A caused B), then ts(A) < ts(B)
Limitation: If ts(A) < ts(B), doesn't mean A caused B (might be unrelated)
```

**Pros**: Simple, single number per event, no overhead  
**Cons**: Can't detect if two events are concurrent (unrelated) or causal

---

## Vector Clocks

Each process tracks logical time of all other processes.

```
Process A: [A:1, B:0, C:0]
Send to B:  [A:1, B:0, C:0]
Process B:  Receive [A:1, B:1, C:0]
            Increment B: [A:1, B:2, C:0]

Compare two events:
  Event1: [A:2, B:1, C:0]
  Event2: [A:1, B:2, C:0]
  
  Neither dominates (A higher in Event1, B higher in Event2)
  → Events are concurrent (unrelated)
  
Compare:
  Event1: [A:2, B:2, C:1]
  Event2: [A:1, B:1, C:0]
  
  Event1 > Event2 in all components
  → Event1 causally depends on Event2
```

**Pros**: Detect causality and concurrency  
**Cons**: Vector grows with number of processes (O(N) storage per event)

---

## Hybrid Logical Clocks (HLC)

Combines physical time (wall clock) + logical component. Better than both.

```
HLC = (physical_timestamp, logical_counter)

HLC.1 = (1626020400.500, 0)  // 500ms, counter=0
HLC.2 = (1626020400.600, 0)  // 600ms, counter=0 (time advanced, reset counter)
HLC.3 = (1626020400.600, 1)  // Same physical time, counter incremented

Advantage: Uses physical time (no large vectors), detects causality (like vector clocks)
Used in: CockroachDB, Spanner for global consistency
```

---

## Google Spanner: TrueTime

Spanner uses GPS/atomic clocks to get physical time with bounded uncertainty.

```
TrueTime API: [earliest, latest]  // Time interval with uncertainty bound

Spanner uses this for:
  - Assign timestamps to transactions
  - Wait for uncertainty window to close before committing
  
Guarantee: If A.commit < B.start, then A.ts < B.ts (strong consistency across globe!)
```

**Cost**: Specialized hardware (GPS + atomic clocks)

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- "The Google Spanner Whitepaper" — Corbett et al.

---

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/) – Ordering in leader election (Raft)
- [Quorum Systems](quorum-systems.md) – Quorum + timestamps for ordering
