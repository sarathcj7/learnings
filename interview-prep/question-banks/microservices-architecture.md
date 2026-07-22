# Microservices Architecture — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## Service Boundaries & Domain-Driven Design

1. What's a microservice? What's too small vs too large?
2. Explain domain-driven design (DDD). How does it relate to microservices?
3. Your system has 50 microservices. Is this a problem? How many should you have?
4. Design service boundaries for an e-commerce platform (orders, payments, inventory, etc).
5. What's the difference between vertical decomposition (by feature) vs horizontal (by layer)?
6. Should each microservice have its own database? Why or why not?
7. Your services share a common database. Can you still call it microservices?
8. Design services for a ride-sharing app (drivers, riders, matching, payments).
9. A service has "users". Should user management be a separate service?
10. How do you split a monolith into microservices? Where do you start?

---

## Communication Patterns

11. Compare synchronous (RPC, REST, gRPC) vs asynchronous (messaging) communication.
12. When would you use REST vs gRPC for service-to-service communication?
13. Your payment service calls inventory service synchronously. It fails 0.1% of the time. How do you handle failures?
14. Design communication for a payment flow: order service -> payment service -> inventory service.
15. What's the relationship between microservices and eventual consistency?
16. Should services communicate directly or through a message bus? Pros and cons?
17. Your service A calls B calls C (chain of calls). C is slow. What's the impact on A?
18. Design asynchronous communication for a user signup flow (email verification, analytics).
19. Compare HTTP REST vs gRPC vs message queues for microservices.
20. What's the difference between choreography and orchestration in microservices?

---

## Reliability & Resilience

21. Explain the circuit breaker pattern. When would you use it?
22. Your circuit breaker is open (failing fast). How long should it stay open?
23. Explain retry logic. When should you retry? Immediately or with backoff?
24. What's the cascading failure problem in microservices?
25. Your service A depends on services B, C, D. If any fails, A fails. How would you design resilience?
26. Design a bulkhead pattern (isolate failures) for a system with many services.
27. What's a fallback? When would you provide one instead of failing?
28. Explain timeout handling in microservices. What timeout should you set?
29. Design a system that degrades gracefully when a service is down.
30. What's the timeout vs retry vs circuit breaker trade-off?

---

## Service Discovery & Orchestration

31. What's service discovery? Why is it needed in microservices?
32. Compare client-side vs server-side service discovery.
33. Your service needs to find the payment service. How does it discover it?
34. Explain service registry. What data does it store?
35. Design service discovery for services across multiple datacenters.
36. What happens when a service instance crashes? How is it removed from discovery?
37. Compare Kubernetes service discovery vs Consul vs Eureka.
38. Your service connects to wrong payment service instance (stale cache). How do you prevent it?
39. Design health checking for service discovery.
40. What's the difference between service discovery and load balancing?

---

## Kubernetes & Container Orchestration

41. What's Kubernetes? What problems does it solve?
42. Explain pods, deployments, and services in Kubernetes.
43. Design a Kubernetes deployment for a web service (scaling, updates, health checks).
44. What's a sidecar container? When would you use it?
45. Explain Kubernetes namespaces. How would you use them?
46. Your service crashes every 10 minutes. How would Kubernetes help? What would you configure?
47. Design Kubernetes configuration for a microservice that scales based on load.
48. What's a DaemonSet? When would you use it vs Deployment?
49. Explain init containers. What would you use them for?
50. Design zero-downtime deployment in Kubernetes.

---

## Data Consistency & Transactions

51. Explain the distributed transaction problem in microservices.
52. What's the saga pattern? How does it coordinate transactions across services?
53. Compare orchestration sagas vs choreography sagas.
54. Your order service transacts with payment and inventory. Design it.
55. What's eventual consistency? How do you ensure correct behavior with it?
56. Your payment succeeds but inventory update fails. How do you recover?
57. Design a system where ordering and payment must be atomic, but inventory is eventual.
58. What's the difference between ACID transactions and saga pattern?
59. How would you handle compensation (reversing a payment)?
60. Design consistency checking for eventual consistency (periodic reconciliation).

---

## API Gateway & Cross-Cutting Concerns

61. What's an API gateway? What should it do?
62. Design an API gateway for multiple client types (mobile, web, third-party).
63. Your API gateway does rate limiting, authentication, logging. Is this too much? Why?
64. Compare API gateway vs service mesh.
65. Should API gateway do authorization or should services?
66. Design API versioning with multiple backend services.
67. Your gateway talks to 100 services. How do you maintain it?
68. What's a BFF (Backend for Frontend)? When would you use it?
69. Design a gateway that handles multiple business domains.
70. How would you handle partial service failures in the gateway?

---

## Service Mesh & Observability

71. What's a service mesh? What problems does it solve?
72. Compare Istio vs Linkerd vs Consul Connect.
73. Explain sidecar proxies. How do they intercept traffic?
74. Design a service mesh for canary deployments (gradual rollout).
75. What's distributed tracing across a service mesh?
76. Your service mesh adds 50ms latency. Is it worth it?
77. Design observability for 50 microservices.
78. How would you debug a slow request across services?

---

## Synthesis & Cross-Cutting

79. Design a complete microservices architecture for Netflix (video, recommendations, payments, subscriptions).
80. Your monolith has 3 databases (users, orders, inventory). Design migration to microservices.
81. Design microservices for an online marketplace (sellers, buyers, payments, shipping).
82. Your system is 100 microservices. Developers are slowed down by coordination overhead. How do you fix it?
83. Design a system that handles multi-region deployment of microservices.

---

## Advanced & Tricky Questions

84. What's the monolith vs microservices trade-off? When is monolith better?
85. Explain Conway's Law: how does organization structure affect architecture?
86. Design a gradual migration from monolith to microservices.
87. Your services have circular dependencies. How do you detect and fix?
88. Explain the "distributed monolith" anti-pattern. How would you avoid it?
89. Design a system where services need to coordinate without tight coupling.
90. What's the relationship between microservices and DevOps?
91. Compare microservices vs modular monolith.
92. Design microservices for offline-first mobile app (sync, conflicts).
93. Explain testing strategies for microservices (unit, integration, contract, e2e).
94. Design a system that can scale services independently.
95. What's the cost of microservices in operational complexity? When is it justified?

---

## Self-Scoring Guide

- **0-25 answered well**: Not ready for interviews on microservices. Study fundamentals first.
- **26-50 answered well**: You know the basics. Study weak areas and resilience patterns.
- **51-95 answered well**: You're interview-ready on microservices.
