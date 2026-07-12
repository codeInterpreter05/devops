# Day 49 — AWS Security Services: Threat Detection & Aggregation

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Every AWS account eventually needs an answer to "how would you know if you'd been breached, and how fast?" GuardDuty and Security Hub are AWS's answer: a managed threat-detection engine (GuardDuty) feeding a managed aggregation-and-compliance layer (Security Hub). This pairing shows up constantly in interviews because it's the fastest way to test whether a candidate has actually operated a security-conscious AWS account versus just knowing IAM exists. Today's interview question — "how would you detect and respond to a compromised EC2 instance automatically?" — is answered almost entirely by these two services plus EventBridge.

This day is split into three files:

1. **This file** — GuardDuty (detection) and Security Hub (aggregation/compliance).
2. **[02-README-Perimeter-And-Data-Protection.md](02-README-Perimeter-And-Data-Protection.md)** — WAF, Shield, and Macie.
3. **[03-README-Audit-Logging-And-Encryption.md](03-README-Audit-Logging-And-Encryption.md)** — CloudTrail and KMS.

## GuardDuty — how it actually detects threats

GuardDuty is **agentless by default** — it does not run software on your EC2 instances. Instead it continuously analyzes data your account is already generating:

- **VPC Flow Logs** — network metadata (source/dest IP, port, bytes, ACCEPT/REJECT). GuardDuty consumes an *internal* copy of this data; you don't need to enable VPC Flow Logs yourself for GuardDuty to see it.
- **DNS logs** — queries made through the AWS-provided DNS resolver. This is how it catches an EC2 instance beaconing out to a known cryptomining or C2 (command-and-control) domain — it cross-references queried domains against AWS's own threat-intelligence feeds (and optionally your own custom threat lists).
- **CloudTrail management + S3 data events** — API call patterns. This is how it catches things like an IAM user suddenly calling `GetCallerIdentity` from an anomalous geography, or unusual `ListBuckets`/`GetObject` volume consistent with data exfiltration.
- **EKS audit logs** — Kubernetes API server audit trail, for detecting suspicious `kubectl` activity or anonymous API access.
- **RDS login activity, Lambda network activity, EBS malware scanning (via a lightweight agent)** — added later as GuardDuty expanded beyond pure network/API telemetry.

Because it works off logs you're already producing internally at the control-plane level, **enabling GuardDuty is a single API call/console click** — no agents to deploy, no sidecars, no instrumentation of your workloads. That low-friction adoption is precisely why it's the default "turn this on in every account" recommendation.

**Finding naming convention** (memorize the pattern, not just examples): `ThreatPurpose:ResourceType/ThreatFamilyName`. For example:
- `UnauthorizedAccess:EC2/SSHBruteForce` — someone is brute-forcing SSH against an instance.
- `CryptoCurrency:EC2/BitcoinTool.B!DNS` — instance is resolving domains associated with cryptomining pools.
- `Recon:IAMUser/MaliciousIPCaller` — an IAM user's API calls are originating from an IP on a known-bad list.
- `Backdoor:EC2/C2Activity` — instance is communicating with a known command-and-control server.

Findings carry a **severity score** (Low 1.0–3.9, Medium 4.0–6.9, High 7.0–8.9) — this drives triage priority and is exactly the kind of number you filter EventBridge rules on so you don't page someone at 3 a.m. for a Low finding.

**Noise control matters in practice**: real accounts generate false positives (a pentest, an internal scanner, a dev instance that legitimately calls unusual APIs). GuardDuty supports **suppression rules** (auto-archive findings matching a filter) and **trusted IP lists** (IPs you tell GuardDuty to never flag, e.g. your own VPN egress or scanning tools) — without these, teams tune out real alerts inside weeks.

**Multi-account posture**: in an AWS Organizations setup, you designate a **GuardDuty administrator account** (usually the Security/Audit account in a landing zone) that auto-enables GuardDuty for every new member account and centralizes findings — this is how real orgs avoid "we forgot to turn GuardDuty on in the account someone spun up last Tuesday."

For testing/demos without waiting for a real attack: `aws guardduty create-sample-findings` generates synthetic findings of every type in your GuardDuty detector, which is exactly what today's lab exercises.

## Security Hub — aggregation and compliance posture

