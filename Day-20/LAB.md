# Day 20 — Lab: Phase 0 Review + Mock Interview

**Goal:** Prove — under a clock, without notes — that Days 1–19 are actually retrievable knowledge, not just recognizable material. This lab has no new tools to install; it's a self-test, a timed troubleshooting challenge against deliberately broken systems, and a mock-interview dry run.

**Prerequisites:** A Linux environment with `sudo` (Ubuntu via WSL2, a VM, or a disposable cloud instance — **not** a machine you can't afford to have temporarily broken, since Lab 2 deliberately breaks things). Completed `QUIZ.md` and `LAB.md` files for Days 1–19 (used as the source material for Lab 1, but must **not** be open during Labs 2–3). A stopwatch or phone timer. Optional: a way to record yourself talking (voice memo) for Lab 3.

---

### Lab 1 — Self-test: cold quiz and lab re-run

This is Pass 1 and Pass 2 from [01-README-Phase-0-Review-Strategy.md](01-README-Phase-0-Review-Strategy.md), made concrete and checklist-driven.

1. For each Day 1–19 `QUIZ.md` that exists in this repo, cover the `## Answers` section and answer every question from memory, in writing. Budget ~5 minutes per day. Do not skip a day because it "feels solid" — that feeling is exactly what this exercise is designed to check.
2. Score each question pass / partial / fail using the honesty rule from file 1: if you had to reconstruct or hedge, it's not a pass.
3. Re-run, from memory and without opening your notes, the core hands-on activity from at least these three labs:
   - Day 1 `LAB.md` — Lab 3 (`find` all `.conf` files under `/etc`, filter by content, sort by mtime).
   - Day 18 `LAB.md`'s storage/LVM/`fstab` exercise (extend/grow a volume, add and test a mount entry).
   - Day 19 `LAB.md`'s performance-diagnosis exercise (use `free`/`vmstat`/`sar`/`strace` against a simulated bottleneck).
4. Log every miss — quiz or lab — into a single running gaps table:

   | Day | Topic | What I missed | Fix action |
   |---|---|---|---|
   | | | | |

**Success criteria:** You have a complete pass/partial/fail scorecard across all available Day 1–19 quizzes, you've re-run at least three prior labs cold, and every miss is captured in the gaps table with a concrete fix action — not just "review more."

---

### Lab 2 — The core hands-on activity: 45-minute timed troubleshooting challenge

This is today's assigned hands-on activity: **given a broken system, diagnose and fix 3 issues in 45 minutes.** You will play both roles — the person who breaks the box, and the person who fixes it — so set up first, then treat the fixing phase as a real timed exercise with notes closed.

**Setup (not timed) — inject three realistic failures:**

All three are contained in a disposable location (`/tmp/labday20`, a loopback filesystem, and background processes) so nothing touches your real root filesystem or a real service.

**Issue 1 — a runaway log file filling a disk.**

```bash
mkdir -p /tmp/labday20
sudo dd if=/dev/zero of=/tmp/labday20/disk.img bs=1M count=300
sudo mkfs.ext4 -q /tmp/labday20/disk.img
sudo mkdir -p /mnt/labday20
sudo mount -o loop /tmp/labday20/disk.img /mnt/labday20
sudo chown "$USER":"$USER" /mnt/labday20
df -h /mnt/labday20   # confirm a ~300MB mount you can safely fill

mkdir -p /mnt/labday20/var/log/rogue-app
yes "ERROR: simulated runaway log line filling disk" >> /mnt/labday20/var/log/rogue-app/app.log &
echo "rogue writer PID: $!"    # note this — you'll "discover" it during the timed challenge
```

Let it run in the background — it should still be actively filling the mount when the timer starts.

**Issue 2 — a misconfigured `fstab` entry blocking a mount.**

```bash
sudo cp /etc/fstab /etc/fstab.labday20.bak    # always back up fstab before editing it

echo "/tmp/labday20/disk.img  /mnt/labday20-data  ext4  defaults  0  2" | sudo tee -a /etc/fstab
sudo mkdir -p /mnt/labday20-data
```

Don't run `mount -a` yet — leave the broken entry in place for the timed challenge to discover (it's missing the `loop` option, so mounting a file-backed image this way will fail with something like `mount: special device ... does not exist`).

**Issue 3 — a runaway process pegging CPU.**

```bash
yes > /dev/null &
echo "CPU hog PID: $!"
```

This pegs one core at 100%. Once all three are running, note the current time and start your 45-minute timer.

**The timed challenge (45 minutes, notes closed):**

Diagnose and fix all three issues. Suggested time budget: ~12 minutes per issue plus a 9-minute buffer, but work in whatever order your diagnosis naturally leads you — don't force a fixed sequence if the symptoms point elsewhere first.

