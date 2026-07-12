# Day 5 — Bash Scripting I: Script Basics

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

Every CI/CD pipeline step, container entrypoint, cloud-init user-data block, Kubernetes init container, and deploy hook eventually bottoms out in a shell script. Bash is the closest thing DevOps has to a universal glue language — and it is also one of the easiest languages to write *plausible-looking, silently broken* code in. Interviewers lean on bash questions specifically because the gap between "can copy a script from Stack Overflow" and "understands why `$VAR` without quotes is a landmine" is a fast, reliable signal of real operational experience.

This day is split into three files:

1. **This file** — the shebang line, variables, and quoting rules.
2. **[02-README-Control-Flow.md](02-README-Control-Flow.md)** — `if`/`case`, test operators, `for`/`while` loops.
3. **[03-README-Exit-Codes-Safe-Scripting.md](03-README-Exit-Codes-Safe-Scripting.md)** — exit codes, `set -euo pipefail`, `trap`, `shellcheck`.

## The shebang line — how the kernel actually uses it

```bash
#!/usr/bin/env bash
```

The shebang (`#!`) is not a bash feature — it's a **Linux kernel** convention handled by the `execve()` syscall. When you run `./script.sh`, the kernel reads the first two bytes of the file. If they are `#!`, it treats the rest of that first line as *"the interpreter that should run this file"*, then re-invokes `execve()` as `<interpreter> <script-path> <original-args>`. Your script never actually "runs itself" — it's handed to whatever binary the shebang names.

Consequences that matter in practice:

- The shebang **must be the very first line**, first two characters, no leading blank line, no BOM. A stray blank line above it and the kernel just sees a script with no interpreter and falls back to `/bin/sh` (or errors), silently changing behavior.
- Historically, Linux limited the shebang line to ~127 bytes and only honored **one** argument after the interpreter path — this is why `#!/usr/bin/env bash -e` doesn't reliably pass `-e` on every platform, and why flags belong on a `set` line inside the script instead.
- `#!/bin/bash` hardcodes the interpreter's absolute path. `#!/usr/bin/env bash` instead asks `env` to resolve `bash` from `$PATH` — more portable across systems where bash isn't at `/bin/bash` (macOS with Homebrew bash at `/opt/homebrew/bin/bash`, NixOS, some minimal containers). Most style guides now recommend `env bash` for scripts you intend to share.
- `#!/bin/sh` is **not** a synonym for bash. On Debian/Ubuntu, `/bin/sh` is a symlink to `dash`, a POSIX-only shell that doesn't support bash-isms like `[[ ]]`, arrays, or `local`. A script written and tested with bashisms but shebanged `#!/bin/sh` will fail (or worse, silently misbehave) the moment someone runs it with `sh script.sh`, which ignores the shebang entirely and forces execution under `sh`.
- If you invoke a script explicitly (`bash script.sh`), the shebang is **ignored** — the interpreter is whatever you named on the command line. The shebang only matters when the file is executed directly (`./script.sh`) and relies on the kernel's exec mechanism, which is also why the script needs its execute bit set (`chmod +x`) for direct execution to work at all.

## Variables

Bash has no static type system — every variable is a string unless you explicitly enter an arithmetic context (`(( ))`) or declare it with `declare -i`.

```bash
NAME="value"          # no spaces around =; spaces make bash think NAME is a command
readonly PI=3.14159     # cannot be reassigned after this
local count=0            # scoped to the enclosing function only
export PATH="$PATH:/opt/tool/bin"   # exported into the environment of child processes
declare -i n=5             # arithmetic-typed: n=n+"abc" evaluates as 0, not a string concat
```

