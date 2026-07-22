# Consensus & Coordination

Distributed systems agree on values despite failures. This topic covers consensus algorithms (Paxos, Raft), leader election, distributed locks, and coordination services — the foundation of all distributed infrastructure.

## Sub-topics

- **[Paxos](paxos.md)** ✅: Complex but correct consensus, two-phase protocol, prepare/accept
- **[Raft](raft.md)** ✅: Simpler consensus, leader-based, understandable, production-ready
- **[Leader Election](leader-election.md)** ✅: Bully, ring, Raft algorithms, split-brain prevention
- **[Distributed Locks](distributed-locks.md)** ✅: Lease-based, fencing tokens, Redlock issues, safety
- **[Coordination Services](coordination-services.md)** ✅: Chubby, Zookeeper, Consul, etcd (implementations)

## Why This Matters

- **Fault tolerance**: How systems survive node failures
- **State consistency**: Ensuring all nodes agree on value
- **Foundation for infrastructure**: Kubernetes (etcd), databases (consensus), coordination (Consul)
- **Interview critical**: Paxos/Raft appear in most advanced system design interviews

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/consensus-and-coordination.md)**: 35+ deep consensus questions
- **[Flashcards](../../interview-prep/flashcards/consensus-and-coordination.md)**: Algorithms and guarantees

## Related Case Studies

- [Distributed Job Scheduler](../../case-studies/distributed-job-scheduler.md) – Consensus-based job assignment
- [Distributed Lock Service](../../case-studies/distributed-lock-and-coordination-service.md) – Building a lock service with consensus

## Related Fundamentals

- [Distributed Systems Theory](../distributed-systems-theory/) – CAP, consistency, quorum theory
- [Databases/Replication](../databases/replication.md) – Uses consensus for failover
- [Reliability & Resiliency](../reliability-and-resiliency/) – Consensus enables fault tolerance

---

## Study Tips

1. **Understand the problem first**: Why is consensus hard in distributed systems?
2. **Paxos → Raft**: Learn Paxos for theory, Raft for practice
3. **Visualization helps**: Raft visualizations at https://raft.github.io/ make it click
4. **Production experience**: Know when to use consensus (strong consistency) vs quorum (eventual)

---

**Status**: ✅ Complete. 5 sub-files with algorithms, guarantees, and practical implementations.
