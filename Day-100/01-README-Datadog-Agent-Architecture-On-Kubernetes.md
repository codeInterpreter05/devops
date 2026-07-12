# Day 100 — Commercial Observability I: Datadog Agent Architecture on Kubernetes

**Phase:** 3 – Observability | **Week:** W17 | **Domain:** Observability | **Flag:** ⚡ Interview-critical

## Brief

By this point you've built (or at least evaluated) an AWS-native and/or self-managed OSS observability stack. Today's topic is the commercial alternative most engineering orgs with budget actually run in production: Datadog (and, as a comparison point, New Relic). Understanding *how the agent architecture works* — not just "it's a SaaS dashboard" — is what separates someone who's clicked around a Datadog trial from someone who's actually operated it in a real cluster and can debug it when metrics stop flowing.

This day is split into three focused files:

1. **This file** — the Datadog Agent, Cluster Agent, and how they deploy on Kubernetes.
2. **[02-README-Datadog-APM-Dashboards-And-Monitors.md](02-README-Datadog-APM-Dashboards-And-Monitors.md)** — APM (service maps, flame graphs), dashboards, and monitors/alerting.
3. **[03-README-Datadog-Logs-Cost-Comparison-And-New-Relic.md](03-README-Datadog-Logs-Cost-Comparison-And-New-Relic.md)** — log management in Datadog, the Datadog-vs-Prometheus/Grafana cost/capability trade-off, and New Relic as an alternative.

## The Datadog Agent: what it actually is

The **Datadog Agent** is a lightweight process (Go binary) that runs on every host/node you want visibility into. It does three jobs:

1. **Collects host-level metrics** — CPU, memory, disk, network — every ~15 seconds by default, via built-in system checks.
2. **Runs integration checks** — Python/Go plugins (300+ built-in) that know how to scrape a specific technology (Postgres, Redis, Nginx, Kafka, etc.) and translate its native metrics into Datadog's format.
3. **Acts as a local aggregation/forwarding point** — for StatsD/DogStatsD custom metrics your application code emits directly, and for APM traces (it runs a local trace-agent component that batches and forwards spans, similar in spirit to the X-Ray daemon covered on Day 99).

## Deployment on Kubernetes: DaemonSet + Cluster Agent

On Kubernetes, the Datadog Agent is deployed as a **DaemonSet** — exactly one Agent pod per node, guaranteed by the DaemonSet controller, so every node gets visibility without you manually placing pods.

```yaml
# Simplified Datadog Agent DaemonSet excerpt (in practice installed via the datadog-operator or Helm chart)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: datadog-agent
spec:
  template:
    spec:
      containers:
        - name: agent
          image: gcr.io/datadoghq/agent:7
          env:
            - name: DD_API_KEY
              valueFrom:
                secretKeyRef: { name: datadog-secret, key: api-key }
            - name: DD_KUBELET_TLS_VERIFY
              value: "false"
          volumeMounts:
            - name: dockersocket
              mountPath: /var/run/docker.sock
            - name: procdir
              mountPath: /host/proc
              readOnly: true
            - name: cgroups
              mountPath: /host/sys/fs/cgroup
              readOnly: true
```

Note the volume mounts: the Agent needs read access to the host's `/proc` and cgroup filesystem to compute accurate per-container resource usage — this is the same mechanism cAdvisor/kubelet use, and it's why the Agent DaemonSet needs elevated host access (frequently flagged in security reviews; the mounts are read-only for exactly this reason).

The **Cluster Agent** is a separate, single-replica Deployment (not a DaemonSet) that offloads cluster-wide work the per-node Agent shouldn't duplicate: talking to the Kubernetes API server for cluster-level metadata (deployments, services, events), computing Horizontal Pod Autoscaler custom/external metrics from Datadog data, and acting as a single point of contact so 500 node-level Agents don't all hammer the API server independently. Node Agents talk to the Cluster Agent over an internal service, and the Cluster Agent talks to the Kubernetes API — a fan-in pattern that keeps API server load bounded regardless of cluster size.

## Autodiscovery

Rather than statically configuring "scrape Postgres on this specific pod IP," the Agent uses **Autodiscovery**: it watches the Kubernetes API for pod events and matches **pod annotations** against known integration templates, automatically starting/stopping the right check as pods come and go.

```yaml
# Pod annotation that tells the Agent to auto-configure a Postgres check
metadata:
  annotations:
    ad.datadoghq.com/postgres.check_names: '["postgres"]'
    ad.datadoghq.com/postgres.init_configs: '[{}]'
    ad.datadoghq.com/postgres.instances: '[{"host": "%%host%%", "port": "5432", "username": "datadog", "password": "%%env_DD_PG_PASSWORD%%"}]'
```

This is what makes Datadog on Kubernetes feel "zero-config" compared to hand-writing static Prometheus scrape configs for every workload — new pods that carry the right annotation are picked up automatically, with no Agent restart or config redeploy needed. The trade-off is that this convenience lives entirely inside Datadog's proprietary annotation format, which is part of the lock-in discussion in file 3.

## Points to Remember

- The Agent runs as a DaemonSet (one per node) and does host metrics, integration checks, and local trace/custom-metric forwarding; the Cluster Agent is a separate single-replica Deployment that handles cluster-wide API server interaction and HPA metric computation.
- The Agent needs read access to host `/proc` and cgroup filesystems to compute accurate per-container resource metrics — the same mechanism kubelet/cAdvisor use.
- The Cluster Agent exists specifically to prevent every node-level Agent from independently hammering the Kubernetes API server at scale — a fan-in pattern.
- Autodiscovery via pod annotations means new workloads get monitored automatically without manual Agent config changes, at the cost of coupling your manifests to Datadog's annotation schema.
- DogStatsD (the Agent's local StatsD-compatible listener) is the standard way application code emits custom metrics without needing to know anything about Datadog's HTTP API directly.

## Common Mistakes

- Deploying the Agent as a regular Deployment instead of a DaemonSet, then wondering why some nodes have no host-level metrics — it needs to run on every node, which only a DaemonSet guarantees.
- Under-provisioning the Cluster Agent (single replica, easy to forget resource requests/limits) and having it become a silent bottleneck or single point of failure for HPA custom metrics at cluster scale.
- Granting the Agent DaemonSet broader host mounts/permissions than needed "to be safe," when the actual requirement is narrowly read-only access to specific `/proc` and cgroup paths — a common finding in security reviews of Datadog deployments.
- Relying entirely on Autodiscovery annotations without documenting them anywhere else, so when someone removes "unused-looking" annotations during a manifest cleanup, monitoring silently stops for that workload with no error surfaced.
- Forgetting that DogStatsD by default aggregates over a flush interval (commonly 10s) — expecting per-request-level granularity from a metric emitted via DogStatsD and being confused when it doesn't show up that way.
