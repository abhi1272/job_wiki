# React Advanced Internals — MERN Senior/Principal Interview Prep

[← Back to Index](../README.md)

---

## 1. Virtual DOM, Fiber & Reconciliation

### Virtual DOM
A lightweight JS object tree mirroring the real DOM. On state change React builds a new Virtual DOM, diffs it against the previous (reconciliation), computes minimal changes, then batches them to real DOM in one operation. Batching matters because real DOM ops trigger layout/paint/composite — expensive.

### The Problem Fiber Solved
Old stack reconciler was **recursive and synchronous** — walked the whole tree in one shot using the call stack. Couldn't be paused. Large trees blocked the main thread 100-200ms → frozen UI, dropped animations.

### Fiber — Interruptible Reconciliation
Rewrote the reconciler as an **iterative linked-list traversal**. Each component = a Fiber node (plain JS object):

```javascript
{
  type: 'div',
  child: FiberNode,      // first child
  sibling: FiberNode,    // next sibling
  return: FiberNode,     // parent
  pendingProps: {},
  memoizedState: {},
  effectTag: 'UPDATE',   // INSERT | UPDATE | DELETE
  alternate: FiberNode,  // previous version (double buffering)
}
```

React walks this tree iteratively, can stop at any node, yield to browser, resume later.

### Two Phases

```
RENDER PHASE  (interruptible — can pause, abort, restart)
  - Walk fiber tree, call render(), run hooks
  - Diff new vs old fibers
  - Build effect list (what DOM changes are needed)

COMMIT PHASE  (synchronous — always runs to completion)
  - Apply all DOM mutations at once
  - Run useLayoutEffect (synchronous, before paint)
  - Run useEffect (after paint)
```

Only the commit phase touches the real DOM — so the DOM is never half-updated.

### Diffing Algorithm — O(n) via Two Heuristics

**1. Same type = update, different type = destroy + remount**
```jsx
// div → span: React destroys entire subtree, creates fresh
<div><Counter /></div>  →  <span><Counter /></span>  // full remount!
```

**2. Keys identify elements across renders**
```jsx
// ❌ Index as key — removing item 0 shifts all keys → re-renders everything
items.map((item, i) => <Item key={i} {...item} />)

// ✅ Stable ID — React matches by identity, skips unchanged items
items.map(item => <Item key={item.id} {...item} />)
```

### Double Buffering
Fiber keeps two trees: `current` (on screen) and `workInProgress` (being built). On commit, workInProgress becomes current. React can abort a render by discarding workInProgress — current is never touched until commit.

### React 18 Concurrent Features (built on Fiber)

**`useTransition`** — mark updates as non-urgent, let urgent ones interrupt:
```jsx
const [isPending, startTransition] = useTransition();

function handleSearch(query) {
  setInputValue(query);           // urgent — runs immediately
  startTransition(() => {
    setSearchResults(filter(data, query)); // non-urgent — can be interrupted
  });
}
// Fast typing no longer freezes the results list
```

**`useDeferredValue`** — defer a value, show stale UI while new one renders:
```jsx
const deferredQuery = useDeferredValue(query);
// query updates immediately (input stays responsive)
// ExpensiveList re-renders with deferredQuery when browser is idle
return <ExpensiveList query={deferredQuery} />;
```

### Interview Answer
> "Virtual DOM is a JS object tree. On state change React builds a new one, diffs against the old using O(n) heuristics (same type = update, different type = remount, keys track identity), then batches minimal DOM changes in the commit phase. Fiber rewrote reconciliation from recursive/sync to interruptible iterative traversal — splitting work into a pausable render phase and a synchronous commit phase. This makes React 18 concurrent features possible: useTransition lets urgent updates interrupt non-urgent renders, useDeferredValue keeps the UI responsive while expensive renders catch up."

---

---

## 2. React Performance — memo, useCallback, useMemo

### Why React.memo alone fails
`React.memo` does shallow prop comparison. Functions and objects created inline produce new references every render — memo is bypassed.

```javascript
// Each render creates new reference — memo comparison fails
const handleClick = () => console.log('clicked'); // 0x001, 0x002, 0x003...
const data = processLargeArray(items);            // new array every render
```

