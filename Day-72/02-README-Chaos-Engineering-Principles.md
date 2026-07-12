# Day 72 — Load & Chaos Testing: Chaos Engineering Principles and LitmusChaos

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** Testing | **Flag:** ⚡ Interview-critical

## Brief

Load testing proves your system handles expected traffic. Chaos engineering proves something different and arguably scarier: **your system survives things actually going wrong** — a node dying, a network partition, a dependency timing out — *while it's still serving real traffic*. This is one of the most conceptually interesting and frequently-interviewed topics in this whole phase, because it inverts the usual instinct (avoid causing failures) into a discipline (deliberately, safely cause failures to build confidence they won't cause an outage). Getting the "how do you do this *safely* in production" part right is exactly what separates a thoughtful answer from a reckless-sounding one.

## The core principles (from the Principles of Chaos Engineering)

1. **Define a "steady state"** — a measurable, normal-operation baseline (e.g., request success rate, p99 latency, throughput) *before* you inject anything. Without this, you have no way to objectively detect whether an experiment actually caused a problem versus normal noise.
2. **Hypothesize that steady state holds in both the control and experimental group** — you're not trying to break the system for its own sake; you're testing a specific hypothesis like "if one pod dies, the steady state metrics remain unaffected because the deployment has enough replicas and the load balancer correctly reroutes."
3. **Introduce real-world, realistic failure events** — pod kills, network latency/partition, CPU/memory pressure, dependency (DNS, database) unavailability — the actual categories of failure that happen in production, not arbitrary/contrived faults.
4. **Run experiments in production, when possible** — this is the most counter-intuitive and most misunderstood principle. Staging environments rarely have production's real traffic patterns, real scale, real data shape, or real dependency behavior — many failure modes simply don't reproduce outside production. This does **not** mean "recklessly break production" — it means the *goal* is production-representative confidence, achieved through careful blast-radius control (below), not through skipping safety practice.
5. **Minimize blast radius** — the practice that makes principle 4 responsible rather than reckless: start with the smallest possible impact (one pod, a tiny percentage of traffic, a single availability zone, a non-critical time window), have an automated abort/rollback mechanism, and expand scope only after building confidence at a smaller scale.
6. **Automate experiments to run continuously** — a one-time chaos experiment only proves the system was resilient *on that day, under that specific configuration*. Continuous, automated chaos testing (in CI/CD, or scheduled) catches resilience regressions introduced by later changes — the same "run scans/tests continuously, not once" theme that's run through every day this week.

## LitmusChaos — chaos engineering for Kubernetes

LitmusChaos is a CNCF chaos engineering framework purpose-built for Kubernetes, expressing experiments as native Kubernetes Custom Resources (`ChaosEngine`, `ChaosExperiment`, `ChaosResult`) — fitting the same "operate it the way you operate everything else in the cluster" philosophy as Gatekeeper/Kyverno from Day 68.

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-chaos
  namespace: default
spec:
  appinfo:
    appns: default
    applabel: 'app=myapp'
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '30'          # experiment runs for 30 seconds
            - name: CHAOS_INTERVAL
              value: '10'          # kill a pod every 10 seconds within that window
            - name: FORCE
              value: 'false'       # graceful delete, not a forced/ungraceful kill
            - name: PODS_AFFECTED_PERC
              value: '25'          # blast-radius control: only 25% of matching pods, not all
```

```bash
kubectl apply -f pod-delete-chaosengine.yaml
kubectl get chaosresult -n default          # check pass/fail verdict after the experiment window
kubectl describe chaosresult pod-delete-chaos-pod-delete
```

`PODS_AFFECTED_PERC: '25'` is the blast-radius-control principle expressed directly in config — you don't kill every replica at once; you start small, prove the deployment's remaining replicas and the Service's load balancing genuinely absorb the loss without violating your steady-state metrics, and only then consider expanding scope.

### Beyond pod-delete — the categories LitmusChaos (and chaos engineering generally) targets

- **Pod-level**: pod-delete, pod-cpu-hog, pod-memory-hog, pod-network-latency/loss.
- **Node-level**: node-drain, node-cpu-hog, node taint simulation.
- **Network**: network partition between services, DNS chaos (simulating a dependency's DNS resolution failing).
- **Application-level**: disk-fill, container-kill (killing a specific sidecar rather than the whole pod).

The common thread: each targets a real, observed production failure category, matching principle 3 above — not arbitrary or synthetic faults invented for their own sake.

## Verifying your safety nets actually work — the point of the exercise

The entire value of a `pod-delete` experiment against a Deployment with an HPA (Horizontal Pod Autoscaler) and multiple replicas is confirming, empirically, that:
- The Deployment's ReplicaSet actually reschedules a replacement pod promptly.
- The Service's endpoints update correctly and traffic reroutes away from the killed pod without user-visible errors.
- If load was also elevated during the experiment, the HPA actually scales out in response, rather than the assumption "HPA is configured, so it must work" going untested until a real incident.

This is precisely why chaos engineering and load testing are grouped together in this day — a genuinely rigorous chaos experiment is often run *while* a load test (file 1) is generating realistic concurrent traffic, so you observe both "does it recover from the fault" and "does it recover fast enough under real load" simultaneously.

## Points to Remember

- Chaos engineering requires a measurable steady-state baseline defined *before* injecting failure — without it, you can't objectively tell whether an experiment caused a real problem or you're just looking at normal noise.
- "Run in production" is a goal for representative confidence, not a license for recklessness — it's only responsible when paired with deliberate blast-radius minimization (small percentage of pods/traffic, automated abort, gradual scope expansion).
- LitmusChaos expresses experiments as Kubernetes CRDs (`ChaosEngine`), fitting the same declarative, GitOps-friendly operating model as the rest of a Kubernetes-native platform.
- `PODS_AFFECTED_PERC` (and equivalents in other chaos tools) is blast-radius control expressed directly as configuration — always start small and prove safety at small scope before expanding.
- A one-time chaos experiment only proves resilience under that day's specific configuration — automating experiments to run continuously is what catches resilience regressions introduced by later, unrelated changes.

## Common Mistakes

- Running a chaos experiment with no defined, measurable steady-state baseline beforehand — with nothing to compare against, you can't objectively distinguish "the experiment caused a real problem" from ordinary metric noise.
- Treating "chaos engineering happens in production" as license to run large-blast-radius experiments (all replicas, all availability zones) without a gradual, small-scope-first approach and an automated abort mechanism — this is how a chaos experiment itself causes the outage it was meant to help prevent.
- Running an experiment once, declaring the system "resilient," and never repeating it — missing resilience regressions introduced by later code/config/infrastructure changes that were never re-validated against the same failure scenario.
- Assuming an HPA or multi-replica Deployment "must" handle a pod loss correctly because it's configured, without ever actually testing it — configuration existing is not the same as configuration working correctly under real failure conditions.
- Scheduling chaos experiments during peak business hours or without notifying on-call/support teams — even a well-scoped experiment can trigger real alerts and confuse responders who don't know it's an intentional test, wasting incident-response effort on a non-incident.
