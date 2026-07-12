# Day 13 — Cheatsheet: TCP/IP & DNS

## tcpdump filter syntax

```
sudo tcpdump -i <iface>               # capture on an interface (any = all interfaces, Linux only)
sudo tcpdump -i any -n                # -n = don't resolve hostnames/ports (faster, less noisy)
sudo tcpdump -i any -w file.pcap       # write raw packets to a file for Wireshark
sudo tcpdump -r file.pcap              # read back a previously captured file
sudo tcpdump -i any -c 20              # stop after 20 packets

# common filter expressions (combine with 'and' / 'or' / 'not')
host 93.184.216.34                     # traffic to/from a specific IP
src host 10.0.0.5                      # source IP only
dst host 10.0.0.5                      # destination IP only
port 443                               # traffic to/from a specific port (either direction)
src port 443                           # source port only
tcp                                    # TCP traffic only
udp                                    # UDP traffic only
port 53                                # DNS (UDP normally, TCP for large/zone-transfer responses)
'port 443 and host example.com'
'tcp[tcpflags] & tcp-syn != 0'              # SYN packets (handshake starts)
'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'    # SYN or SYN-ACK
'tcp[tcpflags] & tcp-fin != 0'              # FIN packets (graceful close)
'tcp[tcpflags] & tcp-rst != 0'              # RST packets (abrupt close)
```

## Wireshark display filters

```
tcp.port == 443                     # TCP traffic on port 443
tls                                  # any TLS record
tls.handshake.type == 1             # ClientHello
tls.handshake.type == 2             # ServerHello
tls.handshake.type == 11            # Certificate
tls.handshake.ciphersuite           # negotiated/offered cipher suite field
tcp.flags.syn == 1                  # SYN or SYN-ACK packets
tcp.flags.fin == 1                  # FIN packets
dns                                  # any DNS traffic
dns.qry.name == "example.com"        # DNS query for a specific name
http2                                # HTTP/2 traffic
quic                                 # HTTP/3 (QUIC) traffic
```
Right-click any packet → **Follow → TCP Stream** / **TLS Stream** to see a whole exchange reassembled in order.

## openssl — s_client (connect to and inspect a live TLS server)

```bash
openssl s_client -connect host:443                          # connect, dump handshake + cert
openssl s_client -connect host:443 -servername host          # send SNI (needed for multi-tenant hosts)
openssl s_client -connect host:443 -tls1_2                    # force TLS 1.2
openssl s_client -connect host:443 -tls1_3                    # force TLS 1.3
openssl s_client -connect host:443 -showcerts                 # print the FULL chain sent by the server
openssl s_client -connect host:443 -state -msg                # log every handshake state/message (RTT counting)
openssl s_client -connect host:443 </dev/null 2>/dev/null      # non-interactive (don't hang waiting on stdin)
```

## openssl — x509 (inspect a certificate)

```bash
openssl x509 -in cert.pem -noout -text                # full human-readable dump
openssl x509 -in cert.pem -noout -subject               # who it was issued to
openssl x509 -in cert.pem -noout -issuer                # who signed it
openssl x509 -in cert.pem -noout -dates                 # notBefore / notAfter
openssl x509 -in cert.pem -noout -ext subjectAltName     # the SAN list (hostnames it's actually valid for)
openssl x509 -in cert.pem -noout -fingerprint -sha256    # cert fingerprint

# pull a server's live leaf cert straight into these:
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -text
```

## dig — queries and record types

```bash
dig example.com                 # default: A record
dig example.com A                # explicit A
dig example.com AAAA             # IPv6
dig example.com MX               # mail exchange
dig example.com NS               # authoritative nameservers
dig example.com TXT              # text records (SPF/DKIM/verification)
dig example.com SOA              # zone metadata (serial, TTLs)
dig -x 93.184.216.34             # reverse lookup (PTR)
dig +trace example.com           # full iterative walk: root -> TLD -> authoritative
dig +short example.com           # just the answer, no metadata
dig +noall +answer example.com   # only the ANSWER SECTION
dig @8.8.8.8 example.com         # query a specific resolver directly
dig example.com +tcp             # force the query over TCP instead of UDP
```

## TCP handshake / teardown quick facts

```
3-way handshake:  SYN -> SYN-ACK -> ACK            (opens a connection, syncs ISNs both ways)
4-way teardown:   FIN -> ACK -> FIN -> ACK          (closes each direction independently)
abrupt teardown:  RST                               (no negotiation — refused/crashed/violated)
TIME_WAIT:        held by the active closer, ~2xMSL, catches a lost final ACK
ss -tan state time-wait | wc -l                     # count TIME_WAIT sockets
```

## TLS handshake quick facts

```
TLS 1.2: ClientHello -> ServerHello+Cert+ServerKeyExchange+Done -> ClientKeyExchange+Finished -> Finished
         (2 round trips before application data)
TLS 1.3: ClientHello+key_share -> ServerHello+key_share+Cert+Finished -> Finished
         (1 round trip; 0-RTT possible on resumption)
Forward secrecy = ephemeral (EC)DHE key exchange; mandatory in TLS 1.3, optional (often absent) pre-1.3.
```

## Well-known ports

```
20/21   FTP (data/control)         53      DNS (UDP + TCP for large/zone-transfer)
22      SSH                        80      HTTP
25      SMTP                       443     HTTPS (TLS)
443/UDP HTTP/3 (QUIC)              123     NTP
6443    Kubernetes API server      10250   kubelet API
```

## HTTP version headers/facts

```
HTTP/1.1  Connection: keep-alive          text protocol, 1 request in flight per connection (app-layer HOL blocking)
HTTP/2    :method :path :scheme :authority (pseudo-headers) — binary framed, multiplexed, HPACK header compression
HTTP/3    Alt-Svc: h3=":443"               runs over QUIC/UDP, per-stream reliability (no transport-layer HOL blocking)
```
