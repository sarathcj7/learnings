# Mastering System Design

A comprehensive, dual-purpose knowledge base for **system design interview preparation** and **production-grade architectural mastery**. Optimized for advanced engineers who already know the fundamentals and want exhaustive depth, edge cases, and real-world patterns.

## Purpose & Philosophy

This repo serves two equal goals:
1. **Interview Readiness**: A structured, repeatable problem-solving framework and 20 classic + advanced case studies for FAANG-style system design rounds.
2. **Mastery**: Deep, production-oriented knowledge of distributed systems theory, consensus algorithms, trade-offs, and scaling patterns that inform real architectural decisions.

The structure is "fundamentals first, then case studies that apply them" — you build mental models on solid theory, then practice synthesizing them under pressure.

## How to Use This Repo

1. **For interview prep**: Start with [Problem-Solving Framework](case-studies/00-problem-solving-framework.md), then practice each case study while referencing its linked fundamentals.
2. **For mastery**: Work through the fundamentals tracker below in the suggested order. For each topic:
   - Read the sub-files (marked "Reviewed")
   - Test yourself with the question bank (marked "Practiced")
   - Use flashcards for spaced-repetition recall
   - Apply the topic in a related case study (marked "Mastered")
3. **Status tracking**: Update your progress in the tables below as you go. Status values: **Not started** / **Reviewed** / **Practiced** / **Mastered**.

## Repo Structure

- **`fundamentals/`**: 20 topic folders, each with 3–6 granular sub-files covering theory, mechanics, trade-offs, production considerations, and real-world examples. 91 files total.
- **`case-studies/`**: 20 complete system design problems (13 core for interviews, 7 advanced for mastery). Includes a problem-solving framework and case-study template.
- **`interview-prep/`**: Question banks (20 files, one per fundamentals topic) and flashcards (20 files) for self-testing and spaced-repetition recall.

## Status Legend

| Status | Criteria | Symbol |
|---|---|---|
| Not started | Not yet engaged|
| Reviewed | Read once, understand high-level concepts | 📖 |
| Practiced | Tested via question bank/flashcards or applied in a case study | 💪 |
| Mastered | Can explain from scratch, handle edge cases, teach it unprompted | 🎯 |

---

## Progress Overview — Fundamentals

| # | Topic |
|---|---|
| 1 | [Networking & Protocols](fundamentals/networking-and-protocols/) |
| 2 | [APIs & Communication](fundamentals/apis-and-communication/) |
| 3 | [Scalability & Load Balancing](fundamentals/scalability-and-load-balancing/) |
| 4 | [Caching](fundamentals/caching/) |
| 5 | [Databases](fundamentals/databases/) |
| 6 | [Distributed Systems Theory](fundamentals/distributed-systems-theory/) |
| 7 | [Consensus & Coordination](fundamentals/consensus-and-coordination/) |
| 8 | [Distributed Transactions](fundamentals/distributed-transactions/) |
| 9 | [Messaging & Streaming](fundamentals/messaging-and-streaming/) |
| 10 | [Microservices Architecture](fundamentals/microservices-architecture/) |
| 11 | [Rate Limiting & Traffic Management](fundamentals/rate-limiting-and-traffic-management/) |
| 12 | [Distributed Data Structures](fundamentals/distributed-data-structures/) |
| 13 | [Storage Systems](fundamentals/storage-systems/) |
| 14 | [Search & Indexing](fundamentals/search-and-indexing/) |
| 15 | [Observability](fundamentals/observability/) |
| 16 | [Security](fundamentals/security/) |
| 17 | [Reliability & Resiliency](fundamentals/reliability-and-resiliency/) |
| 18 | [Batch & Stream Processing](fundamentals/batch-and-stream-processing/) |
| 19 | [Content Delivery & Edge](fundamentals/content-delivery-and-edge/) |
| 20 | [Capacity Planning & Estimation](fundamentals/capacity-planning-and-estimation/) |

---

## Fundamentals — Detailed Tracker

### 1. Networking & Protocols
[Question Bank](interview-prep/question-banks/networking-and-protocols.md) · [Flashcards](interview-prep/flashcards/networking-and-protocols.md)

