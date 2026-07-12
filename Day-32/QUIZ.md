# Day 32 — Quiz: AWS Core VPC & Networking

Try to answer without looking at your notes. Answers are at the bottom.

1. Why is a subnet always confined to a single Availability Zone, and what does that mean for highly-available designs?
2. What actually makes a subnet "public" — is it an inherent property you set on the subnet?
3. What is the implicit route present in every VPC's route table, and why can't it be removed?
4. What's the key difference between an Internet Gateway and a NAT Gateway in terms of traffic direction?
5. What's the difference between "stateful" and "stateless" in the context of Security Groups vs. NACLs, and what's the practical symptom if you forget it while writing NACL rules?
6. Do Security Groups support deny rules? How do you achieve "block this specific IP" if not?
7. In what order are NACL rules evaluated, and why does rule *numbering* matter in a way it doesn't for Security Groups?
8. Why is VPC Peering described as "non-transitive," and what problem does that create as the number of VPCs grows?
9. What specific problem does Transit Gateway solve relative to a full mesh of VPC Peering connections?
10. How is PrivateLink conceptually different from Peering/Transit Gateway in terms of what gets exposed across the VPC boundary?
11. What do VPC Flow Logs actually capture, and what do they NOT capture?
12. **Interview question:** What is the difference between a Security Group and a NACL? Which is stateful?

---

## Answers

1. Subnets are hard-pinned to exactly one AZ by AWS — this is a platform constraint, not a configuration choice. It means any design that wants to survive an AZ outage must use at least two subnets (in two different AZs) per tier; a single subnet, no matter how it's configured, represents a single-AZ point of failure.
2. No — there's no "public" flag on a subnet. A subnet is public only as a *consequence* of its associated route table sending `0.0.0.0/0` traffic to an Internet Gateway (and its instances having public IPs to actually be reachable). The same subnet with a different route table association becomes private.
3. The `local` route for the VPC's own CIDR block, routed to `local` — present automatically in every route table and cannot be removed. It's why any two subnets within the same VPC can reach each other by default (subject to SG/NACL rules) with no manual route configuration.
4. An Internet Gateway provides bidirectional connectivity — both inbound and outbound — and is attached at the VPC level. A NAT Gateway provides outbound-only connectivity for private-subnet resources: they can initiate connections out, but nothing external can initiate a connection in, because NAT only forwards responses to connections the private side already started.
5. Stateful (Security Groups) means the return traffic for an already-permitted connection is automatically allowed — no matching rule needed for the response. Stateless (NACLs) means every direction of traffic must be explicitly allowed independently; a request and its response are treated as two separate, individually-evaluated flows. The practical symptom of forgetting this on a NACL: allowing inbound on a port but forgetting to allow the corresponding ephemeral port range outbound causes connections to appear to "hang" — the request gets in, but the reply can't get back out.
6. No, Security Groups only support allow rules — there's no way to write an explicit deny. To hard-block a specific IP/CIDR regardless of any SG's allow rules, you need a NACL deny rule, since NACLs are the layer that supports both allow and deny.
7. NACL rules are evaluated in ascending rule-number order, and the **first matching rule wins** — evaluation stops there, later rules for the same traffic are never considered. This matters because a low-numbered deny rule can shadow a later, broader allow rule for the same traffic; Security Groups have no equivalent ordering concern since they're a pure union of allow rules with no deny to conflict with.
8. Peering is non-transitive because a peering connection only establishes a direct link between the exact two VPCs involved — there's no automatic routing through an intermediate peered VPC. If A is peered with B, and B is peered with C, A still cannot reach C without its own direct peering connection to C. As VPC count grows, achieving full connectivity via peering alone requires n(n-1)/2 individual connections, which becomes unmanageable quickly (10 VPCs = 45 connections).
9. Transit Gateway replaces the need for a full peering mesh with a hub-and-spoke model: each VPC needs just one attachment to the TGW, and the TGW's own route tables (which can be segmented per attachment) decide reachability — turning an n(n-1)/2 connection-management problem into an n-attachment one, plus giving centralized, segmentable routing policy.
10. Peering and Transit Gateway connect entire *networks* — once connected, either side can potentially route to a broad range of resources in the other, subject to routing/SG/NACL rules. PrivateLink exposes exactly *one specific service* via an interface endpoint (ENI) in the consumer's VPC — the consumer can reach that service and nothing else about the provider's VPC; the provider's internal topology and other resources stay completely invisible to the consumer.
11. Flow Logs capture traffic **metadata**: source/destination IP and port, protocol, packet/byte counts, timestamps, and whether the traffic was ACCEPTed or REJECTed by a Security Group or NACL. They do **not** capture packet payload/content — actual data transferred is not visible in Flow Logs (that requires a different tool, VPC Traffic Mirroring).
12. Strong answer: "A Security Group is a stateful firewall applied at the instance/ENI level — it only supports allow rules, and because it's stateful, return traffic for an already-permitted connection is automatically allowed without a matching outbound rule. A NACL is a stateless firewall applied at the subnet level — it supports both allow and deny rules, evaluated in rule-number order with first-match-wins, and because it's stateless, you have to explicitly allow both the request and its response (including the ephemeral port range) as independent rules. Every packet passes through both: NACL at the subnet boundary, then Security Group at the instance boundary. In practice, most day-to-day access control is done with Security Groups because the stateful, allow-only model is simpler to reason about; NACLs are reserved for coarse subnet-wide guardrails or explicit IP blocklisting, since SGs have no deny rule at all."
