# Day 13 — Lab: TCP/IP & DNS

**Goal:** Capture a real TLS handshake off the wire, decode it packet-by-packet in Wireshark, and identify exactly which cipher suite was negotiated — while also tracing the full DNS resolution chain that got you there in the first place.

**Prerequisites:**
- `tcpdump` — preinstalled on virtually every Linux distro and macOS; capturing packets requires root/sudo (raw sockets).
- **Wireshark** — install the desktop app (`sudo apt install wireshark` / `brew install --cask wireshark`), or use `tshark` (Wireshark's CLI) if a GUI isn't available.
- `openssl` — preinstalled on Linux and macOS.
- `dig` — part of `bind9-dnsutils`/`dnsutils` on Debian/Ubuntu, `bind-utils` on RHEL/Fedora, `brew install bind` on macOS.
- On macOS, tcpdump doesn't support Linux's `any` pseudo-interface the same way — find your active interface first with `ifconfig` or `networksetup -listallhardwareports` (usually `en0`) and substitute it for `any` below.
- Pick a real domain to target throughout — `example.com` is used below since it's a stable, low-traffic domain meant for documentation/examples.

---

### Lab 1 — Walk the DNS resolution chain

1. Look up the basic records:
   ```bash
   dig example.com A
   dig example.com AAAA
   dig example.com NS
   ```
   In the output, identify the `ANSWER SECTION`, the `Query time`, and which server actually answered (the `SERVER:` line at the bottom) — that's your configured recursive resolver, not the authoritative one.
2. Now watch the entire iterative chain happen for real:
   ```bash
   dig +trace example.com
   ```
   Read the output top to bottom — you'll see the query hit a root server first (a referral to the `.com` TLD servers), then a TLD server (a referral to `example.com`'s authoritative nameservers), then finally the authoritative nameserver returning the real `A` record.
3. Compare against a cached, non-traced lookup: `dig example.com` — note it returns almost instantly and shows only the final answer, because your recursive resolver already did (and cached) all of the work `+trace` just made visible.

**Success criteria:** You can point at a `dig +trace` output and correctly label which lines are root, TLD, and authoritative responses, and explain in one sentence why a plain `dig example.com` is faster.

---

### Lab 2 — Capture a live DNS query and identify the resolver

1. Start a capture filtered to DNS traffic only:
   ```bash
   sudo tcpdump -i en0 -n port 53 -w dns.pcap
   ```
   (replace `en0` with `any` on Linux, or your interface name)
2. In a second terminal, trigger a fresh lookup (use a domain you haven't queried recently so it isn't already cached):
   ```bash
   dig +noall +answer somewebsite-you-havent-looked-up.com
   ```
3. Stop the capture (`Ctrl-C`) and open it: `wireshark dns.pcap` (or `tshark -r dns.pcap -Y dns`).
4. Filter on `dns` in Wireshark. Find the query packet — its **destination IP** is the recursive resolver your machine is configured to use (matches the `nameserver` line in `/etc/resolv.conf`, or `127.0.0.53`/`127.0.0.1` if a local stub resolver like `systemd-resolved` is fronting it). Expand the DNS layer and confirm the `Queries` section shows the name you looked up and record type `A`.

**Success criteria:** You can identify, from the packet capture alone (not from reading a config file), the IP address of the resolver your system actually queries.

---

### Lab 3 — Core hands-on activity: capture a TLS handshake with tcpdump

This is today's assigned hands-on activity — do it for real.

1. Start a capture scoped to HTTPS traffic to one host, so the pcap stays small and readable:
   ```bash
   sudo tcpdump -i en0 -w tls.pcap 'port 443 and host example.com'
   ```
2. In a second terminal, generate a fresh connection (use `curl -v` so you can also see what your client believes happened):
   ```bash
   curl -v -o /dev/null -s https://example.com
   ```
3. Stop the capture (`Ctrl-C`) once `curl` returns. You should see a handful of packets logged by tcpdump — that's the entire TCP handshake, TLS handshake, HTTP request/response, and TCP teardown in one file.
4. Open it in Wireshark: `wireshark tls.pcap`.

**Success criteria:** `tls.pcap` contains at least one full TCP+TLS exchange, and you can see distinct clusters of packets in Wireshark's packet list before applying any filters — a TCP handshake, then larger TLS-looking packets, then a small run of packets at the end (the teardown).

