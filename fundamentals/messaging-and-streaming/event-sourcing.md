# Event Sourcing

Storing the complete history of events and deriving state from them instead of just storing current state.

---

## TL;DR

- **Event sourcing**: Store events (immutable), derive state from events
- **Append-only log**: Only inserts, no updates/deletes
- **Advantages**: Complete audit trail, temporal queries, easy replay
- **Disadvantages**: Complex reads, eventual consistency, large storage
- **Use with**: Kafka, DynamoDB Streams, EventStoreDB

---

## Traditional Approach vs Event Sourcing

### Traditional (Current State)

```
Database table: accounts
  id | user_id | balance | updated_at
  1  | 100     | $500    | 2026-01-01
  
Update: "Transfer $100"
  UPDATE accounts SET balance = 400 WHERE id = 1
  
Current state: $400
History: LOST (can't recover previous balance)
```

---

### Event Sourcing (Events Only)

```
Event log: account_events
  id | account_id | event_type    | amount | timestamp
  1  | 1          | Opened        | $1000  | 2026-01-01
  2  | 1          | Withdrawn     | $200   | 2026-01-02
  3  | 1          | Deposited     | $100   | 2026-01-03
  4  | 1          | Withdrawn     | $100   | 2026-01-04
  
Current balance = $1000 - $200 + $100 - $100 = $800

History: Complete (can query any point in time)
Replay: Can recreate state at any past date
Audit: Every change recorded
```

---

## Advantages of Event Sourcing

### Complete Audit Trail

```
Regulatory requirement: "Keep 7-year audit trail"

Event sourcing:
  Automatic (every event is recorded)
  Immutable (can't alter past events)
  
Traditional:
  Need separate audit log
  Manual tracking
  Risk of missing events
```

---

### Temporal Queries

```
Question: "What was my account balance on 2026-01-01?"

Event sourcing:
  Replay events up to 2026-01-01
  Get balance at that point in time
  No extra storage, derived on-demand
  
Traditional:
  Need to store historical snapshots
  Extra storage, complexity
```

---

### Easy Debugging

```
Bug discovered: "Charges doubled in March"

Event sourcing:
  Query: What events happened for each user in March?
  Replay: Reconstruct exactly what happened
  Debug: Find the bug in event handling
  
Traditional:
  Logs only, current state is wrong
  Can't recover what happened
```

---

## Implementing Event Sourcing

### Writing Events

```
User initiates: "Transfer $50"

Application:
  1. Validate: Sufficient balance? Account active?
  2. Create event: TransferInitiated { from: 1, to: 2, amount: 50 }
  3. Write to event log (Kafka, EventStoreDB)
  
Event log:
  Event ID: 1001
  Type: TransferInitiated
  Payload: { from: 1, to: 2, amount: 50 }
  Timestamp: 2026-01-01T10:00:00Z
  
Return: Success
```

---

### Deriving State (Event Handler)

```
Event handler subscribes to: TransferInitiated events

On event received:
  1. Read event: TransferInitiated { from: 1, to: 2, amount: 50 }
  2. Update state:
     accounts[1].balance -= 50
     accounts[2].balance += 50
  3. Update denormalized view (for fast queries)
  
Result: Current state derived from events
```

---

### Snapshots (Optimization)

```
Problem: Replaying 10 years of events = slow!

Solution: Snapshots
  Every 1000 events: Save snapshot
  "At event 10000, account balance was $5000"
  
Replay:
  1. Load snapshot at event 10000
  2. Replay events 10001-10500 (only 500, not 10500)
  
Result: Fast initialization
```

---

## Event Sourcing Patterns

### Pattern 1: CQRS (Command Query Responsibility Segregation)

```
Write path (Event Sourcing):
  Command: TransferMoney
  → Validates, creates event
  → Event log (append-only)
  → Event handlers update denormalized views
  
Read path (Optimized):
  Query: GetAccountBalance
  → Read from denormalized view (fast, indexed)
  → No event replay needed
  
Benefit:
  Writes optimized (append)
  Reads optimized (denormalized)
```

---

### Pattern 2: Event Versioning

```
Version 1 of TransferInitiated:
  { from, to, amount }
  
Version 2 (new requirement, tracking approval):
  { from, to, amount, approver_id, approval_status }
  
Handling:
  Old events (v1): Migrate on read or have adapter
  New events (v2): Have full approval fields
  
Benefit:
  Can evolve schema without rewriting history
```

---

## Challenges & Solutions

### Challenge 1: Large Event Log

```
Problem:
  After 10 years: Billions of events
  Replaying all = slow
  Storage = expensive

Solution:
  Snapshots (periodic compaction)
  Archive old events (cold storage)
  Log compaction (Kafka: keep only latest per key)
```

---

### Challenge 2: Event Versioning & Migration

```
New requirement: Add tax_amount to transfer events

Old events:
  { from: 1, to: 2, amount: 50 }  (no tax_amount)

New events:
  { from: 1, to: 2, amount: 50, tax_amount: 2.50 }

Handling:
  Adapter pattern: When reading old events, compute tax_amount = 0
  Migrate on write: Convert old to new format on first read
  
Benefit: No bulk rewrite of 10-year history
```

---

### Challenge 3: Eventual Consistency

```
User transfers $100:
  1. Event written to log
  2. Event handler updates balance (async)
  3. User queries balance
  
Timing:
  T=0ms: Event written
  T=0-100ms: Event handler processing (delay)
  T=50ms: User queries balance (might be old!)
  
Solution:
  Accept eventual consistency (balance updates within 100ms)
  Or: Return event ID and have user poll until processed
```

---

## Event Sourcing vs CRUD

| Aspect | CRUD | Event Sourcing |
|---|---|---|
| **Storage** | Current state | All events |
| **Updates** | In-place | Append events |
| **History** | No | Complete |
| **Audit** | Manual | Automatic |
| **Consistency** | Immediate | Eventual |
| **Complexity** | Simple | Moderate |
| **Time queries** | Hard | Easy |
| **Debugging** | Hard | Easy |

---

## When to Use Event Sourcing

### Use Event Sourcing When:

✅ Audit trail required (financial, legal)  
✅ Temporal queries ("state at time X")  
✅ Complex state transitions (state machine)  
✅ Event-driven architecture naturally  
✅ Debugging/investigation important  

### Use CRUD When:

❌ Simple CRUD operations  
❌ No audit requirement  
❌ Performance (heavy writes)  
❌ Teams unfamiliar with eventual consistency  

---

## Related Fundamentals

- [Kafka Architecture](kafka-architecture.md) – Append-only event log
- [CQRS Pattern](../microservices-architecture/cqrs-pattern.md) – Read/write separation
- [Reliability](../reliability-and-resiliency/) – Event ordering guarantees

---

**Status**: ✅ Complete. Covers implementation, patterns, challenges, when to use.

