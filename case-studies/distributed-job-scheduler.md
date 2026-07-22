# Distributed Job Scheduler

*Design a job scheduler like Airflow, Kubernetes, Chronos. Millions of jobs, retry logic, dependency management, fault tolerance.*

## Problem Statement

Build a distributed job scheduler. Run 10M+ jobs/day across a cluster. Jobs have dependencies (job B waits for job A), can fail and retry, must persist state across restarts.

## Scale Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Jobs/day** | 10M | ~115 jobs/sec |
| **Peak concurrent** | 10M × 1% = 100k | 100k running jobs |
| **Executors** | 100k jobs / 10 per executor | 10k executors |

## Architecture

```
Job Queue (Kafka):
  Topic: "jobs"
  Partitions: By job_id (order per job, parallel across partitions)
  
Job Metadata (Database):
  job_id, status, retries, dependencies, executor_id
  
Scheduler:
  Read job definitions
  Check dependencies (are prerequisites done?)
  Submit ready jobs to queue
  Monitor completion
  
Executors (workers):
  Consume jobs from queue
  Execute locally
  Report status (done/failed)
  
State Store (Consensus-based, e.g., Zookeeper):
  Track: Job status, executor health, coordinator election
```

## Challenges

### Dependency Management

```
Job A → Job B → Job C

Scheduler logic:
1. Job A ready (no deps) → Submit
2. Job A completes → Mark done
3. Check Job B deps (A done? yes) → Submit B
4. Continue chain

If A fails:
  Retry A (up to max_retries)
  On success: Continue B
  On final fail: Skip B,C (or trigger on_failure handlers)
```

### Executor Failure

```
Executor 1 running Job X crashes.
Scheduler detects: Executor 1 heartbeat missing for 30s
Action:
  Mark Job X as FAILED
  Re-submit to another executor
  If retries < max: Retry
  Else: Mark FAILED, notify user
```

### Idempotency

```
Job submitted twice (network retry):
  First attempt: Runs, produces output
  Second attempt: Runs again (duplicate work, ok if idempotent)
  
Solution: Job ID is idempotency key
  "If job_id=123 already run, return cached result"
```

## Bottlenecks & Scaling

**Bottleneck**: Scheduler checking dependencies (10M jobs).

**Solution**:
- Kafka partitions by job_id (independent jobs in parallel)
- Caching dependency state
- Batch status updates (not per-job)

## Trade-offs

| Choice | Alternative | Rationale |
|---|---|---|
| **Kafka for ordering** | Direct DB polling | Durability + scalability |
| **Consensus for state** | Single coordinator | Fault tolerance (coordinator failure → chaos) |
| **Async execution** | Synchronous | Concurrency + scale |

---

**Status**: ✅ Complete. Shows distributed coordination, fault tolerance, idempotency.
