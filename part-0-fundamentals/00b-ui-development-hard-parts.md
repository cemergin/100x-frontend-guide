<!--
  CHAPTER: 0b
  TITLE: UI Development — The Hard Parts
  PART: 0 — JavaScript & React Fundamentals
  PREREQS: Chapter 0
  KEY_TOPICS: DOM, CSSOM, WebIDL, data binding, one-way data binding, virtual DOM, diffing, reconciliation, composition, declarative UI, UI components
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 0b: UI Development — The Hard Parts

> **Part 0 — JavaScript & React Fundamentals** | Prerequisites: Chapter 0 | Difficulty: Beginner to Intermediate

Every UI framework that has ever existed — React, Angular, Vue, Svelte, jQuery, Backbone, Ember, React Native, SwiftUI, Jetpack Compose — solves the same fundamental problem: **keeping what the user sees in sync with the application's data.**

That's it. That's the entire problem. You have data (a list of todos, a user profile, a shopping cart). You have a screen (pixels on a display). When the data changes, the screen needs to change to match. When the user interacts with the screen (types in an input, clicks a button), the data needs to change to match.

This sounds simple. It is catastrophically hard at scale.

The reason it's hard is not the concept — it's the implementation. The screen is not a JavaScript data structure. It's a tree of C++ objects managed by a rendering engine that was designed in the 1990s for static documents. JavaScript doesn't directly control pixels. It talks to the DOM, which talks to the layout engine, which talks to the painting engine, which talks to the compositor, which talks to the GPU. Every step in that chain has performance implications, ordering constraints, and failure modes.

React's genius was not inventing components or virtual DOM or JSX. React's genius was identifying the core problem — data-to-UI synchronization — and solving it with a simple, predictable model that happens to be built on closures, higher-order functions, and the execution model you learned in Chapter 0.

This chapter gives you the complete picture: what the DOM actually is (it's C++), why manipulating it is expensive, how data binding works (and why it's hard), and how the virtual DOM solves the synchronization problem. By the end, React won't feel like magic. It'll feel like the obvious solution.

### In This Chapter
- The DOM: a C++ data structure accessed through JavaScript bindings
- The rendering pipeline: DOM → CSSOM → Layout → Paint → Composite
- Data binding: the fundamental challenge of UI development
- The Virtual DOM: React's solution, built from scratch
- Reconciliation: the diffing algorithm that makes it fast
- UI composition: components as functions
- Hooks as closures: connecting Chapter 0 to React
- The declarative paradigm: why it won

### Related Chapters
- [Ch 0: JavaScript — The Hard Parts] — closures, event loop, and execution model (prerequisite)
- [Ch 0c: React — From First Principles] — applying these concepts in React
- [Ch 1: React Native Architecture & Internals] — how JSI mirrors the DOM's WebIDL bindings
- [Ch 2: Browser Rendering & Web Fundamentals] — the full browser rendering pipeline in production depth
- [Ch 3: The Rendering Pipeline — Mobile & Web] — React reconciliation, Fiber, and concurrent features

---

## 1. THE DISPLAY PROBLEM

Before we talk about any specific technology, let's state the problem clearly.

You have application state:

```javascript
const state = {
  todos: [
    { id: 1, text: "Learn JavaScript internals", done: true },
    { id: 2, text: "Understand the DOM", done: false },
    { id: 3, text: "Build with React", done: false },
  ],
  filter: "all",
};
```

You have a display:

```
+-------------------------------------+
|  My Todos                           |
|                                     |
|  [x] Learn JavaScript internals     |
|  [ ] Understand the DOM             |
|  [ ] Build with React               |
|                                     |
|  Showing: all | active | completed  |
+-------------------------------------+
```

Now the user checks "Understand the DOM":

```javascript
state.todos[1].done = true;
```

The display needs to update:

```
+-------------------------------------+
|  My Todos                           |
|                                     |
|  [x] Learn JavaScript internals     |
|  [x] Understand the DOM             |  <-- changed
|  [ ] Build with React               |
|                                     |
|  Showing: all | active | completed  |
+-------------------------------------+
```

Only ONE element changed. But how does the system know which element? How does it update just that element without rebuilding the entire display? And how does it do this efficiently when the state is complex, the changes are frequent, and the user expects 60fps responsiveness?

This is the display problem. Every UI framework is an answer to this question.

---

## 2. THE DOM: C++ MEETS JAVASCRIPT

