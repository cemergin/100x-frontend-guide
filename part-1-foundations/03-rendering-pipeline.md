<!--
  CHAPTER: 3
  TITLE: The Rendering Pipeline — Mobile & Web
  PART: I — Foundations
  PREREQS: Chapters 1-2
  KEY_TOPICS: React reconciliation, Fiber, concurrent features, React Compiler, useTransition, useDeferredValue, Suspense
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 3: The Rendering Pipeline — Mobile & Web

> **Part I — Foundations** | Prerequisites: Chapters 1-2 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- `setState` does not update the screen; it schedules work that Fiber processes in a priority-aware, interruptible render phase before committing DOM mutations
- Reconciliation diffs the old and new element trees in O(n) using two heuristics: different element types produce different trees, and `key` props signal stability across renders
- The Fiber architecture splits rendering into render phase (pure, interruptible) and commit phase (synchronous, side-effectful); this split enables concurrent features
- `useTransition` and `useDeferredValue` mark updates as low-priority so the UI stays responsive during expensive renders; Suspense is the boundary that shows fallback UI during async operations
- The React Compiler auto-memoizes components and hooks, eliminating most manual `useMemo`/`useCallback`; understand what it does so you know when to trust it

</details>

Here's what most React developers get dangerously wrong: they think `setState` updates the screen. It doesn't. `setState` tells React "something changed." What happens next is a complex, interruptible, priority-aware pipeline that decides *what* changed, *when* to update it, and *how* to minimize the DOM mutations needed to reflect the new state.

This pipeline is called **reconciliation**, and the engine that powers it is called **Fiber.** Understanding Fiber is not academic trivia — it's the difference between an app that stays responsive under load and an app that freezes for 300ms when you type in a search box.

In Chapter 1, we traced a component from JSX to native views through React Native's architecture. In Chapter 2, we traced how the browser turns DOM mutations into pixels. This chapter fills the gap between them: **how React decides what DOM mutations to make.** This is the rendering pipeline that's shared between web and mobile — the React layer that runs before anything browser-specific or native-specific happens.

By the end of this chapter, you'll understand why `useMemo` exists (and when it doesn't help), how `useTransition` actually works under the hood, what the React Compiler does to your code, and why Suspense is not just a loading spinner API.

### In This Chapter
- Reconciliation: The Algorithm Behind Virtual DOM
- Fiber Architecture: React's Internal Engine
- The Render Phase vs. The Commit Phase
- Concurrent Features: useTransition, useDeferredValue, Suspense
- React Compiler: Auto-Memoization and Beyond
- React Server Components: The Rendering Boundary Shift
- Performance Mental Models for React

### Related Chapters
- [Ch 1: React Native Architecture & Internals] — how the native rendering pipeline consumes React's output
- [Ch 2: Browser Rendering & Web Fundamentals] — how the browser rendering pipeline consumes React's output
- [Ch 8: Styling & Animation] — how Reanimated bypasses React's rendering pipeline entirely
- [Ch 13: Performance Optimization] — applying these concepts to production performance problems

---

## 1. RECONCILIATION: THE CORE ALGORITHM

React's entire value proposition rests on one idea: **you describe what the UI should look like, and React figures out the minimum set of changes needed to make it so.** This process is reconciliation.

### 1.1 The Problem Reconciliation Solves

Comparing two arbitrary trees is an O(n^3) problem. For a tree with 1,000 nodes, that's 1,000,000,000 comparisons — completely impractical for 60fps rendering. React makes two assumptions that reduce this to O(n):

1. **Elements of different types produce different trees.** If a `<div>` changes to a `<span>`, React won't try to diff the children — it tears down the entire subtree and rebuilds it.

2. **The `key` prop hints at stability across renders.** A list item with `key="user-42"` is the same item across renders, even if its position in the array changed.

These heuristics are wrong sometimes (a `<div>` changing to an `<article>` probably has similar children), but they're right often enough that the O(n) algorithm produces correct results in practice.

### 1.2 How Diffing Works

React walks both the old and new element trees simultaneously:

**Same type, same position:**
```tsx
// Before
<div className="old" title="prev">

// After
<div className="new" title="next">

// React: Update className and title attributes. Keep the DOM node.
```

**Different types:**
```tsx
// Before
<div><Counter /></div>

// After
<span><Counter /></span>

// React: Destroy the <div> and its entire subtree (including Counter's state).
// Create a new <span> and mount Counter from scratch.
```

This is why changing a wrapper element type destroys state. If you switch a `<div>` to a `<section>`, every component inside it unmounts and remounts. This is intentional — React assumes different types produce different trees.

**Lists without keys:**
```tsx
// Before
<ul>
  <li>Alice</li>
  <li>Bob</li>
</ul>

// After (prepend)
<ul>
  <li>Zara</li>
  <li>Alice</li>
  <li>Bob</li>
</ul>

// Without keys: React thinks li[0] changed from Alice to Zara,
// li[1] changed from Bob to Alice, and li[2] is new (Bob).
// It updates ALL THREE elements.
```

**Lists with keys:**
```tsx
// After (with keys)
<ul>
  <li key="zara">Zara</li>
  <li key="alice">Alice</li>
  <li key="bob">Bob</li>
</ul>

// With keys: React knows Alice and Bob are unchanged,
// and Zara is a new insertion. Only ONE element created.
```

**The array index key anti-pattern:**
```tsx
// DON'T use index as key for dynamic lists
{items.map((item, index) => (
  <ListItem key={index} data={item} />
))}

// If items are reordered, index 0 still maps to key 0,
// so React reuses the component with wrong data.
// This causes subtle bugs with forms, animations, and internal state.

// DO use a stable identifier
{items.map(item => (
  <ListItem key={item.id} data={item} />
))}
```

