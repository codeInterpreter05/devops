# Day 30 — Lab: Terraform Remote State & Modules

**Goal:** Refactor Day 29's flat VPC/EC2 config into reusable modules, and move its state from local disk into an S3 backend with DynamoDB locking.

**Prerequisites:**
- Completed Day 29 lab (or a similar working VPC + EC2 + SG Terraform config) still on disk.
- AWS CLI configured with permission to create S3 buckets, DynamoDB tables, and the VPC/EC2 resources from Day 29.

---

### Lab 1 — Create the remote backend infrastructure

Bootstrap the S3 bucket + DynamoDB table (this one-time setup is often done manually or via a separate tiny "bootstrap" Terraform config, since a config can't create the backend it's about to use).

```bash
aws s3api create-bucket \
  --bucket my-day30-tf-state-$(whoami) \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

aws s3api put-bucket-versioning \
  --bucket my-day30-tf-state-$(whoami) \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket my-day30-tf-state-$(whoami) \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

**Success criteria:** `aws s3api get-bucket-versioning --bucket my-day30-tf-state-$(whoami)` shows `"Status": "Enabled"`, and `aws dynamodb describe-table --table-name terraform-locks` succeeds.

---

### Lab 2 — Migrate Day 29's config to the S3 backend

1. In your Day 29 project directory, add a backend block to `providers.tf`:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "my-day30-tf-state-YOURNAME"
       key            = "day29-vpc/terraform.tfstate"
       region         = "ap-south-1"
       dynamodb_table = "terraform-locks"
       encrypt        = true
     }
   }
   ```
2. Run `terraform init` — Terraform detects the backend change and offers to migrate existing local state. Confirm with `yes`.
3. Verify: `aws s3 ls s3://my-day30-tf-state-YOURNAME/day29-vpc/` should show `terraform.tfstate`.
4. Confirm `terraform.tfstate` no longer exists locally (or is now just a pointer) and `terraform plan` still shows "no changes" (proving state moved correctly, not that Terraform lost track of your infra).

**Success criteria:** State lives in S3, `terraform plan` reports no drift, and the local `terraform.tfstate` file is gone.

---

### Lab 3 — Refactor into `vpc` and `ec2` modules

1. Create `modules/vpc/{main.tf,variables.tf,outputs.tf}` — move the `aws_vpc` and `aws_subnet` resources there, parameterized by `cidr_block`, `public_subnet_cidrs`, `azs`. Output `vpc_id` and `subnet_ids`.
2. Create `modules/ec2/{main.tf,variables.tf,outputs.tf}` — move the security group, AMI data source, and `aws_instance` there, parameterized by `vpc_id`, `subnet_id`, `allowed_ssh_cidr`. Output `instance_id` and `public_ip`.
3. Rewrite the root `main.tf` to call both modules:
   ```hcl
   module "vpc" {
     source              = "./modules/vpc"
     cidr_block          = var.vpc_cidr
     public_subnet_cidrs = var.public_subnet_cidrs
     azs                 = var.azs
   }

   module "ec2" {
     source           = "./modules/ec2"
     vpc_id           = module.vpc.vpc_id
     subnet_id        = module.vpc.subnet_ids[0]
     allowed_ssh_cidr = "YOUR_IP/32"
   }
   ```
4. Run `terraform plan`. **Read it carefully** — because resource *addresses* changed (`aws_vpc.main` is now `module.vpc.aws_vpc.main`), Terraform will likely propose destroying and recreating everything unless you fix addressing first.
5. Before applying, use `terraform state mv` to re-point existing state at the new module addresses instead of destroying/recreating:
   ```bash
   terraform state mv aws_vpc.main module.vpc.aws_vpc.main
   terraform state mv 'aws_subnet.public[0]' 'module.vpc.aws_subnet.public[0]'
   terraform state mv 'aws_subnet.public[1]' 'module.vpc.aws_subnet.public[1]'
   terraform state mv aws_security_group.web_sg module.ec2.aws_security_group.web_sg
   terraform state mv aws_instance.web module.ec2.aws_instance.web
   ```
6. Run `terraform plan` again — it should now report **no changes** (or only cosmetic ones), proving the refactor didn't touch real infrastructure.

**Success criteria:** Same running infrastructure as Day 29, now organized as two reusable modules, with zero resources destroyed/recreated during the refactor.

---

### Lab 4 — Add variable validation

1. Add a `validation` block to the `vpc` module's `cidr_block` variable requiring it to be a valid CIDR (`can(cidrhost(var.cidr_block, 0))`).
2. Add a `validation` block to the `ec2` module's `allowed_ssh_cidr` variable requiring it end in `/32` (a single IP) using a regex.
3. Deliberately pass an invalid value (`cidr_block = "not-a-cidr"`) and run `terraform plan` — confirm Terraform refuses with your custom error message before making any API calls.

**Success criteria:** An invalid input is rejected at `plan` time with a clear, custom error message — not discovered later as an opaque AWS API error.

---

### Cleanup

```bash
terraform destroy
aws dynamodb delete-table --table-name terraform-locks
aws s3 rm s3://my-day30-tf-state-YOURNAME --recursive
aws s3api delete-bucket --bucket my-day30-tf-state-YOURNAME --region ap-south-1
```

### Stretch challenge

Publish the `vpc` module to a separate local git repo (or a real GitHub repo) and change its `source` in the root config to a `git::` URL pinned to a tag (`?ref=v1.0.0`). Bump a variable's default inside the module, tag `v1.1.0`, and observe that your root config's `plan` output doesn't change until you explicitly bump the `ref` — proving version pinning isolates you from upstream module changes.