For each issue, you must:
1. **Diagnose before fixing** — identify the specific root cause with a command whose output actually demonstrates it (not a guess).
2. **Fix it** — resolve the issue with the minimum safe action.
3. **Verify** — confirm with a follow-up command that the fix actually worked.

**Scoring rubric:**

| Issue | Correct diagnosis (evidence-based, stated before fixing) | Correct fix applied | Verified after fixing |
|---|---|---|---|
| 1 — full disk / runaway log | `df -h` shows near-full mount; located the file via `du -sh` or `find -size +50M`; identified the writer PID via `ps`/`fuser`/`lsof +D` | Killed the writer, truncated/rotated the log | `df -h` shows space reclaimed |
| 2 — broken fstab / blocked mount | Ran `mount -a` or `mount /mnt/labday20-data`, read the exact error, inspected `/etc/fstab` for the bad line | Added the missing `loop` option (or removed the bad line) | `mount -a` succeeds, `df -h`/`mount \| grep labday20-data` confirms it's mounted |
| 3 — CPU-pegging process | Identified the process via `top`/`ps --sort=-pcpu`, correlated with load average vs `nproc` | Killed the correct PID | `top`/`uptime` shows load and CPU usage back to baseline |

Score each cell pass/fail. **Pass** = fixed all three within 45 minutes with correct diagnosis-before-fix on each. **Strong pass** = also narrated each diagnosis out loud as you went, per the think-aloud protocol in file 2 — recruit a friend to listen, or record yourself.

**Success criteria:** All three issues diagnosed (with evidence cited before any fix was applied) and resolved within the 45-minute window, each verified with a follow-up command.

---

### Lab 3 — Mock interview self-run: "Our production server is responding slowly. You have SSH access. Go."

1. Set a 20–25 minute timer. If working solo, talk out loud anyway (recording yourself is strongly recommended — self-grading from memory is unreliable).
2. Either restore the CPU-hog + memory-pressure conditions from Lab 2 (or a subset), or run this purely as a verbal/whiteboard exercise walking through what you'd check and why, in order, if you don't want to re-break a system a third time.
3. Work the scenario following the structured approach from [02-README-Mock-Interview-Playbook.md](02-README-Mock-Interview-Playbook.md).
4. Self-grade against this checklist:
   - [ ] Asked at least one clarifying/scoping question before running anything
   - [ ] Stated the overall diagnostic plan out loud before the first command
   - [ ] Checked load average and core count first (`uptime`, `nproc`)
   - [ ] Investigated CPU → memory → disk → network in a deliberate order, not randomly
   - [ ] Narrated each command's purpose *before* running it, and interpreted the output *before* moving on
   - [ ] Reached a specific, evidence-backed root cause (not a guess)
   - [ ] Proposed both an immediate mitigation and a longer-term/preventive fix
   - [ ] Finished within the time budget
   - [ ] Closed with a 2–3 sentence summary, as if reporting to a lead
5. Log any unchecked box as a gap in the Lab 1 gaps table and schedule a second timed attempt before moving on from this phase.

**Success criteria:** Every checklist item above is checked on at least one timed attempt (repeat the exercise once if the first attempt has more than one or two misses).

---

### Cleanup

Undo everything injected in Lab 2 before finishing — leaving loop mounts, background hogs, or a modified `/etc/fstab` in place will bite you later.

```bash
# stop the simulated processes
pkill -f "yes ERROR: simulated runaway log line" 2>/dev/null
pkill -x yes 2>/dev/null

# unmount and remove the loopback filesystem
sudo umount /mnt/labday20-data 2>/dev/null
sudo umount /mnt/labday20 2>/dev/null
sudo rm -rf /mnt/labday20 /mnt/labday20-data /tmp/labday20

# restore fstab to its pre-lab state
sudo cp /etc/fstab.labday20.bak /etc/fstab
sudo rm -f /etc/fstab.labday20.bak

# confirm nothing lab-related is left behind
mount | grep labday20          # should print nothing
grep labday20 /etc/fstab       # should print nothing
ps aux | grep -E "yes|labday20" | grep -v grep   # should print nothing
```

### Stretch challenge

Re-run Lab 2's setup, but this time inject all three issues simultaneously *and* don't reread the setup steps once the timer starts — have a friend (or a coin flip / random order generator) decide which issues actually got injected, so you're diagnosing genuinely blind rather than working from a known list of three. Alternatively, write (cold, no Googling) a short Python script using `subprocess` (list-form, `timeout=`, `check=True`) that SSHs into a list of hosts, runs `df -h`, `free -h`, and `uptime`, and flags any host whose disk usage exceeds 90% or whose load average exceeds its core count — a realistic automation of the exact diagnostic first step from today's mock scenario.
