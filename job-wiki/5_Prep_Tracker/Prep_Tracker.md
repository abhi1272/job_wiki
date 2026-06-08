# Skills Prep Tracker & Self-Assessment Workbook

This workbook is your study tracker. You can edit this markdown file in VS Code or any text editor to update your ratings, check off completed study units, and add notes as you prepare for interviews.

[← Back to Index](../README.md) | [Next: Cheatsheet (Quick Reference)](../6_Quick_Reference/Cheatsheet.md)

---

## 🚦 Skills Matrix & Self-Assessment

*Update the ratings (`1` to `5`) and status badges as you study. Ratings are pre-filled based on your profile.*

### 🟢 Strong Areas (4-5 Stars) — *Brief review before interviews*

| Skill | Target Rating | Current Rating | Progress / Study Notes |
|---|---|---|---|
| **Node.js / NestJS** | 5 / 5 | `[X] 5/5` | Deep knowledge of event loop, microservice patterns. |
| **JavaScript / TypeScript** | 5 / 5 | `[X] 5/5` | ES2022+ features, async patterns. |
| **React / Next.js** | 5 / 5 | `[X] 5/5` | Virtual DOM reconciliation, useMemo, useCallback. |
| **Kubernetes (GKE/AKS)** | 5 / 5 | `[X] 5/5` | Pod setups, Services, HPA, readiness/liveness checks. |
| **API Gateways (LiteLLM)** | 5 / 5 | `[X] 4.5/5` | Key routing, failover configurations. |
| **Langfuse Observability** | 5 / 5 | `[X] 4/5` | Tracing, latency monitoring, cost calculation. |
| **Docker Containerization** | 5 / 5 | `[X] 5/5` | Multi-stage builds, image footprint minimization. |
| **CI/CD Pipelines** | 5 / 5 | `[X] 4.5/5` | GitLab runner rules, zero-downtime Helm rollouts. |
| **Apache Kafka** | 5 / 5 | `[X] 4/5` | Partitions, Consumer groups, manual offset commits. |
| **MySQL / PostgreSQL** | 5 / 5 | `[X] 4.5/5` | Composite indexing, Join performance, ACID rules. |

---

### 🟡 Intermediate Areas (3 Stars) — *Needs target review*

| Skill | Target Rating | Current Rating | Progress / Study Notes |
|---|---|---|---|
| **Vertex AI (GCP)** | 5 / 5 | `[ ] 3/5` | Review SDK methods and Vector Search deployment pipelines. |
| **GraphQL Schemas** | 5 / 5 | `[ ] 3/5` | Review query resolvers and NestJS Decorator optimization. |
| **Saga Orchestration** | 5 / 5 | `[ ] 3.5/5` | Review failure state compensation steps. |
| **AWS Security Configuration**| 5 / 5 | `[ ] 3/5` | Review AWS WAF rule lists vs Security Groups. |

---

### 🔴 Critical Gap Areas (1-2 Stars) — *Prioritize studying these*

| Skill | Target Rating | Current Rating | Progress / Study Notes |
|---|---|---|---|
| **LoRA / QLoRA Fine-tuning** | 4 / 5 | `[ ] 2/5` | Understand matrix math, target modules, NF4 quantization. |
| **PyTorch & Hugging Face** | 4 / 5 | `[ ] 1.5/5` | Practice loading model weights and tokenizers in Colab. |
| **Playwright UI Testing** | 4 / 5 | `[ ] 2/5` | Review basic selector rules and async wait execution. |
| **Cypress / RTL Hands-on** | 4 / 5 | `[ ] 2/5` | Review frontend component test mock patterns. |
| **RedHat OpenShift** | 4 / 5 | `[ ] 2/5` | Review differences from standard Kubernetes (SCCs, Operators). |

---

## 📅 Study Plan Generator (Targeted Schedules)

Select a study track based on your upcoming interview timeline:

### ⏱️ Track A: The 3-Day Emergency Sprint
*Focus only on core architecture narratives and key definitions. Skip fine-tuning loops.*
* **Day 1: AI Gateway & Agent Builder deep dives.** Rehearse the 60-second introduction. Read the [AI Gateway Deep Dive](../2_Interview_Prep/AI_Gateway_Deep_Dive.md).
* **Day 2: Kubernetes & Microservices.** Read [Kubernetes Internals](../2_Interview_Prep/Kubernetes_Internals.md) and [System Design](../2_Interview_Prep/System_Design_GCP.md).
* **Day 3: Database & CI/CD.** Rehearse the ITBoost stabilization story and review [MySQL at Scale](../2_Interview_Prep/MySQL_At_Scale.md).

### 🗓️ Track B: The 2-Week Balanced Plan
*Combines core architecture review with targeted studying of your gap areas.*
* **Week 1 (Core Strengths):**
  - [ ] Day 1: Tell Me About Yourself script + AI Gateway review.
  - [ ] Day 2: Agent Builder + RAG pipelines deep dive.
  - [ ] Day 3: GCP System Design + Vertex AI.
  - [ ] Day 4: Kubernetes internals & GKE operations.
  - [ ] Day 5: Node.js event loop & scaling (ITBoost cases).
  - [ ] Day 6: MySQL indexing, replica configuration, N+1 query debugging.
  - [ ] Day 7: Behavioural stories review (STAR format).
* **Week 2 (Gap Resolution):**
  - [ ] Day 8: Study LoRA/QLoRA weight math.
  - [ ] Day 9: Practice loading tokenizers/models in Hugging Face.
  - [ ] Day 10: Review Cypress and RTL component testing.
  - [ ] Day 11: Study Playwright and K6 load testing patterns.
  - [ ] Day 12: Review OpenShift security contexts vs standard K8s.
  - [ ] Day 13: Read the JD Analysis files & practice tailoring rules.
  - [ ] Day 14: Review the Cheatsheet and run mock interview questions.

---

## 📝 Personal Study Logs & Progress Notes
*Use this space to write down your notes, links to tutorials, and definitions as you study.*

* **Example Entry:**
  - *Date: 2026-06-08* — Reviewed LoRA weights. Remember: LoRA updates weights by training two low-rank matrices ($A$ and $B$) with rank $r$ (usually 8 or 16), which saves VRAM because the frozen base model weights don't update.
  - *Study Note:* [Write your note here]
  - *Study Note:* [Write your note here]
