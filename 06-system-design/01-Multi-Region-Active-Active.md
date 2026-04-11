# Active-Active Multi-Region Failover Architecture

## 1. Problem Statement
Design a highly available system that survives the complete loss of an AWS Region (e.g., `us-east-1` goes completely dark) with an RPO (Recovery Point Objective) of < 1 minute and an RTO (Recovery Time Objective) of < 5 minutes.

## 2. Staff-Level Design Philosophy
- Multi-region is **expensive** and **complex**. As a Principal/Staff engineer, my first question is: *Do you really need multi-region?* Have we exhausted multi-AZ within a single region? (Multi-AZ gives 99.99% availability).
- If the business mandate is 99.999% and zero-downtime survival of a regional blast radius, we proceed.

## 3. Architecture Deep Dive

### A. Routing Layer (Global Load Balancing)
- **AWS Route 53** with Latency-based or Geolocation routing.
- **Health Checks**: Deep health checks verifying the entire stack in a region.
- **Active-Active vs Active-Passive**: We will use Active-Active. Both `us-east-1` and `us-west-2` take traffic. This prevents the "cold cache" problem and proves the passive region actually works.

### B. Compute Layer
- Stateless microservices deployed on **EKS (Kubernetes)** or **ECS** in both regions.
- Auto-scaling groups governed by CPU/Memory.
- Idempotency ensures that if a request is retried across regions, it doesn’t create duplicate records.

### C. The Hardest Part: Stateful Data Layer
Data replication is bound by the speed of light. Writing across the US takes ~70ms. 

**Database:**
- Use **Amazon Aurora Global Database** (for relational data) or **DynamoDB Global Tables** (for NoSQL).
- *Aurora Global DB*: Asynchronous replication at the storage layer spanning regions (< 1 sec replication lag).
  - One region is the Writer, the other is a Read Replica.
  - If the Writer region fails, we promote the secondary region to Writer. This takes about ~1 minute (RTO achieved).

**Caching:**
- ElastiCache Redis. Replication across regions is notoriously difficult and usually unnecessary. Cache is local to the region.
- Failover impact: The newly promoted region will suffer a cache miss storm. We must pre-warm the cache or employ rate limiting to protect the database during failover.

### D. Asynchronous Messaging
- **SQS/Kafka**: Messages in a regional queue are trapped if the region fails. 
- *Strategy*: Use stateless consumers. If Region A goes down, Region B handles new traffic. We recover Region A's queued messages after it comes back online. Do not try to sync queues across regions (too complex, leads to split-brain).

## 4. Tradeoffs & Explanations (The "Principal" part)

1. **Split-Brain Problem**: If network link drops between regions, both regions might think they are the primary.
   - *Solution*: Rely on AWS Route 53 Application Recovery Controller and Aurora's built-in failover capabilities rather than writing custom consensus algorithms.
2. **Cost**: Running Active-Active doubles infrastructure costs + cross-region data transfer fees. *Justification*: Required to meet revenue protection SLAs.
3. **Data Loss**: Because replication is asynchronous to avoid massive latency, if Region A crashes instantly, we might lose ~1s of data (RPO achieved). Synchronous replication would mean every write pays the 70ms penalty, killing application performance.
