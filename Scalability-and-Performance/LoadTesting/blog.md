# Load Testing Strategies

Load testing verifies that a system can handle expected (and unexpected) traffic loads. It's the empirical validation of capacity planning, performance optimization, and HA design—without it, you don't know if your system works under load until users tell you (painfully). Effective load testing answers: what's the system's breaking point? Does it degrade gracefully? Does auto-scaling work? Do alerts fire?

## Test Types

| Test | Goal | Example |
|------|------|---------|
| Load test | Verify system handles expected load | 1,000 RPS for 1 hour |
| Stress test | Find the breaking point | Ramp up until failures appear |
| Soak test | Find memory leaks, resource exhaustion | Moderate load for 24 hours |
| Spike test | Test sudden traffic increase | 100 → 10,000 RPS instantly |
| Breakpoint test | Find max capacity | Step up RPS until p99 latency exceeds SLO |

## Tooling: k6

```javascript
// k6 load test script
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
    scenarios: {
        // Ramp up, sustain, ramp down
        load_test: {
            executor: 'ramping-vus',
            startVUs: 0,
            stages: [
                { duration: '5m', target: 100 },   // Ramp to 100 users
                { duration: '30m', target: 1000 },  // Ramp to 1000 users
                { duration: '1h', target: 1000 },   // Stay at peak
                { duration: '5m', target: 0 },      // Ramp down
            ],
        },
    },
    thresholds: {
        'http_req_duration': ['p(95)<500'],        // 95% requests < 500ms
        'http_req_failed': ['rate<0.01'],           // Error rate < 1%
        'http_reqs': ['rate>500'],                   // Throughput > 500 RPS
    },
};

export default function () {
    // Realistic user behavior: browse orders
    const res = http.get('https://api.example.com/orders', {
        headers: { 'Authorization': `Bearer ${getRandomToken()}` }
    });
    check(res, { 'status is 200': (r) => r.status === 200 });
    sleep(Math.random() * 3 + 1);  // Think time: 1-4 seconds
}
```

## Load Testing in CI/CD

```yaml
# GitHub Actions: load test as deployment gate
- name: Deploy to staging
  run: helm upgrade order-service ./chart --set environment=staging

- name: Run load test
  run: |
    k6 run load-test.js \
      --out json=results.json \
      --summary-export summary.json

- name: Validate thresholds
  run: |
    FAILED=$(jq '.metrics.http_req_failed.values.rate' summary.json)
    if (( $(echo "$FAILED > 0.01" | bc) )); then
      echo "Error rate exceeded threshold: $FAILED"
      exit 1
    fi

- name: Deploy to production
  if: success()
  run: helm upgrade order-service ./chart --set environment=production
```

## Interpreting Results

```
Typical load test findings:
  - Database connection pool exhausted → need connection pooling
  - CPU spikes at 500 RPS → need more replicas or profiling
  - Memory grows linearly → memory leak detected
  - p99 latency spikes at 800 RPS → GC pauses or lock contention
  - 503 errors at 1000 RPS → ELB/ALB throttle, need higher quota

Key metrics during load test:
  - Application: RPS, error rate, p50/p95/p99 latency
  - Database: Connections, CPU, query duration, locks
  - Infrastructure: CPU, memory, network throughput, disk I/O
  - Auto-scaling: Did it trigger? How fast? Did it over-scale?
```

## Realistic Test Data

```python
# Test data must be realistic
# Don't: all users hitting the same endpoint with the same data
# Do: varied endpoints, realistic payload sizes, session behavior

# Important: randomize user IDs, order IDs, search queries
# Pre-create test data (don't use production data!)
# Consider data skew: some customers have 10,000 orders; most have 10

# Test with realistic payload sizes:
# - Average order: 5 items, 2KB JSON
# - Image upload: 2-5MB files
# - Search query: 50-character strings
```

## Summary

| Test Type | Duration | Load Profile | What It Finds |
|-----------|----------|-------------|---------------|
| Load | 1-4 hours | Sustained expected load | Baseline performance, resource usage |
| Stress | Until break | Ramp to failure | Max capacity, failure mode |
| Soak | 12-48 hours | 80% of expected | Memory leaks, gradual degradation |
| Spike | Minutes | Instant 10x | Auto-scaling speed, surge handling |

Load testing turns assumptions into data. Without it, "the system can handle 10,000 RPS" is a guess. The architect must ensure load testing is part of the deployment pipeline, not a pre-launch panic exercise.
