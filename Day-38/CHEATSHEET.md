# Day 38 — Cheatsheet: HashiCorp Vault

## Server / seal lifecycle

```bash
vault server -config=config.hcl                 # start server (production mode)
vault server -dev                                # dev mode: in-memory, auto-unsealed, NEVER for prod
vault status                                     # sealed? initialized? HA status?

vault operator init -key-shares=5 -key-threshold=3
vault operator unseal <key>                      # repeat until threshold met
vault operator seal                               # manual emergency seal
vault login <token>
```

## Auto-unseal (production pattern)

```hcl
seal "awskms" {
  region     = "ap-south-1"
  kms_key_id = "alias/vault-unseal-key"
}
```

## KV secrets engine (static)

```bash
vault secrets enable -path=secret kv-v2
vault kv put secret/myapp/config db_user=admin db_password=S3cr3t
vault kv get secret/myapp/config
vault kv get -version=2 secret/myapp/config
vault kv delete secret/myapp/config
vault kv destroy -versions=1 secret/myapp/config
```

## AWS secrets engine (dynamic)

```bash
vault secrets enable -path=aws aws
vault write aws/config/root access_key=... secret_key=... region=ap-south-1
vault write aws/roles/s3-read-only credential_type=iam_user policy_document=@policy.json
vault read aws/creds/s3-read-only
```

## Database secrets engine (dynamic)

```bash
vault secrets enable database
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/postgres" \
  allowed_roles="readonly" username=vaultadmin password=...

vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl=1h max_ttl=24h

vault read database/creds/readonly
vault write -f database/rotate-root/mydb          # rotate Vault's own DB admin credential
```

## PKI secrets engine

```bash
vault secrets enable pki
vault write pki/root/generate/internal common_name="internal.example.com" ttl=87600h
vault write pki/roles/web-servers allowed_domains="example.com" allow_subdomains=true max_ttl=720h
vault write pki/issue/web-servers common_name="app.example.com" ttl=24h
```

## Leases

```bash
vault read aws/creds/s3-read-only        # note lease_id in output
vault lease renew <lease_id>
vault lease revoke <lease_id>
vault lease revoke -prefix database/creds/readonly/   # revoke all leases under a path
```

## Auth methods

```bash
# Kubernetes
vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc:443" kubernetes_ca_cert=@ca.crt token_reviewer_jwt=@reviewer-token
vault write auth/kubernetes/role/myapp bound_service_account_names=myapp-sa bound_service_account_namespaces=default policies=myapp-policy ttl=1h

# AWS IAM
vault auth enable aws
vault write auth/aws/role/web-role auth_type=iam bound_iam_principal_arn="arn:aws:iam::123456789012:role/web-instance-role" policies=web-policy ttl=1h
```

## Policies

```hcl
# policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
path "database/creds/readonly" {
  capabilities = ["read"]
}
```

```bash
vault policy write myapp-policy policy.hcl
vault policy list
vault policy read myapp-policy
```

## Vault Agent Injector annotations (Kubernetes)

```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/readonly"
  vault.hashicorp.com/agent-inject-template-db-creds: |
    {{- with secret "database/creds/readonly" -}}
    DB_USER={{ .Data.username }}
    DB_PASSWORD={{ .Data.password }}
    {{- end -}}
```

## Helm install (Agent Injector + server)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault --set "injector.enabled=true"
```
