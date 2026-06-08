# Production RAG Pipeline Deep Dive

This document outlines the architecture, data processing, chunking strategies, and evaluation frameworks for a production-grade Retrieval-Augmented Generation (RAG) pipeline. It represents the pipeline you designed for the No-Code AI Agent Builder at Concentrix.

[← Back to Index](../README.md) | [← Previous: Node.js Event Loop & Scaling](./Node_Event_Loop.md) | [Next: Multi-Agent Orchestration](./Multi_Agent_Orchestration.md)

---

## 🏗️ The 6 Steps of RAG Architecture

```text
1. Ingest ──> 2. Chunk ──> 3. Embed ──> 4. Store (Postgres Vector)
                                             │
   5. Retrieve (Vector Match + BM25) <───────┘
         │
         ▼
   6. Generate (LLM Completion)
```

### 1. Ingest
* **Action:** Extract raw text from varied document formats (PDFs, DOCX, CSVs, markdown).
* **Challenge:** Scan PDFs with headers, footers, columns, or embedded tables without corrupting the reading order.
* **Your Stack:** We used a Python worker library (`PyMuPDF` and `Unstructured`) to extract layout-aware text coordinates, separating paragraphs cleanly.

### 2. Chunk
* **Action:** Break the continuous document text stream into smaller, self-contained segments.
* **Why:** LLMs have finite context windows. Also, sending a whole document is expensive and dilutes the model's attention (needle-in-a-haystack problem).

### 3. Embed
* **Action:** Run chunked text through an embedding model (e.g., `text-embedding-004` on GCP Vertex AI) to get a high-dimensional vector representation (e.g., 768 or 1536 float values). This vector represents the semantic meaning of the chunk.

### 4. Store
* **Action:** Index the vector representations in a database designed for vector similarity search.
* **Your Stack:** We stored chunks in **Cloud SQL PostgreSQL** utilizing the **`pgvector`** extension, building an HNSW (Hierarchical Navigable Small World) index for fast lookup times under scale.

### 5. Retrieve
* **Action:** When a user asks a question, embed their query using the same embedding model. Query the Vector Database for the top K chunks closest to the query embedding (using Cosine Similarity).
* **Hybrid Retrieval:** To improve recall, we combined vector search with keyword-based lexical search (BM25) and unified the rankings using Reciprocal Rank Fusion (RRF).

### 6. Generate
* **Action:** Format the retrieved chunks and the user's original query into a prompt template and send it to the generator LLM (e.g., Anthropic Claude).
* **Example Prompt:**
  ```text
  You are a helpful assistant. Answer the user question using only the context provided below.
  Context:
  --------------
  {retrieved_chunks}
  --------------
  Question: {user_query}
  Answer:
  ```

---

## 📐 Chunking Strategies Explained

Choosing the right chunk size directly impacts search quality and generation accuracy:

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Character-based** | Splits text at fixed character lengths (e.g., 500 characters) with overlap. | Simple, fast, low computational overhead. | Often splits sentences in half, losing context. |
| **Sliding Window** | Token-based sliding window (e.g., 256 tokens chunk size with 64 tokens overlap). | Ensures overlapping context between boundary chunks. | High index duplication; redundant vectors. |
| **Semantic / Layout-aware** | Parses document structure, splitting chunks at headings, tables, or natural paragraph boundaries. | Maintains complete contextual integrity of sections. | Requires layout-aware document extraction tooling. |

---

## 📈 RAG Evaluation Quality Metrics (RAGAS Framework)

To measure the accuracy of our RAG pipeline, we integrated the **RAGAS** evaluation suite to calculate metrics:

```text
                   ┌─────────────── Context ───────────────┐
                   │                                       │
            (Faithfulness)                         (Context Recall)
                   │                                       │
                   ▼                                       ▼
  Query ──(Answer Relevance)──> Response <── (Context Precision)
```

1. **Faithfulness (Groundedness):** Measures if the generated response is based *only* on the retrieved context (no hallucinations).
   - *How:* An LLM evaluator extracts statements from the response and checks if each statement is supported by the context.
2. **Answer Relevance:** Measures if the response directly addresses the user's query.
3. **Context Recall:** Measures if all necessary information to answer the question was successfully retrieved.
4. **Context Precision:** Measures if the retrieved chunks were ranked correctly (relevant chunks placed at the top).

* **Concentrix Results:** By tuning semantic chunk sizes (768 token limit, 128 overlap) and leveraging RRF-based Hybrid Search, we improved our RAG faithfulness score from **74%** to **92%** across customer evaluation datasets.
