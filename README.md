# DevOps Mastery Roadmap

A personalised, day-wise DevOps learning plan — target: ~2 hrs/day, goal is to be interview-ready across Linux, Kubernetes, IaC, CI/CD, DevSecOps, and Observability, with a few certification opportunities (CKA, Terraform Associate, AWS SAA-C03, CKS) along the way.

**Legend:** 🟢 already have exposure &nbsp;·&nbsp; ⚡ interview-critical &nbsp;·&nbsp; 📌 cert/milestone/project day

## How this repo is organized

Every day of the roadmap gets its own `Day-NN/` folder (or `Day-NN-MM/` for multi-day consolidation/capstone blocks). Each folder follows the same structure:

```
Day-NN/
├── 01-README-<Topic>.md      # one or more split notes files — the day's actual content
├── 02-README-<Topic>.md      # (split by sub-topic, 2-4 files depending on the day)
├── ...
├── LAB.md                    # hands-on practicals for the day
├── CHEATSHEET.md             # dense command/syntax quick-reference
├── QUIZ.md                   # self-test questions + the day's interview question, with full answers
└── RESOURCES.md              # the assigned best resource + curated further reading
```

Every README file follows the same internal shape: a **Brief** (what it is, why it matters), a detailed technical walkthrough, **Points to Remember**, and **Common Mistakes**. The source plan lives in [`devops_roadmap.xlsx`](devops_roadmap.xlsx).

## Phase overview

| # | Phase | Focus Area | Key Tools | Days | Cert / Milestone |
|---|---|---|---|---|---|
| 0 | Foundation | Linux, Git deep, Container security, Networking, Python for DevOps | Bash · Docker · Podman · Trivy · Dive · kubectl basics | Days 1–21 | – |
| 1 | Core DevOps | Kubernetes deep, IaC (Terraform), Config mgmt, Cloud (AWS), Secrets, MQ | Helm · ArgoCD · Terraform · Ansible · Vault · EKS · AWS CLI | Days 22–60 | 📌 CKA + Terraform Associate |
| 2 | CI/CD & Security | Pipelines, GitOps, DevSecOps, Supply chain, Deployment strategies | GitHub Actions · GitLab CI · ArgoCD · Trivy · Snyk · Cosign · OPA | Days 61–88 | – |
| 3 | Observability | Metrics, Logging, Tracing, SRE principles, On-call practices | Prometheus · Grafana · Loki · OpenTelemetry · Jaeger · PagerDuty | Days 89–109 | 📌 AWS SAA-C03 |
| 4 | Advanced / Spec. | Service mesh, Platform eng, FinOps, MLOps, Advanced security, Multi-cloud | Istio · Crossplane · Backstage · Kubecost · Kubeflow · Kyverno | Days 110–150 | 📌 CKS |

> **Status:** All 5 phases (Days 1 through 141-150) are complete.

---

## Phase 0 — Foundation (Days 1–21)

| Day | Week | Topic | Flag |
|---|---|---|---|
| [Day-01](Day-01/) | W1 | Linux Fundamentals I | ⚡ |
| [Day-02](Day-02/) | W1 | Linux Fundamentals II | ⚡ |
| [Day-03](Day-03/) | W1 | Processes & System | |
| [Day-04](Day-04/) | W1 | Networking on Linux | ⚡ |
| [Day-05](Day-05/) | W1 | Bash Scripting I | |
| [Day-06](Day-06/) | W1 | Bash Scripting II — Text processing | |
| [Day-07](Day-07/) | W1 | Systemd & Cron | |
| [Day-08](Day-08/) | W2 | Git Deep Dive I | ⚡ |
| [Day-09](Day-09/) | W2 | Git Deep Dive II — Workflows | |
| [Day-10](Day-10/) | W2 | Docker Deep Dive I | ⚡ |
| [Day-11](Day-11/) | W2 | Docker Deep Dive II | |
| [Day-12](Day-12/) | W2 | Container Security | ⚡ |
| [Day-13](Day-13/) | W2 | TCP/IP & DNS | ⚡ |
| [Day-14](Day-14/) | W2 | Load Balancing & Proxies | |
| [Day-15](Day-15/) | W3 | Python for Automation I | |
| [Day-16](Day-16/) | W3 | Python for Automation II — AWS | ⚡ |
| [Day-17](Day-17/) | W3 | YAML / JSON / Config Parsing | |
| [Day-18](Day-18/) | W3 | Storage & Filesystems | |
| [Day-19](Day-19/) | W3 | Performance & Debugging | ⚡ |
| [Day-20](Day-20/) | W3 | Phase 0 Review + Mock Interview | 📌 |
| [Day-21](Day-21/) | W3 | Portfolio & Documentation | 📌 |

