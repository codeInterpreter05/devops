# Day 35 — Lab: AWS Storage & Databases

**Goal:** Set up RDS Multi-AZ with automatic failover and test the failover time with a real connection script — this day's assigned hands-on activity — then get hands-on with S3 lifecycle/versioning, ElastiCache cluster mode, and DynamoDB partition-key behavior.

**Prerequisites:**
- AWS CLI configured with permission to create RDS, S3, ElastiCache, and DynamoDB resources (note: RDS Multi-AZ and ElastiCache cluster mode incur real hourly cost — use a sandbox account and clean up promptly).
- `psql` or `mysql` client installed locally, Python 3 with `boto3`/`psycopg2` for the failover test script.

---

### Lab 1 — The core hands-on activity: RDS Multi-AZ failover test

1. Create a Multi-AZ PostgreSQL instance:
   ```bash
   aws rds create-db-instance --db-instance-identifier day35-multiaz \
     --engine postgres --db-instance-class db.t3.medium --allocated-storage 20 \
     --master-username admin --master-user-password 'ChangeMe123!' --multi-az \
     --backup-retention-period 1
   ```
2. Wait for it to become available: `aws rds wait db-instance-available --db-instance-identifier day35-multiaz`.
3. Write a small Python script that connects to the DB endpoint and writes a timestamp to a table in a loop every 500ms, logging any connection errors with a timestamp:
   ```python
   import psycopg2, time, datetime
   while True:
       try:
           conn = psycopg2.connect(host="<ENDPOINT>", dbname="postgres", user="admin", password="ChangeMe123!", connect_timeout=3)
           cur = conn.cursor()
           cur.execute("SELECT 1")
           print(f"{datetime.datetime.now()} OK")
           conn.close()
       except Exception as e:
           print(f"{datetime.datetime.now()} FAIL: {e}")
       time.sleep(0.5)
   ```
4. Start the script running, then in another terminal trigger a failover:
   ```bash
   aws rds reboot-db-instance --db-instance-identifier day35-multiaz --force-failover
   ```
5. Watch the script's output — measure exactly how long the `FAIL` entries last before `OK` resumes.

**Success criteria:** You have a real, measured failover time (typically 60-120 seconds) from your own script's logs, and can explain why the DNS endpoint doesn't change but the underlying instance it resolves to does.

---

### Lab 2 — Read replica: create, lag, and promote

1. Create a read replica of the same instance:
   ```bash
   aws rds create-db-instance-read-replica --db-instance-identifier day35-replica \
     --source-db-instance-identifier day35-multiaz
   ```
2. Write a batch of rows to the primary, then immediately query the replica — observe (and note the timing of) any lag before the rows appear.
3. Check the `ReplicaLag` CloudWatch metric for the replica.
4. Promote the replica: `aws rds promote-read-replica --db-instance-identifier day35-replica`. Confirm afterward that it's now an independent, writable instance with no relationship to the original primary.

**Success criteria:** You've observed real (even if small) replication lag, and independently confirmed that promotion is a manual, one-way operation — not an automatic failover.

---

### Lab 3 — S3 versioning + lifecycle interaction

1. Create a bucket, enable versioning, upload a file, overwrite it 3 times, then delete it.
2. Run `aws s3api list-object-versions --bucket <bucket>` — identify the delete marker and all 4 real versions underneath it.
3. "Undelete" the object by removing the delete marker (`aws s3api delete-object --bucket <bucket> --key <key> --version-id <delete-marker-version-id>`).
4. Add a lifecycle rule with `NoncurrentVersionExpiration` at 1 day (for lab purposes), and explain in your own words why this rule is necessary for a versioned bucket's storage cost to not grow unbounded.

**Success criteria:** You've restored a "deleted" object using version history alone, and can explain the delete-marker mechanism from direct observation.

---

### Lab 4 — ElastiCache cluster mode and DynamoDB hot partitions

1. Create a small Redis replication group with cluster mode **enabled**, 2 shards. Connect with `redis-cli -c` and set keys until you can observe (via `CLUSTER KEYSLOT <key>`) that different keys land on different shards.
2. Use hash tags (`{user:1}:a` and `{user:1}:b`) and confirm via `CLUSTER KEYSLOT` that both land on the identical slot.
3. Create a DynamoDB table with a **deliberately bad** partition key (a low-cardinality attribute like a fixed `"type": "order"` value shared by every item). Write 100+ items and observe (via table metrics or by reasoning through the design) why this would throttle under real load regardless of provisioned/on-demand capacity.
4. Redesign the table with a better partition key (e.g., `CustomerId`) and a sort key (`OrderId`), and explain why this spreads load across many partitions instead of one.

**Success criteria:** You can point to a specific `CLUSTER KEYSLOT` result proving hash-tag co-location, and can articulate why the first DynamoDB key design creates a hot partition while the second doesn't.

---

### Cleanup

```bash
aws rds delete-db-instance --db-instance-identifier day35-replica --skip-final-snapshot
aws rds delete-db-instance --db-instance-identifier day35-multiaz --skip-final-snapshot
aws elasticache delete-replication-group --replication-group-id my-redis-cluster
aws dynamodb delete-table --table-name Orders
aws s3 rm s3://<bucket> --recursive
aws s3api delete-bucket --bucket <bucket>
```

### Stretch challenge

Enable a DynamoDB Stream on your well-designed `Orders` table with `NEW_AND_OLD_IMAGES`, attach a Lambda trigger that logs a message whenever an order's `Status` changes, and test it by updating an item's status — confirming an event-driven reaction fires without the writer knowing anything about the Lambda consumer.