### The Fix
```jsx
function ParentComponent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // stable reference — only recreates if deps change

  const data = useMemo(() => processLargeArray(items), [items]);
  // only recomputes when items changes, not on count/name changes

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <input value={name} onChange={e => setName(e.target.value)} />
      <ExpensiveChild data={data} onClick={handleClick} />
    </>
  );
}

const ExpensiveChild = React.memo(({ data, onClick }) => { ... });
// Now skips re-render on count/name changes ✅
```

### useCallback vs useMemo — same thing internally
```javascript
useCallback(fn, deps) === useMemo(() => fn, deps)
// Both cache a value in a fiber slot, recompute only when deps change
```

### useRef — stable reference that reads current value
```javascript
// ❌ useCallback with deps recreates on every count change → memo bypassed
const handleClick = useCallback(() => console.log(count), [count]);

// ✅ useRef always reads current value, stable reference
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }); // keep in sync

const handleClick = useCallback(() => {
  console.log(countRef.current); // always fresh
}, []); // stable reference
```

### When NOT to use (interviewers ask this)
```
❌ useMemo for trivial computation — a + b costs less than memo overhead
❌ useCallback on DOM element handlers — <button> doesn't do prop comparison
❌ React.memo on every component — only helps components that re-render with same props frequently
✅ Only optimize after measuring with React DevTools Profiler
```

### Rules of thumb
```
useMemo     → expensive computation (>1ms) on unrelated state changes
useCallback → function passed to a React.memo child
React.memo  → component that frequently receives same props
```

### Interview Answer
> "`React.memo` does shallow comparison — fails when props are inline functions or objects because every render produces a new reference. `useCallback` memoizes the function reference, only recreating when deps change. `useMemo` memoizes computed values. The trap is overuse — wrapping trivial computations or non-memoized children adds overhead with zero benefit. I reach for these only after measuring with React DevTools Profiler."

---

---

## 3. State Management — Redux vs Zustand vs Context

### The Core Problem with Context
Context re-renders **every consumer** when value changes — even consumers that don't use the changed part. No selector mechanism.

```jsx
// CartContext value changes → ProfilePage re-renders too, even though it only uses user
const { user } = useContext(AppContext); // re-renders on EVERY context change
```

### Redux — Use When
- State shared across many distant components
- Frequent complex updates needing predictable logic
- Need DevTools (time-travel debugging, action replay)
- Large team — strict unidirectional flow
- Derived state computed across multiple slices

```javascript
// Redux Toolkit — modern Redux
const cartSlice = createSlice({
  name: 'cart',
  initialState: [],
  reducers: {
    addItem: (state, action) => { state.push(action.payload); }, // Immer-safe
    removeItem: (state, action) => state.filter(i => i.id !== action.payload),
  }
});

// Granular subscription — only re-renders when cart.length changes
const cartCount = useSelector(state => state.cart.length);
```

### Zustand — Use When
Simpler than Redux, zero boilerplate, no Provider needed, same granular subscriptions.
> Note: Zustand does NOT persist by default — persistence is optional middleware.

```javascript
const useCartStore = create((set, get) => ({
  items: [],
  addItem: (item) => set(state => ({ items: [...state.items, item] })),
  total: () => get().items.reduce((sum, i) => sum + i.price, 0),
}));

// No Provider needed anywhere in the tree
const count = useCartStore(state => state.items.length); // granular selector

// Persistence middleware (survives page refresh)
const useCartStore = create(persist(
  (set) => ({ items: [], addItem: (item) => set(s => ({ items: [...s.items, item] })) }),
  { name: 'cart-storage' } // saves to localStorage
));
```

### Context — Use When (it IS right for some things)
```
✅ Auth/session    — changes only on login/logout
✅ Theme           — changes at most once per session
✅ Locale/i18n     — set once, never changes
✅ DI/services     — pass API clients, config down the tree
❌ Cart            — changes on every add/remove → all consumers re-render
❌ Search/filters  — changes on every keystroke → everything re-renders
```

