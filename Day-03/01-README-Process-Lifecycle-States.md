# Day 3 — Processes & System: Process Lifecycle & States

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:**

## Brief

Every piece of software you'll ever operate — a web server, a CI runner, a Kubernetes pod's main container, a database — is, from the kernel's point of view, just a **process**. Before you can debug "why is this container using 100% CPU" or "why won't this pod terminate," you need a correct mental model of what a process actually *is*, how it's born, how it dies, and what the states in between mean. This is asked constantly in interviews in disguised form ("what's a zombie process and why does it happen?", "what's the difference between a zombie and an orphan?") precisely because it separates people who've only run commands from people who understand what's happening underneath them.

This day is split into three focused files:

1. **This file** — what a process is, how it's created (`fork`/`exec`), the process state machine, PID/PPID, zombies vs. orphans, process groups and sessions.
2. **[02-README-Monitoring-Tools.md](02-README-Monitoring-Tools.md)** — `ps`, `top`, `htop`, `lsof`, `strace`, `nice`/`renice` in depth.
3. **[03-README-Signals-Job-Control.md](03-README-Signals-Job-Control.md)** — signals (`SIGTERM` vs `SIGKILL` vs `SIGHUP`), `kill`/`killall`/`pkill`, and job control (`&`, `nohup`, `disown`, `jobs`/`fg`/`bg`).

## What a process actually is

A process is a running instance of a program: an address space (memory), a set of open file descriptors, CPU register state, and bookkeeping the kernel maintains in a `task_struct` (visible to you as an entry under `/proc/<pid>/`). Every process has:

- A **PID** (process ID) — unique while the process is alive.
- A **PPID** (parent process ID) — the PID of whoever created it.
- A **PGID** (process group ID) and **SID** (session ID) — covered below.
- A **UID/GID** it runs as, determining what it's permitted to touch.

Process **1** (`init`, or `systemd` on most modern distros) is the ancestor of every other process on the system — everything traces back to it via the PPID chain. In a container, PID 1 is whatever the container's entrypoint is; this matters because PID 1 has special signal-handling semantics (see file 3) and because a container image without a real init system can accumulate zombies with nothing to reap them.

## How a process is born: `fork()` + `exec()`

On Linux, you cannot directly "create a new program running." Instead:

1. **`fork()`** — an existing process clones itself. The child gets a near-identical copy of the parent's memory (via copy-on-write, so it's cheap, not a real duplication), same open file descriptors, but a **new PID**. Immediately after `fork()`, you have two processes running the *same* code.
2. **`exec()`** (a family: `execve`, `execvp`, etc.) — the child then **replaces** its own memory image with a different program's code. The PID stays the same; everything the process *was running* changes.

This is why `bash script.sh` shows up as one process, but a subshell `(cmd)` or a pipeline `cmd1 | cmd2` forks multiple processes — every `|` in a pipeline is a separate `fork()`+`exec()`, all children of your shell, all running concurrently and connected by pipes.

**Why this matters operationally:** when a shell script does `export VAR=value` before invoking another program, that variable is inherited *only* because `exec()` preserves the environment across the replacement — child processes inherit a **copy** of the parent's environment at the moment of fork, not a live link. Change a variable in a child after fork, and the parent never sees it.

## The process state machine

Every process is in exactly one state at any instant, visible as the `STAT`/`S` column in `ps`:

