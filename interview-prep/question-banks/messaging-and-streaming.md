# Messaging and Streaming — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## Delivery Semantics

1. Explain "at-most-once" delivery. What's the trade-off?
2. Explain "at-least-once" delivery. How do you handle duplicates?
3. Explain "exactly-once" delivery. Is it actually achievable?
4. Your payment system requires exactly-once delivery. Design it.
5. You're building a metrics system that can tolerate lost events. What delivery semantic would you use?
6. What's the difference between delivery guarantee and ordering guarantee?
7. A message broker provides at-least-once delivery. A consumer processes a message, crashes before committing offset. What happens when it restarts?
8. Compare at-least-once (with deduplication) vs exactly-once. Which is simpler to implement?
9. Your messaging system guarantees at-least-once delivery. A duplicate message arrives. How does your consumer detect it?
10. Design a system where exactly-once semantics are critical (financial transactions).

---

## Message Queues & Topics

11. What's the difference between a queue and a topic?
12. When would you use a message queue (RabbitMQ) vs a topic-based system (Kafka)?
13. Explain publish-subscribe. How is it different from point-to-point messaging?
14. Your backend needs to notify multiple services when an order is placed. Use queue or topic? Why?
15. A Kafka topic has 10 partitions. You have 5 consumers. How is traffic distributed?
16. What happens if you add a 6th consumer to that 5-consumer group?
17. Explain consumer groups in Kafka. Why are they useful?
18. Your message queue has 1M messages queued. A consumer crashes and restarts. What's the lag?
19. Design a message queue for a real-time notification system (100k+ messages/sec).
20. What's the difference between RabbitMQ exchanges and Kafka topics?

---

## Kafka Internals & Partitioning

21. Explain partitioning in Kafka. How does it scale throughput?
22. You have a Kafka topic with 3 partitions and replication factor 3. A broker crashes. What happens?
23. A key hashes to partition 0. But partition 0 is down. Does the producer retry or fail?
24. What's the relationship between key, partition, and ordering in Kafka?
25. You need all messages for a user to go to the same partition. How do you ensure this?
26. Explain leader and replica in Kafka. What's ISR (in-sync replicas)?
27. A Kafka partition has 100k messages. Consumer lags 50k. What's your action plan?
28. Design partitioning strategy for a time-series metrics system.
29. What's the trade-off between number of partitions and consumer count?
30. Explain log compaction in Kafka. When would you use it?

---

## Stream Processing

31. What's the difference between stream processing and batch processing?
32. When would you use stream processing (Kafka Streams, Flink) vs batch (Spark)?
33. Design a real-time leaderboard using stream processing.
34. What's windowing in stream processing? Explain tumbling vs sliding windows.
35. Your stream processing job takes 10 seconds to process 1 second of data. Can you still process live data? Why or why not?
36. Explain stateful stream processing. Why is it hard?
37. A stream processing job crashes. How do you ensure no messages are lost or duplicated?
38. Compare Kafka Streams vs Apache Flink vs Spark Streaming.
39. Design a fraud detection system using stream processing.
40. What's backpressure in stream processing? How do you handle it?

---

## Event Sourcing & CQRS

41. Explain event sourcing. What's an event log?
42. Compare event sourcing vs traditional state-based storage.
43. Design an order management system using event sourcing.
44. What's CQRS (Command Query Responsibility Segregation)? Why pair it with event sourcing?
45. Your event log has 1B events. How do you query efficiently?
46. What's snapshotting in event sourcing? Why is it needed?
47. Compare event sourcing for audit logs (read-only) vs transactional systems.
48. Design a system that can replay events to rebuild state at any point in time.
49. An event is published before being persisted. The service crashes. How do you handle it?
50. What's the "dual write" problem in event sourcing? How do you avoid it?

---

## Dead Letter Queues & Error Handling

51. What's a dead letter queue (DLQ)? When should a message go to DLQ?
52. Your message processor fails 1% of messages. They end up in DLQ. How do you handle them?
53. Design a retry strategy: immediate retry vs exponential backoff vs DLQ?
54. A message is sent to DLQ. It's fixed after 1 hour. Can you replay it?
55. How do you monitor DLQ in production? What alerts would you set?

---

## Synthesis & Cross-Cutting

56. Design a real-time analytics pipeline for a social media platform (1B+ events/day).
57. Your microservices communicate via messaging. A service goes down. Design the resilience strategy.
58. Compare synchronous APIs (REST) vs asynchronous messaging for a ride-sharing system.
59. Design a notification system that reaches users across email, SMS, push.
60. You're migrating from HTTP polling to WebSocket + messaging. What are the benefits and risks?

---

## Advanced & Tricky Questions

61. What's the exactly-once semantics gotcha with distributed transactions?
62. Design a messaging system that works across regions with eventual consistency.
63. A Kafka topic has committed offset 1000 but messages 950-1000 are corrupted. How do you recover?
64. Explain offset management in Kafka. When should offsets be committed?
65. Your stream processing job has 100s of partitions. How do you scale state management?
66. What's the CAP theorem implication for event sourcing?
67. Design a messaging protocol for intermittent connectivity (mobile app).
68. Compare Kafka vs Amazon Kinesis vs Google Cloud Pub/Sub for a global event system.
69. What's the overhead of exactly-once semantics vs at-least-once?
70. Design a system that guarantees event ordering across multiple topics.

---

## Self-Scoring Guide

- **0-25 answered well**: Not ready for interviews on messaging. Study fundamentals first.
- **26-45 answered well**: You know the basics. Study weak areas and advanced patterns.
- **46-70 answered well**: You're interview-ready on messaging and streaming.
