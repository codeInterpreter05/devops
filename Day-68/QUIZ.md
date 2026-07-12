# Day 68 — Quiz: Policy as Code

Try to answer without looking at your notes. Answers are at the bottom.

1. What does OPA take as input, and what does it know about Kubernetes/Terraform/any specific system by default?
2. In Rego, what does `deny[msg] { ... }` actually build, and what does an empty result mean for the caller?
3. How does Rego "iterate" over an array like `input.spec.containers[_]` without a `for` loop? What actually happens under the hood?
4. What's the difference between a Gatekeeper `ConstraintTemplate` and a `Constraint`?
5. Name two capabilities Kyverno has that raw Gatekeeper/OPA validation doesn't provide out of the box.
6. Why should a new admission policy be rolled out in audit/dry-run mode before enforcing mode?
7. Why do admission-control policies (Gatekeeper/Kyverno) not automatically fix or flag already-existing non-compliant resources?
8. Why is disallowing `:latest` image tags a real security/reliability control and not just a style preference?
9. What input format does Conftest need for Terraform, and what two commands produce it?
10. Give one concrete example each of a policy that prevents (a) an untraceable deployment and (b) a container-escape risk.
11. What's the biggest structural advantage of using OPA/Rego (via Conftest) for IaC policy versus a tool-specific linter?
12. **Interview question:** How do you prevent engineers from deploying pods without resource limits? Walk me through the implementation.

---

## Answers

1. OPA takes JSON as input and Rego as the policy definition, and produces a JSON decision. By default it knows nothing about any specific system (Kubernetes, Terraform, etc.) — all system-specific meaning comes purely from what's fed to it as `input` and how the Rego is written to interpret that shape.
2. It builds a **set**. Every combination of variable bindings that satisfies the rule body adds one element to the set. An empty result means no violations were found — the caller (e.g., Gatekeeper) interprets "empty `deny` set" as "allow."
3. `[_]` causes Rego to try the rest of the rule body once for every element in the array, each as a separate logical branch/unification, rather than looping imperatively. If any element satisfies the rest of the body, that produces a result (e.g., adds a message to a `deny` set); it isn't "loop through and mutate a variable" the way a `for` loop would work in an imperative language.
4. A `ConstraintTemplate` defines the reusable schema and Rego logic for a policy type (like a class/function definition). A `Constraint` is an instance of that template with actual parameters and scoping (which namespaces/kinds it applies to) supplied — write the Rego once in the template, then create many parameterized Constraints from it.
5. Mutation (auto-fixing resources, e.g., injecting default resource limits, rather than only rejecting) and generation (auto-creating related resources, e.g., a default NetworkPolicy whenever a new Namespace is created). Both are first-class, mature features in Kyverno; Gatekeeper's mutation support is more limited/newer and it has no generation feature.
6. Because a policy can have edge cases not anticipated during writing — rolling straight into enforcing mode risks blocking legitimate, already-running deployment patterns across teams the moment it's applied. Audit/dry-run mode logs what *would* be blocked against real traffic first, letting you review and adjust before anything is actually rejected.
7. Because admission webhooks only intercept new `create`/`update` requests as they pass through the API server — they have no mechanism to reach back and re-evaluate objects that were already persisted to etcd before the policy existed. A separate background audit/scan (Kyverno's `background: true`, or Gatekeeper's periodic audit) is required to detect pre-existing violations.
8. `:latest` is a mutable tag — the same tag can be re-pushed to point at entirely different image content at any time. A Pod that restarts and re-pulls `:latest` can silently start running different code than what was originally tested and deployed, and there's no clean, specific "previous version" to roll back to since the tag itself carries no version history.
9. Conftest needs the Terraform **plan in JSON format**, not raw `.tf` HCL. Produced via `terraform plan -out=tfplan.binary` followed by `terraform show -json tfplan.binary > tfplan.json`.
10. (a) Untraceable deployment: requiring `team`/`cost-center` labels, so every resource can be traced back to an owner during incident response or cost review. (b) Container-escape risk: blocking `privileged: true` containers (or blocking `hostPath` volume mounts), since both give a container near-unrestricted access to the underlying host.
11. The same Rego policy logic and skillset can be reused across multiple systems — Kubernetes admission (via Gatekeeper), Terraform/Dockerfile/K8s-YAML gating in CI (via Conftest), and even API authorization — instead of learning and maintaining a separate tool-specific policy language/config for each system individually.
12. Strong answer: "Two enforcement points: first, in CI, run `conftest test` against the Terraform plan or rendered Kubernetes manifests using a Rego policy that denies any container without `resources.limits` set, catching it before merge. Second, and more importantly for anything not going through that CI path (manual `kubectl apply`, Helm installs, etc.), deploy a Kyverno (or Gatekeeper) `ClusterPolicy` with a `validate` rule requiring `resources.limits.cpu` and `resources.limits.memory` to be present on every container, first in `Audit` mode to see what it would have caught against real traffic, then flipped to `Enforce` so any non-compliant Pod is rejected at admission time before it's ever scheduled. Also enable `background: true` scanning to surface any pre-existing Pods that were deployed before the policy existed, since the webhook itself only catches new admissions." Mention the mutate-to-auto-fix alternative (Kyverno can inject default limits instead of rejecting) as a softer rollout option.
