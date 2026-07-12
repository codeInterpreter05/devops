# Day 22 — Resources: K8s Architecture Deep Dive

## Primary (assigned)

- **Kubernetes.io Concepts — Architecture** (kubernetes.io/docs/concepts/architecture/) — free, the assigned starting point. Covers the control plane and node components with the official diagrams you'll be expected to reproduce from memory.

## Deepen your understanding

- **"The Kubernetes Book" by Nigel Poulton** — the architecture chapters give the clearest plain-English mental model of the control plane/node split available anywhere; worth the read even in preview/sample form.
- **Kubernetes the Hard Way** (github.com/kelseyhightower/kubernetes-the-hard-way) — free. You don't have to finish the whole thing, but manually standing up `etcd`, the apiserver, and the kubelet by hand (even on local VMs) teaches you more about the architecture than any diagram will.
- **etcd documentation — Understanding etcd** (etcd.io/docs) — for the Raft/quorum details referenced in file 1; useful once you want to go beyond "it's a key-value store."

## Reference / lookup

- **Kubernetes API Reference** (kubernetes.io/docs/reference/generated/kubernetes-api) — the definitive spec for every object field; use when you need to know exactly what a field does.
- `kubectl explain <resource>` — e.g. `kubectl explain pod.spec` — on-box, always in sync with your actual cluster's API version, no internet needed.

## Practice

- **KillerCoda / Katacoda-style free Kubernetes scenarios** (killercoda.com/kubernetes) — free, browser-based Kubernetes clusters for exactly this kind of "trace what happens" exploration without needing local resources.
- **minikube + k9s** (this day's lab) — the fastest local loop for repeatedly breaking and observing the control plane without any cloud cost.
