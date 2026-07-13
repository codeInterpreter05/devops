# Day 115 — Lab: Kubernetes Cost Optimisation

**Goal:** Install Kubecost (or OpenCost), find the top 3 over-provisioned deployments in a cluster, right-size them, and measure the resulting savings — the assigned hands-on activity for today, broken into concrete steps.

**Prerequisites:** A running Kubernetes cluster with Prometheus already installed (kind/minikube is fine for the mechanics, though savings numbers will be more meaningful on a cluster with real, varied workloads — a shared staging EKS/GKE cluster is ideal if available). `kubectl` and `helm` configured against it. Metrics Server installed (`kubectl top nodes` should work before you start).

---

### Lab 1 — Install OpenCost (free, fast path)

1. Confirm you have Prometheus reachable in-cluster (install the kube-prometheus-stack if not: `helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace`).
2. Install OpenCost:
   ```bash
   helm repo add opencost https://opencost.github.io/opencost-helm-chart
   helm repo update
   helm install opencost opencost/opencost -n opencost --create-namespace \
     --set opencost.prometheus.external.enabled=true \
     --set opencost.prometheus.external.url=http://prometheus-kube-prometheus-prometheus.monitoring.svc:9090
   ```
3. Port-forward the UI and API:
   ```bash
   kubectl port-forward -n opencost svc/opencost 9090:9090 9003:9003
   ```
4. Open `http://localhost:9090` and confirm you can see cost allocation broken down by namespace.

**Success criteria:** OpenCost's UI shows a non-zero, per-namespace cost breakdown for your cluster.

---

### Lab 2 — Pull a namespace/deployment efficiency report

1. Query the allocation API directly, grouped by namespace and controller (deployment):
   ```bash
   curl "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace,controller" | jq '.data[] | {name: .[0].name, cpuEfficiency: .[0].cpuEfficiency, ramEfficiency: .[0].ramEfficiency, totalCost: .[0].totalCost}'
   ```
2. Sort the results by `totalCost` descending, then by `cpuEfficiency`/`ramEfficiency` ascending, to find deployments that are both expensive AND wasteful.
3. Identify your top 3 candidates for right-sizing (highest cost, lowest efficiency).

**Success criteria:** You have a ranked list of 3 real deployments with their current cost, CPU efficiency %, and memory efficiency %.

---

### Lab 3 — The core hands-on activity: right-size the top 3 offenders

1. For each of the 3 deployments identified in Lab 2, deploy a VPA in recommendation-only mode:
   ```yaml
   apiVersion: autoscaling.k8s.io/v1
   kind: VerticalPodAutoscaler
   metadata:
     name: <deployment-name>-vpa
     namespace: <namespace>
   spec:
     targetRef:
       apiVersion: "apps/v1"
       kind: Deployment
       name: <deployment-name>
     updatePolicy:
       updateMode: "Off"
   ```
   ```bash
   kubectl apply -f vpa-<deployment-name>.yaml
   # Wait several minutes to hours for VPA to gather usage data, then:
   kubectl describe vpa <deployment-name>-vpa -n <namespace>
   ```
2. Record the current `resources.requests` (`kubectl get deploy <name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].resources}'`) next to VPA's `target` recommendation for each of the 3 deployments.
3. Manually patch one of the three deployments to the recommended value (start with the least risky one — ideally one with multiple replicas and a PodDisruptionBudget already in place):
   ```bash
   kubectl set resources deployment/<name> -n <namespace> \
     --requests=cpu=250m,memory=256Mi
   ```
4. Wait 24-48 hours, then re-query the allocation API from Lab 2 for that deployment and compare `totalCost` and efficiency before/after.

**Success criteria:** You have a before/after cost and efficiency comparison for at least one right-sized deployment, with a real percentage or dollar reduction.

---

### Lab 4 — Measure cluster-wide bin-packing efficiency

1. Run `kubectl top nodes` and, separately, `kubectl describe nodes | grep -A 5 "Allocated resources"` for each node — compare **allocated** (requested) vs. **actual usage** per node.
2. Query OpenCost's cluster-level efficiency: `curl "http://localhost:9003/allocation/compute?window=7d&aggregate=cluster" | jq`.
3. Identify whether the cluster's overall low efficiency (if any) is dominated by a few very wasteful deployments (from Lab 2) or spread evenly — this determines whether right-sizing individual workloads or changing node group shapes is the higher-leverage fix.

**Success criteria:** You can state, with a number, whether this cluster's waste is concentrated or diffuse, and which lever (right-sizing vs. bin packing/node shape) would move the needle more.

---

### Cleanup

```bash
kubectl delete vpa --all -n <namespace>          # remove lab VPAs
helm uninstall opencost -n opencost
kubectl delete namespace opencost
# Revert any resource patches applied purely for lab purposes if this was a shared cluster,
# unless the right-sizing was a genuine improvement worth keeping.
```

### Stretch challenge

Set an OpenCost/Kubecost budget alert (or, if using plain OpenCost, write a small script using the allocation API + a cron job) that fires when any single namespace's efficiency drops below 30% over a rolling 7-day window — the automated version of the manual check you just did by hand.
