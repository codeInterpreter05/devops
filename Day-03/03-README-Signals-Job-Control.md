# Day 3 — Processes & System: Signals & Job Control

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:**

## Brief

Killing a process correctly, and running things in the background correctly, are two of the most-used and most-misunderstood skills in daily Linux operations. Get it wrong and you either fail to actually stop a runaway process, corrupt data by yanking it out from under an in-progress write, or lose a long-running job the moment you close your SSH session. This file covers **signals** — the kernel's mechanism for asynchronously notifying a process of an event — and **job control** — the shell's mechanism for managing foreground/background execution and surviving terminal disconnection. Both come up constantly in interviews, most famously as: *"How do you find and kill a process consuming 100% CPU without killing its children?"*

## Signals: what they actually are

A **signal** is a limited, asynchronous notification delivered by the kernel to a process — not data, just a number (1–31 for standard signals) that interrupts the process and, unless the process has registered a custom handler, triggers a **default action** (terminate, terminate + core dump, stop, continue, or ignore). Processes can:

- **Catch** a signal (run their own handler function instead of the default action),
- **Block** a signal (defer delivery until unblocked),
- **Ignore** a signal (explicitly discard it),

**...for most signals.** Two are special-cased by the kernel and cannot be caught, blocked, or ignored by any process, no matter what: `SIGKILL` (9) and `SIGSTOP` (19).

## The signals you need cold