### Decision Framework
```
Rarely changes (auth, theme)          → Context
Frequently changes:
  ├── Complex, large team, DevTools   → Redux Toolkit
  └── Simpler, less boilerplate       → Zustand
```

### Real App Uses All Three
```
Context  → auth user, orgId, theme
Zustand  → UI state (modals, filters, editor drafts), persisted cart
Redux    → complex domain state, multi-slice derived state
```

### Interview Answer
> "Context re-renders every consumer on any value change — fine for rarely-changing state like auth and theme, wrong for frequent updates. Redux gives granular selector subscriptions, DevTools, and strict unidirectional flow — right for complex state in large teams. Zustand gives the same granular subscriptions with zero boilerplate and no Provider — right for medium complexity. Persistence is optional Zustand middleware, not a default. Most production apps use all three for different concerns."

---

---

## 4. Component Patterns — Compound, Controlled/Uncontrolled, Render Props

### Controlled vs Uncontrolled
```jsx
// Uncontrolled — component owns state (Team A: "just make it work")
function Tabs({ defaultTab = 0 }) {
  const [activeTab, setActiveTab] = useState(defaultTab); // internal
  ...
}

// Controlled — parent owns state (Team B: "I need to drive it from URL")
function Tabs({ activeTab, onTabChange }) {
  // no internal state — fully driven by parent
  ...
}

// ✅ Both in one — industry standard pattern (same as React's <input>)
function Tabs({ activeTab: controlled, onTabChange, defaultTab = 0 }) {
  const isControlled = controlled !== undefined;
  const [internal, setInternal] = useState(defaultTab);
  const activeTab = isControlled ? controlled : internal;
  const setActiveTab = isControlled ? onTabChange : (i) => { setInternal(i); onTabChange?.(i); };
  ...
}
// Pass activeTab prop → controlled. Omit it → uncontrolled.
```

### Compound Components (Team C: "I want full control over appearance")
Sub-components share state via Context, compose freely in JSX.

```jsx
const TabsContext = createContext();

function Tabs({ children, defaultTab = 0 }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ children, index }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  return (
    <button aria-selected={activeTab === index} onClick={() => setActiveTab(index)}>
      {children}
    </button>
  );
}

function TabPanel({ children, index }) {
  const { activeTab } = useContext(TabsContext);
  return activeTab === index ? <div role="tabpanel">{children}</div> : null;
}

Tabs.Tab = Tab;
Tabs.Panel = TabPanel;

// Team C can render completely custom tab headers using the same context
function CustomHeader({ index, icon, label }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  return <div className={activeTab === index ? 'active' : ''} onClick={() => setActiveTab(index)}>{icon}{label}</div>;
}
```

### useImperativeHandle — Programmatic Control from Parent
```jsx
const Tabs = forwardRef(function Tabs({ children, defaultTab = 0 }, ref) {
  const [activeTab, setActiveTab] = useState(defaultTab);

  useImperativeHandle(ref, () => ({
    goToTab: (i) => setActiveTab(i),
    getActiveTab: () => activeTab,
    reset: () => setActiveTab(defaultTab),
  }));

  return <TabsContext.Provider value={{ activeTab, setActiveTab }}>{children}</TabsContext.Provider>;
});

// Parent
const tabsRef = useRef();
tabsRef.current.goToTab(2); // jump to tab 2 programmatically
```

### Render Props (maximum flexibility, less readable)
```jsx
<Tabs defaultTab={0}>
  {({ activeTab, setActiveTab }) => (
    <div>
      {tabs.map((tab, i) => (
        <button key={i} onClick={() => setActiveTab(i)}>{tab.label}</button>
      ))}
      <div>{tabs[activeTab].content}</div>
    </div>
  )}
</Tabs>
```

### When to Use Which
```
Uncontrolled         → consumer doesn't need external control
Controlled           → URL/router driven, parent state coordination
Compound components  → design system, customizable appearance, reusable library
useImperativeHandle  → parent needs to trigger child actions imperatively
Render props         → maximum flexibility, full consumer rendering control
```

