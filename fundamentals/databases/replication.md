# Replication

## TL;DR

- **Leader-Follower**: One primary (writes), N followers (replicate writes). Simplest, most common.
- **Multi-Leader**: Multiple primaries, asynchronous replication. Handles region failover, write conflicts.
- **Leaderless**: All nodes equal, quorum-based reads/writes. No single point of failure, complex.
- **Replication lag**: Followers lag behind leader by ms to seconds. Causes stale reads.
- **Consistency models**: Depends on sync strategy. Synchronous = wait, Asynchronous = fast but risky.

## Leader-Follower (Primary-Replica)

```
Write: Client → Leader → Replicate to Followers → ACK

Leader (Primary):
  - Handles all writes
  - Replicates to followers asynchronously (by default)
  
Follower (Replica):
  - Read-only (can handle SELECT queries)
  - Lag behind leader by few ms to seconds
  
Failover:
  - Leader fails → Promote a follower to leader
  - Clients redirect to new leader
  - Other followers replicate from new leader
```

### Replication Lag Problem

```
t=0: Client writes "age=30" to leader
     Leader has age=30
     Followers still have age=25

t=100ms: Follower replicates the write
         Now has age=30

Problem: If client reads from follower during t=0-100ms, sees stale age=25
```

### Synchronous vs Asynchronous

| Type | Pros | Cons |
|---|---|---|
| **Synchronous** | Strong consistency (leader waits for replicas) | Slow writes (wait for replication) |
| **Asynchronous** | Fast writes (leader doesn't wait) | Risk of data loss (leader crash = unreplicated writes lost) |
| **Semi-sync** | Compromise: Leader waits for N replicas | Faster than full sync, safer than async |

---

## Multi-Leader (Multi-Primary)

Multiple leaders accept writes, asynchronously replicate to each other.

```
Region 1: Leader A (accepts writes)
Region 2: Leader B (accepts writes)

Write to A → B sees it after replication lag
Write to B → A sees it after replication lag

Advantage: Regional failover (no single point of failure)
Disadvantage: Write conflicts (A and B write same key differently)
```

### Conflict Resolution

When both leaders write the same key:

```
t=0: Leader A: SET color='red'
t=0: Leader B: SET color='blue' (concurrent write)

Replication arrives:
  A receives "blue" from B
  B receives "red" from A

Which is correct?
  Options: Last-write-wins (timestamp), merge (color='red,blue'), application logic
```

**Mitigation**:
- Partition writes (each user writes to only one region)
- Avoid conflicts (hard in practice)
- Accept eventual consistency (merge conflicts later)

---

## Leaderless (Dynamo-style)

All nodes equal. No single leader = no single point of failure.

```
Write: Client sends write to all N nodes
       Waits for W nodes to confirm
       If W = N: Synchronous, all copies consistent
       If W = 1: Asynchronous, risky but fast

Read: Client reads from R nodes
      If R = N: Guaranteed freshness (read latest)
      If R = 1: Risk of stale data (read lagged node)
      
Write + Read: If W + R > N, quorum overlap ensures freshness
              Example: N=3, W=2, R=2
              Write waits for 2 nodes, read from 2 nodes
              At least 1 node in common → read sees latest write
```

### Read Repair

When client reads from lagged node:

```
Client reads from node A (has version 5)
Client also reads from node B (has version 7)
Client detects A is stale, sends version 7 back to A
A updates to version 7 (read repair)
```

---

## Replication Setup

### Statement-Based

Leader logs every SQL statement, replicas execute.

```
Leader: UPDATE users SET age = age + 1 WHERE id > 100
Replica: Executes same SQL

Problem: Non-deterministic functions (NOW(), RAND()) produce different results on replicas
Result: Replicas diverge from leader
```

### Write-Ahead Log (WAL)

Leader logs every write operation (INSERT, UPDATE, DELETE), replicas apply.

```
Leader: INSERT (id=1, name='Alice')
  → Log write to WAL
  → Apply to data
  → Send WAL to replicas

Replica: Receive WAL entry
  → Apply exact same write
  → Data matches leader
```

**Downside**: WAL is low-level (database engine specific), hard to replicate across different engines.

### Logical Log (Row-Based)

Log rows that changed (new values), replicas apply.

```
Leader: UPDATE users SET age=30 WHERE id=1
  → Log: { id: 1, name: 'Alice', age: 30 }
  → Send to replicas

Replica: Apply { id: 1, name: 'Alice', age: 30 }
  → Works even if engine is different (MySQL → PostgreSQL possible)
```

---

## Failover

When leader crashes:

```
1. Detection: Followers detect leader down (missed heartbeat)
2. Election: Followers vote to promote one (highest LSN/version)
3. Reconfiguration: New leader notifies all followers
4. Client redirect: DNS or load balancer redirects to new leader

RTO (Recovery Time Objective): 30-60 seconds typical
RPO (Recovery Point Objective): Few seconds (async replication lag)
```

### Challenges

- **Split brain**: Network partition causes two leaders elected
- **Data loss**: Committed writes on old leader not on new leader
- **Cascading failures**: New leader also slow/fails

---

## Production Considerations

1. **Replication lag monitoring**: Alert if lag > 10s
2. **Read-after-write consistency**: After write, client reads from leader (not follower) for a short time
3. **Monitoring replication**: Track bytes replicated, lag, catch-up speed
4. **Backups**: Replicas are not backups (both can fail together). Use separate backup strategy.
5. **Cascading replication**: Follower → Follower → Follower chain increases lag (avoid, use star topology)

---

## References

- "Designing Data-Intensive Applications" — Kleppmann, Chapter 5
- PostgreSQL documentation: "High Availability, Load Balancing, and Replication"

---

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/) – Failover election algorithms
- [Sharding & Partitioning](sharding-and-partitioning.md) – Replication + sharding = scale reads and writes
- [Distributed Transactions](../distributed-transactions/) – Cross-replica consistency
