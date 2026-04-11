# OpenTelemetry, Prometheus Alerting, & Grafana

## 1. The Observability Paradigm Shift: OpenTelemetry
At a Staff level, you don't just pick a vendor (Datadog, Dynatrace, New Relic) and install their proprietary agent. Vendors lock you in, and when they raise prices 50%, you are trapped.
- **OpenTelemetry (OTel)** solves this. It is a vendor-agnostic framework for generating, capturing, and exporting telemetry data (Metrics, Logs, Traces).
- **Architecture**:
  - Developers instrument applications using the OTel SDKs (no Datadog or Prometheus libraries required).
  - The OTel Collector runs as a DaemonSet on every Kubernetes node. It receives OTLP (OpenTelemetry Protocol) traffic from the pods.
  - The Collector processes the data (filtering PII, dropping high-cardinality labels) and exports it to *any* backend. You can export Metrics to Prometheus, Traces to Jaeger, and Logs to Loki—or send everything to Datadog if you choose. If you change vendors tomorrow, you just change the OTel exporter config. Zero code changes required for 500+ microservices.

## 2. Alerting Philosophy: Symphony vs. Cacophony 
Junior engineers set alerts for "CPU over 80%." Staff engineers know this destroys on-call morale. Who cares if the CPU is at 90% if the customer is successfully checking out?

### A. Symptom-Based vs. Cause-Based Alerting
- **Cause-Based (Bad)**: Alerting on high memory, CPU, or database I/O. These create alert fatigue. A node running hot isn't inherently an emergency unless it impacts the user.
- **Symptom-Based (Staff Level)**: Alerting on what the customer actually feels. Use the **Four Golden Signals**: Latency, Traffic, Errors, and Saturation.
  - *Example Alert*: "HTTP 500 Error Rate > 2% for 5 minutes."
  - When this fires, the engineer looks at the dashboard to find the *cause* (Oh, the Database CPU is at 100%).

### B. Service Level Objectives (SLOs) and Error Budgets
- **SLI (Service Level Indicator)**: A metric, e.g., "Percentage of HTTP GET /checkout requests completing < 200ms."
- **SLO (Service Level Objective)**: The target, e.g., 99.9%.
- **Alerting on Burn Rate**: Instead of alerting when the Error Rate spikes for 5 minutes, we alert on **Error Budget Burn Rate**. If we are burning our monthly 0.1% error budget so fast that it will be depleted in 4 hours, sound the pager! If it will deplete in 30 days, just create a Jira ticket.

## 3. Prometheus Alertmanager Design
You must design a routing tree in Alertmanager that prevents pager storms.
- **Grouping**: When a network switch fails, 50 databases might go offline, firing 50 alerts. Alertmanager must group these into ONE Slack message: "50 Database Connections Failed."
- **Inhibition**: If a cluster goes down, the `ClusterDown` alert fires. Alertmanager should immediately suppress the 500 subsequent `PodDown` alerts derived from that cluster.
- **Routing**: Send Warnings (e.g., Disk > 80%) to a Slack channel for business hours analysis. Send Criticals (e.g., Symptom-based SLO breaches) directly to PagerDuty to wake someone up.

## 4. Grafana at Scale
- **Anti-Pattern**: Putting 50 graphs on a single dashboard so it takes 15 seconds to load.
- **The "USE" (Utilization, Saturation, Errors) Dashboard**:
  - The top row of any Grafana dashboard should answer: "Is the service healthy right now?" (SLIs, Http Error Rates, Latency percentiles).
  - The rows below are for drilling down: "If not healthy, which node/pod is struggling? Is it memory? Is it upstream API latency?"
- **Grafana as Code**: Do not allow manual dashboard creation in production. Use tooling like **Grizzly** or **Terraform Grafana Provider** to version-control standard dashboards and provision them automatically via GitOps.