### 2.1 What the DOM Actually Is

The DOM (Document Object Model) is NOT a JavaScript API. It is a **C++ data structure** — a tree of C++ objects that live inside the browser's rendering engine (WebCore in WebKit, Blink in Chrome). These objects represent elements on screen: divs, paragraphs, buttons, inputs.

When you write:

```javascript
const div = document.createElement("div");
```

You are NOT creating a JavaScript object. You are asking the browser's C++ engine to create a native C++ object — an instance of the `HTMLDivElement` class in the rendering engine's codebase. JavaScript receives a **handle** to that C++ object through a binding layer called **WebIDL** (Web Interface Definition Language).

```
+----------------------------------------------------------+
|                  JAVASCRIPT ENGINE (V8)                    |
|                                                           |
|   const div = document.createElement("div");              |
|   // div is a JS wrapper around a C++ object              |
|                                                           |
+-----------------------------+-----------------------------+
                              | WebIDL binding
                              v
+----------------------------------------------------------+
|               RENDERING ENGINE (Blink/WebCore)            |
|                                                           |
|   HTMLDivElement* div = new HTMLDivElement();              |
|   // This is the REAL object -- lives in C++ memory       |
|   // Has layout properties, style properties,             |
|   // paint instructions, accessibility info               |
|                                                           |
+----------------------------------------------------------+
```

**Why this matters:** Every DOM operation — creating an element, changing a style, reading a dimension — crosses the JavaScript-to-C++ boundary. This boundary crossing is not free. It involves:

1. Marshaling data from JavaScript's memory model to C++'s memory model
2. Potentially triggering layout recalculation (reflow)
3. Potentially triggering repaint
4. Synchronization between the JS thread and the rendering pipeline

This is why "DOM manipulation is slow" — not because the DOM objects themselves are slow, but because the boundary crossing and the side effects (layout, paint) are expensive.

### 2.2 The Connection to React Native's JSI

If this sounds familiar, it should. In Chapter 1, you'll learn about JSI (JavaScript Interface) — React Native's C++ layer that lets JavaScript hold references to native C++ objects and call methods on them directly.

```
Web DOM:       JavaScript --> WebIDL --> C++ DOM objects (Blink/WebCore)
React Native:  JavaScript --> JSI    --> C++ native objects (Fabric/TurboModules)
```

The concept is identical. In both cases, JavaScript doesn't directly control the UI. It holds handles to native objects through a C++ binding layer. The DOM is literally the original JSI — a bridge between a scripting language and a native rendering engine. Understanding one gives you the mental model for the other.

### 2.3 The DOM Tree

The DOM is a tree. HTML describes the tree's structure. The browser parses HTML into DOM nodes:

```html
<html>
  <body>
    <div id="app">
      <h1>Hello</h1>
      <p>World</p>
    </div>
  </body>
</html>
```

```
document
  +-- html (HTMLHtmlElement)
        +-- body (HTMLBodyElement)
              +-- div#app (HTMLDivElement)
                    |-- h1 (HTMLHeadingElement)
                    |     +-- "Hello" (Text node)
                    +-- p (HTMLParagraphElement)
                          +-- "World" (Text node)
```

Each node is a C++ object. Each node has pointers to its parent, children, and siblings. Traversing the tree, adding nodes, removing nodes — these are all operations on C++ data structures, accessible from JavaScript through the WebIDL bindings.

### 2.4 The CSSOM

Parallel to the DOM, the browser builds the **CSSOM** (CSS Object Model) — a tree of style rules. The CSSOM represents every CSS rule that applies to the page, including browser defaults, external stylesheets, inline styles, and computed styles.

```
DOM Tree + CSSOM --> Render Tree --> Layout --> Paint --> Composite
```

The **Render Tree** combines the DOM and CSSOM: for each visible DOM node, it computes the final styles. Nodes with `display: none` are excluded. Pseudo-elements like `::before` are included even though they're not in the DOM.

### 2.5 The Rendering Pipeline

When you change the DOM, the browser has to recalculate what should appear on screen. The pipeline:

```
1. DOM/CSSOM change detected
          |
          v
2. Style calculation -- which CSS rules apply to which elements?
          |
          v
3. Layout (Reflow) -- where is each element? How big is it?
          |
          v
4. Paint -- what does each element look like? (colors, borders, shadows, text)
          |
          v
5. Composite -- combine painted layers and send to GPU
          |
          v
6. Pixels on screen
```

