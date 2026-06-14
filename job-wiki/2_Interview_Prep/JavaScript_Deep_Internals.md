# JavaScript Deep Internals — MERN Senior/Principal Interview Prep

> Covers concepts tested at 10+ years experience level. Each section includes the concept, real examples, common bugs, and exact phrasing to use in interviews.

[← Back to Index](../README.md)

---

## 1. Closures

### What it is
A closure is a function that retains access to its outer (lexical) scope's variables even after the outer function has finished executing. JS keeps those variables alive via a hidden `[[Environment]]` reference attached to the inner function.

```javascript
function outer() {
  let count = 0;

  function inner() {
    count++;
    console.log(count);
  }

  return inner;
}

const increment = outer(); // outer() is done — but count lives on
increment(); // 1
increment(); // 2
increment(); // 3
```

### Use: Private Variables
```javascript
function createCounter() {
  let count = 0; // private — inaccessible from outside

  return {
    increment: () => ++count,
    decrement: () => --count,
    getCount: () => count,
  };
}

const counter = createCounter();
counter.increment(); // 1
console.log(counter.count); // undefined — truly private
```

### 🐛 Classic Bug — The `var` Loop Trap
```javascript
// ❌ Bug
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
// Prints: 3, 3, 3
// Why: var is function-scoped — one shared i. By the time callbacks run, i = 3.

// ✅ Fix 1 — use let (new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
// Prints: 0, 1, 2

// ✅ Fix 2 — IIFE (pre-ES6 pattern, seen in legacy code)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 1000);
  })(i);
}
// Prints: 0, 1, 2
```

### 🐛 React Stale Closure (senior-level trap)
```javascript
// ❌ Stale closure — count is always 0 inside interval
useEffect(() => {
  const interval = setInterval(() => {
    setCount(count + 1); // count closed over at initial value 0
  }, 1000);
  return () => clearInterval(interval);
}, []);

// ✅ Fix — functional updater doesn't need to close over count
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1);
  }, 1000);
  return () => clearInterval(interval);
}, []);
```

### 🐛 Memory Leak Risk
```javascript
function setup() {
  const largeData = new Array(1000000).fill('x');

  return function() {
    console.log(largeData[0]); // largeData never GC'd as long as this fn exists
  };
}
```

### Interview Answer
> "A closure is a function that retains access to its outer scope's variables through a hidden `[[Environment]]` reference, even after the outer function has returned. The classic bug is the `var` loop trap where all callbacks share one binding — fixed with `let` or an IIFE. In React, stale closures in `useEffect` capture old state values — fixed by using functional updaters or correct dependency arrays."

---

## 2. `this` Binding — 4 Rules

> **Key rule: `this` is determined by how a function is called, not where it's defined.**

### Rule 1 — Default Binding (standalone call)
```javascript
function greet() { console.log(this); }
greet(); // global object (Node: global) — or undefined in strict mode

// Node.js top-level quirk:
console.log(this); // {} — module-level this, NOT global
```

### Rule 2 — Implicit Binding (method call)
```javascript
const user = { name: 'Abhishek', greet() { console.log(this.name); } };
user.greet(); // 'Abhishek' ✅

// 🐛 Losing implicit binding — most common production bug
const fn = user.greet;
fn();                          // undefined — this = global
setTimeout(user.greet, 1000); // undefined — greet is detached
```

### Rule 3 — Explicit Binding (call / apply / bind)
```javascript
function greet(msg) { console.log(`${msg}, ${this.name}`); }
const user = { name: 'Abhishek' };

greet.call(user, 'Hello');       // Hello, Abhishek — args spread
greet.apply(user, ['Hello']);    // Hello, Abhishek — args as array
const bound = greet.bind(user);  // NEW function, this permanently locked
bound('Hi');                     // Hi, Abhishek

// bind cannot be overridden
bound.call({ name: 'Other' }, 'Hi'); // Still "Hi, Abhishek"

// Fix detached callback:
setTimeout(user.greet.bind(user), 1000); // ✅ 'Abhishek'
```

### Rule 4 — new Binding (constructor call)
```javascript
function User(name) {
  this.name = name; // this = the new blank object
}
const u = new User('Abhishek');
// new: 1) creates {}, 2) sets __proto__, 3) calls fn with this={}, 4) returns it
```

### Priority Order
```
new  >  bind/call/apply  >  implicit (obj.fn())  >  default (fn())
```

### Arrow Functions — No Own `this`
Arrow functions lexically inherit `this` from the enclosing scope at definition time. `call`/`apply`/`bind` cannot change their `this`.

