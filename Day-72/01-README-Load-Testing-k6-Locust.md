# Day 72 — Load & Chaos Testing: Load Testing with k6 and Locust

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing | **Flag:** ⚡ Interview-critical

## Brief

Every other test type in this course tells you whether the system behaves *correctly*. Load testing tells you whether it behaves correctly **under realistic (or extreme) traffic** — a system that passes every unit, integration, and e2e test can still fall over the moment real concurrent user volume hits it, because concurrency bugs, connection-pool exhaustion, and resource contention only appear under load. Load testing before you rely on autoscaling/HPA to save you in production is what separates "we hope it scales" from "we've proven it scales, and we know exactly where it breaks."

This day is split into three focused files:

1. **This file** — k6 and Locust for load testing.
2. **[02-README-Chaos-Engineering-Principles.md](02-README-Chaos-Engineering-Principles.md)** — chaos engineering principles and LitmusChaos for Kubernetes.
3. **[03-README-AWS-FIS-Game-Days.md](03-README-AWS-FIS-Game-Days.md)** — AWS Fault Injection Simulator, game days, and fire drills.

## k6 — code-as-load-test, built for CI

k6 (by Grafana Labs) writes load test scenarios in JavaScript, executed by a Go-based engine — you get the readability of JS without JS's runtime overhead limiting how much concurrent virtual-user load a single machine can generate. It's designed from the ground up to be CI-friendly: single binary, scriptable thresholds that fail the run (and thus the pipeline) automatically.

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },   // ramp up to 50 virtual users over 1 minute
    { duration: '3m', target: 50 },   // hold steady at 50 VUs for 3 minutes
    { duration: '1m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // 95th percentile response time must stay under 500ms
    http_req_failed: ['rate<0.01'],     // error rate must stay under 1%
  },
};

export default function () {
  const res = http.get('https://staging.myapp.com/api/v1/products');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response has products': (r) => JSON.parse(r.body).length > 0,
  });
  sleep(1);
}
```

```bash
k6 run load-test.js
```

**Why `thresholds` is the single most important feature for CI/CD use**: without it, k6 just prints statistics for a human to eyeball. With `thresholds`, k6 itself exits non-zero if p95 latency or error rate crosses your defined line — turning a load test into an automated CI gate exactly like `trivy --exit-code 1` does for vulnerability scanning. This is the direct throughline from Day 67's "make security checks a real gate, not just a report" idea, applied to performance.

```yaml
# GitHub Actions: run k6 against staging after every deploy
- name: Load test
  uses: grafana/k6-action@v0.3.1
  with:
    filename: load-test.js
  env:
    K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}   # optional, for k6 Cloud result storage/visualization
```

## Locust — Python-based, better for complex, stateful user journeys

Locust defines load scenarios as Python classes describing "tasks" a simulated user performs, with weighted probabilities — better suited than k6 when your test needs genuinely complex Python logic (stateful multi-step user flows, complex data generation with existing Python libraries) rather than a straightforward sequence of HTTP calls.

```python
# locustfile.py
from locust import HttpUser, task, between

class ShopperUser(HttpUser):
    wait_time = between(1, 3)   # simulated think-time between tasks

    def on_start(self):
        # runs once per simulated user at the start — e.g., log in and store a session token
        response = self.client.post("/login", json={"user": "test", "pass": "test"})
        self.token = response.json()["token"]

    @task(3)   # weight: this task runs 3x more often than weight-1 tasks
    def browse_products(self):
        self.client.get("/api/v1/products", headers={"Authorization": f"Bearer {self.token}"})

    @task(1)
    def checkout(self):
        self.client.post("/api/v1/checkout", json={"items": [1, 2]},
                          headers={"Authorization": f"Bearer {self.token}"})
```

```bash
locust -f locustfile.py --host=https://staging.myapp.com \
  --users 200 --spawn-rate 10 --run-time 5m --headless \
  --html report.html
```

`--headless` runs Locust without its web UI, which is what you want in CI — the interactive UI is great for exploratory local testing but has no place in an automated pipeline.

## k6 vs. Locust — when to reach for which

| | k6 | Locust |
|---|---|---|
| Language | JavaScript | Python |
| Performance overhead per VU | Lower (Go engine) — more virtual users per test machine | Higher (real Python interpreter per simulated user) |
| Best fit | Straightforward HTTP/API load tests, tight CI integration, built-in pass/fail thresholds | Complex, stateful user journeys benefiting from Python's ecosystem (data generation, existing Python client libraries) |
| Distributed load generation | k6 Cloud (hosted) or manual sharding | Native distributed mode (master/worker nodes) built in |

Neither is objectively "better" — the real decision driver is usually **which language your team already writes tooling in**, and whether your test scenarios are simple enough that k6's lighter-weight VUs are worth it, or complex/stateful enough that Locust's full Python runtime per simulated user earns its overhead.

## Points to Remember

- Load testing answers "does this behave correctly *under realistic/extreme concurrency*" — a question unit/integration/e2e tests structurally cannot answer, since they don't simulate concurrent load.
- k6's `thresholds` block (`p(95)<500`, `rate<0.01`) is what turns a load test into an automated CI gate with a real pass/fail exit code — without it, it's just a report a human has to read and judge.
- k6 (JS, Go engine, lightweight VUs) suits straightforward API load tests and tight CI integration; Locust (Python, heavier per-VU cost, native distributed mode) suits complex, stateful multi-step user journeys.
- `--headless` mode is what makes Locust usable in CI — its interactive web UI is for exploratory local runs only.
- Load testing should target a real, dedicated environment (staging, or a shadow/canary slice of production) sized similarly to production — testing against an undersized environment tells you nothing meaningful about production capacity.

## Common Mistakes

- Running a load test without any `thresholds`/pass-fail criteria, so results just get eyeballed (or ignored) by whoever happened to notice the CI job ran — never actually gating anything automatically.
- Load testing against an environment much smaller/cheaper than production (a tiny "staging" tier) and extrapolating those numbers linearly to production capacity — resource contention, connection pool limits, and database performance rarely scale linearly, so this produces a false sense of confidence.
- Load testing only the "happy path" (successful requests) and never including realistic failure/retry/error scenarios in the simulated traffic mix — production traffic always includes some rate of retries, malformed requests, and slow clients that a clean synthetic load test omits.
- Running load tests directly against production with no coordination, no ramp-up, and no kill switch — a poorly-designed load test can itself cause the very outage it was meant to help prevent.
- Choosing k6 or Locust based on hype rather than actual scenario complexity/team language fit — picking Locust for a trivial "hit 3 endpoints repeatedly" test adds unnecessary per-VU overhead versus k6, while forcing a genuinely complex, stateful, Python-library-dependent scenario into k6's JS-only model creates awkward workarounds.
