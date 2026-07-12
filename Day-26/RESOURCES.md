# Day 26 — Resources: RBAC & Security

## Primary (assigned)

- **AWS EKS IRSA documentation** (docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) — free, the assigned starting point. The authoritative walkthrough of OIDC provider setup, trust policy conditions, and the Pod Identity webhook mechanics covered in file 2.

## Deepen your understanding

- **Kubernetes.io — Using RBAC Authorization** (kubernetes.io/docs/reference/access-authn-authz/rbac/) — the full reference for Role/ClusterRole/RoleBinding/ClusterRoleBinding semantics, including the built-in aggregated `view`/`edit`/`admin`/`cluster-admin` ClusterRoles.
- **Kubernetes.io — Pod Security Standards** (kubernetes.io/docs/concepts/security/pod-security-standards/) — the exact rule-by-rule breakdown of `Baseline` vs `Restricted` referenced in file 3.
- **Kubernetes.io — Admission Controllers Reference** (kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) — the full list of built-in admission controllers plus how `MutatingAdmissionWebhook`/`ValidatingAdmissionWebhook` plug into the chain.
- **OPA Gatekeeper documentation** (open-policy-agent.github.io/gatekeeper) and **Kyverno documentation** (kyverno.io) — the two dominant policy-as-code engines for custom validating/mutating admission rules beyond the standard PSA profiles.

## Reference / lookup

- `kubectl explain role.rules` / `kubectl explain clusterrolebinding` — always in sync with your cluster's installed API version.
- **eksctl documentation — IAM Roles for Service Accounts** (eksctl.io/usage/iamserviceaccounts) — the exact `eksctl create iamserviceaccount` flags used in this day's lab.

## Practice

- **KillerCoda / KodeKloud free RBAC labs** — browser-based scenarios specifically built around debugging "why is this Forbidden" RBAC misconfigurations.
- Rebuild the Day 26 lab's IRSA setup using a custom least-privilege IAM policy (scoped to one S3 bucket and specific actions) instead of the broad `AmazonS3ReadOnlyAccess` managed policy — a good self-check that you understand least-privilege IAM, not just IRSA mechanics.
