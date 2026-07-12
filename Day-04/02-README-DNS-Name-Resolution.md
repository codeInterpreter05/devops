# Day 4 — Networking on Linux: DNS & Name Resolution

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

"It's always DNS" is a running joke in DevOps because it's true: DNS is the layer where a huge fraction of "the service is down" incidents actually live, and it's invisible until it breaks. Every `curl`, every microservice-to-microservice call, every `kubectl` command against a remote cluster starts with turning a name into an address. Interviewers ask about DNS constantly because it tests whether you understand a whole chain of config files and lookup order, not just "DNS translates names to IPs."

## How a name becomes an address on Linux

Resolution on a Linux host isn't one lookup — it's a chain, and where in that chain something is misconfigured determines the failure mode:

1. **`/etc/nsswitch.conf`** — tells the C library (glibc's resolver, via `getaddrinfo()`) *which sources to consult, and in what order*, for hostname lookups. The relevant line looks like:
   ```
   hosts: files dns
   ```
   This says: check `files` (i.e. `/etc/hosts`) first, then `dns`. Some systems add `mdns4_minimal` (multicast DNS, e.g. `.local` names on desktop Linux) or `resolve` (systemd-resolved) into this chain.
2. **`/etc/hosts`** — a static, manually maintained name-to-IP map, checked first (per nsswitch order above) before any network query happens. Format:
   ```
   127.0.0.1   localhost
   10.0.0.5    db.internal db
   ```
   Anything listed here **short-circuits DNS entirely** for that name — no network request is made, no TTL, no cache expiry. This is why it's a favorite tool for local testing (pointing a hostname at a different IP without touching DNS) and also a classic source of "why is this still resolving to the old IP" bugs long after the real DNS record changed.
3. **`/etc/resolv.conf`** — if `/etc/hosts` doesn't have an entry, this is where the system learns *which DNS resolvers to query*:
   ```
   nameserver 8.8.8.8
   nameserver 1.1.1.1
   search internal.example.com
   options timeout:2 attempts:3
   ```
   - `nameserver` lines are tried in order (first is primary; others are fallback on timeout, not load-balanced).
   - `search` appends domain suffixes to unqualified names — looking up `db` might actually query `db.internal.example.com` first because of a `search` entry. This is a very common source of confusion when a short hostname resolves differently across environments.
   - On many modern desktop/server distros this file is **auto-generated** by `systemd-resolved` or a DHCP client and gets overwritten — manually editing it can silently be reverted on the next network event or reboot. Check `readlink -f /etc/resolv.conf` — if it points at `/run/systemd/resolve/stub-resolv.conf` or `resolv.conf` is a symlink, you're looking at systemd-resolved's stub, and the real config lives in `resolvectl status` / `systemd-resolved`'s own config, not a file you should hand-edit.

## Querying DNS directly: `dig` vs `nslookup`

`dig` (part of `bind-utils`/`dnsutils`) is the modern, DevOps-standard tool — precise, scriptable, shows the full protocol exchange. `nslookup` is older, interactive-oriented, and its output is less structured, but it's still commonly pre-installed on minimal images/Windows, so you should be able to read both.

```bash
dig example.com                  # full answer, authority, and additional sections
dig example.com +short           # just the IP(s) — good for scripting
dig example.com A                # explicit record type
dig example.com AAAA             # IPv6 address record
dig example.com MX               # mail exchanger records
dig example.com NS               # authoritative nameservers for the domain
dig example.com TXT              # text records (SPF, domain verification, etc.)
dig @8.8.8.8 example.com         # query a specific resolver directly, bypassing /etc/resolv.conf
dig +trace example.com           # walk the full delegation chain from the root servers down
dig -x 93.184.216.34             # reverse lookup: IP -> name (PTR record)
```

Key parts of `dig`'s default output:
- **QUESTION SECTION** — what you asked.
- **ANSWER SECTION** — the record(s) returned, each with a **TTL** (seconds the answer may be cached) — a low TTL you see mid-incident often means someone just changed the record and is waiting for propagation.
- **Query time** and **SERVER** — which resolver actually answered and how long it took; useful for spotting a slow or unreachable resolver.

```bash
nslookup example.com             # basic A record lookup
nslookup example.com 8.8.8.8      # query a specific server
nslookup -type=MX example.com    # explicit record type
```

`dig +trace` is worth understanding conceptually even if rarely run in production debugging: it starts at the root nameservers, follows the referral to the TLD (`.com`) nameservers, then to the domain's authoritative nameservers — demonstrating that DNS is a **distributed, delegated hierarchy**, not one big database. This is the mechanism behind why a freshly registered domain can take time to become globally resolvable (delegation has to propagate) even though the authoritative answer is correct the instant it's published.

## Points to Remember

- Resolution order on Linux is governed by `/etc/nsswitch.conf`'s `hosts:` line — typically `files dns`, meaning `/etc/hosts` always wins over DNS if both have an entry.
- `/etc/hosts` entries never expire and bypass the network entirely — perfect for pinning a name locally, but a classic cause of "it works on my machine" when someone forgot they added an entry.
- `/etc/resolv.conf`'s `nameserver` lines are tried in order as fallbacks, not load-balanced; `search` domain suffixes get appended to unqualified (no-dot) hostnames, which can cause a short name to resolve differently in different environments.
- On systemd-based distros, `/etc/resolv.conf` is frequently a generated/symlinked file managed by `systemd-resolved` — check `resolvectl status` rather than assuming a manual edit will stick.
- `dig +short` for scripts, full `dig` output for debugging (TTL, which server answered, query time), `dig @server` to bypass your configured resolver and isolate whether the problem is the resolver or the record itself.

## Common Mistakes

- Editing `/etc/resolv.conf` directly on a systemd-resolved system and being confused when changes vanish after a reboot/network restart — the file is regenerated, not authoritative.
- Forgetting a stale `/etc/hosts` entry exists, then spending time debugging "DNS propagation" for a record that changed correctly in DNS but is being shadowed locally.
- Treating a low or zero TTL you see in `dig` output as an error — it's often intentional (e.g., during a planned cutover) and just means "don't cache this long."
- Not knowing about `search` domain suffixes and being surprised that `curl http://api` resolves successfully inside a Kubernetes pod (via cluster DNS search domains) but fails identically outside the cluster.
- Using `nslookup`'s default output to diagnose subtle issues (like which specific nameserver responded or TTL nuances) when `dig` surfaces that information much more directly.
