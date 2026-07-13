# Day 114 — Cloud Cost Engineering: Storage & Data Transfer Costs

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** ⚡ Interview-critical

## Brief

Compute gets all the attention in cost conversations, but S3 storage class mismanagement and data transfer charges are the two categories most likely to be *invisibly* wasting money — because unlike an oversized EC2 instance, nobody looks at a dashboard and immediately notices "this data is in the wrong storage class" or "this cross-AZ chatter is costing $8k/month." These are the line items that surprise people during a bill review.

## S3 Intelligent-Tiering: automated lifecycle without guesswork

S3 offers multiple storage classes with different price/retrieval-latency/retrieval-cost tradeoffs: Standard, Standard-IA (Infrequent Access), One Zone-IA, Glacier Instant Retrieval, Glacier Flexible Retrieval, Glacier Deep Archive. The traditional approach — write **S3 Lifecycle rules** that transition objects to cheaper classes after N days — requires you to *know* your access pattern in advance and gets stale as usage patterns change.

**Intelligent-Tiering** removes the guesswork: it monitors access patterns per object and automatically moves objects between tiers:
- Frequent Access (default landing tier, same price as Standard)
- Infrequent Access (after 30 consecutive days with no access)
- Archive Instant Access (after 90 days) — still millisecond retrieval
- Archive Access / Deep Archive Access (optional, opt-in, 90-180+ days) — retrieval takes hours, for genuinely cold data

There's a small **per-object monitoring fee** (a few cents per 1,000 objects per month), so Intelligent-Tiering is a net loss for very small objects (<128KB, which are never auto-tiered anyway and stay in Frequent Access) or for buckets where you already know the exact access pattern with certainty (a nightly-write, never-read-again audit log bucket is cheaper with a hardcoded Lifecycle rule straight to Glacier).

**When to use which:**
- **Unpredictable/mixed access patterns** (user uploads, data lake landing zones, ML training datasets reused sporadically) → **Intelligent-Tiering**, set and forget.
- **Known, uniform access pattern** (logs you never re-read after 7 days) → plain **Lifecycle rules**, cheaper because no monitoring fee.
- **Compliance/archival data accessed rarely if ever** → go straight to **Glacier Deep Archive** via lifecycle, skip the intermediate tiers entirely.

```bash
# Enable Intelligent-Tiering as the default via a bucket-level lifecycle configuration
aws s3api put-bucket-lifecycle-configuration --bucket my-data-lake --lifecycle-configuration '{
  "Rules": [{
    "ID": "auto-intelligent-tiering",
    "Filter": {"Prefix": ""},
    "Status": "Enabled",
    "Transitions": [{"Days": 0, "StorageClass": "INTELLIGENT_TIERING"}]
  }]
}'
```

## Data transfer costs — the hidden bill

Data transfer is the line item almost nobody budgets for correctly, because AWS's pricing here is asymmetric and full of exceptions:

- **Data transfer IN** to AWS is free (from the internet, in almost all cases).
- **Data transfer OUT** to the internet is metered per GB, with the rate *decreasing* at higher volume tiers, and free egress up to 100GB/month at the account level (or more, depending on current AWS free-tier terms).
- **Cross-AZ transfer within the same region** is charged **both ways** (sender and receiver each pay ~$0.01/GB) — this is the one engineers most consistently forget exists, because "it's all in the same region, it should be basically free" is the intuitive (wrong) assumption.
- **Cross-region transfer** is charged at a higher rate than cross-AZ, and again on both legs.
- **Same-AZ transfer** is free.
- **NAT Gateway** adds its own per-GB *processing* charge on top of whatever data transfer charge already applies for traffic passing through it — a NAT Gateway used as the default egress path for a chatty microservice architecture can become a surprisingly large line item purely from the per-GB processing fee, independent of the underlying transfer cost.
- **VPC Endpoints** (Gateway endpoints for S3/DynamoDB are free; Interface endpoints for other services cost per-hour + per-GB, but usually far less than the equivalent NAT Gateway processing charge for high-volume traffic) can eliminate NAT Gateway costs entirely for traffic to AWS services, by routing it privately instead of out through the NAT.

**Where this bites in real architectures:**
- A microservices architecture where every service instance is randomly spread across 3 AZs, and services chat constantly (e.g., a service mesh doing east-west traffic for every request) — each cross-AZ hop is billed both ways. At high request volume this adds up to real money for traffic that never left the region.
- **Fix**: **AZ-aware/topology-aware routing** (Kubernetes' `topology.kubernetes.io/zone` label + `Service Internal Traffic Policy: Local`, or Istio locality load balancing) so a pod preferentially talks to a same-AZ replica of a dependency instead of a random one three AZs away. This trades a small amount of load-balancing evenness for a real reduction in cross-AZ egress charges.
- A NAT Gateway handling terabytes of outbound traffic to, say, an S3 bucket — replaceable with a free S3 Gateway VPC Endpoint, eliminating both the NAT processing fee and (depending on setup) any related data transfer charge for that specific path.

## Points to Remember

- Intelligent-Tiering trades a small per-object monitoring fee for automatic tier optimization — worth it for unpredictable access patterns, not worth it for tiny objects or already-known access patterns.
- Data transfer **into** AWS is (almost always) free; data transfer **out**, cross-AZ, and cross-region are all separately metered, and cross-AZ/cross-region charges apply on *both* sender and receiver legs.
- NAT Gateway bills for **data processed** (per GB) on top of any transfer charge — high-volume traffic through a NAT Gateway to AWS-native services (S3, DynamoDB, etc.) is almost always cheaper via a VPC Gateway/Interface Endpoint.
- Cross-AZ chattiness in a microservice/service-mesh architecture is a real, often-overlooked cost center — topology-aware routing reduces it without an architecture rewrite.
- Same-AZ transfer is free — colocating chatty services in the same AZ (where latency/HA requirements allow) is a legitimate cost lever, not just a performance one.

## Common Mistakes

- Assuming "it's all inside my VPC, in one region" means transfer is free — cross-AZ transfer inside a single region is billed on both ends and is easy to miss until someone actually breaks down the "EC2-Other" or "Data Transfer" line in Cost Explorer by usage type.
- Leaving a NAT Gateway as the default route for high-volume traffic to S3/DynamoDB instead of adding a free Gateway VPC Endpoint — a pure waste with no tradeoff, since the Gateway Endpoint is strictly cheaper and equally reliable for that traffic.
- Enabling Intelligent-Tiering on a bucket full of small (<128KB) objects and expecting savings — those objects never leave Frequent Access tier, so you pay the monitoring fee for nothing.
- Writing a Lifecycle rule that transitions straight to Glacier Deep Archive for data that occasionally *does* need same-day access, then getting hit with expensive expedited-retrieval fees and multi-hour wait times when someone actually needs the data.
- Not distinguishing "cost of storing the data" from "cost of moving the data" when diagnosing an S3-related bill spike — a spike is far more often a change in *access pattern* (more GET requests, more egress) than a change in stored volume.
