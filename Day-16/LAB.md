# Day 16 — Lab: Python for Automation II — AWS

**Goal:** Use boto3 and the AWS CLI to authenticate via roles (not hardcoded keys), operate on S3 and EC2, and build the assigned hands-on script: list all EC2 instances across regions, tag the untagged ones, and export a CSV audit report to S3.

**Prerequisites:**
- An AWS account (the AWS Free Tier covers everything here — IAM, STS, and S3 calls are free; EC2 usage is free-tier eligible for 750 hours/month of `t2.micro`/`t3.micro` for 12 months on a new account). **If you don't want to launch any EC2 instances, Labs 1–4 and the core script in Lab 5 still work against zero or existing instances — the script handles an empty account gracefully.**
- Python 3.9+, `pip install boto3`.
- AWS CLI v2 installed (`aws --version`).
- An IAM user with console/API access and permission to create roles, and to use EC2, S3, and STS (for a personal learning account, `AdministratorAccess` on your own IAM user is acceptable; in a shared/work account, ask for `IAMFullAccess` + `AmazonEC2FullAccess` + `AmazonS3FullAccess` scoped to a sandbox).
- `aws configure` already run once with a base profile (call it `[default]` or `[dev]`) so the CLI/boto3 have *some* starting credentials to bootstrap the labs below.

**Cost/cleanup awareness:** Nothing in Labs 1–4 costs money (IAM, STS, and S3 API calls at this scale are within the free tier or negligible in dollars). Lab 5 optionally launches EC2 instances to have something to audit — if you do, **stop or terminate them immediately after the lab** per the Cleanup section. `t2.micro`/`t3.micro` is free-tier eligible for new accounts; on an older account or after exceeding free-tier hours, leaving instances running does cost money.

---

### Lab 1 — Sanity-check your identity and install tooling

1. Confirm boto3 and the CLI agree on who you are:
   ```bash
   aws sts get-caller-identity --profile default
   ```
   ```python
   import boto3
   print(boto3.client("sts").get_caller_identity())
   ```
2. Note your **Account ID** and **user ARN** from the output — you'll need the account ID for the role ARNs in Lab 2.
3. Confirm the credential chain resolves the way you expect: temporarily unset any `AWS_ACCESS_KEY_ID` env var if one is set, and re-run step 1 to confirm it still falls back to `~/.aws/credentials`.

**Success criteria:** `get_caller_identity()` returns the same account ID and ARN from both the CLI and boto3, and you can state out loud which step of the credential resolution chain supplied the credentials.

---

### Lab 2 — IAM role assumption with STS

This lab creates a second role in your own account and assumes it, to see the credential-swap mechanics for real without needing a second AWS account.

1. Save a trust policy that allows *your own IAM user* to assume the new role (replace `ACCOUNT_ID` and `USER_NAME`):
   ```bash
   cat > /tmp/trust-policy.json <<'EOF'
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "AWS": "arn:aws:iam::ACCOUNT_ID:user/USER_NAME" },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   EOF
   ```
2. Create the role and attach a read-only permission policy:
   ```bash
   aws iam create-role \
     --role-name lab-readonly-role \
     --assume-role-policy-document file:///tmp/trust-policy.json

   aws iam attach-role-policy \
     --role-name lab-readonly-role \
     --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
   ```
3. Assume it from Python and prove the identity actually changed:
   ```python
   import boto3

   sts = boto3.client("sts")
   print("Before:", sts.get_caller_identity()["Arn"])

   resp = sts.assume_role(
       RoleArn="arn:aws:iam::ACCOUNT_ID:role/lab-readonly-role",
       RoleSessionName="lab-day16-session",
   )
   creds = resp["Credentials"]

   assumed_session = boto3.Session(
       aws_access_key_id=creds["AccessKeyId"],
       aws_secret_access_key=creds["SecretAccessKey"],
       aws_session_token=creds["SessionToken"],
   )
   print("After:", assumed_session.client("sts").get_caller_identity()["Arn"])
   print("Expires:", creds["Expiration"])
   ```
4. From the assumed session, try an action the `ReadOnlyAccess` policy does **not** allow (e.g., `assumed_session.client("iam").create_user(UserName="test")`) and confirm you get an `AccessDenied` — this proves the assumed identity is genuinely restricted, not just relabeled.

**Success criteria:** The "After" ARN shows `assumed-role/lab-readonly-role/lab-day16-session`, differs from the "Before" ARN, `creds["Expiration"]` is roughly one hour out, and a write action under the assumed session is denied.

---

### Lab 3 — AWS CLI named profiles with MFA