Using array indices as keys is **only safe** when:
1. The list is never reordered
2. Items are never inserted or deleted (only appended)
3. List items have no internal state (no inputs, no expanded/collapsed state)

In practice, this means almost never.

### 1.3 Component Reconciliation

When React encounters a component during reconciliation, it calls the component function (or the class's `render` method) to get the new element tree. Then it recursively reconciles that tree against the previous one.

```tsx
function UserCard({ user }) {
  return (
    <div className="card">
      <Avatar url={user.avatar} />
      <Name value={user.name} />
      <Bio text={user.bio} />
    </div>
  );
}

// If user.name changed but user.avatar and user.bio didn't:
// 1. React calls UserCard({ user: newUser })
// 2. Gets the new element tree
// 3. Diffs against previous tree
// 4. Avatar: same type, same position → check props → url unchanged → skip
// 5. Name: same type, same position → check props → value changed → re-render Name
// 6. Bio: same type, same position → check props → text unchanged → skip
```

But wait — step 4 says "url unchanged → skip." How does React know? By default, React re-renders *every* child component when a parent re-renders, regardless of whether props changed. **React does not do shallow prop comparison by default.**

```tsx
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <ExpensiveChild /> {/* Re-renders on every click, even with no props */}
    </div>
  );
}
```

This is where `React.memo`, `useMemo`, and `useCallback` come in — and this is what the React Compiler automates. We'll cover all of these in detail.

---

## 2. FIBER ARCHITECTURE: REACT'S INTERNAL ENGINE

Fiber is the complete rewrite of React's core algorithm, introduced in React 16 (2017). Understanding Fiber is essential because it explains *how* React implements reconciliation, concurrency, and Suspense.

### 2.1 The Problem with the Stack Reconciler

Before Fiber, React used a **stack reconciler.** It walked the component tree recursively, and because JavaScript's call stack is synchronous and non-interruptible, once React started reconciling a tree, it had to finish — no matter how long it took.

```
Stack Reconciler (React 15):
  
  reconcile(App)
    reconcile(Header)
      reconcile(Logo)
      reconcile(Nav)
        reconcile(NavItem) ← 300ms into this, user types in search box
        reconcile(NavItem)    Browser can't respond until reconciliation finishes
        reconcile(NavItem)
    reconcile(Content)
      reconcile(SearchResults) ← This takes 200ms
        reconcile(ResultItem)
        reconcile(ResultItem)
        ... (500 items)
    reconcile(Footer)
  
  Total: 500ms of uninterruptible work. Input delayed by 500ms.
```

On a complex page, this could easily block the main thread for hundreds of milliseconds. The user would type a character, and nothing would happen until React finished processing the previous update.

### 2.2 Fiber Nodes

Fiber replaces the recursive stack with a **linked list of fiber nodes.** Each fiber node represents a unit of work — typically one component or one host element. Instead of recursing through the tree, React walks the linked list iteratively, processing one fiber at a time.

A fiber node contains:

```typescript
interface Fiber {
  // Identity
  type: ComponentType | string;    // The component function or element type
  key: string | null;              // The key prop
  
  // Tree structure (linked list)
  child: Fiber | null;             // First child
  sibling: Fiber | null;          // Next sibling
  return: Fiber | null;           // Parent
  
  // State
  memoizedState: any;             // The component's state (hooks linked list)
  memoizedProps: any;             // The last rendered props
  pendingProps: any;              // The new props being reconciled
  
  // Effects
  flags: Flags;                   // What operations to perform (placement, update, deletion)
  subtreeFlags: Flags;            // Aggregated flags from children
  
  // Alternate
  alternate: Fiber | null;        // The "other" version (current ↔ work-in-progress)
  
  // Priority
  lanes: Lanes;                   // The priority lanes this fiber is part of
}
```

**The tree walk pattern:**

```
1. Process current fiber
2. If it has a child → move to child
3. If no child, if it has a sibling → move to sibling
4. If no sibling → move to parent's sibling (return + sibling)
5. Repeat until back at root

     App
    /   \
  Header  Content
  /    \       \
Logo   Nav   Results
      / | \
    NI  NI  NI

Walk order: App → Header → Logo → Nav → NI → NI → NI → Content → Results
```

**Why this matters:** Because it's an iterative loop instead of a recursive call stack, React can **pause at any fiber node, yield to the browser, and resume later.** This is the foundation of concurrent rendering.

### 2.3 Double Buffering: Current and Work-in-Progress

React maintains two fiber trees at all times:

- **Current tree:** The tree that's currently rendered on screen
- **Work-in-progress (WIP) tree:** The tree being built for the next update

Each fiber node has an `alternate` pointer to its counterpart in the other tree. When React finishes building the WIP tree, it swaps them — the WIP tree becomes the current tree (this is the "commit"), and the old current tree becomes the starting point for the next WIP tree.

```
Current Tree (on screen)     Work-in-Progress Tree
       App ←──alternate──→        App
      /   \                      /   \
  Header  Content           Header  Content*
  /    \       \            /    \       \
Logo   Nav   Results      Logo   Nav   Results*
                                        (new data)
                                        
After commit: WIP becomes Current, old Current becomes WIP base
```

**Why double buffering:** This enables concurrent rendering. The WIP tree can be built incrementally over multiple frames. If a higher-priority update comes in, React can discard the WIP tree and start a new one. The current tree on screen is never in an inconsistent state.

### 2.4 The Lanes Model: Priority System

React's priority system uses **lanes** — a bitmask-based priority scheme where each lane represents a different urgency level:

```
SyncLane:              0b0000000000000000000000000000010  // Highest priority
InputContinuousLane:   0b0000000000000000000000000001000  // Mouse move, scroll
DefaultLane:           0b0000000000000000000000000100000  // Normal updates
TransitionLane1:       0b0000000000000000000001000000000  // startTransition
IdleLane:              0b0100000000000000000000000000000  // Lowest priority
```

When you call `setState`, React assigns the update to a lane based on context:
- Inside an event handler → SyncLane (high priority)
- Inside `startTransition` → TransitionLane (low priority)
- Inside `useDeferredValue` → deferred lane

Multiple updates can be in-flight simultaneously on different lanes. React processes higher-priority lanes first and can interrupt work on lower-priority lanes.

```tsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  
  function handleInput(e) {
    // This is SyncLane — updates immediately, user sees keystrokes
    setQuery(e.target.value);
    
    // This is TransitionLane — can be interrupted, doesn't block input
    startTransition(() => {
      setResults(filterItems(e.target.value));
    });
  }
  
  return (
    <>
      <input value={query} onChange={handleInput} />
      <ResultList results={results} />
    </>
  );
}
```

**What actually happens with lanes:**

1. User types "a" → React creates two updates: `setQuery("a")` on SyncLane, `setResults(...)` on TransitionLane
2. React processes SyncLane first → input field shows "a" → committed to DOM in one frame
3. React starts processing TransitionLane → begins reconciling `ResultList` with new data
4. User types "b" before TransitionLane finishes → new SyncLane update interrupts
5. React commits `setQuery("ab")` → input shows "ab"
6. React restarts TransitionLane with "ab" results (previous WIP tree discarded)
7. When TransitionLane finishes → results appear

The user sees instant keystroke feedback even when `filterItems` is expensive. The results update asynchronously without blocking input.

---

## 3. THE RENDER PHASE VS. THE COMMIT PHASE

React's work is divided into two distinct phases with very different characteristics:

### 3.1 The Render Phase (Interruptible)

During the render phase, React:
1. Walks the fiber tree
2. Calls component functions to get new element trees
3. Diffs old and new trees
4. Marks fibers with effect flags (needs placement, update, deletion)
5. Does NOT touch the DOM / native views

**Critical property: the render phase is PURE.** No side effects. No DOM mutations. No network calls. This is what makes it interruptible — if React pauses or restarts the render phase, nothing visible changes.

```tsx
function Component({ data }) {
  // This code runs during the RENDER PHASE
  const processed = data.map(transform);    // Pure computation
  const filtered = processed.filter(pred);   // Pure computation
  
  // DON'T do side effects during render:
  // fetch(url);           // BAD: side effect in render
  // localStorage.set(x);  // BAD: side effect in render
  // ref.current = x;      // BAD: mutation in render
  
  return <List items={filtered} />;
}
```

**Why this matters for concurrent features:** In concurrent mode, React might call your component function, then throw away the result because a higher-priority update came in. If your component has side effects during render, they'll execute but their results won't be visible — a classic source of bugs.

### 3.2 The Commit Phase (Synchronous, Non-Interruptible)

After the render phase completes, React enters the commit phase:
1. Apply all DOM mutations (insertions, updates, deletions)
2. Run `useLayoutEffect` callbacks (synchronously, before paint)
3. Yield to the browser for paint
4. Run `useEffect` callbacks (asynchronously, after paint)

```
Render Phase          Commit Phase           Browser
  (interruptible)       (synchronous)          
                                               
  Walk fiber tree →   Mutation phase:         
  Call components →     Apply DOM changes →   
  Build WIP tree →                            
                      Layout phase:           
                        useLayoutEffect →     
                                              
                                              Paint →
                                              
                      Passive effect phase:   
                        useEffect →           
```

**The commit phase is synchronous because partial DOM updates would leave the screen in an inconsistent state.** If React updated 5 out of 10 elements and then yielded to the browser, the user would see a half-updated UI.

### 3.3 Effect Timing

Understanding when effects run is crucial for avoiding bugs:

```tsx
function Component() {
  const ref = useRef();
  
  // 1. Runs during RENDER (call this "render body")
  //    Pure computation only. May run multiple times.
  console.log('render');
  
  // 2. Runs during COMMIT, BEFORE browser paint
  //    DOM is updated but not yet visible
  //    Good for: measuring DOM, synchronous DOM adjustments
  useLayoutEffect(() => {
    console.log('layout effect');
    // ref.current is available here
    // DOM measurements are accurate
    // But the user hasn't seen the update yet
    
    return () => console.log('layout cleanup');
  });
  
  // 3. Runs AFTER browser paint
  //    Good for: data fetching, subscriptions, logging
  useEffect(() => {
    console.log('effect');
    
    return () => console.log('effect cleanup');
  });
  
  return <div ref={ref}>Content</div>;
}

// Mount order:
// "render" → DOM updated → "layout effect" → paint → "effect"

// Update order:
// "render" → DOM updated → "layout cleanup" → "layout effect" → paint → "effect cleanup" → "effect"

// Unmount order:
// "layout cleanup" → "effect cleanup"
```

### 3.4 Batching

React automatically batches state updates within the same synchronous context:

```tsx
function handleClick() {
  setCount(c => c + 1);     // Doesn't trigger render
  setName('Alice');          // Doesn't trigger render
  setAge(30);               // Doesn't trigger render
  // React renders ONCE after all three updates
}
```

Since React 18, batching works everywhere — event handlers, promises, setTimeout, native event handlers. Before React 18, batching only worked inside React event handlers.

```tsx
// React 18+: ALL of these are batched
setTimeout(() => {
  setCount(c => c + 1);
  setFlag(f => !f);
  // One render
}, 1000);

fetch('/api/data').then(data => {
  setData(data);
  setLoading(false);
  // One render
});
```

**When you need to force a synchronous update:**
```tsx
import { flushSync } from 'react-dom';

function handleClick() {
  flushSync(() => {
    setCount(c => c + 1);
  });
  // DOM is updated here
  
  flushSync(() => {
    setFlag(f => !f);
  });
  // DOM is updated again here
}
```

`flushSync` is rare in practice — you typically only need it when interfacing with non-React code that needs to read the DOM immediately after a state change.

---

## 4. CONCURRENT FEATURES

Concurrent rendering is the payoff for all the Fiber architecture work. It enables React to prepare UI updates in the background without blocking the main thread.

### 4.1 `useTransition`: Marking Updates as Non-Urgent

`useTransition` tells React: "this state update is not urgent — don't let it block more important updates."

```tsx
function TabContainer() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);  // This update runs at transition priority
    });
  }
  
  return (
    <div>
      <TabBar selectedTab={tab} onSelect={selectTab} />
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </div>
  );
}
```

**What `isPending` actually means:** When `startTransition` is called, `isPending` becomes `true` immediately (synchronously). It stays `true` until the transition update is committed. You can use this to show a loading indicator without removing the current content.

**The key insight:** Without `useTransition`, switching tabs would be a high-priority update. If `TabContent` is expensive to render, the tab bar would freeze during the render. With `useTransition`, the tab bar stays responsive — React can interrupt the `TabContent` render to process user interactions.

**When to use `useTransition`:**
- Tab switches
- Page navigations (Expo Router uses this internally)
- Filter/sort operations on large datasets
- Any state update that triggers an expensive re-render

**When NOT to use `useTransition`:**
- Controlled inputs (the user expects immediate feedback)
- Small, fast updates (the overhead isn't worth it)
- Updates that must be synchronous for accessibility (focus management)

### 4.2 `useDeferredValue`: Deferring a Derived Value

`useDeferredValue` is the declarative counterpart to `useTransition`. Instead of wrapping the *update* in a transition, you wrap the *value*:

```tsx
function SearchResults({ query }) {
  // deferredQuery lags behind query during concurrent renders
  const deferredQuery = useDeferredValue(query);
  
  // This check tells you if the deferred value is stale
  const isStale = query !== deferredQuery;
  
  return (
    <div style={{ opacity: isStale ? 0.7 : 1 }}>
      <SlowResultList query={deferredQuery} />
    </div>
  );
}
```

**How it works under the hood:**
1. `query` updates to "ab" (from props or state)
2. React first renders with `deferredQuery` still as "a" (the stale value) — this is fast because `SlowResultList` memoizes and "a" hasn't changed
3. React then starts a background render with `deferredQuery` as "ab" — this is interruptible
4. If `query` changes to "abc" before the background render finishes, React abandons it and starts over with "abc"

**`useTransition` vs. `useDeferredValue`:**

| | useTransition | useDeferredValue |
|---|---|---|
| What you control | When the update is dispatched | Which value is deferred |
| Where to use | When you control the state update | When you receive a value via props |
| Provides loading state | Yes (`isPending`) | Manual comparison |
| Wraps | The setter function | The value |

```tsx
// Use useTransition when you own the state
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  
  function handleChange(e) {
    setQuery(e.target.value); // Urgent: update the input
    startTransition(() => {
      setFilteredResults(filter(e.target.value)); // Deferred
    });
  }
}

// Use useDeferredValue when you receive the value as a prop
function ResultsList({ query }) {
  const deferredQuery = useDeferredValue(query);
  // Render with deferred value
}
```

### 4.3 Suspense: The Loading State Primitive

Suspense is React's mechanism for declaratively specifying loading states. When a component "suspends" (throws a Promise), React shows the nearest `<Suspense>` boundary's fallback until the Promise resolves.

```tsx
function ProfilePage({ userId }) {
  return (
    <Suspense fallback={<ProfileSkeleton />}>
      <ProfileHeader userId={userId} />
      <Suspense fallback={<PostsSkeleton />}>
        <ProfilePosts userId={userId} />
      </Suspense>
    </Suspense>
  );
}
```

**What "suspending" means internally:**
1. Component function is called during render
2. Component calls a data-fetching hook that realizes data isn't ready
3. The hook throws a Promise
4. React catches the thrown Promise
5. React walks up the fiber tree to find the nearest `<Suspense>` boundary
6. React shows the fallback instead of the suspended subtree
7. When the Promise resolves, React re-renders the suspended component
8. This time the data is ready, so the component renders normally
9. React replaces the fallback with the actual content

**You should NOT throw Promises directly.** Use a framework that integrates with Suspense — React Query (TanStack Query), SWR, Relay, Next.js data fetching, or React's `use` hook.

```tsx
// React's `use` hook (React 19+)
function ProfileHeader({ userId }) {
  // `use` can read a Promise — suspends if not yet resolved
  const user = use(fetchUser(userId));
  
  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.bio}</p>
    </div>
  );
}
```

```tsx
// TanStack Query with Suspense
function ProfilePosts({ userId }) {
  const { data: posts } = useSuspenseQuery({
    queryKey: ['posts', userId],
    queryFn: () => fetchPosts(userId),
  });
  
  return posts.map(post => <PostCard key={post.id} post={post} />);
}
```

### 4.4 Suspense and Transitions Working Together

The real power of Suspense emerges when combined with transitions:

```tsx
function App() {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  function switchTab(newTab) {
    startTransition(() => {
      setTab(newTab);
    });
  }
  
  return (
    <>
      <TabBar 
        selected={tab} 
        onSelect={switchTab}
        isPending={isPending}
      />
      <Suspense fallback={<TabSkeleton />}>
        <TabContent tab={tab} />
      </Suspense>
    </>
  );
}

function TabContent({ tab }) {
  // This component suspends while fetching data for the new tab
  const data = use(fetchTabData(tab));
  return <ExpensiveTabView data={data} />;
}
```

**Without `startTransition`:** When you switch tabs, the Suspense fallback shows immediately (the skeleton replaces the current tab content). This feels janky — the user sees content → skeleton → new content.

**With `startTransition`:** The current tab content stays visible while the new tab's data loads. `isPending` is true during this time, so you can dim the current content or show a subtle spinner. The user sees content → dimmed content → new content. Much smoother.

### 4.5 Suspense Boundaries: Architecture Decisions

Where you place `<Suspense>` boundaries is an architectural decision with real UX consequences:

```tsx
// Strategy 1: One big boundary (all-or-nothing loading)
<Suspense fallback={<FullPageSkeleton />}>
  <Header />      {/* Shows skeleton until ALL data loads */}
  <Sidebar />
  <MainContent />
  <Comments />
</Suspense>

// Strategy 2: Granular boundaries (progressive loading)
<>
  <Suspense fallback={<HeaderSkeleton />}>
    <Header />
  </Suspense>
  <div className="layout">
    <Suspense fallback={<SidebarSkeleton />}>
      <Sidebar />
    </Suspense>
    <Suspense fallback={<ContentSkeleton />}>
      <MainContent />
    </Suspense>
  </div>
  <Suspense fallback={<CommentsSkeleton />}>
    <Comments />
  </Suspense>
</>
```

Strategy 2 is almost always better — content appears as it becomes available, giving the user a progressive loading experience. The key is making skeletons that match the final layout so there's no CLS.

**Nested Suspense for waterfall prevention:**

```tsx
// BAD: Sequential fetching (waterfall)
function ProfilePage() {
  const user = use(fetchUser(id));           // Fetches first
  const posts = use(fetchPosts(user.id));     // Fetches after user loads
  
  return <Profile user={user} posts={posts} />;
}

// GOOD: Parallel fetching with separate boundaries
function ProfilePage() {
  return (
    <>
      <Suspense fallback={<UserSkeleton />}>
        <UserHeader id={id} />
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <UserPosts id={id} />      {/* Fetches in parallel */}
      </Suspense>
    </>
  );
}

function UserHeader({ id }) {
  const user = use(fetchUser(id));
  return <h1>{user.name}</h1>;
}

function UserPosts({ id }) {
  const posts = use(fetchPosts(id));
  return posts.map(p => <PostCard key={p.id} post={p} />);
}
```

### 4.6 `useOptimistic`: Instant Feedback

React 19 introduced `useOptimistic` for optimistic UI updates:

```tsx
function MessageThread({ messages, sendMessage }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    (currentMessages, newMessage) => [
      ...currentMessages,
      { ...newMessage, sending: true },
    ]
  );
  
  async function handleSend(text) {
    const newMessage = { id: crypto.randomUUID(), text, sending: true };
    addOptimistic(newMessage);
    
    // When the action completes, optimistic state is replaced
    // by the new `messages` prop from the server
    await sendMessage(text);
  }
  
  return (
    <div>
      {optimisticMessages.map(msg => (
        <Message key={msg.id} message={msg} />
      ))}
    </div>
  );
}
```

The optimistic value is shown immediately. When the async action completes and the parent re-renders with the new real data, the optimistic state is automatically replaced.

---

## 5. THE REACT COMPILER: AUTOMATIC OPTIMIZATION

The React Compiler (originally called "React Forget") is a build-time compiler that automatically adds memoization to your React components. It shipped as stable with React 19 and is one of the most significant changes to how React code is written.

### 5.1 The Problem It Solves

Without the compiler, React developers must manually optimize re-renders:

```tsx
// Without compiler: Manual memoization hell
function ProductList({ products, onSelect }) {
  const sortedProducts = useMemo(
    () => products.sort((a, b) => a.name.localeCompare(b.name)),
    [products]
  );
  
  const handleSelect = useCallback(
    (id) => onSelect(id),
    [onSelect]
  );
  
  return sortedProducts.map(p => (
    <ProductCard 
      key={p.id}
      product={p}
      onSelect={handleSelect}
    />
  ));
}

const ProductCard = React.memo(function ProductCard({ product, onSelect }) {
  return (
    <div onClick={() => onSelect(product.id)}>
      {product.name}
    </div>
  );
});
```

This is tedious, error-prone (wrong dependency arrays), and adds cognitive load to every component.

### 5.2 What the Compiler Does

The React Compiler analyzes your component functions at build time and automatically inserts memoization where beneficial:

```tsx
// What you write (with compiler):
function ProductList({ products, onSelect }) {
  const sortedProducts = products.sort((a, b) => 
    a.name.localeCompare(b.name)
  );
  
  return sortedProducts.map(p => (
    <ProductCard 
      key={p.id}
      product={p}
      onSelect={() => onSelect(p.id)}
    />
  ));
}

function ProductCard({ product, onSelect }) {
  return (
    <div onClick={onSelect}>
      {product.name}
    </div>
  );
}

// What the compiler outputs (conceptually):
function ProductList({ products, onSelect }) {
  const $ = useMemoCache(4); // Compiler's internal memoization hook
  
  let sortedProducts;
  if ($[0] !== products) {
    sortedProducts = products.sort((a, b) => 
      a.name.localeCompare(b.name)
    );
    $[0] = products;
    $[1] = sortedProducts;
  } else {
    sortedProducts = $[1];
  }
  
  let renderedList;
  if ($[0] !== products || $[2] !== onSelect) {
    renderedList = sortedProducts.map(p => (
      <ProductCard 
        key={p.id}
        product={p}
        onSelect={() => onSelect(p.id)}
      />
    ));
    $[2] = onSelect;
    $[3] = renderedList;
  } else {
    renderedList = $[3];
  }
  
  return renderedList;
}
```

**The compiler tracks data flow through your component** and memoizes at the granularity of individual values, not entire components. This is more efficient than manual `useMemo`/`useCallback` because:
- It memoizes exactly what needs memoization (no over-memoization)
- It handles dependency tracking automatically (no wrong dep arrays)
- It can memoize JSX elements, not just values

### 5.3 The Rules of React (Enforced by the Compiler)

The compiler relies on components following the **Rules of React:**

1. **Components and hooks must be pure** — same inputs produce same outputs, no side effects during render
2. **Props and state are immutable** — never mutate them directly
3. **Hook call order must be stable** — no hooks inside conditions or loops
4. **Side effects belong in effects** — not in the render body

```tsx
// BAD: Mutates during render — compiler can't optimize
function Component({ items }) {
  items.sort(); // Mutates the prop!
  return <List items={items} />;
}

// GOOD: Create a new sorted array
function Component({ items }) {
  const sorted = [...items].sort();
  return <List items={sorted} />;
}

// BAD: Side effect during render
function Component({ id }) {
  analytics.track('rendered', { id }); // Side effect!
  return <div>{id}</div>;
}

// GOOD: Side effect in an effect
function Component({ id }) {
  useEffect(() => {
    analytics.track('rendered', { id });
  }, [id]);
  return <div>{id}</div>;
}
```

### 5.4 Compiler Configuration

```javascript
// babel.config.js (for React Native / Expo)
module.exports = {
  plugins: [
    ['babel-plugin-react-compiler', {
      // Target specific components (for gradual adoption)
      sources: (filename) => {
        return filename.includes('src/components');
      },
    }],
  ],
};

// next.config.js (for Next.js)
module.exports = {
  experimental: {
    reactCompiler: true,
  },
};

// For Expo (app.json or app.config.js)
{
  "expo": {
    "experiments": {
      "reactCompiler": true
    }
  }
}
```

### 5.5 Opting Out

If the compiler incorrectly optimizes a component (rare, but possible with code that violates the Rules of React):

```tsx
// Opt out a specific component
function LegacyComponent() {
  'use no memo';
  // Compiler skips this component
  return <div>...</div>;
}
```

### 5.6 Should You Remove Manual Memoization?

**Yes, gradually.** With the compiler enabled:
- Remove `useMemo` — the compiler handles this
- Remove `useCallback` — the compiler handles this
- Remove `React.memo` — the compiler handles prop memoization
- Keep `useMemo` for intentional semantic guarantees (e.g., a ref identity that must be stable)

```tsx
// Before compiler: necessary
const MemoizedChild = React.memo(Child);
const sortedItems = useMemo(() => items.sort(compareFn), [items]);
const handleClick = useCallback(() => onClick(id), [onClick, id]);

// After compiler: just write the obvious code
const sortedItems = items.sort(compareFn);
function handleClick() { onClick(id); }
// Child is automatically memoized by the compiler
```

---

## 6. REACT SERVER COMPONENTS

React Server Components (RSC) fundamentally change the rendering pipeline by splitting it between server and client.

### 6.1 The Two-World Model

```
Server Components                 Client Components
  ┌────────────────┐               ┌────────────────┐
  │ Run on server   │               │ Run on client   │
  │ No state        │               │ Full interactivity │
  │ No effects      │               │ useState, useEffect │
  │ async/await     │               │ Event handlers   │
  │ Direct DB access│               │ Browser APIs     │
  │ Zero JS shipped │               │ JS shipped to client │
  └────────────────┘               └────────────────┘
```

**Server Components are the default** in frameworks that support RSC (Next.js App Router, Expo Router for web). You opt *into* client components with `'use client'`.

```tsx
// Server Component (default) — runs only on the server
import { db } from '@/lib/database';

async function ProductPage({ params }) {
  const product = await db.products.findById(params.id);
  
  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} />
    </div>
  );
}

// Client Component — runs on the client (and server for SSR)
'use client';

function AddToCartButton({ productId }) {
  const [isAdding, setIsAdding] = useState(false);
  
  async function handleClick() {
    setIsAdding(true);
    await addToCart(productId);
    setIsAdding(false);
  }
  
  return (
    <button onClick={handleClick} disabled={isAdding}>
      {isAdding ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

### 6.2 The RSC Rendering Pipeline

```
1. Server receives request
2. React renders Server Components on the server
   - Calls async component functions
   - Resolves data fetching (DB queries, API calls)
   - Produces a serialized tree (RSC Payload)
3. RSC Payload is streamed to the client
4. Client receives the payload
5. React on the client:
   - Deserializes the server tree
   - Hydrates Client Components (adds event handlers, state)
   - Merges server tree + client tree into the final DOM
6. Subsequent navigations:
   - Only fetch new RSC Payload for changed parts
   - Client Components maintain their state
```

**The RSC Payload is NOT HTML.** It's a React-specific serialization format that contains:
- The rendered output of Server Components (like a virtual DOM snapshot)
- References to Client Component bundles (for code splitting)
- Serialized props passed from Server to Client Components

### 6.3 The Boundary Rules

Server and Client Components have strict rules about how they interact:

```tsx
// Server Component CAN render Client Components
function ServerPage() {
  return <ClientButton label="Click me" />; // OK
}

// Client Component CANNOT import Server Components directly
'use client';
function ClientWrapper() {
  // import ServerComponent from './ServerComponent'; // ERROR
  // return <ServerComponent />;
}

// But Client Components CAN receive Server Components as children
'use client';
function ClientLayout({ children }) {
  const [isOpen, setIsOpen] = useState(true);
  return isOpen ? children : null; // children can be Server Components
}

// Usage:
function ServerPage() {
  return (
    <ClientLayout>
      <ServerContent />  {/* Server Component passed as children */}
    </ClientLayout>
  );
}
```

### 6.4 Server Actions

Server Actions are functions that run on the server but can be called from the client:

```tsx
// Server Action (defined with 'use server')
'use server';

async function updateProfile(formData: FormData) {
  const name = formData.get('name') as string;
  const bio = formData.get('bio') as string;
  
  await db.users.update({
    where: { id: currentUser.id },
    data: { name, bio },
  });
  
  revalidatePath('/profile');
}

// Client Component using the Server Action
'use client';

function ProfileForm({ user }) {
  return (
    <form action={updateProfile}>
      <input name="name" defaultValue={user.name} />
      <textarea name="bio" defaultValue={user.bio} />
      <button type="submit">Save</button>
    </form>
  );
}
```

Server Actions integrate with React's form handling, Suspense, and transitions. They're the modern replacement for API routes in many cases.

---

## 7. PERFORMANCE MENTAL MODELS

### 7.1 The Re-render Waterfall

The #1 React performance problem is unnecessary re-renders cascading through the tree:

```tsx
// The problem:
function App() {
  const [theme, setTheme] = useState('light');
  //    ↓ Every state change re-renders App
  //    ↓ Which re-renders Header, Main, Footer
  //    ↓ Which re-renders all their children
  //    ↓ Which re-renders all THEIR children
  return (
    <div data-theme={theme}>
      <Header />      {/* Re-renders even if theme isn't used */}
      <Main />        {/* Re-renders even if theme isn't used */}
      <Footer />      {/* Re-renders even if theme isn't used */}
    </div>
  );
}
```

**Solution 1: Composition (the best solution)**
```tsx
// Move state closer to where it's used
function App() {
  return (
    <div>
      <ThemeWrapper>  {/* Only this subtree re-renders on theme change */}
        <ThemedContent />
      </ThemeWrapper>
      <Header />      {/* Never re-renders from theme changes */}
      <Main />        {/* Never re-renders from theme changes */}
    </div>
  );
}
```

**Solution 2: Children pattern**
```tsx
// The children are created by the PARENT, not by ThemeProvider
// So they're the same reference across renders
function App() {
  return (
    <ThemeProvider>
      <Header />    {/* Same JSX element reference = skip re-render */}
      <Main />
      <Footer />
    </ThemeProvider>
  );
}

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <div data-theme={theme}>
        {children}  {/* children reference hasn't changed */}
      </div>
    </ThemeContext.Provider>
  );
}
```

**Solution 3: React Compiler (automatic)**
With the React Compiler enabled, components that don't use `theme` are automatically skipped because the compiler memoizes their output.

### 7.2 Context Performance

Context is often blamed for performance problems, but the real issue is usually how it's structured:

```tsx
// BAD: One giant context
const AppContext = createContext({
  user: null,
  theme: 'light',
  locale: 'en',
  notifications: [],
  cart: [],
  // Changing ANY value re-renders ALL consumers
});

