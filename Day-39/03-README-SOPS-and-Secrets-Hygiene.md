# Day 39 — Secrets in AWS & K8s: SOPS & Secrets Hygiene

**Phase:** 1 – Core DevOps | **Week:** W6 | **Domain:** Secrets

## Brief

SOPS (Secrets OPerationS, originally from Mozilla) is the other major "encrypt secrets so they're safe to commit to git" tool, alongside Sealed Secrets — but it's more general-purpose (works on YAML/JSON/ENV/INI files, not just Kubernetes Secrets) and increasingly paired with **age**, a modern, much simpler alternative to GPG for the underlying encryption. Beyond any specific tool, this file closes out the week's secrets theme with the non-negotiable hygiene rules — never in git, never in images — that every single tool covered this week (Ansible Vault, HashiCorp Vault, Secrets Manager, ESO, Sealed Secrets, SOPS) exists to enforce.

## SOPS — file-level encryption with partial-file granularity

SOPS's headline feature: it encrypts only the **values** in a structured file (YAML/JSON), leaving **keys** in plaintext — so a diff/PR review still shows *which* keys changed, even though you can't see the new secret values themselves.

```bash
brew install sops age
age-keygen -o key.txt   # generates a public/private age keypair; the public key goes in .sops.yaml
```

```yaml
# .sops.yaml — project-level config, so `sops` knows which key to use automatically
creation_rules:
  - path_regex: secrets/.*\.yaml$
    age: age1qz8x9...yourpublickeyhere
```

```bash
sops secrets/prod.yaml   # opens $EDITOR with plaintext view; encrypts on save, based on .sops.yaml rules
sops -e secrets/prod.yaml > secrets/prod.enc.yaml   # explicit encrypt
sops -d secrets/prod.enc.yaml                        # decrypt to stdout
```

Encrypted file, safe to commit (note only `password` values are ciphertext, keys stay readable):

```yaml
database:
    host: db.internal.example.com
    username: admin
    password: ENC[AES256_GCM,data:Hs8k2...,iv:...,tag:...,type:str]
sops:
    age:
        - recipient: age1qz8x9...
          enc: |
              -----BEGIN AGE ENCRYPTED FILE-----
              ...
              -----END AGE ENCRYPTED FILE-----
    lastmodified: "2026-01-15T10:00:00Z"
    version: 3.8.1
```

### Why `age` over GPG

GPG is powerful but notoriously painful operationally — key management, trust webs, expiring subkeys, and a UX most engineers actively avoid. **age** is deliberately minimal: a keypair is one line to generate, one public key line to distribute, no keyservers, no trust model beyond "here is my public key." SOPS supports both, but new projects overwhelmingly default to age now specifically because it removes GPG's operational overhead without giving up strong modern encryption (ChaCha20-Poly1305 under the hood).

### Consuming SOPS-encrypted files

- **Directly** — `sops -d secrets/prod.yaml | kubectl apply -f -` (decrypt just-in-time, pipe straight in, nothing plaintext ever touches disk).
- **Via the Flux SOPS integration** — Flux (a GitOps controller) can decrypt SOPS-encrypted manifests automatically in-cluster using a key it holds as a K8s Secret, applying them just like any other GitOps-managed resource — this is the most common real-world pattern, letting SOPS-encrypted files sit in a GitOps repo the same way Sealed Secrets does, but with SOPS's finer per-key granularity and non-K8s-specific format support.
- **Via `sops-exec`/direnv-style wrappers** for local development, injecting decrypted values as environment variables for a single command's lifetime without writing plaintext to disk at all.

## SOPS vs. Sealed Secrets — quick comparison

| | SOPS + age | Sealed Secrets |
|---|---|---|
| Scope | Any YAML/JSON/ENV/INI file, any use case | Specifically Kubernetes Secret objects |
| Diff-friendliness | High — only values are ciphertext, keys stay readable | Low — the whole `encryptedData` blob is opaque |
| Cluster dependency | None required to encrypt/decrypt (needs the key, not a live cluster) | Requires the specific cluster's controller to decrypt |
| Common integration | Flux native support | Native Kubernetes controller + `kubeseal` CLI |

