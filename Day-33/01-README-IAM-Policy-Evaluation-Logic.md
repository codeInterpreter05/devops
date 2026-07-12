# Day 33 — AWS IAM Deep Dive: Policy Evaluation Logic

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Nearly every AWS security incident traces back to a misunderstood IAM policy evaluation, not a broken feature — "why does this role still have access when I explicitly denied it" or "why doesn't this policy grant the access I wrote" almost always comes down to not knowing the actual evaluation algorithm. This is deeply interview-critical because it's cheap to test (a short scenario question) and reliably separates people who've internalized the model from people who've only ever clicked "attach policy" until access worked.

This day is split into three files:

1. **This file** — the policy evaluation algorithm itself: Allow vs. Deny, and condition keys.
2. **[02-README-Permission-Boundaries-And-SCPs.md](02-README-Permission-Boundaries-And-SCPs.md)** — permission boundaries, Service Control Policies, and AWS Organizations.
3. **[03-README-IAM-Identity-Center-And-Access-Analyzer.md](03-README-IAM-Identity-Center-And-Access-Analyzer.md)** — IAM Identity Center (SSO) and Access Analyzer.

## The evaluation algorithm — memorize this order

When a principal (user/role) makes a request, AWS evaluates **every applicable policy** — identity-based policies attached to the principal, resource-based policies on the target resource, permission boundaries, SCPs, and session policies — and combines them with this precedence:

1. **Explicit Deny anywhere → request denied.** No further evaluation needed; an explicit deny in *any* applicable policy wins over every other Allow, no matter how many Allows exist elsewhere.
2. **No explicit deny, but no explicit Allow either → implicit deny (default).** IAM is deny-by-default — a principal with zero attached policies can do nothing.
3. **No explicit deny, and an explicit Allow exists → request allowed** (assuming any boundary/SCP intersections below are also satisfied).

The phrase to internalize: **"explicit deny always wins, and everything not explicitly allowed is denied by default."** This is the entire interview answer to "explain how policy evaluation works," and it's also exactly why permission boundaries and SCPs (Day 33 file 2) function as *ceilings* — they work by injecting an implicit deny for anything outside their scope, which the algorithm above then honors unconditionally.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:*", "Resource": "*" },
    { "Effect": "Deny",  "Action": "s3:DeleteBucket", "Resource": "*" }
  ]
}
```

Here, `s3:*` grants everything, but the explicit `Deny` on `s3:DeleteBucket` wins for that specific action regardless of statement order in the JSON — **JSON statement order does not matter**, only Effect type does. This trips people up constantly: policies are not evaluated top-to-bottom like NACL rules (Day 32) — every statement across every applicable policy is gathered first, then Deny-beats-Allow is applied as a single rule, order-independent.

## Identity-based vs. resource-based policies

- **Identity-based**: attached to a user/group/role (`AmazonS3ReadOnlyAccess` attached to a role). Answers "what can this principal do."
- **Resource-based**: attached directly to a resource — an S3 bucket policy, an SQS queue policy, a KMS key policy. Answers "who can access this resource, including principals in *other* AWS accounts." This is the mechanism behind cross-account access: a bucket policy can grant `s3:GetObject` to a principal in a completely different account, without that account needing any identity-based policy referencing the bucket at all.

Both are evaluated together for any request touching a resource that supports resource-based policies — an S3 GetObject call is allowed only if **at least one** relevant policy (identity-based OR resource-based) grants it, **and no** applicable policy denies it. This "OR for allow, but Deny always wins across the union" is a frequent point of confusion — it's not "both must independently allow it" for same-account access; either side granting it is sufficient, as long as neither denies it.

## Condition keys — scoping *when* a policy applies

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1", "us-east-1"] }
  }
}
```

Condition keys let a statement apply only when a runtime condition holds — this example denies *any* action unless the request's target region is one of the two allowed. Common condition keys worth knowing cold:

- `aws:RequestedRegion` — restrict which regions actions can target (the pattern above — a common SCP for regulatory/cost region-locking).
- `aws:SourceIp` — restrict access to requests originating from specific IP ranges (e.g., corporate VPN CIDR).
- `aws:MultiFactorAuthPresent` — require MFA for sensitive actions (`"Bool": {"aws:MultiFactorAuthPresent": "true"}`), a very common real-world guardrail for anything destructive.
- `aws:PrincipalOrgID` — restrict a resource-based policy to only principals within your AWS Organization, a simple but powerful guardrail against accidental cross-account exposure.
- `s3:x-amz-server-side-encryption` — a resource-condition example, used to deny `s3:PutObject` calls that don't specify server-side encryption, enforcing encryption-at-rest at the policy level rather than trusting every uploader to remember.

Condition operators (`StringEquals`, `StringNotEquals`, `Bool`, `IpAddress`, `DateGreaterThan`, etc.) combine with keys to build precise, situational rules — this is what elevates IAM from "static allow/deny" into something that can express real organizational guardrails ("deny everything outside these regions," "require MFA for this," "only allow if request comes from the VPN").

## Points to Remember

- Explicit Deny always wins over any Allow, from any policy, regardless of statement or policy order — this is the single most important IAM fact to have memorized cold.
- Everything not explicitly allowed is implicitly denied — IAM's default posture is deny, not allow.
- Identity-based policies define what a principal can do; resource-based policies define who can reach a resource (including cross-account) — for same-account access either granting it is enough, as long as nothing denies it.
- Condition keys scope a statement to specific runtime circumstances (region, source IP, MFA presence, org membership) — this is how IAM expresses organizational guardrails beyond flat allow/deny.
- Policy statement order in JSON is irrelevant to evaluation — only the Effect type (Allow vs. Deny) and whether the condition matches determines the outcome.

## Common Mistakes

- Assuming policy statements are evaluated top-to-bottom (NACL-style) and being confused when a "later" Allow doesn't override an "earlier" Deny — Effect type wins regardless of order.
- Writing an Allow policy and being confused why access is still denied, without checking for a Deny anywhere else in the principal's attached policies, permission boundary, or applicable SCP — any one of those layers can inject the deny that wins.
- Forgetting that resource-based policies can grant cross-account access entirely independent of any identity-based policy — reviewing only a user's attached policies and missing that a bucket policy elsewhere grants (or denies) the same access.
- Writing a condition key with the wrong logical operator (e.g., using `StringEquals` when the intent required `StringNotEquals`, inverting the entire rule's meaning) and not testing it before attaching to a broad set of principals.
- Not using `aws:MultiFactorAuthPresent` or similar conditions on destructive actions, relying purely on "who has the permission" rather than also gating "under what circumstances that permission is actually usable."