| Sub-topic | File |
|---|---|
| DNS & Service Discovery | [dns-and-service-discovery.md](fundamentals/networking-and-protocols/dns-and-service-discovery.md) |
| TCP/UDP & Transport | [tcp-udp-and-transport.md](fundamentals/networking-and-protocols/tcp-udp-and-transport.md) |
| HTTP Versions & Semantics | [http-versions-and-semantics.md](fundamentals/networking-and-protocols/http-versions-and-semantics.md) |
| TLS & PKI | [tls-and-pki.md](fundamentals/networking-and-protocols/tls-and-pki.md) |
| WebSockets & Real-time Transport | [websockets-and-realtime-transport.md](fundamentals/networking-and-protocols/websockets-and-realtime-transport.md) |

### 2. APIs & Communication
[Question Bank](interview-prep/question-banks/apis-and-communication.md) · [Flashcards](interview-prep/flashcards/apis-and-communication.md)

| Sub-topic | File |
|---|---|
| REST Design | [rest-design.md](fundamentals/apis-and-communication/rest-design.md) |
| GraphQL Fundamentals | [graphql-fundamentals.md](fundamentals/apis-and-communication/graphql-fundamentals.md) |
| gRPC & Protobuf | [grpc-and-protobuf.md](fundamentals/apis-and-communication/grpc-and-protobuf.md) |
| API Versioning & Evolution | [versioning-and-evolution.md](fundamentals/apis-and-communication/versioning-and-evolution.md) |
| Pagination & Idempotency | [pagination-and-idempotency.md](fundamentals/apis-and-communication/pagination-and-idempotency.md) |

### 3. Scalability & Load Balancing
[Question Bank](interview-prep/question-banks/scalability-and-load-balancing.md) · [Flashcards](interview-prep/flashcards/scalability-and-load-balancing.md)

| Sub-topic | File |
|---|---|
| Horizontal vs Vertical Scaling | [horizontal-vs-vertical-scaling.md](fundamentals/scalability-and-load-balancing/horizontal-vs-vertical-scaling.md) |
| Load Balancing Algorithms | [load-balancing-algorithms.md](fundamentals/scalability-and-load-balancing/load-balancing-algorithms.md) |
| L4/L7 & Global Load Balancing | [l4-l7-and-global-load-balancing.md](fundamentals/scalability-and-load-balancing/l4-l7-and-global-load-balancing.md) |
| Scaling Patterns & Bottlenecks | [scaling-patterns-and-bottlenecks.md](fundamentals/scalability-and-load-balancing/scaling-patterns-and-bottlenecks.md) |
| Statelessness & Session Management | [statelessness-and-session-management.md](fundamentals/scalability-and-load-balancing/statelessness-and-session-management.md) |

### 4. Caching
[Question Bank](interview-prep/question-banks/caching.md) · [Flashcards](interview-prep/flashcards/caching.md)

| Sub-topic | File |
|---|---|
| Caching Strategies | [strategies.md](fundamentals/caching/strategies.md) |
| Eviction Policies | [eviction-policies.md](fundamentals/caching/eviction-policies.md) |
| CDN Caching | [cdn.md](fundamentals/caching/cdn.md) |
| Distributed Caching | [distributed-caching.md](fundamentals/caching/distributed-caching.md) |
| Cache Invalidation | [invalidation.md](fundamentals/caching/invalidation.md) |

### 5. Databases
[Question Bank](interview-prep/question-banks/databases.md) · [Flashcards](interview-prep/flashcards/databases.md)

| Sub-topic | File |
|---|---|
| Relational Fundamentals | [relational-fundamentals.md](fundamentals/databases/relational-fundamentals.md) |
| Indexing | [indexing.md](fundamentals/databases/indexing.md) |
| Replication | [replication.md](fundamentals/databases/replication.md) |
| Sharding & Partitioning | [sharding-and-partitioning.md](fundamentals/databases/sharding-and-partitioning.md) |
| Transactions & Isolation Levels | [transactions-and-isolation-levels.md](fundamentals/databases/transactions-and-isolation-levels.md) |
| NoSQL & NewSQL Landscape | [nosql-and-newsql-landscape.md](fundamentals/databases/nosql-and-newsql-landscape.md) |

### 6. Distributed Systems Theory
[Question Bank](interview-prep/question-banks/distributed-systems-theory.md) · [Flashcards](interview-prep/flashcards/distributed-systems-theory.md)