| Signal | Number | Default action | Catchable? | Typical use |
|---|---|---|---|---|
| `SIGHUP` | 1 | Terminate | Yes | Originally "the terminal hung up" (modem disconnected); today, widely repurposed by daemons to mean "reload your config" (e.g., `nginx`, `sshd`) |
| `SIGINT` | 2 | Terminate | Yes | Sent by Ctrl-C from the controlling terminal |
| `SIGQUIT` | 3 | Terminate + core dump | Yes | Ctrl-\ |
| `SIGKILL` | 9 | Terminate | **No — never** | Force-kill; kernel removes the process's scheduling entry immediately once it's not blocked in uninterruptible I/O |
| `SIGTERM` | 15 | Terminate | Yes | The **default and polite** signal sent by plain `kill <pid>` — "please shut down" |
| `SIGSTOP` | 19 | Stop (pause) | **No — never** | Pause a process unconditionally (used by job control's Ctrl-Z under the hood, technically `SIGTSTP` which *is* catchable) |
| `SIGCONT` | 18 | Continue | Yes | Resume a stopped process |
| `SIGCHLD` | 17 | Ignore | Yes | Sent to a parent when a child terminates or stops — this is what a well-behaved parent listens for to call `wait()` and reap zombies (see file 1) |

## `SIGTERM` vs `SIGKILL` — the core distinction

- **`SIGTERM` (15)** asks the process to terminate **gracefully**. A well-written program catches it, closes open file handles, flushes buffers, finishes an in-flight database transaction, deregisters from a load balancer, and *then* exits on its own. This is why `kill <pid>` (no flags) sends `SIGTERM` by default — it's the "ask nicely" signal, and virtually every orchestrator (systemd, Docker, Kubernetes) sends `SIGTERM` first, then waits a grace period (Kubernetes default: 30 seconds, `terminationGracePeriodSeconds`) before escalating.
- **`SIGKILL` (9)** is enforced **directly by the kernel**, bypassing the target process entirely — there is no handler to run, no cleanup, no chance to flush anything. The process is simply removed from the scheduler the moment it's not stuck in uninterruptible I/O (`D` state, see file 1). This is why `kill -9` is a last resort: it can leave temp files uncleaned, database files in an inconsistent state, or locks held forever (until something else notices the holder is gone).

**Practical rule:** always try `SIGTERM` first, wait a few seconds, check if the process actually exited, and only escalate to `SIGKILL` if it's still alive (e.g., ignoring `SIGTERM`, or genuinely wedged). This is exactly what `systemctl stop`, `docker stop`, and Kubernetes pod termination all do under the hood.

## `kill`, `killall`, `pkill` — sending signals

```bash
kill 1234                  # SIGTERM to PID 1234 (default signal if none specified)
kill -15 1234               # explicit SIGTERM (same as above)
kill -9 1234                # SIGKILL — force, no cleanup, last resort
kill -HUP 1234               # send by name instead of number — reload config for many daemons
kill -l                      # list all signal names/numbers

kill -TERM -1234             # note the MINUS sign — targets process GROUP 1234, not PID 1234
kill -- -1234                # same idea; the -- stops -1234 being parsed as a flag

killall nginx                # signal every process whose COMMAND NAME matches "nginx" (SIGTERM by default)
killall -9 nginx              # SIGKILL every nginx process by name

pkill -f "manage.py runserver"   # match against the FULL command line, not just the process name
pkill -9 -u deploy                # SIGKILL every process owned by user "deploy"
```

**`kill` vs `killall` vs `pkill`:** `kill` targets PIDs only. `killall` matches by exact process/command **name** (careful: on some systems, `killall` with no arguments has historically had catastrophic "kill everything" behavior on non-Linux UNIXes — on Linux it's safe and name-scoped, but the ambiguity is why some teams standardize on `pkill`). `pkill` matches by regex against the process name or, with `-f`, the **entire command line** — much more precise when several processes share a generic name like `python` and you need to target one specific script.

## Killing a process tree correctly

Killing a single PID does **not** kill its children — they simply get orphaned and re-parented to PID 1 (see file 1), which is rarely what you want when you meant to stop an entire job (e.g., a parent script that spawned several worker subprocesses).

Correct approaches, in order of precision:

```bash
# 1. If the process and its children share a process group (common for a job started
#    at an interactive or script shell prompt), signal the whole group with a negative PGID:
kill -TERM -$(ps -o pgid= <PID> | tr -d ' ')

# 2. pstree can show you the tree and, on some systems, kill it directly:
pstree -p 1234                 # visualize PID 1234 and all descendants, with PIDs shown
kill $(pstree -p 1234 | grep -oE '\([0-9]+\)' | tr -d '()')   # kill every PID in the tree

# 3. If you control how the job was launched, run it in its own process group deliberately
#    and kill that group — e.g. via `setsid` or a shell's `set -m` + background job PGID.
```

**Why this matters in real incidents:** a runaway build script that forked a dozen workers, killed only at the top-level PID, leaves the workers running — still consuming CPU/memory, now orphaned under `init`, invisible unless you specifically go looking with `ps --forest` or `pstree`. This is one of the most common causes of "I killed it but the server is still slow."

## Job control: `&`, `nohup`, `disown`, `jobs`, `fg`, `bg`

Your interactive shell tracks background **jobs** started in the current session:

```bash
long_task.sh &          # run in the background immediately; shell prints a job number and PID
jobs                     # list background jobs in this shell session, with job numbers
fg %1                    # bring job 1 to the foreground
bg %1                    # resume a stopped job (e.g., Ctrl-Z'd) in the background
Ctrl-Z                   # suspend the current foreground job (sends SIGTSTP, catchable version of SIGSTOP)
kill %1                   # signal by job number instead of PID
```

**The problem `&` alone doesn't solve:** a background job started with `&` is still tied to the shell's session. When the shell exits (you log out, your SSH connection drops), the kernel sends **`SIGHUP`** to every process in that session — and by default, that terminates your background job too, mid-execution.

Two complementary tools solve this:

- **`nohup`** — makes the process **ignore `SIGHUP`** specifically (it sets the signal's disposition to ignored before running your command). It also, by default, redirects stdout/stderr to `nohup.out` if you haven't redirected them yourself, since the terminal they'd write to will be gone.
  ```bash
  nohup ./long_task.sh > output.log 2>&1 &
  ```
- **`disown`** — a shell built-in that removes a job from **the shell's own job table**, so the shell no longer considers it a job it's responsible for and won't send it anything when the shell exits. This doesn't touch signal dispositions at all (unlike `nohup`) — it just detaches the job from the shell's bookkeeping.
  ```bash
  long_task.sh &
  disown            # detach the most recent background job from this shell's job table
  disown -h %1       # variant: keep it in `jobs` list but exempt it from the SIGHUP-on-exit
  ```

**`nohup` vs `disown` — the real difference:** `nohup` changes the *process's* behavior (immunizes it against `SIGHUP` before it even starts) — you must invoke it **at launch time**. `disown` changes the *shell's* bookkeeping about an already-running job — you use it **after** backgrounding something you forgot to `nohup`. In practice: use `nohup ... &` when you know upfront you want a job to survive logout; use `disown` as a rescue for a job you already started with plain `&` and now realize you need to walk away from. For anything you expect to run for a long time unattended in production, prefer a real process supervisor (systemd unit, `tmux`/`screen` session, or a job queue) over ad hoc `nohup`/`disown` — those are better for quick interactive use, not for anything that needs monitoring or restart-on-failure.

## Points to Remember

- `SIGTERM` (15) is catchable/graceful and the default for plain `kill`; `SIGKILL` (9) is unconditional, kernel-enforced, and skips all application cleanup. Always try `SIGTERM` first.
- `SIGKILL` and `SIGSTOP` can never be caught, blocked, or ignored by any process — this is enforced by the kernel itself, not a convention.
- `SIGHUP` originally meant "terminal hung up" but is widely repurposed by daemons today to mean "reload configuration without restarting" (`kill -HUP <pid>` on nginx/sshd).
- Killing a parent PID does not kill its children — they're orphaned, not terminated. Use a negative PGID (`kill -TERM -PGID`) or `pstree` to target an entire tree.
- `nohup` makes a process immune to `SIGHUP` from the moment it starts (must be used at launch); `disown` detaches an already-backgrounded job from the current shell's job table (used after the fact). They solve the same underlying problem from different ends.
- `pkill -f` matches against the full command line, not just the process name — essential when several processes share a generic executable name.

## Common Mistakes

- Reaching for `kill -9` as the first move instead of `SIGTERM` — skips graceful shutdown, can corrupt in-progress writes or leave locks/temp files behind.
- Assuming `kill -9 <pid>` always instantly kills — it does nothing while the target is in uninterruptible `D` state (see file 1); this is often misdiagnosed as "the kill command is broken."
- Killing only the top-level PID of a multi-process job and assuming everything stopped, while forked children live on as orphans (visible only via `ps --forest`/`pstree`, invisible in a plain `ps aux | grep jobname` if the name differs).
- Backgrounding a long job with plain `&` over SSH, disconnecting, and having it die — forgetting that closing the session sends `SIGHUP` to everything in that session unless `nohup`/`disown` was used.
- Confusing `nohup` and `disown` as interchangeable — `nohup` must be applied at launch; slapping `disown` on a job doesn't retroactively make it ignore `SIGHUP`... though in practice `disown` removes the shell's own habit of hupping its jobs on exit, so the visible symptom (survives logout) looks the same even though the mechanism differs. Not knowing the mechanism becomes a problem the moment `nohup.out` fills a disk unexpectedly, or a job needed actual `SIGHUP`-immunity for a different reason (e.g., surviving a *terminal* app crash, not a shell exit).
- Using `killall` on a name shared by unrelated processes (e.g., `killall python`) and taking down more than intended — `pkill -f` with a specific command-line pattern is safer.
