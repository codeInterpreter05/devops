# Day 6 — Bash Scripting II — Text Processing: sed, Small Utilities, Pipelines & jq

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

If `grep` finds and `awk` extracts/aggregates, `sed` **transforms** — redacting a field, fixing line endings, rewriting a config value across hundreds of files. The small utilities (`cut`, `sort`, `uniq`, `wc`, `tr`) are the connective tissue that turns single commands into the pipelines you'll write daily: "give me the top 10 of X" is *always* some arrangement of these five tools. `jq` extends the same philosophy to JSON, which is how most modern log shippers and APIs emit data. Understanding how pipelines and redirection work at the file-descriptor level is what separates "I copy-pasted a command that worked" from "I can build the right one for a new problem."

## `sed` — the stream editor

`sed` applies an editing script to each line as it streams through, without ever holding the whole file in memory. Its commands follow an **address + command** structure:

```bash
sed -n '2,5p' file          # -n suppresses default printing; 2,5 = address range; p = print
sed '3d' file                # address 3, command d(elete)
sed '/^#/d' file              # address = regex (lines starting with #), delete them
sed '/START/,/END/d' file     # address range defined by two patterns
```

The substitution command `s///` is the one you'll use constantly:

```bash
sed 's/foo/bar/' file        # replace FIRST occurrence of foo per line
sed 's/foo/bar/g' file       # g = global, replace ALL occurrences per line
sed 's/foo/bar/2' file       # replace only the 2nd occurrence per line
sed 's/foo/bar/gi' file      # GNU: case-insensitive + global
sed 's|/var/log|/mnt/log|' file   # use | as delimiter instead of / to avoid escaping paths
sed -E 's/([0-9]+)-([0-9]+)/\2-\1/' file   # -E enables ERE; \1 \2 are backreferences to captured groups
```

Like `grep`, `sed` defaults to **BRE** — so `+`, `?`, `|`, `()` need backslashes for their special meaning unless you pass `-E` (GNU/BSD) or `-r` (GNU alias). This is the same BRE/ERE split covered in the grep note, because both tools share the same POSIX regex heritage.

## In-place editing: what `-i` actually does

```bash
sed -i 's/foo/bar/' file          # GNU: edits file directly, no backup
sed -i.bak 's/foo/bar/' file      # GNU: edits in place, keeps file.bak as backup
sed -i '' 's/foo/bar/' file       # BSD/macOS: the '' (empty backup suffix) is MANDATORY, not optional
```

Despite the name, `sed -i` doesn't literally edit bytes in place — under the hood it **writes the result to a new temporary file, then renames the temp file over the original**. This matters in practice:

- It requires **write permission on the containing directory**, not just the target file, because a rename is a directory operation.
- It **changes the file's inode**, which breaks hard links (any other hard link to the old inode now points to the old, unedited content) and can confuse tools watching the file by inode (some log-tailing/file-watcher setups).
- **GNU sed** (Linux) treats the backup-suffix argument to `-i` as optional (`-i` alone = no backup, `-iSUFFIX` = keep a backup with that suffix appended, no space between `-i` and the suffix). **BSD sed** (macOS default) requires an explicit argument, even if empty (`-i ''`) — this is the single most common cross-platform `sed` gotcha for anyone writing a script on a Mac that later runs on a Linux CI runner (or vice versa).

## `cut`, `sort`, `uniq`, `wc`, `tr`

```bash
# cut — extract by delimiter+field or by character position
cut -d',' -f1,3 file.csv       # fields 1 and 3, comma-delimited
cut -d':' -f1 /etc/passwd      # usernames
cut -c1-10 file                # characters 1 through 10 of each line

# sort — order lines
sort file                      # lexicographic, ascending
sort -n file                   # numeric order (lexicographic sort puts "10" before "9"; -n fixes that)
sort -r file                   # reverse
sort -k2,2 file                # sort by the 2nd field only (whitespace-delimited by default)
sort -t',' -k3,3n file.csv     # custom delimiter, sort by 3rd field, numerically
sort -u file                   # sort + de-duplicate in one pass
sort -h file                   # human-numeric: understands 1K, 2M, 3G (great with du -h)

# uniq — collapse ADJACENT duplicate lines (must be sorted first for a true global count!)
uniq file
uniq -c file                   # prefix each line with its count
uniq -d file                   # show only lines that had duplicates
uniq -u file                   # show only lines that were unique

# wc — counts
wc -l file                     # line count
wc -w file                     # word count
wc -c file                     # byte count
wc -m file                     # character count (differs from -c with multi-byte/UTF-8 text)

# tr — translate or delete characters (reads STDIN ONLY — never takes a filename argument)
tr 'a-z' 'A-Z' < file          # uppercase everything
tr -d '\r' < winfile > unixfile   # strip Windows carriage returns
tr -s ' ' < file                 # squeeze repeated spaces into one
tr -cd '[:print:]\n' < file      # keep only printable characters + newline, delete everything else
```

`cut` and naive delimiter-splitting share `awk`'s blind spot: they don't understand **quoted CSV fields**. A value like `"Smith, John"` inside a real CSV will get cut in the wrong place, because `cut`/`awk` just split on every occurrence of the delimiter character, quotes or not.

## The classic "top N" pipeline

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

This idiom appears everywhere because it composes four single-purpose tools into an exact answer to "what are the top N most frequent values of this field": extract the column, **sort it so identical values become adjacent** (required, because `uniq` only collapses neighbors), **count consecutive runs**, then sort the counts numerically descending, then take the top of the list. Each step does one job well — the Unix philosophy in action.

