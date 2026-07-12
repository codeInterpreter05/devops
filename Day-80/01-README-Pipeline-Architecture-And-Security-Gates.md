# Day 80 — Phase 2 Project: Full Pipeline Architecture & Security Gates

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** Review | **Flag:** 📌

## Brief

This is the Phase 2 capstone: build and publish a complete DevSecOps pipeline — GitHub → CI → ECR → Argo CD → EKS, with every security control from the last thirteen weeks actually wired in, not just described in notes. This single project is the artifact a hiring manager will click into first, and "walk me through your CI/CD pipeline and where security is enforced" is one of the most common system-design-style questions at mid/senior DevOps and platform engineer interviews — this project is, quite literally, the answer you'll give. Treat the architecture decisions here with the rigor a real platform team would, because that's exactly what's being demonstrated.

This day is split into two files:

1. **This file** — the end-to-end pipeline architecture and where each security gate lives in it.
2. **[02-README-Load-Testing-And-Documentation.md](02-README-Load-Testing-And-Documentation.md)** — proving the deployed app holds up under load, and writing the documentation that actually gets read.

## The pipeline, stage by stage

```
Developer → GitHub (trunk-based, short-lived branch) → PR opened
   │
   ▼
CI (GitHub Actions): lint → unit tests → SAST → dependency scan → build image
   → image vuln scan (gate) → SBOM → sign image → push to ECR
   │
   ▼
ECR: registry-native scan-on-push, immutable tags, lifecycle policy
   │
   ▼
Argo CD (GitOps, pull-based): watches manifests repo/path for new image tag
   → auto-sync to EKS
   │
   ▼
EKS: admission control (Kyverno/OPA) → Pod Security Admission → NetworkPolicy
   → seccomp/AppArmor → Falco (runtime detection) → K8s audit logging
```

### 1. Source & CI — shift-left gates

