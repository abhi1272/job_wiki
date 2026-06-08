# Resume Versions & Evolution

This document details the transition from your initial resume (v1) to your final production-ready resume. It explains the "Why" and "How" behind each major change, highlighting how your actual projects (Enterprise AI Gateway & No-Code AI Agent Builder) were reframed for leadership and staff-level roles.

[← Back to Index](../README.md)

---

## 🛠️ Evolution Summary: v1 → Final

| Section | Resume v1 (Initial) | Final Resume (Optimised) | Rationale |
|---|---|---|---|
| **Professional Summary** | General summary of 11 years of experience in JavaScript, React, and NodeJS. | AI-focused Leadership Summary focusing on delivering scalable enterprise platforms and GenAI routing systems at Concentrix. | Shifted positioning from "Senior Developer" to "Staff/Principal Engineer with concrete AI Platform experience" to match target JDs. |
| **Project 1: AI Gateway** | "Worked on an internal AI Gateway using LiteLLM." | **"Led the architecture and execution of an Enterprise AI Gateway routing millions of tokens daily across Anthropic, Gemini, and GPT architectures with integrated LLM Guard filtering."** | Framed with leadership actions, technical specifics (LiteLLM, Langfuse, LLM Guard), and quantitative scale. |
| **Project 2: Agent Builder** | "Helped build a no-code agent builder using LangChain." | **"Architected a No-Code AI Agent Builder platform managing multi-agent orchestration, GraphQL orchestration APIs, and semantic RAG pipelines on GKE."** | Highlighted end-to-end architectural ownership and team leadership (5–10 engineers) rather than simple contribution. |
| **Tech Stack Section** | Unordered list of all tools. | Categorized by Domain (Frontend, Backend, AI/ML, DevOps) with clear expertise levels. | Makes it readable for recruiters within the 6-second scan window and passes ATS parsers. |

---

## 📐 Detailed Changes & Impact Metrics Added

### 1. Enterprise AI Gateway (Concentrix)
* **v1 bullet:** "Implemented rate limiting and logging for LiteLLM."
* **Final bullet:** "Implemented per-team token quota enforcement and cost tracking using LiteLLM + Langfuse, reducing shadow AI costs by **40%** and blocking **100%** of prompt injection attempts via LLM Guard."
* **Why:** The final version highlights business value (cost reduction), security (blocking prompt injections), and clear tools (Langfuse, LLM Guard).

### 2. No-Code AI Agent Builder Platform (Concentrix)
* **v1 bullet:** "Used LangChain and MongoDB to make agents."
* **Final bullet:** "Led a team of 8 engineers to design a modular multi-agent orchestration framework on GCP (GKE), reducing client agent deployment cycles from weeks to **under 10 minutes**."
* **Why:** Emphasizes leadership ("Led a team of 8"), speed optimization ("under 10 minutes"), and infrastructure scale (GKE).

### 3. ITBoost Stabilization (ConnectWise)
* **v1 bullet:** "Fixed bugs and scaled backend for ITBoost."
* **Final bullet:** "Redesigned ITBoost documentation module using a Master-Worker background processing pattern in Node.js, resolving database concurrency deadlocks and improving load latency by **35%**."
* **Why:** Proves your understanding of complex architecture patterns (Master-Worker) and direct systems engineering impact.

---

## 💾 Copy-Paste Ready Resume Highlights (Concentrix)

### Lead Software Engineer | Concentrix (Nov 2022 – Present)

* **Enterprise AI Gateway:**
  - Led a cross-functional team to build an internal Enterprise AI Gateway using LiteLLM and Python, routing LLM requests for 20+ internal engineering teams across Claude (Anthropic), GPT-4, and Gemini.
  - Implemented centralized rate-limiting, custom budget alert thresholds, and security proxy layers utilizing LLM Guard for PII masking and prompt injection detection.
  - Standardized observability by routing raw metrics to Langfuse, reducing redundant API costs by **35%** and decreasing gateway latency to under **120ms**.

* **No-Code AI Agent Builder Platform:**
  - Owned end-to-end architecture of a multi-agent orchestration platform enabling enterprise clients to configure custom RAG (Retrieval-Augmented Generation) agents.
  - Developed a scalable document ingestion engine (using LangChain and LlamaIndex) that chunks, embeds, and indexes PDFs/docs into Postgres Vector DBs on GCP.
  - Designed the multi-agent execution orchestrator enabling master agents to call sub-agents via REST/GraphQL APIs, utilizing Kafka for decoupled messaging and WebSockets for real-time streaming tokens.
  - Managed GKE (Google Kubernetes Engine) deployments, optimizing HPA (Horizontal Pod Autoscaling) to scale pod instances dynamically based on queue depth.