## Phase 1 — Core DevOps (Days 22–60)

| Day | Week | Topic | Flag |
|---|---|---|---|
| [Day-22](Day-22/) | W4 | K8s Architecture Deep Dive | ⚡ |
| [Day-23](Day-23/) | W4 | Workloads: Deployments & StatefulSets | ⚡ |
| [Day-24](Day-24/) | W4 | Networking: Services & Ingress | ⚡ |
| [Day-25](Day-25/) | W4 | Storage in K8s | |
| [Day-26](Day-26/) | W4 | RBAC & Security | ⚡ |
| [Day-27](Day-27/) | W4 | Autoscaling | ⚡ |
| [Day-28](Day-28/) | W4 | Helm & Kustomize | |
| [Day-29](Day-29/) | W5 | Terraform Fundamentals | ⚡ |
| [Day-30](Day-30/) | W5 | Terraform Remote State & Modules | ⚡ |
| [Day-31](Day-31/) | W5 | Terraform CI & Advanced Patterns | |
| [Day-32](Day-32/) | W5 | AWS Core: VPC & Networking | ⚡ |
| [Day-33](Day-33/) | W5 | AWS IAM Deep Dive | ⚡ |
| [Day-34](Day-34/) | W5 | AWS Compute: EKS, ECS, Lambda | |
| [Day-35](Day-35/) | W5 | AWS Storage & Databases | |
| [Day-36](Day-36/) | W6 | Ansible Fundamentals | |
| [Day-37](Day-37/) | W6 | Ansible Roles & Vault | |
| [Day-38](Day-38/) | W6 | HashiCorp Vault | ⚡ |
| [Day-39](Day-39/) | W6 | Secrets in AWS & K8s | |
| [Day-40](Day-40/) | W6 | K8s Observability & Debugging | ⚡ |
| [Day-41](Day-41/) | W6 | NetworkPolicies | |
| [Day-42](Day-42/) | W6 | K8s + IaC + AWS Mock Interview | 📌 |
| [Day-43](Day-43/) | W7 | AWS Monitoring & Cost | |
| [Day-44](Day-44/) | W7 | AWS Reliability & DR | ⚡ |
| [Day-45](Day-45/) | W7 | K8s Advanced Scheduling | |
| [Day-46](Day-46/) | W7 | K8s Operators & CRDs | |
| [Day-47](Day-47/) | W7 | Terraform on AWS — Full Project | 📌 |
| [Day-48](Day-48/) | W7 | EKS Deep Dive | ⚡ |
| [Day-49](Day-49/) | W8 | AWS Security Services | ⚡ |
| [Day-50](Day-50/) | W8 | Probes, Resources & QoS | ⚡ |
| [Day-51](Day-51/) | W8 | K8s Upgrades & Cluster Management | |
| [Day-52](Day-52/) | W8 | AWS Messaging Deep Dive | |
| [Day-53](Day-53/) | W8 | Phase 1 Project: Full Stack IaC | 📌 |
| [Day-54](Day-54/) | W8 | CKA Exam Prep I | 📌 |
| [Day-55](Day-55/) | W9 | CKA Exam Prep II | 📌 |
| [Day-56](Day-56/) | W9 | Terraform Associate Exam Prep | 📌 |
| [Day-57](Day-57/) | W9 | Service Mesh Intro | |
| [Day-58](Day-58/) | W9 | AWS CDK | |
| [Day-59](Day-59/) | W9 | Phase 1 Mock Interview Day | 📌 |
| [Day-60](Day-60/) | W9 | Phase 1 → Phase 2 Transition | |

## Phase 2 — CI/CD & Security (Days 61–88)

