# Idempotency & Distributed Systems Resiliency

## 1. Problem Statement
In distributed systems, networks are unreliable. If a client sends a REST API request (e.g., "Charge User $50"), and a network timeout occurs, the client doesn't know if the server processed the request or not. If the client retries, the user might be charged twice.

## 2. Staff-Level Perspective
Junior engineers assume `HTTP 200 OK` means success and everything else means failure. Staff engineers assume that any request can fail *at any point in its lifecycle*—before hitting the server, during processing, or after processing but before the response reaches the client. Thus, **all mutating endpoints must be idempotent**.

## 3. Implementing Idempotency

### The `Idempotency-Key` Header
1. **Client Action**: The client generates a unique UUID (an `Idempotency-Key`) for the transaction and attaches it to the HTTP header.
2. **API Gateway / First layer**: Intercepts the request and checks distributed storage (e.g., Redis or DynamoDB).
   - *Case 1 (Key not found)*: This is a new request. Record the key with status `IN_PROGRESS`. Forward request to backend.
   - *Case 2 (Key found, status IN_PROGRESS)*: A retry happened while the original request was still running. Return `HTTP 409 Conflict`.
   - *Case 3 (Key found, status COMPLETED)*: We've already done this. Return the *exact same cached response payload* and HTTP status as the original successful request, without touching the backend logic again.

## 4. The "Two-Generals Problem" in Microservices
If Microservice A calls Microservice B, and B times out, A must retry safely.
- **Action**: A must pass an `Idempotency-Key` to B.
- **Saga Pattern / Compensating Transactions**: If A succeeds, B succeeds, but C fails, we must have a mechanism to "undo" A and B. Instead of ACID transactions (which don't scale across microservices), we use compensating events. e.g., if "Book Flight" succeeds but "Book Hotel" fails, emit a "Cancel Flight" event.

## 5. Idempotent Consumers (Kafka/SQS)
Messages can be delivered multiple times (At-Least-Once delivery).
- **The Pitfall**: Processing an event like "Checkout Completed" twice.
- **The Fix**: The consumer must check a database table (e.g., `processed_messages`) using the message's unique `event_id` before processing. Better yet, use a relational database with the `event_id` as the Primary Key. A duplicate insert will raise a Unique Constraint Violation, which the consumer can safely catch and ignore.
