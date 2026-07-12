# Day 52 — Lab: AWS Messaging Deep Dive

**Goal:** Build a real S3 → SNS → SQS (multi-consumer) fan-out pipeline — the assigned hands-on activity — plus hands-on drills for visibility timeout, DLQs, and EventBridge pattern matching.

**Prerequisites:** AWS CLI v2 configured against a sandbox account with permissions for S3, SNS, SQS, IAM, and Lambda.

---

### Lab 1 — Core hands-on activity: S3 event → SNS → SQS fan-out

1. Create the SNS topic and two SQS queues (simulating two independent consumers):
   ```bash
   aws sns create-topic --name file-upload-events
   aws sqs create-queue --queue-name resize-consumer-queue
   aws sqs create-queue --queue-name analytics-consumer-queue
   ```
2. Subscribe both queues to the topic:
   ```bash
   aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint <RESIZE_QUEUE_ARN>
   aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint <ANALYTICS_QUEUE_ARN>
   ```
3. Set each queue's access policy to allow the SNS topic to send messages (required — SNS needs explicit permission on the queue):
   ```bash
   aws sqs set-queue-attributes --queue-url <RESIZE_QUEUE_URL> --attributes file://sqs-policy.json
   aws sqs set-queue-attributes --queue-url <ANALYTICS_QUEUE_URL> --attributes file://sqs-policy.json
   ```
4. Create an S3 bucket and configure event notifications to publish to the SNS topic on `s3:ObjectCreated:*`:
   ```bash
   aws s3 mb s3://my-upload-bucket-<yourname>
   aws s3api put-bucket-notification-configuration --bucket my-upload-bucket-<yourname> \
     --notification-configuration file://s3-notification.json
   ```
5. Upload a test file and confirm both queues received a message:
   ```bash
   echo "test" > test.txt
   aws s3 cp test.txt s3://my-upload-bucket-<yourname>/test.txt
   aws sqs receive-message --queue-url <RESIZE_QUEUE_URL>
   aws sqs receive-message --queue-url <ANALYTICS_QUEUE_URL>
   ```

**Success criteria:** A single S3 upload results in a message appearing in *both* SQS queues independently, and you can explain why adding a third consumer later requires zero changes to the S3 bucket or the producer.

---

### Lab 2 — Visibility timeout and duplicate delivery, hands-on

1. Send a message and receive it without deleting it:
   ```bash
   aws sqs send-message --queue-url <RESIZE_QUEUE_URL> --message-body "order-123"
   aws sqs receive-message --queue-url <RESIZE_QUEUE_URL> --visibility-timeout 20
   ```
2. Wait 25 seconds (longer than the visibility timeout), then receive again without ever deleting — confirm the same message reappears:
   ```bash
   sleep 25
   aws sqs receive-message --queue-url <RESIZE_QUEUE_URL>
   ```
3. This time, delete it properly using the receipt handle from the response:
   ```bash
   aws sqs delete-message --queue-url <RESIZE_QUEUE_URL> --receipt-handle <RECEIPT_HANDLE>
   aws sqs receive-message --queue-url <RESIZE_QUEUE_URL>   # should return nothing now
   ```

**Success criteria:** You directly observed a message reappear after visibility timeout expiry with no delete call, and can explain in one sentence why this means consumer code must be idempotent.

---

### Lab 3 — Dead-letter queue for poison messages

1. Create a DLQ and attach a redrive policy to the resize-consumer queue with a low `maxReceiveCount`:
   ```bash
   aws sqs create-queue --queue-name resize-consumer-dlq
   aws sqs set-queue-attributes --queue-url <RESIZE_QUEUE_URL> --attributes '{
     "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"3\"}"
   }'
   ```
2. Send a message, then receive-and-not-delete it 3 times in a row (simulating a consumer that always fails):
   ```bash
   aws sqs send-message --queue-url <RESIZE_QUEUE_URL> --message-body "poison-message"
   for i in 1 2 3; do
     aws sqs receive-message --queue-url <RESIZE_QUEUE_URL> --visibility-timeout 1
     sleep 2
   done
   ```
3. Confirm the message has moved to the DLQ:
   ```bash
   aws sqs receive-message --queue-url <DLQ_URL>
   ```

**Success criteria:** The message appears in the DLQ after exceeding `maxReceiveCount`, and you can explain what would have happened to a worker fleet without a DLQ configured (the message loops forever, consuming capacity).

---

### Lab 4 — EventBridge pattern matching drill

1. Create a rule matching only `stopped`/`terminated` EC2 state changes, targeting a CloudWatch Logs group for easy verification:
   ```bash
   aws logs create-log-group --log-group-name /events/ec2-state-changes
   aws events put-rule --name ec2-state-changes --event-pattern '{
     "source": ["aws.ec2"],
     "detail-type": ["EC2 Instance State-change Notification"],
     "detail": { "state": ["stopped", "terminated"] }
   }'
   aws events put-targets --rule ec2-state-changes --targets "Id"="1","Arn"="arn:aws:logs:<REGION>:<ACCT>:log-group:/events/ec2-state-changes"
   ```
2. Launch and stop a small EC2 instance (or use `aws events put-events` to simulate the event directly, cheaper than a real instance):
   ```bash
   aws events put-events --entries '[{
     "Source": "aws.ec2",
     "DetailType": "EC2 Instance State-change Notification",
     "Detail": "{\"state\":\"stopped\",\"instance-id\":\"i-0123456789\"}"
   }]'
   ```
3. Check the log group for the matched event within a minute or two.

**Success criteria:** You can explain why this rule would NOT match a `running` or `pending` state-change event, purely from reading the pattern.

---

### Cleanup

```bash
aws sns delete-topic --topic-arn <TOPIC_ARN>
aws sqs delete-queue --queue-url <RESIZE_QUEUE_URL>
aws sqs delete-queue --queue-url <ANALYTICS_QUEUE_URL>
aws sqs delete-queue --queue-url <DLQ_URL>
aws s3 rb s3://my-upload-bucket-<yourname> --force
aws events remove-targets --rule ec2-state-changes --ids "1"
aws events delete-rule --name ec2-state-changes
aws logs delete-log-group --log-group-name /events/ec2-state-changes
```

### Stretch challenge

Convert the analytics consumer queue into a FIFO queue with a `MessageGroupId` derived from a simulated `customerId`, send messages for 2 different customers in interleaved order, and prove that each customer's messages are received strictly in order relative to each other, while messages across customers are not required to interleave in send order.
