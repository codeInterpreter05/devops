# Day 26 — RBAC & Security: RBAC Fundamentals

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Every request that reaches `kube-apiserver` — whether from a human running `kubectl`, a CI pipeline, or a pod calling the API from inside the cluster — goes through authentication (who are you?) and then **authorization** (are you allowed to do this?). RBAC (Role-Based Access Control) is how Kubernetes answers the authorization question, and it's the mechanism behind nearly every "permission denied" incident and every real security boundary in a multi-tenant cluster. Interviewers ask about RBAC constantly because it's where "I've used Kubernetes" and "I've operated Kubernetes safely in a shared environment" diverge sharply.

This day is split into three files:

1. **This file** — ServiceAccounts, Roles/ClusterRoles, RoleBindings/ClusterRoleBindings.
2. **[02-README-IRSA.md](02-README-IRSA.md)** — IRSA (IAM Roles for Service Accounts) on EKS.
3. **[03-README-Pod-Security-Admission.md](03-README-Pod-Security-Admission.md)** — Pod Security Standards and admission webhooks.

## Identity first: who is making the request?

RBAC only matters once you know *who* is asking. Two broad identity categories:

- **Users/Groups** — humans or external systems, authenticated via client certs, OIDC tokens (common for SSO integration), or a cloud IAM identity (e.g., `aws-iam-authenticator` mapping an IAM principal to a Kubernetes user on EKS). Kubernetes has **no built-in User object** — user identity is entirely externally managed; Kubernetes just trusts whatever identity the authentication layer hands it.
- **ServiceAccounts** — an actual Kubernetes API object (`kubectl get serviceaccounts`), representing the identity of a **process running inside the cluster** (typically a Pod). Every namespace has a `default` ServiceAccount automatically, and every Pod runs as some ServiceAccount whether you set one explicitly or not.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-reader
  namespace: monitoring
```

```yaml
apiVersion: v1
kind: Pod
metadata: { name: metrics-agent }
spec:
  serviceAccountName: monitoring-reader   # explicit — omit and it silently uses "default"
  containers:
    - name: agent
      image: my-metrics-agent
```

A Pod authenticates to the apiserver using a **projected, auto-rotated token** for its ServiceAccount, mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` by default — this is exactly what tools like `kubectl` running *inside* a pod, or client libraries using in-cluster config, actually use to talk back to the API.

## Roles and ClusterRoles: what actions are allowed

A `Role` (namespaced) or `ClusterRole` (cluster-scoped) is purely a list of permissions — verbs on resources — and grants nothing on its own until bound to an identity.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: monitoring
rules:
  - apiGroups: [""]                 # "" = core API group (pods, services, configmaps, secrets, ...)
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]           # Nodes are cluster-scoped -> only a ClusterRole can grant access to them at all
```

**Key distinction:** a `Role` can only grant access to namespaced resources within its own namespace. A `ClusterRole` can grant access to cluster-scoped resources (Nodes, PersistentVolumes, Namespaces themselves) **or** be reused across multiple namespaces via separate `RoleBinding`s pointing at the same `ClusterRole` (a common pattern to avoid duplicating an identical Role in every namespace).

## RoleBindings and ClusterRoleBindings: connecting identity to permission

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-reader-binding
  namespace: monitoring
subjects:
  - kind: ServiceAccount
    name: monitoring-reader
    namespace: monitoring
roleRef:
  kind: Role                # can also reference a ClusterRole, scoped down to this namespace only
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: Group
    name: sre-team              # e.g., an OIDC group claim
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

The four-object model, summarized: **`Role`/`ClusterRole`** = *what's allowed*; **`RoleBinding`/`ClusterRoleBinding`** = *who gets it, and in what scope*. A `RoleBinding` referencing a `ClusterRole` is a deliberately supported, common pattern — it grants that `ClusterRole`'s permissions **only within the RoleBinding's own namespace**, letting you define one reusable `ClusterRole` (e.g., `view`, `edit`, `admin` — Kubernetes ships these as built-in aggregated ClusterRoles) and bind it per-namespace without duplicating rule definitions.

## Checking and debugging RBAC

```bash
kubectl auth can-i create pods --namespace=monitoring
kubectl auth can-i create pods --as=system:serviceaccount:monitoring:monitoring-reader
kubectl auth can-i '*' '*' --as=jane@example.com          # check for cluster-admin-equivalent access
kubectl get rolebindings,clusterrolebindings -A -o wide | grep monitoring-reader
kubectl describe role pod-reader -n monitoring
```

`kubectl auth can-i` is the single most useful debugging command for "why am I getting Forbidden" — it evaluates the exact same authorization path the apiserver uses, without needing to actually attempt (and fail) the real request.

## Points to Remember

- Authorization only happens after authentication — RBAC decides *what* an already-identified subject can do, it says nothing about *proving who they are*.
- `Role`/`ClusterRole` define permissions; `RoleBinding`/`ClusterRoleBinding` grant them to a subject (User, Group, or ServiceAccount) — permissions and grants are always two separate objects, never combined.
- A `Role` can only touch namespaced resources in its own namespace; only a `ClusterRole` can grant access to cluster-scoped resources (Nodes, PVs, Namespaces) at all.
- A `RoleBinding` can reference a `ClusterRole` to scope its (potentially reusable) permissions down to just one namespace — this is how Kubernetes avoids requiring a separate `Role` definition per namespace for common permission sets.
- Every Pod runs as a ServiceAccount (explicitly set or the namespace's `default`), and that ServiceAccount's token is what the Pod uses to authenticate any calls it makes back to the Kubernetes API.

## Common Mistakes

- Granting `cluster-admin` (via `ClusterRoleBinding` to the built-in `cluster-admin` ClusterRole) to a ServiceAccount "to unblock things quickly" and never revisiting it — this is the single most common RBAC over-privilege incident in real clusters.
- Forgetting that Pods not given an explicit `serviceAccountName` silently run as the namespace's `default` ServiceAccount — if that `default` SA has been bound to broad permissions (intentionally or by accident via a wildcard binding), every unlabeled pod in that namespace inherits them.
- Creating a `Role` and expecting it to restrict access to cluster-scoped resources like Nodes — it structurally cannot; only a `ClusterRole` can even reference those resource types, regardless of the verbs listed.
- Debugging "permission denied" by staring at YAML instead of running `kubectl auth can-i --as=<identity>` first — this command directly answers the question instead of requiring you to manually trace bindings and role rules.
- Binding a `ClusterRole` via `ClusterRoleBinding` when a namespace-scoped `RoleBinding` (referencing that same `ClusterRole`) would have granted the narrower, actually-intended access — accidentally granting cluster-wide permissions when only one namespace's worth was needed.
