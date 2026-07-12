# Day 6 — Lab: Bash Scripting II — Text Processing

**Goal:** Parse an nginx-style access log using `grep`, `awk`, `sed`, and the small utilities to answer real questions — top offending IPs, error-rate breakdowns, CSV extraction — without ever loading the whole file into memory.

**Prerequisites:** A bash shell with `grep`, `awk`, `sed`, `sort`, `uniq`, `cut`, `wc`, `tr` (all standard on any Linux box / macOS) and `jq` installed (`sudo apt install jq` / `brew install jq`).

---

### Lab 1 — Generate a sample nginx access log

Real logs use the "combined" log format: `IP - - [timestamp] "METHOD path HTTP/1.1" status bytes "referer" "user-agent"`. Generate a synthetic one to work against.

1. Create a working directory and generate ~2,000 log lines with a realistic mix of IPs and status codes:
   ```bash
   mkdir -p ~/day6-lab && cd ~/day6-lab
   for i in $(seq 1 2000); do
     ips=(10.0.0.1 10.0.0.2 10.0.0.3 203.0.113.5 203.0.113.9 198.51.100.23 198.51.100.7)
     statuses=(200 200 200 200 301 404 404 403 500 502)
     ip=${ips[$((RANDOM % ${#ips[@]}))]}
     status=${statuses[$((RANDOM % ${#statuses[@]}))]}
     ts=$(date -d "-$((RANDOM % 86400)) seconds" +"%d/%b/%Y:%H:%M:%S %z" 2>/dev/null || date -v-"$((RANDOM % 86400))"S +"%d/%b/%Y:%H:%M:%S %z")
     echo "$ip - - [$ts] \"GET /api/v1/resource/$i HTTP/1.1\" $status $((RANDOM % 5000)) \"-\" \"curl/7.81.0\"" >> access.log
   done
   wc -l access.log
   ```
   (macOS's `date` and GNU `date` take relative-time flags differently — the command above tries GNU syntax first, falls back to BSD syntax.)

**Success criteria:** `access.log` exists with 2000 lines in combined log format, and `head -3 access.log` shows realistic-looking rows.

---

### Lab 2 — `grep`: find errors and validate the shape of the data

1. Count all 4xx and 5xx responses in one pass:
   ```bash
   grep -cE ' (4|5)[0-9]{2} ' access.log
   ```
2. Show only 5xx lines with 2 lines of context after each match:
   ```bash
   grep -A2 -E ' 5[0-9]{2} ' access.log | head -20
   ```
3. Confirm every line matches the expected combined-log shape (a quick sanity/validation pattern before trusting downstream parsing):
   ```bash
   grep -cvE '^[0-9.]+ - - \[.*\] ".*" [0-9]{3} [0-9]+ ".*" ".*"$' access.log
   ```
   This should print `0` — any non-zero count means malformed lines slipped in, which would silently break a naive `awk '{print $9}'` (see file 2's field-shift Common Mistake).

**Success criteria:** You can produce an error count with a single `grep -c`, and you've proven the log has one consistent shape before trusting field-position parsing.

---

### Lab 3 — The core hands-on activity: top 10 IPs, 4xx count, CSV output

This is the assigned hands-on activity — do it for real.

1. Top 10 IPs by request count, using the classic pipeline:
   ```bash
   awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
   ```
2. Same thing, but done as a single `awk` pass with an associative array (no external `sort|uniq` needed for counting, only for final ordering):
   ```bash
   awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn | head -10
   ```
3. Count 4xx errors specifically (field 9 is the HTTP status in combined log format: IP(1) -(2) -(3) [ts](4-5, space-split) "req"(6-8) status(9)):
   ```bash
   awk '$9 ~ /^4/ {c++} END {print c+0}' access.log
   ```
4. Break down 4xx errors **by IP**, sorted descending:
   ```bash
   awk '$9 ~ /^4/ {count[$1]++} END {for (ip in count) print count[ip], ip}' access.log | sort -rn
   ```
5. Output a clean CSV of `ip,status,bytes` for every request:
   ```bash
   awk '{gsub(/"/,"",$0); print $1","$9","$10}' access.log > requests.csv
   head -5 requests.csv
   ```
6. Output the **top 10 IPs** result itself as CSV with a header, using `printf`:
   ```bash
   { echo "ip,count"; awk '{count[$1]++} END {for (ip in count) print ip","count[ip]}' access.log | sort -t',' -k2,2rn | head -10; } > top10.csv
   cat top10.csv
   ```

**Success criteria:** `top10.csv` exists, has a header row plus 10 data rows sorted descending by count, and you can explain out loud why the `sort | uniq -c | sort -rn` idiom needs the *first* `sort` (uniq only merges adjacent duplicates).

---

### Lab 4 — `sed` transforms and a JSON-log variant with `jq`

1. Use `sed` to redact IPs in a copy of the log (replace the last octet with `xxx` — simple privacy-scrubbing exercise):
   ```bash
   sed -E 's/^([0-9]+\.[0-9]+\.[0-9]+)\.[0-9]+/\1.xxx/' access.log > access.redacted.log
   head -3 access.redacted.log
   ```
2. Use `sed -i` to fix a hypothetical typo across the log in place (make a backup first!):
   ```bash
   cp access.log access.log.orig
   sed -i.bak 's/curl\/7.81.0/curl\/8.0.0/' access.log
   diff access.log.bak access.log | head
   ```
   (On macOS, use `sed -i '' 's/.../.../' access.log` — no backup suffix supported the GNU way.)
3. Convert 20 sample lines into a JSON-Lines variant, then re-do the "top IPs" query with `jq` instead of `awk`:
   ```bash
   head -20 access.log | awk '{print "{\"ip\":\""$1"\",\"status\":"$9",\"bytes\":"$10"}"}' > sample.jsonl
   jq -r '.ip' sample.jsonl | sort | uniq -c | sort -rn
   jq -r 'select(.status|tonumber >= 400) | [.ip, .status] | @csv' sample.jsonl
   ```

**Success criteria:** You've transformed data with `sed` (both a filter and a real in-place edit with backup), and produced the same kind of "top N" / "filter by condition" answer via `jq` instead of `awk`, showing you can work with both line-oriented and structured (JSON) logs.

---

### Cleanup

```bash
cd ~ && rm -rf ~/day6-lab
```

---

### Stretch challenge

The interview question for today is: **"How would you extract all unique IP addresses from a 10GB log file efficiently?"** Answer it with a real command, then justify your choice in one or two sentences (why not load it into a Python list, why `sort` doesn't blow up memory the way you might expect, and what `sort`'s external-merge-sort behavior means for huge files):

```bash
awk '{print $1}' access.log | sort -u > unique_ips.txt
wc -l unique_ips.txt
```

Bonus: benchmark it against `awk '!seen[$1]++ {print $1}' access.log` (dedup inside a single `awk` pass using an associative array, no external `sort` at all) on your 2000-line file with `time`, and explain the memory/speed tradeoff between the two approaches at 10GB scale (the `awk` array approach holds one entry per **unique** IP in memory, not one per line — usually far smaller than the file itself for something like IP addresses, whereas GNU `sort` handles inputs larger than RAM by spilling sorted chunks to temp files on disk and merging them, which is exactly why `sort -u` doesn't require loading the whole 10GB into memory either).
