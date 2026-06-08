# Distributed Microservices Design Patterns

This document details key architectural patterns for building resilient, consistent microservices at scale. It draws on patterns you deployed at Concentrix (API Gateway, Redis rate-limiting) and ConnectWise (Saga Pattern and circuit breaking).

[← Back to Index](../README.md) | [← Previous: Kubernetes Internals](./Kubernetes_Internals.md) | [Next: MySQL at Scale](./MySQL_At_Scale.md)

---

## 🛰️ 1. API Gateway Pattern
* **Core Idea:** A single entry point for all client requests. It encapsulates internal microservices, handles routing, rate-limiting, authentication, logging, and protocol translation (e.g., HTTP REST to internal gRPC).
* **Your Project Context:** The **Enterprise AI Gateway** at Concentrix acted as a secure API Gateway. Instead of individual apps calling model providers directly, they hit the gateway. It managed rate-limiting (using Redis) and safety filters (LLM Guard) at the perimeter, abstracting provider credential keys from downstream services.

---

## 🔌 2. Circuit Breaker Pattern
* **Core Idea:** Prevents a cascading failure in a microservice architecture. If a downstream service is slow or failing, the circuit breaker trips, causing all subsequent calls to return immediately with a fallback error response instead of waiting for a timeout and consuming system resources (threads/connection pools).

```text
               Normal State                      Tripped (Open) State
  [Client] ──> [Circuit: Closed] ──> [API]     [Client] ──> [Circuit: Open] ──> [Fail Fast]
```

* **States:**
  - **Closed:** Request is sent to downstream service. Track success/failure ratios.
  - **Open:** Downstream is failing. Request fails fast immediately.
  - **Half-Open:** Periodically sends a test request to see if the downstream service has recovered.
* **Your Experience:** At ConnectWise, we placed circuit breakers (using `opossum` in Node.js) on the database connectors. If PostgreSQL experienced high lock contention and failed to respond within 2 seconds, the circuit tripped, failing request flows immediately to prevent piling up HTTP request threads on the web servers.

---

## 🕸️ 3. Service Mesh (Istio / Linkerd)
* **Core Idea:** A dedicated infrastructure layer added directly to container deployments to handle service-to-service communication.
* **Components:**
  - **Data Plane:** Sidecar proxies (e.g., Envoy) running alongside app containers in a Pod, intercepting all ingress/egress network traffic.
  - **Control Plane:** Manages configuration, TLS certificates, routing rules, and telemetry.
* **Key capabilities:** Provides mutual TLS (mTLS) by default, traffic splitting (canary releases), retry budgets, and distributed request tracing without modifying application code.

---

## 🔄 4. Saga Pattern (Distributed Transactions)

Traditional 2-Phase Commit (2PC) does not scale in microservices because it locks database rows across databases, leading to high latency. Instead, we use the **Saga Pattern** — a sequence of local transactions. Each transaction updates data within a single service. If a step fails, the Saga runs compensating transactions to undo the previous changes.

```text
Saga Orchestrator
       │
       ├─ Step 1: Create Order (Order Service) ──> Success
       ├─ Step 2: Charge Wallet (Wallet Service) ──> FAILED!
       ▼
Compensating Flow
       └─ Compensate Step 1: Cancel Order (Order Service)
```

### Saga Choreography vs. Orchestration

#### Choreography (Event-Driven)
* **How:** Services react to events published on a message broker (e.g., Kafka) without a central coordinator.
* **Pros:** Highly decoupled, simple for small workflows.
* **Cons:** Hard to debug and visualize as the number of services grows.

#### Orchestration (Centralized)
* **How:** A dedicated controller class/service (Orchestrator) manages the state machine, tells services what to execute, and triggers compensation steps if a task fails.
* **Pros:** Clear execution state, easier to troubleshoot.
* **Cons:** Orchestrator acts as a single point of logic definition (must be built to failover gracefully).

### 🛠️ ConnectWise Case Study: Ingestion Consistency
At ConnectWise, we used **Saga Orchestration** to manage document uploads inside the ITBoost modules:
1. **Step 1:** Ingress Service stores raw documents to S3.
2. **Step 2:** Database Metadata Service writes records to PostgreSQL.
3. **Step 3:** ElasticSearch Indexer indexes content.
* **Failure Handling:** If the ElasticSearch Indexer fails (e.g., mapping conflict), the Orchestrator executes compensating steps: deletes the DB record in PostgreSQL and removes the raw file from S3. This ensured 100% data consistency without distributed lock blocks.
