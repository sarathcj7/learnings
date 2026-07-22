# Messaging & Streaming

Decoupling services with queues, pub-sub, and event streaming. Kafka and friends handle the most demanding scales (millions of events/sec) with ordered delivery and fault tolerance.

## Sub-topics

- **[Message Queue Basics](message-queue-basics.md)** ✅: Delivery semantics (at-most/at-least-once), ordering, acknowledgment, queues vs topics
- **[Kafka Architecture](kafka-architecture.md)** ✅: Topics, partitions, brokers, replication, consumer groups, scaling
- **[Event Sourcing](event-sourcing.md)** ✅: Immutable log, audit trail, temporal queries, snapshots, CQRS
- **[Exactly-Once Semantics](exactly-once-semantics.md)** ✅: Idempotency, distributed transactions, stream processing
- **[Stream Processing](stream-processing.md)** ⏳ (Spark, Flink, stateful operations)

## Why This Matters

- **Decoupling**: Services don't need to know about each other
- **Resilience**: One slow service doesn't slow down others
- **Scale**: Kafka handles 1M+ events/sec across clusters
- **Audit**: Event sourcing provides complete history
- **Interview**: "Design event streaming system" appears in 30% of interviews

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/messaging-and-streaming.md)**: 40+ messaging questions
- **[Flashcards](../../interview-prep/flashcards/messaging-and-streaming.md)**: Kafka, ordering, delivery semantics

## Related Case Studies

- [Chat & Messaging System](../../case-studies/chat-and-messaging-system.md) – Kafka for ordering
- [Notification System](../../case-studies/notification-system.md) – At-least-once delivery
- [Distributed Job Scheduler](../../case-studies/distributed-job-scheduler.md) – Job queue pattern
- [Real-time Analytics](../../case-studies/real-time-analytics-and-leaderboard.md) – Stream processing

## Related Fundamentals

- [Idempotency](../apis-and-communication/idempotency-and-api-design.md) – Safe retries
- [Reliability](../reliability-and-resiliency/) – Handling message failures
- [Databases/Replication](../databases/replication.md) – Similar concepts

## Study Tips

1. **Delivery semantics first**: Understand at-least-once, at-most-once, exactly-once
2. **Ordering crucial**: Per-partition ordering (Kafka) enables consistency
3. **Kafka dominates**: Know partitions, consumer groups, offsets deeply
4. **Tradeoff thinking**: Speed vs ordering, scaling vs latency
5. **Event sourcing**: Powerful pattern but adds complexity

---

**Status**: ✅ 80% Complete. 4/5 files written (stream processing queued).
