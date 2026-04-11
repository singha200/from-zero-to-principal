# Kubernetes & OpenShift Upgrades at Scale

## 1. The Staff Engineer's Perspective on Upgrades
Upgrading a Kubernetes cluster is the ultimate test of your infrastructure's resilience. Junior engineers think upgrades are about clicking "Upgrade" in the AWS console or running `oc adm upgrade`. Staff engineers know upgrades are about **API Deprecation management, stateful workload protection, CSI driver validation, and coordinating node reboots without breaching application SLAs.**

## 2. Vanilla Kubernetes (EKS/AKS/GKE) Upgrade Strategy
In vanilla K8s, the Control Plane and Data Plane (Nodes) are upgraded separately.

### A. API Deprecation (The Silent Killer)
- K8s aggressively deprecates old APIs (e.g., `v1beta1` Ingress -> `v1` Ingress). If you upgrade the Control Plane while workloads still define manifests using `v1beta1`, applying those manifests post-upgrade will fail, breaking your CI/CD pipelines.
- **Action**: Use tools like **Pluto** or **Kube-no-trouble (kubent)** to scan Helm charts and GitOps repositories *before* the upgrade. Fail the CI pipeline if deprecated APIs are detected.

### B. Node Upgrades (The Data Plane)
- In EKS, upgrading managed node groups involves creating a new Auto Scaling Group with the new AMI, cordoning the old nodes, and draining them.
- **Stateful Workloads (The Hard Part)**: Stateless pods (Nginx) move easily. Stateful pods (Database on EBS volume) are bound to Availability Zones and take 5+ minutes to detach/reattach volumes.
- **Action**: 
  - Ensure StatefulSets have strict Pod Disruption Budgets (`maxUnavailable: 1`).
  - Upgrade nodes sequentially or in small batches.
  - Test the Container Storage Interface (CSI) drivers in a lower environment first—a bad CSI driver version means EBS volumes won't mount on the new nodes.

## 3. OpenShift Upgrades (The CVO and MCO)
OpenShift differs significantly because it upgrades the *Operating System (Red Hat Enterprise Linux CoreOS)* alongside the Kubernetes platform.

### A. Cluster Version Operator (CVO)
- OpenShift upgrades are managed via Over-The-Air (OTA) updates using the CVO. The CVO reads the target version and sequentially updates all cluster Operators (Networking, Storage, Monitoring, etc.).
- If *one* core Operator enters a `Degraded` state during the upgrade, the entire upgrade pauses. 

### B. Machine Config Operator (MCO)
- The MCO manages the CoreOS operating system. When an OS update is pulled, the MCO must reboot the nodes.
- **The Process**:
  1. MCO cordons the node.
  2. MCO drains the node (respecting PDBs).
  3. MCO applies the new Ignition config (OS update).
  4. Node reboots.
  5. MCO uncordons the node.
- **Staff Optimization**: If you have 100 worker nodes, upgrading one by one takes ~24 hours. To speed this up, Staff engineers group nodes into custom **MachineConfigPools (MCP)** and increase the `maxUnavailable` setting on the MCP, allowing OpenShift to reboot 5 or 10 nodes simultaneously—slashing the upgrade window while maintaining HaProxy/Ingress High Availability.

## 4. Operator Lifecycle Manager (OLM) Updates
In OpenShift, you install third-party software (e.g., Dynatrace, Couchbase, CrunchyData Postgres) via OLM.
- **The Risk**: Red Hat upgrades the OpenShift platform, but your database Operator isn't compatible with the new OCP version.
- **Action**: Staff engineers lock OLM subscriptions to `Manual` approval rather than `Automatic`. Before an OpenShift upgrade, we map the compatibility matrix of every installed Operator against the target OCP version, upgrade the Operators first, verify health, and *then* upgrade OpenShift.

## 5. Blue/Green Cluster Migrations (The Nuclear Option)
Sometimes (e.g., jumping multiple major versions or migrating networking layers like OpenShift SDN to OVN-Kubernetes), in-place upgrades are too risky.
- **Action**: Build a brand new cluster (Green cluster) alongside the old one (Blue cluster). 
- Use GitOps (ArgoCD) to apply all stateless manifests to the Green cluster.
- Use backup/restore tools like **Velero** or **OADP (OpenShift API for Data Protection)** to snapshot and migrate Persistent Volumes from Blue to Green.
- Shift DNS slowly in Route53 from Blue's Ingress to Green's Ingress.
