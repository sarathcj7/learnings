# Disaster Recovery & Business Continuity

RTO/RPO, backup strategies, multi-region failover.

---

## TL;DR

- **RTO**: Recovery Time Objective (max downtime tolerance)
- **RPO**: Recovery Point Objective (max data loss tolerance)
- **Backup**: Daily snapshots to separate region
- **Failover**: Automatic to backup region on disaster
- **Testing**: Regularly test recovery (chaos engineering)

---

## RTO vs RPO

```
Service: Payment system

RTO: 15 minutes (can tolerate 15min downtime)
RPO: 5 minutes (can tolerate losing 5min of transactions)

Strategy:
  Snapshot every 5 minutes → S3 other region
  On disaster: Restore to latest snapshot
  Downtime: 15 minutes (acceptable)
  Data loss: At most 5 minutes
```

---

## Multi-Region Failover

```
Primary region: us-east
Backup region: us-west

Disaster in us-east:
  1. Detect (2min)
  2. Promote us-west to primary (3min)
  3. Redirect DNS (1min)
  Total: 6min downtime

RTO: 15min ✓ (6min < 15min)
RPO: 5min ✓ (replication every 2min)
```

---

**Status**: ✅ Complete.