// GOOD: Split contexts by change frequency
const UserContext = createContext(null);      // Changes rarely
const ThemeContext = createContext('light');   // Changes rarely
const CartContext = createContext([]);         // Changes moderately
const NotificationContext = createContext([]); // Changes frequently
```

**The context value reference trap:**
```tsx
// BAD: New object every render = all consumers re-render
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {/*                         ^^^^^^^^^^^^^^^^^ new object every render */}
      {children}
    </ThemeContext.Provider>
  );
}

// GOOD: Stable reference (compiler handles this automatically)
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  const value = useMemo(() => ({ theme, setTheme }), [theme]);
  
  return (
    <ThemeContext.Provider value={value}>
      {children}
    </ThemeContext.Provider>
  );
}
```

### 7.3 List Virtualization

For lists with hundreds or thousands of items, don't render them all:

```tsx
// React Native: FlashList (recommended over FlatList)
import { FlashList } from '@shopify/flash-list';

function ProductList({ products }) {
  return (
    <FlashList
      data={products}
      renderItem={({ item }) => <ProductCard product={item} />}
      estimatedItemSize={100}
      keyExtractor={item => item.id}
    />
  );
}

// Web: TanStack Virtual
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);
  
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,
  });
  
  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              height: virtualRow.size,
            }}
          >
            <ItemComponent item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 7.4 The Component Rendering Decision Tree

