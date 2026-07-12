# Day 38 — Resources: HashiCorp Vault

## Primary (assigned)

- **HashiCorp Vault tutorials** (developer.hashicorp.com/vault/tutorials) — the assigned starting point. Structured, hands-on, includes dedicated tutorials for unsealing, the database secrets engine, and Kubernetes auth used throughout today's notes and lab.

## Deepen your understanding

- **Vault official docs — Seal/Unseal concepts** (developer.hashicorp.com/vault/docs/concepts/seal) — the authoritative explanation of Shamir's Secret Sharing and auto-unseal mechanisms.
- **Vault official docs — Database secrets engine** — full reference for supported database plugins, creation/revocation statement templating, and root credential rotation.
- **Vault Agent Injector documentation** (developer.hashicorp.com/vault/docs/platform/k8s/injector) — the complete annotation reference used to configure sidecar injection.
- **"Vault: Secrets and Encryption Management" whitepaper (HashiCorp)** — good for the architectural framing interviewers often expect (why dynamic secrets, why leases, why auto-unseal).

## Reference / lookup

- **Vault CLI reference** (developer.hashicorp.com/vault/docs/commands) — every subcommand, useful as a quick lookup while doing the lab.
- **External Secrets Operator docs** (external-secrets.io) — used in the lab to sync Vault secrets into native Kubernetes Secrets; see Day 39 for deeper ESO coverage.

## Practice

- Run today's full lab against a real (free-tier) AWS account and a local `kind` cluster — actually watching an IAM user get created and destroyed by `vault lease revoke` builds far more intuition than reading about it.
- **HashiCorp Vault Learn labs (interactive, browser-based)** — if you don't want to install anything locally yet, these let you run through unsealing and secrets engine exercises in a sandboxed browser terminal first.