| Sub-topic | File |
|---|---|
| CAP & PACELC | [cap-and-pacelc.md](fundamentals/distributed-systems-theory/cap-and-pacelc.md) |
| Consistency Models | [consistency-models.md](fundamentals/distributed-systems-theory/consistency-models.md) |
| Time, Clocks & Ordering | [time-clocks-and-ordering.md](fundamentals/distributed-systems-theory/time-clocks-and-ordering.md) |
| Quorum Systems | [quorum-systems.md](fundamentals/distributed-systems-theory/quorum-systems.md) |

### 7. Consensus & Coordination
[Question Bank](interview-prep/question-banks/consensus-and-coordination.md) · [Flashcards](interview-prep/flashcards/consensus-and-coordination.md)

| Sub-topic | File |
|---|---|
| Paxos | [paxos.md](fundamentals/consensus-and-coordination/paxos.md) |
| Raft | [raft.md](fundamentals/consensus-and-coordination/raft.md) |
| Leader Election | [leader-election.md](fundamentals/consensus-and-coordination/leader-election.md) |
| Distributed Locks | [distributed-locks.md](fundamentals/consensus-and-coordination/distributed-locks.md) |
| Coordination Services | [coordination-services.md](fundamentals/consensus-and-coordination/coordination-services.md) |

### 8. Distributed Transactions
[Question Bank](interview-prep/question-banks/distributed-transactions.md) · [Flashcards](interview-prep/flashcards/distributed-transactions.md)

| Sub-topic | File |
|---|---|
| Two-Phase & Three-Phase Commit | [two-phase-and-three-phase-commit.md](fundamentals/distributed-transactions/two-phase-and-three-phase-commit.md) |
| Saga Pattern | [saga-pattern.md](fundamentals/distributed-transactions/saga-pattern.md) |
| Outbox & Exactly-Once | [outbox-and-exactly-once.md](fundamentals/distributed-transactions/outbox-and-exactly-once.md) |
| Choosing a Transaction Strategy | [choosing-a-transaction-strategy.md](fundamentals/distributed-transactions/choosing-a-transaction-strategy.md) |

### 9. Messaging & Streaming
[Question Bank](interview-prep/question-banks/messaging-and-streaming.md) · [Flashcards](interview-prep/flashcards/messaging-and-streaming.md)

| Sub-topic | File |
|---|---|
| Message Queues | [message-queues.md](fundamentals/messaging-and-streaming/message-queues.md) |
| Pub-Sub Systems | [pub-sub-systems.md](fundamentals/messaging-and-streaming/pub-sub-systems.md) |
| Event Streaming Platforms | [event-streaming-platforms.md](fundamentals/messaging-and-streaming/event-streaming-platforms.md) |
| Delivery Semantics | [delivery-semantics.md](fundamentals/messaging-and-streaming/delivery-semantics.md) |
| Event Sourcing & CQRS | [event-sourcing-and-cqrs.md](fundamentals/messaging-and-streaming/event-sourcing-and-cqrs.md) |

### 10. Microservices Architecture
[Question Bank](interview-prep/question-banks/microservices-architecture.md) · [Flashcards](interview-prep/flashcards/microservices-architecture.md)

| Sub-topic | File |
|---|---|
| Monolith vs Microservices | [monolith-vs-microservices.md](fundamentals/microservices-architecture/monolith-vs-microservices.md) |
| Service Decomposition Patterns | [service-decomposition-patterns.md](fundamentals/microservices-architecture/service-decomposition-patterns.md) |
| Service Discovery & API Gateway | [service-discovery-and-api-gateway.md](fundamentals/microservices-architecture/service-discovery-and-api-gateway.md) |
| Service Mesh | [service-mesh.md](fundamentals/microservices-architecture/service-mesh.md) |
| Sync vs Async Communication | [sync-vs-async-communication.md](fundamentals/microservices-architecture/sync-vs-async-communication.md) |

### 11. Rate Limiting & Traffic Management
[Question Bank](interview-prep/question-banks/rate-limiting-and-traffic-management.md) · [Flashcards](interview-prep/flashcards/rate-limiting-and-traffic-management.md)

