# Storage & Durability

Object storage, block storage, file storage, backup strategies.

## Sub-topics

- **[Object Storage](object-storage.md)** ✅: S3, GCS, 11-nines durability, unlimited scalability
- **[Block Storage](block-storage.md)** ✅: EBS, Persistent Volumes, IOPS/throughput optimization, snapshots
- **[File Storage](file-storage.md)** ✅: NFS, EFS, multi-instance sharing, network filesystems
- **[Backup & Recovery](backup-and-recovery.md)** ✅: Point-in-time recovery, snapshot strategies, testing

## Why This Matters

- **Type selection**: Object for media/backups, block for DBs, file for shared access
- **Cost**: Object ($0.02/GB) vs block ($0.08/GB) vs file ($0.30/GB) — 15x difference
- **Performance**: Block <1ms, file 1-10ms, object 50-100ms — matters for databases
- **Durability**: 11-nines requires multi-region replication
- **Design interviews**: "How would you store 100TB of video?" requires understanding all three

## Interview Prep

- **[Question Bank](../../interview-prep/question-banks/storage-and-durability.md)**: 25+ questions on storage types
- **[Flashcards](../../interview-prep/flashcards/storage-and-durability.md)**: IOPS, throughput, durability SLAs

## Related Fundamentals

- [Databases](../databases/) – Persistent storage underneath
- [Disaster Recovery](../disaster-recovery/) – Backup storage strategies
- [Scalability](../scalability-and-load-balancing/) – Distributing storage load

## Status

✅ **COMPLETE**. 4/4 files written (Object, Block, File Storage, Backup & Recovery).

