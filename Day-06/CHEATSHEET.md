# Day 6 — Cheatsheet: Bash Scripting II — Text Processing

## `grep`

```bash
grep "pattern" file            # literal/basic-regex substring match
grep -i "pattern" file         # case-insensitive
grep -v "pattern" file         # invert match (lines NOT matching)
grep -c "pattern" file         # count of matching lines
grep -n "pattern" file         # show line numbers
grep -o "pattern" file         # print only the matched substring
grep -A3 -B1 "pattern" file    # 3 lines after, 1 line before each match
grep -r "pattern" dir/         # recurse into a directory
grep -l "pattern" *.log        # filenames WITH a match
grep -L "pattern" *.log        # filenames WITHOUT a match
grep -w "word" file             # whole-word match
grep -x "line" file              # whole-line match
grep -F "literal.string" file   # fixed string, no regex (fast)
grep -q "pattern" file           # quiet — exit code only, no output
grep -E 'a+|b?' file              # ERE: +, ?, |, (), {} special
grep    'a\+\|b\?' file           # BRE (default): same meaning, escaped
```

Exit codes: `0` = match found, `1` = no match, `2` = error.

## Regex anchors & classes (BRE/ERE, POSIX)

```
^        start of line
$        end of line
.        any single character
[abc]    any of a, b, c
[^abc]   none of a, b, c
[a-z]    range
[[:digit:]]  POSIX digit class (locale-safe)
[[:alpha:]]  POSIX letter class
[[:space:]]  POSIX whitespace class
```

## `awk`

```bash
awk '{ print $1 }' file                     # 1st field
awk '{ print $1, $3 }' file                  # 1st + 3rd field
awk -F',' '{ print $2 }' file.csv            # custom field separator
awk '/ERROR/ { print }' file                  # regex pattern filter
awk '$3 == "500" { print $1 }' file           # comparison pattern
awk '{ count[$1]++ } END { for (k in count) print count[k], k }' file   # aggregate
awk 'BEGIN{FS=":"} {print $1}' /etc/passwd    # set FS in BEGIN
awk 'NR==FNR{a[$1]=1;next} $1 in a' f1 f2     # classic two-file join idiom
```

Built-ins: `$0` whole record, `$1..$NF` fields, `NF` field count (per record), `NR` record number (global), `FNR` record number (per file), `FS`/`OFS`/`RS` separators.

## `sed`

```bash
sed -n '2,5p' file            # print only lines 2-5
sed '3d' file                  # delete line 3
sed '/^#/d' file                # delete lines matching regex
sed 's/foo/bar/' file            # replace 1st match per line
sed 's/foo/bar/g' file           # replace all matches per line
sed 's/foo/bar/2' file            # replace only the 2nd match per line
sed -E 's/(a)(b)/\2\1/' file       # ERE + backreferences
sed -i 's/foo/bar/' file           # GNU: in-place, no backup
sed -i.bak 's/foo/bar/' file        # GNU: in-place with backup
sed -i '' 's/foo/bar/' file          # BSD/macOS: empty suffix required
```

## `cut` / `sort` / `uniq` / `wc` / `tr`

```bash
cut -d',' -f1,3 file.csv       # fields 1,3 by delimiter
cut -c1-10 file                 # characters 1-10

sort file                       # lexicographic
sort -n file                     # numeric
sort -r file                      # reverse
sort -k2,2 file                    # sort by field 2
sort -t',' -k3,3n file.csv          # custom delim, numeric field 3
sort -u file                         # sort + dedupe
sort -h file                          # human-numeric (1K, 2M, 3G)

uniq file                        # collapse ADJACENT dupes (sort first!)
uniq -c file                      # prefix with count
uniq -d file                       # only duplicated lines
uniq -u file                        # only unique lines

wc -l / -w / -c / -m file        # lines / words / bytes / chars

tr 'a-z' 'A-Z' < file             # translate (stdin only, no filename arg)
tr -d '\r' < f > out                # delete characters
tr -s ' ' < file                     # squeeze repeats
```

## The "top N" pipeline

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

## Pipelines & redirection

```bash
cmd > file        # stdout -> file (truncate)
cmd >> file        # stdout -> file (append)
cmd < file          # stdin <- file
cmd 2> file          # stderr -> file
cmd 2>&1              # stderr -> current stdout target
cmd > file 2>&1        # both -> file (correct order)
cmd 2>&1 > file          # WRONG order: stderr stays on terminal
cmd1 | cmd2                # pipe: cmd1 stdout -> cmd2 stdin, concurrent, small kernel buffer
```

## `jq` (JSON)

```bash
jq '.' file.json                       # pretty-print
jq -c '.' file.jsonl                    # compact, one object per line
jq '.status' file.jsonl                  # extract field (JSON-quoted)
jq -r '.status' file.jsonl                # raw output, no quotes
jq 'select(.status >= 400)' file.jsonl      # filter
jq -r '[.ip, .status] | @csv' file.jsonl     # build CSV row
```

## Efficient unique-IP extraction (interview one-liner)

```bash
awk '{print $1}' huge.log | sort -u > unique_ips.txt      # sort spills to disk, handles > RAM
awk '!seen[$1]++ {print $1}' huge.log > unique_ips.txt    # single pass, memory = unique key count
```
