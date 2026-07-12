# Day 9 — Git Deep Dive II — Workflows: Signed Commits and GPG

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Git | **Flag:** —

## Brief

`git log` shows an `Author` and a `Committer` for every commit, and both are trivially fake — Git trusts whatever `user.name`/`user.email` are set locally, with no identity check whatsoever. Signed commits close that gap by attaching cryptographic proof that a specific keyholder — not just "whoever set that email in their config" — actually created the commit. This matters directly for the branch protection rule from file 1 ("require signed commits") and for supply-chain trust in general: a green "Verified" badge is a claim about *who*, not about *what* the code does.

## Why authorship is spoofable by default

```bash
git config user.name "Not Actually Me"
git config user.email "someone-else@example.com"
git commit -m "this looks like it came from someone else"
```

That commit's `Author` field will show exactly what was typed — Git performs zero verification against any real identity. This is by design: Git was built for fully distributed workflows (email patches, no central server, no accounts) where there was nothing to verify against. The consequence is that `git log --author=` or GitHub's contributor graphs are **claims**, not proof, unless signing is in place.

## How GPG signing works

GPG (GNU Privacy Guard) is an implementation of the OpenPGP standard, built on **asymmetric key pairs**: a private key you keep secret and use to *sign*, and a public key you share, which anyone can use to *verify* a signature came from the holder of the matching private key — without ever having access to the private key itself.

When you sign a commit, Git doesn't sign the diff — it signs the **commit object** itself (the structure containing the tree hash, parent hash(es), author, committer, and message). The signature is embedded in the commit object as an extra header, so it becomes part of the commit's own content and thus part of its SHA — you can't alter a signed commit's message or metadata without invalidating the signature.

Setup:
```bash
gpg --full-generate-key                 # generate a key pair (RSA 4096 or ed25519 recommended)
gpg --list-secret-keys --keyid-format=long   # find your key ID, e.g. 3AA5C34371567BD2

git config --global user.signingkey 3AA5C34371567BD2
git config --global commit.gpgsign true      # sign every commit automatically, not just when passing -S
git config --global tag.gpgsign true         # same, for annotated tags
```

With `commit.gpgsign true` set, `git commit` signs automatically; without it, you'd sign per-commit with `git commit -S`. Verify locally:

```bash
git log --show-signature -1
# gpg: Signature made ...
# gpg: Good signature from "Name <email>" [ultimate]
```

`git log --show-signature` shells out to `gpg --verify` under the hood, checking the signature against the public keys in your local GPG keyring — this is why a signature that shows "Good signature" on one machine can show "public key not found" on another that hasn't imported the signer's public key yet. The signature's validity doesn't depend on the platform at all — it's pure cryptography — but the platform's *display* of a "Verified" badge does depend on the platform knowing your public key.

## GitHub / GitLab "Verified" badge mechanics

- You upload your **public key** to your GitHub/GitLab account (Settings → SSH and GPG keys, for GitHub).
- On every push, the platform checks incoming commits' signatures against public keys registered to *some* account.
- If the signature verifies **and** the signing key's associated email matches a verified email on that account, the commit gets a green **"Verified"** badge in the UI.
- If the signature is valid but the key isn't registered to any account (or the email doesn't match), you get an **"Unverified"** badge — the signature might still be cryptographically valid, the platform just can't attribute it to a known identity.

This is exactly why "require signed commits" as a branch protection rule (file 1) is meaningful: it forces every commit landing on a protected branch to carry a signature verifiable against a registered key, converting "the committer field says Alice" into "Alice's private key was used to produce this commit" — a materially stronger claim, and one that (combined with required PR review) makes it much harder for a compromised or spoofed identity to land code unnoticed.

## SSH-based commit signing — the modern lighter-weight alternative

Git 2.34+ supports using an **SSH key pair** instead of GPG for signing — reusing keys most engineers already generated for authentication, avoiding GPG's notoriously clunky UX (key generation, keyservers, web-of-trust, expiring keys).

```bash
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# an "allowed signers" file lists which public keys are trusted to sign,
# used for *local* verification with `git log --show-signature`
echo 'your-email@example.com ssh-ed25519 AAAA...' >> ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
```

Upload the same SSH public key to GitHub/GitLab a second time, specifically as a **signing key** (GitHub distinguishes "Authentication Key" vs "Signing Key" uploads for the same key) — the platform then verifies commit signatures against it the same way it would a GPG key, badge and all.

**Why teams are moving to this:** no separate GPG toolchain to install, no key-expiry/keyserver ceremony, and it reuses infrastructure (SSH keys) that already exists for pushing over SSH — one fewer credential type to manage and rotate.

## Points to Remember

- `Author`/`Committer` are unauthenticated by default — anyone can set `user.name`/`user.email` to anything; signing is what makes authorship a verifiable claim instead of a string.
- GPG signing uses asymmetric keys: private key signs, public key (shared, uploaded to GitHub/GitLab) verifies. The signature covers the commit object itself, so altering a signed commit invalidates its signature.
- `commit.gpgsign true` signs every commit automatically; `git log --show-signature` verifies locally against your GPG keyring, independent of any platform.
- A platform "Verified" badge requires the public key to be registered to an account *and* the signing key's email to match a verified account email — a valid signature from an unregistered key shows as "Unverified," not invalid.
- SSH-based signing (Git 2.34+, `gpg.format ssh`) is a lighter-weight alternative reusing existing SSH keys, avoiding GPG's setup overhead, and gets the same "Verified" badge treatment on GitHub/GitLab.

## Common Mistakes

- Assuming a commit's `Author` field is proof of who wrote it — it's exactly as trustworthy as a from: header on an email, i.e., not at all, unless the commit is signed.
- Setting `commit.gpgsign true` without the key actually available/unlocked (e.g., on a CI runner with no GPG agent configured), causing every commit/tag operation to fail with a cryptic `gpg failed to sign the data` error.
- Confusing "signature verifies" with "signature is registered to a known identity" — a valid but unregistered-key signature shows as Unverified on GitHub, which looks alarming but isn't the same as an invalid/forged signature.
- Uploading an SSH key only as an "Authentication Key" on GitHub and expecting commit signatures to verify — GitHub requires the same key to also be added as a "Signing Key" before it will check commit signatures against it.
- Enabling "require signed commits" as a branch protection rule without first rolling out key setup/documentation to the team, which silently blocks everyone's merges the next time they push.
