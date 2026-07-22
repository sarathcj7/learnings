# Raft

## TL;DR

- **Raft**: Consensus algorithm, easier to understand than Paxos, same guarantees
- **Three states**: Follower (passive), Candidate (seeks leadership), Leader (active)
- **Leader election**: Timeout-based, random backoff prevents split brain
- **Log replication**: Leader appends entries, followers replicate
- **Key insight**: Safety + liveness + practical, used in production (etcd, Consul, etc.)

## States & Transitions

```
Follower:
  - No leader, no communication for timeout duration
  - Start election: timeout → Candidate

Candidate:
  - Request votes from all (including self)
  - If majority votes: → Leader
  - If lose election: → Follower
  - If another leader found: → Follower

Leader:
  - Send heartbeats to all followers
  - Receive client requests, replicate to followers
  - On follower timeout/lag: Re-send logs
  - On crash: → Follower (old term seen)
```

## Leader Election

```
Follower 1: timeout (randomized 150-300ms)
            Increment term, vote for self
            Request votes from Followers 2, 3
Follower 2: Receives vote request, votes for 1
Follower 3: Receives vote request, votes for 1
Follower 1: Majority (2/3), becomes Leader
            Sends heartbeats to maintain leadership
```

**Why randomized timeout**: Prevents split brain (simultaneous elections). First to timeout wins.

## Log Replication

Leader commits entry once majority replicate:

```
Client: Write X=5
Leader: Append to log [term:2, index:1, X=5]
        Send to followers
Follower 1: Receives, appends to log
Follower 2: Receives, appends to log
Leader: Majority replicated, commits [1]
        Applies to state machine: X=5
        Responds to client: success
```

**Safety**: Log is only committed when leader of that term has majority replication.

## Compared to Paxos

| Aspect | Paxos | Raft |
|---|---|---|
| **Complexity** | High (complex protocol) | Lower (clear roles) |
| **Leader election** | Not in basic Paxos | Built-in |
| **Log replication** | Per-entry voting | Entire log ordered |
| **Understandability** | Hard (papers unclear) | Clear + papers explain |
| **Production use** | Google Chubby | etcd, Consul, Raft libraries |

---

## Key Properties (Raft Guarantees)

1. **Election Safety**: At most one leader per term
2. **Log Matching**: Identical logs on all servers (eventually)
3. **State Machine Safety**: Applied entries never disappear

---

## References

- "Raft Consensus Algorithm" — Ongaro, Ousterhout
- https://raft.github.io/ (visualization, papers, implementations)

---

## Related Fundamentals

- [Paxos](paxos.md) – Complex version, same guarantees
- [Leader Election](leader-election.md) – Raft's leader election deep dive
- [Distributed Locks](distributed-locks.md) – Uses consensus for safety
