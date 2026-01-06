# The Evolution of React Reconciler: A Six-Year Journey (2016-2022)

*How the React team rebuilt the engine of the web's most popular UI library—and why it took six years of battling edge cases only visible at Facebook scale.*

---

## Prologue: The Stack Reconciler Era (2013-2016)

Before Fiber, there was Stack.

React's original reconciliation algorithm was elegantly simple: when state changed, React would recursively walk the component tree, diffing the old virtual DOM against the new one, and applying changes to the real DOM. It used JavaScript's native call stack for this recursion—hence the name "Stack Reconciler."

**The problem?** Once you started, you couldn't stop.

```
// Conceptually, the old reconciler worked like this:
function reconcile(element) {
  // Can't pause here—call stack owns us
  reconcileChildren(element.children); // recursive
  // Can't pause here either
}
```

This created three fundamental limitations:

1. **No interruption**: A large tree update could block the main thread for hundreds of milliseconds, causing dropped frames and janky UIs
2. **No prioritization**: A user typing in an input had to wait for a background data fetch to finish rendering
3. **No incremental work**: Either you rendered everything or nothing

At Facebook scale—with complex, deeply-nested UIs—these limitations became untenable.

---

## Chapter 1: The Birth of Fiber (2016-2017)

### The Decision Point

In 2016, the React team faced a critical architectural decision: **How do you achieve incremental, interruptible rendering in JavaScript?**

Four options emerged:

| Option | Status | Why |
|--------|--------|-----|
| **Web Workers/Threads** | Rejected | JS lacks good thread support; can't safely abort mid-work; can't share memory efficiently |
| **Generator Functions** | Rejected | Stateful—can't resume in middle; syntactic overhead wrapping every function; poor debugging |
| **OCaml Algebraic Effects** | Rejected | Non-trivial JS compilation overhead; would need to clone each fiber to rewind |
| **Custom Fiber Structure** | **Chosen** | Fixed data structure with linked-list traversal; no JS stack recursion; full control |

The insight that unlocked Fiber came from an unexpected source: **algebraic effects**. As Dan Abramov later explained in his essay "Algebraic Effects for the Rest of Us," Fiber's design mirrors the ability to "pause" computation, do something else, and "resume"—without actually using effect handlers.

### The Implementation

Between May and September 2017, a flurry of foundational PRs landed:

- **PR #6690** (May 2016): Initial Fiber infrastructure
- **PR #7034** (June 2016): Host container fibers and priority levels
- **PR #6859** (May 2016): Child reconciler with new coroutines primitive
- **PR #10758** (Sept 2017): Created the `react-reconciler` npm package

The Fiber data structure was revolutionary:

```javascript
// Each Fiber is a unit of work with pointers to:
fiber.child      // first child
fiber.sibling    // next sibling
fiber.return     // parent
fiber.alternate  // previous version (for diffing)
```

Instead of using the JS call stack, Fiber uses a **singly-linked list traversal**. This means React can:
- Stop work at any fiber
- Come back later and resume
- Throw away work-in-progress if priorities change

**September 26, 2017**: React 16.0.0 ships. Fiber is in production.

But this was just the beginning.

---

## Chapter 2: The Async Rendering Preparation (2018)

With Fiber shipped, the team turned to the real goal: **async rendering**. But first, the ecosystem needed preparation.

### The Lifecycle Deprecation

Three lifecycle methods were fundamentally incompatible with async rendering:
- `componentWillMount`
- `componentWillReceiveProps`
- `componentWillUpdate`

Why? In async rendering, the "render phase" (where these run) could be interrupted and restarted. Side effects in these methods could run multiple times.

**PR #12083** introduced `React.StrictMode`, which double-invokes these methods in development to expose unsafe patterns.

### New Context API

The old context API was also broken for async rendering. **PR #11818** (Jan 2018) introduced `React.createContext`—a complete rewrite with proper subscription semantics.

**March 29, 2018**: React 16.3.0 ships with async rendering preparation complete.

---

## Chapter 3: Suspense Arrives (October 2018)

Suspense was React's answer to async data fetching. Instead of imperative loading states:

```javascript
// Before: Manual loading state
if (loading) return <Spinner />;
return <Data />;
```

Suspense made it declarative:

```javascript
// After: Suspense boundary catches promises
<Suspense fallback={<Spinner />}>
  <Data />
</Suspense>
```

Key PRs:
- **PR #12279** (Feb-May 2018): Core Suspense implementation
- **PR #13398** (Aug 2018): `React.lazy` for code splitting
- **PR #13683** (Sept 2018): Scheduler package extracted

**October 23, 2018**: React 16.6.0 ships Suspense for code splitting.

---

## Chapter 4: Hooks Revolution (February 2019)

Hooks didn't just change how we write components—they deeply integrated with Fiber.

```javascript
// Hook state is stored on the Fiber itself
fiber.memoizedState = {
  // Linked list of hooks
  next: { /* useState */ },
  next: { /* useEffect */ },
  // ...
};
```

Key PRs:
- **PR #13968** (Oct 2018): Hooks implementation
- **PR #14679** (Jan 2019): Enable hooks (removed feature flag)

