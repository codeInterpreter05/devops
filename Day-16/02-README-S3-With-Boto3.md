# Day 16 — Python for Automation II — AWS: S3 With Boto3

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** ⚡ Interview-critical

## Brief

S3 is the AWS service almost every automation script touches eventually — artifact storage, log archival, backup targets, generating shareable links for reports. Boto3 gives you two different API styles for the same operations (high-level transfer helpers vs. raw single-request calls), and picking the wrong one for the job either silently fails on large files or wastes effort re-implementing what boto3 already does for you. Presigned URLs in particular come up constantly in interviews because they test whether you actually understand SigV4 signing and credential lifetimes, not just "yeah you can generate a link."

## Upload/download: high-level vs. low-level APIs

### `upload_file` / `download_file` — the high-level transfer manager

```python
import boto3

s3 = boto3.client("s3")

s3.upload_file(
    Filename="backup.tar.gz",
    Bucket="my-audit-bucket",
    Key="backups/2026-07-12/backup.tar.gz",
    ExtraArgs={"ServerSideEncryption": "AES256", "StorageClass": "STANDARD_IA"},
)

s3.download_file(
    Bucket="my-audit-bucket",
    Key="backups/2026-07-12/backup.tar.gz",
    Filename="restore.tar.gz",
)
```

These are convenience wrappers around **`s3transfer`**, boto3's transfer manager. They:

- **Automatically switch to multipart upload/download** once the file crosses a size threshold (default 8 MB) — you don't write any multipart logic yourself.
- **Parallelize** parts across a thread pool for large files, and **retry individual failed parts** rather than restarting the whole transfer.
- Accept a `Callback=` argument for progress reporting and a `Config=` (`boto3.s3.transfer.TransferConfig`) to tune thresholds, concurrency, and chunk size.
- Work with local file paths directly (`upload_file`/`download_file`) or file-like objects in memory (`upload_fileobj`/`download_fileobj`), useful when you're streaming data rather than touching disk.

### `put_object` / `get_object` — the low-level, single-request API

```python
with open("config.json", "rb") as f:
    s3.put_object(
        Bucket="my-audit-bucket",
        Key="config/app-config.json",
        Body=f,
        ContentType="application/json",
    )

response = s3.get_object(Bucket="my-audit-bucket", Key="config/app-config.json")
data = response["Body"].read()   # StreamingBody — read once, can't seek back afterward
```

These map to exactly one HTTP request each:

- `put_object` is a **single `PUT`**, capped at **5 GB** — there is no automatic multipart behavior; pushing a file bigger than that with `put_object` simply fails.
- `get_object`'s `Body` is a `botocore.response.StreamingBody` — a one-shot stream. If you need to read it twice (e.g., compute a hash and also parse it), read it into a variable once and reuse that, or use `Range` requests to re-fetch a slice.
- These calls give you full control over the request (custom headers, exact `ContentType`, conditional writes via `IfNoneMatch`, etc.) and a full response dict (`ETag`, `VersionId`, `ResponseMetadata`) — useful when a script needs to inspect exactly what happened, not just "did it work."

**Rule of thumb:** use `upload_file`/`download_file` (or the `_fileobj` variants) for anything file-sized and >~a few MB, where you want multipart handled for you. Use `put_object`/`get_object` for small, precise payloads (config blobs, generated reports, JSON documents) where you want the raw request/response back.

## Presigned URLs

A presigned URL is a normal S3 URL with a SigV4 signature embedded in the query string, computed using **the credentials of whoever generated it** — anyone holding the URL can perform that one specific action (`GET`, `PUT`, etc.) on that one specific object, without having any AWS credentials of their own.

```python
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-audit-bucket", "Key": "reports/q3-2026.pdf"},
    ExpiresIn=3600,   # seconds
)

upload_url = s3.generate_presigned_url(
    "put_object",
    Params={
        "Bucket": "my-audit-bucket",
        "Key": "uploads/user-42/avatar.png",
        "ContentType": "image/png",
    },
    ExpiresIn=300,
)
```

For **browser-based direct uploads** where you also want to constrain size or enforce fields, `generate_presigned_post` builds a signed form (POST policy) instead of a single signed URL:

```python
post = s3.generate_presigned_post(
    Bucket="my-audit-bucket",
    Key="uploads/${filename}",
    Conditions=[["content-length-range", 0, 5 * 1024 * 1024]],  # cap at 5 MB
    ExpiresIn=300,
)
# post["url"] + post["fields"] get embedded directly into an HTML <form>
```

**Mechanics and expiry, precisely:**

- `ExpiresIn` sets how long the *signature itself* remains valid, capped at **7 days (604800 seconds)** for SigV4 — you cannot generate a longer-lived presigned URL no matter what you pass.
- If the credentials used to sign the URL are **temporary** (an assumed role, an EC2 instance profile), the URL becomes unusable the moment those underlying credentials expire — **even if `ExpiresIn` hasn't elapsed yet**. A presigned URL generated from a role session that expires in 1 hour is dead in 1 hour regardless of whether you set `ExpiresIn=3600*24`.
- Presigned URLs inherit the **permissions of the signer** at generation time — if the signer's policy is later revoked or tightened, the URL stops working immediately even before its stated expiry, since S3 re-checks permissions on each request.

