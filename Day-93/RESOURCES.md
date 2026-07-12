# Day 93 — Resources: OpenSearch Deep Dive

## Primary (assigned)

- **OpenSearch documentation** (opensearch.org/docs/latest) — the assigned free starting point; go directly to "Index Management" (ISM) and "Availability and recovery" (shard allocation, cross-cluster replication) sections.

## Deepen your understanding

- **"Sizing Amazon OpenSearch Service domains" — AWS documentation** (docs.aws.amazon.com/opensearch-service/latest/developerguide/sizing-domains.html) — despite being AWS-specific branding, this is one of the clearest practical write-ups on shard sizing, count, and node sizing tradeoffs available anywhere, and applies to self-managed OpenSearch too.
- **Elastic's "Size your shards" guide** (elastic.co/guide — search "size your shards") — Elasticsearch-branded but directly applicable, since OpenSearch inherited the same underlying Lucene-based shard model; excellent depth on the shard-size sweet spot and why.
- **OpenSearch "Index State Management" documentation** (opensearch.org/docs/latest/im-plugin/ism/index) — the authoritative reference for policy JSON structure, states/actions/transitions, and the full list of supported actions beyond what's covered in this day's notes.
- **OpenSearch "Cross-cluster replication" documentation** (opensearch.org/docs/latest/replication-plugin/index) — full setup and failover/promotion procedure reference.

## Reference / lookup

- **OpenSearch REST API reference** (opensearch.org/docs/latest/api-reference) — the `_cat`, `_cluster`, `_plugins/_ism`, and `_plugins/_alerting` endpoint references used throughout this day's lab and cheatsheet.
- **OpenSearch "Disk-based shard allocation" docs** — the authoritative reference for watermark default values and how to tune them.

## Practice

- **OpenSearch official Docker Compose quickstart** (opensearch.org/docs/latest/install-and-configure/install-opensearch/docker) — the fastest path to a local multi-node cluster for this lab, with OpenSearch Dashboards included.
- **killercoda.com / Elastic's own free sandbox environments** — browser-based scenarios for practicing ISM/ILM policies and shard diagnostics without provisioning your own cluster.
