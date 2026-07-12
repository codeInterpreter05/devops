# Day 78 — Resources: Developer Experience in CI/CD

## Primary (assigned)

- **DORA State of DevOps Report** (dora.dev/research) — free, the assigned starting point for this day. The annual research report defining and benchmarking the four (now five) key metrics, backed by years of survey data across thousands of organizations.

## Deepen your understanding

- **_Accelerate_ by Nicole Forsgren, Jez Humble, and Gene Kim** — the book-length treatment of the DORA research; explains not just *what* the metrics are but the statistical methodology behind why they're validated predictors of organizational performance, not just intuitive-sounding numbers.
- **`nektos/act` GitHub repo** (github.com/nektos/act) — README covers the full flag set, image options, and secrets handling in more depth than any blog post.
- **Trunk Based Development** (trunkbaseddevelopment.com) — a focused reference site (not a single article) covering short-lived branching, release branches, and how TBD interacts with feature flags and CI.
- **LaunchDarkly's engineering blog** (launchdarkly.com/blog) — practical, tool-specific write-ups on progressive rollouts, kill switches, and flag lifecycle management beyond the basics.

## Reference / lookup

- **GitHub Docs — Deployments API and environments** (docs.github.com) — the authoritative reference for recording real deployment events, which is the correct data source for Deployment Frequency and Lead Time.
- **Google's `fourkeys` project** (github.com/dora-team/fourkeys) — an open-source reference implementation of a full DORA metrics collection pipeline; worth reading even if you don't deploy it, as a concrete example of the plumbing described in today's third README.

## Practice

- Instrument your own repo (or this learning repo) using **Lab 4** from today's `LAB.md` — compute all four DORA metrics from real git/CI history rather than treating the definitions as purely theoretical.
- **OverTheWire-style practice isn't applicable here** — instead, the highest-value practice is applying today's caching/matrix/path-filter techniques to a real, currently-slow pipeline you own and measuring the actual before/after wall-clock improvement.
