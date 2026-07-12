# Day 68 — Policy as Code: Gatekeeper and Kyverno for Kubernetes

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** DevSecOps

## Brief

Raw OPA doesn't know anything about Kubernetes — you'd have to build your own admission webhook, wire it to the API server, and handle the plumbing yourself. **Gatekeeper** and **Kyverno** are the two dominant projects that do that integration work for you, letting you express policy as native Kubernetes Custom Resources instead of hand-rolling a webhook server. Knowing both, and being able to articulate *why* a team picks one over the other, is a very common platform-engineering interview probe — it signals you understand Kubernetes admission control, not just "I applied some YAML I copied."

## Kubernetes admission control — the mechanism both rely on

Every `kubectl apply`/`create`/`update` goes through a chain in the API server: authentication → authorization (RBAC) → **admission controllers** → persisted to etcd. Admission controllers come in two flavors:

- **Mutating admission webhooks** — can modify the object before it's persisted (e.g., injecting a sidecar, pinning an image to its digest).
- **Validating admission webhooks** — can only allow or reject the object, optionally with a reason.

Both Gatekeeper and Kyverno register themselves as webhooks (of both kinds) via `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` resources, so the API server calls out to them synchronously on every relevant request. If the webhook times out or errors and `failurePolicy` is set to `Fail` (the safer default for security policies), the API server rejects the request — meaning a broken policy engine can block your entire cluster's deployments, which is why webhook availability/HA matters operationally.

## Gatekeeper — OPA, packaged for Kubernetes

Gatekeeper wraps OPA and adds two Kubernetes-native concepts on top of raw Rego:

- **ConstraintTemplate** — defines the *schema and Rego logic* for a reusable policy type (parameterized, like a class/function definition).
- **Constraint** — an instance of a ConstraintTemplate, supplying actual parameters and scoping (which namespaces/kinds it applies to).

```yaml
# ConstraintTemplate: defines the reusable "require labels" policy logic
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("missing required labels: %v", [missing])
        }
---
# Constraint: an instance requiring "team" and "env" labels, scoped to Deployments
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-and-env
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    labels: ["team", "env"]
```

This separation (template vs. constraint) is Gatekeeper's core value proposition: a platform team writes and reviews the Rego once, and application teams (or even self-service via a PR) just adjust Constraint parameters without ever touching Rego directly.

### Audit mode vs. enforcement

Gatekeeper constraints support `enforcementAction: deny` (block) or `dryrun` (log violations without blocking) — critical for rolling out a new policy safely: run it in `dryrun` against real cluster traffic first, review what *would* have been blocked, then flip to `deny` once you're confident it won't break existing workloads.

## Kyverno — policy as native Kubernetes YAML, no Rego

Kyverno's pitch is directness: policies are plain Kubernetes-style YAML with a JSON-path-like matching syntax, not a separate DSL. For teams without Rego expertise, this is a much lower barrier to entry and to code review (a YAML diff on a PR reads far more obviously than a Rego diff to most Kubernetes engineers).

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce   # or Audit
  background: true                   # also scan existing resources, not just new admissions
  rules:
    - name: check-limits
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "CPU and memory limits are required."
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

Kyverno also supports **mutation** (auto-fixing instead of just rejecting) and **generation** (auto-creating related resources, e.g., a default `NetworkPolicy` whenever a new `Namespace` is created) — capabilities that go beyond what Gatekeeper's validate-only model typically covers out of the box.

```yaml
# Mutate: auto-inject a default resource limit instead of rejecting the Pod outright
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-limits
spec:
  rules:
    - name: default-limits
      match:
        any:
          - resources: { kinds: ["Pod"] }
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"
                resources:
                  limits:
                    +(memory): "512Mi"
                    +(cpu): "500m"
```

## Choosing between them

| | Gatekeeper | Kyverno |
|---|---|---|
| Policy language | Rego (steeper learning curve) | Native YAML (lower barrier) |
| Mutation support | Limited/newer | First-class, mature |
| Resource generation | No | Yes (auto-create related resources) |
| Ecosystem reuse | Shares OPA policies with non-K8s systems (Terraform, CI, APIs via Conftest) | Kubernetes-only |
| Best fit | Teams that need one policy engine across K8s + IaC + CI, or already have Rego expertise | Teams that want fast onboarding and rich mutation/generation, Kubernetes-only scope |

The most defensible interview answer isn't "Kyverno is strictly better" or vice versa — it's recognizing the tradeoff: Gatekeeper's Rego investment pays off if you want the *same* policy engine and language across Kubernetes, Terraform (via Conftest), and CI; Kyverno wins on developer ergonomics and native mutation/generation if your policy needs are Kubernetes-only.

## Points to Remember

- Both are admission webhooks: they intercept API server requests (validating and/or mutating) before objects are persisted — a webhook outage with `failurePolicy: Fail` can block cluster-wide deployments, so HA matters.
- Gatekeeper = OPA/Rego + a ConstraintTemplate/Constraint split (reusable logic vs. parameterized instances) — write Rego once, instantiate many times with different parameters.
- Kyverno = native Kubernetes-style YAML, no separate DSL, with mature mutation ("auto-fix") and generation ("auto-create related resources") support beyond simple validation.
- Always roll out new policies in audit/dry-run mode first (`enforcementAction: dryrun` in Gatekeeper, `validationFailureAction: Audit` in Kyverno) against real traffic before flipping to enforce/deny.
- `background: true` (Kyverno) / periodic audit (Gatekeeper) matters because admission webhooks only catch *new* requests — pre-existing non-compliant resources need a separate background scan to be surfaced.

## Common Mistakes

- Deploying a new policy directly in enforcing mode without a dry-run/audit period first — instantly blocking legitimate deployments across teams because the policy's edge cases weren't tested against real traffic.
- Running Gatekeeper/Kyverno as a single replica with `failurePolicy: Fail` — an unplanned restart or crash of the one pod momentarily blocks *all* admission-controlled requests cluster-wide until it recovers.
- Assuming a validating policy retroactively fixes already-running non-compliant workloads — admission webhooks only intercept new `create`/`update` calls; existing resources need an explicit background audit/scan to even be detected.
- Choosing Kyverno for its ergonomics, then hitting a wall needing OPA's non-Kubernetes reuse (e.g., wanting the exact same policy logic enforced in a Terraform CI gate via Conftest) — and having to duplicate policy logic in two different languages instead of sharing Rego across both.
- Writing an overly broad `match` (e.g., matching all `Pod` kinds cluster-wide including `kube-system`) and inadvertently blocking core cluster components — always scope policies with explicit namespace exclusions for system namespaces.
