# Managing Stateful Workloads Across Multiple Availability Zones

## 1. The Core Infrastructure Problem: EBS is Zonal
At the Staff/Principal level, you must understand the interplay between Kubernetes scheduling and cloud infrastructure. 
- **The Issue**: A Stateless Pod (Nginx) can be scheduled on any node in any Availability Zone (AZ). But a Stateful Pod (Postgres) using an AWS EBS (Elastic Block Store) volume cannot. **EBS volumes are locked to a specific AZ** (e.g., `us-east-1a`).
- If the EC2 node running the Postgres Pod in `us-east-1a` crashes, Kubelet will try to reschedule the Pod. If the cluster autoscaler spins up a new node in `us-east-1b` to host it, the Pod will fail to mount the volume. It will be stuck in a `ContainerCreating` or `Pending` state indefinitely because `us-east-1b` cannot attach a `us-east-1a` EBS volume.

## 2. Staff-Level Solutions to the Zonal Problem

### A. Topology-Aware Volume Provisioning (WaitForFirstConsumer)
- **Junior Mistake**: Creating a PersistentVolumeClaim (PVC) with `volumeBindingMode: Immediate`. The volume gets created in a random AZ before the Pod is scheduled. The Kube-Scheduler might then try to place the Pod in a different AZ, causing a mismatch.
- **Staff Solution**: Use `volumeBindingMode: WaitForFirstConsumer` in the StorageClass. This tells the CSI (Container Storage Interface) driver to wait until the Kube-Scheduler selects a Node for the Pod. Once the Node (and its AZ) is chosen, the EBS volume is provisioned dynamically in that *exact same AZ*.

### B. StatefulSets and Pod Anti-Affinity
If you are running a 3-node MongoDB cluster, you want high availability.
- **Action**: You must force the 3 Pods into 3 *different* AZs.
- **How**: Use `podAntiAffinity` paired with `topologyKey: topology.kubernetes.io/zone`.
- When the first Pod goes to `us-east-1a`, its PVC is created in `1a`. K8s is forced to schedule the second Pod in `us-east-1b` (creating the second PVC in `1b`), and the third Pod in `us-east-1c`. If an entire AWS datacenter burns down, your MongoDB cluster survives.

### C. The EFS Escape Hatch (ReadWriteMany)
- **The Problem**: What if you have a legacy CMS (like WordPress) that requires 5 pods across 3 AZs to read and write to the *exact same underlying network drive*? EBS only supports `ReadWriteOnce` (one node at a time).
- **The Solution**: Use Amazon EFS (Elastic File System). EFS spans the entire Region. You install the EFS CSI driver, create a StorageClass, and your PVC uses `accessModes: [ReadWriteMany]`. 
- **The Tradeoff**: EFS network latency is significantly higher than EBS block storage. Staff engineers never recommend EFS for high-IOPS databases, only for shared media/config files.

## 3. Storage Replication (The Ultimate High Availability)
Kubernetes doesn't replicate data across AZs; the database does.
- Even with PVCs cleanly attached, if `us-east-1a` goes down, that EBS volume is completely inaccessible until the AZ recovers. K8s *cannot* move that data to `1b`.
- **The Staff Design**: Rely on **Application-Layer Replication**. 
  - Install a database Operator (e.g., CrunchyData Postgres). The Operator spins up a Primary DB in AZ 1 and Replicas in AZ 2 and AZ 3. 
  - The Operator manages streaming replication.
  - If AZ 1 dies, the EBS volume dies with it. The Kubernetes Operator detects the failure, promotes the Replica in AZ 2 to Primary, and updates the K8s Service routing. No human intervention needed.
