# Load Testing & Capacity Validation

Techniques to measure system capacity before going to production and find bottlenecks before users do.

---

## TL;DR

- **Load test**: Simulate traffic to find breaking points
- **Ramp-up**: Gradually increase load to find knee point (where performance degrades)
- **Sustained load**: Hold at peak traffic for 30+ minutes to find memory leaks
- **Tools**: Apache JMeter, Gatling, k6, wrk
- **Metrics**: Response time (p50/p99), error rate, throughput, CPU/memory

---

## Load Testing Phases

### Phase 1: Baseline (No Load)

```
Start: Single user makes single request

Measurement:
  Response time: 50ms
  CPU: 5%
  Memory: 2GB
  
Use: Compare against load test results
```

---

### Phase 2: Ramp-Up

```
Gradually increase users from 1 to peak:

T=0min: 1 user
T=1min: 10 users
T=2min: 50 users
T=3min: 100 users
T=4min: 500 users
T=5min: 1000 users (peak)

Observe:
  At what user count does response time increase?
  At what point does error rate spike?
  
Example results:
  1-100 users: 50ms response time (linear)
  100-500 users: 100-200ms (starting to degrade)
  500+ users: 500ms+ (database bottleneck)
  
Conclusion: System can handle 500 concurrent users before degrading
```

---

### Phase 3: Sustained Load

```
Hold at peak (1000 users) for 30-60 minutes:

Monitor:
  Memory usage over time (should be stable)
  Response time (should not degrade over time)
  Error rate (should stay < 1%)
  
Problem detection:
  Memory increasing 1MB/sec → Memory leak! (50GB after 12 hours)
  Response time increasing 1ms/sec → Cache becoming less effective
  Error rate creeping up → Resource exhaustion
  
Action: Stop test, investigate, fix, retest
```

---

### Phase 4: Spike Testing

```
Sudden traffic spike (10x normal peak):

T=0min: 1000 users
T=1min: 10,000 users (spike!)
T=2min: 1000 users

Measure:
  Can system handle spike without crashing?
  Does response time spike severely?
  Are there cascading failures?
  
Expected results:
  Response time might increase 2-3x
  Some errors acceptable (503 overload)
  System should recover once spike ends
```

---

## Load Testing Tools

### Apache JMeter

```
Declarative test plan:
  - Thread pool: 1000 users
  - Ramp-up: 5 minutes (gradual increase)
  - Duration: 30 minutes
  - Request: GET /api/users?limit=50

Results:
  Response time distribution (p50, p95, p99)
  Throughput graph
  Error rate graph
  CPU/memory usage
```

---

### Gatling

```
Code-based test (Scala):

scenario("normal user workflow")
  .exec(http("homepage").get("/"))
  .pause(5)
  .exec(http("login").post("/login").formParam("user", "test"))
  .pause(10)
  .exec(http("feed").get("/feed"))

setUp(
  scenario.inject(
    rampUsers(1000) over 5 minutes
  ).protocols(httpConfig)
)
```

**Advantages**:
- Code-based (version control, CI/CD integration)
- Better performance (JVM-based)
- Simulation realism (think-time between requests)

---

### k6

```
JavaScript-based:

import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 1000,
  duration: '30m',
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.1'],
  },
};

export default function () {
  let response = http.get('http://api.example.com/users/123');
  check(response, {
    'is status 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

---

## Bottleneck Identification via Load Testing

### Scenario: Increasing Response Time

```
Load test results:

100 users: 50ms response time, 20% CPU
500 users: 150ms response time, 60% CPU
1000 users: 500ms response time, 95% CPU

Analysis:
  CPU is the bottleneck (hitting 95%)
  
Potential causes:
  1. Application is CPU-intensive (algorithmic)
  2. Garbage collection pauses
  3. Context switching (too many threads)
  4. Database queries not optimized

Next step:
  Profile application (CPU profiler)
  Identify hot functions
  Optimize (caching, better algorithms, etc.)
```

---

### Scenario: Constant Response Time, Increasing Errors

```
Load test results:

100 users: 50ms response time, 0% error rate
500 users: 52ms response time, 0.1% error rate
1000 users: 51ms response time, 5% error rate
2000 users: 53ms response time, 20% error rate

Analysis:
  Response time stays constant (not network/compute bottleneck)
  Error rate increases (likely connection pool or resource limit)
  
