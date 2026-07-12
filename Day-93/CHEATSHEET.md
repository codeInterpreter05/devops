# Day 93 — Cheatsheet: OpenSearch Deep Dive

## Cluster health & diagnostics

```bash
GET _cluster/health?pretty                       # overall status: green/yellow/red
GET _cat/nodes?v                                   # node list, roles, heap/disk %
GET _cat/indices?v&s=store.size:desc               # indices sorted by size
GET _cat/shards?v                                  # every shard, node, size, state
GET _cluster/allocation/explain?pretty             # WHY a shard is/isn't allocated
GET _cluster/settings?include_defaults=true&filter_path=**.disk.watermark*   # watermark config
```

## Disk-based allocation watermarks (defaults)

```
85%  low watermark        -> stop allocating NEW shards to this node
90%  high watermark        -> actively relocate shards AWAY from this node
95%  flood-stage watermark  -> affected indices become READ-ONLY
```

## ISM (Index State Management) policy skeleton

```json
PUT _plugins/_ism/policies/logs_policy
{
  "policy": {
    "default_state": "hot",
    "states": [
      {"name": "hot",
       "actions": [{"rollover": {"min_size": "30gb", "min_index_age": "1d"}}],
       "transitions": [{"state_name": "warm", "conditions": {"min_index_age": "7d"}}]},
      {"name": "warm",
       "actions": [{"replica_count": {"number_of_replicas": 1}}, {"force_merge": {"max_num_segments": 1}}],
       "transitions": [{"state_name": "cold", "conditions": {"min_index_age": "30d"}}]},
      {"name": "cold",
       "actions": [{"read_only": {}}],
       "transitions": [{"state_name": "delete", "conditions": {"min_index_age": "90d"}}]},
      {"name": "delete", "actions": [{"delete": {}}]}
    ]
  }
}
```

```bash
GET _plugins/_ism/explain/<index-pattern>?pretty   # current state of each index under a policy
POST _plugins/_ism/add/<index>                     # attach a policy to an existing index
```

## Rollover alias pattern

```json
PUT logs-000001
{
  "aliases": { "logs-write": { "is_write_index": true } }
}
```
Applications write to `logs-write`; rollover cuts over the alias to a new backing index automatically.

## Shard sizing target

```
10-50 GB per shard   <- practical sweet spot
too many small shards -> cluster-state overhead, sluggish master/cluster-manager
too few huge shards   -> slow search + slow recovery per shard
```

## Allocation awareness / filtering

```json
PUT _cluster/settings
{ "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone"
} }
```
```json
PUT my-index/_settings
{ "index.routing.allocation.require.box_type": "hot" }
```

## Cross-cluster replication (CCR)

```bash
PUT _plugins/_replication/<follower-index>/_start
{
  "leader_alias": "leader-cluster",
  "leader_index": "logs-2026.07.12"
}
GET _plugins/_replication/<follower-index>/_status
POST _plugins/_replication/<follower-index>/_stop
```

## Alerting monitor skeleton

```json
POST _plugins/_alerting/monitors
{
  "type": "monitor",
  "monitor_type": "query_level_monitor",
  "schedule": {"period": {"interval": 5, "unit": "MINUTES"}},
  "inputs": [{"search": {"indices": ["logs-*"], "query": { "...": "..." }}}],
  "triggers": [{
    "name": "trigger-name",
    "condition": {"script": {"source": "ctx.results[0].hits.total.value > 100"}},
    "actions": [{"name": "notify", "destination_id": "<id>", "message_template": {"source": "..."}}]
  }]
}
```

## OpenSearch terminology vs Elasticsearch

```
OpenSearch          Elasticsearch
ISM                  ILM
cluster manager node  master node
_plugins/_ism         _ilm
_plugins/_alerting     Watcher / Kibana Alerting
```
