# Day 6 — Bash Scripting II — Text Processing: grep & Regex

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

`grep` is the tool you reach for first during almost every incident: "is this error in the logs anywhere?", "which of these 40 config files still reference the old hostname?", "did that deploy actually roll out to all pods?" It's also the backbone of countless CI one-liners (`grep -q "PASS" test-output.log || exit 1`). Regular expressions are the language `grep` (and `sed`, `awk`, `jq`, your editor, and your log aggregator's search box) all speak — understanding them once pays off everywhere, not just in the shell.

This day is split into three files:

1. **This file** — `grep`, regex flavors (BRE vs ERE), anchors, character classes, and the patterns you'll actually type.
2. **[02-README-Awk-Field-Extraction.md](02-README-Awk-Field-Extraction.md)** — `awk`'s record/field model for pulling structured data out of logs.
3. **[03-README-Sed-Pipelines-Utilities.md](03-README-Sed-Pipelines-Utilities.md)** — `sed`, `cut`/`sort`/`uniq`/`wc`/`tr`, pipelines/redirection, and `jq`.

## What `grep` actually does

`grep` (Global Regular Expression Print — the name is a fossil of the `ed` command `g/re/p`) reads input line by line and prints lines that match a pattern. That's it. Everything else is flags that change *what counts as a match* or *what gets printed*.

```bash
grep "ERROR" app.log              # lines containing ERROR, literal substring
grep -i "error" app.log           # case-insensitive
grep -v "DEBUG" app.log           # invert: lines NOT containing DEBUG
grep -c "ERROR" app.log           # count of matching lines, not the lines themselves
grep -n "ERROR" app.log           # prefix each match with its line number
grep -o "ERROR[0-9]*" app.log     # print only the matched substring, not the whole line
grep -A3 -B1 "panic" app.log      # 3 lines After, 1 line Before each match (context)
grep -r "TODO" src/               # recurse into a directory
grep -l "ERROR" *.log             # list only FILENAMES that contain a match
grep -L "ERROR" *.log             # list filenames that do NOT contain a match
grep -w "cat" file                # whole-word match (won't match "concatenate")
grep -x "OK" file                 # whole-LINE match only
grep -F "a.b[c]" file             # fixed string — treat pattern as literal, no regex at all
```

**Exit codes matter more than the output** in scripts: `grep` exits `0` if it found at least one match, `1` if it found none, and `2` on a real error (bad option, unreadable file). This is why `grep -q "pattern" file` (quiet — no output at all) is the idiomatic way to test for a condition in a shell script:

```bash
if grep -q "^ERROR" app.log; then
  echo "errors present"
fi
```

Under `set -e`, a bare `grep` that finds nothing will exit `1` and can kill your script — this is one of the most common "why did my CI job die on a totally normal `grep`" bugs (see Common Mistakes).

## BRE vs ERE: why `grep` sometimes needs `-E`

`grep`'s regex engine understands two different *dialects* of the same underlying grammar, both defined by POSIX:

- **BRE (Basic Regular Expressions)** — `grep`'s default.
- **ERE (Extended Regular Expressions)** — enabled with `grep -E` (historically the separate `egrep` binary, now usually a symlink/wrapper around `grep -E`).

The difference is entirely about **which characters are "special" by default**. In BRE, the characters `+ ? | ( ) { }` are treated as **literal characters** unless you escape them with a backslash (`\+ \? \| \( \) \{ \}`) — backslash *adds* meaning instead of removing it, which is the opposite of every other regex flavor you've probably used (Python, JS, PCRE). In ERE, those characters are special by default and you'd escape them (`\+`) to match them literally — the "normal" convention.

```bash
grep    'a\+'   file     # BRE: one or more 'a' — the \+ is what makes it "one or more"
grep -E 'a+'    file     # ERE: same meaning, no escaping needed
grep    'cat|dog' file   # BRE: matches the literal string "cat|dog" (| has no special meaning!)
grep -E 'cat|dog' file   # ERE: matches "cat" OR "dog"
```

Why this split exists at all is historical — `grep` and `ed`/`sed` implemented the simpler BRE grammar first, and ERE was added later (originally in `egrep`) for more expressive matching, and POSIX standardized both rather than picking one. GNU `grep` implements both with the same underlying engine; `-E` just changes which characters get special treatment.

