# Day 65 — Deployment Strategies: Blue/Green & Canary with Argo Rollouts

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

Rolling Update and Recreate (file 1) answer "how do we replace pods safely" but give you zero control over *which* real users hit the new version, or the ability to automatically bail out based on live production metrics. Blue/Green and Canary — implemented in Kubernetes via **Argo Rollouts**, a drop-in replacement for the standard Deployment controller — add exactly that control, and they're the deployment strategies that come up in almost every "walk me through a zero-downtime deployment" interview question because they demonstrate you think about *risk exposure*, not just mechanics.

## Why vanilla Kubernetes needs Argo Rollouts for this

A standard Kubernetes `Deployment` has no concept of "keep the old version fully running and only cut traffic over on my command," nor "send 10% of traffic to the new version and watch metrics before proceeding." **Argo Rollouts** replaces the `Deployment` resource with a `Rollout` custom resource that implements these richer strategies natively, integrating with a real traffic-splitting layer (a service mesh like Istio/Linkerd, an Ingress controller like NGINX/ALB, or a Gateway API implementation) to actually control percentages of live traffic — not just pod counts.

## Blue/Green deployment

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: my-app-active      # what real users hit
      previewService: my-app-preview    # where the new version is reachable for testing
      autoPromotionEnabled: false        # require a human/automated gate before cutover
      scaleDownDelaySeconds: 300         # keep old version running for 5 min after cutover, for fast rollback
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
  template:
    spec:
      containers:
        - name: app
          image: my-app:v2
```

**Mechanism**: both `v1` (active, fully scaled) and `v2` (preview, fully scaled) run **simultaneously**, at full capacity, but only `v1` receives real user traffic via `activeService`. The `previewService` lets you (or automated smoke tests) hit `v2` directly to validate it before it ever sees real traffic. When you're confident, you **promote** — `activeService` switches to point at `v2`'s pods, and traffic cuts over essentially instantly (a Service selector change, not a gradual pod replacement).

**Key properties**:
- **Instant, all-or-nothing cutover** — unlike a rolling update's gradual pod-by-gradual-pod replacement, blue/green switches all traffic at once. This means rollback is also instant: just switch `activeService` back to the old ReplicaSet, which is exactly why `scaleDownDelaySeconds` matters — keeping the old version's pods running (but idle) for a window after cutover means "instant rollback" is actually available, not just theoretical, because the old pods haven't been torn down yet.
- **Double the resource cost during the transition** — both full-capacity versions run at once, which is the real tradeoff versus Rolling Update's partial-surge overhead.
- **`prePromotionAnalysis`** lets you wire automated checks (smoke tests, synthetic monitoring hitting the preview service) as a gate *before* promotion — combining human judgment and automated verification rather than relying on either alone.

## Canary deployment with traffic splitting

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      trafficRouting:
        istio:
          virtualService:
            name: my-app-vsvc
```

**Mechanism**: unlike blue/green's instant all-or-nothing cutover, canary **gradually shifts a percentage of real production traffic** to the new version, step by step, pausing between steps — either for a fixed duration or indefinitely until manually resumed. `trafficRouting` delegates the actual percentage-based traffic split to a mesh/ingress that supports weighted routing (Istio `VirtualService`, NGINX Ingress canary annotations, ALB weighted target groups, SMI, Gateway API).

**Analysis steps are what make this "automated" rather than just "gradual"**: an `AnalysisTemplate` queries a metrics source (Prometheus, Datadog, CloudWatch) for a defined success condition, and if the new version's real-traffic metrics (error rate, latency, custom business metric) fail the check, Argo Rollouts **automatically aborts and rolls back** — no human needs to be watching a dashboard at 2 AM for this to work correctly.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
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
            sum(rate(http_requests_total{job="my-app",status=~"5.."}[1m]))
            /
            sum(rate(http_requests_total{job="my-app"}[1m]))
```

## Blue/Green vs. Canary — choosing based on actual risk profile

| | Blue/Green | Canary |
|---|---|---|
| Traffic exposure to new version | 0% until instant full cutover | Gradual, small percentage first |
| Blast radius if new version is broken | 100% of users, immediately, at cutover | Limited to the canary percentage, caught by analysis before it grows |
| Resource cost | 2x full capacity during transition | Slight overhead (extra replicas for canary %), much less than 2x |
| Rollback speed | Instant (switch Service back) | Fast, but traffic must ramp back down (usually still fast) |
| Needs a traffic-splitting layer? | No (just two Services + a selector swap) — simpler infra requirement | Yes — mesh/ingress with weighted routing support required |
| Best for | Confident releases where you want instant, clean rollback capability and can afford double resources briefly | High-risk changes where you want to limit blast radius and gate progression on live metrics automatically |

**The honest interview answer**: canary is generally the *safer* default for anything with meaningful production risk, precisely because it limits blast radius mathematically (10% of traffic, not 100%) and can be fully automated with analysis gates — but it requires traffic-splitting infrastructure blue/green doesn't. Blue/green is simpler infra-wise and gives a genuinely instant rollback, but a bad `v2` still hits 100% of users the instant it's promoted, for however long it takes to notice and switch back.

## Points to Remember

- Blue/green: two full-capacity environments, zero traffic to the new one until an instant, all-at-once cutover — simple mental model, double resource cost, instant rollback (if old pods are kept alive via `scaleDownDelaySeconds`).
- Canary: gradual percentage-based traffic shift with pauses, requiring a traffic-splitting layer (Istio/NGINX/ALB/Gateway API) that Argo Rollouts delegates the actual routing to.
- `AnalysisTemplate` + Prometheus (or similar) queries are what turn a canary from "gradual" into "automatically self-aborting on bad metrics" — the actual production-safety mechanism, not just the traffic percentages.
- Canary limits blast radius mathematically (a bad release only affects the canary %); blue/green's blast radius on a bad promotion is 100% of traffic until someone notices and reverts.
- Both strategies require Argo Rollouts (or an equivalent) — vanilla Kubernetes Deployments cannot do either natively.

## Common Mistakes

- Implementing "blue/green" by hand with two Deployments and manually editing a Service selector, then being surprised there's no automated analysis gate, pre-promotion testing hook, or clean rollback bookkeeping — reinventing a worse version of what Argo Rollouts already provides.
- Setting `scaleDownDelaySeconds` to `0` (or omitting it) on a blue/green Rollout, so the old version's pods are torn down immediately at cutover — turning "instant rollback" into "need to redeploy the old version from scratch," defeating the main advantage of blue/green.
- Writing a canary `successCondition` against a metric with too much natural noise/low traffic volume — a canary step with only a handful of requests can fail (or pass) an error-rate threshold by pure chance, giving false confidence either direction.
- Choosing canary for a change that's actually schema-breaking between versions (same problem as Rolling Update in file 1) — canary controls traffic exposure, it does not solve version-compatibility problems between old and new code sharing a database.
- Forgetting that canary requires an actual traffic-splitting mechanism to be correctly configured (Istio `VirtualService`, ALB weighted groups, etc.) — misconfigured routing means `setWeight: 10` doesn't actually mean 10% of real traffic, silently invalidating the whole safety model.