## Pipelines and redirection at the file-descriptor level

Every process starts with three open file descriptors: **0 = stdin, 1 = stdout, 2 = stderr**. Redirection operators just point these descriptors somewhere else before the program runs:

```bash
cmd > file        # stdout (fd 1) -> file, truncating it first
cmd >> file        # stdout -> file, appending
cmd < file         # stdin (fd 0) <- file
cmd 2> file        # stderr (fd 2) -> file
cmd 2>&1           # duplicate fd 2 onto whatever fd 1 currently points to
cmd > file 2>&1    # BOTH stdout and stderr end up in file (order matters — see below)
cmd 2>&1 > file    # stderr goes to the OLD stdout (usually the terminal); only stdout goes to file
```

Redirections are evaluated **left to right**, and `2>&1` copies *the current target* of fd 1 at the moment it runs — it does not create an ongoing link between the two descriptors. `cmd > file 2>&1` works because by the time `2>&1` executes, fd 1 already points at `file`. `cmd 2>&1 > file` does the opposite order: fd 2 is first pointed at whatever fd 1 was (the terminal), and *then* fd 1 gets redirected to `file` — leaving fd 2 still pointing at the terminal. This ordering trap is one of the most-repeated shell interview questions for a reason.

A **pipe** (`|`) connects the stdout of one process directly to the stdin of the next through an in-kernel buffer — both processes are started essentially simultaneously and run **concurrently**, not sequentially, each blocking when the buffer is full (writer) or empty (reader). On Linux this buffer is a fixed size (traditionally 64KB). This is *why* streaming pipelines scale to files far bigger than RAM: at any instant, only a small buffer's worth of data is in flight between stages — `grep`, `awk`, and `sed` each hold only the current line (or a small buffer) in memory, never the whole file. Contrast this with reading an entire file into a language's list/array (e.g. Python's `.readlines()`) or a bash array — memory usage there grows with file size, and a 10GB file means a 10GB(+) resident process.

## `jq` — the same idea, for JSON

Structured JSON logs break `grep`/`sed`/`awk`'s line-and-field assumptions: JSON isn't guaranteed to be one record per line unless the producer specifically emits JSON Lines (`.jsonl`), whitespace/key order isn't stable, and values can be arbitrarily nested. `jq` parses JSON structurally, so queries are correct regardless of formatting:

```bash
jq '.' file.json                        # pretty-print
jq -c '.' file.jsonl                     # compact (one JSON object per line) — good for piping onward
jq '.status' access.jsonl                # extract a field (still JSON-quoted if it's a string)
jq -r '.status' access.jsonl             # -r = raw output, strips quotes from string results
jq 'select(.status >= 400)' access.jsonl # filter — the JSON equivalent of grep/awk conditionals
jq -r '[.remote_addr, .status] | @csv' access.jsonl   # build a CSV row from JSON fields
```

`jq`'s filter language composes the same way pipelines do — `.` is "the current value," `|` chains transformations, and `select()` is jq's `if`. `@csv` and `@tsv` are built-in formatters specifically for turning JSON into flat delimited output, which is the direct JSON analogue of an `awk '{printf "%s,%s\n", ...}'` pipeline.

## Points to Remember

- `sed -i` is not truly in-place — it writes a temp file and renames it, which changes the inode, breaks hard links, and requires directory write permission.
- BSD `sed` (macOS default) requires an explicit `-i ''` argument; GNU `sed` (Linux) treats the suffix as optional. This is the #1 "works on my Mac, breaks in CI" `sed` bug.
- `uniq` only removes **adjacent** duplicates — always `sort` first, or counts will be wrong on unsorted input.
- `2>&1` copies the *current* target of stdout at the time it runs — it must come **after** the `>` redirect for both streams to land in the same file.
- Pipes use a small, fixed-size kernel buffer — this is exactly what makes pipelines memory-flat regardless of input size, which is why `grep`/`awk`/`sed` handle multi-gigabyte files that would crash a script that loads everything into memory first.
- `tr` only reads from stdin — it has no filename argument; you must redirect a file into it with `<`.
- `jq -r` strips the JSON string quoting — forgetting it is the most common reason a jq-built CSV comes out with literal `"quotes"` embedded in every field.

## Common Mistakes

- Running `sed -i 's/foo/bar/' file` on macOS and hitting `sed: 1: "file": invalid command code f` — BSD sed parsed the missing backup-suffix argument as part of the script. Fix: `sed -i '' 's/foo/bar/' file`, or install GNU sed (`brew install gnu-sed`, invoked as `gsed`) for cross-platform scripts.
- Writing `cmd 2>&1 > file` expecting both streams in `file`, then being confused when stderr still prints to the terminal — an ordering bug, not a typo.
- Running `uniq -c` directly on an unsorted file and reporting the (wrong) counts as if they were global — `uniq` silently only merges consecutive lines, so identical values separated by other lines are counted separately.
- Calling `tr 'a-z' 'A-Z' file.txt` expecting it to read the file — `tr` only ever accepts the two translation-set arguments, so a third positional argument is an **extra operand** and both GNU and BSD `tr` reject it outright with a usage error, rather than silently reading the file. Needs `tr 'a-z' 'A-Z' < file.txt`.
- Building "CSV" with `awk`/`cut` on data with quoted, comma-containing fields and getting silently misaligned columns — these tools split on every delimiter occurrence with no concept of quoting.
- Forgetting `jq -r` and shipping a "CSV" file where every field is wrapped in literal double quotes because the default `jq` output is JSON-quoted text.
