# Message Queue Basics

Understanding publish-subscribe, at-least-once delivery, ordering guarantees, and when to use queues vs direct calls.

---

## TL;DR

- **Message queue**: Decouples producer from consumer, enables asynchronous processing
- **At-least-once**: Message guaranteed to be delivered (might be duplicate, app must be idempotent)
- **Ordering**: Messages delivered in order per partition/shard
- **Acknowledgment**: Consumer confirms processing, enables retry on failure
- **Use for**: Async jobs, event sourcing, fan-out, resilience

---

## Why Message Queues

### Without Queue (Direct Call)

```
Service A (payment) → calls → Service B (email)
  
Flow:
  1. Process payment
  2. Call sendEmail(user_email)
  3. Wait for B's response (synchronous)
  
Problems:
  If B is down: A's transaction fails
  If B is slow (10s to send): A's request waits 10s
  Tight coupling: Can't change B without affecting A
  Cascading failure: B's slowness degrades A
```

---

### With Queue (Async via Queue)

```
Service A (payment) → publishes to → Queue (Kafka) → Service B (email)
  
Flow:
  1. Process payment
  2. Publish event: "PaymentProcessed" to queue (1ms)
  3. Return response immediately
  4. B asynchronously reads from queue
  5. B sends email
  
Advantages:
  A doesn't wait for B (async)
  B can be down, message waits (resilience)
  Can add service C (logging), D (analytics) easily (fan-out)
  Loose coupling
```

---

## Delivery Semantics

### At-Most-Once

```
Message delivered 0 or 1 times (not guaranteed):

Implementation:
  Consumer processes message
  Consumer crashes before acknowledging
  Message lost (never delivered again)

Use case: Metrics, logs (losing one point acceptable)

Example:
  Event: "User clicked button"
  If lost: Analytics slightly off (acceptable)
```

---

### At-Least-Once

```
Message delivered 1+ times (might be duplicate):

Implementation:
  Consumer processes message
  Consumer acknowledges (queue removes message)
  If no ack: Message redelivered after timeout
  
Guarantee: Message won't be lost
  
Downside: Might be duplicated
  Consumer must be idempotent (duplicate processing is safe)

Use case: Payments, orders, critical operations

Example:
  Event: "Transfer $50"
  If duplicate: Idempotency key prevents duplicate charge
  Amount correctly transferred once
```

---

### Exactly-Once

```
Message delivered exactly 1 time (distributed transactions):

Implementation: HARD
  Requires 2-phase commit, distributed consensus
  Complex, slow, not always available

Why hard:
  Network can fail after ack but before removal
  Consumer can crash after processing but before ack
  
Workaround: At-least-once + idempotency
  Easier than true exactly-once
  Sufficient for most use cases
```

---

## Message Ordering

### No Ordering Guarantee

```
Producer sends:
  Event 1: "User created"
  Event 2: "User verified email"
  Event 3: "User subscribed"

Queue (no ordering):
  Consumer 1 receives: Event 3, Event 1, Event 2 (random order!)
  Consumer 2 receives: Event 2, Event 3, Event 1

Problem: Events processed out of order
  User subscribed BEFORE email verified
  Invalid state!
```

---

### Single Partition Ordering

```
Messages published to single partition:
  Event 1 → Partition 0 (queue)
  Event 2 → Partition 0
  Event 3 → Partition 0

Queue guarantees: Order within partition
  Consumer receives: Event 1, Event 2, Event 3 (always this order)

Tradeoff:
  Pro: Ordering guaranteed
  Con: Single partition = bottleneck, can't scale

Usage: Kafka partitions, RabbitMQ queues
```

---

### Partitioning by Key

```
Messages partitioned by user ID:
  Event 1: "User 100 created"
    user_id = 100 → Partition 0
  Event 2: "User 100 verified"
    user_id = 100 → Partition 0
  Event 3: "User 200 created"
    user_id = 200 → Partition 1

Result:
  Partition 0: Events for user 100 (ordered)
  Partition 1: Events for user 200 (ordered)
  Different users' events can be out of order (acceptable)

Benefit: Ordered per user, scalable across users
  Partition 0 handles user 100 events
  Partition 1 handles user 200 events
  Parallel processing!
```

---

## Acknowledgment & Retry

### Without Acknowledgment

```
Consumer receives message:
  1. Process message
  2. Return response (no ack)

Queue doesn't know if processed successfully:
  Message might be lost

Result: Unreliable delivery
```

---

### With Acknowledgment

```
Consumer receives message:
  1. Process message
  2. Acknowledge to queue
  3. Queue removes message

If consumer crashes before ack:
  Message stays in queue
  Redelivered to another consumer

Timeout:
  After 30s of no ack: Redelivery triggered
  Balance: Not too fast (thrashing), not too slow (lag)
```

---

## Topic vs Queue

### Queue (1-to-1)

```
Message published to queue
Only ONE consumer reads it

Example: Order processing
  Service A publishes: "Order #123"
  Service B (only one): Processes order
  Message removed after processing
```

---

### Topic/PubSub (1-to-Many)

```
Message published to topic
ALL subscribers receive it

Example: Order created
  Service A publishes: "Order #123 created"
  Service B (email): Sends confirmation email
  Service C (analytics): Logs order
  Service D (inventory): Decrements stock
  
All 4 services receive same message (broadcast)
```

---

## Dead Letter Queue (DLQ)

```
Message fails repeatedly:
  Consumer tries to process
  Throws exception
  Queue retries (exponential backoff)
  After 5 retries: Message goes to Dead Letter Queue

DLQ is separate queue for failed messages:
  Humans can inspect and manually fix
  Can reprocess later
  Prevents message from blocking other processing

Implementation:
  Original queue: max_retries = 5
  On 5th failure: Move to dlq_orders queue
  Monitor dlq_orders for stuck messages
```

---

## Ordering Guarantees Example

### Scenario: User State Machine

```
User workflow:
  1. Created
  2. Verified email
  3. Subscribed
  4. Unsubscribed
  5. Deleted

Events must be processed in order (no jumping from Created to Unsubscribed)

Solution:
  Partition by user_id
  All events for user 123 go to same partition
  Processed in order

Result:
  User 123: Created → Verified → Subscribed → Unsubscribed
  User 456: Created → Subscribed (different user, different partition)
  Scaling: Many partitions (many users in parallel)
```

---

## Trade-offs Summary

| Aspect | Queue (1-to-1) | Topic (1-to-Many) |
|---|---|---|
| **Coupling** | Loose | Loosest |
| **Scaling** | One consumer | Many subscribers |
| **Ordering** | Per queue | Per topic partition |
| **Complexity** | Simple | Moderate |
| **Use case** | Batch jobs, async | Events, pub-sub |

---

## Related Fundamentals

- [Kafka](kafka-architecture.md) – Most popular message queue
- [Event Sourcing](event-sourcing.md) – Append-only event log
- [Reliability](../reliability-and-resiliency/) – Idempotency for at-least-once

---

**Status**: ✅ Complete. Covers delivery semantics, ordering, acknowledgment, queues vs topics.

