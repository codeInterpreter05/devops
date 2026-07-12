# Day 69 — Lab: Jenkins for Legacy Environments

**Goal:** Stand up Jenkins with a Kubernetes plugin for dynamic pod agents, and build a Jenkinsfile equivalent to an existing pipeline (the day's assigned hands-on activity), then compare it against a GitHub Actions equivalent.

**Prerequisites:**
- A local Kubernetes cluster (`kind` or `minikube`) with at least 4GB memory allocated.
- `kubectl`, `helm` installed.
- Docker installed locally.
- (For Lab 4) a GitHub repo you control.

---

### Lab 1 — Stand up Jenkins on Kubernetes

1. Create a cluster and install Jenkins via Helm:
   ```bash
   kind create cluster --name day69
   helm repo add jenkinsci https://charts.jenkins.io
   helm repo update
   helm install jenkins jenkinsci/jenkins -n jenkins --create-namespace \
     --set controller.serviceType=ClusterIP
   ```
2. Get the initial admin password and port-forward to reach the UI:
   ```bash
   kubectl exec -n jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password
   kubectl port-forward -n jenkins svc/jenkins 8080:8080
   ```
3. Log in at `http://localhost:8080` with user `admin` and the password from step 2.
4. Install the **Kubernetes plugin** and the **Pipeline: Declarative** plugin (usually bundled already) via Manage Jenkins → Plugins.

**Success criteria:** You can log into a running Jenkins instance and confirm the Kubernetes plugin is installed and connected to your cluster (Manage Jenkins → Clouds should show a working Kubernetes cloud).

---

### Lab 2 — Write a declarative Jenkinsfile with a dynamic pod agent

1. In a scratch repo, add a `Jenkinsfile`:
   ```groovy
   pipeline {
       agent {
           kubernetes {
               yaml """
   apiVersion: v1
   kind: Pod
   spec:
     containers:
     - name: shell
       image: alpine:3.19
       command: ['cat']
       tty: true
   """
           }
       }
       stages {
           stage('Hello') {
               steps {
                   container('shell') {
                       sh 'echo Hello from a dynamic pod agent && uname -a'
                   }
               }
           }
       }
   }
   ```
2. Create a Pipeline job in Jenkins pointing at this repo/Jenkinsfile (or paste it directly into a Pipeline job's script box for a first pass).
3. Run the build and confirm, in the console log, that a new Pod was created in your `kind` cluster during the build:
   ```bash
   kubectl get pods -n jenkins --watch
   ```

**Success criteria:** You observe an ephemeral agent Pod appear during the build and disappear after it completes, and the build output confirms the `shell` container executed your command.

---

### Lab 3 — The core hands-on activity: build a Jenkinsfile matching a prior pipeline

1. Take the pipeline structure from an earlier CI/CD day in this course (Day 61, a build → scan → deploy pipeline) and reimplement its stages as a Jenkinsfile: `Checkout` → `Build` (Docker build) → `Scan` (Trivy) → `Deploy` (only on `main`, via a `when { branch 'main' }` block).
2. Use a multi-container Kubernetes pod agent so `Build` runs in a `docker` container and `Scan` runs in an `aquasec/trivy` container, each wrapped in its own `container('name') { }` block.
3. Add a `post { failure { ... } }` block that echoes a failure message (stand-in for a real Slack notification).
4. Run it against both a passing and a deliberately CVE-laden test image, confirming the pipeline fails appropriately on the latter.

**Success criteria:** A single Jenkinsfile reproduces the same logical pipeline (build, scan-gate, conditional deploy) as your Day 61 GitHub Actions/GitLab CI pipeline, running on dynamic Kubernetes agents.

---

### Lab 4 — Compare against a GitHub Actions equivalent

1. Write the equivalent `.github/workflows/day69.yml` for the same build → scan → deploy logic from Lab 3.
2. Push both the Jenkinsfile and the workflow file into the same test repo (Jenkins doesn't need to actually be wired to trigger from this repo — this is a side-by-side comparison exercise).
3. Write down, in a short notes file, three concrete syntax/behavior differences you noticed (e.g., how secrets are referenced, how conditional deploy-on-main is expressed, how agent/runner selection differs).

**Success criteria:** You have both pipeline definitions side by side and can articulate, out loud, the mapping between Jenkins concepts and GitHub Actions concepts from memory.

---

### Cleanup

```bash
helm uninstall jenkins -n jenkins
kind delete cluster --name day69
```

### Stretch challenge

Convert the Jenkinsfile's build+scan logic into a reusable Jenkins **shared library** `vars/buildAndScan.groovy`, then rewrite your Lab 3 Jenkinsfile to call it in 3 lines instead of duplicating the full pipeline body — mirroring how you'd extract the same logic into a GitHub Actions reusable workflow.
