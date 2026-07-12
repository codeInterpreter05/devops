# Day 72 — Lab: Load & Chaos Testing

**Goal:** Write and run a k6 load test, run a LitmusChaos pod-delete experiment against a load-bearing Deployment, and verify your HPA responds correctly under the combined pressure — the day's assigned hands-on activity, end to end.

**Prerequisites:**
- A local Kubernetes cluster (`kind` or `minikube`) with `metrics-server` installed (required for HPA to function).
- `k6` installed (`brew install k6` or download binary).
- `kubectl`, `helm` installed.
- A simple HTTP service you can deploy with a Deployment + Service + HPA (a basic Nginx or a small Flask "hello world" app works fine).

---

### Lab 1 — Deploy a target app with an HPA

1. Deploy a small HTTP app with 3 replicas, resource requests set (required for HPA to compute utilization), and an HPA targeting 50% CPU:
   ```bash
   kubectl create deployment myapp --image=hashicorp/http-echo -- -text="hello" -listen=:8080
   kubectl set resources deployment myapp --requests=cpu=50m,memory=32Mi
   kubectl scale deployment myapp --replicas=3
   kubectl expose deployment myapp --port=80 --target-port=8080
   kubectl autoscale deployment myapp --cpu-percent=50 --min=3 --max=10
   ```
2. Port-forward and confirm the app responds:
   ```bash
   kubectl port-forward svc/myapp 8080:80 &
   curl localhost:8080
   ```

**Success criteria:** `kubectl get hpa` shows a working HPA reading real CPU metrics (not `<unknown>`), confirming `metrics-server` is functioning.

---

### Lab 2 — Write and run a k6 load test

1. Write `load-test.js` (as in README file 1) targeting `http://localhost:8080` (via the port-forward), with a ramping stage up to 50 VUs and thresholds on p95 latency and error rate.
2. Run it and observe HPA scaling in real time in a second terminal:
   ```bash
   k6 run load-test.js &
   kubectl get hpa myapp --watch
   ```
3. Confirm k6 exits non-zero if you deliberately set an unrealistically strict threshold (e.g., `p(95)<5`) and re-run — proving the thresholds mechanism works as a real CI gate.

**Success criteria:** You observe `kubectl get hpa` showing replica count increase in response to the load test's CPU pressure, and can demonstrate k6 failing (non-zero exit) when a threshold is deliberately violated.

---

### Lab 3 — The core hands-on activity: LitmusChaos pod-delete under load

1. Install LitmusChaos:
   ```bash
   kubectl apply -f https://litmuschaos.github.io/litmus/litmus-operator-v-latest.yaml
   kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/experiments.yaml
   ```
2. Create a `ChaosServiceAccount` with permissions scoped to the `myapp` deployment (RBAC granting pod list/delete on that specific label selector).
3. Apply a `ChaosEngine` targeting `myapp`, `PODS_AFFECTED_PERC: '33'` (roughly 1 of 3 replicas), `TOTAL_CHAOS_DURATION: '30'`.
4. **Run the k6 load test from Lab 2 and the chaos experiment simultaneously** (in two terminals), then watch:
   ```bash
   kubectl get pods -l app=myapp --watch
   kubectl get chaosresult -n default
   ```
5. Confirm from the k6 output that the error rate/latency thresholds still passed *despite* a pod being killed mid-test — this is the actual proof that your replica count + Service load balancing absorbed the fault without user-visible impact.

**Success criteria:** The `ChaosResult` shows `Verdict: Pass` (or you can articulate why it failed if it didn't — e.g., insufficient replicas), and the concurrent k6 run's thresholds still passed, demonstrating the deployment tolerated a pod loss under real load.

---

### Lab 4 — Locust for a stateful scenario (stretch of the core activity)

1. Write a `locustfile.py` with an `on_start` step and two weighted `@task` methods hitting your target app differently (e.g., GET vs. a simulated POST).
2. Run headless with a moderate user count and confirm the HTML report is generated:
   ```bash
   locust -f locustfile.py --host=http://localhost:8080 --users 50 --spawn-rate 5 --run-time 2m --headless --html report.html
   ```

**Success criteria:** A generated `report.html` shows request statistics broken down per task, demonstrating Locust's weighted multi-task scenario modeling versus k6's more linear script.

---

### Cleanup

```bash
kubectl delete chaosengine pod-delete-chaos
kubectl delete hpa myapp
kubectl delete deployment myapp
kubectl delete svc myapp
kill %1   # stop the port-forward background job if still running
kind delete cluster --name <your-cluster-name>   # if created solely for this lab
```

### Stretch challenge

Reduce `myapp`'s replica count to 1 (no redundancy) and re-run the combined Lab 3 experiment. Confirm the k6 thresholds now fail during the pod-delete window, and write one paragraph explaining exactly why — tying the observed failure directly back to the replica count, to make the "resilience is proven, not assumed" point concrete for yourself.
