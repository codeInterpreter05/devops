# Day 33 — AWS IAM Deep Dive: IAM Identity Center & Access Analyzer

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Policy evaluation logic (file 1) and org-wide guardrails (file 2) tell you what a request *can* do once it's authenticated as some principal. This file covers the two remaining pieces of the real-world picture: how humans actually *get* AWS access without every person needing a separate IAM user in every account (IAM Identity Center), and the tool that continuously audits whether your actual, current configuration has drifted into something dangerous — most notably, public resources (Access Analyzer). Both are the practical tools behind today's hands-on lab.

## IAM Identity Center — SSO for your entire AWS Organization

**IAM Identity Center** (formerly AWS SSO) is AWS's answer to "we have 50 engineers and 12 AWS accounts — nobody should have 12 separate IAM user credentials." It provides one place to manage human identities (or federate from an external identity provider — Okta, Azure AD, Google Workspace) and grant them access across every account in an AWS Organization through **Permission Sets**.

```
Identity Center (management account)
  ├── Users/Groups (local, or synced from Okta/Azure AD via SCIM)
  ├── Permission Sets (e.g., "Developer", "ReadOnly", "Break-Glass-Admin")
  └── Account Assignments: which group gets which permission set, in which account
```

A **Permission Set** is essentially a template for an IAM role — when a user signs in through Identity Center and picks an account, Identity Center federates them into that account by **assuming a role** (created automatically, one per permission-set-per-account) rather than the user ever holding standing IAM credentials in that account at all. This is the key operational win: access is centrally granted/revoked in *one place* (Identity Center), and it propagates to potentially dozens of accounts, instead of an admin having to remember to update IAM in every account individually when someone joins, leaves, or changes teams.

```bash
aws sso login --profile dev-account
aws sts get-caller-identity --profile dev-account
```

Because Identity Center sessions are **temporary, federated credentials** (not long-lived IAM user access keys), this is also the modern, recommended replacement for giving engineers individual IAM users with static access keys — it directly addresses the root cause of a huge share of AWS credential leaks (a static access key with no expiry, sitting in a laptop's `~/.aws/credentials` or accidentally committed to a repo).

## Access Analyzer — continuously auditing for unintended exposure

**IAM Access Analyzer** answers a different question than policy evaluation logic: not "is this policy syntactically correct" but "given everything currently configured, is anything reachable from **outside** the trust zone I intended" — most commonly, "is anything in my account public, or reachable from another AWS account I didn't intend to trust."

```bash
aws accessanalyzer create-analyzer --analyzer-name org-analyzer --type ORGANIZATION
aws accessanalyzer list-findings --analyzer-arn arn:aws:access-analyzer:...
```

Access Analyzer works by **mathematically modeling** every resource-based policy in scope (S3 bucket policies, IAM role trust policies, KMS key policies, SQS/SNS policies, Lambda resource policies, Secrets Manager policies) and determining whether the effective permissions allow access from outside a defined "zone of trust" — typically your AWS Organization. This is a fundamentally different technique from simple rule-matching: it's a formal, provable analysis (using automated reasoning) of what's *actually* reachable given the full policy, not just a pattern match against known-bad configurations, which is what lets it catch novel/unexpected exposure paths, not just a checklist of known S3-misconfiguration patterns.

```
Finding: S3 bucket "customer-uploads" is accessible from outside your zone of trust
  Access level: PUBLIC (via bucket policy Principal: "*")
  Recommended action: Restrict the bucket policy or add a condition
```

Findings are continuously (not just on-demand) generated as configurations change, and an **organization-level analyzer** (as opposed to a single-account analyzer) surfaces findings across every account in the org from one place — directly answering "how do you find out about a public bucket anywhere in the company" without needing every account's own admin to separately run and check their own analyzer.

Access Analyzer also has an **unused access** analysis mode (a newer capability) that flags IAM roles/users with permissions they haven't actually exercised in N days — useful for tightening overly broad policies down toward least privilege based on real usage, not guesswork.

## Putting the whole day together — answering the interview question properly

"An engineer accidentally made an S3 bucket public. How do you prevent this org-wide?" has layers, and a strong answer touches all of them:

1. **Structural prevention**: an SCP across the org denying `s3:PutBucketPublicAccessBlock`/`s3:PutAccountPublicAccessBlock`, so S3's account-level Block Public Access setting can never be disabled by anyone, anywhere in the org (file 2).
2. **Detective control**: an organization-level Access Analyzer continuously flagging any bucket (or other resource) that becomes reachable from outside the org's zone of trust, even through a path nobody anticipated (this file).
3. **Access hygiene**: Identity Center-managed, temporary, centrally-revocable access instead of long-lived per-account IAM users, reducing the number of standing credentials capable of making such a change in the first place (this file).
4. **Ordinary IAM discipline**: least-privilege identity policies so as few principals as possible can modify bucket policies to begin with (file 1).

## Points to Remember

- IAM Identity Center centralizes human access across an entire AWS Organization via Permission Sets, federating users into accounts via temporary assumed-role sessions rather than standing per-account IAM users.
- Identity Center is also the modern answer to "stop issuing long-lived IAM user access keys to engineers" — temporary credentials by default.
- Access Analyzer uses automated reasoning over resource-based policies to determine actual reachability from outside your zone of trust — a provable analysis, not just pattern-matching against known misconfigurations.
- An organization-level Access Analyzer surfaces public/cross-account exposure findings across every account in the org from one place, not per-account.
- Preventing an org-wide class of mistake needs both a structural guardrail (SCP) and a detective control (Access Analyzer) — one prevents, the other catches what still gets through or was already there before the guardrail existed.

## Common Mistakes

- Continuing to issue individual IAM users with long-lived access keys per account instead of adopting Identity Center, missing its main security benefit (temporary, centrally-revocable credentials) not just its convenience benefit.
- Running Access Analyzer only in individual accounts instead of enabling an organization-level analyzer, missing visibility into findings across the rest of the org.
- Treating an Access Analyzer finding as automatically "fix immediately" without checking whether the flagged external access was actually intentional (e.g., a deliberately public static-website bucket) — findings need triage, not blind auto-remediation.
- Assuming Access Analyzer prevents misconfigurations from happening — it's a **detective** control (finds them after the fact, continuously), not a **preventive** one; preventing them structurally still requires SCPs or similar guardrails.
- Forgetting that a Permission Set change in Identity Center doesn't take effect for an already-active session until that session's temporary credentials expire and the user re-authenticates — expecting an instantaneous revocation mid-session.