```
Should this component re-render?
│
├─ Did its props change?
│  ├─ Yes → Re-render (unless memoized and props are shallowly equal)
│  └─ No → Did its state change?
│     ├─ Yes → Re-render
│     └─ No → Did its context change?
│        ├─ Yes → Re-render
│        └─ No → Did parent re-render?
│           ├─ Yes → Re-render (unless React.memo or compiler memoized)
│           └─ No → Don't re-render
```

**With the React Compiler, the answer simplifies to:**
```
Should this component re-render?
│
├─ Did any of its inputs (props, state, context) change?
│  ├─ Yes → Re-render
│  └─ No → Don't re-render (compiler ensures this)
```

### 7.5 Measuring Render Performance

```tsx
// React DevTools Profiler
// Open React DevTools → Profiler tab → Record → Interact → Stop

// Programmatic profiling
import { Profiler } from 'react';

function App() {
  return (
    <Profiler id="Main" onRender={onRenderCallback}>
      <MainContent />
    </Profiler>
  );
}

function onRenderCallback(
  id,              // The "id" prop of the Profiler tree
  phase,           // "mount" or "update"
  actualDuration,  // Time spent rendering the committed update
  baseDuration,    // Time to render the entire subtree without memoization
  startTime,       // When React began rendering this update
  commitTime,      // When React committed this update
) {
  // Send to monitoring
  if (actualDuration > 16) { // More than one frame (60fps)
    console.warn(`Slow render: ${id} took ${actualDuration}ms (${phase})`);
  }
}
```

