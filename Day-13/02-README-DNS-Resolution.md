# Day 13 — TCP/IP & DNS: DNS Resolution Chain

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** ⚡ Interview-critical

## Brief

DNS is the piece of infrastructure everyone assumes "just works" until a deploy goes sideways because a cutover didn't propagate, or a pod can't reach a service because of an `ndots` misconfiguration silently eating request latency in failed lookups. Understanding the actual resolution chain — not just "DNS turns names into IPs" — is what lets you reason about propagation delay, caching behavior, and where a lookup is actually failing when `curl` hangs.

## The full resolution chain

A single `https://example.com` lookup can involve up to five caches/servers before an answer comes back:

```
Browser cache -> OS stub resolver (+ its cache) -> Recursive resolver -> Root server
                                                                            |
                                                              (referral to TLD server)
                                                                            v
                                                                       TLD server (.com)
                                                                            |
                                                            (referral to authoritative NS)
                                                                            v
                                                                  Authoritative nameserver
                                                                     (has the actual A record)
```

1. **Browser cache** — modern browsers keep their own short-lived DNS cache per-process, independent of the OS.
2. **OS stub resolver** — a lightweight resolver (`systemd-resolved` on most modern Linux, `dnsmasq`, or the plain glibc resolver reading `/etc/resolv.conf`) that either answers from its own cache or forwards the query onward.
3. **Recursive resolver** — the server actually configured in `/etc/resolv.conf` (your ISP's resolver, or a public one like `8.8.8.8`/`1.1.1.1`). This is the workhorse: if it doesn't have the answer cached, it does all the walking down the hierarchy on the client's behalf.
4. **Root servers** — 13 logical root server addresses (in reality, globally anycast clusters, not 13 physical machines) that don't know the answer but know which server is authoritative for the TLD (`.com`, `.org`, etc.) and refer the resolver there.
5. **TLD server** — knows which nameservers are authoritative for the specific domain (`example.com`) and refers the resolver there.
6. **Authoritative nameserver** — holds the actual zone data and returns the real A/AAAA/etc. record.

### Iterative vs. recursive queries

These two terms describe **who does the work** at each hop, and it's a common interview trip-up:

- The query from client → recursive resolver is **recursive**: "give me the final answer, or an error — don't just point me somewhere else."
- The queries the recursive resolver sends to root → TLD → authoritative are **iterative**: each server either gives the final answer (if it's authoritative for that name) or a **referral** ("I don't know, but ask this other server") — and the recursive resolver itself follows that chain hop by hop. Root and TLD servers refuse to do full recursive work for arbitrary clients; they only ever hand back referrals.

Watch the entire chain happen live with:
```bash
dig +trace example.com
```
This bypasses your configured recursive resolver's cache and performs the iterative walk itself, printing every hop from root down to the authoritative answer — the single best command for building intuition about this chain.

## Record types

| Type | Holds | Used for |
|---|---|---|
| `A` | An IPv4 address | The most common lookup — "what IP is this hostname" |
| `AAAA` | An IPv6 address | Same as `A` but for IPv6 |
| `CNAME` | Another hostname (an alias) | Pointing one name at another (e.g. `www` → the apex domain, or a managed load balancer's hostname); a name with a `CNAME` cannot have other record types at the same node (the "CNAME can't coexist" rule) |
| `MX` | A mail server hostname + priority | Where to deliver email for this domain |
| `TXT` | Arbitrary text | Domain ownership verification, SPF/DKIM/DMARC email-authentication policies |
| `NS` | A nameserver hostname | Delegation — "these servers are authoritative for this (sub)domain" |
| `SOA` | Zone metadata | Primary nameserver, admin contact, serial number (for zone-transfer versioning), refresh/retry/expire timers, and the **negative-caching TTL** |

Quick lookups:
```bash
dig example.com A          # just the A record
dig example.com AAAA
dig example.com MX
dig example.com NS
dig example.com TXT
dig example.com SOA
dig -x 93.184.216.34        # reverse lookup (PTR) — IP to name
```

## TTL, caching, and why it matters for deploys

Every record ships with a **TTL** (time-to-live, in seconds) telling every cache along the chain how long it's allowed to keep serving that answer without re-querying. This is exactly why you can't just flip a DNS record and expect instant global effect: any resolver that already cached the old value with, say, a 24-hour TTL will keep answering with the old IP for up to 24 hours from when *it* cached it — regardless of when you make the change.

The standard playbook before a DNS cutover (e.g. migrating to a new load balancer IP, or cutting over during a datacenter migration):
1. **Lower the TTL well in advance** of the cutover (e.g., drop a 24h TTL to 60s, a day or more before the change) — but this only helps once the *old*, long TTL has fully expired everywhere, since caches that already grabbed the record under the old TTL won't re-check just because you changed the TTL value itself.
2. Wait out the old TTL's duration.
3. Perform the actual cutover — propagation is now bounded by the short TTL you set.
4. Raise the TTL back up afterward once the new value is confirmed stable — a permanently short TTL means permanently higher query volume against your authoritative/recursive infrastructure.

**Negative caching** matters just as much and is often forgotten: an `NXDOMAIN` (no such record) response gets cached too, for a duration governed by the `SOA` record's minimum-TTL field (RFC 2308). Create a brand-new record and a resolver that queried it *before* it existed may keep returning "does not exist" until that negative-cache window expires — a common source of "I just created the record, why can't anyone resolve it yet" confusion.

## Kubernetes DNS

Inside a cluster, name resolution for pods and Services is handled by **CoreDNS**, running as a Deployment in `kube-system`, exposed via a Service still conventionally named `kube-dns` for backwards compatibility even though the actual server is CoreDNS. CoreDNS's `kubernetes` plugin watches the API server for `Service`, `Endpoints`, and `Pod` objects and answers queries following the cluster's DNS schema:

```
<service>.<namespace>.svc.cluster.local   ->  the Service's ClusterIP
                                                (or, for a headless Service, the pod IPs directly)
<pod-ip-dashed>.<namespace>.pod.cluster.local -> a specific pod's IP
```

Every pod's `/etc/resolv.conf` is populated by the kubelet with the cluster DNS IP as `nameserver`, plus `search` domains that let you use short names:

```
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

This is why, from a pod in namespace `default`, `curl myservice` resolves the Service in the *same* namespace, `curl myservice.other-ns` reaches across namespaces, and `curl myservice.other-ns.svc.cluster.local` is the fully-qualified form.

`ndots:5` is a well-known performance gotcha: it means any name with **fewer than 5 dots** is tried against each `search` suffix, in order, before being treated as already fully-qualified. Looking up an external name like `api.github.com` (2 dots) therefore triggers up to 4 doomed internal lookups (`api.github.com.default.svc.cluster.local`, `...svc.cluster.local`, `...cluster.local`, and the node's own search domain) before finally falling through to the plain external query — multiplying DNS traffic and latency for every external call a pod makes. Fixes in practice: append a trailing dot to fully-qualified external hostnames in application config (`api.github.com.` short-circuits the search list), tune `ndots` via the pod's `dnsConfig`, or run NodeLocal DNSCache to absorb the extra query volume cheaply.

## Points to Remember

- The chain is browser cache → OS stub resolver → recursive resolver → root → TLD → authoritative — the recursive resolver does the actual walking; your client just asks it once.
- Client-to-recursive-resolver queries are recursive ("give me the final answer"); recursive-resolver-to-{root,TLD,authoritative} queries are iterative (referrals, followed by the resolver itself).
- `CNAME` can't coexist with other records at the same name — that's a real, frequently-hit constraint (commonly forcing provider-specific `ALIAS`/`ANAME` workarounds at the zone apex).
- TTL controls propagation delay for updates; lower it *before* a cutover, not during, since the old (longer) TTL is what actually governs how stale caches remain.
- Negative caching (governed by the `SOA` minimum field) means a freshly created record can appear as `NXDOMAIN` to resolvers that queried before it existed, until that cache entry expires.
- In Kubernetes, CoreDNS answers `<service>.<namespace>.svc.cluster.local`; pod `/etc/resolv.conf` search domains are what make short/cross-namespace names work; `ndots:5` is a common hidden source of DNS latency for external calls.

## Common Mistakes

- Flipping a DNS record for a cutover without lowering TTL well in advance — assuming propagation is instant, then being surprised some fraction of traffic hits the old IP for hours.
- Forgetting negative caching exists — creating a record and immediately declaring DNS "broken" when a resolver cached the prior `NXDOMAIN`.
- Confusing `dig`'s default output (which just returns the final `ANSWER SECTION`) with actually understanding the chain — never having run `dig +trace` and therefore not knowing what "iterative" vs "recursive" concretely means when asked in an interview.
- Not knowing about `ndots` and being confused why external API calls from inside a pod are measurably slower than the same call made from a normal VM — it's extra failed internal DNS queries, not network latency.
- Assuming `CNAME` and `A`/`MX`/`TXT` records can coexist at the same name — they can't; this trips people up setting up an apex/root domain record.
