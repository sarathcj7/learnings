# Quorum Systems

## TL;DR

- **Quorum**: Majority of nodes. Write to W, read from R, where W+R > N ensures overlap
- **Formula**: N nodes, W writes needed, R reads needed. W+R > N → read sees latest write
- **Sloppy quorum**: Allow non-quorum writes (faster, less reliable) with read repair
- **Trade-off**: Smaller quorum = faster but lower consistency. Larger = slower but stronger

## Quorum Math

```
N = 3 nodes
W = 2 (write to 2 nodes)
R = 2 (read from 2 nodes)

W + R = 4 > N = 3  → Guaranteed overlap
At least one node in read quorum has latest write
```

### Overlap Guarantee

```
Write quorum: Nodes {A, B}  (write to A and B)
Read quorum: Nodes {B, C}   (read from B and C)

Overlap: B
Result: B has latest write, read quorum includes B → read sees latest
```

---

## Quorum Configurations

### Strong Quorum (W=N, R=1)
- Write waits for all N nodes (slow but safe)
- Read from 1 node (fast, safe because all have latest)
- **Cost**: Write latency is max(all nodes)
- **Use**: Write-seldom systems

### Weak Quorum (W=1, R=N)
- Write to 1 node (fast but risky)
- Read from all N nodes (slow, but reconcile stale values)
- **Cost**: Read latency is max(all nodes)
- **Use**: Read-seldom systems

### Balanced (W=R=N/2+1)
- Both reads and writes wait for majority
- **Cost**: Balanced latency
- **Use**: Most systems (Netflix, Dynamo)

---

## Read Repair & Anti-Entropy

When read encounters stale node:

```
Read from {A: v2, B: v3, C: v3}  // A is stale

Return v3 (majority)
Repair A: Send v3 back to A (write repair)
```

**Merkle trees**: For bulk anti-entropy, compare hashes of data ranges between nodes, sync differences.

---

## Sloppy Quorum

Write to any N nodes (not strict majority), combine with read repair.

```
Strict quorum: W=2 out of 3 nodes
Sloppy quorum: Write to 3 fastest nodes (might not be majority if some are down)

Then read repair brings stale nodes up to date
```

**Advantage**: Works when some nodes slow/down (higher availability)  
**Disadvantage**: Weaker consistency guarantee

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- Dynamo paper: "Quorum Consistency"

---

## Related Fundamentals

- [CAP & PACELC](cap-and-pacelc.md) – Quorum affects CAP choice
- [Replication](../databases/replication.md) – Quorum used in leaderless (Dynamo) replication
- [Consensus & Coordination](../consensus-and-coordination/raft.md) – Raft uses quorum
