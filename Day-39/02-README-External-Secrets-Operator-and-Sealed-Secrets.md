# Day 39 — Secrets in AWS & K8s: External Secrets Operator & Sealed Secrets

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets

## Brief

Kubernetes' native `Secret` object is base64-encoded, **not encrypted**, by default — anyone with `get`/`list` RBAC access to a namespace's secrets, or access to the underlying etcd datastore (unless encryption-at-rest is separately configured), can trivially read every secret's plaintext. Two different, complementary strategies solve the real problem of "how do secrets get into a cluster safely, and how do they stay in sync with a GitOps workflow": **External Secrets Operator (ESO)**, which pulls secrets from an external system like Vault or AWS Secrets Manager and materializes them as native K8s Secrets, and **Sealed Secrets**, which lets you commit an *encrypted* secret directly into git, safe by design, and only your cluster can decrypt it.

## Why a plain Kubernetes `Secret` isn't enough

```bash
kubectl get secret myapp-config -o jsonpath='{.data.password}' | base64 -d
```

That one-liner is all it takes — base64 is an *encoding*, not encryption, reversible by anyone, no key required. The K8s `Secret` object mainly exists to (a) avoid secrets in plain env-var-in-pod-spec form scattered everywhere, and (b) get RBAC-controlled access and separate storage from `ConfigMap`s — it is **not**, by itself, a secrets management solution. Real protection requires etcd encryption at rest (a cluster-admin concern, not a workload concern) plus not committing raw `Secret` manifests to git.

## External Secrets Operator (ESO)

ESO is a Kubernetes operator that continuously syncs secrets **from** an external source of truth **into** native K8s `Secret` objects, so applications keep reading ordinary `Secret`s (zero application code changes) while the actual secret value lives and is managed in Vault, AWS Secrets Manager, SSM Parameter Store, GCP Secret Manager, Azure Key Vault, etc.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: myapp-db-secret        # the native K8s Secret ESO creates/manages
  data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/db-password
        property: password
```

What happens: ESO's controller polls (`refreshInterval`) AWS Secrets Manager, fetches the current value of `prod/myapp/db-password`, and writes/updates a native `Secret` named `myapp-db-secret` in the cluster. A pod mounts that Secret exactly like any other Kubernetes Secret — `envFrom`/`env.valueFrom.secretKeyRef` or a volume mount — with zero awareness that AWS or Vault is involved at all. If the secret changes upstream (e.g., a rotation event in Secrets Manager), ESO picks up the new value on its next poll and updates the K8s Secret; pairing this with a tool like **Reloader** or **Stakater's Reloader** (or a rolling restart trigger) ensures running pods actually pick up the new value rather than keeping a stale one cached in memory from pod start.

**Auth to the cloud provider** typically uses **IRSA** (IAM Roles for Service Accounts, on EKS) or a similar workload-identity mechanism — the ESO controller pod authenticates as an AWS IAM role via its own Kubernetes service account, with no static AWS credentials stored in the cluster at all, mirroring the Vault Kubernetes-auth pattern from Day 38.

## Sealed Secrets (Bitnami) — encrypt-then-commit, for GitOps

ESO assumes a live external secrets backend the cluster can reach at runtime. **Sealed Secrets** takes the opposite approach: it lets you **encrypt a Secret manifest client-side** so the *encrypted* form is safe to commit directly to git (fitting neatly into GitOps workflows like ArgoCD/Flux, which want everything — including secrets — declared as manifests in a repo), and only the specific cluster holding the matching private key can decrypt it back into a real Secret.

```bash
# One-time: install the controller in-cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml

# Encrypt a Secret manifest client-side using kubeseal
kubectl create secret generic myapp-db-secret \
  --from-literal=password=S3cr3t! \
  --dry-run=client -o yaml > secret.yaml

kubeseal --format yaml < secret.yaml > sealedsecret.yaml
```

`sealedsecret.yaml` — safe to commit:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-db-secret
spec:
  encryptedData:
    password: AgBy8hCF8...   # ciphertext, only this cluster's controller can decrypt
```

```bash
kubectl apply -f sealedsecret.yaml
kubectl get secret myapp-db-secret   # the in-cluster controller decrypted it into a real Secret automatically
```

The in-cluster **Sealed Secrets controller** holds an asymmetric key pair; `kubeseal` fetches the **public** key (safe to distribute) to encrypt client-side, but only the controller's private key (never leaves the cluster) can decrypt — so even someone with full read access to the git repository (or a leaked laptop with `sealedsecret.yaml` on it) gains nothing without also compromising that specific cluster's controller.

## ESO vs. Sealed Secrets — when to use which

| | ESO | Sealed Secrets |
|---|---|---|
| Source of truth | External system (Vault, Secrets Manager, etc.) | The encrypted manifest itself, in git |
| Rotation | Automatic — reflects upstream changes on next poll | Manual — you must re-seal and re-commit on every change |
| Fits pure GitOps (everything as committed YAML) | Partially — the `ExternalSecret` manifest is committed, but the *value* lives elsewhere | Fully — the encrypted secret itself lives in git alongside all other manifests |
| Dependency at runtime | Requires the external secrets backend to be reachable | None — cluster only needs its own controller, no external call |
| Best for | Secrets that already live in and are managed by a cloud/Vault system | Teams that want secrets to travel through the exact same git-based promotion pipeline as everything else |

Many real-world clusters run **both**: ESO for anything that already lives in Secrets Manager/Vault (letting rotation flow through automatically), Sealed Secrets for smaller, cluster-specific one-off secrets that don't warrant a separate secrets-manager entry.

## Points to Remember

- A native Kubernetes `Secret` is base64-encoded, not encrypted — it is a storage/RBAC construct, not a secrets management solution by itself; etcd encryption at rest is a separate cluster-level concern.
- ESO continuously syncs secrets *from* an external backend *into* native K8s Secrets on a `refreshInterval`, so rotation upstream flows through automatically; applications need zero code changes.
- Sealed Secrets encrypts client-side with a public key (via `kubeseal`) so the ciphertext is safe to commit to git; only the specific cluster's controller (holding the private key) can decrypt it back into a real Secret.
- ESO needs a reachable external backend at sync time; Sealed Secrets needs nothing but its own in-cluster controller — that's the core operational trade-off between the two.
- Workload identity (IRSA on EKS, Kubernetes auth on Vault) should be how ESO itself authenticates to the backend — never a static, long-lived cloud credential stored as a cluster secret to fetch other secrets.

## Common Mistakes

- Assuming `kubectl get secret -o yaml` output is "encrypted" because it doesn't show plaintext directly — base64 decoding is trivial and instant; treat any RBAC access to Secrets as equivalent to plaintext access.
- Committing a raw Kubernetes `Secret` manifest (not a `SealedSecret`) to a GitOps repo "just this once" — it's now permanently in git history even if later deleted from the live branch, unless history is rewritten.
- Not pairing ESO with a restart/reload mechanism, then wondering why an application keeps using a stale credential long after the upstream secret rotated — the K8s Secret object updated, but the already-running pod's in-memory env vars (set at container start) did not.
- Backing up/exporting a Sealed Secrets controller's private key insecurely (or not backing it up at all) — losing it means every previously sealed secret in git becomes permanently undecryptable, and a hard disaster-recovery gap if the cluster needs rebuilding from scratch.
- Giving the ESO controller's IAM role/service account overly broad `secretsmanager:GetSecretValue` permissions on `*` instead of scoping to the exact secret ARNs/paths it needs — turns a single compromised controller into an all-secrets-readable incident.
