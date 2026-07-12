# Day 54 — Cheatsheet: CKA Exam Prep I

## Exam facts (confirm current numbers before your sitting)

```
17 performance-based tasks, ~2 hours, unequal weighting, 66% to pass
Allowed docs (single tab): kubernetes.io/docs, kubernetes.io/blog, github.com/kubernetes(-sigs)
No external search, no personal notes, remote-proctored via PSI Secure Browser
Curriculum domains: Workloads & Scheduling | Cluster Architecture/Installation/Config | Services & Networking | Storage | Troubleshooting
```

## Exam-start shell setup (type this first, every time)

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
source <(kubectl completion bash)
complete -F __start_kubectl k
export EDITOR=vim
```

## Generate-then-edit (the core speed technique)

```bash
kubectl run mypod --image=nginx $do > pod.yaml
kubectl create deployment myapp --image=nginx --replicas=3 $do > deploy.yaml
kubectl create job myjob --image=busybox $do -- /bin/sh -c "date" > job.yaml
kubectl create cronjob mycron --image=busybox --schedule="*/5 * * * *" $do -- /bin/sh -c "date" > cronjob.yaml
kubectl create configmap my-config --from-literal=key1=value1
kubectl create secret generic my-secret --from-literal=password=supersecret
kubectl expose deployment myapp --port=80 --target-port=8080 --type=ClusterIP
```

## Fast single-field edits

```bash
kubectl set image deployment/myapp nginx=nginx:1.25
kubectl scale deployment myapp --replicas=5
kubectl label pod mypod env=prod
kubectl annotate pod mypod description="test"
kubectl edit deployment myapp
kubectl delete pod stuck-pod $now      # force delete, skip graceful termination
```

## Namespace / discovery speed

```bash
kubectl config set-context --current --namespace=<ns>
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <context-name>
kubectl get pods -A
kubectl get pods -o wide
kubectl api-resources
kubectl explain pod.spec.containers.resources
```

## Cordon / drain / taint drills

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
kubectl label node <node> disktype=ssd
kubectl taint node <node> key=value:NoSchedule
kubectl taint node <node> key=value:NoSchedule-    # remove a taint (trailing dash)
```

## kind — practice clusters

```bash
kind create cluster --name cka-practice
kind create cluster --name cka-practice --config kind-multi-node.yaml
kind get clusters
kubectl cluster-info --context kind-cka-practice
kind delete cluster --name cka-practice
```
```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

## killer.sh

```
2 free sessions per exam purchase, each 36 hours once activated
Intentionally harder than the real exam — a 50-60% score is a reasonable signal
Session 1: early, cold, honest baseline. Session 2: shortly before exam date.
Review ALL solutions after, not just missed tasks.
```
