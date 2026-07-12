# Day 65 — Cheatsheet: Deployment Strategies

## Rolling Update (vanilla Kubernetes default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # extra pods allowed above `replicas`
      maxUnavailable: 25%    # pods allowed unavailable below `replicas`
```
Old + new pods coexist during rollout. Requires an accurate `readinessProbe`.

## Recreate

```yaml
spec:
  strategy:
    type: Recreate
```
All old pods killed first, then new pods created. Full downtime window. Use for breaking migrations / singleton workloads.

## Argo Rollouts — Blue/Green

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: my-app-active
      previewService: my-app-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 300        # keep old pods alive for fast rollback
      prePromotionAnalysis:
        templates: [{ templateName: smoke-test }]
```

```bash
kubectl argo rollouts promote my-app          # manual cutover
kubectl argo rollouts abort my-app            # cancel, revert to stable
```

## Argo Rollouts — Canary

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate }] }
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      trafficRouting:
        istio:
          virtualService: { name: my-app-vsvc }
        # or: nginx: {}  / alb: {}  / smi: {}
```

## AnalysisTemplate (Prometheus-based automated gate)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] < 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{status=~"5.."}[1m]))
            / sum(rate(http_requests_total[1m]))
```

## `kubectl argo rollouts` CLI

```bash
kubectl argo rollouts get rollout my-app --watch
kubectl argo rollouts set image my-app app=my-app:v2
kubectl argo rollouts promote my-app       # advance a paused step
kubectl argo rollouts pause my-app
kubectl argo rollouts abort my-app         # rollback to stable
kubectl argo rollouts retry my-app         # retry after abort
kubectl argo rollouts dashboard            # local visual UI
```

## Feature flags

```javascript
// LaunchDarkly-style
if (flags.isEnabled('new-checkout', { userId, plan })) { ... }

// Unleash-style
if (unleash.isEnabled('new-checkout', context)) { ... }
```
Deploy != release. Kill switch = instant flag flip, no redeploy.

## Expand/contract migration pattern (for zero-downtime + schema change)

```
1. Expand:   add new column/table (backward-compatible, nullable)
2. Dual-write: new code writes both old + new schema
3. Backfill: migrate existing data in the background
4. Cutover: new code reads/writes only new schema
5. Contract: drop old column/table (irreversible — do last, separately)
```

## Decision quick-reference

```
Stateless, compatible change        -> Rolling Update
Breaking migration / singleton       -> Recreate (or expand/contract + Rolling)
Want instant rollback, can afford 2x -> Blue/Green
Want limited blast radius + metrics  -> Canary + AnalysisTemplate
Want product-level on/off control    -> Feature flag (LaunchDarkly/Unleash)
```
