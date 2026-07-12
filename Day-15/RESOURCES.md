# Day 15 — Resources: Python for Automation I

## Primary (assigned)

- **Real Python: Python for DevOps** (realpython.com) — free, the assigned starting point for this day. Covers `subprocess`, automation scripting patterns, and general Python-for-ops workflows with the same practical, example-first style this repo aims for.

## Deepen your understanding

- **Python docs — `subprocess` module** (docs.python.org/3/library/subprocess.html): the authoritative reference for every parameter covered today (`run`, `Popen`, `check_output`, `timeout`, `shell`), including the official "Replacing Older Functions with the subprocess Module" section that explains exactly what `os.system`/`os.popen` used to do and why `subprocess` superseded them.
- **Python docs — Logging HOWTO** (docs.python.org/3/howto/logging.html): the canonical, deep-dive tutorial on loggers/handlers/formatters/levels — more thorough than any blog post, and the fastest way to understand propagation and the root-logger trap covered in file 3.
- **Click documentation** (click.palletsprojects.com): official docs for the `click` framework, including the testing guide for `CliRunner` and the full type system (`click.Path`, `click.Choice`, custom parameter types) referenced in file 2.
- **`pyproject.toml` — Python Packaging User Guide** (packaging.python.org/en/latest/guides/writing-pyproject-toml/): the official, up-to-date guide to what belongs in `pyproject.toml`, how `[project]` vs `[tool.X]` tables work, and how it replaces `setup.py`.

## Reference / lookup

- **OWASP — OS Command Injection** (owasp.org, CWE-78): the security reference for exactly the `shell=True` risk covered in file 1 — useful to cite directly if this comes up in an interview or a security review.
- **Ruff documentation** (docs.astral.sh/ruff): full rule reference for `ruff check`, configuration options under `[tool.ruff]`, and how it maps onto the rules previously split across `flake8`/`isort`/`pyupgrade`.

## Practice

- **Real Python's `subprocess` and CLI tutorials** (realpython.com, search "subprocess" and "click"): hands-on, runnable walkthroughs that pair well with today's LAB.md exercises.
- **Exercism's Python track** (exercism.org/tracks/python): free, mentored practice exercises — useful for drilling `pathlib`/stdlib fluency outside of the DevOps-specific scenarios in this repo.
