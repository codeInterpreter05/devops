# Day 114 — Cloud Cost Engineering: Compute Purchasing Options

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** FinOps | **Flag:** ⚡ Interview-critical

## Brief

On-Demand pricing is the "list price" of AWS compute — almost nobody who runs meaningful production workloads pays it for their steady-state baseline, because AWS offers three structurally different ways to pay less: commit-to-spend (Savings Plans), commit-to-capacity (Reserved Instances), and accept-interruption (Spot). Picking the wrong mix, or picking none at all, is the single most common reason a "$50k/month bill growing 30% monthly" question shows up in interviews — it's usually solvable in a week with zero architecture changes, purely through purchasing strategy.

## Savings Plans vs. Reserved Instances

Both are 1- or 3-year commitments that trade flexibility for a discount (up to ~72% vs. On-Demand for 3-year, all-upfront). The difference is *what* you're committing to.

**Reserved Instances (RIs)** commit to a specific **instance family + region (+ AZ, if zonal)**. Standard RIs can't change instance family; Convertible RIs can be exchanged for a different configuration but exchanges are clunky and don't cover Savings Plans' flexibility.

**Savings Plans** commit to a **dollar-per-hour spend**, not a specific instance:
- **Compute Savings Plans**: most flexible — applies across instance family, size, OS, region, and even to Fargate/Lambda. Up to ~66% discount.
- **EC2 Instance Savings Plans**: locked to a specific instance family in a specific region (but flexible across size/AZ/OS within that family) — slightly deeper discount (~72%) in exchange for that reduced flexibility.

**Why Savings Plans have mostly superseded RIs for new commitments:** modern infra changes instance types constantly (m5 → m6i → m7i migrations, cross-AZ rebalancing via ASGs, mixed Fargate/EC2 workloads). A Standard RI locked to `m5.2xlarge` in `us-east-1a` becomes dead weight the moment you migrate to `m6i` for better price/performance — you're now paying for an idle commitment *and* paying On-Demand for the new instance type. A Compute Savings Plan just keeps applying automatically regardless of what you migrate to.

**The decision framework:**
1. Do you have genuinely stable, predictable steady-state compute (a baseline that doesn't change month to month)? → Buy a **Compute Savings Plan** sized to ~70-80% of your *floor* usage (never 100% — leave headroom for On-Demand/Spot to absorb variability without over-committing).
2. Do you need capacity *reservation* guarantees (e.g., a specific instance type must be available at launch time, not just discounted)? → **RIs** (or **Capacity Reservations**, which are a separate mechanism purely for capacity guarantee, purchasable with or without a pricing discount).
3. Never commit based on peak usage — commit based on your *sustained minimum* over the last 3-6 months, verified in Cost Explorer's RI/Savings Plan recommendations (Billing → Savings Plans → Recommendations), which AWS calculates from your actual usage history.

## Spot instances: strategy, not just "cheaper"

Spot gives up to ~90% discount vs. On-Demand in exchange for AWS being able to reclaim the instance with a **2-minute warning** (via a Spot interruption notice in the instance metadata, `/latest/meta-data/spot/instance-action`) whenever it needs the capacity back for On-Demand customers or the Spot price exceeds your bid.

**Where Spot is appropriate:**
- Stateless, horizontally-scaled workloads (web tier behind a load balancer, batch/ETL jobs, CI runners, ML training checkpointed regularly).
- Anything an orchestrator can reschedule automatically (Kubernetes pods via Karpenter/Cluster Autoscaler, ECS tasks, ASG-managed fleets).

**Where Spot is a mistake:**
- Stateful singletons (a primary database, a Kafka broker without proper replication awareness, anything holding in-memory state with no checkpoint).
- Latency-critical request paths with no graceful degradation if capacity briefly shrinks.

**Making Spot actually reliable in production (this is the part people skip):**
- **Diversify instance types and AZs** — a Spot request pinned to one instance type in one AZ has a much higher interruption rate than a fleet spread across 5-10 compatible instance types across 3 AZs (different types get reclaimed at different times, since Spot capacity pools are independent per type+AZ).
- Use **Spot placement score / capacity-optimized allocation strategy** (in EC2 Fleet / ASG / Karpenter) instead of "lowest price" allocation — lowest-price chases the cheapest pool, which is often cheap *because* it's about to be reclaimed.
- Handle the **2-minute interruption notice** in your application/orchestrator — e.g., Kubernetes' `node-termination-handler` cordons and drains the node gracefully instead of hard-killing pods.
- Mix Spot with On-Demand/Savings-Plan-covered baseline: e.g., an EKS node group split so critical control-plane-adjacent pods run on On-Demand nodes and stateless workers run on Spot — never 100% Spot for anything customer-facing unless you've proven graceful degradation under real interruption load.

## Points to Remember

- Savings Plans commit to **$/hour spend**, RIs commit to a **specific instance configuration** — Savings Plans are more resilient to instance-type migrations and are the default recommendation for new commitments today.
- Size commitments to your **sustained minimum usage**, not peak — use AWS's own Savings Plans/RI recommendations (built from real usage history) rather than guessing.
- Convertible RIs and Capacity Reservations solve different problems than discount-seeking — Capacity Reservations exist purely to guarantee availability, and can be paired with a Savings Plan for both guarantee *and* discount.
- Spot's 2-minute interruption notice is actionable — orchestrators that ignore it (hard-kill instead of drain) turn a graceful reschedule into a customer-facing outage.
- Diversifying instance types/AZs and using capacity-optimized allocation dramatically reduces Spot interruption *frequency* — Spot reliability is mostly an architecture choice, not luck.

## Common Mistakes

- Buying RIs/Savings Plans sized to *peak* usage "to be safe," then paying for idle commitment most of the month — commitments should track the floor, with On-Demand/Spot absorbing the variable portion above it.
- Locking into Standard RIs for an instance family right before a planned migration to a newer generation, creating stranded, non-refundable commitment.
- Running a single-AZ, single-instance-type Spot fleet and being surprised by high interruption rates, then concluding "Spot is unreliable" instead of diversifying the fleet.
- Putting a stateful primary (database, single-replica cache) on Spot to save cost, then treating the resulting outage as a mysterious infra flake instead of a foreseeable consequence of the purchasing choice.
- Ignoring the Spot interruption notice entirely (no drain logic), so instances just vanish mid-request instead of being cordoned and drained — turns a cost optimization into a reliability regression.
- Forgetting Savings Plans/RIs are **regional commitments** by default (zonal RIs are the exception) — assuming a commitment covers usage in a region you're not actually running in.
