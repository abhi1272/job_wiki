# Node.js Advanced Internals — MERN Senior/Principal Interview Prep

[← Back to Index](../README.md)

---

## 1. Streams & Backpressure

### The Problem with Loading Everything into Memory
```javascript
// ❌ Broken for large datasets
const data = await db.collection('orders').find().toArray(); // 2GB in RAM
const csv = convertToCSV(data);  // another 2GB in RAM → heap crash
res.send(csv);                   // client waits entire time before byte 1
```
3 problems: memory crash, high TTFB, no backpressure.

### 4 Stream Types
```
Readable  → source (file, DB cursor, HTTP request body)
Writable  → destination (HTTP response, file, process.stdout)
Transform → reads + transforms + writes (CSV converter, gzip, encrypt)
Duplex    → both (TCP socket, WebSocket)
```

### Production Fix — pipeline
```javascript
import { Transform, pipeline } from 'stream';
import { promisify } from 'util';
import { createGzip } from 'zlib';

const pipelineAsync = promisify(pipeline);

app.get('/report', async (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Encoding', 'gzip');

  const cursor = db.collection('orders').find().stream(); // Readable — batches from DB

  const csvTransform = new Transform({
    objectMode: true,
    transform(doc, encoding, callback) {
      callback(null, `${doc.id},${doc.customer},${doc.amount},${doc.date}\n`);
    }
  });

  try {
    await pipelineAsync(cursor, csvTransform, createGzip(), res);
    // cursor → csvTransform → gzip → response
    // Client receives data immediately, memory stays ~constant (a few MB)
  } catch (err) {
    if (!res.headersSent) res.status(500).json({ error: 'Export failed' });
  }
});
```

### Backpressure — The Critical Concept
Backpressure = the signal from writable to readable: "slow down, I'm full."

```javascript
// ❌ Ignores backpressure — buffer grows → memory leak
readable.on('data', (chunk) => writable.write(chunk));

// ✅ Manual backpressure
readable.on('data', (chunk) => {
  const ok = writable.write(chunk);
  if (!ok) readable.pause();          // writable buffer full — stop reading
});
writable.on('drain', () => readable.resume()); // buffer drained — resume

// ✅✅ pipeline handles backpressure + errors + cleanup automatically
await pipelineAsync(readable, transform, writable);
```

Always use `pipeline` over `.pipe()` — `.pipe()` doesn't handle errors or backpressure edge cases.

### Real Use Cases
```
File export      → DB cursor → CSV transform → gzip → HTTP response
File upload      → request stream → virus scan transform → S3 writable
Log processing   → file readable → filter transform → Elasticsearch
AI tokens        → LLM token stream → WebSocket writable (Agent Builder)
File compression → createReadStream → createGzip → createWriteStream
```

### Interview Answer
> "The naive approach loads everything into memory — for 2GB that crashes Node.js and the client waits the full processing time before byte one. Fix: MongoDB cursor is a Readable stream that fetches in batches; pipe through a Transform for CSV conversion, through gzip, into the HTTP response. Client starts receiving immediately, memory stays constant. The key concept is backpressure — if the client is slow, the writable signals the readable to pause. `pipeline` handles this automatically, which is why I use it over `.pipe()`."

---

---

## 2. Worker Threads vs Child Process vs Clustering

### Why CPU work blocks everything
Node.js is single-threaded. `await` only yields for I/O — pure CPU holds the thread hostage.
100 users hit a 2-second CPU route → 100th user waits 200 seconds.

### Worker Threads — right choice for CPU tasks
Same process, separate thread, shared memory. Keeps main thread free.

```javascript
// image-worker.js
const { workerData, parentPort } = require('worker_threads');
const sharp = require('sharp');

sharp(workerData.buffer).resize(800, 600).jpeg({ quality: 80 }).toBuffer()
  .then(result => parentPort.postMessage({ result }));
```

```javascript
// Production: use a pool — don't spawn per request (expensive)
import Piscina from 'piscina';

const pool = new Piscina({ filename: './image-worker.js', maxThreads: 4 });

app.post('/resize', upload.single('image'), async (req, res) => {
  const result = await pool.run({ buffer: req.file.buffer, width: 800, height: 600 });
  res.set('Content-Type', 'image/jpeg').send(result);
});
```

