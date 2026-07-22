# Distributed ACID: Consistency Levels in Distributed Systems

In a single database, ACID is straightforward. In distributed systems, the definitions blur. This file explores ACID properties across multiple machines, the trade-offs, and alternative consistency models.

---

## TL;DR

- **ACID in centralized DB**: Atomicity (all-or-nothing), Consistency (invariants hold), Isolation (concurrent txns don't interfere), Durability (committed data survives failure)
- **Distributed ACID challenges**: Network partitions, failures mid-transaction, lack of shared memory make guarantees hard/expensive
- **Consistency levels**: Strong (linearizability), Weak (eventual consistency), Tunable (multi-version concurrency control)
- **BASE model**: Basically Available (always responds), Soft state (state may change), Eventually consistent (convergence guaranteed)
- **Trade-off**: Strong consistency requires coordination (slow, blocking). Weak consistency is fast but requires app-level handling of conflicts

---

## ACID Properties in Distributed Context

### Atomicity (All-or-Nothing)

**Single DB**: Transaction commits entirely or not at all.

**Distributed**:
- **2PC/3PC**: Achieve atomicity across multiple databases with blocking (slow, reduces availability)
- **Saga pattern**: Accept weaker atomicity (eventual rollback via compensations), no blocking
- **Tradeoff**: True atomicity requires coordination; distributed systems often settle for "eventual atomicity"

```
Distributed bank transfer:
  Debit account in DB1, credit account in DB2

True atomicity (2PC):
  - If DB1 crashes after debit, DB2 credit rolls back
  - Never a dangling debit without credit
  - Cost: Blocking, high latency, reduced availability

Saga atomicity:
  - If DB1 credit fails, we issue a compensating debit transaction
  - Temporary inconsistency visible (money missing briefly)
  - Cost: Eventual consistency, compensating txn logic, retry complexity
```

### Consistency (Invariant Preservation)

**Single DB**: All ACID properties together ensure invariants hold.

**Distributed**:
- **Strong consistency**: Linearizability. Every read sees the latest write. Requires coordination across nodes.
- **Weak consistency**: Eventual consistency. Temporary disagreement among nodes allowed; nodes converge to same state over time.

```
Example: Bank account balance consistency

Strong consistency:
  After transfer completes, ALL nodes report same balance
  Cost: Replication and coordination overhead
  
Weak consistency:
  Some nodes may report stale balance for seconds/minutes
  Benefit: No replication coordination; accept temporary inconsistency
  App responsibility: Warn user "balance may be updated in moments"
```

### Isolation (Concurrent Txn Independence)

**Single DB**: 
- Isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable)
- Prevent dirty reads, non-repeatable reads, phantom reads

**Distributed**:
- **Local isolation**: Each node maintains its own isolation level (e.g., each DB uses Serializable)
- **Global isolation**: Hard to achieve. Concurrent transactions on different nodes can interleave
- **Snapshot Isolation**: Multi-version concurrency control (MVCC) — read from consistent snapshot, writes go to new version

```
Example: Inventory reservation

Local isolation (Serializable on each node):
  Node A: Txn1 reads product stock (100 units), Node B: Txn2 reads stock (100 units)
  But Txn1 and Txn2 are on different nodes, so they both see 100
  If both deduct, stock goes negative (-0 units across the system)
  
Global isolation (Serializable across nodes):
  Requires all nodes to order transactions globally (expensive, slow)
  
Snapshot isolation (MVCC):
  Txn1 and Txn2 both read from version T0 (100 units)
  If both commit, creates version T1 with stock 98
  Temporarily shows two versions; eventually converges to T1
```

### Durability (Permanent After Commit)

**Single DB**: Write to disk; survives crashes.

**Distributed**:
- **Single replica**: Durability ≈ single-node durability
- **Replicated**: Durability requires replication to N nodes before commit (N=3 common). Cost: higher latency (3x writes)
- **Chain replication**: Intermediate — replication pipeline (P → S1 → S2). Durable at S2, lower latency than waiting for all

---

## Consistency Models

### Strong Consistency (Linearizability)

Every read returns the most recent write. Equivalent to a single, atomic register.

```
Timeline:
  T0: Client writes X = 5 to Node A
  T1: Client reads X from Node B → returns 5

Guarantee: Node B's read reflects Node A's write (no stale data possible)

Implementation: 
  - Synchronous replication (write to all replicas before ACK)
  - Quorum reads/writes (wait for majority)
  - Consensus protocols (Raft, Paxos)
  
Cost: High latency, reduced availability if replicas fail
```

**When to use**: 
- Financial transactions where stale state is unacceptable
- Metadata changes (schema updates) that must be globally visible immediately

### Eventual Consistency

Nodes eventually converge to the same state. Temporarily, read can return stale data.

```
Timeline:
  T0: Client writes X = 5 to Node A
  T1: Client reads X from Node B → returns 3 (stale)
  T2: Node B syncs from Node A
  T3: Client reads X from Node B → returns 5

Guarantee: Eventually, all nodes see the latest write
Implementation: 
  - Asynchronous replication
  - Gossip protocols
  - Anti-entropy (periodic sync)
  
Benefit: Low latency writes (no waiting for replication), high availability (reads served even if replicas down)
```