| Sub-topic | File |
|---|---|
| Algorithms | [algorithms.md](fundamentals/rate-limiting-and-traffic-management/algorithms.md) |
| Distributed Rate Limiting | [distributed-rate-limiting.md](fundamentals/rate-limiting-and-traffic-management/distributed-rate-limiting.md) |
| Backpressure & Load Shedding | [backpressure-and-load-shedding.md](fundamentals/rate-limiting-and-traffic-management/backpressure-and-load-shedding.md) |
| Throttling & Quotas | [throttling-and-quotas.md](fundamentals/rate-limiting-and-traffic-management/throttling-and-quotas.md) |

### 12. Distributed Data Structures
[Question Bank](interview-prep/question-banks/distributed-data-structures.md) · [Flashcards](interview-prep/flashcards/distributed-data-structures.md)

| Sub-topic | File |
|---|---|
| Consistent Hashing | [consistent-hashing.md](fundamentals/distributed-data-structures/consistent-hashing.md) |
| Bloom Filters | [bloom-filters.md](fundamentals/distributed-data-structures/bloom-filters.md) |
| HyperLogLog & Count-Min Sketch | [hyperloglog-and-count-min-sketch.md](fundamentals/distributed-data-structures/hyperloglog-and-count-min-sketch.md) |
| CRDTs | [crdts.md](fundamentals/distributed-data-structures/crdts.md) |
| Merkle Trees | [merkle-trees.md](fundamentals/distributed-data-structures/merkle-trees.md) |

### 13. Storage Systems
[Question Bank](interview-prep/question-banks/storage-systems.md) · [Flashcards](interview-prep/flashcards/storage-systems.md)

| Sub-topic | File |
|---|---|
| Object Storage | [object-storage.md](fundamentals/storage-systems/object-storage.md) |
| Block & File Storage | [block-and-file-storage.md](fundamentals/storage-systems/block-and-file-storage.md) |
| Distributed File Systems | [distributed-file-systems.md](fundamentals/storage-systems/distributed-file-systems.md) |
| Durability & Erasure Coding | [durability-and-erasure-coding.md](fundamentals/storage-systems/durability-and-erasure-coding.md) |

### 14. Search & Indexing
[Question Bank](interview-prep/question-banks/search-and-indexing.md) · [Flashcards](interview-prep/flashcards/search-and-indexing.md)

| Sub-topic | File |
|---|---|
| Inverted Index | [inverted-index.md](fundamentals/search-and-indexing/inverted-index.md) |
| Search Engine Architecture | [search-engine-architecture.md](fundamentals/search-and-indexing/search-engine-architecture.md) |
| Ranking & Relevance | [ranking-and-relevance.md](fundamentals/search-and-indexing/ranking-and-relevance.md) |
| Autocomplete & Prefix Structures | [autocomplete-and-prefix-structures.md](fundamentals/search-and-indexing/autocomplete-and-prefix-structures.md) |

### 15. Observability
[Question Bank](interview-prep/question-banks/observability.md) · [Flashcards](interview-prep/flashcards/observability.md)

| Sub-topic | File |
|---|---|
| Logging | [logging.md](fundamentals/observability/logging.md) |
| Metrics & Monitoring | [metrics-and-monitoring.md](fundamentals/observability/metrics-and-monitoring.md) |
| Distributed Tracing | [distributed-tracing.md](fundamentals/observability/distributed-tracing.md) |
| Alerting & SLOs | [alerting-and-slos.md](fundamentals/observability/alerting-and-slos.md) |

### 16. Security
[Question Bank](interview-prep/question-banks/security.md) · [Flashcards](interview-prep/flashcards/security.md)

| Sub-topic | File |
|---|---|
| Authentication | [authentication.md](fundamentals/security/authentication.md) |
| Authorization | [authorization.md](fundamentals/security/authorization.md) |
| Encryption & Key Management | [encryption-and-key-management.md](fundamentals/security/encryption-and-key-management.md) |
| API & Application Security | [api-and-application-security.md](fundamentals/security/api-and-application-security.md) |

### 17. Reliability & Resiliency
[Question Bank](interview-prep/question-banks/reliability-and-resiliency.md) · [Flashcards](interview-prep/flashcards/reliability-and-resiliency.md)

