# Day 119 — Zero Trust & mTLS: SPIFFE/SPIRE Workload Identity

**Phase:** 4 – Advanced/Specialization | **Week:** W21 | **Domain:** Advanced Security | **Flag:** ⚡ Interview-critical

## Brief

Traditional service-to-service auth relies on shared secrets: a static API key baked into a ConfigMap, a long-lived TLS cert copied to every replica, a database password sitting in a Secret unrotated for years. These are exactly the credentials that leak in logs, get committed to git, or outlive the employee/service that created them. **SPIFFE** (Secure Production Identity Framework For Everyone, a CNCF-graduated specification) defines a standard way to give every workload a cryptographically verifiable, dynamically-issued, short-lived identity — no shared secret ever has to exist. **SPIRE** is the reference implementation. This is the mechanism that makes "mTLS everywhere without static certs to manage" actually operationally possible, and it's exactly what large-scale zero-trust deployments (Google, Netflix, Bloomberg, Uber) use under the hood. It's the concrete, implementable answer when an interviewer pushes past "use mTLS" and asks "okay, but how do workloads actually *get* their certificates without you manually provisioning and rotating them?"

## SPIFFE IDs and SVIDs

A **SPIFFE ID** is a URI: `spiffe://<trust-domain>/<path>`, e.g. `spiffe://example.org/ns/production/sa/payment-service`.

