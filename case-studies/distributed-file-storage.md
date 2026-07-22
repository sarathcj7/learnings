# Distributed File Storage (S3-Style)

*Design object storage like S3, GCS. Petabytes of data, 11 9's durability, global access, immutable objects.*

## Problem Statement

Build an object storage service (like S3). Store exabytes of data across data centers. Objects are immutable (write-once, read-many). Must achieve 99.9999999% (11 9's) durability.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Storage** | Exabytes | Massive |
| **Requests/sec** | 10M reads/sec | High throughput |
| **Durability** | 11 9's | Extreme reliability |

## Architecture

```
Upload:
  Client PUT /bucket/key
  → API Gateway (auth, routing)
  → Write to primary storage (3 replicas, different racks)
  → Async replicate to other regions
  
Download:
  Client GET /bucket/key
  → Route to nearest replica (geographically)
  → Serve from cache if hot
  
Metadata:
  Bucket, key, size, etag, storage class
  → Stored in fast database (per-region replicas)
```

## Durability Strategy

### Replication

```
Object "photo.jpg" (100 MB):
  Stored on:
    - 3 copies in primary region (different racks)
    - 2 copies in backup region (for DR)
    - Total: 5 copies
  
Durability math:
  If disk failure rate = 0.1%/year
  Probability all 5 copies fail = (0.001)^5 ≈ 10^-15 (11 9's!)
```

### Erasure Coding

```
Instead of 5 replicas, use (12, 8) erasure code:
  Split 8 MB object into 12 chunks
  Can recover from loss of any 4 chunks
  
Efficiency:
  5 replicas: 500% overhead
  (12,8) code: 150% overhead
  Same durability, 3x less storage
  
Trade-off: Recover requires reading from multiple nodes (slower)
```

## Bottlenecks & Scaling

**Bottleneck**: Metadata lookups (10M requests/sec).

**Solution**:
- Distributed metadata service (sharded by bucket)
- Caching hot buckets
- Consistent hashing for shard assignment

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Erasure coding** | Replication | Lower cost (150% vs 500% overhead) |
| **Immutable objects** | Mutable | Simplifies consistency, versioning, replication |
| **Regional replicas** | Single region | Disaster recovery, geographic latency reduction |

---

**Status**: ✅ Complete. Shows durability, immutability, geo-replication.
