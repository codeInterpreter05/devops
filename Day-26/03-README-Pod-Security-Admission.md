# Day 26 — RBAC & Security: Pod Security Standards & Admission Webhooks

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

RBAC (file 1) controls *who can create what kind of object*. It says nothing about *how dangerously a Pod's spec is configured* — a user could have perfectly legitimate RBAC permission to create Pods, and still create one that runs as root, mounts the host's filesystem, and disables every container isolation boundary. **Pod Security Standards** and **admission webhooks** are the layer that governs *what a Pod is allowed to look like*, closing that gap. Together with RBAC and IRSA, this is the third leg of the Kubernetes security model interviewers expect you to be able to describe as a coherent whole, not three disconnected facts.

## Pod Security Standards: three built-in profiles

Kubernetes defines three standard profiles (replacing the deprecated `PodSecurityPolicy`, removed in 1.25):

- **`Privileged`** — no restrictions at all. Effectively "anything goes" — appropriate only for trusted, infra-level workloads (CNI/CSI DaemonSets that genuinely need host access) and not something app teams should be granted.
- **`Baseline`** — blocks the most obviously dangerous escalations while remaining broadly compatible with typical containerized apps: no privileged containers, no host namespaces (`hostNetwork`, `hostPID`, `hostIPC`), restricted `hostPath` usage, no dangerous Linux capabilities beyond the default set.
- **`Restricted`** — the hardened, defense-in-depth profile: on top of everything `Baseline` blocks, it requires running as non-root, requires `allowPrivilegeEscalation: false`, requires dropping **all** Linux capabilities (`ALL`) and only adding back `NET_BIND_SERVICE` if needed, requires a seccomp profile (`RuntimeDefault`), and forbids privilege escalation entirely.

These profiles are enforced via the built-in **Pod Security Admission (PSA)** controller, configured per-namespace with labels — no separate policy engine required for the standard three levels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted        # log violations without blocking (useful during migration)
    pod-security.kubernetes.io/warn: restricted           # warn the user via kubectl, without blocking
```

Having three independent modes (`enforce`, `audit`, `warn`) per namespace is deliberately useful for rollout: you can set `warn`/`audit` to `restricted` while leaving `enforce` at `baseline`, see which existing workloads *would* fail under the stricter profile (via `kubectl` warnings and audit logs), fix them, and only then flip `enforce` to `restricted` without a surprise outage.

A `Restricted`-compliant pod's security context typically looks like:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: { type: RuntimeDefault }
  containers:
    - name: app
      image: myapp
      securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
        readOnlyRootFilesystem: true
```

```bash
kubectl label namespace prod pod-security.kubernetes.io/enforce=restricted
kubectl run test-pod --image=nginx --privileged -n prod    # should be rejected outright with a clear PSA error message
```

## Admission webhooks: the general-purpose extension point

Pod Security Admission covers the standard three profiles, but real organizations often need custom, business-specific rules that go beyond them ("every image must come from our internal registry," "every pod must have resource limits set," "no `latest` tags in prod"). That's what **admission webhooks** are for — an external HTTP(S) service that the apiserver calls out to as part of every request's admission phase, *before* the object is persisted to etcd (see Day 22's request lifecycle).

Two kinds, always evaluated in this order:
- **`MutatingWebhookConfiguration`** — can modify the object (e.g., Istio's sidecar injector adding a container, IRSA's Pod Identity webhook adding environment variables/volumes, OPA Gatekeeper's mutation feature setting defaults).
- **`ValidatingWebhookConfiguration`** — can only accept or reject, never modify. This is where policy engines like **OPA/Gatekeeper** and **Kyverno** plug in to enforce arbitrary custom rules.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata: { name: require-resource-limits }
webhooks:
  - name: require-limits.mypolicy.io
    clientConfig:
      service:
        name: policy-webhook-svc
        namespace: policy-system
        path: "/validate"
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    failurePolicy: Fail                # what to do if the webhook itself is unreachable — Fail (safe) vs Ignore (permissive)
    sideEffects: None
    admissionReviewVersions: ["v1"]
```

`failurePolicy` is a genuinely important, often-overlooked field: `Fail` means if your policy webhook pod is down, **every matching request is blocked** (fail closed — safer, but can cause a cluster-wide outage of Pod creation if the webhook itself has an incident). `Ignore` means requests proceed unchecked if the webhook is unreachable (fail open — availability-friendly, but a webhook outage silently becomes a policy bypass). Choosing the wrong one for the wrong policy is a common, high-impact production incident (either "our whole cluster stopped scheduling anything" or "our security policy was silently bypassed for two hours and nobody noticed").

## Points to Remember

- RBAC controls *who can act*; Pod Security Standards/admission control governs *what the object itself is allowed to contain* — both are needed, they solve different problems.
- The three PSA profiles (`Privileged`, `Baseline`, `Restricted`) are enforced per-namespace via labels, with independent `enforce`/`audit`/`warn` modes that support a safe, staged rollout rather than an all-or-nothing switch.
- Mutating webhooks run before validating webhooks, and can change the object; validating webhooks can only accept/reject — this ordering is fixed and matters when reasoning about what a policy engine can and can't do.
- `PodSecurityPolicy` is deprecated and removed since Kubernetes 1.25 — Pod Security Admission (namespace labels) plus custom validating webhooks (OPA Gatekeeper/Kyverno) is the current standard approach.
- `failurePolicy: Fail` vs `Ignore` on a webhook is a real availability-vs-security tradeoff decision, not a default to leave unconsidered.

## Common Mistakes

- Flipping a namespace straight to `enforce: restricted` in production without first running `warn`/`audit` mode to see what would break — causing an unplanned outage of legitimate workloads that happened to run as root or lacked a `securityContext`.
- Assuming RBAC alone is "the security model" for a cluster and never configuring Pod Security Standards or any admission policy — a user with RBAC permission to create Pods can still create a fully privileged, host-mounting pod with no further restriction unless PSA/webhooks are in place.
- Setting `failurePolicy: Fail` on a policy webhook without ensuring that webhook service itself is highly available (multiple replicas, its own resource limits, monitoring) — a single webhook pod crash can halt all matching object creation cluster-wide.
- Forgetting that PSA `enforce` labels are set **per namespace**, not cluster-wide by default — a cluster can have wildly inconsistent security postures across namespaces if this isn't deliberately standardized (e.g., via a namespace-creation template or policy-as-code check).
- Confusing what a mutating webhook (e.g., a sidecar injector) silently changes about a Pod with what was actually submitted — debugging "why does my pod have an extra container I didn't specify" without realizing a mutating webhook added it during admission.
