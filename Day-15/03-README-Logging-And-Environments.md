# Day 15 — Python for Automation I: Logging & Environments

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Python DevOps | **Flag:** —

## Brief

A script that uses `print()` for diagnostics works fine on your laptop and falls apart the moment it runs unattended — in cron, in a CI job, in a systemd service, on someone else's machine. Real automation tools need structured, leveled, redirectable output (the `logging` module), a reproducible dependency environment (virtual envs), and a standard way to declare what the project needs and how it's configured (`pyproject.toml`). None of this is optional polish — it's the difference between a script and a tool other people (including future you) can run reliably and debug when it breaks at 3am.

## Why `print()` is wrong for real tools

`print()` has no concept of severity, no way to redirect output separately for "normal operation" vs "something's wrong," and no way to turn verbosity up or down without editing code. Concretely:

- You can't silence `print()` debug statements in production without deleting or commenting them out.
- You can't send warnings to stderr and info to stdout (or to a file) without manually branching every call.
- You can't attach a timestamp, module name, or severity level without reformatting every single call by hand.
- Third-party libraries you import can't hook into your `print()` calls to integrate with your output — but they *can* hook into `logging`, because it's the stdlib-standard mechanism every library is expected to use.

`logging` solves all of this with a small amount of one-time setup.

## The logging module: loggers, handlers, formatters

Three concepts compose to give you full control:

- **Logger** — the object you call `.debug()`/`.info()`/`.warning()`/`.error()`/`.critical()` on. Loggers are organized in a hierarchy by dotted name (`"myapp.disk"` is a child of `"myapp"`), and you almost always get one with `logging.getLogger(__name__)` rather than the anonymous root logger.
- **Handler** — decides *where* a log record goes: `StreamHandler` (console), `FileHandler` (a file), `RotatingFileHandler`/`TimedRotatingFileHandler` (a file that rotates by size/time), `SysLogHandler` (syslog), or handlers for external systems. A single logger can have multiple handlers (e.g., INFO+ to console, DEBUG+ to a file).
- **Formatter** — decides the *shape* of each line: timestamp, level, logger name, message.

```python
import logging

logger = logging.getLogger("diskcheck")
logger.setLevel(logging.DEBUG)   # logger's own floor — handlers can filter further, but never see below this

file_handler = logging.FileHandler("diskcheck.log")
file_handler.setLevel(logging.DEBUG)
file_handler.setFormatter(logging.Formatter(
    "%(asctime)s %(levelname)-8s %(name)s: %(message)s"
))

console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)     # console only shows INFO+, file keeps everything including DEBUG
console_handler.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

logger.addHandler(file_handler)
logger.addHandler(console_handler)

logger.debug("Connecting to %s", host)      # only goes to the file
logger.info("%s OK: %d%% used", host, pct)  # goes to both
logger.warning("ALERT: %s at %d%%", host, pct)
```

**Always use `%s`-style lazy formatting** (`logger.info("value: %s", val)`), not f-strings (`logger.info(f"value: {val}")`), inside log calls. With lazy formatting, the string interpolation only happens if the message will actually be emitted at the current log level — with an f-string, Python builds the string every time regardless of whether logging will use it, wasting work on DEBUG calls in a production system running at INFO level.

## Log levels — what they mean operationally

| Level | Numeric | When to use |
|---|---|---|
| `DEBUG` | 10 | Verbose internals useful only when actively troubleshooting — variable dumps, "entering function X" |
| `INFO` | 20 | Normal operational milestones — "checked host X, 42% used", "connected to database" |
| `WARNING` | 30 | Something unexpected but not fatal — a retry, a deprecated config key, threshold breach |
| `ERROR` | 40 | An operation failed and could not complete — a host unreachable, a file couldn't be written |
| `CRITICAL` | 50 | The whole program/service is in serious trouble — can't continue at all |

`logger.setLevel(X)` sets the floor: anything below `X` is dropped before reaching any handler. Each handler can *further* raise its own floor (as in the example above — the console handler drops DEBUG even though the logger allows it), but a handler can never see a level lower than what the logger itself allows through.

