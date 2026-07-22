# System Design Problem-Solving Framework

A repeatable 10-step methodology for tackling any system design interview problem in 45–60 minutes. This framework is used consistently across all case studies in this repo.

## Overview

The goal is to move from vague product idea to a concrete, reasoned architectural design that demonstrates:
- Clear thinking and systematic problem decomposition
- Knowledge of relevant distributed systems concepts
- Practical trade-off reasoning
- Awareness of real-world constraints and failure modes

This framework is designed for a live 1-hour interview. Adjust pacing based on interviewer feedback and remaining time.

---

## Step 1: Clarify Requirements (5 min)

**Goal**: Make assumptions explicit and align with the interviewer on what you're actually building.

**What to do**:
- Restate the problem in your own words to confirm understanding.
- Ask about **functional requirements**: What does the system do? What are the key features?
- Ask about **non-functional requirements**: Scale (DAU/MAU, QPS, data size), latency targets, consistency/availability trade-offs, geographic scope.
- Ask about **edge cases or constraints**: Regulatory (PII, compliance), technical (existing infra), business (cost/latency SLA).

**Common pitfalls**:
- Jumping into a solution before nailing scope — leads to designing the wrong system.
- Asking open-ended "What matters most?" — be specific: "Do we prioritize read or write throughput?" "Is strong consistency critical or eventual OK?"
- Not mentioning your assumptions aloud — the interviewer needs to hear your reasoning.

**Example questions to ask**:
- "Are we designing for peak throughput or average?"
- "Do we need geo-distributed redundancy or is a single region OK?"
- "Is this a read-heavy or write-heavy system?"
- "What's the customer-facing SLA on latency and uptime?"

---

## Step 2: Estimate Scale (5 min)

**Goal**: Ground your design in concrete numbers so you know which bottlenecks to expect.

**What to do**:
- Estimate **DAU/MAU** (daily/monthly active users) from the requirements.
- Calculate **QPS** (queries per second):
  - Active concurrency ≈ DAU × (avg session duration) / (seconds in day)
  - Peak QPS ≈ 5–10× average QPS
- Estimate **storage**:
  - Data per user/request (size of one object)
  - Total storage = (users × data per user) or (daily events × data per event)
  - Growth rate for 1-year/5-year projections
- Estimate **bandwidth** (throughput in MB/s or Gbps):
  - Bytes per request × QPS

**Common pitfalls**:
- Confusing concurrent users with QPS.
- Forgetting peak traffic: design for peak, not average.
- Underestimating data growth: cache and DB sizing both depend on this.

**Example order of magnitude reference**:
- 1 million DAU → ~10 QPS average, ~50 QPS peak
- 100 million DAU → ~1000 QPS average, ~5000 QPS peak
- 1 TB data → ~1 large instance or distributed store
- 1 GB/s throughput → multiple regional CDN nodes + origin infra

---

## Step 3: Define the API (5 min)

**Goal**: Clarify the contract between clients and the system.

**What to do**:
- Write out 3–5 key endpoints (REST) or RPC methods (gRPC). Focus on read and write hot paths.
- For each, specify:
  - Request/response shape (JSON schema or similar)
  - Success and error cases
  - Any idempotency or ordering guarantees
- Mention pagination, filtering, sorting if relevant.

**Common pitfalls**:
- Over-specifying 20 endpoints — focus on the critical path.
- Forgetting error codes — 400 Bad Request is not the same as 503 Service Unavailable, and the difference matters for retry logic.
- API design as an afterthought — a good API clarifies what the system must support.

**Example template**:
```
POST /users/{userId}/create-post
Request: { content: string, media: [url], timestamp: int64 }
Response: { postId, createdAt, success: bool }
Errors: 400 (invalid content), 401 (unauthorized), 503 (service down)
Idempotent: yes (postId is immutable and client-supplied)
```

---

## Step 4: Design the Data Model (5 min)

**Goal**: Specify the entities, schema, and storage choices that back the API.

**What to do**:
- Draw the entity-relationship diagram: Users, Posts, Comments, etc. and their relationships.
- For each entity, sketch the schema:
  - Denormalize or normalize? Join cost vs redundancy trade-off.
  - Write-heavy vs read-heavy? (Affects indexing strategy, replication choice)
- Justify storage choice:
  - SQL (relational): For structured, highly relational data with strong consistency needs.
  - NoSQL (key-value, document): For semi-structured, high-throughput writes, horizontal scaling.
  - Time-series DB: For metrics/logs with time-based retention.

**Common pitfalls**:
- Forgetting to mention indexes — "What queries do we need to run fast?"
- Picking NoSQL just because it's "scalable" — relational DBs scale fine with sharding.
- Not thinking about write contention — if 1 row gets 10k updates/sec, that's a hot spot.

**Example sketch**:
```
Users: { userId (PK), name, email, createdAt, followers (denormalized count) }
Posts: { postId (PK), userId (FK), content, likes (denormalized), createdAt (indexed) }
Followers: { userId (PK), followerId (PK), createdAt (sparse index on userId+createdAt for timeline queries) }
```

