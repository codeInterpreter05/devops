# Day 64 — ArgoCD Deep Dive: Health Checks & Notifications

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** GitOps | **Flag:** ⚡ Interview-critical

## Brief

"Synced" and "healthy" are two different, independent questions ArgoCD answers about every `Application`, and conflating them is a common source of false confidence ("the dashboard is all green, but the app is actually down"). Notifications close the loop by telling humans/systems about sync and health events without anyone needing to babysit the ArgoCD UI. Together these are what make ArgoCD a real operational tool rather than just a fancy diffing engine.

## Health checks — is what's deployed actually working?

**Sync status** (Synced / OutOfSync) only answers "does the live cluster state match Git?" It says nothing about whether the deployed thing is actually *working*. **Health status** answers that second question, and ArgoCD computes it differently per Kubernetes resource type using built-in, type-aware logic:

| Resource | "Healthy" means |
|---|---|
| `Deployment` | `status.replicas == status.updatedReplicas == status.availableReplicas` and matches the desired replica count — i.e., the rollout actually completed and pods are Ready, not just that the Deployment object was created |
| `Service` | Depends on type — a `LoadBalancer` Service isn't Healthy until it has an actual external IP/hostname assigned, not just "the object exists" |
| `Ingress` | Healthy once it has an assigned address |
| `Job` | Healthy once `status.succeeded >= 1` |
| `PVC` | Healthy once `status.phase == Bound` |
| Custom Resources (CRDs) | **Unknown by default** — ArgoCD has no built-in logic for arbitrary CRDs (e.g., a `Certificate` from cert-manager, or your own operator's CR) |

The overall `Application` health is the **worst** health status among all its constituent resources — one unhealthy Pod-backing resource makes the whole `Application` show unhealthy, even if everything else is fine, so investigating "why is this Application unhealthy" means finding the specific resource dragging the aggregate down, not assuming the whole thing is broken.

### Custom health checks for CRDs

Since ArgoCD can't know what "healthy" means for your own CRD out of the box, you define it yourself using a **Lua script** in the `argocd-cm` ConfigMap:

```yaml
# argocd-cm ConfigMap
data:
  resource.customizations.health.mycompany.io_MyCustomResource: |
    hs = {}
    if obj.status ~= nil and obj.status.phase == "Running" then
      hs.status = "Healthy"
      hs.message = obj.status.message
      return hs
    end
    hs.status = "Progressing"
    hs.message = "Waiting for MyCustomResource to become Running"
    return hs
```

This Lua snippet is given the resource object (`obj`, the CR's actual live manifest) and must return a table with `status` set to one of `Healthy`, `Progressing`, `Degraded`, `Suspended`, `Missing`, or `Unknown`. Writing these for any CRD-based operator you manage via ArgoCD (cert-manager `Certificate`s, an internal platform's custom CRDs, etc.) is a real, common piece of platform-engineering work — without it, ArgoCD will show `Unknown` health for those resources forever, which is unhelpful noise in dashboards and alerts.

## Notifications — closing the loop without manual UI polling

The **ArgoCD Notifications** engine (originally a separate `argocd-notifications` controller, now typically bundled) triggers configurable messages to Slack, email, webhooks, PagerDuty, etc., based on `Application` events — sync succeeded, sync failed, health degraded, and so on.

```yaml
# argocd-notifications-cm ConfigMap
data:
  service.slack: |
    token: $slack-token

  template.app-sync-failed: |
    message: |
      Sync failed for {{.app.metadata.name}}: {{.app.status.operationState.message}}

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
```

```yaml
# On the Application resource itself — subscribe it to a trigger
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
```

Mechanics:
- **`service`** defines *how* to deliver (Slack token, SMTP config, webhook URL, etc.) — configured once, reused across many triggers/templates.
- **`template`** defines the message content, with access to the full `Application` object's fields (`{{.app.*}}`) via Go templating.
- **`trigger`** defines *when* to fire — a condition expression evaluated against the `Application`'s live state, checked on every reconciliation.
- **Subscriptions** (the annotation on the `Application`, or org-wide default subscriptions in the ConfigMap) tie a specific trigger+template+service combination to specific `Application`s — so different teams/services can route notifications to different Slack channels without duplicating trigger/template definitions.

### Practical notification triggers worth setting up

- **Sync failed** — the deploy didn't apply cleanly; needs immediate attention.
- **Health degraded** — the deploy applied, but the resulting workload isn't healthy (e.g., `CrashLoopBackOff`) — arguably more urgent than a sync failure, since it means something is actually broken in the running system, not just pending.
- **Sync succeeded to production** — lower urgency, but valuable as an audit trail in a deploy-announcements channel, especially for teams practicing "everyone sees what's live."
- **`OutOfSync` for longer than N minutes on a manual-sync Application** — a signal that drift has been sitting unreviewed, worth a periodic reminder rather than only reacting live.

## Points to Remember

- Sync status (matches Git?) and health status (is it working?) are independent axes — a fully Synced Application can still be Unhealthy, and that combination is exactly the case worth alerting on.
- An `Application`'s overall health is the worst health status among its resources — diagnosing unhealthy status means finding the specific dragging resource, not assuming uniform failure.
- CRDs have `Unknown` health by default; writing a Lua-based custom health check in `argocd-cm` is standard practice for any CRD-backed operator you manage via ArgoCD.
- The Notifications engine has three composable pieces — `service` (how to send), `template` (what to say), `trigger` (when) — subscribed to specific `Application`s via annotations.
- "Health degraded" notifications are often more urgent than "sync failed" ones, since degraded health means something live is actually broken, not just pending a fix.

## Common Mistakes

- Treating "Synced" (green checkmark) as equivalent to "working" — a Deployment can be perfectly Synced to Git while its pods are stuck in `CrashLoopBackOff`, which only the separate health status surfaces.
- Never writing custom health checks for CRD-managed resources, leaving them permanently `Unknown` in dashboards — which trains the team to ignore health status for those resources entirely, defeating the purpose.
- Setting up only a "sync failed" notification and skipping "health degraded" — missing the more operationally urgent signal that a supposedly successful deploy actually broke something at runtime.
- Routing every single notification trigger to one firehose Slack channel instead of scoping subscriptions per-team/per-Application, causing alert fatigue and people muting the channel entirely.
- Forgetting that notification triggers are evaluated against live reconciliation state — a flapping health status (briefly degraded, then healthy again) can fire repeated notifications if the trigger condition isn't debounced/scoped carefully.
