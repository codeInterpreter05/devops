# Day 5 — Bash Scripting I: Exit Codes & Safe Scripting

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

This is the file that separates "a script that happened to work when I tested it" from "a script that fails loudly and safely the moment something goes wrong." `set -euo pipefail` at the top of a script is close to a universal convention in production bash, `trap` is how you guarantee cleanup happens even on failure, and `shellcheck` is the tool that catches the exact class of quoting/word-splitting mistakes covered in the other two files, automatically, before code review. This is also one of the most commonly asked bash interview questions — see the quiz for the exact phrasing you should be ready to answer.

## Exit codes — the contract every command makes

Every process that exits returns a small integer, `0`–`255`, to its parent. Bash exposes the most recently completed foreground command's exit code in `$?`. Convention (not a hard kernel rule, but very widely followed):

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | General/catch-all error |
| `2` | Misuse of a shell builtin (e.g., bad arguments) |
| `126` | Command found but not executable (permissions, or wrong file type executed as a binary) |
| `127` | Command not found (typo, or not on `$PATH`) |
| `128+N` | Process terminated by signal `N` — `130` = `128+2` = `SIGINT` (Ctrl-C), `137` = `128+9` = `SIGKILL`, `143` = `128+15` = `SIGTERM` |

```bash
grep -q "ERROR" app.log
echo $?          # 0 if found, 1 if not found, 2 if app.log doesn't exist/other error

exit 0            # explicit success
exit 1            # explicit generic failure — put this at the end of error branches
```

`$?` is volatile — it's overwritten by the **next** command that runs, including something as innocuous as `echo`. If you need to inspect it, capture or act on it immediately: `cmd; rc=$?; echo "exit was $rc"`, not `cmd; echo "ran"; echo $?` (the second `echo` has already reset `$?` to its own exit code, which is always 0).

## `set -e` (errexit) — stop on the first failure

```bash
set -e
```

Without it, bash's default behavior is to plow ahead after a failing command — a script that does `cd /some/dir` (which fails silently if the directory doesn't exist) followed by `rm -rf ./*` will delete the *current* directory's contents instead, because the failed `cd` didn't stop execution. `set -e` makes bash exit immediately when any command returns non-zero, which is almost always what you want for a script instead of "continue running with an already-broken assumption."

**The well-known gotchas** (these are asked about specifically because they trip up nearly everyone):

- **Inside a conditional, `set -e` is suspended.** `if some_command; then` will not trigger an exit even if `some_command` fails — that failure is the *expected, tested-for* outcome of the `if`. Same applies to the test part of `while`/`until`, and to any command that's an operand of `&&` or `||`.
- **Only the last command in a pipeline is checked**, unless `pipefail` is also set. `false | true` "succeeds" under plain `set -e` because `true` (the last stage) exited 0 — the failure of `false` is invisible.
- **A command's failure is ignored if its exit status is explicitly being checked**, e.g. `some_command || true`, or when it's the condition of `&&`/`||` chains generally.
- Historically, `set -e` behaved inconsistently across bash versions/subshells for certain function-call and command-substitution edge cases — don't treat it as an absolute guarantee; treat it as "the default failure path," and still check exit codes explicitly for anything critical.

## `set -u` (nounset) — catch references to unset variables

```bash
set -u
echo "$UNDEFINED_VAR"   # errors instead of silently expanding to an empty string
```

Without `set -u`, a typo'd variable name (`$BACKUP_DRI` instead of `$BACKUP_DIR`) silently expands to an empty string — which, combined with something like `rm -rf "$BACKUP_DIR"/*`, is exactly how "delete everything" accidents happen. `set -u` turns that typo into an immediate script error instead of a silent, catastrophic no-op-that-isn't. One deliberate exception: positional parameters and `$*`/`$@` when no arguments were passed can behave specially across bash versions — if a script legitimately expects to run with zero arguments, guard explicitly (`"${1:-}"`) rather than relying on default behavior.

## `set -o pipefail` — make pipeline failures visible

```bash
set -o pipefail
grep "ERROR" app.log | sort | uniq -c
```

By default, a pipeline's exit status is just the exit status of its **last** command — every earlier stage's failure is thrown away. If `grep` fails to find the log file at all but `sort`/`uniq` still run (on empty input) and exit 0, the whole pipeline reports success even though the first stage never actually worked. `pipefail` changes this: the pipeline's exit status becomes the **last non-zero exit status among all stages** (or 0 if all succeeded). This matters constantly in scripts that pipe output through `grep`/`awk`/`sort` for filtering — without `pipefail`, upstream failures are invisible.

