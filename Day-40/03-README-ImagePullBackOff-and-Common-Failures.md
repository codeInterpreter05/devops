# Day 40 — K8s Observability & Debugging: ImagePullBackOff & Other Common Failures

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

Beyond CrashLoopBackOff/OOMKilled (file 2), a handful of other pod failure states account for the vast majority of real-world Kubernetes incidents: `ImagePullBackOff`, `Pending` (scheduling failures), and `Init:Error`/failed init containers. Each has a distinct, fast diagnostic path once you recognize the pattern — the interview signal here is the same as file 2's: can you go from "the pod is broken" to "here is the specific, verifiable cause" quickly, using the right command for each symptom.

## `ImagePullBackOff` / `ErrImagePull`

```bash
kubectl describe pod myapp-x2j4k | grep -A10 Events
```

`ErrImagePull` is the first failed attempt; `ImagePullBackOff` is the same exponential-backoff waiting pattern as CrashLoopBackOff, but for image pulls specifically. The Events section almost always states the exact underlying reason in plain text. The common root causes, in roughly descending frequency:

1. **Typo in the image name or tag** — `myapp:v1.2.3` vs. the actually-pushed `myapp:v1.2.4`, or a registry path typo. Fastest check: `docker pull <exact image string>` from your own machine (assuming you have equivalent registry access) to confirm the image genuinely exists as named.
2. **Missing or incorrect `imagePullSecrets`** — private registries (ECR, private Docker Hub, GHCR) require the pod (or its ServiceAccount) to reference a `Secret` of type `kubernetes.io/dockerconfigjson` with valid credentials. Events will show `unauthorized` or `403 Forbidden` in this case, distinct from a plain "not found."
3. **Expired registry credentials** — especially common with **AWS ECR**, where auth tokens expire every **12 hours**. A cluster that's been fine for months can suddenly start failing image pulls if whatever mechanism refreshes that ECR login token (a CronJob, an operator, IRSA-based dynamic auth) breaks or isn't running — this is a very common real "it was working yesterday" incident.
4. **Network egress blocked** — a NetworkPolicy (see Day 41) or a node's security group/firewall blocking the node's outbound connection to the registry — Events typically show a timeout rather than an auth or not-found error, which is the tell that distinguishes this cause from the other three.
5. **Rate limiting** — Docker Hub's anonymous pull rate limits can cause intermittent `ImagePullBackOff` at scale (many nodes pulling the same public image simultaneously); the fix is usually authenticating pulls or mirroring images to a private registry/pull-through cache.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<user> --docker-password=<pass>
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```

## `Pending` — the pod never even got scheduled

A pod stuck in `Pending` (no container statuses at all yet) means the **scheduler** couldn't find a node to place it on. `describe` again is step one — the Events section will state the scheduling failure reason directly:

- **Insufficient resources** — `0/5 nodes are available: 5 Insufficient cpu` — every node's allocatable CPU/memory is already claimed by other pods' `requests`, even if actual usage is lower (the scheduler bin-packs based on `requests`, not live usage).
- **Node affinity/taint mismatches** — the pod requires a `nodeSelector`/`affinity` rule or **toleration** for a taint (e.g., `dedicated=gpu:NoSchedule`) that no available node satisfies.
- **PersistentVolumeClaim not bound** — a pod requesting a PVC that can't be satisfied (no matching PersistentVolume, storage class issue, or the underlying cloud volume is stuck in another AZ than any schedulable node) stays `Pending` indefinitely until the storage problem resolves.
- **Cluster autoscaler lag** — in a cluster with autoscaling, `Pending` can be entirely expected and transient while a new node boots — check `kubectl get nodes` and cluster-autoscaler logs before assuming something is actually broken.

## `Init:Error` / `Init:CrashLoopBackOff` — failing before the main container even starts

Init containers run sequentially, fully to completion, **before** any of the pod's main containers start. If an init container fails, the main application container **never even attempts to start** — this is a common source of confusion when someone checks `kubectl logs <pod>` (which defaults to the *first* container, often assumed to be the main app) and sees nothing relevant.

```bash
kubectl logs myapp-x2j4k -c <init-container-name>
kubectl describe pod myapp-x2j4k    # shows which specific init container is stuck/failing, and why
```

Common init container failure causes: waiting on a dependency that never becomes ready (a classic `wait-for-db` init container looping forever because the database Service has no ready endpoints), a misconfigured volume mount, or a permissions problem writing to a shared `emptyDir` volume that a later container also mounts.

## `CreateContainerConfigError` / `CreateContainerError`

Usually a **configuration reference problem** discovered at container-creation time, before the process even attempts to start: a `configMapKeyRef`/`secretKeyRef` pointing at a ConfigMap/Secret (or a specific key within one) that doesn't exist, or a volume mount referencing a missing resource. `describe`'s Events section names the exact missing reference directly — this class of failure is almost always fixable by comparing the pod spec's references against `kubectl get configmap`/`kubectl get secret` in the same namespace.

## A unified mental model for pod failure triage

| Symptom | First thing to check |
|---|---|
| `Pending`, no containers started | `describe` → scheduling Events (resources, affinity, taints, PVC) |
| `ImagePullBackOff`/`ErrImagePull` | `describe` → Events (typo, auth, network, rate limit) |
| `CreateContainerConfigError` | `describe` → Events (missing ConfigMap/Secret key reference) |
| `Init:Error`/stuck on init | `logs -c <init-container>` + `describe` (which init container, why) |
| `CrashLoopBackOff` | `logs --previous` + exit code (see file 2) |
| `Running` but app misbehaving | `logs`, `exec`/`kubectl debug`, check readiness probe results |

The consistent pattern across every single one of these: **`kubectl describe` first, always** — it's the fastest path to a specific, named reason in the vast majority of pod failures, and only after that do you reach for `logs`, `exec`, or deeper tools.

## Points to Remember

- `ImagePullBackOff` root causes cluster into four buckets: typo/nonexistent tag, missing/wrong `imagePullSecrets`, expired registry auth (ECR's 12-hour token expiry is a classic recurring one), or blocked network egress/rate limiting — the exact Events text usually tells you which.
- `Pending` means the scheduler couldn't place the pod at all — check resource requests vs. node allocatable capacity, taints/tolerations/affinity, and unbound PersistentVolumeClaims before assuming a deeper bug.
- Init containers run to completion sequentially before any main container starts; a stuck/failing init container means the main app container logs will show nothing, because it never started — check `logs -c <init-container-name>` instead.
- `CreateContainerConfigError` almost always means a referenced ConfigMap/Secret (or a specific key inside one) doesn't exist in that namespace — compare the pod spec's references against what actually exists.
- `kubectl describe` first, every time, regardless of which of these symptoms you're looking at — it is consistently the fastest way to get a specific, actionable reason.

## Common Mistakes

- Assuming `kubectl logs <pod>` with no `-c` flag shows the main application container's output, when the pod actually has init containers or multiple containers — you may be looking at the wrong container's (empty or irrelevant) logs entirely.
- Treating every `ImagePullBackOff` as "the image doesn't exist" and re-pushing/re-tagging images repeatedly, when the actual cause is expired registry credentials (especially ECR's 12-hour token lifetime) — wastes time rebuilding something that was never broken.
- Not distinguishing `Pending` (never scheduled) from `CrashLoopBackOff` (scheduled, started, but keeps exiting) — they require completely different diagnostic paths, and conflating them wastes time looking at logs for a pod that never even ran.
- Forgetting that PersistentVolumeClaims can silently keep a pod `Pending` indefinitely if the underlying cloud volume lives in a different availability zone than any schedulable node — a very common, easy-to-miss cross-AZ scheduling constraint.
- Assuming a `Pending` pod in an autoscaling cluster is necessarily broken, when it may simply be waiting on the cluster autoscaler to provision a new node — checking `kubectl get nodes`/autoscaler logs before escalating avoids false-alarm incidents.
