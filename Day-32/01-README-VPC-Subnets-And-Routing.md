# Day 32 — AWS Core: VPC & Networking: VPCs, Subnets & Routing

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

VPC networking is the AWS topic asked about most consistently in DevOps/SRE interviews, because everything else you deploy — EC2, EKS, RDS, Lambda-in-a-VPC — sits inside one, and a wrong subnet/route table design is the classic "works until you try to actually secure or scale it" mistake. This file covers the actual network plumbing: the VPC container, public vs. private subnets, and how route tables + gateways decide where traffic goes.

This day is split into three files:

1. **This file** — VPC, subnets, route tables, Internet Gateway, NAT Gateway.
2. **[02-README-Security-Groups-And-NACLs.md](02-README-Security-Groups-And-NACLs.md)** — Security Groups vs. NACLs, the classic stateful-vs-stateless interview question.
3. **[03-README-VPC-Connectivity-And-Flow-Logs.md](03-README-VPC-Connectivity-And-Flow-Logs.md)** — VPC Peering vs. Transit Gateway, PrivateLink, and Flow Logs.

## The VPC — your private slice of the AWS network

A **VPC (Virtual Private Cloud)** is an isolated, logically-defined network within an AWS region, defined by a CIDR block (e.g., `10.0.0.0/16`, giving you 65,536 IP addresses). Nothing inside one VPC can talk to anything in another VPC (or the internet) unless you explicitly wire connectivity — this default-deny-by-isolation posture is deliberate; AWS's model is "prove you want connectivity," not "prove you want isolation."

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```

Every AWS account gets a **default VPC** per region with permissive default settings (public subnets, an Internet Gateway already attached) meant for quick experimentation — production workloads should always use a purpose-built VPC with deliberately designed subnets, not the default one.

## Subnets — carving the VPC into AZ-bound pieces

A **subnet** is a range within the VPC's CIDR, pinned to exactly **one Availability Zone** — this is a hard AWS constraint, not a convention, and it's why highly-available designs always span multiple subnets across multiple AZs.

```bash
aws ec2 create-subnet --vpc-id vpc-0abc123 --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a
```

**Public vs. private is not an inherent property of a subnet** — AWS doesn't have a "public subnet" checkbox. A subnet is public *only because* its route table sends `0.0.0.0/0` traffic to an Internet Gateway. This is the single most common point of confusion for people new to VPCs: "public" and "private" are a *consequence of routing*, not a subnet attribute you set directly.

### The classic 3-tier layout (exactly what today's hands-on activity builds)

```
VPC 10.0.0.0/16
├── public subnet   10.0.1.0/24  (AZ-a)  — ALB, NAT Gateway, bastion
├── public subnet   10.0.2.0/24  (AZ-b)
├── app subnet      10.0.11.0/24 (AZ-a)  — EC2/EKS nodes, no direct internet inbound
├── app subnet      10.0.12.0/24 (AZ-b)
├── data subnet     10.0.21.0/24 (AZ-a)  — RDS, ElastiCache, no internet route at all
└── data subnet     10.0.22.0/24 (AZ-b)
```

Each tier gets a subnet in *at least two* AZs for availability, and each tier has a progressively more restrictive route table — public subnets route outbound to an IGW, app subnets route outbound through a NAT Gateway (so they can reach the internet for package updates/API calls, but nothing outside can initiate a connection to them), and data subnets typically have **no** route to the internet at all.

## Route tables — the actual decision-maker

A **route table** is an ordered list of CIDR-to-target rules; every subnet is associated with exactly one route table (though one route table can be shared by many subnets). Traffic leaving an instance is matched against the **most specific matching route** (longest prefix match, same principle as regular IP routing) — not the first rule listed.

```
Destination        Target
10.0.0.0/16         local                  (implicit, always present, can't be removed)
0.0.0.0/0            igw-0abc123           (public subnet: route everything else to the internet gateway)
```

```
Destination        Target
10.0.0.0/16         local
0.0.0.0/0            nat-0def456           (private subnet: route everything else through NAT)
```

Every VPC gets an implicit, unremovable `local` route for its own CIDR — this is why instances in different subnets of the same VPC can always reach each other by default (subject to security groups/NACLs), with no explicit route needed.

## Internet Gateway (IGW) vs. NAT Gateway — the direction of travel matters

- **Internet Gateway**: attached to the VPC itself (not a subnet), and provides **bidirectional** internet connectivity — both inbound and outbound. A subnet is "public" because its route table points `0.0.0.0/0` at an IGW, *and* its instances have public IPs to actually be reachable.
- **NAT Gateway**: sits *inside a public subnet*, and provides **outbound-only** internet connectivity for resources in private subnets — a private instance can initiate a connection out (e.g., `yum update`, calling an external API), but nothing on the internet can initiate a connection *in* to it, because the NAT Gateway only translates and forwards traffic for connections it already knows the private side started.

```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-0abc123 --internet-gateway-id igw-0abc123

aws ec2 allocate-address --domain vpc                       # Elastic IP for the NAT Gateway
aws ec2 create-nat-gateway --subnet-id subnet-public-a --allocation-id eipalloc-0abc123
```

This asymmetry (IGW = both directions, NAT = outbound only) is exactly why the 3-tier design works: app-tier instances can reach out to pull updates or call third-party APIs, but nothing external can reach in and start a session with them directly — inbound access to the app tier only ever comes through the load balancer sitting in the public subnet.

**Cost note worth knowing for interviews**: NAT Gateways bill per-hour *and* per-GB processed, and are a genuinely significant line item at scale — a common cost-optimization move is one NAT Gateway per AZ (for availability) rather than one per subnet, or a single shared NAT Gateway for non-production accounts where the reduced availability is an acceptable tradeoff.

## Points to Remember

- A subnet is pinned to exactly one AZ — multi-AZ availability always means multiple subnets, never one subnet spanning AZs.
- "Public" and "private" describe a subnet's **route table**, not an inherent subnet property — there's no such flag on the subnet itself.
- Every VPC has an implicit, unremovable `local` route for its own CIDR, which is why same-VPC traffic always has a path (before security group/NACL rules are considered).
- IGW provides bidirectional internet access at the VPC level; NAT Gateway provides outbound-only access for private subnets and lives inside a public subnet itself.
- Route table matching uses longest-prefix-match, same as standard IP routing — the most specific route wins, not the first one listed.

## Common Mistakes

- Assuming a subnet is "private" just because no one assigned it a public IP range, without checking whether its route table still points `0.0.0.0/0` at an IGW — the routing, not the naming or IP assignment habit, is what actually determines reachability.
- Putting a NAT Gateway in a private subnet (it needs to be in a *public* subnet, since it itself needs a route to the IGW to do its job).
- Designing a "highly available" architecture with only one subnet per tier, not realizing that subnet is pinned to a single AZ and represents a single point of failure.
- Forgetting that a NAT Gateway (unlike the older NAT instance approach) has no built-in cross-AZ failover — each AZ typically needs its own NAT Gateway for true AZ-level resilience, at added cost.
- Not accounting for NAT Gateway data-processing charges when estimating cost for a private-subnet-heavy architecture — it's frequently a bigger and more surprising line item than the compute it's supporting.
