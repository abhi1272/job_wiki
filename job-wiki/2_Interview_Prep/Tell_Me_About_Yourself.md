# "Tell Me About Yourself" (60-Second Script)

This script is designed to introduce your profile in an interview. It hooks the interviewer by leading with your seniority (11 years, Lead Engineer), highlighting your two major projects at Concentrix (AI Gateway and Agent Builder), and closing with why you are a fit for the role.

[← Back to Index](../README.md) | [Next: GCP System Design](./System_Design_GCP.md)

---

## 🎙️ Speech Script
> Speak at a moderate, confident pace. This script takes approximately 60–75 seconds to read out loud.

"Sure! I'm Abhishek, a Lead Full-Stack and AI Platform Engineer with **11 years of experience** building high-throughput microservices and scalable cloud platforms.

Over the past few years at **Concentrix**, I've focused on building enterprise AI platforms. I led the development of our **Enterprise AI Gateway** from scratch, which uses LiteLLM and Python to route, cost-track, and secure LLM calls across Claude, GPT-4, and Gemini for our internal engineering teams. 

Alongside this, I architected our **No-Code AI Agent Builder Platform** on GKE, leading a team of 8 engineers. We built a React UI, a Node/Express orchestration layer, and decoupled worker threads with Kafka and WebSockets to stream LLM tokens in real-time. We also built custom RAG pipelines using LangChain and Postgres Vector DBs.

Before Concentrix, I stabilized core document processing pipelines at **ConnectWise** by redesigning their asynchronous workloads into a Node.js Master-Worker execution pattern, which resolved critical database deadlock issues and improved performance by 35%.

I specialize in building full-stack applications with React and Node.js while owning the cloud platform infrastructure on GCP and Kubernetes. I'm looking for my next challenge as a Lead/Staff Engineer where I can bridge robust product architecture with cutting-edge AI orchestration. That's why I'm excited about this opportunity."

---

## 💡 Key Design Choices of the Script

1. **Seniority First:** Anchoring with "11 years of experience" and "Lead" title establishes your leveling right away.
2. **Anchor AI Gateway & Agent Builder:** These are your two core projects. They show you understand both API routing (infrastructure/observability) and agent orchestration (LangChain/RAG).
3. **ConnectWise System Pattern:** Mentioning the "Master-Worker pattern" at ConnectWise proves you know distributed microservices and database concurrency, not just AI helper tools.
4. **Transition Hook:** The end maps directly to what top product companies want: someone who can write React/Node but also deploy Docker/Kubernetes on GCP, and understands LLMs.
