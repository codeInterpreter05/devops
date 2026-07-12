# Day 22 — K8s Architecture Deep Dive: Pod Scheduling Lifecycle

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

This is where the previous two files (control plane, worker node) come together into the answer to *the* classic Kubernetes interview question: "walk me through what happens when you run `kubectl apply -f pod.yaml`." Being able to narrate this end-to-end, naming which component does what and in what order, is one of the highest-signal things you can demonstrate in a DevOps interview — it proves you understand the architecture rather than having memorized a diagram.

## Step-by-step: `kubectl apply -f pod.yaml`

1. **`kubectl` (client-side)** reads the YAML, converts it to JSON, and figures out whether this is a create or an update by comparing against the object's `last-applied-configuration` annotation (this is the "apply" 3-way merge — vs. `kubectl create`, which is a dumb one-shot create with no merge logic).
2. **HTTPS request to `kube-apiserver`** — authenticated via your kubeconfig's client cert/token/OIDC token.
3. **Authentication** — the apiserver verifies *who* you are (cert CN, bearer token, OIDC claims).
4. **Authorization (RBAC)** — the apiserver checks *whether that identity is allowed* to do this verb (`create`/`update`) on this resource (`pods`) in this namespace. This is where a `Role`/`RoleBinding` denies or allows the request (see Day 26).
5. **Admission Controllers** run in two phases:
   - **Mutating admission webhooks** — can modify the object (e.g., Istio's sidecar injector adding a container, or a policy engine adding default labels).
   - **Validating admission webhooks** — can only accept or reject (e.g., Pod Security admission rejecting a privileged container, OPA/Gatekeeper enforcing "every pod must have a resource limit").
6. **Persisted to `etcd`** — the apiserver writes the Pod object (with `status.phase: Pending`, no `spec.nodeName` yet) as a key under `/registry/pods/<namespace>/<name>`.
7. **`kube-scheduler`'s watch fires** — it's been watching the apiserver for pods with no `nodeName`. It runs the Filter → Score pipeline (see file 1) and picks a node.
8. **Scheduler writes back to the apiserver** — a `Binding` object, effectively setting `pod.spec.nodeName = <chosen node>`. This write also goes to `etcd`. The scheduler itself never contacts the node directly.
9. **The kubelet on that node's watch fires** — it sees a new Pod bound to itself, and starts the actual work:
   - Pulls the container image if not already cached (via the CRI → containerd → registry).
   - Sets up the pod sandbox (a pause container that holds the shared network namespace).
   - Attaches any volumes (calling out to CSI drivers if needed — see Day 25).
   - Calls CRI `CreateContainer`/`StartContainer` for each container in the pod spec, in order, respecting `initContainers` (which must all complete first, sequentially) before the main containers start.
   - Runs `postStart` lifecycle hooks (fire-and-forget after the container starts).
10. **Kubelet reports status back to the apiserver** continuously — `Pending → ContainerCreating → Running`, plus probe results, which get written to `etcd` again.
11. **Endpoint controller** (in `kube-controller-manager`) notices the new Pod became `Ready` (passed its readiness probe) and matches a Service's selector, and adds it to that Service's `Endpoints`/`EndpointSlice` — which is what `kube-proxy` on every node picks up to update its routing rules.

The whole thing is a chain of **watch-triggered reconciliation loops**, not a single synchronous call — `kubectl apply` returns as soon as step 6 completes (object accepted and stored); everything from step 7 onward happens asynchronously, which is exactly why `kubectl apply` can return success immediately while the pod is still `Pending` seconds or minutes later.

```bash
# Watch this whole sequence happen in near-real-time
kubectl apply -f pod.yaml
kubectl get events --watch                 # see Scheduled -> Pulling -> Pulled -> Created -> Started, in order
kubectl get pod <name> -w                  # watch the phase transition
```

## etcd deep dive: what it actually stores and why it matters

Building on file 1's etcd introduction — every object you can `kubectl get` is a serialized (protobuf, by default) value at a deterministic key path:

```
/registry/pods/<namespace>/<name>
/registry/deployments/<namespace>/<name>
/registry/secrets/<namespace>/<name>
/registry/services/<namespace>/<name>
```

Two properties matter a lot operationally:

- **`etcd` is versioned via a monotonically increasing `resourceVersion`.** Every watch (from the scheduler, kubelets, controllers, and your own `kubectl get -w`) is really "give me everything from `resourceVersion` X onward." This is what makes optimistic concurrency work: if you `kubectl edit` a Deployment that someone else just changed, you get a `409 Conflict` because your request's `resourceVersion` is stale — not a lost update.
- **Secrets are stored in `etcd` as base64, not encrypted, by default.** Base64 is an *encoding*, not encryption — anyone with `etcdctl` access to the raw data (or an unencrypted etcd snapshot/backup) can trivially read every Secret in the cluster. Production clusters should enable **encryption at rest** (`EncryptionConfiguration` with a KMS provider) so the apiserver encrypts Secret values before writing them to etcd. On EKS, this is the "envelope encryption with a KMS key" checkbox you should always enable.

```bash
# Confirm whether Secrets are encrypted at rest (self-managed cluster, has etcdctl access)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret --print-value-only | head -c 100
# If this prints readable base64/plaintext-looking data -> not encrypted at rest.
# If it prints binary garbage prefixed with "k8s:enc:..." -> encryption at rest is active.
```

## Points to Remember

- The scheduling flow is: client → apiserver (authn → authz → admission) → etcd write (Pending, no node) → scheduler watch fires → Binding write (node assigned) → kubelet watch fires → CRI pulls image & starts containers → status reported back → Endpoint controller updates Service routing.
- `kubectl apply` returning "success" only means the object was validated and persisted — it says nothing about whether the pod is actually running yet. Always follow up with `kubectl get pods -w` or `kubectl describe pod`.
- Every component past the apiserver reacts to **watches**, not polling — this is why the system is both scalable (no polling storm) and eventually consistent (there's always a propagation delay).
- `resourceVersion` is what makes concurrent edits safe (optimistic concurrency, `409 Conflict` on stale writes) — it's not just a debugging artifact.
- Secrets are base64-encoded, not encrypted, in etcd by default — encryption at rest must be explicitly configured.

## Common Mistakes

- Saying `kubectl apply` "deploys the pod" — it only creates/updates the API object. The actual deployment is a chain of asynchronous reactions by the scheduler and kubelet.
- Assuming a `409 Conflict` on `kubectl apply`/`edit` means something is broken — it's optimistic concurrency control working as intended; re-fetch and reapply.
- Treating Secrets as "encrypted because they're in Kubernetes" — without explicit encryption-at-rest configuration, they're just base64 in etcd, readable by anyone with etcd/backup access.
- Forgetting `initContainers` run sequentially and must all succeed before any main container starts — a hanging init container (e.g., waiting on a dependency that's down) is a very common cause of a pod stuck in `Init:0/1` that people mistake for the main container being the problem.
- Not distinguishing "Pending because unscheduled" (check `kubectl describe pod` events for scheduler-side reasons: insufficient resources, no matching node, unsatisfied affinity) from "Pending because scheduled but image pull failing" (check kubelet-side events: `ImagePullBackOff`, registry auth failure) — these have completely different root causes and fixes.
