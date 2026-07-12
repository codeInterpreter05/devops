# Day 46 — K8s Operators & CRDs: Real-World Operators — CloudNativePG & Strimzi

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** Kubernetes | **Flag:** –

## Brief

Concepts land better against concrete examples. CloudNativePG (Postgres) and Strimzi (Kafka) are two of the most mature, widely-deployed Operators in the ecosystem, and studying how each encodes its domain's hardest operational problems — failover for a relational database, partition/replica management for a distributed log — makes the abstract "Operator pattern" concrete. This is also directly the software used in today's hands-on lab.

## CloudNativePG — Postgres as a Kubernetes-native Operator

CloudNativePG (CNPG) is a CNCF Postgres Operator built specifically *for* Kubernetes (not a Kubernetes-wrapped version of an existing Postgres HA tool like Patroni) — meaning its failover/replication logic is designed around Kubernetes primitives from the ground up rather than adapted from a VM-era tool.

**Core custom resource — `Cluster`:**
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: {name: pg-cluster}
spec:
  instances: 3
  storage:
    size: 10Gi
    storageClass: gp3
  bootstrap:
    initdb: {database: app, owner: app}
  backup:
    barmanObjectStore:
      destinationPath: s3://my-backups/pg-cluster
      s3Credentials:
        accessKeyId: {name: s3-creds, key: ACCESS_KEY_ID}
        secretAccessKey: {name: s3-creds, key: SECRET_ACCESS_KEY}
