# Day 5 — Cheatsheet: Bash Scripting I

## Shebang

```bash
#!/usr/bin/env bash     # preferred — resolves bash via $PATH, portable
#!/bin/bash              # hardcoded path, less portable
#!/bin/sh                 # NOT bash on Debian/Ubuntu (it's dash) — no arrays, [[ ]], local
```

## Variables

```bash
NAME="value"              # no spaces around =
readonly PI=3.14159         # cannot be reassigned
local x=1                   # function-scoped only (use inside functions)
export VAR="value"           # visible to child processes

echo "$VAR"                 # simple reference
echo "${VAR}_suffix"          # braces needed to disambiguate from following text
echo "${VAR:-default}"        # use default if VAR unset/empty (doesn't assign)
echo "${VAR:=default}"        # same, but also assigns default to VAR
echo "${VAR:?error msg}"       # exit with error if VAR unset/empty
echo "${VAR:+alt}"              # use alt only if VAR IS set
echo "${VAR%pattern}"            # strip shortest matching suffix
echo "${VAR#pattern}"             # strip shortest matching prefix
```

## Quoting

```bash
'single quotes'      # fully literal, no expansion at all
"double quotes"       # $VAR, $(cmd), $((expr)) still expand; no word-splitting/globbing
unquoted $VAR           # word-splits on $IFS + glob-expands — avoid for variables

$(command)              # command substitution — preferred (nests cleanly)
`command`                # backticks — legacy, avoid, escaping nightmare when nested
```

## `if` / test operators

```bash
if command; then ...; elif other; then ...; else ...; fi   # tests EXIT CODE, not a boolean

[ "$a" = "$b" ]        # POSIX test command — quote everything, spaces mandatory
[[ $a == $b ]]          # bash keyword — safer, no word-splitting on unquoted vars inside
[[ $a =~ ^[0-9]+$ ]]     # regex match (bash-only)
(( a > b ))               # arithmetic context — non-zero result = true
```

| Test | Meaning |
|---|---|
| `-e` | exists | `-f` | regular file | `-d` | directory |
| `-r` `-w` `-x` | readable/writable/executable | `-s` | size > 0 |
| `-z` | string empty | `-n` | string non-empty |
| `=` `!=` (`==` in `[[ ]]`) | string equal/not-equal |
| `-eq` `-ne` `-gt` `-lt` `-ge` `-le` | numeric comparisons |

## `case`

```bash
case "$var" in
  pattern1|pattern2) cmd ;;      # ;; = stop here
  dev*) cmd ;&                    # ;& = fall through unconditionally (bash 4+)
  *) default_cmd ;;                # catch-all — always include one
esac
```

## Loops

```bash
for x in a b c; do echo "$x"; done          # list
for f in *.txt; do echo "$f"; done            # glob — safe, no subshell
for (( i=0; i<5; i++ )); do echo "$i"; done    # C-style

while (( i < 5 )); do ((i++)); done             # while test succeeds
until ping -c1 host &>/dev/null; do sleep 2; done   # until test succeeds (inverse)

while IFS= read -r line; do echo "$line"; done < file.txt   # correct line-by-line read
# NEVER: for line in $(cat file)  — word-splits and destroys line boundaries

break        # exit innermost loop
continue      # skip to next iteration
break 2        # exit two levels of nested loops
```

## Exit codes

```bash
echo $?          # exit code of last command (capture immediately — next cmd overwrites it)
exit 0             # success
exit 1              # general error
```

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | general error |
| 2 | misuse of shell builtin |
| 126 | found but not executable |
| 127 | command not found |
| 128+N | killed by signal N (130=SIGINT/Ctrl-C, 137=SIGKILL, 143=SIGTERM) |

## `set -euo pipefail`

```bash
set -e            # exit immediately on any unhandled non-zero exit
set -u             # error on reference to unset variable
set -o pipefail     # pipeline exit code = worst stage's exit code, not just the last

set -euo pipefail    # combined — put this right after the shebang
```

Gotchas: `set -e` does NOT fire inside `if`/`while`/`until` conditions, for non-final pipeline stages (without `pipefail`), or for commands feeding `&&`/`||`.

## `trap`

```bash
trap 'rm -rf "$TMP_DIR"' EXIT          # runs on any exit — normal, error, or set -e triggered
trap 'echo "Interrupted"; exit 130' INT TERM
trap 'echo "Error at line $LINENO"' ERR
```

## `shellcheck`

```bash
shellcheck script.sh                      # lint a script, prints SC#### findings
shellcheck -x script.sh                    # follow sourced files
# shellcheck disable=SC2086                # inline suppress (use sparingly, with a reason)
```
