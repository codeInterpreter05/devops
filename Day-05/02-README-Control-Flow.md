# Day 5 — Bash Scripting I: Control Flow

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

Control flow is what turns a list of commands into an actual program: branching on whether a file exists, retrying a flaky network call, looping over every server in a list, or parsing arguments passed to a deploy script. Bash's control-flow constructs look similar to other languages on the surface but have a fundamentally different evaluation model underneath — understanding *that* difference is what separates people who can read a script from people who can debug one at 2 a.m.

## `if` / `elif` / `else` — testing exit codes, not booleans

```bash
if command; then
  echo "command succeeded"
elif other_command; then
  echo "other_command succeeded"
else
  echo "neither succeeded"
fi
```

The critical mental model: **`if` doesn't evaluate a boolean expression — it runs a command and checks its exit status.** `0` means "true" (success), any non-zero value means "false" (failure). This is why `if grep -q pattern file; then` works with no explicit comparison: `grep`'s own exit code (0 = found a match, 1 = no match, 2 = error) *is* the condition. `[ ]`, `[[ ]]`, and `(( ))` aren't special `if` syntax — they're themselves commands/keywords that happen to return 0 or 1, so `if` can test them like anything else.

### `[ ]` — the `test` command

`[ ]` is literally the `test` command with a trailing `]` as syntactic sugar (there's a real `/usr/bin/[` binary, though bash has a builtin version too). Because it's an ordinary command, its arguments are subject to normal word splitting and globbing, so spacing and quoting matter strictly:

```bash
[ "$a" = "$b" ]     # correct — space after [ and before ]; quotes prevent word-splitting/empty-arg errors
[ $a = $b ]           # dangerous — if $a is empty, this becomes "[ = $b ]", a syntax error
```

### `[[ ]]` — bash keyword, safer and more capable

```bash
[[ $a == $b ]]                # no word-splitting/globbing on unquoted vars inside [[ ]] — safer by default
[[ $filename == *.log ]]       # glob pattern matching
[[ $version =~ ^[0-9]+\.[0-9]+$ ]]   # regex matching (bash-only; sets ${BASH_REMATCH[@]})
[[ $a == "foo" && $b == "bar" ]]     # && / || work directly inside [[ ]]
```

`[[ ]]` is a bash **keyword**, parsed specially — it doesn't word-split or glob-expand unquoted variables inside it the way `[ ]` does, supports `&&`/`||`/`<`/`>` without escaping, and adds glob (`==`) and regex (`=~`) matching. Prefer `[[ ]]` in bash scripts; fall back to `[ ]` only if you specifically need POSIX `sh` portability.

### `(( ))` — arithmetic context

```bash
(( a > b ))            # numeric comparison; no $ needed on variable names inside
(( count++ ))           # arithmetic increment
if (( retries >= max_retries )); then break; fi
```

`(( ))` evaluates a C-like arithmetic expression and returns exit status 0 if the result is non-zero (true), 1 if the result is zero (false) — note this is *inverted* from how you might expect a "return value" to work, but consistent with "0 = success" for exit codes.

### Test operator reference

| Operator | Meaning |
|---|---|
| `-e file` | file exists (any type) |
| `-f file` | exists and is a regular file |
| `-d file` | exists and is a directory |
| `-L file` | exists and is a symlink |
| `-r` / `-w` / `-x` | readable / writable / executable |
| `-s file` | exists and has size > 0 |
| `-z str` | string is empty |
| `-n str` | string is non-empty |
| `str1 = str2` (or `==` in `[[ ]]`) | string equality |
| `str1 != str2` | string inequality |
| `n1 -eq n2`, `-ne`, `-gt`, `-lt`, `-ge`, `-le` | numeric equal/not-equal/greater/less/ge/le |

Numeric and string comparisons use **different operators** on purpose in `[ ]`/`[[ ]]` (`-eq` vs `=`) — using `=` on two variables you assume are numbers silently does a *string* comparison (`"10" = "010"` is false), which is a classic subtle bug.

## `case` statements

```bash
case "$env" in
  prod|production)
    echo "deploying to production"
    ;;
  staging)
    echo "deploying to staging"
    ;;
  dev*)
    echo "matches any dev-prefixed environment"
    ;;
  *)
    echo "unknown environment: $env" >&2
    exit 1
    ;;
esac
```

`case` patterns are **glob-style**, not regex (`*`, `?`, `[abc]` work; `^`, `+`, `\d` do not). `;;` ends a branch normally; `;&` (bash 4+) falls through unconditionally into the next branch's code regardless of its pattern; `;;&` (bash 4+) falls through but still re-tests the next pattern. `case` is the idiomatic tool for argument parsing, environment dispatch, and menu-driven scripts — cleaner than a long `if/elif` chain of string comparisons.

## `for` loops

```bash
# Iterate a list
for env in dev staging prod; do
  echo "Checking $env"
done

# Iterate files — safe, because bash glob-expands *.txt directly (no subshell, no word-splitting risk)
for file in *.txt; do
  echo "Processing $file"
done

# C-style, for numeric ranges
for (( i = 0; i < 5; i++ )); do
  echo "Iteration $i"
done
```

**The classic anti-pattern to never write:** `for file in $(ls *.txt)`. This runs `ls`, word-splits its output on whitespace, then loops over the pieces — breaking on any filename containing a space and adding an unnecessary subprocess. `for file in *.txt` achieves the same thing correctly, because bash itself expands the glob into an array-like word list before the loop even starts.

**Iterating lines of a file** has the same trap in a different shape. Never do `for line in $(cat file)` — that word-splits on every space *and* every newline, destroying line boundaries. The correct idiom:

```bash
while IFS= read -r line; do
  echo "Line: $line"
done < input.txt
```

`IFS=` (empty) prevents leading/trailing whitespace from being stripped off each line, and `-r` prevents `read` from interpreting backslashes in the input as escape sequences — both are needed for a truly faithful line-by-line read.

## `while` / `until` loops

```bash
count=0
while (( count < 5 )); do
  echo "count=$count"
  (( count++ ))
done

until ping -c1 "$HOST" &>/dev/null; do
  echo "waiting for $HOST..."
  sleep 2
done
```

`while` loops while its test command succeeds (exit 0); `until` is the mirror image — loops while its test command *fails*, useful for "wait until X becomes true" polling patterns common in deploy/health-check scripts. `break` exits the innermost loop early; `continue` skips to the next iteration. Both accept an optional numeric argument (`break 2`) to affect an outer loop from a nested one.

## Points to Remember

- `if` tests a command's **exit code**, not a boolean — `0` is true, anything else is false. This is why `if some_command; then` needs no comparison operator.
- `[ ]` is the `test` command (subject to word-splitting bugs); `[[ ]]` is a bash keyword (safer, supports `==` globs and `=~` regex); `(( ))` is for arithmetic and treats non-zero results as "true."
- Numeric comparisons use `-eq`/`-ne`/`-gt`/`-lt`/`-ge`/`-le`; string comparisons use `=`/`!=`/`==`. Using the wrong family silently produces a string comparison on numbers or vice versa.
- `case` patterns are shell globs, not regular expressions. `;;` = stop, `;&` = fall through unconditionally, `;;&` = fall through and re-test.
- Never iterate `$(ls ...)` or `$(cat ...)` in a `for` loop — use glob expansion (`for f in *.txt`) or `while IFS= read -r line; do ... done < file` instead.
- `until` is `while`'s inverse — loop continues while the test *fails*, which reads naturally for "wait until ready" polling loops.

## Common Mistakes

- Writing `if [ $status == "ok" ]` — `==` is a `[[ ]]`-ism; POSIX `[ ]` only guarantees `=` for string equality (many `[ ]` implementations accept `==` too, but it's non-portable and inconsistent).
- Comparing numbers with `=`/`==` instead of `-eq` (or vice versa, comparing strings with `-eq`), producing comparisons that "work" until an edge case (leading zeros, non-numeric input) breaks them.
- Forgetting quotes inside `[ ]` (`[ $var = foo ]`) — if `$var` is empty or contains spaces/glob characters, this can throw a syntax error or silently misbehave; `[[ $var == foo ]]` sidesteps this specific failure mode.
- Using `case` and forgetting the final `*)` catch-all branch, so unexpected input silently falls through the entire statement with no error and no action taken.
- Looping with `for line in $(cat file)` to "read a file line by line" — breaks on any whitespace within a line and doesn't preserve line boundaries at all.
- Writing an infinite `while true; do ... done` polling loop with no `sleep` and no `break`/timeout condition, hammering a resource (CPU, an API endpoint) at full speed.
