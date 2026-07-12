# Day 74 — Compliance & Audit: CIS Benchmarks

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** DevSecOps

## Brief

CIS (Center for Internet Security) Benchmarks are the industry's most widely referenced hardening standards — vendor-neutral, community-developed, and frequently cited directly in SOC2 and PCI-DSS audits as "reasonable baseline security configuration." Any DevOps engineer working with cloud or Kubernetes infrastructure will eventually be asked "is this CIS compliant?" and the honest answer requires knowing what CIS Benchmarks actually check, which tools automate the check, and — critically — that passing a CIS scan is necessary but not sufficient for real security. This is also one of the most concretely testable interview topics in DevSecOps because the tooling (kube-bench, Prowler, Checkov) produces objective pass/fail output you can screenshot and discuss.

This day is split into three files:

1. **This file** — what CIS Benchmarks are, how kube-bench and Checkov automate checking them for Kubernetes/Linux/IaC.
2. **[02-README-SOC2-GDPR-Controls.md](02-README-SOC2-GDPR-Controls.md)** — SOC2 Trust Service Criteria and GDPR technical controls relevant to DevOps.
3. **[03-README-Compliance-As-Code.md](03-README-Compliance-As-Code.md)** — audit logging strategy, Prowler, and AWS Security Hub standards.

## What a CIS Benchmark actually is

A CIS Benchmark is a document — hundreds of pages for something like the CIS Kubernetes Benchmark — containing numbered, testable recommendations, each tagged as:

- **Level 1** — basic hardening with minimal functional impact, should apply almost everywhere (e.g., "ensure the API server `--anonymous-auth` is set to false").
- **Level 2** — stricter hardening that may reduce functionality or require more operational overhead (e.g., disabling certain debug/profiling endpoints entirely).
- **Scored vs. Not Scored** — whether automated tooling can definitively check it (scored) or whether it requires manual/organizational judgment (not scored, e.g., "ensure your incident response plan includes X").

Each recommendation has: a description, rationale, an **audit procedure** (the exact command to check current state), and a **remediation procedure** (the exact command/config to fix it). This structure is precisely why the benchmarks are automatable — `kube-bench` and similar tools are literally encodings of the audit procedures into scripts.

## CIS Kubernetes Benchmark + `kube-bench`

The CIS Kubernetes Benchmark checks control-plane and worker-node configuration — not application-layer security. It covers things like:

- API server flags (`--anonymous-auth=false`, `--authorization-mode` excludes `AlwaysAllow`, `--profiling=false`)
- etcd encryption at rest and TLS between etcd peers
- Kubelet flags (`--anonymous-auth=false`, `--read-only-port=0`, client certificate rotation)
- File permissions on control-plane config files (`/etc/kubernetes/manifests/*.yaml` should be `600`, owned by root)

```bash
# Run kube-bench as a Job on a node (self-hosted / kubeadm clusters)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench

# Target a specific CIS profile version explicitly
kube-bench run --targets master,node --benchmark cis-1.24
```

Sample output structure:

```
[INFO] 1 Master Node Security Configuration
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive
[FAIL] 1.2.1 Ensure that the --anonymous-auth argument is set to false
[WARN] 1.1.9 Ensure that the Container Network Interface file permissions are set to 600 (manual check required)
```

**Critical nuance for managed clusters (EKS/GKE/AKS):** kube-bench's control-plane checks (section 1.x) are largely **inapplicable** on EKS/GKE/AKS because the cloud provider manages the control plane and you have no access to `kube-apiserver` flags or etcd. On these platforms, only the **node** (section 4.x — kubelet configuration) and sometimes **policies** (section 5.x — RBAC, pod security) sections are meaningful, and CIS publishes separate benchmark variants (e.g., "CIS Amazon EKS Benchmark") that reflect this shared-responsibility split. Running vanilla `kube-bench` against an EKS cluster and reporting the master-node failures as real findings is a common, embarrassing mistake — those checks fail because you structurally cannot access what they're checking, not because anything is misconfigured.

## CIS AWS Benchmark: what it checks

The CIS AWS Foundations Benchmark covers account-level and service-level hardening: root account MFA, IAM password policy, CloudTrail enabled in all regions, S3 buckets not publicly accessible, security groups not allowing unrestricted ingress on port 22/3389, VPC flow logs enabled. This overlaps heavily with what **Prowler** and **AWS Security Hub's CIS standard** check automatically (see file 3) — CIS AWS is the specification, Prowler/Security Hub are implementations of the audit procedure.

## CIS checks for IaC: `Checkov`

Checkov shifts CIS-style compliance checking left, into Terraform/CloudFormation/Kubernetes YAML **before** anything is ever applied — catching a misconfigured S3 bucket or an overly permissive security group at PR time instead of after it's live and already scanned as a finding.

```bash
checkov -d ./terraform --framework terraform
```

```
Check: CKV_AWS_18: "Ensure the S3 bucket has access logging configured"
  FAILED for resource: aws_s3_bucket.data
  File: /main.tf:12-18

Check: CKV_AWS_23: "Ensure S3 bucket does not allow public read access"
  PASSED for resource: aws_s3_bucket.data
```

Wiring Checkov into CI as a blocking gate on Terraform PRs (similar to the SAST gate from Day 73) means a CIS-benchmark violation is rejected at review time, not discovered weeks later during an audit or a Prowler scan.

```yaml
# .github/workflows/terraform-checkov.yml
- name: Run Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: terraform/
    framework: terraform
    soft_fail: false     # fail the build on any finding above threshold
```

## Points to Remember

- CIS Benchmarks are structured, versioned, numbered recommendations with explicit audit and remediation procedures per item — that structure is why they're automatable.
- Level 1 vs. Level 2 = basic hardening vs. stricter hardening with potential functional trade-offs; Scored vs. Not Scored = automatable vs. requiring manual/organizational judgment.
- On managed Kubernetes (EKS/GKE/AKS), control-plane (master node) CIS checks are largely inapplicable — the cloud provider owns that layer. Use the cloud-specific benchmark variant (e.g., CIS EKS Benchmark) and expect only node/policy sections to be meaningful.
- `kube-bench` audits a running cluster's node/control-plane config against the CIS Kubernetes Benchmark; `Checkov` audits IaC source (Terraform/CFN/K8s YAML) before it's ever applied — shift-left vs. runtime compliance.
- Passing a CIS scan is a floor, not a ceiling — it catches known, common misconfigurations, not business-logic vulnerabilities or novel attack paths specific to your architecture.

## Common Mistakes

- Running plain `kube-bench run` against an EKS/GKE/AKS cluster and reporting all master-node FAILs as real findings in a compliance report, without realizing those checks are structurally inapplicable to a managed control plane.
- Treating "100% CIS compliant" as equivalent to "secure" — CIS is a baseline hardening checklist, not a substitute for threat modeling, penetration testing, or application-layer security review.
- Only scanning infrastructure after it's deployed (via kube-bench/Prowler) and never shifting the same checks left into Checkov on IaC — this means every finding is discovered live in production instead of blocked at PR time.
- Blindly remediating every Level 2 finding without evaluating the functional trade-off it documents — some Level 2 hardening steps genuinely break specific legitimate use cases and need a documented exception, not silent non-compliance.
- Not pinning/tracking which CIS Benchmark *version* (e.g., `cis-1.24` vs `cis-1.27`) a scan ran against — benchmark versions change with each Kubernetes release, and comparing scan results across versions without noting this leads to confusing "regression" reports that are really just newer/different checks.
