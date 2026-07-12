# Day 39 — Cheatsheet: Secrets in AWS & K8s

## AWS Secrets Manager

```bash
aws secretsmanager create-secret --name prod/myapp/db-password --secret-string '{"password":"S3cr3t!"}'
aws secretsmanager get-secret-value --secret-id prod/myapp/db-password --query SecretString --output text
aws secretsmanager put-secret-value --secret-id prod/myapp/db-password --secret-string '{"password":"New!"}'
aws secretsmanager rotate-secret --secret-id prod/myapp/db-password --rotation-lambda-arn <arn>
aws secretsmanager delete-secret --secret-id prod/myapp/db-password --force-delete-without-recovery
```

## SSM Parameter Store

```bash
aws ssm put-parameter --name /myapp/prod/db-password --value 'S3cr3t!' --type SecureString --key-id alias/myapp-key
aws ssm get-parameter --name /myapp/prod/db-password --with-decryption --query Parameter.Value --output text
aws ssm get-parameters-by-path --path /myapp/prod/ --recursive --with-decryption
aws ssm delete-parameter --name /myapp/prod/db-password
```

## Kubernetes Secret basics (and why they're not enough)

```bash
kubectl create secret generic myapp --from-literal=password='S3cr3t!'
kubectl get secret myapp -o jsonpath='{.data.password}' | base64 -d   # trivial to decode — NOT encryption
```

## External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: { name: aws-secretsmanager }
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef: { name: eso-sa }
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: myapp-db-secret }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: aws-secretsmanager, kind: SecretStore }
  target: { name: myapp-db-secret }
  data:
    - secretKey: password
      remoteRef: { key: prod/myapp/db-password, property: password }
```

## Sealed Secrets (Bitnami)

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml

kubectl create secret generic myapp --from-literal=password='S3cr3t!' --dry-run=client -o yaml > secret.yaml
kubeseal --format yaml < secret.yaml > sealedsecret.yaml
kubectl apply -f sealedsecret.yaml    # controller decrypts in-cluster automatically
kubeseal --fetch-cert > pub-cert.pem   # save/back up the public cert for offline sealing
```

## SOPS + age

```bash
age-keygen -o key.txt

# .sops.yaml
# creation_rules:
#   - path_regex: secrets/.*\.yaml$
#     age: <public-key>

sops secrets/prod.yaml           # open decrypted in $EDITOR, re-encrypts on save
sops -e -i secrets/prod.yaml      # encrypt in place
sops -d secrets/prod.yaml         # decrypt to stdout
sops -d secrets/prod.yaml | kubectl apply -f -   # decrypt + apply, no plaintext on disk
```

## Secret scanning / hygiene

```bash
gitleaks detect --source . -v
trufflehog filesystem .
git-secrets --scan
```

```dockerfile
# BuildKit secret mount (safe build-time secret usage — NOT persisted in image layers)
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=db_password \
    DB_PASSWORD=$(cat /run/secrets/db_password) ./configure.sh
```
```bash
DOCKER_BUILDKIT=1 docker build --secret id=db_password,src=./db_password.txt .
```