| Sub-topic | File |
|---|---|
| Redundancy & Failover | [redundancy-and-failover.md](fundamentals/reliability-and-resiliency/redundancy-and-failover.md) |
| Circuit Breakers, Retries & Timeouts | [circuit-breakers-retries-and-timeouts.md](fundamentals/reliability-and-resiliency/circuit-breakers-retries-and-timeouts.md) |
| Bulkheads & Isolation | [bulkheads-and-isolation.md](fundamentals/reliability-and-resiliency/bulkheads-and-isolation.md) |
| Chaos Engineering | [chaos-engineering.md](fundamentals/reliability-and-resiliency/chaos-engineering.md) |
| Disaster Recovery | [disaster-recovery.md](fundamentals/reliability-and-resiliency/disaster-recovery.md) |

### 18. Batch & Stream Processing
[Question Bank](interview-prep/question-banks/batch-and-stream-processing.md) · [Flashcards](interview-prep/flashcards/batch-and-stream-processing.md)

| Sub-topic | File |
|---|---|
| Batch Processing & MapReduce | [batch-processing-and-mapreduce.md](fundamentals/batch-and-stream-processing/batch-processing-and-mapreduce.md) |
| Stream Processing Fundamentals | [stream-processing-fundamentals.md](fundamentals/batch-and-stream-processing/stream-processing-fundamentals.md) |
| Lambda & Kappa Architectures | [lambda-and-kappa-architectures.md](fundamentals/batch-and-stream-processing/lambda-and-kappa-architectures.md) |
| Exactly-Once Processing | [exactly-once-processing.md](fundamentals/batch-and-stream-processing/exactly-once-processing.md) |

### 19. Content Delivery & Edge
[Question Bank](interview-prep/question-banks/content-delivery-and-edge.md) · [Flashcards](interview-prep/flashcards/content-delivery-and-edge.md)

| Sub-topic | File |
|---|---|
| CDN Architecture | [cdn-architecture.md](fundamentals/content-delivery-and-edge/cdn-architecture.md) |
| Edge Computing | [edge-computing.md](fundamentals/content-delivery-and-edge/edge-computing.md) |
| Geo-Routing & Anycast | [geo-routing-and-anycast.md](fundamentals/content-delivery-and-edge/geo-routing-and-anycast.md) |
| Static & Dynamic Content Acceleration | [static-and-dynamic-content-acceleration.md](fundamentals/content-delivery-and-edge/static-and-dynamic-content-acceleration.md) |

### 20. Capacity Planning & Estimation
[Question Bank](interview-prep/question-banks/capacity-planning-and-estimation.md) · [Flashcards](interview-prep/flashcards/capacity-planning-and-estimation.md)

| Sub-topic | File |
|---|---|
| Back-of-Envelope Math | [back-of-envelope-math.md](fundamentals/capacity-planning-and-estimation/back-of-envelope-math.md) |
| Capacity Planning | [capacity-planning.md](fundamentals/capacity-planning-and-estimation/capacity-planning.md) |
| Benchmarking & Load Testing | [benchmarking-and-load-testing.md](fundamentals/capacity-planning-and-estimation/benchmarking-and-load-testing.md) |
| Cost Modeling | [cost-modeling.md](fundamentals/capacity-planning-and-estimation/cost-modeling.md) |

---

## Case Studies — Tracker

