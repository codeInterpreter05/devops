# Day 69 — Jenkins: Shared Libraries, Kubernetes Agents, and Blue Ocean

**Phase:** 2 – CI/CD & Security | **Week:** W11 | **Domain:** CI/CD

## Brief

A single Jenkinsfile is fine for one repo. The moment an org has 50 repos all needing "checkout, build, scan, deploy," copy-pasting the same 200 lines of Groovy into every repo becomes a maintenance nightmare — one security fix needs to be manually applied 50 times. **Shared libraries** solve exactly this, the same way a shared Terraform module or a reusable GitHub Actions composite action does. Separately, **dynamic Kubernetes agents** solve Jenkins' other classic pain point: static build servers that sit idle most of the day and become a bottleneck during peak hours. Both are what separate "someone who's used Jenkins" from "someone who's operated Jenkins at scale."

## Jenkins Shared Libraries — reusable pipeline code

A shared library is a separate Git repository containing reusable Groovy code, imported into any Jenkinsfile with a single line. It follows a specific directory convention:

```
(shared-library-repo)/
├── vars/
│   └── standardPipeline.groovy      # global "var" — callable directly as a step
├── src/
│   └── org/company/PipelineUtils.groovy   # regular Groovy classes
└── resources/
    └── org/company/deploy-template.yaml    # non-Groovy resource files (loadable via libraryResource)
```

`vars/standardPipeline.groovy`:
```groovy
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    sh "docker build -t ${config.appName}:${env.BUILD_NUMBER} ."
                }
            }
            stage('Scan') {
                steps {
                    sh "trivy image --exit-code 1 --severity CRITICAL ${config.appName}:${env.BUILD_NUMBER}"
                }
            }
            stage('Deploy') {
                when { branch 'main' }
                steps {
                    sh "./deploy.sh ${config.appName} ${config.environment}"
                }
            }
        }
    }
}
```

Every application's Jenkinsfile then shrinks to a handful of lines:
```groovy
@Library('company-shared-library@v2.3.0') _

standardPipeline(appName: 'checkout-service', environment: 'staging')
```

Pinning the library to a version tag (`@v2.3.0`) rather than a branch is the critical practice here — otherwise, every single pipeline across every repo silently picks up shared-library changes the instant they're merged, with zero ability to roll a single team back independently. This is the exact same "pin to a tag, not `:latest`/`main`" principle from image tags and Terraform module versions, applied to Jenkins.

### Registering a shared library

Configured either globally (Manage Jenkins → System → Global Pipeline Libraries) for org-wide availability, or dynamically per-Jenkinsfile via `@Library('name@version')`. Global registration typically also allows `implicit: true`, making the library available without an explicit `@Library` import — convenient, but it means every pipeline is implicitly coupled to that library's existence, which can surprise people debugging an unfamiliar Jenkinsfile.

## Jenkins + Kubernetes — dynamic pod agents

Classic Jenkins runs builds on a fixed pool of **static agents** — VMs or bare-metal machines registered ahead of time, sitting around whether or not there's work to do. This wastes compute at idle times and creates a hard ceiling during busy periods (queued builds waiting for a free executor). The **Kubernetes plugin** flips this: Jenkins provisions a brand-new Pod as the build agent *on demand*, for the duration of a single build, then tears it down — true elastic, pay-for-what-you-use compute, and a guaranteed clean environment every single build (no leftover state from a previous job's dependencies polluting the next one).

```groovy
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    command: ['cat']
    tty: true
  - name: trivy
    image: aquasec/trivy:latest
    command: ['cat']
    tty: true
"""
        }
    }
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
        }
        stage('Scan') {
            steps {
                container('trivy') {
                    sh 'trivy image --exit-code 1 --severity CRITICAL myapp:${BUILD_NUMBER}'
                }
            }
        }
    }
}
```

Notice the **multi-container pod pattern**: each `container('name')` block runs its steps inside that specific sidecar container within the same Pod, sharing the same workspace volume — this is exactly how Kubernetes-native CI runners (Tekton, GitLab's Kubernetes executor) work too, and understanding it here transfers directly.

**Why `privileged: true` on the `docker` container**: building Docker images inside a Kubernetes Pod (Docker-in-Docker) needs elevated kernel access most containers don't have by default. This is a real, non-trivial security tradeoff — many orgs instead use rootless/daemonless builders like **Kaniko** or **Buildah** specifically to avoid granting `privileged: true` to CI agent pods.

## Blue Ocean — a modern UI layer on classic Jenkins

Blue Ocean is a plugin that replaces Jenkins' famously dated default UI with a visual, drag-and-droppable pipeline editor and a much clearer stage-by-stage visualization of running/completed builds. It doesn't change how pipelines execute underneath — it's purely a better window into the same Jenkinsfile-driven pipelines. As of recent years Blue Ocean's active development has slowed (Jenkins' own UI has absorbed some of its ideas), but it's still commonly found in existing Jenkins installs and worth recognizing when you see it.

## Points to Remember

- Shared libraries turn duplicated pipeline Groovy across dozens of repos into a single, versioned, centrally-maintained dependency — pin to a version tag, never to a floating branch, for the same reason you don't deploy against `:latest` images.
- `vars/*.groovy` files become directly callable pipeline steps (like `standardPipeline(...)`); `src/` holds regular importable Groovy classes for more complex logic.
- Dynamic Kubernetes agents provision a fresh Pod per build and tear it down after — solving both idle-capacity waste and environment-drift between builds, at the cost of per-build Pod startup latency.
- The multi-container Pod pattern (`container('name') { steps }`) lets one build use several purpose-built images (a build tool, a scanner, a deploy CLI) sharing one workspace — the same underlying pattern used by Kubernetes-native CI systems generally.
- `privileged: true` for Docker-in-Docker build agents is a real security exposure; Kaniko/Buildah exist specifically to build container images without it.

## Common Mistakes

- Importing a shared library without pinning a version (`@Library('company-lib') _` with no `@version`) — every pipeline across the org silently changes behavior the moment someone merges to the library's default branch, with no controlled rollout and no easy per-team rollback.
- Granting `privileged: true` to every CI agent pod by default "because Docker builds needed it once," rather than scoping it only to the specific pipelines that actually build images, and evaluating Kaniko/Buildah as a non-privileged alternative first.
- Assuming a Kubernetes agent Pod's ephemeral filesystem behaves like a persistent static agent's — expecting caches (`~/.m2`, `node_modules`, Docker layer cache) to survive between builds by default, when a fresh Pod every time means every cache starts cold unless you explicitly wire in a persistent volume or external cache (e.g., a shared registry-based build cache).
- Forgetting that stages in a Kubernetes-agent pipeline still need explicit `container('name')` wrapping — steps issued outside any `container()` block run in the default `jnlp` agent container, not whichever sidecar you intended, leading to confusing "command not found" errors for tools that only exist in a sidecar image.
- Treating Blue Ocean as a different pipeline engine rather than a UI skin — debugging a pipeline failure by staring at Blue Ocean's visualization alone instead of checking the underlying raw console log, which often has detail the visual view doesn't surface.