```javascript
// ❌ Arrow as object method — wrong
const user = {
  name: 'Abhishek',
  greet: () => console.log(this.name) // this = module scope = {}
};
user.greet(); // undefined

// ✅ Arrow inside a method — fixes callback this
const user = {
  name: 'Abhishek',
  greetLater() {                   // regular fn — this = user
    setTimeout(() => {
      console.log(this.name);      // ✅ inherits from greetLater
    }, 1000);
  }
};
user.greetLater(); // 'Abhishek'
```

### Summary Table
| Call pattern | `this` value |
|---|---|
| `fn()` | global / undefined (strict) |
| `obj.fn()` | obj |
| `fn.call(x)` | x |
| `fn.bind(x)()` | x (permanent) |
| `new Fn()` | new instance |
| Arrow function | enclosing scope's `this` (fixed) |

### Interview Answer
> "`this` is determined at call time by 4 rules: default (global/undefined), implicit (the calling object), explicit (call/apply/bind), and new (fresh instance). Priority: new > bind > implicit > default. Arrow functions have no own `this` — they lexically inherit it from the surrounding scope at definition, making them ideal for callbacks inside methods but wrong for object methods themselves."

---

## 3. Prototypal Inheritance

### `__proto__` vs `prototype` — Cleared Up
- **`__proto__`** — on every object. The actual chain link JS follows during property lookup.
- **`prototype`** — only on functions. Becomes the `__proto__` of instances created with `new`.

```javascript
function User(name) { this.name = name; }
User.prototype.greet = function() { console.log(`Hi, ${this.name}`); };

const u = new User('Abhishek');
u.__proto__ === User.prototype // true ✅
```

### The Prototype Chain
```
u (instance)
  name: 'Abhishek'
  __proto__ ──→ User.prototype
                  greet: fn
                  __proto__ ──→ Object.prototype
                                  hasOwnProperty, toString...
                                  __proto__ ──→ null
```

### Manual Inheritance (pre-class)
```javascript
function Animal(name) { this.name = name; }
Animal.prototype.eat = function() { console.log(`${this.name} eats`); };

function Dog(name, breed) {
  Animal.call(this, name); // inherit properties
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype); // wire up chain
Dog.prototype.constructor = Dog;                 // fix constructor ref
Dog.prototype.bark = function() { console.log(`${this.name} barks`); };

const d = new Dog('Bruno', 'Lab');
d.bark(); // Bruno barks   — Dog.prototype
d.eat();  // Bruno eats    — Animal.prototype (walked up)
```

### `class` is Syntactic Sugar
```javascript
class Animal {
  constructor(name) { this.name = name; }
  eat() { console.log(`${this.name} eats`); }
}
class Dog extends Animal {
  constructor(name, breed) {
    super(name); // = Animal.call(this, name)
    this.breed = breed;
  }
  bark() { console.log(`${this.name} barks`); }
}

// extends does:
Dog.prototype.__proto__ === Animal.prototype // true
Dog.__proto__ === Animal                     // true (static inheritance)
```

### Own vs Inherited Properties
```javascript
const d = new Dog('Bruno', 'Lab');
d.hasOwnProperty('name');  // true — set directly on d
d.hasOwnProperty('bark');  // false — on Dog.prototype
d.hasOwnProperty('eat');   // false — on Animal.prototype

for (let k in d) console.log(k); // name, breed, bark, eat (whole chain)
Object.keys(d);                  // ['name', 'breed'] (own only)
```

### Interview Answer
> "Every JS object has a `__proto__` link to its prototype. Property lookup walks this chain until found or null. Functions also have `prototype` — this becomes the `__proto__` of instances created with `new`. `class`/`extends` is pure syntax sugar over this: `extends` sets `Dog.prototype.__proto__ = Animal.prototype`. Key distinction: `__proto__` is on every object and is the actual link; `prototype` is only on functions and is what gets assigned as `__proto__` on `new`."

---

## 4. Async/Await Internals & Promises

### What async/await compiles to
`async/await` is syntactic sugar over Promises + Generators. The engine transforms each `await` into a `.then` chain:

```javascript
// What you write:
async function fetchData() {
  const a = await callA();
  const b = await callB();
  return a + b;
}

// What it becomes internally:
function fetchData() {
  return new Promise(resolve => {
    callA().then(a => {
      callB().then(b => {
        resolve(a + b);
      });
    });
  });
}
```

### Sequential vs Parallel — The Core Interview Question
```javascript
// ❌ Sequential — 500ms total (300 + 200)
async function slow() {
  const a = await callA(); // waits 300ms
  const b = await callB(); // only starts after callA
  return a + b;
}

// ✅ Parallel — 300ms total (max of both)
async function fast() {
  const [a, b] = await Promise.all([callA(), callB()]);
  return a + b;
}

// ❌ Common mistake — still sequential despite Promise.all!
await Promise.all([await callA(), await callB()]); // await inside = sequential
```

