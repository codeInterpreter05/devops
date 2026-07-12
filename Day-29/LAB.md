# Day 29 — Lab: Terraform Fundamentals

**Goal:** Provision a real VPC + subnets + EC2 instance + security group entirely in Terraform, and understand exactly what happens in state at each step.

**Prerequisites:**
- An AWS account with an IAM user/role that has permission to create VPC, EC2, and security group resources (a sandbox/free-tier account strongly recommended — this lab creates real billable-adjacent resources, though everything here fits in the free tier if cleaned up).
- AWS CLI installed and configured: `aws configure` (or `aws sso login` if using IAM Identity Center).
- Terraform installed. Recommended: install via `tfenv` so you can pin a version.

```bash
brew install tfenv        # macOS; see RESOURCES.md for other OS install methods
tfenv install latest
tfenv use latest
terraform -version
```

---

### Lab 1 — Initialize a project and understand `init`

1. Create a working directory:
   ```bash
   mkdir -p ~/terraform-day29 && cd ~/terraform-day29
   ```
2. Create `providers.tf`:
   ```hcl
   terraform {
     required_version = ">= 1.5.0"
     required_providers {
       aws = {
         source  = "hashicorp/aws"
         version = "~> 5.0"
       }
     }
   }

   provider "aws" {
     region = "ap-south-1"
   }
   ```
3. Run `terraform init` and observe the `.terraform/` directory and `.terraform.lock.hcl` file it creates.
4. Run `cat .terraform.lock.hcl` — identify the exact provider version resolved.

**Success criteria:** You can explain what `init` downloaded and why `.terraform.lock.hcl` should be committed to git while `.terraform/` should not.

---

### Lab 2 — Variables, locals, and a VPC

1. Create `variables.tf`:
   ```hcl
   variable "vpc_cidr" {
     type    = string
     default = "10.20.0.0/16"
   }

   variable "public_subnet_cidrs" {
     type    = list(string)
     default = ["10.20.1.0/24", "10.20.2.0/24"]
   }

   variable "azs" {
     type    = list(string)
     default = ["ap-south-1a", "ap-south-1b"]
   }
   ```
2. Create `locals.tf`:
   ```hcl
   locals {
     name_prefix = "day29-lab"
   }
   ```
3. Create `main.tf` with the VPC and two subnets:
   ```hcl
   resource "aws_vpc" "main" {
     cidr_block           = var.vpc_cidr
     enable_dns_hostnames = true
     tags = { Name = "${local.name_prefix}-vpc" }
   }

   resource "aws_subnet" "public" {
     count                   = length(var.public_subnet_cidrs)
     vpc_id                  = aws_vpc.main.id
     cidr_block              = var.public_subnet_cidrs[count.index]
     availability_zone       = var.azs[count.index]
     map_public_ip_on_launch = true
     tags = { Name = "${local.name_prefix}-public-${count.index}" }
   }
   ```
4. Run `terraform fmt` then `terraform validate`. Fix any errors before proceeding.
5. Run `terraform plan` and read every line of output — confirm it says "2 to add" (VPC + ... wait, count the subnets too — should be 3 resources: 1 VPC + 2 subnets).

**Success criteria:** `terraform validate` passes, and you can explain in your own words what `count.index` is doing in the subnet resource.

---

### Lab 3 — Security group + EC2 instance, then apply for real

1. Add to `main.tf` a security group allowing SSH from your IP only, and an EC2 instance:
   ```hcl
   data "aws_ami" "amazon_linux" {
     most_recent = true
     owners      = ["amazon"]
     filter {
       name   = "name"
       values = ["al2023-ami-*-x86_64"]
     }
   }

   resource "aws_security_group" "web_sg" {
     name        = "${local.name_prefix}-sg"
     description = "Allow SSH from my IP"
     vpc_id      = aws_vpc.main.id

     ingress {
       from_port   = 22
       to_port     = 22
       protocol    = "tcp"
       cidr_blocks = ["YOUR_IP/32"]   # replace with: curl -s ifconfig.me
     }

     egress {
       from_port   = 0
       to_port     = 0
       protocol    = "-1"
       cidr_blocks = ["0.0.0.0/0"]
     }
   }

   resource "aws_instance" "web" {
     ami                    = data.aws_ami.amazon_linux.id
     instance_type          = "t3.micro"
     subnet_id              = aws_subnet.public[0].id
     vpc_security_group_ids = [aws_security_group.web_sg.id]
     tags = { Name = "${local.name_prefix}-web" }
   }
   ```
2. Add `outputs.tf`:
   ```hcl
   output "vpc_id"     { value = aws_vpc.main.id }
   output "subnet_ids" { value = aws_subnet.public[*].id }
   output "instance_public_ip" { value = aws_instance.web.public_ip }
   ```
3. Run `terraform plan -out=tfplan`, read it fully, then `terraform apply tfplan`.
4. Once applied, run `terraform show` and `terraform output` — confirm the outputs match the AWS console.
5. Open the AWS console and manually change the EC2 instance's `Name` tag. Run `terraform plan` again — observe that Terraform detects the drift and wants to revert it.

**Success criteria:** You have a running EC2 instance you provisioned entirely through Terraform, and you've personally witnessed Terraform detect manual drift.

---

### Lab 4 — Workspaces

1. From the same directory, run `terraform workspace list` — note you're in `default`.
2. Create a new workspace: `terraform workspace new sandbox2`.
3. Run `terraform plan` — notice Terraform now believes there is *nothing* deployed (this workspace has its own empty state), even though the `default` workspace's VPC still exists in AWS.
4. Switch back: `terraform workspace select default`. Run `terraform plan` — confirm it now sees your real resources again.
5. Delete the extra workspace: `terraform workspace select default && terraform workspace delete sandbox2`.

**Success criteria:** You can explain, from direct experience, that workspaces switch *state*, not configuration — and that this is why two workspaces can show completely different `plan` output against the exact same `.tf` files.

---

### Cleanup

```bash
terraform destroy
# confirm with "yes" — verify in the AWS console that the VPC, subnets, SG, and instance are gone
rm -rf .terraform .terraform.lock.hcl terraform.tfstate* tfplan
```

### Stretch challenge

Add a `count`-based second EC2 instance in the second subnet, use `terraform state list` to see both instance addresses, then use `terraform state mv` to rename one resource's local name without destroying/recreating it (e.g., rename `aws_instance.web` to `aws_instance.app`). Confirm with `terraform plan` that this produces **zero** planned changes — proving `state mv` only edits Terraform's bookkeeping, not real infrastructure.
