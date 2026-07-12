# Day 23 — Resources: Workloads — Deployments & StatefulSets

## Primary (assigned)

- **Kubernetes.io — StatefulSets** (kubernetes.io/docs/concepts/workloads/controllers/statefulset/) — free, the assigned starting point. Precisely documents ordering guarantees, identity, and storage semantics referenced throughout this day's notes.

## Deepen your understanding

- **Kubernetes.io — Deployments** (kubernetes.io/docs/concepts/workloads/controllers/deployment/) — the official reference for rollout strategy fields (`maxUnavailable`, `maxSurge`, `revisionHistoryLimit`) with worked examples.
- **Kubernetes.io — DaemonSet** (kubernetes.io/docs/concepts/workloads/controllers/daemonset/) — covers taints/tolerations interactions and update strategies (`RollingUpdate` vs `OnDelete`) in depth.
- **Kubernetes.io — Container Lifecycle Hooks** (kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) — the authoritative timing guarantees (and non-guarantees) for `postStart`/`preStop`.
- **"Kubernetes Patterns" by Bilgin Ibryam & Roland Huß** (O'Reilly) — the chapters on StatefulSets, DaemonSets, and graceful termination patterns are excellent for connecting the mechanics to real architecture decisions.

## Reference / lookup

- `kubectl explain statefulset.spec` / `kubectl explain deployment.spec.strategy` — always in sync with your cluster's actual API version.
- **Kubernetes API Reference** (kubernetes.io/docs/reference/generated/kubernetes-api) — exact field semantics for `volumeClaimTemplates`, `podManagementPolicy`, lifecycle hooks.

## Practice

- **KillerCoda Kubernetes scenarios** (killercoda.com/kubernetes) — free browser-based labs; several are dedicated specifically to rolling updates/rollbacks and StatefulSet ordering.
- Redeploy the Day 23 lab's Redis StatefulSet with `podManagementPolicy: Parallel` and compare scale-up time/ordering behavior against the default `OrderedReady` — a good self-check that you actually understand the distinction rather than just having read about it.
