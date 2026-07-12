# Day 4 — Cheatsheet: Networking on Linux

## `ip` — interfaces, addresses, routes

```bash
ip addr show               # or: ip a — all interfaces + addresses
ip addr show eth0           # one interface
ip -4 addr                 # IPv4 only
ip -6 addr                 # IPv6 only
ip link show                # link state (UP/DOWN, LOWER_UP), MAC, MTU — no IP info
ip addr add 10.0.0.5/24 dev eth0     # add an address (root)
ip addr del 10.0.0.5/24 dev eth0     # remove an address (root)
ip route show                # routing table
ip route get 8.8.8.8          # which route/interface would be used to reach a destination
ip neigh show                 # ARP/neighbor table (MAC <-> IP on local segment)
```

## `ifconfig` / `netstat` (legacy — read-only knowledge)

```bash
ifconfig                    # like `ip addr`, net-tools (often not installed by default now)
ifconfig eth0                # one interface
netstat -tulpn               # like `ss -tulpn`, net-tools — slower, parses /proc
```

## `ss` — socket statistics (modern `netstat` replacement)

```bash
ss -tulwn                  # TCP+UDP, listening only, numeric, wide
ss -tulpn                   # + owning process (needs sudo for others' procs)
ss -tan                      # all TCP sockets, numeric
ss -uan                      # all UDP sockets, numeric
ss -s                         # summary counts by protocol/state
ss -o state established '( dport = :443 or sport = :443 )'   # filter by state + port
```
Flags: `-t` TCP, `-u` UDP, `-l` listening only, `-n` numeric (skip DNS), `-p` process, `-a` all, `-o` show timers.

## `ping` — reachability + round-trip time (ICMP)

```bash
ping example.com            # continuous, Ctrl+C to stop
ping -c 4 example.com        # send 4 and stop
ping -i 0.2 example.com       # interval in seconds
ping -s 1000 example.com      # packet size (test MTU/fragmentation issues)
```
Note: ICMP is not TCP/UDP — a blocked ping does not mean the actual service is down.

## `traceroute` — hop-by-hop path via TTL expiry

```bash
traceroute example.com
traceroute -n example.com      # numeric, skip reverse-DNS per hop (faster)
traceroute -I example.com       # use ICMP Echo probes
traceroute -T -p 443 example.com  # use TCP SYN probes to a specific port
```
`* * *` at a hop = that router isn't replying (dropped/rate-limited ICMP) — not necessarily a broken path.

## `dig` — DNS queries (modern, scriptable)

```bash
dig example.com                  # full answer/authority/additional sections
dig example.com +short           # just the answer(s) — scripting-friendly
dig example.com A                # A record (IPv4)
dig example.com AAAA             # AAAA record (IPv6)
dig example.com MX                # mail exchangers
dig example.com NS                 # authoritative nameservers
dig example.com TXT                 # text records
dig @8.8.8.8 example.com             # query a specific resolver directly
dig +trace example.com                # walk the delegation chain from root down
dig -x 93.184.216.34                   # reverse lookup (PTR)
```

## `nslookup` — DNS queries (legacy, interactive-oriented)

```bash
nslookup example.com
nslookup example.com 8.8.8.8       # query a specific server
nslookup -type=MX example.com       # explicit record type
nslookup -type=NS example.com
```

## Name resolution config files

```bash
cat /etc/nsswitch.conf | grep hosts   # resolution order, e.g. "hosts: files dns"
cat /etc/hosts                         # static name -> IP overrides, checked first
cat /etc/resolv.conf                    # nameserver + search domain config
getent hosts example.com                 # resolve exactly as the system would (respects nsswitch order)
resolvectl status                          # systemd-resolved's actual view (if in use)
```

## `curl` — HTTP client

```bash
curl https://example.com               # GET, print body
curl -v https://example.com             # verbose: DNS, TCP connect, TLS, request/response headers
curl -I https://example.com              # HEAD only — headers, no body
curl -o out.html https://example.com      # save to file
curl -L https://example.com                # follow redirects
curl -X POST -d '{"a":1}' -H 'Content-Type: application/json' https://api.example.com
curl -w '%{http_code} %{time_total}\n' -o /dev/null -s https://example.com   # status + timing only
curl --resolve example.com:443:1.2.3.4 https://example.com   # force an IP, bypass DNS
```

## `wget` — HTTP/FTP downloader

```bash
wget https://example.com/file.tar.gz        # download a file
wget -O out.tar.gz https://example.com/file  # save with a specific name
wget -c https://example.com/file.tar.gz       # resume a partial download
wget --mirror https://example.com/docs/        # recursive mirror of a site/path
```

## `netcat` (`nc`) — raw sockets

```bash
nc -zv example.com 443           # test if a TCP port is open (-z: no data, -v: verbose)
nc -u -zv example.com 53           # UDP port test (unreliable — no handshake to confirm "open")
nc -l -p 8080                        # listen on a port
nc example.com 80                     # connect and speak raw text manually
printf 'GET / HTTP/1.0\r\nHost: x\r\n\r\n' | nc example.com 80   # manual raw HTTP request
```

## `tcpdump` — packet capture

```bash
sudo tcpdump -i any -n                        # capture all interfaces, numeric (no DNS lookups)
sudo tcpdump -i eth0 port 80                    # capture only port 80 traffic on eth0
sudo tcpdump -i any 'port 53 or port 443'          # DNS + HTTPS traffic
sudo tcpdump -i any -w capture.pcap                 # write raw capture to a file
sudo tcpdump -r capture.pcap -n                      # read a capture file back
sudo tcpdump -r capture.pcap -A 'port 80'              # show ASCII payload (readable for plaintext HTTP)
sudo tcpdump -i any -s 0 host example.com                # full packets, filter by host
```
Reading TCP flags in output: `[S]` = SYN, `[S.]` = SYN-ACK, `[.]` = ACK (or plain data ack), `[P.]` = PSH-ACK (data push), `[F.]` = FIN-ACK, `[R]` = RST (reset/refused).
