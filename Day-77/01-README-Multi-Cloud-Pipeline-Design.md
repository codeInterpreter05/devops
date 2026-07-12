# Day 77 — Multi-cloud CI/CD: Pipeline Design & Cloud-Agnostic Tooling

**Phase:** 2 – CI/CD & Security | **Week:** W12 | **Domain:** CI/CD

## Brief

"We're multi-cloud" is one of the most misused phrases in infrastructure conversations — it can mean anything from "we run one workload on GCP because a team acquired it in an acquisition" to a genuinely engineered active-active deployment across AWS and GCP with automated failover. Understanding how to actually *build* a pipeline that deploys to more than one cloud provider — and, more importantly, when that engineering investment is justified — is what separates a credible answer to a multi-cloud interview question from buzzword-matching. Today's hands-on activity (deploying the same minimal app to both AWS Lambda and GCP Cloud Run from one pipeline) is a small, concrete example of the pattern at the core of any real multi-cloud setup.

This day is split into three files:

1. **This file** — designing a single pipeline that deploys to multiple cloud providers, and the cloud-agnostic tooling choices that make it feasible.
2. **[02-README-Terraform-Workspaces-Multicloud.md](02-README-Terraform-Workspaces-Multicloud.md)** — using Terraform workspaces and provider blocks to manage multi-cloud infrastructure from one codebase.
3. **[03-README-Cost-Repatriation.md](03-README-Cost-Repatriation.md)** — the real cost of multi-cloud and when repatriation (moving back to fewer providers) is the right call.

## One pipeline, multiple cloud targets: the structural approach

The key design decision is where provider-specific logic lives. A pipeline that deploys to both AWS and GCP should have a shared "what to deploy" definition, with provider-specific "how to deploy it" steps kept isolated and parallel — not interleaved:

```yaml
# .github/workflows/multi-cloud-deploy.yml
name: Multi-Cloud Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myregistry/myapp:${{ github.sha }}

  deploy-aws-lambda:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/gha-deploy-lambda
          aws-region: us-east-1
      - name: Deploy to Lambda
        run: |
          aws lambda update-function-code \
            --function-name myapp \
            --image-uri myregistry/myapp@${{ needs.build.outputs.digest }}

  deploy-gcp-cloudrun:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/gha-pool/providers/gha-provider
          service_account: gha-deploy@myproject.iam.gserviceaccount.com
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy myapp \
            --image=myregistry/myapp@${{ needs.build.outputs.digest }} \
            --region=us-central1 \
            --platform=managed
```

Two structural choices matter here:

- **A single `build` job produces one artifact (by digest) that both cloud-specific deploy jobs consume** — you build once, deploy to N targets, rather than rebuilding per cloud (which risks subtle drift between what's running on AWS vs. GCP if the builds aren't byte-identical).
- **`deploy-aws-lambda` and `deploy-gcp-cloudrun` run in parallel** (both only `need: build`), each authenticating via that cloud's own OIDC federation mechanism (`aws-actions/configure-aws-credentials` with `role-to-assume`, `google-github-actions/auth` with Workload Identity Federation) — no long-lived static credentials for either cloud, following the same least-privilege principle from Day 73's signing job.

## Cloud-agnostic tooling: what actually helps, and what's a false promise

**Genuinely cloud-agnostic tools:**
- **Terraform** — a single HCL codebase can target AWS, GCP, Azure, and dozens of others via different providers, with a consistent workflow (`plan`/`apply`) regardless of target. This is the most mature and widely adopted piece of the multi-cloud tooling story (see file 2).
- **Kubernetes** — if your compute runs in Kubernetes (EKS, GKE, AKS, or self-managed), the workload-level manifests (Deployments, Services) are largely portable; only the cluster-provisioning layer and certain integrations (load balancer annotations, storage classes, IAM-to-K8s-RBAC bridges like IRSA vs. Workload Identity) differ per cloud.
- **OpenTelemetry** — vendor-neutral observability instrumentation that can export to any backend (CloudWatch, Cloud Monitoring, Datadog, etc.) without changing application code.

**Tools/abstractions that overpromise:**
- **"Cloud-agnostic" PaaS abstractions** that try to unify serverless compute (Lambda vs. Cloud Functions vs. Azure Functions) behind one API often end up exposing only the lowest-common-denominator feature set, forcing you to avoid provider-specific capabilities (e.g., Lambda's provisioned concurrency, or Cloud Run's request-based autoscaling nuances) that you're likely paying for and would benefit from using directly.
- **Multi-cloud Kubernetes "meta-platforms"** that claim to abstract away which cloud you're on entirely — in practice, storage, networking, and IAM integration differences leak through constantly, and debugging becomes harder because you're now troubleshooting through an extra abstraction layer on top of already-complex per-cloud primitives.

The realistic take: **Terraform for provisioning and Kubernetes for workload portability get you real, practical multi-cloud leverage. Trying to abstract away every cloud-specific service (serverless compute, managed databases, IAM) behind a unified API generally costs more in lost functionality and debugging complexity than it saves.**

## Points to Remember

- Structure a multi-cloud pipeline as one shared build producing a single artifact (referenced by digest), fanned out to parallel, provider-specific deploy jobs — not per-cloud rebuilds or interleaved provider logic.
- Use each cloud's native OIDC/workload-identity federation (`aws-actions/configure-aws-credentials`, `google-github-actions/auth`) rather than long-lived static credentials stored per cloud — the same least-privilege principle applies across every provider you add.
- Terraform and Kubernetes are the tools that deliver real cross-cloud leverage today; unified serverless/PaaS abstraction layers tend to cost more in lost feature access and debugging complexity than they save.
- "Multi-cloud" as a marketing phrase covers a huge range of actual engineering investment — always ask what specifically is deployed where, and why, before assuming a sophisticated active-active setup exists.
- Building once and deploying the same artifact to multiple clouds avoids drift between what's actually running on each provider.

## Common Mistakes

- Rebuilding the artifact separately for each cloud target instead of building once and deploying the same digest everywhere — this risks subtle behavioral drift between clouds if the build environments aren't perfectly identical.
- Storing long-lived static IAM keys/service-account JSON for each cloud in CI secrets instead of using OIDC federation for both — doubling (or N-ing) your long-lived-credential attack surface as you add cloud providers.
- Adopting a "universal" serverless abstraction layer expecting full feature parity across providers, then discovering key features (concurrency controls, specific trigger types, IAM integration nuances) aren't actually exposed by the abstraction.
- Interleaving cloud-specific logic within a single job/script (e.g., one big bash script with `if [[ $CLOUD == aws ]]` branches) instead of separate, parallel jobs — this makes failures harder to attribute and prevents the two deployments from running concurrently.
- Treating "we use Kubernetes so we're cloud-agnostic" as a fully solved problem — cluster provisioning, storage classes, load-balancer integration, and IAM-to-RBAC bridges (IRSA on EKS vs. Workload Identity on GKE) still differ meaningfully per cloud and need explicit handling.