### The Microtask Queue
```
Call Stack → Microtask Queue (Promises, async resumes) → Macrotask Queue (setTimeout, I/O)
Rule: after every task, drain ALL microtasks before next task
```

```javascript
console.log('1');
setTimeout(() => console.log('2'), 0); // macrotask
Promise.resolve().then(() => console.log('3')); // microtask
console.log('4');
// Output: 1, 4, 3, 2
```

### Async/Await Execution Order (common interview question)
```javascript
async function main() {
  console.log('A');
  await Promise.resolve(); // pause — resume queued as microtask
  console.log('B');
}
console.log('start');
main();
console.log('end');
// Output: start, A, end, B
// Why: main pauses at await, 'end' runs, then microtask resumes main
```

### Promise Combinators
```javascript
// Promise.all — all must succeed, first rejection kills it
const [a, b] = await Promise.all([callA(), callB()]);

// Promise.allSettled — all run, never rejects, get status per result
const results = await Promise.allSettled([callA(), callB()]);
results.forEach(r => r.status === 'fulfilled' ? use(r.value) : log(r.reason));

// Promise.race — first to settle wins (use for timeouts)
const result = await Promise.race([callA(), timeout(5000)]);

// Promise.any — first to FULFILL wins, only rejects if ALL reject (CDN fallbacks)
const result = await Promise.any([mirror1(), mirror2(), mirror3()]);
```

### Error Handling Pattern
```javascript
// ✅ Production pattern — return errors as values, don't throw
async function fetchData() {
  try {
    const [a, b] = await Promise.all([callA(), callB()]);
    return { data: a + b, error: null };
  } catch (err) {
    logger.error('fetchData failed', { err });
    return { data: null, error: err.message };
  }
}
```

### Interview Answer
> "`async/await` is sugar over Promises which use the microtask queue. An `await` pauses the function and queues its continuation as a microtask — which runs before any setTimeout. Sequential awaits are waterfall (sum of all durations); `Promise.all` starts all promises simultaneously so total time equals the slowest. `allSettled` is for when you need all results regardless of failures; `race` is for timeouts; `any` is for fastest-wins scenarios."

---

## 5. The `var` Loop + Promise.all Combo Trap

```javascript
async function run() {
  const results = [];
  for (var i = 0; i < 3; i++) {
    results.push(new Promise(resolve => {
      setTimeout(() => resolve(i), i * 100);
    }));
  }
  const values = await Promise.all(results);
  console.log(values);
}
run();
// Output: [3, 3, 3]
// Why: var = one shared i. Loop ends with i=3. All setTimeout callbacks
// run after loop, all see i=3. The timeout delay (i*100) is irrelevant
// to the resolved value.
// Fix: change var to let — each iteration gets its own i binding.
```

---

## 6. Generators & Iterators

### The Core Idea
Normal functions run to completion. Generators can **pause and resume** using `yield`. They produce values lazily — one at a time, on demand.

```javascript
function* counter() {
  yield 1;   // pause, hand back 1
  yield 2;   // pause, hand back 2
  yield 3;   // pause, hand back 3
}

const gen = counter(); // does NOT run — returns generator object
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }
```

### Infinite ID Generator (interview question)
```javascript
function* idGenerator(prefix = 'req') {
  let id = 1;
  while (true) {           // infinite loop is safe — yield pauses it
    yield `${prefix}-${Date.now()}-${id++}`;
  }
}

const getId = idGenerator();
getId.next().value; // 'req-1718345600000-1'
getId.next().value; // 'req-1718345600000-2'

// Real use — Express request ID middleware
app.use((req, res, next) => {
  req.id = getId.next().value;
  next();
});
```

### Generators are Iterators — works with for...of, spread, destructuring
```javascript
function* range(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

for (const n of range(1, 5)) console.log(n); // 1,2,3,4,5
const nums = [...range(1, 5)];               // [1,2,3,4,5]
const [a, b, c] = range(10, 20);            // 10, 11, 12
```

### Two-Way Communication (yield receives values too)
```javascript
function* calculator() {
  const x = yield 'Give me first number';
  const y = yield 'Give me second number';
  return x + y;
}
const calc = calculator();
calc.next();     // { value: 'Give me first number', done: false }
calc.next(10);   // x = 10. { value: 'Give me second number', done: false }
calc.next(20);   // y = 20. { value: 30, done: true }
```
This two-way mechanism is exactly what async/await uses — the engine passes resolved Promise values back in via `.next(resolvedValue)`.

