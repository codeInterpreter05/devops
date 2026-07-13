# Day 120 — CKS Exam Prep: Supply Chain Security & Cluster Hardening

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** 📌 Milestone

## Brief

**Supply chain security** is about everything upstream of "the pod is running": where the image came from, whether it was scanned, whether it's signed, whether the registry it was pulled from is trustworthy, and — critically — whether any of that is *enforced automatically* rather than relying on a human remembering to check. **Cluster/system hardening** is about shrinking the attack surface of the cluster and its host OS via the CIS Kubernetes Benchmark. **NetworkPolicy** reappears here as the concrete enforcement mechanism tying both together: restricting egress so that even a workload running a compromised or unscanned image can't freely reach an arbitrary internet host, is a supply-chain mitigation as much as it's a network-segmentation one.

## kube-bench: CIS Benchmark automation

The **CIS Kubernetes Benchmark** is a detailed, versioned checklist (matched to K8s minor versions — e.g., a CIS Benchmark revision targeting K8s 1.24 vs. 1.28) of specific, testable configuration checks across the control plane, etcd, kubelet, and cluster-wide policies: "ensure `--anonymous-auth` is set to `false`," "ensure the etcd data directory has permissions of `700` or more restrictive," and hundreds more like it.

**kube-bench** (Aqua Security, open source) automates running these checks against a live cluster's actual configuration files and running process flags, and reports PASS/FAIL/WARN per check along with the exact remediation.

```bash
kube-bench run --targets master,node,etcd,policies
kube-bench run --targets master              # control-plane checks only
kube-bench --version 1.24                     # pin the CIS Benchmark version to match your cluster
```

On **managed Kubernetes** (EKS/GKE/AKS), you don't control the control plane at all, so master-node checks are largely inapplicable — cloud providers publish their own benchmark variants that skip those checks and add cloud-specific ones instead (e.g., `kube-bench run --benchmark eks-1.0`). Running the vanilla master benchmark against a managed cluster produces a wall of FAILs for things you have no ability (and no need) to fix.

**The realistic exam/production workflow:** kube-bench's output frequently *is* the task — "run it, fix the top N FAILs" — which means the practiced skill is translating its remediation text directly into a manifest edit. A FAIL on `--insecure-port` becomes editing `/etc/kubernetes/manifests/kube-apiserver.yaml`, setting `--insecure-port=0`, and waiting for the kubelet to notice the changed static pod manifest and restart it — there's no `kubectl apply` involved for static pods at all.

## Trivy Operator: continuous scanning inside the cluster

**Trivy** (Aqua Security) is a single scanner covering multiple concerns at once: OS package vulnerabilities, language-dependency (e.g., `package-lock.json`, `requirements.txt`) vulnerabilities, misconfigurations in Dockerfiles/K8s manifests/Terraform, secrets accidentally baked into an image layer, and license issues. The standalone CLI (`trivy image <image>`) is what typically runs as a CI pipeline gate — but that only protects images going *through* your pipeline.

The **Trivy Operator** runs continuously *inside* the cluster instead: it installs CRDs (`VulnerabilityReport`, `ConfigAuditReport`, `ExposedSecretReport`, `RbacAssessmentReport`, `ClusterComplianceReport`), watches for Pod/Deployment/etc. creation, triggers scan Jobs automatically, and writes results back as those CR objects — turning "did anyone remember to scan this image" from a pipeline-dependent hope into an always-on, queryable cluster property.

```bash
# every workload cluster-wide with at least one CRITICAL CVE, without re-scanning anything manually
kubectl get vulnerabilityreports -A -o json | \
  jq '.items[] | select(.report.summary.criticalCount > 0)'
```

**Why this isn't redundant with CI-time scanning:** CI-time scanning blocks a bad image from ever being *pushed*. The Trivy Operator catches images that got into the cluster anyway — pulled directly from a registry outside your pipeline, deployed before a CVE was publicly disclosed, or running on a base image that had a new CVE disclosed *after* your image was already deployed and running. Defense in depth: one layer stops it at the door, the other keeps watching what's already inside.

## Admission control: blocking, not just reporting

