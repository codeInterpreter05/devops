# Day 33 — Cheatsheet: AWS IAM Deep Dive

## Policy evaluation order (memorize)

```
1. Explicit Deny anywhere (identity policy, resource policy, boundary, SCP)  -> DENIED, full stop
2. No explicit deny AND no explicit allow                                    -> DENIED (implicit default)
3. No explicit deny AND an explicit Allow exists                             -> ALLOWED
```
Statement/policy JSON order does NOT matter — only Effect type + condition match.

## Basic policy structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "Bool": { "aws:MultiFactorAuthPresent": "true" },
        "StringEquals": { "aws:RequestedRegion": "ap-south-1" }
      }
    }
  ]
}
```

## Common condition keys

```
aws:RequestedRegion          # restrict to specific regions
aws:SourceIp                 # restrict by source IP/CIDR
aws:MultiFactorAuthPresent   # require MFA
aws:PrincipalOrgID           # restrict resource policy to your AWS Organization
aws:CurrentTime / DateGreaterThan / DateLessThan   # time-bound access
s3:x-amz-server-side-encryption   # enforce encryption on PutObject
iam:PermissionsBoundary      # require a specific boundary on created roles
```

## Permission boundaries (per-principal ceiling)

```
Effective permissions = identity policy ∩ permission boundary
```
```bash
aws iam put-role-permissions-boundary --role-name lab-developer \
  --permissions-boundary arn:aws:iam::ACCOUNT:policy/DeveloperBoundary
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT:role/lab-developer \
  --action-names s3:CreateBucket iam:CreateUser
```
Self-escalation guard:
```json
{
  "Effect": "Allow", "Action": "iam:CreateRole", "Resource": "*",
  "Condition": { "StringEquals": { "iam:PermissionsBoundary": "arn:aws:iam::ACCOUNT:policy/DeveloperBoundary" } }
}
```

## SCPs (org/OU-wide ceiling — Organizations)

```
Effective account permissions = identity policy ∩ SCP (applies even to root user)
```
```json
{
  "Effect": "Deny", "Action": "*", "Resource": "*",
  "Condition": { "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1","us-east-1"] } }
}
```
```json
{ "Effect": "Deny", "Action": ["s3:PutBucketPublicAccessBlock","s3:PutAccountPublicAccessBlock"], "Resource": "*" }
```
SCPs never grant — only restrict what identity policies can already grant. Attach at org root, OU, or account level; inherits down the OU tree.

## IAM Identity Center (SSO)

```bash
aws sso login --profile dev-account
aws sts get-caller-identity --profile dev-account
```
```
Identity Center -> Permission Sets -> Account Assignments
User signs in -> picks account -> federated via temporary assumed-role session
```

## Access Analyzer

```bash
aws accessanalyzer create-analyzer --analyzer-name org-analyzer --type ORGANIZATION
aws accessanalyzer create-analyzer --analyzer-name acct-analyzer --type ACCOUNT
aws accessanalyzer list-findings --analyzer-arn <ARN>
aws accessanalyzer get-finding --analyzer-arn <ARN> --id <FINDING_ID>
```
Flags resources reachable from outside your zone of trust (S3, IAM roles, KMS, SQS/SNS, Lambda, Secrets Manager) using automated reasoning over policies — not simple pattern matching.

## S3 Block Public Access (structural guardrail)

```bash
aws s3api put-public-access-block --bucket my-bucket \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3control put-public-access-block --account-id ACCOUNT_ID \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```
