# Day 33 — AWS IAM Deep Dive: Permission Boundaries, SCPs & AWS Organizations

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Identity-based policies alone don't scale to "let developers create their own IAM roles without letting them create an all-powerful one" or "guarantee that no account in our company can ever use a region we don't operate in." Permission boundaries and Service Control Policies (SCPs) are AWS's two mechanisms for capping the *maximum possible* permissions a principal can have, regardless of what any identity-based policy grants — one at the single-account/single-principal level, one at the entire-account/entire-org level. Understanding the difference (and that both are "ceilings," never "grants") is directly tied to this day's interview question about preventing an accidentally-public S3 bucket organization-wide.

## Permission boundaries — a per-principal ceiling

A **permission boundary** is a managed policy attached to a *specific* IAM user or role that defines the **maximum** permissions that principal can ever have — it never grants anything by itself; it only caps what the principal's own identity-based policies can grant.

**Effective permissions = intersection of (identity-based policy) AND (permission boundary).** A principal can only do what *both* the identity policy and the boundary allow — not the union, the intersection.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:*", "ec2:*", "iam:*"], "Resource": "*" },
    { "Effect": "Deny",  "Action": "iam:*", "Resource": "*" }
  ]
}
```

If this is set as a permission boundary on a role whose identity policy grants `s3:*`, `ec2:*`, and `iam:*`, the role can effectively only use `s3:*` and `ec2:*` — the boundary's deny on `iam:*` caps that out regardless of the identity policy granting it.

**The canonical real-world use case**: letting developers self-service create IAM roles for their own applications, without letting them (accidentally or maliciously) create a role broader than they themselves are permitted to have.

```json
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": { "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary" }
  }
}
```

This condition forces any role a developer creates to *itself* carry the `DeveloperBoundary` permission boundary — closing the loophole where a developer permitted to call `iam:CreateRole` could otherwise create an unrestricted admin role and simply assume that instead.

## Service Control Policies (SCPs) — an org/account-wide ceiling

**SCPs** apply the same "ceiling, never a grant" concept, but at the level of an **AWS Organizations** OU (Organizational Unit) or entire account, affecting **every principal in that account**, including its root user — something no identity-based policy or permission boundary can do (a permission boundary only affects the specific principal it's attached to; it never touches the account's root user or other principals).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1", "us-east-1"] }
      }
    }
  ]
}
```

Attached to an OU containing every production account, this SCP makes it **structurally impossible** for anyone in those accounts — including an account's own root user or an admin with `AdministratorAccess` — to take any action in a disallowed region. This is the practical, org-wide mechanism behind this day's interview question: "an engineer accidentally made an S3 bucket public — how do you prevent this org-wide?" The answer is an SCP (or, more precisely for this specific case, the S3 Block Public Access setting enforced via SCP so no account can disable it) — a per-account IAM policy fix only protects that one account; an SCP protects every account under the OU it's attached to, permanently, regardless of what any individual account administrator does locally.

```json
{
  "Effect": "Deny",
  "Action": ["s3:PutBucketPublicAccessBlock", "s3:PutAccountPublicAccessBlock"],
  "Resource": "*"
}
```
An SCP denying the ability to *disable* S3's account-level Block Public Access setting ensures no engineer, no matter how privileged locally, can turn off that guardrail — the org-wide "prevent this from ever happening again" answer.

## AWS Organizations — the container SCPs live in

**AWS Organizations** is the management layer that groups multiple AWS accounts under one payer/management account, arranged into a tree of **Organizational Units (OUs)** (e.g., `Production`, `Staging`, `Sandbox` OUs, each containing several accounts). SCPs attach to OUs (or individual accounts, or the org root) and inherit down the tree — an SCP on the org root applies to every account in every OU; an SCP on a specific OU applies only to accounts within it and its children.

This is also the mechanism behind **consolidated billing** and delegated administration of other AWS services (e.g., a delegated Security Hub or GuardDuty admin account that can see findings across the whole org) — Organizations is the structural backbone that makes org-wide guardrails (SCPs), org-wide identity (Identity Center, next file), and org-wide security tooling all possible in one place.

## Points to Remember

- Both permission boundaries and SCPs are **ceilings, never grants** — they can only restrict what an identity-based policy would otherwise allow, never add permissions on their own.
- Permission boundaries scope to a *specific* principal (a user or role) and are set by whoever manages IAM in that account; SCPs scope to an *entire account or OU* within AWS Organizations, and affect every principal in scope including the root user.
- Effective permissions under a permission boundary = intersection of identity policy and boundary; the same intersection logic applies between an account's effective permissions and any applicable SCP.
- SCPs cannot grant permissions by themselves (a common misconception) — an account still needs identity-based policies granting the actual permissions; the SCP only ever narrows what's possible.
- Preventing an org-wide class of mistake (like public S3 buckets) requires an SCP at the OU/org level — a per-account IAM fix only protects the one account it's applied to.

## Common Mistakes

- Writing an SCP that tries to *grant* a permission (e.g., an SCP with only Allow statements and no other identity policy behind it) and being confused why nothing actually happens — SCPs never grant, they only restrict what identity policies can already grant.
- Assuming a permission boundary on one developer's role protects the whole account — it only constrains that specific principal; other principals (including the root user) are entirely unaffected by someone else's permission boundary.
- Forgetting that SCPs apply to the account's root user too — teams sometimes get surprised when even a full administrator/root session is blocked by an org-wide SCP, expecting root to be exempt (it is not).
- Not using the `iam:PermissionsBoundary` condition key when granting `iam:CreateRole`/`iam:CreateUser`, leaving a loophole where a permission-boundary-restricted developer can still create a brand-new, unrestricted principal and use that instead.
- Applying a broad, restrictive SCP directly to the org root without testing in a lower-risk OU first, potentially locking out even break-glass/root access across every single account in the organization simultaneously.
