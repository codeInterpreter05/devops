# Day 26 — Cheatsheet: RBAC & Security

## ServiceAccounts

```bash
kubectl create serviceaccount <name> -n <ns>
kubectl get serviceaccounts -n <ns>
kubectl describe sa <name> -n <ns>
```

```yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: monitoring-reader   # omit -> uses "default" SA silently
```

## Role / ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role                          # namespaced-resource permissions, scoped to one namespace
metadata: { name: pod-reader, namespace: monitoring }
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole                    # required for cluster-scoped resources (nodes, pv, namespaces)
metadata: { name: node-reader }
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list"]
```

## RoleBinding / ClusterRoleBinding

```bash
kubectl create rolebinding <name> --role=pod-reader \
  --serviceaccount=<ns>:<sa-name> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=node-reader --group=sre-team
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding                    # can reference a ClusterRole to scope it to ONE namespace
metadata: { name: binding, namespace: monitoring }
subjects:
  - kind: ServiceAccount
    name: monitoring-reader
    namespace: monitoring
roleRef: { kind: Role, name: pod-reader, apiGroup: rbac.authorization.k8s.io }
```

## Debugging RBAC

```bash
kubectl auth can-i create pods -n <ns>
kubectl auth can-i create pods --as=system:serviceaccount:<ns>:<sa>
kubectl auth can-i '*' '*' --as=<user>              # check for cluster-admin-equivalent access
kubectl get rolebindings,clusterrolebindings -A -o wide
```

## IRSA (EKS)

```bash
eksctl utils associate-iam-oidc-provider --cluster <cluster> --approve
eksctl create iamserviceaccount --cluster <cluster> --namespace <ns> --name <sa> \
  --attach-policy-arn <policy-arn> --approve
kubectl get sa <sa> -n <ns> -o yaml | grep role-arn
kubectl exec <pod> -- env | grep AWS_               # AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
kubectl exec <pod> -- aws sts get-caller-identity     # confirms assumed-role session, not static keys
```

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-role
```

## Pod Security Admission

```bash
kubectl label namespace <ns> pod-security.kubernetes.io/enforce=restricted
kubectl label namespace <ns> pod-security.kubernetes.io/warn=restricted
kubectl label namespace <ns> pod-security.kubernetes.io/audit=restricted
```

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: { type: RuntimeDefault }
  containers:
    - securityContext:
        allowPrivilegeEscalation: false
        capabilities: { drop: ["ALL"] }
        readOnlyRootFilesystem: true
```

## Admission webhooks

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration    # or MutatingWebhookConfiguration
metadata: { name: my-policy }
webhooks:
  - name: my-policy.example.com
    clientConfig: { service: { name: policy-svc, namespace: policy-system, path: "/validate" } }
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    failurePolicy: Fail       # Fail = fail closed (safer) / Ignore = fail open (available)
    sideEffects: None
    admissionReviewVersions: ["v1"]
```
