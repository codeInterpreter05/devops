# Day 6 — Bash Scripting II — Text Processing: awk for Field Extraction

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Shell | **Flag:**

## Brief

`awk` is what turns "grep found the lines with errors" into "here are the top 10 offending IPs, sorted, as CSV." Nearly every classic ops one-liner — parsing access logs, summing column values from `df`/`du` output, reshaping `ps` output, building a quick report from a CSV export — is `awk` doing field extraction and aggregation in a single pass. It shows up constantly in interviews because it's a compact way to test whether you understand *streaming* data processing rather than just knowing individual commands.

## The record/field model

`awk` reads input one **record** at a time (by default, a record = one line) and splits each record into **fields** using a separator. You then write `pattern { action }` pairs, and `awk` runs the action for every record where the pattern matches.

```bash
awk '{ print $1 }' file        # print the 1st field of every line
awk '{ print $1, $3 }' file    # print 1st and 3rd fields, separated by OFS (default: space)
awk '{ print }' file           # print the whole record ($0) — same as no action at all
awk '/ERROR/ { print }' file   # pattern = regex; only print matching lines (like grep)
awk '$3 == "500" { print $1 }' file   # pattern = a comparison
```

Key built-in variables:

| Variable | Meaning |
|---|---|
| `$0` | The whole current record (line) |
| `$1`, `$2`, … `$NF` | Individual fields |
| `NF` | Number of fields **in the current record** — recalculated per line |
| `NR` | Record number **across all input** processed so far (global counter) |
| `FNR` | Record number **within the current file** — resets to 1 for each new file |
| `FS` | Input Field Separator (default: any run of whitespace, and it strips leading/trailing whitespace too) |
| `OFS` | Output Field Separator, used when `awk` rebuilds `$0` (default: single space) |
| `RS` | Input Record Separator (default: newline — change it to parse multi-line records) |

`NR` vs `FNR` is a real gotcha when processing multiple files: `awk 'FNR==1{print FILENAME}' *.log` prints the first line of *each* file, because `FNR` resets per file while `NR` keeps climbing across all of them.

## `BEGIN` / `END` and pattern-action pairs

```bash
awk 'BEGIN{ print "starting" } { print } END{ print "done" }' file
```

