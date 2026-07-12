# Day 15 — Python for Automation I: Filesystem & CLI Tooling

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

Almost every automation script touches the filesystem (reading configs, writing logs, walking directories for files to process) and almost every automation script is invoked from a command line with flags and arguments. Doing both with legacy stdlib patterns (`os.path` string-joining, hand-rolled `sys.argv` parsing) produces brittle, unreadable tools. This file covers the modern, idiomatic way to do both: `pathlib` for filesystem work, and `click` for CLIs — the combination you'll actually reach for when building real internal tooling, and what most production DevOps Python codebases standardize on today.

## `os` and `os.path` — the legacy way

The `os` module (and its `os.path` submodule) is how filesystem work was done before Python 3.4. It's still everywhere in older code and in the stdlib itself, so you need to recognize it, but new code should rarely start here.

```python
import os

path = os.path.join("/var", "log", "app.log")     # string concatenation with OS-correct separators
exists = os.path.exists(path)
is_dir = os.path.isdir("/var/log")
files = os.listdir("/var/log")                     # returns plain strings, no metadata
for root, dirs, files in os.walk("/etc"):           # recursive directory walk
    for f in files:
        full = os.path.join(root, f)

env_val = os.environ.get("HOME")
os.makedirs("/tmp/a/b/c", exist_ok=True)             # exist_ok avoids FileExistsError on rerun
```