Probable cause:
  Database connection pool exhausted
  OR: Message queue backing up
  OR: Memory limit reached
  
Investigation:
  Check database logs for connection pool errors
  Check message queue depth
  Monitor memory usage during test
```

---

### Scenario: Memory Leak

```
Load test: 1000 users for 30 minutes

Memory usage over time:
  T=0min: 2GB
  T=10min: 2.5GB
  T=20min: 3.2GB
  T=30min: 4.1GB
  
Analysis:
  Memory growing ~70MB per minute
  Projected: 50GB after 12 hours

Cause: Memory leak (objects not being garbage collected)

Investigation:
  Heap dump analysis
  Find objects that should be freed but aren't
  Common culprits:
    - Static collections (never cleared)
    - Event listeners not unregistered
    - Caching without eviction
```

---

## Load Testing Best Practices

### 1. Test Representative Workload

```
Don't:
  All GET requests (unrealistic)
  All simple queries (doesn't stress database)

Do:
  70% reads, 30% writes (realistic for this system)
  Mix of simple and complex queries
  Include cache misses (don't warm cache 100%)
  
Example:
  1. GET /users/:id (fast, cached)
  2. POST /transfer (slow, write-heavy)
  3. GET /analytics (medium, complex query)
  4. GET /recommendations (slow, ML inference)
  
Proportion: 40% GET users, 10% transfers, 30% analytics, 20% recommendations
```

---

### 2. Run Tests Multiple Times

```
First run: Baseline
Second run: Confirm results (eliminate noise)
Third run: With monitoring enabled (might be slower)

Verify:
  Consistent results (if vary wildly, test environment is flaky)
  Identify trends (are we degrading over time?)
```

---

### 3. Test in Staging

```
Never test in production (DDoS risk)

Staging should be:
  ✓ Same hardware as production (or smaller, scaled proportionally)
  ✓ Same database (production replica or anonymized copy)
  ✓ Same configuration (same timeouts, pool sizes, etc.)
  ✗ Not: Tiny VM that doesn't reflect production

If staging is 10x smaller:
  Scale results by 10x when extrapolating to production
```

---

## Common Load Testing Mistakes

### Mistake 1: Testing Single Endpoint

```
❌ Wrong:
  Load test only: GET /users/:id
  Seems to handle 10k QPS easily
  
✓ Correct:
  Load test full workflow (login → browse → search → checkout)
  Realistic traffic pattern
  Catches interaction bottlenecks
```

---

### Mistake 2: Ignoring Connection Warmup

```
❌ Wrong:
  Start test with 10k concurrent users immediately
  First 30 seconds: Many connection setup failures
  Results artificially low
  
✓ Correct:
  Ramp up gradually (1000 users over 5 minutes)
  Connections established
  Test steady-state behavior
```

---

### Mistake 3: Not Monitoring

```
❌ Wrong:
  Just run load test, look at final results
  Miss when breakdown occurred
  Don't know what caused it
  
✓ Correct:
  Monitor during test:
    - Response time graph (when did it increase?)
    - Error rate graph (when did it spike?)
    - CPU/memory graphs (resources exhausted when?)
    - Database metrics (queries/sec, pool utilization)
  Correlate: "At 500 users, DB pool hit 100% utilization and errors spiked"
```

---

## Load Test Checklist

- [ ] Define realistic workload (% of each operation type)
- [ ] Identify success criteria (p99 < 500ms, error < 0.1%)
- [ ] Test in staging environment
- [ ] Ramp up gradually (test find knee point)
- [ ] Sustained load for 30+ minutes
- [ ] Spike test (10x traffic)
- [ ] Monitor system during test (CPU, memory, disk, network)
- [ ] Check for memory leaks
- [ ] Identify bottleneck (CPU, database, network, memory)
- [ ] Document results and improvement plan
- [ ] Retest after optimizations

---

## Related Fundamentals

- [Capacity Planning/Scale Estimation](scale-estimation-framework.md) – Pre-testing estimation
- [Monitoring & Alerting](../reliability-and-resiliency/monitoring-and-alerting.md) – Metrics to track
- [Reliability](../reliability-and-resiliency/) – Circuit breaker under load

---

**Status**: ✅ Complete. Covers phases, tools, bottleneck identification, best practices.

