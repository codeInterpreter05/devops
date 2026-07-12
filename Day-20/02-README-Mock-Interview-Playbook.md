# Day 20 — Phase 0 Review + Mock Interview: Mock Interview Playbook

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Review | **Flag:** 📌

## Brief

A candidate who knows the right commands can still fail a live troubleshooting interview by running them silently, in a random order, with no explanation — from the interviewer's chair, that's indistinguishable from someone who got lucky or is pattern-matching from memory without understanding. Conversely, a candidate who narrates a clear, methodical plan and only gets halfway to the root cause often scores *better*, because the interviewer's real question isn't "did you fix it" — it's "if I put you in front of a production incident at 3am, will you be systematic and communicate clearly under pressure, or will I be debugging both the server and your silence." Today's mock interview exists to drill the communication skill specifically, since it's the one Days 1–19 never explicitly practiced, and to rehearse the exact scenario assigned for this phase under realistic time pressure before it costs you a real offer.

## Running and self-grading a mock interview

You don't need a partner to get most of the value, though one helps. Either way, treat it as a real interview, not a study session:

1. **Set the conditions.** Pick a timer (20–25 minutes is realistic for a single scenario), close your notes, and if you're doing this solo, say your reasoning out loud anyway — silently thinking through the problem does not train the same muscle as verbalizing it under time pressure. Recording yourself (voice memo or screen capture) makes the self-grading step far more honest than trying to remember what you said.
2. **Read the scenario once, then start the timer.** Real interviewers don't repeat the prompt on demand — if you need to reconfirm scope, that's a clarifying question you ask out loud, which itself is part of what's being graded.
3. **Work the problem exactly as described below**, narrating throughout.
4. **Self-grade immediately after**, while it's fresh, using the rubric below. Be honest — the entire point is to surface gaps before a real interview does it for you.

### Self-grading rubric

| Dimension | What "strong" looks like |
|---|---|
| Clarifying questions | Asked 1–3 scoping questions before running anything (scope: one server or many? recent changes? what does "slow" mean?) |
| Stated plan | Said the overall diagnostic order out loud *before* the first command, not after |
| Method / order | Followed a deliberate broad-to-narrow sequence (load → CPU → memory → disk → network → app/logs), not random command soup |
| Narration | Explained *why* before running each command and interpreted its output before moving to the next one |
| Root cause | Landed on a specific, evidence-backed cause ("swap is thrashing because available memory hit zero at 14:02, confirmed by `vmstat`'s `si`/`so`"), not a guess ("maybe it's the disk?") |
| Fix | Proposed both an immediate mitigation and a longer-term prevention step, not just one or the other |
| Composure | Kept working through a dead-end command/output instead of freezing or jumping topics randomly |
| Time | Finished with a stated root cause and fix inside the time budget |

Score each dimension pass/partial/fail. Any two "fail" dimensions on a first attempt is normal — that's the whole point of doing this before the real thing. Redo the scenario (or a variant) after addressing the weakest dimension rather than moving on immediately; fixing "narration" specifically requires another timed rep, not just reading this file again.

## Structured communication: the think-aloud protocol

The single highest-leverage habit here is: **state your hypothesis and your next command before you run it, then state what the output tells you before running the next one.** This is not padding — it's the mechanism by which an interviewer (or a teammate watching you work an incident) can catch you before you go down a wrong path, and it's the difference between "I fixed it" and "I can be trusted to fix the next one too."

Concrete pattern to internalize:

> "My working theory right now is **[X]**. To check that, I'm going to run **[command]**, and I expect to see **[roughly this]** if I'm right."
> *(run it)*
> "Okay, that shows **[actual output]** — that **[confirms / rules out]** my theory, because **[reason]**. Next I want to check **[Y]**."

Notice the shape: hypothesis → prediction → command → interpretation → next hypothesis. This loop, repeated, *is* the interview. A candidate who just runs commands and reads output back verbatim ("okay, it says load average 15.2...") without stating what that number means or what they expected has skipped the part that's actually being evaluated.

Two specific habits interviewers explicitly look for:

