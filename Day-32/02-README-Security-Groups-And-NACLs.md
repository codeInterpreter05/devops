# Day 32 — AWS Core: VPC & Networking: Security Groups vs. NACLs

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

"What's the difference between a Security Group and a NACL, and which is stateful?" is this day's assigned interview question, and for good reason — it's asked in nearly every AWS-adjacent interview because it's a compact way to test whether you actually understand AWS's two-layer network filtering model or just know the two names. Get this cold: the *stateful vs. stateless* distinction is the whole answer, and it has real operational consequences for how you write rules.

## Two independent layers of filtering

Every packet touching an EC2 instance in a VPC passes through **both** a NACL (at the subnet boundary) and a Security Group (at the instance/ENI boundary) — they're not alternatives, they're layered, like a building's perimeter fence (NACL) plus a lock on each individual office door (Security Group).

```
Internet → NACL (subnet boundary) → Security Group (instance/ENI boundary) → EC2 instance
```

## Security Groups — stateful, instance-level, allow-only

```bash
aws ec2 create-security-group --group-name web-sg --vpc-id vpc-0abc123
aws ec2 authorize-security-group-ingress --group-id sg-0abc123 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0
```

- **Stateful**: if you allow inbound traffic on port 443, the **response** traffic is automatically allowed back out — you never need a matching outbound rule for a connection's reply. The SG tracks connection state (source/dest IP, port, sequence) and recognizes return traffic as belonging to an already-permitted flow.
- **Allow rules only** — there is no "deny" rule in a Security Group. Everything not explicitly allowed is implicitly denied. This is a meaningfully simpler mental model than NACLs: you can't accidentally create a rule that blocks something another rule allows, because there's no blocking rule at all, just an ever-growing allow-list.
- **Evaluated per-instance (technically per-ENI)**, and **rules are additive across all attached SGs** — if an instance has three security groups attached, the effective permission set is the union of all their allow rules.
- Can reference **other security groups as the source/destination** instead of a CIDR (`--source-group sg-0xyz`), which is how you express "allow traffic from anything wearing the `app-tier` security group" without hardcoding IP ranges — critical for autoscaling groups where instance IPs change constantly.

## NACLs (Network ACLs) — stateless, subnet-level, allow AND deny

```bash
aws ec2 create-network-acl --vpc-id vpc-0abc123
aws ec2 create-network-acl-entry --network-acl-id acl-0abc123 \
  --rule-number 100 --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id acl-0abc123 \
  --rule-number 200 --protocol tcp --port-range From=22,To=22 \
  --cidr-block 0.0.0.0/0 --rule-action deny --ingress
```

- **Stateless**: NACLs do **not** track connection state — an allowed inbound request's response is a *separate* packet that must *itself* be explicitly allowed by an **outbound** rule. This is the single biggest practical gotcha: if you allow inbound 443 but forget an outbound rule for the ephemeral port range (typically 1024-65535) that the response uses, connections will appear to "hang" — the request gets in, but the reply can't get out.
- **Both allow and deny rules**, evaluated **in rule-number order, lowest first, first match wins** — unlike Security Groups' pure union-of-allows model, NACL rule *order* matters and an early deny can shadow a later allow for the same traffic.
- **Evaluated per-subnet** — every instance in that subnet is subject to the same NACL, regardless of which Security Groups are attached to it individually.
- The **default NACL** (auto-created per VPC) allows all traffic in both directions by default; a **custom NACL** denies all traffic by default until you add explicit allow rules.

## Side-by-side

| | Security Group | NACL |
|---|---|---|
| Scope | Instance/ENI | Subnet |
| State tracking | Stateful (return traffic auto-allowed) | Stateless (must explicitly allow both directions) |
| Rule types | Allow only | Allow and Deny |
| Evaluation | All rules evaluated, union of allows | Rule-number order, first match wins |
| Typical use | Day-to-day "what can talk to this instance" | Coarse subnet-wide guardrails, explicit IP blocklisting |

## Why both exist — and when NACLs actually earn their place

In practice, most teams do nearly all fine-grained access control with Security Groups and leave NACLs at their permissive defaults — SGs' stateful, allow-only, group-referencing model is simply easier to reason about day-to-day. NACLs earn their place for a narrower set of jobs Security Groups can't do at all:
- **Explicit IP-range blocklisting** — SGs have no deny rule, so if you need to hard-block a specific malicious CIDR regardless of any SG's allow rules, a NACL deny is the only native tool for it.
- **Defense-in-depth / blast-radius containment** — a subnet-wide guardrail that holds even if someone misconfigures an individual instance's Security Group (e.g., a NACL that denies all inbound from the internet on the data-tier subnet, regardless of what any future SG on any future instance in that subnet might allow).

## Points to Remember

- Security Groups are stateful and instance-scoped with allow-only rules; NACLs are stateless and subnet-scoped with both allow and deny rules evaluated in numbered order.
- "Stateful" means return traffic for an already-permitted connection is automatically allowed — this is the crux of the interview question, know it cold.
- Because NACLs are stateless, you must explicitly allow the ephemeral port range outbound (and inbound, symmetrically) or responses to otherwise-permitted requests will silently fail.
- Traffic passes through both layers for every packet — they're complementary defense-in-depth, not a choice between one or the other.
- Security Groups can reference other Security Groups as a source — essential for dynamic environments like autoscaling groups where hardcoded IPs don't work.

## Common Mistakes

- Answering "SGs are stateful, NACLs are stateless" correctly but being unable to explain *why* that matters operationally (the missing-ephemeral-port-outbound-rule symptom) — interviewers often push for this follow-up specifically.
- Adding a NACL deny rule with too high a rule number, placing it *after* an earlier, broader allow rule that already matches the same traffic — since NACLs evaluate in order and stop at first match, the deny never actually triggers.
- Forgetting that Security Group rule changes take effect immediately for both new and existing connections, while people sometimes assume (incorrectly) that only new connections are affected — actually established connections tracked by the stateful engine keep working until they end, but a newly tightened rule does apply to genuinely new connection attempts right away.
- Using NACLs as the primary access-control mechanism out of a mistaken belief they're "more secure" because they support explicit deny — for most day-to-day instance-level access control, this just adds stateless-tracking complexity without a corresponding security benefit over well-scoped Security Groups.
- Leaving the default NACL's "allow all" rules in place while believing a restrictive custom NACL is protecting a subnet — always verify which NACL is actually associated with the subnet in question.