| State | Symbol | Meaning |
|---|---|---|
| Running | `R` | Actually executing on a CPU, or ready to run and waiting for a CPU timeslice (the scheduler's run queue) |
| Sleeping (interruptible) | `S` | Waiting on an event (I/O, a timer, a lock) but can be woken by a signal at any time — the overwhelming majority of idle processes sit here |
| Uninterruptible sleep | `D` | Waiting on a low-level I/O operation (typically disk or NFS) that the kernel considers non-interruptible — **cannot be killed, not even with `SIGKILL`**, until the I/O completes or times out |
| Stopped | `T` | Suspended by a job-control signal (`SIGSTOP`, `SIGTSTP`) or because it's being traced (`ptrace`/debugger); resumes on `SIGCONT` |
| Zombie | `Z` | Has finished executing (exited) but its entry still exists in the process table because its **exit status hasn't been collected yet** |

`ps` often appends modifier letters: `s` (session leader), `+` (foreground process group), `l` (multi-threaded), `<` (high priority), `N` (low priority/niced).

**Why `D` state matters in production:** a process stuck in `D` (e.g., waiting on a hung NFS mount or a failing disk) will not respond to `kill -9`. This is the classic case where `kill -9` "doesn't work" — it's not that the signal failed, it's that the kernel won't deliver *any* signal until the uninterruptible syscall returns. The only real fixes are resolving the underlying I/O stall (fix the NFS server, wait it out) or, in the worst case, rebooting.

## Zombies vs. orphans — the interview-critical distinction

These are opposite failure modes and get mixed up constantly:

- **Zombie (`Z`)**: the **child has died**, but the **parent is still alive** and hasn't called `wait()`/`waitpid()` to collect its exit status. The kernel keeps a minimal entry (PID, exit code) around specifically so the parent *can* retrieve it later. A zombie consumes no memory or CPU — just a slot in the process table — but it is not killable, because it isn't running anything; there's nothing a signal can act on. The only cure is for the parent to reap it (call `wait()`) or for the parent to die (see below).
- **Orphan**: the **parent has died** while the child is still running. The kernel automatically **re-parents** the orphan to PID 1 (`init`/`systemd`), which is specifically designed to periodically call `wait()` on all its children — including ones it didn't originally start. So orphaned processes get cleaned up automatically once they eventually exit; they don't pile up.

**The link between them:** a zombie only becomes a permanent problem if its parent never reaps it and also never exits. If a buggy parent process is long-running (e.g., a custom process supervisor with a bug in its `SIGCHLD`/`wait()` handling) and keeps spawning short-lived children without reaping them, zombies accumulate under that parent indefinitely. Killing the *parent* clears them immediately — the zombies get re-parented to init, which reaps already-dead processes instantly.

## Process groups and sessions

Two more layers of grouping matter for job control (details in file 3), but the concepts belong here:

- A **process group** (PGID) is a set of related processes — typically a pipeline (`cmd1 | cmd2 | cmd3` are all one process group) — that can be signaled together as a unit.
- A **session** (SID) is a set of process groups associated with a single controlling terminal, created when you log in. The **session leader** is usually your login shell.
- Exactly one process group per session is the **foreground** group (the one connected to the terminal's input/Ctrl-C); the rest are **background** groups.

This is why `kill -TERM -1234` (note the **minus sign** before the PID) sends the signal to every process in **process group** 1234, not just PID 1234 — critical for killing an entire pipeline or a shell and all its children at once, instead of orphaning the children of the process you killed.

## Points to Remember

- A process is created by `fork()` (duplicate) then usually `exec()` (replace the code) — the PID survives `exec()`, only the running program changes.
- States: `R` running/runnable, `S` interruptible sleep, `D` uninterruptible sleep (**not killable, even by `SIGKILL`**), `T` stopped, `Z` zombie.
- A **zombie** = dead child, exit status not yet collected by the parent. Consumes a process-table slot, zero CPU/memory, cannot be signaled.
- An **orphan** = parent died first; child gets re-parented to PID 1, which reaps it automatically when it eventually exits. Orphans are self-resolving; zombies are not (unless their parent dies or reaps them).
- `kill -9 <pid>` on a zombie does nothing — there's no running process left to signal.
- `-<PGID>` (negative number) in `kill` targets an entire process group, not a single PID.

## Common Mistakes

- Trying to `kill -9` a zombie process and being confused when it "won't die" — a zombie is already dead; what you're seeing is just the unreaped exit-status entry. Fix the parent, don't fight the zombie.
- Assuming `D` state processes can always be killed with enough force (`kill -9`, repeated signals). They cannot — you must resolve the blocking I/O or reboot.
- Confusing zombie with orphan in an interview answer — they are caused by opposite parent/child death orders and have opposite remediation (kill the parent vs. nothing to do, it self-resolves).
- Believing a zombie process is "wasting resources" and urgently needs cleanup — it costs essentially nothing (no CPU, negligible memory); the real bug worth fixing is the parent that isn't reaping its children, not the zombie itself.
- Forgetting that environment variables set in a child after `fork()` never propagate back to the parent — a common source of "why didn't my `export` inside a subshell/background job take effect" confusion.
