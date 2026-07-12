# Day 69 — Cheatsheet: Jenkins

## Declarative pipeline skeleton

```groovy
pipeline {
    agent any                       // or: agent none, agent { label 'x' }, agent { kubernetes { yaml "..." } }

    environment {
        APP_ENV = 'staging'
        AWS_KEY = credentials('aws-key-id')   // masked in logs automatically
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Name') {
            when { branch 'main' }            // conditional stage execution
            steps {
                sh 'echo hello'
                script {                       // drop into raw Groovy when needed
                    def x = [1,2,3]
                    x.each { echo "${it}" }
                }
            }
        }
    }

    post {
        always  { junit 'results.xml' }
        success { echo 'ok' }
        failure { slackSend(channel: '#ci', message: 'build failed') }
    }
}
```

## Scripted pipeline skeleton

```groovy
node {
    stage('Build') {
        sh 'make build'
    }
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy') {
            sh './deploy.sh'
        }
    }
}
```

## Kubernetes dynamic agent (multi-container pod)

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
    securityContext: { privileged: true }
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
        stage('Build') { steps { container('docker') { sh 'docker build -t app .' } } }
        stage('Scan')  { steps { container('trivy')  { sh 'trivy image --exit-code 1 app' } } }
    }
}
```

## Shared library usage

```groovy
@Library('company-shared-library@v2.3.0') _
standardPipeline(appName: 'checkout-service', environment: 'staging')
```

Directory layout of a shared library repo:
```
vars/standardPipeline.groovy   # def call(Map config) { ... } -> callable as standardPipeline(...)
src/org/company/Utils.groovy   # regular Groovy classes
resources/...                  # static files, via libraryResource()
```

## Jenkins CLI / common ops

```bash
# Get initial admin password (Helm install)
kubectl exec -n jenkins -it svc/jenkins -c jenkins -- cat /run/secrets/additional/chart-admin-password

# Port-forward to reach the UI
kubectl port-forward -n jenkins svc/jenkins 8080:8080

# Trigger a build via REST API
curl -X POST http://JENKINS_URL/job/my-job/build --user user:token
```

## Jenkins → GitHub Actions concept mapping

| Jenkins | GitHub Actions |
|---|---|
| `Jenkinsfile` | `.github/workflows/*.yml` |
| `stage()` | `jobs.<id>.steps` |
| `agent` | `runs-on:` |
| Shared Library | Reusable workflow (`workflow_call`) / composite action |
| `credentials('id')` | Secrets, or OIDC (`permissions: id-token: write`) |
| `when { branch 'main' }` | `if: github.ref == 'refs/heads/main'` |
| `post { failure {} }` | `if: failure()` step |
