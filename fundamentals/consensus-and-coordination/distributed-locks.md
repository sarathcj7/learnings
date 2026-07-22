# Distributed Locks

## TL;DR

- **Problem**: Ensure only one process modifies shared resource at a time (distributed)
- **Lease-based**: Lock expires after timeout, auto-released
- **Fencing tokens**: Unique token per lock, server rejects old tokens (prevents stale writes)
- **Chubby/Zookeeper**: Production locks using consensus algorithms
- **Redlock controversy**: Redis Redlock algorithm has known issues

## Lock Mechanics

```
Process A: LOCK resource="user:123"
          Returns: lock_id=abc123, expire_at=t+30s

Process A: Modify user:123 (safely locked)

Process A: UNLOCK resource="user:123"
          Releases lock

Process B: Can now acquire lock on user:123
```

## Lease-Based Locks

Lock expires automatically (crash-safe).

```
GET lock("db_connection")
  → {holder: ProcessA, expiry: t+10s}

Process A crashes at t=5s

At t=11s:
  Lock has expired
  Process B: GET lock → No holder, can acquire
  
Benefit: Dead process doesn't hold lock forever
```

## Fencing Tokens (Safety)

Lock includes unique token. Server rejects old tokens.

```
Process A: LOCK resource → token=42, expire_at=t+30s
Process A: WRITE value=100 WITH token=42
          Server applies (token matches current)

Process A: GC pause 40s, misses heartbeat
Process B: LOCK resource → token=43, expire_at=t+30s
          Lock assigned to B
Process B: WRITE value=200 WITH token=43
          Server applies

Process A: Resumes, tries WRITE value=101 WITH token=42
          Server rejects (token 42 is old, current is 43)
```

**Why it works**: Even if A doesn't know it lost lock, server prevents stale writes.

## Lock Contention

Multiple processes waiting for same lock.

```
LOCK queue:
  Process A (holder, expires t+10s)
  Process B (waiting)
  Process C (waiting)

At t+10s: A's lock expires
          Server: Next waiter B gets lock (FIFO or random?)
          Notification: B, C learn lock state changed
```

**Challenge**: Thundering herd (100 processes waiting, all wake up on release)

## Redlock Controversy

Redis Redlock (distributed lock via Redis cluster):

```
LOCK: Set key on 5 Redis nodes with expiry
      Wait for majority to confirm (before returning)
      
Problem: Clock skew
  Node A: System clock skews backward 10 seconds
  Lock: SET key EX 30 (server thinks expires at t+30)
  Reality: Lock set at t-10, so actually expires at t+20
  Lock expires early, two clients hold lock simultaneously!
```

**Better**: Use coordination service (Chubby, Zookeeper, Consul) with consensus.

---

## Production: Chubby, Zookeeper, Consul

These use consensus algorithms (Paxos/Raft) for safety.

```
Chubby (Google): Strongly consistent, used for distributed locks
Zookeeper: Apache Zookeeper, used by Hadoop, Kafka, etc.
Consul: Modern service discovery + locking

Key: Consensus guarantees no split-brain, even with network partitions
```

---

## References

- "The Chubby lock service for loosely-coupled distributed systems" — Burrows (Google)
- "Is Distributed Locking Safe?" — Chris Dwan (Redlock debate)

---

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/) – Uses consensus for safety
- [Raft](raft.md) – Algorithm used by Consul, etcd
- [CAP & PACELC](../distributed-systems-theory/cap-and-pacelc.md) – Consistency guarantees