**The performance implications:**

- **Changing `color`:** skips Layout, does Paint + Composite. Cheap.
- **Changing `width`:** triggers Layout + Paint + Composite. Expensive.
- **Changing `transform` or `opacity`:** only Composite. Very cheap (GPU only).
- **Adding/removing DOM nodes:** potentially triggers Style + Layout + Paint + Composite. Most expensive.

This is why React (and every framework) tries to minimize DOM operations. Every DOM change can trigger an expensive pipeline. Batching changes, minimizing layout thrashing, and using compositor-only properties are fundamental optimization strategies.

### 2.6 Layout Thrashing

The most insidious DOM performance problem is **layout thrashing** — alternating between reading and writing DOM properties:

```javascript
// BAD -- layout thrashing
for (const el of elements) {
  const height = el.offsetHeight; // READ -- forces layout calculation
  el.style.height = height + 10 + "px"; // WRITE -- invalidates layout
  // Next iteration: READ forces a NEW layout calculation
}
```

Each read after a write forces the browser to recalculate layout synchronously (normally layout is batched). If you do this in a loop with 1000 elements, you force 1000 layout recalculations instead of 1.

```javascript
// GOOD -- batch reads, then batch writes
const heights = elements.map(el => el.offsetHeight); // all reads first
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + "px"; // all writes after
});
// Browser does ONE layout recalculation
```

React's Virtual DOM eliminates this problem entirely — all "reads" happen on the virtual DOM (a JavaScript data structure, no layout involved), and all "writes" are batched into a single DOM update.

---

## 3. DATA BINDING

Data binding is the mechanism that connects application data to UI elements. There are several approaches, each with different trade-offs.

### 3.1 Manual DOM Manipulation (Imperative)

The oldest approach: you manually update the DOM whenever your data changes.

```javascript
// State
let count = 0;

// Initial render
const button = document.createElement("button");
button.textContent = `Count: ${count}`;
document.body.appendChild(button);

// Update handler
button.addEventListener("click", () => {
  count++;
  button.textContent = `Count: ${count}`; // manually update the DOM
});
```

This works for simple cases. It falls apart at scale because:

1. **You must track every piece of DOM that depends on each piece of data.** If `userName` appears in the header, the sidebar, and a greeting message, you need to update three DOM elements whenever `userName` changes.

2. **You must remember to update the DOM.** If you change `count` but forget to update `button.textContent`, the UI is out of sync. There's no mechanism to catch this.

3. **Complex UIs become unmanageable.** A realistic app has hundreds of data-to-DOM relationships. Tracking them manually leads to bugs, stale UI, and code that's impossible to refactor.

jQuery was the most popular framework for manual DOM manipulation. It made the API nicer (`$('#button').text('Count: ' + count)`) but didn't solve the fundamental problem: you still had to manually track every data-to-DOM relationship.

### 3.2 Two-Way Data Binding (Angular 1, Knockout)

Two-way data binding automatically syncs data and DOM in both directions:
- Data changes --> DOM updates automatically
- DOM changes (user input) --> data updates automatically

```html
<!-- Angular 1 style (conceptual) -->
<input ng-model="userName" />
<p>Hello, {{userName}}</p>
```

When the user types in the input, `userName` updates automatically. When `userName` changes from code, the input and paragraph update automatically.

**The problem:** Two-way binding creates circular dependencies. A changes B, B changes A, A changes B... In complex UIs, it becomes extremely hard to trace why something changed. A style update in one component triggers a cascade of bindings that updates a dozen other components. Debugging becomes a nightmare because there's no single direction of data flow.

Angular 1's `$digest` cycle would run through all bindings repeatedly until nothing changed — or until it hit a cycle limit and threw an error. This was both slow (checking every binding on every change) and unpredictable (cascading updates could cause unexpected behavior).

### 3.3 One-Way Data Flow (React's Approach)

React chose a fundamentally different model: **one-way data flow with explicit events.**

```
Data flows DOWN (parent --> child via props)
Events flow UP (child --> parent via callbacks)
```

```jsx
function Parent() {
  const [name, setName] = useState("Alice");
  
  return (
    <div>
      <Input value={name} onChange={setName} />    {/* data down, event up */}
      <Greeting name={name} />                      {/* data down */}
    </div>
  );
}

function Input({ value, onChange }) {
  return <input value={value} onChange={e => onChange(e.target.value)} />;
}

function Greeting({ name }) {
  return <p>Hello, {name}</p>;
}
```