### Real Node.js Use Cases

**1. Paginated DB reads — process millions of records without OOM**
```javascript
async function* getUsers(db) {
  let page = 0;
  const pageSize = 100;
  while (true) {
    const users = await db.collection('users').find()
      .skip(page * pageSize).limit(pageSize).toArray();
    if (users.length === 0) return;
    for (const user of users) yield user;
    page++;
  }
}

for await (const user of getUsers(db)) {
  await sendEmail(user); // processes 10M users, max 100 in memory at once
}
```

**2. Rate-limited API calls**
```javascript
async function* rateLimitedFetch(urls, delayMs = 1000) {
  for (const url of urls) {
    yield await fetch(url).then(r => r.json());
    await new Promise(r => setTimeout(r, delayMs));
  }
}
```

### Why Async/Await is Built on Generators
```javascript
// The engine treats async functions as generator runners:
// async function → generator where await = yield
// On each yield (await), the runner waits for the Promise,
// then feeds the resolved value back via .next(resolvedValue)

// This:
async function fetchData() { const a = await callA(); return a; }

// Is equivalent to:
runAsync(function* fetchData() { const a = yield callA(); return a; });
```

### Quick Mental Model
```
Normal function:  run → return                              (one shot)
Generator:        run → pause → resume → pause → resume → done  (resumable)
Async function:   generator + auto-resume on Promise resolve
```

### Interview Answer
> "A generator is a pausable function using `function*` and `yield`. It returns a generator object; calling `.next()` runs it to the next `yield`. This enables lazy evaluation — values produced on demand. In Node.js I use async generators for paginated DB reads to avoid loading millions of records into memory, for unique ID generation, and for rate-limited API calls. Generators are also the foundation of async/await — the engine transforms `await` into `yield` and feeds resolved values back via `.next()`."

---

## 7. Map, Set, WeakMap, WeakSet

### Map vs Object
| | `Object {}` | `Map` |
|---|---|---|
| Key types | strings & symbols only | any type |
| Order | not guaranteed for int keys | guaranteed insertion order |
| Size | O(n) via Object.keys() | O(1) via map.size |
| Performance | fast for small static keys | better for frequent add/delete |

```javascript
// Object fails with object keys
const obj = {};
const key = { id: 1 };
obj[key] = 'val'; // key becomes '[object Object]' — wrong!

// Map handles object keys correctly
const map = new Map();
map.set(key, 'val');
map.get(key); // 'val' ✅
```

**O(1) lookup from large dataset — your instinct was right:**
```javascript
const users = await db.collection('users').find().toArray();
// ❌ O(n) on every call
users.find(u => u._id === id);
// ✅ Build once, O(1) forever
const userMap = new Map(users.map(u => [u._id, u]));
userMap.get(id);
```

**Request deduplication:**
```javascript
const inFlightRequests = new Map();

async function fetchWithDedup(url) {
  if (inFlightRequests.has(url)) return inFlightRequests.get(url);
  const promise = fetch(url).then(r => r.json());
  inFlightRequests.set(url, promise);
  try { return await promise; } finally { inFlightRequests.delete(url); }
}
```

### Set vs Array
```javascript
// Deduplication
const unique = [...new Set([1, 2, 2, 3, 3])]; // [1, 2, 3]

// O(1) membership check — blocked users, feature flags
const blocked = new Set(['user1', 'user2']);
blocked.has(req.userId); // O(1) vs array.includes() which is O(n)
```

### WeakMap — Prevents Memory Leaks
Keys must be objects. Held weakly — when the key object is GC'd, the entry disappears too. A regular Map would hold the key alive forever.

```javascript
// ✅ Per-request metadata without leaking memory
const requestMeta = new WeakMap();

app.use((req, res, next) => {
  requestMeta.set(req, { startTime: Date.now(), userId: req.headers['x-user-id'] });
  next();
});
// When req is done and GC'd, WeakMap entry disappears automatically
// Regular Map here = entries accumulate = memory leak
```

### When to Use Each
```
Object   → string keys, JSON serializable, simple config
Map      → any key type, O(1) size, ordered, frequent mutations
Set      → unique values, O(1) has(), deduplication, allowlists
WeakMap  → cache/metadata on objects that must not block GC
WeakSet  → track processed objects without blocking GC
```

### Interview Answer
> "Use Map over objects for non-string keys, guaranteed order, and frequent add/delete — common pattern is building an ID-to-object lookup from DB results for O(1) access. Set for unique values and fast membership checks. WeakMap for metadata on objects that shouldn't prevent garbage collection — per-request caches in Express middleware. A regular Map there accumulates entries indefinitely and leaks memory."