**When to use**:
- User profiles (eventual updates OK)
- Social feed (temporary inconsistency acceptable)
- Analytics data

### Causal Consistency

Related events are seen in order. Unrelated events may be out of order.

```
Scenario:
  User U posts: "Hello"
  User V comments: "Hi"

Causal guarantee:
  All readers see post before comment
  Post "happened before" comment (causal dependency)
  
Unrelated events:
  User W posts: "Goodbye"
  User X posts: "Goodnight"
  May be seen in either order (no causal dependency)
```

### Read-Your-Writes (Session Consistency)

Client's own writes are visible to client's subsequent reads. Other clients may not see writes yet.

```
Example:
  User updates profile name to "Alice"
  User refreshes profile → sees "Alice"
  Guaranteed even if write hasn't replicated everywhere
  
Other users might still see old name "Bob" temporarily
```

---

## BASE Model (vs. ACID)

ACID is strong. BASE is weak but more available.

### BASE Properties

- **Basically Available**: System responds to requests even during failures (weaker than ACID Availability)
- **Soft state**: State may change without input (replication in progress, replicas reaching consistency)
- **Eventually consistent**: Convergence guaranteed over time

```
ACID: Immediate consistency, reduced availability
BASE: Weak consistency, high availability

Example: DNS
  - Change TTL on A-record
  - ACID: All DNS servers update immediately (strong consistency)
  - BASE: Servers update asynchronously; converge within TTL (eventual consistency, high availability)
```

---

## Practical Consistency Strategies

### Quorum-Based Consistency

Read/write from W nodes (out of N total replicas). If W + R > N, reads see all writes.

```
Example: N=5 replicas
  W=3 (write to 3 replicas before ACK)
  R=3 (read from 3 replicas, return latest)
  
  Scenario:
    Replica 1, 2, 3 have X=5
    Replica 4, 5 have X=3 (stale)
    
  Read from 3: Likely gets X=5 from majority
  Write updates minority (4, 5 eventually catch up via anti-entropy)
```

**Trade-off**: Higher W/R → stronger consistency but slower. Lower W/R → faster but weaker.

### Version Vectors & Conflict Detection

Track causal dependencies to detect concurrent writes.

```
Write to A: [A:1, B:0]
Write to B: [A:0, B:1]

Both writes are concurrent (neither happened-before the other).
System detects conflict, may ask user to resolve or pick latest.

Replica A: [A:2, B:1] → Happened after both writes (causal successor)
```

### Write-Through with Eventual Consistency

Combine strong local consistency (single-node write) with eventual global consistency.

```
User updates profile:
  1. Update local database (strong consistency)
  2. Enqueue replication task
  3. Return to user (user sees update)
  4. Async: Replicate to other regions (eventual consistency)
```

---

## Trade-offs Summary

| Property | Strong Consistency | Eventual Consistency |
|----------|-------------------|----------------------|
| **Latency** | High (wait for sync) | Low (immediate response) |
| **Availability** | Lower (failures block) | Higher (always responds) |
| **Complexity** | Lower (app sees single state) | Higher (app handles stale data) |
| **Scalability** | Limited (coordination overhead) | High (independent replicas) |
| **Cost** | Higher (sync replication) | Lower (async replication) |

---

## Real-World Examples

### Banking (Strong Consistency)

Account balance, money transfer — must be strongly consistent to prevent double-spending, lost transfers.

Implementation:
- 2PC or distributed consensus (Raft)
- Quorum writes/reads with high W/R values

### Social Media (Eventual Consistency)

User follows, post likes, comments — can tolerate eventual consistency for better availability.

Implementation:
- Write-back caching (local update, async replication)
- Multi-region replication with eventual sync
- Conflict-free replicated data types (CRDTs) for automatic merging

### E-Commerce (Tunable)

- Inventory: Strong consistency (prevent overselling)
- Recommendations: Eventual consistency (stale data acceptable)
- Checkout: Strong consistency (payment, final order)

---

## Related Fundamentals

- **[Two-Phase Commit](two-phase-commit.md)**: Protocol for achieving distributed ACID
- **[Saga Pattern](saga-pattern.md)**: Weak atomicity alternative to 2PC
- **[Consensus & Coordination](../consensus-and-coordination/)**: Raft, Paxos for distributed agreement
- **[Databases - Transactions & Isolation](../databases/transactions-and-isolation-levels.md)**: Single-node ACID foundation

---

## Key Takeaways

1. **ACID in distributed systems is costly**: Strong consistency requires coordination (blocking, high latency).
2. **Consistency exists on a spectrum**: Linearizable → Causal → Eventual. Choose based on requirements.
3. **BASE model embraces eventual consistency**: Higher availability, lower latency, more complexity.
4. **Quorum-based reads/writes** allow tunable consistency without full 2PC overhead.
5. **App-level handling** of conflicts and stale data is key to building eventual-consistency systems.

---

**Study Tips**

- Compare ACID (single DB) vs. ACID (distributed) side-by-side; note the costs and trade-offs.
- Trace a quorum read/write for N=5, W=3, R=3. Which reads guarantee seeing all writes?
- Design a system with both strong (inventory) and eventual (recommendations) consistency tiers.
- Debate: 2PC vs. Saga for a financial institution. What are the trade-offs?

---

**Status**: ✅ Complete