### Interview Answer
> "I'd build a compound component with both controlled and uncontrolled modes. Sub-components communicate via Context — consumers compose freely, giving full appearance control. For controlled mode I check if the activeTab prop is passed and defer to it, otherwise use internal state — same pattern as React's input element. For programmatic external control I expose goToTab via useImperativeHandle and forwardRef. Render props are an alternative but compound components are cleaner in JSX."

---

---

## 5. Custom Hooks — useFetch with Cleanup

### When to extract to a custom hook
```
✅ Same stateful logic in multiple components → useFetch, useDebounce, useWindowSize
✅ useEffect with setup/cleanup that belongs together → useWebSocket, useEventListener
✅ Complex logic making component hard to read → useFormValidation, useInfiniteScroll
❌ Trivial single useState — adds indirection for no benefit
```

### Production useFetch
```javascript
export function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [refreshKey, setRefreshKey] = useState(0);

  const refetch = useCallback(() => setRefreshKey(k => k + 1), []);

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController(); // cancel fetch on unmount

    async function fetchData() {
      setLoading(true);
      setError(null);
      try {
        const resp = await fetch(url, { signal: controller.signal });
        if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
        setData(await resp.json());
      } catch (err) {
        if (err.name === 'AbortError') return; // expected on unmount — not an error
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    fetchData();
    return () => controller.abort(); // cleanup: runs on unmount OR url change

  }, [url, refreshKey]);

  return { data, loading, error, refetch };
}
```

### Why AbortController is critical
Without cleanup: fetch completes after unmount → `setData` runs on dead component → memory leak + React warning "Can't perform state update on unmounted component".

With `controller.abort()` in cleanup: fetch cancelled → AbortError thrown → caught and silently ignored → no state updates on dead component.

### Common mistakes in useFetch
```javascript
// ❌ Not awaiting async call — resp is a Promise, not data
const resp = axios.get(url).catch(...)

// ❌ axios uses resp.data not resp.json (resp.json() is the fetch API)

// ❌ url missing from deps — won't re-fetch on URL change
useEffect(() => { fetchData() }, []) // should be [url]

// ❌ No cleanup — memory leak on unmount
```

### Interview Answer
> "Extract to a custom hook when the same stateful logic appears across multiple components, when useEffect has setup/cleanup that belongs together, or when logic is complex enough to hurt readability. The critical thing in useFetch is AbortController cleanup — without it, state updates run on unmounted components causing leaks and React warnings. The cleanup function calls controller.abort(), which triggers AbortError that we catch and silently ignore."

---

---

## 6. Intermediate-Level Q&A (4–7 Years Experience)

> These questions come up as warmups even in principal-level interviews. Easy to blank on under pressure.

---

### Q1 — useEffect Dependency Array Rules

**Q: What are the rules for the useEffect dependency array? What happens if you get it wrong?**

```javascript
// Rule: every reactive value used inside useEffect must be in the dependency array
// Reactive = state, props, context, anything derived from them

// ❌ Missing dependency — stale closure
function Component({ userId }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setData); // userId used but not in deps
  }, []); // only runs on mount — ignores userId changes
}

// ✅ Correct — re-runs when userId changes
useEffect(() => {
  fetchUser(userId).then(setData);
}, [userId]);

// Cleanup runs BEFORE next effect, and on unmount
useEffect(() => {
  const sub = subscribe(userId);
  return () => sub.unsubscribe(); // runs when userId changes before re-subscribing
}, [userId]);

// Stable references don't need to be in deps
const fetchUser = useCallback((id) => api.get(id), []); // stable
useEffect(() => { fetchUser(userId); }, [userId, fetchUser]); // fetchUser won't retrigger

// Common gotcha — object/array in deps
useEffect(() => { ... }, [{ id: userId }]); // ❌ new object every render = infinite loop
useEffect(() => { ... }, [userId]);         // ✅ primitive value — stable comparison
```

**The 3 things useEffect cleanup does:**
```
1. Runs when component unmounts
2. Runs when deps change (before next effect fires)
3. Does NOT run after every render — only when deps change
```

