# Day 72 — Cheatsheet: Load & Chaos Testing

## k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },
    { duration: '3m', target: 50 },
    { duration: '1m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://myapp.com/api');
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```

```bash
k6 run load-test.js
k6 run --vus 50 --duration 30s load-test.js   # override stages inline
k6 run --out json=results.json load-test.js    # export raw metrics
```

## Locust

```python
from locust import HttpUser, task, between

class MyUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        self.token = self.client.post("/login", json={...}).json()["token"]

    @task(3)
    def read(self):
        self.client.get("/api/products")

    @task(1)
    def write(self):
        self.client.post("/api/checkout", json={...})
```

```bash
locust -f locustfile.py --host=https://myapp.com \
  --users 200 --spawn-rate 10 --run-time 5m --headless --html report.html
```

## LitmusChaos

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata: { name: pod-delete-chaos, namespace: default }
spec:
  appinfo: { appns: default, applabel: 'app=myapp', appkind: deployment }
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - { name: TOTAL_CHAOS_DURATION, value: '30' }
            - { name: CHAOS_INTERVAL, value: '10' }
            - { name: FORCE, value: 'false' }
            - { name: PODS_AFFECTED_PERC, value: '25' }
```

```bash
kubectl apply -f pod-delete-chaosengine.yaml
kubectl get chaosresult -n default
kubectl describe chaosresult <name>
```

## AWS Fault Injection Simulator

```bash
aws fis create-experiment-template --cli-input-json file://template.json
aws fis start-experiment --experiment-template-id EXT123456789
aws fis get-experiment --id EXP123456789
aws fis stop-experiment --id EXP123456789    # manual abort
```

```json
{
  "targets": { "web-tier": { "resourceType": "aws:ec2:instance", "selectionMode": "PERCENT(25)" } },
  "actions": { "stop": { "actionId": "aws:ec2:stop-instances", "targets": { "Instances": "web-tier" } } },
  "stopConditions": [{ "source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:...:alarm:high-error-rate" }]
}
```

## HPA quick reference (for validating chaos/load recovery)

```bash
kubectl autoscale deployment myapp --cpu-percent=50 --min=3 --max=10
kubectl get hpa myapp --watch
kubectl top pods -l app=myapp
```

## Chaos engineering principles (memorize the order)

```
1. Define steady state (measurable baseline)
2. Hypothesize steady state holds under the fault
3. Inject realistic, real-world failure
4. Run in production when possible (for representative confidence)
5. Minimize blast radius (small %, automated abort, expand gradually)
6. Automate to run continuously
```
