# Day 121-130 — Resources: Multi-Cloud & Edge

## Primary (assigned)

- **Cluster API documentation** (cluster-api.sigs.k8s.io) — the assigned best resource for this block. Read the "Concepts" section first (Cluster, Machine, MachineDeployment, infrastructure/bootstrap providers) before touching `clusterctl` — the CLI makes far more sense once the object model is clear. The "Quick Start" walks through exactly the kind-bootstrap → `clusterctl init` → generate → apply flow used in LAB.md Part 3, for whichever infrastructure provider you pick.

## GKE / EKS / AKS — official docs and comparisons

- **GKE documentation** (cloud.google.com/kubernetes-engine/docs) — specifically the "Autopilot overview," "VPC-native clusters," and "Workload Identity" pages, which back file 1's architecture claims directly.
- **Amazon EKS documentation** (docs.aws.amazon.com/eks) — specifically "Amazon VPC CNI plugin for Kubernetes" (for the ENI/prefix-delegation mechanics) and "IAM roles for service accounts."
- **Azure AKS documentation** (learn.microsoft.com/azure/aks) — specifically "Azure CNI Overlay" and "Use Microsoft Entra Workload ID."
- **Kubernetes the Hard Way — cloud provider comparisons** and vendor-neutral writeups comparing managed Kubernetes control-plane models (search "GKE vs EKS vs AKS control plane") — useful for triangulating claims across multiple independent sources rather than relying on any one vendor's marketing page.

## Anthos and Azure Arc

- **Google Cloud Anthos documentation** (cloud.google.com/anthos/docs) — "Fleet management overview," "Config Sync," and "Policy Controller" pages cover the mechanics behind file 2's fleet/GitOps/policy sections.
- **Anthos Attached Clusters documentation** — specifically how to register EKS/AKS clusters into a GCP fleet; the concrete alternative to the older, heavier Anthos on AWS/Azure offering.
- **Azure Arc-enabled Kubernetes documentation** (learn.microsoft.com/azure/azure-arc/kubernetes) — "Connect a Kubernetes cluster," "Cluster extensions," and the "GitOps (Flux v2)" pages cover file 2's Arc section directly.
- **CNCF case studies** (cncf.io/case-studies) — search for multi-cluster/fleet-management writeups; useful for citing a real production example of Anthos or Arc adoption instead of a purely hypothetical one in an interview.

## Istio multi-cluster

- **Istio documentation — "Multi-Cluster Installation"** (istio.io/latest/docs/setup/install/multicluster) — the authoritative source for the Multi-Primary vs. Primary-Remote topology decision and the exact `istioctl` install sequence used in file 3/LAB.md Part 4.
- **Istio documentation — "Deploy an app across multiple clusters"** — the source for the `helloworld`/`sleep` sample used to verify cross-cluster load balancing.
- **Istio documentation — "Locality Load Balancing"** — covers `localityLbSetting` and `outlierDetection` in depth; read this if Q9/Q9's answer in QUIZ.md wasn't immediately obvious.
- **Istio documentation — "Certificate Management" / `istio-csr`** (via cert-manager's documentation) — the production-grade approach to the shared-root-CA requirement covered in file 3, beyond the manual `samples/certs` walkthrough.

## Cloudflare Workers and edge computing

- **Cloudflare Workers documentation** (developers.cloudflare.com/workers) — start with "How Workers works" for the V8 isolate model, then "KV," "Durable Objects," "R2," and "D1" for the storage bindings covered in file 4.
- **Cloudflare R2 documentation** — specifically the "Pricing" page, which states the zero-egress model directly from the source rather than secondhand.
- **"Zero Cold Starts" and V8 isolates engineering writeups** (Cloudflare's own engineering blog, search "how Workers works" or "V8 isolates") — deeper mechanical explanation of why isolate-based execution avoids the cold-start problem structurally, not just as a marketing claim.

## Cloud cost and egress pricing references

- **AWS Data Transfer pricing page** (aws.amazon.com/ec2/pricing/on-demand — "Data Transfer" section) and **AWS NAT Gateway pricing page** — the primary source for the exact current tiered rates; this block's cheatsheet ranges will drift from these over time, so check the live page before quoting a specific number in a real conversation.
- **Google Cloud Network pricing page** (cloud.google.com/vpc/network-pricing) — covers the Premium vs. Standard network tier distinction that affects GCP's egress rate, referenced in file 4.
- **Azure Bandwidth pricing page** (azure.microsoft.com/pricing/details/bandwidth) — the Azure equivalent, including the Azure NAT Gateway pricing details.
- **"The AWS Bill You Didn't Expect" / cloud-cost-focused engineering blogs** (e.g., the Duckbill Group / Corey Quinn's writing on AWS billing, and similar FinOps-focused sources for GCP/Azure) — genuinely useful for the "hidden costs" framing this block's interview question demands; these sources specialize in exactly the NAT/LB-processing-fee gotchas covered in file 4.
- **FinOps Foundation** (finops.org) — the "Data Transfer" and "Cost Allocation" framework pages are a good structured reference for how organizations actually track and attribute cross-cloud/cross-account data-transfer cost in practice, beyond a single napkin-math estimate.