**Interview answer:**
> "Every value used inside useEffect that comes from render scope must be in the dependency array — state, props, or values derived from them. Missing a dep causes a stale closure where the effect reads an old value forever. Objects and arrays in deps cause infinite loops because they produce a new reference every render — use primitives. Cleanup runs on unmount AND before the next effect when deps change."

---

### Q2 — Key Prop: When It Remounts vs Updates

**Q: What does the key prop actually do? When does it cause a full remount?**

```jsx
// React uses key to identify elements between renders
// Same key = same element → UPDATE (preserve state)
// Different key = different element → DESTROY + REMOUNT (reset state)

// ❌ Index as key — problem when list changes
const list = ['a', 'b', 'c'];
list.map((item, i) => <Input key={i} defaultValue={item} />);
// Remove 'a' → ['b', 'c']
// key 0 still exists → React thinks it's the SAME Input → keeps 'a' value → bug

// ✅ Stable ID as key
list.map(item => <Input key={item.id} defaultValue={item.value} />);

// POWER MOVE — intentional remount to reset state
// Problem: same component, different user → old state bleeds into new user
<UserProfile userId={userId} />
// If UserProfile uses internal state, switching userId doesn't reset it

// ✅ Force remount by changing key — resets all state and effects
<UserProfile key={userId} userId={userId} />
// React destroys old instance and creates fresh one when userId changes
// This is cleaner than useEffect + manual reset in most cases

// Animations — key change triggers exit + enter animation
<AnimatedModal key={modalId} />
```

**Interview answer:**
> "Key is React's identity for elements across renders. Same key = update existing element, keep state. Different key = destroy and remount fresh. Index as key breaks when items are added/removed because indices shift — React gets confused about which element is which. The power use: force a full remount by changing key — when you need a component to fully reset when a prop changes, adding that prop as key is cleaner than manually resetting state in useEffect."

---

### Q3 — React 18 Automatic Batching

**Q: How does React 18 batch state updates differently from React 17?**

```javascript
// React 17 — batching only inside React event handlers
function handleClick() {
  setA(1); // } batched — one render
  setB(2); // }
}

// React 17 — NOT batched outside React handlers
setTimeout(() => {
  setA(1); // render 1
  setB(2); // render 2 — two renders!
}, 1000);

// React 18 — automatic batching EVERYWHERE
setTimeout(() => {
  setA(1); // } batched — one render
  setB(2); // }
}, 1000);

fetch('/api').then(() => {
  setA(1); // } batched in React 18
  setB(2); // }
});

// Opt out of batching when you need intermediate renders
import { flushSync } from 'react-dom';

flushSync(() => setA(1)); // forces render immediately
flushSync(() => setB(2)); // forces another render — use sparingly
```

**Why batching matters:**
```
Without: 2 state updates = 2 renders = 2 DOM operations
With:    2 state updates = 1 render = 1 DOM operation
React 18 brings this consistency to async code for free
```

**Interview answer:**
> "React 17 batched state updates only inside React event handlers — setTimeout and Promises triggered separate renders per setState call. React 18 adds automatic batching everywhere: inside setTimeout, fetch callbacks, native event listeners — all updates in the same call stack collapse into one render. You can opt out with `flushSync` when you need an immediate DOM update, but that's rare."

---

### Q4 — useRef vs createRef

**Q: When do you use useRef vs createRef? What are the common use cases?**

```javascript
// createRef — creates a new ref object on every render (designed for class components)
function BadComponent() {
  const inputRef = createRef(); // new ref every render — previous DOM ref lost
  return <input ref={inputRef} />;
}

// useRef — same ref object across renders (correct for function components)
function GoodComponent() {
  const inputRef = useRef(null); // stable ref — persists for component lifetime
  
  const focusInput = () => inputRef.current?.focus();
  
  return <input ref={inputRef} />;
}

// Use cases:
// 1. DOM access
const canvasRef = useRef(null);
useEffect(() => {
  const ctx = canvasRef.current.getContext('2d');
  // draw...
}, []);
return <canvas ref={canvasRef} />;

// 2. Mutable value that doesn't trigger re-render
const renderCount = useRef(0);
useEffect(() => { renderCount.current++; }); // never causes re-render

// 3. Previous value tracking
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }); // update AFTER render
  return ref.current; // returns value from BEFORE this render
}

// 4. Avoiding stale closure (from Section 2)
const latestCallback = useRef(callback);
useEffect(() => { latestCallback.current = callback; });
// interval can call latestCallback.current — always fresh, stable ref
```

