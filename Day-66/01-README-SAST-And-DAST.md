# Day 66 — Shift-Left Security: SAST & DAST

**Phase:** 2 – CI/CD & Security | **Week:** W10 | **Domain:** DevSecOps | **Flag:** ⚡ Interview-critical

## Brief

"Shift-left" means moving security testing as early as possible in the development lifecycle — ideally into the developer's editor and every pull request — instead of a security team finding problems in a pre-release audit or, worse, after an incident. The cost of fixing a vulnerability grows by roughly an order of magnitude at each stage it survives (code review vs. staging vs. production incident), which is the entire economic argument for shift-left. SAST and DAST are the two foundational, complementary automated testing categories every DevSecOps pipeline needs, and understanding *why* they're complementary (not redundant) is a frequent interview probe.

This day is split into three files:

1. **This file** — SAST (static analysis) and DAST (dynamic analysis), what each actually catches and why neither alone is sufficient.
2. **[02-README-Dependency-And-Secret-Scanning.md](02-README-Dependency-And-Secret-Scanning.md)** — dependency/SCA scanning and secret scanning (Gitleaks, Trufflehog).
3. **[03-README-Pre-Commit-Hooks-And-Security-Culture.md](03-README-Pre-Commit-Hooks-And-Security-Culture.md)** — pre-commit hooks for security, and developer security training/culture.

## SAST — Static Application Security Testing

SAST tools analyze **source code directly, without running it**, looking for known-dangerous patterns: SQL built via string concatenation, unsanitized input flowing into a shell command, hardcoded credentials, use of deprecated/broken crypto primitives, and so on.

```yaml
# GitHub Actions example: Semgrep as SAST in CI
jobs:
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
```

```bash
# Bandit for Python specifically
pip install bandit
bandit -r ./src -ll   # -ll = only report medium+ severity
```

**How it actually works**: SAST tools build an abstract syntax tree (or similar intermediate representation) of your code and pattern-match against a rule set — either simple syntactic patterns (`semgrep` rules look a lot like the code itself, with metavariables) or full **taint analysis**, tracing whether data from an untrusted source (`request.args`, user input) can reach a dangerous sink (a raw SQL query, `eval()`, a shell command) without passing through a sanitizer in between.

**What SAST is good at**: catching whole classes of bugs *before the code ever runs* — a hardcoded API key, a classic SQL injection pattern, use of `md5`/`eval`/`pickle.loads` on untrusted input. It runs in seconds to minutes, integrates directly into the PR review flow, and doesn't need a running application or test environment.

**What SAST fundamentally cannot catch**: anything that only manifests at runtime with real request/response behavior — a misconfigured CORS header actually being exploitable, an authentication bypass that only shows up when you trace an actual HTTP request through multiple services, a race condition, or business-logic flaws (e.g., "can user A view user B's invoice by changing an ID in the URL" — the code might look syntactically fine while still being logically broken).

## DAST — Dynamic Application Security Testing

DAST tools test a **running application from the outside**, the same way an actual attacker would — sending crafted HTTP requests and observing the real responses, with no access to (or need for) source code.

```yaml
# GitHub Actions example: OWASP ZAP baseline scan against a running app
jobs:
  dast:
    runs-on: ubuntu-latest
    steps:
      - name: ZAP baseline scan
        uses: zaproxy/action-baseline@v0.12.0
        with:
          target: 'https://staging.example.com'
          fail_action: true
```

**How it actually works**: the scanner crawls the application (following links, submitting forms), then actively fuzzes every discovered input (query params, form fields, headers, cookies) with payloads designed to trigger known vulnerability classes — injection payloads, XSS payloads, path traversal attempts — and inspects the actual HTTP responses for signs the payload succeeded (reflected input, error messages leaking stack traces, unexpected status codes, timing differences).

**What DAST is good at**: finding real, exploitable, runtime-observable issues — actual reflected/stored XSS, working SQL injection, misconfigured security headers, exposed debug endpoints, authentication/session flaws — with a very low false-positive rate for the things it does flag, because it's demonstrating actual exploitation, not just a suspicious code pattern. It also finds issues in the *deployed configuration*, not just the code (e.g., a security header that's correct in code but stripped by a misconfigured reverse proxy in front of it).

**What DAST fundamentally cannot catch**: anything it can't reach by crawling/fuzzing the exposed surface — code paths behind complex multi-step workflows the scanner doesn't know how to drive, business logic requiring domain knowledge, or vulnerabilities in code that's never actually invoked during the scan's exploration. It also runs much slower than SAST (minutes to hours against a real running target) and necessarily happens later in the pipeline, since there has to be a running instance to test.

## Why you need both, not one instead of the other

| | SAST | DAST |
|---|---|---|
| Needs running app? | No | Yes |
| Speed | Fast (seconds–minutes) | Slow (minutes–hours) |
| Pipeline stage | Earliest — on every commit/PR | Later — against a deployed staging/review environment |
| False positive rate | Higher (flags suspicious patterns, not confirmed exploits) | Lower (demonstrates actual exploitation) |
| Finds business-logic/auth flaws? | Rarely | Yes, if the scan can reach and exercise them |
| Finds config/infra-level issues? | No (code only) | Yes (actual deployed headers, exposed endpoints) |

They're complementary precisely because they look at different layers: SAST is about *what the code says*, DAST is about *what the running system actually does*. A codebase can pass SAST cleanly (no obviously dangerous code patterns) and still be exploitable at runtime due to misconfiguration, and a codebase can have "safe-looking" code by pattern but be logically broken in a way only a real crawl-and-fuzz exercise surfaces.

## Points to Remember

- SAST analyzes source code without execution (fast, early, PR-blocking); DAST tests a running instance externally (slower, later, needs a real deployed target).
- SAST uses pattern matching / taint analysis on an AST; DAST crawls and fuzzes real HTTP endpoints and inspects real responses.
- SAST catches hardcoded secrets, classic injection patterns, dangerous function usage; DAST catches actual exploitable runtime behavior, misconfigurations, and issues in the deployed environment (not just the code).
- SAST has a higher false-positive rate (flags patterns, not confirmed exploits); DAST has a lower false-positive rate but can miss anything outside its crawl/fuzz reach.
- "Shift-left" doesn't mean "replace DAST with SAST" — it means running both as early and often as feasible, with SAST gating every PR and DAST running against every staging deploy (at minimum).

## Common Mistakes

- Treating SAST results as ground truth and blocking every flagged finding without triage — SAST's false-positive rate means a blanket "zero findings" policy either trains developers to ignore the tool or wastes enormous time chasing non-issues.
- Assuming a clean SAST scan means the application is secure — it says nothing about runtime configuration, business logic flaws, or issues introduced by infrastructure (proxies, load balancers, misconfigured headers).
- Running DAST only right before a major release (or not at all) instead of against every staging deployment — by the time DAST-only findings surface, the vulnerable code may have been in production for weeks.
- Pointing a DAST scanner at a production environment without authorization/rate-limiting awareness — an aggressive fuzzing scan against prod can itself cause an outage (effectively a self-inflicted DoS).
- Choosing SAST or DAST as an either/or budget decision — they cover genuinely different failure modes, and skipping one leaves an entire class of vulnerabilities structurally undetected regardless of how well-tuned the other is.