```tsx
// Why Did You Render (development tool)
// npm install @welldone-software/why-did-you-render
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    trackHooks: true,
  });
}

// Then on specific components:
MyComponent.whyDidYouRender = true;
```

---

## 8. REACT 19 AND BEYOND

### 8.1 The `use` Hook

`use` is a new hook that can read Promises and Context:

```tsx
// Reading a Promise (replaces use + Suspense manual wiring)
function UserProfile({ userPromise }) {
  const user = use(userPromise); // Suspends until resolved
  return <h1>{user.name}</h1>;
}

// Reading Context (can be called conditionally!)
function ThemeButton({ showTheme }) {
  if (showTheme) {
    const theme = use(ThemeContext); // Unlike useContext, this is conditional
    return <button className={theme}>Themed</button>;
  }
  return <button>Default</button>;
}
```

`use` is special because it **doesn't follow the rules of hooks** — it can be called inside conditions and loops. It's not really a hook in the traditional sense; it's a new primitive.

### 8.2 Actions

Actions are the unifying concept behind form handling, Server Actions, and optimistic updates in React 19:

```tsx
function TodoForm() {
  const [error, submitAction, isPending] = useActionState(
    async (previousState, formData) => {
      const title = formData.get('title');
      const result = await createTodo(title);
      
      if (result.error) {
        return { error: result.error };
      }
      
      return { error: null };
    },
    { error: null }
  );
  
  return (
    <form action={submitAction}>
      <input name="title" />
      {error?.error && <p className="error">{error.error}</p>}
      <button disabled={isPending}>
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
    </form>
  );
}
```

