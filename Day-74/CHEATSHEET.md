# Day 74 — Cheatsheet: Compliance & Audit

## Prowler

```bash
pip install prowler
prowler aws                                    # full scan, current profile
prowler aws --compliance cis_2.0_aws           # only CIS-mapped checks
prowler aws --compliance soc2_aws              # only SOC2-mapped checks
prowler aws --compliance gdpr_aws              # only GDPR-mapped checks
prowler aws -M csv,html,json-ocsf -o ./report/ # multiple output formats
prowler aws -c s3_bucket_public_access         # run a single specific check
prowler aws -f us-east-1,eu-west-1             # scope to specific regions
prowler aws -l                                 # list all available checks
```

## kube-bench

```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench

kube-bench run --targets master,node          # explicit target
kube-bench run --benchmark cis-1.24           # explicit CIS profile version
kube-bench run --benchmark eks-1.2.0          # EKS-specific profile (managed control plane)
kube-bench run --benchmark gke-1.2.0          # GKE-specific profile
```

## Checkov

```bash
checkov -d ./terraform --framework terraform
checkov -f main.tf                             # single file
checkov -d . --framework kubernetes            # scan K8s manifests
checkov -d . --skip-check CKV_AWS_18           # skip a specific check
checkov -d . --compact --quiet                 # concise CI-friendly output
checkov -d . --soft-fail                       # report only, don't fail build
```

## AWS Security Hub

```bash
aws securityhub enable-security-hub
aws securityhub batch-enable-standards --standards-subscription-requests \
  '[{"StandardsArn":"arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0"}]'
aws securityhub get-findings --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'
aws securityhub get-enabled-standards
```

## CloudTrail (audit logging baseline)

```bash
aws cloudtrail create-trail --name org-trail --s3-bucket-name org-cloudtrail-logs \
  --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging --name org-trail
aws cloudtrail put-event-selectors --trail-name org-trail \
  --event-selectors '[{"ReadWriteType":"All","IncludeManagementEvents":true}]'
aws cloudtrail validate-logs --trail-arn <arn> --start-time <ts>   # verify tamper-evidence digests
```

## Common account hardening one-liners

```bash
aws s3control put-public-access-block --account-id <id> \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws iam update-account-password-policy \
  --minimum-password-length 14 --require-symbols --require-numbers \
  --max-password-age 90 --password-reuse-prevention 5

aws ec2 enable-ebs-encryption-by-default

aws iam get-account-summary | grep MFA        # check root/account MFA status
```

## SOC2 Trust Service Criteria (quick recall)

```
Security          (mandatory) — access control, encryption, vuln mgmt, IR
Availability       — uptime SLAs, DR, monitoring
Processing Integrity — data processed accurately/completely
Confidentiality     — protection of data marked confidential
Privacy            — PII collection/use/retention/disposal
```

## GDPR technical control checklist

```
[ ] Data residency: EU personal data in EU region (or valid transfer mechanism, e.g. SCCs)
[ ] Right to erasure: real delete capability across DB + caches + warehouse + logs
[ ] Data minimization: no indefinite retention of raw PII in logs/analytics
[ ] Breach notification: detection + alerting fast enough to notify within 72h
[ ] Encryption/pseudonymization: at rest, in transit, and in non-prod/analytics copies
```
