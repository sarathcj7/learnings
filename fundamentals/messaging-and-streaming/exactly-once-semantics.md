# Exactly-Once Semantics (EOS)

Delivering messages exactly one time despite failures. More complex than it sounds.

---

## TL;DR

- **Exactly-once**: Message processed precisely once (no dupes, no loss)
- **Hard in distributed systems**: Requires coordination across producer, queue, consumer
- **Practical approach**: At-least-once + idempotency
- **Use for**: Financial transactions, critical updates
- **Trade-off**: Complexity and latency vs simplicity and speed

---

## Why Exactly-Once is Hard

### Failure Modes

```
Producer sends message:
  1. Message goes to queue
  2. Producer crashes before marking sent
  3. Producer restarts, retries
  4. Queue now has duplicate
  
Queue delivers to consumer:
  1. Message delivered to consumer
  2. Consumer processes it
  3. Consumer crashes before acknowledging
  4. Queue thinks it failed, redelivers
  
Consumer processes:
  1. Processes message
  2. Acknowledges to queue
  3. Queue loses the ack (network failure)
  4. Redelivers (consumer processes twice)
```

---

## Achieving Exactly-Once

### Approach 1: Idempotent Processing (Recommended)

```
At-least-once delivery + idempotency key:

Producer:
  message_id = uuid()
  Send { data, message_id } to queue
  
Queue:
  Delivers to consumer(s)
  May deliver multiple times (at-least-once)
  
Consumer:
  On receive:
    Check: Have we processed this message_id before?
    Yes → Return cached result (no duplicate processing)
    No → Process and store { message_id, result }
  
Result:
  Duplicate delivery → Same processing (idempotent)
  Effective exactly-once without distributed coordination
```

---

### Approach 2: Distributed Transactions (Complex)

```
Transactional processing:

Queue & Consumer & Database in single transaction:
  1. Read message from queue
  2. Process message (update database)
  3. Acknowledge to queue
  4. COMMIT all or ROLLBACK all
  
If consumer crashes:
  Transaction rolls back
  Message stays in queue
  Redelivered to another consumer
  
Challenges:
  Requires 2-phase commit (complex, slow)
  Requires database + queue to support transactions (not all do)
  High latency (waiting for coordination)
```

---

## Stream Processing: Exactly-Once

### Kafka Streams / Spark Streaming

```
Framework handles exactly-once:

Checkpoint mechanism:
  1. Read batch of messages from Kafka partition
  2. Process batch
  3. Commit offset (checkpoint) atomically with processing
  
If failure:
  Restart from last checkpoint
  Reprocess from that point
  Framework deduplicates (via offset)
  
Implementation:
  SELECT COUNT(*) FROM events
  After failure, rerun, get same COUNT (no double-counting)
```

---

## Financial Example: Payment Processing

### Traditional Approach (Risky)

```
Customer clicks: "Send $100"

Application:
  1. Debit customer account
  2. Publish event: "Transfer initiated"
  3. Process event
  4. Credit recipient
  5. Return success

If step 3 fails (process event):
  Money debited but not credited (lost!)
  If retry: Credit twice (duplicate!)
```

---

### Exactly-Once Approach

```
Customer clicks: "Send $100"

Transaction boundary:
  BEGIN
  1. Debit customer account
  2. Create event (in same database)
  3. Mark as "pending processing"
  COMMIT

Event handler:
  Reads: "Transfer pending" events
  For each:
    Credit recipient
    Mark as "processed"
    If fails: Retry (idempotent, checking if already processed)
  COMMIT

Result:
  Money debited once
  Money credited once
  No race conditions, no duplicates
```

---

## Exactly-Once with Kafka

### Transactional Writes

```
Kafka 0.11+ supports transactions:

Producer:
  BEGIN TRANSACTION
  Send event 1
  Send event 2
  Send event 3
  COMMIT

Result:
  All 3 events written atomically
  No partial writes
  Consumers see all or nothing
```

---

### Consuming & Processing

```
Consumer group:
  Process messages
  Commit offset atomically with processing
  
Kafka guarantees:
  If consumer crashes before committing offset:
    Message redelivered (no loss)
  If consumer commits and then crashes:
    Offset saved (message won't be reprocessed)
    
Key: Atomic commit (offset + processing result)
```

---

## Trade-offs Summary

| Approach | Latency | Complexity | Correctness |
|---|---|---|---|
| **At-most-once** | Low | Low | Lossy (unreliable) |
| **At-least-once + idempotency** | Low | Medium | Exactly-once (practical) |
| **Exactly-once (distributed TX)** | High | High | Exactly-once (theoretical) |

---

## When to Use Exactly-Once

### Financial Transactions (Must Use)

```
Payment system:
  ✅ Exactly-once required
  ❌ At-least-once unacceptable (double-charge!)
  
Implementation: Idempotency keys or distributed transactions
```

---

### Metrics & Analytics (At-Least-Once OK)

```
User analytics:
  "Page view count is 1,000,001 (expected 1,000,000)"
  Off by 1 event: Usually acceptable
  
Implementation: At-least-once (simpler)
```

---

### Operational Events (At-Least-Once OK)

```
Log aggregation:
  "CPU usage 60%"
  Duplicate entry: Usually harmless
  
Implementation: At-least-once
```

---

## Production Checklist

- [ ] Identify exactly-once requirements per flow
- [ ] Implement idempotency keys for critical operations
- [ ] Choose queue with atomic commit support
- [ ] Consumer tracks processed messages
- [ ] Handle duplicates gracefully
- [ ] Monitor for processing delays (lag)
- [ ] Test failure scenarios (producer crash, consumer crash)
- [ ] Document exactly-once guarantees

---

## Related Fundamentals

- [Kafka Architecture](kafka-architecture.md) – Transactions, offsets
- [Idempotency](../apis-and-communication/idempotency-and-api-design.md) – Practical exactly-once
- [Reliability](../reliability-and-resiliency/) – Handling failures

---

**Status**: ✅ Complete. Covers idempotency, distributed transactions, stream processing, trade-offs.

