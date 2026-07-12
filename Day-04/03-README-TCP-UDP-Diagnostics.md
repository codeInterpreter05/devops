# Day 4 — Networking on Linux: TCP vs UDP & Diagnosing Connections

**Phase:** 0 – Foundation | **Week:** W1 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

This is where addressing (file 1) and naming (file 2) come together into an actual conversation between two machines. Understanding the TCP three-way handshake, how UDP differs, and how to watch these things happen live with `ping`, `traceroute`, `curl -v`, `netcat`, and `tcpdump` is the single most interview-tested networking skill in DevOps — "walk me through what happens when you type `curl https://example.com`" is a real, extremely common interview question precisely because it forces you to demonstrate this whole stack at once. Today's core hands-on lab (see `LAB.md`) has you capture and read this exchange for real.

## TCP vs UDP: two fundamentally different contracts

**TCP (Transmission Control Protocol)** is **connection-oriented** and **reliable**: before any application data flows, both sides perform a handshake to agree the connection exists, then TCP guarantees (via sequence numbers, acknowledgments, and retransmission) that bytes arrive **in order** and **without loss**, or the connection errors out. This costs round trips and overhead — you pay for reliability with latency.

**UDP (User Datagram Protocol)** is **connectionless**: a sender just fires a datagram at a destination IP:port with no handshake, no ordering guarantee, and no delivery guarantee. If a packet is lost, UDP does nothing about it — that's left to the application (or simply accepted as loss). This is not "TCP but worse" — it's a deliberate trade: lower latency, no head-of-line blocking, ideal for DNS queries (small, easily retried), video/voice streaming (a late packet is worse than a lost one), and any protocol that implements its own reliability layer on top (like QUIC/HTTP-3, which runs over UDP but rebuilds reliability itself, avoiding some of TCP's specific bottlenecks).

| | TCP | UDP |
|---|---|---|
| Connection setup | 3-way handshake required | None — just send |
| Reliability | Guaranteed delivery + ordering | None — best effort |
| Overhead | Higher (headers, ACKs, retransmits) | Minimal |
| Use cases | HTTP/HTTPS, SSH, databases | DNS queries, streaming, VoIP, QUIC/HTTP-3 |
| Head-of-line blocking | Yes (one lost segment blocks later ones until retransmitted) | No |

## The TCP three-way handshake, mechanically

Before any HTTP request can be sent, the client and server must establish a TCP connection:

1. **SYN** — client sends a segment with the `SYN` flag set and an initial sequence number, essentially saying "I want to open a connection, here's where my byte numbering starts."
2. **SYN-ACK** — server responds with `SYN` and `ACK` set: it acknowledges the client's sequence number (`ACK`) and sends its own initial sequence number (`SYN`) — this is why it's one segment doing two jobs.
3. **ACK** — client acknowledges the server's sequence number. The connection is now `ESTABLISHED` on both sides, and only *now* does the client send its actual HTTP request.

This is why every TCP-based request pays at least one full round-trip of latency **before** any application data moves — and for HTTPS, the TLS handshake happens *after* this, adding more round trips still (TLS 1.2 typically needs 2 more round trips, TLS 1.3 needs 1, or 0 with session resumption). This stacking of round trips is exactly why reducing RTT (e.g., serving from a nearby CDN edge) matters so much for perceived web performance, independent of how fast the origin server generates the response.

Closing a TCP connection is asymmetric and worth knowing: it's a **4-way** teardown (`FIN`, `ACK`, `FIN`, `ACK`) because each side must independently signal it's done sending — a connection can be "half-closed" with one side still sending data after the other has finished. This is also the origin of the `TIME_WAIT` state you'll see in `ss -tan` output on busy servers — the side that initiates the close holds the connection in `TIME_WAIT` for a period to guarantee any delayed duplicate packets are handled correctly.

## `ping` — is there a live host, and how far away is it (RTT)?

```bash
ping example.com          # continuous, Ctrl+C to stop
ping -c 4 example.com      # send exactly 4 and stop
ping -i 0.2 example.com    # interval between packets (root may be required for very low intervals)
```

`ping` sends **ICMP Echo Request** packets and waits for **ICMP Echo Reply**. It tells you: is the host reachable, and what's the round-trip time — but it says nothing about whether the *service* you actually care about (port 443, a specific app) is up, because ICMP is a completely different protocol from TCP/UDP. Many hosts and cloud security groups deliberately block ICMP for security-through-obscurity reasons, so "ping fails" does **not** reliably mean "host is down" — a working `curl` to the same host despite failed pings is normal and expected in cloud environments, not a contradiction.

## `traceroute` — mapping the path, hop by hop, via TTL expiry

```bash
traceroute example.com
traceroute -n example.com     # numeric only, skip reverse-DNS per hop (much faster)
traceroute -I example.com     # use ICMP Echo instead of UDP probes (traceroute's default varies by OS)
```

`traceroute` exploits the IP **TTL (Time To Live)** field: every router that forwards a packet decrements its TTL by 1, and if TTL hits 0, the router drops the packet and sends back an **ICMP Time Exceeded** message to the original sender — identifying itself in the process. `traceroute` sends a first probe with TTL=1 (dies at the first hop, which replies), then TTL=2 (dies at the second hop), and so on, incrementing until a probe finally reaches the destination. Each "hop" line in the output is one router's reply to a probe that expired exactly at its doorstep. This is also why traceroute output can show `* * *` for a hop — that router is configured to silently drop expired packets rather than reply, or is rate-limiting/blocking the ICMP responses, without the destination itself being unreachable.

## `curl -v` — watching an HTTP(S) request happen, step by step

```bash
curl -v https://example.com
```

Reading `curl -v` output in order tells the whole story from file 1+2's addressing/DNS through this file's TCP/TLS layer to the actual HTTP exchange:

```
*   Trying 93.184.216.34:443...              <- DNS already resolved; now TCP connect attempt
* Connected to example.com (93.184.216.34) port 443   <- TCP three-way handshake completed
* ALPN: curl offers h2,http/1.1
* TLS handshake, ...                          <- TLS negotiation (cert exchange, cipher agreement)
* SSL connection using TLSv1.3 / ...
* Server certificate: ...                      <- certificate chain validated
> GET / HTTP/1.1                               <- '>' lines: request curl SENT
> Host: example.com
> User-Agent: curl/8.x
> Accept: */*
>
< HTTP/1.1 200 OK                               <- '<' lines: response curl RECEIVED
< Content-Type: text/html; charset=UTF-8
< Content-Length: 1256
<
<!doctype html>...                              <- response body
```

`>` lines are literally the bytes curl sent; `<` lines are what came back. `*` lines are curl's own diagnostic narration (DNS, connect, TLS). This is the exact sequence an interviewer wants when they ask "what happens when you `curl https://example.com`": DNS resolution → TCP handshake → TLS handshake (if HTTPS) → HTTP request line + headers sent → HTTP response status + headers + body received → connection closed or kept alive.

## `netcat` (`nc`) — a raw socket Swiss-army knife

```bash
nc -zv example.com 443          # -z: scan/zero-I/O mode (just test if port is open), -v: verbose
nc -l -p 8080                    # listen on a port (great for a throwaway test endpoint)
echo -e "GET / HTTP/1.0\r\n\r\n" | nc example.com 80   # manually speak raw HTTP
nc -u -zv example.com 53          # -u: UDP mode — note UDP "open" checks are unreliable (no handshake to confirm)
```

`nc -z` is the fast way to answer "is this port even reachable" without invoking a full application protocol — extremely useful for checking security-group/firewall issues in isolation from application-level errors (a `nc -zv` failure means the problem is network/firewall layer; success but `curl` still failing means the problem is higher up the stack, in TLS or the application itself).

## Points to Remember

- TCP guarantees ordered, reliable delivery via a 3-way handshake (SYN, SYN-ACK, ACK) before any data moves; UDP has no handshake and no delivery guarantee — a deliberate latency/reliability trade-off, not a lesser protocol.
- Closing TCP is a 4-way process (independent `FIN` from each side) — this asymmetry is why `TIME_WAIT` sockets exist and pile up on busy servers.
- `ping` uses ICMP, not TCP/UDP — a blocked/failed ping does not mean the actual service (e.g., HTTPS) is down; many production hosts intentionally block ICMP.
- `traceroute` works by deliberately sending packets with increasing TTL so each hop's router "expires" one and replies with ICMP Time Exceeded — `* * *` often means a router is dropping/rate-limiting ICMP, not that the path is broken.
- `curl -v`'s `>` = sent, `<` = received, `*` = curl's own commentary — the full sequence (DNS → TCP → TLS → HTTP) is the canonical answer to "what happens when you curl a URL."
- `nc -zv` isolates network/port reachability from application-layer failures — a cheap first diagnostic step before blaming the app.

## Common Mistakes

- Concluding a host or service is "down" purely because `ping` fails, without testing the actual port/protocol in question (`nc -zv`, `curl`) — ICMP being blocked is normal and unrelated to TCP service health.
- Treating UDP as strictly worse than TCP instead of understanding it's a deliberate design choice for latency-sensitive or self-reliable protocols (DNS, streaming, QUIC).
- Forgetting that TLS handshake round trips are *in addition to* the TCP handshake — assuming "HTTPS is just HTTP with encryption" and being surprised by the extra latency on cold connections without keep-alive/session resumption.
- Reading `traceroute`'s `* * *` hops as proof the connection is broken at that exact point, rather than considering that intermediate routers commonly suppress ICMP replies while still forwarding traffic fine.
- Not distinguishing curl's `*` diagnostic lines from actual wire traffic (`>`/`<`) — quoting a `*` line as "what the server sent" in a debugging writeup.
- Using plain `nc` UDP scans (`nc -u -zv`) as a reliable "is this UDP port open" test — without a handshake, a lack of response is ambiguous (could be open-but-silent, or genuinely filtered).
