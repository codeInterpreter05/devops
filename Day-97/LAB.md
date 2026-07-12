# Day 97 — Lab: Incident Management

**Goal:** Write a real, usable runbook for a plausible failure mode from your own experience, then run a mock incident end-to-end (declare, respond, resolve) and conduct a genuine blameless postmortem retrospective — the assigned hands-on activity, done in full.

**Prerequisites:** None technical — this lab is process-focused. A text editor, and ideally 1-2 other people to role-play the mock incident (if solo, role-play multiple parts yourself, switching hats explicitly).

---

### Lab 1 — Pick a real failure mode and write a runbook

1. Think back to your internship/coursework/personal projects — pick one concrete failure mode you've actually seen or can very plausibly imagine (e.g., "database connection pool exhaustion," "a background job queue backing up," "disk fills up on a log-heavy host," "a third-party API starts rate-limiting us").
2. Write a runbook for it following the structure from `03-README-Runbooks-And-Playbooks.md`: trigger condition, diagnosis steps with expected outputs, remediation with exact commands, escalation/rollback path.
3. Have someone else (or yourself, 30 minutes later, pretending you don't have context) try to follow it literally, step by step, without asking you clarifying questions. Note every point where they hesitated or had to guess.

**Success criteria:** A runbook document that a stranger with reasonable technical background, but zero context on your specific system, could follow to at least correctly diagnose the failure mode (even if the exact remediation commands are illustrative/hypothetical for your setup).

---

### Lab 2 — Declare and run a mock incident

1. Write a one-paragraph scenario based on your Lab 1 failure mode actually happening in production, with a specific (fictional) user-facing symptom and a plausible severity.
2. Assign severity (P1-P4) using the framework from file 1 — write down your reasoning (blast radius, business criticality, workaround availability).
3. Role-play declaring the incident: write the first message you'd post in an `#incident-XXX` channel, including severity, initial impact assessment, and who's the Incident Commander vs. responder (even if solo, explicitly name which "hat" you're wearing at each point).
4. Write 3-4 timestamped "status update" messages as if 15 minutes had passed between each, following the "communication during incidents" guidance from file 2 (including at least one "still investigating, no new info" update).
5. Write the "resolved" message, including a rough duration and impact summary.

**Success criteria:** A realistic, timestamped incident channel transcript from declaration to resolution that a teammate reading it cold could follow — demonstrating IC/responder role separation and a real update cadence (not just declare + resolve with nothing in between).

---

### Lab 3 — Conduct a mock blameless postmortem

1. Using your Lab 2 incident transcript, write a full postmortem document: Summary, Timeline (pull exact "timestamps" from your Lab 2 transcript), Impact, What went well / what went poorly, and Root cause.
2. Run a 5 Whys analysis on the root cause, writing out all 5 "why" steps explicitly (model on the connection-pool-exhaustion example in file 4). At why #3 or #4, deliberately check yourself: are you drifting toward blaming a person? If so, rewrite that step to target the system/process instead.
3. Write at least 3 concrete action items, each with an explicit (fictional but specific) owner and due date, and classify each as: fixes root cause directly / improves detection / improves response process.

**Success criteria:** A complete postmortem document where the 5 Whys chain ends on a systemic/process-level cause (not "a person made a mistake"), and every action item has an owner, a due date, and is specific enough that "was this actually done" could be objectively checked later.

---

### Lab 4 — Retrospective on the retrospective

1. Re-read your own postmortem from Lab 3 with a critical eye. Answer honestly: does any part of it, even subtly, read as blaming a person rather than the system? Rewrite any sentence that does.
2. Check: if this were a real recurring pattern (this same category of incident, 3rd time in 6 months), would your action items actually prevent a 4th occurrence, or are they superficial ("add more monitoring" with no specifics)?

**Success criteria:** You can point to at least one place you initially wrote something blame-adjacent and explain how you reframed it, and at least one action item you tightened from vague to concrete.

---

### Cleanup

No system state to clean up — archive your Lab 1-3 documents somewhere you'll actually reference them (they're good interview prep material and portfolio evidence of process maturity).

### Stretch challenge

Take the same incident and re-run the 5 Whys as a **contributing-factors analysis** instead (list every independent factor that had to be true for the incident to reach the severity it did — not a single linear chain). Compare: did the fishbone-style analysis surface a contributing factor the strict 5-Whys chain missed?
