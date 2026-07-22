# Block Storage

Persistent block devices (EBS, Persistent Volumes, local SSD). Structured storage for databases and applications requiring low-latency, high-throughput access.

---

## TL;DR

- **Block storage**: Raw disk abstraction, partitions and filesystems on top
- **Examples**: AWS EBS, Google Persistent Disk, Kubernetes PVs
- **Performance**: 10-100k IOPS, latency 1-10ms
- **Use**: Databases, file systems, VM storage
- **Durability**: Replication within/across zones (99.99% uptime)
- **Trade-off**: More expensive than object storage, lower latency

---

## EBS (Elastic Block Store) Basics

### EBS Volume Types

```
gp3 (General Purpose - default):
  Throughput: 125-250 MB/s
  IOPS: 3,000-16,000
  Latency: 1ms average
  Cost: $0.08/GB/month
  Best for: Most workloads (databases, web servers)
  
io2 (High Performance):
  Throughput: Up to 1,000 MB/s
  IOPS: Up to 64,000
  Latency: <1ms
  Cost: $0.25/GB/month (5x more expensive)
  Best for: High-throughput databases (Oracle, SQL Server)
  
st1 (Throughput Optimized):
  Throughput: 125-250 MB/s
  IOPS: Up to 500
  Cost: $0.045/GB/month
  Best for: Sequential reads (data warehouses, Hadoop)
  
sc1 (Cold):
  Throughput: 60-125 MB/s
  Cost: $0.015/GB/month (cheapest)
  Best for: Infrequent access, backups
```

---

### Volume Lifecycle

```
1. Create volume
   - Size: 100 GB
   - Type: gp3
   - Zone: us-east-1a
   - Cost: $8/month
   
2. Attach to EC2 instance
   /dev/sdf → /mnt/data (mounted)
   
3. Filesystem created
   ext4, xfs, ntfs
   Partition table: MBR or GPT
   
4. Application uses
   Database writes to /mnt/data/db.sqlite
   
5. Snapshot (backup)
   Creates point-in-time copy
   Stored in S3 (cheaper)
   
6. Detach
   Can attach to different instance
   (Data persists)
   
7. Delete
   Volume and data destroyed
   Snapshots remain (can restore)
```

---

## Storage Performance: IOPS and Throughput

### IOPS vs Throughput

```
IOPS (Input/Output Per Second):
  How many I/O operations per second
  Measured in I/O operations (not bytes)
  
  gp3: 16,000 IOPS
  io2: 64,000 IOPS
  
Example workload:
  Database (random read/writes)
  4KB per I/O
  At 10,000 IOPS: 40 MB/s throughput
  Latency-sensitive (financial system)
  
Throughput (MB/s):
  Raw bytes per second
  Measured in Megabytes/second
  
Example workload:
  Video streaming
  Needs: 5 MB/s per concurrent user
  1000 users = 5 GB/s (need multiple volumes or instance)
```

---

### Real-World Performance Example

```
Scenario: MySQL database

Workload:
  Writes: 90% (updates, inserts)
  Reads: 10% (SELECT queries)
  
Performance target:
  5,000 queries/sec
  Each query = 4 I/Os (leaf page, index, internal page, write)
  Total: 20,000 IOPS needed
  
Volume chosen: io2 (64,000 IOPS available)
  Actual usage: 20,000 IOPS (within limit)
  Throughput: ~80 MB/s (within 1000 MB/s limit)
  Latency: 0.5-1ms p99
  
If gp3 used instead:
  gp3 IOPS limit: 16,000
  Can't handle 20,000 IOPS needed
  Queries queue up
  Application slows down (throttling)
```

---

## Replication and Durability

### Multi-AZ Replication

```
EBS volume (default):
  us-east-1a: Master copy on one physical disk
  Replication: 2 additional copies within AZ
  Synchronous replication (wait for all copies)
  
Result:
  Can survive 1 disk failure
  Cannot survive entire AZ failure
  Uptime: 99.95%
  
Multi-AZ enabled (better durability):
  us-east-1a: Master copy
  us-east-1b: Standby copy
  Asynchronous replication (fast writes)
  
  On us-east-1a failure:
    1. Detect (1-2 minutes)
    2. Promote us-east-1b copy
    3. Detach old volume
    4. Attach new volume
  
  Downtime: 5-10 minutes
  Uptime: 99.99%
```

---

## Snapshots and Recovery

### Snapshot Process

```
Volume: 1000 GB database
Snapshot initiated:
  
First snapshot (full):
  Entire 1000 GB captured
  Stored in S3 (incremental)
  Physically: Stored on disk with multiple replicas
  Time: 30-60 minutes
  Cost: $0.05 per GB/month
  
Subsequent snapshots (incremental):
  Only changed blocks copied
  If 100 GB changed since last snapshot:
    Snapshot size: 100 GB (not 1000 GB)
    Time: 10 minutes
    Incremental saving: 90% faster
```

---

### Restore from Snapshot

