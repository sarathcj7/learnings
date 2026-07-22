# Backup Strategies

Full, incremental, and differential backups. RPO/RTO, cross-region replication, testing.

---

## TL;DR

- **Full backup**: Copy entire dataset (slow, complete recovery)
- **Incremental backup**: Copy only changes since last backup (fast, requires chain)
- **Differential backup**: Copy all changes since last full backup (faster than incremental)
- **RPO**: Recovery Point Objective (max acceptable data loss) → Backup frequency
- **RTO**: Recovery Time Objective (max acceptable downtime) → Backup restoration speed
- **Cross-region**: Backup to different region for disaster protection
- **Testing**: Regularly verify backups are restorable (practice restores)

---

## Backup Types

### Full Backup

Complete copy of entire dataset.

```
Dataset size: 1TB
Time: 10 minutes (100MB/s write speed)
Cost: 1TB storage

Backup schedule:
  Weekly full backups
  Mon full: 1TB
  Tue full: 1TB
  Wed full: 1TB
  ...
  
Storage overhead: 4TB for 4 weeks

Recovery:
  Restore from one full backup
  Time: 10 minutes
  Simplest (no dependency chain)
```

### Incremental Backup

Copy only data changed since last backup.

```
Day 1: Full backup (1TB)
  Storage: 1TB
  Time: 10 minutes
  
Day 2: Incremental (100GB changed)
  Storage: 100GB
  Time: 1 minute
  
Day 3: Incremental (80GB changed)
  Storage: 80GB
  Time: 48 seconds
  
Day 4: Incremental (90GB changed)
  Storage: 90GB
  Time: 54 seconds
  
Total storage for week: 1TB + 100GB + 80GB + 90GB = 1.27TB
Time per backup: ~1 minute (vs 10 minutes for full)

Recovery (Day 4 data loss):
  1. Restore Day 1 full backup (1TB)
  2. Restore Day 2 incremental
  3. Restore Day 3 incremental
  4. Restore Day 4 incremental
  
  Total time: 10 + 1 + 48s + 54s = ~11 minutes
  Dependency chain: Must restore in order
```

### Differential Backup

Copy all changes since last full backup.

```
Day 1: Full backup (1TB)
  Storage: 1TB
  
Day 2: Differential (100GB changed)
  Storage: 100GB
  Time: 1 minute
  
Day 3: Differential (150GB changed since Day 1)
  Storage: 150GB (includes Day 2 changes + new changes)
  Time: 90 seconds
  
Day 4: Differential (200GB changed since Day 1)
  Storage: 200GB (includes Days 2-3 changes + new changes)
  Time: 2 minutes
  
Total storage for week: 1TB + 100GB + 150GB + 200GB = 1.45TB
Slower than incremental (more data), faster to restore

Recovery (Day 4 data loss):
  1. Restore Day 1 full backup (1TB, 10 min)
  2. Restore Day 4 differential (200GB, 2 min)
  
  Total time: 12 minutes
  Simpler: Only 2 backups needed (vs 4 for incremental)
```

---

## RPO vs RTO vs Backup Frequency

### RPO: Recovery Point Objective

Maximum acceptable data loss (time between backups).

```
Critical system (payment processor):
  RPO: 5 minutes
  → Must backup every 5 minutes
  → Can tolerate losing at most 5 minutes of transactions
  
  Backup strategy:
    Every 5 minutes: Incremental backup to S3
    Every hour: Full backup to separate region
    
  Cost: High (frequent backups)
  Storage: ~300GB/day (5-minute intervals)

Normal system (blog):
  RPO: 24 hours
  → Daily backup acceptable
  → Can tolerate losing entire day of posts
  
  Backup strategy:
    Daily: Full backup at 2am
    
  Cost: Low (1 backup/day)
  Storage: 1TB/month

RPO = How often can we afford to backup?
```

### RTO: Recovery Time Objective

Maximum acceptable downtime before system restored.

```
RTO = 30 minutes (max downtime tolerance)

Strategy 1: Restore from backup
  Restore 1TB from S3: 10 minutes
  Restore incremental backups: 5 minutes
  Total: 15 minutes < 30 minutes ✓ (acceptable)
  
Strategy 2: Manual restore
  Notify team: 5 minutes
  Locate backup: 2 minutes
  Download backup: 10 minutes
  Verify integrity: 5 minutes
  Restore: 10 minutes
  Total: 32 minutes > 30 minutes ✗ (too slow)
  
  Improvement: Automate restore process
  Automated restore: 15 minutes ✓

RTO = How fast can we restore?
```

---

## Cross-Region Backups

Backup data to different geographic region for disaster protection.

