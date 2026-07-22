# Coordination Services

## TL;DR

- **Chubby** (Google): Distributed lock service, strong consistency
- **Zookeeper** (Apache): Coordination, configuration, discovery
- **Consul** (HashiCorp): Service discovery + distributed locks + KV store
- **etcd** (CoreOS): Key-value store for configuration, Kubernetes uses it
- **All use Paxos/Raft**: Consensus ensures correctness

## Chubby (Google)

Distributed lock service used internally.

```
Primary purpose: Distributed locks and agreement
Example use: Master election for distributed systems

API:
  CREATE /lock
  LOCK /lock (blocking until acquired)
  UNLOCK /lock
  
Safety: Chubby is strongly consistent (uses Paxos)
```

## Zookeeper (Apache)

Coordination for Hadoop ecosystem.

```
Model: File system-like hierarchy
  /hadoop/job-1/status
  /hadoop/job-1/workers

Features:
  - Watches: Notify client when path changes
  - Sequences: Atomic counter increments
  - Ephemeral: Deleted when client disconnects
  
Used by: Kafka (broker coordination), HBase (master election)
```

## Consul (HashiCorp)

Service mesh + coordination.

```
Features:
  - Service registration: Services register themselves
  - Health checks: Consul checks if services alive
  - DNS: Query services via DNS
  - KV store: Distributed configuration
  - Distributed locks: Built-in locking
  
Used for: Service discovery in microservices architectures
```

## etcd (CoreOS)

Key-value store with strong consistency.

```
Primary use: Kubernetes cluster state
  etcd stores: Pod definitions, Node state, ConfigMaps, Secrets
  
Guarantees: Strongly consistent (uses Raft)

API:
  PUT key=value
  GET key
  WATCH key (notify on changes)
  
Used by: Kubernetes (official), Docker Swarm, Flannel
```

## When to Use

| Tool | Lock | Discovery | Config | Consensus |
|---|---|---|---|---|
| **Chubby** | ✅ | No | No | Paxos |
| **Zookeeper** | ✅ | Basic | ✅ | Paxos |
| **Consul** | ✅ | ✅✅ | ✅ | Raft |
| **etcd** | ✅ | No | ✅ | Raft |

---

## References

- Chubby whitepaper (Google)
- Zookeeper guide
- Consul documentation
- etcd documentation

---

## Related Fundamentals

- [Consensus & Coordination](../consensus-and-coordination/) – These implement consensus
- [Raft](raft.md) – Algorithm used by Consul, etcd
- [Distributed Locks](distributed-locks.md) – What these services provide
