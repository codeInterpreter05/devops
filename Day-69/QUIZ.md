# Day 69 — Quiz: Jenkins for Legacy Environments

Try to answer without looking at your notes. Answers are at the bottom.

1. What problem did "Pipeline as Code" (the Jenkinsfile) solve compared to Jenkins' original UI-configured jobs?
2. What's the actual difference between declarative and scripted pipelines — is it a different execution engine?
3. When would you need a `script { }` block inside a declarative pipeline?
4. Why is `credentials('id')` inside `environment { }` safer than referencing a secret via string interpolation in a `sh` step?
5. What are the three directories in a Jenkins shared library repo, and what's each one for?
6. Why should you pin a shared library import to a version tag instead of a branch?
7. What real operational problem do dynamic Kubernetes agents solve compared to static Jenkins agents?
8. Why might a team deliberately avoid `privileged: true` for a Docker-build agent Pod, and what are two alternatives?
9. What is Blue Ocean, and does it change how pipelines actually execute?
10. Name three concrete things you should do in Phase 1/2 of a Jenkins-to-GitHub-Actions migration *before* touching the riskiest, most business-critical pipeline.
11. What's the GitHub Actions equivalent of a Jenkins shared library?
12. **Interview question:** Your company uses Jenkins and wants to migrate to GitHub Actions. What's your migration strategy?

---

## Answers

1. It made pipeline definitions version-controlled, code-reviewable, and reproducible from a clean Jenkins instance — versus the original UI-driven job configuration (stored as XML), which had no code review, no diffable history, and could silently drift between what the UI showed and what was actually configured.
2. No — both compile down to Groovy running on the Jenkins controller. The difference is syntax and structure: declarative imposes a fixed, opinionated shape (`pipeline { agent {} stages {} }`) with restricted logic inside `steps`, while scripted is raw, unrestricted Groovy wrapped around Jenkins pipeline steps.
3. When you need real programming logic that declarative's restricted `steps` syntax doesn't allow directly — loops, complex conditionals, dynamically building a set of parallel stages, calling custom Groovy logic. `script { }` drops you into full scripted-Groovy mode for just that section while keeping the rest of the pipeline declarative.
4. `credentials('id')` causes Jenkins to automatically mask the resulting value anywhere it appears in console/build logs. Double-quoted Groovy string interpolation (`"${env.SECRET}"`) happens before the shell even sees the string, so the secret value gets written directly into the console log in plaintext, bypassing masking entirely.
5. `vars/` — Groovy scripts that become directly callable pipeline steps (e.g., `vars/standardPipeline.groovy` callable as `standardPipeline(...)`). `src/` — regular importable Groovy classes for more complex reusable logic. `resources/` — static non-Groovy files loadable at runtime via `libraryResource()`.
6. Because pinning to a floating branch means every pipeline across every consuming repo silently picks up any change merged to that branch immediately, with no controlled rollout and no way to roll back just one team/repo independently — the exact same reasoning as avoiding `:latest` image tags or unpinned Terraform module sources.
7. Static agents sit idle (wasting compute) when there's no work, and create a hard capacity ceiling (queued builds) during peak load. Dynamic Kubernetes agents provision a fresh Pod per build on demand and tear it down afterward — elastic capacity plus a guaranteed clean environment every build, with no state leaking between jobs.
8. `privileged: true` grants the Pod broad access to the host kernel/devices, which is a real security exposure if that agent Pod is ever compromised (e.g., via a malicious dependency pulled during the build). Alternatives: Kaniko or Buildah, which can build container images without requiring a privileged Docker daemon.
9. Blue Ocean is a plugin providing a modern, visual pipeline editor and clearer stage-by-stage build visualization. It's purely a UI layer — it does not change how pipelines are defined (still Jenkinsfile-driven) or how they execute underneath.
10. Any three of: inventory and classify every existing Jenkins job by risk/complexity; start migrating the low-risk, high-repetition pipelines first (not the scariest one); run the new GitHub Actions workflow in parallel with the existing Jenkins job (without yet using it as the deploy source of truth) to compare behavior/timing before cutover; migrate shared/common pipeline logic into reusable workflows or composite actions before migrating the many individual pipelines that depend on it.
11. A **reusable workflow** (triggered via `workflow_call`) or a **composite action** — both let you centralize logic once and have many individual repo workflows call into it, the same role Jenkins shared libraries play.
12. Strong answer: "I'd treat this as a phased migration, never a big-bang cutover. First, inventory every existing Jenkins job and classify by risk and complexity, and start with the simplest, most repetitive pipelines to validate the new tooling cheaply. Before migrating individual application pipelines, I'd port shared Jenkins-library logic into GitHub Actions reusable workflows or composite actions, so I'm not re-deriving the same build+scan+deploy boilerplate across every repo. For each migrated pipeline, I'd run it in parallel with the existing Jenkins job for a few weeks — comparing outputs and timings — before making GitHub Actions the source of truth for deploys, which catches subtle differences like default shell behavior or secret injection quirks early. Secrets migration is also a chance to move off long-lived static credentials onto OIDC-based short-lived cloud credentials rather than just copying old values into GitHub Secrets. Finally, I'd decommission Jenkins infrastructure only after the most complex, highest-risk pipelines have proven stable on the new system — keeping Jenkins available as a rollback path until then." Mention that some Jenkins plugin behaviors don't map 1:1 and may need to be redesigned rather than directly translated.
