# Messaging and Streaming — Flashcards

**Format**: Question · Answer (collapsed). Review, self-grade, repeat until automatic.

---

## Delivery Semantics

### Q: What's at-most-once delivery?
<details><summary>Answer</summary>

Message delivered 0 or 1 times. No retries on failure. Message may be lost. Use for: non-critical data (metrics, analytics). No deduplication needed.

</details>

### Q: What's at-least-once delivery?
<details><summary>Answer</summary>

Message delivered 1+ times. Retries on failure until success. Message may be duplicated. Use for: most cases (orders, payments). Requires idempotent consumers.

</details>

### Q: What's exactly-once delivery?
<details><summary>Answer</summary>

Message delivered exactly 1 time. Theoretically impossible across network (requires coordination). In practice: at-least-once + idempotent consumer = exactly-once behavior. Kafka supports transactional writes.

</details>

### Q: How do you handle duplicates in at-least-once delivery?
<details><summary>Answer</summary>

Make consumer idempotent: check if message already processed (deduplicate by message ID). Store in deduplication cache (Redis) or check database (idempotency key). Accept idempotent design.

</details>

### Q: What's the difference between delivery guarantee and ordering guarantee?
<details><summary>Answer</summary>

Delivery: message reaches consumer (at-most vs at-least vs exactly-once). Ordering: messages processed in order. Kafka: at-least-once delivery + order per partition. RabbitMQ: at-most-once by default.

</details>

---

## Message Queues vs Topics

### Q: What's the difference between a queue and a topic?
<details><summary>Answer</summary>

Queue: point-to-point, one consumer processes each message. Topic: pub-sub, multiple consumers see same message. Queue: simple, ordered. Topic: flexible, decoupled publishers.

</details>

### Q: When would you use RabbitMQ vs Kafka?
<details><summary>Answer</summary>

RabbitMQ: job queues, task distribution (traditional messaging). Kafka: event streaming, distributed log, high throughput. RabbitMQ: strong ordering, transactional. Kafka: replay-able, partition-based scale.

</details>

### Q: What's publish-subscribe (pub-sub)?
<details><summary>Answer</summary>

Publishers send messages to topics. Multiple subscribers receive all messages from subscribed topics. Decouples producers/consumers. Scalable to many subscribers. Example: event notifications.

</details>

### Q: What are consumer groups in Kafka?
<details><summary>Answer</summary>

Set of consumers reading same topic. Kafka distributes partitions across consumers. If 5 consumers, each gets 2 of 10 partitions. One consumer fails, partitions rebalance. Scales consumption horizontally.

</details>

---

## Kafka Internals

### Q: How does partitioning scale throughput in Kafka?
<details><summary>Answer</summary>

Each partition is independent, stored on separate brokers. Multiple producers write to different partitions in parallel. Multiple consumers read different partitions in parallel. Throughput = N partitions × partition throughput.

</details>

### Q: What happens if a Kafka partition leader is down?
<details><summary>Answer</summary>

Kafka picks a new leader from in-sync replicas (ISR). Brief unavailability (seconds). Producers buffer until leader recovered. Consumers may see slight lag. Replication factor 3 = tolerates 1 broker failure.

</details>

### Q: What's key-partition mapping in Kafka?
<details><summary>Answer</summary>

Message key is hashed to determine partition (same key always goes to same partition). No key: round-robin across partitions. With key: preserves ordering for that key, enables joins on key.

</details>

### Q: What's ISR (in-sync replicas) in Kafka?
<details><summary>Answer</summary>

Replicas that are caught up with leader (within lag threshold). Only ISR replicas are eligible to become leader. More ISR = safer (can tolerate more failures). Kafka waits for ISR to acknowledge before acking producer.

</details>

### Q: What's log compaction in Kafka?
<details><summary>Answer</summary>

Kafka compacts topic, keeping only latest value per key. Useful for state/snapshots (e.g., user profile changes). Saves space. Example: "user_123: {name: Alice, age: 30}" - only latest state kept.

</details>

---

## Stream Processing

### Q: What's the difference between stream processing and batch?
<details><summary>Answer</summary>

Stream: process data as it arrives (low latency). Batch: process data periodically (high throughput). Stream: continuous computation. Batch: one-time computation.

</details>

### Q: When would you use stream processing vs batch?
<details><summary>Answer</summary>

Stream: real-time alerts (fraud detection), dashboards (live metrics), recommendations. Batch: daily reports, bulk analytics, model training. Hybrid: stream + batch for comprehensive analysis.

</details>

### Q: What's windowing in stream processing?
<details><summary>Answer</summary>

Partition infinite stream into time windows (tumbling, sliding, session). Tumbling: 1min windows (non-overlapping). Sliding: 1min windows, slide every 30s (overlapping). Enables aggregations on bounded data.

</details>

### Q: What's stateful stream processing?
<details><summary>Answer</summary>

Maintain state across events (e.g., user session count). Challenges: fault tolerance (persist state), rebalancing (migrate state). Kafka Streams, Flink handle this. Enable complex operations (joins, aggregations).

</details>

### Q: What's backpressure in stream processing?
<details><summary>Answer</summary>

Producer faster than consumer (events pile up). Backpressure: slow down producer or buffer. Solution: bounded queues (block producer), drop messages (sample), or scale consumers.

</details>

---

## Event Sourcing & CQRS

### Q: What's event sourcing?
<details><summary>Answer</summary>

Store state as immutable sequence of events (event log). Current state = replay all events. Benefit: complete history, audit trail, replay to any point. Trade-off: complex, need snapshots for large logs.

</details>

### Q: What's CQRS?
<details><summary>Answer</summary>

Command Query Responsibility Segregation: separate read and write models. Write model: event-sourced. Read model: denormalized views (optimized for queries). Benefit: independent scaling, optimized for each pattern.

</details>

### Q: What's snapshotting in event sourcing?
<details><summary>Answer</summary>

Periodically save state snapshot (every 1000 events). On replay, start from snapshot + replay remaining events. Avoids replaying full 10M event history. Trade-off: complexity, storage.

</details>

### Q: What's the dual-write problem?
<details><summary>Answer</summary>

Writing to event log AND read model asynchronously. If service crashes between writes, inconsistency. Solution: single write to event log, async update read model (event-driven).

</details>

---

## Dead Letter Queues & Error Handling

### Q: What's a dead letter queue (DLQ)?
<details><summary>Answer</summary>

Messages that can't be processed (poison messages) are sent to DLQ. Separate from main queue to avoid blocking consumers. Allows inspection and manual retry later.

</details>

### Q: When should a message go to DLQ?
<details><summary>Answer</summary>

Max retries exceeded. Message is malformed (parsing fails). Message fails permanently (not transient). Example: invalid JSON, missing required fields.

</details>

### Q: What's an exponential backoff retry strategy?
<details><summary>Answer</summary>

Retry after 1s, 2s, 4s, 8s (exponential). Cap at 60s. Avoids thundering herd (all retrying at once). DLQ after max retries. Better than immediate retry (gives transient failures time to recover).

</details>

---

## Self-Scoring Guide

- Mark each flashcard: immediate answer (✓) or struggled (✗)
- Review ✗ cards daily until they become automatic
- Interview readiness: 15+ correct on first pass
