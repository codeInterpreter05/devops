# Day 51 — K8s Upgrades & Cluster Management: Workload Safety During Upgrades

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Kubernetes | **Flag:** —

## Brief

An upgrade strategy at the cluster/node-group level (previous file) is only half the story — the other half is making sure the *workloads* running on those nodes survive the process without an outage. Cordon, drain, and PodDisruptionBudgets are the three mechanisms Kubernetes gives you to make node replacement (whether for upgrades, scaling, or maintenance) safe for the applications sitting on top. This is exactly the mechanism EKS-managed node group upgrades invoke automatically under the hood — understanding it manually is what lets you debug it when the automation doesn't behave the way you expected.

## Cordon — stop new scheduling, without touching existing pods

```bash
kubectl cordon <node-name>
```
Marks a node `Unschedulable` — the scheduler will not place any *new* pod on it, but every pod **already running** on that node keeps running untouched. Cordon is the first, non-disruptive step: it lets you "freeze" a node in preparation for later action without affecting current traffic at all.

```bash
kubectl get nodes    # cordoned nodes show SchedulingDisabled in STATUS
kubectl uncordon <node-name>    # reverse it
```

## Drain — evict existing pods safely

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
`drain` does three things in sequence: (1) cordons the node if not already cordoned, (2) evicts every pod on the node (using the **Eviction API**, not a raw delete — this matters, see below), and (3) waits for each eviction to actually succeed (respecting PodDisruptionBudgets) before moving to the next, up to a timeout.

Two flags you'll type constantly, and why they're needed:
- **`--ignore-daemonsets`** — DaemonSet-managed pods (like a log collector or CNI agent) are meant to run on every node, including this one when it comes back; `drain` can't "evict" them somewhere else since they're tied to the node itself, so you tell it to skip them rather than fail the whole drain.
- **`--delete-emptydir-data`** — pods using `emptyDir` volumes have node-local scratch data that will simply be lost when the pod is evicted anyway (it's ephemeral by definition); this flag is an explicit acknowledgment so you don't drain blind to that fact.

**Why eviction, not deletion**: `kubectl drain` calls the `Eviction` subresource on each pod rather than just deleting it. The Eviction API explicitly checks any relevant `PodDisruptionBudget` before allowing the eviction to proceed — if evicting this pod would violate a PDB (see below), the eviction request is **rejected** (`429 Too Many Requests`, "Cannot evict pod as it would violate the pod's disruption budget"), and `drain` will retry until the PDB allows it or the operation times out. A plain `kubectl delete pod` bypasses this check entirely — it doesn't ask permission, it just terminates the pod, PDB or not, which is precisely why manual `delete` should never substitute for `drain` during planned maintenance.

## PodDisruptionBudget (PDB) — the guardrail that makes drain safe

A PDB declares the **minimum availability** (or maximum simultaneous unavailability) you're willing to tolerate for a set of pods (matched by label selector) during **voluntary disruptions** — node drains, cluster-autoscaler scale-downs, and similar planned operations initiated through the Eviction API.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2        # OR maxUnavailable: 1 (mutually exclusive with minAvailable)
  selector:
    matchLabels:
      app: my-app
```

- **`minAvailable`** — at least this many (count or percentage, e.g. `"50%"`) matching pods must remain available at all times; an eviction that would drop below this is refused.
- **`maxUnavailable`** — at most this many matching pods may be unavailable simultaneously; functionally the inverse expression of the same constraint, choose whichever reads more naturally for your replica count (e.g., `maxUnavailable: 1` on a 3-replica Deployment is equivalent in effect to `minAvailable: 2`, but `maxUnavailable: 1` scales more sensibly if replica count changes later).

**Critical distinction — voluntary vs. involuntary disruptions**: a PDB only protects against **voluntary** disruptions performed through the Eviction API (drains, autoscaler scale-downs, manual `kubectl drain`). It does **not** protect against **involuntary** disruptions — a node hardware failure, a kernel panic, an out-of-resource kill, a spot instance reclamation. A PDB cannot stop a physical node from dying; it can only make orchestrated, planned operations respect an availability floor. This distinction is a frequent interview trap: a PDB is not a general high-availability guarantee, it's specifically a brake on Kubernetes' own voluntary eviction machinery.

**Practical upgrade flow combining all three:**
```bash
# For each node being upgraded/replaced, one at a time:
kubectl cordon node-1                                    # stop new scheduling
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --timeout=300s
# drain blocks on PDB-protected pods until it's safe to evict them,
# or until timeout — investigate manually if it times out rather than forcing it
# ... perform the actual upgrade/replacement of node-1 ...
kubectl uncordon node-1                                   # (if reusing the node) allow scheduling again
```

If `drain` hangs, it's almost always because a PDB genuinely can't be satisfied — e.g., a Deployment with `minAvailable: 3` but only 3 replicas total, meaning *zero* voluntary disruption is ever allowed. That's not a bug in `drain`; it's the PDB doing exactly what it was configured to do, and the real fix is either scaling the Deployment up first or loosening the PDB temporarily.

## Points to Remember

- Cordon = stop future scheduling only, no effect on running pods. Drain = cordon + evict existing pods safely via the Eviction API, respecting PDBs.
- `drain` uses the Eviction API, not raw pod deletion — that's precisely what makes it PDB-aware; `kubectl delete pod` bypasses PDBs entirely.
- `--ignore-daemonsets` and `--delete-emptydir-data` are required in almost every real drain because DaemonSet pods can't be "moved" and emptyDir data is inherently ephemeral.
- A PDB expresses `minAvailable` or `maxUnavailable` (mutually exclusive) for a labeled set of pods, and only constrains **voluntary** disruptions (drains, autoscaler actions) — it does nothing against node failure or involuntary kills.
- A drain that hangs indefinitely is usually a PDB correctly refusing to let availability drop below its configured floor — check replica count vs. PDB math before assuming `drain` itself is broken.

## Common Mistakes

- Using `kubectl delete pod` instead of `kubectl drain`/eviction during planned node maintenance, silently bypassing PDBs and potentially taking an application below its minimum availability during what was supposed to be a safe operation.
- Setting `minAvailable` equal to total replica count (e.g., `minAvailable: 3` on a 3-replica Deployment), which makes the PDB block *every* voluntary eviction forever — effectively preventing any node drain from ever completing for that workload.
- Believing a PDB protects against all forms of disruption, including hardware failure or spot instance interruption — it explicitly does not; those are involuntary disruptions outside the Eviction API's control.
- Forgetting `--ignore-daemonsets` and having `kubectl drain` fail/hang on DaemonSet-managed pods that were never going to be "evicted" successfully in the way regular pods are.
- Draining nodes one-by-one without checking cumulative effect across multiple simultaneous node replacements (e.g., an automated script draining several nodes in parallel) — even with correct PDBs per-Deployment, uncoordinated parallel drains across nodes hosting the same Deployment's replicas can still cause more simultaneous disruption than intended if the automation doesn't serialize appropriately.
