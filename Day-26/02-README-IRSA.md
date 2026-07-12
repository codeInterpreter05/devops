# Day 26 — RBAC & Security: IRSA (IAM Roles for Service Accounts)

**Phase:** 1 – Core DevOps | **Week:** W4 | **Domain:** Kubernetes | **Flag:** ⚡ Interview-critical

## Brief

RBAC (previous file) controls access **within** the Kubernetes API. But pods routinely need to call **AWS APIs** directly — reading from S3, writing to DynamoDB, calling Secrets Manager. The old, insecure way was baking long-lived AWS access keys into a Kubernetes `Secret` and mounting them into the pod. IRSA is AWS/EKS's answer to "how do we give a specific pod AWS permissions without ever handling a static credential" — and "why is IRSA better than a Secret full of AWS keys" is one of the most direct, frequently asked interview questions in this domain because it tests real production security judgment, not just API syntax.

## The problem with AWS keys in a Kubernetes Secret

```yaml
# The pattern IRSA replaces — don't do this
apiVersion: v1
kind: Secret
metadata: { name: aws-creds }
type: Opaque
data:
  AWS_ACCESS_KEY_ID: <base64>
  AWS_SECRET_ACCESS_KEY: <base64>
```

This is bad for several concrete, specific reasons:
- **Static, long-lived credentials** — they don't expire on their own; if leaked (via a misconfigured log line, a compromised container, a careless `kubectl describe`/exec), they're valid until someone manually rotates or revokes them.
- **Secrets in `etcd` are base64, not encrypted, by default** (see Day 22) — anyone with `etcd` access or an unencrypted backup can read them directly.
- **No fine-grained scoping to a specific workload** — the same key pair often ends up shared across every pod in a namespace/cluster that "needs AWS access," rather than each workload having exactly the permissions it needs.
- **No audit trail tying API calls back to a specific Kubernetes workload** — CloudTrail sees the IAM user/key, not "this specific pod in this specific namespace."

## How IRSA actually works

IRSA is built entirely on **OIDC federation** — no custom AWS agent, no credential injection sidecar, no long-lived secret anywhere.

1. **The EKS cluster has an OIDC provider** registered in IAM (every EKS cluster gets one; `eksctl utils associate-iam-oidc-provider` sets this up if missing). This makes IAM trust tokens issued by the cluster's own OIDC issuer.
2. **You create an IAM Role with a trust policy** scoped to a specific Kubernetes ServiceAccount (namespace + name), not to the whole cluster:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE" },
       "Action": "sts:AssumeRoleWithWebIdentity",
       "Condition": {
         "StringEquals": {
           "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:prod:s3-uploader",
           "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com"
         }
       }
     }]
   }
   ```
3. **The ServiceAccount is annotated** with that IAM role's ARN:
   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: s3-uploader
     namespace: prod
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-uploader-role
   ```
4. **The EKS Pod Identity webhook** (a mutating admission controller running in every EKS cluster) sees any Pod using an annotated ServiceAccount and automatically injects two things: an environment variable (`AWS_ROLE_ARN`) and a **projected, auto-rotated OIDC token volume** (`AWS_WEB_IDENTITY_TOKEN_FILE`), mounted read-only into the pod — no manual volume config required.
5. **The AWS SDK inside your application** (any modern SDK version — this is standard `WebIdentityToken` credential-chain support, zero custom code needed) automatically detects those two environment variables, reads the projected token, and calls `sts:AssumeRoleWithWebIdentity` to exchange it for **short-lived, auto-refreshed AWS temporary credentials** — typically valid for about an hour, refreshed continuously by the SDK in the background.

```bash
kubectl describe sa s3-uploader -n prod                       # confirm the role-arn annotation
kubectl exec <pod> -- env | grep AWS_                          # AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
kubectl exec <pod> -- cat $AWS_WEB_IDENTITY_TOKEN_FILE | cut -d. -f2 | base64 -d   # inspect the JWT claims
aws sts get-caller-identity                                      # from inside the pod, confirms the assumed role
```

## Why this is strictly better than a static-key Secret

| | Static AWS keys in a Secret | IRSA |
|---|---|---|
| Credential lifetime | Indefinite until manually rotated | ~1 hour, auto-refreshed |
| Where it's stored | `etcd` (base64, not encrypted by default) | Never stored — a JWT is exchanged live via STS each time |
| Scope | Whatever the key pair was originally granted; often shared | Per-ServiceAccount, via IAM trust policy `sub` condition — least privilege by workload |
| Leak blast radius | Full key usable from anywhere, until revoked | Stolen token still requires the specific ServiceAccount identity and is short-lived |
| Rotation operational burden | Manual (or a separate rotation pipeline you must build) | None — handled entirely by STS/SDK |
| Audit trail | Shows the IAM user/key, not the workload | CloudTrail can show the assumed-role session tied to the originating ServiceAccount |

## Points to Remember

- IRSA maps a specific Kubernetes ServiceAccount (namespace + name) to a specific IAM role, via OIDC federation and an IAM trust policy condition on the `sub` claim — not a whole cluster or namespace at a time.
- No static AWS credentials are ever created, stored, or transmitted — the pod gets a short-lived OIDC token (auto-rotated), which the AWS SDK exchanges for temporary credentials via `sts:AssumeRoleWithWebIdentity`.
- The Pod Identity webhook (a mutating admission controller) is what actually injects the environment variables and token volume — this happens automatically for any pod using an IRSA-annotated ServiceAccount, no extra pod spec configuration needed.
- The trust policy's `sub` condition is the actual security boundary — get this wrong (too broad a match, e.g. matching all ServiceAccounts in a namespace) and you've handed the IAM role's permissions to more workloads than intended.
- IRSA credentials are short-lived and auto-refreshed by the SDK — you never manually rotate anything, which eliminates an entire category of "we forgot to rotate this key" incidents.

## Common Mistakes

- Writing the trust policy's `sub` condition too broadly (e.g., matching an entire namespace via a wildcard instead of a specific ServiceAccount name), inadvertently granting the IAM role's permissions to every pod in that namespace, not just the intended one.
- Forgetting to actually annotate the ServiceAccount with `eks.amazonaws.com/role-arn` and then being confused why the pod has no AWS access at all — the Pod Identity webhook only acts on annotated ServiceAccounts; an unannotated one gets no injected credentials.
- Assuming an older/vendored AWS SDK version automatically supports the web identity token credential chain — very old SDK versions may need an update; this is a rare but real "IRSA is configured correctly but the app still can't authenticate" root cause.
- Mixing IRSA with a manually mounted static-key Secret "just in case" — most SDKs check for static credentials in certain environment variables first in their credential chain, so a leftover key can silently override IRSA's short-lived credentials without anyone noticing, defeating the whole point.
- Not rotating/reviewing IAM role trust policies and permissions as workloads change — IRSA solves credential rotation, but it doesn't automatically solve "this role's permissions have grown too broad over 18 months of ad-hoc additions"; least-privilege still requires ongoing review.
