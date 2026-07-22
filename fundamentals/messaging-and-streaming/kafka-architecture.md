# Kafka Architecture & Design

Deep dive into Kafka's architecture, partitioning, replication, and why it's the de facto standard for event streaming.

---

## TL;DR

- **Kafka**: Distributed append-only log, scales to millions of events/sec
- **Topics & Partitions**: Logical topic sharded across partitions (parallelism)
- **Broker**: Kafka server that stores partitions
- **Replication factor**: 3 copies (1 leader, 2 replicas) for durability
- **Consumer groups**: Multiple consumers share partitions for parallel processing
- **Offset**: Position in log, consumer tracks to resume from last processed

---

## Kafka Components

### Topics

```
Topic: "order-events"
  Used for: Orders, payments, shipping

Topic: "user-events"
  Used for: User signups, profile changes

Topic: "log-events"
  Used for: Application logs
  
Each topic = logical stream of events
```

---

### Partitions

```
Topic: "order-events"
  Partition 0: Orders for user 0, 1000, 2000, ...
  Partition 1: Orders for user 1, 1001, 2001, ...
  Partition 2: Orders for user 2, 1002, 2002, ...
  Partition 3: Orders for user 3, 1003, 2003, ...
  
Partitioning by user_id:
  Each partition handles 1/4 of users
  Ordered within partition (user 100's orders in order)
  Parallel processing (4 partitions in parallel)
  
Why: Ordering + scale
```

---

### Brokers

```
Kafka cluster:
  Broker 1: Partition 0 leader, Partition 1 replica, Partition 3 replica
  Broker 2: Partition 1 leader, Partition 0 replica, Partition 2 replica
  Broker 3: Partition 2 leader, Partition 3 replica, Partition 0 replica

Leader responsibilities:
  Receive writes from producers
  Send reads to consumers
  
Replica responsibilities:
  Keep copy of partition (sync with leader)
  Failover if leader dies
```

---

## Replication & Durability

### Write Process

```
Producer sends event:
  Event: { order_id: 123, total: $50 }
  Topic: order-events
  Partition: 0 (based on order_id % 4)

Broker 1 (leader of partition 0):
  1. Write to log (disk)
  2. Send to Broker 2 (replica)
  3. Send to Broker 3 (replica)
  
Acknowledgment strategy:
  ack=1: Wait for leader write only (fast, data loss risk if leader dies)
  ack=all: Wait for all replicas (slow, durable)
  ack=0: Don't wait for anything (fastest, no guarantee)
  
Typical production: ack=all (durability matters)
```

---

### Failover

```
Broker 1 (leader of partition 0) dies:

Before:
  Partition 0: Leader on Broker 1
  Replicas: Broker 2, Broker 3

Election triggered:
  Zookeeper/Controller detects Broker 1 dead
  Promotes Broker 2 or Broker 3 as new leader
  
After (if Broker 2 elected):
  Partition 0: Leader on Broker 2
  Replicas: Broker 1 (when recovered), Broker 3
  
Result: No message loss (all had replicas)
  Downtime: ~500ms (leader election time)
```

---

## Consumer Groups

### Parallel Processing

```
Topic: order-events (4 partitions)
  Partition 0: 10k events
  Partition 1: 10k events
  Partition 2: 10k events
  Partition 3: 10k events

Consumer group: "order-processor" (4 consumers)
  Consumer A: Processes partition 0 (10k events)
  Consumer B: Processes partition 1 (10k events)
  Consumer C: Processes partition 2 (10k events)
  Consumer D: Processes partition 3 (10k events)

Concurrency:
  4 consumers × throughput per consumer = 4x throughput
  Can scale: Add more partitions + consumers
```

---

### Scaling Consumers

```
Start: 1 partition, 1 consumer
  Consumer A handles all events
  Throughput: 1000 events/sec

Bottleneck: Consumer A maxed out

Scale:
  Add partition 1
  Add consumer B
  
Now:
  Consumer A: Partition 0
  Consumer B: Partition 1
  Total throughput: 2000 events/sec
  
Can scale to 100 partitions × 100 consumers = 100x throughput
```