---

## What's Next

- [ ] Phase 2 — React Advanced (Fiber, Concurrent Mode, patterns)
- [ ] Phase 3 — Node.js Advanced (Streams, Worker Threads, memory leaks)
- [ ] Phase 4 — MongoDB Advanced (schema design, aggregation, sharding)
- [ ] Phase 5 — MERN System Design

---

## 8. Design Patterns in JavaScript (MERN Context)

### Factory — LLM Provider Selection
Creates objects without exposing creation logic. Caller says *what*, factory decides *how*.

```javascript
class LLMProviderFactory {
  static create(providerName, config) {
    switch (providerName) {
      case 'anthropic': return new AnthropicProvider(config);
      case 'openai':    return new OpenAIProvider(config);
      case 'gemini':    return new GeminiProvider(config);
      default: throw new Error(`Unknown provider: ${providerName}`);
    }
  }
}
const provider = LLMProviderFactory.create(req.body.model, config);
// Add new provider = add one case. Nothing else changes. (Open/Closed Principle)
```

### Singleton — DB/Redis/Kafka Clients
One instance shared across the app. Node module caching makes this natural.

```javascript
// db.js — module-level export IS a singleton (Node caches imports)
const client = new MongoClient(process.env.MONGO_URI);
export default client; // same instance everywhere it's imported
```
Your Langfuse client, Redis client, Kafka producer — all singletons.

### Observer — Kafka / EventEmitter
Objects subscribe to events, get notified when they fire. Kafka is the distributed version.

```javascript
class AgentOrchestrator extends EventEmitter {
  async runAgent(agentId, input) {
    this.emit('agent:start', { agentId });
    const result = await this.executeAgent(agentId, input);
    this.emit('agent:complete', { agentId, result });
    return result;
  }
}

orchestrator.on('agent:complete', ({ tokens }) => langfuse.track(tokens));
orchestrator.on('agent:complete', ({ agentId }) => quotaService.deduct(agentId));
orchestrator.on('agent:complete', ({ agentId }) => websocket.push(agentId, 'done'));
// Add new behavior = add a listener. Zero changes to orchestrator.
```

### Saga — Multi-Service Transactions
Long-running distributed workflows where each step has a compensating rollback action.

```javascript
class AgentDeploymentSaga {
  async execute(agentConfig) {
    const compensations = [];
    try {
      const agent = await agentService.create(agentConfig);
      compensations.push(() => agentService.delete(agent.id));

      await vectorDB.createNamespace(agent.id);
      compensations.push(() => vectorDB.deleteNamespace(agent.id));

      await k8sService.deployAgent(agent.id);
      compensations.push(() => k8sService.removeDeployment(agent.id));

      return { success: true, agentId: agent.id };
    } catch (err) {
      for (const compensate of compensations.reverse()) {
        await compensate().catch(e => logger.error('Compensation failed', e));
      }
      throw err;
    }
  }
}
// Why not DB transaction? Steps touch MongoDB, K8s API, Stripe — no shared tx boundary.
// Saga gives eventual consistency with explicit rollback.
```

**Two types:**
- **Orchestration** — central saga controls all steps (your code above). Easier to debug.
- **Choreography** — services emit/listen to Kafka events in sequence. Your Agent Builder with Kafka.

### Strategy — LLM Routing Rules
Interchangeable algorithms injected at runtime.

```javascript
class CostOptimizedStrategy {
  selectProvider(req) { return req.complexity < 0.5 ? 'haiku' : 'sonnet'; }
}
class LatencyOptimizedStrategy {
  selectProvider(req) { return 'gpt-4o-mini'; }
}
class ComplianceStrategy {
  selectProvider(req) { return req.orgCountry === 'EU' ? 'anthropic-eu' : 'claude-3'; }
}

const router = new TokenRouter(
  team.priority === 'cost' ? new CostOptimizedStrategy() : new LatencyOptimizedStrategy()
);
```

### Decorator — NestJS Guards, Interceptors, Audit Logs
Adds behavior without modifying the original class.

```javascript
function AuditLog(target, methodName, descriptor) {
  const original = descriptor.value;
  descriptor.value = async function(...args) {
    const start = Date.now();
    try {
      const result = await original.apply(this, args);
      logger.info(`${methodName} ok`, { ms: Date.now() - start });
      return result;
    } catch (err) {
      logger.error(`${methodName} failed`, { err }); throw err;
    }
  };
  return descriptor;
}

class AgentService {
  @AuditLog
  async deployAgent(config) { ... } // logging added without touching this method
}
```