Both solve "safe to commit to git," and picking one is often more about whether you're only dealing with Kubernetes Secrets (Sealed Secrets is simpler, purpose-built) or need to encrypt broader config across multiple formats/tools (SOPS is more general).

## The non-negotiable rule: never in git, never in images

Every mechanism this week — Ansible Vault, HashiCorp Vault, Secrets Manager/Parameter Store, ESO, Sealed Secrets, SOPS — exists because the default failure mode is catastrophically easy: a developer pastes a real API key into a config file, `git add .`, `git commit`, `git push`, and now that secret is in git history **forever**, readable by anyone with repo access, indexed by every clone, and — if the repo is ever made public even briefly, or a fork/mirror exists — potentially scraped by automated credential-harvesting bots within minutes.

Concrete practices that actually prevent this:
- **`.gitignore` real secret files by pattern** (`*.env`, `secrets/*.yaml` unless explicitly the `.enc.yaml`/sealed form) *before* the first commit that could contain one — gitignore only prevents *future* accidental adds, it does nothing for history already committed.
- **Pre-commit hooks / secret scanners** (`gitleaks`, `trufflehog`, `git-secrets`) run locally and in CI on every push — catching a leaked credential pattern (AWS access key format, private key headers, common API key prefixes) before it merges, and ideally before it's even pushed.
- **Never bake secrets into container image layers** — a `COPY .env .` or `ARG DB_PASSWORD` in a Dockerfile embeds that value into the image layer *permanently*; even if a later layer deletes the file, the value still exists in the earlier layer and is trivially extractable with `docker history`/`docker save` + inspection. Secrets belong in runtime injection (env vars from an orchestrator, mounted Secret volumes, ESO-synced Secrets) — never in the build context or image itself.
- **Rotate immediately, don't just delete, if a secret does leak** — removing a secret from the latest commit does not invalidate it; anyone who already has a clone (or scraped it via automation) still has the working credential. The only real remediation is rotating the actual credential at its source (AWS, database, third-party API), then cleaning history as a secondary step.

## Points to Remember

- SOPS encrypts values while leaving keys in plaintext, which is what makes SOPS-encrypted diffs reviewable in a PR — a real, practical advantage over whole-blob encryption schemes.
- `age` is the modern default keypair mechanism for SOPS specifically because it avoids GPG's operational complexity (keyservers, trust webs, subkey expiry) while using strong modern encryption underneath.
- SOPS is format/tool-agnostic (YAML/JSON/ENV/INI, any consumer); Sealed Secrets is Kubernetes-Secret-specific — pick based on whether Kubernetes Secrets are your only use case.
- A secret baked into a Docker image layer persists in that layer permanently and is extractable even if a later layer deletes the file — secrets must be injected at runtime, never at build time.
- Deleting a leaked secret from the latest git commit does not revoke it — rotating the actual credential at its source is the only real fix; git history cleanup is secondary cleanup, not remediation.

## Common Mistakes

- Believing `.gitignore` retroactively protects a secret that was already committed in an earlier commit — it only prevents new additions; the secret remains in history and reachable via `git log`/`git show` on that commit.
- Using `ARG` for secrets in a Dockerfile, thinking build args aren't persisted — they can still be inspected in image history/layer metadata unless very carefully using BuildKit's `--secret` mount feature (which is the actually-safe mechanism for build-time secret usage).
- Encrypting an entire file with SOPS but committing it with a filename/extension that doesn't match `.sops.yaml`'s `path_regex`, so a later `git add` or automated tooling accidentally commits a plaintext version alongside or instead of the encrypted one.
- Treating "we deleted the leaked API key from the repo" as complete remediation without rotating the actual key at the provider — the exposure window before deletion is often enough for automated scrapers to have already captured and used it.
- Sharing a SOPS/age private key or a Sealed Secrets controller's private key over an insecure channel (Slack DM, email) "just to unblock someone quickly" — undermines the entire security model these tools are built on, since possession of that key is equivalent to possession of every secret it protects.
