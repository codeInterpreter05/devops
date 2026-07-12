# Day 15 — Lab: Python for Automation I

**Goal:** Build a real, safe, loggable CLI tool that checks disk usage across servers over SSH and alerts on a threshold — while separately drilling the underlying pieces (`subprocess` safety, `pathlib`, `argparse`/`click`, `logging`, venv/`pyproject.toml`) so each is muscle memory, not just something you copy-pasted into the final tool.

**Prerequisites:**
- Python 3.11+ installed (`python3 --version`).
- At least one SSH target you can reach without a password prompt (key-based auth). Options, easiest first:
  - **macOS**: enable Remote Login (System Settings → General → Sharing → Remote Login) and SSH to `localhost` or `127.0.0.1` using your own SSH key (`ssh-copy-id $(whoami)@localhost` if not already set up).
  - **Linux**: SSH to `localhost` the same way, or spin up a throwaway SSH-enabled container: `docker run -d --name sshtest -p 2222:22 -e PUBLIC_KEY="$(cat ~/.ssh/id_ed25519.pub)" lscr.io/linuxserver/openssh-server` and connect with `ssh -p 2222 linuxserver.io@localhost`.
  - A real lab VM or cloud instance you already have key-based access to.
- `pip` available for installing `click`, `ruff`, `mypy` into a virtual environment (created in Lab 1).

---

### Lab 1 — Project scaffold: venv + `pyproject.toml`

1. Create a project directory and a virtual environment:
   ```bash
   mkdir -p ~/diskcheck && cd ~/diskcheck
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install --upgrade pip
   ```
2. Create `pyproject.toml`:
   ```toml
   [project]
   name = "diskcheck"
   version = "0.1.0"
   description = "Check disk usage across servers via SSH"
   requires-python = ">=3.11"
   dependencies = ["click>=8.1"]

   [project.optional-dependencies]
   dev = ["ruff>=0.4", "mypy>=1.8"]

   [build-system]
   requires = ["setuptools>=68"]
   build-backend = "setuptools.build_meta"

   [tool.ruff]
   line-length = 100
   target-version = "py311"

   [tool.mypy]
   python_version = "3.11"
   disallow_untyped_defs = true
   ```
3. Install the project in editable mode with dev dependencies:
   ```bash
   pip install -e ".[dev]"
   ```
4. Confirm isolation: run `which python` and `which pip` — both should point inside `.venv/`, not your system Python. Deactivate (`deactivate`) and run `which python` again to see it fall back to system Python.

**Success criteria:** `pip list` inside the activated venv shows `click`, `ruff`, and `mypy` installed, and `which python` resolves to a path inside `~/diskcheck/.venv/`. Reactivating (`source .venv/bin/activate`) restores that state instantly.

---

### Lab 2 — `subprocess` safely: see the injection risk, then fix it

1. Create `injection_demo.py`:
   ```python
   import subprocess

   # Simulate untrusted input (e.g., from an API response or user form field)
   untrusted_hostname = "localhost; echo INJECTED > /tmp/pwned.txt"

   # DANGEROUS — do not run this pattern against real infrastructure
   subprocess.run(f"ping -c 1 {untrusted_hostname}", shell=True)
   ```
2. Run it: `python injection_demo.py`, then check `cat /tmp/pwned.txt` — you'll see `INJECTED`, proving the semicolon broke out of the intended `ping` command and ran an arbitrary second command.
3. Now fix it with list-form arguments (`shell=False`, the default):
   ```python
   import subprocess

   untrusted_hostname = "localhost; echo INJECTED > /tmp/pwned2.txt"
   result = subprocess.run(["ping", "-c", "1", untrusted_hostname], capture_output=True, text=True)
   print(result.returncode, result.stderr.strip())
   ```
   Run it, then check `ls /tmp/pwned2.txt` — it should **not** exist. The entire string, semicolon included, was passed to `ping` as one literal (invalid) hostname argument, and `ping` simply failed to resolve it.
4. Write a third version using `shlex.split()` to safely turn a trusted, fixed command string into an argument list, and add `timeout=5` and `check=False` handling for the non-zero exit:
   ```python
   import shlex
   import subprocess

   cmd = shlex.split("df -h /")
   result = subprocess.run(cmd, capture_output=True, text=True, timeout=5)
   print(result.stdout)
   ```

**Success criteria:** You've reproduced a real shell injection with `shell=True` + string interpolation, confirmed the list-argument form neutralizes it, and can explain in one sentence why: no shell ever parses the string with `shell=False`, so metacharacters like `;` have no special meaning.

---

### Lab 3 — `pathlib` in practice

1. In your `~/diskcheck` project, write `disk_scan.py` that recursively finds every `.log` file under `/var/log` (or `/tmp` if you lack permissions there) and reports total size:
   ```python
   from pathlib import Path

   root = Path("/var/log")
   log_files = list(root.rglob("*.log"))
   total_bytes = sum(f.stat().st_size for f in log_files if f.is_file())

   print(f"Found {len(log_files)} .log files under {root}")
   print(f"Total size: {total_bytes / 1024:.1f} KB")

   largest = max(log_files, key=lambda f: f.stat().st_size, default=None)
   if largest:
       print(f"Largest: {largest} ({largest.stat().st_size / 1024:.1f} KB)")
   ```
