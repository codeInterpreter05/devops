# Day 65 — Lab: Deployment Strategies

**Goal:** Implement a real canary deployment with Argo Rollouts (10% -> 25% -> 50% -> 100%) with automatic rollback on errors, and directly compare it against Rolling Update and Recreate behavior.

**Prerequisites:**
- A local Kubernetes cluster (`kind`/`minikube`) with the Argo Rollouts controller installed:
  ```bash
  kubectl create namespace argo-rollouts
  kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
  ```
- The `kubectl argo rollouts` plugin: `brew install argoproj/tap/kubectl-argo-rollouts`.
- NGINX Ingress or a simple mock traffic router — for this lab we'll use Argo Rollouts' built-in **canary without a service mesh** (weighted ReplicaSets only, via basic Service selection) to keep the prerequisites light; a stretch challenge covers real Istio traffic splitting.
- Prometheus running in-cluster if you want live analysis (`kube-prometheus-stack` via Helm is the fastest path) — optional; Lab 3 includes a fallback using a simple job-based check if you skip Prometheus.

---

### Lab 1 — Rolling Update vs Recreate, side by side

1. Deploy a simple app with Rolling Update (the default):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata: { name: demo-rolling }
   spec:
     replicas: 6
     strategy:
       type: RollingUpdate
       rollingUpdate: { maxSurge: 1, maxUnavailable: 1 }
     selector: { matchLabels: { app: demo-rolling } }
     template:
       metadata: { labels: { app: demo-rolling } }
       spec:
         containers:
           - name: app
             image: nginxdemos/hello:plain-text
             readinessProbe:
               httpGet: { path: /, port: 80 }
               periodSeconds: 2
   ```
2. Apply it, then update the image tag and watch the rollout:
   ```bash
   kubectl apply -f demo-rolling.yaml
   kubectl set image deployment/demo-rolling app=nginxdemos/hello:0.3 && kubectl rollout status deployment/demo-rolling -w
   kubectl get pods -l app=demo-rolling -w   # watch old/new pods coexist
   ```
3. Duplicate the manifest as `demo-recreate.yaml` with `strategy.type: Recreate`, apply, update the image the same way, and watch `kubectl get pods -w` — note the gap where **zero** pods are Ready.

**Success criteria:** You've directly observed old/new pod coexistence during Rolling Update, and a full pod-count-zero gap during Recreate.

---

### Lab 2 — The core hands-on activity: canary with Argo Rollouts

This is today's assigned hands-on activity.

1. Convert the Deployment into a `Rollout`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Rollout
   metadata: { name: demo-canary }
   spec:
     replicas: 10
     strategy:
       canary:
         steps:
           - setWeight: 10
           - pause: { duration: 30s }
           - setWeight: 25
           - pause: { duration: 30s }
           - setWeight: 50
           - pause: { duration: 30s }
           - setWeight: 100
     selector: { matchLabels: { app: demo-canary } }
     template:
       metadata: { labels: { app: demo-canary } }
       spec:
         containers:
           - name: app
             image: nginxdemos/hello:plain-text
   ---
   apiVersion: v1
   kind: Service
   metadata: { name: demo-canary-svc }
   spec:
     selector: { app: demo-canary }
     ports: [{ port: 80 }]
   ```
2. Apply it, then trigger a rollout by changing the image, watching progress with the plugin:
   ```bash
   kubectl apply -f rollout.yaml
   kubectl argo rollouts set image demo-canary app=nginxdemos/hello:0.3
   kubectl argo rollouts get rollout demo-canary --watch
   ```
3. Watch it step through 10% -> 25% -> 50% -> 100%, pausing 30s at each step, and observe the ReplicaSet pod counts changing at each weight (e.g., 10% of 10 replicas = 1 canary pod).
4. Use the dashboard for a visual view: `kubectl argo rollouts dashboard` then open the printed URL.

**Success criteria:** A completed canary rollout you watched step through all four weights, with the plugin output as evidence.

---

### Lab 3 — Automated rollback on failed analysis

1. Add an `AnalysisTemplate` that checks a trivial condition (simulate a "bad metric" without needing full Prometheus by checking an HTTP endpoint's status code via a Job-based analysis, or install `kube-prometheus-stack` if you have time):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: AnalysisTemplate
   metadata: { name: http-check }
   spec:
     metrics:
       - name: web-check
         provider:
           web:
             url: http://demo-canary-svc.default.svc.cluster.local
             jsonPath: "{$.status}"
         successCondition: "result == 200"
         failureLimit: 1
   ```
2. Reference it in the rollout's canary steps: add `- analysis: { templates: [{ templateName: http-check }] }` after the first `setWeight: 10` step.
3. Deliberately break the app (point the image at a tag that 404s or doesn't exist) and trigger a rollout — watch Argo Rollouts detect the failed analysis and automatically abort, scaling the canary back to 0 and restoring 100% traffic to the previous stable version:
   ```bash
   kubectl argo rollouts set image demo-canary app=nginxdemos/hello:does-not-exist
   kubectl argo rollouts get rollout demo-canary --watch
   ```

**Success criteria:** You've triggered and observed an automatic rollback — no manual `rollback` command needed — and can point to the exact analysis step that caused it.

---

### Cleanup

```bash
kubectl delete -f demo-rolling.yaml -f demo-recreate.yaml -f rollout.yaml
kubectl delete analysistemplate http-check
kubectl delete namespace argo-rollouts
```

### Stretch challenge

Install Istio (or use `kubectl argo rollouts` with an NGINX Ingress canary annotation setup) so `setWeight` actually controls a real service-mesh-level traffic percentage rather than approximate ReplicaSet pod-count ratios, and confirm with `curl` in a loop that roughly the configured percentage of requests hit the new version at the 25% step.
