# Day 6 — Resources: Bash Scripting II — Text Processing

## Primary (assigned)

- **Awk tutorial (grymoire.com)** — free, exhaustive, and the assigned starting point for this day. Covers `awk`'s record/field model, patterns, and aggregation with far more depth and worked examples than most modern tutorials bother with.

## Deepen your understanding

- **grymoire.com's sed tutorial** — the same author's companion piece on `sed`, equally thorough, covering addresses, the hold space, and multi-line editing beyond what's in today's notes.
- **`man 7 regex`** (or `man re_format` on BSD/macOS) — the authoritative, on-box reference for POSIX BRE/ERE syntax, including the character classes (`[[:digit:]]`, etc.) used throughout today's material.
- **jq Manual** (jqlang.github.io/jq/manual) — official, example-rich reference for `jq`'s filter language; essential once logs move from plain text to structured JSON.
- **"The Unix Programming Environment" (Kernighan & Pike)** — old (1984) but still the clearest explanation of *why* Unix tools compose into pipelines the way they do; explains the philosophy behind `grep`/`sed`/`awk`/`sort` as a toolkit rather than isolated commands.

## Reference / lookup

- **explainshell.com** — paste any `grep`/`awk`/`sed` one-liner and get every flag broken down inline; especially useful for decoding dense pipelines you find in the wild.
- **regex101.com** (set the flavor to POSIX or PCRE as needed) — build and test regex patterns interactively with live match highlighting before dropping them into a `grep`/`sed`/`awk` command.

## Practice

- **cmdchallenge.com** — short, browser-based challenges that specifically exercise `grep`/`awk`/`sed`/`cut`/`sort` pipelines against realistic text-processing problems.
- **OverTheWire: Bandit** (overthewire.org/wargames/bandit) — several levels directly require combining `grep`, `sort`, `uniq`, and pattern matching to find specific data hidden in large files, reinforcing today's "top N" and filtering idioms under real constraints.
