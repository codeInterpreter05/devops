# Day 74 — Resources: Compliance & Audit

## Primary (assigned)

- **Prowler documentation** (docs.prowler.com) — the assigned starting point. Covers installation, running scans, the `--compliance` flag mapping checks to CIS/SOC2/GDPR/PCI-DSS/NIST, and output formats. Directly drives today's hands-on lab.

## Deepen your understanding

- **CIS Benchmarks — Center for Internet Security** (cisecurity.org/cis-benchmarks): free to download (registration required) for Kubernetes, AWS, Linux distros, and dozens of other platforms — the actual source specification that kube-bench/Prowler/Checkov encode.
- **AWS Well-Architected Framework — Security Pillar** (docs.aws.amazon.com/wellarchitected): explains the reasoning behind many CIS AWS Benchmark items (why MFA, why encryption defaults, why least privilege) rather than just listing checks.
- **GDPR official text, Articles 17 (erasure), 32 (security of processing), 33 (breach notification)** (gdpr-info.eu): short, readable, and worth reading directly rather than only secondhand summaries — the actual legal language clarifies scope questions engineers commonly get wrong.
- **AICPA SOC2 Trust Services Criteria** (aicpa-cima.com): the official criteria document SOC2 auditors work from — useful for understanding exactly what "CC6.1," "CC7.2," etc. reference when a Prowler/Security Hub finding cites them.

## Reference

- **kube-bench GitHub repo** (github.com/aquasecurity/kube-bench): benchmark profile list (`cis-1.24`, `eks-1.2.0`, `gke-1.2.0`, etc.) and config file structure for customizing checks.
- **Checkov policy index** (checkov.io or github.com/bridgecrewio/checkov): searchable list of all `CKV_*` check IDs across Terraform/CloudFormation/Kubernetes/Dockerfile frameworks.
- **AWS Security Hub — supported standards** (docs.aws.amazon.com/securityhub): reference for which compliance standards (CIS, PCI-DSS, NIST 800-53, AWS FSBP) are natively supported and their ARNs for `batch-enable-standards`.

## Practice

- **A personal AWS free-tier sandbox account** — the only safe place to run Prowler's writable-remediation commands and intentionally misconfigure resources (public S3 bucket, open security group) to see findings appear, then fix them and watch them disappear.
- **kind or minikube + kube-bench** — free, local, no cloud cost, and the fastest way to compare "self-hosted control plane" (most master-node checks apply) against an EKS cluster (most don't).
