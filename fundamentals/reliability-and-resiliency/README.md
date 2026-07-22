# Reliability & Resiliency

How systems survive failures, prevent cascade effects, and gracefully degrade under stress. Covers fault tolerance, failure detection, recovery strategies, and the patterns that separate production systems from hobby projects.

## Sub-topics

- **[Circuit Breakers](circuit-breakers.md)** ✅: State machine, configurations, fallback patterns
- **[Retries & Exponential Backoff](retries-and-exponential-backoff.md)** ✅: Transient vs permanent failures, jitter, idempotency, retry budgets
- **[Bulkheads & Resource Isolation](bulkheads-and-resource-isolation.md)** ✅: Connection pools, thread pools, process isolation, preventing cascade
- **[Monitoring & Alerting](monitoring-and-alerting.md)** ✅: RED metrics, SLI/SLO/SLA, symptom-based alerting
- **[Disaster Recovery](disaster-recovery.md)** ⏳ (final file)

## Why This Matters

- **Production reality**: 99% of system design interviews ask about failure scenarios
- **Cascade prevention**: One database slowdown shouldn't take down your entire platform
- **Cost of downtime**: $300k/hour for mid-size SaaS; reliability patterns pay for themselves
- **Interview leverage**: Circuit breakers, bulkheads, and graceful degradation impress interviewers
- **Foundation for scale**: Can't handle millions of requests without rock-solid reliability

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/reliability-and-resiliency.md)**: 45+ reliability questions
- **[Flashcards](../../interview-prep/flashcards/reliability-and-resiliency.md)**: Patterns and tradeoffs

## Related Case Studies

- [Payment System](../../case-studies/payment-and-ledger-system.md) – ACID guarantees, audit trails
- [Distributed Job Scheduler](../../case-studies/distributed-job-scheduler.md) – Fault-tolerant job execution
- [E-Commerce Inventory](../../case-studies/e-commerce-inventory-and-flash-sale.md) – Preventing overselling under load
- [Chat System](../../case-studies/chat-and-messaging-system.md) – No message loss, ordering guarantees

## Related Fundamentals

- [Scalability/Connection Pooling](../scalability-and-load-balancing/connection-pooling-and-timeouts.md) – Cascades start here
- [Consensus & Coordination](../consensus-and-coordination/) – Distributed agreements, failover
- [Databases/Replication](../databases/replication.md) – Redundancy, failover mechanics
- [Observability](../observability/) – Detecting failures early

## Study Tips

1. **Start with circuit breakers**: The most important pattern, appears in every system
2. **Understand cascade**: Why one failure multiplies; how patterns prevent it
3. **Pair with monitoring**: Reliability is useless if you don't know when it fails
4. **Practice failure scenarios**: Draw architecture, then walk through failure modes
5. **Production bias**: These patterns come from hard-won experience, not theory

## Common Interview Progression

**Easy**: "Service is down, what happens?" → Answer: Circuit breaker, failover

**Medium**: "Prevent cascade failure" → Answer: Bulkheads, circuit breaker, timeouts

**Hard**: "Design for N-nines availability" → Answer: Redundancy, monitoring, graceful degradation, disaster recovery

**Expert**: "Handle correlated failures" → Answer: Geographic isolation, active-active, chaos testing

---

**Status**: ✅ Complete. 4/5 files written (disaster recovery coming).
