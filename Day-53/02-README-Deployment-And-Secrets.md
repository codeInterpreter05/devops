# Day 53 — Phase 1 Project: Deployment & Secrets

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Review | **Flag:** 📌 Milestone project

## Brief

Once the infrastructure layer (previous file) exists, the application needs a repeatable, auditable way to get onto the cluster — that's Helm (packaging) plus ArgoCD (continuous, GitOps-driven delivery). And it needs credentials for RDS/ElastiCache without those credentials ever living in a Kubernetes Secret checked into Git or a Helm values file — that's External Secrets Operator pulling from AWS Secrets Manager. This pairing (GitOps deployment + externally-sourced secrets) is what most real platform teams converge on, and it's exactly the kind of detail that shows an interviewer you've thought about the *whole* delivery pipeline, not just "kubectl apply."

## Helm — packaging the application

Helm turns a directory of Kubernetes manifests into a versioned, parameterized, installable unit (a "chart"), solving the problem of "how do I deploy the same app to dev/staging/prod with different resource sizes/replica counts/environment variables without maintaining three copies of near-identical YAML."

```
my-app/
├── Chart.yaml
├── values.yaml              # defaults
├── values-prod.yaml         # prod overrides
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── external-secret.yaml
```

```yaml
# templates/deployment.yaml (excerpt)
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            requests: { cpu: {{ .Values.resources.requests.cpu }}, memory: {{ .Values.resources.requests.memory }} }
            limits:   { cpu: {{ .Values.resources.limits.cpu }}, memory: {{ .Values.resources.limits.memory }} }
```
```bash
helm install my-app ./my-app -f values-prod.yaml -n production
helm upgrade my-app ./my-app -f values-prod.yaml -n production
helm rollback my-app 1 -n production      # instant rollback to a previous release revision
```

**Why Helm and not raw manifests for anything beyond a toy project**: templating avoids duplication across environments, `helm rollback` gives you a one-command way back to a known-good release without reconstructing old YAML by hand, and the chart becomes the single versioned artifact that both a human and ArgoCD can point at.

## ArgoCD — GitOps continuous delivery

