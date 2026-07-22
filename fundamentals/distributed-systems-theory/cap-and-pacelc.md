# CAP & PACELC

## TL;DR

- **CAP theorem**: In distributed systems, choose 2 of 3: Consistency, Availability, Partition tolerance
  - CA: SQL (not partition tolerant)
  - CP: Databases that stop on partition (MongoDB, strong consistency mode)
  - AP: Dynamo-style, always available but eventual consistency
- **PACELC**: Extension of CAP. PAC: Pick 2. Else (no partition): Choose Latency vs Consistency
- **The truth**: You can't ignore partition tolerance (network partitions happen), so really it's CP vs AP with latency/consistency tradeoff

## CAP Theorem

Three guarantees:

### Consistency (C)
All nodes see the same data. Strong consistency.

```
Write X=5 to node 1
Read X from node 2 immediately → 5 (not 3)
```

### Availability (A)
System always responds (even with failures).

```
Node 1 down? Node 2 still responds
Every request gets a response (not error or timeout)
```

### Partition Tolerance (P)
System works despite network partitions (nodes can't communicate).

```
Network split: Nodes A and B can't talk
System continues to function (P = yes)
System halts (P = no)
```

### The Three Combinations

**CA**: Ignore partitions, prioritize consistency + availability
- Trade: Can't survive network split (rare, but happens)
- Example: Traditional SQL database (single data center)

**CP**: Consistency + partition tolerance
- Trade: Sacrifice availability (some nodes may not respond during partition)
- Example: MongoDB with strong consistency, stops writes during partition

**AP**: Availability + partition tolerance  
- Trade: Sacrifice strong consistency (eventual consistency)
- Example: DynamoDB, Cassandra (always respond, but might be stale)

---

## PACELC Extension

CAP is incomplete. It ignores the normal case (no partition).

```
If Partition exists: Choose Consistency or Availability (CAP)
Else (no partition): Choose Latency or Consistency

PACELC: P(artition) A(vailability) C(onsistency) E(lse) L(atency) C(onsistency)
```

### Else Case (No Partition)

```
Network is healthy (no partition)

CP system: 
  Choices: Write to all replicas (strong consistency, slow)
           or Write to one (fast, but risky if partition later)
  Trade: Latency vs Consistency

Example: PostgreSQL with synchronous replication
  - Wait for all replicas (strong consistency, slow)
  vs Async replication (fast, risky)
```

---

## What Does It Really Mean?

### CAP is Misunderstood

"Pick 2" is misleading. You can't ignore P (partitions happen).

**Reality**: You're choosing between:
- **CP (Consistency Priority)**: Tolerate losing availability on partition (stop processing)
- **AP (Availability Priority)**: Tolerate staleness on partition (keep serving)

### The Real Decision Tree

```
1. Partitions will happen (P is inevitable)
2. On partition, choose:
   - CP: Stop / refuse writes (consistent but unavailable)
   - AP: Keep serving stale (available but inconsistent)

3. No partition, choose:
   - Latency: Use async replication (fast but risky)
   - Consistency: Use sync replication (slow but safe)
```

---

## Examples

### CP System: MongoDB (Strong Consistency Mode)

```
Primary leader receives write
Waits for majority replicas to acknowledge
Only then returns success

On partition: If primary isolated, it stops accepting writes (consistency, not availability)
```

**CAP**: CP  
**PACELC**: PC (prioritizes consistency in both cases)

### AP System: DynamoDB / Cassandra

```
Write to any node (you choose quorum)
Returns immediately (might not be replicated yet)
Other nodes get update asynchronously

On partition: Different nodes might have different versions (eventual consistency)
```

**CAP**: AP  
**PACELC**: AE-LA (availability in both, then latency tradeoff)

---

## Production Implications

1. **Accept that partitions happen**: Design for failure
2. **Plan for Else**: No partition is the normal case, so latency/consistency tradeoff matters more than CAP choice
3. **Hybrid**: Use different systems for different guarantees
   - Transactions: CP (accept slow)
   - Cache: AP (accept stale)
4. **Detectingpartitions**: How do you know you're in a partition? (Timeout, health checks)

---

## References

- "CAP Theorem" — Brewer et al.
- "CAP Twelve Years Later" — Brewer (reframe of CAP)
- PACELC paper

---

## Related Fundamentals

- [Consistency Models](consistency-models.md) – What "consistency" actually means
- [Consensus & Coordination](../consensus-and-coordination/) – How to achieve CP
- [Replication](../databases/replication.md) – Sync vs async affects CAP/PACELC choice
