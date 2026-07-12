# Day 34 — Lab: AWS Compute (EKS, ECS, Lambda)

**Goal:** Deploy a Lambda function triggered by SQS — this day's assigned hands-on activity — then deploy an equivalent consumer on Kubernetes (reusing earlier RabbitMQ/K8s knowledge) and directly compare the two approaches.

**Prerequisites:**
- AWS CLI configured, SAM CLI installed (`pip install aws-sam-cli` or `brew install aws-sam-cli`).
- A running Kubernetes cluster you can deploy to (minikube/kind is fine for the comparison half — doesn't need to be EKS for this lab).
- `eksctl` installed if you want to do the optional EKS-specific steps.

---

### Lab 1 — The core hands-on activity: Lambda triggered by SQS

1. Create the queue and a dead-letter queue:
   ```bash
   aws sqs create-queue --queue-name lab-day34-dlq
   DLQ_ARN=$(aws sqs get-queue-attributes --queue-url <DLQ_URL> --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)
   aws sqs create-queue --queue-name lab-day34-queue --attributes '{
     "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\",\"maxReceiveCount\":\"3\"}"
   }'
   ```
2. Scaffold a SAM app: `sam init` (choose Python, "Hello World" template), then edit `template.yaml` to add an SQS event source:
   ```yaml
   Resources:
     ProcessorFunction:
       Type: AWS::Serverless::Function
       Properties:
         Handler: app.handler
         Runtime: python3.12
         Timeout: 10
         Events:
           SQSEvent:
             Type: SQS
             Properties:
               Queue: arn:aws:sqs:REGION:ACCOUNT:lab-day34-queue
               BatchSize: 5
   ```
3. Write `app.py` so top-level code initializes a boto3 client, and the handler logs each message's body plus `context.aws_request_id`:
   ```python
   import boto3, json
   dynamodb = boto3.resource("dynamodb")   # top-level: cold-start-once init

   def handler(event, context):
       for record in event["Records"]:
           print(f"Processing: {record['body']} (request {context.aws_request_id})")
       return {"batchItemFailures": []}
   ```
4. `sam build && sam deploy --guided`.
5. Send test messages and watch CloudWatch Logs:
   ```bash
   for i in 1 2 3 4 5; do
     aws sqs send-message --queue-url <QUEUE_URL> --message-body "order-$i"
   done
   sam logs -n ProcessorFunction --tail
   ```

**Success criteria:** Messages sent to SQS are visibly processed by the Lambda function, batched (BatchSize=5), with logs showing the batch being handled in one invocation.

---

### Lab 2 — Observe a cold start vs. a warm invocation

1. After a period of no invocations (5+ minutes), send one message and check the CloudWatch Logs `REPORT` line for `Init Duration` — this is the cold start cost.
2. Immediately send a second message and check its `REPORT` line — confirm there's no `Init Duration` field this time (warm invocation).
3. Note the actual millisecond difference between the two.

**Success criteria:** You can point to two specific `REPORT` log lines and explain, using real numbers from your own function, exactly what a cold start costs versus a warm invocation.

---

### Lab 3 — Force a failure and watch the DLQ

1. Modify the handler to raise an exception for any message body containing `"fail"`.
2. Redeploy, then send `"fail-me"` as a message body.
3. Watch it retry (per `maxReceiveCount`) and eventually land in the dead-letter queue:
   ```bash
   aws sqs receive-message --queue-url <DLQ_URL>
   ```

**Success criteria:** You've observed a failing message get retried the configured number of times before landing in the DLQ, and can explain why DLQs matter for poison-message handling.

---

### Lab 4 — Compare with a Kubernetes-based consumer

1. Deploy a simple Python consumer (using `boto3`, polling the same SQS queue in a loop) as a Kubernetes Deployment with 1 replica, in your minikube/kind cluster.
2. Send the same 5 test messages and confirm the K8s pod's logs show the same processing.
3. Write a short comparison (in your lab notes, not necessarily a file) covering: startup/cold-start behavior, cost model (idle pod cost vs. Lambda's pay-per-invocation), scaling behavior (HPA/manual replica count vs. Lambda's automatic concurrency scaling), and operational overhead (who patches the base image / runtime in each case).

**Success criteria:** You have a working consumer in both models and can articulate, from direct hands-on comparison, at least three concrete operational differences — not just "Lambda is serverless."

---

### Cleanup

```bash
sam delete
aws sqs delete-queue --queue-url <QUEUE_URL>
aws sqs delete-queue --queue-url <DLQ_URL>
kubectl delete deployment sqs-consumer
```

### Stretch challenge

Enable provisioned concurrency (`AllocatedConcurrentExecutions=1`) on the Lambda function, redeploy, and re-run Lab 2 — confirm the `Init Duration` field disappears entirely even after an idle period, and calculate roughly what that eliminated cold start would cost you monthly if the function ran continuously versus its normal on-demand pricing, to build real intuition for when provisioned concurrency is actually worth paying for.
