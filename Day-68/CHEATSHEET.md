# Day 68 — Cheatsheet: Policy as Code

## OPA CLI

```bash
opa eval --data policy.rego --input input.json "data.pkg.deny"   # one-shot evaluation
opa run policy.rego                                               # interactive REPL
opa test . -v                                                     # run *_test.rego unit tests
opa fmt -w policy.rego                                             # auto-format Rego
```

## Rego syntax essentials

```rego
package mypackage             # namespace, referenced as data.mypackage.*

import future.keywords.in     # enables `in` keyword syntax

deny[msg] {                   # builds a SET; each satisfying binding adds one msg
    input.kind == "Pod"       # implicit AND between lines
    x := input.spec.containers[_]   # [_] = iterate every element (no for-loop)
    not x.resources.limits    # negation
    msg := sprintf("bad: %s", [x.name])
}

count(deny) > 0                # check violations exist
count(deny) == 0                # check no violations

test_something {               # unit test convention: test_ prefix
    count(deny) > 0 with input as {"kind": "Pod", ...}
}
```

## Gatekeeper

```bash
kubectl get constrainttemplates
kubectl get constraints
kubectl get k8srequiredlabels        # example: list instances of a constraint kind
kubectl describe <constraint-kind> <name>   # see violations
```
```yaml
spec:
  enforcementAction: dryrun   # or: deny (default is deny if omitted)
```

## Kyverno

```bash
kubectl get clusterpolicy
kubectl get policyreport -A          # audit-mode findings across the cluster
kyverno apply policy.yaml --resource pod.yaml   # test a policy against a manifest, offline
```
```yaml
spec:
  validationFailureAction: Audit     # or: Enforce
  background: true                  # also scan pre-existing resources
  rules:
    - name: rule-name
      match:
        any: [{ resources: { kinds: ["Pod"] } }]
      validate:
        message: "..."
        pattern: { ... }             # or: deny.conditions for logic-based checks
      mutate:
        patchStrategicMerge: { ... } # auto-fix instead of/alongside validate
```

## Conftest

```bash
conftest test input.json --policy policy/        # test any structured input
conftest test deployment.yaml --policy policy/    # K8s YAML directly
conftest test Dockerfile --policy policy/          # Dockerfiles directly
conftest test tfplan.json --policy policy/ -o json # machine-readable output

# Terraform-specific flow: plan must be converted to JSON first
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policy/
```

## Common policy patterns (Kyverno pattern syntax)

```yaml
image: "!*:latest"                 # disallow latest tag
resources:
  limits:
    memory: "?*"                    # "?*" = must be present, any value
    cpu: "?*"
securityContext:
  =(privileged): "false"            # "=()" = if field present, must equal this
labels:
  team: "?*"
  cost-center: "?*"
```
