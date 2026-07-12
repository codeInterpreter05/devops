# Day 63 — Cheatsheet: GitLab CI Deep Dive

## Minimal pipeline skeleton

```yaml
stages:
  - test
  - build
  - deploy

test-job:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test
```

## Stages vs jobs vs needs

```yaml
# All jobs in the same stage run in parallel; next stage waits for all of current stage
stages: [test, build, deploy]

# needs: bypass strict stage ordering, start as soon as dependency finishes
build-job:
  stage: build
  needs: [test-job]   # doesn't wait for other test-stage jobs
```

## `rules:` — top-to-bottom, first match wins

```yaml
job:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: never
    - changes:
        - src/**/*
      when: on_success
    - when: never   # catch-all — no implicit "runs anyway"
```

Common predefined variables:
```
$CI_COMMIT_BRANCH
$CI_COMMIT_SHA
$CI_COMMIT_TAG
$CI_PIPELINE_SOURCE       # push, merge_request_event, schedule, web, api...
$CI_MERGE_REQUEST_ID
$CI_PROJECT_NAME
$CI_REGISTRY / $CI_REGISTRY_IMAGE / $CI_REGISTRY_USER / $CI_REGISTRY_PASSWORD
```

## `extends` (GitLab-aware deep merge)

```yaml
.node-defaults:
  image: node:20-alpine
  before_script: [npm ci]

test-job:
  extends: .node-defaults    # or extends: [.a, .b] for multiple
  stage: test
  script: [npm test]         # arrays are REPLACED, not merged
```

## YAML anchors (plain YAML, single-document only)

```yaml
.template: &template
  image: node:20-alpine

test-job:
  <<: *template
  script: [npm test]
```

## `include:` — compose pipelines from multiple files

```yaml
include:
  - local: '.gitlab/ci/test.yml'
  - project: 'my-group/ci-templates'
    ref: main
    file: '/templates/deploy.yml'
  - template: 'Security/SAST.gitlab-ci.yml'
```

## Matrix (`parallel: matrix:`)

```yaml
test-job:
  parallel:
    matrix:
      - NODE_VERSION: ['18', '20', '22']
  image: node:$NODE_VERSION-alpine
  script: [npm test]
```

## Container registry (built in, zero setup)

```yaml
build-image:
  image: docker:24
  services: [docker:24-dind]
  variables: { DOCKER_TLS_CERTDIR: "/certs" }
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
```

## Environments

```yaml
deploy-prod:
  environment:
    name: production
    url: https://example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  script: [./deploy.sh]
```

## Runners

```bash
gitlab-runner register \
  --url https://gitlab.com/ \
  --registration-token <TOKEN> \
  --executor docker \
  --docker-image alpine:latest \
  --tag-list "docker,internal-vpc"
```

```yaml
job:
  tags: [internal-vpc]   # must match a subset of runner's tags
```

## Cache

```yaml
job:
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/
```

## GitHub Actions -> GitLab CI quick translation

```
on.push.branches            -> rules: - if: '$CI_COMMIT_BRANCH == "main"'
jobs.<j>.needs               -> needs: [a, b]
strategy.matrix              -> parallel: matrix:
secrets.X                    -> $X (CI/CD Variable, masked/protected)
composite action             -> extends + shared template
workflow_call                -> include: (project/template)
actions/cache                -> cache: key/paths
Environment + reviewer        -> environment: + when: manual (+ protected environments)
```
