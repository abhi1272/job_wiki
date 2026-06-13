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

## What's Next

- [ ] Phase 2 — React Advanced (Fiber, Concurrent Mode, patterns)
- [ ] Phase 3 — Node.js Advanced (Streams, Worker Threads, memory leaks)
- [ ] Phase 4 — MongoDB Advanced (schema design, aggregation, sharding)
- [ ] Phase 5 — MERN System Design
