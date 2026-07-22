# Chaos Engineering

Intentionally inject failures to verify system resilience. Chaos Monkey, failure injection, learning from failures.

---

## TL;DR

- **Chaos engineering**: Intentionally break systems in controlled ways (production-safe)
- **Chaos Monkey**: Random instance termination (Netflix tool)
- **Failure injection**: Kill processes, simulate network latency, storage unavailability
- **Blast radius**: Limit impact (single region, percentage of traffic)
- **Hypothesis-driven**: Make predictions, run experiment, learn from results
- **Production readiness**: Chaos experiments validate system's disaster recovery

---

## Why Chaos Engineering?

```
Failure modes without chaos engineering:
  1. Deploy new code → Causes bug in failover logic
  2. Customer reports: "Traffic went down for 30 minutes"
  3. Post-mortem: "We didn't know our backup wasn't working"
  4. Cost: $100k revenue lost, reputation damage
  
Failure modes with chaos engineering:
  1. Intentionally kill database server
  2. Observe: "Failover took 5 minutes, replication lag was 1GB"
  3. Fix: Improve replication protocol, automate failover
  4. Deploy fix, run chaos experiment again
  5. Result: "Failover now 30 seconds, no data loss"
  6. Confidence: We know system survives this failure
```

---

## Chaos Monkey (Netflix)

Randomly terminates instances to force resilience.

### How It Works

```
Netflix infrastructure: 1000s of EC2 instances across regions

Chaos Monkey runs:
  Hourly: Select random instance
  Action: Kill EC2 instance (terminate)
  Scope: Production traffic affected
  
System must:
  1. Detect instance failure (~30 seconds)
  2. Reroute traffic to healthy instances
  3. Auto-scale new instance to replace killed one
  
Result:
  Engineers build resilience into applications
  "If my app can survive random termination, it's production-ready"
```

### Configuration

```
Chaos Monkey settings:
  Frequency: Kill 1 instance every 30 minutes
  Scope: Production environment only
  Blast radius: <1% impact to overall system
  
  Blacklist:
    - Critical paths (authentication, payment processing)
    - During deployments (unstable time)
    - Between 9am-5pm during business hours (visibility needed)
  
  Whitelist:
    - Stateless services (web servers)
    - Services with redundancy (3+ instances minimum)
```

---

## Failure Injection Techniques

### Instance/Service Termination

```
Hypothesis:
  "If database server crashes, app automatically fails over to replica"
  
Experiment:
  1. Kill primary database server
  2. Measure time until replica promoted
  3. Verify queries work during failover
  
Results:
  Expectation: 5 seconds failover
  Actual: 45 seconds (slow detection)
  Action: Add heartbeat monitoring, reduce detection time
  
Repeat: Test again to verify fix
```

### Network Latency Injection

```
Hypothesis:
  "If database responds slowly (500ms latency), requests timeout correctly"
  
Experiment using toxiproxy:
  1. Insert toxiproxy between app and database
  2. Configure: Add 500ms latency to all queries
  3. Verify: App times out after 10 seconds (configured timeout)
  4. Check: Error handling and retry logic work
  
Results:
  Expected: Graceful degradation, error responses
  Actual: Some requests hang forever (retry loop bug)
  Action: Add circuit breaker, limit retries
  
After fix: Re-run experiment, confirm recovery works
```

### Storage Unavailability

```
Hypothesis:
  "If S3 becomes unavailable, app detects and caches locally"
  
Experiment:
  1. Mount S3 filesystem to app servers
  2. Simulate S3 timeout (respond with 503 Service Unavailable)
  3. Measure time until app switches to local cache
  4. Verify: No customer-facing errors
  
Results:
  Expected: <1 second fallback to cache
  Actual: 30 seconds (slow error detection)
  Action: Reduce S3 timeout threshold
  
Learnings: Local cache reduced latency by 10x
```

### Dependency Degradation

```
Hypothesis:
  "If payment processor is slow, checkout still completes"
  
Experiment:
  1. Rate-limit payment processor API (10 req/sec vs 1000 req/sec)
  2. Run load test (1000 concurrent checkout requests)
  3. Measure: Queue time, payment latency, user timeouts
  
Results:
  Expected: Queue requests, retry later, no errors
  Actual: Requests timeout, carts abandoned
  Action: Implement exponential backoff, queue requests locally
  
After fix: Users can checkout without payment processor degradation
```

---

## Blast Radius

Limit impact of chaos experiments to contain failures.

### Safe Experimentation

```
Blast radius levels (escalating):

Level 1: Staging environment (no real users)
  Hypothesis: "Killing database should trigger failover"
  Risk: None (test environment)
  
Level 2: Canary deployment (1% of traffic)
  Hypothesis: "New feature handles failures gracefully"
  Risk: 1% user impact if gone wrong
  
Level 3: Production (low traffic hours)
  Hypothesis: "Service survives zone failure"
  Risk: Minimal (off-peak hours, limited impact)
  
Level 4: Production (peak hours)
  Only after levels 1-3 confirm resilience
  Risk: High impact possible
  Only for critical resilience validation
```

### Blast Radius Controls

