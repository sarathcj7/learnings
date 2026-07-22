# Paxos

## TL;DR

- **Paxos**: Consensus algorithm for distributed systems. Complex but provably correct.
- **Three roles**: Proposer (suggests value), Acceptor (votes), Learner (learns outcome)
- **Two phases**: Phase 1 (prepare/promise), Phase 2 (accept/accepted)
- **Key idea**: Majority consensus ensures consistency despite failures
- **Problem**: Hard to implement correctly, Raft is simpler alternative

## The Problem

How do N distributed processes agree on a value when some may fail?

```
Process A: "value = 1"
Process B: "value = 2"
Process C: "value = 3"

How do they reach consensus?
- Majority voting: Value chosen if > N/2 accept it
- Fault tolerance: Can tolerate up to (N-1)/2 failures
```

## Two Phases

### Phase 1: Prepare (Establish Promise)

Proposer asks acceptors: "Promise not to accept lower proposal numbers?"

```
Proposer sends: prepare(proposalNumber=10)
Acceptor 1 checks: "No promise yet, so promise"  → Returns: promise(10)
Acceptor 2 checks: "No promise yet, so promise" → Returns: promise(10)
Acceptor 3 crashes (no response)

Proposer has promises from 2/3 acceptors (majority)
```

### Phase 2: Accept (Get Consensus)

Proposer sends: "Accept my value now"

```
Proposer sends: accept(proposalNumber=10, value=X)
Acceptor 1 checks: "Promised 10, can accept" → Accepts
Acceptor 2 checks: "Promised 10, can accept" → Accepts
Acceptor 3 still crashed

Majority (2/3) accepted value X
Proposer learns: Value X chosen
```

## Pros & Cons

**Pros**:
- Provably correct (safety guaranteed)
- Tolerates (N-1)/2 failures
- Works in asynchronous networks (no timing assumptions)

**Cons**:
- Very complex (hard to implement)
- Two round trips = latency
- Hard to understand, debug, extend

## Practical Issue: Implementation

Paxos papers don't specify many details:
- Leader election? (Not in basic Paxos)
- Reconfiguration? (Changing membership)
- Disk I/O? (Making it durable)

Result: Hard to deploy correctly. Google's Chubby and others took months to get it right.

**Better alternative**: Raft (similar guarantees, easier to understand and implement)

---

## References

- "The Part-Time Parliament" — Lamport (Paxos paper)
- "Paxos Made Simple" — Lamport (simplified version)

---

## Related Fundamentals

- [Raft](raft.md) – Simpler consensus algorithm, same guarantees
- [Distributed Systems Theory](../distributed-systems-theory/) – Why consensus is hard
- [Leader Election](leader-election.md) – Choosing a leader in Paxos
