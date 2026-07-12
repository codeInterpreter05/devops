# Day 33 — Lab: AWS IAM Deep Dive

**Goal:** Use the IAM Policy Simulator to test permission boundaries hands-on, then set up Access Analyzer to actually find public/externally-reachable resources — this day's assigned hands-on activity — and personally cause and then remediate a public-bucket scenario.

**Prerequisites:**
- An AWS account (sandbox account strongly recommended — this lab deliberately creates a temporarily public S3 bucket) with IAM admin access.
- AWS CLI configured.

---

### Lab 1 — Build a role, then bound it, then test with the Policy Simulator

1. Create an IAM role `lab-developer` with an identity policy granting broad access: `s3:*`, `ec2:*`, `iam:*` on `*` (deliberately over-permissioned, to demonstrate the boundary's effect).
2. Create a permission boundary policy `DeveloperBoundary` that allows `s3:*` and `ec2:*` but explicitly denies `iam:*`.
3. Attach `DeveloperBoundary` as the permission boundary on `lab-developer` (not as a regular attached policy — use the "Set permissions boundary" action specifically).
4. Open the **IAM Policy Simulator** (policysim.aws.amazon.com) or run:
   ```bash
   aws iam simulate-principal-policy \
     --policy-source-arn arn:aws:iam::ACCOUNT_ID:role/lab-developer \
     --action-names s3:CreateBucket ec2:RunInstances iam:CreateUser
   ```
5. Confirm the simulator reports `s3:CreateBucket` and `ec2:RunInstances` as **allowed**, but `iam:CreateUser` as **denied** — even though the identity policy alone would have allowed all three. Explain in your own words why (the intersection with the permission boundary).

**Success criteria:** You can show, using the Policy Simulator, that a permission boundary caps an identity policy's grant down to the intersection — not by editing the identity policy itself, but by adding the separate boundary.

---

### Lab 2 — Close the self-escalation loophole

1. Add a policy to `lab-developer` allowing `iam:CreateRole`, but scope it with a condition requiring `iam:PermissionsBoundary` to equal the ARN of `DeveloperBoundary`.
2. Using the Policy Simulator (or a real `aws iam create-role` call, if your sandbox allows it), test creating a role **without** specifying that permission boundary — confirm it's denied.
3. Test creating a role **with** the correct boundary attached at creation time — confirm it's allowed.

**Success criteria:** You can explain, from direct testing, how this condition prevents a developer from using `iam:CreateRole` to create an unrestricted role for themselves as an end-run around their own boundary.

---

### Lab 3 — Cause, detect, and fix a public S3 bucket

1. Create an S3 bucket and deliberately attach a public bucket policy:
   ```bash
   aws s3api create-bucket --bucket lab-day33-public-test-$(whoami) --region ap-south-1 \
     --create-bucket-configuration LocationConstraint=ap-south-1
   aws s3api put-bucket-policy --bucket lab-day33-public-test-$(whoami) --policy '{
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow", "Principal": "*",
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::lab-day33-public-test-YOURNAME/*"
     }]
   }'
   ```
2. Enable IAM Access Analyzer (account-level, since Organizations may not be available in a sandbox account):
   ```bash
   aws accessanalyzer create-analyzer --analyzer-name lab-analyzer --type ACCOUNT
   ```
3. Wait a few minutes, then list findings:
   ```bash
   aws accessanalyzer list-findings --analyzer-arn <ARN_FROM_STEP_2>
   ```
4. Confirm a finding appears identifying the bucket as publicly accessible. Read the finding's detail — note what specifically it flags (the `Principal: "*"` statement).
5. Remediate: remove the public bucket policy, and additionally enable S3 Block Public Access on the bucket/account level so this class of mistake is structurally harder to repeat:
   ```bash
   aws s3api delete-bucket-policy --bucket lab-day33-public-test-YOURNAME
   aws s3api put-public-access-block --bucket lab-day33-public-test-YOURNAME \
     --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
   ```
6. Re-run `list-findings` — confirm the finding is now marked resolved/archived.

**Success criteria:** You've personally created a public bucket, had Access Analyzer detect it, and fixed it both at the resource level (removing the policy) and structurally (Block Public Access) — and watched the finding resolve.

---

### Lab 4 — SCP simulation (if you have an Organizations management account) or written exercise

If you have access to an AWS Organizations management account:
1. Create an SCP denying `s3:PutBucketPublicAccessBlock` and `s3:PutAccountPublicAccessBlock` on a test OU, attach it, and confirm (via Policy Simulator or a real test in a member account) that even an account administrator can no longer disable Block Public Access.

If you do not have an Organizations setup available:
1. Write out, in your own words, the exact SCP JSON that would achieve this, and explain specifically why an SCP (rather than a per-account IAM policy) is the correct tool for making this restriction apply to *every* account in the org, including future accounts added later.

**Success criteria:** Either a working, tested SCP, or a written SCP plus a clear explanation of why account-level IAM policies alone can't achieve the same org-wide guarantee.

---

### Cleanup

```bash
aws iam delete-role --role-name lab-developer   # after detaching/deleting its policies and boundary
aws accessanalyzer delete-analyzer --analyzer-name lab-analyzer
aws s3 rm s3://lab-day33-public-test-YOURNAME --recursive
aws s3api delete-bucket --bucket lab-day33-public-test-YOURNAME
```

### Stretch challenge

Set up IAM Identity Center (if you have an Organizations management account available) with at least one Permission Set (e.g., `ReadOnlyAccess`) assigned to a test group, and confirm you can `aws sso login` and assume that access without ever creating a standing IAM user. If Identity Center isn't available in your sandbox, research and write a short comparison of Identity Center Permission Sets vs. manually managing IAM roles per account, focused specifically on what happens operationally when an employee leaves the company under each model.
