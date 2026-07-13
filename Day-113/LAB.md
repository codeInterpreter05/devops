# Day 113 — Lab: Crossplane & Self-Service Infra

**Goal:** Install Crossplane on a real cluster, provision cloud resources (an S3 bucket and an RDS instance) through Kubernetes-native CRDs, then build a Composite Resource/Composition abstraction so a "developer" only ever needs to fill in a size parameter — the actual assigned hands-on activity for today.

**Prerequisites:** A Kubernetes cluster (`kind` works, though provider calls will hit a real cloud account — use a disposable/sandbox AWS account, or substitute a provider you have safe test credentials for), `kubectl`, Helm, and AWS credentials with least-privilege access scoped to a throwaway sandbox account/region.

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace
kubectl get pods -n crossplane-system
```

---

### Lab 1 — Install the AWS provider and configure credentials

1. Install the S3 and RDS providers:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: pkg.crossplane.io/v1
   kind: Provider
   metadata:
     name: provider-aws-s3
   spec:
     package: xpkg.upbound.io/upbound/provider-aws-s3:v1
   ---
   apiVersion: pkg.crossplane.io/v1
   kind: Provider
   metadata:
     name: provider-aws-rds
   spec:
     package: xpkg.upbound.io/upbound/provider-aws-rds:v1
   EOF
   kubectl get providers
   ```
2. Create a Secret with your sandbox AWS credentials and a `ProviderConfig` referencing it:
   ```bash
   kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./aws-credentials.txt
   kubectl apply -f - <<EOF
   apiVersion: aws.upbound.io/v1beta1
   kind: ProviderConfig
   metadata:
     name: aws-provider-config
   spec:
     credentials:
       source: Secret
       secretRef:
         namespace: crossplane-system
         name: aws-creds
         key: creds
   EOF
   ```

**Success criteria:** `kubectl get providers` shows both providers `HEALTHY: True`, and `kubectl get providerconfig` shows your config with no errors.

---

### Lab 2 — The core hands-on activity: provision an S3 bucket and RDS instance via raw CRDs

This is the assigned hands-on activity for today — provision real infrastructure through Kubernetes, not the AWS console or a Terraform run.

1. Provision an S3 bucket:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: s3.aws.upbound.io/v1beta1
   kind: Bucket
   metadata:
     name: crossplane-lab-bucket
   spec:
     forProvider:
       region: us-east-1
     providerConfigRef:
       name: aws-provider-config
   EOF
   kubectl get bucket crossplane-lab-bucket -w
   ```
2. Provision an RDS instance:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: rds.aws.upbound.io/v1beta1
   kind: Instance
   metadata:
     name: crossplane-lab-db
   spec:
     forProvider:
       region: us-east-1
       engine: postgres
       engineVersion: "15"
       instanceClass: db.t3.micro
       allocatedStorage: 20
       username: labadmin
       passwordSecretRef:
         namespace: crossplane-system
         name: db-password
         key: password
       skipFinalSnapshot: true
     providerConfigRef:
       name: aws-provider-config
   EOF
   kubectl get instance.rds.aws.upbound.io crossplane-lab-db -w
   ```
3. Confirm both resources report `READY: True` and `SYNCED: True`, then verify they exist in the AWS console (or via `aws s3 ls` / `aws rds describe-db-instances`).
4. Manually delete the S3 bucket from the AWS console, then watch Crossplane's next reconcile loop recreate it (or report a diff, depending on your `managementPolicies` setting) — this is the drift-correction behavior described in the notes.

**Success criteria:** Both resources show `READY`/`SYNCED` in `kubectl get`, exist in AWS, and you've directly observed Crossplane reacting to manual out-of-band drift.

---

### Lab 3 — Build the self-service abstraction: XRD + Composition

1. Define the platform-owned `XPostgreSQLInstance` API (the XRD) and a matching `Composition` that maps `size: small` onto the concrete `Instance` resource from Lab 2 (encryption on, `db.t3.micro`, `allocatedStorage: 20`) and `size: large` onto a bigger, encrypted, multi-AZ configuration.
2. Apply the XRD and Composition.
3. As "the developer," create only this:
   ```yaml
   apiVersion: database.example.org/v1alpha1
   kind: PostgreSQLInstance
   metadata:
     name: my-app-db
   spec:
     parameters:
       size: small
   ```
4. Confirm the full chain: `kubectl get postgresqlinstance`, then `kubectl get xpostgresqlinstance`, then `kubectl get instance.rds.aws.upbound.io` — trace the same request through Claim → Composite Resource → concrete managed resource.

**Success criteria:** You can explain, pointing at each `kubectl get` output, exactly how one `size: small` field turned into a specific, policy-compliant RDS configuration without the developer ever writing an `Instance` CRD.

---

### Cleanup

```bash
kubectl delete postgresqlinstance my-app-db
kubectl delete instance.rds.aws.upbound.io crossplane-lab-db
kubectl delete bucket crossplane-lab-bucket
kubectl delete composition xpostgresqlinstances.database.example.org
kubectl delete compositeresourcedefinition xpostgresqlinstances.database.example.org
kubectl delete providerconfig aws-provider-config
kubectl delete secret aws-creds -n crossplane-system
helm uninstall crossplane -n crossplane-system
```

### Stretch challenge

Set `spec.deletionPolicy`/`managementPolicies` on the RDS `Instance` so that deleting the `Claim` **orphans** the real RDS instance instead of deleting it, then verify by deleting the Claim and confirming the database still exists in AWS afterward — explain in one paragraph why a platform team would deliberately want this behavior for production databases but not for cheap, disposable resources like S3 buckets used for scratch data.
