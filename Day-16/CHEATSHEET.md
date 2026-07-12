# Day 16 — Cheatsheet: Python for Automation II — AWS

## Session, client, resource

```python
import boto3

boto3.client("s3")                                   # default session, default credential chain
boto3.resource("s3")                                  # object-oriented wrapper (subset of services)

session = boto3.Session(profile_name="dev", region_name="us-east-1")
session.client("ec2")
session.resource("dynamodb")

# Explicit (temporary) credentials — normally only used right after an assume_role call
boto3.Session(
    aws_access_key_id="...",
    aws_secret_access_key="...",
    aws_session_token="...",
)
```

## Credential resolution chain (first match wins)

```
1. Explicit params to client()/Session()
2. Environment variables (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_SESSION_TOKEN)
3. ~/.aws/credentials  (profile via AWS_PROFILE, default: [default])
4. ~/.aws/config       (region, role_arn/source_profile chaining)
5. Assume-role provider (role_arn + source_profile in config)
6. ECS container credentials (AWS_CONTAINER_CREDENTIALS_RELATIVE_URI)
7. EC2 instance metadata service (IMDS) — instance profile
8. EKS IRSA web identity token (AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN)
```

## STS — AssumeRole / GetCallerIdentity / GetSessionToken

```python
sts = boto3.client("sts")

sts.get_caller_identity()   # {"UserId", "Account", "Arn"}

resp = sts.assume_role(
    RoleArn="arn:aws:iam::111122223333:role/my-role",
    RoleSessionName="identifiable-session-name",
    DurationSeconds=3600,          # 900–43200, capped by role's MaxSessionDuration
)
creds = resp["Credentials"]        # AccessKeyId, SecretAccessKey, SessionToken, Expiration

assumed = boto3.Session(
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
)
```

```bash
aws sts get-caller-identity --profile dev

aws sts assume-role \
  --role-arn arn:aws:iam::111122223333:role/my-role \
  --role-session-name cli-session

aws sts get-session-token \
  --serial-number arn:aws:iam::ACCOUNT_ID:mfa/USERNAME \
  --token-code 123456 \
  --duration-seconds 3600
```

## IMDS (EC2 instance profile credentials)

```bash
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
```

## S3 — upload / download

```python
s3 = boto3.client("s3")

# High-level (auto multipart via s3transfer, retries, progress callback)
s3.upload_file("local.txt", "bucket", "key.txt", ExtraArgs={"ServerSideEncryption": "AES256"})
s3.download_file("bucket", "key.txt", "local.txt")
s3.upload_fileobj(file_obj, "bucket", "key.txt")
s3.download_fileobj("bucket", "key.txt", file_obj)

# Low-level (single request, no auto multipart, 5 GB PUT cap)
s3.put_object(Bucket="bucket", Key="key.txt", Body=b"data")
resp = s3.get_object(Bucket="bucket", Key="key.txt")
data = resp["Body"].read()          # StreamingBody — one-shot read
```

## S3 — presigned URLs / POST

```python
s3.generate_presigned_url(
    "get_object", Params={"Bucket": "bucket", "Key": "key.txt"}, ExpiresIn=3600  # max 604800 (7d)
)
s3.generate_presigned_url(
    "put_object", Params={"Bucket": "bucket", "Key": "key.txt", "ContentType": "text/plain"}, ExpiresIn=300
)
s3.generate_presigned_post(
    Bucket="bucket", Key="uploads/${filename}",
    Conditions=[["content-length-range", 0, 5 * 1024 * 1024]], ExpiresIn=300
)
```

## S3 — multipart tuning

```python
from boto3.s3.transfer import TransferConfig

config = TransferConfig(
    multipart_threshold=25 * 1024 * 1024,
    multipart_chunksize=25 * 1024 * 1024,
    max_concurrency=10,
    use_threads=True,
)
s3.upload_file("big.iso", "bucket", "big.iso", Config=config)
```

## S3 — pagination

```python
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket="bucket", Prefix="logs/"):
    for obj in page.get("Contents", []):
        print(obj["Key"])
```

## S3 — 404 handling

```python
from botocore.exceptions import ClientError

try:
    s3.download_file("bucket", "missing.txt", "/tmp/out.txt")
except ClientError as e:
    if e.response["Error"]["Code"] == "404":
        print("not found")
    else:
        raise
```

## EC2 — describe / filter / paginate

```python
ec2 = boto3.client("ec2", region_name="us-east-1")

paginator = ec2.get_paginator("describe_instances")
for page in paginator.paginate(
    Filters=[
        {"Name": "instance-state-name", "Values": ["running"]},
        {"Name": "tag:Environment", "Values": ["prod"]},
    ]
):
    for reservation in page["Reservations"]:
        for instance in reservation["Instances"]:
            print(instance["InstanceId"], instance["State"]["Name"])
```

## EC2 — start / stop / tag / wait

```python
ec2.start_instances(InstanceIds=["i-0123456789abcdef0"])
ec2.stop_instances(InstanceIds=["i-0123456789abcdef0"], Force=False)

ec2.create_tags(
    Resources=["i-0123456789abcdef0"],
    Tags=[{"Key": "Owner", "Value": "unassigned"}],
)

ec2.get_waiter("instance_stopped").wait(InstanceIds=["i-0123456789abcdef0"])
ec2.get_waiter("instance_running").wait(InstanceIds=["i-0123456789abcdef0"])
```

## EC2 — all regions

```python
bootstrap = boto3.client("ec2", region_name="us-east-1")
regions = [r["RegionName"] for r in bootstrap.describe_regions()["Regions"]]

for region in regions:
    ec2 = boto3.client("ec2", region_name=region)
    # ... paginate describe_instances per region
```

## AWS CLI — profiles

```bash
aws configure --profile dev                 # interactive: access key, secret, region, output
aws configure list --profile dev
aws sts get-caller-identity --profile dev
export AWS_PROFILE=dev                       # avoid --profile on every command
```

```ini
# ~/.aws/config
[profile dev]
region = us-east-1
output = json

[profile prod]
region = eu-west-1
role_arn = arn:aws:iam::999988887777:role/ProdDeployRole
source_profile = dev
mfa_serial = arn:aws:iam::123456789012:mfa/username
```

## `aws-vault`

```bash
aws-vault add dev                            # store long-term key, encrypted in OS keychain
aws-vault list                               # profiles + cached sessions
aws-vault exec dev -- aws s3 ls
aws-vault exec prod --mfa-token=123456 -- terraform apply
aws-vault remove dev                         # delete stored credentials
```
