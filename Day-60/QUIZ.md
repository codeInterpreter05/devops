# Day 60 — Quiz: Phase 1 → Phase 2 Transition

Try to answer without looking at your notes. Answers are at the bottom.

1. What's the difference between "recognition," "recall," and "application" as levels of knowing a technical topic, and which two does a real interview actually test?
2. Why is re-attempting old `QUIZ.md` files from memory a more reliable gap-analysis tool than just re-reading the corresponding README notes?
3. Give an example of converting a vague gap ("I should get better at Terraform") into a concrete mini-project with a specific "done" definition.
4. Name at least four things a portfolio audit checklist should check for in a repo before it's pinned publicly.
5. If you discover a real (even if old/possibly-expired) credential committed in your git history, what's the correct remediation sequence — and why isn't "just delete the file in a new commit" sufficient?
6. Why does a portfolio README that explains a tradeoff ("chose one NAT gateway over one-per-AZ for cost, since this is non-HA") read as stronger evidence than one that only lists what was built?
7. Why is "evidence of debugging a real problem" considered disproportionately valuable in a portfolio compared to another clean, successful build?
8. Why should Phase 2 planning be explicitly timeboxed (e.g., under an hour) rather than done exhaustively?
9. What's the risk of treating "rest and consolidate" as an instruction to skip the day's work entirely, versus what it's actually meant to signal?
10. List the 5 parts of the suggested reflection post structure.
11. Why does the reflection post's value depend partly on actually publishing it rather than keeping it private?
12. **Interview-style prompt for today (there is no formal interview question assigned; treat this as your own self-check):** In under 90 seconds, summarize your single biggest technical growth area from Phase 1 and the specific mini-project you're going to use to close it in Phase 2.

---

## Answers

1. Recognition is "this makes sense when I read it" — the weakest and most misleading level, since it produces false confidence. Recall is being able to explain a topic cold, unaided, from memory. Application is being able to decide what to do given a novel scenario you haven't seen phrased exactly that way before. Real interviews (and real production incidents) test recall and application almost exclusively — recognition alone doesn't transfer to either.
2. Re-reading notes re-triggers recognition ("yes, this looks familiar/correct") without ever testing whether you could have produced that answer unaided — it systematically overestimates your actual competence. Answering from memory first, with notes closed, is the only way to distinguish genuine recall from familiarity with the text.
3. Gap: "I should get better at Terraform" is not actionable. Concrete version: "My Terraform experience is all single-environment" → mini-project: refactor an existing project into a proper multi-environment structure (workspaces or per-environment state) with a CI pipeline that plans on PR and applies on merge, gated by environment → done-definition: a working repo demonstrating exactly that pipeline, not just a feeling of having "learned multi-environment Terraform."
4. Any four of: a README explaining what/why/how to reproduce; a paragraph describing a real tradeoff/decision made; a proper `.gitignore` excluding `.terraform/`, `*.tfstate`, real `*.tfvars`, `node_modules/`, `.env`; a scan for accidentally committed secrets in history; a legible (not pure "fix fix fix") commit history; no leftover unedited tutorial boilerplate/placeholder text.
5. Rotate/revoke the credential immediately, treating it as compromised regardless of whether it still appears active — then scrub it from git history entirely using a tool like `git filter-repo` or BFG Repo-Cleaner, and force-push the cleaned history. Deleting the file in a new commit is not sufficient because the secret remains fully readable in the repository's git history (every prior commit), accessible to anyone who clones the repo or views its commit log.
6. Because it demonstrates engineering judgment — the ability to weigh alternatives and articulate a reasoned decision — rather than just the ability to execute steps from a tutorial. A hiring manager evaluating a portfolio is specifically trying to distinguish "followed instructions" from "made decisions," and a stated tradeoff is direct evidence of the latter.
7. Because debugging a real problem is direct, undeniable evidence of the exact skill that interviews and production incidents actually test — hypothesis formation, systematic investigation, and problem resolution — whereas a clean successful build only proves you can follow a known-good path, which most portfolios already show repeatedly and which reveals little about how you'd behave when something breaks unexpectedly.
8. Because exhaustive re-planning is itself a form of procrastination that delays actually starting the mini-projects — a focused, prioritized plan you'll actually execute is more valuable than a comprehensive roadmap that becomes another thing to maintain instead of doing the work.
9. The risk is conflating "deliberately lighter, different kind of work" (reflection, planning, portfolio cleanup) with "no work at all" — today is still meant to produce real artifacts (a gap list, a cleaned portfolio, a mini-project plan, a published reflection); it's a change in intensity and type of work, not a day off from the curriculum.
10. Where you started; what surprised you (technically, and about the learning process); the hardest topic and specifically why; one concrete thing you built that you're proud of; what's next (your Phase 2 mini-project plan).
11. A private reflection only serves the personal-consolidation half of its purpose; publishing it additionally demonstrates communication skill to anyone evaluating your portfolio later, which is itself a relevant, increasingly-valued DevOps skill (writing clear incident postmortems and design docs) — keeping it private forfeits that second, real benefit entirely.
12. This is a self-check — there's no single correct answer, but a strong response names one specific, genuinely-still-shaky topic (not a vague area) and pairs it with the exact mini-project and done-definition from your own Lab 3 answer, delivered concisely within the time limit — proving you can compress your own gap analysis into an interview-ready 90-second answer, which is itself good practice for real interview time constraints.