Security Hub does not detect anything itself — it's a **normalization and aggregation layer**. It ingests findings from GuardDuty, Macie, Inspector (vulnerability scanning), IAM Access Analyzer, Firewall Manager, and third-party tools (Qualys, Prisma Cloud, Palo Alto, etc.), and converts them all into a single schema: the **AWS Security Finding Format (ASFF)**. This is the same reason Kubernetes has a common resource schema — without it, every consumer (a dashboard, a ticketing integration, a SIEM) would need custom parsers per source tool.

Beyond aggregation, Security Hub runs **automated compliance checks** against named standards:
- **CIS AWS Foundations Benchmark** — the most commonly asked-about one in interviews; things like "root account MFA enabled," "no security groups allow unrestricted ingress on port 22."
- **AWS Foundational Security Best Practices (FSBP)** — AWS's own opinionated baseline.
- **PCI DSS**, **NIST 800-53** — for regulated workloads.

Each standard produces a **compliance score** (percentage of checks passing) — this is the number that ends up in a compliance dashboard for auditors, and it's continuously re-evaluated, not a one-time scan.

**Insights** are saved, reusable queries over findings (e.g., "group all High-severity findings by resource type") — useful for spotting trends rather than firefighting one finding at a time.

**Cross-Region aggregation**: Security Hub is regional by default, but you can designate one **aggregation Region** that pulls in findings from every other enabled Region in the account — critical if your workloads are multi-region, since otherwise you'd be tabbing between consoles.

## Wiring it together for automated response

The interview question "how would you detect and respond to a compromised EC2 instance automatically" is really asking you to describe this pipeline:

1. GuardDuty detects anomalous behavior (e.g., `Backdoor:EC2/C2Activity`) and publishes a finding.
2. GuardDuty findings are automatically sent to the **default EventBridge event bus** — no extra plumbing needed to get them off GuardDuty.
3. An **EventBridge rule** matches on finding type/severity (e.g., severity >= 7) and fans out to targets:
   - **SNS topic** → pages the on-call engineer / posts to Slack.
   - **Lambda function** → automated containment: strip the instance's security group and attach an isolation SG (no ingress/egress except to a forensics subnet), snapshot the EBS volume for later analysis, tag the instance `quarantined=true`, optionally revoke the instance's IAM role session via a deny-all inline policy.
   - **Step Functions** → for a multi-step remediation workflow with approval gates instead of a single Lambda doing everything blindly.
4. Security Hub gives the aggregated view and lets a human trigger a **custom action** for anything the automation didn't fully resolve.

The reason this whole chain is agentless and API-driven end-to-end is exactly why it's held up as the "modern cloud-native SOC" pattern versus bolting a traditional on-prem IDS onto EC2.

## Points to Remember

- GuardDuty is agentless for its core detection (VPC Flow Logs, DNS, CloudTrail) — it reads telemetry your account already produces; only Malware Protection/Runtime Monitoring add an agent-like component.
- Finding format is `ThreatPurpose:ResourceType/ThreatFamilyName` — recognizing this pattern lets you reason about unfamiliar finding names in an exam or incident.
- Security Hub aggregates and normalizes (ASFF) — it does not itself detect threats; it consumes GuardDuty/Macie/Inspector/third-party findings.
- Compliance standards (CIS, FSBP, PCI DSS, NIST) run as continuous automated checks in Security Hub, producing a live compliance score, not a one-time audit snapshot.
- GuardDuty findings land on the default EventBridge bus automatically — that's the hook for automated remediation, not a direct SNS integration from GuardDuty itself.
- Use a delegated administrator account (AWS Organizations) so GuardDuty/Security Hub auto-enable for every new account — manual per-account enablement is how gaps happen.

## Common Mistakes

- Enabling GuardDuty/Security Hub in one account and assuming it covers the whole AWS Organization — without a delegated administrator + auto-enable, new accounts start with nothing turned on.
- Treating every GuardDuty finding as equally urgent, causing alert fatigue — filter and route by severity, and use suppression rules for known-benign patterns (internal pentests, scanners).
- Building an auto-remediation Lambda that fully isolates a "compromised" instance based on a single Low-severity finding, causing a self-inflicted outage on a false positive — gate destructive automation behind severity/confidence thresholds and require a human step for anything below High.
- Forgetting that Security Hub findings and GuardDuty findings are region-scoped — not configuring cross-Region aggregation and then missing findings from a Region you don't check often.
- Assuming Security Hub's compliance score means "we are compliant" for an actual audit — it's a strong signal but auditors still want evidence packages, not just a dashboard percentage.
