# Day 68 — Policy as Code: OPA and Rego Fundamentals

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** DevSecOps

## Brief

Every organization eventually hits the same wall: "we have rules about what's allowed to run/deploy/exist, but they live in a wiki page nobody reads." Policy as Code is the fix — encoding those rules as version-controlled, testable, automatically-enforced logic instead of tribal knowledge. **Open Policy Agent (OPA)** is the general-purpose engine behind most of this in the cloud-native world, and its query language, **Rego**, is what you actually write policies in. Understanding OPA/Rego is foundational before Gatekeeper, Kyverno, or Conftest make sense — they're all just OPA wearing a different integration on top.

This day is split into three focused files:

1. **This file** — what OPA is, its architecture, and Rego language fundamentals.
2. **[02-README-Gatekeeper-Kyverno.md](02-README-Gatekeeper-Kyverno.md)** — Gatekeeper (OPA for Kubernetes admission) vs. Kyverno as a native alternative.
3. **[03-README-Common-Policies-Conftest.md](03-README-Common-Policies-Conftest.md)** — concrete common policies and Conftest for testing IaC.

## What OPA actually is

OPA is a **general-purpose policy engine**, decoupled from any specific system. It doesn't know anything about Kubernetes, Terraform, or your API gateway by itself — you give it two things: **input** (JSON — a Kubernetes admission request, a Terraform plan, an API request, anything) and **policy** (written in Rego), and it evaluates the policy against the input and returns a decision. This decoupling is the whole point: the same policy engine can gate Kubernetes deployments, Terraform applies, API authorization, and CI pipeline steps, because to OPA, it's all just "JSON in, decision out."

```
        ┌─────────────┐
Input →│             │
(JSON)  │  OPA engine  │→ Decision (JSON: allow/deny + reasons)
Policy →│  (Rego)      │
(.rego) │             │
        └─────────────┘
```

In Kubernetes specifically, OPA is deployed as an **admission webhook** (via Gatekeeper — see file 2): the API server calls out to OPA on every `create`/`update` request, passing the object as `input`, and OPA's policy decides whether to allow it.

## Rego — the language, from first principles

Rego is a **declarative, logic-based query language** (descended from Datalog), not an imperative scripting language. This trips people up constantly: you don't write "if X then deny," you write rules that describe *when a condition holds true*, and OPA evaluates all of them.

### A minimal policy

```rego
package kubernetes.admission

import future.keywords.in

# Deny if any container has no resource limits set
deny[msg] {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not container.resources.limits
    msg := sprintf("Container '%s' has no resource limits", [container.name])
}
```

Key things to internalize:
- `package kubernetes.admission` — namespaces the rule; policies are organized into packages, referenced elsewhere as `data.kubernetes.admission.deny`.
- `deny[msg] { ... }` — this defines a **set** called `deny`. Every combination of variable bindings that satisfies the body inside `{ }` adds one more `msg` to the set. If the body can't be satisfied, nothing is added — `deny` ends up empty, which callers interpret as "no violations."
- `container := input.request.object.spec.containers[_]` — the `[_]` is **iteration**: this line doesn't pick one container, it generates a new logical branch for *every* container in the array, and the rest of the body runs against each one independently. This is the single most important mental shift coming from imperative languages: Rego doesn't have a `for` loop — iteration happens by *unification* over multiple matches.
- `not container.resources.limits` — Rego uses **implicit AND**: every line in the body must be true (a conjunction). There's no explicit `&&`.
- Rules with **no matching bindings simply don't fire** — there's no `else` needed for "no violation," an empty result *is* "no violation." This is different from most languages where you'd need explicit handling of the "nothing matched" case.

### Testing a policy locally

```bash
# opa eval: run a query against a policy file and some input, no server needed
opa eval --data policy.rego --input input.json "data.kubernetes.admission.deny"

# opa run REPL for interactive exploration
opa run policy.rego
> data.kubernetes.admission.deny
```

### Rego unit tests

```rego
package kubernetes.admission

test_deny_missing_limits {
    count(deny) > 0 with input as {
        "request": {
            "kind": {"kind": "Pod"},
            "object": {"spec": {"containers": [{"name": "app"}]}}
        }
    }
}

test_allow_with_limits {
    count(deny) == 0 with input as {
        "request": {
            "kind": {"kind": "Pod"},
            "object": {"spec": {"containers": [{"name": "app", "resources": {"limits": {"cpu": "500m"}}}]}}
        }
    }
}
```

```bash
opa test . -v
```

Rego's built-in `with input as {...}` syntax lets you override the input for a test case inline — you don't need a separate JSON file per test, which keeps policy + tests colocated and fast to iterate on.

## Points to Remember

- OPA is a general-purpose, decoupled policy engine: input (JSON) + policy (Rego) → decision. It has no built-in knowledge of Kubernetes, Terraform, or anything else — that context comes entirely from what you feed it as input.
- Rego is declarative, not imperative — you describe conditions under which something is true; there's no `for` loop, no explicit `if/else` for the "nothing matched" case.
- `rule[x] { ... }` builds a **set**; iterating over an array with `[_]` creates a new logical branch per element rather than looping — this is the biggest conceptual jump from traditional programming languages.
- Rego bodies are an implicit AND (conjunction) of every line — no explicit `&&` operator needed.
- `opa eval`, `opa run` (REPL), and `opa test` let you develop and validate policies entirely locally, no cluster or webhook required, before ever wiring them into an admission controller.

## Common Mistakes

- Writing Rego like an imperative language — trying to force a `for` loop or explicit early-return logic instead of leaning into unification/iteration via `[_]` and multiple rule bodies. This produces confusing, overly verbose Rego that fights the language.
- Forgetting that an empty `deny` set means "allow" — writing a rule that's supposed to allow-list something but structuring it as a `deny` rule that never gets satisfied, so everything silently passes regardless of intent.
- Not namespacing packages clearly, leading to naming collisions when multiple teams' policies get loaded into the same OPA instance (`data.kubernetes.admission.deny` vs. two different teams both writing to `data.deny` and overwriting each other's evaluation context).
- Testing policies only after wiring them into a live admission webhook — skipping local `opa eval`/`opa test` iteration, which is far faster to develop against and doesn't require a cluster round-trip for every change.
- Assuming Rego's `==` does deep structural equality the same way as JSON.stringify comparison in JS — it does, for the most part, but ordering-sensitive comparisons (e.g., comparing unordered sets represented as arrays) can produce surprising false negatives; use Rego's actual `set` type when order shouldn't matter.
