# Day 89 — Prometheus Fundamentals: Alertmanager

**Phase:** 3 – Observability | **Week:** W15 | **Domain:** Metrics | **Flag:** ⚡ Interview-critical

## Brief

Prometheus itself only *evaluates* alerting rules and fires alerts into Alertmanager — it doesn't decide who gets paged, how noisy duplicate alerts get deduplicated, or how a "database down" alert should suppress the 40 "can't connect to database" alerts it inevitably causes downstream. That's Alertmanager's entire job: routing, grouping, silencing, and inhibition. Interviewers ask about this because a team that only knows how to write a PromQL alert expression but not how routing/inhibition work will page an entire on-call rotation for a single root-cause incident — a very visible, very expensive mistake in production.

## How an alert gets from Prometheus rule to a page

1. A Prometheus **alerting rule** evaluates a PromQL expression on a schedule; if the condition is true for at least the rule's `for:` duration, the alert transitions from `pending` to `firing`.
2. Prometheus pushes firing alerts (as a set of labels + annotations) to **Alertmanager**.
3. Alertmanager **groups** related alerts together, applies **routing** to decide which receiver(s) should get notified, checks **inhibition** and **silence** rules to suppress anything that shouldn't notify, and finally sends the notification (Slack, PagerDuty, email, webhook, etc.) via the matched **receiver**.

```yaml
# Prometheus alerting rule
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: job:http_requests:error_ratio5m > 0.05
        for: 10m
        labels:
          severity: critical
          team: checkout
        annotations:
          summary: "Error rate above 5% for {{ $labels.job }}"
          description: "{{ $value | humanizePercentage }} of requests are failing."
```

The `for: 10m` matters as much as the expression itself: without it, a single noisy 30-second blip would fire and immediately resolve, paging someone for a non-issue. `for:` requires the condition to hold continuously across evaluations before transitioning to `firing`, filtering out transient noise.

## Routes — a tree, not a flat list

```yaml
route:
  receiver: default-slack
  group_by: [alertname, team]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
      continue: true
    - match:
        team: checkout
      receiver: checkout-slack
```

Routing is a **tree**: the top-level `route` is the default; child `routes` are evaluated in order, and the **first matching child wins** *unless* it sets `continue: true`, in which case evaluation keeps going into subsequent siblings too (useful when you want both a PagerDuty page *and* a Slack notification for the same critical alert, as shown above).

- `group_by` controls which alerts get **bundled into a single notification** — e.g., grouping by `alertname` + `team` means if 15 pods in the checkout team all trip the same alert simultaneously, you get **one** notification listing 15 instances, not 15 separate pages.
- `group_wait` — how long to wait after the *first* alert in a new group fires, to see if related alerts join it before sending the initial notification (batches a burst of near-simultaneous alerts into one message).
- `group_interval` — how long to wait before sending a notification about *new* alerts added to an *already-notified* group.
- `repeat_interval` — how long to wait before re-sending a notification for an alert that's still firing and hasn't changed — this is what stops you from getting paged every single evaluation cycle for a known, ongoing incident.

## Receivers

A receiver defines *where* a notification goes — Slack webhook, PagerDuty integration key, email SMTP config, or a generic webhook for anything else:

```yaml
receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - routing_key: '<PAGERDUTY_INTEGRATION_KEY>'
  - name: checkout-slack
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/...'
        channel: '#checkout-alerts'
        send_resolved: true
```

`send_resolved: true` is worth calling out specifically: without it, a channel gets a flood of "firing" messages and never learns when things recovered — leading to permanently-unresolved-looking alert channels and alert fatigue.

## Inhibition rules — suppress the noise, not the signal

Inhibition suppresses alerts that match a certain pattern **if another alert (the "source") is already firing** — the classic use case is: if the whole cluster/node is down, don't also page for every individual service that depends on it.

```yaml
inhibit_rules:
  - source_matchers:
      - alertname="NodeDown"
    target_matchers:
      - severity="warning"
    equal: ['node']
```

This says: if `NodeDown` is firing for a given `node`, suppress any `warning`-severity alert that shares the same `node` label — because every pod/service on that node failing is a *symptom*, not 30 separate incidents. `equal` is critical: it scopes the suppression to alerts sharing a specific label value, so you don't accidentally inhibit warnings on unrelated, healthy nodes.

**Inhibition vs silence — a common interview mix-up:** a **silence** is a manually-created, time-bounded mute (e.g., "mute all alerts matching `env=staging` for the next 2 hours during a planned maintenance window") — someone has to create it, typically via the Alertmanager UI or API, and it expires. An **inhibition rule** is a standing, always-on relationship defined in config ("this class of alert always suppresses that class of alert when both are true") that requires no human action to trigger.

## Points to Remember

- The pipeline is: PromQL rule (`for:` duration) → firing alert → Alertmanager grouping/routing/inhibition → receiver notification.
- `group_by`/`group_wait`/`group_interval`/`repeat_interval` exist specifically to prevent alert storms and duplicate paging for the same ongoing issue.
- Routes are a tree evaluated top-to-bottom; the first match wins unless `continue: true` lets evaluation fall through to additional matching siblings.
- Inhibition rules automatically suppress downstream/symptom alerts when a root-cause alert is firing and they share matching labels (`equal:`) — silences are manual, time-bounded mutes a human creates.
- `send_resolved: true` is what makes a Slack/webhook receiver actually announce recovery, not just failures.

## Common Mistakes

- Writing an alert without a `for:` duration, causing pages for single-scrape blips that self-resolve before anyone can even look.
- Not setting `equal:` correctly (or at all) in an inhibition rule — it can inhibit far more broadly than intended, silently swallowing real alerts on unrelated resources that happen to share an `alertname`.
- Confusing silences (manual, temporary) with inhibition (automatic, standing) in interviews or in postmortems — "we should silence this class of alert whenever the parent fails" is actually describing an inhibition rule, not a silence.
- Setting `repeat_interval` too short (pages every few minutes for a known ongoing issue — alert fatigue) or too long (a genuinely new occurrence of the same alertname goes unnoticed because the team assumes it's the same old notification).
- Forgetting `continue: true` when a critical alert needs to reach *both* PagerDuty and a Slack channel — without it, only the first matching route fires and the second notification path never triggers.
- Building receivers/routes without `group_by`, causing a single root-cause event (like a shared dependency going down) to generate one notification per affected instance instead of one grouped notification.