2. Extend it to write a summary report to a new file using `pathlib` only (no `open()`):
   ```python
   report = Path("log_report.txt")
   report.write_text(f"Scanned {root}: {len(log_files)} files, {total_bytes / 1024:.1f} KB total\n")
   print(report.read_text())
   ```
3. Practice path manipulation without touching disk: given `Path("/var/log/nginx/access.log")`, derive `nginx.access.log.bak` in the *current* directory using `.name`, `.parent`, and `.with_suffix()` — write the one-liner and confirm the output path.

**Success criteria:** Your script runs without any `os.path` calls, correctly reports file count/size using only `Path` methods, and you can explain what `.stem`, `.suffix`, and `.with_suffix()` each return for a multi-dot filename like `access.log.gz`.

---

### Lab 4 — CLI tooling: build the same tiny tool twice

1. Build a `greet.py` CLI using **`argparse`** that accepts a required `--name` and an optional `--times` (default 1), printing the greeting that many times:
   ```python
   import argparse

   parser = argparse.ArgumentParser()
   parser.add_argument("--name", required=True)
   parser.add_argument("--times", type=int, default=1)
   args = parser.parse_args()

   for _ in range(args.times):
       print(f"Hello, {args.name}!")
   ```
   Run: `python greet.py --name Sarthak --times 3`.
2. Rebuild the identical tool as `greet_click.py` using **`click`**:
   ```python
   import click

   @click.command()
   @click.option("--name", required=True)
   @click.option("--times", type=int, default=1, show_default=True)
   def greet(name, times):
       for _ in range(times):
           click.echo(f"Hello, {name}!")

   if __name__ == "__main__":
       greet()
   ```
   Run: `python greet_click.py --name Sarthak --times 3`, then `python greet_click.py --help` and compare the auto-generated help text to argparse's.
3. Write a test for the click version using `CliRunner`, in `test_greet_click.py`:
   ```python
   from click.testing import CliRunner
   from greet_click import greet

   def test_greet_default_times():
       result = CliRunner().invoke(greet, ["--name", "Test"])
       assert result.exit_code == 0
       assert result.output.count("Hello, Test!") == 1

   def test_greet_requires_name():
       result = CliRunner().invoke(greet, [])
       assert result.exit_code != 0
   ```
   Run it: `pip install pytest && pytest test_greet_click.py -v`.

**Success criteria:** Both CLIs behave identically from the outside; you can articulate one concrete reason the `click` version was less code or easier to test than the `argparse` version (the `CliRunner` test with no subprocess involved is the clearest example).

---

### Lab 5 — Logging done properly

1. Create `logging_demo.py` with a named logger, two handlers (console at `INFO`, file at `DEBUG`), and a formatter that includes timestamp and level:
   ```python
   import logging

   logger = logging.getLogger("diskcheck")
   logger.setLevel(logging.DEBUG)
   logger.propagate = False

   fh = logging.FileHandler("diskcheck.log")
   fh.setLevel(logging.DEBUG)
   fh.setFormatter(logging.Formatter("%(asctime)s %(levelname)-8s %(message)s"))

   ch = logging.StreamHandler()
   ch.setLevel(logging.INFO)
   ch.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

   logger.addHandler(fh)
   logger.addHandler(ch)

   logger.debug("This should only be in the file")
   logger.info("This appears in both")
   logger.warning("ALERT-style message, both destinations")
   ```
2. Run it, then `cat diskcheck.log` — confirm the DEBUG line is present in the file but was not printed to your console.
3. Change `logger.propagate = False` to `True` (or delete the line — `True` is the default), add `logging.basicConfig(level=logging.INFO)` at the very top of the script, rerun, and observe the `WARNING` line now appears **twice** in the console. Explain why in a comment, then revert the fix.

**Success criteria:** You can produce (on purpose) and then explain the duplicate-log-line bug caused by propagation to a configured root logger, and your final script's console only shows INFO+ while the file captures everything from DEBUG up.

---

### Lab 6 — The core hands-on activity: `diskcheck` CLI over SSH

This is the assigned hands-on activity for today. Build a real CLI tool combining everything above: `click` for the interface, `subprocess` (safe, list-form, with timeout) to run `df` over SSH, and `logging` to a file plus console.

