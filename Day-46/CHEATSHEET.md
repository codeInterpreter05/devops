# Day 46 — Cheatsheet: K8s Operators & CRDs

## CRD basics

```bash
kubectl get crd
kubectl explain <kind>.spec --api-version=<group>/<version>
kubectl api-resources | grep <group>
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {name: <plural>.<group>}
spec:
  group: <group>
  names: {kind: <Kind>, plural: <plural>, singular: <singular>}
  scope: Namespaced   # or Cluster
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema: {type: object, properties: {...}}
```

## Reconciliation loop model

```
watch -> work queue -> reconcile(key):
  desired = read spec
  actual  = observe cluster/external state
  diff -> take minimal action
  update .status
(level-triggered: full state compared every time, idempotent, safe to re-run)
```

## Operator SDK scaffolding (Go-based)

```bash
operator-sdk init --domain example.com --repo github.com/you/my-operator
operator-sdk create api --group mygroup --version v1 --kind MyKind --resource --controller
make manifests     # regen CRD YAML + RBAC from kubebuilder markers
make install       # kubectl apply CRDs
make run           # run controller locally against current kubeconfig context
make docker-build docker-push IMG=<repo>/my-operator:tag
make deploy IMG=<repo>/my-operator:tag
```

Key kubebuilder markers:
```go
// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Required
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Ready",type=string,JSONPath=".status.ready"
```

## Owner references & finalizers

```yaml
metadata:
  ownerReferences:
  - apiVersion: postgresql.cnpg.io/v1
    kind: Cluster
    name: pg-cluster
    uid: <uid>
    controller: true
    blockOwnerDeletion: true
  finalizers:
  - mygroup.example.com/cleanup
```

## CloudNativePG (Postgres Operator)

```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.24.0.yaml
kubectl get cluster
kubectl get cluster pg-cluster -o jsonpath='{.status.currentPrimary}'
kubectl get pods -l cnpg.io/cluster=pg-cluster
kubectl get svc | grep pg-cluster        # <name>-rw / -ro / -r
kubectl patch cluster pg-cluster --type merge -p '{"spec":{"instances":4}}'
kubectl cnpg status pg-cluster            # requires kubectl-cnpg plugin
```

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: {name: pg-cluster}
spec:
  instances: 3
  storage: {size: 10Gi}
  bootstrap:
    initdb: {database: app, owner: app}
```

## Strimzi (Kafka Operator)

```bash
kubectl create namespace kafka
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl get kafka -n kafka
kubectl get kafkatopic -n kafka
kubectl get kafkauser -n kafka
```

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: orders
  labels: {strimzi.io/cluster: my-cluster}
spec:
  partitions: 3
  replicas: 3
```

## Decision cheat

```
Existing mature Operator for this software?  -> use it (CNPG, Strimzi, Prometheus Operator, cert-manager)
Genuinely custom internal lifecycle logic?    -> consider writing one (Go > Ansible > Helm, by flexibility)
Just need templated manifests, no ongoing
  reconciliation/day-2 logic?                 -> plain Helm chart is enough, skip the Operator
```
