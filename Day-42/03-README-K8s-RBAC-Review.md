# Day 42 — Review: Kubernetes RBAC

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Review | **Flag:** 📌 Review/consolidation day

## Brief

RBAC (Role-Based Access Control) is Kubernetes' authorization layer — it answers "now that we know *who* you are (authentication), what are you actually allowed to *do*." It's the natural companion to this week's NetworkPolicy material (which controls network-layer access) and Vault/secrets material (which controls credential access) — together these three form the core of "least privilege" as actually practiced on a real Kubernetes platform. This review file consolidates RBAC for a live exercise: given a scenario, write the exact `Role`/`ClusterRole` and binding needed, and reason about why a broader grant would be wrong.

## The four RBAC objects

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Single namespace | Defines a set of permissions (verbs on resources) within one namespace |
| `ClusterRole` | Cluster-wide | Defines permissions either cluster-wide, OR reusable across namespaces via a `RoleBinding` |
| `RoleBinding` | Single namespace | Grants a `Role` (or a `ClusterRole`, scoped down to this namespace) to specific subjects |
| `ClusterRoleBinding` | Cluster-wide | Grants a `ClusterRole` cluster-wide, across every namespace, to specific subjects |

The commonly-missed nuance: a **`ClusterRole` bound via a namespaced `RoleBinding`** grants those permissions **only within that RoleBinding's namespace**, even though the `ClusterRole` itself is a cluster-scoped object. This pattern is actually the standard way to define a reusable permission set once (e.g., a `ClusterRole` called `pod-reader`) and grant it narrowly, namespace by namespace, via separate `RoleBinding`s — versus a `ClusterRoleBinding`, which would grant it everywhere at once.

## Example — a Role scoped to reading pods/logs in one namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-log-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: oncall-pod-log-reader
  namespace: production
subjects:
  - kind: User
    name: oncall-engineer@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-log-reader
  apiGroup: rbac.authorization.k8s.io
```

Reading this precisely: `apiGroups: [""]` refers to the **core API group** (pods, services, configmaps, secrets, nodes — anything with no group prefix in the API); `pods/log` is a **subresource**, meaning read access to pod logs is granted *separately* from read access to the pod object itself — granting `get` on `pods` does **not** automatically grant `get` on `pods/log`, a very commonly-missed detail when troubleshooting "I have the Role but `kubectl logs` still says forbidden."

## ServiceAccounts — RBAC for workloads, not just humans

Every pod runs as a **ServiceAccount** (defaulting to `default` in its namespace if none is specified) — RBAC applies identically whether the "subject" is a human `User`, a `Group`, or a `ServiceAccount`. This is exactly the mechanism underlying Day 38's Vault Kubernetes auth and Day 39's ESO IRSA-based auth: the pod's ServiceAccount identity is what external systems (and Kubernetes' own API server) use to authorize what that pod can do.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-configmap-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# pod spec
spec:
  serviceAccountName: myapp-sa
  containers: [...]
```

**Never leave a pod on the `default` ServiceAccount if it needs any specific API permissions** — the `default` ServiceAccount typically has no bound permissions out of the box in most clusters, but explicitly creating a purpose-named ServiceAccount per application is the standard practice for auditability (you can tell exactly which workload a given API call came from) and for least-privilege scoping (each app gets only the RBAC it specifically needs, not a shared grant meant for something else).

## Diagnosing an RBAC denial

```bash
kubectl auth can-i get pods --as=system:serviceaccount:production:myapp-sa -n production
kubectl auth can-i list secrets --as=oncall-engineer@example.com -n production
kubectl auth can-i '*' '*' --as=admin-user           # check for cluster-admin-equivalent access
```

`kubectl auth can-i` is the single fastest tool for RBAC troubleshooting — it directly answers "would this specific subject be allowed to do this specific verb on this specific resource," without needing to manually trace through every Role/ClusterRole/Binding by hand. The actual API server error message when access is denied (`... is forbidden: User "..." cannot get resource "pods" in API group "" in the namespace "production"`) also directly names the missing verb/resource/namespace combination — read it literally; it's telling you exactly what to add, not a generic permissions error.

## Points to Remember

- `Role`/`RoleBinding` are namespace-scoped; `ClusterRole`/`ClusterRoleBinding` are cluster-scoped — but a `ClusterRole` can be bound narrowly via a namespaced `RoleBinding`, which is the standard pattern for defining reusable permission sets once and granting them selectively.
- Subresources (like `pods/log`, `pods/exec`) require their own explicit `resources` entry — permission on the parent resource does not automatically extend to its subresources.
- Every pod authenticates to the API server as a ServiceAccount; give each application its own dedicated ServiceAccount rather than relying on `default`, for both auditability and least-privilege scoping.
- `kubectl auth can-i --as=<subject>` is the fastest, most direct way to verify or debug RBAC permissions — always reach for it before manually tracing Role/Binding YAML by hand.
- Forbidden error messages from the API server name the exact missing verb, resource, and namespace — read them literally as the fix instructions, not as a vague permissions failure.

## Common Mistakes

- Granting `get`/`list` on `pods` and assuming that also covers pod logs or exec access — `pods/log` and `pods/exec` are separate subresources requiring their own explicit RBAC rule.
- Using a `ClusterRoleBinding` when a namespaced `RoleBinding` (referencing a `ClusterRole`) would have been sufficient — granting cluster-wide access when only one namespace's worth of access was actually needed, a classic least-privilege violation.
- Leaving workloads on the `default` ServiceAccount and gradually bolting broad permissions onto it over time (because it's shared across every unlabeled pod in the namespace) instead of giving each application its own narrowly-scoped ServiceAccount from the start.
- Granting wildcard permissions (`resources: ["*"]`, `verbs: ["*"]`) "to unblock things quickly" during initial development and never tightening them before shipping to production — a very common source of an overly permissive real-world cluster.
- Debugging an RBAC denial by re-reading YAML manually instead of immediately running `kubectl auth can-i --as=<subject>`, which answers the exact question directly and far faster than manual tracing.
