# Interview Prep Resources

Dedicated materials for system design interview preparation: question banks for self-testing and flashcards for spaced-repetition learning.

## Question Banks

20 topic-specific question banks with 15–50 open-ended questions each. No answers provided — use to self-test against the fundamentals.

**Status**: Caching complete (53 questions). Others are stubs — fill in as you study each topic.

- [Networking & Protocols](question-banks/networking-and-protocols.md)
- [APIs & Communication](question-banks/apis-and-communication.md)
- [Scalability & Load Balancing](question-banks/scalability-and-load-balancing.md)
- [Caching](question-banks/caching.md) ✅ Complete
- [Databases](question-banks/databases.md)
- [Distributed Systems Theory](question-banks/distributed-systems-theory.md)
- [Consensus & Coordination](question-banks/consensus-and-coordination.md)
- [Distributed Transactions](question-banks/distributed-transactions.md)
- [Messaging & Streaming](question-banks/messaging-and-streaming.md)
- [Microservices Architecture](question-banks/microservices-architecture.md)
- [Rate Limiting & Traffic Management](question-banks/rate-limiting-and-traffic-management.md)
- [Distributed Data Structures](question-banks/distributed-data-structures.md)
- [Storage Systems](question-banks/storage-systems.md)
- [Search & Indexing](question-banks/search-and-indexing.md)
- [Observability](question-banks/observability.md)
- [Security](question-banks/security.md)
- [Reliability & Resiliency](question-banks/reliability-and-resiliency.md)
- [Batch & Stream Processing](question-banks/batch-and-stream-processing.md)
- [Content Delivery & Edge](question-banks/content-delivery-and-edge.md)
- [Capacity Planning & Estimation](question-banks/capacity-planning-and-estimation.md)

## Flashcards

20 topic-specific flashcard sets with 15–35 Q/A pairs for spaced-repetition learning. Use a spaced-repetition app (Anki, Quizlet) or review weekly.

**Status**: Caching complete (35 cards). Others are stubs.

Same links as question banks, but in the `flashcards/` folder.

## How to Use

1. **Study loop**:
   - Read fundamentals sub-file → Answer question bank → Review flashcards → Take a case study → Mark topic as "Practiced"

2. **Interview prep**:
   - Read [problem-solving framework](../case-studies/00-problem-solving-framework.md)
   - Study high-value topics (databases, distributed systems, caching, scaling)
   - Practice case studies under time pressure (45–60 min each)
   - Use question banks to identify weak areas, go back to fundamentals

3. **Spaced repetition**:
   - Review flashcards daily for a week (new material)
   - Then 2–3x/week (practiced material)
   - Then weekly (mastered material)

## Interview Framework

Before tackling case studies:
- [Problem-Solving Framework](../case-studies/00-problem-solving-framework.md): 10-step methodology (5–60 min interview)
- [Case Study Template](../case-studies/01-case-study-template.md): Exact structure to follow, with detailed URL shortener example

## Case Study Gallery

Apply the framework to real systems:

**Core** (must know for interviews):
- [URL Shortener](../case-studies/url-shortener.md) ✅ Complete
- [Distributed Rate Limiter](../case-studies/distributed-rate-limiter.md) ✅ Complete
- [Distributed Cache](../case-studies/distributed-cache.md)
- [News Feed & Social Timeline](../case-studies/news-feed-and-social-timeline.md)
- [Chat & Messaging](../case-studies/chat-and-messaging-system.md)
- [Video Streaming](../case-studies/video-streaming-platform.md)
- [Search Autocomplete](../case-studies/search-autocomplete.md)
- [Web Crawler](../case-studies/web-crawler.md)
- [Notification System](../case-studies/notification-system.md)
- [Ride-Sharing (Uber-style)](../case-studies/ride-sharing-service.md)
- [Distributed Job Scheduler](../case-studies/distributed-job-scheduler.md)
- [Distributed File Storage (S3-style)](../case-studies/distributed-file-storage.md)
- [Distributed KV Store (Dynamo-style)](../case-studies/distributed-key-value-store.md)

**Advanced** (deepen mastery, cover infrastructure):
- [API Gateway Design](../case-studies/api-gateway-design.md)
- [Payment & Ledger System](../case-studies/payment-and-ledger-system.md)
- [Collaborative Document Editing](../case-studies/collaborative-document-editing.md)
- [Real-Time Analytics & Leaderboard](../case-studies/real-time-analytics-and-leaderboard.md)
- [Distributed Message Queue (Kafka-style)](../case-studies/distributed-message-queue.md)
- [Distributed Lock/Coordination Service](../case-studies/distributed-lock-and-coordination-service.md)
- [E-Commerce Inventory & Flash Sale](../case-studies/e-commerce-inventory-and-flash-sale.md)

---

## Recommended Study Order

### Interview Prep (2–4 weeks, 1–2 hours/day)

1. **Foundations** (3 days):
   - Problem-solving framework
   - Case study template
   - Caching fundamentals + case study

2. **Core topics** (1–2 weeks):
   - Databases, Distributed Systems, Consensus, Scalability
   - Practice 1 case study per day from "Core" list

3. **Breadth** (1 week):
   - Remaining topics: APIs, Networking, Storage, Search, Messaging
   - 2–3 case studies from "Advanced" list

4. **Polish** (3 days):
   - Review weak areas (question banks)
   - Mock interviews with peers
   - Time-pressure case studies

### Mastery Path (ongoing, 1–2 hours/week)

1. Study one fundamentals topic per week in depth
2. Complete all case studies
3. Review flashcards 2–3x/week (spaced repetition)
4. Every month, re-solve a case study and compare to writeup

---

## Tips for Success

- **Active recall**: Don't just read. Answer question banks without peeking at fundamentals.
- **Teach others**: Explaining a system design to a peer forces clarity.
- **Whiteboard practice**: System design is a communication exercise. Practice explaining on a whiteboard.
- **Time yourself**: Interviews are 45–60 min. Practice solving case studies in that window.
- **Iterate**: Your first design is rarely optimal. Spend time on trade-offs and bottleneck analysis.
- **Stay humble**: There's always more to learn. "I hadn't considered that" is a perfectly good answer in an interview.

---

**Last updated**: 2026-07-23  
**Complete**: Caching topic + 2 case studies (URL Shortener, Rate Limiter)  
**Next**: Add other case studies, fill in fundamentals content as you study
