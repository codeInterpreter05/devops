# Day 54 — Lab: CKA Exam Prep I

**Goal:** Complete 3 timed, exam-style practice tasks on killer.sh (the assigned hands-on activity), plus drill `kubectl` speed techniques on a local `kind` cluster beforehand so the simulator session measures real readiness rather than unfamiliarity with the tooling.

**Prerequisites:** `kind` and `kubectl` installed locally, Docker running. A killer.sh session (either purchased standalone, or one of the two free sessions bundled with an active CKA exam registration).

---

### Lab 1 — Set up your exam-day shell configuration and drill it

1. Spin up a practice cluster:
   ```bash
   kind create cluster --name speed-drill
   ```
2. Configure the exact aliases/exports you'll use on exam day:
   ```bash
   alias k=kubectl
   export do="--dry-run=client -o yaml"
   export now="--force --grace-period=0"
   source <(kubectl completion bash)
   complete -F __start_kubectl k
   ```
3. Time yourself performing 5 object-creation tasks using generate-then-edit, not from-scratch YAML:
   - A Pod named `web` running `nginx:1.25` with a CPU request of `100m` and limit of `250m`.
   - A Deployment named `api` with 3 replicas of `httpd`, exposed via a ClusterIP Service on port 80.
   - A ConfigMap named `app-config` with two literal keys.
   - A CronJob named `cleanup` running every 10 minutes that echoes the date.
   - A NetworkPolicy that denies all ingress to pods labeled `app=web` except from pods labeled `role=frontend`.
4. Record how long each took. Repeat the same 5 tasks on a fresh cluster a second time and compare your times.

**Success criteria:** Your second-attempt times are meaningfully faster than your first, confirming the aliases/generate-then-edit workflow is becoming automatic rather than something you have to think through.

---

### Lab 2 — Multi-node cluster architecture drill

1. Create a 3-node `kind` cluster (1 control-plane, 2 workers):
   ```bash
   kind create cluster --name arch-drill --config kind-multi-node.yaml
   ```
2. Practice cluster-architecture-style tasks under a timer:
   - Cordon one worker node, drain it, and confirm existing pods rescheduled onto the other worker.
   - Label one worker node `disktype=ssd` and create a Pod with a `nodeSelector` targeting that label; confirm it schedules only there.
   - Use `kubectl taint` to taint a worker node `NoSchedule`, then confirm a plain Pod (no toleration) can't schedule there, and one with a matching toleration can.

**Success criteria:** You can perform cordon/drain, nodeSelector scheduling, and taint/toleration tasks each in under 5 minutes without referring to notes.

---

### Lab 3 — Complete 3 CKA practice tasks on killer.sh (core hands-on activity)

1. Start (or resume) a killer.sh session.
2. Pick 3 tasks spanning different curriculum domains (e.g., one Workloads & Scheduling, one Services & Networking, one Troubleshooting) rather than 3 tasks from the same domain.
3. Set a strict 7-minute timer per task. If you're not done at 7 minutes, mark it and move to the next — do not extend the timer.
4. After attempting all 3, review the official solutions for each — including ones you got right — and note any faster command sequence you didn't use.

**Success criteria:** You have a written note (even informal) of exactly which technique or command you missed for each task, to fold into further `kind`-based drilling before your next simulator session.

---

### Cleanup

```bash
kind delete cluster --name speed-drill
kind delete cluster --name arch-drill
```

### Stretch challenge

Recreate all 3 killer.sh tasks you just attempted from memory on a fresh `kind` cluster, without looking at the killer.sh solutions, timing yourself again — confirm you can now solve them meaningfully faster than your first attempt on the simulator itself.