### 8.3 Document Metadata

React 19 supports rendering `<title>`, `<meta>`, and `<link>` directly in components:

```tsx
function BlogPost({ post }) {
  return (
    <>
      <title>{post.title}</title>
      <meta name="description" content={post.excerpt} />
      <meta property="og:title" content={post.title} />
      <link rel="canonical" href={`https://blog.com/${post.slug}`} />
      
      <article>
        <h1>{post.title}</h1>
        <p>{post.content}</p>
      </article>
    </>
  );
}
```

React automatically hoists these to the `<head>`. No need for `react-helmet` or framework-specific head management.

---

## 9. THE COMPLETE RENDERING TIMELINE

Let's put it all together — a complete trace of what happens when a user clicks a button in a React application:

```
T+0ms    User clicks button
T+0ms    Browser enqueues pointer event
T+1ms    Browser dispatches click event to React's root handler
T+1ms    React determines the event target fiber
T+1ms    React calls your onClick handler
T+1ms    Your handler calls setState(newValue)
T+1ms    React creates an update object, assigns it to SyncLane
T+1ms    React schedules a render via microtask

T+2ms    RENDER PHASE BEGINS
T+2ms    React starts at the fiber that owns the state
T+2ms    Calls the component function with new state
T+3ms    React diffs the returned elements against the current fiber tree
T+3ms    Walks child fibers, calling component functions as needed
T+4ms    React Compiler's memoization skips unchanged subtrees
T+5ms    Render phase complete — effect flags set on changed fibers