**Use cases:** letting an end user download a private report without giving them AWS credentials or making the bucket public; letting a browser or mobile client upload directly to S3 (bypassing your application server as a bottleneck) while still constraining what/where they can write.

## Multipart uploads

AWS recommends multipart above roughly 100 MB and **requires** it above 5 GB (the max size for a single `PUT`); the overall max object size via multipart is 5 TB. `upload_file`/`upload_fileobj` invoke it automatically via `TransferConfig`, but you can tune or drive it manually:

```python
from boto3.s3.transfer import TransferConfig

config = TransferConfig(
    multipart_threshold=25 * 1024 * 1024,   # switch to multipart above 25 MB
    multipart_chunksize=25 * 1024 * 1024,   # 25 MB per part
    max_concurrency=10,                       # parallel part uploads
    use_threads=True,
)
s3.upload_file("disk-image.iso", "my-audit-bucket", "isos/disk-image.iso", Config=config)
```

For full manual control (e.g., resumable uploads across process restarts, uploading parts sourced from different workers), the raw API is a three-call sequence: `create_multipart_upload` → repeated `upload_part` (each returning an `ETag` you must collect) → `complete_multipart_upload` with the full list of `{PartNumber, ETag}`. Manual multipart is rarely worth hand-rolling unless you need that level of control — `TransferConfig` covers the overwhelming majority of cases.

## Common gotchas

**Region mismatches.** S3 buckets live in a specific region even though the bucket namespace is global. A client created for `us-east-1` operating on a bucket that actually lives in `eu-west-1` gets a `PermanentRedirect` or `IllegalLocationConstraintException`. Always create the client with the bucket's actual region, or resolve it first:

```python
location = s3.get_bucket_location(Bucket="my-audit-bucket")["LocationConstraint"]
region = location or "us-east-1"   # get_bucket_location returns None/empty for us-east-1, oddly
s3_correct_region = boto3.client("s3", region_name=region)
```

**ACLs vs. bucket policies.** Since April 2023, new buckets default to **`BucketOwnerEnforced`** object ownership, which **disables ACLs entirely**. A `put_object(..., ACL="public-read")` or `upload_file(..., ExtraArgs={"ACL": "public-read"})` against such a bucket fails with `AccessControlListNotSupported`. Access control on modern buckets should be expressed as a **bucket policy** (a resource-based JSON policy attached to the bucket) rather than per-object ACLs — ACLs are effectively legacy at this point.

**Pagination.** `list_objects_v2` returns at most 1000 keys per call (`IsTruncated` + `NextContinuationToken` indicate more). Looping manually is error-prone — use a paginator:

```python
paginator = s3.get_paginator("list_objects_v2")
all_keys = []
for page in paginator.paginate(Bucket="my-audit-bucket", Prefix="logs/"):
    for obj in page.get("Contents", []):
        all_keys.append(obj["Key"])
```

**Key naming.** A leading `/` in a key (`/logs/app.log` instead of `logs/app.log`) creates an object whose name literally starts with a slash — it shows up as an unexpected empty-named "folder" in the console and is easy to introduce by accident when concatenating paths with `os.path.join` (which behaves differently across OSes) instead of plain string formatting.

**404 handling.** A missing key on `download_file`/`get_object` raises `botocore.exceptions.ClientError`, not a clean `FileNotFoundError` — you must catch it and inspect the error code:

```python
from botocore.exceptions import ClientError

try:
    s3.download_file("my-audit-bucket", "missing-key.txt", "/tmp/out.txt")
except ClientError as e:
    if e.response["Error"]["Code"] == "404":
        print("Object not found")
    else:
        raise
```

## Points to Remember

- `upload_file`/`download_file` (and `_fileobj` variants) handle multipart transparently via `s3transfer` — use them for anything file-sized.
- `put_object`/`get_object` are single-request, low-level calls — `put_object` caps at 5 GB, and `get_object`'s `Body` is a one-shot `StreamingBody`.
- Presigned URLs are capped at 7 days by SigV4 regardless of `ExpiresIn`, and die early if signed with temporary (STS) credentials that expire sooner than the stated `ExpiresIn`.
- Multipart is required above 5 GB, recommended above ~100 MB; `TransferConfig` controls the threshold/chunk size/concurrency boto3 uses automatically.
- New buckets default to ACLs disabled (`BucketOwnerEnforced`) — use bucket policies for access control, not `ACL=` parameters.
- Always paginate `list_objects_v2` — never assume a single call returns everything.

## Common Mistakes

- Using `put_object` on a multi-gigabyte file and hitting the 5 GB single-`PUT` limit instead of switching to `upload_file`/multipart.
- Setting a long `ExpiresIn` on a presigned URL generated from an assumed-role session and being surprised it stops working an hour later — the underlying STS credential expiry silently wins.
- Forgetting `get_object`'s `Body` can only be read once, then hitting empty results on a second `.read()` call later in the same function.
- Passing `ACL="public-read"` against a bucket with the modern default object-ownership setting and getting `AccessControlListNotSupported`, instead of writing a bucket policy.
- Creating an S3 client without specifying (or resolving) the bucket's actual region, then hitting `PermanentRedirect` errors that look like permissions issues but are actually routing issues.
- Not paginating `list_objects_v2` and silently processing only the first 1000 objects in a bucket with more — the code "works" in testing (small bucket) and quietly under-processes in production.
