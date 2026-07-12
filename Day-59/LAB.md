# Day 59 — Lab: Phase 1 Mock Interview Day

**Goal:** Run a full, timed, recorded 60-minute mock DevOps interview covering behavioral, system design, and hands-on debugging — then review your own recording critically and produce a written list of concrete improvements. This is the assigned hands-on activity, expanded into a structured exercise rather than a passive "think about your answers" review.

**Prerequisites:** A way to record yourself answering out loud (phone camera, laptop webcam, or just screen+audio if doing a terminal-based scenario) — video is more useful than audio-only because you can observe body language/hesitation. A timer. Ideally, a friend/peer to play interviewer for at least one round (a real second voice asking follow-ups is far harder to fully simulate solo) — but a solo run with a strict timer is still valuable and is the default assumed here.

---

### Lab 1 — Behavioral warm-up (10 minutes)

1. Set a 2-minute timer per question. Answer out loud, recording yourself, for:
   - "Tell me about a time you debugged a difficult production issue."
   - "Tell me about a technical decision you disagreed with and how you handled it."
   - "Walk me through your DevOps learning journey and why you're pursuing this career."
2. Use the **STAR** structure (Situation, Task, Action, Result) explicitly — check afterward whether your answer actually had all four parts or wandered.

**Success criteria:** Each answer fits within ~2 minutes without rambling, and includes a concrete, specific result (a number, a time saved, an outage duration) rather than a vague conclusion.

---

### Lab 2 — System design (25 minutes)

1. Set a 25-minute timer. Answer, out loud and recorded, exactly as you would live: **"Design a multi-tenant SaaS platform on EKS with tenant isolation."**
2. Force yourself to spend the first 3-4 minutes purely asking/stating clarifying questions and requirements before naming a single AWS/Kubernetes service.
3. Sketch the architecture on paper or a whiteboard tool while narrating — don't just talk, produce an artifact, since real system design interviews are almost always whiteboard/diagram-driven.
4. Explicitly cover, out loud: the isolation model choice and its tradeoff, concrete Kubernetes-level isolation mechanisms (ResourceQuota, NetworkPolicy, IRSA), data-layer isolation, and the tenant-onboarding control plane.

**Success criteria:** A recorded 25-minute answer plus a photo/screenshot of your architecture sketch, with all four required coverage areas from step 4 addressed without prompting.

---

### Lab 3 — Hands-on debugging scenarios (15 minutes)

1. Have a peer (or yourself, written on a card beforehand and read cold) present these two scenarios verbally, one at a time, 7 minutes each:
   - "A Deployment's pods are stuck in `CrashLoopBackOff`. Walk me through your debugging process."
   - "`terraform apply` is failing with a state-related error. Walk me through it."
2. Answer entirely verbally — no typing real commands, this is a talk-track exercise simulating a phone screen or whiteboard-only round, not a hands-on lab environment.
3. For each, explicitly state your hypothesis before naming the command that would test it (e.g., "I suspect this is OOMKilled, so I'd check..." before "...`kubectl describe pod`").

**Success criteria:** Both scenario answers include explicit hypothesis-before-command narration, matching the talk-tracks in file 02, in your own words (not memorized verbatim).

---

### Lab 4 — Self-review and improvement list

1. Watch your own recordings in full. Do not skip this — it is the single highest-value part of today.
2. For each section, write down: (a) one thing you did well, (b) one specific weakness (a filler-word habit, a missed clarifying question, an answer that ran too long), (c) one concrete fix you'll practice before your next mock.
3. Time-check: did system design run over 25 minutes? Did any behavioral answer exceed 2-3 minutes? Note any pacing issues explicitly.

**Success criteria:** A written self-review document with at least 3 concrete, specific (not vague) improvement items — "talk less" is not specific enough; "I spent 4 minutes on the isolation model before mentioning data-layer isolation at all — lead with a coverage checklist next time" is.

---

### Cleanup

Save your recording and self-review notes somewhere durable (not just your phone's camera roll) — you'll want to compare against a second mock later in your job search to measure actual improvement.

### Stretch challenge

Recruit a friend or peer (even one outside DevOps) to play a skeptical interviewer for a second full round, allowed to interrupt with follow-up questions ("why not just use separate clusters for everyone?" / "what does that cost at 1000 tenants?") — handling unscripted follow-ups live is meaningfully harder than a monologue and is much closer to a real interview's actual difficulty.
