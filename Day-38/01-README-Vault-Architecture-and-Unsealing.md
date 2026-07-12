# Day 38 — HashiCorp Vault: Architecture & Unsealing

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets | **Flag:** ⚡ Interview-critical

## Brief

Ansible Vault (Day 37) encrypts secrets *at rest in files*. HashiCorp Vault is a different, much bigger thing entirely: a dedicated **secrets management server** that stores, generates, and tightly controls access to secrets at runtime, with audit logging, fine-grained policies, and — critically — the ability to generate secrets that don't exist until the moment they're needed and expire shortly after. Every team running anything beyond a handful of services eventually hits the "secrets sprawl" problem: API keys in Slack messages, DB passwords in `.env` files copied around, the same static credential used everywhere with no way to know who has it or revoke it individually. Vault is the industry-standard answer, and "explain Vault's unsealing process" is one of the most reliable ways an interviewer checks whether you've actually operated Vault versus read its landing page.

This day is split into three files:

1. **This file** — Vault's core architecture, sealing/unsealing, and Shamir's Secret Sharing.
2. **[02-README-Secret-Engines-and-Auth-Methods.md](02-README-Secret-Engines-and-Auth-Methods.md)** — secret engines (KV, AWS, PKI, Database) and auth methods (Kubernetes, AWS IAM).
3. **[03-README-Dynamic-Secrets-and-K8s-Injection.md](03-README-Dynamic-Secrets-and-K8s-Injection.md)** — dynamic secrets for databases and the Agent Sidecar Injector.

## What Vault actually is

Vault is a server (typically run in a highly-available cluster) that sits between your applications and every sensitive credential they need. Instead of an application reading a static DB password from an environment variable, it **authenticates to Vault** (proving its identity via one of several auth methods) and **requests a secret**, which Vault generates or retrieves on demand, subject to a policy that says exactly what that identity is allowed to read.

The core value proposition, in three parts:
1. **Centralized secret storage** with encryption at rest and a full audit log of every access.
2. **Fine-grained access control** — policies define exactly which paths an identity can read/write, down to the individual secret.
3. **Dynamic secrets** — Vault can generate a brand-new, short-lived credential (a DB user, an AWS IAM credential) at request time and automatically revoke it later, instead of everyone sharing one long-lived static secret forever (covered in depth in file 3).

## The seal — Vault's core security model

Vault's storage backend (the physical place secrets live — often Consul, integrated Raft storage, or a cloud KMS-backed store) holds only **encrypted** data. The encryption key needed to decrypt that data is itself protected by another layer: the **master key**, generated when Vault is first initialized. Vault is said to be **sealed** whenever it doesn't have the master key in memory — in the sealed state, Vault can respond to health checks but **cannot** decrypt anything or serve any secret, even to a legitimate, authenticated request. This is by design: sealing is the mechanism that protects Vault's data even if someone gets direct access to the underlying storage disk — a stolen storage snapshot is just ciphertext without the master key.

Vault becomes sealed:
- Every time the Vault process **starts** (it always boots sealed).
- If storage becomes corrupted or the process detects a serious internal problem (auto-seals as a safety measure).
- If manually sealed by an operator (`vault operator seal`) — an emergency "kill switch" if a compromise is suspected.

## Shamir's Secret Sharing — how unsealing actually works

Rather than storing the master key as one single, all-powerful secret that any one operator could leak or lose, Vault (in its default configuration) splits it using **Shamir's Secret Sharing (SSS)** into **N key shares**, requiring any **T** of them (a threshold) to reconstruct the master key — commonly configured as 5 shares, 3 required (`-key-shares=5 -key-threshold=3`).

```bash
vault operator init -key-shares=5 -key-threshold=3
```

This produces 5 **unseal keys** (distributed to 5 different trusted operators — no single person should hold more than one) and a **root token** (Vault's initial highest-privilege credential, meant to be used briefly to configure real auth methods/policies, then revoked).

To unseal:

```bash
vault operator unseal <key-1>
vault operator unseal <key-2>
vault operator unseal <key-3>
```

Vault needs **any 3 of the 5** keys, submitted one at a time (in any order, by any 3 of the 5 keyholders, possibly from different machines/sessions) — after the threshold is met, Vault reconstructs the master key in memory, decrypts the encryption key it protects, and transitions to **unsealed**, ready to serve requests. Why this matters operationally: no single compromised or disgruntled operator can unseal Vault alone (protects against insider threat and single-point key leakage), but you're also not fully dependent on any *one* specific person being available (protects against bus-factor/availability risk) — you just need *some* quorum of the trusted group.

### Auto-unseal — the production-standard alternative

Manually gathering 3 human operators to type in unseal keys every time a Vault node restarts doesn't scale operationally, especially for auto-scaling Vault clusters or frequent restarts. **Auto-unseal** delegates the unsealing step to an external, already-highly-available KMS (AWS KMS, GCP Cloud KMS, Azure Key Vault, or a Vault Enterprise HSM):

```hcl
seal "awskms" {
  region     = "ap-south-1"
  kms_key_id = "alias/vault-unseal-key"
}
```

With auto-unseal, Shamir shares are still generated at init time (as a "recovery key" mechanism for extreme scenarios), but routine restarts unseal automatically by calling out to the KMS to decrypt the stored key — no human unseal ceremony needed for normal operations. This is now the standard production pattern; manual Shamir unsealing is mostly discussed for its conceptual security model and used in learning/dev environments or when a team specifically wants no dependency on a cloud KMS.

## Points to Remember

- Vault stores only encrypted data at rest; being "sealed" means the master key needed to decrypt it isn't currently in memory — a sealed Vault cannot serve any secret, period, even to an authenticated caller.
- Shamir's Secret Sharing splits the master key into N shares requiring a threshold T to reconstruct — no single operator can unseal Vault alone, but the whole group isn't a single point of failure either.
- `vault operator init` produces the unseal key shares and an initial root token; the root token should be used briefly to bootstrap real auth/policies, then revoked — it should never be a long-term operational credential.
- Vault always boots sealed — every restart requires either the manual Shamir unseal ceremony or an auto-unseal mechanism (cloud KMS) to reach the unsealed, serving state.
- Auto-unseal (AWS KMS/GCP KMS/Azure Key Vault) is the production default because it removes the operational burden of gathering human operators on every restart, while Shamir shares remain as a recovery mechanism.

## Common Mistakes

- Treating the root token as a normal, everyday credential embedded in scripts/CI — it's meant to be used briefly at initial setup to create scoped policies/auth methods, then revoked (`vault token revoke`); leaving it active and widely used is a major blast-radius risk.
- Storing all N Shamir unseal key shares in the same place (e.g., one password manager vault, one engineer's laptop) — this defeats the entire point of splitting the key, since compromising that one location gives an attacker the whole threshold.
- Confusing "sealed" with "down"/"unhealthy" — a sealed Vault process is running and can answer health/status checks, it simply refuses to decrypt or serve secrets until unsealed; monitoring that only checks "is the process up" will miss a sealed-and-therefore-useless Vault.
- Forgetting that every single Vault node restart (rolling upgrade, crash, autoscaling event) requires unsealing again unless auto-unseal is configured — teams that set up manual Shamir unsealing in production without auto-unseal often get paged for routine restarts.
- Assuming a higher key-threshold is always "more secure" without considering operational risk — setting `key-threshold` equal to `key-shares` (e.g., 5-of-5) means losing even a single keyholder's share permanently locks you out of ever unsealing Vault again, since there's no operator token 2FA style recovery.
