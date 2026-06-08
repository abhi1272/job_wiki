# 35-Hour Train Study Plan

This study plan is structured to help you organize your preparation hours during a long train commute or dedicated study sprint. It breaks down 35 hours into focused blocks targeting your primary interview areas.

[← Back to Index](../README.md) | [← Previous: Cheatsheet](./Cheatsheet.md) | [Next: Reverse Interview Questions](./Questions_To_Ask.md)

---

## 🕒 Study Plan Breakdown

```text
  [ Hours 1-8 ]   ──>   [ Hours 9-16 ]  ──>  [ Hours 17-24 ] ──> [ Hours 25-30 ] ──> [ Hours 31-35 ]
  Project Prep           System Design        AI & Concepts       Gaps Review       Mock & Review
```

---

## 🗓️ Hourly Schedule & Topics

### 📖 Block 1: Project Anchoring (8 Hours)
*Goal: Solidify your project stories until you can explain them smoothly.*
* **Hour 1–2:** Rehearse your **60-Second Introduction**. Record yourself and listen to your pacing. Adjust to ensure the AI Gateway and Agent Builder are front-and-center.
* **Hour 3–5:** Read the [AI Gateway Deep Dive](../2_Interview_Prep/AI_Gateway_Deep_Dive.md). Outline the exact configuration values of LiteLLM, Langfuse, and LLM Guard on paper.
* **Hour 6–8:** Read the [Agent Builder Deep Dive](../2_Interview_Prep/Agent_Builder_Deep_Dive.md). Focus on explaining RAG chunking parameters and Multi-agent orchestrator logic.

### 📐 Block 2: System Design & Infrastructure (8 Hours)
*Goal: Master GKE cluster operations, database scaling, and microservice patterns.*
* **Hour 9–11:** Study [GCP System Design](../2_Interview_Prep/System_Design_GCP.md) and [Kubernetes Internals](../2_Interview_Prep/Kubernetes_Internals.md). Learn how to explain readiness/liveness differences and how KEDA HPA autoscaling works.
* **Hour 12–14:** Read [Microservices Patterns](../2_Interview_Prep/Microservices_Patterns.md). Memorize Saga Orchestration (ConnectWise) and write down compensation steps.
* **Hour 15–16:** Study [MySQL at Scale](../2_Interview_Prep/MySQL_At_Scale.md). Practice explaining the N+1 query problem, indexing rules (EXPLAIN), and read replication lag issues.

### 🧠 Block 3: Distributed Concepts & AI (8 Hours)
*Goal: Review Kafka event streaming and Vertex AI concepts.*
* **Hour 17–19:** Study [Kafka Deep Dive](../3_Concepts/Kafka_Deep_Dive.md). Practice explaining partition scaling, manual offset commits, and Dead Letter Queues.
* **Hour 20–21:** Read [Vertex AI on GCP](../3_Concepts/Vertex_AI_GCP.md). Understand Gemini API call streams and GKE Workload Identity credentials authentication.
* **Hour 22–24:** Review [System Design Patterns](../3_Concepts/System_Design_Patterns.md). Draw simple block diagrams of caching, pub/sub, and circuit breaker patterns.

### 🛠️ Block 4: Target Gaps Resolution (6 Hours)
*Goal: Build confidence in your weaker technical areas.*
* **Hour 25–27:** Study [LLM & Fine-Tuning](../3_Concepts/LLM_Fine_Tuning.md). Memorize the conceptual math of LoRA (low-rank matrices A and B) and NF4 quantization parameters.
* **Hour 28–30:** Review [JD Analysis and Gaps](../4_JD_Analysis/Principal_Solution_Architect.md). Memorize your mitigation answers for Playwright, K6, and Cypress.

### 🎯 Block 5: Behavioral Stories & Mocks (5 Hours)
*Goal: Practice behavioral stories and mock execution.*
* **Hour 31–32:** Review the [5 Behavioral Stories](../2_Interview_Prep/Behavioural_Stories.md). practice explaining them using the STAR format (Situation, Task, Action, Result).
* **Hour 33–34:** Practice answering [Leadership Questions](../2_Interview_Prep/Leadership_Questions.md) (onboarding plans, delegation, conflicts).
* **Hour 35:** Read the [Checklists](./Checklists.md) and the [Reverse Interview Questions](./Questions_To_Ask.md). Run a final quick review of your Cheatsheet one-liners.
