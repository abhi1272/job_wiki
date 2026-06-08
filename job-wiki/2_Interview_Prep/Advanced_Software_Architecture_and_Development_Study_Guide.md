# Advanced Software Architecture and Development Study Guide

This study guide provides a comprehensive review of modern software engineering principles, ranging from professional career positioning for senior architects to the deep technical internals of the Node.js runtime and system design methodologies.

--------------------------------------------------------------------------------

## Part 1: Short-Answer Quiz
**Instructions:** Answer the following questions in two to three sentences based on the provided technical documentation and professional guides.

1. **What is the core philosophy behind the "Node way" regarding the size and scope of modules?**
	Keep modules small and focused. Each module should do one clear job, so the code is easier to read, test, and reuse.
	**Example:** one module only validates email format, while another only sends emails.
2. **Explain the "CAR" strategy and its importance in a Solutions Architect's professional documentation.**
	CAR means Challenge, Action, Result. It helps you explain not just what you built, but why it mattered to the business.
	**Example:** "Challenge: high API latency. Action: added Redis cache. Result: response time improved by 40%."
3. **Distinguish between High-Level Design (HLD) and Low-Level Design (LLD) in a system design context.**
	HLD is the big picture: services, databases, APIs, and how they connect. LLD is the implementation detail: classes, methods, data structures, and patterns.
	**Example:** HLD says "use API Gateway + 3 microservices"; LLD defines "OrderService classes and method flow."
4. **What does the phrase "unleashing Zalgo" refer to in Node.js development, and why is it dangerous?**
	It means a function sometimes runs callback immediately and sometimes later, which makes behavior unpredictable. This can create random bugs that are hard to debug.
	**Example:** callback runs before event listener is attached in one case, but after attachment in another case.
5. **Describe the role of the Event Demultiplexer within the Reactor Pattern.**
	It watches many I/O sources (sockets/files) and tells the event loop which ones are ready. So Node handles many connections without one thread per request.
	**Example:** from 10,000 open sockets, it returns only the sockets that currently have data.
6. **What are the primary indicators of a memory leak in a Node.js production environment?**
	Memory usage keeps increasing over time even when traffic is stable. You may also see frequent GC pauses, slower performance, and eventual crash with out-of-memory.
	**Example:** heap grows from 300MB to 1.5GB every few hours and never comes down.
7. **How does a cell-based architecture improve the resilience of microservices?**
	It splits the platform into isolated cells so one failure does not affect everything. If one cell is unhealthy, traffic can be moved to healthy cells.
	**Example:** Cell A in one region fails, but Cell B and Cell C continue serving users.
8. **What is the "Small Core" principle in the Node.js ecosystem?**
	Node core stays minimal and stable, while most innovation happens in npm packages. This keeps the runtime lean and lets the community move fast.
	**Example:** instead of adding every feature to Node core, teams use external libraries like `express` or `fastify`.
9. **Explain the function of *libuv* in the Node.js architecture.**
	*libuv* is the engine that powers Node's event loop and async I/O. It hides OS differences so async code works similarly on Linux, macOS, and Windows.
	**Example:** networking code behaves consistently even though Linux uses `epoll` and Windows uses IOCP internally.
10. **In the context of enterprise AI, what is the purpose of an AI Gateway, such as the one implemented at Concentrix?**
	It gives one secure entry point to multiple LLM providers. It also handles access control, safety checks, rate limits, and cost tracking in one place.
	**Example:** teams call one internal endpoint, and gateway routes requests to GPT or Claude based on policy and budget.

--------------------------------------------------------------------------------

## Part 2: Answer Key

1. **Node Way:** Keep modules small and single-purpose.
	**Quick example:** split "auth" and "email" into separate packages.
2. **CAR Strategy:** Show Challenge, Action, and measurable Result.
	**Quick example:** "Reduced cloud bill by 25% after rightsizing clusters."
3. **HLD vs. LLD:** HLD is architecture overview; LLD is code-level design.
	**Quick example:** HLD defines services, LLD defines class responsibilities.
