# Day 32 — Cheatsheet: AWS Core VPC & Networking

## VPC & subnets

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a
aws ec2 describe-vpcs
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-xxx
```

3-tier layout:
```
public  10.0.1.0/24, 10.0.2.0/24   -> route 0.0.0.0/0 -> IGW
app     10.0.11.0/24, 10.0.12.0/24 -> route 0.0.0.0/0 -> NAT GW
data    10.0.21.0/24, 10.0.22.0/24 -> no internet route
```

## Internet Gateway / NAT Gateway

```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-public-a --allocation-id eipalloc-xxx
```
IGW = bidirectional, attached to VPC. NAT GW = outbound-only, lives in a public subnet, needs an EIP.

## Route tables

```bash
aws ec2 create-route-table --vpc-id vpc-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --gateway-id igw-xxx
aws ec2 create-route --route-table-id rtb-xxx --destination-cidr-block 0.0.0.0/0 --nat-gateway-id nat-xxx
aws ec2 associate-route-table --subnet-id subnet-xxx --route-table-id rtb-xxx
aws ec2 describe-route-tables --route-table-ids rtb-xxx
```
Every VPC has an implicit `local` route for its own CIDR — cannot be removed.

## Security Groups (stateful, allow-only, instance-level)

```bash
aws ec2 create-security-group --group-name web-sg --vpc-id vpc-xxx
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-xxx --protocol tcp --port 22 --source-group sg-bastion
aws ec2 revoke-security-group-ingress --group-id sg-xxx --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 describe-security-groups --group-ids sg-xxx
```

## NACLs (stateless, allow+deny, subnet-level, ordered)

```bash
aws ec2 create-network-acl --vpc-id vpc-xxx
aws ec2 create-network-acl-entry --network-acl-id acl-xxx --rule-number 100 \
  --protocol tcp --port-range From=443,To=443 --cidr-block 0.0.0.0/0 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id acl-xxx --rule-number 100 \
  --protocol tcp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow --egress
aws ec2 replace-network-acl-association --association-id aclassoc-xxx --network-acl-id acl-xxx
```
Lowest rule-number match wins. Must explicitly allow the ephemeral port range (1024-65535) outbound or stateless return traffic silently fails.

## SG vs NACL — the interview answer

| | Security Group | NACL |
|---|---|---|
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Scope | Instance/ENI | Subnet |
| Rules | Allow only | Allow + Deny |
| Eval | Union of all allows | Ordered, first match wins |

## VPC Peering / Transit Gateway / PrivateLink

```bash
aws ec2 create-vpc-peering-connection --vpc-id vpc-A --peer-vpc-id vpc-B
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx
# non-transitive: A<->B and B<->C does NOT give A<->C

aws ec2 create-transit-gateway --description "hub"
aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id tgw-xxx --vpc-id vpc-A --subnet-ids subnet-a1

aws ec2 create-vpc-endpoint-service --network-load-balancer-arns arn:...
aws ec2 create-vpc-endpoint --vpc-id vpc-consumer --service-name com.amazonaws.vpce.region.vpce-svc-xxx --vpc-endpoint-type Interface
```

## Flow Logs

```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx \
  --traffic-type ALL --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/flow-logs-role

# record format:
# version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```