Skip the MFA-specific steps if you don't have a virtual MFA device set up on your IAM user yet — set one up first via IAM console (Security credentials → Assign MFA device → virtual MFA app like Google Authenticator/Authy), it takes two minutes and is worth having regardless.

1. Add a role-chained profile to `~/.aws/config` that assumes `lab-readonly-role` from Lab 2 through your base profile, requiring MFA:
   ```ini
   [profile lab-readonly]
   role_arn = arn:aws:iam::ACCOUNT_ID:role/lab-readonly-role
   source_profile = default
   mfa_serial = arn:aws:iam::ACCOUNT_ID:mfa/YOUR_USERNAME
   region = us-east-1
   ```
   (This requires updating the trust policy from Lab 2 to also require `aws:MultiFactorAuthPresent` — add a `Condition` block: `"Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}`.)
2. Run a command against the profile — the CLI will prompt for your MFA code interactively:
   ```bash
   aws s3 ls --profile lab-readonly
   ```
3. Separately, request a standalone MFA-authenticated session via `GetSessionToken` (no role switch, just your own identity with MFA satisfied):
   ```bash
   aws sts get-session-token \
     --serial-number arn:aws:iam::ACCOUNT_ID:mfa/YOUR_USERNAME \
     --token-code 123456 \
     --duration-seconds 3600
   ```
4. In one sentence, write down the difference between what step 2 (`AssumeRole` via a chained profile) and step 3 (`GetSessionToken`) actually gave you.

**Success criteria:** `aws s3 ls --profile lab-readonly` succeeds only after entering a valid MFA code, and you can articulate that `AssumeRole` switches identity while `GetSessionToken` re-issues your own identity's permissions with MFA satisfied.

---

### Lab 4 — S3 upload, download, and presigned URLs

1. Create a bucket (bucket names are globally unique — pick something distinctive):
   ```bash
   aws s3 mb s3://YOUR_UNIQUE_BUCKET_NAME --region us-east-1
   ```
2. Upload and download a file both ways, confirming the round trip:
   ```python
   import boto3

   s3 = boto3.client("s3")
   bucket = "YOUR_UNIQUE_BUCKET_NAME"

   with open("/tmp/hello.txt", "w") as f:
       f.write("hello from day 16\n")

   s3.upload_file("/tmp/hello.txt", bucket, "lab/hello.txt")
   s3.download_file(bucket, "lab/hello.txt", "/tmp/hello-downloaded.txt")

   with open("/tmp/hello-downloaded.txt") as f:
       assert f.read() == "hello from day 16\n"
   print("Round trip OK")
   ```
3. Generate a presigned GET URL with a short expiry and fetch it with `curl` (no AWS credentials involved on the `curl` side):
   ```python
   url = s3.generate_presigned_url(
       "get_object", Params={"Bucket": bucket, "Key": "lab/hello.txt"}, ExpiresIn=60
   )
   print(url)
   ```
   ```bash
   curl -s "<paste the URL>"
   ```
4. Wait for the URL to expire (60 seconds), then retry the same `curl` and confirm it now fails with `AccessDenied`/`ExpiredToken` in the XML error response.
5. Generate a presigned **PUT** URL and use it to upload a file with `curl`, without ever configuring AWS credentials in the shell doing the upload:
   ```python
   put_url = s3.generate_presigned_url(
       "put_object", Params={"Bucket": bucket, "Key": "lab/uploaded-via-curl.txt"}, ExpiresIn=120
   )
   print(put_url)
   ```
   ```bash
   curl -X PUT --data-binary "uploaded without credentials" "<paste the PUT URL>"
   ```

**Success criteria:** The presigned GET URL works before expiry and fails after it; the presigned PUT URL successfully creates a new object in the bucket from a shell with zero AWS credentials configured.

---

### Lab 5 — Core hands-on: cross-region EC2 audit, auto-tag, CSV to S3

This is the assigned hands-on activity. If you don't want to spend money/quota launching instances, you can run this against whatever EC2 footprint already exists in the account (including zero instances — the script should handle that gracefully and produce an empty-but-valid CSV).

**Optional setup — launch a couple of throwaway free-tier instances to have something to audit:**
```bash
aws ec2 run-instances --image-id ami-0c02fb55956c7d316 --count 2 \
  --instance-type t2.micro --region us-east-1
```
(Use `aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2"` to find a current Amazon Linux AMI ID for your region if the one above is stale.) Launch these **without** a `Name`/other tag, so the script below has something untagged to find.

**The script:**

