# Day 48 — EKS Deep Dive: EKS Networking Model & Add-ons (CoreDNS, kube-proxy, VPC CNI)

**Phase:** 1 – Core DevOps | **Week:** W7 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

This file directly answers this day's interview question — "explain the EKS networking model, how does VPC CNI differ from other CNI plugins" — which is one of the highest-signal EKS questions asked because it separates people who've only deployed workloads onto EKS from people who understand what's actually happening at the network layer underneath every Pod IP.

## The three core EKS add-ons

Every EKS cluster needs three components running to function at all, all manageable as **EKS Add-ons** (a first-class EKS feature — `aws eks create-addon` — giving AWS-managed version tracking and update orchestration, versus manually applying their manifests yourself):

- **CoreDNS** — cluster DNS server, resolving `<service>.<namespace>.svc.cluster.local` to ClusterIPs (and reverse). Runs as a Deployment (not a DaemonSet — it's a shared service, not a per-node agent), typically 2 replicas for HA. On Fargate-only clusters, CoreDNS itself needs a Fargate profile matching `kube-system` or it has nowhere to run.
- **kube-proxy** — runs as a DaemonSet on every node, implementing Kubernetes Services by programming iptables rules (or IPVS, configurable) that redirect traffic destined for a Service's ClusterIP to one of its backing pod IPs — this is the mechanism that makes a stable Service IP actually load-balance across ever-changing pod IPs.
- **VPC CNI (`amazon-vpc-cni-k8s`)** — the network plugin actually assigning IP addresses to pods (see below) — this is the piece that makes EKS's networking model fundamentally different from most other Kubernetes distributions.

## The EKS networking model — VPC CNI's core idea

**Standard Kubernetes CNI plugins (Calico, Flannel, Cilium in overlay mode) create a separate, overlay pod network** — pods get IPs from a distinct CIDR range that doesn't exist in the underlying physical/VPC network, and traffic between pods on different nodes is encapsulated (VXLAN or similar) and decapsulated at the destination node — the underlying network infrastructure has no idea pod IPs exist.

**AWS VPC CNI does something structurally different: every pod gets a real, routable IP address directly from the VPC's CIDR block** — the same address space your EC2 instances, RDS databases, and everything else in the VPC use. Mechanically:
- Each EC2 worker node is attached with multiple **ENIs (Elastic Network Interfaces)**, and each ENI can hold multiple **secondary private IP addresses** (the exact number of ENIs/IPs-per-ENI is instance-type-dependent — larger instances support more, which is why very small instance types can hit surprisingly low pod-density limits).
- The VPC CNI's `ipamd` daemon on each node pre-allocates a "warm pool" of these secondary IPs, so pod scheduling can assign an IP instantly rather than waiting on an EC2 API call to attach a new ENI/IP on the critical path.
- Because pod IPs are real VPC IPs, they're **natively routable** everywhere in the VPC (and peered VPCs, Transit Gateway-connected networks) without any encapsulation overhead — no VXLAN tunnel, no overlay MTU tax, and pods are directly visible to VPC-native tooling: VPC Flow Logs show real pod-to-pod traffic, Security Groups can (with the "Security Groups for Pods" feature) be attached directly to individual pods, and on-prem/other-VPC systems can route to a pod IP directly without any additional NAT/gateway logic.

**The direct tradeoff this enables, and the cost of it**: this design trades **IP address scarcity** for **network simplicity and performance**. Because every pod consumes a real VPC IP, your pod density per node is bounded by (ENIs per instance type × IPs per ENI) − reserved addresses, not by CPU/memory alone — a genuinely common EKS scaling wall that has nothing to do with compute capacity. AWS's mitigations: **prefix delegation mode** (each secondary "slot" on an ENI can be assigned a /28 IPv4 prefix — 16 IPs — instead of a single IP, multiplying effective pod density per node by roughly 16x) and **custom networking** (assigning pod IPs from a secondary, non-VPC-primary CIDR attached via secondary ENIs, letting you carve out a much larger pod address space than your primary VPC CIDR would otherwise allow without exhausting it).

**Why this is "the interview answer"**: contrasting VPC CNI's "flat, routable, real-IP" model against a typical overlay CNI's "encapsulated, virtual-IP" model is exactly the conceptual distinction interviewers are checking for — plus knowing the direct consequence (IP exhaustion as a real, distinct scaling constraint, and the prefix-delegation/custom-networking mitigations) demonstrates you've actually hit this wall in practice rather than just reading the marketing pitch.

## AWS Load Balancer Controller — the layer above CNI

Distinct from CNI (which handles *pod* networking) — the **AWS Load Balancer Controller** watches `Ingress` and `Service type=LoadBalancer` objects and provisions actual ALBs/NLBs (see Day 47) to front them. Two traffic modes worth knowing:
- **Instance mode** (NLB, targets = node instance ID, traffic hits `NodePort` then gets kube-proxy-routed to a pod, possibly on a different node) — simpler, works with any CNI.
- **IP mode** (ALB or NLB, targets = pod IPs directly) — only possible *because* VPC CNI gives pods real, directly-routable VPC IPs; the load balancer targets pods directly, skipping the kube-proxy/NodePort hop entirely, reducing latency and avoiding an extra network hop. This is the default/recommended mode for ALB Ingress specifically, and it's a direct, concrete consequence of the VPC CNI networking model described above — worth explicitly connecting these two facts in an interview answer.

## Points to Remember

- CoreDNS (cluster DNS, Deployment), kube-proxy (Service-to-pod routing via iptables/IPVS, DaemonSet), and VPC CNI (pod IP assignment) are the three foundational EKS add-ons every cluster needs.
- VPC CNI assigns pods real, routable VPC IP addresses directly from ENIs/secondary IPs — unlike overlay CNIs (Calico/Flannel-style) that use an encapsulated, virtual pod network invisible to the underlying VPC.
- This design trades IP address scarcity (pod density bounded by ENI/IP limits per instance type) for network simplicity/performance (no encapsulation overhead, native VPC tooling visibility, direct routability).
- Prefix delegation (assigning /28 prefixes instead of single IPs per ENI slot) and custom networking (secondary CIDR for pods) are the standard mitigations for VPC CNI's IP exhaustion constraint.
- ALB IP-target-mode routes directly to pod IPs (skipping the kube-proxy/NodePort hop) — only possible because of VPC CNI's flat, routable networking model.

## Common Mistakes

- Not knowing pod density per EKS node is capped by ENI/secondary-IP limits (instance-type dependent), then being confused when a node reports plenty of free CPU/memory but the scheduler still won't place more pods there.
- Assuming EKS networking works like a generic overlay-network Kubernetes cluster (e.g., assuming pod IPs are only cluster-internal) and being surprised that pod IPs are directly visible/routable in VPC Flow Logs and from other VPC-connected systems.
- Forgetting that CoreDNS needs its own Fargate profile on Fargate-only clusters (it's a Deployment, not something that runs "for free" on the control plane) — a common "why is DNS resolution failing" root cause on new Fargate-only clusters.
- Enabling "Security Groups for Pods" or custom networking without understanding the ENI-trunking prerequisites, leading to pods stuck in `ContainerCreating` due to IP/ENI allocation failures with an unhelpful-looking error.
- Treating VPC CNI's IP-per-pod behavior as strictly better in all cases without acknowledging real production clusters do sometimes need prefix delegation or custom networking to avoid exhausting a VPC's CIDR space as pod count scales.
