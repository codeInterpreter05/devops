# Day 38 — Lab: HashiCorp Vault

**Goal:** Run Vault locally, unseal it properly (not in dev mode), enable the AWS secrets engine to generate short-lived AWS credentials, and inject a dynamic secret into a Kubernetes pod via the External Secrets Operator (ESO) so the pod never sees a static credential.

**Prerequisites:**
- Vault CLI installed (`brew install vault` or download from releases.hashicorp.com).
- Docker + a local Kubernetes cluster (`kind create cluster` or `minikube start`).
- `kubectl` and `helm` installed.
- An AWS account with a low-privilege IAM user you're comfortable letting Vault manage for the AWS secrets engine test (or skip Lab 2 and use dev-mode only if you don't want to touch real AWS).

---

### Lab 1 — Run Vault locally and go through a real unseal ceremony

1. Start Vault in server mode (not `vault server -dev`, which auto-unseals and defeats the point of this lab) with a local file storage backend:
   ```bash
   mkdir -p ~/vault-lab/data
   cat > ~/vault-lab/config.hcl <<'EOF'
   storage "file" {
     path = "/vault-lab/data"
   }
   listener "tcp" {
     address     = "127.0.0.1:8200"
     tls_disable = "true"
   }
   EOF
   vault server -config=~/vault-lab/config.hcl &
   export VAULT_ADDR="http://127.0.0.1:8200"
   ```
2. Initialize with 5 key shares, threshold 3:
   ```bash
   vault operator init -key-shares=5 -key-threshold=3 > ~/vault-lab/init-output.txt
   cat ~/vault-lab/init-output.txt
   ```
3. Confirm it's sealed:
   ```bash
   vault status
   ```
4. Unseal using any 3 of the 5 keys from the output file:
   ```bash
   vault operator unseal <key-1>
   vault operator unseal <key-2>
   vault operator unseal <key-3>
   vault status   # should now show sealed = false
   ```
5. Log in with the root token from the init output:
   ```bash
   vault login <root-token>
   ```

**Success criteria:** `vault status` shows `Sealed: false` only after supplying exactly the threshold number of distinct unseal keys — confirm supplying only 2 keys leaves it sealed.

---

### Lab 2 — Enable the AWS secrets engine and generate short-lived credentials

1. Enable the engine and configure it with a bootstrap IAM user's keys:
   ```bash
   vault secrets enable -path=aws aws
   vault write aws/config/root access_key=<AWS_ACCESS_KEY> secret_key=<AWS_SECRET_KEY> region=ap-south-1
   ```
2. Create a role scoped to read-only S3 access:
   ```bash
   vault write aws/roles/s3-read-only credential_type=iam_user policy_document=-<<EOF
   {
     "Version": "2012-10-17",
     "Statement": [{"Effect": "Allow", "Action": ["s3:GetObject","s3:ListBucket"], "Resource": "*"}]
   }
   EOF
   ```
3. Request a credential and inspect the lease:
   ```bash
   vault read aws/creds/s3-read-only
   vault lease renew <lease_id_from_output>
   ```
4. Verify in the AWS IAM console that a new, temporary IAM user was actually created.
5. Revoke it manually before it expires and confirm it's gone from IAM:
   ```bash
   vault lease revoke <lease_id>
   ```

**Success criteria:** You can see a Vault-generated IAM user appear in the AWS console after `vault read`, and disappear immediately after `vault lease revoke` — proving Vault is really calling the AWS API, not just faking values locally.

---

### Lab 3 — Inject Vault secrets into a Kubernetes pod via External Secrets Operator

1. Install ESO into your local cluster:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
   ```
2. Put a test secret in Vault's KV engine (simpler than dynamic AWS creds for this ESO wiring exercise):
   ```bash
   vault secrets enable -path=secret kv-v2
   vault kv put secret/myapp/config api_key=demo-key-12345
   ```
3. Enable Kubernetes auth on Vault and bind a role to a test service account (requires Vault to reach your cluster's API — for `kind`, this needs some network wiring; alternatively use a Vault token-based `SecretStore` for simplicity in this lab):
   ```bash
   vault auth enable kubernetes
   ```
4. Create an ESO `SecretStore` pointing at Vault and an `ExternalSecret` referencing `secret/myapp/config`.
5. Apply both and confirm a native Kubernetes `Secret` object is created and synced:
   ```bash
   kubectl get externalsecret
   kubectl get secret myapp-config -o jsonpath='{.data.api_key}' | base64 -d
   ```

**Success criteria:** The plaintext value you put into Vault's KV engine shows up as a real Kubernetes `Secret`, created and kept in sync by ESO — with no plaintext value ever written into a YAML manifest or git.

---

### Cleanup

```bash
kubectl delete externalsecret --all
kill %1   # stop the backgrounded vault server process
rm -rf ~/vault-lab
kind delete cluster   # if you created one for this lab
```

### Stretch challenge

Swap Lab 3's static KV secret for the dynamic `database/creds/readonly` path against a real Postgres container, and confirm ESO's refresh interval actually produces a *new* username/password pair in the synced Kubernetes Secret each time the lease is refreshed.
