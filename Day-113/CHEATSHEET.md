# Day 113 — Cheatsheet: Crossplane & Self-Service Infra

## Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
kubectl get pods -n crossplane-system
kubectl crossplane version         # if the kubectl plugin is installed
```

## Providers

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata: { name: provider-aws-s3 }
spec: { package: xpkg.upbound.io/upbound/provider-aws-s3:v1 }
```
```bash
kubectl get providers                 # list installed providers + health
kubectl get providerrevisions          # installed package revisions
```

## ProviderConfig (credentials)

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata: { name: aws-provider-config }
spec:
  credentials:
    source: Secret
    secretRef: { namespace: crossplane-system, name: aws-creds, key: creds }
```

## Raw managed resource (direct provisioning)

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata: { name: my-bucket }
spec:
  forProvider: { region: us-east-1 }
  providerConfigRef: { name: aws-provider-config }
```

## XRD (platform-owned API)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata: { name: xpostgresqlinstances.database.example.org }
spec:
  group: database.example.org
  names: { kind: XPostgreSQLInstance, plural: xpostgresqlinstances }
  claimNames: { kind: PostgreSQLInstance, plural: postgresqlinstances }
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  size: { type: string, enum: [small, medium, large] }
```

## Composition (implementation)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata: { name: xpostgresqlinstances-aws }
spec:
  compositeTypeRef: { apiVersion: database.example.org/v1alpha1, kind: XPostgreSQLInstance }
  resources:
  - name: rdsinstance
    base:
      apiVersion: rds.aws.upbound.io/v1beta1
      kind: Instance
      spec:
        forProvider: { engine: postgres, instanceClass: db.t3.micro }
    patches:
    - fromFieldPath: "spec.parameters.size"
      toFieldPath: "spec.forProvider.instanceClass"
      transforms:
      - type: map
        map: { small: db.t3.micro, medium: db.t3.large, large: db.r5.xlarge }
```

## Claim (what the developer actually creates)

```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata: { name: my-app-db }
spec:
  parameters: { size: small }
```

## Useful get/describe commands

```bash
kubectl get providers
kubectl get providerconfigs
kubectl get compositeresourcedefinitions
kubectl get compositions
kubectl get postgresqlinstance                       # the Claim (namespaced)
kubectl get xpostgresqlinstance                        # the Composite Resource (cluster-scoped)
kubectl get instance.rds.aws.upbound.io                 # the concrete managed resource
kubectl describe postgresqlinstance my-app-db           # trace status/events for the whole chain
kubectl get managed                                       # all managed resources across providers, one shot
```

## Crossplane vs Terraform, one line each

```
Terraform : plan/apply on demand, state file, drift caught on next run
Crossplane: continuous in-cluster reconciliation, no state file, drift caught immediately
```
