# Day 68 — Lab: Policy as Code

**Goal:** Write and enforce real Kyverno policies against a live cluster, and gate a Terraform plan with Conftest — the day's assigned hands-on activity, done end to end.

**Prerequisites:**
- A local Kubernetes cluster (`kind` or `minikube`).
- `kubectl`, `helm` installed.
- `opa` CLI installed (`brew install opa` or download the binary) for the Rego fundamentals lab.
- `conftest` installed (`brew install conftest` or download from GitHub releases).
- Terraform installed, with a small throwaway `.tf` config (an S3 bucket + a security group works well).

---

### Lab 1 — Rego fundamentals with `opa eval` and `opa test`

1. Create `/tmp/day68/policy.rego`:
   ```rego
   package day68

   deny[msg] {
       input.request.kind.kind == "Pod"
       container := input.request.object.spec.containers[_]
       not container.resources.limits
       msg := sprintf("Container '%s' has no resource limits", [container.name])
   }
   ```
2. Create `/tmp/day68/input.json` with a Pod admission request that has a container missing `resources.limits`.
3. Evaluate it:
   ```bash
   opa eval --data policy.rego --input input.json "data.day68.deny"
   ```
4. Add a Rego unit test (`policy_test.rego`) asserting `count(deny) > 0` for the bad input and `count(deny) == 0` for a fixed input with limits set. Run:
   ```bash
   opa test . -v
   ```

**Success criteria:** You can predict, before running it, whether `opa eval` will return an empty or non-empty `deny` set for a given input — and your unit tests pass for both the violating and compliant cases.

---

### Lab 2 — The core hands-on activity: 5 Kyverno policies

1. Install Kyverno on a local `kind` cluster:
   ```bash
   kind create cluster --name day68
   helm repo add kyverno https://kyverno.github.io/kyverno/
   helm install kyverno kyverno/kyverno -n kyverno --create-namespace
   kubectl wait --for=condition=Ready pods --all -n kyverno --timeout=120s
   ```
2. Write and apply these 5 `ClusterPolicy` resources, each in `Audit` mode first:
   - `require-resource-limits` (from README file 2)
   - `disallow-latest-tag`
   - `require-labels` (require `team` and `cost-center`)
   - `disallow-privileged`
   - a `block-hostpath` policy (block any Pod spec with a `hostPath` volume)
3. Confirm they're loaded: `kubectl get clusterpolicy`
4. Try applying a deliberately non-compliant Pod (privileged, `:latest` tag, no labels, no limits, with a `hostPath` volume) and observe the audit results:
   ```bash
   kubectl get policyreport -A
   ```
5. Flip all 5 to `validationFailureAction: Enforce` and re-apply the same bad Pod — confirm it's now rejected outright with a clear error naming which policy blocked it.

**Success criteria:** All 5 policies exist, are loaded, and — in Enforce mode — a single non-compliant Pod manifest is rejected with error messages identifying every violated rule.

---

### Lab 3 — Gate a Terraform plan with Conftest

1. In a scratch directory, write a minimal Terraform config that creates an S3 bucket with `acl = "public-read"` and a security group rule opening port 22 to `0.0.0.0/0`.
2. Write `policy/terraform.rego` with `deny` rules for both (from README file 3).
3. Run:
   ```bash
   terraform init
   terraform plan -out=tfplan.binary
   terraform show -json tfplan.binary > tfplan.json
   conftest test tfplan.json --policy policy/
   ```
4. Confirm Conftest reports both violations and exits non-zero.
5. Fix the Terraform config (private ACL, restrict the CIDR) and re-run — confirm a clean pass with exit code 0.

**Success criteria:** You have a working Conftest gate that fails on the two seeded misconfigurations and passes once they're fixed, entirely without ever running `terraform apply`.

---

### Cleanup

```bash
kind delete cluster --name day68
rm -rf /tmp/day68
# In your Terraform scratch dir:
terraform destroy -auto-approve   # only if you actually applied real cloud resources; skip if you only ran `plan`
```

### Stretch challenge

Take one of your 5 Kyverno policies and reimplement its equivalent logic as a raw Gatekeeper `ConstraintTemplate` + `Constraint` pair in Rego. Compare how many lines of YAML/Rego each approach took, and write one sentence on which felt easier to review in a PR.