### Pattern → Your Project Map
| Pattern | Where you used it |
|---|---|
| Factory | LLM provider selection in AI Gateway |
| Singleton | Langfuse, Redis, Kafka producer at startup |
| Observer | Kafka events between Agent Builder services |
| Saga | Agent deployment across MongoDB + K8s + Billing |
| Strategy | LLM routing rules per team (cost/latency/compliance) |
| Decorator | NestJS guards, interceptors, audit logging |

### Interview Answer
> "In the AI Gateway I used Factory for LLM provider instantiation — adding a provider is one line, nothing else changes. Singleton for shared clients — Node module caching makes this natural. Strategy for routing rules injected per team at runtime. In the Agent Builder, Observer via Kafka — orchestrator emits, billing/observability/k8s react independently. For multi-step deployments across systems without a shared transaction boundary, I used Saga with compensating actions for rollback."

---

## 9. Design Patterns — Functional Style (no classes needed)

In modern Node.js you rarely need classes. Every pattern works with plain functions.

### Factory → function returning an object
```javascript
function createLLMProvider(name, config) {
  const providers = {
    anthropic: (cfg) => ({ complete: (p) => callAnthropic(p, cfg) }),
    openai:    (cfg) => ({ complete: (p) => callOpenAI(p, cfg) }),
    gemini:    (cfg) => ({ complete: (p) => callGemini(p, cfg) }),
  };
  if (!providers[name]) throw new Error(`Unknown provider: ${name}`);
  return providers[name](config);
}
```

### Singleton → module export (Node caches it)
```javascript
// db.js
const client = new MongoClient(uri);
export default client; // same instance everywhere — simplest singleton
```

### Observer → closure-based event bus
```javascript
function createEventBus() {
  const listeners = {};
  return {
    on(event, fn)   { (listeners[event] ??= []).push(fn); },
    off(event, fn)  { listeners[event] = (listeners[event] || []).filter(l => l !== fn); },
    emit(event, data) { (listeners[event] || []).forEach(fn => fn(data)); }
  };
}
// listeners is private — only accessible via on/off/emit
```

### Strategy → functions are already first-class
```javascript
const costStrategy     = (req) => req.complexity < 0.5 ? 'haiku' : 'sonnet';
const latencyStrategy  = (req) => 'gpt-4o-mini';

function routeRequest(req, strategy) { return strategy(req); }

const strategy = team.priority === 'cost' ? costStrategy : latencyStrategy;
routeRequest(req, strategy); // swap at runtime, no classes needed
```

### Decorator → Higher Order Function (HOF)
```javascript
function withAuditLog(fn, name) {
  return async function(...args) {
    const start = Date.now();
    try {
      const result = await fn(...args);
      logger.info(`${name} ok`, { ms: Date.now() - start });
      return result;
    } catch (err) {
      logger.error(`${name} failed`, { err }); throw err;
    }
  };
}

const deployAgent = withAuditLog(async (config) => { /* logic */ }, 'deployAgent');
// Express middleware is exactly this pattern
```

### When class vs function
```
Use CLASS  → NestJS (DI requires it), inheritance needed, multiple stateful instances
Use FUNCTION → Node.js services, utilities, closures for private state, simpler testing
```

---

## 10. Intermediate-Level Q&A (4–7 Years Experience)

> These questions sound simple but interviewers use them to expose gaps. Read these before any technical screen.

---

### Q1 — Hoisting & Temporal Dead Zone

**Q: What is hoisting? How do var, let, and const differ?**

`var` declarations are hoisted to the top of their function scope and initialized to `undefined`. `let` and `const` are hoisted to the block scope but NOT initialized — accessing them before the declaration throws a `ReferenceError`. The gap between the start of the block and the declaration line is the **Temporal Dead Zone (TDZ)**.

```javascript
console.log(a); // undefined — var hoisted + initialized
console.log(b); // ReferenceError — TDZ for let
var a = 1;
let b = 2;

// Function declarations are fully hoisted (name + body)
greet(); // works
function greet() { console.log('hello'); }

// Function expressions are NOT fully hoisted
sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log('hi'); };
// var sayHi is hoisted as undefined, calling undefined() throws
```

**Interview answer:**
> "`var` is hoisted and initialized to `undefined` — you can reference it before the line, it just returns undefined. `let` and `const` are also hoisted but not initialized, so accessing them before declaration throws a ReferenceError — that window is the Temporal Dead Zone. Function declarations are fully hoisted including their body; function expressions are not."

---

### Q2 — Destructuring & Spread/Rest

**Q: Walk me through destructuring with defaults. What's the difference between spread and rest?**

