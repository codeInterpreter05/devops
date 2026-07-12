# Day 4 — Resources: Networking on Linux

## Primary (assigned)

- **Julia Evans's Networking Zine** (free PDF, jvns.ca/networking-zine) — the assigned starting point for today. Short, illustrated, and excellent at building real intuition for how DNS, TCP, and packets actually behave — exactly the mental model this day's notes build on.

## Deepen your understanding

- **`man tcp`** and **`man 7 ip`** — run these on any Linux box; the `tcp(7)` and `ip(7)` man pages document socket states, options, and header fields directly from the source of truth, no internet required.
- **High Performance Browser Networking, Ch. 1–2** (Ilya Grigorik, free online at hpbn.co) — the clearest deep dive into TCP's handshake, congestion control, and why latency (not bandwidth) dominates real-world web performance.
- **Cloudflare Learning Center — "What happens when you visit a website?"** and their DNS/TCP/TLS explainer articles — concise, accurate, and written by people who operate this infrastructure at scale.
- **RFC 793 (TCP)** and **RFC 1035 (DNS)** — the actual specifications. Dense, but worth skimming once to see where terms like "three-way handshake" and "TTL" originate verbatim.

## Reference / lookup

- `man ip`, `man ss`, `man dig`, `man tcpdump` — your on-box references; `man tcpdump`'s EXAMPLES section alone covers most filter syntax you'll ever need.
- **explainshell.com** — paste any `tcpdump`/`curl`/`dig` invocation with flags to get each part explained inline.
- **Wireshark documentation** (wireshark.org/docs) — even if you're using `tcpdump` day to day, Wireshark's protocol reference pages are the best illustrated guide to what each TCP/DNS/TLS field means, and `tcpdump` capture files (`.pcap`) open directly in it for visual inspection.

## Practice

- **overthewire.org — Bandit / Natas** — not networking-specific, but several later levels require using `nc`/`curl` against real listening services to extract flags, which is good muscle memory for this day's tools.
- **neverssl.com / httpbin.org** — safe, plain-HTTP and inspectable-HTTP endpoints purpose-built for practicing `curl -v` and `tcpdump` without TLS obscuring the payload, or for testing headers/methods/redirects interactively.