---

## Step 5: High-Level Architecture (8 min)

**Goal**: Draw the bird's-eye system design and label each component.

**What to do**:
- Draw a Mermaid diagram (or sketch) with:
  - **Client layer**: Web, mobile, internal APIs
  - **Edge/API Gateway**: Load balancing, authentication, rate limiting
  - **Service layer**: 3–5 key microservices (if applicable) or monolith
  - **Cache layer**: Redis, Memcached
  - **Database layer**: Primary replica, read replicas, sharding strategy
  - **Async layer**: Message queues, event streaming
  - **Storage**: Blob storage for media, search indexes
  - **Observability**: Logging, metrics, tracing
- Label data flow: Synchronous paths (solid arrows), asynchronous (dashed).
- Identify single points of failure and mark them.

**Common pitfalls**:
- Drawing boxes without labeling what lives inside (is that "Service" a monolith or multiple services?).
- Missing the cache layer — 80% of improvements come from caching.
- Not showing how data consistency is maintained — is cache invalidation on write? TTL-based?

**Example component responsibilities**:
- **API Gateway**: Routes requests, enforces rate limits, terminates TLS, routes to services
- **User Service**: Authentication, profile management
- **Feed Service**: Generates news feeds, applies ranking, stores dendencies
- **Cache**: Caches hot feeds (1h TTL), user profiles (24h), trending topics (5min)

---

## Step 6: Deep Dive on Critical Components (15 min)

**Goal**: Explain the mechanics of 2–3 bottleneck components in detail.

**What to do**:
- Identify the hardest parts of your design. Usually:
  - Hot data paths (read/write amplification)
  - State management (caching, session storage)
  - Cross-service consistency
  - Real-time coordination

- For each, explain:
  - **Mechanics**: Sequence diagrams showing the data flow for a typical request.
  - **Why this choice**: What problem does it solve? What trade-off did you make?
  - **Failure scenario**: What happens if this component fails or is overwhelmed?

**Common pitfalls**:
- Giving a surface-level explanation of a deep problem (e.g., "we use Redis" without explaining cache-aside pattern, thundering herd, invalidation strategy).
- Skipping the sequence diagram — it forces you to think through ordering and consistency.
- Not mentioning Byzantine fault tolerance or partition tolerance trade-offs.

**Example deep dive**:
> "The feed generation is tricky because we have 100M users, each with ~500 followers. For every post, we'd need to fan-out to 500 followers, which is 100M writes/sec at peak. So we do write-behind caching: posts go to a queue, background workers compute personalized feeds asynchronously, and cache the top 100 posts per user with a 1-hour TTL. On cache miss, we merge real-time with cached results."

---

## Step 7: Identify Bottlenecks & Scale the Design (10 min)

**Goal**: Anticipate failure points at 10x, 100x, and 1000x load and evolve the design.

**What to do**:
- Take your current scale estimate from Step 2.
- Ask: "What breaks first if we 10x this?"
  - Is it the database (too many queries)?
  - Network bandwidth?
  - A single service becoming a hot spot?
  - The cache not having enough memory?
- For each bottleneck, propose a solution:
  - Add more replicas (reads) or shards (writes).
  - Increase cache size or add a cache layer earlier.
  - Partition by time, geography, or user ID.
  - Use async processing to move work out of the critical path.
- Repeat for 100x and 1000x.

**Common pitfalls**:
- Designing for infinite scale from day one (premature optimization).
- Not prioritizing: "We'd shard everything" is weak. Prioritize: "First we'd shard the user DB by user ID, then partition messaging queue by topic partition."
- Forgetting operational costs: "Add 100 cache instances" sounds easy until someone asks about failover and monitoring.

**Example scaling narrative**:
> "At 10x (50k QPS), the feed service becomes CPU-bound. We'd add service replicas behind a load balancer. At 100x (500k QPS), the primary database becomes write-bound on the posts table. We'd shard by post ID (range or hash). At 1000x, we'd need multi-region: cache layer in each region, replicate user/follower data read-only to each region's DB."

---

## Step 8: Discuss Trade-offs & Alternatives (5 min)

**Goal**: Show that you've considered and consciously rejected alternatives.

**What to do**:
- For each major decision, mention the alternative you rejected and why:
  - "We chose Cassandra (eventual consistency) over PostgreSQL (strong consistency) because write throughput was more important than immediate consistency."
  - "We cache with a 1-hour TTL instead of write-through invalidation because TTL is simpler to reason about and eventual consistency is tolerable."
  - "We use a message queue for notifications instead of calling the notification service synchronously to avoid blocking the main request."

**Common pitfalls**:
- Presenting your design as the only option — "This is just how you do it."
- Not weighing pros and cons — "DynamoDB is better because it's managed." (Better at what? Cost? Consistency?)

