# Questions to Ask the Interviewer (Reverse Interviewing)

Ask these strategic questions at the end of your interviews to evaluate the team's engineering standards, product roadmap, and operational practices.

[← Back to Index](../README.md) | [← Previous: 35-Hour Train Study Plan](./Train_Study_Plan.md) | [Next: Checklists](./Checklists.md)

---

## 🛠️ Architecture & Platform Questions
*Recommended for Principal and Staff-level technical rounds.*

1. **"How do you handle API key security, prompt safety filters, and cost observability for LLM endpoints across different teams?"**
   - *Why ask:* Shows you understand enterprise AI operations. It opens a path to talk about how you solved this using the LiteLLM/LLM Guard gateway stack.
2. **"What does your current deployment flow look like? Do you rely on standard rolling updates, or do you run blue-green/canary releases on Kubernetes?"**
   - *Why ask:* Initiates discussions about cluster infrastructure, showing you are comfortable with container operations (K8s/Docker).
3. **"How does the engineering team handle database scaling (e.g. read replication, connection pooling, sharding) as query volume increases?"**
   - *Why ask:* Shows you think about database optimization and scaling, not just application code.

---

## 👥 Engineering Culture & Leadership Questions
*Recommended for Hiring Manager or Lead Engineer rounds.*

1. **"What is your approach to system design reviews? How do you resolve disagreements when two senior engineers propose different architectural designs?"**
   - *Why ask:* Highlights your focus on collaborative engineering. You can share your **Data over Opinions** benchmark spike practice.
2. **"How do you balance shipping new features quickly with resolving technical debt and refactoring legacy systems?"**
   - *Why ask:* Shows you care about long-term code quality and systems stability.
3. **"What does a successful onboarding plan look like for a new engineer joining your team? What should they accomplish in their first 30 days?"**
   - *Why ask:* Connects to your own **30-60-90 day onboarding blueprint** and shows you think like a team leader.

---

## 🚀 Product & Growth Questions
*Recommended for Director of Engineering or Product Manager rounds.*

1. **"What are the biggest technical bottlenecks currently blocking the team's product roadmap for the next two quarters?"**
   - *Why ask:* Displays immediate interest in helping solve the team's core problems.
2. **"How does the company evaluate the business impact of the AI systems you are deploying (e.g. cost per customer session, relevance accuracy, user engagement)?"**
   - *Why ask:* Aligns engineering efforts with business value, which is a key trait of Staff and Principal engineers.
3. **"What are the qualities that separate a good engineer on this team from someone who is exceptional and drives significant impact?"**
   - *Why ask:* Displays strong ambition and a desire to deliver high-quality work.
