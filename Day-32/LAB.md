# Day 32 — Lab: AWS Core VPC & Networking

**Goal:** Design and build a real 3-tier VPC (public/app/data subnets) with proper routing in Terraform, then deliberately break and fix connectivity to build real intuition about Security Groups, NACLs, and Flow Logs.

**Prerequisites:**
- AWS CLI configured, and Terraform installed (from Day 29).
- Comfortable with basic Terraform workflow (`init`/`plan`/`apply`) from Days 29-31.

---

### Lab 1 — The core hands-on activity: build the 3-tier VPC in Terraform

1. Create a new Terraform project and define the VPC and 6 subnets (public/app/data × 2 AZs) as described in `01-README-VPC-Subnets-And-Routing.md`:
   ```hcl
   resource "aws_vpc" "main" {
     cidr_block           = "10.0.0.0/16"
     enable_dns_hostnames = true
     tags = { Name = "day32-vpc" }
   }

   resource "aws_subnet" "public" {
     count                   = 2
     vpc_id                  = aws_vpc.main.id
     cidr_block              = "10.0.${count.index + 1}.0/24"
     availability_zone       = data.aws_availability_zones.available.names[count.index]
     map_public_ip_on_launch = true
     tags = { Name = "day32-public-${count.index}" }
   }

   resource "aws_subnet" "app" {
     count             = 2
     vpc_id            = aws_vpc.main.id
     cidr_block        = "10.0.${count.index + 11}.0/24"
     availability_zone = data.aws_availability_zones.available.names[count.index]
     tags = { Name = "day32-app-${count.index}" }
   }

   resource "aws_subnet" "data" {
     count             = 2
     vpc_id            = aws_vpc.main.id
     cidr_block        = "10.0.${count.index + 21}.0/24"
     availability_zone = data.aws_availability_zones.available.names[count.index]
     tags = { Name = "day32-data-${count.index}" }
   }

   data "aws_availability_zones" "available" { state = "available" }
   ```
2. Add the Internet Gateway, a NAT Gateway (in `public[0]`, with its Elastic IP), and three route tables (public → IGW, app → NAT, data → no internet route) plus the associations tying each subnet to its route table.
3. Run `terraform plan`, review it fully, then `terraform apply`.

**Success criteria:** `aws ec2 describe-route-tables` confirms the public route table has a `0.0.0.0/0 → igw-...` route, the app route table has `0.0.0.0/0 → nat-...`, and the data route table has neither.

---

### Lab 2 — Prove the routing asymmetry with real instances

1. Launch a `t3.micro` in `public[0]` with a public IP, and one in `app[0]` with no public IP. Attach a Security Group to each allowing SSH from your IP and all traffic within the VPC CIDR.
2. SSH into the public instance directly. From there, SSH into the app instance using its private IP (this bastion-hop pattern is standard for reaching private subnets).
3. From the app instance, run `curl -m 5 https://checkip.amazonaws.com` — confirm it succeeds (outbound via NAT Gateway works).
4. From your own machine, try to `curl`/`ping`/SSH directly to the app instance's private IP — confirm it's unreachable (no route from the internet to a private subnet, by design).

**Success criteria:** You've directly observed the NAT Gateway's outbound-only asymmetry: the app instance can initiate outbound connections but cannot be reached from outside.

---

### Lab 3 — Security Groups vs. NACLs, hands-on

1. On the app instance's Security Group, temporarily remove the SSH inbound rule. Confirm you can no longer SSH in from the bastion — this is the SG blocking it.
2. Restore the SG rule. Now create a custom NACL on the `app` subnets that **denies** inbound SSH (port 22) from `10.0.1.0/24` (the bastion's subnet) with a low rule number, and allows everything else with a higher rule number.
3. Try SSH from the bastion to the app instance again — confirm it now fails, even though the Security Group still allows it. This proves NACL deny overrides SG allow (both layers must permit the traffic).
4. Fix the NACL (remove or fix the deny rule, or raise its rule number above a broader allow) and confirm SSH works again.
5. On the same custom NACL, add an inbound allow for port 443 but **forget** the matching outbound rule for the ephemeral port range (1024-65535). Attempt a connection that would need a response through it and observe the "hangs" symptom described in the security groups README.

**Success criteria:** You've personally broken and restored connectivity using both an SG rule and a NACL rule, and observed the stateless-NACL ephemeral-port gotcha firsthand.

---

### Lab 4 — Flow Logs as the diagnostic tool

1. Enable VPC Flow Logs on the whole VPC, destination CloudWatch Logs, traffic type `ALL`.
2. Re-run the blocked SSH attempt from Lab 3 step 3 (re-apply the bad NACL rule temporarily).
3. Query the Flow Logs (CloudWatch Logs Insights or `aws logs filter-log-events`) for entries matching the app instance's ENI and port 22 — find the `REJECT` entry corresponding to your blocked attempt.
4. Explain in your own words: how would you tell, from Flow Logs alone, whether a failed connection was rejected at the network layer versus simply not answered by the application?

**Success criteria:** You can locate a specific REJECT entry in Flow Logs corresponding to a connection attempt you personally made, and articulate the REJECT-vs-no-response distinction.

---

### Cleanup

```bash
terraform destroy
# then confirm manually:
aws ec2 describe-nat-gateways --filter Name=tag:Name,Values=day32-* --query 'NatGateways[].NatGatewayId'
aws ec2 describe-addresses --filters Name=tag:Name,Values=day32-*
```
NAT Gateways and their Elastic IPs are billed hourly — verify via the console that both are actually deleted/released after `destroy`, since a failed destroy step can leave a NAT Gateway (and its EIP) running.

### Stretch challenge

Add a second VPC and set up VPC Peering to the first. Add routes on both sides, then try to reach a third VPC (peered only with the second, not the first) from the first VPC's instance — confirm the non-transitivity described in `03-README-VPC-Connectivity-And-Flow-Logs.md` for yourself, then fix it by either adding a direct peering connection or replacing the peering mesh with a Transit Gateway.