Data only flows in one direction: from `Parent` down to `Input` and `Greeting`. If `Input` wants to change the data, it calls the `onChange` callback (an event flowing up). The parent updates its state, and the new data flows back down.

**Why this is better:**
1. **Predictable:** You can always trace where data came from. Follow the props.
2. **Debuggable:** If the greeting shows the wrong name, the bug is either in `Parent`'s state or in `Greeting`'s props. There are only two places to look.
3. **Scalable:** Even in a large app, data flow is a tree. It doesn't form cycles.

The trade-off: more boilerplate (you need to pass callbacks explicitly) and sometimes "prop drilling" (passing props through many layers). But the predictability is worth it, and solutions like context and state management libraries handle the boilerplate.

---

## 4. THE VIRTUAL DOM

Now we arrive at React's core innovation. The Virtual DOM is the mechanism that solves the display problem efficiently: keeping the DOM in sync with application data without the performance cost of manual DOM manipulation and without the unpredictability of two-way binding.

### 4.1 The Concept

Instead of manipulating the real DOM directly, React maintains a **JavaScript representation** of the DOM — a plain JavaScript object tree that mirrors the structure of the real DOM.

```javascript
// A virtual DOM node (simplified)
{
  type: "div",
  props: {
    className: "container",
    children: [
      {
        type: "h1",
        props: {
          children: "Hello, World"
        }
      },
      {
        type: "p",
        props: {
          children: "Welcome to React"
        }
      }
    ]
  }
}
```

This is just a JavaScript object. Creating it is fast — no C++ boundary crossing, no layout calculations, no rendering pipeline. It's just allocating objects in the JavaScript heap.

### 4.2 The Algorithm

When data changes, React:

1. **Renders** — calls your component functions to produce a NEW virtual DOM tree
2. **Diffs** — compares the new tree with the previous tree to find what changed
3. **Commits** — applies ONLY the changes to the real DOM

```
State Change
     |
     v
New Virtual DOM <-- Component functions return new tree
     |
     v
    DIFF  <-- Compare new tree with old tree
     |
     v
Minimal DOM Operations <-- Only update what actually changed
     |
     v
Browser renders the changes
```

This is the key insight: **React never reads from the real DOM to decide what to update.** It compares two JavaScript object trees (fast) and computes the minimal set of DOM operations (the expensive part is minimized).

### 4.3 Building a Virtual DOM from Scratch

Let's build a simplified virtual DOM to understand how it works. This is roughly 50 lines of code — the core concept is not complex.

**Step 1: Creating virtual nodes**

```javascript
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "string"
          ? { type: "TEXT", props: { nodeValue: child, children: [] } }
          : child
      ),
    },
  };
}
```

This is what `React.createElement` does (and what JSX compiles to). It creates a plain JavaScript object describing an element.

```javascript
// JSX:        <div className="app"><h1>Hello</h1></div>
// Compiles to:
createElement("div", { className: "app" },
  createElement("h1", null, "Hello")
);
// Returns:
// {
//   type: "div",
//   props: {
//     className: "app",
//     children: [{
//       type: "h1",
//       props: {
//         children: [{
//           type: "TEXT",
//           props: { nodeValue: "Hello", children: [] }
//         }]
//       }
//     }]
//   }
// }
```

**Step 2: Rendering virtual nodes to the real DOM**

```javascript
function render(vnode, container) {
  // Create the real DOM element
  const element = vnode.type === "TEXT"
    ? document.createTextNode("")
    : document.createElement(vnode.type);
  
  // Set properties
  Object.keys(vnode.props)
    .filter(key => key !== "children")
    .forEach(key => {
      if (key.startsWith("on")) {
        // Event handlers: onClick --> click
        const eventType = key.toLowerCase().substring(2);
        element.addEventListener(eventType, vnode.props[key]);
      } else {
        element[key] = vnode.props[key];
      }
    });
  
  // Recursively render children
  vnode.props.children.forEach(child => render(child, element));
  
  // Append to container
  container.appendChild(element);
}
```

This creates the real DOM from the virtual DOM. It's a recursive tree walk that creates elements, sets properties, and appends children.

**Step 3: Diffing two virtual DOM trees**

