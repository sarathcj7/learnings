# Distributed Transactions

Coordinating atomic changes across multiple databases and services. From blocking protocols (2PC) to compensating transactions (Sagas) to eventual consistency — the fundamentals of distributed atomicity.

## Sub-topics

- **[Two-Phase Commit](two-phase-commit.md)**: The classic protocol for distributed ACID. Prepare/commit phases, blocking, failure modes, and why it doesn't scale.
- **[Saga Pattern](saga-pattern.md)**: Break long transactions into independent local transactions with compensating logic. Choreography vs. orchestration.
- **[Distributed ACID](distributed-acid.md)**: ACID properties in distributed systems. Consistency levels, eventual consistency, BASE model, quorum-based consistency.

## Why This Matters

- **Atomicity at scale**: Applications must coordinate state across multiple services. How do you ensure "all or nothing"?
- **Trade-offs**: Strong ACID (2PC) vs. eventual consistency (Sagas). Blocking vs. availability. Simplicity vs. complexity.
- **Resilience**: Failures are inevitable in distributed systems. How do you recover from partial failures?
- **Real-world complexity**: Most systems use hybrid approaches (strong consistency for critical data, eventual for the rest).

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/distributed-transactions.md)**: Self-test recall on 2PC, Sagas, ACID properties
- **[Flashcards](../../interview-prep/flashcards/distributed-transactions.md)**: Spaced-repetition drills

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/): Raft, Paxos for distributed agreement
- [Databases - Transactions & Isolation](../databases/transactions-and-isolation-levels.md): Single-node ACID foundation
- [Messaging & Streaming](../messaging-and-streaming/): Event-driven sagas, event sourcing
- [Microservices Architecture](../microservices-architecture/): Sagas are core to microservices resilience
- [Reliability & Resiliency](../reliability-and-resiliency/): Failure recovery in distributed systems

## Related Case Studies

- [Distributed Cache](../../case-studies/distributed-cache.md) – Consistency in cache layers
- [Payment System](../../case-studies/payment-system.md) – Distributed transactions in financial systems

---

**Study Tips**

1. **Start with 2PC**: Understand the protocol, then see why it's impractical at scale.
2. **Learn both Saga styles**: Choreography (loose coupling) vs. orchestration (centralized control).
3. **Think in trade-offs**: Every choice (2PC vs. Saga, strong vs. eventual) sacrifices something.
4. **Design practice**: For each case study, decide on 2PC vs. Saga vs. tunable consistency.

---

**Status**: ✅ Complete
