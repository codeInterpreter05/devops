# Day 32 — AWS Core: VPC & Networking: Peering, Transit Gateway, PrivateLink & Flow Logs

**Phase:** 1 – Core DevOps | **Week:** W5 | **Domain:** AWS | **Flag:** ⚡ Interview-critical

## Brief

Once you have more than one VPC — which happens fast in any real org (per-environment VPCs, per-team VPCs, a shared-services VPC) — you need a way for them to talk to each other, and a way to expose/consume a service across VPC boundaries without exposing it to the whole internet. This file covers the three connectivity patterns AWS offers for that, plus Flow Logs, the tool you reach for when any of this networking doesn't behave the way the previous two files predicted.

## VPC Peering — direct, 1:1, non-transitive

```bash
aws ec2 create-vpc-peering-connection --vpc-id vpc-A --peer-vpc-id vpc-B
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-0abc123
```

A **peering connection** is a direct, private network link between exactly two VPCs — traffic flows over AWS's backbone, never the public internet, and you add explicit routes in each VPC's route table pointing relevant CIDRs at the peering connection (`pcx-...`) as the target.

**The critical property to remember: peering is non-transitive.** If VPC A is peered with VPC B, and VPC B is peered with VPC C, **A cannot reach C** through B — there is no automatic routing through an intermediate peered VPC, by design (AWS deliberately avoids letting you accidentally create a routing mesh that leaks connectivity you didn't explicitly grant). If A needs to reach C, you need a *direct* A↔C peering connection, or a Transit Gateway.

This non-transitivity is exactly why peering **doesn't scale** past a handful of VPCs — full mesh connectivity between *n* VPCs requires n(n-1)/2 individual peering connections, each with its own route table entries to maintain. 5 VPCs = 10 connections; 10 VPCs = 45. This combinatorial blowup is the direct motivation for Transit Gateway.

## Transit Gateway — a hub-and-spoke router for many VPCs

```bash
aws ec2 create-transit-gateway --description "org-wide hub"
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-0abc123 --vpc-id vpc-A --subnet-ids subnet-a1
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-0abc123 --vpc-id vpc-B --subnet-ids subnet-b1
```

A **Transit Gateway (TGW)** is a managed, regional router that many VPCs (and on-prem networks via VPN/Direct Connect) attach to — instead of a full mesh of peering connections, every VPC has **one** attachment to the TGW, and the TGW's route tables decide what can reach what. Adding a new VPC to the network means one new attachment, not *n* new peering connections.

TGW also supports **route table segmentation** — you can create multiple TGW route tables and associate different VPC attachments with different ones, so (for example) a "shared services" VPC route table lets everything reach it, while a "prod" route table only allows prod VPCs to reach each other and explicitly excludes dev, all within one Transit Gateway rather than needing separate peering topologies.

**When to choose which:**
- **Peering**: 2 VPCs, or a small, stable, unlikely-to-grow number of VPCs — simpler, no hourly TGW charge, no extra managed component.
- **Transit Gateway**: anything beyond a handful of VPCs, or when you need centralized, segmented routing policy (e.g., hub-and-spoke with a shared-services hub) — the added cost (hourly + per-GB) buys you operational sanity as the VPC count grows.

## PrivateLink — expose a *service*, not a *network*

```bash
aws ec2 create-vpc-endpoint-service --network-load-balancer-arns arn:aws:elasticloadbalancing:...
aws ec2 create-vpc-endpoint --vpc-id vpc-consumer --service-name com.amazonaws.vpce.ap-south-1.vpce-svc-0abc123 --vpc-endpoint-type Interface
```

**PrivateLink** is a fundamentally different model from peering/TGW: instead of connecting two *networks* (where each side can potentially route to anything in the other), it exposes **one specific service** behind an ENI (an "interface endpoint") in the consumer's VPC — the consumer can reach *that service* and nothing else about the provider's VPC; the provider's internal network topology, other resources, and CIDR ranges are never exposed to the consumer at all.

This is exactly the model AWS itself uses for many of its managed services (`com.amazonaws.<region>.s3`, `...secretsmanager`, `...ec2` interface endpoints) — letting a private-subnet instance with no internet route still call the S3 or Secrets Manager API without traversing a NAT Gateway or the public internet. It's also the pattern for exposing your *own* internal service (behind an NLB) to another team's VPC, or to a customer's VPC in a SaaS context, **without peering the networks together** — a materially smaller blast radius than "now these two VPCs can route to each other."

**Interview-relevant distinction**: peering/TGW = "these networks can reach each other" (broad); PrivateLink = "that VPC can reach this one specific service" (narrow, service-level). Prefer PrivateLink whenever the actual requirement is "consume a service," not "merge networks."

## VPC Flow Logs — the diagnostic tool for all of the above

```bash
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids vpc-0abc123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flow-logs-role
```

Flow Logs capture metadata about IP traffic going to/from network interfaces in a VPC, subnet, or single ENI — source/dest IP and port, protocol, byte/packet counts, and crucially, whether the traffic was **ACCEPT**ed or **REJECT**ed (by a Security Group or NACL). This is the tool you reach for when "my app can't connect to X and I don't know if it's a Security Group, a NACL, or something else entirely" — a REJECT entry tells you traffic is actually being blocked at the network layer (versus, say, the application never listening on that port, which Flow Logs wouldn't show since the packet is never rejected, just unanswered).

```
<version> <account-id> <interface-id> <srcaddr> <dstaddr> <srcport> <dstport> <protocol> <packets> <bytes> <start> <end> <action> <log-status>
2 123456789012 eni-0abc123 10.0.1.5 10.0.21.10 54321 5432 6 10 4800 1699999000 1699999060 REJECT OK
```

Flow Logs record **metadata, not packet payloads** — you get "who talked to whom, on what port, and was it allowed," not the actual data transferred (that's a packet-capture/mirroring problem, a different AWS feature — VPC Traffic Mirroring). Ship them to CloudWatch Logs for quick `filter` queries, or to S3 for cheaper long-term storage and Athena-based analysis at scale.

## Points to Remember

- VPC Peering is direct and non-transitive — A↔B and B↔C peering does **not** let A reach C; this is the single fact interviewers most often check.
- Transit Gateway replaces the n(n-1)/2 peering-connection blowup with one attachment per VPC, plus supports segmented routing via multiple TGW route tables.
- PrivateLink exposes a single service across a VPC boundary via an ENI, without connecting the two networks at large — narrower blast radius than peering/TGW by design.
- Flow Logs show ACCEPT/REJECT decisions and traffic metadata, not payload content — your first diagnostic step for "is this a network-layer block or something else."
- Choose peering for a couple of stable VPCs, Transit Gateway once you're managing more than a handful or need routing segmentation, and PrivateLink whenever the actual need is "reach one service," not "merge two networks."

## Common Mistakes

- Assuming transitive reachability through a chain of peering connections (A↔B, B↔C implies A↔C) — it does not, and this is a frequent source of "why can't these two VPCs talk" confusion.
- Building a full peering mesh as the VPC count grows past a handful instead of migrating to Transit Gateway, ending up with an unmanageable number of individual peering connections and route table entries.
- Reaching for VPC Peering or Transit Gateway when the actual requirement was "let this other team call my API" — over-provisioning full network connectivity (and its blast radius) when PrivateLink's single-service exposure was the right-sized tool.
- Not enabling Flow Logs until *after* an incident, and then having no historical ACCEPT/REJECT data to diagnose what actually happened during the outage window.
- Expecting Flow Logs to show *why* traffic was rejected (which specific SG/NACL rule) — they show *that* it was rejected; correlating to the specific rule still requires reviewing the SG/NACL configuration directly.
