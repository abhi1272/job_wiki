# MERN Interview Master Index — Abhishek Kumar

> Single entry point for all interview prep. Find any topic instantly.
> 📘 = Comprehensive deep-dive (new) | 📄 = Focused reference (original)

---

## 🟡 A — Behavioural & Soft Skills
*Use these the day before and morning of interview*

| Topic | File | What's inside |
|---|---|---|
| Tell me about yourself | [Tell_Me_About_Yourself.md](./Tell_Me_About_Yourself.md) | 60-second script |
| Why leaving / why this role | [Why_Leaving.md](./Why_Leaving.md) | Context + phrasing |
| STAR stories (5 scenarios) | [Behavioural_Stories.md](./Behavioural_Stories.md) | Concentrix, ConnectWise, Niyuj, Barclays, Infosys |
| Leadership & conflict | [Leadership_Questions.md](./Leadership_Questions.md) | Delegation, underperformers, tight deadlines |
| All Q&A combined | [Complete_Interview_QA.md](./Complete_Interview_QA.md) | Full bank of questions + scripted answers |

---

## 🟠 B — JavaScript Deep Internals
*Study this for any MERN, full-stack, or principal-level role*

| Topic | File | Sections |
|---|---|---|
| **PRIMARY — All JS internals** | 📘 [JavaScript_Deep_Internals.md](./JavaScript_Deep_Internals.md) | Closures, `this` binding (4 rules), prototypal inheritance, async/await internals, microtask queue, generators, Map/WeakMap/Set, design patterns (class + functional) + **§10 Intermediate Q&A** (hoisting/TDZ, destructuring, map/filter/reduce, optional chaining, typeof quirks, Object methods, short-circuit) |

---

## 🔵 C — React
*Study this for any full-stack or frontend-heavy role*

| Topic | File | Sections |
|---|---|---|
| **PRIMARY — Advanced React** | 📘 [React_Advanced_Internals.md](./React_Advanced_Internals.md) | Fiber + reconciliation, memo/useCallback/useMemo, Redux vs Zustand vs Context, compound components, controlled/uncontrolled, custom hooks + AbortController + **§6 Intermediate Q&A** (useEffect deps, key remount, React 18 batching, useRef, controlled forms, children prop, functional setState) |
| Hooks quick reference | 📄 [React_Hooks_Performance.md](./React_Hooks_Performance.md) | useMemo, useCallback, reconciliation, React 18 overview |

> ⚠️ `React_Advanced_Internals.md` is the deeper version. Read it first. Use `React_Hooks_Performance.md` as a quick cheat sheet.

---

## 🟢 D — Node.js
*Study this for backend, API, or infrastructure roles*

| Topic | File | Sections |
|---|---|---|
| **PRIMARY — Advanced Node.js** | 📘 [NodeJS_Advanced_Internals.md](./NodeJS_Advanced_Internals.md) | Streams + backpressure, worker threads vs clustering, memory leaks (detection + prevention), production error handling |
| Event loop fundamentals | 📄 [Node_Event_Loop.md](./Node_Event_Loop.md) | Loop phases, libuv, clustering basics |
| CI/CD & deployment | 📄 [CI_CD_Pipeline.md](./CI_CD_Pipeline.md) | GitLab flow, zero-downtime deploy, rollback |

> ⚠️ `NodeJS_Advanced_Internals.md` goes deeper. `Node_Event_Loop.md` is good for a 5-min refresher on loop phases.

---

## 🟣 E — Databases
*Study this for backend, data-heavy, or architect roles*

| Topic | File | Sections |
|---|---|---|
| **PRIMARY — MongoDB Advanced** | 📘 [MongoDB_Advanced_Internals.md](./MongoDB_Advanced_Internals.md) | Embed vs reference (ESR), aggregation pipeline, compound indexes (ESR rule), transactions |
| MySQL at scale | 📄 [MySQL_At_Scale.md](./MySQL_At_Scale.md) | Replicas, ProxySQL, indexing, N+1, ACID |

---

## 🔴 F — System Design
*Study this for principal, staff, or architect roles — highest weight*