4. **Zalgo:** Avoid APIs that are sometimes sync and sometimes async.
	**Quick example:** always use `setImmediate` or Promise to keep callback timing predictable.
5. **Event Demultiplexer:** Reports which I/O operations are ready.
	**Quick example:** wakes event loop only when a socket has new data.
6. **Memory Leaks:** Watch for steady heap growth and frequent GC.
	**Quick example:** unbounded in-memory map keyed by user ID.
7. **Cell-based Architecture:** Isolates failures to one cell.
	**Quick example:** one cell outage affects 10% traffic, not 100%.
8. **Small Core:** Keep Node runtime lean; push features to ecosystem.
	**Quick example:** use npm packages for advanced features instead of core changes.
9. **libuv:** Handles async I/O and event loop cross-platform.
	**Quick example:** same app runs on macOS and Windows with consistent async behavior.
10. **AI Gateway:** Central control point for model access and governance.
	**Quick example:** block unsafe prompts and route low-priority jobs to cheaper models.

--------------------------------------------------------------------------------

## Part 3: Essay Questions

1. **Architectural Evolution:** Discuss the trade-offs between Monolithic and Microservices architectures. Include an analysis of "integrator" versus "disintegrator" factors and how they influence service granularity.
2. **The Impact of Asynchrony:** Analyze the Reactor Pattern as the foundation for Node.js. How does the single-threaded, non-blocking I/O model change the way developers approach concurrency compared to traditional multi-threaded blocking I/O?
3. **Strategic Technical Leadership:** Using the examples provided in the Solutions Architect resume guide, explain how a senior engineer can successfully communicate technical complexity to non-technical stakeholders while emphasizing business value.
4. **Security and Compliance in Modern Platforms:** Evaluate the necessity of integrating safety layers (such as LLM Guard) and privacy controls (like PII leakage prevention) in enterprise AI platforms. What are the long-term risks of neglecting these during the architectural phase?
5. **Debugging and Optimization:** Compare the different strategies for identifying and preventing memory leaks in high-volume Node.js applications. Discuss the role of heap snapshots, profiling tools, and proactive monitoring in maintaining system stability.

--------------------------------------------------------------------------------

## Part 4: Glossary of Key Terms

| Term | Definition |
| --- | --- |
| **API Gateway** | A centralized entry point in microservices used for managing APIs, request routing, and enforcing security policies. |
| **Bounded Context** | A concept from Domain-Driven Design (DDD) defining a specific area where a domain model is consistent and valid. |
| **CAP Theorem** | A principle stating a distributed system can only provide two of three guarantees: Consistency, Availability, and Partition Tolerance. |
| **Circuit Breaker** | An architectural pattern that monitors service health to prevent cascading failures by isolating failing components. |
| **CPS (Continuation-Passing Style)** | A functional programming style where the result of a function is propagated by passing it to a callback rather than returning it directly. |
| **Event Loop** | The core mechanism that iterates over the event queue and dispatches events to their associated handlers in a non-blocking system. |
| **HLD (High-Level Design)** | The architectural overview of a system focusing on components, modules, and their interactions. |
| **LLD (Low-Level Design)** | A detailed design focusing on class structures, data models, and the internal logic of individual modules. |
| **Microservices** | An architectural style that organizes an application as a collection of loosely coupled, independently deployable services. |
| **Non-blocking I/O** | A mode where system calls return immediately, allowing the thread to continue execution even if the data is not yet available. |
| **RAG (Retrieval-Augmented Generation)** | A pipeline involving document ingestion, vector embedding, and semantic search to provide AI agents with context from internal knowledge bases. |
| **Reactor Pattern** | An asynchronous programming pattern that dispatches events from a demultiplexer to associated handlers via an event loop. |
| **SOLID Principles** | A set of five design principles intended to make software designs more understandable, flexible, and maintainable. |
| **Vector Database** | A specialized database used in RAG pipelines to store and retrieve data based on semantic similarity rather than exact matches. |
| **Zero-downtime Restart** | A deployment strategy enabled by the cluster module or load balancers to update an application without interrupting service. |
