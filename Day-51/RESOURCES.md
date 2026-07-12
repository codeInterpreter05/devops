# Day 51 — Resources: K8s Upgrades & Cluster Management

## Primary (assigned)

- **AWS EKS upgrade documentation** (docs.aws.amazon.com/eks — "Updating a cluster" and "Updating a managed node group") — the assigned starting point; covers the exact control-plane-then-nodes workflow and version-skew rules specific to EKS.

## Deepen your understanding

- **Kubernetes docs: Version Skew Policy** — the authoritative source on how far kubelets/kube-proxy/kubectl can lag behind the API server, and why upgrade ordering is mandatory rather than a best practice.
- **Kubernetes docs: Deprecated API Migration Guide** — the canonical, version-by-version list of every removed/deprecated API and its replacement; the reference `pluto`'s findings map back to.
- **Kubernetes docs: Disruptions** (concepts/workloads/pods/disruptions) — the official explanation of voluntary vs. involuntary disruptions and how PodDisruptionBudgets fit into that model.
- **Velero documentation** (velero.io/docs) — install guides per cloud provider, backup/restore concepts, and the difference between snapshot-based and file-system (Restic/Kopia) backup for persistent volumes.

## Reference / lookup

- `kubectl explain poddisruptionbudget.spec` — field-level reference straight from your cluster.
- **Fairwinds `pluto` GitHub repo** (github.com/FairwindsOps/pluto) — full CLI reference and the list of Kubernetes versions/APIs it tracks.
- **AWS EKS best practices guide — Cluster Upgrades** (aws.github.io/aws-eks-best-practices) — AWS's own opinionated runbook for upgrade sequencing in real production accounts.

## Practice

- **minikube** — the most practical way to rehearse a real minor-version upgrade (as in today's lab) without touching a real cluster or incurring cloud cost.
- **killer.sh / KodeKloud Kubernetes playgrounds** — scenario-based practice for cordon/drain/PDB interactions under time pressure, useful prep for both real incidents and CKA-style exam questions later this week.
