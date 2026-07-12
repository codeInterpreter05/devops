# Day 73 — Cheatsheet: Full CI/CD Pipeline Project

## Pipeline stage order (hard gate chain)

```
test -> sast -> build -> scan -> sign -> push -> (ArgoCD reconciles)
```

## GitHub Actions job dependency + gating

```yaml
jobs:
  a:
    runs-on: ubuntu-latest
    steps: [...]
  b:
    needs: a              # b only runs if a succeeds
    runs-on: ubuntu-latest
  notify:
    needs: [a, b]
    if: always()          # run even if a or b failed
```

```yaml
permissions:
  contents: read
  id-token: write         # OIDC / keyless signing / cloud auth
  security-events: write   # upload SARIF to code scanning
```

## Trivy scan (blocking gate)

```bash
trivy image --severity CRITICAL,HIGH --exit-code 1 --format sarif -o results.sarif myimage@sha256:...
```

```yaml
- uses: aquasecurity/trivy-action@0.24.0
  with:
    image-ref: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
    severity: CRITICAL,HIGH
    exit-code: '1'
```

## Cosign keyless signing

```bash
cosign sign --yes ghcr.io/org/app@sha256:<digest>
cosign verify \
  --certificate-identity-regexp ".*" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/org/app@sha256:<digest>
```

## Digest vs. tag — always pass digest between jobs

```yaml
outputs:
  digest: ${{ steps.build.outputs.digest }}   # from docker/build-push-action
```
```
myimage:latest         # mutable — can point to a different image tomorrow
myimage@sha256:abc123  # immutable — always this exact content
```

## ArgoCD Application — staging (auto) vs. production (manual)

```yaml
# staging: auto-sync + self-heal
syncPolicy:
  automated:
    prune: true
    selfHeal: true

# production: no automated block = manual only
syncPolicy: {}
```

```bash
argocd app list
argocd app sync myapp-production          # manual promotion
argocd app diff myapp-production          # see pending changes before syncing
argocd app history myapp-production       # audit trail of past syncs
argocd app rollback myapp-production <id> # roll back to a prior revision
```

## Kustomize image bump (GitOps repo update)

```bash
cd overlays/staging
kustomize edit set image myapp=ghcr.io/org/app@sha256:<digest>
git commit -am "chore: bump staging image" && git push
```

## Slack notify-on-failure job

```yaml
- name: Notify Slack on failure
  if: contains(needs.*.result, 'failure')
  uses: slackapi/slack-github-action@v1.27.0
  with:
    channel-id: 'C0123456789'
    payload: |
      {"text": "🔴 Pipeline failed: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"}
  env:
    SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

## PR preview environment (namespace-per-PR)

```bash
kubectl create namespace pr-42 --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install myapp-pr-42 ./chart -n pr-42 --set image.tag=$SHA
kubectl delete namespace pr-42 --ignore-not-found   # teardown
```

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
```

## Useful `gh` CLI commands for pipeline debugging

```bash
gh run list --workflow=ci-cd.yml --limit 5
gh run view <run-id> --log-failed
gh run rerun <run-id> --failed
gh workflow view ci-cd.yml
```
