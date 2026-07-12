# Day 79 — Runtime Security III: Kubernetes Audit Logs & Incident Response

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** DevSecOps | **Flag:** —

## Brief

Falco tells you *that* something suspicious happened inside a container. The Kubernetes API server's **audit log** tells you *who authenticated and issued the exact API request* that caused it — the two together are the complete answer to today's interview question. This file also covers what to actually do next: a real incident response playbook for Kubernetes, because detecting a compromise is only useful if you have a practiced, correct response, not a panicked one.

## How Kubernetes audit logging works

Every request to the Kubernetes API server — `kubectl exec`, `kubectl get secrets`, a controller reconciling a resource, anything — can be recorded by the API server's built-in **audit** subsystem. This is configured via an **audit policy file** that defines, per rule, which requests to log and at what detail level:

- **Stages** — a single request can generate multiple audit events across its lifecycle: `RequestReceived`, `ResponseStarted`, `ResponseComplete`, and `Panic`. Most policies only care about `ResponseComplete` (the request finished, log the outcome) to avoid doubling log volume.
- **Levels** — how much detail to capture per matched rule:
  - `None` — don't log at all (for known-noisy, low-value requests).
  - `Metadata` — log the request metadata (user, timestamp, resource, verb) but not the request/response bodies.
  - `Request` — metadata plus the request body.
  - `RequestResponse` — metadata plus both request and response bodies (most detail, most storage cost).

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Full detail on secrets access — high-value, low-volume
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]

  # Full detail on exec/attach — this is the exact rule that answers today's interview question
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach"]

  # Don't bother logging high-volume, low-value read-only system traffic
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: ""
        resources: ["endpoints", "services"]

  # Metadata-only for everything else, as a catch-all
  - level: Metadata
    omitStages: ["RequestReceived"]
```

The API server is started with flags pointing at this policy and where to write results:

```
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=10
--audit-log-maxsize=100
```

For shipping audit events to an external SIEM instead of (or in addition to) a local file, `--audit-webhook-config-file` points at a webhook backend that receives each event over HTTP — the standard pattern for centralizing audit data across a fleet of clusters rather than trawling per-node log files during an incident.

## The exact answer to "how would you detect an exec into prod"

A `kubectl exec` request shows up in the audit log as a request against the `pods/exec` **subresource**, with a `connect` verb. A matching audit event looks like:

```json
{
  "kind": "Event",
  "level": "RequestResponse",
  "auditID": "5c73d...",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/production/pods/checkout-7d9f/exec?command=%2Fbin%2Fbash&stdin=true&stdout=true&tty=true",
  "verb": "connect",
  "user": {
    "username": "alice@company.com",
    "groups": ["system:authenticated"]
  },
  "sourceIPs": ["203.0.113.42"],
  "objectRef": {
    "resource": "pods",
    "subresource": "exec",
    "namespace": "production",
    "name": "checkout-7d9f"
  },
  "requestReceivedTimestamp": "2026-07-12T14:02:11Z"
}
```

This is the **authoritative** record: it names the exact authenticated user, their source IP, the exact pod, and the exact timestamp — independent of anything happening inside the container. Falco's `Terminal shell in container` alert is the **corroborating, real-time behavioral signal** from inside the container itself. A complete, credible interview answer names both: *"I'd correlate a Falco alert on an interactive shell spawning inside the container (`proc.tty != 0`) with the Kubernetes audit log's `pods/exec` `connect` event for the same pod and timestamp, which gives me the authenticated identity and source IP that Falco alone can't provide."*

## Incident response playbook: Falco fires on a production container

1. **Contain, without destroying evidence.** The instinct to immediately `kubectl delete pod` is usually wrong — it destroys the compromised container's filesystem and process state before you've captured anything. Prefer: isolate the pod's network access via a deny-all `NetworkPolicy` scoped to that pod's labels, or cordon the node (`kubectl cordon <node>`) to stop new scheduling there, while leaving the pod itself running and observable.
2. **Collect evidence while it still exists.** Once the pod is deleted, an interactively-created shell's history and any in-memory attacker activity are gone. Capture: `kubectl logs`, `kubectl describe pod` (for events/status), a filesystem snapshot (`kubectl cp` out of the container, or `crictl exec`/`crictl inspect` at the node level for lower-level forensics), and the exact matching audit log entries for that pod/namespace/timeframe.
3. **Correlate the timeline.** Line up the Falco alert timestamp against the audit log's `pods/exec` event and against any surrounding events (a suspicious `Secret` read, an unusual outbound connection alert) to reconstruct what actually happened and in what order — this reconstructed timeline is also exactly what a postmortem write-up needs.
4. **Eradicate.** Rotate every credential the pod had access to — its ServiceAccount token, any mounted Secrets, any cloud IAM role it could assume — treating all of them as compromised regardless of whether you can prove they were actually used. Redeploy from a known-good, re-scanned image; patch whatever vulnerability or misconfiguration allowed the initial access in the first place (this is often where a CI/CD security gate from earlier in this phase — a missed CVE, an overly permissive RBAC role, a container that shouldn't have had `exec` access at all — turns out to be the actual root cause).
5. **Recover.** Restore traffic/scheduling gradually, watching monitoring and Falco for any recurrence, rather than flipping everything back on at once.
6. **Post-incident.** Write a blameless postmortem. Feed the specific gap that allowed this back into prevention: a new/tightened Falco rule, a tighter seccomp/AppArmor profile, an RBAC change removing unnecessary `exec` permission from a role, or a new CI gate — closing the loop from "detected once" to "structurally harder to happen again" is what separates a mature response from just cleaning up and moving on.

## Points to Remember

- Audit policy has two independent knobs: **stage** (which point in a request's lifecycle to log) and **level** (`None`/`Metadata`/`Request`/`RequestResponse` — how much detail to capture) — tune per-resource to balance forensic value against log volume/cost.
- `pods/exec` and `pods/attach` are subresources worth explicitly logging at `RequestResponse` level — they're exactly the audit trail for "who ran commands inside a running container."
- The complete, correct answer to "detect an exec into a container" is **Falco (real-time behavioral detection inside the container) correlated with the Kubernetes audit log (authoritative record of the authenticated API request)** — not either one alone.
- Never immediately delete/kill a suspicious pod as your first response — you destroy forensic evidence. Contain (network isolation, cordon) before you eradicate.
- Any credential a compromised pod had access to must be treated as compromised and rotated, regardless of whether you can prove it was actually exfiltrated or used.

## Common Mistakes

- Not enabling audit logging at all, or leaving it at the default (often minimal or off, depending on the managed Kubernetes provider) until after an incident already happened and there's no record to investigate.
- Setting every rule to `RequestResponse` "to be safe," generating enormous log volume dominated by low-value, high-frequency read/watch traffic — this both costs real money to store and index, and makes the genuinely important events (like a `pods/exec` call) harder to find in the noise.
- Reflexively deleting or restarting a compromised pod first, destroying the exact filesystem/process-state evidence needed to determine what actually happened and how the attacker got in.
- Investigating and remediating a single compromised pod without rotating the credentials it had access to — an attacker who captured a ServiceAccount token or mounted Secret can continue using it long after the pod itself is gone.
- Treating incident response as "detect, clean up, done" without a postmortem step that feeds a concrete prevention change (a Falco rule, an RBAC tightening, a seccomp profile) back into the system — without that loop, the same class of incident recurs.
