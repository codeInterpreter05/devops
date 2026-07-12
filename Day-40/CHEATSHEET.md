# Day 40 — Cheatsheet: K8s Observability & Debugging

## Core debugging commands

```bash
kubectl describe pod <name> -n <ns>              # step 1, almost always — Events section is gold
kubectl get pods -n <ns>                          # status + restart count at a glance
kubectl get pods -n <ns> -o wide                  # + node, IP
kubectl logs <pod>                                # current container
kubectl logs <pod> -c <container>                 # specific container
kubectl logs <pod> --previous                     # PREVIOUS (crashed) instance — often the real error
kubectl logs <pod> -f                             # follow/stream
kubectl logs <pod> --since=10m
kubectl logs -l app=myapp --all-containers --prefix
kubectl exec -it <pod> -- /bin/sh                 # needs a shell IN the image
kubectl exec <pod> -c <container> -- env
kubectl port-forward pod/<name> 8080:80
kubectl port-forward svc/<name> 8080:80
```

## Events

```bash
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>
kubectl get events -n <ns> -w
# default TTL ~1h — gone after that, ship to persistent logging for later analysis
```

## Ephemeral debug containers (K8s 1.25+)

```bash
kubectl debug -it <pod> --image=busybox --target=<container>          # add debug container to running pod
kubectl debug <pod> -it --image=busybox --copy-to=<pod>-debug          # copy pod + add debug container
kubectl debug node/<node-name> -it --image=busybox                     # debug a NODE
```

## Exit codes cheat table

```
0    clean exit
1    generic app error
137  128+9  = SIGKILL  -> almost always OOM kill or forced kill (check probe failures too)
139  128+11 = SIGSEGV  -> segfault, native/low-level bug
143  128+15 = SIGTERM  -> graceful shutdown (scale-down, rolling update)
```

## OOMKilled diagnosis

```bash
kubectl describe pod <name> | grep -A5 "Last State"   # Reason: OOMKilled, Exit Code: 137
kubectl top pod <name> --containers                     # needs metrics-server
```

```yaml
resources:
  requests: { memory: "256Mi" }   # scheduling / bin-packing only, NOT a cap
  limits:   { memory: "512Mi" }   # kernel cgroup hard limit -> OOM kill if exceeded
```

## ImagePullBackOff triage

```bash
docker pull <exact-image:tag>                         # confirm the image really exists as named
kubectl create secret docker-registry regcred \
  --docker-server=<registry> --docker-username=<u> --docker-password=<p>
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```
Common causes: typo/tag mismatch, missing/expired `imagePullSecrets` (ECR tokens expire every 12h), blocked egress, registry rate limiting.

## Pending pod triage

```bash
kubectl describe pod <name>            # scheduling Events: Insufficient cpu/memory, taints, PVC unbound
kubectl get nodes                       # node capacity / autoscaler state
kubectl describe node <name>            # allocatable vs. requested
kubectl get pvc                         # check Bound vs Pending
```

## Init container / config errors

```bash
kubectl logs <pod> -c <init-container-name>      # init container logs (main app logs will be empty/irrelevant)
kubectl get configmap,secret -n <ns>              # cross-check referenced names/keys actually exist
```

## Tools

```bash
k9s                          # TUI: d=describe, l=logs, s=shell, ctrl-d=delete
stern <selector>              # tail logs from MULTIPLE pods, multiplexed
stern myapp --since 5m --tail 50
kubectl-debug                 # ephemeral container plugin (mostly superseded by `kubectl debug`)
```