## `basicConfig()` vs manual setup — and the root logger trap

`logging.basicConfig(level=logging.INFO, format="...")` is a one-line convenience that configures the **root logger**. It's fine for a quick script, but it has a well-known trap: `basicConfig()` only has an effect the *first* time it's called in a process (subsequent calls are silently ignored unless you pass `force=True`), and if any imported library calls `logging.basicConfig()` before you do (or calls `logging.warning(...)` directly, which implicitly configures the root logger with defaults), your configuration may never take effect.

For anything beyond a throwaway script, get your own named logger (`logging.getLogger(__name__)` or a fixed app name) and configure handlers on it explicitly, as shown above, rather than relying on `basicConfig()` and the root logger. This also avoids **propagation surprises**: by default, a child logger's records propagate up to the root logger's handlers *too* — so without care you can end up with duplicate log lines (once from your handler, once from the root's). Set `logger.propagate = False` if you attach handlers directly to a non-root logger and don't want root's handlers to also fire.

## Virtual environments — why isolate

Every Python installation has one global `site-packages`. Without isolation, installing `click==8.1` for one project and `click==7.0` for another (or upgrading a shared dependency for one script) breaks the other. A **virtual environment** gives each project its own isolated `site-packages` and its own `pip`, so dependencies never collide across projects, and you can reproduce the exact environment a tool was built and tested against.

```bash
python3 -m venv .venv              # create a venv in .venv/ using the stdlib venv module
source .venv/bin/activate          # activate it (Windows: .venv\Scripts\activate)
python -m pip install --upgrade pip
pip install click ruff mypy
deactivate                          # leave the venv, back to system/global Python
```

`venv` is stdlib (no extra install needed) and is the standard baseline; tools like `virtualenv` (third-party, slightly faster/more features), `pipenv`, and `poetry` build on the same underlying concept but add dependency-locking and workflow features. For DevOps scripting work, plain `venv` plus `pip` and a `pyproject.toml` is usually all you need — reach for `poetry`/`pipenv` when a project needs reproducible lockfiles across a team.

**Never install project dependencies into system/global Python** (`sudo pip install ...`) — it risks breaking OS-level tools that depend on a specific Python/package version (many Linux distros ship Python-based system utilities), and it makes "what does this script actually need" undiscoverable. Always work inside an activated venv.

## `pyproject.toml` — the modern packaging & config standard

Historically, Python projects scattered configuration across `setup.py` (packaging/build logic as executable Python — a security and reproducibility smell since it runs arbitrary code just to install a package), `setup.cfg`, `requirements.txt` (a flat, unpinned-by-default list of dependencies with no place for tool config), and separate config files per tool (`.flake8`, `mypy.ini`, `pytest.ini`). **`pyproject.toml`** (formalized by PEP 518 for build-system declaration and PEP 621 for project metadata) consolidates all of this into one declarative, static TOML file:

```toml
[project]
name = "diskcheck"
version = "0.1.0"
description = "Check disk usage across servers via SSH"
requires-python = ">=3.11"
dependencies = [
    "click>=8.1",
]

[project.optional-dependencies]
dev = ["ruff>=0.4", "mypy>=1.8", "pytest>=8.0"]

[project.scripts]
diskcheck = "diskcheck.cli:main"    # installs a `diskcheck` command that calls cli.main()

[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]   # pycodestyle errors, pyflakes, isort, pyupgrade

[tool.mypy]
python_version = "3.11"
disallow_untyped_defs = true
warn_unused_ignores = true
```

Why this is the standard now: it's **declarative** (metadata is data, not executable code, unlike `setup.py`), **tool-agnostic** (any tool — `pip`, `build`, `ruff`, `mypy`, `pytest` — reads its own `[tool.X]` table from the same file), and it replaces the old `requirements.txt` for declaring dependencies while still letting you generate a pinned `requirements.txt` or lockfile for exact reproducibility if needed (`pip freeze > requirements.txt`, or a proper lockfile via `pip-tools`/`poetry`/`uv`). Install a project defined this way with `pip install .` (or `pip install -e .` for an editable/development install where code changes take effect without reinstalling).

## `ruff` — fast linting and formatting

`ruff` is a linter (and formatter) written in Rust that replaces the old stack of `flake8` + `isort` + `pyupgrade` + (increasingly) `black`, running 10-100x faster because it's not interpreted Python checking Python. Configured entirely under `[tool.ruff]` in `pyproject.toml`:

```bash
ruff check .          # lint (style violations, unused imports, likely bugs)
ruff check --fix .    # auto-fix what's safely fixable
ruff format .         # format code (black-compatible style)
```

## `mypy` — static type checking

Python is dynamically typed at runtime, but type hints (`def check(host: str, timeout: int = 10) -> bool:`) let `mypy` catch type errors **before** you run the code — critical for automation tools where a type mismatch (passing a `Path` where a function expects `str`, or forgetting a function can return `None`) might otherwise only surface when a specific code path runs in production.

```bash
mypy diskcheck/       # type-check a package
```

```python
def check_disk_usage(host: str, timeout: int) -> int | None:
    ...

usage = check_disk_usage("web1", 10)
if usage > 80:   # mypy flags this: usage could be None, comparing None > 80 is a TypeError at runtime
    ...
```

`mypy` forces you to be explicit about cases like the one above — the fix is `if usage is not None and usage > 80:` — catching a real bug statically instead of during an on-call incident.

## Points to Remember

- `print()` has no severity levels, no redirection, and can't be tuned without editing code — use `logging` for anything that runs unattended.
- Use `logger.info("value: %s", val)` lazy formatting, not f-strings, inside log calls — avoids wasted string building at suppressed levels.
- A logger's `setLevel()` is a hard floor; handlers can raise that floor further per-destination (e.g., DEBUG to file, INFO to console) but never lower it.
- Prefer named loggers (`logging.getLogger(__name__)`) with explicit handlers over relying on `basicConfig()` and the root logger — avoids the "someone else called basicConfig first" trap and duplicate-log-line propagation bugs.
- Virtual environments (`python -m venv .venv`) isolate per-project dependencies — never `sudo pip install` into system Python.
- `pyproject.toml` is the modern, declarative standard for both packaging metadata and per-tool configuration (`[tool.ruff]`, `[tool.mypy]`), replacing `setup.py` (executable, non-declarative) and scattered per-tool config files.
- `ruff` replaces `flake8`+`isort`+`pyupgrade` (and can replace `black`) with one fast tool; `mypy` catches type errors statically, before runtime.

## Common Mistakes

- Sprinkling `print()` statements through a "finished" automation tool instead of `logging` — works until the tool runs in cron/systemd where stdout isn't watched, and there's no way to distinguish routine output from a real problem.
- Using f-strings inside log calls (`logger.debug(f"host={host}")`) — builds the string even when DEBUG is suppressed, wasting CPU at scale across thousands of calls.
- Calling `logging.basicConfig()` and assuming it always takes effect — it's a no-op if the root logger already has handlers (e.g., an imported library configured it first), producing "my logging config isn't working" confusion.
- Attaching a handler to a custom logger without setting `propagate = False`, then wondering why every log line appears twice (once via your handler, once via the root logger's default handler).
- Installing project dependencies globally (`pip install click` outside any venv, or worse, `sudo pip install`) — works until two projects need conflicting versions, or a system tool that depends on Python breaks after an upgrade.
- Committing a `.venv/` directory to version control instead of `.gitignore`-ing it and committing `pyproject.toml` (and a lockfile if used) — bloats the repo and doesn't even guarantee reproducibility across OSes.
- Still maintaining a hand-written `requirements.txt` with no version pins alongside a `pyproject.toml`, so the two drift and it's unclear which one is authoritative.
- Writing `def get_usage(host) -> int:` when the function can actually return `None` on failure, then having `mypy` (if it were run) or a production `TypeError` catch what a proper `Optional[int]`/`int | None` annotation would have flagged during development.
