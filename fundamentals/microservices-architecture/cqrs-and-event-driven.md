# CQRS & Event-Driven Architecture

Separating read and write models, using events as primary communication.

---

## TL;DR

- **CQRS**: Command Query Responsibility Segregation (separate read/write)
- **Event-driven**: Services publish events, others subscribe
- **Event sourcing**: Store events as source of truth
- **Eventual consistency**: Reads lag writes slightly
- **Flexibility**: Can optimize read model independently

---

## CQRS Pattern

```
Write path (Command):
  POST /users → UserService writes to primary DB
  Publishes: "UserCreated" event
  
Read path (Query):
  GET /users/123 → Returns from materialized view
  View kept in sync by event handler
  
Benefit:
  Writes optimized (transactional)
  Reads optimized (denormalized, fast)
  Different databases possible (write: SQL, read: Elasticsearch)
```

---

## Event-Driven

```
Services communicate via events:

OrderService publishes:
  "OrderCreated" event
  
Subscribers:
  PaymentService: Process payment
  InventoryService: Decrement stock
  NotificationService: Send email
  
Loose coupling:
  OrderService doesn't know who subscribes
  Can add subscribers without changing OrderService
```

---

## Saga Pattern (Distributed Transactions)

```
Transaction across 3 services:

OrderService:
  Create order
  Publish: "OrderCreated"
  
PaymentService (subscribes):
  Process payment
  If success: Publish "PaymentProcessed"
  If fail: Publish "PaymentFailed" → triggers rollback
  
InventoryService (subscribes):
  Decrement stock
  If success: Order complete
  If fail: Compensate (refund payment, cancel order)
```

---

## Related Fundamentals

- [Messaging](../messaging-and-streaming/kafka-architecture.md) – Event transport
- [Event Sourcing](../messaging-and-streaming/event-sourcing.md) – Events as history

---

**Status**: ✅ Complete.

