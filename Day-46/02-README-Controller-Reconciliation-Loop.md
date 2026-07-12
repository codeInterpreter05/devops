# Day 46 — K8s Operators & CRDs: Controller Reconciliation Loop & Operator SDK

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

The reconciliation loop is the single mechanism underlying not just Operators but Kubernetes itself — the Deployment controller, ReplicaSet controller, and every built-in controller all work this exact same way. Understanding it deeply means you understand *why* Kubernetes is declarative and self-healing rather than an imperative task runner, and it's the concrete mechanism behind the abstract "Operator" concept from the previous file.

## The reconciliation loop, mechanically

Every controller (built-in or custom) runs the same fundamental cycle:

```
watch(desired state changes OR observed state changes)
  -> reconcile(objectKey)
       -> read current desired state (the custom resource spec)
       -> read current actual state (query the cluster/external system)
       -> diff desired vs actual
       -> take the minimum action needed to move actual toward desired
       -> update .status to reflect what happened
  -> repeat forever (on any relevant change, or periodic resync)
```

**Level-triggered, not edge-triggered — this is the single most important property.** The controller does not process "an event happened, react to it once." Instead, every reconcile call re-derives the *full* current state and compares it against the *full* desired state, then acts — regardless of *why* reconcile was triggered. This makes controllers naturally resilient to **missed events**: if the controller pod restarts and misses 10 minutes of changes, the next reconcile still converges correctly because it's comparing full current state to full desired state, not replaying a missed event log. Edge-triggered systems (act only on the specific event) are fragile to missed events in exactly the way level-triggered systems aren't.

**The reconcile function should be idempotent** — calling it 1 time or 100 times with the same inputs produces the same end state, with no side effects from repeated calls. This is why Operators are safe to reconcile on a periodic resync timer *in addition to* watch-triggered reconciles (a common pattern — e.g., resync every 5-10 minutes) as a safety net against any missed or dropped watch events, without that periodic resync causing harm.

**A concrete Postgres Operator reconcile example:**
1. Watch fires because `Cluster.spec.instances` changed from 3 to 4.
2. Reconcile reads the `Cluster` object: desired = 4 instances.
3. Reconcile queries the cluster: currently 3 StatefulSet replicas exist and are healthy, current primary is `pg-0`.
4. Diff: need 1 more replica.
5. Action: scale the StatefulSet to 4, wait for the new pod to become ready, run the Postgres-specific logic to join it as a streaming replica, update `Cluster.status.readyInstances` to reflect progress.
6. If interrupted (Operator pod restarts) halfway through — next reconcile re-reads current state (3 healthy + 1 provisioning) and continues from wherever reality actually is, not from where it assumed it left off.

## Controller-runtime concepts worth knowing by name

