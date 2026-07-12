# Day 45 — Cheatsheet: K8s Advanced Scheduling

## nodeSelector

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

## Node affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/arch, operator: In, values: ["amd64"]}
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - {key: node-lifecycle, operator: In, values: ["on-demand"]}
```
Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. Terms across list = OR. Expressions within one term = AND.

## Taints & tolerations

```bash
kubectl taint nodes NODE key=value:NoSchedule       # add taint
kubectl taint nodes NODE key=value:NoSchedule-       # remove taint (trailing -)
kubectl describe node NODE | grep Taints
```

```yaml
tolerations:
- key: "lifecycle"
  operator: "Equal"       # or "Exists" (no value needed)
  value: "spot"
  effect: "NoSchedule"    # or PreferNoSchedule / NoExecute
  tolerationSeconds: 300  # only relevant for NoExecute
```
Effects: `NoSchedule` (blocks new), `PreferNoSchedule` (soft), `NoExecute` (evicts running pods too).

## Pod affinity / anti-affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - {key: app, operator: In, values: ["web-frontend"]}
      topologyKey: "topology.kubernetes.io/zone"
```
`topologyKey: kubernetes.io/hostname` = per-node. `topology.kubernetes.io/zone` = per-AZ.

## Topology spread constraints

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
  labelSelector:
    matchLabels: {app: web-frontend}
```

## PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: business-critical}
value: 1000000
globalDefault: false
```
```yaml
spec:
  priorityClassName: business-critical
```
Reserved system range: ≥ 1,000,000,000 (`system-cluster-critical`, `system-node-critical`).

## Preemption / eviction inspection

```bash
kubectl get pod <pod> -o jsonpath='{.status.nominatedNodeName}'
kubectl get events --sort-by=.lastTimestamp | grep -i preempt
```

## Descheduler (kubernetes-sigs/descheduler)

```bash
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
helm install descheduler descheduler/descheduler --set kind=CronJob --set schedule="*/15 * * * *"
```
Strategies: `RemoveDuplicates`, `LowNodeUtilization`, `RemovePodsViolatingNodeAffinity`, `RemovePodsViolatingNodeTaints`, `RemovePodsViolatingTopologySpreadConstraint`, `RemovePodsViolatingInterPodAntiAffinity`.

## Quick decision table

```
Which node?        -> nodeSelector / nodeAffinity
Never on X nodes    -> taint X, no toleration on critical pods
Spread across pods  -> podAntiAffinity (binary) / topologySpreadConstraints (proportional)
Who wins under pressure -> PriorityClass + preemption
Fix drift over time -> Descheduler
```
