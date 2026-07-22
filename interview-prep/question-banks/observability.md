# Observability — Mock Interview Question Bank

## How to Use This

Attempt each question before reading the fundamentals. Write your answer aloud or on paper. This forces active recall. Then check your answer against the fundamentals sub-files. Mark questions you struggle with and revisit them.

---

## Metrics & RED

1. Explain the RED method. What metrics does it track?
2. What's a rate, error rate, and duration? How would you measure them?
3. Your API has 1000 QPS with 5% error rate. Is this healthy? What's the context?
4. Explain the difference between metrics and events.
5. What's the difference between gauge and counter metrics?
6. Design a metrics system for a microservice (response time, QPS, errors).
7. Your service has multiple endpoints. How do you track RED metrics per endpoint?
8. What's a histogram metric? When would you use it vs a gauge?
9. Your P99 latency is 100ms but P50 is 5ms. What does this tell you?
10. Design alerting rules based on RED metrics. When would you alert vs observe?

---

## SLI, SLO, & Error Budgets

11. Explain SLI (Service Level Indicator). Give examples.
12. What's the difference between SLI and SLO (Service Level Objective)?
13. Your SLO is "99.9% availability". What does that mean in downtime per month?
14. Design an SLO for a real-time messaging service.
15. Explain error budgets. How would you use them for deployment decisions?
16. You have a 99.9% SLO for availability. You're now at 99.5% after a partial outage. What do you do?
17. What's the difference between availability and latency SLOs?
18. Design SLIs for an internal service (less critical than public API).
19. How would you measure SLO compliance retrospectively?
20. Your SLO is "P99 latency < 100ms". You hit 105ms. Did you breach SLO?

---

## Logging

21. Explain the difference between info, warning, error, and debug log levels.
22. What's structured logging? Why is it better than free-text logs?
23. Your system produces 1 TB of logs per day. How do you store and query them?
24. Design a logging strategy for a microservices architecture.
25. Your logs contain user passwords (accidental). What's your response?
26. What's log sampling? When would you use it?
27. Explain log correlation IDs (request IDs). Why are they useful?
28. Your service logs "error: database connection failed". Is this helpful? How would you improve it?
29. How long should you retain logs? What's the trade-off?
30. Design a system that logs user actions for compliance while respecting privacy.

---

## Distributed Tracing

31. What's distributed tracing? How is it different from logging?
32. Explain spans and traces in distributed tracing.
33. Your request spans 5 services. How do you trace it end-to-end?
34. What's trace sampling? When would you sample at 1% vs 100%?
35. Your P99 latency is high. How would you use distributed tracing to debug it?
36. Explain the relationship between trace ID, span ID, and parent span ID.
37. You want to trace cross-region requests. How does this change distributed tracing?
38. What's the overhead of 100% tracing vs sampling? When would you choose each?
39. Design a tracing strategy for a payment processing system (visibility + performance).
40. How would you correlate logs and traces for debugging?

---

## Profiling & Performance

41. What's CPU profiling? How would you use it?
42. Explain memory profiling. What problems can you detect?
43. Your service has high CPU usage but no obvious hot code. How do you debug?
44. What's the difference between CPU profiling and instrumentation profiling?
45. Design a profiling strategy for production (low overhead, high information).
46. Your service uses 10 GB memory. How would you find the memory leak?
47. What's continuous profiling? Why is it useful?
48. Explain flame graphs. How would you read one?
49. Your database queries are slow. Would you use CPU profiling or query profiling?
50. Design profiling for a Java application (GC pauses, CPU, memory).

---

## Alerting & Monitoring

51. Design an alerting strategy: which metrics would you alert on?
52. Your alert fires every minute. What's the problem? How do you fix it?
53. Explain alert fatigue. How would you reduce it?
54. Your SLO is 99.9% but your alerting threshold is 95%. Is this aligned?
55. Design an on-call escalation policy based on alert severity.
56. Your service has 1000+ metrics. How do you avoid alert overload?
57. What's the difference between monitoring and alerting?
58. Design alerting for a database outage that should trigger immediately.
59. Your alert fires before SLO breach (early warning). How do you calibrate it?
60. Explain anomaly detection in alerting. When would you use it vs static thresholds?

---

## Synthesis & Cross-Cutting

61. Design a complete observability stack (metrics, logs, traces) for a microservices platform.
62. Your P99 latency increased 2x overnight but QPS is stable. Debug it using observability.
63. Design observability for a batch processing system (correctness + performance).
64. Your payment API has strict SLA (99.99% uptime). Design monitoring and alerting.
65. A feature rollout is causing issues. How would you use observability to detect and rollback?

---

## Advanced & Tricky Questions

66. What's the observability blind spot with distributed systems? How do you address it?
67. Design a zero-observability baseline (no alerts initially). How would you add them safely?
68. Explain cardinality explosion in metrics (e.g., metrics with user_id as label).
69. Your tracing shows a service is 100ms slower, but profiling shows no hot code. What's wrong?
70. Design a cost-efficient observability strategy for a high-volume system.
71. How would you detect cascading failures using observability?
72. What's the relationship between observability and testing?
73. Design observability for a machine learning pipeline (data + model + inference).
74. Explain probabilistic data structures (HyperLogLog, t-digest) in observability.
75. Design a post-mortem process using observability data.

---

## Self-Scoring Guide

- **0-20 answered well**: Not ready for interviews on observability. Study fundamentals first.
- **21-45 answered well**: You know the basics. Study weak areas and tracing.
- **46-75 answered well**: You're interview-ready on observability.
