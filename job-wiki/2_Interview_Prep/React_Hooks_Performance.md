# React Hooks & Frontend Performance Optimisation

This document covers frontend engineering concepts in React, focusing on render cycles, hook memoization, reconciliation, and concurrent features. It relates to frontend performance issues you resolved when building the interactive node-graph editor in the No-Code Agent Builder Platform at Concentrix.

[← Back to Index](../README.md) | [← Previous: CI/CD Pipeline Flow](./CI_CD_Pipeline.md) | [Next: Node.js Event Loop & Scaling](./Node_Event_Loop.md)

---

## ⚡ React Reconciliation & Virtual DOM

To update the browser DOM efficiently, React uses a Virtual DOM.

1. **Virtual DOM:** React keeps a lightweight virtual representation of the user interface in memory.
2. **Reconciliation:** When a component's state or props change, React builds a new virtual representation tree. It compares (diffs) this new tree with the previous one.
3. **The Diffing Algorithm:**
   - Elements of different types (e.g., changing a `<div>` to a `<span>`) will trigger a full rebuild of that subtree.
   - For lists, React relies on the **`key` prop**. The key must be a unique, stable identifier (never use `Math.random()` or list indices). Stable keys allow React to identify which items moved, were added, or were removed, preventing unnecessary DOM re-renders.

---

## ⚙️ Hook Memoization: `useMemo` vs. `useCallback`

Every time a React component's state updates, the entire function re-executes. This means all local variables are re-allocated, and functions are re-created.

### 1. `useMemo`
* **Purpose:** Caches the *result* of a calculation between renders.
* **When to use:** For CPU-expensive operations (e.g., filtering a large list, parsing markdown data arrays).
* **Code Example (Filtering search results):**
  ```jsx
  const filteredSkills = useMemo(() => {
    return skills.filter(skill => skill.name.toLowerCase().includes(searchQuery.toLowerCase()));
  }, [skills, searchQuery]); // Only runs when skills or searchQuery changes
  ```

### 2. `useCallback`
* **Purpose:** Caches the *function definition itself* between renders.
* **When to use:** When passing a callback function as a prop to a child component that is optimized using `React.memo`. If you don't wrap the callback in `useCallback`, the child will re-render anyway because functions are objects, and a new reference is created on every parent render (failing shallow reference equality checks).
* **Code Example:**
  ```jsx
  const handleRatingChange = useCallback((skillId, rating) => {
    updateSkillRating(skillId, rating);
  }, []); // Caches reference; doesn't trigger child re-renders on parent state updates
  ```

---

## 🚀 React 18 Concurrent Features

React 18 introduced a concurrent renderer, allowing React to interrupt, pause, and resume rendering operations.

### 1. `useTransition`
* **What:** Splits updates into **urgent** updates (typing in an input, clicking a toggle) and **non-urgent** transition updates (rendering a heavy list, fetching data).
* **GKE Platform UI Example:** In your No-Code Agent Builder, when a user typed a search query in a node-selection drawer, the character typed had to display immediately (urgent). The filter of 100+ complex nodes was wrapped in a transition:
  ```jsx
  const [isPending, startTransition] = useTransition();
  const [searchQuery, setSearchQuery] = useState("");
  const [filterResult, setFilterResult] = useState([]);

  const handleChange = (e) => {
    setSearchQuery(e.target.value); // Urgent
    startTransition(() => {
      setFilterResult(heavyNodeSearch(e.target.value)); // Non-urgent: can be interrupted
    });
  };
  ```

### 2. `useDeferredValue`
* **What:** Returns a deferred version of a value that lag-updates behind the main state. It is similar to debouncing, but React handles it automatically, triggering the render of the deferred component as soon as the main thread is idle.

---

## 🛠️ Case Study: Node Graph Performance at Concentrix

### The Problem:
In the No-Code Agent Builder, dragging a node on the canvas triggered a state update. Because all nodes were stored in a single list state, updating one node's coordinates triggered a full re-render of all 50+ nodes, causing visible UI lag (frame rate dropped to 15 FPS).

### The Fixes:
1. **Windowing / Virtualization:** Implemented a container wrapper that only rendered nodes currently inside the user's viewport.
2. **React.memo Optimization:** Wrapped node cards in `React.memo` and passed coordinates using simple custom store subscription references.
3. **useCallback:** Wrapped drag handler callbacks in `useCallback` to prevent child nodes from invalidating their memo state.
* **Result:** Drag latency dropped from **80ms** to **under 5ms**, achieving a stable 60 FPS viewport.