- **"Tell me your plan first."** Before touching the keyboard, say the shape of your investigation out loud: "I'll start broad — system load, then check whether it's CPU, memory, disk, or network bound, then narrow into whichever one shows saturation, then check application logs." You can be wrong about the details and still score well here, because it demonstrates you have *a* method, not that you have the *right* one memorized.
- **Narrate hypotheses before commands, not after.** "I want to check memory next, because if load is high but CPU usage is low, something is usually waiting on I/O, and swapping is the most common cause" — said *before* running `free -h` — reads completely differently than running `free -h` first and retroactively explaining why you ran it.

## Full worked walkthrough: "Our production server is responding slowly. You have SSH access. Go."

This is the assigned scenario for today. Below is what a strong candidate's actual path looks like — narration, commands, plausible outputs, and the reasoning connecting each step to the next.

**Step 0 — clarifying questions, before touching the keyboard:**

> "Before I start — a few quick scoping questions. Is this one server, or one of several behind a load balancer where the others are healthy? Did anything change recently — a deploy, a config change, a traffic spike? And when you say 'slow,' do we mean high latency on requests, or actual timeouts/errors? Do we have any existing metrics dashboard, or am I working purely from the shell?"

Say the interviewer answers: *one of three app servers behind an ALB, the other two are healthy; a deploy went out about 40 minutes ago; users are seeing multi-second page loads but no outright errors; no dashboard access, shell only.*

> "Good — that already tells me something: since only one of three identical nodes is affected, this is probably node-local rather than a shared downstream dependency like the database or an external API, because I'd expect all three nodes to be equally slow if it were shared. The recent deploy is also a strong lead. I'll still verify system-level health first before assuming it's the deploy, though, since I don't want to anchor on the first plausible story."

**Step 1 — orient:**

> "I'll start broad: load average, uptime, and core count, so a bare load number means something."

```bash
uptime
# 14:32:07 up 12 days,  3:41,  2 users,  load average: 14.02, 11.87, 6.30
nproc
# 4
```

> "Load average of 14 on a 4-core box is heavily saturated — rising over the last 15 minutes based on the three numbers trending up. But load average alone doesn't tell me *what's* saturated — high load can mean CPU-bound, or it can mean processes stuck waiting on I/O. I need to check actual CPU usage next before assuming it's compute-bound."

**Step 2 — CPU:**

```bash
top -bn1 | head -15
# %Cpu(s):  8.1 us,  3.2 sy,  0.0 ni, 20.4 id, 66.9 wa,  0.0 hi,  1.4 si,  0.0 st
```

> "That's the key signal: only ~11% of CPU time is actually being used (`us`+`sy`), but 67% is `wa` — time the CPU spent idle *waiting on I/O*. So this is high load without high CPU usage, which per the USE method points me at saturation somewhere else, most likely disk. Before I jump to disk, though, I want to rule out memory pressure, because heavy swapping shows up exactly as high I/O wait — the swap device is a disk."

**Step 3 — memory:**

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.8G        7.2G        112M        180M        520M        190M
# Swap:          2.0G        1.9G         88M

vmstat 1 5
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free  buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  6  1946000 114820 40200 480000  812  640   900  1100 2200 3100  9  3 20 68
```

> "There it is — available memory is down to 190MB out of 7.8GB, swap is 95% full, and `vmstat` shows continuous swap-in/swap-out activity (`si`/`so` both in the hundreds, not zero) along with 6 processes in the blocked (`b`) state. That's a textbook memory-exhaustion-causing-swap-thrashing pattern: something has consumed nearly all physical RAM, the kernel is aggressively swapping, and swapping *is* disk I/O — which is exactly the 67% `wa` I saw a moment ago. This also explains why the whole server feels slow rather than just one endpoint: swap thrashing degrades everything running on the box, not just the process that caused it."

**Step 4 — identify the culprit and correlate with the deploy:**

```bash
ps -eo pid,ppid,pcpu,pmem,rss,etime,cmd --sort=-pmem | head -6
#   PID  PPID %CPU %MEM    RSS     ELAPSED CMD
#  8842  1    12.1 61.4  4812000    0:41:03 /usr/bin/python3 /opt/app/worker.py
#  ...