**Interview answer:**
> "`createRef` creates a new ref each render — use it in class components only. `useRef` returns the same object across renders — the correct choice in function components. The two use cases: accessing DOM nodes directly, and storing mutable values that shouldn't trigger re-renders (render counters, previous values, interval IDs). Changing `ref.current` never causes a re-render — that's the distinction from state."

---

### Q5 — Controlled Forms: Correct Pattern

**Q: How do you handle a form with multiple fields in React?**

```jsx
// ❌ One useState per field — verbose and doesn't scale
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [password, setPassword] = useState('');

// ✅ Single state object — scales to any number of fields
function LoginForm({ onSubmit }) {
  const [form, setForm] = useState({ name: '', email: '', password: '' });
  const [errors, setErrors] = useState({});
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value })); // [name] = computed property
    if (errors[name]) setErrors(prev => ({ ...prev, [name]: '' })); // clear error on type
  };

  const validate = () => {
    const newErrors = {};
    if (!form.email.includes('@')) newErrors.email = 'Invalid email';
    if (form.password.length < 8) newErrors.password = 'Min 8 chars';
    return newErrors;
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    const validationErrors = validate();
    if (Object.keys(validationErrors).length) {
      setErrors(validationErrors);
      return;
    }
    setLoading(true);
    try { await onSubmit(form); }
    finally { setLoading(false); }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={form.email} onChange={handleChange} />
      {errors.email && <span>{errors.email}</span>}
      <input name="password" type="password" value={form.password} onChange={handleChange} />
      {errors.password && <span>{errors.password}</span>}
      <button type="submit" disabled={loading}>
        {loading ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

**Interview answer:**
> "I use a single state object for all fields with a generic `handleChange` that uses `e.target.name` as the key — computed property `[name]`. This scales to any number of fields with one handler. `e.preventDefault()` stops the browser's native form submission. For validation I run it on submit and attach errors to a separate errors object rendered next to each field. For real forms I'd reach for React Hook Form to avoid re-rendering on every keystroke."

---

### Q6 — Children Prop & Component Composition

**Q: How does the children prop work? When do you use it over passing components as regular props?**

```jsx
// children — whatever you put between open and close tags
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="content">{children}</div>
    </div>
  );
}

<Card title="Profile">
  <Avatar src={user.avatar} />   {/* children can be anything */}
  <p>{user.bio}</p>
</Card>

// Multiple named slots — component as prop pattern
function Layout({ header, sidebar, children }) {
  return (
    <div className="layout">
      <header>{header}</header>
      <aside>{sidebar}</aside>
      <main>{children}</main>
    </div>
  );
}

<Layout
  header={<NavBar user={user} />}
  sidebar={<FilterPanel filters={filters} />}
>
  <ProductGrid items={items} />
</Layout>

// Inspecting children
React.Children.count(children)    // count
React.Children.map(children, child => React.cloneElement(child, { extraProp: true }))
// cloneElement lets you inject props into children — used in compound components

// children as function (render prop pattern)
function Toggle({ children }) {
  const [on, setOn] = useState(false);
  return children({ on, toggle: () => setOn(o => !o) });
}

<Toggle>
  {({ on, toggle }) => <button onClick={toggle}>{on ? 'ON' : 'OFF'}</button>}
</Toggle>
```

**Interview answer:**
> "`children` is what renders between the component's tags — it's just a prop that can be JSX, strings, arrays, or functions. I prefer it over a `content` prop for complex markup because it reads like HTML and doesn't need to be serialized. For multiple named slots I pass components as regular props — `header`, `sidebar`, `footer`. Children as a function (render props) lets the parent share state with children without lifting it — useful when you control the state logic but need the consumer to control the rendering."

---

### Q7 — useState: Functional Updates & Initialization

**Q: When must you use the functional form of setState? What is lazy initialization?**

```javascript
// ❌ Stale state in closures — common bug with fast updates
const [count, setCount] = useState(0);
const handleClick = () => {
  setCount(count + 1); // closes over count at render time
  setCount(count + 1); // both use the same stale count → only +1 total
};

