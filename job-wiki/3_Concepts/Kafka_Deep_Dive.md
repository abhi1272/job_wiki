# Apache Kafka — Core Concepts & Deep Dive

This document details Apache Kafka's distributed logging architecture, message ordering guarantees, consumer group scaling, and fault tolerance. It connects to how you utilized Kafka at Concentrix to decouple document parsing workloads for RAG ingestion.

[← Back to Index](../README.md) | [← Previous: Why Leaving (Interview Prep)](../2_Interview_Prep/Why_Leaving.md) | [Next: LLM & Fine-Tuning](./LLM_Fine_Tuning.md)

---

## 🏗️ Core Kafka Architecture

Kafka is a distributed, partitioned, replicated commit log service.

```text
Producer ──> [ Topic: document-ingestion ]
                  ├── Partition 0 ── [Offset 0, 1, 2...] ──> Consumer Group: Python Workers (Instance 1)
                  ├── Partition 1 ── [Offset 0, 1, 2...] ──> Consumer Group: Python Workers (Instance 2)
                  └── Partition 2 ── [Offset 0, 1, 2...] ──> (Idle or Shared)
```

### 1. Topics and Partitions
* **Topic:** A category/feed name to which messages are published.
* **Partition:** Topics are split into partitions. A partition is an ordered, immutable sequence of messages that is continually appended to (a commit log).
* **Ordering Guarantee:** Kafka only guarantees message ordering *within a single partition*, not across partitions in a topic. If global ordering is required, you must use a single partition (which limits throughput).

### 2. Consumer Groups & Scale Out
* A **Consumer Group** is a set of consumers cooperating to read data from a topic.
* **Partition Assignment:** Each partition is assigned to *exactly one* consumer instance within a group.
  - If you have **3 partitions** and **2 consumers**, one consumer reads from 2 partitions, and the other reads from 1.
  - If you have **3 partitions** and **3 consumers**, each reads from 1 partition.
  - If you have **3 partitions** and **4 consumers**, the 4th consumer remains idle. To scale consumption, you must design topics with sufficient partitions.

### 3. Offsets & Commits
* **Offset:** A unique integer sequential ID assigned to each message within a partition.
* **Auto-Commit vs. Manual Commit:**
  - *Auto-commit (default):* Consumer automatically commits offsets periodically. This can lead to **data loss** if the consumer commits offsets but crashes before completing database writes.
  - *Manual Commit:* In our Python workers, we disabled auto-commit. We committed the offset only *after* the document chunk was parsed, embedded, and written to PostgreSQL, guaranteeing **at-least-once** delivery.

---

## 🔒 Fault Tolerance & Processing Guarantees

### 1. Exactly-Once Semantics (EOS)
Kafka supports exactly-once processing using transactional API hooks:
- The producer writes to multiple partitions within a transaction block.
- The consumer commits offsets within the same transaction. If any step fails, the transaction aborts, and downstream consumers configured with `read_committed` isolation level will not see the aborted messages.

### 2. Dead Letter Queue (DLQ) Pattern
When a consumer encounters a malformed message (poison pill) that it can never parse (e.g., a corrupted PDF format), retrying it endlessly will block the partition pipeline.
* **Our Solution:** We wrap the parsing code in a `try-catch` block. If parsing fails with an unrecoverable validation error, the consumer writes the payload along with metadata logs to a **`document-ingestion-dlq`** topic and commits the original offset to continue processing. We configure a separate monitor dashboard on the DLQ to alert the engineering team.

---

## 🛠️ Concentrix Use Case: Kafka Buffer Ingestion
In the No-Code Agent Builder, when an enterprise client uploaded a 1,000-page document directory, it triggered thousands of page chunks.
* **Decoupled Buffer:** Instead of running the parsing inside the NestJS API server (which would crash due to memory limits), the API writes the job metadata to Kafka's `document-ingestion` topic.
* **Scalable Python Workers:** A pool of Python workers on GKE read from Kafka partitions, processing pages in parallel. Kafka acted as a shock absorber, protecting PostgreSQL databases from ingestion traffic spikes.