### Child Process vs Worker Threads
| | Worker Threads | Child Process |
|---|---|---|
| Memory | Shared (SharedArrayBuffer) | Separate process memory |
| Startup | ~10ms | ~100ms (new Node.js instance) |
| Crash isolation | Crash kills whole process | Only kills the child |
| Use when | CPU work, shared memory needed | Untrusted code, full isolation |

### SharedArrayBuffer — zero-copy shared memory
```javascript
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);
const worker = new Worker('./worker.js', { workerData: { sharedBuffer } });
sharedArray[0] = 42; // worker sees this immediately — no serialization
// vs Child Process: JSON serialize → IPC → deserialize = slow for large buffers
```

### Clustering — scales server across CPU cores (different problem)
```javascript
if (cluster.isPrimary) {
  os.cpus().forEach(() => cluster.fork()); // 8 processes on 8-core machine
  cluster.on('exit', () => cluster.fork()); // auto-restart
} else {
  app.listen(3000); // each worker is a full Express server
}
// OS load-balances connections across all 8 workers
// Does NOT fix per-request CPU blocking — just increases overall throughput
```

### Production Setup (both together)
```
PM2 cluster mode (8 processes for throughput)
  └── Each process has a Worker thread pool (4 threads for CPU tasks)
```

### When to Use Which
```
CPU-intensive single request  → Worker Threads (Piscina pool)
Untrusted/isolated work       → Child Process
Scale across all CPU cores    → Cluster (PM2 cluster mode)
```

### Interview Answer
> "CPU work blocks the single Node.js thread — await doesn't help for pure CPU. Worker Threads keep the main thread free by running CPU work on separate threads in the same process. In production I'd use Piscina worker pool capped at CPU core count. Child Process gives more crash isolation but slower startup and no shared memory — better for untrusted work. Clustering forks the entire server across cores for throughput but doesn't fix per-request CPU latency. Production: PM2 cluster mode + Worker thread pool inside each worker."

---

---

## 3. Memory Leaks — Detection and Prevention

### What a leak looks like
```
Healthy: ▲ ▼ ▲ ▼ ▲ ▼  — sawtooth (GC collecting)
Leak:    ▲   ▲   ▲    — steady climb (references held, GC can't collect)
```

### 5 Most Common Causes

**1. Global arrays accumulating without eviction**
```javascript
const requestLog = []; // global, grows forever
app.use((req, res, next) => { requestLog.push(req.body); next(); });
// Fix: cap size
if (requestLog.length > 1000) requestLog.shift();
```

**2. Event listeners not removed**
```javascript
// ❌ New listener per request, never removed — emitter holds res in closure
emitter.on('data', (data) => res.write(data));

// ✅ Remove on client disconnect
const handler = (data) => res.write(data);
emitter.on('data', handler);
req.on('close', () => emitter.off('data', handler));
// Node warns: MaxListenersExceededWarning when this happens
```

**3. Closures holding large objects**
```javascript
// ❌ Closure captures all 500MB even though only summary is needed
const largeData = fetchEntireDataset();
return () => largeData.summary; // largeData never GC'd

// ✅ Extract before closing over
const summary = largeData.summary; // largeData goes out of scope → GC can collect
return () => summary;
```

**4. Forgotten intervals**
```javascript
// ❌ Interval holds reference to 'this' forever
setInterval(() => this.collect(), 5000); // no clearInterval anywhere

// ✅ Always provide cleanup
this.interval = setInterval(() => this.collect(), 5000);
stop() { clearInterval(this.interval); }
```

**5. Unbounded caches (most common in production)**
```javascript
// ❌ Map grows forever — 1M unique users = 1M entries = heap full
const cache = new Map();
cache.set(id, await db.findUser(id)); // never evicted

// ✅ LRU cache with size limit and TTL
import LRU from 'lru-cache';
const cache = new LRU({ max: 500, ttl: 1000 * 60 * 5 }); // auto-evicts
```

### How to Find the Leak

**Step 1 — Confirm with memoryUsage**
```javascript
setInterval(() => {
  const { heapUsed, heapTotal, rss } = process.memoryUsage();
  console.log({ heapUsed: `${Math.round(heapUsed/1024/1024)}MB` });
}, 30000);
// Steady growth over hours = leak confirmed
```

