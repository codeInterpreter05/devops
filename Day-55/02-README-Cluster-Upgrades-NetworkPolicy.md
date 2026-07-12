# Day 55 — CKA Exam Prep II: Cluster Upgrades & NetworkPolicy

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Certifications | **Flag:** 📌

## Brief

Cluster upgrades and NetworkPolicy are two more CKA task categories that show up in almost every exam attempt, and both are areas where people who've only used managed Kubernetes (EKS/GKE/AKS click-to-upgrade) have never actually done the work by hand. The exam (and real self-managed clusters, on-prem or kubeadm-based) expects you to know the manual, ordered `kubeadm upgrade` sequence node by node, and to reason about NetworkPolicy as the whitelist-only, additive firewall model it actually is — not just recite `kubectl apply -f policy.yaml`.

## Cluster upgrades with `kubeadm` — the sequence, and why order matters

Kubernetes supports upgrading only **one minor version at a time** (e.g., 1.28 → 1.29, never 1.28 → 1.30 directly) because the API machinery, feature gates, and deprecations are only tested and guaranteed compatible one minor version apart. Skipping versions can silently break CRDs, webhooks, or controller behavior that assumed intermediate deprecation cycles happened.

The upgrade order is always: **control plane first, one node at a time if multiple control-plane nodes exist, then worker nodes one at a time** — never upgrade workers ahead of the control plane, since newer worker kubelets can (in some version skews) refuse to register against an older API server, and the API server must remain the newest component in the cluster at all times per Kubernetes' version-skew policy.

On the **first** control-plane node:

```bash
# 1. Find what versions are available
apt update && apt-cache madison kubeadm

# 2. Upgrade kubeadm itself first (the tool, not the cluster)
apt-mark unhold kubeadm && apt install -y kubeadm=1.29.0-1.1 && apt-mark hold kubeadm

# 3. Check what the upgrade will actually do (dry-run-ish plan)
kubeadm upgrade plan

# 4. Apply it (control plane components: API server, scheduler, controller-manager, etcd)
kubeadm upgrade apply v1.29.0
```

On **additional** control-plane nodes (if any), skip `apply` and use:

```bash
kubeadm upgrade node
```

Then, **on every node** (control-plane and worker), upgrade kubelet and kubectl, and drain first:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# on the node itself:
apt-mark unhold kubelet kubectl && apt install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1 && apt-mark hold kubelet kubectl
systemctl daemon-reexec && systemctl restart kubelet

kubectl uncordon <node-name>
```

**Why drain before upgrading kubelet:** the kubelet restart briefly stops reporting node status and can drop running pod supervision; draining first evicts workloads gracefully to other nodes rather than letting them get stuck in an ambiguous state during the restart window. `--ignore-daemonsets` is required because DaemonSet pods are meant to run on every node including this one and the scheduler won't (and shouldn't) reschedule them elsewhere — attempting to evict them without this flag just errors out.

## NetworkPolicy — whitelist-only, and only if a CNI enforces it

The single most important mental model: **NetworkPolicy is default-allow until you create the first policy selecting a pod, then that pod becomes default-deny for whatever traffic direction the policy governs, and only explicitly allowed traffic gets through.** There is no "deny" rule type — policies are purely additive allow-lists. To deny everything, you write a policy with an empty `podSelector: {}` and no `ingress`/`egress` rules for that direction.

```yaml
# deny all ingress to pods labeled app=backend in namespace 'prod'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  # no ingress rules = nothing is allowed in
```

```yaml
# then allow only from pods labeled app=frontend, on port 8080
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Both policies apply to the same pod selector — NetworkPolicy rules across multiple objects targeting the same pods are **additive (OR'd together)**, not a chain evaluated in order. This is the opposite mental model from, say, iptables rule ordering or a traditional firewall's first-match-wins list.

**The exam-critical caveat: NetworkPolicy objects are inert without a CNI plugin that enforces them.** Flannel, by default, does **not** enforce NetworkPolicy at all — you can `kubectl apply` a deny-all policy and traffic will flow right through, because nothing in the data plane is reading these objects. Calico, Cilium, and Weave Net do enforce them. If a lab task says "traffic should be blocked" and it isn't, the very first thing to check is `kubectl get pods -n kube-system` for which CNI is running — the policy YAML being "correct" is necessary but not sufficient.

## Points to Remember

- Upgrade one minor version at a time; control plane before workers; `kubeadm upgrade apply` on the first control-plane node, `kubeadm upgrade node` on the rest.
- `kubelet`/`kubectl` are upgraded per-node manually (`apt`/`yum`), separate from `kubeadm upgrade` which only touches control-plane static pod manifests.
- Always `drain` before touching kubelet on a node, `uncordon` after — this is a standard, testable sequence.
- NetworkPolicy is default-allow cluster-wide until the first policy selects a pod; after that, it's default-deny in the direction(s) that policy declares, and only additively allowed traffic gets through.
- Multiple NetworkPolicies on the same pods are OR'd, not layered/ordered like a traditional firewall.
- NetworkPolicy does nothing without an enforcing CNI (Calico/Cilium/Weave yes, plain Flannel no) — verify the CNI before debugging the YAML.

## Common Mistakes

- Attempting to skip a minor version during upgrade (1.27 → 1.29) and hitting cryptic API incompatibility errors mid-upgrade.
- Upgrading kubelet on worker nodes before the control plane, then having those kubelets refused/degraded due to the API server being older than the version-skew policy allows.
- Forgetting `--ignore-daemonsets` on `kubectl drain` and having the command hang/error waiting to evict pods it structurally cannot evict.
- Writing a "deny all" NetworkPolicy with `spec.ingress: []` (empty list, meaning explicitly no rules — correct) versus omitting `ingress` entirely under a policyType that expects it — subtly different YAML, easy to get backward under time pressure.
- Debugging a NetworkPolicy that "isn't working" for 20 minutes without first checking whether the cluster's CNI plugin (e.g., Flannel) even enforces NetworkPolicy at all.
- Assuming NetworkPolicies combine as an ordered allow/deny chain rather than understanding they're purely additive per selected pod.
