# Day 61 — Cheatsheet: GitHub Actions Fundamentals

## Minimal workflow skeleton

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "hello"
```

## Triggers (`on`)

```yaml
on:
  push:
    branches: [main, 'release/**']
    paths: ['src/**']
    paths-ignore: ['**.md']
  pull_request:
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '0 2 * * *'      # UTC, best-effort
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
  workflow_call:              # makes this workflow reusable
  repository_dispatch:        # external webhook trigger
```

## Jobs & steps

```yaml
jobs:
  build:
    needs: [test, lint]              # ordering — otherwise parallel
    if: github.event_name == 'push'  # conditional job execution
    runs-on: ubuntu-latest
    container: node:20-alpine        # optional: pin exact toolchain
    services:
      postgres:
        image: postgres:16
        ports: ['5432:5432']
    outputs:
      digest: ${{ steps.push.outputs.digest }}
    steps:
      - id: push
        run: echo "digest=abc123" >> "$GITHUB_OUTPUT"
```

## Job-to-job data passing

```yaml
# producer job
outputs:
  value: ${{ steps.step-id.outputs.key }}
steps:
  - id: step-id
    run: echo "key=hello" >> "$GITHUB_OUTPUT"

# consumer job
needs: producer
run: echo "${{ needs.producer.outputs.value }}"
```

## Contexts

```yaml
${{ github.repository }}          # owner/repo
${{ github.actor }}               # who triggered it
${{ github.sha }}                 # commit SHA
${{ github.ref }}                 # refs/heads/main or refs/pull/123/merge
${{ github.event_name }}          # push, pull_request, etc.
${{ github.event.pull_request.title }}  # UNTRUSTED — never inline into run: directly
${{ secrets.MY_SECRET }}          # masked in logs automatically
${{ vars.MY_VAR }}                # non-secret config
${{ runner.os }}                  # Linux, Windows, macOS
${{ needs.<job>.outputs.<key> }}
${{ steps.<id>.outputs.<key> }}
```

## Safe untrusted-input pattern

```yaml
# WRONG — injection risk
- run: echo "${{ github.event.pull_request.title }}"

# RIGHT
- run: echo "$PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

## Matrix strategy

```yaml
strategy:
  fail-fast: false          # default true — cancels rest on first failure
  max-parallel: 2
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20, 22]
    include:
      - os: ubuntu-latest
        node: 22
        experimental: true
    exclude:
      - os: macos-latest
        node: 18
runs-on: ${{ matrix.os }}
continue-on-error: ${{ matrix.experimental == true }}
```

## Reusable workflow (`workflow_call`)

```yaml
# reusable.yml
on:
  workflow_call:
    inputs:
      env-name: { required: true, type: string }
    secrets:
      token: { required: true }
    outputs:
      digest: { value: ${{ jobs.build.outputs.digest }} }

# caller
jobs:
  release:
    uses: ./.github/workflows/reusable.yml
    with: { env-name: production }
    secrets: inherit    # or explicit: secrets: { token: ${{ secrets.X }} }
```

## Composite action

```yaml
# .github/actions/my-action/action.yml
runs:
  using: 'composite'
  steps:
    - run: npm ci
      shell: bash        # REQUIRED on every run: step in a composite action

# usage
- uses: ./.github/actions/my-action
```

## Caching

```yaml
# shorthand via setup-* action
- uses: actions/setup-node@v4
  with: { node-version: '20', cache: 'npm' }

# manual, general-purpose
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: npm-${{ runner.os }}-

# Docker layer cache
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: app:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## AWS auth (static keys — see Day 62 for OIDC)

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ vars.AWS_REGION }}
- uses: aws-actions/amazon-ecr-login@v2
  id: ecr-login
```

## `act` — run workflows locally

```bash
act push -j build                     # run the 'build' job for a push event
act pull_request -j test               # simulate a PR event
act --secret-file .secrets             # load secrets from file (gitignore it!)
act -l                                 # list all jobs discoverable in the repo
```