Every gate here runs **before** an image ever reaches a registry, catching cheap-to-fix problems early (tying directly back to Day 78's fast-feedback principles — these should still complete in a few minutes, not become the new bottleneck):

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm test

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with:
          config: p/owasp-top-ten

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npx audit-ci --critical

  build-and-scan-image:
    needs: [lint-and-test, sast, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t app:${{ github.sha }} .
      - name: Scan image (gate on critical/high)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: '1'
      - name: Generate SBOM
        run: syft app:${{ github.sha }} -o cyclonedx-json > sbom.json
      - name: Sign image
        run: cosign sign --key env://COSIGN_KEY app:${{ github.sha }}
        env:
          COSIGN_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
      - name: Push to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker tag app:${{ github.sha }} ${{ secrets.ECR_REGISTRY }}/app:${{ github.sha }}
          docker push ${{ secrets.ECR_REGISTRY }}/app:${{ github.sha }}
```

The `needs:` chain is the gate mechanism — `build-and-scan-image` only runs if lint, SAST, and dependency scanning all pass, and Trivy's `exit-code: '1'` on critical/high findings fails the job outright, blocking the push to ECR. This is the concrete, demonstrable answer to "where do you enforce security in your pipeline": **at the job-dependency level, with a non-zero exit code as the actual enforcement mechanism** — not a dashboard someone has to remember to check.

Don't skip **secret scanning** as its own gate — run `gitleaks` (or GitHub's native secret scanning/push protection) on every PR to catch a committed credential before it ever reaches a shared branch, independent of the container-focused gates above.

### 2. ECR — registry-native controls

Beyond CI-time scanning, ECR itself provides **scan-on-push**, giving you a second, independent check that doesn't rely on the pipeline being configured correctly (defense in depth: two independently-implemented scanners are less likely to share the same blind spot). Use **immutable image tags** (tag-by-digest or a strict SHA-based tagging convention) so a tag can never be silently overwritten after the fact — this matters enormously for audit trail integrity: "what was actually running in production last Tuesday" must have one unambiguous answer.

### 3. Argo CD — why GitOps (pull-based) instead of CI pushing directly

A CI pipeline that runs `kubectl apply` or `helm upgrade` directly against the cluster requires giving CI long-lived cluster credentials — a genuinely large attack surface, since a compromised CI runner or leaked CI secret then has direct write access to production. **GitOps flips this**: Argo CD runs *inside* the cluster and **pulls** desired state from a Git repo (polling or via webhook), rather than an external system pushing into it. Cluster credentials never need to leave the cluster boundary; Git becomes the single, auditable source of truth for "what should be running," and every change to that desired state is a reviewable, revertible commit.

The CI pipeline's job, then, is only to **update the manifests repo** with the new image tag (via a bump commit, or automatically via **Argo CD Image Updater** watching the ECR repository for new tags) — Argo CD's own reconciliation loop does the actual deploy:

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=<account>.dkr.ecr.<region>.amazonaws.com/app
    argocd-image-updater.argoproj.io/app.update-strategy: digest
spec:
  project: default
  source:
    repoURL: https://github.com/you/app-manifests
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`selfHeal: true` is worth calling out explicitly in an interview: it means Argo CD actively reverts manual `kubectl edit`-style drift back to what's declared in Git, which is exactly the property that makes Git the actual source of truth rather than an aspirational one.

### 4. EKS admission & runtime — the last line of defense

Even a perfectly-scanned, signed image can be deployed with a dangerous pod spec — **admission control** is where you enforce policy on the spec itself, independent of image content. Kyverno (or OPA/Gatekeeper) policies as actual code:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign-signature
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - imageReferences: ["<account>.dkr.ecr.*.amazonaws.com/*"]
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      <cosign public key>
                      -----END PUBLIC KEY-----
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged-and-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged
      match:
        resources: { kinds: [Pod] }
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            =(securityContext):
              =(runAsNonRoot): "true"
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

This is where **Day 79's runtime security controls plug directly into this pipeline**: the same `RuntimeDefault` seccomp/AppArmor profiles and non-root enforcement from that day become Kyverno-enforced requirements here (not just best-practice suggestions in a README), NetworkPolicies default-deny by namespace, and Falco runs as a DaemonSet watching everything that *does* make it past every prior gate — because no set of pre-deploy controls is complete without a detective layer for what they miss.

## Points to Remember

- Full lifecycle coverage is the actual differentiator: **shift-left** (SAST/dependency/secret scanning pre-build) + **build-time** (image scan, SBOM, signing) + **admission-time** (policy-engine gating at deploy) + **runtime** (Falco, seccomp/AppArmor, audit logs) — a pipeline demonstrating all four layers is a materially stronger portfolio artifact than one that only scans and stops.
- GitOps (Argo CD pulling from Git) avoids giving CI direct cluster credentials — this is a security architecture decision, not just a deployment convenience, and you should be able to explain *why* in exactly those terms.
- Enforcement happens through concrete mechanisms — a `needs:` job dependency chain plus a non-zero exit code in CI, `validationFailureAction: Enforce` in Kyverno — not through a dashboard or a document that says a check "should" happen.
- Immutable image tags (by digest or strict SHA convention) are what make "what was running in production at time X" an answerable, auditable question.
- `selfHeal: true` in Argo CD's sync policy is what makes Git the actual (not just aspirational) source of truth, by reverting manual cluster drift automatically.

## Common Mistakes

- Building a pipeline that scans for vulnerabilities but never actually fails the build on a critical finding (missing `exit-code: '1'` or equivalent) — a scan that only produces a report nobody reads is not a gate.
- Giving the CI pipeline direct, long-lived cluster-admin credentials to `kubectl apply` from CI instead of adopting a GitOps pull model — this is the single most common "explain your architecture" interview follow-up trap: "why not just have CI deploy directly," and the credential-exposure answer is the one that demonstrates real security thinking.
- Treating admission control (Kyverno/OPA) as redundant with CI-time image scanning — they check different things (image content vs. pod spec) at different, complementary points in the lifecycle, and skipping either leaves a real gap.
- Forgetting that a signed-and-scanned image can still be run with a dangerous spec (privileged, root, no resource limits) if nothing enforces pod-spec policy at admission time — signing answers "is this the image we built," not "is this pod spec safe to run."
- Presenting this as a static architecture diagram without being able to point to the actual enforcing line of config for each gate — an interviewer probing this project will ask "show me exactly where that's enforced," and "it's a policy we follow" is a much weaker answer than a specific YAML field.
