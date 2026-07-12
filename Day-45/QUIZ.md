# Day 45 — Quiz: K8s Advanced Scheduling

Try to answer without looking at your notes. Answers are at the bottom.

1. What's the key limitation of `nodeSelector` that node affinity was built to fix?
2. What's the difference between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`?
3. What does "IgnoredDuringExecution" actually mean, and what does it imply about a pod whose node's labels change after it starts running?
4. What's the fundamental conceptual difference between affinity and a taint/toleration pair — attraction vs. what?
5. Name the three taint effects and explain the one that can evict already-running pods.
6. What does `topologyKey` actually represent, and how does choosing `kubernetes.io/hostname` vs `topology.kubernetes.io/zone` change the meaning of an anti-affinity rule?
7. What does `maxSkew` control in a topology spread constraint, and how does `whenUnsatisfiable: ScheduleAnyway` change its enforcement compared to `DoNotSchedule`?
8. Does a PodDisruptionBudget protect a pod from being preempted by a higher-priority pending pod? Why or why not?
9. What is the reserved priority value range for system-critical components, and why shouldn't application workloads use values in that range?
10. What problem does the Descheduler solve that the default scheduler fundamentally cannot?
11. Do Descheduler evictions respect PodDisruptionBudgets? How does that compare to preemption?
12. **Interview question:** How do you ensure your critical pods never land on spot/preemptible nodes?

---

## Answers

1. `nodeSelector` only supports exact-match key-value pairs ANDed together — no OR logic, no operators like `In`/`NotIn`/`Exists`, and no "soft preference" option. Node affinity adds richer operators and two strength levels (required/preferred).
2. `required...` is a hard constraint — the scheduler will not place the pod on a non-matching node, and the pod stays Pending if no match exists. `preferred...` is a weighted soft constraint — the scheduler tries to honor it but will still schedule the pod elsewhere if necessary.
3. It means affinity/anti-affinity rules are evaluated only at scheduling time, not continuously. If a node's labels change after a pod is already running there in a way that would now violate the rule, the pod is **not** evicted — it just keeps running where it is until it's rescheduled for some other reason.
4. Affinity describes what a pod *wants* (attraction toward matching nodes/pods). Taints describe what a node *refuses* (repulsion of pods), and a toleration is only permission to ignore that refusal — it does not make the scheduler prefer or attract the pod to that node.
5. `NoSchedule` (blocks new pod placement, doesn't affect already-running pods), `PreferNoSchedule` (soft version of NoSchedule), and `NoExecute` (blocks new placement AND evicts already-running pods that lack a matching toleration, subject to a `tolerationSeconds` grace period). `NoExecute` is the one that can evict running pods.
6. `topologyKey` is a node label whose value defines what "same" or "different" means for the rule — the scheduler groups nodes by that label's value rather than comparing nodes individually. `kubernetes.io/hostname` makes anti-affinity mean "not on the same physical node"; `topology.kubernetes.io/zone` makes it mean "not in the same availability zone" — a much broader and more resilience-relevant guarantee.
7. `maxSkew` is the maximum allowed difference in matching-pod count between the topology domain with the most and the one with the fewest. With `whenUnsatisfiable: DoNotSchedule`, violating the constraint blocks scheduling (pod stays Pending); with `ScheduleAnyway`, the scheduler still tries to minimize skew as a preference but will schedule the pod even if the constraint can't be perfectly satisfied.
8. No — PodDisruptionBudgets only protect against *voluntary* disruptions (drains, scale-down). Preemption is treated as a necessary action to schedule a higher-priority pending pod, so the scheduler is explicitly allowed to violate a PDB's minAvailable/maxUnavailable guarantee when preempting.
9. Values ≥ 1,000,000,000 (one billion) are reserved for system-critical PriorityClasses like `system-cluster-critical` and `system-node-critical`, used by core components such as CoreDNS and kube-proxy. Application workloads shouldn't use this range because it's intended to keep core cluster functionality unambiguously above any application-level priority, avoiding confusing interactions with critical system pods.
10. The default scheduler only makes a placement decision once, at pod creation — it never revisits already-running pods even if the cluster state has changed in ways that make that placement suboptimal (uneven utilization, new nodes joining later, taint/affinity drift). The Descheduler periodically evaluates running pods against configurable strategies and evicts ones that violate them, letting the scheduler re-place them better.
11. Yes — Descheduler evictions are voluntary disruptions and go through the standard Eviction API, so they do respect PodDisruptionBudgets. This is the opposite of preemption, which is allowed to violate PDBs because it's treated as a necessary (not voluntary) action.
12. Strong answer: "Taint every spot/preemptible node (e.g., `lifecycle=spot:NoSchedule`) and only add a matching toleration to the specific workloads you've explicitly decided are safe to run on spot capacity. Critical pods, having no toleration for that taint, are structurally unable to be scheduled there — enforced by the scheduler, not by a convention someone has to remember. Node affinity alone isn't sufficient here because a 'preferred' rule doesn't guarantee anything, and even a 'required' rule pinning workloads to on-demand nodes only protects the workloads you remembered to configure — new critical pods without that rule could still land on spot. Taints put the opt-in burden on the exception (spot-tolerant batch workloads) rather than requiring every critical workload to opt out correctly. In practice, managed node groups and Karpenter often auto-apply a taint like `karpenter.sh/capacity-type=spot` for exactly this reason, so check what's already there before adding a custom one."
