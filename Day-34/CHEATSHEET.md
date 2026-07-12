# Day 34 — Cheatsheet: AWS Compute (EKS, ECS, Lambda)

## EKS node options

```bash
eksctl create cluster --name my-cluster --region ap-south-1
eksctl create nodegroup --cluster my-cluster --managed --node-type t3.medium --nodes 3 --nodes-min 2 --nodes-max 6
eksctl create nodegroup --cluster my-cluster --managed=false --node-type t3.medium --nodes 3   # self-managed
eksctl create fargateprofile --cluster my-cluster --namespace my-app
```

| | Self-managed | Managed node group | Fargate |
|---|---|---|---|
| Patching | You | AWS-assisted | N/A (no node) |
| DaemonSets | Yes | Yes | No |
| Cost model | Per-instance | Per-instance | Per-pod |
| Custom AMI/kernel | Yes | Limited | No |

## ECS

```bash
aws ecs create-cluster --cluster-name my-cluster
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs create-service --cluster my-cluster --service-name my-app \
  --task-definition my-app --desired-count 3 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx]}"
aws ecs update-service --cluster my-cluster --service my-app --desired-count 5
aws ecs describe-services --cluster my-cluster --services my-app
```

## ECS vs EKS — one-line answer

```
ECS: AWS-native, simple, no control plane fee, no multi-cloud portability
EKS: full Kubernetes, ecosystem + portability, real learning curve + ops cost
Choose EKS when: multi-cloud need, existing K8s expertise, need CNCF ecosystem tool
Choose ECS when: AWS-only, small team, standard container workload
```

## Lambda — cold starts & concurrency

```python
# top-level = cold-start-once init
import boto3
client = boto3.client("dynamodb")

def handler(event, context):
    ...   # runs every invocation
```

```bash
aws lambda put-provisioned-concurrency-config \
  --function-name my-fn --qualifier prod \
  --provisioned-concurrency-config AllocatedConcurrentExecutions=10

aws lambda put-function-concurrency --function-name my-fn --reserved-concurrent-executions 20

# CloudWatch Logs REPORT line shows cold start:
# REPORT ... Init Duration: 412.30 ms  <- present only on cold start
```

## Lambda layers

```bash
aws lambda publish-layer-version --layer-name common-utils \
  --zip-file fileb://layer.zip --compatible-runtimes python3.12
```
```json
"Layers": ["arn:aws:lambda:region:account:layer:common-utils:3"]
```
Combined function + layers unzipped size limit: 250MB.

## SAM CLI

```bash
sam init
sam build
sam deploy --guided
sam logs -n FunctionName --tail
sam local invoke FunctionName -e event.json
sam delete
```

## SQS as a Lambda trigger

```yaml
Events:
  SQSEvent:
    Type: SQS
    Properties:
      Queue: arn:aws:sqs:region:account:my-queue
      BatchSize: 5
```
```bash
aws sqs create-queue --queue-name my-queue --attributes '{
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"<DLQ_ARN>\",\"maxReceiveCount\":\"3\"}"
}'
```

## EventBridge

```bash
aws events put-rule --name daily-report --schedule-expression "cron(0 6 * * ? *)"
aws events put-targets --rule daily-report --targets "Id"="1","Arn"="arn:aws:lambda:...:function:fn"
aws events put-rule --name s3-uploads --event-pattern '{"source":["aws.s3"],"detail-type":["Object Created"]}'
```
