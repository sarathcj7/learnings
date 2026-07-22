# Cost Optimization & TCO

Making systems efficient means spending less on infrastructure. Understanding cost drivers and optimization strategies separates profitable companies from money-losers.

---

## TL;DR

- **Infrastructure cost**: Largest component (50-70% of ops budget)
- **Cost drivers**: Compute, storage, bandwidth, managed services premiums
- **Optimization timeline**: Short term (instance rightsizing), medium (architecture), long term (custom infrastructure)
- **Trade-offs**: Cost vs performance, simplicity, operational overhead
- **Monitoring**: Track cost per customer, cost per QPS to spot waste

---

## Cost Breakdown

### Typical SaaS Cost Structure

```
Monthly budget: $100k

Infrastructure (70%): $70k
  Compute (servers): $40k (58%)
  Database (managed): $15k (21%)
  Storage (S3): $10k (14%)
  Network bandwidth: $5k (7%)
  
Ops & Engineering (20%): $20k
  On-call engineers: $12k
  Tools/monitoring: $5k
  Incident costs: $3k
  
Development (10%): $10k
  CI/CD, testing, tooling

Result: Save 10% on infrastructure → $7k/month = $84k/year savings!
```

---

## Cost Drivers per Component

### Compute (Largest Cost)

```
AWS EC2 pricing examples (on-demand):
  t3.xlarge (4 CPU, 16GB RAM): $0.166 per hour = $1,200/month
  c5.4xlarge (16 CPU, 32GB RAM): $0.68 per hour = $5,000/month
  m5.24xlarge (96 CPU, 384GB RAM): $4.61 per hour = $33,500/month
  
Optimization options:
  1. Reserved instances (pay upfront) → 30-40% cheaper
  2. Spot instances (spare capacity) → 70% cheaper (but interruptible)
  3. Right-sizing (use smaller than needed) → 30-50% savings
  4. Consolidation (use fewer servers) → 20-40% savings
```

---

### Database (Second Cost Driver)

```
AWS RDS pricing (managed database):
  db.t3.medium (2 CPU, 4GB): $160/month
  db.r5.2xlarge (8 CPU, 64GB): $3,400/month
  
Managed service premium:
  Self-hosted equivalent: 50-60% cheaper
  But requires: DBA team ($200k+ salary)
  
Decision:
  < $5k/month: Use managed (easier)
  > $5k/month: Consider self-hosted (requires DBA)
```

---

### Storage (Lowest Cost)

```
AWS S3:
  Standard storage: $0.023 per GB per month
  1TB of data: $23/month (cheap!)
  
Bandwidth:
  Data egress: $0.09 per GB (expensive!)
  
Example:
  1TB stored: $23/month
  1TB egress per month: $92/month (4x storage cost!)
  
  Optimization: CDN caching → reduce egress 80% → save $74/month
```

---

## Short-Term Optimizations (No Code Changes)

### 1. Right-Sizing Instances

```
Current deployment:
  10 servers: c5.4xlarge ($50k/month)
  CPU utilization: 30%
  Memory utilization: 40%
  
Analysis:
  If 30% CPU utilized, can use smaller instance type
  
Downsize to: t3.xlarge (meets 30% max)
  Cost: $12k/month (75% savings!)
  
Tradeoff: Smaller headroom (can't handle spikes well)
Solution: Keep 2 large instances, downsize 8 → $20k/month (60% savings)
```

---

### 2. Reserved Instances

```
On-demand: $50k/month
Reserved (1-year): 30% discount
  Annual cost: $50k × 12 × 0.7 = $420k
  Monthly cost: $35k (saves $15k/month = $180k/year)
  
Tradeoff:
  Pro: 30% cheaper
  Con: Commitment (can't reduce on short notice)
  
When to use: Baseline capacity that won't change
When not to: Startup with unknown growth
```

---

### 3. Spot Instances

```
On-demand: $5 per hour
Spot: $1.50 per hour (70% cheaper!)

Use case: Batch jobs, cache rebuild, non-critical work

Example:
  Spend 100 hours on batch job per month
  On-demand: $500/month
  Spot: $150/month (saves $350/month)
  
Tradeoff:
  Pro: Very cheap
  Con: Can be interrupted (AWS reclaims with 2min notice)
  
Solution: Use for stateless, retryable work
```

---

## Medium-Term Optimizations (Architecture Changes)

### 1. Database Optimization

```
Current:
  4 RDS instances (db.r5.4xlarge): $13.6k/month
  Storage: 2TB = $46/month
  Data egress: Not applicable (internal)
  
Optimization options:
  1. Add read replicas for scaling: More cost
  2. Shard data: Reduce per-instance size
     Shard 1-4 on db.r5.2xlarge: $6.8k/month (50% savings)
  3. Switch to Aurora (managed sharding): Similar cost, better performance
  4. Cache more (Redis): $2k/month, reduces DB load 40%
     New DB cost: $6.8k × 60% = $4.1k (+ $2k cache = $6.1k total savings)
```

