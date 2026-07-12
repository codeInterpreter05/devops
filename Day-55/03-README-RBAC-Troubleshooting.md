# Day 55 — CKA Exam Prep II: RBAC & Cluster Troubleshooting

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Certifications | **Flag:** 📌

## Brief

RBAC task patterns and cluster-level debugging round out the heavily-tested CKA categories. RBAC is tested because it's the mechanism that answers "who can do what" in every real cluster, and getting the Role/RoleBinding vs. ClusterRole/ClusterRoleBinding distinction wrong is one of the most common production security incidents (over-permissioned service accounts). Debugging is tested because the exam is explicitly designed to simulate "something is broken, find out why, using only the tools available on the box" — which is exactly the job.

## RBAC — the four objects and how they compose

Kubernetes RBAC has exactly four object kinds, and they compose in a fixed pattern:

- **Role** — a set of permissions (verbs on resources), scoped to **one namespace**.
- **ClusterRole** — the same shape as a Role, but not namespaced — it can apply cluster-wide *or* be reused inside a specific namespace via a RoleBinding (this second use is a common exam trick: ClusterRoles aren't only for cluster-wide access).
- **RoleBinding** — grants a Role (or a ClusterRole) to a subject (User/Group/ServiceAccount), scoped to **one namespace**.
- **ClusterRoleBinding** — grants a ClusterRole to a subject **cluster-wide**, across all namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

The trick worth internalizing: **binding a ClusterRole via a namespaced RoleBinding grants those permissions only inside that RoleBinding's namespace** — this is the standard pattern for reusing a common ClusterRole (like the built-in `view`, `edit`, `admin` ClusterRoles) across many namespaces without duplicating the Role YAML in each one.

```bash
# grant the built-in 'view' ClusterRole to a ServiceAccount, but only in namespace 'dev'
kubectl create rolebinding dev-viewers \
  --clusterrole=view \
  --serviceaccount=dev:ci-deployer \
  --namespace=dev
```

## Checking effective permissions fast

Never eyeball YAML to figure out what a subject can do — `kubectl auth can-i` asks the API server directly, using the real authorization chain:

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:dev:ci-deployer -n dev
kubectl auth can-i '*' '*' --as=jane                 # check for cluster-admin-like access
kubectl auth can-i --list --as=jane -n dev            # list everything jane can do in dev
```

`--as` impersonates a subject for the check (requires impersonation permission yourself, which cluster-admin has by default). This is the fastest way to verify an RBAC task actually did what you intended — write the Role/Binding, then immediately confirm with `can-i` instead of assuming the YAML was correct.

## ServiceAccounts — the identity workloads actually use

Every pod runs as a ServiceAccount (the `default` one in its namespace, if none specified) — and that ServiceAccount's token is what any code inside the pod uses to talk to the API server. This is the identity RBAC actually binds to for in-cluster workloads (as opposed to human `User`/`Group` subjects, which Kubernetes doesn't manage directly — those come from your external auth provider, e.g., a cloud IAM integration or OIDC).

```bash
kubectl create serviceaccount ci-deployer -n dev
kubectl get pod mypod -n dev -o jsonpath='{.spec.serviceAccountName}'
```

Modern Kubernetes (1.24+) no longer auto-creates a long-lived Secret token for every ServiceAccount — tokens are now short-lived and projected into the pod via the `TokenRequest` API by default, mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. If you need a durable token for out-of-cluster use, you request one explicitly:

```bash
kubectl create token ci-deployer -n dev --duration=1h
```

## Cluster-level debugging — the systematic path, not guesswork

The exam (and production incidents) reward a **layered** debugging approach rather than randomly poking things:

1. **Cluster health first:** `kubectl get nodes` (are all nodes `Ready`?), `kubectl get pods -n kube-system` (are control-plane static pods and CNI pods healthy?).
2. **Static pod logs live on the node's filesystem as container logs, not `kubectl logs` if the API server itself is down:** `crictl ps -a`, `crictl logs <container-id>` work even when the API server is unreachable, because `crictl` talks directly to the container runtime (containerd/CRI-O) via CRI, bypassing Kubernetes entirely.
3. **kubelet issues:** `systemctl status kubelet`, `journalctl -u kubelet -f` — the kubelet is a systemd-managed service (not a static pod), so its logs live in journald, not in `/var/log/pods`.
4. **API server / scheduler / controller-manager issues:** these run as static pods themselves — check `/etc/kubernetes/manifests/*.yaml` for typos in a recently-edited manifest (a classic exam-injected fault), and `crictl logs` on their container IDs.
5. **Application-level:** `kubectl describe pod` (check the Events section at the bottom first — it's the fastest signal for scheduling failures, image pull errors, and failed probes), then `kubectl logs`, then `kubectl exec` to poke inside.

**Why `crictl` matters specifically for the exam:** a common injected fault is "the API server won't start" (e.g., a bad flag was added to its static pod manifest). `kubectl` is useless here because there's no API server to talk to yet — you must use `crictl ps -a` and `crictl logs` against the *containers* directly to read the API server's crash output and find the bad config.

## Points to Remember

- Role/RoleBinding = namespaced; ClusterRole/ClusterRoleBinding = cluster-wide — but a ClusterRole can be bound namespace-scoped via a regular RoleBinding (very common, very testable pattern).
- `kubectl auth can-i --as=<subject> --list -n <ns>` is the fast, authoritative way to verify RBAC — don't hand-trace YAML.
- Pods authenticate to the API server as their ServiceAccount; tokens are short-lived and projected by default since 1.24, request explicit tokens with `kubectl create token`.
- When the API server itself might be down, `kubectl` is useless — drop to `crictl ps -a` / `crictl logs` (talks to the container runtime directly via CRI) and `journalctl -u kubelet` for the kubelet itself.
- `kubectl describe pod`'s Events section is almost always the fastest lead in an app-level debugging task — check it before diving into logs.

## Common Mistakes

- Creating a ClusterRoleBinding when a namespace-scoped RoleBinding (referencing a ClusterRole) was the actually-intended, least-privilege answer — accidentally granting cluster-wide access.
- Assuming a Role can reference resources across namespaces — it can't; cross-namespace access always requires a ClusterRole (bound however you like).
- Forgetting that RBAC is deny-by-default and additive: multiple RoleBindings/ClusterRoleBindings targeting a subject are unioned, there's no explicit "deny" rule type, mirroring the same additive logic as NetworkPolicy.
- Trying `kubectl logs`/`kubectl get` to debug a broken control plane when the API server itself is the thing that's down — wasting exam time before realizing `crictl` is the only tool that works in that state.
- Editing a static pod manifest and not noticing a YAML indentation/typo error, then wondering why the pod never comes back — `crictl logs` on the crash-looping container almost always shows the exact parse error.
- Confusing kubelet's logging (journald, since it's a systemd unit) with static pod container logging (via the container runtime) — looking in the wrong place for each.