// ✅ Functional update — receives guaranteed latest state
const handleClick = () => {
  setCount(prev => prev + 1); // prev is always current
  setCount(prev => prev + 1); // count goes up by 2
};

// Required: whenever new state depends on old state
// And always inside setTimeout, setInterval, event handlers that close over state

// Lazy initialization — runs once on mount, avoids expensive ops on every render
// ❌ JSON.parse runs on every render
const [data, setData] = useState(JSON.parse(localStorage.getItem('data') || '{}'));

// ✅ Function form — only runs on first render
const [data, setData] = useState(() => JSON.parse(localStorage.getItem('data') || '{}'));

// Object state — always spread to avoid losing other fields
const [user, setUser] = useState({ name: '', email: '', role: 'viewer' });

setUser({ name: 'Abhishek' }); // ❌ loses email and role!
setUser(prev => ({ ...prev, name: 'Abhishek' })); // ✅ keeps other fields
```

**Interview answer:**
> "Use the functional form `setState(prev => ...)` whenever the new state depends on the old value — especially in event handlers, timeouts, and intervals where the closure captures stale state. Multiple updates in one handler need functional form to queue correctly. Lazy initialization: passing a function to `useState(fn)` instead of a value means the initializer runs only once on mount — important for expensive operations like parsing localStorage or computing from large datasets."

---

## 7. Class Lifecycle → Hooks Mapping

```
constructor           → useState (initial state)
componentDidMount     → useEffect(() => {}, [])
componentDidUpdate    → useEffect(() => {}, [deps])
componentWillUnmount  → useEffect(() => { return () => cleanup }, [])
shouldComponentUpdate → React.memo + useMemo + useCallback
getDerivedStateFromError → no hook — must use class for Error Boundary
componentDidCatch     → no hook — must use class for Error Boundary
```

---

## 8. All React Hooks Reference

```
BASIC
useState          → local state
useEffect         → side effects, lifecycle
useContext        → consume context

PERFORMANCE
useMemo           → memoize computed value
useCallback       → memoize function reference
memo              → skip re-render if props unchanged

REFS
useRef            → DOM reference or mutable value (no re-render)
useImperativeHandle → expose ref methods to parent (with forwardRef)

ADVANCED
useReducer        → complex state (multiple related fields)
useLayoutEffect   → like useEffect but fires before browser paint
useId             → generate unique IDs (React 18)

REACT 18
useTransition     → mark update as non-urgent
useDeferredValue  → defer a value update
useSyncExternalStore → subscribe to external stores safely
```

---

## 9. React 18 — Concurrent Features

### useTransition — mark update as non-urgent

```jsx
const [isPending, startTransition] = useTransition();

const handleSearch = (e) => {
  setInputValue(e.target.value);        // urgent — update input immediately

  startTransition(() => {
    setSearchResults(filterData(value)); // non-urgent — can be interrupted
  });
};

return (
  <>
    <input value={inputValue} onChange={handleSearch} />
    {isPending && <Spinner />}
    <Results data={searchResults} />
  </>
);
```

### useDeferredValue — defer a derived value

```jsx
const [search, setSearch] = useState('');
const deferredSearch = useDeferredValue(search);

// search updates immediately (input responsive)
// deferredSearch updates when React has time
const results = useMemo(() => filterData(deferredSearch), [deferredSearch]);
```

### Difference

```
useTransition    → you control what's deferred (wrap the setter)
useDeferredValue → defer a value you don't control
```

### Automatic batching

```javascript
// React 17 — two re-renders inside setTimeout
setTimeout(() => {
  setCount(1);   // re-render
  setName('A');  // re-render
});