---

### 2. Caching Strategy

```
Current: 100k req/sec, all hit database
  Database load: 100k QPS → need 50 large instances ($50k/month)
  
Add Redis cache:
  Cache cost: $2k/month
  If 80% hit rate: Database load drops 80%
  New database need: 10 instances ($10k/month)
  Savings: $40k/month (minus $2k cache) = $38k/month gross savings!
```

---

### 3. CDN for Content

```
Current: Serve images from origin servers
  Bandwidth egress: 100GB/day × $0.09/GB = $270k/month
  
Add Cloudflare CDN:
  CDN cost: $2k/month (cached hits)
  Bandwidth from origin: 20GB/day (80% cache hit)
  Egress cost: 20GB × 30 × $0.09 = $54/month
  Total: $2k + $54 = $2.054k/month
  
  Savings: $270k - $2k = $268k/month!!! (99% savings)
```

---

## Long-Term Optimizations (Custom Infrastructure)

### 1. Self-Hosting Database

```
Managed database (RDS):
  $15k/month infrastructure
  
Self-hosted:
  Hardware: $30k (server cost)
  DBA salary: $200k/year = $16.6k/month
  Total: $46.6k/month
  
Seems expensive BUT:
  Saves on managed premium, enables custom tuning
  Breakeven at 10 years ($550k vs $600k)
  
When justified: > $15k/month database spend
```

---

### 2. Custom Kubernetes Cluster

```
Managed Kubernetes (AWS EKS):
  Control plane: $0.1 per cluster per hour = $73/month
  Nodes (10): t3.xlarge @ $120/month = $1200/month
  Total: $1.3k/month
  
Self-hosted Kubernetes:
  Nodes (10): $1200/month
  No control plane cost
  Requires: Kubernetes expertise
  Total: $1.2k/month (saves $100/month, but risky)
  
ROI: Negative. Only do if you have platform team.
```

---

### 3. Custom Infrastructure

```
At $1M+/month infrastructure spend:
  Even 5% savings = $50k/month

Options:
  - Co-location in data centers (cheaper bandwidth)
  - Custom hardware (optimize for workload)
  - In-house infrastructure team
  
Example: Netflix, Google, Amazon
  Built custom infrastructure → 50% cost reduction
  But: Requires 50-100 engineers, only justified at massive scale
```

---

## Cost Monitoring & Alerts

### Metrics to Track

```
Cost per customer:
  Total infrastructure cost / paying customers
  Alert if increasing (indicates inefficiency)
  
Cost per QPS:
  Total infrastructure cost / peak QPS
  Benchmark: $1-5 per QPS typical (depends on SLA)
  Alert if > threshold (potential waste)
  
Cost per feature:
  Attribute infrastructure cost to feature (harder)
  Example: Search feature costs $10k/month
  If low usage, might discontinue
  
Bandwidth/storage as % of total:
  If bandwidth > 30% → Opportunity for CDN
  If storage > 20% → Opportunity for archival
```

---

## Optimization Decision Tree

```
Start: Minimize cost

1. Is infrastructure > 50% of budget?
   No → Stop (not main cost driver)
   Yes → Continue
   
2. Is peak utilization < 50%?
   Yes → Right-size instances (quick win, 30% savings)
   No → Continue
   
3. Database cost > 20% of infrastructure?
   Yes → Add caching or shard (40-50% savings)
   No → Continue
   
4. Bandwidth > 30% of infrastructure?
   Yes → Add CDN (50-80% savings)
   No → Continue
   
5. Infrastructure > $10k/month?
   Yes → Consider reserved instances (30% savings)
   No → Stop (managed cost fine)
   
6. Infrastructure > $50k/month?
   Yes → Consider self-hosting or custom hardware
   No → Stop (no longer makes economic sense)
```

---

## Trade-offs Summary

| Choice | Alternative | Rationale |
|---|---|---|
| **Right-sizing** | Over-provisioning | Saves cost with minimal risk |
| **Reserved instances** | On-demand | 30% savings if utilization stable |
| **Managed services** | Self-hosted | Simpler ops, worth premium unless very large |
| **CDN** | Origin servers | Huge bandwidth savings, small CDN cost |
| **Caching** | Scale database | Enable 10x+ scale at 1/10th cost |

---

## Related Fundamentals

- [Capacity Planning/Scale Estimation](scale-estimation-framework.md) – Right-sizing starts here
- [Database Sizing](database-sizing.md) – Understand what drives DB costs
- [Caching](../caching/) – Massive cost leverage (10x scale, 1x cost)

---

**Status**: ✅ Complete. Covers cost drivers, short/medium/long-term optimizations, monitoring.