`\d`, `\w`, `\s` (Perl-style shorthand classes) are **not** part of POSIX BRE or ERE at all — GNU `grep` only understands them with `-P` (PCRE mode), which is a GNU extension not available in BSD/macOS's default `grep`. Use POSIX character classes (below) if you need portable regex.

## Anchors and character classes

```bash
^pattern      # start of line
pattern$      # end of line
^$            # matches empty lines exactly
.             # any single character (must be escaped \. to mean a literal dot)
[abc]         # any one of a, b, c
[^abc]        # any character EXCEPT a, b, c
[a-z]         # range
[0-9]{3}      # ERE only: exactly 3 digits (BRE needs \{3\})
[[:digit:]]   # POSIX class: any digit — locale-aware, portable across BRE/ERE
[[:alpha:]]   # any letter
[[:space:]]   # any whitespace (space, tab, newline)
[[:upper:]]   # any uppercase letter
```

POSIX character classes (`[[:digit:]]`, `[[:alpha:]]`, etc.) exist because plain ranges like `[a-z]` are **locale-dependent** — in some locales, collation order pulls accented characters into that range unexpectedly. `[[:digit:]]` is the portable, locale-safe way to say "a digit" without relying on `\d` (which isn't standard) or ASCII range assumptions.

## Patterns you'll actually type in DevOps work

```bash
# Rough IPv4 matcher (not RFC-perfect, but good enough for log triage)
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' access.log

# HTTP 4xx/5xx status codes in a space-delimited log field
grep -E ' [45][0-9]{2} ' access.log

# Lines that start with a timestamp-looking pattern
grep -E '^\[?[0-9]{4}-[0-9]{2}-[0-9]{2}' app.log

# Multiple keywords, any of them (ERE alternation)
grep -E 'panic|fatal|OOMKilled' app.log

# Case-insensitive whole-word match for a hostname across configs
grep -riw 'staging-db' /etc/

# Fast literal search (no regex interpretation) when your pattern has no special chars
grep -F '10.0.4.12' access.log
```

`-F` matters more than it looks: with a literal string, `grep` can use substring-search algorithms (Boyer–Moore-style skipping) instead of running a full regex engine, which is measurably faster on large files — always prefer `-F` when your search term isn't actually a pattern.

## Points to Remember

- BRE (`grep` default) requires escaping `+ ? | ( ) { }` to get their special meaning; ERE (`grep -E`, `egrep`) treats them as special by default. Same engine, different dialect.
- `grep` exit codes: `0` = match found, `1` = no match, `2` = error. `1` is not a failure — it's information. `set -e` scripts that don't account for this can die on a routine "no match" grep.
- `\d`, `\w`, `\s` are **not POSIX** — they only work with GNU `grep -P` (PCRE mode), which macOS's default (BSD) `grep` doesn't support. Use `[[:digit:]]`/`[0-9]` for portability.
- `^` and `$` anchor to **line** boundaries in default mode, not to the start/end of the whole input — relevant when a "line" in your data isn't what you assume (e.g., embedded `\r` from Windows line endings makes `$` match in an unexpected place).
- `-F` (fixed string) is faster than regex mode and should be your default when matching a literal string with no wildcards.

## Common Mistakes

- Writing `grep 'a+b'` expecting "one or more a followed by b" and getting a literal match for the three-character string `a+b` instead — forgetting BRE requires `\+` or `-E`.
- Unescaped `.` in patterns like `192.168.1.1` matching unintended strings (`192a168a1a1`) because `.` means "any character," not "literal dot" — should be `192\.168\.1\.1`.
- Treating `grep`'s exit code `1` as a script error under `set -e`/`set -euo pipefail`, causing pipelines to abort when a search legitimately finds nothing. Fix: `grep ... || true` (when a miss is fine) or explicitly branch on the exit code.
- Assuming `-P` (PCRE, `\d`, lookaheads) works everywhere — it's a GNU `grep` extension. Scripts written and tested on Linux CI runners silently break when someone runs them on macOS's BSD `grep`.
- Piping to `grep` for filename matching (`ls | grep pattern`) instead of using `grep`'s own `-l`/`--include` or `find -name` — works most of the time, but breaks on filenames with newlines and is a slower detour through an extra process.
