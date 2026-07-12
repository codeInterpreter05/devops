# Day 52 — Cheatsheet: AWS Messaging Deep Dive

## SQS

```bash
aws sqs create-queue --queue-name my-queue
aws sqs create-queue --queue-name my-queue.fifo --attributes FifoQueue=true,ContentBasedDeduplication=true

aws sqs send-message --queue-url <URL> --message-body "hello"
aws sqs send-message --queue-url <FIFO_URL> --message-body "hello" --message-group-id "customer-123"

aws sqs receive-message --queue-url <URL> --visibility-timeout 30 --wait-time-seconds 10  # long polling
aws sqs change-message-visibility --queue-url <URL> --receipt-handle <H> --visibility-timeout 300
aws sqs delete-message --queue-url <URL> --receipt-handle <H>

# Redrive policy (DLQ)
aws sqs set-queue-attributes --queue-url <URL> --attributes '{
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"5\"}"
}'
```

| | Standard | FIFO |
|---|---|---|
| Order | best-effort | strict per MessageGroupId |
| Delivery | at-least-once | exactly-once (5-min dedup) |
| Throughput | ~unlimited | 300/3000 msg/s (or high-throughput mode) |

## SNS

```bash
aws sns create-topic --name my-topic
aws sns subscribe --topic-arn <ARN> --protocol sqs --notification-endpoint <QUEUE_ARN>
aws sns subscribe --topic-arn <ARN> --protocol email --notification-endpoint you@example.com
aws sns publish --topic-arn <ARN> --message "hello" --message-attributes '{"event_type":{"DataType":"String","StringValue":"order_created"}}'
aws sns set-subscription-attributes --subscription-arn <SUB_ARN> --attribute-name FilterPolicy \
  --attribute-value '{"event_type":["order_created","order_updated"]}'
```

Fan-out pattern: `S3 event -> SNS topic -> N x SQS queues -> N independent consumers`

## EventBridge

```bash
aws events put-rule --name my-rule --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Instance State-change Notification"]}'
aws events put-rule --name scheduled --schedule-expression "rate(5 minutes)"
aws events put-targets --rule my-rule --targets "Id"="1","Arn"="<TARGET_ARN>"
aws events put-events --entries '[{"Source":"custom.app","DetailType":"OrderCreated","Detail":"{\"id\":1}"}]'
aws events remove-targets --rule my-rule --ids "1"
aws events delete-rule --name my-rule
```

Routes by content/structure (deep JSON `detail` match), ~200 native AWS sources auto-publish, supports archive+replay and scheduled rules.

## Kinesis Data Streams

```bash
aws kinesis create-stream --stream-name my-stream --shard-count 2
aws kinesis put-record --stream-name my-stream --partition-key "user-123" --data "hello"
aws kinesis get-shard-iterator --stream-name my-stream --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON
aws kinesis get-records --shard-iterator <ITERATOR>
aws kinesis describe-stream-summary --stream-name my-stream
```

Log model: read does NOT delete. Retention 24h (default) - 365 days. Shard = 1MB/s in, 2MB/s out (classic) or dedicated w/ Enhanced Fan-Out.

## MSK

```bash
aws kafka create-cluster --cluster-name my-cluster --broker-node-group-info file://broker-info.json \
  --kafka-version "3.5.1" --number-of-broker-nodes 3
aws kafka list-clusters
aws kafka create-cluster-v2 --cluster-name serverless-cluster --serverless '{}'   # MSK Serverless
```

## Messaging comparison (memorize the one-liners)

| | Model | Best fit |
|---|---|---|
| RabbitMQ | smart broker, rich routing (exchanges) | complex routing, RPC-style, moderate throughput |
| SQS | managed queue, claim-and-delete | simple decoupling, zero ops, no replay needed |
| Kafka/MSK | distributed log, dumb broker/smart consumer | high-throughput, multi-consumer, replayable |
