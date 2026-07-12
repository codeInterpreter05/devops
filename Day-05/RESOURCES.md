# Day 5 — Resources: Bash Scripting I

## Primary (assigned)

- **Bash Guide** (mywiki.wooledge.org/BashGuide) — free, thorough, and written by the people behind the Greybeards' `#bash` IRC channel FAQ. Covers exactly the ground this day is built on — quoting, variables, control flow, and the "gotchas" — with far more rigor than most tutorials. The assigned starting point for this day.

## Deepen your understanding

- **BashPitfalls** (mywiki.wooledge.org/BashPitfalls) — a companion page to the Bash Guide listing the most common real-world scripting mistakes (many covered in today's notes) with explanations of exactly why each one breaks.
- **Google Shell Style Guide** (google.github.io/styleguide/shellguide.html) — a concise, opinionated reference on formatting, quoting, and structuring bash scripts the way production engineering orgs actually expect.
- **Julia Evans — "Bash scripting" zines/blog posts** (jvns.ca) — short, illustrated, practical explanations of `set -e`, quoting, and other bash internals, good for building real intuition quickly.
- **`man bash`** — the canonical, complete reference. Dense, but authoritative — use it to verify exact behavior of parameter expansion forms (`${VAR:-x}` etc.) and `set` options once you've built intuition from the guide above.

## Reference / lookup

- **ShellCheck** (shellcheck.net) — paste a script into the web UI (or run the CLI tool, as in today's lab) to get every quoting/word-splitting/logic issue explained with a linked `SC####` code. The single most useful tool for catching exactly the mistakes covered in this day's notes.
- **explainshell.com** — paste any shell command or script line and get every token, flag, and quoting choice broken down inline.

## Practice

- **OverTheWire: Bandit** (overthewire.org/wargames/bandit) — reinforces shell fundamentals hands-on via progressively harder SSH challenges; several late-game levels specifically require writing small bash scripts.
- **Exercism's Bash track** (exercism.org/tracks/bash) — free, structured coding exercises with automated feedback, good for drilling control flow, quoting, and script structure in isolation before applying it to real infra scripts.
