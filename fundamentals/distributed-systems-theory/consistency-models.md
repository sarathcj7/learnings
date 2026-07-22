# Consistency Models

## TL;DR

- **Strong consistency**: All reads see latest write (slow, coordinated)
- **Eventual consistency**: Reads eventually see writes (fast, asynchronous)
- **Read-after-write**: After write, subsequent reads see that write
- **Causal consistency**: Related operations see causal order
- **Monotonic consistency**: Processes see increasingly up-to-date values

## Strong Consistency (Linearizability)

All operations appear to execute atomically in total order. Latest write is always visible.

```
Write X=1 at t=0
Write X=2 at t=1
Read X at t=2  → 2 (always latest)
```

**Cost**: Coordination (locks, synchronous replication) = slow

**When**: Financial transactions, inventory (must be exact)

---

## Eventual Consistency

Reads eventually see writes, but may be stale temporarily.

```
Write X=1 to primary
Primary to replicas asynchronously
Read X from replica at t=1ms  → 0 (old value, stale)
Read X from replica at t=100ms  → 1 (new value, caught up)
```

**Cost**: Temporary staleness, complexity (conflicts possible)

**When**: Social media, cache, analytics (staleness tolerable)

---

## Read-After-Write Consistency

After client writes, their own reads see that write.

```
Client writes X=5
Client reads X  → 5 (sees own write, even if replicas lag)
Other clients may still see X=3 (eventual)
```

**Implementation**: Route client's reads to leader for a short time after write.

**When**: User-centric systems (user updates profile, sees their change immediately)

---

## Causal Consistency

If event B depends causally on A, all processes see A before B.

```
User writes comment (event A)
User replies to comment (event B, causally depends on A)

All processes see: Comment, then Reply (not Reply, then Comment)
```

**When**: Social media, messaging (maintain causal order for user understanding)

---

## Monotonic Consistency

Process never sees an older version (values only move forward).

```
Process reads X=3 at t=1
Process reads X=2 at t=2  → Violates monotonic (backward!)

Monotonic ensures: X values go 0 → 1 → 2 → 3 (never decrease)
```

**When**: Counters, leaderboards (never go backward)

---

## Spectrum

```
Strongest (Slowest): Strong Consistency
                     ↓
                Causal Consistency  
                     ↓
                Monotonic Consistency
                     ↓
Weakest (Fastest):   Eventual Consistency
```

**Trade-off**: Strength vs speed. Choose based on requirements.

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- "Consistency and Isolation in Databases" — Adya et al.

---

## Related Fundamentals

- [Transactions & Isolation Levels](../databases/transactions-and-isolation-levels.md) – Consistency in single database
- [CAP & PACELC](cap-and-pacelc.md) – Consistency in distributed systems
- [Consensus & Coordination](../consensus-and-coordination/) – How to enforce strong consistency