---

### Consumer Lag

```
Producer writes events faster than consumer processes:

Partition 0:
  Events in log: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 (10 events)
  
Consumer:
  Last offset processed: 5
  
Lag = 10 - 5 = 5 events behind
  Consumer catching up (processing events 6, 7, 8, 9)
  
Monitor:
  If lag grows constantly: Consumer can't keep up
  Add more consumers (more parallelism)
```

---

## Offset Management

### Offset Tracking

```
Consumer processes events:
  Event at offset 0: Process order #1
  Event at offset 1: Process order #2
  Event at offset 2: Process order #3
  
Consumer crashes after processing offset 2:
  Restart consumer
  Check: Last committed offset = 2
  Resume from offset 3
  
Result: No duplicate processing, no message loss
```

---

### Commit Strategies

```
Strategy 1: Auto-commit (simple)
  Consumer processes event
  Automatically commit offset after 5 seconds
  
Problem:
  Consumer might crash before 5s
  Offset committed but event not fully processed
  Duplicate on restart
  
Strategy 2: Manual commit (safe)
  Consumer processes event
  Consumer explicitly commits offset
  
Benefit:
  Only commit after fully processed
  No duplicates
  
Code example:
  record = consumer.poll()
  process(record)
  consumer.commit()  // Explicit
```

---

## Throughput & Scaling

### Performance Characteristics

```
Single broker (3 replicas):
  Throughput: 100k events/sec
  Latency: 100ms (end-to-end)
  
Cluster (3 brokers, 12 partitions):
  Throughput: 1M events/sec (4x with more partitions)
  Latency: 100ms (same as single broker)

Scaling:
  More partitions = more parallelism
  More brokers = more capacity
  More consumers = more throughput
```

---

### Retention Policy

```
Retention by time:
  Keep events for 7 days
  Older events automatically deleted
  
Use case: Most topics (events are ephemeral)

Retention by size:
  Keep events until log reaches 1GB
  Oldest events evicted

Retention unlimited:
  Keep all events forever
  Use case: Event sourcing, audit logs
  
Trade-off: Storage cost vs retention window
```

---

## Anti-Patterns & Pitfalls

### Anti-Pattern 1: Single Partition Topic

```
Topic: "orders" (1 partition only)

Problem:
  Only 1 consumer can process (1 partition = max 1 active consumer)
  Can't scale (even with 100 consumers, only 1 active)
  Bottleneck: Single partition on single broker
  
Solution:
  Create topic with 10+ partitions upfront
  Even if only 1 consumer now, enables future scaling
```

---

### Anti-Pattern 2: Not Committing Offsets

```
Consumer processes events but never commits:

Problem:
  Consumer restarts
  No offset committed
  Reprocesses all events from beginning
  Duplicates everywhere
  
Solution:
  Explicit offset commit after processing
  Or use auto-commit with careful timing
```

---

### Anti-Pattern 3: Slow Consumer

```
Consumer takes 10 seconds per event:
  1000 events in queue
  Consumer lag grows 100 events/sec
  
Issue:
  Partition rebalancing might trigger (thinking consumer dead)
  Cascading failures

Solution:
  Make consumer faster (optimize code, parallelize)
  Or split topic across more partitions and add consumers
```

---

## Kafka vs RabbitMQ vs NATS

| Aspect | Kafka | RabbitMQ | NATS |
|---|---|---|---|
| **Throughput** | 1M+ events/sec | 50k events/sec | 100k+ events/sec |
| **Latency** | 100ms | 10ms | 5ms |
| **Ordering** | Per partition | Per queue | No guaranteed |
| **Replication** | Built-in | Plugin | Cluster mode |
| **Use case** | Event streaming | Task queues | Real-time messaging |

---

## Related Fundamentals

- [Message Queue Basics](message-queue-basics.md) – Delivery semantics, ordering
- [Event Sourcing](event-sourcing.md) – Kafka as event store
- [Reliability](../reliability-and-resiliency/) – Exactly-once semantics

---

**Status**: ✅ Complete. Covers architecture, replication, consumers, scaling, anti-patterns.

