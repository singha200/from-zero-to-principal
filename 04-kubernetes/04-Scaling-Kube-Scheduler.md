# Scaling and Optimizing the Kube-Scheduler

## 1. The Role of the Kube-Scheduler
The `kube-scheduler` is the control plane component responsible for acting as the cluster's matchmaker. It assigns newly created Pods to Nodes. It operates in two phases:
1. **Filtering (Predicates)**: Which nodes possess the exact criteria to host this pod? (e.g., enough CPU, matching node selectors, tolerations for taints).
2. **Scoring (Priorities)**: Out of the eligible nodes, which one is the *best*? (e.g., which node currently has the least allocated RAM, or which node satisfies image locality best).

## 2. When the Scheduler Becomes the Bottleneck
At 100 nodes, the scheduler is invisible. At 5,000 nodes doing 500 pod placements a second (e.g., a massive CI/CD workload or a spark data-processing job), the default scheduler will buckle.
- **The Symptoms**: Pods stay in a `Pending` state for minutes even when there is plenty of empty compute capacity. The scheduler is mathematically overloaded trying to calculate scores for 5,000 nodes for 500 pods constantly.

## 3. Staff-Level Optimizations for the Scheduler

### A. Tuning PercentageOfNodesToScore
- **The Problem**: By default, K8s attempts to find and score *all* eligible nodes to find the absolute mathematically "best" node. At 5,000 nodes, sorting 5,000 arrays takes too long.
- **The Fix**: You can modify the `kube-scheduler` config to tune `percentageOfNodesToScore`. If set to 10%, once the scheduler finds 500 eligible nodes, it stops searching the rest of the cluster and just picks the best out of that 500. You sacrifice finding the "perfect" node for finding a "good enough" node exponentially faster.

### B. Pod Topology Spread Constraints (Advanced Affinity)
- **The Problem**: `PodAntiAffinity` is binary. If you tell K8s "don't put two API pods on the same node", and you have 5 nodes but 6 API pods, the 6th pod stays `Pending` forever.
- **The Fix**: `topologySpreadConstraints`. This is the modern, scalable way to distribute workloads.
  ```yaml
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
  ```
  - This tells the scheduler: "Try your best to evenly distribute these pods across the 3 AWS Availability Zones (`maxSkew: 1`). But if one AZ is full, `ScheduleAnyway` (don't leave the pod Pending, just put it anywhere so we get the compute power)."

### C. Multiple Schedulers (The Data Science Use Case)
- Sometimes, you have vastly different scheduling needs. 
- Example: You run normal web microservices, but you also run massive Machine Learning jobs that require complex GPU bin-packing (packing as many pods onto a single node as possible to exhaust the GPU, rather than spreading them out).
- **The Fix**: You run a **Custom Scheduler**. 
  - You deploy a second `kube-scheduler` binary into the cluster, configured with entirely different scoring algorithms (e.g., `MostAllocated` instead of `LeastAllocated`).
  - In your Machine Learning Pod manifest, you explicitly specify `schedulerName: custom-ml-scheduler`. 
  - The default scheduler ignores the ML pod, and your custom scheduler handles it.

### D. Scheduler Framework (Plugins)
In modern K8s, creating a custom scheduler from scratch is rare. The Staff-level approach is to use the **Scheduling Framework**.
- It exposes Extension Points (`PreFilter`, `Filter`, `PostFilter`, `Score`).
- You can write a small Go plugin, compile it into the scheduler, and inject custom business logic. Example: "If it's Tuesday, prioritize nodes in the cheaper `us-east-1` region; on Wednesday, prioritize `us-east-2`." You manipulate the `Score` phase directly.