ArgoCD flips the traditional CI/CD push model on its head: instead of a pipeline running `kubectl apply`/`helm upgrade` against the cluster (push-based, meaning your CD system needs cluster credentials), ArgoCD runs **inside the cluster**, continuously watching a Git repository, and **pulls** changes — reconciling the cluster's actual state to match whatever's declared in Git.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-manifests.git
    targetRevision: main
    path: charts/my-app
    helm:
      valueFiles: [values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # remove resources deleted from Git
      selfHeal: true     # revert manual cluster drift back to match Git
```

**Why "pull, not push" is the actual architectural win, not just a buzzword**: no CI system needs long-lived, broad cluster-admin credentials sitting in a pipeline's secret store (a common lateral-movement target if the CI system itself is compromised) — ArgoCD's in-cluster controller needs credentials to *read* the Git repo, not the other way around. `selfHeal: true` also means **Git is the actual source of truth, continuously enforced** — a manual `kubectl edit` fixing something in a hurry gets automatically reverted back to match Git within moments, which sounds inconvenient until you realize it's exactly what prevents "we don't know why prod differs from what's in the repo" drift from ever accumulating silently.

**App-of-Apps pattern**: for the multi-component stack this project describes (the app itself, External Secrets Operator, the AWS Load Balancer Controller, KEDA, Karpenter — next file), a common ArgoCD pattern is one parent `Application` whose source is a directory of *other* `Application` manifests, so the entire platform's desired state — not just one app — is bootstrapped and reconciled from a single Git commit.

## Secrets: External Secrets Operator + AWS Secrets Manager

The problem this solves: application code needs the RDS connection string, the Redis endpoint/auth token, and any third-party API keys — but none of that should ever exist as plaintext in a Helm `values.yaml` committed to Git, nor as a Kubernetes `Secret` object created by hand outside of GitOps (which would itself need to be synced/managed somehow, and base64-encoded Secret manifests in Git are only obfuscated, not actually secret).

**External Secrets Operator (ESO)** runs in-cluster and continuously syncs values *from* an external secret store (AWS Secrets Manager, in this project) *into* native Kubernetes `Secret` objects, which your app then consumes exactly like any other Secret (env var, mounted file) — the app itself has zero AWS-specific code.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa   # IRSA-bound service account — no static AWS keys anywhere
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-db-credentials
  namespace: production
spec:
  secretStoreRef: { name: aws-secrets-manager, kind: SecretStore }
  target: { name: app-db-credentials }    # the K8s Secret ESO creates/maintains
  refreshInterval: 1h                      # re-pull from Secrets Manager periodically, catching rotations
  data:
    - secretKey: password
      remoteRef: { key: prod/app/db, property: password }
    - secretKey: host
      remoteRef: { key: prod/app/db, property: host }
```

**Why `refreshInterval` matters**: RDS's `manage_master_user_password` (previous file) rotates the password automatically on a schedule inside Secrets Manager. Without ESO's periodic refresh pulling the new value into the Kubernetes Secret (and something restarting pods to pick it up — a Reloader-style controller, or an app that re-reads the mounted secret file), the cluster would keep using a now-stale password after a rotation, causing an outage the moment the old password is invalidated. This end-to-end chain — RDS rotates in Secrets Manager → ESO refreshes the K8s Secret → pods pick up the new value — is exactly the kind of "does the whole system actually work together" reasoning interviewers are probing for with today's architecture-walkthrough question.

**IRSA (IAM Roles for Service Accounts)** is what lets ESO's pods assume an IAM role scoped to `secretsmanager:GetSecretValue` on specific secret ARNs only — no static AWS access keys stored anywhere in the cluster, and blast radius is limited to exactly the secrets that role is allowed to read.

## Points to Remember

- Helm packages and parameterizes manifests for repeatable multi-environment deploys and gives you `helm rollback` as an immediate recovery path.
- ArgoCD pulls from Git rather than being pushed to by CI — the cluster never hands out broad credentials to an external pipeline, and `selfHeal` continuously reconciles away manual drift.
- The App-of-Apps pattern lets one parent ArgoCD Application bootstrap an entire platform stack from a single Git source, not just one component.
- External Secrets Operator syncs values from AWS Secrets Manager into native Kubernetes Secrets, so application code and Helm charts never contain plaintext credentials.
- IRSA gives in-cluster controllers (Load Balancer Controller, ESO) scoped IAM permissions without any static AWS credentials living in the cluster.
- `refreshInterval` on an ExternalSecret is what makes automatic credential rotation (e.g., RDS-managed password rotation) actually propagate to running pods instead of silently going stale.

## Common Mistakes

- Committing a raw Kubernetes `Secret` manifest (even base64-encoded) to the same Git repo ArgoCD syncs from — base64 is encoding, not encryption; anyone with repo read access can decode it instantly.
- Setting `selfHeal: true` without team-wide awareness, then being confused when a "quick fix" via `kubectl edit` during an incident gets silently reverted moments later — the fix needs to go into Git, not directly onto the cluster.
- Forgetting to set a `refreshInterval` (or setting it too long) on an ExternalSecret, so an automatic credential rotation in Secrets Manager doesn't reach the cluster until long after the old credential has already been invalidated, causing an avoidable outage.
- Giving the ESO or Load Balancer Controller's IAM role overly broad permissions (e.g., `secretsmanager:GetSecretValue` on `*` instead of specific ARNs) instead of scoping IRSA roles tightly — this turns a compromised in-cluster controller into a much bigger blast radius than necessary.
- Deploying Helm charts directly via `helm install`/`helm upgrade` from a laptop or ad-hoc CI step alongside an ArgoCD-managed environment — mixing push and pull deployment models on the same resources causes ArgoCD to detect (and potentially revert) changes it didn't originate, and creates confusion about what the actual source of truth is.