**Example trade-off discussion**:
> "We chose eventual consistency for the feed cache rather than strong consistency. Pro: simpler, faster propagation. Con: users might see stale likes counts for up to an hour. Trade-off: for a social feed, an hour of staleness is acceptable; for a bank account balance, it's not."

---

## Step 9: Handle Failure Scenarios (5 min)

**Goal**: Show defensive thinking. Assume parts of your system will fail.

**What to do**:
- Walk through failure scenarios and mitigation:
  - **Service failure**: "If the feed service crashes, we have 3 replicas and a load balancer. Traffic auto-fails over."
  - **Database failure**: "We have a read replica in the same region and another in a different region. A primary DB failure triggers an automatic failover to the read replica in ~5 seconds."
  - **Cache failure**: "Cache misses go to the primary DB. The database is sized to handle 10% of requests without cache, so we degrade gracefully."
  - **Network partition**: "If the local cache can't reach the primary DB, we serve stale data from cache and log the error. We retry connections exponentially."

**Common pitfalls**:
- Assuming everything works ("Our system is so reliable that failures won't happen.").
- Treating all failures the same — a node failure is different from a region-wide outage.
- Not mentioning your observability plan — "How will you know the failover actually worked?"

---

## Step 10: Anticipate Follow-ups & Twist Questions (2 min)

**Goal**: If time permits, pose likely follow-up questions the interviewer might ask. Show you've already thought about them.

**What to do**:
- Think about variations or edge cases:
  - "What if we need to support multi-region write-write consistency?" (Add eventual consistency or weak quorum system)
  - "What if the rate limiter itself becomes a bottleneck?" (Distribute it, use approximate counters)
  - "What if a user deletes their post but it's already in a follower's cache?" (Delayed invalidation, cache versioning)
  - "What if we need analytics on this system?" (Add event streaming, data warehouse)
- Briefly outline how you'd tackle each — not a full re-solve, just a direction.

**Common pitfalls**:
- Not having an answer ready for common twists (they will come).
- Panicking when the interviewer asks something you didn't anticipate — "I hadn't thought of that" is honest, but "Let me think through that..." and working through it calmly is better.

---

## Time Allocation Guide

For a 45–60 minute interview:
- Step 1 (Clarify): 5 min
- Step 2 (Scale): 5 min
- Step 3 (API): 5 min
- Step 4 (Data Model): 5 min
- Step 5 (High-Level Architecture): 8 min
- Step 6 (Deep Dive): 15 min ← Spend most time here. This is where expertise shows.
- Step 7 (Bottlenecks): 10 min
- Step 8 (Trade-offs): 5 min
- Step 9 (Failures): 5 min
- Step 10 (Follow-ups): 2 min (only if time permits)
- **Flex time / Interviewer questions**: 5–15 min

Adjust based on interviewer signals. If they spend time drilling on one area (e.g., "Tell me more about the cache invalidation"), depth there > breadth elsewhere.

---

## Common Pitfalls Across All Steps

1. **Talking too much without breathing**: Pause, let the interviewer ask questions. They want a dialogue, not a monologue.
2. **Using acronyms without explaining**: "We use CQRS" means nothing if not explained. "We separate read and write paths via CQRS, so we can optimize each independently."
3. **Premature optimization**: Don't solve 10-year scaling problems. Design for now (steps 1–6), then address scale (step 7).
4. **Ignoring operational costs**: It's easy to design something that works. It's harder to design something that's operationally sane (monitoring, alerting, on-call burden).
5. **Not defending your choices**: If the interviewer challenges you, don't immediately fold. Defend your reasoning first, then consider their point.
6. **Being dogmatic**: "Microservices are always better" is wrong. Monoliths, microservices, and monolithic services all have a place.

---

## Interviewer Signals to Watch For

- **Leaning in, asking deeper questions**: You're on the right track, go deeper.
- **Glazed look, short "okay"s**: You're going too deep, or off on a tangent. Pivot to high-level.
- **Challenging an assumption**: They want to see how you defend it. Don't just fold.
- **"How would you measure that?"**: They want you to tie architecture to observability.
- **"What's the cost?"**: They're thinking operationally. Connect your design choices to infra cost and engineering time.

---

## Post-Interview Self-Check

After the interview (or a practice round), review:
- [ ] Did I ask clarifying questions or assume scope?
- [ ] Did I ground my design in numbers (QPS, storage, latency)?
- [ ] Did I justify every major component and design choice?
- [ ] Did I think about failure modes and mitigations?
- [ ] Did I balance breadth and depth (some deep dives, but not getting lost in one area)?
- [ ] Did I show knowledge of relevant distributed systems concepts?
- [ ] Did I defend my trade-offs?
- [ ] Did I listen to interviewer feedback and adapt?

---

## Next Steps

Apply this framework to each case study in this repo. Time yourself. Get feedback. Refine your explanations. The goal is to internalize this structure so it runs on autopilot, freeing mental energy for the hard parts: choosing good technologies and explaining trade-offs clearly.
