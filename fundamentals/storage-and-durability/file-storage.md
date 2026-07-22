# File Storage

Network-based file systems (NFS, EFS). Shared storage for multiple instances, applications, and services.

---

## TL;DR

- **File storage**: Shared filesystem over network (NFS, EFS, SMB)
- **Use**: Multiple instances accessing same files, configuration sharing
- **Latency**: 1-10ms (network overhead vs block storage)
- **Throughput**: 100-1000 MB/s per instance
- **Scalability**: Grows dynamically (no provisioning)
- **Cost**: $0.30/GB/month for EFS
- **Trade-off**: Slower than block, simpler than object storage

---

## NFS (Network File System)

### NFS Architecture

```
NFS Server:
  Stores files and metadata
  Responds to mount requests
  Handles concurrent access
  
NFS Clients (multiple):
  Instance A: mounts /data
  Instance B: mounts /data
  Instance C: mounts /data
  
All instances see same files
  If A writes file.txt
  B and C see the update immediately
```

---

### Mount and Access

```
Server setup (manual):
  1. Export directory: /data
  2. Configure /etc/exports
     /data 10.0.0.0/8(rw,sync,no_subtree_check)
  3. Start NFS service
  4. Give out IP: 10.0.0.10

Client setup:
  sudo mount -t nfs 10.0.0.10:/data /mnt/shared
  
  Now client has access:
  cd /mnt/shared
  ls  # See all files
  echo "hello" > file.txt  # Write to shared storage
  
Persistence:
  Files in /mnt/shared persist after client restart
  Another client can access same files
  
Limitations:
  Manual server setup (not cloud-native)
  Single point of failure (server dies = no access)
  Network latency (10-20ms for NFS operations)
```

---

## EFS (Elastic File System - AWS)

### EFS vs EBS

```
EBS:
  Block device (must partition and format)
  Attached to single instance
  Persistent across reboots
  Performance: 16k-64k IOPS
  Downtime to switch instances: minutes
  
EFS:
  Filesystem (already formatted, ready to mount)
  Attached to multiple instances simultaneously
  Scales automatically
  Performance: 7.5k MB/s throughput
  Can switch instances instantly
  Fully managed (no admin overhead)
```

---

### EFS Mount and Usage

```
Setup (AWS cloud):
  1. Create EFS (one-click)
  2. Security group: Allow NFS port 2049
  3. Instances: Add to same VPC/AZ
  
Mount on instance:
  sudo mount -t nfs4 -o nfsvers=4.1 fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/data
  
  Or use EFS mount helper:
  sudo mount -t efs -o tls fs-12345678:/ /mnt/data
  
Applications:
  Web server reads templates from /mnt/data/templates
  Background job writes logs to /mnt/data/logs
  Database reads schema files from /mnt/data/schemas
  
Storage grows automatically:
  10 GB used → Charged for 10 GB
  100 GB used → Grows and charged for 100 GB
  No provisioning needed
```

---

### Performance Modes

```
General Purpose (default):
  Latency: ~1ms
  Throughput: 7.5 MB/s per instance
  Cost: $0.30/GB/month
  Best for: Most workloads (web servers, apps)
  
Max IO:
  Latency: Higher (5-10ms)
  Throughput: Unlimited (multiple instances)
  Cost: $0.40/GB/month
  Best for: Highly parallel workloads (HPC, data science)
  
Trade-off:
  General Purpose: Low latency, single-digit MB/s per instance
  Max IO: Higher latency, can add more instances for linear scaling
```

---

## Performance Characteristics

### Throughput Scaling

```
General Purpose EFS:
  Single instance: 7.5 MB/s throughput
  
  Bottleneck: Single connection
  Each instance gets 7.5 MB/s regardless of region
  
Scaling strategy:
  Need 50 MB/s total throughput
  Add 7 instances (7 × 7.5 = 52.5 MB/s)
  
  Each instance independently can read/write
  Aggregate throughput adds up
  
Max IO mode:
  No single-instance limit
  Throughput scales with instance count
  Need 50 MB/s: Might need only 5-10 instances
  
Example workload:
  Distributed ML training
  100 instances read training data from EFS
  Each reads at 7.5 MB/s
  Total: 750 MB/s (data always fresh to disk)
```

---

### Latency Comparison

```
Local SSD (EC2 attached):
  Latency: 0.5ms (microseconds)
  Throughput: 10 GB/s
  Availability: Single instance
  
EBS volume:
  Latency: 1-2ms
  Throughput: 250-1000 MB/s
  Availability: Single instance, persistent
  
EFS:
  Latency: 1-3ms
  Throughput: 7.5 MB/s per instance
  Availability: Multiple instances, automatic scaling
  
When EFS latency matters:
  Real-time analytics (< 50ms queries)
  High-frequency trading (< 10ms)
  Recommendation engines (< 100ms)
  
  If application acceptable with 100ms latency: EFS is fine
  Network overhead negligible
```

