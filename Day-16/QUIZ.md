# Day 16 — Quiz: Python for Automation II — AWS

Try to answer without looking at your notes. Answers are at the bottom.

1. What's the practical difference between `boto3.client("s3")` and `boto3.resource("s3")`, and which should you default to for new automation code?
2. List the boto3 credential resolution chain, in order, for at least the first four sources it checks.
3. Why are IAM roles preferred over long-lived IAM user access keys for scripts and automation?
4. What does `sts.assume_role()` return, and what field distinguishes the resulting credentials from a normal long-lived access key pair?
5. What is an EC2 instance profile, and how does a boto3 script running on an EC2 instance get credentials without any code referencing them at all?
6. What's the difference between IMDSv1 and IMDSv2, and why was IMDSv2 introduced?
7. What's the difference between `upload_file`/`download_file` and `put_object`/`get_object` in boto3's S3 client?
8. How do presigned URLs work, and what actually determines their real maximum lifetime when generated using temporary (STS) credentials?
9. Why must you paginate `describe_instances` and `list_objects_v2`, and what's a realistic consequence of not doing so?
10. What problem does `aws-vault` solve that a plain `~/.aws/credentials` file doesn't?
11. How would you use boto3 to find EC2 instances with no tags at all, given that there's no server-side "untagged" filter?
12. **Interview question:** How does IAM role assumption work in Python? What's the difference between STS and instance profiles?

---

## Answers

1. `client` is a low-level, 1:1 mapping onto the AWS API — every service and every operation is available, and responses are raw dicts. `resource` is a higher-level, object-oriented wrapper built on top of the client, but only exists for a subset of services and is receiving no new features going forward. Default to `client` for new code since it's guaranteed complete coverage; use `resource` only where the object-oriented style is a clear readability win (most commonly S3).

2. In order: (1) explicit credential parameters passed to `client()`/`Session()`, (2) environment variables (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`), (3) the shared credentials file `~/.aws/credentials`, (4) the config file `~/.aws/config` (including role/source-profile chaining), followed by (5) the assume-role provider, (6) ECS container credentials, (7) EC2 instance metadata (IMDS), and (8) EKS IRSA web identity tokens. The first source found is used exclusively — there's no partial merging across sources.

3. Roles issue temporary credentials via STS that auto-expire (max 12 hours, often much less) and can auto-rotate with zero code changes, versus access keys which are static, never expire on their own, and must be manually rotated/revoked. Roles are also individually auditable per assumed session (unique `RoleSessionName` in CloudTrail), their trust policies can require conditions like MFA that are impossible to enforce on bearer-style static keys, and there's no secret material that can leak from a git commit or a misconfigured backup the way an access key pair can.

4. It returns a `Credentials` dict containing `AccessKeyId`, `SecretAccessKey`, `SessionToken`, and `Expiration`, plus an `AssumedRoleUser` dict with the resulting identity's ARN. The `SessionToken` is the key distinguishing field — every request made with these credentials must include it, and it's what marks them as temporary; a normal long-lived access key pair has no session token at all.

5. An instance profile is the IAM object that attaches a role to an EC2 instance. Once attached, the Instance Metadata Service (a link-local HTTP endpoint at `169.254.169.254`, reachable only from within the instance) automatically serves temporary credentials for that role. Because this is step 7 in boto3's default credential resolution chain, a script that calls `boto3.client(...)` with no explicit credentials on such an instance picks these up automatically, and botocore refreshes them in the background before they expire — no `assume_role` call appears anywhere in the application code.

6. IMDSv1 answers plain, unauthenticated `GET` requests to the metadata endpoint. IMDSv2 requires first obtaining a session token via a `PUT` request, then including that token on subsequent `GET` requests. IMDSv2 was introduced to close a server-side request forgery (SSRF) attack path: many SSRF vulnerabilities in web apps can only forge simple GET-style requests, which IMDSv1 would happily answer with the instance's IAM credentials (this is essentially how the 2019 Capital One breach happened); requiring a PUT to get a token first blocks that class of attack. Best practice is to enforce IMDSv2-only (`HttpTokens: required`) on instance metadata options.

7. `upload_file`/`download_file` (and the `_fileobj` variants) are high-level transfer-manager calls that automatically switch to multipart transfer above a size threshold, parallelize parts, and retry failed parts individually — good for file-sized data. `put_object`/`get_object` are single-request, low-level calls: `put_object` is capped at 5 GB with no automatic multipart, and `get_object`'s `Body` is a one-shot `StreamingBody` that can only be read once. Use the high-level pair for general file transfer and the low-level pair when you need precise control over a single request or are handling small, in-memory payloads.

8. A presigned URL embeds a SigV4 signature computed with the signer's credentials directly in the query string, letting anyone holding the URL perform that one action on that one object without having AWS credentials themselves. `ExpiresIn` sets the signature's stated validity, capped at 7 days (604800 seconds) for SigV4 regardless of what you pass. However, if the credentials used to sign the URL were temporary (an assumed role or instance profile), the URL becomes unusable the moment those underlying credentials expire — even if `ExpiresIn` hasn't elapsed yet. So the real max lifetime is `min(ExpiresIn, time remaining on the signing credentials)`.

9. Both APIs cap the number of items returned per call (`describe_instances` can return a `NextToken` once results get large; `list_objects_v2` returns at most 1000 keys per call). Not paginating means a script silently processes only the first page — it will "work" in testing against small accounts/buckets and then quietly under-process (miss instances, miss objects) once the account/bucket grows past the page size, without raising any error to indicate data was left out.

10. A plain `~/.aws/credentials` file stores long-lived access keys in plaintext on disk, readable by anything with filesystem access (malware, a careless backup, a shared screen) and never expiring on its own. `aws-vault` stores that same long-lived key encrypted in the OS-native secret store (e.g., macOS Keychain) and, on each invocation, uses it to obtain short-lived STS credentials that are injected only as environment variables into the specific command being run — the long-lived key itself is never exposed to that command or left in plaintext at rest.

11. There's no EC2 filter for "no tags," so you fetch all instances (paginated, across whatever regions are relevant) and filter client-side by checking whether each instance's `Tags` key is absent or empty: `[i for i in instances if not i.get("Tags")]`.

12. **Model answer:** IAM role assumption in Python happens through `boto3.client("sts").assume_role(RoleArn=..., RoleSessionName=...)`, which calls AWS STS and returns temporary credentials — an `AccessKeyId`, `SecretAccessKey`, `SessionToken`, and an `Expiration` timestamp (default 1 hour, up to 12 hours if the role's `MaxSessionDuration` allows it). You typically build a new `boto3.Session` from those three values and create clients from that session to act as the assumed role. This requires the role to have a trust policy naming your identity (or service) as an allowed principal.

    STS and instance profiles solve the same underlying problem — issuing short-lived, auto-expiring credentials instead of static keys — but at different points in the flow. **STS `AssumeRole` is an explicit API call** your code (or the CLI) makes on demand, useful for cross-account access, switching to a more restricted role for a specific task, or CI pipelines assuming a deploy role. **An instance profile is implicit and automatic**: it's a role attached to an EC2 instance (or the equivalent for ECS tasks/EKS pods via IRSA), and the Instance Metadata Service serves and auto-refreshes temporary credentials for that role without any `assume_role` call appearing in application code at all — boto3's default credential chain just finds them there. In short: instance profiles are how a *compute resource* gets an identity baked in by its environment; STS `AssumeRole` is how *code* deliberately switches to a different identity mid-flight.
