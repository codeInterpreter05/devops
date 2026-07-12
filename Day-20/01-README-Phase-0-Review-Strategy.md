# Day 20 — Phase 0 Review + Mock Interview: Review Strategy & Troubleshooting Methodology

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Review | **Flag:** 📌

## Brief

Three weeks of new material compresses fast. By the time you're on Day 19, Day 1's exact `find` flags or Day 6's exact LVM commands have already started to blur — not because you didn't learn them, but because passive exposure decays without deliberate retrieval. This is the well-established "testing effect" from learning science: actively recalling information under mild difficulty (a cold quiz, a lab redone from memory) cements it far more durably than rereading notes, which *feels* productive but mostly builds familiarity, not recall. Review days exist to convert three weeks of "I've seen this" into "I can produce this on demand" — because that's the only form of knowledge that survives an interview whiteboard or a 3am page.

The second reason this day matters: real interviews and real incidents never give you open notes. An interviewer watching you Google `iostat` flags mid-answer, or an on-call engineer tabbing to Stack Overflow while a service is down, is a candidate/engineer who hasn't internalized the material yet — and the gap between "knows it" and "can produce it live, under time pressure, while talking" is exactly what today is designed to close, cheaply, before it costs you a job offer or extends an outage.

This day is split into two files:

1. **This file** — a systematic self-review framework for auditing Days 1–19, an integrated troubleshooting methodology that ties Linux fundamentals + storage + performance (the USE method) + basic networking debugging into one diagnostic flow, and the case for deliberately writing scripts without Googling.
2. **[02-README-Mock-Interview-Playbook.md](02-README-Mock-Interview-Playbook.md)** — how to run and self-grade a mock interview, structured think-aloud communication technique, and a full worked walkthrough of today's assigned scenario.

## Structuring the self-review: three passes, not one reread

A vague instruction like "review your notes" fails because rereading is passive — your eyes recognize the material and your brain quietly (and wrongly) files that as "I know this." Instead, run three distinct passes, each with a different failure mode it's designed to catch.

### Pass 1 — cold quiz re-attempt

Open each `QUIZ.md` from Day 1 through Day 19. Cover the `## Answers` section (fold it, scroll past it, whatever it takes) and answer every question in writing, from memory, roughly 5 minutes per day's quiz. Do not partial-credit yourself generously:

- **Full credit** — you could say the answer out loud, confidently, without hedging ("I think it's roughly...").
- **Partial / miss** — you reconstructed the answer piecemeal, second-guessed yourself, or got the shape right but fumbled a specific flag/number/term. This still counts as a gap, even if you eventually landed on the right answer — under interview or incident pressure, hesitation reads the same as not knowing.

### Pass 2 — cold lab re-run

You don't have time to redo all 19 labs in full. Time-box this and weight it toward your weakest areas from Pass 1, but always include a few high-yield anchors regardless of how confident you feel:

- Day 1's `find` lab (locate `.conf` files under `/etc`, filter by content, sort by mtime) — the single most-reused command pattern in this entire phase.
- Day 18's storage/LVM/`fstab` lab (extend a volume group, grow a filesystem, add and test a mount) — the topic with the most fiddly, easy-to-forget syntax.
- Day 19's performance-diagnosis lab (`free`, `vmstat`, `sar`, `strace` against a simulated bottleneck) — the topic most directly reused in today's mock interview.

Run each from memory, no copy-pasting from your own notes. If you have to open notes mid-lab, that's fine — but log it as a gap *immediately*, finish the step so you don't lose the rest of the pass, and move on.

### Pass 3 — actually close the gaps

This is the pass most self-review plans skip, and it's the only one that matters. Passes 1 and 2 just produce a list — a list you never act on is just a document that makes you feel like you reviewed. Keep a single running table across the whole review, and treat it as a backlog to clear before the day ends:

| Day | Topic | What I missed | Fix action |
|---|---|---|---|
| 1 | `find` vs `locate` | Blanked on which one uses a prebuilt index | Re-read Day 1 Cheatsheet, redo Lab 3 cold tomorrow morning |
| 18 | `fstab` loop-mounted images | Forgot the `loop` mount option is required for file-backed images | Redo the fstab lab section from Day 18, no notes |
| 19 | `iostat -x` columns | Couldn't recall what `%util` vs `avgqu-sz` each measure | Re-read Day 19 cheatsheet, write both definitions from memory tomorrow |

Don't move to the mock interview (file 2) until this table is either empty or every remaining row has a concrete, scheduled fix — not "review more later."