// React 18 — one re-render (batched automatically everywhere)
setTimeout(() => {
  setCount(1);
  setName('A');  // → one re-render
});
```

**Interview one-liner:**
> "React 18 makes rendering interruptible. useTransition wraps non-urgent updates so user input is never blocked. useDeferredValue defers a value when you don't control the setter. Automatic batching reduces re-renders outside event handlers too."

---

## 10. Error Boundaries

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    logToSentry(error, info.componentStack);  // log to observability
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          Something went wrong.
          <button onClick={() => this.setState({ hasError: false })}>
            Retry
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// usage — wrap at feature level, not top level
<ErrorBoundary>
  <AgentWidget />
</ErrorBoundary>
```

**Three things to know:**
```
1. Must be class component — no hook equivalent
   react-error-boundary library gives hook-friendly wrapper

2. Does NOT catch:
   → async errors (setTimeout, fetch)
   → event handlers
   → only catches render/lifecycle errors

3. Wrap at feature level — one broken feature shouldn't crash entire app
```

**Your project answer:**
> "We wrapped each agent widget in an ErrorBoundary — if one agent's UI crashed it showed a retry button without affecting other agents. Errors logged to our observability stack via componentDidCatch."

---

## 11. Performance — Initial Load

```
Diagnose:
  Chrome Network tab    → bundle size
  Chrome Coverage tab   → unused JS
  React DevTools Profiler → slow components

Fixes in order of impact:

1. Code splitting + lazy loading
   const Page = lazy(() => import('./Page'))
   → each route loads its own chunk
   → home page doesn't load dashboard code

2. Suspense — show skeleton not blank screen
   <Suspense fallback={<Skeleton />}>
     <LazyComponent />
   </Suspense>

3. Bundle analysis
   webpack-bundle-analyzer → find large deps → replace or tree-shake

4. useTransition — keep UI responsive during heavy renders

5. useDeferredValue — defer slow search/filter

6. SSR (Next.js) — real fix for blank screen
   server sends HTML → user sees content before JS loads
```

---

## 12. Custom Hooks — Interview Ready

### useDebounce

```javascript
const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);  // cleanup cancels previous timer
  }, [value, delay]);

  return debouncedValue;
};
```

### useFetch

```javascript
const useFetch = (url) => {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const controller = new AbortController();

    const fetchData = async () => {
      setLoading(true);
      try {
        const resp = await axios.get(url, { signal: controller.signal });
        setData(resp.data);
      } catch (err) {
        if (err.name === 'CanceledError') return;
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
    return () => controller.abort();
  }, [url]);

  return { data, error, loading };
};
```

### useLocalStorage

```javascript
const useLocalStorage = (key, initialValue) => {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  const setStoredValue = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, setStoredValue];
};
```

### useIntersectionObserver

```javascript
const useIntersectionObserver = (ref) => {
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      setIsVisible(entry.isIntersecting);
    });
    if (ref.current) observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref]);

  return isVisible;
};
```

---

## 13. Large List Performance — Virtualization

```jsx
import { FixedSizeList } from 'react-window';

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  );

  return (
    <FixedSizeList height={600} itemCount={items.length} itemSize={50} width="100%">
      {Row}
    </FixedSizeList>
  );
}
// renders only ~12 visible rows regardless of list size
```

```
react-window   → fixed/variable size rows, lightweight
react-virtuoso → dynamic heights, easiest API
```

---

## 14. State Architecture — Multi-team (Principal Level)

```
Shell owns global state:
  Auth / User     → Context, MFEs read only
  Notifications   → shell owns UI, MFEs fire events
  Theme           → CSS variables, no JS state needed
  Feature flags   → shell fetches, exposes via Context

MFEs own local state:
  Never import shell's store directly (tight coupling)
  Communicate via custom events or window.__shell__ API

window.__shell__ = {
  getUser: () => store.getState().user,
  notify: (msg) => store.dispatch(showNotification(msg))
}
```

**Interview answer:**
> "Shell owns global state — auth, notifications, theme. MFEs consume via exposed APIs, never import shell's store. Theme is CSS variables so MFEs don't need JS state at all. Each MFE owns its own local state and is independently deployable."
