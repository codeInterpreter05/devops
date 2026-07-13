# Day 113 — Crossplane & Self-Service Infra: Crossplane Fundamentals

**Phase:** 4 – Advanced/Specialization | **Week:** W19 | **Domain:** Platform Eng | **Flag:** —

## Brief

Crossplane is the concrete technology that makes yesterday's "self-service infrastructure via Kubernetes CRDs" idea real: it turns the Kubernetes API server into a universal control plane for *any* infrastructure, cloud or otherwise, using the exact same reconciliation model Kubernetes uses for Pods and Deployments. This matters practically because it's the leading way platform teams build genuinely self-service, guardrailed provisioning without hand-rolling a custom API — and it's the subject of today's flagged interview question about how it differs from Terraform.

This day is split into three files:
1. **This file** — the core reconciliation model, Providers, and the Composite Resource / Composition abstraction.
2. **[02-README-Crossplane-Vs-Terraform.md](02-README-Crossplane-Vs-Terraform.md)** — the architectural comparison and what problem Crossplane solves for platform teams specifically.
3. **[03-README-Beyond-Crossplane-Kratix-And-Port.md](03-README-Beyond-Crossplane-Kratix-And-Port.md)** — Kratix for multi-cluster platform delivery and Port as a UI layer.

## Crossplane's core idea: infrastructure as Kubernetes-native CRDs

Crossplane extends the Kubernetes API with CRDs representing cloud resources, and runs **controllers** that continuously reconcile the *desired state* declared in those CRDs against the *actual state* in the cloud provider — exactly the same control-loop pattern a `Deployment` controller uses to keep the right number of Pods running. The difference from a `Deployment` is just what's being reconciled: instead of Pods, it's an S3 bucket, an RDS instance, a VPC.

```bash
kubectl apply -f - <<EOF
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-app-data
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: aws-provider-config
EOF
```

Apply that, and a Crossplane **provider controller** notices the new `Bucket` object, calls the AWS API to create the actual bucket, and continuously reconciles — if someone manually deletes the bucket out-of-band in the AWS console, Crossplane's next reconcile loop notices the drift and recreates it (or flags it, depending on policy), the same self-healing behavior Kubernetes gives you for Pods.

## Providers — pluggable cloud API coverage

A **Provider** is a Crossplane extension (itself distributed as a package, installed via a `Provider` CRD) that adds CRDs and controllers for a specific cloud or system's API surface:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.0.0
```

There are providers for AWS, GCP, Azure, and dozens of others (even non-cloud systems like GitHub, Datadog, Helm) — each maps that system's resources onto Kubernetes CRDs, generated (in the modern Upbound-maintained providers) largely from the underlying cloud SDK/Terraform-provider schema, which is why Crossplane provider coverage tracks close behind Terraform provider coverage for the same clouds.

## Composite Resources (XRs) and Compositions — the self-service abstraction layer

Raw provider CRDs (a `Bucket`, an `RDSInstance`) are still cloud-specific and low-level — exactly the complexity a platform team wants to hide from app developers. Crossplane's answer is a two-part abstraction:

1. **`CompositeResourceDefinition` (XRD)** — defines a new, custom, platform-owned API (the abstraction app developers actually see), e.g. a `PostgreSQLInstance` with a simple schema (`size: small|medium|large`, `region`).
2. **`Composition`** — the implementation: a template mapping that simple XRD schema onto one or more concrete provider resources (an `RDSInstance` plus a `SecurityGroup` plus a `Secret` populated with connection details), with the platform team encoding all the "correct" defaults (encryption on, backups on, private subnet only) into the template itself.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.org
spec:
  group: database.example.org
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  size:
                    type: string
                    enum: ["small", "medium", "large"]
```

An application developer then only ever writes:

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-service-db
spec:
  parameters:
    size: small
```

...and gets a fully-configured, policy-compliant RDS instance (with whatever the platform team's `Composition` decided "small" means — instance class, storage, backup retention, encryption, network placement) without ever seeing an `RDSInstance` CRD or an AWS console. This `Claim` → Composite Resource → concrete managed resources chain **is** self-service infrastructure with structural guardrails, in the exact sense described in yesterday's notes: the developer's only exposed choice is `size`, so there's no way to accidentally provision an unencrypted, publicly-exposed database — that option simply isn't in the schema.

## Points to Remember

- Crossplane's core mechanism is the same reconciliation control loop Kubernetes uses everywhere else — desired state (CRD spec) versus actual cloud state, continuously reconciled, with drift correction as a natural side effect.
- Providers are pluggable packages that add CRDs/controllers for a specific system's API (AWS, GCP, Azure, GitHub, etc.) — install/upgrade them like any other Crossplane package.
- XRDs define the platform-owned, developer-facing abstraction (simple schema); Compositions define how that abstraction maps onto real, opinionated, policy-compliant provider resources — this pairing is what makes Crossplane a self-service *engine*, not just another IaC tool.
- A `Claim` is the namespaced object an app developer actually creates; it's bound to a cluster-scoped Composite Resource (the `XR`), which is what actually owns the underlying managed resources — this indirection lets platform teams manage XRs/Compositions separately from app-facing claims.
- Drift correction is automatic and continuous, same as Pod reconciliation — someone manually changing a cloud resource out-of-band gets reconciled back (or flagged) on the next loop, not just detected on the next `terraform plan`.

## Common Mistakes

- Exposing raw provider resources (`RDSInstance`, `Bucket`) directly to application developers instead of building an XRD/Composition abstraction — this defeats the entire point of self-service guardrails and just relocates Terraform-style complexity into YAML.
- Forgetting that a `Composition` change doesn't automatically apply to already-provisioned resources retroactively in the way you might expect — understand your Crossplane version's update/patch propagation behavior before assuming a Composition edit instantly fixes every existing instance.
- Not setting resource-level deletion policies deliberately — accidentally deleting a `Claim` can cascade to deleting the real cloud resource (an RDS instance with real data) if the policy isn't set to what you actually intend.
- Treating provider CRD schemas as stable/self-explanatory without checking the specific provider version's docs — schemas evolve between provider versions (especially community versus Upbound-official providers), and copy-pasted YAML from an older tutorial can silently fail to apply.
- Ignoring RBAC on the XRD/Claim layer — if every developer has cluster-wide access to create any Composite Resource directly (bypassing the namespaced Claim), the guardrail model breaks down; RBAC needs to be scoped so developers can only create Claims, not XRs or raw managed resources.
