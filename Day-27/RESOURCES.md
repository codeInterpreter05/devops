# Day 27 — Resources: Autoscaling

## Primary (assigned)

- **KEDA docs** (keda.sh/docs) **+ Karpenter docs** (karpenter.sh/docs) — free, the assigned starting point. KEDA's docs cover the full `ScaledObject`/`TriggerAuthentication` model and the scaler catalog; Karpenter's docs cover `NodePool`/`EC2NodeClass` configuration and consolidation behavior in depth.

## Deepen your understanding

- **Kubernetes.io — Horizontal Pod Autoscaling** (kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) — the authoritative reference for HPA's `behavior` fields, metric types (`Resource`/`Pods`/`Object`/`External`), and the full autoscaling/v2 API.
- **Kubernetes.io — Vertical Pod Autoscaler** (github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) — the project's own README covers the Recommender/Updater/Admission Controller split and `updateMode` options clearly.
- **AWS — Cluster Autoscaler vs Karpenter comparison** (aws.github.io/aws-eks-best-practices/karpenter/) — EKS Best Practices Guide's direct side-by-side, written specifically to help teams choose between the two.
- **KEDA scalers catalog** (keda.sh/docs/latest/scalers/) — browse the full list of 60+ built-in event sources to get a sense of the breadth beyond just SQS/RabbitMQ.

## Reference / lookup

- `kubectl explain horizontalpodautoscaler.spec` / `kubectl explain scaledobject.spec` (once KEDA CRDs are installed) — always in sync with your cluster's installed API version.
- **Kubernetes API Reference** (kubernetes.io/docs/reference/generated/kubernetes-api) — exact semantics of HPA `behavior`, `stabilizationWindowSeconds`, and scaling policies.

## Practice

- **KEDA samples repo** (github.com/kedacore/samples) — ready-to-run examples for SQS, RabbitMQ, Kafka, and cron-based scaling, good for practicing beyond this day's RabbitMQ lab.
- Reuse any internship-era RabbitMQ/queue-consumer experience directly in the Day 27 lab — mapping a familiar real workload onto KEDA's `ScaledObject` model is the fastest way to make this day's concepts concrete rather than abstract.
