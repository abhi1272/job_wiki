# Project Deep Dive — Enterprise AI Gateway

This document details the architecture, design choices, and business impact of the **Enterprise AI Gateway** you led and built from scratch at Concentrix.

[← Back to Index](../README.md) | [← Previous: GCP System Design](./System_Design_GCP.md) | [Next: Agent Builder Deep Dive](./Agent_Builder_Deep_Dive.md)

---

## 📋 Project Context & Problem

At Concentrix, multiple engineering teams started integrating LLMs into their workflows. This led to:
1. **API Key Proliferation:** Lack of centralized key management led to keys leaked in git repositories.
2. **Cost Overruns:** No visibility into which teams were spending what on API tokens. Teams could accidentally loop prompts and run up massive bills.
3. **Safety Risks:** No centralized firewall to prevent prompt injection or block PII leaks to public external models.

---

## 🛠️ Architecture & Tech Stack

```text
Team Clients ──> [LLM Request] ──> [LLM Guard Filter] ──> [LiteLLM Router] ──> [Model Providers]
                                          │                     │
                                   (PII / Safety check)  (Logs to Langfuse)
```

* **Core Routing Engine:** LiteLLM (Python, wrapping Anthropic, OpenAI, Gemini).
* **Observability & Billing:** Langfuse (self-hosted integration tracking token usage, costs, latency, and system performance).
* **Security Proxy:** LLM Guard (PII detection and prompt injection scanner).
* **Infrastructure:** Deployments on Google Kubernetes Engine (GKE), secured by GCP Secret Manager and Cloud Memorystore (Redis) for token counting.

---

## ⚙️ Core Gateway Features & Technical Implementation

### 1. Cost Observability & Cost Tracking (LiteLLM + Langfuse)
* **How:** When a client sends a payload to the gateway, LiteLLM parses the target model (e.g., `claude-3-5-sonnet`) and calculates input/output token counts dynamically.
* **Tracking:** We configured a post-call middleware that asynchronous routes request metadata (tokens, latency, model, cost, user metadata) directly to a centralized self-hosted Langfuse server.
* **Outcome:** Created a multi-tenant dashboard breaking down LLM usage per internal department, showing a **35%** reduction in total API costs by identifying duplicate routing requests and unused endpoints.

### 2. Multi-Tenant Rate-Limiting & Quota Budgets
* **Rate Limiting:** Implemented at the API-key/team level using a sliding-window rate-limiting algorithm in Redis (Cloud Memorystore). We set thresholds (e.g., 50 requests/min and 100k tokens/min).
* **Quota Enforcement:** The gateway checks Redis for daily/monthly budget allocations. If a team exceeds their monthly dollar budget (e.g., $500/month), the gateway returns a `429 Too Many Requests` status with a custom error message.
* **Auto-Alerts:** Triggers Slack alerts when a team hits **80%** of their monthly budget.

### 3. LLM Guard Security Proxy (PII + Injection Filtering)
* **Prompt Injection Detection:** Configured LLM Guard scanner middleware on all incoming prompts. It uses a small, fast classifier to evaluate the request and block it if the score exceeds `0.75`.
* **PII Masking:** Before sending payloads to external APIs, the gateway checks for sensitive variables (emails, credit card patterns, phone numbers) and masks them with placeholder values (e.g., `[REDACTED_EMAIL]`). On response streams, the gateway reverses the masking.

---

## 💻 Code Snippet: LiteLLM Router Config
> This snippet shows how we configure routing and failover across multiple model endpoints:

```yaml
model_list:
  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: os.environ/CLAUDE_API_KEY
      rpm: 1000
      tpm: 80000
  - model_name: claude-3-5-sonnet-backup
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: os.environ/CLAUDE_BACKUP_API_KEY
      rpm: 500
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
      rpm: 2000

router_settings:
  routing_strategy: latency-based-routing
  allowed_fails: 3
  cooldown_time: 30
  failover_target: claude-3-5-sonnet-backup
```

---

## 📈 Quantitative Results

* **API Cost Savings:** Reduced Concentrix's shadow AI model spending by **35%** in the first quarter of deployment.
* **Gateway Performance:** Maintained a latency overhead of under **12ms** for processing rate-limit rules, PII filters, and logging.
* **Security Incidents:** Blocked **100%** of detected prompt injection payloads, preventing model manipulation in customer-facing test applications.
