# Day 59 — Phase 1 Mock Interview Day: Debugging K8s & Terraform State Scenarios

**Phase:** 1 – Core DevOps | **Week:** W9 | **Domain:** Review | **Flag:** 📌

## Brief

The "hands-on/scenario" portion of a real interview is where memorized theory gets separated from actual operational instinct. Interviewers describe a broken system verbally (or show you a terminal) and watch *how you investigate* — the order you check things in, whether you form a hypothesis before acting, and whether you narrate your reasoning instead of guessing silently. This file walks through the two scenario types named for today: a debugging Kubernetes scenario, and a Terraform state problem scenario — both as talk-tracks you should be able to reproduce fluently, out loud, in a mock interview.

## Scenario 1 — "A Deployment's pods are stuck in `CrashLoopBackOff`. Walk me through your debugging process."

A strong live answer follows a **narrowing funnel**, narrated as you go — this is the format to rehearse:

1. **"First, I'd look at the pod's status and recent events"** — `kubectl get pods -n <ns>` (confirm the restart count and exact status), then `kubectl describe pod <pod> -n <ns>` and read the **Events** section at the bottom first — it's the fastest signal (image pull errors, failed liveness/readiness probes, OOMKilled, failed volume mounts all show up here explicitly, often before you'd find them any other way).

2. **"Then I'd check the container's actual logs from before it crashed"** — `kubectl logs <pod> -n <ns> --previous` (the `--previous` flag is the detail that separates people who've actually debugged a crash loop from people reciting `kubectl logs`; the *current* container instance may not have logged the failure yet, or may be too new — the previous, crashed instance's logs usually have the real stack trace/error).

3. **"If the logs don't explain it, I'd check whether it's an application-level crash or an infra-level issue"** — `kubectl describe pod` again for `OOMKilled` (exit code 137 = SIGKILL, very often from a memory limit being hit — check `resources.limits.memory` against actual usage via `kubectl top pod`), versus a non-zero application exit code (application bug, bad config, missing environment variable/Secret).

4. **"I'd check whether this is isolated to one pod or affects the whole Deployment"** — if only one replica is crash-looping and others are healthy, that points toward node-specific or data-specific issues (a corrupted volume, a node with resource pressure) rather than a bad image/config affecting all replicas equally.

5. **"Finally, I'd check for a recent change that correlates"** — `kubectl rollout history deployment <name> -n <ns>`, and check whether this started right after a deploy (bad image, bad config change) versus happening to a long-stable Deployment (something external changed — a dependency's API, a Secret rotated/expired, a quota hit).

**Why narrating matters as much as the technical steps:** an interviewer explicitly wants to hear you form and test hypotheses out loud ("if it's OOMKilled, I'd expect exit code 137 and I'd check memory limits next") rather than listing commands with no reasoning connecting them — the narration is the actual signal being evaluated, not just whether you know `kubectl logs --previous` exists.

## Scenario 2 — "Terraform apply is failing with a state-related error. Walk me through it."

The most common concrete framings and the reasoning behind each:

- **"Error: resource already exists"** — someone (or another process) created the real resource outside Terraform, or a previous `apply` partially succeeded and the state wasn't updated to match (e.g., it crashed mid-run). Talk-track: *"I wouldn't force-recreate — first I'd check if the resource is already correctly tracked with `terraform state list`, and if not, I'd `terraform import` it rather than risk destroying something that might already be serving production traffic."*

- **"Error: state lock could not be acquired"** — another `apply`/`plan` is running concurrently, or a previous run crashed and left a stale lock (common with a killed CI job). Talk-track: *"First I'd confirm no other run is genuinely in progress — check the CI system, ask the team — before ever force-unlocking, because `terraform force-unlock` bypasses the exact safety mechanism that prevents concurrent-write corruption. Only after confirming it's a stale lock from a crashed process would I run `terraform force-unlock <lock-id>`."*

- **"Plan shows an unexpected destroy/recreate of a critical resource (e.g., a database) after a trivial-looking HCL change"** — almost always caused by a change to an immutable/force-new attribute (e.g., an RDS engine version bump that isn't in-place upgradeable, a resource rename without `state mv`, a change to a tag that's part of a resource's identifying key for some providers). Talk-track: *"I would never apply a plan that proposes destroying a stateful resource without understanding exactly why first — I'd check the plan's `-/+` reasoning field, cross-reference the provider's docs for which attributes force replacement, and if it's a rename, fix it properly with `terraform state mv` instead of letting it destroy and recreate."*

- **"State file is out of sync with real infrastructure (drift)"** — someone made a manual console change. Talk-track: *"I'd run `terraform plan` to see the full drift, discuss with the team whether the manual change was intentional, and either import/update Terraform to reflect the intentional change, or apply to revert the drift back to the declared state — the wrong move is applying blindly without understanding why the drift happened, since that could undo a legitimate emergency fix someone made by hand."*

**What this scenario tests, structurally:** whether your default instinct under a scary-looking Terraform error is to force through it (`-auto-approve`, `force-unlock`, blindly re-applying) or to pause, understand the actual state/reality mismatch, and choose the least destructive path — this reflects real production judgment far more than knowing every CLI flag.

## Points to Remember

- For Kubernetes crash-loop debugging: Events section of `describe pod` first, then `logs --previous`, then correlate with `kubectl top pod` (OOM) and rollout history — narrate each step as a hypothesis, not a memorized checklist.
- Exit code 137 = SIGKILL, very commonly from a memory limit (OOMKilled); worth stating explicitly in an interview since it shows you know *why* the number means something, not just that you've seen it before.
- For Terraform state errors, the unifying good instinct is: pause, understand the real-vs-state mismatch, prefer `import`/`state mv` over force/destroy, and treat `force-unlock` as a last resort only after confirming no run is genuinely still in progress.
- An unexpected destroy-and-recreate in a plan almost always traces to either an immutable/force-new attribute change or a missing `state mv` after a rename/refactor — know both causes cold.
- What's being scored in these scenarios is the reasoning process and hypothesis-forming out loud, not just arriving at the right final answer.

## Common Mistakes

- Jumping to `kubectl logs` without `--previous` on a crash-looping pod and getting an empty/unhelpful result, then being stuck because the useful logs were in the terminated container instance.
- Not checking the Events section of `kubectl describe pod` first — it frequently states the actual problem (`OOMKilled`, `ImagePullBackOff`, `FailedScheduling`) directly, saving several debugging steps.
- Reflexively running `terraform force-unlock` or `-auto-approve` the moment a scary error appears, without first confirming whether a concurrent run is genuinely active or whether a plan's destructive action is actually intended.
- Treating every Terraform "must replace" warning as something to just accept and apply, instead of checking whether it's an avoidable `state mv` situation.
- In an interview, describing the fix without ever stating the underlying "why" (e.g., saying "I'd run `--previous`" without explaining that the crashed instance's logs are what you actually need) — the reasoning is the signal, not just the correct command name.
