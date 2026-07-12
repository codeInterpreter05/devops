# Day 49 — Cheatsheet: AWS Security Services

## GuardDuty

```bash
aws guardduty create-detector --enable                          # turn on GuardDuty in this region
aws guardduty get-detector --detector-id <ID>                    # check status/config
aws guardduty list-detectors                                     # list detector IDs in this region
aws guardduty create-sample-findings --detector-id <ID>           # generate synthetic findings (all types)
aws guardduty list-findings --detector-id <ID>                    # list finding IDs
aws guardduty get-findings --detector-id <ID> --finding-ids <FID>  # full detail on a finding
aws guardduty archive-findings --detector-id <ID> --finding-ids <FID>  # suppress/archive
aws guardduty create-ip-set --detector-id <ID> --name trusted --format TXT --location s3://bucket/ips.txt --activate
aws guardduty delete-detector --detector-id <ID>                   # disable GuardDuty
```

Finding naming pattern: `ThreatPurpose:ResourceType/ThreatFamilyName`
Severity bands: Low 1.0–3.9 · Medium 4.0–6.9 · High 7.0–8.9

## Security Hub

```bash
aws securityhub enable-security-hub --enabled-standards StandardsSubscriptionArns=<ARN>
aws securityhub get-enabled-standards                              # list active standards
aws securityhub describe-standards-controls --standards-subscription-arn <ARN>
aws securityhub get-findings --filters '{"ProductName":[{"Value":"GuardDuty","Comparison":"EQUALS"}]}'
aws securityhub batch-update-findings --finding-identifiers '[{"Id":"<ID>","ProductArn":"<ARN>"}]' --workflow '{"Status":"RESOLVED"}'
aws securityhub disable-security-hub
```

Standards: CIS AWS Foundations Benchmark · AWS Foundational Security Best Practices (FSBP) · PCI DSS · NIST 800-53

## WAF

```bash
aws wafv2 create-web-acl --name my-acl --scope REGIONAL --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myacl \
  --rules file://rules.json

aws wafv2 list-web-acls --scope REGIONAL
aws wafv2 get-web-acl --name my-acl --scope REGIONAL --id <ID>
aws wafv2 associate-web-acl --web-acl-arn <ARN> --resource-arn <ALB_ARN>
```

Scope: `REGIONAL` (ALB, API Gateway, AppSync) vs `CLOUDFRONT` (must create in `us-east-1`).
Rule actions: `Allow` · `Block` · `Count` (test mode, never blocks) · `CAPTCHA` / `Challenge`.
Always ship new rules in `Count` first, verify, then switch to `Block`.

## Shield

```bash
aws shield describe-subscription           # check if Shield Advanced is active
aws shield list-protections                # resources under Shield Advanced protection
aws shield create-protection --name my-alb --resource-arn <ALB_ARN>
```

Standard = free, automatic, L3/L4. Advanced = paid, DRT access, cost protection, better L7 coverage w/ WAF.

## Macie

```bash
aws macie2 enable-macie
aws macie2 create-classification-job --job-type ONE_TIME --name s3-scan \
  --s3-job-definition '{"bucketDefinitions":[{"accountId":"<ACCT>","buckets":["my-bucket"]}]}'
aws macie2 list-classification-jobs
aws macie2 get-findings --finding-ids <FID>
aws macie2 disable-macie
```

## CloudTrail

```bash
aws cloudtrail create-trail --name org-trail --s3-bucket-name <BUCKET> --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging --name org-trail
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 5
aws cloudtrail get-trail-status --name org-trail
aws cloudtrail put-event-selectors --trail-name org-trail --event-selectors file://data-events.json   # enable data events
```

Management events: on by default, 90-day free history. Data events: require explicit trail config, extra cost.

## KMS

```bash
aws kms create-key --description "app data key" --tags TagKey=Name,TagValue=app-cmk    # CMK
aws kms create-alias --alias-name alias/app-cmk --target-key-id <KEY_ID>
aws kms enable-key-rotation --key-id <KEY_ID>                       # optional auto-rotation for CMK
aws kms describe-key --key-id alias/aws/s3                          # inspect an AWS managed key
aws kms encrypt --key-id alias/app-cmk --plaintext fileb://secret.txt --output text --query CiphertextBlob | base64 --decode > secret.enc
aws kms decrypt --ciphertext-blob fileb://secret.enc --output text --query Plaintext | base64 --decode
aws kms schedule-key-deletion --key-id <KEY_ID> --pending-window-in-days 7
aws kms cancel-key-deletion --key-id <KEY_ID>
```

Envelope encryption flow: KMS generates data key → caller encrypts data locally with plaintext data key → plaintext data key discarded, only encrypted data key stored → decrypt sends encrypted data key back to KMS for unwrap.