| # | Case Study | Tier | Key Fundamentals | File |
|---|---|---|---|---|
| — | [Problem-Solving Framework](case-studies/00-problem-solving-framework.md) | Reference | All | [00-problem-solving-framework.md](case-studies/00-problem-solving-framework.md) |
| — | [Case Study Template](case-studies/01-case-study-template.md) | Reference | All | [01-case-study-template.md](case-studies/01-case-study-template.md) |
| 1 | [URL Shortener](case-studies/url-shortener.md) | Core | Databases, Caching, Distributed Data Structures | [url-shortener.md](case-studies/url-shortener.md) |
| 2 | [Distributed Rate Limiter](case-studies/distributed-rate-limiter.md) | Core | Rate Limiting, Distributed Systems Theory | [distributed-rate-limiter.md](case-studies/distributed-rate-limiter.md) |
| 3 | [Distributed Cache](case-studies/distributed-cache.md) | Core | Caching, Distributed Data Structures, Reliability | [distributed-cache.md](case-studies/distributed-cache.md) |
| 4 | [Distributed KV Store (Dynamo-style)](case-studies/distributed-key-value-store.md) | Core | Distributed Systems Theory, Consensus, Data Structures | [distributed-key-value-store.md](case-studies/distributed-key-value-store.md) |
| 5 | [News Feed & Social Timeline](case-studies/news-feed-and-social-timeline.md) | Core | Databases, Caching, Messaging | [news-feed-and-social-timeline.md](case-studies/news-feed-and-social-timeline.md) |
| 6 | [Chat & Messaging System](case-studies/chat-and-messaging-system.md) | Core | Networking, Messaging, Distributed Systems | [chat-and-messaging-system.md](case-studies/chat-and-messaging-system.md) |
| 7 | [Video Streaming Platform](case-studies/video-streaming-platform.md) | Core | Content Delivery, Storage, Batch/Stream Processing | [video-streaming-platform.md](case-studies/video-streaming-platform.md) |
| 8 | [Search Autocomplete / Typeahead](case-studies/search-autocomplete.md) | Core | Search & Indexing, Data Structures, Caching | [search-autocomplete.md](case-studies/search-autocomplete.md) |
| 9 | [Web Crawler](case-studies/web-crawler.md) | Core | Data Structures, Storage, Messaging | [web-crawler.md](case-studies/web-crawler.md) |
| 10 | [Notification System](case-studies/notification-system.md) | Core | Messaging, Rate Limiting, Reliability | [notification-system.md](case-studies/notification-system.md) |
| 11 | [Ride-Sharing Service](case-studies/ride-sharing-service.md) | Core | Databases, Distributed Systems, Scalability | [ride-sharing-service.md](case-studies/ride-sharing-service.md) |
| 12 | [Distributed Job Scheduler](case-studies/distributed-job-scheduler.md) | Core | Consensus, Messaging, Distributed Transactions | [distributed-job-scheduler.md](case-studies/distributed-job-scheduler.md) |
| 13 | [Distributed File Storage (S3-style)](case-studies/distributed-file-storage.md) | Core | Storage, Databases, Data Structures | [distributed-file-storage.md](case-studies/distributed-file-storage.md) |
| 14 | [API Gateway Design](case-studies/api-gateway-design.md) | Advanced | Microservices, Rate Limiting, Security | [api-gateway-design.md](case-studies/api-gateway-design.md) |
| 15 | [Payment & Ledger System](case-studies/payment-and-ledger-system.md) | Advanced | Distributed Transactions, Databases, Security | [payment-and-ledger-system.md](case-studies/payment-and-ledger-system.md) |
| 16 | [Collaborative Document Editing](case-studies/collaborative-document-editing.md) | Advanced | Data Structures (CRDTs), Networking, Distributed Systems | [collaborative-document-editing.md](case-studies/collaborative-document-editing.md) |
| 17 | [Real-Time Analytics & Leaderboard](case-studies/real-time-analytics-and-leaderboard.md) | Advanced | Stream Processing, Data Structures | [real-time-analytics-and-leaderboard.md](case-studies/real-time-analytics-and-leaderboard.md) |
| 18 | [Distributed Message Queue (Kafka-style)](case-studies/distributed-message-queue.md) | Advanced | Messaging, Consensus, Storage | [distributed-message-queue.md](case-studies/distributed-message-queue.md) |
| 19 | [Distributed Lock/Coordination Service](case-studies/distributed-lock-and-coordination-service.md) | Advanced | Consensus, Distributed Systems | [distributed-lock-and-coordination-service.md](case-studies/distributed-lock-and-coordination-service.md) |
| 20 | [E-Commerce Inventory & Flash Sale](case-studies/e-commerce-inventory-and-flash-sale.md) | Advanced | Databases, Distributed Transactions, Rate Limiting | [e-commerce-inventory-and-flash-sale.md](case-studies/e-commerce-inventory-and-flash-sale.md) |

---

## Interview-Prep Resources

- **[Problem-Solving Framework](case-studies/00-problem-solving-framework.md)**: A 10-step methodology for tackling any system design problem in 45–60 minutes, with time allocation and common pitfalls.
- **[Case Study Template](case-studies/01-case-study-template.md)**: The exact structure every case study follows. Use this to draft your own practice problems.
- **Question Banks**: [interview-prep/question-banks/](interview-prep/question-banks/) — 20 topic-specific files, each with 15–25 open-ended questions (no answers). Self-test before diving into the fundamentals deep-dives.
- **Flashcards**: [interview-prep/flashcards/](interview-prep/flashcards/) — 20 topic-specific files with collapsed Q/A pairs for spaced-repetition recall of key definitions, formulas, and decision trees.

