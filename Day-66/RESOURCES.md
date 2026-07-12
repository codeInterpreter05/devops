# Day 66 — Resources: Shift-Left Security

## Primary (assigned)

- **OWASP DevSecOps Guideline** (owasp.org/www-project-devsecops-guideline/) — free, the assigned starting point. Covers the full shift-left lifecycle (SAST, DAST, SCA, secret scanning, and pipeline integration) in one vendor-neutral reference.

## Deepen your understanding

- **OWASP Top Ten** (owasp.org/www-project-top-ten/) — the vulnerability classes SAST/DAST rule sets are built around; essential context for understanding *why* specific findings matter.
- **Semgrep Registry** (semgrep.dev/explore) — browse real rule sets (`p/security-audit`, `p/owasp-top-ten`, language-specific rules) to see exactly what patterns SAST tools actually look for.
- **Gitleaks documentation** (github.com/gitleaks/gitleaks) and **TruffleHog documentation** (github.com/trufflesecurity/trufflehog) — both READMEs cover detector rules, verification behavior, and CI integration in depth.
- **"Log4Shell" retrospectives** (search CVE-2021-44228 writeups from CISA or Cloudflare's blog) — the canonical case study for why SCA must scan the full transitive dependency tree, referenced in file 2.
- **pre-commit framework documentation** (pre-commit.com) — the tool used in file 3's examples; covers hook management, baseline files, and CI integration (`pre-commit run --all-files` as a CI step, for defense in depth).

## Reference / lookup

- **Bandit documentation** (bandit.readthedocs.io) — full rule reference for Python-specific SAST.
- **OWASP ZAP documentation** (zaproxy.org) — DAST scanning modes (baseline vs. full active scan) and CI integration via `zaproxy/action-baseline`.
- **National Vulnerability Database (nvd.nist.gov)** and **GitHub Advisory Database (github.com/advisories)** — the underlying CVE data sources SCA tools query against.

## Practice

- **OWASP Juice Shop** (owasp.org/www-project-juice-shop) — a deliberately vulnerable web application, free and open-source, purpose-built for practicing SAST/DAST/manual security testing without touching a real system. Directly usable for a deeper version of today's lab.
- Run Semgrep and Gitleaks against any large, real open-source repo (with permission/appropriate scope) to see the volume and variety of real-world findings versus the intentionally small lab repo — builds intuition for triage at scale.
