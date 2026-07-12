# Day 11 — Cheatsheet: Docker Deep Dive II

## Network drivers quick reference

```
bridge    default, per-host, NAT'd via docker0/user-defined bridge, needs -p to publish ports
host      no isolation, shares host's netns directly, -p is ignored (no namespace to map into)
none      loopback only, no eth0, no route anywhere
overlay   multi-host, VXLAN-based, requires Swarm mode (docker swarm init)
```

## `docker network` commands

```bash
docker network ls                                   # list all networks
docker network create --driver bridge mynet          # create a user-defined bridge network (gets DNS)
docker network create --driver overlay --attachable myoverlay   # requires swarm mode
docker network inspect mynet                         # subnet, connected containers + their IPs
docker network connect mynet <container>              # attach a running container to a network
docker network disconnect mynet <container>            # detach
docker network rm mynet
docker network prune                                  # remove all unused networks

docker run --network mynet ...                        # attach at run time
docker run --network host ...
docker run --network none ...
docker run --network-alias db ...                      # extra resolvable name on a user-defined network
```

## Namespaces / veth inspection (Linux host)

```bash
docker inspect -f '{{.State.Pid}}' <container>          # get container's PID
sudo ln -sfT /proc/$PID/ns/net /var/run/netns/<name>    # expose netns to ip netns tooling
sudo ip netns list
sudo ip netns exec <name> ip addr                        # eth0 + lo from inside, via host tools
sudo ip netns exec <name> ip route
ip link show docker0                                     # the default bridge
ip link show | grep veth                                 # host-side veth ends (one per running container)
brctl show                                                # bridge -> attached veth interfaces
```

## DNS

```bash
docker exec <c> cat /etc/resolv.conf     # nameserver 127.0.0.11 = Docker's embedded DNS
docker exec <c> getent hosts <name>       # resolve another container/service by name
docker exec <c> cat /etc/hosts            # static entries only — NOT how service-name DNS works
```
Default `bridge` network -> no embedded DNS (IP only). User-defined bridge/overlay -> DNS works. Compose creates a user-defined network automatically.

## Volumes vs bind mounts vs tmpfs

```bash
# named volume — Docker-managed, lives under /var/lib/docker/volumes
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:16
docker run --mount type=volume,source=pgdata,target=/var/lib/postgresql/data postgres:16
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata
docker volume prune                       # remove all unreferenced volumes

# bind mount — exact host path, host-dependent
docker run -v /home/deploy/app:/usr/src/app node:20
docker run --mount type=bind,source=/home/deploy/app,target=/usr/src/app node:20

# tmpfs — RAM-only, gone when container stops
docker run --tmpfs /app/cache:size=64m,noexec tomcat
docker run --mount type=tmpfs,destination=/app/cache,tmpfs-size=64m tomcat
```

| | Managed by | Survives `docker rm`? | Host-path dependent? | Typical use |
|---|---|---|---|---|
| Named volume | Docker | Yes | No | Prod DB data |
| Bind mount | You | Yes (it's just a host dir) | Yes | Local dev, live reload |
| tmpfs | Kernel (RAM) | No | No | Secrets, scratch, cache |

## Compose v2 essentials

```bash
docker compose version          # confirm v2 plugin (space, not hyphen)
docker compose up -d             # start stack, detached
docker compose up -d --build     # rebuild images first
docker compose ps                # service status incl. health
docker compose logs -f <svc>     # follow logs for one service
docker compose exec <svc> sh     # shell into a running service
docker compose stop <svc>        # stop one service
docker compose restart <svc>
docker compose down              # stop + remove containers/networks (keeps named volumes)
docker compose down -v           # also remove named volumes — DESTROYS DATA
docker compose up -d --scale app=3   # multiple replicas of one service (drop container_name first)
```

## Compose file skeleton

```yaml
services:
  app:
    image: myapp:latest
    depends_on:
      redis:
        condition: service_healthy   # service_started (default) | service_healthy | service_completed_successfully
    networks: [backend]
    volumes:
      - app_data:/data
    environment:
      REDIS_HOST: redis
    deploy:
      resources:
        limits: { cpus: "0.50", memory: 128M }
    restart: on-failure

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

networks:
  backend:
    driver: bridge

volumes:
  app_data:
```

## Resource limit flags

```bash
docker run --memory=512m --memory-swap=512m --cpus=1.5 myapp
```
```
--memory / -m       hard RAM ceiling for the container's cgroup -> OOM killer on breach
--memory-swap       combined mem+swap ceiling; == --memory disables extra swap headroom
--cpus               hard CPU ceiling (fractional cores), throttles, does not kill
--cpu-shares         relative weight (default 1024), only matters under contention
```

## `docker stats` / `docker inspect`

```bash
docker stats                             # live table: CPU %, MEM USAGE/LIMIT, NET I/O, BLOCK I/O
docker stats --no-stream                  # one-shot snapshot
docker stats <c1> <c2>                    # scope to specific containers

docker inspect <c> --format '{{.HostConfig.Memory}}'      # configured memory limit (bytes)
docker inspect <c> --format '{{.HostConfig.NanoCpus}}'    # configured CPU limit
docker inspect <c> --format '{{.State.OOMKilled}}'         # true if kernel OOM killer took it down
```
