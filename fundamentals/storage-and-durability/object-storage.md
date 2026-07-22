# Object Storage

S3-style storage for unstructured data (images, videos, backups).

---

## TL;DR

- **Object storage**: Key-value for large files (S3, GCS, Azure Blob)
- **Durability**: 11-nines (99.999999999%)
- **Scaling**: Unlimited storage
- **Cost**: $0.02/GB/month
- **Use**: Media, backups, data lakes

---

## S3 Basics

```
Bucket: storage-example
Objects:
  /users/profile.jpg (1MB)
  /videos/demo.mp4 (500MB)
  /backups/2026-01-01.tar.gz (100GB)

Access:
  GET /users/profile.jpg
  PUT /videos/demo.mp4
  DELETE /backups/old.tar.gz
```

---

## Durability & Replication

```
S3 replicates across 3 AZs
Survives 2 AZ failures
Durability: 99.999999999%

Probability of data loss: ~1 in 10 billion
```

---

## Related Fundamentals

- [Disaster Recovery](disaster-recovery.md) – Backup strategy

---

**Status**: ✅ Complete.

