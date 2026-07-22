# Backup and Recovery

Strategies for backing up data, recovery testing, and minimizing RPO/RTO.

---

## TL;DR

- **RPO** (Recovery Point Objective): Max acceptable data loss (5 min, 1 hour, 1 day)
- **RTO** (Recovery Time Objective): Max acceptable downtime (15 min, 1 hour)
- **Backup frequency**: Hourly, daily, weekly based on RPO
- **Recovery testing**: Regularly test restores to ensure backups work
- **Off-site backup**: Protect against regional disasters
- **Backup incremental**: Only backup changed data (save time and storage)

---

## Backup Strategies

### Full Backup

```
Full backup: Copy entire dataset
Size: 500 GB database
Time: 30 minutes
Storage: 500 GB
Cost: $10/month (S3 storage)

Frequency: Weekly (Sunday)

Restore time: 30 minutes
Data loss: Up to 1 week

Use case:
  Low-frequency data changes
  Long retention requirements (compliance)
  Small enough that weekly copy is acceptable
```

---

### Incremental Backup

```
Incremental backup: Copy only changed blocks since last backup

Base backup: Sunday (500 GB)
Monday: 10 GB changed → Backup 10 GB
Tuesday: 5 GB changed → Backup 5 GB
Wednesday: 15 GB changed → Backup 15 GB

Storage usage:
  500 + 10 + 5 + 15 = 530 GB (vs 3500 GB if all full)
  85% storage savings

Restore full Wednesday:
  Restore Sunday (500 GB) - 30 min
  Apply Monday (10 GB) - 3 min
  Apply Tuesday (5 GB) - 2 min
  Apply Wednesday (15 GB) - 4 min
  Total: 39 minutes

Backup time: Faster (only changed data)
Restore time: Slower (must replay changes)
```

---

### Differential Backup

```
Differential backup: Copy all blocks changed since last FULL backup

Base backup: Sunday (500 GB)
Monday: 10 GB changed → Backup 10 GB (from Sunday)
Tuesday: 5 GB new + 10 GB modified = 15 GB → Backup 15 GB (from Sunday)
Wednesday: 20 GB new + 10 GB modified = 30 GB → Backup 30 GB (from Sunday)

Storage usage:
  500 + 10 + 15 + 30 = 555 GB (more than incremental)
  But less than full weekly backups

Restore Wednesday:
  Restore Sunday (500 GB) - 30 min
  Apply Wednesday differential (30 GB) - 10 min
  Total: 40 minutes

Advantage: Faster restore than incremental (no replay chain)
Disadvantage: More backup storage than incremental
```

---

### Continuous/Real-Time Backup

```
Continuous replication: Every write replicated to backup storage instantly

Setup:
  Primary database: Receives writes
  Backup storage: Replicates every transaction
  Replication lag: < 1 second

RPO: < 1 second (almost no data loss)
RTO: 1-2 minutes (promote backup as primary)

Use case: Financial systems, payment processing
Cost: High (continuous replication expensive)

Implementation:
  Database replication (MySQL, PostgreSQL with streaming)
  Write-ahead logs shipped to S3
  Backup storage always up-to-date
```

---

## Backup Frequency and RPO

### Setting RPO Based on Business Needs

```
Online banking:
  RPO: 1 minute (acceptable data loss: 1 minute)
  Backup frequency: Every minute
  Backup storage: $1000/month
  Cost justified: Financial accuracy critical

Video streaming metadata:
  RPO: 1 hour (acceptable data loss: 1 hour)
  Backup frequency: Every hour
  Backup storage: $100/month
  Cost: Reasonable, data recreation possible

Analytics data:
  RPO: 1 day (acceptable data loss: 1 day)
  Backup frequency: Once daily
  Backup storage: $10/month
  Cost: Cheap, data can be recomputed

Choosing RPO:
  Business requirement (SLA, regulations)
  Cost tolerance
  Data criticality
  Recovery capability
```

---

## Recovery Testing

### Backup Verification

```
Monthly backup test:

1. Select random backup (e.g., week 3, day 2)
2. Restore to separate environment (not production)
3. Verify:
   - Database integrity (no corruption)
   - Data completeness (all tables present)
   - Timestamp: Restore point correct
   - Application works: Queries run successfully

What happens if test fails:
  Backup believed working actually corrupt
  Disaster strikes: No restore possible
  Business loss: Entire database unrecoverable

Test result: 95% of companies discover backup issues only during actual disaster
Solution: Regular recovery testing (monthly mandatory)
```

---

### Disaster Recovery Drill

```
Quarterly drill: Full production failover simulation

Step 1: Simulate primary region failure
  Create snapshot of production data
  Deploy to backup region (isolated)
  Do NOT use production traffic

Step 2: Test failover automation
  DNS points to backup region
  Applications connect to backup region
  Measure: Time to full functionality

Step 3: Validate data
  Backup region has all data
  No data loss observed
  All users can be served

Step 4: Document findings
  Issues discovered during drill
  Fix before real disaster
  Update playbooks

Results:
  Typical RTO claims: 1 hour
  Actual discovery: 3 hours needed
  Time to fix: Weeks before real disaster
  Value: Prevents catastrophic failure
```

