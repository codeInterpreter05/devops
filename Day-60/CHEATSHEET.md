# Day 60 — Cheatsheet: Phase 1 → Phase 2 Transition

## Gap analysis: recognition vs. recall vs. application

```
Recognition  "this makes sense when I read it"        -> weakest, false confidence
Recall       "I can explain this cold, unaided, 60s"   -> real test #1
Application  "I can decide what to do in a scenario"    -> real test #2, what interviews/incidents test
```

## Gap -> mini-project conversion pattern

```
Gap: vague competence claim
  -> Mini-project: a concrete build/break/fix exercise
    -> Done-definition: a specific artifact (repo, postmortem writeup), not a feeling

Example:
  Gap: "understand NetworkPolicy in theory, never debugged one"
  Mini-project: build 3-tier app, intentionally break NetworkPolicy, diagnose with kubectl only
  Done: a written investigation doc in the portfolio repo
```

## Portfolio audit checklist

```bash
# secret scan (rough pass)
git log -p | grep -iE "AKIA|secret|password"
# or, properly:
gitleaks detect --source . -v

# check what's actually ignored
cat .gitignore   # expect: .terraform/, *.tfstate, *.tfvars (if real values), node_modules/, .env

# if a secret IS found in history:
# 1. rotate/revoke the credential immediately (assume compromised)
# 2. scrub history: git filter-repo --path <file> --invert-paths   (or BFG Repo-Cleaner)
```

```
README checklist per pinned repo:
[ ] WHAT it is
[ ] WHY you built it
[ ] HOW to reproduce it
[ ] one tradeoff/decision paragraph (not just steps)
```

## Reflection post structure (5 parts)

```
1. Where you started       - specific, honest starting point
2. What surprised you      - technically, and about the process/pacing
3. Hardest topic + why     - name it specifically, not "Kubernetes is hard"
4. One thing you built     - proud of, with enough detail to be credible
5. What's next             - your Phase 2 mini-project plan, stated publicly
```

## Phase 2 planning — timeboxed

```
1. List top 5 gaps, ranked by job-posting relevance
2. Convert top 3 into mini-projects (gap -> project -> done-definition)
3. Cross-check against existing Phase 2 curriculum — don't duplicate
4. Timebox this whole exercise to < 1 hour
5. Put at least one item on an actual calendar
```

## Today's non-negotiables

```
[ ] Re-attempted 10-15 quizzes from memory, honest scoring
[ ] 3-5 repos pinned, each passing the portfolio checklist
[ ] 3-row gap -> mini-project -> done-definition table, one item scheduled
[ ] Reflection post published (not just drafted), 400+ words
[ ] Genuinely stepped away from new technical input for part of the day
```
