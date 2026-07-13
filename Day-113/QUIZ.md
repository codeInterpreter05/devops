# Day 113 — Quiz: Crossplane & Self-Service Infra

Try to answer without looking at your notes. Answers are at the bottom.

1. What is the core reconciliation model Crossplane uses, and how is it identical to how a `Deployment` controller behaves?
2. What is a `Provider` in Crossplane, and what does it actually add to the cluster?
3. What are the two parts of the self-service abstraction layer, and what does each one define?
4. In the Claim → Composite Resource → managed resource chain, which object does the application developer actually create, and which one owns the real cloud resource?
5. Give a concrete example of a "structural guardrail" that an XRD/Composition can enforce which a manual review process could still fail to catch.
6. What's the single biggest architectural difference between Terraform and Crossplane, phrased as one sentence?
7. Why doesn't Crossplane need a separate state file the way Terraform does?
8. What extra tooling does Terraform typically need to become a safe, governed self-service system, and why does Crossplane get that "for free"?
9. What is a Kratix Promise, and what three things does it declare?
10. Why does Kratix explicitly model a split between a "platform cluster" and "worker clusters"?
11. How does Port relate to Backstage, and what's the core tradeoff between choosing one over the other?
12. **Interview question:** How does Crossplane differ from Terraform? What problem does it solve for platform teams?

---

## Answers

1. Crossplane's controllers continuously compare desired state (what's declared in a CRD's spec) against actual state (what really exists via the cloud API) and reconcile any difference — the identical loop a `Deployment` controller runs to keep the declared number of Pods running, just applied to cloud resources instead of Pods.
2. A `Provider` is a pluggable package (installed via a `Provider` CRD) that adds new CRDs and their corresponding controllers for a specific system's API surface — e.g., `provider-aws-s3` adds a `Bucket` CRD and the controller logic to reconcile it against the real AWS S3 API.
3. A `CompositeResourceDefinition` (XRD), which defines the new, simplified, platform-owned API schema developers actually see (e.g., `size: small|medium|large`), and a `Composition`, which defines how that simple schema maps onto one or more concrete, opinionated provider resources with the platform team's defaults baked in.
4. The developer creates the `Claim` (a namespaced object, e.g. `PostgreSQLInstance`). The `Claim` is bound to a cluster-scoped Composite Resource (the `XR`), and it's the `XR` that actually owns and manages the underlying concrete managed resources (like the real `Instance`/`Bucket`).
5. An XRD schema that only exposes a `size` enum (`small`/`medium`/`large`) with no field to disable encryption or make a database publicly accessible — the unsafe option simply doesn't exist in the schema, so there's nothing for a developer to misconfigure even accidentally, unlike a manual review process which depends on a human catching every violation every time.
6. Terraform is push-based/on-demand — you trigger convergence by running plan/apply — while Crossplane is control-loop-based/continuous — convergence is always running via in-cluster controllers.
7. Because the "state" Crossplane needs is just whatever's currently declared in its CRD objects, which are stored in Kubernetes' own etcd — the same place all other cluster state already lives — so there's no need for a separate file/backend to track desired state independently.
8. Terraform typically needs something like Atlantis, Terraform Cloud, or a custom wrapper to control who can run `apply`, with what variables, against what state, and with what approval — essentially building a governed multi-tenant API on top of a CLI tool. Crossplane gets this governance largely for free because Kubernetes' existing RBAC, namespaces, and admission-control primitives already provide exactly that kind of governed, multi-tenant API surface.
9. A Kratix Promise is a bundle that packages a repeatable platform capability for delivery across multiple clusters. It declares: the API schema for a developer-facing request (similar in spirit to an XRD), a pipeline (a sequence of container-based steps that can call Crossplane, Terraform, Helm, or anything else to fulfill the request), and scheduling logic for which worker cluster should fulfill a given request.
10. Because organizations running many Kubernetes clusters (per-team, per-region, per-environment) want one consistent self-service experience regardless of which specific cluster ultimately does the work — the platform cluster is where requests land and get scheduled, and worker clusters are chosen dynamically (by label, capacity, region) to actually fulfill them, which single-cluster-centric Crossplane doesn't directly model on its own.
11. Port covers the same core pillars as Backstage (catalog, self-service actions/forms, scorecards) but is delivered as a managed SaaS product rather than something you self-host. The core tradeoff is build-vs-buy: Backstage gives full control and no licensing cost at the price of operating and maintaining the backend yourself; Port trades that operational burden for speed and a subscription cost.
12. Strong answer: "Terraform runs a plan/apply cycle on demand, tracked against a separate state file, so drift is only caught the next time someone runs it. Crossplane runs continuously reconciling Kubernetes-native CRDs against real cloud state, using Kubernetes' own etcd as the source of truth, so drift is caught and corrected immediately without a separate state file. For platform teams specifically, Crossplane solves the problem of building a safe, governed, self-service provisioning API — because Kubernetes' RBAC, CRD, and admission-control model already gives you exactly that kind of multi-tenant API surface for free, whereas building the equivalent guardrails on top of raw Terraform requires bolting on extra tooling like Atlantis or Terraform Cloud. In practice, a lot of platform teams use both: Terraform for foundational, rarely-changed infrastructure, and Crossplane Claims for the frequent, per-team self-service resources developers request on demand."
