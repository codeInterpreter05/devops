# Day 32 — Resources: AWS Core VPC & Networking

## Primary (assigned)

- **AWS VPC documentation** (docs.aws.amazon.com/vpc) + **AWS re:Invent VPC networking talks** (YouTube, search "re:Invent VPC deep dive") — the assigned combination: read the official docs for the mechanics, then watch a re:Invent networking deep-dive talk (e.g., any year's "Advanced VPC Design and New Capabilities for Amazon VPC") for the real-world architecture reasoning that docs alone don't convey.

## Deepen your understanding

- **AWS — "Security Groups vs. Network ACLs" documentation page** (docs.aws.amazon.com/vpc/latest/userguide/vpc-security-comparison.html) — the official side-by-side table, worth reading directly since this exact comparison is today's interview question.
- **AWS Whitepaper — "Building a Scalable and Secure Multi-VPC AWS Network Infrastructure"** — covers the Peering → Transit Gateway progression with real architecture diagrams.
- **AWS — "What is AWS PrivateLink?" documentation** — short, focused explanation of the interface-endpoint model and how it differs from peering/TGW.
- **AWS — "Logging IP traffic using VPC Flow Logs" documentation** — full field reference for the Flow Logs record format used in Lab 4.

## Reference / lookup

- **AWS CLI VPC command reference** (`aws ec2 help`, or docs.aws.amazon.com/cli/latest/reference/ec2/) — exact syntax for every command in today's cheatsheet.
- **VPC Reachability Analyzer** (AWS console feature) — automatically traces why two resources can/can't reach each other across SGs, NACLs, and route tables; very useful once you've built the mental model manually in today's lab.

## Practice

- **AWS Skill Builder — "Networking Fundamentals for Amazon VPC" (free digital course)** — structured, free, hands-on practice reinforcing exactly the subnet/routing/gateway concepts from today.