## Troubleshooting methodology refresher: one integrated diagnostic flow

Days 1–19 taught the USE method, storage tools, and performance tools somewhat separately. In a real incident (or interview), you don't get to pick which topic applies — you get "the server is slow" and have to bring everything together. The **USE method** (Utilization, Saturation, Errors — from Day 19) is the unifying frame: for *every* resource, ask the same three questions.

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `top`/`mpstat -P ALL 1` — % busy per core | Load average vs `nproc`; `vmstat`'s `r` (run-queue) column | `dmesg` for machine-check/hardware faults (rare) |
| Memory | `free -h` — used vs available | Swap activity: `vmstat`'s `si`/`so` columns, non-zero and growing | `dmesg \| grep -i oom` / `journalctl -k` for OOM-killer events |
| Disk | `df -h` (space), `iostat -x` `%util` (busy time) | `iostat -x` `avgqu-sz`/`aqu-sz` (queue depth); processes stuck in `D` state (`ps aux \| awk '$8=="D"'`) | `dmesg` for I/O errors, `smartctl -a` for hardware health |
| Network | `sar -n DEV 1`, `ss -s` summary | Retransmits/connection backlog: `ss -ti`, `netstat -s` retransmit counters | Interface errors/drops: `ip -s link`; DNS failures via `dig` |

Applied as a single top-down flow on a live "slow server," the order that wastes the least time is:

1. **Orient** — `uptime` (load average trend), `nproc` (core count), `uname -a`, `w` (who's logged in, how long it's been up).
2. **CPU** — is the load average high *and* CPU actually busy (`top` shows high `us`/`sy`)? If load is high but CPU is mostly idle, that's your first clue the bottleneck is elsewhere (usually I/O — processes in `D` state waiting on disk still count toward load average without using CPU).
3. **Memory** — `free -h`: is "available" near zero? Is swap in active use? Heavy `si`/`so` in `vmstat` means the box is thrashing, which will make *everything* look CPU-slow even though the real bottleneck is memory pressure spilling onto disk.
4. **Disk** — `df -h` for space exhaustion, `iostat -x 1 5` for `%util` (utilization) and `avgqu-sz`/`aqu-sz` (saturation). A disk at high `%util` with a growing queue explains sluggishness that has nothing to do with CPU or app code.
5. **Network** — `ss -tunlp` for listening sockets and connection states (a pile of `CLOSE_WAIT` means something isn't closing connections — usually app-level, not networking); `ping`/`traceroute` for reachability and path; `dig`/`nslookup` for DNS resolution time (a surprisingly common hidden cause of "slow" — every request doing a slow or failing DNS lookup before it even opens a connection); `curl -v` to actually exercise the HTTP layer, which `ping` and `traceroute` cannot touch.
6. **Application layer** — once the system-level resources look clean, pull logs (`journalctl -u <service>`, or `/var/log/...` per the FHS knowledge from Day 1) and use `strace -p <pid>` / `lsof -p <pid>` (Day 19) to see exactly what a specific process is blocked on.

This ordering matters because it's cheap-to-expensive and broad-to-narrow: five minutes of `uptime`/`free`/`df` rules out entire categories before you spend time attaching `strace` to a specific process.

## Basic networking debugging as a standard diagnostic layer

Networking wasn't a dedicated day in this phase, but "the network is fine, right?" is asked in nearly every real incident and every SRE-style interview, so it belongs in the integrated flow above as a first-class citizen, not an afterthought:

- **`ping -c 4 <host>`** — confirms L3/ICMP reachability and gives a rough latency figure. It does **not** confirm the actual service works: ICMP can be blocked by a firewall while the real TCP port is fine, or ICMP can succeed while the application is completely wedged. Treat it as a reachability check only, never as "the service is up."
- **`traceroute <host>`** (or `tracepath` if `traceroute` isn't installed) — shows the hop-by-hop path and where latency/loss is introduced. The last hop that responds before timeouts start is your prime suspect for where the problem lives (though the timeouts themselves can be caused by hops that simply don't reply to the probe type used — don't over-trust a single traceroute run).
- **`ss -tunlp`** (the modern replacement for `netstat`) — shows listening sockets and, with `-t state established`, active connections. A large number of connections stuck in `CLOSE_WAIT` points at the application not closing sockets it's done with; a pile of `TIME_WAIT` at high connection churn is usually normal but can exhaust ephemeral ports under extreme load.
- **`dig <host>`** / **`nslookup <host>`** — confirms DNS resolves at all, and `dig`'s timing output tells you *how long* resolution took. A misbehaving or unreachable resolver adds latency to every single outbound connection an app makes, and it's invisible to `ping`/`curl` checks that use a cached or hardcoded IP.
- **`curl -v <url>`** — the layer `ping`/`traceroute` can't reach: actual HTTP semantics. Add `-w` with a timing format string to break down DNS lookup, TCP connect, TLS handshake, and time-to-first-byte separately (see the Cheatsheet) — this pinpoints *which* layer of the request is slow instead of just "the request is slow."

## "Write a script without Googling" — deliberate retention practice

Recognition ("I'd know the right `subprocess` pattern if I saw it") is not the same skill as recall ("I can write it cold"). Interviews test recall directly ("write a function that..."), and incidents test it under worse conditions — a stressed on-call engineer with a laptop in a coffee shop and one bar of signal doesn't get a smooth Stack Overflow session. The fix is to practice recall specifically, not just re-expose yourself to the material.

Pick a handful of representative scripts spanning this phase's domains and write each fully from scratch, closed-book, timed at roughly 10–15 minutes each:

1. **Bash** — find every file over 100MB modified in the last 7 days under `/var`, print them sorted by size, largest first.
2. **Bash** — a health-check loop that `curl`s an endpoint every 5 seconds and logs any non-200 response with a timestamp.
3. **Python** — safely (list-form `subprocess.run`, explicit `timeout=`, `check=True`) run `df -h` over SSH on a list of remote hosts and parse free space out of the output.
4. **Python + boto3** — list every EC2 instance tagged `Environment=prod` that's in the `running` state, printing instance ID and public IP.
5. **YAML + Jinja2** — render an nginx upstream block from a YAML list of backend servers, and know what breaks if that YAML list is accidentally written as a scalar string instead of a list.

Grade cold-write attempts pass/fail, not on cosmetic polish: getting the right module, the right safety pattern, and the right general shape is a pass even if you fumble an exact flag name — go look it up *after* to fill that specific gap. Reaching for `shell=True` with f-string interpolation, forgetting `timeout=`/`check=True`, or not knowing the minimum IAM action for a `describe_instances` call is a real gap, not a typo, and goes straight onto the Pass 3 table above.

## Points to Remember

- Rereading notes is not review — retrieval under mild difficulty (cold quiz, cold lab) is what actually cements recall; treat any answer you had to reconstruct or hedge on as a gap, even if you got there eventually.
- Run three passes: cold quiz, cold lab (weighted toward weak spots plus a few fixed anchors — Day 1 `find`, Day 18 `fstab`, Day 19 performance tools), then actually close every logged gap before calling the review done.
- The USE method (Utilization/Saturation/Errors) is not CPU-specific — apply the same three questions to memory, disk, and network for a complete, non-random diagnostic sweep.
- The correct triage order is cheap-and-broad first, expensive-and-narrow last: `uptime`/`free`/`df` before `strace`/`lsof` on a specific PID.
- High load average with low CPU usage is a specific, memorizable signal: something is waiting (usually disk I/O or a blocked syscall), not computing.
- Basic networking checks (`ping`, `traceroute`, `ss`, `dig`, `curl -v`) belong in every systemic troubleshooting flow, not just "networking-specific" incidents — DNS latency and socket-state pileups are common root causes hiding behind "the app is slow."
- Writing a script cold and timed is the only realistic simulation of interview/incident conditions — recognizing correct code when you read it is a weaker, different skill.

## Common Mistakes

- Calling "I reread my Day 1–19 notes" a review — this builds familiarity, not recall, and collapses the first time you're asked to produce the answer instead of recognize it.
- Building a gaps table during Pass 1/2 and never scheduling time to actually close the entries — the table becomes a monument to gaps instead of a tool for fixing them.
- Applying the USE method only to CPU out of habit, missing that a disk-saturated or memory-thrashing box can look identical to a CPU problem in a quick glance at `top`.
- Treating networking as "not my topic since it wasn't an explicit day" and skipping `ping`/`traceroute`/`ss`/`dig`/`curl -v` practice — these come up in nearly every real troubleshooting interview regardless of how the curriculum was split.
- Spending all available review time on Day 1–5 material (recency/primacy bias) and neglecting Days 15–19, which are just as likely to be probed and were learned more recently, hence often falsely assumed to be "still fresh."
- Practicing scripts by reading a reference solution and nodding along ("yeah, I'd have written that") instead of writing your own attempt first, cold, and only then comparing — the comparison step is where the actual learning happens.