`BEGIN` runs once, before any input is read (no `$0`/fields exist yet — useful for setting `FS`, printing a header, initializing counters). `END` runs once, after all input is consumed (fields still hold the *last* record's values, but you typically use it to print accumulated totals). Neither requires an action-less pattern — an empty pattern with an action means "run this action for every record"; a pattern with no action defaults to `{ print $0 }` (this is why `awk '/ERROR/' file` behaves like `grep ERROR file`).

## Setting the field separator

```bash
awk -F',' '{ print $2 }' data.csv          # comma-delimited
awk -F'\t' '{ print $1 }' data.tsv         # tab-delimited
awk 'BEGIN{FS=":"} { print $1 }' /etc/passwd   # set FS inside BEGIN instead of via -F
```

Gotcha worth internalizing: **the default `FS` (whitespace) collapses runs of spaces/tabs and ignores leading whitespace** — `"  a   b  c"` splits cleanly into 3 fields. But the moment you set `FS` to an explicit single character like `,`, `awk` no longer collapses repeats: `"a,,b"` splits into **three** fields (`a`, ``, `b`), with an empty middle field. This trips people up constantly on CSVs with empty cells.

## Aggregation with associative arrays

This is where `awk` stops being "a fancy `cut`" and becomes a real aggregation tool — arrays indexed by string keys (like a dictionary/map):

```bash
# Count occurrences of the 1st field (e.g. IP addresses) without piping to sort|uniq at all
awk '{ count[$1]++ } END { for (ip in count) print count[ip], ip }' access.log

# Sum a numeric column (e.g. bytes transferred)
awk '{ total += $10 } END { print total }' access.log

# Count lines matching a condition (4xx status codes assumed in field 9)
awk '$9 ~ /^4/ { c++ } END { print c+0 }' access.log
```

`~` is the regex-match operator (`!~` is "does not match"). `count[ip]++` auto-initializes unseen keys to 0 before incrementing — no need to pre-declare the array. **Iteration order of `for (key in array)` is not guaranteed** (POSIX doesn't specify it, and different `awk` implementations order it differently) — if you need a specific order, pipe the output through `sort` afterward, exactly like the classic `awk '{count[$1]++} END{for (k in count) print count[k], k}' | sort -rn | head` "top N" idiom.

## Rebuilding `$0` after changing `OFS`

```bash
awk -F',' 'BEGIN{OFS="|"} { print $1, $2 }' file.csv     # works: OFS applies when you print fields explicitly
awk -F',' 'BEGIN{OFS="|"} { $1=$1; print }' file.csv     # needed if you want to print $0 itself with the new separator
```

Setting `OFS` alone does **not** retroactively rewrite `$0` — `awk` only rebuilds `$0` (using the current `OFS`) when you **assign to any field**, even a no-op assignment like `$1=$1` or `NF=NF`. This is a very common source of "I set OFS but `print $0` still shows commas" confusion.

## Formatted / CSV output with `printf`

```bash
awk -F' ' '{ printf "%s,%s,%s\n", $1, $9, $10 }' access.log
```

`printf` (unlike `print`) gives exact control over formatting — essential when the output needs to be valid CSV, padded columns, or a specific number of decimal places.

## Why `awk` scales where a shell loop doesn't

`awk` is a single process that reads its input as a stream, one record at a time, applying compiled pattern-action logic per line — it never loads the whole file into memory. Contrast this with the common anti-pattern `while read line; do ... done < file` combined with spawning `cut`/`grep`/`echo` *inside* the loop — that forks a new process per line, which is drastically slower on large files. `awk` (and `sed`, `grep`) do the looping *and* the field logic inside one process, in C, which is why a single `awk` command outperforms an equivalent shell loop by orders of magnitude on multi-million-line files.

## Points to Remember

- `NF` is recalculated per record (dynamic); `NR` is a running total across all input; `FNR` resets per file.
- Default `FS` (whitespace) collapses repeated separators and trims edges; an explicit single-character `FS` (e.g. `FS=","`) does **not** collapse repeats — empty fields become real empty strings.
- `count[key]++` auto-vivifies array entries — no initialization needed, and referencing `count[key]` in a boolean context safely returns `0`/empty if unseen.
- `for (key in array)` order is unspecified — pipe to `sort` if you need determinism.
- Changing `OFS` only affects output the next time `$0` is rebuilt (via a field assignment) — it does not retroactively change an already-read `$0`.
- A pattern with no action defaults to `{ print $0 }`; an action with no pattern runs on every record.

## Common Mistakes

- Assuming `awk -F','` behaves like a "smart CSV parser" — it does not understand quoted fields, so a value like `"Smith, John"` inside a real CSV gets split into two fields incorrectly. Plain `awk`/`cut` field-splitting is for simple delimited data only; for quoted CSV, use `csvkit`, a real CSV library, or gawk's `--csv`/`FPAT` extensions (GNU-only).
- Hardcoding a field number like `$9` for a log's status code and having it silently break the day someone adds a field upstream (e.g., nginx config adds `$http_x_forwarded_for`), shifting every subsequent column by one — validate assumptions with `NF` checks or named-field parsing (e.g., JSON logs + `jq`) when the format might change.
- Confusing `NR` and `FNR` when processing multiple files and getting per-file line numbers wrong (or vice versa).
- Forgetting that `OFS` doesn't apply until a field is reassigned, then being confused that `print $0` still shows the original separator.
- Using a shell `while read` loop with `cut`/`awk` called per iteration instead of doing the whole job in one `awk` invocation — correct in output, dramatically slower at scale.