**Step 2 — Heap snapshot comparison (Chrome DevTools)**
```javascript
import v8 from 'v8';
app.get('/debug/heap-snapshot', (req, res) => {
  res.json({ file: v8.writeHeapSnapshot() });
});
// Take snapshot 1 → run under load → take snapshot 2
// Chrome DevTools Memory tab → Compare → sort by "Objects between snapshots"
// Growing objects = your leak
```

**Step 3 — clinic.js (fastest in practice)**
```bash
npm install -g clinic
clinic doctor -- node server.js
npx autocannon http://localhost:3000 -d 60
# Browser report shows memory, CPU, event loop lag — tells you the cause
```

### Production Prevention
```javascript
emitter.setMaxListeners(20);        // crash-fast on listener leak
req.setTimeout(30000, () => res.status(408)); // prevent hanging request accumulation
const cache = new LRU({ max: 1000, ttl: 300_000 }); // never plain Map for caching
const requestMeta = new WeakMap(); // auto GC'd when request obj dies
```

### Interview Answer
> "Steady memory growth means references are held preventing GC. I start with `process.memoryUsage()` logged regularly to confirm heapUsed is growing, then take two heap snapshots 30 minutes apart in Chrome DevTools and compare — growing objects between snapshots are the leak. The five causes: global arrays without eviction, event listeners not removed on disconnect, closures holding large objects when only a small field is needed, forgotten intervals, and unbounded Maps as caches. Prevention: LRU caches with limits, WeakMap for request metadata, cleanup listeners in `req.on('close')`, setMaxListeners to detect early."

---

---

## 4. Production Error Handling

### Custom Error Classes
```javascript
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;           // machine-readable: 'USER_NOT_FOUND'
    this.isOperational = true;  // expected error — app keeps running
    Error.captureStackTrace(this, this.constructor);
  }
}
class NotFoundError extends AppError {
  constructor(resource) { super(`${resource} not found`, 404, 'NOT_FOUND'); }
}
class ValidationError extends AppError {
  constructor(msg) { super(msg, 400, 'VALIDATION_ERROR'); }
}
```

### asyncHandler — eliminates try-catch from every route
```javascript
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

// Routes become clean
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.findUser(req.params.id);
  if (!user) throw new NotFoundError('User'); // semantic throw
  res.json(user);
}));
```

### Centralized Error Handler (last middleware, 4 params)
```javascript
const errorHandler = (err, req, res, next) => {
  if (err.isOperational) {
    // Expected error — safe to tell client what happened
    return res.status(err.statusCode).json({
      status: 'error', code: err.code, message: err.message
    });
  }
  // Programmer bug — log everything, never leak internals
  logger.error('Unhandled error', { err, url: req.url, userId: req.user?.id });
  res.status(500).json({ status: 'error', code: 'INTERNAL_ERROR', message: 'Something went wrong' });
};

app.use(routes);
app.use((req, res, next) => next(new NotFoundError(`Route ${req.method} ${req.url}`))); // 404
app.use(errorHandler); // last
```

### Process-Level Error Handling
```javascript
process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled rejection', { reason });
  server.close(() => process.exit(1)); // graceful shutdown
});

process.on('uncaughtException', (err) => {
  logger.error('Uncaught exception — shutting down', { err });
  server.close(() => process.exit(1)); // programmer bug — must restart
});

process.on('SIGTERM', () => {        // K8s sends this before killing pod
  server.close(() => { db.disconnect(); process.exit(0); });
});
```

### Operational vs Programmer Errors
```
Operational  → expected: 404, 400, 401, DB timeout — recover, keep running
Programmer   → unexpected: TypeError, null reference — log, restart (unknown state)
```

### Interview Answer
> "Raw try-catch in every route duplicates handling, leaks internals, and has no structure. Production pattern: custom error classes with statusCode/code/isOperational, an asyncHandler wrapper that catches async errors and passes to next(), centralized error handler as last middleware (operational errors get clean responses, programmer errors get logged + generic 500), and process.on handlers for anything outside Express with graceful shutdown. Key distinction: operational errors the app recovers from; programmer errors mean unknown state — app should restart."