This is where the magic happens. Given an old virtual DOM and a new virtual DOM, compute the minimal set of changes:

```javascript
function diff(oldVNode, newVNode, parent, index = 0) {
  const existingElement = parent.childNodes[index];
  
  // Case 1: New node added (no old node)
  if (!oldVNode) {
    render(newVNode, parent);
    return;
  }
  
  // Case 2: Old node removed (no new node)
  if (!newVNode) {
    parent.removeChild(existingElement);
    return;
  }
  
  // Case 3: Node type changed -- replace entirely
  if (oldVNode.type !== newVNode.type) {
    const newElement = createRealElement(newVNode);
    parent.replaceChild(newElement, existingElement);
    return;
  }
  
  // Case 4: Same type -- update props and recurse on children
  updateProps(existingElement, oldVNode.props, newVNode.props);
  
  // Diff children recursively
  const maxLength = Math.max(
    oldVNode.props.children.length,
    newVNode.props.children.length
  );
  for (let i = 0; i < maxLength; i++) {
    diff(
      oldVNode.props.children[i],
      newVNode.props.children[i],
      existingElement,
      i
    );
  }
}

function updateProps(element, oldProps, newProps) {
  // Remove old props not in new props
  Object.keys(oldProps)
    .filter(key => key !== "children" && !(key in newProps))
    .forEach(key => { element[key] = ""; });
  
  // Set new/changed props
  Object.keys(newProps)
    .filter(key => key !== "children" && oldProps[key] !== newProps[key])
    .forEach(key => { element[key] = newProps[key]; });
}
```

**That's it.** This is the core of the virtual DOM: create virtual nodes, diff old and new trees, apply minimal changes to the real DOM.

### 4.4 Why the Virtual DOM Is Fast

Common misconception: "The Virtual DOM is fast because JavaScript is faster than the DOM."

**Actual reason:** The Virtual DOM is fast because it **minimizes DOM operations.** DOM operations are expensive (C++ boundary, layout recalculation, paint). JavaScript object comparisons are cheap. By doing the comparison in JavaScript land and only touching the DOM for actual changes, React avoids the most expensive part of UI updates.

Consider updating a list of 1000 items where 3 items changed:

**Without Virtual DOM:**
```javascript
// Option A: Replace everything (simple but slow)
// Wipe the container and re-render all items from scratch.
// Creates 1000 DOM elements, triggers full layout recalculation.

// Option B: Manually track changes (fast but complex)
// You need to know which 3 items changed and update them specifically.
// This is what jQuery apps had to do -- and it's a maintenance nightmare.
```

**With Virtual DOM:**
```javascript
// React creates two JavaScript object trees, diffs them,
// and finds the 3 items that changed.
// Only 3 DOM operations are performed.
// You write the same code regardless of how many items changed.
```

The Virtual DOM is an optimization that trades a small JavaScript overhead (creating and diffing virtual nodes) for a massive reduction in DOM operations. For most UIs, this trade-off is overwhelmingly positive.

### 4.5 The Key Prop and List Reconciliation

When diffing lists, React needs to match old children with new children. Without hints, it compares by position:

```
Old:  [A, B, C]
New:  [A, C]

Position-based diff:
  Position 0: A --> A (no change)
  Position 1: B --> C (update B to C -- wrong! B was removed, not changed)
  Position 2: C --> (removed)
```

This produces incorrect results. React would update B's content to match C's, then remove the last element. The DOM operations are wrong, and any state held by the B component is lost.

The `key` prop fixes this:

```jsx
// With keys, React matches by identity, not position
{items.map(item => (
  <ListItem key={item.id} data={item} />
))}
```

```
Old:  [A(key=1), B(key=2), C(key=3)]
New:  [A(key=1), C(key=3)]

Key-based diff:
  key=1: A --> A (no change)
  key=2: B --> (removed)
  key=3: C --> C (no change)
```

Now React correctly identifies that B was removed and C stayed. The DOM operation is a single removal, and C's component state is preserved.

**This is why you should never use array index as a key for dynamic lists.** If items can be reordered, inserted, or deleted, the index changes and React can't correctly identify which items moved:

```
Old:  [A(key=0), B(key=1), C(key=2)]

// Insert X at beginning
New:  [X(key=0), A(key=1), B(key=2), C(key=3)]

// React thinks: key=0 changed from A to X, key=1 changed from B to A, etc.
// It updates EVERY element instead of just inserting one.
```

