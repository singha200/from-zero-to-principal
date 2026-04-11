# Distributed Rate Limiter at Scale

## 1. Problem Statement
Design a rate limiter for an API cluster (e.g., millions of requests per second). It must prevent abusive traffic, enforce pricing tiers, and introduce minimal latency to valid requests.

## 2. Staff-Level Design Philosophy
- A rate limiter sits in the critical path of *every single request*. If it fails, either the entire site goes down (fail closed), or backend systems get crushed (fail open). 
- We must optimize for ultra-low latency. Disk I/O or distant database calls are unacceptable.

## 3. Algorithms & Tradeoffs

### A. Token Bucket
- **How it works:** Buckets hold tokens. A token is added every `X` seconds up to a max capacity. A request costs 1 token.
- **Tradeoff:** Great for bursty traffic, memory-efficient.

### B. Leaky Bucket
- **How it works:** Requests enter a queue and are processed at a constant rate.
- **Tradeoff:** Smooths out traffic, but bursts of traffic can fill the queue with old requests causing new requests to drop.

### C. Fixed Window vs Sliding Window Log vs Sliding Window Counter
- *Fixed Window*: Resets at the top of the minute. Risk of 2x traffic bursts at the boundary (end of min 1 + start of min 2).
- *Sliding Window Counter*: A hybrid that smooths traffic perfectly and uses minimal memory. Best choice for production.

## 4. Architecture Deep Dive

### Where does it live?
1. **Application Layer**: Too late. The traffic has already consumed server resources.
2. **API Gateway / Edge**: The ideal spot. Implement it in NGINX, Envoy, or AWS API Gateway, keeping garbage traffic completely off our compute clusters.

### Distributed Storage (The hard part)
If we run multiple API gateways, they need a central truth to know how many requests user A has made.
- **Centralized DB (Redis)**: 
  - All nodes query Redis to check/update the count.
  - **Problem**: Redis becomes a bottleneck. Network latency added to every request.
  - **Concurrency Issue**: `GET` and `SET` operations from multiple gateways can overwrite each other (race condition).
  - *Fix*: Use Lua scripts in Redis to make `GET` and `SET` atomic, or use Redis's `INCR`.

### Local Caching & Asynchronous Sync (Staff Level Optimization)
- Querying Redis for *every* request is too slow.
- **Solution**: Each API gateway node maintains a local, somewhat inaccurate count in memory. 
- It syncs this count asynchronously with the central Redis server every few seconds.
- **Tradeoff**: We tolerate a relaxed rate limit (allowing a few extra requests) in exchange for incredibly fast `P99` latency. This avoids the "Redis bottleneck."

## 5. Failure Modes
- "What if Redis goes down?"
  - **Fail Open vs Fail Closed**: We should **Fail Open**. It is better to let some malicious traffic through to the backend than to block 100% of legitimate paying customers. Our backend autoscaling will absorb the hit until the rate limiter recovers.
