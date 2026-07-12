# Day 39 — Lab: Secrets in AWS & K8s

**Goal:** Migrate a plaintext Kubernetes Secret to AWS Secrets Manager and sync it back into the cluster via External Secrets Operator — then compare the experience against Sealed Secrets and SOPS for the same value.

**Prerequisites:**
- A local Kubernetes cluster (`kind`/`minikube`) with `kubectl` and `helm`.
- AWS CLI configured with credentials that can create/read Secrets Manager secrets.
- `sops`, `age`, and `kubeseal` installed (`brew install sops age kubeseal`).
- Completed Day 38's Vault lab is helpful context but not required.

---

### Lab 1 — Start with the anti-pattern, then measure it

1. Create a plaintext Kubernetes Secret the "quick and wrong" way:
   ```bash
   kubectl create secret generic myapp-db-secret --from-literal=password='S3cr3t!'
   ```
2. Prove it's trivially readable:
   ```bash
   kubectl get secret myapp-db-secret -o jsonpath='{.data.password}' | base64 -d
   ```
3. Write down (in a notes file) exactly who/what could read this today: anyone with `get secrets` RBAC in this namespace, anyone with access to an unencrypted etcd snapshot.

**Success criteria:** You can decode the "secret" in one command, driving home why base64 is not encryption.

---

### Lab 2 — Migrate to AWS Secrets Manager

1. Delete the plaintext Secret and create the real source of truth in AWS instead:
   ```bash
   kubectl delete secret myapp-db-secret
   aws secretsmanager create-secret --name lab/myapp/db-password --secret-string '{"password":"S3cr3t!"}'
   ```
2. Confirm you can read it back via the AWS CLI:
   ```bash
   aws secretsmanager get-secret-value --secret-id lab/myapp/db-password --query SecretString --output text
   ```

**Success criteria:** The secret's authoritative copy now lives in Secrets Manager, not as a raw file or manifest anywhere in your working directory.

---

### Lab 3 — The core hands-on activity: sync via External Secrets Operator

1. Install ESO:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
   ```
2. Set up IAM auth for ESO (for a local `kind` cluster without IRSA, use a scoped IAM user's static keys stored as a Kubernetes Secret purely for this lab — note out loud why this is a lab-only shortcut and IRSA/workload identity is the real production pattern):
   ```bash
   kubectl create secret generic aws-creds -n external-secrets \
     --from-literal=access-key=<AWS_ACCESS_KEY> \
     --from-literal=secret-access-key=<AWS_SECRET_KEY>
   ```
3. Create a `SecretStore`:
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: SecretStore
   metadata:
     name: aws-secretsmanager
   spec:
     provider:
       aws:
         service: SecretsManager
         region: ap-south-1
         auth:
           secretRef:
             accessKeyIDSecretRef:
               name: aws-creds
               namespace: external-secrets
               key: access-key
             secretAccessKeySecretRef:
               name: aws-creds
               namespace: external-secrets
               key: secret-access-key
   ```
4. Create an `ExternalSecret`:
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: myapp-db-secret
   spec:
     refreshInterval: 1m
     secretStoreRef:
       name: aws-secretsmanager
       kind: SecretStore
     target:
       name: myapp-db-secret
     data:
       - secretKey: password
         remoteRef:
           key: lab/myapp/db-password
           property: password
   ```
5. Apply both and confirm the native Secret reappears, this time managed by ESO:
   ```bash
   kubectl apply -f secretstore.yaml -f externalsecret.yaml
   kubectl get secret myapp-db-secret -o jsonpath='{.data.password}' | base64 -d
   ```
6. Rotate the value directly in AWS and confirm ESO picks it up within `refreshInterval`:
   ```bash
   aws secretsmanager put-secret-value --secret-id lab/myapp/db-password --secret-string '{"password":"RotatedPass!"}'
   sleep 70
   kubectl get secret myapp-db-secret -o jsonpath='{.data.password}' | base64 -d
   ```

**Success criteria:** The K8s Secret's value changes automatically after you rotate it in AWS, with zero manual `kubectl` edits — proving ESO is a live, continuous sync, not a one-time copy.

---

### Lab 4 — Compare against Sealed Secrets and SOPS for the same value

1. **Sealed Secrets**: install the controller, then seal the same password:
   ```bash
   kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml
   kubectl create secret generic myapp-db-secret-v2 --from-literal=password='S3cr3t!' --dry-run=client -o yaml | kubeseal --format yaml > sealedsecret.yaml
   cat sealedsecret.yaml   # confirm it's ciphertext, safe to commit
   kubectl apply -f sealedsecret.yaml
   ```
2. **SOPS + age**: encrypt the same value as a standalone file:
   ```bash
   age-keygen -o key.txt
   cat > .sops.yaml <<EOF
   creation_rules:
     - path_regex: secrets/.*\.yaml$
       age: $(grep 'public key' key.txt | awk '{print $NF}')
   EOF
   mkdir secrets
   echo "password: S3cr3t!" > secrets/db.yaml
   sops -e -i secrets/db.yaml
   cat secrets/db.yaml   # confirm the value is now ENC[...], key stays readable
   ```
3. Write one paragraph comparing the three approaches (ESO, Sealed Secrets, SOPS) for this specific secret: which required a live external dependency at decrypt time, which was safest to commit directly to git, and which gave the most readable diff.

**Success criteria:** You have three working artifacts (an `ExternalSecret`, a `SealedSecret`, and a SOPS-encrypted file) for the same underlying value, and can articulate the trade-off between them from direct experience.

---

### Cleanup

```bash
kubectl delete externalsecret myapp-db-secret
kubectl delete secret myapp-db-secret myapp-db-secret-v2 aws-creds -n external-secrets --ignore-not-found
aws secretsmanager delete-secret --secret-id lab/myapp/db-password --force-delete-without-recovery
rm -rf secrets sealedsecret.yaml key.txt .sops.yaml
kind delete cluster   # if created solely for this lab
```

### Stretch challenge

Wire up a `Reloader`-style annotation (or write a tiny script) that watches for the `myapp-db-secret` Kubernetes Secret changing and triggers a rolling restart of a dummy Deployment consuming it — proving that ESO's sync alone is not enough; running pods need an explicit mechanism to pick up rotated values.