Scanning and reporting alone doesn't stop a non-compliant workload from running — that requires an **admission controller** (Kyverno or OPA/Gatekeeper) that can reject the request outright at creation time.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-trusted-registry-and-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: block-untrusted-registry
    match:
      any:
      - resources: {kinds: ["Pod"]}
    validate:
      message: "Images must come from registry.internal.example.com"
      pattern:
        spec:
          containers:
          - image: "registry.internal.example.com/*"
  - name: require-image-signature
    match:
      any:
      - resources: {kinds: ["Pod"]}
    verifyImages:
    - imageReferences:
      - "registry.internal.example.com/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <cosign public key>
              -----END PUBLIC KEY-----
```

`cosign`/sigstore signs an image **by digest, not tag** (tags are mutable — someone can repoint `myapp:latest` to a different image tomorrow; a digest can't be silently swapped), either with a static keypair or, in more modern setups, "keyless" signing anchored to an OIDC identity and recorded in a public transparency log (Rekor). The policy above verifies that signature *at admission time* — an unsigned or tampered image is rejected before it ever schedules, not merely flagged for someone to notice later.

`validationFailureAction: Enforce` vs. `Audit` matters for how you roll a new policy out: a brand-new policy should generally start in `Audit` (report violations, block nothing) to catch policy bugs or legitimate-but-unanticipated workloads before you flip to `Enforce` cluster-wide — CKS-style scenarios sometimes explicitly ask for the audit-first version, mirroring real progressive-rollout practice.

## NetworkPolicy as a hardening checklist item

The default-allow-everything behavior covered on Day 119 reappears here framed differently: it's not just a service-mesh concern, it's a **cluster hardening checklist item**. Every namespace running production workloads should have, at minimum, a default-deny-ingress-and-egress policy with explicit allows layered on top — CKS-style tasks frequently hand you a namespace with zero policies and a description of exactly which flows should be permitted, expecting you to lock everything else down.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-payment-egress
  namespace: production
spec:
  podSelector:
    matchLabels: {app: checkout}
  policyTypes: ["Egress"]
  egress:
  - to: []
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
  - to:
    - podSelector: {matchLabels: {app: payment-service}}
    ports:
    - {protocol: TCP, port: 8443}
```

**The single most common NetworkPolicy mistake, in both CKS practice and real production incidents:** writing the default-deny-egress rule and forgetting the DNS-egress allow. Every pod in the namespace then loses the ability to resolve names — which frequently presents as a vague "the service is unreachable" connection error rather than an obvious DNS failure, since many HTTP client libraries surface DNS resolution failures with the same generic message as a refused TCP connection, making the real cause much harder to spot under time pressure than it should be.

## Points to Remember

- kube-bench automates CIS Kubernetes Benchmark checks against live config; pin its benchmark version to your cluster's K8s version, and use the cloud-specific benchmark variant (not the vanilla master checks) on managed control planes you don't own.
- Trivy CLI in CI blocks bad images at push time; the Trivy Operator continuously scans what's already running in-cluster — the two are complementary, not redundant.
- Reporting (Trivy Operator, kube-bench) tells you what's wrong; admission control (Kyverno/OPA) is what actually stops a non-compliant workload from running — you need both.
- Image signing verifies a specific digest, not a mutable tag — always sign/verify by digest, since a tag can be silently repointed to different content later.
- Roll new admission policies out in `Audit` mode before flipping to `Enforce`, mirroring the same progressive-rollout discipline as any other production policy change.
- A default-deny-egress NetworkPolicy needs an explicit DNS allow rule (UDP/TCP 53) or every pod in the namespace silently loses name resolution.

## Common Mistakes

- Running kube-bench's vanilla master-node checks against a managed EKS/GKE/AKS cluster and treating every resulting FAIL as an actionable finding, instead of using the cloud-specific benchmark variant that reflects what you actually control.
- Relying solely on CI-time image scanning and assuming that's sufficient supply-chain coverage, missing that an image can become vulnerable *after* it's already deployed (new CVE disclosed against an already-running base image) with nothing left watching it.
- Deploying a Kyverno/OPA policy straight to `Enforce` on a cluster with existing workloads, discovering a policy bug or an unanticipated legitimate image pattern only after it starts blocking real deployments cluster-wide.
- Signing or verifying container images by tag instead of digest, leaving a gap where a tag can be repointed to different (potentially malicious) content after the signature was originally checked.
- Writing a default-deny-egress NetworkPolicy without an explicit DNS-egress allow, then spending exam or on-call time chasing a "connection refused"-looking failure that's actually a DNS resolution failure.
