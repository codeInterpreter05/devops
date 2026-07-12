# Day 62 — Advanced GitHub Actions: Self-Hosted Runners on EKS & Environments

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** CI/CD | **Flag:** None

## Brief

GitHub-hosted runners are convenient but come with real limits: capped runtime, fixed hardware specs, no access to private VPC resources, and per-minute cost that adds up fast at scale. Self-hosted runners — especially ones running as ephemeral pods inside your own EKS cluster — solve all three, at the cost of you now owning runner security and lifecycle. Pair that with GitHub Environments (deployment gates with required human approval) and you have the two building blocks of a production-grade, access-controlled deployment pipeline.

## Why self-host runners on EKS specifically

Running runners as Kubernetes pods (rather than static EC2 boxes) gets you:
- **Elastic scaling** — spin up runner pods on demand as jobs queue, scale to zero when idle, instead of paying for idle EC2 instances around the clock.
- **VPC-native network access** — a runner pod inside your EKS cluster's VPC can reach private RDS instances, internal APIs, and other resources GitHub-hosted runners simply cannot reach without exposing them to the internet.
- **Consistent, version-controlled runner images** — you build and control the runner's base image (same discipline as any other container), instead of trusting GitHub's shared image and its periodic silent updates.
- **Cost control at scale** — for high CI volume, self-hosted compute on infrastructure you already run is typically cheaper than GitHub-hosted minutes.

The standard tool for this is **Actions Runner Controller (ARC)**, a Kubernetes operator that manages runner pods as custom resources.

```yaml
# Example ARC RunnerScaleSet-style config (actions-runner-controller / ARC v2, gha-runner-scale-set chart values)
githubConfigUrl: https://github.com/my-org/my-repo
githubConfigSecret: gha-runner-secret   # PAT or GitHub App credentials, stored as a K8s Secret
minRunners: 0
maxRunners: 10
runnerScaleSetName: eks-runners
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        resources:
          requests: { cpu: '500m', memory: '1Gi' }
          limits: { cpu: '2', memory: '4Gi' }
```

```yaml
# In your workflow, target the self-hosted pool by label instead of ubuntu-latest
jobs:
  deploy:
    runs-on: eks-runners
    steps:
      - uses: actions/checkout@v4
      - run: kubectl apply -f k8s/
```

Key operational points:
- **Ephemeral runners are the security default you want** — configure ARC so each runner pod handles exactly one job and is destroyed afterward. A long-lived, job-reusing runner accumulates state (cached credentials, leftover files, installed tooling) between unrelated jobs — a real lateral-movement risk if one job is malicious or compromised.
- **Never use self-hosted runners on public repos without hard isolation** — anyone can open a PR against a public repo and, if workflows targeting self-hosted runners run automatically on `pull_request` from forks, get arbitrary code execution *inside your infrastructure*. GitHub explicitly warns about this; the mitigation is requiring approval for first-time contributors' workflow runs, or simply never wiring self-hosted runners to fork-triggerable events on public repos.
- **RBAC scoping** — the ARC controller and runner pods should run under a Kubernetes ServiceAccount with the minimum RBAC needed (deploy to specific namespaces, not cluster-admin), and IAM Roles for Service Accounts (IRSA) or Pod Identity if the runner itself needs AWS access — layering the OIDC pattern from file 1 at the Kubernetes level too.

## GitHub Environments with required reviewers

An **Environment** (Settings → Environments) is a named deployment target (`staging`, `production`) that a job can be bound to, unlocking:
- **Required reviewers** — the job pauses and waits for a specified person/team to click "Approve" before it proceeds, directly in the GitHub UI.
- **Wait timer** — a mandatory delay before the job can run (e.g., a 10-minute "cool-off" window).
- **Environment-scoped secrets/variables** — a `production` Environment can hold different (and more tightly access-controlled) secrets than `staging`, even within the same repo.
- **Deployment branch/tag restrictions** — e.g., only allow the `production` Environment to be targeted from `main` or from tags matching `v*`.

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.PROD_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1
      - run: ./deploy.sh
```

When this job runs, GitHub pauses execution right before the job starts and shows a banner: "Waiting for approval." The configured reviewers get a notification and must explicitly approve in the Actions UI before `deploy.sh` executes — this is the standard mechanism behind "production deploys require a human sign-off" without needing a separate change-management tool bolted on.

### Combining Environments with OIDC scoping

The trust policy `sub` claim from file 1 can be scoped to `repo:org/repo:environment:production`, meaning the IAM role can **only** be assumed by a job that is bound to the `production` Environment — combining "only this Environment's approved jobs" (GitHub-side gate) with "only this exact identity claim" (AWS-side gate) for genuine defense in depth, rather than relying on either control alone.

## Points to Remember

- Self-hosted runners on EKS trade GitHub's managed convenience for elastic scaling, VPC-native access, and cost control at volume — via a controller like Actions Runner Controller (ARC).
- Ephemeral, single-job runner pods are the secure default — reused/long-lived runners risk state leaking between unrelated jobs.
- Never expose self-hosted runners to workflows triggerable by fork PRs on a public repo — that's a direct path to arbitrary code execution inside your infrastructure.
- GitHub Environments add required reviewers, wait timers, environment-scoped secrets, and branch/tag restrictions — the standard mechanism for gating production deploys with human approval.
- Environment scoping and OIDC `sub` scoping compose — bind a role's trust policy to `environment:production` for two independent, stacked access controls.

## Common Mistakes

- Running self-hosted runners with `pull_request` (not `pull_request_target`-with-review-gating) triggers on a public repo, letting any external contributor's PR execute code on your infrastructure.
- Configuring runners to persist across jobs "for speed," then discovering credentials or artifacts from one team's job are readable by the next job that happens to land on the same runner.
- Giving the runner pod's ServiceAccount or IAM role broad/cluster-admin-equivalent permissions instead of scoping to exactly the namespaces/resources the deploy actually touches.
- Forgetting to set deployment branch restrictions on a `production` Environment, so a feature branch can accidentally trigger a job bound to production secrets.
- Assuming "required reviewers" alone is sufficient access control and skipping OIDC `sub` scoping — a compromised or misconfigured workflow file could still assume an overly broad role if the trust policy isn't also tightened.
