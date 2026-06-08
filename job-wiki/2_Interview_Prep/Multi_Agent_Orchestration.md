# Multi-Agent Orchestration Frameworks

This document covers system design concepts for multi-agent LLM systems. It represents the orchestration mechanics you built and deployed for the No-Code AI Agent Builder Platform at Concentrix.

[← Back to Index](../README.md) | [← Previous: RAG Pipeline Deep Dive](./RAG_Pipeline_Deep_Dive.md) | [Next: Behavioural Stories](./Behavioural_Stories.md)

---

## 🗺️ Multi-Agent Orchestration Patterns

When tasks grow complex, a single LLM agent with tools fails because the routing logic becomes too broad. We decompose the system into specialized, smaller agents coordinating with each other:

```text
Pattern A: Router Agent          Pattern B: Conversational           Pattern C: Hierarchical
    [Router Agent]                  [Agent A] <───► [Agent B]            [Master Orchestrator]
       │       │                                                            │           │
       ▼       ▼                                                            ▼           ▼
   [Agent 1] [Agent 2]                                                  [Agent 1]   [Agent 2]
```

### 1. Router Agent Pattern
* **How it works:** A single coordinator (the router) evaluates the user's initial prompt and delegates execution to *exactly one* specialized agent (e.g. routing a query to either the "Refund Agent" or "Tech Support Agent").
* **When to use:** Simple classification-based workloads.

### 2. Conversational (Peer-to-Peer) Agent Pattern
* **How it works:** Agents communicate directly with each other as peers, publishing messages to a shared history queue or a message broker.
* **Example:** A "Writer Agent" writes a draft and passes it to an "Editor Agent", which returns critique comments. The "Writer Agent" refines the draft until a stop condition is met.
* **When to use:** Iterative tasks (writing, code reviews, design refinement).

### 3. Hierarchical (Master-Worker) Agent Pattern
* **How it works:** A high-level **Master Orchestrator Agent** splits a complex task into multiple sub-steps. It spawns specialized worker agents (sub-agents) to execute specific sub-tasks, collects their outputs, synthesizes them, and returns the final answer to the client.
* **Concentrix Context:** This was the primary pattern in the No-Code Agent Builder. If a user asked: *"Retrieve quarterly sales data and draft an executive email,"* the Master Agent spawned a **SQL Agent** to pull data, passed the data to an **Analytics Agent** to compile statistics, and finally called a **Copywriting Agent** to draft the email template.

---

## ⚠️ Challenges & Technical Solutions in Production

Deploying multi-agent systems in production introduces unique system engineering challenges:

### 1. Loop Starvation (Infinite Agent Loops)
* **Challenge:** Two agents can get stuck in an execution ping-pong loop (e.g., Agent A requests data, Agent B asks for clarification, repeating infinitely and consuming LLM tokens).
* **Solution:** 
  - **Max Iteration Limit:** Set hard limits in the orchestrator runner code (e.g., maximum of 10 step loops per user invocation).
  - **System Prompt Guardrails:** Explicitly instruct agents in their system prompt: *"If you are unable to resolve the task in 3 attempts, return a detailed failure report to the user instead of retrying."*

### 2. Context Window Dilution
* **Challenge:** As agents converse, the history of messages grows. Sending the entire historical message thread on every agent turn increases API costs and degrades the model's accuracy.
* **Solution:** 
  - **Summarized Memory:** Implement a background summarizing middleware. When the conversation history exceeds 4,000 tokens, the orchestrator triggers an LLM utility call to summarize old messages, condensing them while keeping key variables active.
  - **System State Separation:** Keep agent-to-agent technical variables out of the chat history. Store them in a centralized structured state map in Redis, only sending relevant parameters to individual sub-agents.

### 3. Latency & API Timeouts
* **Challenge:** A hierarchical execution chain might take 60 seconds to finish, exceeding HTTP API gateway limits and resulting in timeouts.
* **Solution:** 
  - **Asynchronous Execution:** Run agent tasks asynchronously. The API backend returns an execution token `job_id` immediately upon invocation.
  - **WebSocket Push:** The client connects via WebSockets. The orchestrator pushes status updates (e.g., *"SQL Agent starting..."*, *"Drafting email..."*) and streams the final output in real-time, preventing HTTP connection dropouts.
