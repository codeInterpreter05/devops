# Day 78 — Cheatsheet: Developer Experience in CI/CD

## Fast feedback: caching, matrix, path filters, concurrency

```yaml
# Cache keyed on lockfile hash (invalidates automatically on dependency change)
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-

# Matrix / test sharding
strategy:
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npm test -- --shard=${{ matrix.shard }}/4

# Skip jobs for irrelevant path changes
on:
  pull_request:
    paths:
      - 'src/**'
      - 'package-lock.json'

# Cancel superseded runs on rapid pushes
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

## `act` — local GitHub Actions runner

```bash
brew install act                      # macOS install
act -n                                 # dry run, show what would execute
act push                               # run push-triggered workflows
act pull_request                       # simulate a PR event
act -j build                           # run a specific job
act -s GITHUB_TOKEN=xxx                # pass a secret inline
act --secret-file .secrets             # pass secrets from a file (gitignore this!)
act -P ubuntu-latest=catthehacker/ubuntu:full-latest   # bigger, closer-to-real image
```

`.actrc` (repo root, shared defaults):
```
-P ubuntu-latest=catthehacker/ubuntu:act-latest
--container-architecture linux/amd64
```

## PR preview environment lifecycle

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy:
    if: github.event.action != 'closed'
    steps:
      - run: kubectl create namespace pr-${{ github.event.number }} --dry-run=client -o yaml | kubectl apply -f -
  teardown:
    if: github.event.action == 'closed'
    steps:
      - run: kubectl delete namespace pr-${{ github.event.number }} --ignore-not-found
```

## Trunk-based dev — the checklist

```
[ ] Branches live < 1-2 days before merging to main
[ ] CI on every merge to main is fast (<5 min) and required to pass
[ ] Incomplete work ships behind a feature flag, not a long-lived branch
[ ] Branch protection: required status checks + review before merge
```

## Feature flags (LaunchDarkly-style)

```hcl
# flags as code — Terraform provider
resource "launchdarkly_feature_flag" "new_checkout_flow" {
  project_key    = "web-app"
  key            = "new-checkout-flow"
  variation_type = "boolean"
  variations { value = "true" }
  variations { value = "false" }
}
```

```
Rollout pattern:  5% -> watch error rate -> 25% -> 100% -> remove flag + dead code
Kill switch:      wrap risky path in a flag from day 1, so incident response = flip flag, not redeploy
```

## DORA metrics — formulas and elite benchmarks

```
Deployment Frequency     how often you deploy to prod         Elite: on-demand, multiple/day
Lead Time for Changes    commit -> running in production      Elite: < 1 hour
Change Failure Rate      failed_deploys / total_deploys        Elite: 0-15%
Time to Restore (MTTR)   detection -> resolved                 Elite: < 1 hour
Reliability (5th metric) SLO attainment (org-defined)          n/a (context-specific)
```

```bash
# rough DORA data-gathering commands
git log --since="30 days ago" --oneline main | wc -l     # deploy-frequency proxy (merges)
git log --format='%H %aI' <sha> -1                        # commit timestamp for lead-time calc
```

## Related tools

```
act              local GitHub Actions runner
fourkeys         Google's open-source DORA metrics pipeline
Sleuth / LinearB / Faros AI    commercial DORA metrics platforms
LaunchDarkly     feature-flag-as-a-service
Argo CD PR generator   ApplicationSet-based per-PR environments
```
