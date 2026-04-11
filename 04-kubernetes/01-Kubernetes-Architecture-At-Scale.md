# Kubernetes & OpenShift Architecture at Scale

## 1. Staff-Level K8s Philosophy
When discussing Kubernetes/OpenShift (OCP), junior engineers explain how to write a Pod manifest. Staff engineers explain how to run a 1,000-node multi-tenant cluster securely without Etcd buckling under the pressure, while guaranteeing fair resource distribution and workload isolation.

## 2. Control Plane Scaling (Etcd & API Server)
- **The Bottleneck**: Etcd is an MVCC (Multi-Version Concurrency Control) key-value store. It is heavily I/O bound. If Etcd is slow, the API server is slow, and the whole cluster degrades.
- **Staff Action**: 
  - Host Etcd on ultra-fast NVMe SSDs, isolated from the control plane worker components.
  - Disable overly noisy logging plugins or aggressive custom controllers that constantly LIST/WATCH all pods across the cluster (which crushes the API server).
  - Use `kube-apiserver` rate limiting (API Priority and Fairness) so that a faulty custom operator cannot DDOS the cluster's control plane.

## 3. Kubernetes / OpenShift Networking
- **Ingress vs. Service Mesh (Istio)**: 
  - Standard Ingress is fine for basic routing. At scale, use a Service Mesh (Istio/Linkerd) or API Gateways (Envoy/Cilium). Mesh provides mTLS between microservices, advanced traffic shifting (Canary/Circuit breakers), and distributed tracing without modifying application code.
- **OpenShift Router**: OCP uses an HAProxy-based router. For high traffic, the router pods themselves need aggressive pod anti-affinity (to spread across dedicated Infra nodes) and horizontal scaling.

## 4. Multi-Tenancy & Security Context Constraints (SCC)
OpenShift is much stricter out-of-the-box than vanilla K8s. 
- **The Problem**: A team wants to run a daemonset that requires `privileged` access to attach a volume.
- **The Staff Approach**: We do not assign the generic `privileged` SCC to their service account. That compromises the node. Instead, we write a custom SCC or Kubernetes PodSecurityPolicy/PodSecurityAdmission that *only* allows the exact Linux Capabilities required (e.g., `CAP_SYS_ADMIN`), drops everything else, and enforces a read-only root filesystem.

## 5. Workload Eviction, QoS, and OOMKills
Junior engineers just add more RAM. Staff engineers understand Quality of Service (QoS).
- **Burstable vs Guaranteed**: 
  - If a pod sets `requests == limits`, its QoS is **Guaranteed**. It is the last to get evicted in a node memory pressure situation.
  - If a pod sets `requests < limits`, it is **Burstable**. 
- **The noisy neighbor problem**: If the node runs out of memory, the Kubelet kills pods. If you don't aggressively set resource limits, a Java app with a memory leak in Team A will cause Kubelet to kill Team B's stable app.
- **Action**: Enforce `LimitRanges` at the namespace level (or via `ResourceQuotas`) so developers *must* declare resources. Use Mutating Admission Webhooks (like Kyverno or OPA Gatekeeper) to automatically inject baseline limits if they are missing.

## 6. High Availability: Disruption Budgets & Anti-Affinity
- **Pod Disruption Budgets (PDBs)**: Mandatory for all tier-1 services. It explicitly tells the cluster autoscaler or node drainer: "You are allowed to terminate nodes for upgrades, but you CANNOT reduce the available replicas of the Payment service below 2."
- **Pod Anti-Affinity**: PDBs are useless if all 3 replicas are scheduled on the exact same AWS EC2 node, and that node dies. Use `podAntiAffinity` rules to force the Kube-Scheduler to distribute replicas across different Availability Zones (failure domains).
