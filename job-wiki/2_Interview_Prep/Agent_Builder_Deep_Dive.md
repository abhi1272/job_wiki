# Project Deep Dive — No-Code AI Agent Builder Platform

This document details the architecture, orchestration logic, and execution metrics of the **No-Code AI Agent Builder Platform** you owned end-to-end at Concentrix.

[← Back to Index](../README.md) | [← Previous: AI Gateway Deep Dive](./AI_Gateway_Deep_Dive.md) | [Next: Kubernetes Internals](./Kubernetes_Internals.md)

---

## 📋 Project Context & Problem

Enterprise clients wanted to deploy LLM-based autonomous agents to automate processes (customer support, data ingestion, reporting). However:
1. **Engineering Bottlenecks:** Every new agent deployment took engineering weeks of custom coding (LangChain configuration, custom RAG setup, UI building).
2. **Brittle Architectures:** Simple prompt engineering failed on complex tasks. Agents needed a way to invoke other specialized agents (multi-agent workflows).
3. **No-Code Requirement:** Business teams needed a visual, drag-and-drop React interface to assemble these agents.

---

## 🛠️ Technology Stack & Roles

* **Role:** Lead Software Engineer & End-to-End Architect, leading a team of 8 engineers.
* **Frontend:** React, Redux (state), Custom Node-Graph UI (based on React Flow).
* **Orchestration & API Backend:** Node.js with NestJS (GraphQL + REST APIs).
* **AI Orchestration Workers:** Python, LangChain, LlamaIndex, Claude API, MongoDB (configs).
* **Database & Retrieval:** Cloud SQL PostgreSQL (`pgvector`), MongoDB Atlas.
* **Messaging & Memory:** Apache Kafka, Redis, WebSockets.
* **Infrastructure:** Docker, Kubernetes (GKE), Terraform.

---

## ⚙️ Core Technical Implementation Details

### 1. Multi-Agent Orchestration (Master-Worker Agent Pattern)
To handle complex user tasks, we implemented a hierarchical agent pattern:
* **The Master Agent:** An orchestrator agent that receives the user request.
* **Sub-Agents:** Specialized tool-use agents (e.g., *Database Reader Agent, PDF Summary Agent, Email Sender Agent*).
* **Communication:** Master and sub-agents coordinate state via local execution steps. The Python runner evaluates the output step-by-step using LangChain's ReAct framework, invoking child agents as custom tools via REST requests.

### 2. Retrieval-Augmented Generation (RAG) Ingestion Pipeline
We designed a production-grade ingestion flow:
```text
Document Ingest (PDF/TXT) ──> Apache Kafka ──> Python Worker ──> Semantic Chunking ──> VertexAI Embedding ──> pgvector (Cloud SQL)
```
1. **Semantic Chunking:** Files are processed using a layout-aware document parser, grouping text into logical sections based on headings and tables (instead of simple character limits).
2. **Vectorization:** Generated embeddings using `text-embedding-004` (Vertex AI API).
3. **Storage:** Stored vectors in PostgreSQL using `pgvector`.
4. **Hybrid Retrieval:** To improve accuracy, we combine vector cosine similarity search with lexical search (BM25 keyword search) using reciprocal rank fusion (RRF) at query time.

### 3. Persistent Agent Memory (Redis + WebSockets)
* **Short-Term Memory:** Agent session variables and intermediate agent execution logs are stored in a high-speed Redis cluster, enabling sub-agents to access preceding context within the execution graph.
* **Streaming Feedback:** Real-time token streaming is accomplished by passing a custom streaming callback (SSE) from the Claude API inside the Python runner, pushing chunks through Kafka into the WebSocket service, which streams it to the React client in real time.

---

## 💻 Code Snippet: Python RAG Search & RRF
> This shows our implementation of the Hybrid Search retrieval routing using Python:

```python
from pgvector.django import CosineDistance
from django.db.models import F
import math

def hybrid_retrieval(query_vector, search_query_text, limit=5):
    # 1. Vector Search retrieve (using pgvector Cosine Distance)
    vector_results = DocumentChunk.objects.annotate(
        distance=CosineDistance('embedding', query_vector)
    ).order_by('distance')[:limit * 2]
    
    # 2. Text Search retrieve (BM25 Match)
    text_results = DocumentChunk.objects.filter(
        content__search=search_query_text
    )[:limit * 2]
    
    # 3. Reciprocal Rank Fusion (RRF) Algorithm
    rrf_scores = {}
    k = 60
    
    for rank, doc in enumerate(vector_results):
        rrf_scores[doc.id] = rrf_scores.get(doc.id, 0) + (1.0 / (rank + 1 + k))
        
    for rank, doc in enumerate(text_results):
        rrf_scores[doc.id] = rrf_scores.get(doc.id, 0) + (1.0 / (rank + 1 + k))
        
    sorted_docs = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)[:limit]
    return DocumentChunk.objects.filter(id__in=[doc_id for doc_id, score in sorted_docs])
```

---

## 📈 Quantitative Results

* **Deployment Velocity:** Reduced agent deployment times for enterprise clients from **2-3 weeks** to **under 10 minutes**.
* **System Capacity:** Successfully supported **500+ parallel active agent runs** on GKE with a sub-second UI response latency.
* **RAG Retrieval Quality:** Reached **92% relevance accuracy** (using RAGAS framework evaluations), preventing hallucinations in client support pipelines.
