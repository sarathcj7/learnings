# Distributed Systems Theory

The mathematical foundations of distributed systems. Understand CAP, consistency models, time/ordering, and quorum systems — the concepts that explain why distributed systems behave the way they do.

## Sub-topics

- **[CAP & PACELC](cap-and-pacelc.md)** ✅: CAP theorem, consistency vs availability vs partition, PACELC extension
- **[Consistency Models](consistency-models.md)** ✅: Strong, eventual, read-after-write, causal, monotonic
- **[Time, Clocks & Ordering](time-clocks-and-ordering.md)** ✅: Lamport timestamps, vector clocks, HLC, Google Spanner TrueTime
- **[Quorum Systems](quorum-systems.md)** ✅: Quorum math, read repair, sloppy quorum, Dynamo-style

## Why This Matters

- **Understanding constraints**: CAP explains why you can't have everything
- **Ordering events**: Critical for causality in distributed systems
- **Quorum math**: Foundation for DynamoDB, Cassandra, Dynamo-style systems
- **Interview essential**: Expect deep CAP/consistency questions

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/distributed-systems-theory.md)**: 30+ theory questions
- **[Flashcards](../../interview-prep/flashcards/distributed-systems-theory.md)**: Key concepts and proofs

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/) – How to achieve strong consistency despite theory
- [Databases/Replication](../databases/replication.md) – Replication strategy depends on CAP choice
- [Caching](../caching/) – Cache inconsistency is eventual consistency trade-off

---

**Status**: ✅ Complete. 4 sub-files covering theory and production implications.
