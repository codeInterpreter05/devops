# Day 50 — Resources: Probes, Resources & QoS

## Primary (assigned)

- **Kubernetes docs: Managing Resources for Containers** (kubernetes.io/docs/concepts/configuration/manage-resources-containers) — the assigned starting point; covers requests/limits mechanics directly from the source.

## Deepen your understanding

- **Kubernetes docs: Configure Liveness, Readiness and Startup Probes** — the canonical reference for every probe field (`initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold`) with worked YAML examples for each probe mechanism (`exec`, `tcpSocket`, `httpGet`, `grpc`).
- **Kubernetes docs: Pod Quality of Service Classes** — the exact rules for how Guaranteed/Burstable/BestEffort are derived, straight from the spec, useful for settling any ambiguity about edge cases (init containers, multi-container pods).
- **Kubernetes docs: Node-pressure Eviction** — explains exactly how the kubelet decides which pods to evict under memory/disk pressure and in what order, which is the mechanism underlying "QoS determines eviction priority."
- **"Kubernetes Resource Limits" — Robusta.dev / other SRE blog deep dives on OOMKilled debugging** — practical, incident-shaped walkthroughs of diagnosing exit code 137 in real clusters, complementing the more abstract official docs.

## Reference / lookup

- `kubectl explain pod.spec.containers.livenessProbe` / `kubectl explain pod.spec.containers.resources` — authoritative field-level docs straight from your cluster's API, no internet needed.
- **Kubernetes docs: Limit Ranges** and **Resource Quotas** — full field reference for every constraint type each object supports (including object-count quotas, storage quotas, and scope selectors).

## Practice

- **Killercoda / Katacoda-style free Kubernetes playgrounds** — spin up a throwaway cluster in the browser to run today's lab (OOM triggering, probe toggling) without needing local infra.
- **minikube or kind locally** — for repeatable, disposable experimentation with `stress` containers and probe exec files, exactly as used in today's lab.
