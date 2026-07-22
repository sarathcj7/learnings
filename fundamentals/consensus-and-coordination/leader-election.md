# Leader Election

## TL;DR

- **Bully algorithm**: Highest ID wins
- **Ring algorithm**: Sequential voting around ring
- **Raft**: Randomized timeout + voting (most popular today)
- **Challenge**: Detect old leader failure before starting election
- **Split brain**: Two leaders claim power (disaster, must prevent)

## Problem

How do processes choose a leader when current leader fails?

```
3-node cluster: Leader = Node 1
Node 1 crashes
Nodes 2, 3 detect failure (timeout)
Nodes 2, 3 want to be leader
Need to agree on exactly one
```

## Bully Algorithm

Highest ID wins. Higher ID can override lower ID.

```
Node 3 detects leader (Node 1) dead
Node 3: "I'm running for leader, anyone higher?"
Node 2: "I'm higher ID, I'm leader"
Node 3: Backs down
Result: Node 2 becomes leader (highest alive)
```

**Pros**: Simple  
**Cons**: Assumes IDs are comparable, election storms (if many nodes crash)

## Ring Algorithm

Sequential voting around a ring of processes.

```
Node 3: "Starting election, I'm alive"
        Sends message to Node 1 (next in ring)
Node 1: "Got election from 3, I'm alive, forward it"
        Sends to Node 2
Node 2: "Got election from 3→1, I'm alive, forward it"
        Back to Node 3
Node 3: "Got my own election message back, majority alive = {1,2,3}"
        "Highest ID in set is 1, so Node 1 is leader"
```

**Pros**: Works even if some nodes dead  
**Cons**: Ring can be slow, O(N) messages

## Raft (Production Standard)

Timeout-based, random backoff, voting.

```
Follower (no heartbeat for >150ms):
  Timeout! Increment term
  Vote for myself
  Send RequestVote to all
  
Other followers: Vote if term is higher

First to get majority votes becomes leader

Lower timeout nodes → Higher chance of leadership (biased)
Random backoff (150-300ms) prevents simultaneous elections
```

**Why it works**:
- Only one leader per term (voting ensures quorum)
- Random backoff prevents ties
- Heartbeats maintain leadership

## Split Brain (Disaster)

Two leaders exist simultaneously:

```
Network partition:
  Nodes {1, 2} in partition A (Node 1 leader, partition A minority)
  Node 3 in partition B (elect Node 3 leader)

Now:
  Client writes to Node 1 (in partition A) → Rejected (quorum needs 2/3)
  Client writes to Node 3 (in partition B) → Accepted (quorum is 1 node)
  
When partition heals:
  Node 1 sees term from Node 3, steps down as leader (Raft property: higher term wins)
  Node 3 is leader
  Data written by Node 1 (minority partition) is discarded (lost!)
```

**Prevention**: Quorum majority. Minority partition cannot commit.

---

## References

- "Designing Data-Intensive Applications" — Kleppmann
- Raft paper

---

## Related Fundamentals

- [Raft](raft.md) – Raft's leader election mechanism
- [Consensus & Coordination](../consensus-and-coordination/) – Overall consensus