```javascript
// Array destructuring with defaults and skipping
const [a, , b = 10] = [1, 2];  // a=1, skip index 1, b=10 (default)

// Object destructuring with rename and default
const { name: userName = 'Guest', age = 0 } = { name: 'Abhishek' };
// userName = 'Abhishek', age = 0 (field doesn't exist)

// Nested destructuring
const { address: { city } } = { address: { city: 'Mumbai' } };

// REST — collects remaining items
function sum(first, ...rest) {  // rest = array of remaining args
  return rest.reduce((acc, n) => acc + n, first);
}
sum(1, 2, 3, 4); // 10

// SPREAD — expands into individual items
const arr1 = [1, 2], arr2 = [3, 4];
const merged = [...arr1, ...arr2]; // [1, 2, 3, 4]

// Spread for shallow copy (common bug)
const original = { a: 1, nested: { b: 2 } };
const copy = { ...original };
copy.nested.b = 99; // mutates original.nested.b too — shallow copy only
```

**Interview answer:**
> "Destructuring lets you extract values from arrays/objects with rename and default support. Rest collects remaining elements into an array — it's a parameter that must come last. Spread expands an iterable in-place — useful for merging, copying, and passing arrays as function arguments. Key gotcha: spread is a SHALLOW copy — nested objects are still shared references."

---

### Q3 — Array Methods: map, filter, reduce

**Q: Explain map, filter, and reduce with a real example. What's the difference between map and forEach?**

```javascript
const orders = [
  { id: 1, status: 'paid',    amount: 100, category: 'food' },
  { id: 2, status: 'pending', amount: 200, category: 'tech' },
  { id: 3, status: 'paid',    amount: 150, category: 'food' },
];

// filter — returns matching items (same structure, subset)
const paid = orders.filter(o => o.status === 'paid');
// [{ id:1 ... }, { id:3 ... }]

// map — transforms every item (same length, different shape)
const amounts = orders.map(o => o.amount);
// [100, 200, 150]

// reduce — collapse to a single value
const total = orders.reduce((sum, o) => sum + o.amount, 0);
// 450

// Combine: sum of paid food orders
const paidFoodTotal = orders
  .filter(o => o.status === 'paid' && o.category === 'food')
  .reduce((sum, o) => sum + o.amount, 0);
// 250

// map vs forEach
// map returns a NEW array — use when you need the result
// forEach returns undefined — use for side effects only
const doubled = orders.map(o => o.amount * 2); // new array
orders.forEach(o => console.log(o.id));         // side effect, returns undefined
```

**Interview answer:**
> "`filter` keeps items that pass a predicate — same structure, fewer items. `map` transforms every item — same count, different shape. `reduce` collapses the whole array to a single value — most flexible but reads less clearly, so I use filter+map when possible. `map` vs `forEach`: map returns a new array; forEach returns undefined and is for side effects only — assigning forEach's result is a common bug."

---

### Q4 — Optional Chaining & Nullish Coalescing

**Q: What do `?.` and `??` solve? When would `??` behave differently from `||`?**

```javascript
// Problem: TypeError: Cannot read property of undefined
const city = user.address.city; // throws if address is null/undefined

// ✅ Optional chaining — short-circuits to undefined instead of throwing
const city = user?.address?.city;          // undefined if any link is null/undefined
const firstTag = post?.tags?.[0];         // works with arrays
const result = obj?.method?.();           // works with function calls

// Problem with || — treats 0, '', false as falsy → uses fallback unintentionally
const count = user.count || 10; // user.count = 0 → returns 10 (wrong!)

// ✅ Nullish coalescing — only null/undefined triggers fallback
const count = user.count ?? 10; // 0 → returns 0 ✅
const name  = user.name ?? 'Guest'; // '' (empty string) → returns '' ✅

// Real MERN example — agent config with safe defaults
const maxTokens = agent?.config?.maxTokens ?? 1000;
const providers = agent?.enabledProviders ?? ['anthropic'];
```

| Operator | Falsy values that trigger fallback |
|---|---|
| `\|\|` | `null, undefined, 0, '', false, NaN` |
| `??` | `null, undefined` only |

**Interview answer:**
> "Optional chaining short-circuits to `undefined` instead of throwing when a chain link is null or undefined — essential when working with API responses where fields may be absent. Nullish coalescing `??` provides a default only for null/undefined, unlike `||` which also replaces `0`, empty string, and `false`. In practice I always use `??` for defaults on numeric and boolean config values."

---

### Q5 — `typeof`, `instanceof`, and `==` vs `===`

**Q: What does typeof null return and why? When does `==` cause bugs?**

