# Day 46 — K8s Operators & CRDs: Custom Resource Definitions & the Operator Pattern

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

Kubernetes ships with a fixed vocabulary — Pods, Deployments, Services, ConfigMaps — but real infrastructure needs domain-specific concepts: "a highly-available Postgres cluster," "a Kafka topic," "a TLS certificate that auto-renews." CRDs let you extend the Kubernetes API with your own resource types, and the Operator pattern is what makes those custom resources actually *do* something — turning Kubernetes from "a container scheduler" into "a platform that manages the full lifecycle of anything you can describe declaratively." This is one of the most conceptually important ideas in the modern Kubernetes ecosystem — nearly every serious piece of infrastructure software (databases, message queues, service meshes, cert management) now ships as an Operator, and "what problem do Operators solve" is a very common interview question because it tests whether you understand *why*, not just *how to install one*.

This day is split into three files:

1. **This file** — what a CRD is and the Operator pattern conceptually.
2. **[02-README-Controller-Reconciliation-Loop.md](02-README-Controller-Reconciliation-Loop.md)** — the reconciliation loop mechanics and Operator SDK basics.
3. **[03-README-Real-World-Operators.md](03-README-Real-World-Operators.md)** — concrete examples: CloudNativePG (Postgres) and Strimzi (Kafka).

## Custom Resource Definitions (CRDs)

A CRD is a Kubernetes object that **teaches the API server a new resource type**. Once applied, the new type behaves exactly like a built-in one from the client's perspective — it's stored in etcd, versioned, has a schema, supports `kubectl get/describe/apply`, respects RBAC, and can be watched via the standard Kubernetes API/watch mechanism.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.postgresql.cnpg.io
spec:
  group: postgresql.cnpg.io
  names:
    kind: Cluster
    plural: clusters
    singular: cluster
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              instances: {type: integer}
              storage:
                type: object
                properties:
                  size: {type: string}
```

Once this CRD exists, a user can write:
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: {name: my-db}
spec:
  instances: 3
  storage: {size: 10Gi}
```
and `kubectl apply -f` it just like a Deployment. **But applying this YAML alone does nothing beyond store the object in etcd.** A CRD by itself is just a schema — a new shape of data Kubernetes now knows how to validate and store. There's no Postgres cluster running yet. That's the missing piece the Operator provides.

**Schema validation matters here**: the `openAPIV3Schema` (based on OpenAPI v3, using the CRD's own subset of it) is what makes `kubectl apply` reject a malformed custom resource *before* it's even stored — e.g., rejecting `instances: "three"` (wrong type) or a missing required field, the same experience as built-in resource validation.

**CRD versioning** matters for anything long-lived: multiple `versions` can coexist (`v1alpha1`, `v1beta1`, `v1`), only one marked `storage: true` (the version etcd actually persists), and conversion webhooks handle translating between versions when clients request a different one than what's stored — this is exactly how core Kubernetes resources evolve their own APIs over time (e.g., `Ingress` moving from `extensions/v1beta1` to `networking.k8s.io/v1`), and CRDs follow the identical mechanism.

## The Operator pattern

**An Operator = CRD(s) + a Controller that watches them and takes action to make reality match the desired state declared in the custom resource.** The term comes from the idea of encoding **human operational knowledge** ("how do I run this piece of software reliably — provisioning, failover, backups, upgrades, scaling") into software that runs continuously inside the cluster, instead of that knowledge living in a person's head or a runbook.

**What a good Operator actually automates, beyond what a Deployment/StatefulSet already gives you:**
- **Day-2 operations**: backups, point-in-time restore, minor/major version upgrades, failover promotion, credential rotation — these are the operationally hard parts that generic Kubernetes primitives don't know how to do for a specific piece of software.
- **Domain-specific self-healing**: a generic StatefulSet's liveness probe can restart a crashed Postgres pod, but it has no idea how to correctly promote a replica to primary during a failover, rejoin a rebuilt replica to the cluster, or resize storage safely — a Postgres Operator encodes exactly that Postgres-specific logic.
- **Declarative lifecycle management**: instead of a runbook saying "to add a read replica, SSH in and run these 6 commands," you just change `spec.instances: 3` to `4` and the Operator's reconciliation loop handles provisioning, joining, and health-checking the new replica.

**When to use an existing Operator vs. write your own** (this day's interview question: "when should you write your own?"):
- **Use an existing, mature Operator** (CloudNativePG for Postgres, Strimzi for Kafka, Prometheus Operator, cert-manager) for anything with a healthy open-source ecosystem — this is the overwhelming majority of cases. These projects encode years of hard-won operational knowledge you shouldn't try to replicate.
- **Write your own** only when you have genuinely custom, internal operational logic that doesn't map to any existing tool — e.g., automating your company's own bespoke internal platform's lifecycle (provisioning a full "tenant" consisting of a namespace + quota + specific app configuration + DNS record, as one atomic declarative unit), or a proprietary piece of software with no existing Operator and complex enough day-2 operations to justify the investment. Building an Operator is a real, ongoing engineering commitment — you're committing to maintain a piece of control-plane software, not just a Helm chart.
- **A strong interview answer explicitly warns against overusing the pattern**: not every application needs an Operator — a well-configured Helm chart with existing generic resources (Deployment, StatefulSet, HPA) is often entirely sufficient when there's no complex, ongoing lifecycle management need beyond initial deployment.

## Points to Remember

- A CRD only extends the API's *vocabulary* (schema, storage, validation) — it does nothing by itself. The Operator (its Controller) is what turns the custom resource into actual running infrastructure and ongoing lifecycle management.
- CRDs support the same versioning/schema-validation/RBAC mechanics as built-in Kubernetes resources, including multiple API versions and conversion between them.
- The Operator pattern's value is encoding **day-2 operational knowledge** (backup, failover, upgrade, scaling logic specific to the software) as code that runs continuously, not just day-1 deployment.
- Default to using a mature, existing Operator for well-known software (databases, message queues, cert management); only build a custom Operator when you have genuinely bespoke, ongoing operational logic that no existing tool covers.
- Building an Operator is an ongoing engineering commitment (you now maintain control-plane software), not a one-time task like writing a Helm chart.

## Common Mistakes

- Applying a CRD's YAML and expecting the underlying software to actually run — without the corresponding Operator/Controller deployed and watching, the custom resource is inert data sitting in etcd.
- Writing a custom Operator for a problem an existing, well-maintained open-source Operator already solves — reinventing complex, hard-won operational logic (failover, backup/restore) that took mature projects years to get right.
- Treating an Operator's CRD schema as fixed forever — not planning for schema evolution (multiple API versions, conversion webhooks) when the custom resource's shape needs to change down the line.
- Underestimating the ongoing maintenance burden of a hand-rolled Operator — it's a long-lived piece of software that must be upgraded alongside Kubernetes API changes, not a one-off script.
- Conflating "we use CRDs" with "we use the Operator pattern" — some CRDs exist purely as configuration/data objects consumed by an external system, without an in-cluster controller reconciling them; that's a valid pattern too, but it's not the same thing as an Operator.