Use a stable, unique identifier (database ID, UUID) as the key.

### 4.6 React's Diffing Assumptions

React's diffing algorithm is O(n) — linear time — because it makes two simplifying assumptions:

1. **Elements of different types produce different trees.** If a `<div>` becomes a `<span>`, React doesn't try to reuse anything — it destroys the entire subtree and rebuilds it. This is correct 99.99% of the time and avoids an expensive tree-matching algorithm.

2. **The `key` prop identifies stable elements across renders.** For lists, keys tell React which items are the same, which are new, and which were removed. Without keys, React falls back to position-based comparison.

These assumptions reduce the diffing problem from O(n^3) (the theoretical minimum for general tree edit distance) to O(n), making it practical for real-time UI updates.

---

## 5. UI COMPOSITION

### 5.1 Components as Functions

A component is a function that accepts data (props) and returns a virtual DOM description (JSX):

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// This is equivalent to:
function Greeting({ name }) {
  return createElement("h1", null, `Hello, ${name}!`);
}
```

The function receives data, returns a description of UI. It doesn't touch the DOM directly. React calls the function, gets the description, and handles all DOM operations.

This is a **pure function** in the mathematical sense: same input, same output. Given `{ name: "Alice" }`, it always returns the same virtual DOM. No side effects, no global state mutation, no DOM manipulation.

### 5.2 Composition over Configuration

Components compose by nesting:

```jsx
function App() {
  return (
    <Layout>
      <Header>
        <Logo />
        <Navigation />
      </Header>
      <Main>
        <TodoList />
        <Sidebar />
      </Main>
      <Footer />
    </Layout>
  );
}
```

Each component is a self-contained unit. `Header` doesn't know about `TodoList`. `Footer` doesn't know about `Sidebar`. They compose together to form the full UI, but they're independently authored, tested, and maintained.

This is composition — building complex things from simple things. It's more flexible than inheritance because you can combine components in any arrangement, not just in a fixed class hierarchy.

### 5.3 Lists with map()

Arrays of data become arrays of components with `map()`:

```jsx
function TodoList({ todos }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} />
      ))}
    </ul>
  );
}
```

This is a higher-order function pattern from Chapter 0: `map` takes a function that transforms data into UI descriptions. Each call to the function creates a new virtual DOM node. The array of virtual DOM nodes becomes the children of the `<ul>`.

### 5.4 The children Prop

Components can accept other components as children:

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="card-body">
        {children}
      </div>
    </div>
  );
}

// Usage:
<Card title="User Profile">
  <Avatar url={user.avatar} />
  <UserInfo name={user.name} email={user.email} />
</Card>
```

`children` is just a prop — the JSX elements nested inside the component tag. `Card` doesn't know what its children are. It just renders them inside its layout. This enables generic container components (cards, modals, layouts) that work with any content.

### 5.5 Render Props and Function-as-Child

Components can accept functions as children or props, enabling powerful composition patterns:

```jsx
function MouseTracker({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  useEffect(() => {
    const handler = (e) => setPosition({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", handler);
    return () => window.removeEventListener("mousemove", handler);
  }, []);
  
  return children(position);
}

// Usage:
<MouseTracker>
  {({ x, y }) => (
    <div>Mouse is at ({x}, {y})</div>
  )}
</MouseTracker>
```

The child is a function that receives data and returns JSX. `MouseTracker` handles the logic (tracking the mouse), and the consumer decides how to render it. This separates behavior from presentation — a powerful composition pattern.

(In modern React, custom hooks have largely replaced render props for sharing stateful logic, but the pattern still appears in libraries.)

---

## 6. HOOKS AS CLOSURES

This section bridges Chapter 0 (closures) with React. If you understand closures, you understand hooks. They are the same mechanism.

### 6.1 useState Is a Closure

Let's build a simplified `useState` to see how it works:

```javascript
// React's internal state (simplified)
let hooks = [];     // array of hook values
let hookIndex = 0;  // current hook index

function useState(initialValue) {
  const currentIndex = hookIndex; // capture the index in a closure
  
  if (hooks[currentIndex] === undefined) {
    hooks[currentIndex] = initialValue; // first render: initialize
  }
  
  const setState = (newValue) => {
    hooks[currentIndex] = newValue; // closure over currentIndex
    reRender(); // trigger a re-render
  };
  
  hookIndex++;
  return [hooks[currentIndex], setState];
}

function reRender() {
  hookIndex = 0; // reset for the new render
  // Re-call the component function...
}
```

