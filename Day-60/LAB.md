# Day 60 — Lab: Phase 1 → Phase 2 Transition

**Goal:** Close out Phase 1 with a real gap analysis, a cleaned-up portfolio, a concrete Phase 2 mini-project plan, and a written reflection post — the assigned hands-on activity. This is a self-assessment and consolidation lab, not a new-tool lab: no new installs are required today.

**Prerequisites:** Access to your GitHub account and every repo you created across Days 1-59, this repo's own `QUIZ.md` files from each prior day, and roughly 2-3 hours of uninterrupted time.

---

### Lab 1 — Gap analysis: re-attempt quizzes from memory

1. Pick 10-15 `QUIZ.md` files spread across Phase 1 (aim for at least one from each week, weighted toward topics you feel least confident about — Linux, containers, Kubernetes, Terraform, CI/CD, networking, cloud, service mesh should all have at least one represented).
2. For each, close the corresponding README notes and answer every question from memory, out loud or in writing.
3. Score yourself honestly: ✅ (answered confidently and correctly), 🟡 (partial/shaky), ❌ (couldn't answer).
4. Compile every 🟡 and ❌ into a single flat list — this is your real gap list, not a guess.

**Success criteria:** A written list of at least 5 concretely-named gaps (e.g., "Role vs ClusterRole scoping via RoleBinding" not "RBAC in general").

---

### Lab 2 — Portfolio audit checklist

Run this checklist against every public repo you plan to keep visible:

```
[ ] Top-level README explains WHAT it is, WHY you built it, and HOW to reproduce it
[ ] At least one paragraph describing a real tradeoff or decision you made (not just steps taken)
[ ] .gitignore excludes .terraform/, *.tfstate, real *.tfvars, node_modules/, .env
[ ] `git log -p | grep -iE "AKIA|secret|password"` (or a real secret scanner like gitleaks) run and clean
[ ] Commit history is legible (not required to be perfect, but not pure "fix fix fix" noise)
[ ] No leftover tutorial boilerplate / placeholder text never replaced
```

1. Pick your strongest 3-5 repos and run the full checklist against each.
2. Fix anything that fails — rewrite a thin README, add a missing `.gitignore` entry (and if a secret is found in history, treat it as compromised: rotate it, then use a tool like `git filter-repo` or BFG to scrub history, don't just delete the file going forward).
3. Pin those 3-5 repos on your GitHub profile.

**Success criteria:** 3-5 pinned repos that each pass every checklist item, with no secrets found in history.

---

### Lab 3 — Convert top gaps into a Phase 2 mini-project plan

1. From Lab 1's gap list, pick your top 3 (prioritize by how often the topic appears in job postings you actually want, per the previous note's guidance).
2. For each, write: the gap, a specific mini-project that would close it, and a one-sentence "done" definition (a concrete artifact, not a feeling).
3. Check whether your Phase 2 curriculum already plans to cover any of these deeply — if so, mark it "covered by Phase 2" instead of duplicating the work yourself.
4. Put whatever's left on an actual calendar/schedule, not just a list.

**Success criteria:** A written table of exactly 3 gap → mini-project → done-definition rows, with at least one item explicitly scheduled on a real calendar.

---

### Lab 4 — The core hands-on activity: write and publish a reflection post

1. Using the 5-part structure from the companion README (where you started, what surprised you, the hardest topic and why, one thing you're proud of, what's next), write a reflection post of at least 400 words.
2. Be specific — name actual topics, actual struggles, and reference at least one real lab or bug you hit during Phase 1.
3. Publish it somewhere real: a personal blog, dev.to, LinkedIn, or at minimum a pinned `REFLECTION.md` in your main portfolio repo — a reflection that stays entirely private defeats the "demonstrates communication skill" half of its purpose.

**Success criteria:** A published (not just drafted) reflection post, at least 400 words, containing at least one specific named struggle and one specific named accomplishment.

---

### Cleanup

None — today produces artifacts (a gap list, a pinned portfolio, a mini-project plan, a published post), it doesn't create anything to tear down.

### Stretch challenge

Share your reflection post with one real person (a peer, a mentor, a DevOps community/Discord/Slack you're part of) and ask for one piece of honest feedback — publishing into a void is easy; asking for and receiving real feedback is the harder, more valuable version of this exercise.