---

## Backup Storage and Off-Site

### On-Site vs Off-Site

```
On-site backup (same region):
  Storage: Local disk or EBS snapshots
  Pros: Fast restore (milliseconds)
  Cons: Region disaster = all backups lost
  Example: Data center fire destroys both primary and backup
  
Off-site backup (different region):
  Storage: S3 in different region or different cloud
  Pros: Protected against regional disasters
  Cons: Slower restore (network latency)
  Recovery time: 15-30 minutes (data transfer)
  
3-2-1 Backup Rule:
  3 copies: Original + 2 backups
  2 media: Different storage types (disk + tape/cloud)
  1 off-site: At least 1 copy remote location
  
Implementation:
  Copy 1: Incremental to local EBS (fast restore)
  Copy 2: Daily full to S3 same region (economy)
  Copy 3: Daily full to S3 different region (disaster recovery)
  
Cost: More storage, but acceptable risk reduction
```

---

## Recovery Procedures

### Step-by-Step Recovery

```
Disaster: Production database corrupted (2:30 PM)

Phase 1: Detection (2:30 PM - 2:32 PM)
  Alerts: Write failures spike
  Monitoring: 5-minute detection
  Decision: Corrupt database, require recovery
  
Phase 2: Preparation (2:32 PM - 2:35 PM)
  Access backup: Retrieve S3 backup from 1:00 PM
  Verify integrity: Check backup metadata
  Estimate: 30 GB database, 10 MB/s = 50 minutes
  
Phase 3: Restore (2:35 PM - 3:25 PM)
  Download backup (50 min)
  Check data integrity (5 min)
  Point database to backup storage
  
Phase 4: Validation (3:25 PM - 3:35 PM)
  Run queries: Verify data
  Check consistency: No corrupted indexes
  Application health check
  
Phase 5: Cutover (3:35 PM - 3:40 PM)
  Update DNS: Point to recovered database
  Monitor: Application connections successful
  
Result:
  RTO achieved: 70 minutes vs 4-hour SLA ✓
  RPO: 35 minutes data loss (acceptable)
  Customers: 1 hour 10 min downtime (on par with SLA)
```

---

## Backup and Recovery Anti-Patterns

### Anti-Pattern 1: No Recovery Testing

```
Company policy:
  Backup database daily ✓
  Retention: 30 days ✓
  Disaster strikes: Database corrupted
  
Recovery attempt:
  Restore from backup
  "Restore failed: Backup corrupt"
  No recovery possible
  
Root cause:
  Last recovery test: Never
  Backup file format changed: Unknown
  Corruption undetected: No verification
  
Result: Total data loss, business shutdown
```

---

### Anti-Pattern 2: Single Point of Failure

```
Backup setup:
  Database on /var/lib/database
  Backup script: cp -r /var/lib/database /backup/
  Backup location: Same disk
  
Disaster: Disk failure
  Primary database: Gone
  Backup: Gone (same disk)
  
Solution:
  At least 2 different storage media
  At least 1 off-site copy
  Verify integrity regularly
```

---

### Anti-Pattern 3: Untested Restore Time

```
SLA: RTO 1 hour

Claimed recovery:
  Backup restore: 30 minutes
  Application startup: 10 minutes
  Total: 40 minutes ✓
  
Reality during disaster:
  Backup file download: 1 hour (network slow)
  Data integrity check: 15 minutes (corruption found)
  Database recovery: 30 minutes (must rebuild indexes)
  Application startup: 5 minutes
  Total: 2 hours 50 minutes ✗
  
SLA violated by 2x
```

---

## Backup Tools and Technologies

### Database-Specific Tools

```
MySQL:
  mysqldump: Logical backup (human-readable SQL)
  Physical backup: Copy /var/lib/mysql with xtrabackup
  Replication: Continuous to replica server
  
PostgreSQL:
  pg_dump: Logical backup
  pg_basebackup: Physical backup with streaming
  WAL archiving: Write-ahead logs to S3
  
MongoDB:
  mongodump: Logical backup
  File snapshots: LVM snapshots of data files
  Replication: Replica sets for continuous backup

Tradeoff:
  Logical: Human-readable, portable, large size
  Physical: Faster, smaller, less portable
  Replication: Fastest restore, more complex
```

---

### Cloud Backup Services

```
AWS:
  EBS snapshots: Block storage backups
  AWS Backup: Centralized backup across services
  DMS: Database migration and continuous replication
  
Google:
  Persistent disk snapshots: Incremental backups
  Cloud SQL automated backups: Database-specific
  Cross-region replication
  
Azure:
  Managed backups: Database-specific
  Backup Vault: Centralized management
  Site Recovery: Disaster recovery automation
```

---

## Related Fundamentals

- [Disaster Recovery Planning](../disaster-recovery/disaster-recovery-planning.md) – RTO/RPO strategies
- [Backup Strategies](../disaster-recovery/backup-strategies.md) – Deep dive into backup types
- [Block Storage](block-storage.md) – EBS snapshots

---

**Status**: ✅ Complete. Covers backup strategies, RPO/RTO, recovery testing, tools.
