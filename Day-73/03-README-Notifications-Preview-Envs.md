# Day 73 — Full CI/CD Pipeline Project: Notifications & PR Preview Environments

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD | **Flag:** 📌 Capstone

## Brief

A pipeline that silently fails at 2 a.m. and waits for someone to notice on the GitHub Actions tab isn't operationally complete. Two features turn a pipeline from "something that runs" into "something a team can actually trust and iterate against quickly": **failure notifications** that reach humans where they already are (Slack), and **PR preview environments** that let reviewers click a live URL instead of trusting a diff. Both are cheap to add and disproportionately valuable — they're also frequently the first thing a hiring manager asks about, because they signal you've operated a pipeline for a real team, not just gotten one to go green once.

## Slack notifications on failure

The naive approach — a Slack message on every push regardless of outcome — trains a team to mute the channel within a week. The correct pattern: **notify on failure and on recovery, stay silent on repeated success.**

```yaml
  notify:
    needs: [test, sast, build, scan, sign, push]
    if: always()                     # run even if an earlier job failed
    runs-on: ubuntu-latest
    steps:
      - name: Determine overall status
        id: status
        run: |
          if [[ "${{ contains(needs.*.result, 'failure') }}" == "true" ]]; then
            echo "outcome=failure" >> "$GITHUB_OUTPUT"
          else
            echo "outcome=success" >> "$GITHUB_OUTPUT"
          fi
      - name: Notify Slack on failure
        if: steps.status.outputs.outcome == 'failure'
        uses: slackapi/slack-github-action@v1.27.0
        with:
          channel-id: 'C0123456789'
          payload: |
            {
              "text": "🔴 Pipeline failed on `${{ github.ref_name }}` — <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|view run> (commit ${{ github.sha }} by ${{ github.actor }})"
            }
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

Key mechanics:

- **`if: always()`** on the `notify` job is essential — by default, if any job in `needs:` fails, dependent jobs are *skipped*, not run. Without `always()`, your failure notifier would itself be skipped exactly when you need it, which is a surprisingly common bug in first-draft pipelines.
- **`contains(needs.*.result, 'failure')`** inspects every upstream job's `result` (`success`, `failure`, `cancelled`, `skipped`) via the `needs` context — this is how one job can summarize the outcome of several without re-running anything.
- Include a **direct link to the failed run** (`github.server_url`/`repository`/`actions/runs/run_id`) — a notification that just says "build failed" with no link creates friction that erodes trust in the whole notification system.
- Prefer a bot token (`slackapi/slack-github-action` with `SLACK_BOT_TOKEN`) over legacy incoming webhooks where possible — webhooks are a single static secret per-channel with no rotation/audit trail; a scoped bot token can be revoked/rotated centrally and audited in Slack's admin console.

## PR preview environments

A preview environment stands up a temporary, isolated instance of the app for every open pull request, torn down automatically when the PR closes. This turns code review from "reading a diff and trusting it" into "clicking a URL and testing the actual behavior" — invaluable for frontend changes, API contract changes, and anything a reviewer can't fully evaluate from a text diff alone.

```yaml
  preview:
    if: github.event_name == 'pull_request'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy preview namespace
        run: |
          NS="pr-${{ github.event.number }}"
          kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -
          helm upgrade --install "myapp-$NS" ./chart \
            --namespace "$NS" \
            --set image.tag=${{ github.sha }} \
            --set ingress.host="pr-${{ github.event.number }}.preview.myapp.dev"
      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 Preview deployed: https://pr-${context.issue.number}.preview.myapp.dev`
            })

  preview-cleanup:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Tear down preview namespace
        run: kubectl delete namespace "pr-${{ github.event.number }}" --ignore-not-found
```

Design considerations that matter in practice:

- **Namespace-per-PR** is the cheapest isolation boundary in Kubernetes — cheaper than a whole separate cluster, and `kubectl delete namespace` cleanly removes every resource created within it (no orphan-resource cleanup script needed).
- **Cleanup on `closed`, not just `merged`** — a PR can be closed without merging, and abandoned preview environments are a classic source of silently accumulating cloud cost. Trigger cleanup on the `pull_request: [closed]` event, which fires for both merged and unmerged closures.
- **TTL as a safety net.** Even with a cleanup job, add a scheduled workflow (`on: schedule`) that deletes any `pr-*` namespace older than, say, 48 hours — cleanup jobs can fail to run (e.g., if the workflow file itself was deleted in the PR being closed), so a time-based backstop prevents orphaned environments from running forever.
- **Cost-bound the preview environment** — set low resource requests/limits and consider scaling to zero (e.g., via KEDA or a simple cron-based scale-down outside business hours) since these environments exist only for the lifetime of code review, not production traffic patterns.

## Points to Remember

- Notify on failure (and ideally on recovery-from-failure), not on every run — indiscriminate notifications get muted and stop being useful.
- `if: always()` is required on a summary/notification job so it still runs when an upstream job in its `needs:` list failed — GitHub Actions skips dependents of failed jobs by default.
- Include a direct link to the failed run in the notification; a message without a link adds friction instead of removing it.
- PR preview environments use namespace-per-PR for cheap isolation; always pair the teardown trigger with a time-based TTL backstop in case the explicit cleanup step never runs.
- Comment the live preview URL directly on the PR so reviewers don't have to guess or construct it themselves.

## Common Mistakes

- Forgetting `if: always()` on the notify job, so it silently gets skipped on the exact failure runs it exists to report.
- Sending a Slack message on every single pipeline run (success or failure), causing the channel to be muted within days and defeating the point of alerting.
- Cleaning up preview environments only on PR `merged`, leaving orphaned namespaces (and their cloud cost) behind for every PR that was closed without merging.
- No TTL/backstop job — if the cleanup step itself is broken or skipped (e.g., due to a workflow syntax error introduced in that same PR), the preview environment runs forever.
- Giving preview environments production-sized resource requests, quietly inflating cluster cost as concurrent open PRs accumulate.