T+5ms    COMMIT PHASE BEGINS
T+5ms    React walks fibers with effect flags
T+5ms    Applies DOM insertions, updates, deletions
T+6ms    Runs useLayoutEffect callbacks

T+6ms    React yields to browser
T+6ms    Browser runs style recalculation
T+7ms    Browser runs layout
T+7ms    Browser runs paint
T+8ms    Browser composites and displays

T+8ms    TOTAL: 8ms from click to visual update (well within 16ms frame budget)

T+9ms    Browser runs requestAnimationFrame callbacks
T+10ms   React runs useEffect callbacks (passive effects)
```

For a transition update (e.g., `startTransition(() => setState(...))`), the timeline includes yield points:

```
T+0ms    startTransition callback runs
T+0ms    setState assigned to TransitionLane (low priority)
T+0ms    React schedules render via scheduler

T+5ms    RENDER PHASE BEGINS (interruptible)
T+5ms    React processes 10 fibers
T+10ms   React yields to browser (5ms time slice)
T+10ms   Browser handles any pending events
T+11ms   React resumes, processes 10 more fibers
T+16ms   React yields again
...
T+40ms   Render phase complete

T+40ms   COMMIT PHASE (same as sync, non-interruptible)
T+42ms   Visual update appears
```

The key difference: during transition renders, React yields every ~5ms, ensuring the main thread stays responsive for user input. A sync update would block for the full 40ms.

---

## 10. CHAPTER SUMMARY

React's rendering pipeline is a carefully orchestrated sequence:
1. **Reconciliation** diffs the new element tree against the current fiber tree
2. **Fiber architecture** makes this work interruptible and priority-aware
3. **The Render Phase** is pure and can be paused/restarted
4. **The Commit Phase** is synchronous and applies DOM mutations atomically
5. **Concurrent features** (useTransition, useDeferredValue, Suspense) let you control priority
6. **React Compiler** eliminates manual memoization
7. **React Server Components** split rendering between server and client

**The mental model that matters:**

React is not a DOM manipulation library. React is a **scheduling framework** that manages the priority and timing of UI updates. The Fiber architecture, lanes system, and concurrent features all exist to ensure that high-priority updates (user input) are never blocked by low-priority work (data processing, navigation transitions).

The engineers who understand this pipeline don't reach for `useMemo` as a first resort — they structure their component tree so expensive computations are isolated, transitions are non-blocking, and the commit phase is small. They know that a `useEffect` runs after paint and a `useLayoutEffect` runs before. They know that the React Compiler handles the mechanical optimization, freeing them to focus on architectural decisions that no compiler can automate.

---

> **Next:** [Chapter 4: TypeScript at Scale] covers how to structure TypeScript for large codebases — project references, build performance, and type-level architecture patterns.
