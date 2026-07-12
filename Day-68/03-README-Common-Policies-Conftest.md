# Day 68 — Policy as Code: Common Policies and Conftest for IaC

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** DevSecOps

## Brief

Policy engines are only as useful as the actual policies written for them — this file covers the small set of guardrails that show up in nearly every production Kubernetes cluster (no `:latest` tags, mandatory labels, resource limits, no privileged pods), and **Conftest**, which brings the same OPA/Rego approach to Infrastructure as Code (Terraform plans, Dockerfiles, Kubernetes YAML, Helm output) *before* anything is even applied to a cluster or cloud account. This is the "shift policy left even further" move: catching a violation at `terraform plan` time in CI is strictly better than catching it after `terraform apply` already created a non-compliant resource.

## The five policies you'll write in almost every cluster

**1. No `:latest` tag**
```yaml
# Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-tag
      match:
        any: [{ resources: { kinds: ["Pod"] } }]
      validate:
        message: "Images must not use the ':latest' tag or be untagged."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```
Why this matters: `:latest` is a mutable pointer (as covered Day 67) — a Pod that restarts and re-pulls `:latest` can silently come up running completely different code than what was originally deployed and tested. It also breaks rollback: you can't roll back to "the previous `:latest`" because there's no record of which digest that was.

**2. Require team/ownership labels**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-label
      match:
        any: [{ resources: { kinds: ["Deployment", "Pod"] } }]
      validate:
        message: "The 'team' and 'cost-center' labels are required."
        pattern:
          metadata:
            labels:
              team: "?*"
              cost-center: "?*"
```
Why: without ownership labels at admission time, a cost/incident review six months later has no reliable way to answer "whose workload is this and who do we page?" — retrofitting labels onto years of existing resources is far more painful than enforcing them from day one.

**3. Resource limits required** (shown fully in file 2's Kyverno example) — prevents a single misbehaving Pod from starving a node of CPU/memory and taking down unrelated workloads scheduled on the same node (noisy-neighbor problem).

**4. Block privileged pods**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-privileged-containers
      match:
        any: [{ resources: { kinds: ["Pod"] } }]
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            =(securityContext):
              =(privileged): "false"
            containers:
              - =(securityContext):
                  =(privileged): "false"
```
Why: a privileged container has essentially unrestricted access to the host — it can access all host devices, bypass most container isolation, and is a direct path to full node (and often cluster) compromise if the container is ever exploited. This is consistently rated one of the highest-severity Kubernetes misconfigurations.

**5. Block `hostPath` volumes**
Related to #4 — a Pod mounting a `hostPath` volume (especially something like `/` or `/var/run/docker.sock`) can read/write arbitrary files on the node or escape the container entirely. Both #4 and #5 target the same underlying risk: container escape to host compromise.

## Conftest — the same idea, applied to Infrastructure as Code

Conftest takes structured config (Terraform plan JSON, Dockerfiles parsed into JSON/HCL, Kubernetes YAML, Helm chart output) and runs Rego policies against it — completely independent of any live cluster or cloud API call. This lets you catch violations at **CI time on a pull request**, before `terraform apply` ever touches real infrastructure.

```rego
# policy/terraform.rego
package main

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.after.acl == "public-read"
    msg := sprintf("S3 bucket '%s' must not be publicly readable", [resource.address])
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group_rule"
    resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
    resource.change.after.from_port == 22
    msg := sprintf("Security group rule '%s' opens SSH (22) to the entire internet", [resource.address])
}
```

```bash
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policy/
```

### CI pipeline example — block a Terraform apply on policy violations

```yaml
jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - run: terraform init
      - run: terraform plan -out=tfplan.binary
      - run: terraform show -json tfplan.binary > tfplan.json

      - name: Install Conftest
        run: |
          wget https://github.com/open-policy-agent/conftest/releases/download/v0.55.0/conftest_0.55.0_Linux_x86_64.tar.gz
          tar xzf conftest_0.55.0_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin

      - name: Run policy checks
        run: conftest test tfplan.json --policy policy/  # non-zero exit fails the job
```

Conftest also directly supports Kubernetes YAML and Dockerfiles as input formats, so the *exact same* Rego skills used for Terraform apply cleanly to `conftest test deployment.yaml --policy policy/k8s/` or `conftest test Dockerfile --policy policy/docker/` — this cross-format reuse is Conftest/OPA's biggest structural advantage over tool-specific linters that only understand one input format each.

## Points to Remember

- The five canonical cluster policies — no `:latest`, required labels, resource limits, no privileged pods, no `hostPath` — target two failure modes: unpredictable/untraceable deployments (tags, labels) and container-escape/host-compromise risk (privileged, hostPath).
- `:latest` is dangerous specifically because it's a *mutable* pointer — a restarted Pod re-pulling `:latest` can silently run different code than what was tested, and there's no clean "previous version" to roll back to.
- Conftest applies OPA/Rego to **static config** (Terraform plan JSON, Kubernetes YAML, Dockerfiles) with no live cluster/cloud dependency — this is how you gate `terraform apply` in CI *before* infrastructure is created, not after.
- The Terraform workflow is always: `terraform plan` → `terraform show -json` (convert plan to JSON) → `conftest test` against that JSON — Conftest doesn't read `.tf` HCL directly for plan-level checks, it reads the JSON plan output.
- One Rego skillset reused across Kubernetes admission (Gatekeeper), CI-time IaC gating (Conftest), and potentially API authorization — this cross-system reuse is OPA's core structural advantage over policy tools that are scoped to a single system.

## Common Mistakes

- Writing a policy that blocks privileged containers but forgetting `hostPath` volumes (or vice versa) — both are container-escape vectors and are usually paired, but teams often address only the more famous one (`privileged: true`) and miss the equally dangerous `hostPath` mount.
- Running Conftest against raw `.tf` files expecting Terraform-plan-level detail (e.g., computed values, provider defaults) — you need the *plan* JSON (`terraform show -json tfplan.binary`), not the source HCL, to see what will actually be created/changed.
- Enforcing required labels only at the Kubernetes admission layer and not also validating them in the Terraform/CI layer for resources that provision the label-bearing objects (e.g., an EKS node group's tags) — leaving gaps between IaC-created and directly-`kubectl apply`d resources.
- Treating "no `:latest` tag" as sufficient on its own without pairing it with image signing/digest pinning (Day 67) — a specific version tag (`:v1.2.3`) can *also* be re-pushed to point at different content; only a digest is truly immutable.
- Writing Conftest policies that only run in CI and never enforcing the equivalent at admission time (or vice versa) — a resource created outside the gated CI path (manual console change, `kubectl apply` bypassing the pipeline) slips through with no check at all. Defense in depth means checking at both the IaC/CI layer and the runtime/admission layer.
