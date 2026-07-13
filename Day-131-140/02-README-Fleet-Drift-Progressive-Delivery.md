# Day 131-140 — GitOps at Scale II: Fleet Management, Drift Detection, and Progressive Delivery at Scale

**Phase:** 4 – Advanced/Specialization | **Week:** W23-W24 | **Domain:** Advanced | **Flag:** —

## Brief

ApplicationSets solve "how do I template one Application definition across many targets." They don't solve two other problems that show up specifically once you're operating 50+ clusters: (1) some orgs need a tool that treats *clusters themselves* as the primary managed unit rather than Kubernetes-API-reachable destinations (Rancher **Fleet**'s model), and (2) rolling out a change safely across a fleet needs more than "sync and hope" — it needs drift detection to know what state you're actually in, and progressive delivery to limit blast radius while rolling out. This file covers both.

## Rancher Fleet — architecture and how it differs from ArgoCD

Fleet is Rancher's GitOps engine, built around a different core abstraction than ArgoCD: instead of "one Application talks to one destination cluster via a stored kubeconfig/service account," Fleet uses an **agent-per-downstream-cluster** pull model from day one.

```
Git repo (fleet.yaml + manifests)
        │
        ▼
Fleet Manager (runs in the "upstream"/management cluster)
        │  watches GitRepo CRs, builds Bundles
        ▼
   Bundle  ──fan-out──▶  BundleDeployment (per target cluster)
                              │
                              ▼
                    Fleet Agent (runs INSIDE each downstream cluster,
                    pulls its BundleDeployment, applies it locally)
```

- **GitRepo** — a CR pointing at a Git repo/path/branch, the Fleet equivalent of an ArgoCD `Application` source.
- **Bundle** — the compiled set of resources Fleet produces from a `GitRepo` (rendered Helm/Kustomize/raw YAML), before it's targeted at specific clusters.
- **BundleDeployment** — a Bundle scoped to one specific downstream cluster — this is what the cluster's local agent actually reconciles against.
- **Fleet Agent** — runs *inside* every downstream cluster (registered once, e.g. via a Rancher-generated registration manifest) and pulls its `BundleDeployment`s — the actual apply happens locally, in-cluster, not from the management cluster pushing out. This mirrors ArgoCD's pull-based security model, just organized around clusters-as-first-class-objects rather than Applications-as-first-class-objects.
- **Cluster targeting via `targetCustomizations`** in `fleet.yaml`: define per-cluster or per-cluster-group overrides (Helm value overrides, different replica counts) without forking the base manifests — conceptually the same problem ApplicationSet's Git `files` generator solves, solved with Fleet's own targeting syntax instead.

```yaml
# fleet.yaml — lives at the root of the Git path Fleet watches
defaultNamespace: my-app
helm:
  chart: ./chart
targetCustomizations:
  - name: prod-overrides
    helm:
      values:
        replicaCount: 5
    clusterSelector:
      matchLabels:
        env: production
  - name: canary-overrides
    helm:
      values:
        replicaCount: 1
    clusterSelector:
      matchLabels:
        rollout-group: canary
```

**Fleet vs. ArgoCD, when each fits better:**

| | ArgoCD | Fleet |
|---|---|---|
| Primary unit | Application (source → destination) | Cluster (agent pulls its own BundleDeployment) |
| Best fit | Kubernetes-native shops, deep Argo Rollouts/ApplicationSet ecosystem | Rancher-managed fleets, edge/IoT-style large cluster counts, mixed cluster lifecycle (Fleet also handles cluster *provisioning* via Rancher) |
| Multi-cluster scaling primitive | ApplicationSet generators (Cluster/Matrix/Git) | `targetCustomizations` + cluster/cluster-group labels, native to every GitRepo |
| UI/ecosystem | Argo ecosystem (Rollouts, Notifications, Image Updater) | Rancher UI, integrated with Rancher's cluster management |

In practice, many orgs running Rancher for cluster lifecycle management use **Fleet for the base platform layer** (namespaces, RBAC, network policy, monitoring agents — things every cluster needs) and **ArgoCD for application-level GitOps** (business services, per-tenant Applications) — they are not mutually exclusive, and "complement" is often the realistic answer, not "replace."

## Config drift detection

**Drift** = the live cluster state no longer matches the state declared in Git. It happens constantly in practice — someone `kubectl edit`s a Deployment during an incident, a controller (HPA, VPA, a mutating webhook) mutates a field the GitOps tool doesn't own, or a `kubectl scale` gets run by habit.

**How ArgoCD detects it:**
- ArgoCD continuously **diffs** the live cluster state against the rendered manifests from Git, using a **normalized** comparison (it understands that Kubernetes and controllers add default fields, so it doesn't flag every server-side default as drift — this is configurable per-resource via `ignoreDifferences`).
- Result shows up as **`OutOfSync`** application status (vs. `Synced`), visible in the UI, CLI, and as a Prometheus metric (`argocd_app_info{sync_status="OutOfSync"}`) — this is what you'd alert on at fleet scale, not manual dashboard-watching across 50+ clusters.
- `argocd app diff <app-name>` shows the exact field-level delta between live and desired state — the first command to reach for when triaging why an app shows OutOfSync.

```bash
argocd app diff checkout-service-prod
argocd app get checkout-service-prod --show-operation
```

**Correcting drift — two modes:**
- **Manual sync** (`argocd app sync <app>`) — someone/something explicitly triggers reconciliation; drift is visible but not auto-corrected until then.
- **`selfHeal: true`** (part of `syncPolicy.automated`) — ArgoCD actively re-applies the Git-declared state the moment drift is detected, without waiting for a manual sync or the next polling interval. This is what turns "GitOps" from "a deploy mechanism" into "a continuously-enforced desired state" — any out-of-band change gets reverted automatically, usually within seconds of the next reconciliation loop (default 3 minutes, plus this is also driven by webhook-triggered refreshes on Git pushes).

```yaml
syncPolicy:
  automated:
    prune: true       # delete resources removed from Git
    selfHeal: true     # revert any live drift back to Git's declared state
  syncOptions:
    - ApplyOutOfSyncOnly=true   # at scale, skip re-applying resources that ARE in sync, to cut reconciliation cost
```

**At 50+ cluster scale, drift detection has a specific operational concern:** running a full diff/reconcile loop against every Application on every cluster on a tight interval is real API-server load, multiplied across the fleet. `ApplyOutOfSyncOnly=true` and tuning `ARGOCD_RECONCILIATION_TIMEOUT` are the standard levers; also consider ArgoCD's **Notifications** controller to alert on sustained `OutOfSync` rather than relying on someone noticing in the UI.

**Fleet's drift model** is conceptually the same (agent compares live state to its `BundleDeployment`, and can be configured to `forceSyncGeneration`-bump to re-apply), but it doesn't have as fine-grained a `selfHeal`-equivalent toggle as ArgoCD per-Application — drift correction in Fleet is closer to "always reconciling" by default per Bundle.

## Progressive delivery at scale — Argo Rollouts

A plain `Deployment` synced by ArgoCD is all-or-nothing (rolling update, no traffic-shaping, no automated abort). **Argo Rollouts** replaces `Deployment` with a `Rollout` CRD that adds canary/blue-green strategies with **automated, metrics-driven promotion/abort** — the piece a basic rolling update structurally cannot provide.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 2m }
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: checkout-service
        - setWeight: 25
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
      trafficRouting:
        istio:                      # or nginx / smi / alb / appmesh
          virtualService:
            name: checkout-service-vsvc
            routes: [primary]
  selector:
    matchLabels: { app: checkout-service }
  template:
    metadata: { labels: { app: checkout-service } }
    spec:
      containers:
        - name: checkout-service
          image: registry.internal/checkout-service:v2.3.1
```

**AnalysisTemplate** is what makes a step's promotion automated rather than a timer-based guess — it runs a metric query (Prometheus, Datadog, CloudWatch, Kayenta, etc.) and compares against a threshold; failing analysis **auto-aborts and rolls back the canary**, no human paged at 2am to eyeball a dashboard.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result >= 0.95
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.observability:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

**Progressive delivery across many clusters** — the actual "at scale" part — comes from combining Rollouts with the ApplicationSet patterns from file 1: each cluster gets its own `Rollout` (rendered per-cluster by an ApplicationSet's Cluster or Matrix generator), so canary progression happens **independently per cluster**, not as one global percentage across the fleet. Two common scale-specific patterns:

- **Wave-based cluster rollout**: use ApplicationSet's `strategy: RollingSync` (a feature of the ApplicationSet controller itself, distinct from the per-cluster Rollout's own canary steps) to stagger *which clusters* get the new Application version first, in labeled waves — e.g., `canary` region clusters first, then `standard` tier, then `critical` tier last — while *within* each cluster, Argo Rollouts independently canaries the pods.
  ```yaml
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: rollout-group
              operator: In
              values: [canary]
          maxUpdate: 100%
        - matchExpressions:
            - key: rollout-group
              operator: In
              values: [standard]
          maxUpdate: 25%     # only update 25% of matched clusters at a time
        - matchExpressions:
            - key: rollout-group
              operator: In
              values: [critical]
          maxUpdate: 10%
  ```
- **Shared AnalysisTemplate, per-cluster metrics endpoint**: templating the Prometheus `address` per cluster (via the same generator params used for the rest of the template) so each cluster's canary analysis queries *that cluster's* metrics, not a single global Prometheus that may not even have visibility into every fleet member.

This two-layer model — ApplicationSet `RollingSync` gating *which clusters* move, Argo Rollouts gating *how traffic shifts within* a cluster — is the standard answer to "how do you progressively deliver to 50+ clusters without a single bad build reaching all of them simultaneously."

## Points to Remember

- Fleet's core unit is the **cluster** (agent-per-cluster, pull-based, `GitRepo` → `Bundle` → `BundleDeployment`); ArgoCD's core unit is the **Application** — this difference drives which tool fits a given org's operating model, and they're commonly run together (Fleet for base platform layer, ArgoCD for app-level GitOps).
- Drift = live state ≠ Git-declared state; ArgoCD surfaces it as `OutOfSync` and lets you inspect it with `argocd app diff`; `selfHeal: true` is what turns detection into automatic correction instead of a dashboard you have to watch.
- At 50+ clusters, drift-checking cost is real load — use `ApplyOutOfSyncOnly=true` and alert on sustained `OutOfSync` via Notifications rather than manual monitoring.
- A basic rolling `Deployment` has no automated abort mechanism — **Argo Rollouts + AnalysisTemplate** is what makes canary promotion/rollback metric-driven instead of a fixed timer and a hope.
- At fleet scale, progressive delivery is two layers: ApplicationSet `RollingSync` staggers *which clusters* update first (wave-based, by label), while each cluster's own `Rollout` independently canaries traffic within that cluster.

## Common Mistakes

- Treating Fleet and ArgoCD as strictly competing choices instead of considering the common real-world pattern of using each for a different layer (platform baseline vs. application GitOps).
- Enabling `selfHeal: true` fleet-wide without first understanding what "live drift" currently exists — this can trigger a wave of unexpected reverts the moment it's turned on, surprising teams who'd been quietly hand-patching resources.
- Rolling out a canary strategy with `pause` steps but no `analysis` step — this is just a slow rolling update with extra YAML, not real progressive delivery; nothing is actually gating promotion on real signal.
- Running canary analysis against one global Prometheus instance that doesn't actually have per-cluster metrics visibility across a 50+ cluster fleet, so the "analysis" is silently querying the wrong (or empty) data.
- Rolling out to all 50+ clusters simultaneously with a plain Cluster generator and no `RollingSync` strategy — a single bad build reaches the entire fleet at once instead of a canary wave catching it on a handful of clusters first.
- Ignoring drift-checking cost at scale — running full unthrottled reconciliation across hundreds of Applications on a short interval and then being surprised the ArgoCD API server or the target clusters' API servers are under real load.
