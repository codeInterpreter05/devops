# Day 74 — Lab: Compliance & Audit

**Goal:** Run Prowler against a real AWS account, fix the top 10 findings, and generate a compliance report you could hand to an auditor — plus validate Kubernetes/IaC compliance with kube-bench and Checkov.

**Prerequisites:**
- An AWS account you're authorized to scan (a personal sandbox account is ideal — **do not run this against an employer's production account without explicit permission**).
- AWS CLI configured with credentials that have at minimum `SecurityAudit` managed policy attached.
- Python 3.9+ (`pip install prowler`), Docker, and a kind/minikube cluster with `kube-bench` access.
- Terraform code from an earlier Phase 1 day, if available, for the Checkov exercise.

---

### Lab 1 — Run Prowler and read the output

1. Attach the `SecurityAudit` AWS managed policy to your IAM user/role (read-only, safe for scanning):
   ```bash
   aws iam attach-user-policy --user-name <you> --policy-arn arn:aws:iam::aws:policy/SecurityAudit
   ```
2. Run a full scan:
   ```bash
   prowler aws -M csv,html,json-ocsf -o ./prowler-report/
   ```
3. Open the HTML report. Sort by severity. Identify the count of CRITICAL and HIGH findings.
4. Pick 3 findings and, for each, find the exact remediation command in the report's detail view.

**Success criteria:** You have a full Prowler HTML report and can explain, for any 3 findings, exactly what's wrong and the exact AWS CLI/console fix.

---

### Lab 2 — Fix the top 10 findings

1. From the sorted report, work through the 10 highest-severity findings. Typical candidates and fixes:
   ```bash
   # Enable account-level S3 Block Public Access
   aws s3control put-public-access-block --account-id <id> \
     --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

   # Enforce a stronger IAM password policy
   aws iam update-account-password-policy \
     --minimum-password-length 14 --require-symbols --require-numbers \
     --require-uppercase-characters --require-lowercase-characters \
     --max-password-age 90 --password-reuse-prevention 5

   # Enable EBS encryption by default
   aws ec2 enable-ebs-encryption-by-default

   # Create a multi-region CloudTrail if missing
   aws cloudtrail create-trail --name org-trail --s3-bucket-name <bucket> \
     --is-multi-region-trail --enable-log-file-validation
   aws cloudtrail start-logging --name org-trail
   ```
2. Enable MFA on the root account via the console (this cannot be done via CLI) if it's flagged.
3. Re-run `prowler aws --compliance cis_2.0_aws` and confirm your fixed checks now show `PASS`.

**Success criteria:** At least 10 findings that were `FAIL` on the first scan show `PASS` on the re-scan, with before/after evidence saved.

---

### Lab 3 — Generate a compliance report mapped to SOC2

1. Run:
   ```bash
   prowler aws --compliance soc2_aws -M html -o ./soc2-report/
   ```
2. Open the report and find 5 checks explicitly mapped to a SOC2 Trust Service Criteria (e.g., CC6.1, CC7.2).
3. Write one paragraph, as if for an auditor, summarizing your account's current SOC2-relevant posture and open remediation items.

**Success criteria:** You can produce a report artifact and a short written summary suitable for handing to a compliance stakeholder.

---

### Lab 4 — kube-bench against your cluster

1. Run kube-bench against your kind/minikube cluster:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
   kubectl logs -f job/kube-bench
   ```
2. If using a managed cluster (EKS), instead run the EKS-specific kube-bench profile:
   ```bash
   kube-bench run --benchmark eks-1.2.0
   ```
3. Identify and explain: which section(s) are inapplicable due to it being managed (if EKS), and which node-level findings are real and actionable.

**Success criteria:** You can distinguish real findings from findings that are structural non-issues on a managed platform, and can explain why.

---

### Lab 5 — Checkov against your Terraform

1. Run Checkov against any Terraform module from a Phase 1 project:
   ```bash
   checkov -d ./terraform --framework terraform --compact
   ```
2. Fix at least 3 findings directly in the Terraform source (e.g., enabling S3 versioning, restricting a security group CIDR, enabling encryption).
3. Re-run and confirm those checks pass.
4. Add Checkov as a blocking step in a GitHub Actions workflow for that repo.

**Success criteria:** Checkov runs in CI and fails the build on a reintroduced violation (test this by deliberately reverting one fix and pushing).

---

### Cleanup

```bash
rm -rf ./prowler-report ./soc2-report
kubectl delete job kube-bench
aws iam detach-user-policy --user-name <you> --policy-arn arn:aws:iam::aws:policy/SecurityAudit   # if scanning was scoped only for this lab
```

### Stretch challenge

Schedule the Prowler scan to run weekly via a GitHub Actions `on: schedule` workflow, uploading the HTML report as a build artifact each run, and post a Slack summary (count of CRITICAL/HIGH findings, delta from last week) using the pattern from Day 73's notification job.
