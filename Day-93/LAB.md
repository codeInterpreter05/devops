# Day 93 — Lab: OpenSearch Deep Dive

**Goal:** Implement a real hot → warm → cold → delete ILM/ISM policy against a live OpenSearch cluster, measure the performance impact of each tier transition, and practice the exact diagnostic workflow for a "cluster slow, disk at 85%" scenario.

**Prerequisites:** Docker (for a local multi-node OpenSearch cluster) or a Kubernetes cluster with the `opensearch-operator`/Helm chart. `curl` or a REST client. At least 3 vCPU / 4GB RAM free for a 3-node local cluster.

---

### Lab 1 — Stand up a local multi-node OpenSearch cluster

1. Using Docker Compose (fastest path — grab OpenSearch's official `docker-compose.yml` sample or write a minimal 2-node one):
   ```bash
   docker network create opensearch-net
   docker run -d --name os-node1 --network opensearch-net \
     -p 9200:9200 -e "discovery.type=single-node" \
     -e "OPENSEARCH_INITIAL_ADMIN_PASSWORD=YourStrongPass123!" \
     opensearchproject/opensearch:2.15.0
   ```
2. Confirm it's up:
   ```bash
   curl -k -u admin:YourStrongPass123! https://localhost:9200/_cluster/health?pretty
   ```
3. Deploy OpenSearch Dashboards pointed at it and confirm you can log in.

**Success criteria:** `_cluster/health` returns `status: green` (or `yellow` for a genuine single-node cluster, since replicas can't be allocated with only one node), and Dashboards loads in a browser.

---

### Lab 2 — Core hands-on activity: implement hot (7d) → warm (30d) → cold (90d) → delete

This is the assigned hands-on activity for today.

1. Create an ISM policy matching the pattern from this day's first README (adjust `min_index_age` values to minutes for lab purposes so transitions happen quickly instead of waiting real days):
   ```bash
   curl -k -u admin:YourStrongPass123! -X PUT "https://localhost:9200/_plugins/_ism/policies/lab_logs_policy" \
     -H 'Content-Type: application/json' -d '{
       "policy": {
         "default_state": "hot",
         "states": [
           {"name": "hot", "actions": [{"rollover": {"min_size": "1gb"}}],
            "transitions": [{"state_name": "warm", "conditions": {"min_index_age": "2m"}}]},
           {"name": "warm", "actions": [{"replica_count": {"number_of_replicas": 0}}, {"force_merge": {"max_num_segments": 1}}],
            "transitions": [{"state_name": "cold", "conditions": {"min_index_age": "5m"}}]},
           {"name": "cold", "actions": [{"read_only": {}}],
            "transitions": [{"state_name": "delete", "conditions": {"min_index_age": "10m"}}]},
           {"name": "delete", "actions": [{"delete": {}}]}
         ]
       }
     }'
   ```
2. Create an index with a rollover alias and attach the policy:
   ```bash
   curl -k -u admin:YourStrongPass123! -X PUT "https://localhost:9200/lab-logs-000001" \
     -H 'Content-Type: application/json' -d '{
       "aliases": {"lab-logs-write": {"is_write_index": true}},
       "settings": {"plugins.index_state_management.policy_id": "lab_logs_policy"}
     }'
   ```
3. Bulk-index a few thousand sample documents into the `lab-logs-write` alias (write a small loop or use `curl -X POST .../lab-logs-write/_doc` in a loop).
4. Watch the index transition states in real time:
   ```bash
   curl -k -u admin:YourStrongPass123! "https://localhost:9200/_plugins/_ism/explain/lab-logs-*?pretty"
   ```
5. Once it reaches `cold`, confirm it's genuinely read-only by attempting a write and observing the rejection.

**Success criteria:** You've watched a real index transition hot → warm → cold, observed the replica count and segment count change at each stage, and confirmed the cold-stage index rejects writes.

---

### Lab 3 — Shard sizing and allocation diagnostics

1. Check current shard distribution:
   ```bash
   curl -k -u admin:YourStrongPass123! "https://localhost:9200/_cat/shards?v"
   curl -k -u admin:YourStrongPass123! "https://localhost:9200/_cat/indices?v&s=store.size:desc"
   ```
2. Check disk watermark status:
   ```bash
   curl -k -u admin:YourStrongPass123! "https://localhost:9200/_cluster/settings?include_defaults=true&filter_path=**.disk.watermark*"
   ```
3. Run the allocation-explain diagnostic (the actual command you'd run for the "disk at 85%" interview scenario):
   ```bash
   curl -k -u admin:YourStrongPass123! -X GET "https://localhost:9200/_cluster/allocation/explain?pretty"
   ```
4. Simulate disk pressure (if using Docker, you can shrink the container's available disk via a small volume, or just note the commands you'd run in production and explain expected output at each watermark threshold).

**Success criteria:** You can run all three diagnostic commands from memory and explain what "healthy" output looks like for each, versus what you'd expect to see during a real disk-pressure incident.

---

### Lab 4 — Set up an alerting monitor

1. Create a query-level monitor that fires if more than 10 documents matching a specific field appear in the last 5 minutes (adjust the example from this day's third README to match your indexed test data's actual fields).
2. Trigger it by bulk-indexing enough matching documents to cross the threshold.
3. Confirm the monitor's trigger fired by checking `_plugins/_alerting/monitors/<id>` history or the configured notification destination.

**Success criteria:** You've seen a monitor transition from not-firing to firing based on real indexed data crossing your configured threshold.

---

### Cleanup

```bash
docker stop os-node1 && docker rm os-node1
docker network rm opensearch-net
```

### Stretch challenge

Configure a second local OpenSearch node/cluster and set up cross-cluster replication for one index from your lab cluster to it; then deliberately stop the "leader" and confirm the follower still serves (read-only) queries for previously-replicated data.
