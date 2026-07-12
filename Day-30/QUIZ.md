# Day 30 — Quiz: Terraform Remote State & Modules

Try to answer without looking at your notes. Answers are at the bottom.

1. What two distinct problems does the S3 + DynamoDB backend pattern solve, and which service solves which?
2. What is the one required attribute schema for a DynamoDB table used as a Terraform lock table?
3. What happens if two engineers run `terraform apply` at the same moment against the same S3 backend with no lock table configured?
4. What does `terraform_remote_state` actually give you access to, and why is that a security consideration when sharing across teams?
5. What is the difference between a root module and a child module?
6. How does a module's `variable` block differ from its `output` block, in terms of direction of data flow?
7. Why should you pin a git- or registry-sourced module to a specific tag/version?
8. What's the difference between what a variable's `type` constraint catches versus what a `validation` block catches?
9. How does Terraform decide the order to create (and later destroy) resources that live in different modules?
10. When is `depends_on` necessary, given that most dependencies are inferred automatically?
11. Name three things Terraform Cloud provides beyond "just storing state remotely."
12. **Interview question:** How do you safely share Terraform state between team members and prevent concurrent runs?

---

## Answers

1. S3 (or any object storage backend) solves **durability and sharing** — state lives in one centrally accessible place instead of a laptop. DynamoDB solves **locking** — it prevents two `apply`s from running concurrently against the same state and corrupting it. They're separate concerns: you can have remote storage without locking (still fixes sharing, not concurrency safety).
2. A single string-type primary key named `LockID` (hash key). Terraform manages all the item's actual content itself.
3. Without a lock, both processes read the same "last known" state, compute their own plans, and both write updated state back at the end — whichever finishes last overwrites the other's state update, silently losing track of resources the earlier apply created or changed (a lost-update / race condition).
4. It gives access to **the entire state file** of the source configuration, not just the specific output being consumed — meaning any consumer of `terraform_remote_state` can technically read every attribute of every resource in that state, including secrets. This is why broad cross-team access to another team's raw state is a real security consideration, and why some orgs prefer narrower data-sharing mechanisms (e.g., SSM Parameter Store) for values that must cross team boundaries.
5. The root module is the directory where you actually run `terraform apply` — the entry point. A child module is any directory of `.tf` files called from elsewhere via a `module` block; its resource names are scoped locally to that module instance, letting the same module be called multiple times without naming collisions.
6. A module's `variable` blocks are its **inputs** — set by whoever calls the module. Its `output` blocks are its **return values** — read by the caller after the module runs, via `module.<name>.<output>`. Inputs flow in, outputs flow out.
7. An unpinned module source (tracking a branch, or a version constraint with no upper bound) means the module's behavior can change between two runs of the exact same root configuration, with no corresponding change visible in your own repo — a silent, moving target that breaks the reproducibility Terraform is supposed to guarantee.
8. A `type` constraint catches **shape** errors — wrong data type entirely (e.g., passing a string where a `number` is declared). A `validation` block catches **value** errors within the correct type — e.g., a syntactically valid string that isn't one of the allowed environment names, or a well-formed but nonsensical CIDR — with a custom, author-defined error message.
9. Terraform builds a dependency graph from attribute references — if one module's input references another module's output, that's an edge in the graph, and Terraform creates the referenced module first. Destroy order is the reverse of create order, derived from the same graph, so dependents are torn down before their dependencies.
10. When a real dependency exists but no attribute reference expresses it — e.g., a Lambda function that must be created after an IAM policy is attached, but whose resource block doesn't reference any attribute of that policy. `depends_on` should be a last resort since it hides the "why" of the dependency from anyone reading the code later.
11. Any three of: remote/managed execution of `plan`/`apply` (not just state storage); built-in locking and versioned state without manually assembling S3+DynamoDB; VCS-triggered runs (PR comments with plan output); Sentinel/OPA policy-as-code gating before apply; team-based access control and a private module registry.
12. Strong answer: "Use a remote backend instead of local state — typically S3 for durable, versioned, encrypted storage, paired with a DynamoDB table (or newer S3-native locking) so Terraform takes a lock before any plan/apply and a second concurrent run fails fast instead of racing and corrupting state. I'd also enable S3 bucket versioning so a bad state write can be rolled back, restrict IAM access to the state bucket/table since state contains plaintext secrets, and — if the team wants managed execution and policy gating on top of just storage — consider Terraform Cloud or a similar tool instead of maintaining the S3+DynamoDB setup by hand."
