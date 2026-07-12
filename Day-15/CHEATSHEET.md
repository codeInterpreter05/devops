# Day 15 — Cheatsheet: Python for Automation I

## `subprocess`

```python
import subprocess

# The safe default: list of args, no shell involved
result = subprocess.run(
    ["df", "-h", "/"],
    capture_output=True,   # populate .stdout / .stderr
    text=True,             # decode to str, not bytes
    timeout=10,            # raises TimeoutExpired if exceeded
    check=True,            # raises CalledProcessError on non-zero exit
)
result.stdout            # captured stdout (str, because text=True)
result.stderr            # captured stderr
result.returncode        # 0 on success

# DANGEROUS pattern — never interpolate untrusted input into shell=True
subprocess.run(f"ping -c 1 {host}", shell=True)     # shell injection risk (CWE-78)

# Safe alternative for shell-string input you don't control the format of
import shlex
subprocess.run(shlex.split("rsync -avz src/ dest/"))  # shell=False by default
safe = shlex.quote(host)
subprocess.run(f"ping -c 1 {safe}", shell=True)       # only if shell features truly needed

# Older, narrower helpers (still fine for quick one-liners)
subprocess.check_output(["git", "rev-parse", "HEAD"], text=True).strip()
subprocess.check_call(["terraform", "apply", "-auto-approve"])

# Popen — streaming, concurrency, manual pipe control
proc = subprocess.Popen(["ping", "-c", "4", "host"], stdout=subprocess.PIPE, text=True)
for line in proc.stdout:
    print(line.rstrip())
proc.wait(timeout=15)

# Popen piping two processes safely (ps aux | grep python)
p1 = subprocess.Popen(["ps", "aux"], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["grep", "python"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()
out, _ = p2.communicate()          # use communicate(), never manual read()+write() (deadlock risk)

# Exception handling
try:
    subprocess.run(["ssh", host, "df", "-h"], timeout=10, check=True, capture_output=True, text=True)
except subprocess.TimeoutExpired:
    ...
except subprocess.CalledProcessError as exc:
    exc.returncode, exc.stdout, exc.stderr
except FileNotFoundError:
    ...   # binary itself missing/not on PATH (only with shell=False)

subprocess.STDOUT       # merge stderr into stdout
subprocess.DEVNULL      # discard output entirely
```

## `os` / `os.path`

```python
import os

os.path.join("/var", "log", "app.log")
os.path.exists(path); os.path.isdir(path); os.path.isfile(path)
os.listdir("/var/log")                # plain strings
os.walk("/etc")                        # (root, dirs, files) recursive generator
os.makedirs("/tmp/a/b/c", exist_ok=True)
os.environ.get("HOME")
os.environ["MY_VAR"] = "value"
os.getcwd(); os.chdir("/tmp")
os.getpid(); os.cpu_count()
os.chmod(path, 0o644); os.access(path, os.W_OK)
```

## `pathlib`

```python
from pathlib import Path

p = Path("/var/log") / "app.log"     # join with /
p.exists(); p.is_file(); p.is_dir()
p.parent; p.name; p.stem; p.suffix
p.with_suffix(".json")
p.with_name("other.log")

p.read_text(); p.write_text("data\n")
p.read_bytes(); p.write_bytes(b"data")

Path("/etc").glob("*.conf")           # non-recursive
Path("/etc").rglob("*.conf")          # recursive (like find)
list(Path(".").iterdir())              # direct children only

Path("/tmp/a/b").mkdir(parents=True, exist_ok=True)   # like mkdir -p
p.unlink(missing_ok=True)              # delete file, no error if missing
p.stat().st_size                        # size in bytes
p.resolve()                             # absolute, symlink-resolved path
Path(__file__).resolve().parent          # dir containing the current script
```

## `argparse`

```python
import argparse

parser = argparse.ArgumentParser(description="...")
parser.add_argument("hosts", nargs="+")                     # positional, 1+ values
parser.add_argument("--threshold", type=int, default=80)
parser.add_argument("--verbose", action="store_true")
parser.add_argument("--format", choices=["json", "text"], default="text")
args = parser.parse_args()

# Subcommands
sub = parser.add_subparsers(dest="command", required=True)
check_p = sub.add_parser("check")
check_p.add_argument("--host", required=True)
args = parser.parse_args()
if args.command == "check": ...
```

## `click`

```python
import click

@click.command()
@click.option("--host", "hosts", multiple=True, required=True)
@click.option("--threshold", default=80, show_default=True, type=int)
@click.option("--verbose", is_flag=True)
@click.argument("logfile", type=click.Path(exists=True), required=False)
def main(hosts, threshold, verbose, logfile):
    click.echo(f"checking {hosts}")

if __name__ == "__main__":
    main()

# Groups / subcommands
@click.group()
def cli(): ...

@cli.command()
def check(): ...

# Types: click.Path(exists=True), click.Choice([...]), click.IntRange(0, 100)

# Testing
from click.testing import CliRunner
result = CliRunner().invoke(main, ["--host", "web1"])
result.exit_code; result.output
```

## `logging`

```python
import logging

logger = logging.getLogger(__name__)       # or a fixed app name
logger.setLevel(logging.DEBUG)             # hard floor for this logger
logger.propagate = False                    # avoid duplicate lines via root logger

fh = logging.FileHandler("app.log")
fh.setLevel(logging.DEBUG)
fh.setFormatter(logging.Formatter("%(asctime)s %(levelname)-8s %(message)s"))

ch = logging.StreamHandler()               # defaults to stderr
ch.setLevel(logging.INFO)
ch.setFormatter(logging.Formatter("%(levelname)s: %(message)s"))

logger.addHandler(fh)
logger.addHandler(ch)

logger.debug("...")
logger.info("host %s at %d%%", host, pct)  # lazy %-formatting, not f-strings
logger.warning("...")
logger.error("...")
logger.critical("...")

# Rotating file handler (size-based)
from logging.handlers import RotatingFileHandler
rfh = RotatingFileHandler("app.log", maxBytes=5_000_000, backupCount=3)

# Quick one-off script (root logger, not for real tools)
logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")
```

| Level | Value | Use |
|---|---|---|
| DEBUG | 10 | verbose internals |
| INFO | 20 | normal milestones |
| WARNING | 30 | unexpected, non-fatal |
| ERROR | 40 | operation failed |
| CRITICAL | 50 | program-level failure |

## Virtual environments

```bash
python3 -m venv .venv                  # create
source .venv/bin/activate              # activate (Linux/macOS)
.venv\Scripts\activate                 # activate (Windows)
python -m pip install --upgrade pip
pip install click ruff mypy
pip freeze > requirements.txt          # snapshot exact versions
deactivate                              # leave the venv
which python                            # sanity check: should point inside .venv
```

## `pyproject.toml`

```toml
[project]
name = "diskcheck"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["click>=8.1"]

[project.optional-dependencies]
dev = ["ruff>=0.4", "mypy>=1.8", "pytest>=8.0"]

[project.scripts]
diskcheck = "diskcheck.cli:main"

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[tool.mypy]
python_version = "3.11"
disallow_untyped_defs = true
```

```bash
pip install -e .              # editable install (dev)
pip install -e ".[dev]"        # editable install + dev extras
pip install .                  # regular install
```

## `ruff` / `mypy`

```bash
ruff check .            # lint
ruff check --fix .      # lint + auto-fix
ruff format .           # format (black-compatible)
mypy diskcheck/         # static type-check a package
mypy --strict file.py   # strictest mode, single file
```
