# Day 38 — HashiCorp Vault: Dynamic Secrets & Kubernetes Injection

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets | **Flag:** ⚡ Interview-critical

## Brief

This is the file that directly answers today's flagged interview question: *"How do dynamic secrets in Vault work for database credentials? What's the rotation cycle?"* Dynamic secrets are Vault's signature feature and the conceptual leap that separates it from "just an encrypted key-value store." Understanding the full lifecycle — lease, TTL, renewal, revocation — and how that gets delivered into a running pod via the Agent Sidecar Injector is exactly the kind of depth that distinguishes a candidate who has actually run Vault against a real database from one who's only read the marketing page.

## The core idea: credentials that don't exist until requested

With a **static** secret (a password stored in KV, or worse, a `.env` file), the same credential is used by every caller, indefinitely, until someone manually rotates it — which in practice, at most companies, means "rarely to never," because rotation is manual, risky, and easy to postpone.

With a **dynamic** secret, nothing exists in the database ahead of time. When an application requests a credential:

1. The app authenticates to Vault (via Kubernetes auth, AppRole, etc. — file 2).
2. The app reads from a path like `database/creds/readonly`.
3. Vault's **database secrets engine** connects to the actual database *at that moment*, using its own admin-level connection, and runs a **creation statement** to create a brand-new database role/user with a randomly generated username and password.
4. Vault hands that new username/password back to the app, along with a **lease** — a record of "this credential exists, it was granted to this token, and it expires at time X."
5. The app uses those credentials to connect to the database directly (Vault is not a proxy in the query path — it only mediates credential issuance).

```bash
vault read database/creds/readonly
```

```
Key                Value
---                -----
lease_id           database/creds/readonly/2f6a614c-...
lease_duration     1h
username           v-token-readonly-a1b2c3d4e5f6
password           A1b2C3d4E5f6G7h8!
```

Each *call* to `database/creds/readonly` mints a **different** username/password — two application instances calling the same role at the same time get two distinct, independent credentials, each individually trackable and individually revocable in Vault's lease database. This is the property that makes credential leaks in logs, crash dumps, or a compromised container far less catastrophic: the blast radius is one short-lived credential, not "the one password everything shares."

## Leases, TTL, renewal, and revocation — the rotation cycle

Every dynamic secret is issued with a **lease**:
- **`default_ttl`** — how long the credential is valid before Vault automatically revokes it, if nothing renews the lease.
- **`max_ttl`** — the hard ceiling; even with renewals, a lease cannot be extended past this regardless of how many times it's renewed.
- **Renewal** — a long-running application can call `vault lease renew <lease_id>` before expiry to extend the credential's life without getting a brand new one, useful when a connection pool wants to keep using the same credential a bit longer.
- **Revocation** — Vault revokes automatically when the lease expires (running a **revocation statement** against the database — typically `DROP ROLE`/`DROP USER`), or can be revoked immediately on demand: `vault lease revoke <lease_id>` — e.g., the moment you detect a compromised pod, you kill its lease and the underlying DB user disappears, immediately cutting off database access regardless of what the application itself does.

```bash
vault write database/roles/readonly \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  revocation_statements="DROP ROLE IF EXISTS \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

The **"rotation cycle"** the interview question asks about is really this whole loop: request → mint → short TTL → automatic revocation (or renew-until-`max_ttl`) → repeat on next request. There's no scheduled cron-style "rotate the password every 30 days" step at all — rotation is implicit and continuous, because every single credential is inherently temporary by construction, not batch-rotated after the fact.

### Root credential rotation (a separate, related concept)

Vault also supports rotating the **root/admin credential it uses to connect to the database itself** (distinct from the per-request dynamic creds it issues to callers):

```bash
vault write -f database/rotate-root/mydb
```

This changes Vault's own privileged DB connection password to a value **only Vault knows** — even a Vault administrator can no longer retrieve the plaintext of that root credential afterward, which closes the loop on "who could have this password" all the way up the privilege chain.

## Vault Agent — the sidecar that removes app-side Vault logic

Making every application natively speak the Vault API (authenticate, request creds, watch for renewal, handle revocation, retry on failure) is a real engineering burden multiplied across every service in a fleet. **Vault Agent** solves this by running as a separate process (or Kubernetes sidecar container) that does all of that on the application's behalf, writing the resulting secret to a location the app already knows how to read — usually a file — completely decoupling application code from Vault's API.

```hcl
# agent-config.hcl
auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "myapp"
    }
  }
  sink "file" {
    config = { path = "/vault/token" }
  }
}

template {
  source      = "/vault/templates/db-creds.tpl"
  destination = "/vault/secrets/db-creds.env"
}
```

The app just reads `/vault/secrets/db-creds.env` like any other config file — it has zero Vault-specific code, zero knowledge that dynamic secrets or Kubernetes auth are even involved.

## The Vault Agent Sidecar Injector for Kubernetes

Rather than hand-writing a sidecar container and Vault Agent config into every pod spec, the **Vault Agent Injector** (a Kubernetes mutating admission webhook, deployed via the official `vault` Helm chart) automatically injects an init container and/or sidecar into any pod carrying specific annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/readonly"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/readonly" -}}
          DB_USER={{ .Data.username }}
          DB_PASSWORD={{ .Data.password }}
          {{- end -}}
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myapp:latest
```

The mutating webhook intercepts pod creation, injects an init container that fetches the initial secret before the main container starts (so the app never races against a missing file), plus a sidecar that keeps renewing/re-fetching the secret and rewriting the file for as long as the pod lives. The application container itself needs **no changes at all** — no Vault SDK, no auth logic, just a file mounted at a known path that Vault Agent keeps fresh in the background, and the sidecar handles credential expiry (re-fetching before the TTL runs out) transparently to the app.

## Points to Remember

- Dynamic secrets don't exist until requested: Vault connects to the real backend (DB, AWS) at request time, creates a brand-new credential, and hands it back with a lease — no shared, long-lived static secret involved.
- Rotation is continuous by construction — TTL-bound leases expire and auto-revoke (running a revocation statement, e.g. `DROP ROLE`) rather than a scheduled batch "change the password" job.
- `default_ttl` is the normal lifespan; `max_ttl` is the hard ceiling regardless of renewals; `vault lease revoke` lets you kill a specific credential immediately (e.g., on detecting compromise).
- `database/rotate-root` rotates Vault's own admin connection credential to the database, to a value no human (including Vault admins) can retrieve afterward.
- The Agent Sidecar Injector uses pod annotations to auto-inject an init container + sidecar that authenticate, fetch, template, and continuously refresh secrets to a file — application code needs zero Vault awareness.

## Common Mistakes

- Describing dynamic secrets as "Vault automatically rotates the database's one shared password on a schedule" — that's not it; each request mints an independent, individually-trackable credential, and there generally is no single shared password to rotate at all once dynamic secrets are adopted.
- Setting `max_ttl` far higher than actually needed "just in case," which undermines the whole point of short-lived credentials — a leaked credential with a 30-day `max_ttl` is nearly as dangerous as a static one.
- Forgetting that an application must actively renew a lease (or the Agent must do it for them) before expiry — assuming credentials "just work forever once issued" leads to production outages when a long-running connection pool's underlying DB user gets revoked mid-operation.
- Applying the Vault Agent Injector annotations but forgetting to also configure a proper Kubernetes auth role bound to the right service account/namespace — the injected sidecar will fail to authenticate, and the failure often only surfaces as pod startup delay/crash, not an obviously Vault-related error.
- Treating Vault as a query proxy sitting between the app and the database — it isn't; Vault only issues the credential, the application connects to the database directly afterward using that credential.
