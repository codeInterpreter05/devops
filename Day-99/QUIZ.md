# Day 99 — Quiz: AWS-native Observability

Try to answer without looking at your notes. Answers are at the bottom.

1. What are the two ways to deploy Container Insights on EKS, and what does the modern add-on path give you that the manual DaemonSet path doesn't?
2. Why can enabling Container Insights on a high-pod-churn EKS cluster produce a surprisingly large CloudWatch bill?
3. Is CloudWatch Logs Insights an indexed search system like Elasticsearch? Why does this distinction matter for cost and query latency?
4. Write a Logs Insights query that computes the count of log events per minute where `level = "ERROR"`.
5. What is the `X-Amzn-Trace-Id` header for, and what happens to a distributed trace if one service in the call chain fails to propagate it?
6. What is X-Ray's default sampling rate, and why does sampling exist at all rather than tracing every request?
7. What problem does ADOT solve that the raw X-Ray SDK does not, and how does it solve it architecturally?
8. What's the functional difference between a CloudWatch Synthetics "heartbeat" canary and a "UI flow" canary — and why might a heartbeat check pass while the app is actually broken for users?
9. Is Amazon Managed Prometheus (AMP) a different query language than open-source Prometheus? What does "API-compatible" mean in this context?
10. Name two things Amazon Managed Grafana (AMG) takes off your plate compared to running Grafana yourself.
11. **Interview question:** Compare self-managed Prometheus + Grafana vs Amazon Managed Prometheus + Managed Grafana. When do you use each?
12. What's one operational risk of self-managing your monitoring stack (Prometheus + Grafana) that "your monitoring system going down" specifically refers to?

---

## Answers

1. Manually deploying the CloudWatch agent + Fluent Bit as DaemonSets, or installing the `amazon-cloudwatch-observability` EKS add-on. The add-on path also gives you "Container Insights with enhanced observability" — control-plane metrics (API server, etcd, scheduler) that the manual path doesn't collect by default, plus one command instead of managing manifests yourself.
2. Container Insights publishes metrics as CloudWatch custom metrics, and it creates a metric dimension per container/pod. A cluster with thousands of short-lived pods generates a large number of distinct metric series, and CloudWatch bills per metric-per-month — high pod churn directly drives cost up.
3. No — Logs Insights scans the raw log events in your selected time range at query time; it does not maintain a persistent index. This matters because cost and query latency scale with **bytes scanned in the time window**, not with how selective your `filter` clause is — narrowing the time range is the primary lever for both.
4. `fields @timestamp | filter level = "ERROR" | stats count() as errors by bin(1m)`
5. It carries the trace ID (and parent segment ID, sampling decision) across service boundaries so all segments from one logical request can be stitched into a single trace. If one service doesn't read and forward the header, the trace breaks into two (or more) disconnected fragments with no visible link between them in the Service Map.
6. Default: 1 request per second, plus 5% of any additional requests, per service. Sampling exists because tracing every request at scale would be expensive (X-Ray bills per trace recorded) and largely redundant — most identical/low-value requests (like health checks) don't need individual traces to prove the service is healthy.
7. The raw X-Ray SDK ties your instrumentation directly to X-Ray's proprietary trace format, creating vendor lock-in. ADOT packages the vendor-neutral OpenTelemetry SDKs/Collector — your app emits OTLP data to a local ADOT Collector, and the Collector's exporter configuration (not your application code) decides the backend (X-Ray, CloudWatch EMF, or a third party). Swapping backends later means changing an exporter config, not re-instrumenting every service.
8. A heartbeat canary just checks HTTP status/response time for a URL — it can return 200 while the page is fully broken due to a client-side JavaScript error, since it never renders or executes anything. A UI-flow canary actually drives a headless browser (Puppeteer/Selenium) through real interactions (click login, submit form), so it catches rendering and JS-level failures a heartbeat check structurally cannot see.
9. No — AMP implements the same PromQL query language and is API-compatible with the open-source Prometheus `remote_write` protocol, meaning existing scrape configs, dashboards, and alert rules mostly carry over unchanged; you just point `remote_write` at an IAM SigV4-authenticated AMP endpoint instead of (or in addition to) local storage.
10. Any two of: running/patching/upgrading the Grafana server itself, managing Grafana's backing database, wiring up SSO/authentication (AMG integrates IAM Identity Center/SAML natively), and maintaining HA for the Grafana workspace.
11. Strong answer: self-managed Prometheus + Grafana gives full ecosystem control (Prometheus Operator CRDs, custom exporters, arbitrary retention via Thanos/Cortex/Mimir) and can be cheaper at high, steady-state scale, but you own HA, storage scaling, upgrades, and auth — real operational burden. AMP + AMG remove that operational burden (AWS handles HA/scaling/upgrades, PromQL and remote_write stay the same) but cost scales per-sample/per-user and ties you to an AWS-only backend. Use self-managed when you have platform engineers who already run this stack well and want maximum cost efficiency/control at scale; use managed when you want faster time-to-value, don't want to be on-call for your own monitoring infrastructure, or are already standardizing on AWS-native tooling. There is no universally correct answer — naming the actual trade-off dimensions is the strong signal, not picking a side dogmatically.
12. If your self-managed Prometheus/Grafana stack itself has an outage (e.g., a Prometheus replica crash-loops, storage fills up, or a bad config locks out Grafana), you lose visibility into your production systems at potentially the exact moment you need it most — and your team, not a vendor, is on the hook to fix your own monitoring tooling before you can even diagnose the original problem.