## `set -euo pipefail` — the standard opening line

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Combining all three is close to a de facto standard for the first line of any non-trivial bash script: stop on unhandled errors, catch unset-variable typos, and don't let pipeline failures hide behind a successful last stage. It is **not** a silver bullet — it doesn't catch logic errors, doesn't protect command substitutions used in contexts where their exit code isn't checked (e.g. `local var=$(cmd_that_fails)` historically masked the inner command's failure in some bash versions because `local` itself "succeeds"), and doesn't replace explicit error handling for things you expect might fail. Treat it as a safety net under your code, not a substitute for it.

## `trap` — guaranteed cleanup and signal handling

```bash
TMP_DIR="$(mktemp -d)"
cleanup() {
  rm -rf -- "$TMP_DIR"
}
trap cleanup EXIT

trap 'echo "Interrupted" >&2; exit 130' INT
trap 'echo "Error at line $LINENO" >&2' ERR
```

`trap` registers a command or function to run when the script receives a given signal or reaches a given pseudo-event. `EXIT` fires on **any** script termination — normal completion, `exit`, or an error triggering `set -e` — making it the reliable place to put cleanup logic (removing temp files/dirs, releasing a lock file) that must run no matter *how* the script ends. `ERR` fires specifically when a command fails (subject to the same suspension rules as `set -e`, e.g. not inside conditionals). `INT`/`TERM` let you handle Ctrl-C or a `kill` gracefully instead of leaving things in a half-finished state.

## `shellcheck` — static analysis for the mistakes above

```bash
shellcheck backup.sh
```

`shellcheck` is a linter purpose-built for shell scripts. It catches exactly the class of bugs described across these three files — unquoted variables (`SC2086`), useless use of `cat`, comparing strings with `-eq`, unreachable code after `exit`, using `[ ]` where quoting was missed, variables that are assigned but never used, and dozens more, each with a numbered `SC####` code and a linked explanation. It exits non-zero if it finds issues at or above the configured severity, which makes it trivial to wire into a CI pipeline or pre-commit hook so bad scripts never get merged. Running it against a script you "know is fine" is a humbling and educational first experience — it's rare for a script over ~20 lines to come back completely clean the first time.

## Points to Remember

- `$?` holds the exit code of the *last* command and is overwritten by the very next command — capture it immediately if you need it (`rc=$?`).
- Exit code conventions: `0` success, `1` general error, `2` builtin misuse, `126` not executable, `127` not found, `128+N` killed by signal `N` (130 = Ctrl-C, 137 = SIGKILL, 143 = SIGTERM).
- `set -e` does **not** fire inside `if`/`while`/`until` conditions, for non-final stages of a pipeline (without `pipefail`), or for commands whose result feeds `&&`/`||`.
- `set -u` turns unset-variable typos into hard errors instead of silent empty-string expansions — the single highest-leverage flag against accidental `rm -rf` disasters.
- `set -o pipefail` makes a pipeline's exit status reflect the worst failure among *all* its stages, not just the last one.
- `trap cleanup EXIT` is the reliable way to guarantee cleanup runs on any exit path, including ones triggered by `set -e`.
- `shellcheck` should be run against every script before it ships; each finding has an `SC####` code you can look up for the exact reasoning.

## Common Mistakes

- Assuming `set -e` alone makes a script bulletproof, then being surprised when a failing command inside an `if` condition, or a non-last pipeline stage, doesn't stop execution.
- Checking `$?` after an intervening command (like a log `echo`) has already overwritten it, always reading `0`.
- Treating `set -euo pipefail` as sufficient error handling on its own, with no explicit checks around commands whose failure needs a specific response (retry, fallback, custom message) rather than "stop immediately."
- Wrapping a command that's *expected* to sometimes fail (like a health-check `curl`) in a script with `set -e` and no `|| true`/explicit `if`, causing the whole script to abort on an expected, handled condition.
- Never running `shellcheck` until something breaks in production, instead of running it routinely (or in CI) as part of writing every script.
- Forgetting that `trap ... EXIT` handlers themselves run with whatever `set -e`/`set -u` state is active — an unquoted variable or unset var inside the cleanup function can itself fail or misbehave.