```python
import csv
import datetime
import io

import boto3
from botocore.exceptions import ClientError


def get_all_regions():
    ec2 = boto3.client("ec2", region_name="us-east-1")
    return [r["RegionName"] for r in ec2.describe_regions()["Regions"]]


def list_all_instances(regions):
    rows = []
    for region in regions:
        ec2 = boto3.client("ec2", region_name=region)
        paginator = ec2.get_paginator("describe_instances")
        try:
            for page in paginator.paginate():
                for reservation in page["Reservations"]:
                    for instance in reservation["Instances"]:
                        tags = {t["Key"]: t["Value"] for t in instance.get("Tags", [])}
                        rows.append({
                            "Region": region,
                            "InstanceId": instance["InstanceId"],
                            "InstanceType": instance["InstanceType"],
                            "State": instance["State"]["Name"],
                            "LaunchTime": instance["LaunchTime"].isoformat(),
                            "Tags": tags,
                        })
        except ClientError as exc:
            print(f"Skipping region {region}: {exc}")
    return rows


def tag_untagged(rows):
    tagged_count = 0
    for row in rows:
        if row["Tags"]:
            continue
        ec2 = boto3.client("ec2", region_name=row["Region"])
        new_tags = {
            "Owner": "unassigned",
            "AutoTaggedOn": datetime.date.today().isoformat(),
        }
        ec2.create_tags(
            Resources=[row["InstanceId"]],
            Tags=[{"Key": k, "Value": v} for k, v in new_tags.items()],
        )
        row["Tags"] = new_tags
        tagged_count += 1
    return tagged_count


def export_csv_to_s3(rows, bucket, key):
    buffer = io.StringIO()
    fieldnames = ["Region", "InstanceId", "InstanceType", "State", "LaunchTime", "Tags"]
    writer = csv.DictWriter(buffer, fieldnames=fieldnames)
    writer.writeheader()
    for row in rows:
        writer.writerow({**row, "Tags": str(row["Tags"])})

    s3 = boto3.client("s3")
    s3.put_object(Bucket=bucket, Key=key, Body=buffer.getvalue().encode("utf-8"))


if __name__ == "__main__":
    BUCKET = "YOUR_UNIQUE_BUCKET_NAME"
    KEY = f"ec2-audit/{datetime.date.today().isoformat()}.csv"

    regions = get_all_regions()
    print(f"Scanning {len(regions)} regions...")

    instances = list_all_instances(regions)
    print(f"Found {len(instances)} instances total.")

    tagged = tag_untagged(instances)
    print(f"Tagged {tagged} previously-untagged instances.")

    export_csv_to_s3(instances, BUCKET, KEY)
    print(f"Exported audit report to s3://{BUCKET}/{KEY}")
```

Run it, then verify the result:
```bash
aws s3 cp s3://YOUR_UNIQUE_BUCKET_NAME/ec2-audit/$(date +%F).csv - | column -s, -t
aws ec2 describe-instances --region us-east-1 \
  --query "Reservations[].Instances[].Tags" --output table
```

**Success criteria:** The script runs to completion without unhandled exceptions across all regions in the account, every instance that was untagged before the run now has `Owner=unassigned` and `AutoTaggedOn` tags, and a well-formed CSV listing every instance (region, ID, type, state, launch time, tags) exists at the expected S3 key.

---

### Cleanup

```bash
# Terminate any lab instances you launched (skip if you audited pre-existing infra you want to keep)
aws ec2 describe-instances --region us-east-1 \
  --filters "Name=tag:AutoTaggedOn,Values=$(date +%F)" \
  --query "Reservations[].Instances[].InstanceId" --output text

aws ec2 terminate-instances --region us-east-1 --instance-ids <ids from above>

# Empty and remove the lab bucket
aws s3 rm s3://YOUR_UNIQUE_BUCKET_NAME --recursive
aws s3 rb s3://YOUR_UNIQUE_BUCKET_NAME

# Remove the lab IAM role
aws iam detach-role-policy --role-name lab-readonly-role \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
aws iam delete-role --role-name lab-readonly-role

# Remove local scratch files
rm -f /tmp/trust-policy.json /tmp/hello.txt /tmp/hello-downloaded.txt
```

Double-check `aws ec2 describe-instances --region <each region you touched>` shows nothing left running (`terminated` or absent) — this is the step people skip and then get a surprise bill for.

### Stretch challenge

Convert the Lab 5 script into a scheduled automation with **no static credentials anywhere**: package it to run on an EC2 instance (or as a Lambda function) with an IAM **instance profile** (or Lambda execution role) that has exactly the permissions it needs (`ec2:DescribeRegions`, `ec2:DescribeInstances`, `ec2:CreateTags`, `s3:PutObject` scoped to the audit bucket/prefix only — nothing broader), trigger it on an EventBridge scheduled rule (e.g., daily at 06:00 UTC), and confirm via CloudTrail that the calls show up under the role's identity, not a user's.
