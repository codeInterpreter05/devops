# Day 78 — Developer Experience in CI/CD: Measuring It With DORA Metrics

**Phase:** 2 – CI/CD & Security | **Week:** W13 | **Domain:** CI/CD | **Flag:** —

## Brief

Everything in the first two files of today — fast feedback, preview environments, trunk-based dev, feature flags — is a means to an end, and DORA metrics are how you prove the end was actually reached. Originating from Google's DevOps Research and Assessment (DORA) team's multi-year research program (published as the annual *State of DevOps Report* and the book *Accelerate*), these four metrics are the closest thing the industry has to a scientifically validated way to measure software delivery performance. This is asked constantly in interviews for a specific reason: it's a fast way to check whether a candidate thinks about delivery performance as *measured outcomes* or only as *tools they've used*.

## The four (now five) key metrics, precisely defined

1. **Deployment Frequency** — how often you successfully deploy to production. Not "how often CI runs," not "how often you merge to `main`" — specifically, how often code actually reaches production users. Elite performers: on-demand, multiple deploys per day, per service. Low performers: fewer than once every six months.

2. **Lead Time for Changes** — the time from a commit landing (or from work starting, depending on the exact definition your org uses — commit-to-deploy is the more common and more measurable version) to that change running successfully in production. Elite: less than one hour. Low: more than six months.

3. **Change Failure Rate (CFR)** — the percentage of deployments to production that result in degraded service requiring remediation (a rollback, a hotfix, a patch). Formula: `failed deployments / total deployments`. Elite: 0–15%. Low: 46–60% (note: high performers deploy far more often *and* fail less often per deploy — frequency and stability are not actually in tension, which is the counterintuitive, headline finding of the whole DORA research program, directly contradicting the older intuition that "moving slower and deploying less often is safer").

4. **Mean Time to Restore (MTTR) / Time to Restore Service** — how long it takes to recover from a failure in production once it's detected. Elite: less than one hour. Low: more than one week.

5. **Reliability** (added in more recent DORA reports as a fifth metric) — whether the system meets user-facing reliability targets (availability, latency, correctness) as measured against SLOs. This one is intentionally softer/organization-defined, acknowledging that the original four are excellent proxies for *delivery* performance but don't fully capture whether the thing being delivered actually stays reliable for users.

## How you'd actually instrument this, not just define it

Definitions are cheap; an interviewer who's actually operated a pipeline will ask "okay, how do you *get* these numbers." The honest answer, mapped to real tooling:

- **Deployment Frequency**: count actual production deployment events, not merges. If you're on GitHub, the [Deployments API](https://docs.github.com/en/rest/deployments) records an event each time a deployment to an environment (e.g., `production`) fires — tag your CD job to create one:

  ```yaml
  - uses: actions/github-script@v7
    with:
      script: |
        await github.rest.repos.createDeployment({
          owner: context.repo.owner,
          repo: context.repo.repo,
          ref: context.sha,
          environment: 'production',
          auto_merge: false
        });
  ```

  Or, if you deploy via Argo CD, its sync events (`argocd app history <app>`) are an equally valid, arguably more accurate source of truth — a GitOps sync to the production `Application` *is* the deploy.

- **Lead Time for Changes**: requires joining two timestamps — the commit's authored/committed time (`git log --format=%aI <sha>`) and the deployment event time for that same commit's SHA. A minimal homegrown version: query closed PRs via the GitHub API, find the first production deployment whose SHA includes that PR's merge commit, and compute the delta. At any real scale, teams use a dedicated tool (Sleuth, LinearB, Faros AI, or Google's own open-source **`fourkeys`** project) rather than hand-rolling this indefinitely — but you should be able to explain the underlying computation even if you use a tool for it.

- **Change Failure Rate**: needs a definition of "failed" your org agrees on up front — typically: a deployment followed within some window (e.g., 1 hour, or "before the next successful deploy") by a rollback, a hotfix deploy, or an incident being opened and linked to that deploy. `failed_deploys_this_period / total_deploys_this_period`.

- **MTTR**: needs incident tooling with real timestamps — when an incident was **detected** (not when the underlying bug was introduced — that distinction matters, see Common Mistakes) versus when it was **resolved**. PagerDuty, Opsgenie, or even a disciplined incident-tracking spreadsheet with two timestamp columns is sufficient; the metric is the delta between them, averaged (or better, reported as a percentile, since a mean is easily skewed by one very long outlier incident).

A simple internal pipeline for all four: push deployment + incident events into a small time-series/metrics store (Prometheus with a custom exporter, or just a table in Postgres) and build a Grafana dashboard computing the four ratios/deltas over rolling windows (weekly, monthly) — this is genuinely a good small project to have built and be able to talk through in an interview.

## Points to Remember

- The four core metrics: **Deployment Frequency**, **Lead Time for Changes**, **Change Failure Rate**, **Time to Restore Service** — plus **Reliability** as a fifth, more recently added, SLO-based metric.
- Elite performer benchmarks (memorize these, they get asked directly): on-demand multiple deploys/day, lead time under an hour, MTTR under an hour, change failure rate 0–15%.
- The counterintuitive core finding of the whole DORA research program: **speed and stability are not a tradeoff** — elite performers are both faster and more reliable than low performers, not one at the expense of the other.
- Deployment frequency counts actual **production deployments**, not merges to `main` and not CI runs — conflating these is the single most common measurement mistake.
- MTTR's clock starts at **detection**, not at when the underlying defect was introduced — a bug that sat silently in production for a month but was fixed within 20 minutes of being noticed is a 20-minute MTTR, not a month-long one.

## Common Mistakes

- Measuring "PRs merged per day" or "CI runs per day" and calling it Deployment Frequency — these are proxies for activity, not for code actually reaching production users, and they can be gamed (many tiny merges) without any real delivery-speed improvement.
- Gaming the metrics directly — e.g., inflating deployment frequency with meaningless micro-deploys, or narrowing the definition of "failure" so aggressively that real production incidents don't count toward change failure rate. DORA metrics are meant to be outcome indicators the org trusts, not a KPI to be optimized in isolation from the actual goal (fast, safe delivery of real value).
- Comparing raw DORA numbers across teams with fundamentally different risk profiles (a payments team and an internal-tools team) without context — the benchmarks are useful for tracking your *own* trend over time and against industry bands, not for a naive team-vs-team leaderboard.
- Starting the MTTR clock at when the bug was introduced (or when it started silently affecting a subset of users) instead of when it was actually detected — this conflates "how good is our detection" (a separate, also-important signal, often tracked as Mean Time to Detect) with "how good is our recovery."
- Treating DORA metrics as the entire picture of developer productivity — they measure delivery pipeline performance specifically; pairing them with a framework like **SPACE** (Satisfaction, Performance, Activity, Communication, Efficiency) gives a fuller, less gameable picture of actual developer experience and productivity.
