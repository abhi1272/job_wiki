# Interview Cheatsheet (One-Liners & Formulas)

This cheatsheet contains key one-liners, mathematical formulas, and command cheat sheets for quick review right before an interview.

[← Back to Index](../README.md) | [← Previous: Prep Tracker Workbook](../5_Prep_Tracker/Prep_Tracker.md) | [Next: 35-Hour Train Study Plan](./Train_Study_Plan.md)

---

## 🎙️ Core Project One-Liners (Your Anchors)

### 1. Enterprise AI Gateway (Concentrix)
> **"A centralized routing proxy built using LiteLLM, Python, and GKE that routes, secures, and tracks LLM calls across multiple providers, masking PII via LLM Guard and saving Concentrix 35% in API costs."**

### 2. No-Code AI Agent Builder (Concentrix)
> **"An enterprise SaaS platform built on GKE with a NestJS backend and Python workers, enabling clients to visually configure multi-agent execution graphs, stream tokens via WebSockets, and run hybrid RAG search indexes."**

### 3. ITBoost Stabilization (ConnectWise)
> **"Stabilized ConnectWise's ITBoost document processor by refactoring blocking indexing operations into a Node.js Master-Worker thread pattern, eliminating DB deadlocks and dropping latency by 35%."**

---

## 🧮 System Design & AI Formulas

### 1. Approximate Nearest Neighbor (ANN) Search
* RAG Vector database retrieval uses Cosine Distance to evaluate semantic similarity:
  $$\text{Cosine Similarity}(A, B) = \frac{A \cdot B}{\|A\| \|B\|}$$
* In Postgres, `pgvector` index formats:
  - **IVFFlat:** Inverted File Index. Good for quick indexing, but search accuracy drops if clustering changes.
  - **HNSW (Hierarchical Navigable Small World):** Builds a multi-layer graph. High search recall, fast query speeds, but uses more RAM and indexing time.

### 2. Reciprocal Rank Fusion (RRF)
RRF combines search results from multiple search systems (Vector + Keyword BM25) into a unified rank score:
$$RRF\_Score(d \in D) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
*where $k$ is a constant (typically 60), and $r_m(d)$ is the rank of document $d$ in search system $m$.*

### 3. LoRA Rank Parameter Size
$$\text{Trainable Parameters} = r \times (d + k)$$
*where $r$ is the low rank ($r \ll d, k$), $d$ is input dimension, and $k$ is output dimension.*

---

## 🛠️ CLI Command Reference

### Kubernetes Troubleshooting
```bash
# Check pod status and events (look for warnings)
kubectl describe pod <pod-name> -n production

# Read logs of a crashed pod before its restart
kubectl logs <pod-name> --previous -n production

# View CPU/Memory consumption of pods
kubectl top pods -n production
```

### Kafka Operations
```bash
# Inspect topic partitions and consumer offsets
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group python-ingest-group
```

### Database Operations
```bash
# Analyze query execution and check if index is used
EXPLAIN SELECT * FROM users WHERE tenant_id = 123;
```