---

## Use Cases and Patterns

### Pattern 1: Shared Configuration

```
Setup:
  Central config server
  Maintains: /mnt/shared/configs/app.conf
  All instances mount EFS
  
Workflow:
  1. Ops updates /mnt/shared/configs/app.conf
  2. All instances see change (no redeploy)
  3. Application polls config file
  4. Picks up new values
  
Benefit:
  Feature flags (turn on/off features)
  Deployment rollback (revert config instantly)
  A/B testing (different configs per region)
  
Downside:
  Config changes not versioned by default
  Requires monitoring for correctness
```

---

### Pattern 2: Content Distribution

```
Use case: Web server serving user-uploaded content

Traditional (EBS):
  Each instance has own /var/www/uploads
  File A uploaded to Instance 1
  User requests from Instance 2: 404 Not Found
  Need load balancer affinity (sticky sessions)
  
With EFS:
  /mnt/data/uploads (mounted on all instances)
  File A uploaded to Instance 1 → /mnt/data/uploads/file_a
  User requests from Instance 2 → /mnt/data/uploads/file_a ✓
  Works instantly, no session affinity needed
  Scales horizontally (add instances, same storage)
```

---

### Pattern 3: Log Aggregation

```
Scenario: 100 instances, each writing logs

Problematic (instance-local):
  /var/log/app.log on each instance
  Central log server collects via syslog (network overhead)
  Network latency adds up
  
Optimized (EFS):
  /mnt/logs/instance-001.log (on EFS)
  /mnt/logs/instance-002.log (on EFS)
  ...
  
  Instances write to EFS (local filesystem performance)
  Central analyzer reads from single mount point
  No network hops (just one mount to EFS)
```

---

## Scaling Limits

### Single Filesystem Limits

```
Maximum capacity:
  Unlimited (can grow to petabytes)
  
Maximum throughput:
  General Purpose: 7.5 MB/s per instance
  Max IO: Scales with instances (up to 500 MB/s)
  
Maximum file count:
  Unlimited
  
Maximum file size:
  47.9 TB per file
  
Maximum concurrent connections:
  Scales automatically (cloud-managed)
```

---

### When to Scale Out

```
Performance target: 500 MB/s sustained read throughput

Option 1: General Purpose EFS
  7.5 MB/s per instance
  Need: 500 ÷ 7.5 = 67 instances
  Cost: Expensive if only reading files
  
Option 2: S3 (Object Storage)
  Multi-part download
  10,000 requests/sec possible
  Can achieve 500 MB/s with 50 concurrent connections
  Cost: $0.02/GB (cheaper than EFS)
  Tradeoff: Eventual consistency, not POSIX filesystem
  
Recommendation:
  < 50 MB/s: Use EFS
  > 500 MB/s: Use S3 or distributed cache (Redis)
  50-500 MB/s: Either works (depends on access pattern)
```

---

## Troubleshooting Common Issues

### Issue 1: High Latency

```
Symptom: Application slow, I/O wait high

Check:
  1. Instance type: Older instances might have poor network
  2. Throughput limit: 7.5 MB/s per instance (max for GP)
  3. File size: Accessing many small files is slow
  
Solution:
  - Use newer instance type (better network)
  - Upgrade to Max IO mode
  - Batch small file operations (reduce syscalls)
```

---

### Issue 2: "Stale NFS Handle" Errors

```
Symptom: Occasional application crashes, "stale NFS handle"

Cause: EFS mount lost connection, handle became invalid

Solution:
  1. Add NFS mount options:
     mount -t nfs4 -o nfsvers=4.1,hard,timeo=600 ...
  2. hard: Retry forever (not soft: fail quickly)
  3. timeo: Increase timeout to 600 (10 minutes)
  4. Ensures connection is reestablished automatically
```

---

### Issue 3: Throughput Bottleneck

```
Symptom: Backup job slow, hitting 7.5 MB/s ceiling

Measurement:
  dd if=/mnt/data/bigfile of=/dev/null bs=1M
  Result: 7.5 MB/s (consistent)
  
Root cause:
  Single instance limitation
  
Solutions:
  1. Parallel reads from multiple instances
     Instance A: Read first 10GB
     Instance B: Read second 10GB
     Total: 15 MB/s (double throughput)
     
  2. Use S3 transfer (faster for bulk data)
     aws s3 sync /mnt/data s3://mybucket --parallel
```

---

## Related Fundamentals

- [Object Storage](object-storage.md) – S3, alternative to shared filesystems
- [Block Storage](block-storage.md) – EBS, single-instance persistence
- [Distributed Systems](../distributed-systems-fundamentals/) – Handling network failures

---

**Status**: ✅ Complete. Covers NFS, EFS, performance, use cases, scaling limits.
