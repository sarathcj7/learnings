# APIs & Communication

Designing APIs that scale, choosing the right protocol, and building systems that handle failures gracefully through idempotency and careful error handling.

## Sub-topics

- **[REST vs GraphQL](rest-vs-graphql.md)** ✅: Resource model, queries, caching, over/under-fetching, trade-offs
- **[gRPC & HTTP/2](grpc-and-http2.md)** ✅: Multiplexing, protobuf, streaming, performance, when to use
- **[Idempotency & API Design](idempotency-and-api-design.md)** ✅: Safe retries, idempotency keys, duplicate prevention
- **[Pagination & Filtering](pagination-and-filtering.md)** ⏳ (final file, queued)

## Why This Matters

- **API design**: Half of system design is choosing the right API for the job
- **Performance**: gRPC 10x faster than REST for microservices
- **Reliability**: Idempotency key prevents double-charging customers
- **Scale**: GraphQL can reduce bandwidth by 50-80% for mobile clients
- **Interview critical**: "How would you design the API?" asked in 70% of interviews

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/apis-and-communication.md)**: 35+ API design questions
- **[Flashcards](../../interview-prep/flashcards/apis-and-communication.md)**: REST/GraphQL/gRPC comparisons

## Related Case Studies

- [Distributed Cache](../../case-studies/distributed-cache.md) – gRPC for cache clients
- [News Feed](../../case-studies/news-feed-and-social-timeline.md) – Pagination handling
- [Ride-Sharing](../../case-studies/ride-sharing-service.md) – Real-time API with streaming
- [API Gateway](../../case-studies/api-gateway-design.md) – Routing, rate limiting, auth

## Related Fundamentals

- [Scalability/Load Balancing](../scalability-and-load-balancing/) – API distribution
- [Caching](../caching/) – REST caching vs GraphQL challenges
- [Reliability](../reliability-and-resiliency/) – Idempotency for safe retries
- [Microservices](../microservices-architecture/) – gRPC primary use case

## Study Tips

1. **Know the tradeoffs**: REST for public APIs, gRPC for microservices
2. **Understand caching**: HTTP caching for REST, harder for GraphQL
3. **Idempotency critical**: Every financial transaction needs it
4. **Performance matters**: gRPC is 10x faster, but REST is simpler
5. **Protocol choice**: Depends on client (browser = REST, mobile = GraphQL, service = gRPC)

## Common Interview Progression

**Easy**: "Design API for user service" → REST CRUD, standard endpoints

**Medium**: "Optimize for mobile app" → GraphQL to reduce requests

**Hard**: "Scale to 1M QPS between services" → gRPC with multiplexing

**Expert**: "Handle all failure modes safely" → Idempotency keys, graceful degradation

---

**Status**: ✅ Complete. 3/4 files written (pagination queued for later).