```
Killing instances:
  Max instances to kill: 5 (not 50)
  Time between kills: 10 minutes (allow recovery)
  Region limit: Single region only
  Blast radius: <5% traffic impact
  
Latency injection:
  Max latency: 500ms (not 10s)
  Duration: 30 seconds (not 1 hour)
  Affected services: Non-critical APIs only
  Rollback plan: Automatic if error rate >5%
  
Network partition:
  Scope: Internal network only
  Duration: <5 minutes
  Affected zones: One AZ only
  Automatic recovery: Restore connection after experiment
```

---

## Hypothesis-Driven Chaos

Structure experiments around predictions.

### Experiment Template

```
Title: "Database failover is automatic and fast"

Hypothesis:
  "When primary database becomes unavailable,
   system automatically promotes replica within 30 seconds,
   and queries complete successfully"

Method:
  1. Kill primary database server
  2. Monitor time until replica promoted
  3. Measure query latency during failover
  4. Check error rate (<1%)

Blast radius: 5% traffic (canary deployment)
Rollback plan: If error rate >5%, restore primary immediately

Expected results:
  ✓ Failover time: <30 seconds
  ✓ Error rate: <1%
  ✓ Query latency: <1 second after failover

Actual results:
  ✗ Failover time: 45 seconds (exceeded SLA)
  ✓ Error rate: 0.5% (acceptable)
  ✓ Query latency: 800ms (acceptable)

Findings:
  1. Detection logic is slow (15 seconds overhead)
  2. Replica promotion is fast (20 seconds)
  3. Connection reestablishment is fast (5 seconds)

Actions:
  1. Improve heartbeat monitoring (reduce detection time to 5s)
  2. Re-run experiment to verify improvement
  3. Target: <20 seconds total failover time
```

---

## Production Readiness Checklist

Chaos experiments validate production readiness.

```
Before deploying to production, run chaos experiments:

[ ] Database failure
    Kill primary DB, verify failover works, check RTO
    
[ ] Instance failure
    Terminate random server, verify traffic reroutes
    
[ ] Network latency
    Add 500ms latency between services, verify timeouts work
    
[ ] Dependency failure
    Block calls to external API, verify graceful degradation
    
[ ] Cascading failure
    Kill multiple services, verify circuit breakers work
    
[ ] Storage failure
    Simulate S3 unavailability, verify cache fallback
    
[ ] Rate limiting
    Overload critical service, verify backpressure handling
    
Only deploy if all experiments pass
```

---

## Common Chaos Tools

| Tool | Purpose | Example |
|---|---|---|
| **Chaos Monkey** | Kill random instances | Netflix, Amazon |
| **Gremlin** | Commercial chaos platform | Failure injection as a service |
| **Toxiproxy** | Network chaos (latency, failure) | Inject delays between services |
| **Pumba** | Docker chaos (kill containers) | Kubernetes/Docker environment |
| **Locust** | Load testing + chaos | Overload systems intentionally |
| **Chaos Toolkit** | Open-source framework | Define experiments in code |

---

## Learning from Failures

### Post-Experiment Analysis

```
After chaos experiment, document:

1. What we learned:
   - Failover took 45 seconds (slower than expected)
   - No data was lost
   - Some users saw timeout errors
   
2. Why it happened:
   - Heartbeat interval was 15 seconds (slow detection)
   - Replica promotion logic was synchronous (not parallelized)
   - Client retry timeout was too short (5 seconds)
   
3. What we'll fix:
   - Reduce heartbeat to 5 seconds
   - Parallelize replica promotion
   - Increase client timeout to 60 seconds
   
4. Next steps:
   - Implement fixes
   - Re-run experiment to verify
   - Document new failover process
```

### Continuous Improvement

```
Chaos engineering is iterative:

Month 1:
  Run: Kill 1 instance/day
  Learn: Failover takes 45 seconds
  Fix: Improve heartbeat monitoring
  Target: <20 second failover
  
Month 2:
  Run: Kill 2 instances/day
  Learn: Failover now 18 seconds ✓
  Learn: But cascading failure breaks circuit breakers
  Fix: Improve circuit breaker logic
  Target: Zero cascading failures
  
Month 3:
  Run: Kill 3 instances + inject latency
  Learn: System survives all scenarios
  Confidence: Ready for production chaos
  Action: Chaos Monkey enabled 24/7
```

---

## Production Considerations

1. **Monitoring**: Real-time dashboards to observe experiment impact
2. **Runbook**: Documented procedure to manually recover if needed
3. **Alerting**: Automatic alerts if experiment exceeds blast radius
4. **Rollback**: Automated restoration if error rate spikes
5. **Team**: On-call team reviews experiment results
6. **Documentation**: Post-mortems from each experiment

---

## Related Fundamentals

- [Disaster Recovery Planning](disaster-recovery-planning.md) – Recovery objectives
- [Backup Strategies](backup-strategies.md) – Testing backup restoration
- [Reliability and Resiliency](../reliability-and-resiliency/) – System robustness

---

**Status**: ✅ Complete. Covers chaos principles, experiments, blast radius, tools, learning.
