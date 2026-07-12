# Day 46 — Resources: K8s Operators & CRDs

## Primary (assigned)

- **Operator SDK documentation** (sdk.operatorframework.io) — free, the assigned starting point. Covers scaffolding, the Go/Ansible/Helm authoring approaches, and the underlying controller-runtime concepts directly.

## Deepen your understanding

- **"Custom Resources" (Kubernetes docs)** — the authoritative reference on CRD schema, versioning, and conversion webhooks.
- **CoreOS "Introducing Operators" blog post** (the original 2016 post that coined the term) — short and still the clearest one-paragraph framing of *why* the pattern exists.
- **controller-runtime godoc / book** (book.kubebuilder.io) — the Kubebuilder book is the best deep-dive into reconciliation loop mechanics, owner references, and finalizers with real Go code.
- **CloudNativePG documentation** (cloudnative-pg.io/documentation) — directly documents the failover, backup/PITR, and connection-pooling mechanics covered in this day's third file.
- **Strimzi documentation** (strimzi.io/docs) — covers the full CRD family (`Kafka`, `KafkaTopic`, `KafkaUser`, `KafkaConnect`) and Cruise Control integration in depth.

## Reference / lookup

- `kubectl explain <kind>.spec --api-version=<group>/<version>` — inline schema reference for any installed CRD, straight from your own cluster.
- **OperatorHub.io** — searchable catalog of published Operators; check here before deciding to build your own for any well-known piece of software.

## Practice

- **CloudNativePG "Quickstart" guide** — the fastest path to reproducing this day's lab if you want a second pass with more configuration options (backup, pooling) than the lab covers.
- **Strimzi "Quick Start"** (strimzi.io/quickstarts) — a Kafka-specific hands-on equivalent, useful for extending today's lab beyond Postgres to compare how a second real Operator's CRDs and reconciliation behave.