- **Trust domain** — the administrative security boundary that issues and vouches for identities (typically one per organization, environment, or cluster). Everything under a trust domain shares one root of trust.
- **Path** — workload-specific, structured however your registration scheme decides (SPIRE's Kubernetes integration commonly encodes namespace and ServiceAccount, but the path is just a string you define via registration entries — nothing forces this exact shape).

The **SVID (SPIFFE Verifiable Identity Document)** is the credential that proves a workload holds a given SPIFFE ID. Two forms:

- **X.509-SVID** — an X.509 certificate with the SPIFFE ID embedded in the URI SAN (Subject Alternative Name) field. Used directly for mTLS — no separate "identity token" layered on top of the TLS handshake, the identity *is* the certificate.
- **JWT-SVID** — a JWT with the SPIFFE ID as the `sub` claim. Used where terminating mTLS isn't practical (L7 proxies that need to forward identity, some serverless contexts, or systems that just want a bearer credential for a single call rather than a continuous TLS session).

Both are **short-lived by design** (SPIRE's default is one hour) and rotated automatically and continuously by the SPIRE Agent — the workload never has to restart or manually refresh anything. This single property is what actually eliminates the classic "static cert nobody remembered to rotate before it expired at 2am" failure mode, and shrinks the usable window of a stolen credential to roughly an hour instead of a year.

## SPIRE architecture: Server + Agent, and the attestation chain

- **SPIRE Server** — the central authority for a trust domain. Holds (or is backed by) the CA. Stores **registration entries**: policy statements mapping a set of workload *selectors* to a SPIFFE ID. Signs SVIDs when an Agent proves a workload matches an entry.
- **SPIRE Agent** — runs on every node (a DaemonSet in Kubernetes). Talks to the Server to attest the node itself, then serves the local **Workload API** over a Unix domain socket that workloads on that node call to ask "who am I, and prove it."

The reason this is actually trustworthy — not just "ask the agent nicely" — is the two-stage attestation chain:

1. **Node attestation.** The Agent proves *which node it's running on* to the Server using a platform-specific cryptographic fact it can't fabricate — e.g., a Kubernetes **Projected Service Account Token (`k8s_psat`)** bound to that exact node, or a cloud instance identity document (AWS IID, GCP metadata signature). This yields the Agent its own SVID and establishes it as a legitimate attestor for that node.
2. **Workload attestation.** When a process on that node calls the local Workload API socket asking for its identity, the Agent does **not** trust anything the calling process claims about itself. It walks from the caller's PID, through the kernel, to the cgroup and container ID, and queries the container runtime/kubelet for that container's actual metadata — namespace, ServiceAccount, pod labels, image. These are OS/kernel-level facts the workload cannot lie about (it would need to already have host-level compromise to forge them, at which point far bigger problems exist).
3. **Selector matching.** The Agent compares the attested facts (`k8s:ns:production`, `k8s:sa:payment-service`, etc.) against the Server's registration entries. A match authorizes issuance of that specific SPIFFE ID's SVID — nothing else.
4. **Issuance.** The Agent requests the Server sign the SVID for the matched identity and hands it back over the local socket. No network hop the workload could intercept from elsewhere, no shared secret transmitted.

A concrete registration entry:

```bash
spire-server entry create \
  -parentID spiffe://example.org/spire/agent/k8s_psat/demo-cluster/<agent-node-id> \
  -spiffeID  spiffe://example.org/ns/production/sa/payment-service \
  -selector  k8s:ns:production \
  -selector  k8s:sa:payment-service
```

**Why this beats "just use the Kubernetes ServiceAccount token":** a K8s SA token (even a modern audience-scoped projected token) is fundamentally an artifact of one cluster's API server — it's not designed to be verified by an arbitrary workload in another cluster, another cloud, or a bare VM without that workload also talking to your K8s API server (or you building your own token-exchange layer). A SPIFFE X.509-SVID is a self-contained, portable, standard TLS certificate any TLS-capable peer can verify against a trust bundle — it's the identity layer that makes mTLS work *between* clusters and clouds, not just within one.

## Workload identity federation

Large organizations don't run one flat trust domain — they run multiple clusters, multiple clouds, sometimes on-prem plus cloud, and merging them into a single trust domain would mean a single compromised CA blast-radius covering everything. **SPIFFE Federation** solves this by letting independent trust domains exchange **trust bundles** (the set of root/intermediate CA certificates each domain uses to validate its own SVIDs) via a federation endpoint (SPIRE Server's `/bundle` endpoint, or the SPIFFE Federation API). Once Trust Domain A has Trust Domain B's bundle (and vice versa), a workload in A can validate an SVID presented by a workload in B during an mTLS handshake — without the two domains sharing a CA, a network, or an admin team.

**The real-world pattern you'll actually touch day-to-day is the same shape, even where it isn't SPIFFE-branded:** AWS **IRSA** (IAM Roles for Service Accounts) and GCP **Workload Identity Federation** both let a Kubernetes pod assume a cloud IAM identity with zero static cloud credentials stored anywhere. Mechanically: the cluster's OIDC issuer endpoint is registered as a trusted external identity provider inside the cloud account's IAM configuration; a pod's projected ServiceAccount token (an OIDC-compliant JWT, signed by the cluster) is exchanged — via `sts:AssumeRoleWithWebIdentity` on AWS, or the equivalent STS call on GCP — for short-lived, automatically-expiring cloud credentials. That's federation: two independent trust domains (the K8s cluster and the cloud IAM system) agreeing to verify each other's tokens instead of one sharing a static secret with the other. This is precisely why leaked long-lived AWS access keys sitting in a pod's environment variables remain one of the most common real-world breach vectors — IRSA/Workload Identity Federation exists specifically to make that class of credential unnecessary.

## Points to Remember

- SPIFFE ID = `spiffe://trust-domain/path`, a stable identifier; SVID = the cryptographic proof of that ID, either an X.509 cert (for mTLS) or a JWT (for bearer-style use where TLS termination isn't practical).
- SVIDs are short-lived (SPIRE default ~1 hour) and auto-rotated by the Agent with zero app-side restart required — this is what makes "no static secrets" operationally real, not just a slogan.
- Workload attestation trusts kernel/OS-level facts about the calling process (PID → cgroup → container → runtime metadata), never anything the workload itself asserts — that's the actual security property, not "SPIRE trusts the agent."
- Node attestation and workload attestation are two distinct steps — a compromised Agent identity alone isn't enough without also satisfying workload selectors for a specific registration entry.
- Federation lets separate trust domains (clusters, clouds, on-prem) verify each other's identities by exchanging trust bundles, without merging into one blast-radius domain — IRSA and GCP Workload Identity Federation are the same conceptual pattern applied to cloud IAM.

## Common Mistakes

- Treating a SPIFFE ID like a secret that needs to be hidden — it's a public identifier (like a hostname), not a credential; the SVID (the signed document) is what carries the actual proof, and even that is meant to be presented, not concealed like a password.
- Registering overly broad selectors (e.g., matching on namespace alone, `k8s:ns:production`, with no ServiceAccount or label selector) so every workload in a namespace gets the same powerful identity — defeats the least-privilege point of having per-workload identity at all.
- Forgetting that SVID rotation requires the *application* to actually reload the cert from the Workload API stream rather than reading it once at startup and caching it forever — a workload that ignores rotation events will silently start failing mTLS handshakes once its cached SVID expires.
- Assuming a K8s ServiceAccount token alone is a drop-in replacement for a SPIFFE SVID for cross-cluster/cross-cloud mTLS — it works within one cluster's API server trust boundary but wasn't designed as a portable, arbitrary-peer-verifiable TLS identity.
- Merging multiple environments (e.g., staging and production) into one trust domain "to save setup time" — collapses the blast radius boundary that federation exists to preserve, so a staging compromise can present credentials production workloads will trust.
