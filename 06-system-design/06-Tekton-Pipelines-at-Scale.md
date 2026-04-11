# OpenShift Pipelines (Tekton) & Pipeliens-as-Code at Scale

## 1. The Migration from Jenkins to Tekton
Jenkins is centralized, imperative, and runs on a master/slave architecture that often becomes a monolithic single point of failure ("The Jenkins Master is down, nobody can deploy!").
**Tekton (OpenShift Pipelines)** is Cloud-Native. Every pipeline step executes as an isolated Pod/Container in a Kubernetes namespace. No master node. Infinitely scalable.

## 2. Tekton Architectural Primitives
- **Task**: A reusable function (e.g., "git-clone", "buildah-build", "snyk-scan").
- **Pipeline**: A graph connecting Tasks together.
- **PipelineRun**: The actual execution of the Pipeline (which spawns Pods).

## 3. Staff-Level Design: The "Paved Road" Catalog
- **Problem**: 500 teams writing 500 different `Pipeline.yaml` files. If you need to update your Docker registry URL, you have to contact 500 teams to update their code.
- **Solution**: The Platform Team builds a centralized **Tekton Task Catalog**.
- We use **ClusterTasks** (or a shared Git repository of Tasks). Teams are only allowed to reference the centralized tasks. 
- If the security team mandates a new container scanning tool, we update the one central `build-and-push` Task. Automatically, all 500 microservices start using the new scanner on their next PipelineRun. The developers didn't have to change a single line of code.

## 4. Pipelines-as-Code (PAC)
In vanilla Tekton, triggering a pipeline requires messy Webhooks and `EventListener` configurations. OpenShift introduced **Pipelines-as-Code**.
- **How it works**: A developer drops a `.tekton/pipeline.yaml` into their application Git repository.
- When they open a Pull Request, the PAC controller intercepts the webhook from GitHub/GitLab, parses the `.tekton/pipeline.yaml`, and automatically provisions the `PipelineRun` on OpenShift.
- It pushes the status (Pass/Fail) directly back to the GitHub PR UI.
- **Why Staff Engineers love it**: It shifts the CI configuration directly into the developer's repository, providing Git history for CI changes, while still executing securely on the centralized OpenShift cluster.

## 5. Scaling Tekton (The Etcd Threat)
- **The Danger**: Because Tekton represents every single run as Kubernetes objects (`PipelineRun`, `TaskRun`, `Pod`), running 1,000 pipelines a day generates massive amounts of obsolete data.
- K8s keeps these objects in Etcd. Etcd has a hard limit of 8GB (and degrades rapidly before that). If you let Tekton run wild, your cluster's Etcd will fill up, corrupt, and destroy your OpenShift cluster.
- **The Staff Optimization**: 
  - Mandate strict **Tekton retention policies**. Run an automated CronJob (or use the built-in Tekton Pruner) to delete any `PipelineRun` older than 7 days.
  - Export the logs and metadata to a cheaper system (like Splunk / ElasticSearch / AWS S3) *before* pruning them from K8s, so auditability is maintained without crushing the cluster.

## 6. Security (Buildah & SCCs)
- **Docker-in-Docker is dead**: You cannot run a Docker daemon inside an OpenShift container securely. It requires root access.
- **Action**: Use **Buildah** (native to OpenShift Pipelines) or **Kaniko**. Buildah runs completely daemonless and unprivileged, allowing Jenkins-less container image building without compromising OpenShift's strict Security Context Constraints (SCCs).
