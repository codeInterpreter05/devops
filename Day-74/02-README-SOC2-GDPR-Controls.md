# Day 74 — Compliance & Audit: SOC2 & GDPR Technical Controls

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** DevSecOps

## Brief

SOC2 and GDPR are legal/business frameworks, not technical specifications — but a huge fraction of both is satisfied (or violated) by decisions DevOps engineers make: who has prod access, how long logs are retained, whether backups are encrypted, whether an EU customer's data ever crosses a region boundary. Interviewers ask about this because organizations increasingly expect infrastructure engineers to understand *why* certain controls exist, not just implement them mechanically when Security hands over a checklist. Getting this wrong at a real company means failed audits, lost enterprise deals (SOC2 reports are often a contractual requirement to close B2B deals), and in GDPR's case, fines up to 4% of global annual revenue.

## SOC2: the Trust Service Criteria that touch infrastructure

SOC2 (System and Organization Controls 2) is an audit framework built around five **Trust Service Criteria (TSC)**; "Security" is mandatory, the other four are opt-in based on what you claim to provide:

| Criterion | What it means operationally |
|---|---|
| **Security** (mandatory) | Access controls, encryption, vulnerability management, change management, incident response |
| **Availability** | Uptime SLAs, disaster recovery, capacity planning, monitoring/alerting |
| **Processing Integrity** | Data is processed accurately/completely (validation, error handling in pipelines) |
| **Confidentiality** | Data marked confidential is protected — often overlaps with encryption + access control |
| **Privacy** | How PII is collected, used, retained, disposed of — overlaps heavily with GDPR |

DevOps-relevant controls that show up in nearly every SOC2 audit:

- **Access reviews**: quarterly (or more frequent) review of who has IAM/prod access, with evidence (a ticket, a spreadsheet, an access-review tool export) that someone actually reviewed and revoked stale access. This is why **least-privilege IAM** and **automated deprovisioning on offboarding** aren't just good practice — they're audit evidence.
- **Change management**: every production change traceable to a PR, an approval, and a deploy record. This is *exactly* what the Day 73 pipeline (PR → review → CI gates → GitOps-audited deploy) produces as a side effect — a mature CI/CD pipeline is, almost incidentally, SOC2 change-management evidence.
- **Encryption at rest and in transit**: EBS/RDS/S3 encryption enabled, TLS everywhere, no plaintext secrets in logs or environment variables.
- **Logging and monitoring**: centralized, tamper-evident logs (CloudTrail, VPC Flow Logs, Kubernetes audit logs) retained for a defined period, with alerting on suspicious activity (see file 3 for the logging strategy in depth).
- **Vulnerability management**: a documented, evidenced cadence of scanning (Trivy/Prowler/Checkov, from Day 74's other files and Phase 2 broadly) and remediating findings within defined SLAs (e.g., "critical CVEs patched within 7 days").

**SOC2 Type I vs. Type II** matters operationally: Type I audits controls' *design* at a point in time ("do these controls exist and make sense?"); Type II audits *operating effectiveness over a period* (typically 6–12 months) — meaning auditors will sample actual evidence across that window (e.g., "show me three access-review tickets from the last two quarters," not just "show me the access-review policy document"). This is why compliance-as-code and consistent, automated evidence generation (Day 74 file 3) matters so much more for Type II — manually reconstructing 6–12 months of evidence after the fact is painful and error-prone.

## GDPR technical controls relevant to DevOps

GDPR is EU law protecting personal data of EU residents, but its technical requirements are universal engineering concerns regardless of where you're deployed:

- **Data residency / data localization**: personal data of EU subjects often must stay within the EU (or an "adequate" jurisdiction) unless specific transfer mechanisms (SCCs — Standard Contractual Clauses) are in place. Practically: know which AWS/GCP/Azure **region** your EU customer data lives in, and audit for accidental cross-region replication (a common gap: a backup job or a logging pipeline silently shipping data to a US region for "convenience").
- **Right to erasure ("right to be forgotten")**: a user can request deletion of their personal data, and you must actually be able to delete it — not just soft-delete/flag it. This has real infrastructure implications: if PII is scattered across a primary DB, several caches, a data warehouse, and years of log files, "delete this user's data" becomes a genuine engineering project. Designing for this from day one (a single source of truth for PII with foreign-key-driven cascading deletes, and log scrubbing/redaction pipelines) is far cheaper than retrofitting it under audit pressure.
- **Data minimization**: don't collect/retain more than necessary. Operationally: log retention policies that don't keep raw request bodies (which may contain PII) indefinitely "just in case," and pipelines that don't copy full production PII into staging/dev environments (a shockingly common violation — "restore prod DB to staging for testing" without any anonymization step).
- **Breach notification**: a confirmed personal-data breach must be reported to the relevant supervisory authority within **72 hours** of the organization becoming aware. This makes your **detection speed** (centralized logging/alerting, from file 3) a direct compliance requirement, not just an operational nicety — you cannot notify within 72 hours of something you don't detect until day 40.
- **Encryption and pseudonymization**: explicitly called out in GDPR Article 32 as "appropriate technical measures" — encryption at rest/in transit and pseudonymizing PII in non-production/analytics contexts directly reduces breach severity and, in some interpretations, breach notification obligations if the leaked data was encrypted with keys the attacker didn't obtain.

## Where SOC2 and GDPR controls overlap vs. diverge

Most encryption, access-control, and logging work satisfies both simultaneously — but they diverge in emphasis: SOC2 cares about *organizational process and evidence* (can you prove you did the review?), while GDPR cares about *data subject rights and data handling specifics* (can you actually delete/export one person's data on request?). A team that's SOC2-compliant can still be GDPR-non-compliant if, for example, they have great access-review evidence but no actual mechanism to fulfill a data-subject erasure request within the legally required timeframe.

## Points to Remember

- SOC2 "Security" criterion is mandatory; Availability/Processing Integrity/Confidentiality/Privacy are opt-in based on what services you claim — know which ones your org has actually scoped in.
- Type I = controls exist and are designed correctly at a point in time; Type II = controls actually operated effectively over a 6–12 month window, with sampled evidence — automate evidence generation or Type II audits become miserable.
- A well-built CI/CD pipeline (PR review, gated deploys, GitOps audit trail) is itself SOC2 change-management evidence almost by accident.
- GDPR's "right to erasure" requires an actual, complete deletion capability across every system holding PII — design a single PII source of truth early, don't retrofit under audit pressure.
- The 72-hour GDPR breach notification clock starts at *awareness*, which makes detection speed (centralized logging + alerting) a hard compliance requirement, not just operational hygiene.

## Common Mistakes

- Restoring a production database snapshot (with real customer PII) into a staging/dev environment for testing, without any anonymization/masking step — a frequent, silent GDPR data-minimization violation.
- Treating SOC2 as a one-time certification project instead of an ongoing operating discipline — Type II audits sample evidence across months, and gaps in the middle of that window fail the audit even if things look fine on audit day.
- Assuming "we're not an EU company" exempts you from GDPR — it applies based on whether you process EU residents' data, regardless of where your company is headquartered.
- Building analytics/logging pipelines that capture full request/response bodies (including PII) with indefinite retention "for debugging," creating both a data-minimization violation and a much larger breach blast radius.
- Confusing "soft delete" (setting a `deleted_at` flag) with actual erasure — a right-to-erasure request typically requires the data to be genuinely unrecoverable, not just hidden from normal queries while still sitting in the database and backups.
