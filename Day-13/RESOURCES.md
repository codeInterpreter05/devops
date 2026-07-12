# Day 13 — Resources: TCP/IP & DNS

## Primary (assigned)

- **High Performance Browser Networking** (free, hpbn.co) — the assigned resource for today. Ilya Grigorik's book covers TCP, TLS, and HTTP/1.1-2 (written pre-HTTP/3, but still the best single explanation of *why* each protocol layer behaves the way it does) from a practicing engineer's perspective rather than a pure spec summary.

## Deepen your understanding

- **Wireshark User's Guide & Wiki** (wireshark.org/docs, wiki.wireshark.org) — the official docs for every display filter used in today's lab; the Wiki's protocol pages (e.g. `wiki.wireshark.org/TLS`) are more practical than the RFCs themselves.
- **Cloudflare Learning Center** (cloudflare.com/learning) — short, well-illustrated articles covering exactly this day's topics: "What happens in a TLS handshake," "What is DNS," "HTTP/2 vs HTTP/1.1," "What is QUIC." A good second explanation if a concept didn't click from the READMEs alone.
- **RFC 9293** — the current TCP specification (obsoletes RFC 793); read the connection-state-diagram section for the definitive picture of `TIME_WAIT` and teardown.
- **RFC 8446** — TLS 1.3, for the `ClientHello`/`ServerHello` message formats at the byte level rather than the summarized version in this day's notes.
- **RFC 9114** (HTTP/3) and **RFC 9000** (QUIC transport) — the protocol actually replacing TCP+TLS+HTTP/2's layering.

## Reference / lookup

- `man dig`, `man tcpdump`, `openssl s_client -help` / `openssl x509 -help` — on-box references for every flag used today.
- **SSL Labs — SSL Server Test** (ssllabs.com/ssltest) — point it at any public HTTPS endpoint to get its full negotiated protocol/cipher-suite support and certificate chain graded, without needing your own capture.

## Practice

- **Wireshark Sample Captures** (wiki.wireshark.org/SampleCaptures) — pre-recorded `.pcap` files (including TLS and DNS traffic) to practice filtering and decoding without needing to generate your own traffic first.
- **DNSViz** (dnsviz.net) — visualizes a domain's full DNS delegation chain and DNSSEC status graphically; useful for seeing the root → TLD → authoritative structure as a diagram instead of `dig +trace` text output.