The team also introduced strict rules—hooks must be called in the same order every render—enforced by an ESLint plugin (**PR #15025**).

**February 6, 2019**: React 16.8.0 ships Hooks.

---

## Chapter 5: The Dark Forest of Bugs

Here's where the story gets interesting. Every architectural solution revealed new edge cases—many only visible at Facebook scale.

### The Tearing Problem (2018-2021)

**Tearing** occurs when different parts of the UI show inconsistent data because an external store updated mid-render.

```
┌─────────────────────────────────────┐
│  Component A reads store: value = 1 │
│        ↓ (render interrupted)       │
│     Store updates: value = 2        │
│        ↓ (render resumes)           │
│  Component B reads store: value = 2 │
│        ↓                            │
│  UI shows INCONSISTENT state! 💥    │
└─────────────────────────────────────┘
```

The first solution, `useMutableSource` (**PR #18000**, Feb 2020), required store authors to implement a versioning contract. Most didn't.

Even the fix had bugs—**PR #18912** (May 2020) fixed tearing in `useMutableSource` itself.

Finally, **PR #22239** (Sept 2021) introduced `useSyncExternalStore` with a simpler insight: **force synchronous re-render if the store value changed during render**. Simple, but it worked.

### Infinite Loop Bugs

Concurrent rendering created new ways to loop forever:

| Bug | Description |
|-----|-------------|
| #17949 | Expired partial tree causes infinite loops |
| #22277 | Sync update in passive effect triggers itself |
| #24247 | Unmemoized `useDeferredValue` creates new object every render |
| #26625 | Render phase updates cause infinite loops |

**PR #28279** (Feb 2024) finally added a 50-update limit guard—but this was years after the original architecture shipped.

### Dropped Updates

The Fiber architecture stores updates on linked lists. Interrupting and resuming must carefully transfer these updates:

| Bug | Description |
|-----|-------------|
| #18091 | `React.memo` drops lower priority updates on bailout |
| #18384 | Updates dropped inside suspended tree |
| #25578 | `useSyncExternalStore` dropped update in render phase |

### Priority Starvation

The original priority system used "expiration times"—a single number representing when work must complete. This had a fundamental flaw:

> A high-priority IO task could starve low-priority CPU tasks indefinitely, but you couldn't express "wait for IO before CPU work."

**PR #18796** (May 2020) introduced **Lanes**—a bitmask model allowing multiple independent priority levels:

```javascript
// Lanes are bits that can be combined
const SyncLane        = 0b0001;
const InputLane       = 0b0010;
const DefaultLane     = 0b0100;
const TransitionLane  = 0b1000;

// Can track multiple in-flight renders
currentLanes = SyncLane | TransitionLane; // 0b1001
```

---

## Chapter 6: React 17 - The Stepping Stone (October 2020)

React 17 shipped with **no new features**. Instead, it laid groundwork:

- Event delegation moved from `document` to the root container
- Effects list replaced with `subtreeFlags` bitmask (**PR #19673**)
- Full Lanes implementation replacing expiration times

**October 20, 2020**: React 17.0.0 ships.

---

## Chapter 7: Concurrent React Ships (March 2022)

After six years of work, concurrent rendering was finally stable enough for general availability:

- `createRoot`/`hydrateRoot` APIs enable concurrent mode
- `useTransition`/`startTransition` separate urgent from non-urgent updates
- Automatic batching enabled by default
- `useSyncExternalStore` solves tearing

**March 29, 2022**: React 18.0.0 ships. Concurrent React is real.

---

## Chapter 8: The Modern Era (2023-2025)

### Server Components (RSC)

Server Components extend the component model to the server:

- **PR #17285** (Nov 2019): Flight protocol for streaming RSC
- **PR #25207** (Sept 2022): `use(promise)` for Server Components
- **PR #25479** (Oct 2022): Async functions in Server Components

### React 19 and Actions

- **PR #28491** (March 2024): `useActionState` hook
- **PR #28097** (Jan 2024): Async actions in `startTransition`
- **PR #28348** (Feb 2024): `ref` as normal prop (no more `forwardRef`)

**December 5, 2024**: React 19.0.0 ships with Actions, stable RSC, and Compiler preview.

### View Transitions (2024-2025)

- **PR #31975** (Dec 2024): `ViewTransition` component
- **PR #32785** (Jan 2025): `startGestureTransition` API

---

## Epilogue: Why Six Years?

The decision graph documenting this journey contains **218 nodes** and **330 edges**—goals, decisions, actions, observations, and outcomes all interconnected.

Why did concurrent rendering take six years (2016-2022)?

> **Each architectural solution revealed new edge cases only visible at scale—tearing, starvation, dropped updates, infinite loops—requiring multiple iterations.**

The team had to:
1. Build the Fiber architecture (2016-2017)
2. Prepare the ecosystem (2018)
3. Ship incremental features: Suspense, Hooks (2018-2019)
4. Fix fundamental bugs discovered only at Facebook scale (2018-2021)
5. Replace expiration times with Lanes (2020)
6. Solve tearing definitively (2021)
7. Make it all stable (2022)

Some features were even reverted after shipping:
- **#27027**: Infinite loop detection (false positives)
- **#31080**: Non-blocking prerendering (rendering issues)
- **#17941**: Expiring partial trees (regression)

---

## The Core Insight

As documented in node #216 of the decision graph:

> **React's power comes from treating rendering as a pure function of state. Fiber enabled making this interruptible without breaking the abstraction.**

And from Dan Abramov's influence (node #218):

> **Fiber's design mirrors algebraic effects—the ability to "pause" computation, do something else, and "resume"—without actually using effect handlers.**

This is what made React's concurrent rendering possible: a custom runtime that simulates features JavaScript doesn't have, built painstakingly over six years of production battle-testing.

---

*This document was generated from the [React Codebase Decision Graph](./index.html), which tracks 218 architectural decisions, actions, and observations across React's development history.*
