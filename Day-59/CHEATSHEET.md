# Day 59 — Cheatsheet: Phase 1 Mock Interview Day

## The 4-part interview shape

```
1. Behavioral / background        ~5-10 min   STAR format, concrete result at the end
2. System design                  ~20-30 min  clarify -> choose model -> mechanisms -> tradeoffs
3. Hands-on / scenario debugging  ~15-20 min  hypothesis -> command that tests it -> narrate
4. Your questions for them        ~5 min      ask about on-call, deploy frequency, postmortems
```

## STAR behavioral structure

```
Situation  - brief context, one or two sentences
Task       - what you specifically needed to solve/own
Action     - what YOU did (not "we"), the concrete steps
Result     - a number or measurable outcome (time saved, MTTR reduced, incident count)
```

## System design: clarify-first checklist

```
- How many tenants? (10s vs 1000s changes everything)
- Isolation driven by compliance, or just noisy-neighbor concerns?
- B2B (few large tenants) or B2C-at-scale (many small tenants)?
- Any regulated/compliance-bound tenants requiring stronger guarantees?
```

## EKS multi-tenancy isolation spectrum

```
namespace-per-tenant     cheapest, fastest    RBAC + NetworkPolicy + ResourceQuota
node-pool-per-tenant     stronger blast radius  taints/tolerations, node affinity
cluster-per-tenant       strongest isolation   separate control plane, highest cost/ops overhead
-> most real systems: hybrid (namespace default, dedicated tier for large/regulated tenants)
```

## Concrete isolation mechanisms to name explicitly

```
Compute:  ResourceQuota / LimitRange per namespace
Network:  default-deny NetworkPolicy between tenant namespaces
Identity: IRSA (IAM Roles for Service Accounts) — tenant workload can only reach its own resources
Admission: Pod Security Standards — block privilege escalation
Data:     per-tenant schema / row-level security / separate DB / partition-key + IAM condition keys
Control:  automated tenant onboarding (namespace+quota+RBAC+IAM created programmatically, never by hand)
```

## Crash-loop debugging talk-track order

```
1. kubectl get pods -n <ns>                          restart count, current status
2. kubectl describe pod <pod> -n <ns>                 -> read Events section FIRST
3. kubectl logs <pod> -n <ns> --previous               crashed instance's logs, not the new one
4. kubectl top pod <pod> -n <ns>                      check against resources.limits.memory (OOMKilled = exit 137)
5. kubectl rollout history deployment <name> -n <ns>  correlate with a recent change
```

## Terraform state error talk-track

```
"resource already exists"      -> terraform state list, then import (don't force-recreate)
"state lock could not be acquired" -> confirm no run is genuinely active BEFORE force-unlock
unexpected destroy/recreate    -> check for immutable/force-new attr, or missing `state mv` after rename
drift detected                 -> discuss with team before applying; could be an intentional manual fix
```

## Self-review checklist after recording

```
[ ] Behavioral answers each <= ~2-3 min, STAR structure, concrete result
[ ] System design opened with clarifying questions before naming a technology
[ ] System design covered: model choice+tradeoff, compute isolation, data isolation, control plane
[ ] Debugging scenarios stated hypothesis BEFORE command each time
[ ] Asked genuine questions in the "your turn" slot (on-call, deploy freq, postmortems)
[ ] Wrote down >= 3 specific (not vague) improvement items
```
