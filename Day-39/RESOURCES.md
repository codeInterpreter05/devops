# Day 39 — Resources: Secrets in AWS & K8s

## Primary (assigned)

- **External Secrets Operator documentation** (external-secrets.io) — the assigned starting point. Covers every supported `SecretStore` provider (AWS, Vault, GCP, Azure), `ExternalSecret` templating, and auth patterns used in today's lab.

## Deepen your understanding

- **AWS documentation — Secrets Manager vs. Systems Manager Parameter Store** (AWS's own comparison page) — the authoritative breakdown of pricing, rotation support, and API differences covered in file 1.
- **Sealed Secrets GitHub repo and docs** (github.com/bitnami-labs/sealed-secrets) — full `kubeseal` CLI reference, backup/restore guidance for the controller's private key (critical for disaster recovery).
- **SOPS GitHub repo** (github.com/getsops/sops) — full format support (YAML/JSON/ENV/INI/BINARY) and integration examples, including the Flux SOPS decryption integration.
- **age specification and tooling** (github.com/FiloSottile/age) — short, readable spec if you want to understand exactly what `age-keygen` and the encryption format are doing.

## Reference / lookup

- **`gitleaks` and `trufflehog` documentation** — configuring pre-commit hooks and CI scanning to catch leaked secrets before they merge.
- **Docker BuildKit secrets documentation** — the `--mount=type=secret` mechanism for safely using secrets at build time without persisting them into image layers.

## Practice

- Complete today's lab end to end against a real (free-tier) AWS account and local `kind` cluster, producing all three artifacts (`ExternalSecret`, `SealedSecret`, SOPS-encrypted file) for the same value — directly comparing the three is the fastest way to internalize their trade-offs.
- Run `gitleaks detect` against a few of your own older personal repos — finding (or not finding) real accidental secret commits in your own history is a memorable lesson in why these tools exist.