```

**What the Operator actually does with this, beyond what a StatefulSet alone could:**
- **Provisions a primary + N-1 streaming replicas**, each as its own Pod (not a single StatefulSet doing generic replica management) — because Postgres replica roles (primary vs. standby) are fundamentally asymmetric, and CNPG models that directly rather than pretending all replicas are identical.
- **Automatic failover**: continuously monitors primary health; on primary failure, promotes the most up-to-date standby to primary, reconfigures the remaining standbys to replicate from the new primary, and updates the Kubernetes `Service` selector so application traffic transparently follows the new primary — no manual DNS/connection-string change needed, no human paged for a routine failover.
- **Continuous physical backup + PITR**: integrates with Barman (a mature, standalone Postgres backup tool) to continuously ship WAL (write-ahead log) segments to object storage (S3-compatible), enabling point-in-time recovery — directly analogous to the RDS PITR mechanism from Day 44, but self-managed on your own Kubernetes cluster instead of an AWS-managed service.
- **Rolling minor version upgrades** with automatic re-election of the primary, done pod-by-pod to avoid downtime.
- **Connection pooling** via an optional integrated PgBouncer `Pooler` CRD, and read/write traffic splitting via separate `-rw`/`-ro`/`-r` Services CNPG creates automatically (write traffic always to primary, read traffic optionally load-balanced across replicas).

**Why this matters relative to "just run Postgres in a StatefulSet"**: a hand-rolled StatefulSet has no concept of "which pod is currently primary," would need an external tool (or a human) to handle failover promotion, and has no built-in backup/PITR orchestration — CNPG is precisely the codified operational expertise that turns "I have 3 Postgres pods" into "I have a self-healing HA Postgres cluster."

## Strimzi — Kafka as a set of composable CRDs

Strimzi provides a family of CRDs, each modeling a distinct piece of a Kafka deployment, reconciled by a shared Cluster Operator (plus optional additional operators for finer control):

- **`Kafka`** — the broker cluster itself: broker count, storage (per-broker persistent volumes), listener configuration (plain/TLS/OAuth, internal/external), resource requests/limits.
- **`KafkaTopic`** — a single topic, declaratively: partitions, replication factor, retention/cleanup config — meaning topic creation/config becomes a GitOps-able YAML file instead of an imperative `kafka-topics.sh --create` command run by hand.
- **`KafkaUser`** — declarative ACL and authentication credential management per user/application, integrated with Kafka's SASL/TLS auth mechanisms.
- **`KafkaConnect`** / **`KafkaConnector`** — manages Kafka Connect worker deployments and individual connector configurations (e.g., a Debezium CDC connector) declaratively.
- **`KafkaMirrorMaker2`** — cross-cluster topic replication (e.g., for cross-region Kafka DR, directly analogous in spirit to Day 44's S3 CRR but for a Kafka log).

**Operational logic Strimzi encodes that you'd otherwise script by hand:**
- **Rolling restarts that respect Kafka's replication guarantees** — when broker configuration changes or a version upgrade happens, Strimzi restarts brokers one at a time, waiting for the partition replicas on the restarting broker to catch up and the cluster to report healthy (in-sync replica set fully reconciled) before proceeding to the next broker — get this wrong by hand and you risk under-replicated partitions or data loss during a rolling upgrade.
- **Cruise Control integration** — an optional component Strimzi can wire in for automated partition rebalancing across brokers (e.g., after adding new broker capacity, or to correct uneven disk usage) — a genuinely hard rebalancing problem that would otherwise require deep Kafka expertise to do safely by hand.
- **Certificate lifecycle** — Strimzi runs its own internal CA and automatically issues/rotates TLS certificates for inter-broker and client-broker communication, removing a whole category of "cert expired and nobody noticed" operational risk.

**Why the multi-CRD design matters**: splitting `Kafka`, `KafkaTopic`, and `KafkaUser` into separate CRDs (rather than one giant `Kafka` object with topics/users nested inside) lets different teams own different concerns with appropriately scoped RBAC — an application team can be granted permission to create `KafkaTopic`/`KafkaUser` objects in their namespace without ever having permission to touch the underlying `Kafka` broker cluster resource itself, mirroring a real organizational separation of platform-team vs. application-team responsibilities.

## Points to Remember

- CNPG's `Cluster` CRD encodes Postgres-specific failover (primary promotion, service selector update) and backup/PITR (via Barman + object storage) — operational knowledge a generic StatefulSet has no concept of.
- CNPG automatically maintains separate read-write and read-only Services, so applications don't need custom logic to find "whichever pod is currently primary."
- Strimzi splits Kafka into multiple CRDs (`Kafka`, `KafkaTopic`, `KafkaUser`, `KafkaConnect`) specifically so RBAC can scope platform-level vs. application-level concerns separately.
- Strimzi's rolling restarts are Kafka-replication-aware — they wait for in-sync replicas to catch up broker-by-broker, avoiding the data-loss/under-replication risk of a naive rolling restart.
- Both Operators demonstrate the same underlying principle from file 01: the CRD is just the desired-state schema; the actual hard, domain-specific operational logic lives in the controller reconciling it.

## Common Mistakes

- Treating `KafkaTopic`/`KafkaUser` objects as purely descriptive/inert config, forgetting they're live-reconciled — deleting a `KafkaTopic` custom resource can actually delete the underlying Kafka topic (depending on Strimzi's topic-deletion configuration), not just remove a Kubernetes record.
- Manually editing Postgres or Kafka configuration directly on a pod (e.g., `kubectl exec`-ing in and changing `postgresql.conf` by hand) instead of through the CRD's spec — the next reconcile can silently revert the manual change, since the Operator always drives actual state back toward the declared spec.
- Skipping the backup/object-storage configuration on a CNPG `Cluster` because "it's just a test," then not having a real PITR/backup path if that "test" cluster becomes load-bearing.
- Under-provisioning storage/replica count on either Operator's CRD to save cost, not accounting for the fact that in-place resize or replica-count changes for a live database/broker cluster are genuinely riskier operations than getting the initial sizing right.
- Assuming Strimzi/CNPG remove all need for Kafka/Postgres expertise — the Operator automates the mechanical *execution* of well-understood operational playbooks; you still need to understand what a healthy cluster looks like to configure it sensibly and diagnose real incidents.
