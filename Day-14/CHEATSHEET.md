# Day 14 — Cheatsheet: Load Balancing & Proxies

## nginx: upstream & proxy_pass basics

```nginx
upstream backend_pool {
    server 10.0.0.1:8080;              # plain pool member
    server 10.0.0.2:8080 weight=3;     # gets 3x traffic of an unweighted peer
    server 10.0.0.3:8080 backup;       # only used if all non-backup servers are down
    server 10.0.0.4:8080 down;         # manually out of rotation (maintenance/draining)
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_pool;             # no trailing slash: matched prefix kept
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## nginx: load-balancing & sticky-session directives

```nginx
upstream backend_pool {
    # choose ONE algorithm directive (default with none = round robin)
    least_conn;                            # fewest active connections
    ip_hash;                               # sticky by client IP (OSS built-in)
    hash $cookie_jsessionid consistent;    # sticky by cookie, consistent hashing (OSS built-in)

    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

```nginx
# nginx Plus only — not available in open-source nginx:
upstream backend_pool {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    health_check interval=5s fails=2 passes=2 uri=/healthz;
}
```

## nginx: passive health checks (open source)

```nginx
server 10.0.0.1:8080 max_fails=2 fail_timeout=10s;
# 2 failures within 10s -> mark down for the next 10s, then retry
# default if omitted: max_fails=1 fail_timeout=10s

proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
# controls what counts as a "failure" worth failing over from
```

## nginx: reload / validate / debug commands

```bash
nginx -t                     # validate config syntax before touching anything live
nginx -s reload              # graceful reload — old workers finish in-flight requests
nginx -s quit                # graceful shutdown
nginx -T                     # dump full merged config (all includes expanded)
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log
```

## HAProxy: frontend/backend basics

```
global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend http_in
    bind *:80
    acl is_api path_beg /api
    use_backend api_servers if is_api
    default_backend web_servers

backend web_servers
    balance roundrobin              # roundrobin | leastconn | source | uri
    server web1 10.0.0.1:8080 check
    server web2 10.0.0.2:8080 check
```

## HAProxy: active health checks & ACLs

```
server web1 10.0.0.1:8080 check inter 5s rise 2 fall 3
# check          = enable active health check (TCP connect by default)
# inter 5s       = probe every 5s
# rise 2         = 2 consecutive successes to mark UP
# fall 3         = 3 consecutive failures to mark DOWN

backend web_servers
    option httpchk GET /healthz       # make the check HTTP-level instead of plain TCP
    http-check expect status 200

# ACL examples for L7 routing:
acl is_api    path_beg /api
acl is_admin  hdr(host) -i admin.example.com
acl is_https  ssl_fc
use_backend api_servers if is_api
```

## HAProxy: stats page & admin

```
listen stats
    bind *:8404
    stats enable
    stats uri /haproxy?stats
    stats refresh 10s
```

## SSL/TLS termination quick reference

```nginx
# nginx: terminate TLS, forward plaintext to backend
server {
    listen 443 ssl;
    ssl_certificate     /etc/nginx/certs/example.com.pem;
    ssl_certificate_key /etc/nginx/certs/example.com.key;
    location / { proxy_pass http://backend_pool; }
}
```

```
# HAProxy: terminate TLS
frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
    default_backend web_servers

# HAProxy: TLS passthrough (route on SNI without decrypting)
frontend tls_passthrough
    mode tcp
    bind *:443
    acl sni_app1 req.ssl_sni -i app1.example.com
    use_backend app1_backend if sni_app1
```

## curl: useful debugging invocations

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://host/          # status code only
curl -sv http://host/ 2>&1 | grep -i "< HTTP"                   # response headers only
curl -k https://host/                                           # ignore self-signed cert errors
curl --resolve example.com:443:127.0.0.1 https://example.com/   # test SNI routing locally
for i in $(seq 1 9); do curl -s http://host/; echo; done         # observe LB distribution
```