- **Referencing**: `$VAR` and `${VAR}` are equivalent for simple lookups, but braces are *required* when the variable name is immediately followed by text that could merge with it (`"${VAR}_suffix"` vs. the ambiguous `"$VAR_suffix"`, which bash parses as the variable `VAR_suffix`), and for parameter-expansion operators:
  - `${VAR:-default}` — use `default` if `VAR` is unset or empty (doesn't assign it).
  - `${VAR:=default}` — same, but also assigns `default` to `VAR`.
  - `${VAR:?error message}` — exit with an error if `VAR` is unset or empty.
  - `${VAR:+alt}` — use `alt` only if `VAR` *is* set.
  - `${VAR%pattern}` / `${VAR#pattern}` — strip a suffix/prefix match.
- **Scope**: variables are global by default, even inside functions, unless declared `local`. This trips up people coming from languages with block scope — a variable set inside a function leaks into the caller's scope unless you explicitly mark it `local`.
- **Environment vs. shell variable**: a plain assignment (`NAME=value`) only exists in the current shell. `export NAME=value` promotes it into the environment block inherited by child processes (any command or script you subsequently run from that shell). A common bug: setting a variable, forgetting to `export` it, then wondering why a script it calls doesn't see it.

## Quoting rules

This is the single highest-value bash topic for avoiding real production bugs. Bash performs **word splitting** and **pathname expansion (globbing)** on the result of unquoted variable and command expansions — and only there. Literal strings and quoted expansions are exempt.

```bash
FILES="report final.txt"
for f in $FILES; do echo "$f"; done
# unquoted: word-split on whitespace -> loops twice: "report" then "final.txt"
# (even though $FILES was meant to be one filename with a space in it)

rm -rf $BACKUP_DIR/*      # if $BACKUP_DIR is empty/unset, this expands to: rm -rf /*
rm -rf -- "${BACKUP_DIR:?}"/*   # safe: quoted, and errors out loudly if unset
```

- **Double quotes (`"..."`)**: variable expansion (`$VAR`), command substitution (`$(...)`), and arithmetic expansion (`$((...))`) still happen, but the *result* is treated as a single word — no word splitting, no globbing. This is the default you should reach for almost everywhere: `echo "$VAR"`, `cp "$src" "$dst"`.
- **Single quotes (`'...'`)**: everything inside is completely literal. No expansion of any kind, not even `$VAR`. Useful for things like `awk '{print $1}'` where `$1` is awk's syntax, not bash's, and must not be touched by the shell.
- **Unquoted**: subject to word splitting on `$IFS` (default: space, tab, newline) and glob expansion (`*`, `?`, `[...]` get expanded against the filesystem). This is almost never what you want for a variable holding a path or user-controlled value.
- **`$(...)` vs. backticks (`` `...` ``)**: functionally similar (both do command substitution and strip trailing newlines from the output), but `$(...)` is strongly preferred: it nests cleanly (`$(cmd1 $(cmd2))`), doesn't require escaping inner double quotes, and is visually easier to match. Backticks require backslash-escaping nested backticks and quotes, which gets unreadable fast.
- Command substitution **strips trailing newlines** (but not embedded ones) — `X=$(printf 'a\nb\n')` sets `X` to `"a\nb"` with no trailing newline, which surprises people diffing output.

## Points to Remember

- The shebang is a kernel convention (`execve()` reads `#!`), not a bash feature — it must be byte 1, line 1, and is ignored entirely if the script is invoked as `bash script.sh` instead of `./script.sh`.
- `#!/bin/sh` on Debian/Ubuntu is `dash`, a POSIX shell without arrays, `[[ ]]`, or `local` — don't shebang a bashism-using script with `/bin/sh`.
- `#!/usr/bin/env bash` resolves `bash` from `$PATH` and is more portable than hardcoding `#!/bin/bash`.
- Variables are untyped strings by default, global by default (use `local` in functions), and not visible to child processes unless `export`ed.
- `${VAR:-default}`, `${VAR:=default}`, `${VAR:?msg}`, `${VAR:+alt}` are the four core "default value" parameter-expansion forms — know what each does without looking it up.
- Double-quote every variable and command substitution unless you specifically want word splitting/globbing (which is rare, and should be a deliberate, commented choice).
- `$(...)` over backticks — nests cleanly, no escaping headaches.

## Common Mistakes

- Leaving variable expansions unquoted (`rm -rf $DIR/*`, `cp $SRC $DST`) — the #1 source of bash bugs: breaks on filenames with spaces, and turns an unset/empty variable into a glob that can match far more than intended.
- Putting spaces around `=` in an assignment (`NAME = value`) — bash parses this as running a command named `NAME` with arguments `=` and `value`, producing a confusing "command not found" error.
- Forgetting `export` and then being confused why a variable set in a parent shell isn't visible inside a script it invokes as a separate process.
- Using `for f in $(ls *.txt)` instead of `for f in *.txt` — the former word-splits `ls`'s output (breaks on spaces in filenames, mangles the list) and is redundant since bash already glob-expands `*.txt` directly.
- Relying on `#!/bin/sh` when the script actually uses bash-only features (arrays, `[[ ]]`, `local`, `function` keyword) — works fine when bash *is* `/bin/sh` on your machine, then breaks the moment it runs on Debian/Ubuntu or in a minimal container image.
- Mixing single and double quotes incorrectly, e.g. wrapping an entire command in single quotes when you actually needed a variable inside it to expand (`echo 'Backing up $DIR'` prints the literal string `$DIR`, not its value).