1. In `~/diskcheck`, create `diskcheck/__init__.py` (empty) and `diskcheck/cli.py`:
   ```python
   """diskcheck: check disk usage across remote servers via SSH."""

   import logging
   import sys
   from pathlib import Path

   import click

   LOG_FILE = Path("diskcheck.log")


   def setup_logging(verbose: bool) -> logging.Logger:
       logger = logging.getLogger("diskcheck")
       logger.setLevel(logging.DEBUG if verbose else logging.INFO)
       logger.propagate = False
       if logger.handlers:          # avoid duplicate handlers if called twice (e.g. in tests)
           return logger

       file_handler = logging.FileHandler(LOG_FILE)
       file_handler.setFormatter(logging.Formatter(
           "%(asctime)s %(levelname)-8s %(message)s"
       ))
       console_handler = logging.StreamHandler()
       console_handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

       logger.addHandler(file_handler)
       logger.addHandler(console_handler)
       return logger


   def check_disk_usage(host: str, timeout: int, logger: logging.Logger) -> int | None:
       """SSH into host, run `df` on `/`, return usage percent, or None on failure."""
       import subprocess

       cmd = ["ssh", "-o", "BatchMode=yes", "-o", "ConnectTimeout=5", host, "df", "-h", "/"]
       try:
           result = subprocess.run(cmd, capture_output=True, text=True, timeout=timeout, check=True)
       except subprocess.TimeoutExpired:
           logger.error("Timed out connecting to %s", host)
           return None
       except subprocess.CalledProcessError as exc:
           logger.error("SSH to %s failed (exit %d): %s", host, exc.returncode, exc.stderr.strip())
           return None
       except FileNotFoundError:
           logger.error("ssh binary not found on PATH")
           return None

       lines = result.stdout.strip().splitlines()
       if len(lines) < 2:
           logger.error("Unexpected df output from %s: %r", host, result.stdout)
           return None

       fields = lines[1].split()
       try:
           return int(fields[4].rstrip("%"))
       except (IndexError, ValueError):
           logger.error("Could not parse usage percent from %s: %r", host, fields)
           return None


   @click.command()
   @click.option("--host", "hosts", multiple=True, required=True, help="Server to check (repeatable).")
   @click.option("--threshold", default=80, show_default=True, type=int, help="Alert threshold percent.")
   @click.option("--timeout", default=10, show_default=True, type=int, help="SSH timeout in seconds.")
   @click.option("--verbose", is_flag=True, help="Enable debug logging.")
   def main(hosts: tuple, threshold: int, timeout: int, verbose: bool) -> None:
       """Check root filesystem usage on one or more servers over SSH."""
       logger = setup_logging(verbose)
       exit_code = 0

       for host in hosts:
           logger.debug("Checking %s", host)
           usage = check_disk_usage(host, timeout, logger)
           if usage is None:
               exit_code = 2
               continue
           if usage >= threshold:
               logger.warning("ALERT: %s is at %d%% disk usage (threshold %d%%)", host, usage, threshold)
               exit_code = max(exit_code, 1)
           else:
               logger.info("%s OK: %d%% disk usage", host, usage)

       sys.exit(exit_code)


   if __name__ == "__main__":
       main()
   ```
2. Register it as an installable script in `pyproject.toml`:
   ```toml
   [project.scripts]
   diskcheck = "diskcheck.cli:main"
   ```
   Reinstall in editable mode: `pip install -e .`
3. Run it against your test host(s):
   ```bash
   diskcheck --host localhost --threshold 80 --verbose
   ```
   (Substitute your Docker container's `user@localhost -p 2222` style target if you're not using direct localhost SSH — you'll need to adapt the `cmd` list in step 1 to include `-p 2222` for that target, which is good practice for handling per-host connection options.)
4. Force an alert by lowering the threshold below your actual usage: `diskcheck --host localhost --threshold 1` — confirm you see an `ALERT` line, the process exits with code `1` (`echo $?` after running), and the same line is in `diskcheck.log`.
5. Force a failure path: run `diskcheck --host doesnotexist.invalid --timeout 3` — confirm it logs an `ERROR` (not a crash) and exits with code `2`.
6. Lint and type-check what you built:
   ```bash
   ruff check diskcheck/
   mypy diskcheck/
   ```
   Fix anything they flag (a common one: `hosts: tuple` should be `hosts: tuple[str, ...]` for `mypy` to be fully satisfied).

**Success criteria:** `diskcheck --host <target> --threshold 80` runs end-to-end over real SSH, correctly classifies OK/ALERT/ERROR per host, exits with a meaningful code (`0` all OK, `1` alert triggered, `2` a host errored), writes a timestamped record of every check to `diskcheck.log`, and passes `ruff check` with no errors.

---

### Cleanup

```bash
deactivate                       # leave the venv
docker rm -f sshtest 2>/dev/null # if you used the Docker SSH target
rm -f /tmp/pwned.txt /tmp/pwned2.txt
# Keep ~/diskcheck if you want to extend it later, or remove entirely:
# rm -rf ~/diskcheck
```

### Stretch challenge

Extend `diskcheck` to read its host list from a config file instead of repeated `--host` flags — accept `--hosts-file servers.txt` (one hostname per line, via `pathlib`), merge it with any `--host` flags given, de-duplicate, and skip blank lines/comments (`#`). Add a `--json` flag that, when set, emits a single JSON summary object (host → usage/status) to stdout instead of the human-readable log lines, so the tool can be piped into `jq` or another script — while still writing the full human-readable log to `diskcheck.log` regardless.
