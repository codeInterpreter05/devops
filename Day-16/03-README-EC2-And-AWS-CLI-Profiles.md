# Day 16 — Python for Automation II — AWS: EC2 Automation & CLI Profiles

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** ⚡ Interview-critical

## Brief

Auditing and managing EC2 fleets is one of the most common real-world automation tasks — "which instances are running across all our regions," "shut down anything untagged," "stop dev instances after hours to save cost." Doing this correctly requires understanding that boto3 clients are **region-scoped** (there's no single API call that returns "every instance everywhere"), that responses are **paginated** and **deeply nested**, and that credentials for this kind of script should come from a properly configured profile — ideally one that never stores a plaintext long-lived key at all.

## EC2 automation with boto3

### `describe_instances`: the nested response shape

EC2's data model groups instances into **reservations** (a historical artifact of how EC2 originally batched launches) — a single `run_instances` call can produce a reservation containing multiple instances, so `describe_instances` returns a list of reservations, each containing a list of instances. You always have to flatten this:

```python
import boto3

ec2 = boto3.client("ec2", region_name="us-east-1")

response = ec2.describe_instances()
for reservation in response["Reservations"]:
    for instance in reservation["Instances"]:
        print(instance["InstanceId"], instance["InstanceType"], instance["State"]["Name"])
```

### Pagination

`describe_instances` can return a `NextToken` once you have enough instances/reservations that the response would otherwise be too large. Never assume one call is complete — use a paginator, which handles `NextToken` looping for you:

```python
paginator = ec2.get_paginator("describe_instances")
all_instances = []
for page in paginator.paginate():
    for reservation in page["Reservations"]:
        all_instances.extend(reservation["Instances"])
```

### Filtering, including by tag

`Filters` is a list of `{"Name": ..., "Values": [...]}` dicts, ANDed together (multiple values within one filter are ORed):

```python
paginator = ec2.get_paginator("describe_instances")
pages = paginator.paginate(
    Filters=[
        {"Name": "instance-state-name", "Values": ["running", "stopped"]},
        {"Name": "tag:Environment", "Values": ["prod"]},
    ]
)
```

There is **no server-side filter for "has no tags"** — EC2's filter API can match a tag's presence/value, but not its absence. To find untagged instances you must fetch everything and check client-side:

```python
def is_untagged(instance):
    return not instance.get("Tags")   # key absent entirely, or present but empty

untagged = [i for i in all_instances if is_untagged(i)]
```

### Starting, stopping, and waiting

```python
ec2.start_instances(InstanceIds=["i-0123456789abcdef0"])
ec2.stop_instances(InstanceIds=["i-0123456789abcdef0"], Force=False)

# State transitions are asynchronous — a waiter polls until the target state is reached
waiter = ec2.get_waiter("instance_stopped")
waiter.wait(InstanceIds=["i-0123456789abcdef0"])
```

`stop_instances`/`start_instances` return immediately once the request is *accepted*, not once the instance has actually reached the target state — checking `instance["State"]["Name"]` right after calling `stop_instances` will often still show `stopping`, not `stopped`. Use a waiter (or poll `describe_instances` yourself) if the script's next step depends on the instance actually being stopped.

### Iterating across every region

Boto3 clients are bound to one region at creation time — there is no "all regions" EC2 API. You have to enumerate regions first, then create a client per region:

```python
ec2_bootstrap = boto3.client("ec2", region_name="us-east-1")   # any region works for describe_regions
regions = [r["RegionName"] for r in ec2_bootstrap.describe_regions()["Regions"]]

all_instances = []
for region in regions:
    ec2 = boto3.client("ec2", region_name=region)
    paginator = ec2.get_paginator("describe_instances")
    try:
        for page in paginator.paginate():
            for reservation in page["Reservations"]:
                all_instances.extend(reservation["Instances"])
    except Exception as exc:
        print(f"Skipping {region}: {exc}")
```

**Gotcha:** `describe_regions()` by default only returns regions your account has **opted into** (some regions like `me-south-1`, `ap-east-1` require explicit opt-in and are excluded unless you pass `AllRegions=True`) — and even with `AllRegions=True`, you can't actually query a disabled region's resources, so the practical default (opted-in regions only) is usually what you want for an audit script anyway. Also wrap the per-region loop in a `try/except`: an account-level restriction or an unreachable region shouldn't kill the entire script.

## AWS CLI named profiles

The CLI and boto3 share the same configuration files: `~/.aws/config` (region, output format, and role/profile chaining) and `~/.aws/credentials` (the actual key material for profiles that use static keys).

```ini
# ~/.aws/config
[profile dev]
region = us-east-1
output = json

[profile prod]
region = eu-west-1
role_arn = arn:aws:iam::999988887777:role/ProdDeployRole
source_profile = dev
mfa_serial = arn:aws:iam::123456789012:mfa/sarthu
```

```ini
# ~/.aws/credentials
[dev]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

Using `prod` above: the CLI (or boto3, via `Session(profile_name="prod")`) uses `dev`'s static keys to call STS `AssumeRole` against `ProdDeployRole`, prompting for the MFA code from the device identified by `mfa_serial`, and **caches** the resulting temporary token under `~/.aws/cli/cache/` so you're not re-prompted for every command within the token's validity window.

```bash
aws s3 ls --profile prod                 # or: export AWS_PROFILE=prod
aws sts get-caller-identity --profile prod
```

## MFA-protected access

For ad-hoc MFA (outside of a chained profile), request a temporary session directly via STS:

```bash
aws sts get-session-token \
  --serial-number arn:aws:iam::123456789012:mfa/sarthu \
  --token-code 123456 \
  --duration-seconds 3600
```

This returns `AccessKeyId`/`SecretAccessKey`/`SessionToken` you can export as environment variables or drop into a temporary named profile. Note this is **`GetSessionToken`**, distinct from `AssumeRole` — it grants the *same* permissions the calling IAM user already has (with the added `aws:MultiFactorAuthPresent` condition satisfied), it does not switch identity to a role. Use `GetSessionToken` when you just need to satisfy an MFA-required policy condition on your own user; use `AssumeRole` when you need to become a *different* identity (cross-account access, a service role, a more restricted role for a specific task).

## `aws-vault` as the safer alternative

Plaintext keys in `~/.aws/credentials` are a standing liability — anything with filesystem access (malware, a misconfigured backup job, a shared screen) can read them, and they don't expire on their own. `aws-vault` replaces that file with OS-native encrypted storage and only ever hands **temporary** credentials to the processes you actually run:

```bash
aws-vault add dev                          # stores the long-term key pair encrypted in the OS keychain
aws-vault list                             # shows configured profiles and any cached sessions
aws-vault exec dev -- aws ec2 describe-instances --region us-east-1
aws-vault exec prod --mfa-token=123456 -- terraform apply
```

Under `aws-vault exec`, the child process gets `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN` env vars for a short-lived STS session — the actual long-lived key never leaves the encrypted store and is never written to disk in plaintext. This is the direct fix for the risk that a plain `~/.aws/credentials` file represents, and it composes cleanly with MFA-protected and role-chained profiles.

## Points to Remember

- `describe_instances` nests instances inside reservations — always flatten with a double loop (or a comprehension) before processing.
- Always paginate (`get_paginator("describe_instances")`) — a single unpaginated call is not guaranteed to return everything once an account has enough instances.
- There's no "instances with no tags" server-side filter — fetch and check `instance.get("Tags")` client-side.
- `start_instances`/`stop_instances` are asynchronous; use a waiter if subsequent logic depends on the instance having actually reached the target state.
- EC2 clients are region-scoped — cross-region automation means enumerating `describe_regions()` and creating one client per region.
- `GetSessionToken` (MFA on your own identity) and `AssumeRole` (switching to a different identity) solve different problems — know which one a given profile/script actually needs.
- `aws-vault` stores long-lived keys encrypted and only exposes short-lived STS credentials to running commands — it doesn't replace IAM roles, it hardens the local-credential-storage problem.

## Common Mistakes

- Writing `for instance in response["Instances"]` directly against a `describe_instances` response and getting a `KeyError` — forgetting the `Reservations` nesting layer.
- Assuming one `describe_instances()` call returns the whole fleet and silently missing instances once pagination kicks in at scale.
- Trying to filter for untagged instances via `Filters=[{"Name": "tag-key", "Values": [""]}]` or similar — there's no such filter; it must be done client-side after fetching.
- Checking instance state immediately after `stop_instances()` and treating a `stopping` result as a failure, instead of waiting for `stopped` via a waiter or a polling loop.
- Hardcoding a single region in what's meant to be a fleet-wide audit script, silently missing every instance outside that region.
- Confusing `GetSessionToken` with `AssumeRole` — using `GetSessionToken` when the actual goal is to assume a cross-account role (it can't do that; it only re-issues the calling user's own permissions with MFA satisfied).
- Keeping long-lived keys in plaintext `~/.aws/credentials` on a laptop used for production access instead of migrating to `aws-vault` (or, better, eliminating static keys entirely in favor of SSO/role-based access).
