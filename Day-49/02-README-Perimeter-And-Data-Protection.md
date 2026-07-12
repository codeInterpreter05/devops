# Day 49 — AWS Security Services: Perimeter & Data Protection

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

GuardDuty and Security Hub (previous file) tell you *something bad is happening inside your account*. WAF, Shield, and Macie are about **prevention and exposure reduction**: stopping malicious HTTP traffic before it reaches your app (WAF), absorbing volumetric network attacks before they take you offline (Shield), and finding sensitive data sitting somewhere it shouldn't be before it becomes a breach headline (Macie). These three come up constantly in "how do you protect a public-facing application" and "how do you handle PII/compliance" interview threads.

## AWS WAF — Web Application Firewall

WAF sits in front of **CloudFront, Application Load Balancer, API Gateway, or AppSync**, inspecting HTTP(S) requests before they reach your application, and evaluates them against **Web ACLs (Access Control Lists)** made of **rules**.

Rule types worth knowing cold:
- **Managed rule groups** — AWS-managed (e.g., `AWSManagedRulesCommonRuleSet` covers OWASP-style generic threats, `AWSManagedRulesSQLiRuleSet` for SQL injection, `AWSManagedRulesKnownBadInputsRuleSet`) or Marketplace-managed (from vendors like F5, Fortinet). You subscribe to these rather than writing detection logic yourself — this is the practical default for 90% of use cases.
- **Custom rules** — string/regex match on headers, query strings, body; geo-match (block/allow by country); IP set match (block a list of known-bad IPs, or your own dynamically-updated list fed by a Lambda that watches for abuse).
- **Rate-based rules** — block an IP automatically once it exceeds N requests in a 5-minute rolling window. This is the go-to for basic layer-7 DDoS/scraping mitigation and is *the* most commonly cited WAF rule in interviews.
- **Rule actions**: `Allow`, `Block`, `Count` (log but don't block — critical for testing a new rule in production without breaking traffic), and `CAPTCHA`/`Challenge` for bot mitigation.

**Why `Count` mode matters operationally**: you never ship a new WAF rule straight to `Block` in production. You deploy it in `Count`, watch CloudWatch metrics/logs for how much legitimate traffic it *would* have blocked (false positive rate), and only flip to `Block` once you trust it. Skipping this step is the classic way a WAF rollout takes down real customer traffic.

## AWS Shield — DDoS protection

- **Shield Standard** — automatically active on **every AWS account, for free**, protecting against common network/transport layer (L3/L4) DDoS attacks (SYN floods, UDP reflection). No configuration needed.
- **Shield Advanced** — a paid subscription (flat annual fee, applies per-organization) adding:
  - Enhanced detection for **larger and more sophisticated attacks**, including some Layer 7 (application layer) attacks when paired with WAF.
  - **24/7 access to the AWS DDoS Response Team (DRT)** — this is the actual differentiator most people care about; during an active attack you get a human team helping write emergency WAF rules.
  - **Cost protection** — AWS credits you back for the scaling costs (extra CloudFront/ALB/EC2 usage) incurred *because* of a DDoS attack, so you're not financially punished for being attacked.
  - Requires resources to be in scope (ALB, CloudFront, Global Accelerator, EIP, Route 53) and works best combined with a WAF Web ACL on the same resource.

**Mental model**: Shield Standard = the seatbelt that's always on. Shield Advanced = the insurance policy plus a response team, and it only makes financial sense for businesses where downtime is expensive enough to justify the subscription cost.

## Amazon Macie — sensitive data discovery in S3

Macie uses **machine learning + pattern matching** to scan S3 buckets and classify the data inside objects — it answers "do we have PII/credentials/financial data sitting somewhere we didn't realize, and is that bucket even supposed to be public?"

- **Managed data identifiers**: pre-built detectors for common sensitive data types — credit card numbers, US Social Security Numbers, AWS access keys accidentally committed into a file, passport numbers, bank account numbers.
- **Custom data identifiers**: your own regex + keyword proximity rules, for org-specific sensitive patterns (internal employee IDs, project code names tied to unreleased products, etc.).
- **Automated sensitive data discovery**: runs on a schedule across your S3 estate and produces a **statistical sample-based finding** rather than scanning every object in every large bucket every time (cost control) — you can also target specific buckets for a full one-off job.
- Findings feed into Security Hub (same ASFF normalization as GuardDuty) and can trigger EventBridge-driven remediation, e.g., automatically tagging or quarantining a bucket found to have exposed PII, or alerting the data owner.
- Macie also flags **bucket-level policy risk** (public access, missing encryption, missing versioning) as part of its bucket inventory view — it's not purely about file contents.

**Why this matters beyond compliance checkbox**: the most common real-world "we got breached" story isn't a sophisticated attacker defeating your WAF — it's an S3 bucket that was made public by mistake (or a bucket policy typo) and happened to contain a database backup with customer PII in it. Macie is specifically built to catch that class of mistake before an external researcher/attacker finds it first.

## Points to Remember

- WAF operates at Layer 7 (HTTP) in front of CloudFront/ALB/API Gateway/AppSync; Shield operates primarily at Layer 3/4 (network/transport), with Shield Advanced extending some L7 protection when paired with WAF.
- Always roll out new WAF rules in `Count` mode first, verify false-positive rate via logs/metrics, then switch to `Block`.
- Shield Standard is free and automatic for everyone; Shield Advanced adds the DRT (human response team), cost protection credits, and stronger L7/large-attack coverage — it's a paid, opt-in layer.
- Rate-based WAF rules are the standard first line of defense against basic scraping/L7 DDoS — block an IP once it crosses a request-count threshold in a rolling window.
- Macie classifies data *inside* S3 objects (PII, credentials, financial data) using managed + custom data identifiers, and also flags bucket-level exposure risk (public access, no encryption).
- Both WAF and Macie findings/logs can feed Security Hub and drive EventBridge-based automated response, the same pattern as GuardDuty.

## Common Mistakes

- Deploying a new WAF managed rule group directly in `Block` mode in production and breaking legitimate traffic (common with rules like SQLi detection that can false-positive on legitimate JSON payloads containing SQL-like keywords).
- Assuming Shield Standard covers everything and being surprised during a large or application-layer attack that Shield Advanced (and its DRT) was needed — Standard is a baseline, not a complete DDoS strategy for critical, high-value targets.
- Running Macie exactly once as a "compliance checkbox" instead of scheduling recurring automated discovery jobs — sensitive data exposure is a moving target as new objects get uploaded constantly.
- Forgetting that WAF Web ACLs are **regional** (except when attached to CloudFront, which is global) — deploying a Web ACL in one region and assuming it protects resources in another.
- Not tuning Macie's custom data identifiers for org-specific patterns and relying solely on managed identifiers, which miss internal-only sensitive formats (internal ID schemes, proprietary key formats).
- Ignoring cost: running Macie's full-scan discovery job across a multi-petabyte bucket estate without understanding it's priced per GB scanned — surprise bills are common if this isn't scoped/scheduled thoughtfully.