---

### Lab 4 — Decode the handshake in Wireshark and identify the cipher suite

1. In Wireshark, apply the display filter:
   ```
   tls.handshake.type == 1
   ```
   This isolates the `ClientHello`. Expand it in the packet detail pane and find the **Cipher Suites** list — this is every cipher suite *your client* offered, not what got chosen. Also note the `server_name` extension (SNI) — this is how the server, which may host many domains on one IP, knows which certificate to present.
2. Now filter for the response:
   ```
   tls.handshake.type == 2
   ```
   This is the `ServerHello`. Expand it and find the single **Cipher Suite** field — this is the one the server actually picked from your offered list. Also note the negotiated TLS version here (for TLS 1.3, check the `supported_versions` extension rather than the legacy version field, which TLS 1.3 sets to `TLS 1.2` for middlebox-compatibility reasons).
3. Right-click any packet in the exchange and choose **Follow → TLS Stream** to see the whole handshake laid out as one reassembled sequence, in order.
4. Write down the exact cipher suite name shown (e.g. `TLS_AES_128_GCM_SHA256` for TLS 1.3, or `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` for TLS 1.2) and identify which part of the name is the key exchange algorithm, which is the bulk cipher, and which is the hash used for the handshake MAC.

**Success criteria:** You can state the exact negotiated cipher suite from this capture, and explain what each segment of its name means (key exchange / authentication / bulk cipher+mode / MAC).

---

### Lab 5 — Cross-check with openssl, and inspect the certificate chain

1. Reproduce the same connection outside of a browser, with full visibility into the handshake:
   ```bash
   openssl s_client -connect example.com:443 -servername example.com -tls1_3 </dev/null 2>/dev/null | grep -E "Protocol|Cipher"
   ```
   Confirm this matches what you read off the `ServerHello` in Wireshark.
2. Pull the full certificate chain as sent by the server:
   ```bash
   openssl s_client -connect example.com:443 -servername example.com -showcerts </dev/null
   ```
   Count how many certificates come back (leaf + intermediate(s) — the root should NOT be among them).
3. Extract just the leaf certificate's human-readable details:
   ```bash
   openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
     | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
   ```
4. Identify: the `Subject`, the `Issuer` (the CA), the validity window, and the list of names in `subjectAltName` — confirm the hostname you connected to is actually in that SAN list.

**Success criteria:** You can name the issuing CA, state the certificate's expiry date, and confirm the SAN list covers the hostname you connected to, all without opening a browser.

---

### Lab 6 — Correlate the TCP handshake with the TLS handshake in one capture

1. Reopen `tls.pcap` from Lab 3 in Wireshark and filter:
   ```
   tcp.flags.syn == 1
   ```
   You should see exactly two packets: the `SYN` (client → server) and the `SYN-ACK` (server → client) — the final plain `ACK` doesn't have the SYN flag set so it won't show here.
2. Clear the filter and look at the packet immediately after that plain ACK — it should be the `ClientHello`, confirming TLS only starts riding on top of an *already-established* TCP connection.
3. Filter `tcp.flags.fin == 1` and confirm you see the FIN packets closing the connection at the end, after the HTTP response.
4. In one sentence, explain why the TLS handshake could never start before the TCP handshake completes.

**Success criteria:** You can point to the exact packet number where the TCP handshake ends and the TLS handshake begins, in the same capture file.

---

### Cleanup

```bash
rm -f dns.pcap tls.pcap
```

---

### Stretch challenge

Force a TLS 1.2-only connection and compare it against the TLS 1.3 capture from Lab 5:
```bash
openssl s_client -connect example.com:443 -tls1_2 -state -msg </dev/null 2>&1 | grep -E "SSL_connect|Protocol|Cipher"
```
Count the distinct handshake messages logged (`-msg`) for TLS 1.2 versus TLS 1.3, and confirm TLS 1.2 needs an extra round trip before `Finished` compared to TLS 1.3's single round trip covered in `03-README-TLS-And-HTTP-Evolution.md`. (Alternative stretch: repeat Lab 2's DNS capture, but this time run `dig +trace` while capturing on port 53 — you'll see the tool talk directly to root/TLD/authoritative servers, bypassing your usual recursive resolver, which visibly changes which IPs show up as destinations in the capture.)
