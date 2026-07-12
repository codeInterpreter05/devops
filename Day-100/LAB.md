# Day 100 — Lab: Datadog / New Relic (Commercial Tools)

**Goal:** Actually connect a commercial observability platform to a real Kubernetes cluster, generate traces/logs/metrics, and compare what you see against your existing OTel/Prometheus setup from earlier days — not just read the marketing pages.

**Prerequisites:** A Datadog trial account (datadoghq.com/free-datadog-trial), a New Relic free-tier account (newrelic.com/signup), a Kubernetes cluster (kind/minikube/EKS from earlier labs) with `kubectl` access, and Helm installed.

---

### Lab 1 — Deploy the Datadog Agent to your cluster

1. Add the Datadog Helm repo and get your API key from the Datadog trial account (Organization Settings → API Keys):
   ```bash
   helm repo add datadog https://helm.datadoghq.com
   helm repo update
   ```
2. Install the Agent with the Cluster Agent enabled:
   ```bash
   helm install datadog-agent datadog/datadog \
     --set datadog.apiKey=<YOUR_API_KEY> \
     --set datadog.site='datadoghq.com' \
     --set clusterAgent.enabled=true \
     --set datadog.logs.enabled=true \
     --set datadog.apm.portEnabled=true
   ```
3. Confirm the DaemonSet and Cluster Agent Deployment are both running:
   ```bash
   kubectl get daemonset,deployment -l app.kubernetes.io/name=datadog
   ```
4. In the Datadog console, go to **Infrastructure → Host Map** and confirm your cluster's nodes appear within a couple of minutes.

**Success criteria:** You can see your cluster's nodes and pods populated in Datadog's Infrastructure view, and you can explain the difference between what the DaemonSet Agent pods and the Cluster Agent pod are each responsible for.

---

### Lab 2 — Instrument a workload for APM and read a flame graph

1. Deploy a small sample app with `dd-trace` instrumentation (or use Datadog's official demo app if you don't have one handy: `kubectl apply -f https://raw.githubusercontent.com/DataDog/apm-workshop-python/main/k8s/deployment.yaml` or equivalent for your stack).
2. Add the required Agent-related env vars to the workload's pod spec (`DD_AGENT_HOST` from the Downward API, `DD_TRACE_ENABLED=true`).
3. Generate traffic against the app (`kubectl port-forward` + `curl` in a loop, or a simple load generator).
4. In Datadog, go to **APM → Traces**, open a slow trace, and identify the widest bar in the flame graph. Note which span it is and how much of the total request time it accounts for.
5. Open **APM → Service Map** and confirm your service and its dependencies appear as nodes.

**Success criteria:** You can point to a specific span in a real flame graph and state, in your own words, why it's the bottleneck — not a hypothetical, an actual trace from your own traffic.

---

### Lab 3 — Set up a monitor and trigger it

1. Create a metric monitor in Datadog on CPU usage for your workload's pods, threshold e.g. `avg(last_5m):avg:kubernetes.cpu.usage.total{pod_name:<your-pod>} > <a value slightly above idle>`.
2. Generate CPU load against the pod (`kubectl exec` into it and run a CPU-burning loop, or scale up traffic) until the monitor fires.
3. Confirm you receive the notification (configure a notification channel — email is the fastest route for a lab).
4. Convert the monitor to a composite monitor by adding a second condition (e.g., also require elevated request latency) and explain why this reduces false-positive pages compared to the single-condition version.

**Success criteria:** You've triggered a real alert end-to-end (metric crosses threshold → notification received), and you can explain the false-positive-reduction rationale for composite monitors in your own words.

---

### Lab 4 — Compare Datadog APM traces to your OTel/ADOT setup (the assigned hands-on activity)

1. If you have an OTel-instrumented service from Day 99, point its OTLP exporter at the Datadog Agent instead of (or alongside) the ADOT Collector — the Agent accepts OTLP on the same standard ports.
2. Generate the same request pattern against both setups.
3. Compare: trace completeness, how fast you can locate the bottleneck span in each UI, and how much configuration each required.
4. Write down (in your lab notes, not for submission) one concrete thing Datadog's APM UI did better and one thing your OTel-based setup did better (e.g., portability vs. polish).

**Success criteria:** You have a first-hand, evidence-based comparison — not a guess — between an OTel-based tracing setup and Datadog APM using the *same* underlying traffic.

---

### Lab 5 — Sign up for New Relic and do the equivalent quick comparison

1. Sign up for the New Relic free tier and install the Kubernetes integration (`newrelic install -n kubernetes-open-source-integration` via New Relic's guided CLI installer, or the Helm chart if you prefer explicit control).
2. Confirm cluster data appears in the New Relic **Kubernetes cluster explorer**.
3. Run one NRQL query against your data, e.g.:
   ```sql
   SELECT average(cpuUsedCores) FROM K8sContainerSample WHERE clusterName = 'your-cluster' TIMESERIES
   ```
4. Compare the query experience (NRQL) to Datadog's tag-based metric query syntax and CloudWatch Logs Insights' pipe syntax from Day 99 — note which felt most natural to you and why.

**Success criteria:** You've run at least one real NRQL query against your own cluster's data and can describe one concrete difference between NRQL and Datadog's query approach.

---

### Cleanup

```bash
helm uninstall datadog-agent
kubectl delete -f <sample-app-deployment.yaml>   # remove the instrumented demo workload
# Remove the New Relic Kubernetes integration
helm uninstall newrelic-bundle -n newrelic 2>/dev/null || true
# Cancel/downgrade trial accounts if you don't intend to keep using them, to avoid surprise billing after trial expiry
```

### Stretch challenge

Build one Datadog dashboard combining a metric widget (CPU/memory), an APM widget (p99 latency for your service), and a log widget (error count) on the same screen — then use trace-to-log correlation to jump from a slow trace directly to the log lines emitted during that exact span, and time how long that took versus how long it would take to manually grep logs across services for the same answer.
