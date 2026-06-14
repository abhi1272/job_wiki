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

## What's Next

- [ ] React 18 — Suspense, lazy loading, streaming SSR
