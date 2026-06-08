# System Design Patterns Cheatsheet

This cheatsheet provides a quick reference guide on when to use specific architectural patterns to resolve scalability, latency, consistency, and reliability challenges.

[← Back to Index](../README.md) | [← Previous: Cloud & Security](./Cloud_Security.md) | [Next: Vertex AI on GCP](./Vertex_AI_GCP.md)

---

## 🛠️ Pattern Decision Matrix

| Challenge | Target Pattern | Core Technology | Primary Use Case |
|---|---|---|---|
| **High Latency / Repeated Reads** | Caching (Cache-Aside) | Redis, Memcached | Storing user session tokens, static configuration settings, or identical LLM query prompts. |
| **System Coupling / Spiky Traffic** | Message Queue / Event Stream | Apache Kafka, RabbitMQ | Offloading large PDF document indexing pipelines from the main API thread to async workers. |
| **Cascading Failures / Slow APIs** | Circuit Breaker | Opossum (Node.js), Hystrix | Stopping request flows immediately if database query connection pools time out. |
| **Distributed Transaction Failure** | Saga Pattern (Orchestration) | Node.js state machine | Reverting database writes and deleting S3 files if an elasticsearch indexing job fails. |
| **Read-Heavy Query Bottlenecks** | CQRS (Command Query Responsibility Segregation) | PostgreSQL + Elasticsearch | Separating database write paths from complex search indices. |

---

## 💡 Pattern Deep Dives & Implementations

### 1. Caching Strategy (Cache-Aside)
* **When to use:** When read operations are significantly higher than write operations, and data changes infrequently.
* **The Logic:**
  1. Application receives request for data.
  2. Checks Redis. If it's a **Cache Hit**, return data immediately.
  3. If it's a **Cache Miss**, read from SQL database, store it in Redis with a Time-To-Live (TTL), and return data.
* **Cache Invalidation:** Ensure you set a short TTL (e.g. 5 minutes) or delete the cache entry explicitly during write operations.

### 2. Message Queues (Kafka) vs. Pub/Sub (RabbitMQ)
* **Message Queue (RabbitMQ):**
  - *Behavior:* Messages are routed to queues, consumed, and then deleted from the queue.
  - *When to use:* Simple task routing where each task must be processed by exactly one worker.
* **Event Stream (Kafka):**
  - *Behavior:* Messages are appended to a log and persisted. Multiple consumer groups can read from the same stream at their own pace.
  - *When to use:* High-throughput log tracing, real-time analytics, or microservices event orchestration.

### 3. Saga Orchestration (ConnectWise Practice)
* **When to use:** Multi-step business transactions crossing multiple database boundaries where ACID cannot be maintained.
* **Implementation Tip:** Keep your orchestrator database stateless. Persist the current execution step index (e.g. `Step 2 completed`) in a database so the orchestrator can resume execution after a crash without corrupting transaction state.
