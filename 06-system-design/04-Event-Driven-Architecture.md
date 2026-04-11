# Event-Driven Architecture (EDA) & Managing Asynchrony

## 1. Problem Statement
Synchronous HTTP microservices create coupled, fragile systems. If Service A calls Service B, which calls Service C, a failure in C brings down A. Also, A must wait for the entire chain to complete.

## 2. Staff-Level Philosophy on EDA
Event-driven systems decouple services, allowing them to scale and fail independently. However, Staff engineers know that **EDA trades temporal coupling for operational complexity**. You solve the "Service C is down" problem, but you introduce the "Where did my message go?" problem.

## 3. The Shift: Choreography vs. Orchestration

### Orchestration (The Conductor)
- A central component (e.g., AWS Step Functions or Netflix Conductor) tells everyone what to do.
- **Pros**: Easy to monitor the state of a workflow (e.g., Order #123 is at step 4). Easy to implement timeouts and retries.
- **Cons**: The Orchestrator becomes a monolith of business logic and a single point of failure.

### Choreography (The Dance)
- Services emit events ("OrderCreated"), and other services react to them without knowing who emitted them.
- **Pros**: Highly decoupled. A new service can start listening to "OrderCreated" without changing the Order service.
- **Cons**: Extremely difficult to debug. You need robust Distributed Tracing (OpenTelemetry) to track a business workflow across multiple event boundaries. 

*Recommendation*: Use Choreography for simple, loosely bound events. Use Orchestration for complex, critical business workflows (like Payment Processing) where explicit state management is mandatory.

## 4. Handling Poison Pills & Dead Letter Queues (DLQ)
- **The Problem**: A malformed message hits your queue. The consumer crashes. The queue redelivers it. The consumer crashes again. This blocks the entire partition/queue.
- **The Solution**: 
  1. Configure max retries (e.g., 3 times).
  2. If it fails, move the message to a Dead Letter Queue (DLQ).
  3. Emit an alert on the DLQ. Engineers can inspect the message, fix the consumer bug, and "replay" the DLQ.

## 5. The Outbox Pattern
- **The Problem**: You need to save to your database AND emit a Kafka event. If you save to the DB, but Kafka is down, your system is inconsistent. (Dual-write problem).
- **The Solution (The Outbox Pattern)**:
  1. Use a database transaction to write the business entity (e.g., `Order`) AND an event record (e.g., `OrderCreated`) into an `outbox` table in the *same* database.
  2. A separate background process (e.g., Debezium CDC - Change Data Capture) reads the `outbox` table and pushes the event to Kafka.
  3. This guarantees At-Least-Once delivery without relying on distributed transactions.
