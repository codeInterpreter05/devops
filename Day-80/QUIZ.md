# Day 80 — Quiz: Phase 2 Project — Full Pipeline

Try to answer without looking at your notes. Answers are at the bottom.

1. Name the five stages of the full pipeline (GitHub → CI → ECR → Argo CD → EKS) and one concrete security control that belongs at each.
2. What actually makes a vulnerability scan a "gate" rather than just a report? Give the specific mechanism.
3. Why should CI update a manifests repo rather than run `kubectl apply`/`helm upgrade` directly against the cluster?
4. What does `selfHeal: true` do in an Argo CD `Application`'s sync policy, and why is it the property that makes Git the *actual* source of truth?
5. Why is a signed, fully-scanned container image still not sufficient on its own to guarantee a safe deployment? What closes that gap?
6. Why should image tags be immutable (by digest or strict SHA), and what question does that let you answer later?
7. In a k6 load test, why does reporting p95/p99 latency matter more than reporting the average?
8. Name three things worth watching *during* a load test run, beyond the final summary output.
9. Why is a Mermaid diagram embedded in the README generally preferable to a linked external image for an architecture diagram?
10. What is the single highest-signal section of a capstone project README, per the portfolio-audit discipline carried over from Day 60, and why?
11. If Falco or seccomp profiles add measurable latency overhead under load, is that a reason to remove them? How should you think about that tradeoff?
12. **Interview-style prompt for today (no single interview question is assigned for this capstone; treat this as your own self-check):** In under 90 seconds, walk an interviewer through your full pipeline architecture, naming where each security control is enforced and one real tradeoff you made.

---

## Answers

1. GitHub (source control, trunk-based branching, PR-gated merges) → CI (GitHub Actions: SAST, dependency scan, image scan, signing) → ECR (registry-native scan-on-push, immutable tags) → Argo CD (GitOps pull-based sync, no cluster credentials leave the cluster) → EKS (Kyverno/OPA admission control, seccomp/AppArmor, Falco runtime detection, K8s audit logging). Any one reasonable control per stage is acceptable.
2. A gate requires the check to actually fail the job/pipeline on a violation — concretely, a non-zero exit code (e.g., Trivy's `exit-code: '1'` on critical/high findings) combined with a `needs:` dependency so downstream jobs (like pushing to ECR) don't run if the scan job failed. A scan that only produces a report someone has to remember to read is not a gate.
3. Giving CI direct cluster credentials (to run `kubectl apply`/`helm upgrade`) means a compromised CI runner or leaked CI secret has direct write access to production — a large, avoidable attack surface. With GitOps, Argo CD runs inside the cluster and pulls desired state from Git; cluster credentials never need to leave the cluster boundary, and CI's blast radius is limited to "can modify a Git repo," which is separately reviewable and revertible.
4. `selfHeal: true` makes Argo CD actively detect and revert manual, out-of-band changes made directly against the cluster (e.g., via `kubectl edit`) back to what's declared in the Git repo. This is the property that makes Git the actual (not just intended) source of truth — without it, manual drift could silently diverge from Git and nothing would correct it.
5. Signing and scanning verify properties of the **image** (its content, provenance, known vulnerabilities) — they say nothing about the **pod spec** it's deployed with. A perfectly clean image can still be run privileged, as root, with no resource limits, or without a seccomp profile. Admission control (Kyverno/OPA) closes this gap by enforcing policy on the spec itself at deploy time.
6. Immutable tags prevent a tag from being silently overwritten/repointed to a different image after the fact. This lets you answer, unambiguously and auditably, "what image was actually running in production at time X" — a mutable tag (like a reused `:latest` or a reused version tag) makes that question impossible to answer reliably after the fact.
7. An average hides tail latency — a system can have a low average while a meaningful fraction of real users experience much worse (a slow p95/p99). Since real users experience individual requests, not an aggregate statistic, p95/p99 more honestly reflects what a real subset of users is experiencing, especially under spike load.
8. Any three of: HPA scale-out behavior/timing (`kubectl get hpa -w`), resource usage approaching configured limits (`kubectl top pods`), OOMKilled or CrashLoopBackOff events during the spike (`kubectl get pods -w`), and whether security tooling (Falco, seccomp) measurably adds latency overhead under load.
9. A Mermaid block is committed as plain text inside the README itself — it has no external dependency that can go stale or 404, it renders natively on GitHub with zero extra steps for a reader, and it's version-controlled and diffable like any other text change, unlike a binary image hosted elsewhere.
10. A specific, honestly-described real tradeoff or problem encountered and how it was resolved. This is the highest-signal section because it's direct, hard-to-fake evidence of engineering judgment and debugging skill — the exact thing an interviewer is trying to distinguish from "followed a tutorial," and most portfolios have none of this at all.
11. Not automatically — it's a real tradeoff to reason about explicitly, not a reason to reflexively remove the control. The right approach is to measure the actual overhead, weigh it against the security value the control provides, and either accept it, tune the profile/rule to reduce overhead, or make a documented, deliberate decision — the point of measuring it at all is so the tradeoff is made consciously and can be explained, not left as an unexamined assumption in either direction.
12. This is a self-check with no single correct answer, but a strong response follows the pipeline stage-by-stage (source → CI → registry → GitOps deploy → cluster), names the *specific enforcing mechanism* at each stage (not just "we scan for vulnerabilities" but "Trivy with `exit-code: 1` gating the push-to-ECR job"), and closes with one genuinely specific tradeoff from your own Lab 5 README — compressed to fit a realistic interview time constraint, which is itself the skill being practiced.
