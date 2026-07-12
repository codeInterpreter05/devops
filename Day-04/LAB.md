# Day 4 — Lab: Networking on Linux

**Goal:** Go from reading about DNS/TCP/HTTP to actually watching them happen on the wire — capture a real HTTP request with `tcpdump` and correlate every packet against `curl -v`'s own account of the same request.

**Prerequisites:** A Linux environment — Ubuntu via WSL2, a VM, or a cloud instance (a container without `NET_ADMIN`/raw-socket capability will not be able to run `tcpdump`; a VM or bare WSL2 instance is safer for this lab). Install the tools:

```bash
# Debian/Ubuntu
sudo apt update
sudo apt install -y iproute2 dnsutils curl wget netcat-openbsd tcpdump

# RHEL/Fedora/Amazon Linux
sudo dnf install -y iproute bind-utils curl wget nmap-ncat tcpdump
```

Confirm everything is present:
```bash
ip -V && ss -V && dig -v && curl -V && wget -V && nc -h 2>&1 | head -1 && tcpdump --version
```

---

### Lab 1 — Inspect your interfaces and sockets

1. List every interface and its addresses:
   ```bash
   ip addr show
   ```
   Identify your primary interface (not `lo`), note its `inet` address and CIDR prefix, and check whether it shows `UP` and `LOWER_UP`.
2. Check the routing table and confirm which interface/gateway would be used to reach the public internet:
   ```bash
   ip route show
   ip route get 8.8.8.8
   ```
3. List every listening socket on the box, with the owning process:
   ```bash
   sudo ss -tulpn
   ```
   Find `sshd` in the list (if present) and note whether it's bound to `0.0.0.0` (all interfaces) or a specific address.
4. Start a throwaway listener and observe it appear in `ss`:
   ```bash
   nc -l -p 8080 &
   ss -tulpn | grep 8080
   kill %1
   ```

**Success criteria:** You can state your box's IP/CIDR, its default route, and identify at least one listening service and whether it's globally or locally bound — all without looking anything up.

---

### Lab 2 — DNS lookups from the command line

1. Resolve a real domain multiple ways:
   ```bash
   dig example.com
   dig example.com +short
   dig example.com A
   dig example.com AAAA
   dig example.com NS
   ```
2. Query a specific public resolver directly, bypassing your system's configured DNS:
   ```bash
   dig @1.1.1.1 example.com
   ```
3. Compare with `nslookup`:
   ```bash
   nslookup example.com
   ```
4. Inspect your resolution configuration:
   ```bash
   cat /etc/nsswitch.conf | grep hosts
   cat /etc/resolv.conf
   cat /etc/hosts
   ```
5. Prove `/etc/hosts` overrides DNS: add a fake mapping, then resolve it.
   ```bash
   echo "127.0.0.1 fake-lab-host.test" | sudo tee -a /etc/hosts
   ping -c 1 fake-lab-host.test
   getent hosts fake-lab-host.test
   ```
   (`getent hosts` shows what the resolver chain actually returns, respecting `nsswitch.conf` order — unlike `dig`, which always queries DNS directly and ignores `/etc/hosts`.)

**Success criteria:** You can explain, using your own `/etc/resolv.conf` and `/etc/nsswitch.conf`, exactly which file is consulted first and why `dig` and `getent hosts` can legitimately return different answers for the same name.

---

### Lab 3 — The core hands-on activity: trace a full HTTP request with tcpdump + curl -v

This is today's assigned hands-on activity — DNS → TCP handshake → response, captured and correlated for real.

1. Open **two terminals** (or two panes/`tmux` splits) on the same machine.
2. In **terminal A**, start a packet capture on your primary interface, filtered to DNS and port 80/443 traffic, writing to a file so you can inspect it afterward:
   ```bash
   sudo tcpdump -i any -n -s 0 -w /tmp/day4-capture.pcap 'port 53 or port 80 or port 443'
   ```
   Leave it running.
3. In **terminal B**, run a verbose HTTP request against a plain (non-HTTPS) test endpoint first, so the exchange is fully readable without TLS in the way:
   ```bash
   curl -v http://neverssl.com
   ```
   Let it finish, then also run the HTTPS version to see the extra TLS round trip:
   ```bash
   curl -v https://example.com
   ```
4. Back in **terminal A**, stop the capture with `Ctrl+C`, then read it back in human-readable form:
   ```bash
   sudo tcpdump -r /tmp/day4-capture.pcap -n
   ```
5. In the `tcpdump` output, identify and label each of these in order:
   - The **DNS query** (`port 53`, a UDP packet asking for `neverssl.com`/`example.com`) and its **response** (the answer, matching what `dig` showed you in Lab 2).
   - The **TCP three-way handshake**: a packet with flag `[S]` (SYN) from your IP to the server, a `[S.]` (SYN-ACK) back, and a final `[.]` (ACK) from your IP — all on the same source/destination port pair, before any HTTP appears.
   - The **HTTP request** itself: run `sudo tcpdump -r /tmp/day4-capture.pcap -A 'port 80'` and find the literal `GET / HTTP/1.1` text and headers in plaintext (this only works for the non-HTTPS request — HTTPS payload will appear as encrypted noise, which is itself worth observing).
   - The connection teardown: `[F.]` (FIN-ACK) packets from each side.
6. Line up the `curl -v` output from terminal B against the `tcpdump` timeline from terminal A: curl's `* Trying <ip>:80...` corresponds to the SYN packet; `* Connected to ...` corresponds to the completed 3-way handshake; the `>` lines correspond to the plaintext `GET` request seen in `tcpdump -A`; the `<` lines correspond to the response packets.

**Success criteria:** You can point to a specific packet in your `tcpdump` capture for each of: the DNS query/response, the SYN, the SYN-ACK, the ACK, the HTTP request bytes, and the FIN teardown — and explain out loud how each one maps to a line in your `curl -v` output.

---

### Lab 4 — `traceroute` and `netcat` in practice

1. Map the path to a public host and note how many hops it takes:
   ```bash
   traceroute -n example.com
   ```
   If any hops show `* * *`, don't assume the path is broken — cross-check that `curl`/`ping` to the final destination still works.
2. Test raw port reachability without invoking HTTP at all:
   ```bash
   nc -zv example.com 443
   nc -zv example.com 25      # likely blocked/filtered — compare the failure mode to the success above
   ```
3. Manually speak raw HTTP over a plain socket, without curl, to see there's nothing magic about it:
   ```bash
   printf 'GET / HTTP/1.0\r\nHost: neverssl.com\r\n\r\n' | nc neverssl.com 80
   ```

**Success criteria:** You can explain the difference between a `traceroute` hop that fails to reply (`* * *`) and a destination that is genuinely unreachable, and you've sent a raw HTTP request by hand with `nc` and recognized the same response structure `curl -v` showed you in Lab 3.

---

### Cleanup

```bash
sudo sed -i '/fake-lab-host.test/d' /etc/hosts   # remove the test /etc/hosts entry from Lab 2
rm -f /tmp/day4-capture.pcap
jobs -l                                           # confirm no leftover `nc -l` listeners from Lab 1
```

### Stretch challenge

Re-run Lab 3's capture against an HTTPS-only site, then use `tcpdump -r <file> -n` (without `-A`) to find the **TLS ClientHello** packet immediately after the TCP handshake, before any HTTP appears — and explain in one or two sentences why the HTTP request/response text is unreadable in this capture even though the TCP handshake looks identical to the plaintext case.
