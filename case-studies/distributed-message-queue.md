# Distributed Message Queue (Kafka-Style)

*Design Apache Kafka: billions of messages, consumer groups, partitions, fault tolerance, retention.*

## Problem Statement

Build a distributed message queue (like Kafka). Process 1M+ messages/sec. Multiple consumers can process same topic. Messages persisted for days/weeks. Must be fault-tolerant.

## Architecture

```
Topics (logical):
  Topic = "user-events"
  
Partitions (physical):
  Topic has 10 partitions (0-9)
  Message hashed to partition: hash(key) % 10
  
Broker Cluster:
  Broker 1: Partitions {0, 3, 6, 9}
  Broker 2: Partitions {1, 4, 7}
  Broker 3: Partitions {2, 5, 8}
  (each partition replicated to 2+ brokers)
  
Producer:
  Send message → Broker holding partition
  Async replicate to replicas
  
Consumer Group:
  Group "analytics": Consumers {A, B, C}
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C
  (load balanced across group)
  
Consumer tracks offset:
  "I've read up to offset 1000"
  On reconnect: Start from offset 1000
```

## Durability

```
Message persisted:
  - Written to primary broker (disk)
  - Replicated to in-sync replicas
  - Ack when majority replicated
  
Retention:
  - Keep messages for 7 days
  - Automatic cleanup (retention policy)
```

## Bottlenecks & Scaling

**Bottleneck**: Disk I/O (writing to 1M messages/sec).

**Solution**:
- Batch writes (group messages, write once)
- Sequential disk I/O (fast)
- Compression (messages before writing)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Partitioning** | Single queue | Parallelism across consumers |
| **Consumer groups** | Single consumer | Multiple independent groups process same topic |
| **Persistent log** | In-memory queue | Fault tolerance, replay capability |

---

**Status**: ✅ Complete. Shows distributed queue, partitioning, consumer groups.