- **Informers/watches**: instead of polling the API server, controllers use a `SharedInformer` — a local, cached, kept-in-sync copy of relevant objects, updated via the Kubernetes watch API. This is why controllers can react in near-real-time without hammering the API server with polling requests.
- **Work queue**: events from the informer get pushed into a rate-limited work queue (deduplicated by object key), and reconcile workers pull from that queue — this decouples "an event arrived" from "reconcile actually runs," allowing retries with backoff on failure without losing the underlying event.
- **Owner references / garbage collection**: resources a controller creates (e.g., a StatefulSet created by a Postgres Operator) are typically stamped with an `ownerReference` back to the custom resource — deleting the `Cluster` object cascades deletion of everything it owns, via Kubernetes' built-in garbage collector, without the Operator needing custom cleanup logic.
- **Finalizers**: when a custom resource needs guaranteed cleanup of *external* (non-Kubernetes) resources before deletion completes — e.g., an Operator managing a cloud database that must be properly deprovisioned first — a finalizer is added to the object; Kubernetes blocks the object's actual deletion until the Operator removes the finalizer, guaranteeing the cleanup logic runs even if `kubectl delete` was issued.
- **Status subresource**: custom resources typically split `spec` (what the user wants, only they can edit) from `status` (what's actually happening, only the controller updates) — enforced via RBAC and the `/status` subresource, which is exactly the same spec/status split every built-in Kubernetes resource uses.

## Operator SDK basics

The **Operator SDK** (part of the Operator Framework) is a scaffolding/code-generation toolkit for building Operators, sitting on top of `controller-runtime` (the underlying Go library implementing everything described above). It supports three build approaches:

1. **Go-based Operators** — full control, scaffolds a `controller-runtime` project with boilerplate for CRD types, RBAC manifests, and a reconcile function stub you fill in. This is the path for genuinely custom logic.
2. **Ansible-based Operators** — the reconcile logic is an Ansible playbook/role instead of Go code; useful if your team's operational knowledge already lives as Ansible and you want to encode it as a controller without learning Go.
3. **Helm-based Operators** — the reconcile logic is literally "apply this Helm chart with values derived from the custom resource's spec" — essentially turns a Helm chart into a CRD-driven, continuously-reconciled deployment instead of a one-shot `helm install`. The simplest option, but limited to what Helm templating can express — no genuinely custom day-2 logic (failover, backup orchestration) beyond what the chart's templates already encode.

**Practical scaffolding workflow (Go-based)**:
```bash
operator-sdk init --domain example.com --repo github.com/you/my-operator
operator-sdk create api --group postgresql --version v1 --kind Cluster --resource --controller
# generates: api/v1/cluster_types.go (CRD Go struct), controllers/cluster_controller.go (Reconcile stub)
make manifests   # regenerates CRD YAML + RBAC from Go struct annotations (kubebuilder markers)
make install     # applies CRDs to your cluster
make run         # runs the controller locally against your cluster for fast iteration
```

The `+kubebuilder:` comment annotations above Go struct fields (e.g., `// +kubebuilder:validation:Minimum=1`) are how `controller-gen` generates the CRD's OpenAPI schema and RBAC manifests automatically from your Go types — you define the API once in Go, and the YAML artifacts (CRD, RBAC ClusterRole) are generated, not hand-written, keeping schema and code in sync.

## Points to Remember

- Reconciliation is level-triggered: every call re-derives full desired vs. actual state and acts on the diff, rather than reacting to a specific event — this is what makes controllers naturally resilient to missed or out-of-order events.
- Reconcile functions must be idempotent — safe to call repeatedly with no harmful side effects, which is what makes periodic resync (a safety net alongside watch-triggered reconciles) safe.
- Owner references give free cascading garbage collection; finalizers guarantee cleanup logic runs for external resources before Kubernetes finalizes an object's deletion.
- `spec` = desired state (user-writable), `status` = observed state (controller-writable only) — the same split every built-in Kubernetes resource follows, enforced via the `/status` subresource and RBAC.
- Operator SDK offers three approaches (Go, Ansible, Helm) trading off flexibility against ease of authoring — Helm-based Operators are simplest but can't express custom day-2 logic beyond templating.

## Common Mistakes

- Writing a reconcile function with side effects that aren't idempotent (e.g., blindly appending to a list, or unconditionally incrementing a counter) — breaks the moment reconcile runs more than once for the same state, which it always eventually will.
- Forgetting to add a finalizer for external (non-Kubernetes) resource cleanup, leaving orphaned cloud resources (e.g., an external database, a cloud load balancer) after the custom resource is deleted, because Kubernetes' built-in garbage collection only knows about objects inside the cluster.
- Writing custom cleanup logic that duplicates what `ownerReferences` already give you for free — reinventing cascading deletion instead of relying on Kubernetes' built-in garbage collector.
- Reacting to *why* a reconcile was triggered (treating it as edge-triggered) instead of always re-deriving full current vs. desired state — this reintroduces the exact fragility to missed events that level-triggered design is supposed to eliminate.
- Choosing a Helm-based Operator for a use case that actually needs genuine custom day-2 logic (e.g., orchestrated failover), then hitting a wall because Helm templating fundamentally can't express imperative operational steps.