| Topic | File | Sections |
|---|---|---|
| **PRIMARY — MERN System Design** | 📘 [MERN_System_Design.md](./MERN_System_Design.md) | Real-time streaming (WebSocket + Kafka + Redis Pub/Sub), JWT vs Sessions (hybrid), File upload pipeline (pre-signed S3), Caching strategy (TTL + invalidation + stampede), Multi-tenancy (pool vs silo) |
| GCP full-stack design | 📄 [System_Design_GCP.md](./System_Design_GCP.md) | Enterprise multi-tenant SaaS on GCP |
| Microservices | 📄 [Microservices_Patterns.md](./Microservices_Patterns.md) | API Gateways, circuit breakers, service mesh, Saga |
| Kubernetes | 📄 [Kubernetes_Internals.md](./Kubernetes_Internals.md) | Pod lifecycle, HPA, probes, crash loops |
| Architecture quiz | 📄 [Advanced_Software_Architecture_and_Development_Study_Guide.md](./Advanced_Software_Architecture_and_Development_Study_Guide.md) | Short-answer quiz, essay prompts, glossary |

---

## 🟤 G — AI / LLM Projects (Your Strongest Area)
*Lead with these — differentiate yourself from other MERN candidates*

| Topic | File | Sections |
|---|---|---|
| AI Gateway deep dive | 📄 [AI_Gateway_Deep_Dive.md](./AI_Gateway_Deep_Dive.md) | Observability, rate-limiting, safety routing, LiteLLM |
| Agent Builder deep dive | 📄 [Agent_Builder_Deep_Dive.md](./Agent_Builder_Deep_Dive.md) | Multi-agent orchestration, RAG, tool-use, GKE |
| Multi-agent patterns | 📄 [Multi_Agent_Orchestration.md](./Multi_Agent_Orchestration.md) | Router, conversational, hierarchical patterns |
| RAG pipeline | 📄 [RAG_Pipeline_Deep_Dive.md](./RAG_Pipeline_Deep_Dive.md) | Chunking, embeddings, retrieval evaluation |

---

## ⚫ H — Failed / Weak Areas (Practice These Most)
*Whatever you got wrong in past interviews — review before every interview*

| Topic | File |
|---|---|
| Failed interview questions tracker | 📄 [Failed_Interview_Questions_Master_Sheet.md](./Failed_Interview_Questions_Master_Sheet.md) |

---

## 🎯 Quick Study Plans by Interview Type

### For a Full-Stack MERN Interview (3-5 years company)
```
Day before:  B (JS) → C (React) → D (Node.js)
Morning of:  A (Behavioural) → H (Failed questions)
```

### For a Principal / Staff Engineer Interview
```
Day 2 before: F (System Design) → E (MongoDB)
Day before:   B (JS) → G (AI Projects)
Morning of:   A (Behavioural) → H (Failed questions)
```

### For an Architect / Solution Architect Interview
```
Day 2 before: F (System Design) → G (AI Projects)
Day before:   A (Behavioural) → E (MongoDB)
Morning of:   H (Failed questions) → 6_Quick_Reference/Cheatsheet.md
```

### For a Backend-Heavy Interview
```
Day before:  D (Node.js) → E (MongoDB) → F (System Design)
Morning of:  A (Behavioural) → H (Failed questions)
```

---

## 📊 Coverage Status

| Category | Depth | Notes |
|---|---|---|
| JavaScript | ✅ Deep | 9 concepts, class + functional patterns |
| React | ✅ Deep | 5 concepts incl. Fiber internals, concurrent mode |
| Node.js | ✅ Deep | 4 concepts incl. streams, memory leaks |
| MongoDB | ✅ Deep | Schema design, aggregation, ESR indexing, transactions |
| System Design | ✅ Deep | 5 real-world designs with full code |
| AI/LLM | ✅ Strong | Your actual projects — strongest differentiator |
| Behavioural | ✅ Covered | 5 STAR stories + full Q&A bank |
| Kubernetes | 🟡 Moderate | Internals covered, needs HPA + Istio practice |
| CI/CD | 🟡 Moderate | GitLab covered, needs ArgoCD + Helm depth |