**The closure:** `setState` closes over `currentIndex`. When you call `setState`, it knows which position in the `hooks` array to update — because `currentIndex` was captured in the closure when `useState` was called.

**Why hooks must be called in the same order:** React uses the array index to associate each `useState` call with its stored value. If you call hooks conditionally, the indices shift and React associates the wrong state with the wrong hook.

```jsx
// BAD -- conditional hook
function Component({ showName }) {
  if (showName) {
    const [name, setName] = useState("Alice"); // index 0... sometimes
  }
  const [count, setCount] = useState(0); // index 1... or 0?
}
```

If `showName` is `true` on the first render, `name` is at index 0 and `count` is at index 1. If `showName` is `false` on the next render, `count` is at index 0 — but React thinks index 0 is `name`'s state. The values are scrambled.

**This is not a limitation of the API design — it's a consequence of using an array indexed by call order.** The "rules of hooks" (don't call conditionally, don't call in loops) directly follow from this implementation.

### 6.2 useEffect Is a Closure

```javascript
function useEffect(callback, deps) {
  const currentIndex = hookIndex;
  const prevDeps = hooks[currentIndex];
  
  // Check if dependencies changed
  const hasChanged = !prevDeps || deps.some((dep, i) => dep !== prevDeps[i]);
  
  if (hasChanged) {
    // Store new deps
    hooks[currentIndex] = deps;
    // Schedule the callback to run after render
    queueMicrotask(() => {
      const cleanup = callback(); // callback may return a cleanup function
      // Store cleanup for next run
    });
  }
  
  hookIndex++;
}
```

The callback you pass to `useEffect` is a closure. It closes over the component's variables at the time of that render. This is why stale closures happen — the effect's callback captures values from one render and doesn't see updates from later renders.

### 6.3 Custom Hooks Are Closure Composition

Custom hooks are just functions that call other hooks. The closures compose:

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  
  return [value, setValue];
}
```

`useLocalStorage` creates two closures (from `useState` and `useEffect`) and composes them. The `useEffect` closure captures `key` and `value` from the component's scope. When `value` changes, the effect re-runs and syncs to localStorage.

This is closures all the way down. There's no class hierarchy, no inheritance, no `this`. Just functions closing over variables and composing with other functions.

### 6.4 The Stale Closure Pattern (Revisited)

Now that you see hooks as closures, let's revisit the stale closure pattern with full understanding:

```jsx
function SearchResults() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    // This callback closes over `query` from THIS render
    fetchResults(query).then(data => {
      setResults(data);
    });
  }, [query]);
  
  // ...
}
```

Each render creates a new closure for the `useEffect` callback. That closure captures `query` from its render. When `query` changes, the old closure is discarded and a new one runs with the new `query`. The dependency array `[query]` tells React "re-run this effect when `query` changes" — which means "create a new closure capturing the new `query`."

But there's a subtle bug: if the user types fast, multiple fetches fire. An earlier fetch might resolve after a later one, overwriting correct results with stale ones. The fix:

```jsx
useEffect(() => {
  let cancelled = false; // local variable in this closure
  
  fetchResults(query).then(data => {
    if (!cancelled) { // check before updating
      setResults(data);
    }
  });
  
  return () => { cancelled = true; }; // cleanup sets cancelled to true
}, [query]);
```

The cleanup function closes over the same `cancelled` variable as the fetch callback. When the effect re-runs (because `query` changed), the cleanup from the previous run sets `cancelled = true`, and the previous fetch's `.then` callback sees `cancelled` as `true` and skips the state update.

This is closures solving a real asynchronous problem — race condition prevention through shared mutable state inside a closure scope.

---

## 7. THE DECLARATIVE PARADIGM

### 7.1 Imperative vs Declarative

**Imperative:** You tell the computer HOW to do something, step by step.

```javascript
// Imperative: create a todo list
const ul = document.createElement("ul");
document.body.appendChild(ul);

