# Day 54 — CKA Exam Prep I: kubectl Speed Tricks

**Phase:** 1 – Core DevOps | **Week:** W8 | **Domain:** Certifications | **Flag:** 📌 Milestone

## Brief

Given roughly 7 minutes per task, typing full YAML manifests by hand for every object is the single biggest time sink candidates fall into. The CKA exam rewards fluency with `kubectl`'s **imperative commands** (generate-and-modify instead of write-from-scratch) and a small set of shell aliases/environment variables that are explicitly sanctioned exam prep (they're just standard bash features, not exam-specific tools) — mastering these is a mechanical skill you build through repetition, not conceptual understanding you can reason your way into under time pressure.

## The core technique: generate YAML imperatively, then edit

Almost every task that needs a custom object (a Pod with specific probes, a Deployment with specific resource limits) is faster solved by using `kubectl create`/`kubectl run` with `--dry-run=client -o yaml` to generate a *starting point*, redirecting it to a file, then editing just the parts that need customizing — rather than typing the whole manifest from memory.

```bash
# Generate a Pod manifest without actually creating it, then edit
kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml
vim pod.yaml     # add probes, resource limits, etc. — editing beats typing from scratch
kubectl apply -f pod.yaml

# Same pattern for a Deployment
kubectl create deployment myapp --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Same pattern for a Job/CronJob
kubectl create job myjob --image=busybox --dry-run=client -o yaml -- /bin/sh -c "date" > job.yaml
kubectl create cronjob mycron --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- /bin/sh -c "date" > cronjob.yaml

# ConfigMap / Secret from literals — no YAML needed for simple cases at all
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
kubectl create secret generic my-secret --from-literal=password=supersecret
```

**Why `--dry-run=client -o yaml` specifically** (not `--dry-run=server`): client-side dry-run generates the object locally without even contacting the API server to validate it — it's faster and doesn't depend on cluster-side admission webhooks/validation that could slow down or alter the generated output. You want a fast, predictable starting template, then you validate for real when you `apply` it.

## Fast edits to existing objects

```bash
kubectl edit deployment myapp                      # opens the live object in $EDITOR — set export EDITOR=vim once
kubectl set image deployment/myapp nginx=nginx:1.25 # change just the image, no YAML editing at all
kubectl scale deployment myapp --replicas=5
kubectl label pod mypod env=prod
kubectl annotate pod mypod description="test pod"
kubectl expose deployment myapp --port=80 --target-port=8080 --type=ClusterIP   # generate a Service imperatively
```

`kubectl set image` and `kubectl scale` in particular are worth using reflexively instead of `kubectl edit` + manual YAML change whenever the task is *only* about changing one field — it's strictly faster and has no room for a YAML typo.

## Aliases and shell setup — sanctioned, standard, and expected

```bash
alias k=kubectl
export do="--dry-run=client -o yaml"     # usage: kubectl run pod1 --image=nginx $do
export now="--force --grace-period=0"     # usage: kubectl delete pod stuck-pod $now (instant delete, skip graceful termination)

source <(kubectl completion bash)          # tab-completion for kubectl itself
complete -F __start_kubectl k               # tab-completion also works for the k alias
```

These are explicitly mentioned in the official CKA candidate documentation as acceptable exam-day setup (they're standard bash, present in the terminal environment you're given, not third-party tools) — setting them up **at the very start of the exam**, before touching any task, is standard practice among candidates and instructors alike, since the time investment (under a minute) pays for itself many times over across 17 tasks.

## Namespace and output-format speed tricks

```bash
kubectl config set-context --current --namespace=<ns>    # stop typing -n <ns> on every single command for a task scoped to one namespace
kubectl get pods -A                                        # all namespaces, one command, instead of a namespace-by-namespace loop
kubectl get pods -o wide                                   # node placement, IP — often needed for troubleshooting tasks
kubectl explain pod.spec.containers.resources               # in-terminal field reference, faster than searching docs for simple field lookups
kubectl api-resources                                        # find the right resource type/shortname fast (po, deploy, svc, ns, etc.)
```

Setting the current context's default namespace at the start of a namespace-scoped task removes an entire class of typos (`-n prod` vs. `-n production` vs. forgetting `-n` and hitting `default` by mistake) for the rest of that task.

## Points to Remember

- Generate-then-edit (`--dry-run=client -o yaml` piped to a file) beats typing YAML from scratch for almost every object-creation task — this is the single highest-leverage exam technique.
- `kubectl set image`, `kubectl scale`, `kubectl label`, `kubectl expose` handle common single-field changes faster and with less error surface than `kubectl edit` + manual YAML editing.
- `alias k=kubectl`, `export do="--dry-run=client -o yaml"`, and `export now="--force --grace-period=0"` are standard, exam-sanctioned setup — do this in the first minute of the exam, every time.
- Setting the default namespace via `kubectl config set-context --current --namespace=<ns>` for a namespace-scoped task eliminates repetitive, error-prone `-n` typing for the rest of that task.
- `kubectl explain <resource>.<field path>` is faster than searching the allowed docs website for a quick field-name/structure check.

## Common Mistakes

- Typing full Pod/Deployment YAML from memory under time pressure instead of generating a starting template with `--dry-run=client -o yaml` — slower and far more prone to indentation/typo errors that cost debugging time.
- Forgetting to set up aliases/shortcuts at the start of the exam, then paying the "typing `kubectl` in full, every time, for two hours" tax across all 17 tasks.
- Using `kubectl edit` for a one-field change (like bumping an image tag) when `kubectl set image` does it in one line with no risk of accidentally breaking unrelated YAML while in the editor.
- Not switching the default namespace for a namespace-scoped task, then repeatedly forgetting `-n <namespace>` and getting confusing "not found" errors against the `default` namespace instead.
- Confusing `--dry-run=client` with `--dry-run=server` — server-side dry-run actually contacts the API server (slower, and can behave unexpectedly if admission webhooks/validation are involved) when all you wanted was a fast local YAML template.
