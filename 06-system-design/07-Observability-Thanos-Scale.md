# Observability at Scale: Prometheus & Thanos

## 1. The Limitation of Vanilla Prometheus
Prometheus is the de facto standard for Kubernetes monitoring, but it has severe limitations as you scale:
- **It is not highly available by default**: If a Prometheus pod dies, you lose data for that window. You can run two identical Prometheus pods (HA pairs), but then you have two separate datasets with no native way to merge them.
- **Short-Term Storage**: Prometheus stores data locally (TSDB) on disk. Retaining 1 year of metrics at High Cardinality (millions of time series) will quickly max out any EBS volume and cause the pod to OOMKill trying to load data into memory.
- **Isolated Clusters**: If you have 10 K8s clusters, you have 10 Prometheuses. Grafana dashboards become a nightmare because you essentially have to look at 10 different dashboards.

## 2. Staff-Level Architecture: Enter Thanos
Thanos transforms Prometheus into a Highly Available, Long-Term Storage, Global View system.

### A. Global View (Thanos Querier & Sidecar)
- **Mechanism**: We inject a `thanos-sidecar` into the Prometheus pod.
- We deploy a central `thanos-query` component.
- **Result**: When a user queries Grafana, Grafana asks `thanos-query`. `thanos-query` reaches out via gRPC to all the `thanos-sidecars` across all 10 clusters simultaneously, aggregates the responses, deduplicates the data (if running Prometheus HA pairs), and returns a single, unified result. You can now write PromQL that sums CPU usage across the entire global fleet.

### B. Infinite Long-Term Storage (Thanos Store Gateway)
- **Mechanism**: The `thanos-sidecar` uploads compacted TSDB blocks from Prometheus to cheap object storage (AWS S3, GCP GCS) every two hours.
- When Grafana requests data from 6 months ago, `thanos-query` asks the `thanos-store` component. `thanos-store` uses an index to efficiently fetch only the needed chunks from S3.
- **Result**: Prometheus only needs a tiny local disk (say, holding 24 hours of data). S3 handles years of data for pennies on the dollar. No more scaling expensive block storage.

### C. Thanos Compactor (Cost Optimization)
Querying 1 year of 15-second interval data from S3 is slow and expensive.
- The `thanos-compactor` runs as a singleton batch job against the S3 bucket. It downsamples old data (e.g., turning 15-second data points into 5-minute data points for data > 30 days old, and 1-hour data points for data > 6 months old). This makes decade-long queries execute immediately.

## 3. High Cardinality (The Silent Killer)
At a Staff level, the biggest observability problem isn't infrastructure; it's developers abusing metric labels.
- **The Crime**: A developer adds a `user_id` label to an HTTP request counter. Since there are 5 million users, they just generated 5 million unique time series per metric. Prometheus OOMKills instantly.
- **The Fix**: 
  - Mandate **OpenTelemetry** collectors at the edge.
  - Implement Drop Rules in Prometheus to filter out high-cardinality labels before they hit the TSDB. 
  - Educate teams: "Metrics (Prometheus) are for aggregation (e.g., total hits). Traces/Logs (Jaeger/Elastic) are for high cardinality tracking (e.g., hits for User 123)."