for (const todo of todos) {
  const li = document.createElement("li");
  li.textContent = todo.text;
  if (todo.done) {
    li.style.textDecoration = "line-through";
  }
  li.addEventListener("click", () => {
    todo.done = !todo.done;
    li.style.textDecoration = todo.done ? "line-through" : "none";
  });
  ul.appendChild(li);
}
```

You create each element, set each property, attach each event handler, manage each update manually. You're the one tracking the data-to-DOM relationship.

**Declarative:** You tell the computer WHAT the result should look like, given the current data.

```jsx
// Declarative: describe the todo list
function TodoList({ todos, onToggle }) {
  return (
    <ul>
      {todos.map(todo => (
        <li
          key={todo.id}
          style={{ textDecoration: todo.done ? "line-through" : "none" }}
          onClick={() => onToggle(todo.id)}
        >
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

You describe the desired UI for any given state. React figures out HOW to get from the current DOM to the desired DOM. You never create, update, or remove DOM elements directly.

### 7.2 Why Declarative Wins

1. **Less bugs.** You can't forget to update a DOM element because you never update DOM elements. You describe what should appear, and the reconciler handles the DOM.

2. **Easier to reason about.** Given state X, the UI is always Y. There's a direct mapping from data to display. You don't need to trace a sequence of mutations to understand what the UI looks like — just look at the current state.

3. **Testable.** A declarative component is a pure function: input (props/state) --> output (virtual DOM). You can test it by asserting that given specific inputs, it produces the expected output. No DOM mocking needed for the component logic itself.

4. **Optimizable.** Because the framework controls the DOM operations, it can optimize them — batching, reordering, skipping unnecessary updates. With imperative code, every optimization is manual.

5. **Portable.** A declarative UI description is platform-independent. The same component logic can target the web DOM, React Native's native views, a terminal, a PDF renderer, or anything else. The reconciler is the adapter between your declaration and the target platform.

### 7.3 The Reconciler as Adapter

This portability point is crucial for understanding React's architecture:

```
Your Component (declarative description)
        |
        v
   React Reconciler (diffing, scheduling)
        |
   +----+----+
   v         v
react-dom   react-native
(DOM ops)   (native view ops)
```

The reconciler is the same. Your components are the same. Only the **renderer** changes. This is why React Native exists — React separated the "what to render" (components) from the "how to render" (DOM vs native views). The declarative paradigm enables this separation.

### 7.4 The Mental Shift

The declarative paradigm requires a different way of thinking:

**Imperative thinking:** "When the user clicks, find the counter element and increment its text content."

**Declarative thinking:** "The counter element always displays the current count. When the user clicks, update the count. The UI will reflect the new count automatically."

In imperative code, you think about transitions: "how do I get from state A to state B?"
In declarative code, you think about snapshots: "what does the UI look like when the state is X?"

React components are snapshot functions. Each render is a snapshot. You describe the snapshot for each possible state, and React handles the transitions between snapshots. This is a simpler mental model once you internalize it, and it scales to arbitrarily complex UIs.

---

## SUMMARY: THE UI DEVELOPMENT MENTAL MODEL

1. **The DOM is a C++ data structure.** JavaScript accesses it through WebIDL bindings. Every DOM operation crosses a language boundary and may trigger the rendering pipeline. This is why DOM operations are expensive.

2. **The display problem** is keeping data and UI in sync. Every framework solves this differently: manual manipulation (jQuery), two-way binding (Angular 1), one-way data flow (React).

3. **The Virtual DOM** is a JavaScript representation of the DOM. React creates virtual DOM trees, diffs them, and applies minimal changes to the real DOM. This trades cheap JavaScript computation for expensive DOM operations.

4. **Reconciliation** uses keys and type comparisons to efficiently match old and new virtual DOM nodes. Keys are critical for list performance — always use stable, unique identifiers.

5. **Components are functions** that accept data and return UI descriptions. They compose by nesting. This is functional composition applied to UI.

6. **Hooks are closures.** useState is a closure over an array index. useEffect is a closure that runs after render. The rules of hooks follow from the array-based implementation. Stale closures are the most common hook bug, and they're a direct consequence of the closure mechanism.

7. **Declarative UI** describes what the UI should look like given current data. The framework handles the DOM mutations. This is simpler, more testable, and more portable than imperative DOM manipulation.

Every concept in this chapter — the DOM as C++, virtual DOM diffing, component composition, hooks as closures — will appear concretely in the React chapter that follows and in the React Native chapters in Part I. You now have the conceptual framework; the next chapter fills in React's specific API.

---

> **Next:** [Chapter 0c: React -- From First Principles](./00c-react-fundamentals.md) -- JSX, components, props, state, hooks, and the complete React mental model.