sudo journalctl -u app-worker --since "-45 min" | tail -20
# ... "processed batch 18422" repeated with growing gaps between lines ...
```

> "`worker.py` is holding 4.8GB RSS by itself and has been running for 41 minutes — almost exactly since the deploy 40 minutes ago. That's a strong correlation, not proof, but combined with steadily growing memory use over its lifetime it looks like a memory leak introduced in the new release, not a one-off spike. I'd want to check `pmap 8842` or watch RSS over a couple of minutes to confirm it's still climbing rather than plateaued, but given the time constraint I'm confident enough in this read to act."

**Step 5 — state the root cause and the fix, both immediate and long-term:**

> "My conclusion: the deploy 40 minutes ago introduced a memory leak in the worker process. It has consumed nearly all available RAM, forcing the kernel into heavy swapping, which saturates disk I/O and makes the entire server — not just that worker — feel slow, even though CPU usage itself is low. That matches every signal: high load with low CPU, near-zero available memory, active swap thrashing, and a single process whose memory footprint lines up with the deploy timing.
>
> **Immediate mitigation:** restart the worker process (`systemctl restart app-worker`) to reclaim the leaked memory and stop the swapping — this should bring load back down within a minute or two. Since it's one of three nodes and the other two are healthy, I'd also consider pulling this node out of the ALB target group first so restarting it doesn't cause a blip for in-flight requests.
> **Longer-term fix:** this only buys time — the leak is still in the new release. I'd flag the deploy to the team as the likely cause, get eyes on the diff for anything obviously leak-shaped (an unbounded cache, a growing list/queue never drained, connections not closed), and add a memory-based alert (e.g., available memory or swap usage crossing a threshold) so this is caught in minutes next time instead of via a user-reported slowness ticket after 40 minutes of degradation."

That closing summary — cause, immediate fix, root-cause fix, and a prevention step — is what a "complete" answer looks like. Stopping at "restart it, that fixed it" is a common and costly gap (see Common Mistakes below).

## Points to Remember

- State your plan before your first command — a stated method scores well even when the details are imperfect; silent command-running scores poorly even when it's correct.
- Narrate hypothesis → prediction → command → interpretation, in that order, every time — this loop is what's actually being evaluated, not just whether you eventually got the right answer.
- Ask 1–3 scoping/clarifying questions before diagnosing anything — scope (one host vs. many), recent changes, and what "slow" concretely means change the whole investigation.
- Follow broad-to-narrow: load average → CPU → memory → disk → network → application logs. Don't jump straight to `strace`-ing a random process because it "feels" like the culprit.
- High load average with low CPU usage is a specific signal (something is waiting, usually on I/O) — say that out loud when you see it; it demonstrates you understand what load average actually measures.
- Always close with both an immediate mitigation and a longer-term/preventive fix — a root cause without a two-part fix is an unfinished answer.
- Self-grade honestly against the rubric and redo the weakest dimension with a fresh timed rep — rereading this playbook again is not the same as another timed attempt.

## Common Mistakes

- Jumping straight to a fix ("let me just restart the service") before establishing why it's slow — even if the fix works, an interviewer (and a postmortem) will ask what you'd have missed if restarting hadn't helped.
- Running commands silently and only narrating the final answer — the interviewer can't distinguish systematic reasoning from a lucky guess if they never hear the reasoning.
- Panicking when the clock is visible and abandoning the methodical order for random command-guessing — this is exactly the failure mode the timer is designed to surface in practice, so it can be fixed before it happens for real.
- Not asking any clarifying questions and silently assuming scope (e.g., investigating as if it's the only server, when it's one of several, or vice versa) — wastes time chasing the wrong category of cause.
- Fixating on the first plausible hypothesis and stopping the investigation the moment *any* command "look bad" without confirming it's actually the bottleneck (e.g., seeing swap usage and immediately declaring "it's memory" without checking whether swap activity is actually ongoing versus a stale, already-settled number).
- Treating interviewer questions or hints mid-scenario as interruptions rather than signal — an interviewer asking "are you sure that's the bottleneck?" is often steering you, not testing your resolve to ignore them.
- Giving a root cause with no fix, or a fix with no root cause ("I'd restart it" with no explanation of why that would help, or a long correct diagnosis that never actually answers "so what do you do about it").
- Ending the scenario without a summary — a strong close is 2–3 sentences stating cause, immediate action, and prevention, as if reporting to a lead; trailing off after the last command leaves the interviewer to guess whether you actually reached a conclusion.