```javascript
// typeof quirks
typeof null        // 'object' — historical bug in JS, never fixed (backward compat)
typeof undefined   // 'undefined'
typeof []          // 'object' — arrays are objects
typeof function(){} // 'function' — only callable objects get this
typeof NaN         // 'number' — Not a Number is typed as number

// Correct type checks
Array.isArray([])  // true — use this not typeof
Number.isNaN(NaN)  // true — use this not isNaN() (isNaN('') = true, wrong)

// instanceof — checks prototype chain
[] instanceof Array   // true
[] instanceof Object  // true (Array.prototype.__proto__ === Object.prototype)

// == coercion traps
null == undefined   // true — they equal each other
null == 0           // false
'' == false         // true — both coerced to 0
[] == false         // true — [] → 0, false → 0
[] == ![]           // true (this one breaks everyone's brain)

// Use === always except one legitimate use case:
// null == undefined covers both in one check:
if (value == null) // true for both null and undefined — intentional use of ==
```

**Interview answer:**
> "`typeof null` returns `'object'` — a legacy bug from 1995 that can't be fixed without breaking the web. For arrays use `Array.isArray`. For NaN use `Number.isNaN` — `typeof NaN` is `'number'`. Use `===` always; `==` coerces types in non-obvious ways — empty string and false both become 0 and compare equal. The one legitimate use: `value == null` checks for both null and undefined in one expression."

---

### Q6 — `Object` Utility Methods

**Q: How do you merge objects, clone them, and iterate over their keys?**

```javascript
const defaults = { model: 'claude-3', maxTokens: 1000, temperature: 0.7 };
const overrides = { maxTokens: 2000, stream: true };

// Merge (right wins)
const config = { ...defaults, ...overrides };
// { model: 'claude-3', maxTokens: 2000, temperature: 0.7, stream: true }

// Object.assign — same as spread, mutates first arg
Object.assign({}, defaults, overrides);

// Iterate
Object.keys(config)    // ['model', 'maxTokens', 'temperature', 'stream']
Object.values(config)  // ['claude-3', 2000, 0.7, true]
Object.entries(config) // [['model', 'claude-3'], ['maxTokens', 2000], ...]

// Transform
const redacted = Object.fromEntries(
  Object.entries(config).filter(([k]) => k !== 'apiKey')
);

// Freeze — prevents mutation (use for config objects)
const LIMITS = Object.freeze({ maxFileSize: 500, maxAgents: 10 });
LIMITS.maxAgents = 999; // silently fails (TypeError in strict mode)

// Check own property (safe vs hasOwnProperty)
Object.hasOwn(config, 'model') // true — recommended over config.hasOwnProperty()
// hasOwnProperty can be overridden; Object.hasOwn cannot
```

**Interview answer:**
> "For merging I use spread `{ ...a, ...b }` — right-most value wins, clean and readable. `Object.entries` gives `[key, value]` pairs making it easy to filter or transform with array methods. `Object.freeze` prevents mutation — useful for shared config constants. `Object.hasOwn` is safer than `hasOwnProperty` because it can't be shadowed by a property named `hasOwnProperty`."

---

### Q7 — Short-Circuit Evaluation & Common Gotchas

**Q: What is short-circuit evaluation? What are common bugs with it in React?**

```javascript
// AND (&&) — returns first falsy OR last truthy
true && 'hello'   // 'hello'
false && 'hello'  // false
null && doWork()  // null — doWork never called

// OR (||) — returns first truthy OR last falsy
null || 'default'  // 'default'
'value' || 'default' // 'value'

// ❌ React rendering bug with 0
const count = 0;
return <div>{count && <Component />}</div>;
// Renders: <div>0</div> — not nothing! 0 is falsy but is a valid React child

// ✅ Fix — coerce to boolean
return <div>{count > 0 && <Component />}</div>;
return <div>{!!count && <Component />}</div>;
return <div>{Boolean(count) && <Component />}</div>;

// ❌ Another gotcha — false string
const show = 'false'; // string, not boolean
show && <Component />  // renders! 'false' is truthy

// Comma operator (rare but seen in minified code)
const x = (1, 2, 3); // x = 3 — evaluates all, returns last

// Optional chaining with short-circuit
const log = config?.debug && console.log; // null if no config
```

**Interview answer:**
> "Short-circuit evaluation means `&&` stops at the first falsy value and `||` stops at the first truthy. The classic React bug: `{count && <Component />}` renders a literal `0` when count is zero because 0 is falsy but React renders numbers — fix with `count > 0 &&`. Similarly, `||` for defaults replaces `0` and empty string — use `??` instead for those cases."
