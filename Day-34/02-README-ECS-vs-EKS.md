# Day 34 — AWS Compute: EKS, ECS, Lambda: ECS vs. EKS

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** —

## Brief

"When would you choose ECS over EKS? What are the operational tradeoffs?" — this day's assigned interview question — is really asking whether you understand that Kubernetes is a *choice with a cost*, not a default every container workload should reach for. ECS (Elastic Container Service) is AWS's own, simpler, AWS-native container orchestrator; it solves the same fundamental problem (run and schedule containers reliably) with a fraction of the conceptual surface area, at the cost of being AWS-only and less flexible. Being able to argue *both* directions convincingly is what separates a real opinion from parroted Kubernetes enthusiasm.

## What ECS actually is

ECS is AWS's own orchestrator — no separate control plane to think about, no `kubectl`, no CRDs, no Helm. You define a **Task Definition** (the container spec: image, CPU/memory, env vars, IAM task role), a **Service** (how many copies of a task to keep running, load-balancer integration, deployment strategy), and a **Cluster** (a logical grouping — either backed by EC2 instances you manage, or **Fargate**, ECS's original home for the serverless-container idea years before EKS Fargate existed).

```json
{
  "family": "my-app",
  "containerDefinitions": [{
    "name": "app",
    "image": "123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
    "cpu": 256,
    "memory": 512,
    "portMappings": [{ "containerPort": 8080 }]
  }],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "256",
  "memory": "512"
}
```

```bash
aws ecs create-service --cluster my-cluster --service-name my-app \
  --task-definition my-app --desired-count 3 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx]}"
```

ECS integrates natively and simply with the rest of AWS — an ECS service's rolling deployment, ALB target group registration, and IAM task roles are first-class, no add-on controllers or CRDs required to get standard "run this container, keep N copies healthy, put it behind a load balancer" behavior.

## Side-by-side: the real operational tradeoffs

| | ECS | EKS |
|---|---|---|
| Control plane | None to think about — fully AWS-managed, free | Managed, but you pay an hourly control-plane fee and still think in Kubernetes concepts |
| Learning curve | Small — Task Definitions, Services, Clusters | Large — full Kubernetes API, controllers, CRDs, Helm, RBAC |
| Portability | AWS-only — a Task Definition doesn't run anywhere else | Portable — same manifests broadly work on any Kubernetes (GKE, AKS, on-prem) |
| Ecosystem | Smaller — mostly AWS-native tools | Enormous — Helm charts, operators, service meshes, the entire CNCF ecosystem |
| Multi-cloud / hybrid | Not applicable, AWS-specific | A real option if that's a requirement |
| Fargate support | Yes, and simpler to reach for than EKS Fargate | Yes (Day 34 file 1), with more caveats (no DaemonSets, etc.) |
| Ops team size needed | Small teams can run it comfortably with little dedicated expertise | Usually wants at least one engineer who's genuinely comfortable operating Kubernetes |

## When ECS is the right call

- **Team is AWS-only and has no near-term multi-cloud/hybrid requirement** — Kubernetes's biggest selling point (portability across clouds) is worth nothing if you'll only ever run on AWS.
- **Small platform team, or no dedicated platform/infra engineer** — ECS's smaller conceptual surface means a small team can operate it reliably without a Kubernetes specialist on staff.
- **Standard "run stateless containers behind a load balancer" workloads** — the majority of application workloads don't need Kubernetes's more advanced scheduling, CRD-based extensibility, or operator ecosystem; ECS covers this core case with far less to learn and operate.
- **Faster time-to-production for a new service** — no cluster bootstrapping conceptual overhead, no RBAC/namespace design decisions up front.

## When EKS is the right call

- **Genuine multi-cloud or hybrid requirement**, or wanting to avoid AWS lock-in for the orchestration layer specifically.
- **Need the CNCF/Kubernetes ecosystem** — a specific Helm chart, a specific operator (e.g., a database operator, cert-manager, external-dns), a service mesh (Istio/Linkerd) — these are Kubernetes-native and don't have ECS equivalents.
- **Already have Kubernetes expertise on the team** (from a previous job, or another environment) — the learning-curve cost is sunk, so EKS's larger ecosystem becomes a pure upside with little added cost.
- **Standardizing across a large org running mixed environments** (some on-prem, some AWS, some other clouds) where "one API surface everywhere" has real organizational value even within a single cloud.

## Points to Remember

- ECS is AWS-native, simpler, and cheaper to operate for teams with no multi-cloud need and no existing Kubernetes expertise; EKS's biggest wins (portability, ecosystem, ecosystem-native tooling) only pay off if you actually need them.
- ECS has no separate control-plane concept to manage or pay for as a line item the way EKS does; its "cluster" is a lighter, AWS-native grouping concept.
- The single biggest real-world driver of the ECS vs. EKS decision is usually **existing team expertise and multi-cloud requirements**, not raw technical capability — both can run standard containerized workloads reliably.
- ECS supports Fargate too (and did before EKS Fargate existed) — "serverless containers" is not an EKS-exclusive idea.
- Choosing Kubernetes by default because it's the industry-standard resume-building choice, without an actual multi-cloud/ecosystem requirement, is a real and common operational-cost mistake — this is exactly the nuance interviewers are checking for with this question.

## Common Mistakes

- Defaulting to EKS "because Kubernetes is the standard" without a concrete reason (multi-cloud, specific ecosystem tool, existing expertise) that actually requires it — taking on real operational complexity for no corresponding benefit.
- Underestimating the ongoing operational cost of running EKS well — RBAC design, upgrade cadence, CNI/networking choices, node lifecycle — versus assuming "managed control plane" means "no meaningful ops burden."
- Assuming ECS can't do rolling deployments, health checks, auto-scaling, or load-balancer integration — it supports all of these natively; the perception that "ECS is basic and EKS is powerful" undersells ECS for the majority of standard workloads.
- Migrating a small, AWS-only team to EKS for a single Kubernetes-only tool (a specific Helm chart or operator) without weighing whether an ECS-native equivalent or workaround exists first.
- Not accounting for the EKS control-plane hourly fee plus the added engineering-hours cost of Kubernetes expertise when doing a genuine cost comparison between the two — the "which is cheaper" answer is rarely just about compute pricing.