| Day | Week | Topic | Flag |
|---|---|---|---|
| [Day-61](Day-61/) | W10 | GitHub Actions Fundamentals | ⚡ |
| [Day-62](Day-62/) | W10 | Advanced GitHub Actions | |
| [Day-63](Day-63/) | W10 | GitLab CI Deep Dive | |
| [Day-64](Day-64/) | W10 | ArgoCD Deep Dive | ⚡ |
| [Day-65](Day-65/) | W10 | Deployment Strategies | ⚡ |
| [Day-66](Day-66/) | W10 | Shift-Left Security | ⚡ |
| [Day-67](Day-67/) | W11 | Container & Registry Security | ⚡ |
| [Day-68](Day-68/) | W11 | Policy as Code | |
| [Day-69](Day-69/) | W11 | Jenkins (for Legacy Environments) | |
| [Day-70](Day-70/) | W11 | Artifact Management & Registries | |
| [Day-71](Day-71/) | W11 | Testing in CI Pipelines | |
| [Day-72](Day-72/) | W11 | Load & Chaos Testing | ⚡ |
| [Day-73](Day-73/) | W12 | Full CI/CD Pipeline Project | 📌 |
| [Day-74](Day-74/) | W12 | Compliance & Audit | |
| [Day-75](Day-75/) | W12 | Database Migrations in CI/CD | ⚡ |
| [Day-76](Day-76/) | W12 | CI/CD & DevSecOps Review | |
| [Day-77](Day-77/) | W12 | Multi-cloud CI/CD | |
| [Day-78](Day-78/) | W13 | Developer Experience (DX) in CI/CD | |
| [Day-79](Day-79/) | W13 | Runtime Security | |
| [Day-80](Day-80/) | W13 | Phase 2 Project: Full Pipeline | 📌 |
| [Day-81-88](Day-81-88/) | W13-W14 | Phase 2 Consolidation + Interview Prep | 📌 |

## Phase 3 — Observability (Days 89–109)

| Day | Week | Topic | Flag |
|---|---|---|---|
| [Day-89](Day-89/) | W15 | Prometheus Fundamentals | ⚡ |
| [Day-90](Day-90/) | W15 | Grafana Dashboards | ⚡ |
| [Day-91](Day-91/) | W15 | Thanos & Long-term Metrics | |
| [Day-92](Day-92/) | W15 | Loki & Log Aggregation | ⚡ |
| [Day-93](Day-93/) | W15 | OpenSearch Deep Dive | |
| [Day-94](Day-94/) | W16 | OpenTelemetry | ⚡ |
| [Day-95](Day-95/) | W16 | Jaeger & Tempo | |
| [Day-96](Day-96/) | W16 | SLOs, SLAs & Error Budgets | ⚡ |
| [Day-97](Day-97/) | W16 | Incident Management | ⚡ |
| [Day-98](Day-98/) | W16 | Toil Elimination & Reliability Patterns | |
| [Day-99](Day-99/) | W17 | AWS-native Observability | |
| [Day-100](Day-100/) | W17 | Datadog / New Relic (Commercial Tools) | |
| [Day-101-109](Day-101-109/) | W17-W18 | Phase 3 Project + AWS SAA-C03 Prep | 📌 |

## Phase 4 — Advanced / Specialization (Days 110–150)

| Day | Week | Topic | Flag |
|---|---|---|---|
| [Day-110](Day-110/) | W19 | Istio Deep Dive | |
| [Day-111](Day-111/) | W19 | Cilium & eBPF | ⚡ |
| [Day-112](Day-112/) | W19 | Platform Engineering Concepts | |
| [Day-113](Day-113/) | W19 | Crossplane & Self-service Infra | |
| [Day-114](Day-114/) | W20 | Cloud Cost Engineering | ⚡ |
| [Day-115](Day-115/) | W20 | Kubernetes Cost Optimisation | |
| [Day-116](Day-116/) | W20 | MLOps Fundamentals | ⚡ |
| [Day-117](Day-117/) | W20 | Model Serving & Kubeflow | ⚡ |
| [Day-118](Day-118/) | W20 | Agentic AI Infrastructure | ⚡ |
| [Day-119](Day-119/) | W21 | Zero Trust & mTLS | ⚡ |
| [Day-120](Day-120/) | W21 | CKS Exam Prep | 📌 |
| [Day-121-130](Day-121-130/) | W21-W22 | Multi-cloud & Edge | |
| [Day-131-140](Day-131-140/) | W23-W24 | GitOps at Scale | |
| [Day-141-150](Day-141-150/) | W25 | Final Interview Blitz | 📌 |

---

## Interview tip

Always frame answers around **why** you made an architectural decision, not just what you used — from-scratch build experience is what separates a strong answer from a memorized one.
