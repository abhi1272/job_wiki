# Node.js Event Loop & Scaling Architecture

This document covers Node.js internals and backend scaling strategies. It maps directly to issues you resolved at ConnectWise (stabilizing ITBoost CPU bottlenecks) and Concentrix (scaling the NestJS orchestration engine).

[← Back to Index](../README.md) | [← Previous: React Hooks & Performance](./React_Hooks_Performance.md) | [Next: RAG Pipeline Deep Dive](./RAG_Pipeline_Deep_Dive.md)

---

## 🔄 The Node.js Event Loop Explained

Node.js is single-threaded but achieves high concurrency by offloading input/output (I/O) tasks to the operating system kernel or the Libuv thread pool.

```text
Incoming Requests ──> [ Event Loop ] ── (Phases) ──> [ Libuv Thread Pool ] (File I/O, Crypto)
                             │
                             ▼
                    [ V8 Engine Exec ]
```

### Event Loop Phases:

1. **Timers:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`.
2. **Pending Callbacks:** Executes I/O callbacks deferred from the previous loop iteration (e.g., TCP errors).
3. **Idle, Prepare:** Used internally by Node.js.
4. **Poll:** Retrieves new I/O events. Node will block here if there are no timers or setImmediate callbacks scheduled.
5. **Check:** Executes callbacks registered with `setImmediate()`.
6. **Close Callbacks:** Executes close events (e.g., `socket.on('close')`).

### ⚡ Microtasks (nextTick and Promises)
* **`process.nextTick()` and Promise callbacks** do not belong to a specific loop phase. Instead, they execute **immediately after the current operation finishes**, before the event loop transitions to the next phase. If you recurse `process.nextTick()`, you will starve the event loop, blocking all I/O.

---

## 🚫 Blocking the Event Loop & How to Avoid It

Because Node.js executes JavaScript on a single thread, any heavy CPU execution (e.g., large JSON parsing, array sorting, cryptography, parsing local files) will block the thread, causing all other incoming HTTP requests to time out.

### ConnectWise Case Study: ITBoost Optimization
* **The Problem:** The ITBoost document search indexing was parsing huge file arrays inline. This blocked the event loop for 4 seconds, causing other users' requests to timeout and drop.
* **The Solution:** We refactored the heavy parsing workload using two approaches:
  1. **Split Workloads (setImmediate):** Breaking heavy array processing into smaller chunks using `setImmediate` loops to allow the event loop to handle intermediate I/O events between steps.
  2. **Worker Threads (CPU Scaling):** Offloaded file parsing to separate thread runtimes using Node's `worker_threads` module, freeing up the main event loop to route API traffic.

---

## ⚙️ Scaling Node.js Applications

### 1. Vertical Scaling: The Cluster Module
* **What:** Spawns a process pool, sharing the same TCP port.
* **Mechanism:** Node's `cluster` module utilizes a master process that forks worker processes (typically one worker per CPU core). The master process uses a Round-Robin algorithm to distribute incoming connection requests to the workers.
* **Pros:** Multi-core utilization on a single virtual machine.

### 2. Worker Threads vs. Cluster Processes
* **Cluster Process:** A separate process with its own memory space, V8 instance, and Node runtime. Communication is done via IPC (Inter-Process Communication) serialization.
* **Worker Thread:** Runs inside the *same* process, sharing memory space but running on a separate execution thread with its own event loop. Good for sharing raw binary buffers (using `SharedArrayBuffer`) for memory efficiency during heavy processing (e.g. video manipulation or large JSON parses).

### 3. Horizontal Scaling (Kubernetes Pods)
* While clustering is good for single instances, in a production GKE cluster (Concentrix setup), we scale horizontally by spinning up multiple single-threaded Kubernetes Pod replicas. This allows us to scale nodes up and down dynamically using the HPA (Horizontal Pod Autoscaler) based on cluster-wide resource metrics.