```
Primary region: us-east (main application)
Backup region: us-west (S3 or equivalent)

Disaster scenario:
  us-east region destroyed by earthquake
  
  Traditional backup (same region):
    ✗ Backup destroyed too
    ✗ No recovery possible
    
  Cross-region backup:
    ✓ us-west backup untouched
    ✓ Restore from us-west
    ✓ Service recovered

Implementation:
  1. Point-in-time backups every 5 minutes
  2. Replicate to us-west S3 (async, low latency cost)
  3. Database snapshots encrypted and stored
  
Cost:
  S3 storage (us-west): $0.023/GB/month
  Data transfer (us-east → us-west): $0.02/GB
  For 1TB database:
    Storage: $23/month
    Transfer: $20/month (for 1TB/month transfers)
    Total: ~$40-50/month
```

### Replication vs Backup

```
Replication (Active-Active):
  Primary: us-east
  Secondary: us-west (hot standby)
  
  Data sync: Real-time (milliseconds)
  RTO: <1 minute (automatic failover)
  RPO: 0 (no data loss)
  Cost: 2x infrastructure (expensive)
  Use: Critical systems (banks, payment processors)
  
Backup (Active-Passive):
  Primary: us-east
  Backup: us-west (cold storage)
  
  Data sync: Periodic (hourly/daily)
  RTO: 10-30 minutes (manual failover)
  RPO: 1-24 hours (data loss possible)
  Cost: Low (1.2x storage only)
  Use: Normal systems (most applications)
```

---

## Backup Testing

Regular testing ensures backups are usable (not corrupted).

```
Common failure modes:
  ✗ Backup file corrupted (wrong restore procedure)
  ✗ Missing dependencies (restore schema but not data)
  ✗ Restore slower than expected (RTO violated)
  ✗ Restore silently fails (wrong version)

Testing checklist:
  - [ ] Restore backup to staging environment
  - [ ] Run integrity checks (CHECKSUM database)
  - [ ] Query sample data (verify content correctness)
  - [ ] Measure restore time (confirm within RTO)
  - [ ] Test incremental chain (restore full + incremental)
  - [ ] Document restore procedure (for incident)
  - [ ] Practice manual restore (team training)

Recommended frequency:
  Critical systems: Weekly full test
  Normal systems: Monthly full test
  All systems: Annual "disaster recovery drill"
```

### Backup Verification Example

```
Testing process:

1. Restore backup to staging:
   aws s3 cp s3://backups/prod.2026-07-20.tar.gz .
   tar -xzf prod.2026-07-20.tar.gz
   restoreDB prod.backup prod.staging
   
2. Verify integrity:
   SELECT COUNT(*) FROM users; → 1,000,000 ✓
   SELECT SUM(balance) FROM accounts; → $50B ✓
   
3. Spot check:
   SELECT * FROM users WHERE userId = 12345;
   → Get user details, verify correctness
   
4. Measure time:
   Restore start: 2026-07-23 10:00:00
   Restore end: 2026-07-23 10:12:00
   Duration: 12 minutes (within 30min RTO ✓)
   
5. Document:
   Backup tested: 2026-07-23
   Result: PASS
   Time: 12 minutes
   Issues: None
```

---

## Incremental vs Differential Comparison

| Aspect | Incremental | Differential |
|---|---|---|
| **Storage (1 week)** | 1.27TB | 1.45TB |
| **Backup time** | 1 min each | 1-2 min each |
| **Restore time** | 11 min (4 backups) | 12 min (2 backups) |
| **Restore complexity** | Chain dependency | Simple (full + latest) |
| **Failure impact** | Lost backup breaks chain | Only latest differential matters |
| **Use case** | Large datasets, slow changes | Medium datasets, moderate changes |

---

## Production Considerations

1. **Backup encryption**: AES-256 encryption at rest and in transit
2. **Retention policy**: Daily for 30 days, monthly for 1 year
3. **Monitoring**: Alert if backup misses SLA (didn't complete in time)
4. **Restore testing**: Automated weekly restore tests to staging
5. **Documentation**: Runbook for manual restore in case of automation failure
6. **Cost optimization**: Use tiered storage (hot S3 for recent, Glacier for old)
7. **Checksum verification**: Compute and store MD5 for integrity checks

---

## Related Fundamentals

- [Disaster Recovery Planning](disaster-recovery-planning.md) – RTO/RPO definitions
- [Chaos Engineering](chaos-engineering.md) – Testing disaster scenarios
- [Replication](../databases/replication.md) – Alternative to backup strategy

---

**Status**: ✅ Complete. Covers backup types, RPO/RTO, cross-region, testing, trade-offs.
