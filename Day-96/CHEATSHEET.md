# Day 96 — Cheatsheet: SLOs, SLAs & Error Budgets

## Vocabulary

```
SLI  -> Service Level Indicator: the MEASUREMENT (a ratio/percentile, e.g. good/total requests)
SLO  -> Service Level Objective: the TARGET for that measurement (e.g. 99.9% over 30d) — internal
SLA  -> Service Level Agreement: the CONTRACT with customers, has financial penalties — external
         SLA (loosest) < SLO (tighter, internal safety margin) < actual performance
```

## Error budget = allowed downtime table

```
SLO       Downtime/month(30d)   Downtime/week   Downtime/year
99%       7h 18m                1h 41m          3d 15h
99.9%     43.8 min   <- MEMORIZE THIS ONE
99.95%    21.9 min              5.04 min        4h 22m
99.99%    4.38 min              1.01 min        52.6 min
99.999%   26.3 sec              6.05 sec        5.26 min

Derivation: window_minutes x (1 - SLO_target)
  30 days x 24h x 60min x (1 - 0.999) = 43,200 x 0.001 = 43.2 min
Each extra "nine" = 10x less allowed downtime = exponentially harder to sustain.
```

## SLI PromQL patterns

```promql
# Availability SLI
sum(rate(http_requests_total{job="x", code!~"5.."}[30d]))
/
sum(rate(http_requests_total{job="x"}[30d]))

# Latency SLI (needs a histogram bucket AT your threshold)
sum(rate(http_request_duration_seconds_bucket{job="x", le="0.3"}[30d]))
/
sum(rate(http_request_duration_seconds_count{job="x"}[30d]))
```

## Burn rate

```
Burn rate 1   -> consuming budget exactly at sustainable pace (lands exactly at SLO at window end)
Burn rate 10  -> consuming 10x too fast -> 30-day budget gone in 3 days
Burn rate 14.4 sustained 1h  -> ~2% of 30d budget burned -> PAGE (fast detection)
Burn rate 3   sustained 6h   -> ~2% of 30d budget burned -> TICKET (slow, more time to react)

Alert on burn rate (leading indicator), NOT "budget already exhausted" (lagging, too late).
```

## Multi-window burn-rate alert (why two windows)

```
short window  -> fast detection of severe outages, but noisy (blips)
long window   -> confirms SUSTAINED degradation, but slow to detect fast outages
Fire alert only when BOTH short AND long window exceed the threshold multiplier.
```

## Sloth SLO spec (skeleton)

```yaml
version: "prometheus/v1"
service: "checkout-api"
slos:
  - name: "requests-availability"
    objective: 99.9
    sli:
      events:
        error_query: sum(rate(http_requests_total{job="x",code=~"5.."}[{{.window}}]))
        total_query: sum(rate(http_requests_total{job="x"}[{{.window}}]))
    alerting:
      name: HighErrorRate
      page_alert:   {labels: {severity: page}}
      ticket_alert: {labels: {severity: ticket}}
```

```bash
docker run --rm -v "$(pwd)":/work ghcr.io/slok/sloth:v0.11.0 \
  generate -i /work/slo.yaml -o /work/slo-rules.yaml
```

## SLO-based vs threshold alerting

```
Threshold alert:  "error_rate > 1%"                 -> ignores budget, ignores urgency, one-size-fits-all
SLO/burn-rate:    "at this rate, will exhaust budget before window resets" -> relative to THIS service's
                                                        own tolerance, encodes urgency via severity tier

Page on: user-facing symptom / budget-threatening burn rate
Ticket on: cause-level infra signals that don't yet threaten the SLO (disk filling, one pod flapping)
```

## Error budget policy (decision rule)

```
Budget healthy    -> ship fast, take risks, deploy freely
Budget exhausted  -> freeze non-critical releases, redirect eng effort to reliability work
                     (pre-agreed BEFORE the incident, not argued mid-postmortem)
```
