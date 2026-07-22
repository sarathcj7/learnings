# Distributed Lock & Coordination Service (Zookeeper-Style)

*Design Zookeeper / Consul: distributed locks, leader election, configuration management, service discovery.*

## Problem Statement

Build a coordination service. Multiple services need to agree on leader, manage shared configuration, implement distributed locks. Single source of truth for distributed state.

## Architecture

```
Tree of znodes (nodes):
  /app/config
  /app/locks/resource-1
  /app/services/web-1, web-2, web-3

Leader Election:
  Candidate 1 creates /app/leader-1 (ephemeral)
  Candidate 2 creates /app/leader-2 (ephemeral)
  Candidate 3 creates /app/leader-3 (ephemeral)
  
  Consensus (Raft/Paxos):
    Majority votes (3 candidates, 2 votes needed)
    Candidate 1 elected leader
    Other candidates: WATCH /app/leader-1 (notified if gone)
  
  Candidate 1 crashes:
    /app/leader-1 auto-deleted (ephemeral)
    Others notified (watch triggered)
    New election starts
```

## Use Cases

### Distributed Locks

```
Resource: database-backup

Acquire lock:
  CREATE /locks/db-backup (exclusive)
  If already exists: Wait (watch for deletion)
  On delete: Automatically eligible again

Release lock:
  DELETE /locks/db-backup
  Others woken up
```

### Configuration Management

```
Service reads config from /app/config:
  {"timeout": 30, "retries": 3}
  
Admin updates config:
  SET /app/config {...updated...}
  
All services watching /app/config:
  Notified of change immediately
  Reload config
```

## Bottlenecks & Scaling

**Bottleneck**: Write throughput (strong consistency = slower).

**Solution**:
- Use for coordination only (slow, but critical)
- Cache reads locally (fast path)
- Batch updates if possible

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Strong consistency** | Eventual consistency | Coordination requires agreement |
| **Consensus (Raft)** | Gossip protocol | Consensus guarantees correctness |
| **Ephemeral nodes** | Explicit cleanup | Auto-cleanup on disconnect (crash-safe) |

---

**Status**: ✅ Complete. Shows coordination, leader election, locks.
