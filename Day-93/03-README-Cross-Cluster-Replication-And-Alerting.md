# Day 93 — OpenSearch Deep Dive: Cross-Cluster Replication & Alerting

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Logging | **Flag:** —

## Brief

The final piece of the OpenSearch picture is what happens *across* clusters (disaster recovery, geo-distribution) and *on top of* stored data (turning search queries into actual alerts, the OpenSearch equivalent of what Alertmanager/Grafana alerting do for metrics/logs elsewhere in this phase). Both features matter for the same underlying reason: once logs are valuable enough to build ILM policies and shard-sizing discipline around (this day's first two files), they're valuable enough to need durability across cluster failure and automated detection of patterns within them.

## Cross-cluster replication (CCR)

OpenSearch's **Cross-Cluster Replication** continuously replicates indices from a **leader** cluster to a **follower** cluster, at the shard level — the follower cluster pulls operations from the leader's transaction log (translog) and replays them, keeping a near-real-time copy without requiring the application to write to both clusters itself.

```json
PUT _plugins/_replication/follower-index/_start
{
  "leader_alias": "leader-cluster",
  "leader_index": "logs-2026.07.12",
  "use_roles": {
    "leader_cluster_role": "replication_leader_role",
    "follower_cluster_role": "replication_follower_role"
  }
}
```

**Why this matters operationally**, beyond "it's a backup":
- **Disaster recovery** — a follower cluster in a different region/data center can be promoted to accept writes if the leader cluster becomes unavailable, without needing to restore from a snapshot (which is slower and loses whatever wasn't captured since the last snapshot).
- **Geo-distributed read scaling** — teams in a different region can query a local follower cluster with much lower latency than querying across a WAN to the leader, useful for globally-distributed teams doing log search/analytics.
- **Compliance / data residency** — replicating specific indices to a cluster physically located in a required jurisdiction, without duplicating your entire logging pipeline's ingest path.

**Important distinction from snapshots**: CCR is continuous and near-real-time (shipping ongoing operations), whereas **snapshots** (to S3/a shared filesystem repository) are point-in-time backups taken on a schedule. CCR gives you a warm, queryable, near-current standby cluster; snapshots give you a cold, restorable-but-not-immediately-queryable backup further back in time. Mature setups typically use **both** — CCR for fast failover, snapshots for longer-term/immutable backup and protection against replicating a *logical* corruption (a bad delete/mapping change replicates to the follower too, since CCR faithfully mirrors the leader's operations; a snapshot from before the bad change does not).

## Alerting in OpenSearch

OpenSearch's alerting plugin lets you define a **monitor** (a scheduled query against one or more indices) and **triggers** (conditions on the monitor's query result that, when true, fire **actions** — notifications).

```json
POST _plugins/_alerting/monitors
{
  "type": "monitor",
  "name": "high-5xx-error-rate",
  "monitor_type": "query_level_monitor",
  "enabled": true,
  "schedule": { "period": { "interval": 5, "unit": "MINUTES" } },
  "inputs": [{
    "search": {
      "indices": ["logs-*"],
      "query": {
        "query": {
          "bool": {
            "filter": [
              { "range": { "@timestamp": { "gte": "now-5m" } } },
              { "term": { "status_code": "500" } }
            ]
          }
        }
      }
    }
  }],
  "triggers": [{
    "name": "too-many-500s",
    "severity": "1",
    "condition": { "script": { "source": "ctx.results[0].hits.total.value > 100" } },
    "actions": [{
      "name": "notify-slack",
      "destination_id": "<slack-destination-id>",
      "message_template": { "source": "More than 100 5xx errors in the last 5 minutes." }
    }]
  }]
}
```

This is structurally analogous to a Prometheus alerting rule (Day 89) — a query on a schedule, a condition that decides whether it's "firing," and a notification action — except the query language is OpenSearch's query DSL against indexed log documents instead of PromQL against time series. Two monitor types matter:
- **Query-level monitors** — like the example above, run one query, evaluate one trigger condition against the aggregate result.
- **Bucket-level monitors** — run an aggregation query (e.g., `terms` aggregation by `service`) and evaluate the trigger **per bucket**, so you can alert individually per-service ("any service whose error count exceeds threshold," rather than one global check) from a single monitor definition — conceptually similar to how a single PromQL alerting rule with a `by (job)` grouping can fire multiple distinct alert instances, one per label value.

**OpenSearch also supports anomaly detection** (a separate plugin, using an unsupervised ML algorithm — Random Cut Forest) that can feed into a monitor as an alternative to fixed static thresholds, useful for metrics/log-derived signals with strong daily/weekly seasonality where a fixed "> 100" threshold would either be too sensitive off-peak or too lax at peak.

## Points to Remember

- CCR replicates indices at the shard level from a leader to a follower cluster continuously, by replaying leader translog operations — used for disaster recovery, geo-distributed read scaling, and data-residency compliance.
- CCR and snapshots are complementary, not redundant: CCR is a warm, queryable, near-real-time standby; snapshots are cold, point-in-time, and immune to *logical* corruption that CCR would faithfully replicate.
- OpenSearch alerting monitors = scheduled query + trigger condition + notification action, structurally analogous to a Prometheus alerting rule but over indexed documents via the OpenSearch query DSL instead of PromQL.
- Bucket-level monitors let one monitor definition alert per-group (e.g., per service) from an aggregation query, rather than requiring one monitor per entity.
- Anomaly detection (Random Cut Forest) is available as an alternative to static thresholds for signals with strong seasonality, where a fixed number would be wrong at different times of day/week.

## Common Mistakes

- Relying on CCR alone as a "backup" without also taking periodic snapshots — a bad mapping change, accidental bulk delete, or other logical corruption on the leader replicates faithfully to the follower, so CCR does not protect against that class of failure the way an older point-in-time snapshot does.
- Not accounting for replication lag under high write throughput or network issues between leader and follower regions — CCR is near-real-time, not synchronous; a failover during a lag spike can lose the most recent unreplicated writes.
- Building one query-level monitor per service/team instead of a single bucket-level monitor with a `terms` aggregation — creates unnecessary monitor sprawl and makes threshold changes require updating many separate monitors instead of one.
- Setting alert thresholds as flat static numbers for signals with obvious daily/weekly seasonality (e.g., login volume, request rate) — causing constant off-peak false alarms or missed peak-time real incidents; anomaly detection or time-aware thresholds are the actual fix, not just tuning the number up or down.
- Forgetting that alerting monitors consume cluster query resources on every scheduled run — a large number of frequent, expensive aggregation-based monitors can itself become a meaningful load on the cluster, worth factoring into the same shard/capacity planning covered in file 2.
