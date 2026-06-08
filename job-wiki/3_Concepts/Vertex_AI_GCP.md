# Vertex AI on GCP — Architecture & Integration

This document covers Google Cloud Platform's **Vertex AI** suite. It explains how to interact with Gemini models, use Vector Search for large-scale RAG, and manage secure enterprise authentication. It reflects your GKE/GCP platform experiences at Concentrix.

[← Back to Index](../README.md) | [← Previous: System Design Patterns Cheatsheet](./System_Design_Patterns.md) | [Next: Principal Solution Architect (JD Analysis)](../4_JD_Analysis/Principal_Solution_Architect.md)

---

## 🏗️ Vertex AI Architecture Overview

Vertex AI is Google Cloud's unified developer platform for Generative AI. It allows you to access foundational models, train/fine-tune custom weights, and run vector-based retrieval at scale.

```text
GKE Application Pod
        │
  (Workload Identity OIDC Token)
        ▼
   Vertex AI API
        ├── Gemini API (Model Garden) ──> Text/Multimodal Completion
        └── Vector Search (Matching Engine) ──> ANN Index Search
```

---

## ⚙️ Core Components Explained

### 1. Gemini API & Model Garden
* **Model Garden:** A curated repository of Google models (Gemini 1.5 Pro, Flash) and open-source models (Llama-3, Mistral, Gemma).
* **Usage:** We call Gemini endpoints using the official `@google-cloud/vertexai` SDK. Gemini 1.5 Pro supports a **2-million token context window**, which is highly suitable for large document processing.

### 2. Vertex AI Vector Search (formerly Matching Engine)
* **What:** A fully managed, high-performance vector database that performs Approximate Nearest Neighbor (ANN) search on billions of embeddings.
* **Scale Advantage:** In our Agent Builder, we used `pgvector` for small-to-medium datasets. For large client projects (millions of document chunks), we offloaded vector indexes to Vertex AI Vector Search. It uses Hierarchical Navigable Small World (HNSW) graphs to return matches in under **10ms**.
* **Index Deployment:** Index files (embeddings stored as JSON arrays in Cloud Storage) are deployed to a Vertex AI Index Endpoint.

### 3. Authentication Patterns (Workload Identity)
* **Production Auth:** Never store GCP Service Account JSON keys in application containers.
* **The Pattern:**
  1. We create a Google Cloud Service Account (GSA) with the role **Vertex AI User** (`roles/aiplatform.user`).
  2. We bind this GSA to our Kubernetes Service Account (KSA) using IAM policy bindings.
  3. Inside the NestJS API pod, we initialize the Vertex AI SDK using Google Application Default Credentials (ADC). The SDK automatically retrieves OIDC security tokens from GKE's metadata server to authorize requests.

---

## 💻 Code Snippet: Calling Gemini API on GCP
> This snippet shows how to initialize and call Gemini using Node.js and Workload Identity authentication:

```javascript
const { VertexAI } = require('@google-cloud/vertexai');

// 1. Initialize Vertex AI
// Note: Google SDK automatically looks for credentials via environment variables 
// or GKE workload identity metadata endpoint if run inside the cluster.
const vertex_ai = new VertexAI({
  project: 'concentrix-ai-platform',
  location: 'us-central1'
});

async function generateAgentResponse(systemInstruction, userPrompt) {
  // 2. Load the model from Model Garden
  const generativeModel = vertex_ai.getGenerativeModel({
    model: 'gemini-1.5-flash-001',
    generationConfig: {
      maxOutputTokens: 2048,
      temperature: 0.2,
    },
    systemInstruction: systemInstruction,
  });

  const request = {
    contents: [{ role: 'user', parts: [{ text: userPrompt }] }],
  };

  try {
    const streamingResp = await generativeModel.generateContentStream(request);
    
    // 3. Process the stream
    for await (const item of streamingResp.stream) {
      process.stdout.write(item.candidates[0].content.parts[0].text);
    }
    
    const response = await streamingResp.response;
    return response.candidates[0].content.parts[0].text;
  } catch (error) {
    console.error('Vertex AI API invocation failed:', error);
    throw error;
  }
}
```
---

## 💡 Practical Interview Tip
* If asked about Vertex AI compared to LangChain, explain that **Vertex AI is the infrastructure provider** hosting the model weights and vector indexes, while **LangChain is the developer orchestration library** used to build prompt flows and manage sequential model calls in application code.
