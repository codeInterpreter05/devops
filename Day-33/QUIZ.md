# Day 33 — Quiz: AWS IAM Deep Dive

Try to answer without looking at your notes. Answers are at the bottom.

1. State the exact three-step precedence AWS uses to evaluate IAM policies.
2. Does the order of statements within a policy JSON document, or the order of multiple attached policies, affect evaluation? Why or why not?
3. What is the difference between an identity-based policy and a resource-based policy? Which one enables cross-account access?
4. What does a permission boundary actually do — does it grant permissions on its own?
5. Write the formula for a principal's effective permissions when a permission boundary is attached.
6. What loophole does the `iam:PermissionsBoundary` condition key close, specifically?
7. How does an SCP differ from a permission boundary in terms of scope, and does an SCP affect an account's root user?
8. Can an SCP grant a permission that no identity-based policy in the account grants? Why or why not?
9. What is IAM Identity Center, and what's the mechanism by which a user actually gets access to a specific account (hint: it's not a standing IAM user)?
10. How does Access Analyzer determine that a resource is "publicly accessible" — is it pattern-matching against known bad configurations?
11. Name a condition key you'd use to require MFA for a sensitive action, and one you'd use to restrict actions to specific AWS regions.
12. **Interview question:** An engineer accidentally made an S3 bucket public. How do you prevent this org-wide?

---

## Answers

1. (1) An explicit Deny in any applicable policy denies the request immediately, overriding any Allow. (2) If there is no explicit deny and no explicit allow, the request is denied by default (implicit deny). (3) If there is no explicit deny and at least one explicit Allow exists, the request is allowed.
2. No — order does not matter. IAM gathers every applicable statement across every relevant policy first, then applies the Deny-beats-Allow rule as a single, order-independent evaluation. This differs from NACLs (Day 32), which are evaluated in rule-number order with first-match-wins.
3. An identity-based policy is attached to a user/group/role and defines what that principal can do. A resource-based policy is attached directly to a resource (e.g., an S3 bucket policy, KMS key policy) and defines who can access that resource — including principals in entirely different AWS accounts, which is the mechanism that enables cross-account access without the other account needing any identity policy referencing the resource.
4. No — a permission boundary never grants anything by itself. It only sets a ceiling on what an identity-based policy attached to the same principal is allowed to actually grant. Without any identity-based policy, a permission boundary alone grants nothing.
5. Effective permissions = identity-based policy ∩ permission boundary (the intersection, not the union) — a principal can only do what both the identity policy and the boundary independently allow.
6. It closes the loophole where a principal permitted to call `iam:CreateRole` (or `CreateUser`) could otherwise create a brand-new, unrestricted role/user with no boundary at all and use that as an end-run around their own permission boundary. Requiring the condition forces any role they create to itself carry the specified boundary.
7. A permission boundary attaches to and affects a single, specific principal (a user or role) within an account. An SCP attaches to an AWS Organizations OU or account and affects every principal within that scope, including the account's own root user — something a permission boundary, scoped to one principal, cannot do.
8. No — like permission boundaries, SCPs never grant permissions; they only restrict what identity-based policies in the account can already grant. An SCP with only Allow statements and no other supporting identity policy accomplishes nothing on its own.
9. IAM Identity Center is AWS's centralized SSO service for managing human access across an entire AWS Organization via Permission Sets (templates that become IAM roles per account) and account assignments. A user gets access to a specific account by signing in through Identity Center and being federated into that account via a temporary assumed-role session — not by holding a standing IAM user/credentials in that account.
10. No — Access Analyzer uses automated/formal reasoning to mathematically model the actual effective permissions granted by every relevant resource-based policy (S3, IAM role trust policies, KMS, SQS/SNS, Lambda, Secrets Manager) and determines provably whether access is possible from outside a defined zone of trust (typically the AWS Organization). This lets it catch genuinely novel or unexpected exposure paths, not just matches against a fixed checklist of known misconfiguration patterns.
11. MFA: `"Bool": {"aws:MultiFactorAuthPresent": "true"}`. Region restriction: `"StringEquals"` or `"StringNotEquals"` on `aws:RequestedRegion`.
12. Strong answer: "Layer structural prevention and detection. Structurally, attach an SCP across the relevant OU (or the whole org) that denies `s3:PutBucketPublicAccessBlock` and `s3:PutAccountPublicAccessBlock`, so no one — including an account admin — can disable S3's Block Public Access setting anywhere in the org; this makes the specific mistake structurally impossible going forward. On the detection side, enable an organization-level IAM Access Analyzer so any resource (not just S3) that becomes reachable from outside the org's zone of trust is flagged continuously, catching cases that predate the guardrail or slip through some other path. I'd also reduce standing access broadly via IAM Identity Center with least-privilege permission sets, so fewer principals can even modify bucket policies in the first place."
