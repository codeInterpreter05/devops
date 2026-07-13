# Day 110 — Quiz: Istio Deep Dive

Try to answer without looking at your notes. Answers are at the bottom.

1. What three legacy components merged into istiod, and what did each of them do?
2. Why was Mixer removed from the Istio architecture entirely (rather than merged)?
3. How does traffic actually get redirected into the Envoy sidecar without the application knowing?
4. What's the difference between automatic and manual sidecar injection, and when would you choose manual?
5. What does a `Gateway` resource configure, and what does it deliberately *not* configure?
6. In a `VirtualService`, if `timeout: 5s` and a route has `retries: { attempts: 3, perTryTimeout: 3s }`, what actually happens on a slow backend?
7. Where does "circuit breaking" actually live in Istio's CRDs, and what are its two components?
8. Why is Istio's outlier detection decentralized per-Envoy instead of a single mesh-wide breaker?
9. What is PERMISSIVE mTLS mode for, and why is it the recommended starting point rather than STRICT?
10. What identity format does Istio use for workload certificates, and why is it tied to the ServiceAccount rather than the pod IP?
11. Why can trace fragments appear in Jaeger even though Envoy automatically propagates trace headers between hops?
12. **Interview question:** How does Istio implement mTLS between services? What certificates does it use and how are they rotated?

---

## Answers

1. **Pilot** (translated `VirtualService`/`DestinationRule` config into Envoy xDS and pushed it to sidecars), **Citadel** (the CA — issued and rotated workload certificates), and **Galley** (config ingestion/validation). All three merged into the single istiod deployment in Istio 1.5.
2. Mixer performed policy/telemetry checks out-of-process, meaning every request incurred an extra network round trip to Mixer — this added meaningful latency at scale, so its functionality was moved in-process into Envoy (via WASM/native extensions) and Mixer was deprecated entirely rather than folded into istiod.
3. An init container (`istio-init`, or a CNI plugin in some setups) programs `iptables` `PREROUTING`/`OUTPUT` rules so inbound traffic is redirected to Envoy's inbound listener (port 15006) and outbound traffic is redirected to Envoy's outbound listener (15001) before it leaves the pod — all below the socket layer, so the application code is unaware.
4. Automatic injection uses a namespace label plus a Kubernetes mutating webhook that fires on pod creation. Manual injection (`istioctl kube-inject`) renders the same result client-side into plain YAML. Choose manual when you want GitOps-friendly, fully rendered manifests without relying on a live webhook at apply time, or when you want to audit exactly what gets injected before it reaches the cluster.
5. A `Gateway` configures which ports/protocols/hosts are exposed at the mesh edge (ingress or egress). It does not configure routing rules — that's the `VirtualService`'s responsibility.
6. The outer `timeout: 5s` wins. With `perTryTimeout: 3s` and `attempts: 3`, only about one full retry attempt (plus a partial second one) fits inside the 5-second window before the whole request is aborted — the retry budget is effectively truncated by the outer timeout.
7. Inside `DestinationRule.trafficPolicy`. Its two components are `connectionPool` (limits concurrent connections/requests to a destination) and `outlierDetection` (passively ejects endpoints returning consecutive errors).
8. A centralized breaker would be a single point of failure and would add a network hop to every routing decision. Per-Envoy decentralized tracking lets each sidecar react immediately to what it observes, at the cost of slightly different ejection timing across different clients calling the same backend — an acceptable tradeoff since the goal (stop overloading a failing endpoint) is still achieved.
9. PERMISSIVE lets a sidecar accept both plaintext and mTLS on the same port, so services without a sidecar yet (not yet migrated into the mesh) keep working while meshed services automatically start using mTLS between each other. It's the safe way to roll out mesh-wide security without breaking not-yet-injected callers; STRICT is the target end state once migration is verified complete.
10. SPIFFE format (`spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`), encoded in the certificate's SAN. It's tied to the ServiceAccount because pod IPs are ephemeral and get reused across completely different workloads, while an identity should represent "what this workload is," not "where it currently happens to run."
11. Envoy automatically forwards trace headers hop-to-hop at the proxy layer, but if a service's own application code makes a *new* outbound call without copying the incoming trace headers (`x-request-id`, `traceparent`/`b3`) onto that new call, the proxy has nothing to propagate — the trace breaks into disconnected fragments at that point, which is an application-level responsibility, not something Envoy can fix on its own.
12. Strong answer: "Istio gives every workload a SPIFFE-based cryptographic identity tied to its Kubernetes ServiceAccount, encoded in the SAN of a short-lived X.509 certificate (24h by default). The `istio-agent` in the sidecar generates a key and CSR, sends it to istiod's built-in CA over the same gRPC channel used for xDS, and istiod validates it against the pod's ServiceAccount token before signing. The signed cert is delivered to Envoy via the SDS API, never touching disk. Because the certs are short-lived, `istio-agent` automatically requests a new one before expiry — rotation is continuous and silent, with no manual renewal step and no per-workload Kubernetes Secret to manage. Enforcement is controlled by `PeerAuthentication`, typically rolled out as PERMISSIVE first, then tightened to STRICT once every workload is confirmed to be in the mesh."
