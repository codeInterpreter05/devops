# Day 49 — Lab: AWS Security Services

**Goal:** Turn on real threat detection and aggregation in an AWS account, generate a simulated finding, and wire up an automated alert — the assigned hands-on activity, broken into concrete steps.

**Prerequisites:**
- An AWS account where you can create IAM roles/policies (a sandbox account, not production).
- AWS CLI v2 installed and configured (`aws configure`), with a profile that has admin-ish permissions for this lab.
- Basic familiarity with `aws` CLI JSON output; `jq` installed is helpful but not required.

---

### Lab 1 — Enable GuardDuty and generate sample findings

1. Enable GuardDuty:
   ```bash
   aws guardduty create-detector --enable
   ```
   Note the `DetectorId` returned — you'll need it for every subsequent GuardDuty command.
2. Confirm it's active:
   ```bash
   aws guardduty get-detector --detector-id <DETECTOR_ID>
   ```
3. Generate sample findings covering every finding type GuardDuty supports:
   ```bash
   aws guardduty create-sample-findings --detector-id <DETECTOR_ID>
   ```
4. List the findings and pick a few IDs:
   ```bash
   aws guardduty list-findings --detector-id <DETECTOR_ID> --max-results 10
   ```
5. Get full detail on one finding (look at `Severity`, `Type`, and `Resource`):
   ```bash
   aws guardduty get-findings --detector-id <DETECTOR_ID> --finding-ids <FINDING_ID>
   ```

**Success criteria:** You can name at least 3 finding types by their `ThreatPurpose:ResourceType/ThreatFamilyName` pattern and explain what real-world behavior each represents.

---

### Lab 2 — Enable Security Hub and see findings aggregated

1. Enable Security Hub with the AWS Foundational Security Best Practices standard:
   ```bash
   aws securityhub enable-security-hub --enabled-standards StandardsSubscriptionArns="arn:aws:securityhub:<REGION>::standards/aws-foundational-security-best-practices/v/1.0.0"
   ```
2. Confirm GuardDuty findings now appear in Security Hub (may take a few minutes):
   ```bash
   aws securityhub get-findings --filters '{"ProductName":[{"Value":"GuardDuty","Comparison":"EQUALS"}]}'
   ```
3. Check your current compliance score for the enabled standard:
   ```bash
   aws securityhub get-enabled-standards
   aws securityhub describe-standards-controls --standards-subscription-arn <SUBSCRIPTION_ARN> | head -50
   ```
4. Find every control currently `FAILED` and pick one to investigate — read the remediation guidance in its description.

**Success criteria:** You can explain, in your own words, the difference between "GuardDuty found this" and "Security Hub is showing this" — i.e., that Security Hub did not detect anything itself.

---

### Lab 3 — SNS alert on a GuardDuty finding (core hands-on activity)

1. Create an SNS topic and subscribe your email:
   ```bash
   aws sns create-topic --name guardduty-alerts
   aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint you@example.com
   ```
   Confirm the subscription from the email you receive.
2. Create an EventBridge rule matching GuardDuty findings of Medium severity or higher:
   ```bash
   aws events put-rule \
     --name guardduty-medium-plus \
     --event-pattern '{"source":["aws.guardduty"],"detail-type":["GuardDuty Finding"],"detail":{"severity":[{"numeric":[">=",4]}]}}'
   ```
3. Grant EventBridge permission to publish to your SNS topic, then set it as the rule's target:
   ```bash
   aws sns add-permission \
     --topic-arn <TOPIC_ARN> \
     --label AllowEventBridgePublish \
     --aws-account-id <YOUR_ACCOUNT_ID> \
     --action-name Publish

   aws events put-targets \
     --rule guardduty-medium-plus \
     --targets "Id"="1","Arn"="<TOPIC_ARN>"
   ```
4. Trigger another sample finding batch (Lab 1, step 3) and confirm you receive an email within a few minutes.

**Success criteria:** A synthetic GuardDuty finding results in an email notification, end-to-end, without manual polling of the console.

---

### Cleanup

```bash
aws events remove-targets --rule guardduty-medium-plus --ids "1"
aws events delete-rule --name guardduty-medium-plus
aws sns delete-topic --topic-arn <TOPIC_ARN>
aws securityhub disable-security-hub
aws guardduty delete-detector --detector-id <DETECTOR_ID>
```

### Stretch challenge

Extend Lab 3 so the EventBridge rule also triggers a Lambda function that automatically tags the affected EC2 instance with `guardduty-alert=true` and describes the instance's current security groups into CloudWatch Logs — the first building block of an automated containment workflow. Do not actually modify/remove security groups in this lab (that's a production-grade action you should only automate after thorough testing).
