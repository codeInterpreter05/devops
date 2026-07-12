# Day 13 — TCP/IP & DNS: TLS Handshake & HTTP Evolution

**Phase:** 0 – Foundation | **Week:** W2 | **Domain:** Networking | **Flag:** ⚡ Interview-critical

## Brief

By the time your `curl https://api.example.com` returns a response, three separate handshakes may have already happened: DNS, TCP, and TLS — and then, on top of the established TLS connection, an HTTP version negotiation shapes how efficiently the actual request/response gets carried. This note covers the TLS handshake in enough depth to actually read a Wireshark capture (today's lab), what's really inside the certificate your browser trusts, and why HTTP went from 1.1 to 2 to 3 — each version solving a specific, nameable problem the previous one had.

## The TLS handshake

**TLS 1.2 (2 round trips before application data):**
```
Client                                          Server
  | -- ClientHello ------------------------> |
  |    (TLS versions offered, cipher suites,   |
  |     client random, SNI, ALPN)              |
  |                                             |
  | <- ServerHello ----------------------------- |
  |    (chosen version, chosen cipher suite,    |
  |     server random)                          |
  | <- Certificate ------------------------------ |
  | <- ServerKeyExchange (ephemeral DH params,  |
  |     signed by the cert's private key) ------ |
  | <- ServerHelloDone --------------------------- |
  |                                             |
  | -- ClientKeyExchange ------------------> |
  |    (client's ephemeral DH public value)     |
  | -- [ChangeCipherSpec] ------------------> |
  | -- Finished (encrypted, MAC over the       |
  |     whole transcript so far) -----------> |
  |                                             |
  | <- [ChangeCipherSpec] ------------------------ |
  | <- Finished ----------------------------------- |
  |                                             |
  |     <-- application data now flows -->      |
```
Both sides independently compute the same shared secret from the (EC)DHE exchange, derive session keys from it, and the `Finished` messages are each side's proof — encrypted with the freshly derived key, and containing a MAC of the entire handshake so far — that nothing was tampered with in transit.

**TLS 1.3 (1 round trip, or 0 on session resumption):** TLS 1.3 collapses this by having the client *guess* which key-exchange group the server will pick and send its ephemeral public key speculatively in the `ClientHello` itself (a `key_share` extension). The server replies with `ServerHello` + its own key share — at which point both sides can already derive the handshake keys — and everything after that (`EncryptedExtensions`, `Certificate`, `CertificateVerify`, `Finished`) is sent encrypted in the *same* flight. The client's `Finished` completes the handshake in a single round trip. TLS 1.3 also removed a pile of legacy weak/insecure options entirely: static RSA key exchange, CBC-mode ciphers implicated in BEAST/Lucky13-style attacks, RC4, compression (implicated in CRIME), and renegotiation — there's no "downgrade to a weak cipher" path because those ciphers no longer exist in the protocol. (One artifact worth knowing: TLS 1.3's `ServerHello` still sets the legacy version field to `TLS 1.2` for middlebox compatibility — the real negotiated version is in the `supported_versions` extension.)

Watch the actual negotiated version and cipher suite:
```bash
openssl s_client -connect example.com:443 -tls1_3 </dev/null 2>/dev/null | grep -E "Protocol|Cipher"
```

### Ephemeral Diffie-Hellman and forward secrecy

Modern TLS mandates **ephemeral** (EC)DHE key exchange — a fresh, random private key generated *just for this one handshake* and discarded the moment the connection ends. The actual session key is derived from that ephemeral exchange, not directly from the server's long-term certificate key. This buys **forward secrecy**: if an attacker records today's encrypted traffic and, a year from now, steals the server's long-term private key (the one tied to its certificate), they still cannot decrypt that recorded session — the ephemeral key needed to derive its specific session key was thrown away right after the handshake and never existed anywhere else. Contrast this with old-style static RSA key exchange (TLS 1.2 and earlier permitted it): the premaster secret was encrypted directly with the server's long-term RSA public key, so compromising that one key later makes *every* past recorded session under that certificate retroactively decryptable. TLS 1.3 makes ephemeral exchange mandatory — there's no cipher suite offering static RSA key exchange left in the protocol at all.

## What's inside an X.509 certificate

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -text
```
The fields that matter operationally:

- **Subject** — who the cert was issued to (Common Name / Organization). Modern browsers/clients **do not trust the CN for hostname matching** — they've required `subjectAltName` since ~2017.
- **Issuer** — which CA signed this certificate.
- **Subject Public Key Info** — the public key and algorithm (RSA-2048, EC P-256, etc.) — this is the key TLS actually authenticates the handshake with, not the key used to derive the session (that's the ephemeral DH key above).
- **Validity** (`Not Before` / `Not After`) — the window the cert is valid in; an expired cert fails validation regardless of how correct everything else is.
- **X509v3 Subject Alternative Name (SAN)** — the actual list of DNS names (and/or IPs) the certificate is valid for. This is what gets checked against the hostname you connected to.
- **Basic Constraints** (`CA:TRUE`/`CA:FALSE`) — whether this certificate is itself allowed to sign other certificates (a CA cert) or is a leaf/end-entity cert.
- **Authority/Subject Key Identifier** — used to link a cert to the specific issuer cert that signed it, for chain building.
- **CRL Distribution Points / OCSP responder** — where a client can check whether this certificate has been revoked before its expiry.

**Chain of trust:** the server's own (leaf) certificate is signed by an intermediate CA's private key; that intermediate's certificate is in turn signed by a root CA. Root CA certificates are **self-signed** and pre-installed in the OS/browser trust store — they're never sent over the wire, because the whole point is the client already trusts them locally. A correctly configured server sends the leaf certificate **plus all intermediates** (never the root) so the client can walk the chain up to a root it already has:
```bash
openssl s_client -connect example.com:443 -showcerts </dev/null
```
A server that omits an intermediate produces the classic "works in Chrome (which cached/fetched the missing intermediate elsewhere or has AIA fetching), fails in `curl`/mobile apps/older clients" bug — verify a full, correctly-ordered chain with `-verify_return_error`.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

**HTTP/1.1** is plain text, request/response, one at a time per connection by default. `Connection: keep-alive` (persistent connections) fixed HTTP/1.0's "new TCP connection per request" cost, but a connection can still only have one request in flight without a response before the next can start on it — **head-of-line (HOL) blocking at the application layer**. Browsers work around this by opening roughly six parallel TCP (and TLS) connections per origin — which means six handshakes, six slow-start ramp-ups, and six times the server-side connection bookkeeping, just to get some parallelism.

**HTTP/2** introduces a binary framing layer and **multiplexes many logical streams over a single TCP connection** — each frame is tagged with a stream ID and reassembled at the other end, so requests and responses interleave freely without one blocking another *at the application layer*. It also adds **HPACK** header compression: a static table of common header name/value pairs plus a per-connection dynamic table built as headers are actually sent, which matters enormously because HTTP headers (cookies, user-agent, auth tokens) are extremely repetitive across every request to the same origin. The catch: it's still **one TCP connection**. If a single packet is lost, TCP's strict in-order delivery guarantee blocks *every* multiplexed stream's frames behind that one lost packet until it's retransmitted and arrives — **transport-level head-of-line blocking**, and HTTP/2 cannot fix this because the problem lives below it, in TCP itself.

**HTTP/3** solves exactly that remaining problem by dropping TCP entirely and running over **QUIC**, a transport built on top of UDP. QUIC reimplements reliability and congestion control itself, but per-stream: a lost packet only stalls the one stream whose data was inside it, and frames for other streams that already arrived can be delivered to the application immediately — no transport-level HOL blocking. QUIC also folds the TLS 1.3 handshake into its own transport handshake, so establishing a secure connection isn't "TCP handshake, then a separate TLS handshake on top" — it's combined into effectively one round trip (or zero, on resumption). QUIC connections are identified by a **Connection ID** instead of the traditional 4-tuple (src/dst IP:port), which is what enables **connection migration**: a mobile client switching from Wi-Fi to cellular gets a new IP, but the QUIC connection ID stays the same, so the connection survives the network change — a TCP connection tied to the old 4-tuple would simply break.

## Points to Remember

- TLS 1.2 needs 2 round trips before application data flows; TLS 1.3 needs 1 (or 0 with resumption) by having the client speculatively send a key share in its very first message.
- Forward secrecy comes from *ephemeral* (EC)DHE — a session key derived from a throwaway key, not the certificate's long-term key — so stealing the server's private key later can't decrypt past traffic. TLS 1.3 makes this mandatory.
- Browsers check the certificate's `subjectAltName`, not the Common Name, for hostname matching — CN-only certs fail in modern clients.
- The chain of trust is leaf → intermediate(s) → root; the server must send the leaf + intermediates (never the root), or clients without a cached/fetched intermediate will fail to verify.
- HTTP/1.1's HOL blocking is at the application layer (one request in flight per connection); browsers work around it with ~6 parallel connections.
- HTTP/2 fixes application-layer HOL blocking via multiplexed streams over one TCP connection, but is still exposed to transport-layer HOL blocking from packet loss.
- HTTP/3 (QUIC/UDP) is the only one of the three that solves transport-layer HOL blocking, because it does per-stream reliability instead of one ordered TCP byte stream for everything.

## Common Mistakes

- Assuming TLS 1.3's speed improvement is about weaker crypto — it's the opposite: it's faster *and* removes weak ciphers/static RSA key exchange, it's not a trade-off between speed and security.
- Believing a valid `Common Name` alone makes a certificate trusted for a hostname — modern clients ignore CN and check SAN only.
- Sending only the leaf certificate from a server (omitting intermediates) and not noticing because your own browser already has the intermediate cached — then being surprised when `curl` or a mobile app fails to verify the chain.
- Thinking HTTP/2's multiplexing eliminates all head-of-line blocking — it only fixes the application-layer version; a single dropped packet still stalls every stream until TCP retransmits it, which is precisely the problem HTTP/3 exists to solve.
- Assuming HTTP/3/QUIC is "just HTTP/2 over UDP" — the reliability and congestion-control logic is fundamentally reimplemented inside QUIC itself, since UDP provides none of that natively.
