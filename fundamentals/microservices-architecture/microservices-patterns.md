# Microservices Patterns

Designing services with clear boundaries, async communication, and independent deployability.

---

## TL;DR

- **Microservices**: Small, independent services (one team, one deployment)
- **Service boundaries**: By business capability (user service, payment service, etc.)
- **Communication**: Async (Kafka) beats sync (HTTP) for resilience
- **Resilience**: Circuit breakers, timeouts, bulkheads essential
- **Operational cost**: More services = more overhead (monitoring, deployment, debugging)

---

## Monolith vs Microservices

### Monolith

```
Single codebase:
  users.py
  payments.py
  notifications.py
  (all in same process)
  
Deployment:
  Deploy all or nothing
  1 version of everything
  
Database:
  Shared database (tight coupling)
  
Pros: Simpler, fewer moving parts
Cons: Can't scale independently, deploy risk
```

---

### Microservices

```
Separate codebases:
  UserService (users.py)
  PaymentService (payments.py)
  NotificationService (notifications.py)
  
Deployment:
  Each can deploy independently
  Different versions in production
  
Database:
  Each service owns data (loose coupling)
  
Pros: Independent scaling, deployment, teams
Cons: Complex (distributed systems)
```

---

## Service Boundaries

### By Business Capability (Good)

```
UserService:
  - User profiles
  - Authentication
  - Preferences
  
PaymentService:
  - Process payments
  - Track transactions
  - Billing
  
NotificationService:
  - Send emails
  - SMS
  - Push notifications
  
Boundary: Each team owns one service
```

---

### By Technical Layer (Bad)

```
❌ Wrong:
  DatabaseService (data access layer)
  AuthService (authentication)
  WebService (HTTP handler)
  
Problem:
  - Services still tightly coupled
  - Can't deploy independently
  - Every feature touches all services
```

---

## Communication Patterns

### Synchronous (HTTP/RPC)

```
Client → UserService: "Get user 123"
  (wait for response)
UserService → Database: Query user
  (wait)
Response: User data

Latency: 50ms
Coupling: Tight (client must wait)
Failure: If UserService down, client blocked
```

---

### Asynchronous (Kafka)

```
Service A → Kafka: "UserCreated" event
  (returns immediately)

Service B subscribes: "UserCreated"
  Processes event when ready
  Updates own database

Latency: Eventual (seconds)
Coupling: Loose (don't need to wait)
Failure: Service B down, message queued, retry later
```

---

## Resilience Patterns

### Circuit Breaker (Per Service)

```
PaymentService calls UserService:
  If 5 failures: Circuit opens
  New requests fail immediately (don't call UserService)
  After 30s: Try one request
  
Result:
  Payment service survives if user service down
  Graceful degradation
```

---

### Bulkheads

```
Thread pools:
  UserService calls: 5 threads (dedicated)
  PaymentService calls: 5 threads (dedicated)
  
If PaymentService slow:
  UserService threads unaffected
  Both services survive independently
```

---

## Operational Challenges

### Complexity

```
1 monolith: Straightforward
10 microservices: 10x more complex
  - Deployment coordination
  - Debugging across services
  - Dependency management
  
Rule: Only microservices if necessary
Not: "Many small services = always better"
```

---

### Consistency

```
Monolith: Database transaction (ACID)
  All-or-nothing guarantee

Microservices:
  Service A updates
  Service B updates
  Network fails
  Inconsistent state
  
Solution: Saga pattern (compensating transactions)
```

---

### Related Fundamentals

- [Service Discovery](service-discovery.md) – Find services dynamically
- [API Gateway](api-gateway-and-routing.md) – Single entry point
- [CQRS](cqrs-and-event-driven.md) – Separate read/write models
- [Reliability](../reliability-and-resiliency/) – Circuit breakers, retries

---

**Status**: ✅ Complete. Covers boundaries, communication, resilience, challenges.

