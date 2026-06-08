# Leadership, Delegation & Team Management

This document outlines your approach to engineering leadership, task delegation, conflict resolution, and developer onboarding, reflecting your experience leading 5 to 10 engineers at Concentrix.

[← Back to Index](../README.md) | [← Previous: Behavioural Stories](./Behavioural_Stories.md) | [Next: Why Leaving / Why This Role](./Why_Leaving.md)

---

## 👥 1. How do you describe your leadership style?

"I describe my leadership style as **Empathetic and Action-oriented**. As a Lead Engineer, my primary role is to clear roadblocks for my team, set high technical standards, and help engineers grow their skills.

I believe in giving engineers ownership of features. For example, when building the No-Code Agent Builder, instead of assigning tasks, I gave engineers ownership of specific modules (e.g. one owned WebSockets, another owned UI graphs). I then acted as a system design consultant, doing code reviews and helping them work through challenges, rather than micromanaging their execution. This builds trust and encourages taking pride in code quality."

---

## 🔄 2. How do you handle conflict within your engineering team?

"I handle conflict by focusing on **Data over Opinions** and resolving issues early.

### Real-World Example at Concentrix:
When architecting our Agent Builder storage, a senior engineer wanted to use MongoDB for everything, while another engineer argued for PostgreSQL with `pgvector` for vector storage. The discussion was stalling our sprint progress.

* **My Action:** I set up a design spike. I asked both engineers to build a small prototype index of 10,000 vectors on each database and test them for query search speed and ease of hybrid search (RRF) integration.
* **The Resolution:** The benchmarks showed that while MongoDB was great for storing agent schema configurations, Postgres + `pgvector` was 3x faster for retrieval and had native support for Cosine Distance. We agreed to use both: MongoDB for agent node layout configs, and Postgres for RAG vector storage. By shifting the conversation to benchmarks, we made a collaborative decision without ego."

---

## 📋 3. What is your approach to task delegation?

"I delegate tasks based on two factors: **Engineer Skill Level** and **Growth Goals**.

* **For Senior Engineers:** I define the high-level system requirements (e.g., *'We need a secure rate-limiting proxy using Redis for our gateway'*) and give them full ownership of design, API schema, and implementation.
* **For Mid/Junior Engineers:** I break the task into smaller deliverables. I pair them with a senior engineer or spend time with them detailing the interface signatures before they begin coding.
* **Growth Goals:** If an engineer wants to learn infrastructure, I don't give the next Kubernetes configuration task to the person who already knows it. I assign it to the learning engineer and have the experienced engineer review their work, balancing delivery speed with team growth."

---

## 🚀 4. How do you onboard a new developer to your team?

"I use a structured **30-60-90 Day Onboarding Plan** to help new developers feel welcome and productive quickly.

1. **Day 1–30 (Understand & Ship):** 
   - Assign a dedicated 'onboarding buddy' (usually a senior teammate).
   - Get local development environments configured within the first two days (we maintain docker-compose setups to make this one-click).
   - Goal: Ship a small bug fix or simple feature to production in Week 1 to build confidence in our CI/CD processes.
2. **Day 31–60 (Feature Ownership):** 
   - Assign ownership of a medium-sized feature, requiring them to collaborate with product managers and other engineering teams.
3. **Day 61–90 (Autonomy):** 
   - The developer should be comfortable picking up complex tickets, participating in design reviews, and reviewing code for peers.
- **Feedback Loop:** I schedule weekly 1-on-1s during onboarding to check in, answer questions, and address any gaps in documentation."
