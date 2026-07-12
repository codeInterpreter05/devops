# Day 42 — Review: Terraform State Debugging

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Review | **Flag:** 📌 Review/consolidation day

## Brief

Terraform state is where every "it worked in the plan but broke in the apply" and "why does Terraform want to destroy and recreate this resource I didn't touch" incident ultimately gets diagnosed. This review file consolidates the debugging muscle you need for a live "Terraform state debugging exercise" — a common practical-interview format where you're handed a broken/drifted state and asked to reason through it live, not just recite what state is.

## What state actually is, and why debugging it is different from debugging code

Terraform state (`terraform.tfstate`) is Terraform's **only** record of the mapping between your HCL resource blocks and the real-world infrastructure objects they correspond to (an AWS instance ID, a security group ID, etc.). Unlike application code bugs, a state problem usually isn't "the logic is wrong" — it's "Terraform's belief about reality has diverged from actual reality," and the fix is almost always about **reconciling belief with reality**, not changing code first.

## The standard diagnostic sequence

1. **`terraform plan`** — always step one. Read the diff carefully: is Terraform proposing to *modify*, *destroy and recreate*, or *just re-tag* a resource? Each implies a different root cause.
2. **`terraform state list`** — see everything Terraform currently believes it manages, and cross-check against what you expect to exist.
3. **`terraform state show <resource.address>`** — dump the full recorded attributes for one resource, compare against the real object (via cloud console/CLI) to spot exactly which attribute diverged.
4. **`terraform refresh`** (or, in modern Terraform, `terraform plan -refresh-only`) — reconcile state with the real infrastructure's *current* values, without changing any actual infrastructure — this alone often resolves "phantom diffs" caused by out-of-band changes (someone changed a tag in the console).
5. **Check for drift specifically** — did someone modify this resource manually (console click-ops) outside Terraform? `-refresh-only` plans surface this directly by showing what would update in state alone.

## Common root causes and their fixes

### "Terraform wants to destroy and recreate a resource I didn't touch"

Usually one of:
- **A force-new attribute changed** — some resource arguments (e.g., an EC2 instance's `availability_zone`, or many `ami` changes) cannot be updated in place; changing them forces `destroy` then `create`. Check `terraform plan`'s `-/+` annotations and the provider docs for which specific argument is force-new.
- **Provider version drift** — upgrading a provider version can change a resource's internal schema/defaults, causing Terraform to see a difference that didn't exist before, even though you changed nothing in your HCL.
- **A dependent resource's output changed**, cascading into a forced replacement (e.g., a security group's ID changed because *it* was recreated, and every resource referencing that ID as an argument gets flagged too).

Fix: identify the exact triggering attribute from the plan output, decide if the change (and resulting replacement) is actually intended, and if not, either revert the triggering change or use `lifecycle { ignore_changes = [...] }` if the drift is expected and acceptable to ignore going forward.

### "Someone manually changed something in the console, now plan looks wrong"

```bash
terraform plan -refresh-only
terraform apply -refresh-only    # updates STATE ONLY to match reality, no infra change
terraform plan                   # now shows the real diff, cleanly, against reconciled state
```

This is the correct sequence — `-refresh-only` explicitly separates "sync state to reality" from "change reality to match code," which prevents accidentally reverting someone's manual (possibly urgent, undocumented) emergency fix the moment you next run `apply`.

### "State is locked and no one can run apply"

Terraform uses a **state lock** (via DynamoDB for S3 backends, or built into other backends) to prevent two concurrent applies from corrupting state simultaneously. A lock that never releases usually means a previous run crashed/was killed mid-apply without releasing it.

```bash
terraform force-unlock <LOCK_ID>
```

This should only be used after confirming **no other apply is genuinely still running** — force-unlocking while another process is actually mid-write is exactly the corruption scenario the lock exists to prevent.

### "Resource exists in the cloud but not in state" (or vice versa)

- **Exists in cloud, not in state** — commonly from a resource created manually, or from a previous `terraform state rm`/partial apply. Fix: `terraform import <resource.address> <cloud-id>` to bring it under management without recreating it.
- **Exists in state, not in cloud** (someone deleted it manually, or it failed to actually create despite state believing it did) — `terraform plan` will show Terraform wanting to create it fresh; if that's undesired, `terraform state rm <resource.address>` removes the stale state entry without touching real infrastructure.

### Corrupted or manually-edited state

State is JSON and, in a true emergency, can be hand-edited — but this is a last resort, and always preceded by a backup:

```bash
terraform state pull > backup.tfstate
# ... make careful surgical edits, or use `terraform state mv`/`state rm` instead of raw editing when possible ...
terraform state push backup.tfstate   # only if you had to hand-edit; prefer state subcommands over raw edits
```

`terraform state mv` is the safe, supported way to handle refactors (renaming a resource, moving it into a module) without a destroy/recreate — it updates the state mapping directly, leaving real infrastructure completely untouched.

## Points to Remember

- State debugging is about reconciling Terraform's *belief* with *reality*, not fixing application logic — `plan`, `state list`, `state show`, and `-refresh-only` are the core diagnostic tools, in that order.
- A forced destroy/recreate almost always traces to a force-new attribute change, a provider version bump altering schema, or a cascading dependency — the plan output's specific attribute diff tells you which.
- `-refresh-only` updates state to match reality without touching real infrastructure — always run this before a normal `apply` when you suspect out-of-band/console drift, to avoid accidentally reverting someone's manual change.
- `terraform import` brings an existing real resource under Terraform management; `terraform state rm` removes a stale state entry without touching real infrastructure; `terraform state mv` safely renames/relocates a resource in state during refactors.
- `force-unlock` should only be used after confirming no other apply is genuinely in progress — using it prematurely risks the exact concurrent-write corruption the lock mechanism exists to prevent.

## Common Mistakes

- Running a normal `terraform apply` immediately after spotting an unexpected diff, without first running `-refresh-only` to check whether the diff is really "code says X, someone else already changed reality to Y" — risking silently reverting a legitimate manual emergency fix.
- Hand-editing the state JSON file directly as a first resort, instead of using the purpose-built `terraform state mv`/`state rm`/`import` subcommands, which are far less error-prone and don't require you to get the JSON schema exactly right.
- Force-unlocking state reflexively when an apply "seems stuck," without first confirming via CI logs/team communication that no other apply process is genuinely still running.
- Treating every forced replacement as a Terraform bug, rather than checking the specific provider documentation for which argument is marked "Forces new resource" — most of these are working as documented, not a defect.
- Forgetting to back up state (`terraform state pull > backup.tfstate`) before any manual state surgery — a mistake mid-edit with no backup can turn a minor drift issue into a much larger, harder-to-recover incident.
