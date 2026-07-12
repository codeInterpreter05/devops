# Day 38 — HashiCorp Vault: Secret Engines & Auth Methods

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets | **Flag:** ⚡ Interview-critical

## Brief

Once Vault is unsealed, two concepts do all the real work: **secret engines** (what kind of secret Vault actually manages and how — static key-value storage vs. dynamically-generated cloud/database credentials vs. certificate issuance) and **auth methods** (how a human or a workload proves its identity to Vault before it's allowed to read anything). Understanding both — and specifically how they compose with **policies** — is what separates "I've run `vault kv put` once" from actually understanding how Vault fits into a real platform.

## Secret engines — pluggable backends for different secret types

A secret engine is enabled at a **mount path** and determines what happens when you read/write under that path. Multiple engines of the same type can be mounted at different paths for different purposes.

```bash
vault secrets enable -path=secret kv-v2
vault secrets enable -path=aws aws
vault secrets enable -path=pki pki
vault secrets enable -path=database database
```

### KV (Key-Value) — static secret storage

The simplest engine: Vault just stores whatever key-value data you put in, encrypted at rest, versioned (in KV v2).

```bash
vault kv put secret/myapp/config db_user=admin db_password=S3cr3t
vault kv get secret/myapp/config
vault kv get -version=2 secret/myapp/config     # KV v2 keeps version history
vault kv delete secret/myapp/config              # soft delete (recoverable)
vault kv destroy -versions=1 secret/myapp/config # permanent, unrecoverable
```

KV is **static** — Vault doesn't generate or rotate this data itself; you (or your CI pipeline) write it in, and it sits there until you change or delete it. It's a major upgrade over "secrets in a `.env` file in git" (encryption, access control, audit log, versioning) but it's not dynamic secrets — that distinction matters for the interview question in file 3.

### AWS secrets engine — generating IAM credentials on demand

```bash
vault write aws/config/root \
  access_key=<bootstrap-key> \
  secret_key=<bootstrap-secret> \
  region=ap-south-1

vault write aws/roles/s3-read-only \
  credential_type=iam_user \
  policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [{"Effect": "Allow", "Action": "s3:GetObject", "Resource": "*"}]
}
EOF

vault read aws/creds/s3-read-only
```

Reading `aws/creds/s3-read-only` has Vault call the AWS API itself (using its own configured root/bootstrap credentials) to **create a brand-new IAM user or STS token** matching that policy, hand it to the caller with a lease/TTL, and later automatically revoke it. Nobody ever shares one long-lived AWS access key — each caller gets their own freshly-minted, time-bound credential.

### PKI secrets engine — Vault as an internal Certificate Authority

```bash
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
vault write pki/root/generate/internal common_name="internal.example.com" ttl=87600h
vault write pki/roles/web-servers allowed_domains="example.com" allow_subdomains=true max_ttl="720h"
vault write pki/issue/web-servers common_name="app.example.com" ttl="24h"
```

This turns Vault into a private CA that issues short-lived TLS certificates on demand — used heavily for internal service-to-service mTLS, where certs commonly get issued with TTLs measured in hours or days instead of the year-plus TTLs typical of public CA certs, forcing frequent rotation as a security property rather than an operational afterthought.

### Database secrets engine — dynamic DB credentials (deep dive in file 3)

```bash
vault secrets enable database
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db.internal:5432/postgres" \
  allowed_roles="readonly" \
  username="vaultadmin" password="..."

vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" max_ttl="24h"

vault read database/creds/readonly
```

## Auth methods — how identities prove who they are to Vault

Every request to Vault (except a small set of unauthenticated/status endpoints) requires a valid **token**, obtained by successfully authenticating through an **auth method**. Different auth methods suit different callers:

| Auth method | Who uses it | How identity is proven |
|---|---|---|
| Userpass / LDAP / Okta | Humans | Username + password / SSO |
| **Kubernetes** | Pods running in a K8s cluster | The pod's own service account JWT, validated by Vault against the cluster's API |
| **AWS IAM** | EC2 instances, Lambda, anything with an AWS identity | A signed AWS STS `GetCallerIdentity` request, verified by Vault calling AWS itself |
| AppRole | CI pipelines, non-human machine workloads generally | A `role_id` (like a username) + `secret_id` (like a password), often injected by the orchestrator |

### Kubernetes auth method — the common pattern for workloads in-cluster

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@ca.crt \
  token_reviewer_jwt=@reviewer-token

vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=default \
  policies=myapp-policy \
  ttl=1h
```

A pod authenticates by presenting its **projected service account token** (the JWT Kubernetes automatically mounts into every pod, see Day 40/41 territory) to Vault's `/auth/kubernetes/login` endpoint. Vault validates that JWT against the Kubernetes API's TokenReview endpoint to confirm it's a real, currently-valid token for the expected service account/namespace, then issues a Vault token scoped to the matching `policies`. This is the mechanism that lets a pod get real secrets with **zero static credentials baked into its image or environment** — its Kubernetes-native identity *is* its Vault identity.

### AWS IAM auth method — for EC2/Lambda workloads

```bash
vault auth enable aws
vault write auth/aws/role/web-role \
  auth_type=iam \
  bound_iam_principal_arn="arn:aws:iam::123456789012:role/web-instance-role" \
  policies=web-policy \
  ttl=1h
```

An EC2 instance authenticates by having Vault verify a signed `sts:GetCallerIdentity` request using the instance's own IAM instance profile credentials — again, no static Vault credential needs to be pre-distributed; the cloud-native identity (IAM role) *is* the login mechanism.

## Policies — what an authenticated identity is actually allowed to do

Every token is associated with one or more **policies** (written in HCL) that define path-level permissions:

```hcl
# myapp-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "database/creds/readonly" {
  capabilities = ["read"]
}
```

```bash
vault policy write myapp-policy myapp-policy.hcl
```

Policies are the actual authorization layer — auth methods only answer "who are you," policies answer "what are you allowed to touch." A token from any auth method with the same attached policy has identical permissions — this separation is what lets you swap/add auth methods without redesigning your whole permission model.

## Points to Remember

- Secret engines are mounted at a path and determine engine behavior: KV = static storage you manage, AWS/Database = dynamically generated credentials Vault creates and revokes, PKI = Vault acting as an internal CA issuing short-lived certs.
- Auth methods answer "who is this caller" (Kubernetes service account JWT, AWS IAM STS identity, AppRole, userpass); policies answer "what can this caller do" — they're deliberately decoupled.
- The Kubernetes auth method lets a pod authenticate using its own projected service account token, validated against the cluster's TokenReview API — no static Vault credential ever needs to live in the pod spec or image.
- Policies are HCL documents defining path-level capabilities (`read`, `create`, `update`, `delete`, `list`, `sudo`), attached to auth method roles — the same policy can be reused across different auth methods.
- KV is not the same thing as dynamic secrets — KV just stores what you put there; AWS/Database/PKI engines actively generate new, time-bound credentials on each read.

## Common Mistakes

- Calling KV-stored secrets "dynamic secrets" — they're static values Vault merely stores and versions; nothing about them is auto-generated or auto-revoked, unlike the AWS/Database/PKI engines.
- Binding a Kubernetes auth role too broadly (e.g., `bound_service_account_names=*` across all namespaces) — effectively any pod in the cluster can authenticate as that role, defeating the purpose of workload-scoped identity.
- Leaving the AWS secrets engine's root/bootstrap IAM credentials overly permissive ("just give Vault admin access to make it easy") — since Vault uses those credentials to mint all downstream dynamic credentials, an overly broad root credential undermines the least-privilege boundaries you're trying to enforce per role.
- Forgetting to set `max_lease_ttl` sensibly on secret engines — a database or AWS credential engine with no sane TTL cap can end up issuing credentials that live far longer than intended if a role's `default_ttl`/`max_ttl` aren't both explicitly set.
- Conflating "enabling an auth method" with "granting access" — enabling `kubernetes` auth just turns on the mechanism; without a `role` bound to specific service accounts/namespaces and a policy attached, nothing meaningful is actually authorized yet.
