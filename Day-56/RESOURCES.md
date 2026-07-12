# Day 56 — Resources: Terraform Associate Exam Prep

## Primary (assigned)

- **HashiCorp Study Guide — Terraform Associate (003)** (developer.hashicorp.com/terraform/tutorials/certification-003) — the assigned starting point; the official objective list plus HashiCorp's own linked tutorials for each domain. Treat this as your syllabus, not just a reading.

## Deepen your understanding

- **Terraform docs — State** (developer.hashicorp.com/terraform/language/state) — the authoritative deep-dive on how state works, remote backends, locking, and the `terraform_remote_state` data source.
- **Terraform docs — CLI Commands: state / import / taint / graph** — command-by-command reference with every flag; worth reading in full once, since the exam tests exact command behavior, not just intent.
- **HashiCorp — Sentinel documentation** (developer.hashicorp.com/sentinel) — even without paid TFC/TFE access, the docs and example policy library are free to read and are the best way to understand the `tfplan`/`tfstate`/`tfconfig` imports Sentinel policies use.
- **Terraform Up & Running by Yevgeniy Brikman** (O'Reilly) — the most respected book-length treatment; the state management and team-workflow chapters go well beyond exam scope into real production patterns.

## Reference / lookup

- **Terraform Registry** (registry.terraform.io) — browse providers and modules; the exam expects you to know module versioning constraint syntax (`~>`, `>=`) and where the public registry fits into module sourcing.
- **HashiCorp Terraform Associate practice exam** (official, free) — the closest simulation of real question style and difficulty; take it more than once, spaced apart, to see which domains stay weak.

## Practice

- **Spin up a free Terraform Cloud org** (app.terraform.io) and migrate a throwaway local-state project to a remote workspace — the single best way to internalize domain 9 (TFC/TFE capabilities) instead of just reading about it.
- **KodeKloud / A Cloud Guru Terraform Associate practice questions** — timed question banks with explanations, good for spaced-repetition drilling in the two weeks before your exam date.