`os.path.join` is correct and still used (even `pathlib` doesn't remove the *need* for OS-aware separators), but everything is strings — no method chaining, no type that represents "a path" as an object, and you constantly re-derive things (extension, parent, stem) with separate `os.path.splitext`, `os.path.dirname`, `os.path.basename` calls instead of attributes.

## `pathlib` — the modern, preferred way

Since Python 3.4 (and solidified as the idiomatic default well before 3.11), `pathlib.Path` represents a filesystem path as an **object**, not a string, with methods and operators that compose naturally:

```python
from pathlib import Path

log_dir = Path("/var/log")
log_file = log_dir / "app.log"          # `/` operator joins paths — reads like the path itself

log_file.exists()
log_file.is_file()
log_file.parent                          # Path("/var/log")
log_file.name                            # "app.log"
log_file.stem                            # "app"
log_file.suffix                          # ".log"
log_file.with_suffix(".json")            # Path("/var/log/app.json") — swap extension cleanly

log_file.read_text()                     # read whole file as str, no manual open()/close()
log_file.write_text("hello\n")           # write whole file, creates/truncates

for conf in Path("/etc").glob("*.conf"):        # non-recursive glob
    print(conf)
for conf in Path("/etc").rglob("*.conf"):       # recursive glob (like find)
    print(conf)

Path("/tmp/build/output").mkdir(parents=True, exist_ok=True)   # like mkdir -p
```

Why this matters beyond style:
- **Cross-platform correctness** is automatic — `Path` uses the right separator (`/` vs `\`) based on the OS without you thinking about it, and `PurePath`/`PureWindowsPath`/`PurePosixPath` let you manipulate paths for a *different* OS than the one you're running on (useful for tools that generate configs for other platforms).
- **Chaining reads naturally**: `Path(__file__).resolve().parent.parent / "config" / "settings.yaml"` is far more legible than the equivalent nested `os.path.join`/`os.path.dirname` calls.
- **Type safety** — a function that takes a `Path` instead of `str` documents intent, and modern libraries (including much of the stdlib itself, and most CLI frameworks) accept `Path` objects directly.
- `os.path` functions still work fine and `pathlib.Path` objects are largely interchangeable with strings in most APIs (they implement `__fspath__`), so migration is incremental — you don't need to rewrite everything at once, but new code should default to `pathlib`.

| Task | `os`/`os.path` | `pathlib` |
|---|---|---|
| Join paths | `os.path.join(a, b)` | `a / b` |
| Check exists | `os.path.exists(p)` | `p.exists()` |
| Is directory | `os.path.isdir(p)` | `p.is_dir()` |
| List directory | `os.listdir(p)` | `p.iterdir()` |
| Recursive walk | `os.walk(p)` | `p.rglob("*")` |
| Get extension | `os.path.splitext(p)[1]` | `p.suffix` |
| Read whole file | `open(p).read()` | `p.read_text()` |
| Make dirs | `os.makedirs(p, exist_ok=True)` | `p.mkdir(parents=True, exist_ok=True)` |
| Absolute path | `os.path.abspath(p)` | `p.resolve()` |

`os` is still the right tool for things that aren't inherently about *paths*: process/environment info (`os.environ`, `os.getpid()`, `os.cpu_count()`), permission bits (`os.chmod`, `os.access`), and low-level file descriptor operations.

## Building CLIs: `argparse`

`argparse` is stdlib (no dependency needed) and handles the basics: positional args, optional flags, type coercion, auto-generated `--help`.

```python
import argparse

parser = argparse.ArgumentParser(description="Check disk usage on remote hosts.")
parser.add_argument("hosts", nargs="+", help="Hostnames to check")
parser.add_argument("--threshold", type=int, default=80, help="Alert threshold percent")
parser.add_argument("--verbose", action="store_true", help="Enable debug logging")

args = parser.parse_args()
print(args.hosts, args.threshold, args.verbose)
```

Subcommands (like `git commit`, `git push`) require the more verbose `add_subparsers()` API:

```python
parser = argparse.ArgumentParser()
subparsers = parser.add_subparsers(dest="command", required=True)

check_parser = subparsers.add_parser("check", help="Check disk usage")
check_parser.add_argument("--host", required=True)

report_parser = subparsers.add_parser("report", help="Generate a report")
report_parser.add_argument("--format", choices=["json", "text"], default="text")

args = parser.parse_args()
if args.command == "check":
    ...
```

This works, but it's verbose, imperative, and testing it means constructing `sys.argv`-like input by hand — there's no clean way to invoke "the check subcommand's logic" without going through the whole parser.

## Building CLIs: `click` — preferred for real tools

`click` (a third-party package, `pip install click`) uses **decorators** to declare a CLI's shape, which is more declarative, more composable, and easier to test than `argparse`.

```python
import click

@click.command()
@click.option("--host", "hosts", multiple=True, required=True, help="Server to check (repeatable)")
@click.option("--threshold", default=80, show_default=True, type=int, help="Alert threshold percent")
@click.option("--verbose", is_flag=True, help="Enable debug logging")
def check(hosts, threshold, verbose):
    """Check disk usage on one or more servers."""
    for host in hosts:
        click.echo(f"Checking {host}...")

if __name__ == "__main__":
    check()
```

Subcommands, the way `click` is meant to be used, via **groups**:

```python
@click.group()
def cli():
    """diskcheck: disk usage automation tool."""

@cli.command()
@click.option("--host", required=True)
def check(host):
    """Check a single host."""
    click.echo(f"checking {host}")

@cli.command()
@click.argument("logfile", type=click.Path(exists=True))
def report(logfile):
    """Summarize a log file."""
    click.echo(f"reporting on {logfile}")

if __name__ == "__main__":
    cli()
```

Run as `python tool.py check --host web1` or `python tool.py report app.log`.

### Why `click` over `argparse` for real DevOps tools

- **Built-in type system**: `click.Path(exists=True)`, `click.Choice([...])`, `click.IntRange(min, max)` validate and convert input declaratively — `argparse` requires manual `type=` callables and manual range checks.
- **Composable groups and nesting**: multi-level subcommand trees (`tool db migrate`, `tool db seed`) are natural with `click.Group`; `argparse`'s subparser nesting gets unwieldy fast.
- **Testability**: `click.testing.CliRunner` lets you invoke commands in-process and assert on output/exit codes without spawning a subprocess or touching real `sys.argv`:
  ```python
  from click.testing import CliRunner

  def test_check_alerts_over_threshold():
      runner = CliRunner()
      result = runner.invoke(check, ["--host", "web1", "--threshold", "50"])
      assert result.exit_code == 1
      assert "ALERT" in result.output
  ```
- **Automatic `--help`**, colored output helpers (`click.style`, `click.secho`), confirmation prompts (`click.confirm`), progress bars (`click.progressbar`) — all the polish real internal tools need, without hand-rolling it.
- **Ecosystem convention**: most modern Python CLI tools (`pip`, `black`, `flask`, `pip-tools`, many internal DevOps tools) are built on `click` or its cousin `typer` (which adds type-hint-driven definitions on top of click) — familiarity transfers directly.

`argparse` remains the right choice for small, single-file, dependency-free scripts (e.g., something meant to run with only the stdlib on a locked-down box) or quick internal scripts where pulling in a dependency isn't worth it. For anything you'll maintain, test, and hand to teammates, `click` is worth the dependency.

| | `argparse` | `click` |
|---|---|---|
| Dependency | stdlib, none needed | third-party (`pip install click`) |
| Style | imperative (`parser.add_argument(...)`) | declarative (decorators) |
| Subcommands | `add_subparsers()`, verbose | `@click.group()`, natural nesting |
| Validation | manual `type=`/`choices=` | rich built-in types (`Path`, `Choice`, `IntRange`) |
| Testing | manual `sys.argv` patching | `CliRunner` — in-process, no subprocess |
| Best for | tiny scripts, zero-dependency constraints | real, maintained internal tools |

## Points to Remember

- `pathlib.Path` is the modern default for filesystem work — object-oriented, chainable, cross-platform-correct by construction; `os.path` still works but produces more verbose, string-based code.
- The `/` operator on `Path` joins paths; `.parent`, `.name`, `.stem`, `.suffix` replace separate `os.path.dirname`/`splitext`/`basename` calls.
- `os` is still the right module for non-path OS interaction: environment variables, process info, permission bits, file descriptors.
- `argparse` is stdlib and fine for small scripts; `click` (decorator-based, richer validation, `CliRunner` for testing, natural subcommand groups) is preferred for real, maintained CLI tools.
- `click.Path(exists=True)` and similar typed options validate input at parse time, before your function body even runs — fail fast instead of raising a raw exception deep in your logic.

## Common Mistakes

- Building paths with plain string concatenation (`base_dir + "/" + filename`) instead of `Path` or `os.path.join` — breaks on Windows and is fragile around missing/duplicate slashes.
- Using `Path.glob("**/*.conf")` when you meant `Path.rglob("*.conf")` (or vice versa) — `glob("**/...")` requires the `**` pattern explicitly and behaves differently from `rglob`, which always recurses.
- Forgetting `parents=True` on `Path.mkdir()` when the parent directory doesn't exist yet, causing a `FileNotFoundError` instead of creating the full chain like `mkdir -p`.
- Forgetting `exist_ok=True` on `Path.mkdir()` in scripts that may be re-run, causing a `FileExistsError` on the second run instead of idempotent behavior.
- Writing a growing internal tool with raw `argparse` subparsers past 2-3 subcommands, ending up with hundreds of lines of imperative parser wiring that `click` groups would express in a fraction of the code.
- Testing a `click` command by shelling out to `python tool.py ...` in a subprocess for every test case instead of using `CliRunner.invoke()`, making the test suite far slower and harder to debug than it needs to be.
- Mixing `str` and `Path` inconsistently through a codebase (some functions take `str`, others `Path`), forcing constant `str(path)` / `Path(path)` conversions at boundaries — pick one (prefer `Path`) and convert only at the edges (e.g., when calling `subprocess` on older Python versions that need strings, though modern `subprocess` accepts `Path` objects directly).
