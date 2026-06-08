# Behavioural Interview Stories (STAR Format)

This document contains 5 professional behavioural stories, structured using the STAR (Situation, Task, Action, Result) format. Each story is anchored to a specific company in your work history.

[← Back to Index](../README.md) | [← Previous: Multi-Agent Orchestration](./Multi_Agent_Orchestration.md) | [Next: Leadership & Team Management](./Leadership_Questions.md)

---

## 🌟 Story 1: Scaling AI Gateway (Concentrix)
* **Focus Area:** Technical Leadership, High-Scale Architecture, Cost Optimization.

* **Situation:** Within Concentrix, shadow AI usage was growing rapidly. Multiple engineering teams were building redundant connections to OpenAI and Anthropic, leading to leaked API keys, high vendor bills, and no security audit logs for PII data.
* **Task:** I was tasked with leading the team to build a centralized, enterprise-grade AI Gateway from scratch within 3 months, serving as the single point of entry for all LLM calls.
* **Action:** I architected the gateway using LiteLLM (for multi-provider routing) and Python on Google Kubernetes Engine (GKE). To handle safety, I integrated LLM Guard middleware to block prompt injections and mask PII. I built a multi-tenant rate-limiting and budget alert module using a sliding window counter algorithm in Redis (Cloud Memorystore). I also routed all transaction logs asynchronously to Langfuse to track costs and latency without adding processing delay.
* **Result:** The gateway was successfully deployed to production within the deadline. It consolidated API keys, blocked **100%** of prompt injection attempts in testing, reduced shadow AI model costs by **35%**, and maintained a routing latency overhead of under **12ms**.

---

## 🌟 Story 2: Stabilising ITBoost (ConnectWise)
* **Focus Area:** Problem Solving, Performance Tuning, Distributed Systems.

* **Situation:** ConnectWise's core ITBoost documentation portal was experiencing severe performance issues. During peak usage hours, background document indexing tasks blocked the single-threaded Node.js event loop, resulting in HTTP database connection pools dropping, SQL deadlocks, and system downtime.
* **Task:** I was assigned to resolve these concurrency locks and stabilize the platform's backend document indexing.
* **Action:** I analyzed the slow query logs and event loop timings. I refactored the indexing logic to decouple heavy parsing from the main API thread. I designed and implemented a **Master-Worker background processing pattern** utilizing Node.js's `worker_threads` module. I offloaded the document indexing to separate system threads and restructured database transactions in MySQL, replacing nested locks with optimistic locks.
* **Result:** These changes eliminated the database deadlocks entirely. Ingestion page latency improved by **35%**, and system CPU utilization during indexing dropped by **45%**, restoring system stability during peak traffic.

---

## 🌟 Story 3: Delivering Campaign Management (Niyuj)
* **Focus Area:** Fast Delivery under Pressure, Cloud Migration.

* **Situation:** At Niyuj, we were building a multi-channel marketing campaign management system for a major client. Due to a sudden change in client timeline requirements, we had to compress our delivery schedule by 4 weeks to hit a critical marketing launch date.
* **Task:** We needed to accelerate feature delivery for the campaign builder interface and deploy the backend infrastructure to Microsoft Azure.
* **Action:** I stepped up to lead the sprint planning meetings to prioritize critical core features. I built the campaign logic engine using Node.js microservices and worked on the campaign canvas UI in React. To speed up deployments, I wrote Terraform templates to spin up Azure App Services, Azure SQL Databases, and configured Azure DevOps CI/CD pipelines to run tests and deploy automatically.
* **Result:** By streamlining the build process, our team delivered the campaign engine one week ahead of the compressed deadline. The client launched their marketing campaign on time, driving over **50,000 new user sign-ups** in the first week.

---

## 🌟 Story 4: MEAN Stack Dashboards (Barclays)
* **Focus Area:** Process Automation, Operational Efficiency.

* **Situation:** At Barclays, team operations reports and access audits were compiled manually by operations analysts. This involved downloading raw spreadsheets from multiple internal servers, cleaning data, and pasting it into PowerPoint decks, taking up to 3 days per week.
* **Task:** I wanted to build an automated, real-time analytics portal that would retrieve, process, and display these metrics on a live dashboard.
* **Action:** I gathered requirements from operations stakeholders. I built an Automation Portal using the **MEAN (MongoDB, Express, Angular, Node.js) stack**. I wrote scheduled cron scripts in Node to fetch CSV files from source servers using SFTP, parse the data, and update MongoDB. I built responsive dashboard views in Angular, featuring charts showing user activity and security compliance metrics.
* **Result:** The new Automation Portal eliminated the manual reporting workflow. It saved operations analysts **12 hours of manual work per week** and provided leadership with instant access to audit metrics.

---

## 🌟 Story 5: IVR Fraud Detection (Infosys)
* **Focus Area:** Performance Tuning, Legacy Systems.

* **Situation:** While working as a System Engineer at Infosys, our client's Interactive Voice Response (IVR) phone system had high call-drop rates during fraud check calls. The fraud database logic was too slow, taking over 3 seconds to verify caller IDs against historical threat lists, causing the telephone line to timeout.
* **Task:** I had to optimize the caller ID check query database logic to run in under 300 milliseconds.
* **Action:** I inspected the database execution plan and realized the threat check query was performing a full table scan on a table containing over 10 million phone number logs. I optimized the query by:
  1. Adding a composite index on `(caller_id, status)`.
  2. Restructuring the query to search only the past 30 days of threat logs instead of the entire table history.
  3. Configuring a Redis cache layer to store white-listed numbers, bypassing database checks for known clean callers.
* **Result:** The fraud check response time dropped from **3.2 seconds** to **under 80 milliseconds** (a 97% improvement). This reduced IVR call timeouts to zero, preserving critical security checks without impacting caller experience.