```
Disaster scenario:
  Production database volume corrupted
  Last snapshot: 2 hours ago
  Data loss acceptable: Up to 2 hours
  
Recovery:
  1. Create volume from snapshot (500 GB)
     Size: 500 GB
     Availability Zone: us-east-1c (could be different AZ)
     Status: "creating" (2 minutes)
  
  2. Attach to new instance
     /dev/sdf → /mnt/restored
  
  3. Verify data
     Database is accessible
     Data current up to snapshot time (2h old)
  
  4. Application recovers
     Customers see 2 hours of data loss
     Acceptable within RTO/RPO SLA
     
Total recovery time: 15 minutes
Total data loss: 2 hours
```

---

## RAID Configurations

### RAID 0 (Striping)

```
Goal: Increase throughput

Setup:
  Volume A: 500 GB io2, 32k IOPS
  Volume B: 500 GB io2, 32k IOPS
  
  RAID 0 stripe across both volumes
  
Result:
  Total capacity: 1000 GB
  Total IOPS: 64k (32k + 32k)
  Total throughput: 1000 MB/s
  
Downside:
  Fault tolerance: NONE
  If Volume B fails: Entire stripe lost (data corruption)
  
Use case:
  High-throughput databases (acceptable failure risk)
  Cache layers (data not critical)
  
Not recommended for:
  Critical databases (data loss unacceptable)
  Persistent state
```

---

### RAID 1 (Mirroring)

```
Goal: Durability + throughput

Setup:
  Volume A: 500 GB (data)
  Volume B: 500 GB (mirror of A)
  
Writes:
  Write to A
  Write to B (both must succeed)
  Synchronous (slow)
  
Reads:
  Can read from A or B (in parallel)
  2x read throughput vs single volume
  
Fault tolerance:
  Single volume failure: Still operational
  If Volume A fails:
    Keep reading from Volume B
    Repair: Rebuild from B to new volume
    Time: 1-2 hours
  
Trade-off:
  Capacity halved (500 GB effective, 1000 GB provisioned)
  Cost: 2x single volume
  More durable
  
Use case:
  Critical databases (durability > capacity)
```

---

### RAID 10 (1+0)

```
Goal: Both durability and throughput

Setup:
  Volume A: 250 GB → Mirrored to Volume B
  Volume C: 250 GB → Mirrored to Volume D
  Volumes A+C striped together
  
Result:
  Total capacity: 500 GB effective (1000 GB provisioned)
  Throughput: 2x (striped pair)
  Durability: Any single volume can fail
  
Fault tolerance:
  Lose A: C still works (mirrors protect C)
  Lose B: A still works (mirrors protect A)
  
Complex but optimal for:
  Mission-critical databases (both need speed + durability)
  Financial systems
```

---

## Volume Expansion and Migration

### Online Volume Expansion

```
Current state:
  Volume size: 500 GB
  Free space: 50 GB (90% full)
  Application: Running without issues
  
Volume expansion:
  AWS console: Modify volume size → 1000 GB
  Time: Immediately (size increased)
  
Application-level:
  Filesystem still sees 500 GB initially
  Need to expand filesystem:
    - sudo resize2fs /dev/xvdf
    - Time: 30 seconds
    - No downtime (ext4 supports online expand)
  
Result:
  Application sees 1000 GB
  No service interruption
  Growth cost: $40/month (500 more GBs)
  
Note:
  Cannot shrink volumes (need new smaller volume)
```

---

### Volume Migration

```
Scenario: Need to migrate from gp2 to gp3 (newer, faster)

Option 1: Snapshot and restore
  1. Snapshot old volume
  2. Create new gp3 from snapshot
  3. Attach to instance
  4. Test
  5. Detach old, keep new
  
  Downtime: 5-10 minutes
  
Option 2: Filesystem-level migration (zero downtime)
  1. Attach new gp3 volume alongside old gp2
  2. rsync data from gp2 to gp3
  3. Switch mount point (kill app, remount, restart)
  4. Verify
  5. Detach old volume
  
  Downtime: < 1 minute (app restart only)
```

---

## Performance Tuning

### I/O Scheduling

```
Default: CFQ (Completely Fair Queueing)
  - Fair to all processes
  - Good for interactive use
  
For databases: Change to deadline or noop
  - Minimize latency variance
  - Prioritize urgent I/Os
  
Example (Linux):
  echo deadline > /sys/block/xvdf/queue/scheduler
  
Result:
  p99 latency: 10ms → 2ms
  More predictable performance
```

---

### EBS Optimization

```
EBS-optimized instance:
  Dedicated I/O pipeline to EBS
  No sharing with network traffic
  
Cost difference: +$0.05/hour for io2-optimized instance
  But: Guarantees throughput
  
Benefit:
  Throughput: 1000 MB/s guaranteed
  Latency: More predictable
  
Without optimization:
  Throughput: Shared with network
  Latency: Can spike if network busy
  
Critical for:
  High-performance databases
  Streaming applications
```

---

## Related Fundamentals

- [Object Storage](object-storage.md) – S3, durability comparison
- [Backup and Recovery](backup-and-recovery.md) – Snapshot strategies
- [Disaster Recovery Planning](../disaster-recovery/disaster-recovery-planning.md) – RTO/RPO

---

**Status**: ✅ Complete. Covers EBS, IOPS, replication, RAID, snapshots, performance tuning.
