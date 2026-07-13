# Day 117 — Model Serving & Kubeflow: GPU Workload Scheduling

**Phase:** 4 – Advanced/Specialization | **Week:** W20 | **Domain:** MLOps | **Flag:** ⚡ Interview-critical

## Brief

Everything covered so far in Days 116-117 works fine on CPU-only nodes for smaller models — but training and serving large models means scheduling scarce, expensive GPU resources correctly, and Kubernetes' default scheduler wasn't originally designed with GPUs' peculiarities in mind (they aren't subdividable the way CPU/memory are, by default). Getting GPU scheduling wrong wastes the single most expensive line item in most ML infrastructure budgets.

## Why GPUs need special handling in Kubernetes

CPU and memory are natively "requestable" resources the scheduler subdivides fractionally (`500m` CPU, `256Mi` memory). GPUs are not — by default, `nvidia.com/gpu: 1` is an **integer, non-fractional** resource request, because a single physical GPU historically couldn't be safely shared between two arbitrary pods at the kernel/driver level. This has real consequences:

- **A pod requesting `nvidia.com/gpu: 1` gets exclusive access to one whole physical GPU**, even if its workload only uses 20% of that GPU's compute — there's no default fractional sharing, unlike CPU requests.
- The **NVIDIA device plugin** (a DaemonSet running on GPU nodes) is what exposes `nvidia.com/gpu` as a schedulable resource to the Kubernetes scheduler in the first place — without it, the scheduler has no idea GPUs exist on a node.

```yaml
apiVersion: v1
kind: Pod
metadata: {name: gpu-training-job}
spec:
  containers:
    - name: train
      image: my-registry/train-gpu:v1
      resources:
        limits:
          nvidia.com/gpu: 1     # requests == limits required for GPU (no overcommit allowed)
  nodeSelector:
    accelerator: nvidia-a100     # pin to the right GPU node pool/type
```

Note `requests` isn't even shown as a separate field commonly used for GPUs — Kubernetes requires GPU requests and limits to be equal; you cannot request less than you're limited to, because there's no meaningful notion of a GPU "burst" the way there is for CPU.

## GPU node pools on EKS

A GPU-enabled EKS node group uses a GPU-capable instance type (`p4d`, `p5`, `g5`, etc.) with the **NVIDIA AMI** (or a custom AMI with drivers baked in) and the device plugin DaemonSet deployed:

```bash
eksctl create nodegroup --cluster my-cluster --name gpu-nodes \
  --node-type g5.2xlarge --nodes 0 --nodes-min 0 --nodes-max 5 \
  --node-labels accelerator=nvidia-a10g \
  --node-taints nvidia.com/gpu=present:NoSchedule
```

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/main/deployments/static/nvidia-device-plugin.yml
```

**The taint on GPU nodes is deliberate and important**: without it, any regular CPU-only pod that happens to get scheduled onto a GPU node (because it fits resource-wise) will occupy that node's CPU/memory capacity and effectively strand the GPU it's sitting next to, since the scheduler doesn't know or care that a "spare" GPU on that node is going unused by a pod that never asked for one. Tainting GPU nodes (`nvidia.com/gpu=present:NoSchedule`) and having only GPU-requesting pods tolerate that taint keeps GPU nodes reserved exclusively for workloads that actually need the GPU.

```yaml
# GPU-requesting pod must explicitly tolerate the taint
tolerations:
  - {key: nvidia.com/gpu, operator: Exists, effect: NoSchedule}
```

## Karpenter for GPU nodes

The same Karpenter mechanics from Day 115 apply to GPU node provisioning, with GPU-specific requirements:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: {name: gpu-training}
spec:
  template:
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["g5.xlarge", "g5.2xlarge", "g5.4xlarge"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]   # training jobs that checkpoint can tolerate spot; see below
      taints:
        - {key: nvidia.com/gpu, value: "present", effect: NoSchedule}
```

GPU capacity is scarcer and more expensive than general compute — this is precisely where Karpenter's scale-to-zero-when-idle behavior has outsized cost impact: a training job that runs for 3 hours a day shouldn't keep a `p4d.24xlarge` node (multiple dollars per hour) running the other 21 hours. GPU nodes with `consolidateAfter` set aggressively low is one of the highest-ROI cost levers in an ML platform.

**GPU workloads and Spot**: training jobs that checkpoint regularly (saving model state every N steps) can tolerate Spot interruption reasonably well — resume from the last checkpoint on a new node. Long-running training with no checkpointing, or latency-critical real-time inference serving, generally should not run on Spot GPU nodes, since GPU Spot capacity pools are typically smaller and interruption more disruptive to restart from scratch.

## Fractional GPU sharing (advanced, worth knowing exists)

For inference workloads that genuinely don't need a whole GPU (a small model serving low QPS), several mechanisms exist to avoid the "1 pod = 1 whole GPU, even at 20% utilization" waste:

- **NVIDIA Multi-Instance GPU (MIG)** — hardware-level partitioning (available on A100/H100-class GPUs) that splits one physical GPU into multiple fully-isolated instances, each schedulable as its own resource.
- **Time-slicing** (NVIDIA device plugin config) — multiple pods share one GPU by time-slicing compute, without hardware isolation — higher risk of noisy-neighbor contention but works on GPUs without MIG support.
- These are advanced/niche compared to the default 1-GPU-per-pod model, but worth naming in an interview if asked how you'd serve many small models efficiently on expensive GPU hardware — it directly extends the "50 models, independent scaling" theme from the previous file into the GPU-cost dimension.

## Points to Remember

- GPUs are a non-fractional, integer Kubernetes resource by default — `requests` must equal `limits`, and one pod occupying `nvidia.com/gpu: 1` gets the whole physical GPU regardless of actual utilization.
- The NVIDIA device plugin DaemonSet is what makes `nvidia.com/gpu` visible to the scheduler at all — GPU nodes without it simply can't be scheduled onto for GPU workloads.
- Taint GPU nodes and require GPU-requesting pods to tolerate that taint — otherwise ordinary CPU pods can land on GPU nodes and strand the GPU capacity next to them.
- GPU nodes are the highest-cost-per-hour compute in most ML infra budgets — aggressive scale-to-zero (Karpenter `consolidateAfter` tuned low) has outsized ROI here compared to general-purpose compute.
- Spot is viable for checkpointing training jobs but risky for long uninterrupted training runs or latency-critical inference — know which category a given GPU workload falls into before choosing capacity type.

## Common Mistakes

- Requesting `nvidia.com/gpu: 1` for a workload that only needs a fraction of a GPU's compute, without evaluating MIG or time-slicing as alternatives, and then wondering why GPU utilization cluster-wide looks poor despite "everything requesting what it needs."
- Forgetting to taint GPU nodes, resulting in regular CPU-only pods landing on expensive GPU nodes and silently wasting the attached GPU capacity that no scheduled pod is using.
- Leaving GPU node pools running 24/7 with a high `consolidateAfter` (or no autoscaling at all) for training workloads that only actually run a few hours a day — the single most expensive idle-resource mistake in ML infrastructure.
- Running long, non-checkpointed training jobs on Spot GPU capacity and losing all progress on interruption — checkpointing needs to be in place *before* Spot is a safe choice, not assumed as a given.
- Forgetting that GPU `requests` must equal `limits` and trying to set a lower request "to allow overcommit" the way one might for CPU — Kubernetes will reject or ignore this for the default (non-fractional) GPU resource model.
