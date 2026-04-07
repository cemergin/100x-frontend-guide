<!--
  CHAPTER: 0c
  TITLE: React — From First Principles
  PART: 0 — JavaScript & React Fundamentals
  PREREQS: Chapters 0, 0b
  KEY_TOPICS: JSX, components, props, state, hooks, useState, useEffect, useRef, useCallback, useMemo, useReducer, useContext, custom hooks, component lifecycle, React 19 features
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 0c: React — From First Principles

> **Part 0 — JavaScript & React Fundamentals** | Prerequisites: Chapters 0, 0b | Difficulty: Beginner to Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- JSX compiles to function calls (`React.createElement` or the automatic runtime); understanding this prevents mystifying errors
- `useState` is a closure over an array index -- React stores state in a hidden linked list on the Fiber node, not inside your component
- `useEffect` runs after paint, cleans up before the next run, and its dependency array controls when it fires; getting dependencies wrong causes infinite loops or stale closures
- Memoization (`React.memo`, `useMemo`, `useCallback`) only helps when you have measured a re-render problem; premature memoization adds complexity for no gain
- React 19 introduces `use()`, Server Components, Server Actions, `useOptimistic`, and `useFormStatus` -- a fundamental shift toward server-first rendering

</details>

You now understand JavaScript's execution model — closures, the event loop, promises, prototypes. You understand why UI development is hard — the DOM is C++, data binding is the central problem, and the virtual DOM is React's solution. This chapter connects everything: React's actual API, explained through the lens of the fundamentals you've already mastered.

Most React tutorials teach you to memorize the API. "useState returns a value and a setter." "useEffect runs after render." "useCallback memoizes a function." These statements are true and completely useless for building production applications. They tell you WHAT but not WHY. They don't explain why your effect runs in an infinite loop, why your memoization doesn't help performance, or why your context re-renders everything.

This chapter is different. Every hook is explained as the closure it actually is. Every rendering behavior is traced through the execution model. Every pattern is justified by the underlying mechanism. When you finish this chapter, you won't just know React's API — you'll understand it well enough to predict its behavior in any situation, debug any problem, and make architecture decisions with confidence.

We're also going to cover React 19's newest features — `use()`, Server Components, Server Actions, `useActionState`, `useFormStatus`, and `useOptimistic` — because these represent a fundamental shift in React's mental model that you need to understand if you're building anything new.

### In This Chapter
- JSX: what it compiles to and why that matters
- Components: functions that return UI descriptions
- Props and children: one-way data flow in practice
- useState: a closure over an array index
- useEffect: side effects, cleanup, and dependency arrays
- useRef: a mutable box that doesn't trigger re-renders
- Memoization: React.memo, useMemo, useCallback, and when they actually help
- useReducer: complex state logic with the reducer pattern
- Context: global-ish state without prop drilling
- Custom hooks: composing stateful logic
- React 19: use(), Server Components, Server Actions, and the new primitives

### Related Chapters
- [Ch 0: JavaScript — The Hard Parts] — closures and execution model (prerequisite)
- [Ch 0b: UI Development — The Hard Parts] — virtual DOM and data binding (prerequisite)
- [Ch 1: React Native Architecture & Internals] — how React runs in a native context
- [Ch 3: The Rendering Pipeline] — Fiber architecture, concurrent rendering, React Compiler
- [Ch 9: State Management at Scale] — Zustand, Jotai, and when to reach beyond hooks

---

## 1. JSX IS NOT HTML

This is the first misconception to eliminate. JSX looks like HTML:

```jsx
<div className="container">
  <h1>Hello, World</h1>
  <button onClick={handleClick}>Click me</button>
</div>
```

But it is NOT HTML. It is **syntactic sugar for function calls.** The JSX compiler (Babel, SWC, or the TypeScript compiler) transforms JSX into JavaScript:

### 1.1 The Classic Transform (Pre-React 17)

```jsx
// JSX:
<div className="container">
  <h1>Hello</h1>
</div>

// Compiles to:
React.createElement("div", { className: "container" },
  React.createElement("h1", null, "Hello")
);
```

Every JSX element becomes a `React.createElement()` call. The first argument is the element type (a string for HTML elements, a function for components). The second is the props object. The rest are children.

This is why you had to `import React from 'react'` in every file — the compiled code calls `React.createElement`, so `React` had to be in scope.

### 1.2 The New JSX Transform (React 17+)

React 17 introduced a new JSX transform that doesn't require importing React:

```jsx
// JSX:
<div className="container">
  <h1>Hello</h1>
</div>

// Compiles to:
import { jsx as _jsx } from "react/jsx-runtime";
_jsx("div", {
  className: "container",
  children: _jsx("h1", { children: "Hello" })
});
```

The compiler automatically inserts the import. The function signature is slightly different (`children` is a prop instead of a rest argument), but the concept is identical: JSX becomes function calls that return plain JavaScript objects.

### 1.3 What createElement Returns

Both transforms produce the same thing — a plain JavaScript object (a "React element"):

```javascript
{
  type: "div",
  props: {
    className: "container",
    children: {
      type: "h1",
      props: {
        children: "Hello"
      }
    }
  },
  key: null,
  ref: null,
}
```

This is the virtual DOM node from Chapter 0b. It's a JavaScript description of what should appear on screen. React takes this description, diffs it against the previous one, and applies minimal DOM changes.

### 1.4 Why This Understanding Matters

Once you understand that JSX is function calls returning objects, several things become clear:

**Why `className` instead of `class`:** JSX compiles to JavaScript. `class` is a reserved keyword in JavaScript. `className` is a property name that can be used in an object literal.

**Why `{}` for expressions:** JSX expressions `{variable}` are just JavaScript expressions interpolated into function arguments. They're evaluated at call time, like any other function argument.

**Why you can't use `if` in JSX:** `if` is a statement, not an expression. You can't put a statement inside a function argument. Use ternaries or `&&` instead:

```jsx
// This doesn't work — if is a statement:
// <div>{if (show) { <p>Hello</p> }}</div>

// These work — they're expressions:
<div>{show ? <p>Hello</p> : null}</div>
<div>{show && <p>Hello</p>}</div>
```

**Why fragments exist:** A function can only return one value. If you want to return multiple elements without a wrapper div, use a Fragment (`<>...</>`), which creates a special element type that the reconciler handles without creating a real DOM node.

**Why you can use map() for lists:** `map()` returns an array. An array of React elements is a valid children value. React renders each element in the array.

```jsx
// This is just:
// _jsx("ul", { children: items.map(item => _jsx("li", { children: item.name })) })
<ul>
  {items.map(item => (
    <li key={item.id}>{item.name}</li>
  ))}
</ul>
```

### 1.5 Component Elements vs DOM Elements

When the type is a string, it's a DOM element. When the type is a function, it's a component:

```jsx
// DOM element — type is a string
<div className="foo" />
// { type: "div", props: { className: "foo" } }

// Component — type is a function
<UserProfile name="Alice" />
// { type: UserProfile, props: { name: "Alice" } }
```

When React encounters a component element, it calls the function: `UserProfile({ name: "Alice" })`. The return value is another tree of React elements, which React continues to process recursively until the tree contains only DOM elements.

---

## 2. COMPONENTS

### 2.1 The Mental Model

A component is a function that accepts props and returns JSX. That's the complete definition.

```jsx
function Welcome({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

React calls this function when:
1. The component first appears in the tree (mount)
2. The component's props change
3. The component's state changes
4. The component's parent re-renders

Each call produces a new React element tree (a new "snapshot" of the UI). React diffs the new snapshot against the old one and applies changes to the DOM.

### 2.2 Why Function Components Won

Class components:

```jsx
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

Function components:

```jsx
function Welcome({ name }) {
  return <h1>Hello, {name}</h1>;
}
```

Function components won because:

1. **Simpler.** No `class`, no `constructor`, no `this`, no `render()` method. Just a function.

2. **Hooks enable composition.** Class components share logic through higher-order components (HOCs) and render props — patterns that create "wrapper hell" in the component tree. Hooks share logic through function composition — you just call a function.

3. **Closures replace `this`.** As Chapter 0 showed, closures are more predictable than `this` binding. No more `.bind(this)` in constructors. No more `this.state` vs `this.props` confusion.

4. **Easier to type-check.** A function component's type is just `(props: Props) => JSX.Element`. A class component's type involves `constructor`, `render`, lifecycle methods, and instance variables.

5. **React Compiler can optimize them.** The upcoming React Compiler (covered in section 7) can automatically memoize function components. Class components can't be optimized the same way because of mutable `this`.

### 2.3 The Re-Render Mental Model

A critical concept: **when a component re-renders, its entire function body runs again.**

```jsx
function Counter() {
  console.log("Counter rendering"); // logs on EVERY render
  const [count, setCount] = useState(0);
  
  const doubled = count * 2; // recalculated on every render
  
  function handleClick() {
    setCount(count + 1);
  }
  
  return (
    <div>
      <p>{count} (doubled: {doubled})</p>
      <button onClick={handleClick}>Increment</button>
    </div>
  );
}
```

Every time `count` changes:
1. React calls `Counter()` again
2. `useState` returns the new count value (from its internal storage)
3. `doubled` is recalculated
4. A new `handleClick` function is created (it closes over the new `count`)
5. A new React element tree is returned
6. React diffs the new tree against the old one
7. Only the changed text node is updated in the DOM

This "render on every change" model is what makes React predictable — the component function is a pure mapping from state to UI. But it's also why memoization exists: sometimes the recalculation or the child re-renders are expensive and unnecessary.

---

## 3. PROPS AND CHILDREN

### 3.1 One-Way Data Flow

Data in React flows in one direction: parent to child, through props.

```jsx
function App() {
  const [user, setUser] = useState({ name: "Alice", role: "admin" });
  
  return (
    <div>
      <Header userName={user.name} />
      <Dashboard user={user} onUpdate={setUser} />
    </div>
  );
}

function Header({ userName }) {
  return <h1>Welcome, {userName}</h1>;
}

function Dashboard({ user, onUpdate }) {
  return (
    <div>
      <UserCard name={user.name} role={user.role} />
      <EditButton onEdit={() => onUpdate({ ...user, name: "Bob" })} />
    </div>
  );
}
```

The data flow graph:

```
App (owns user state)
  |
  |-- userName --> Header (read-only)
  |
  |-- user, onUpdate --> Dashboard
       |
       |-- name, role --> UserCard (read-only)
       |
       |-- onEdit --> EditButton (calls onUpdate via closure)
```

Data flows down. When `EditButton` needs to change the data, it calls a callback (`onEdit`), which calls the parent's state setter (`onUpdate`), which updates `App`'s state, which triggers a re-render from `App` downward with the new data.

### 3.2 Props Are Read-Only

Components must not modify their props. This is not just a convention — it's fundamental to React's diffing model. If a child mutated its props, the parent's reference to the object would also change, making diffing unpredictable.

```jsx
// BAD — never mutate props
function BadComponent({ user }) {
  user.name = "modified"; // DO NOT DO THIS
}

// GOOD — treat props as read-only
function GoodComponent({ user }) {
  const displayName = user.name.toUpperCase(); // derive, don't mutate
}
```

### 3.3 The Children Prop

Everything between a component's opening and closing tags becomes the `children` prop:

```jsx
<Card>
  <h2>Title</h2>
  <p>Content</p>
</Card>

// The Card component receives:
// props.children = [<h2>Title</h2>, <p>Content</p>]
```

`children` can be anything: elements, strings, numbers, arrays, even functions (render props). It's just a regular prop with special JSX syntax.

### 3.4 Prop Drilling and Its Solutions

In a deep component tree, you sometimes need to pass props through many layers:

```jsx
function App() {
  const [theme, setTheme] = useState("dark");
  return <Layout theme={theme}><Main theme={theme}><Sidebar theme={theme}>...</Sidebar></Main></Layout>;
}
```

`theme` passes through `Layout` and `Main` just to reach `Sidebar`. These intermediate components don't use `theme` — they just forward it. This is **prop drilling**.

Solutions:
1. **Component composition** — restructure to reduce depth (often the best answer)
2. **Context** — provide values to deeply nested components without passing props (section 9)
3. **State management library** — Zustand, Jotai (Chapter 9)

---

## 4. STATE WITH useState

### 4.1 The Mechanism (Recap)

From Chapter 0b, you know that `useState` is a closure over an array index in React's internal fiber. Let's now explore the full API and its subtleties.

```jsx
const [count, setCount] = useState(0);
```

- `count`: the current state value for this render
- `setCount`: a function that schedules a state update and triggers a re-render
- `0`: the initial value (only used on first render)

### 4.2 State Updates Are Asynchronous (Batched)

`setCount` does not immediately change `count`. It schedules an update. React batches multiple state updates and processes them together:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count); // still 0!
  }
  // Result: count becomes 1, not 3
}
```

Why? Each `setCount(count + 1)` reads `count` from the closure. In this render, `count` is `0`. All three calls schedule `setCount(0 + 1)`. React batches them and applies `count = 1`.

### 4.3 Functional Updates

To update based on the previous state (not the closure value), use the functional form:

```jsx
function handleClick() {
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  // Result: count becomes 3
}
```

Each updater function receives the latest pending state. The first gets `0`, returns `1`. The second gets `1`, returns `2`. The third gets `2`, returns `3`.

**Rule of thumb:** Use functional updates whenever the new state depends on the previous state. This avoids stale closure bugs and works correctly with batching.

### 4.4 Lazy Initialization

If the initial state requires expensive computation, pass a function:

```jsx
// BAD — runs on every render
const [data, setData] = useState(expensiveComputation());

// GOOD — runs only on first render
const [data, setData] = useState(() => expensiveComputation());
```

React calls the initializer function only during the first render. On subsequent renders, it's ignored. This is a minor optimization but matters for things like reading from localStorage or parsing large datasets.

### 4.5 State and Object Identity

React uses `Object.is()` to determine if state changed. For primitives, this is straightforward. For objects and arrays, you must create new references:

```jsx
// BAD — mutating state directly
const [user, setUser] = useState({ name: "Alice", age: 30 });

function updateName() {
  user.name = "Bob"; // mutation!
  setUser(user); // Object.is(user, user) === true — no re-render!
}

// GOOD — create a new object
function updateName() {
  setUser({ ...user, name: "Bob" }); // new reference, React re-renders
}

// GOOD — for arrays
const [items, setItems] = useState([1, 2, 3]);
setItems([...items, 4]);           // add
setItems(items.filter(x => x !== 2)); // remove
setItems(items.map(x => x === 2 ? 20 : x)); // update
```

This immutability requirement comes directly from the diffing model: React needs to quickly detect changes, and reference comparison (`Object.is`) is O(1) while deep comparison is O(n).

### 4.6 When to Use Multiple useState vs One State Object

```jsx
// Multiple individual states
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [age, setAge] = useState(0);

// Single state object
const [form, setForm] = useState({ name: "", email: "", age: 0 });
```

**Use multiple `useState` when** values are independent (changing one doesn't affect another).

**Use a single state object when** values are related and often update together. But consider `useReducer` for complex state objects (section 8).

---

## 5. EFFECTS WITH useEffect

### 5.1 What Effects Are

Effects are side effects — anything that interacts with the outside world beyond returning JSX:
- Fetching data
- Setting up subscriptions (WebSocket, event listeners)
- Manually changing the DOM
- Setting timers
- Logging

```jsx
useEffect(() => {
  // This runs AFTER the component renders and the DOM is updated
  document.title = `Count: ${count}`;
}, [count]);
```

### 5.2 The Dependency Array

The dependency array controls WHEN the effect re-runs:

```jsx
// Runs after EVERY render
useEffect(() => { /* ... */ });

// Runs only on mount (first render)
useEffect(() => { /* ... */ }, []);

// Runs when `count` or `name` change
useEffect(() => { /* ... */ }, [count, name]);
```

React compares the current dependencies with the previous ones using `Object.is()`. If any dependency changed, the effect re-runs.

### 5.3 The Cleanup Function

The function returned from the effect is the cleanup. It runs:
- Before the effect re-runs (when dependencies change)
- When the component unmounts

```jsx
useEffect(() => {
  const connection = createWebSocket(roomId);
  connection.connect();
  
  return () => {
    connection.disconnect(); // cleanup: disconnect when roomId changes or unmount
  };
}, [roomId]);
```

**Why cleanup matters:** Without it, you'd accumulate connections. Each time `roomId` changes, a new connection opens but the old one never closes. Cleanup prevents resource leaks.

### 5.4 Common Pitfalls

**Pitfall 1: Missing dependencies**

```jsx
// BUG: effect uses `count` but doesn't list it as a dependency
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // stale closure! always logs the initial value
  }, 1000);
  return () => clearInterval(id);
}, []); // count is missing from deps
```

The ESLint rule `react-hooks/exhaustive-deps` catches this. Always include every value from the component scope that the effect reads.

**Pitfall 2: Object dependencies causing infinite loops**

```jsx
// BUG: infinite loop
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  const options = { method: "GET", headers: { "Content-Type": "application/json" } };
  
  useEffect(() => {
    fetch(`/api/users/${userId}`, options).then(r => r.json()).then(setUser);
  }, [userId, options]); // options is a NEW object every render!
}
```

`options` is created on every render, so it's always a new reference, and the effect runs every render. Fix: move `options` inside the effect, memoize it with `useMemo`, or extract it as a constant outside the component.

**Pitfall 3: Using effects for things that should be event handlers**

```jsx
// BAD: effect that should be an event handler
useEffect(() => {
  if (submitted) {
    sendAnalytics("form_submitted");
  }
}, [submitted]);

// GOOD: event handler
function handleSubmit() {
  setSubmitted(true);
  sendAnalytics("form_submitted"); // directly in the handler
}
```

Effects are for synchronization (keeping things in sync with props/state). One-time actions in response to user events belong in event handlers.

### 5.5 The useEffect Timeline

```
Render 1 (mount):
  1. Component function runs
  2. React commits changes to DOM
  3. Browser paints
  4. useEffect callback runs
  
Render 2 (count changed):
  1. Component function runs
  2. React commits changes to DOM
  3. Browser paints
  4. useEffect CLEANUP from render 1 runs
  5. useEffect callback for render 2 runs

Unmount:
  1. useEffect CLEANUP from last render runs
```

Effects run AFTER the browser paints. This means users see the UI update before the effect runs. For cases where you need to measure the DOM or prevent visual flicker, use `useLayoutEffect` instead (it runs synchronously after DOM mutation, before paint).

---

## 6. REFS WITH useRef

### 6.1 The Mental Model

`useRef` creates a mutable object that persists across renders:

```jsx
const countRef = useRef(0);
// countRef = { current: 0 }
```

Unlike state, mutating a ref does NOT trigger a re-render. The object persists for the entire lifetime of the component. This makes refs useful for:

1. **Accessing DOM elements**
2. **Storing mutable values that don't affect rendering** (timer IDs, previous values, instance variables)
3. **Holding references to third-party library instances**

### 6.2 DOM Refs

The most common use — getting a reference to a DOM element:

```jsx
function TextInput() {
  const inputRef = useRef(null);
  
  function handleClick() {
    inputRef.current.focus(); // directly access the DOM element
  }
  
  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>Focus Input</button>
    </>
  );
}
```

React sets `inputRef.current` to the DOM element after mounting and sets it to `null` on unmount.

### 6.3 Storing Mutable Values

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  const intervalRef = useRef(null);
  
  function start() {
    intervalRef.current = setInterval(() => {
      setCount(prev => prev + 1);
    }, 1000);
  }
  
  function stop() {
    clearInterval(intervalRef.current);
  }
  
  useEffect(() => {
    return () => clearInterval(intervalRef.current); // cleanup on unmount
  }, []);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

The interval ID is stored in a ref because:
- It needs to persist across renders (can't be a local variable)
- Changing it shouldn't trigger a re-render (shouldn't be state)
- It's a side-effect implementation detail, not data that affects the UI

### 6.4 Previous Value Pattern

```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  
  return <p>Now: {count}, Before: {prevCount}</p>;
}
```

This works because:
1. During render, `ref.current` still holds the value from the previous effect
2. After render, the effect updates `ref.current` to the new value
3. On the next render, `ref.current` returns what was set in the previous effect

### 6.5 Ref vs State — When to Use Which

| Use State | Use Ref |
|-----------|---------|
| Value affects what's displayed | Value is an implementation detail |
| Changing it should re-render | Changing it should NOT re-render |
| The user can see it | The user can't see it |
| Examples: count, name, isOpen | Examples: timer ID, DOM element, previous value |

**The rule:** If the value affects the UI (what the user sees), it's state. If it doesn't, it's a ref.

---

## 7. MEMOIZATION

### 7.1 The Problem

When a component re-renders, ALL its children re-render too, even if their props haven't changed:

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <ExpensiveChild /> {/* re-renders on every count change! */}
    </div>
  );
}

function ExpensiveChild() {
  // Expensive rendering logic...
  return <div>I'm expensive to render</div>;
}
```

`ExpensiveChild` receives no props from `count`. It produces the same output every time. But it re-renders every time `Parent` re-renders because that's React's default: re-render the entire subtree.

### 7.2 React.memo

`React.memo` wraps a component and skips re-rendering if props haven't changed:

```jsx
const ExpensiveChild = React.memo(function ExpensiveChild() {
  // Only re-renders if props actually change
  return <div>I'm expensive to render</div>;
});
```

React.memo does a shallow comparison of props. If all props are the same (by reference for objects, by value for primitives), the component skips rendering and reuses the previous output.

### 7.3 useMemo

Memoizes a computed value:

```jsx
function FilteredList({ items, filter }) {
  // Only recomputes when items or filter change
  const filtered = useMemo(() => {
    return items.filter(item => item.category === filter);
  }, [items, filter]);
  
  return (
    <ul>
      {filtered.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

Without `useMemo`, `items.filter(...)` runs on every render, even if `items` and `filter` haven't changed. With `useMemo`, it only recomputes when the dependencies change.

### 7.4 useCallback

Memoizes a function reference:

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  
  // Without useCallback: new function on every render
  // With useCallback: same function reference if `count` hasn't changed
  const handleClick = useCallback(() => {
    console.log(count);
  }, [count]);
  
  return <MemoizedChild onClick={handleClick} />;
}
```

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. It returns the same function reference across renders as long as the dependencies don't change.

**Why this matters:** If you pass a callback to a `React.memo` child, a new function reference on every render defeats the memo:

```jsx
// BAD: memo is useless because onClick is a new function every render
const Child = React.memo(function Child({ onClick }) { /* ... */ });

function Parent() {
  return <Child onClick={() => console.log("clicked")} />; // new fn each render
}

// GOOD: stable function reference
function Parent() {
  const handleClick = useCallback(() => console.log("clicked"), []);
  return <Child onClick={handleClick} />;
}
```

### 7.5 When Memoization Hurts

Memoization has a cost:
- `useMemo` and `useCallback` allocate memory for the cached value and dependency array
- They do a shallow comparison of dependencies on every render
- They add complexity to the code

**Don't memoize when:**
- The computation is cheap (simple arithmetic, string concatenation)
- The component renders infrequently
- The memo overhead exceeds the render cost (micro-optimizations on trivial components)

**Do memoize when:**
- Computation is genuinely expensive (filtering/sorting large arrays, complex calculations)
- The result is passed to a `React.memo` child that would otherwise re-render
- The value is used as a dependency in another hook (preventing cascading re-runs)

### 7.6 React Compiler (React Forget)

React Compiler is an automatic memoization tool that makes most manual `useMemo` and `useCallback` unnecessary. It analyzes your components at build time and automatically inserts memoization where beneficial.

```jsx
// What you write:
function TodoList({ todos, filter }) {
  const filtered = todos.filter(t => t.category === filter);
  return <ul>{filtered.map(t => <TodoItem key={t.id} todo={t} />)}</ul>;
}

// What React Compiler produces (conceptually):
function TodoList({ todos, filter }) {
  const filtered = useMemo(() => todos.filter(t => t.category === filter), [todos, filter]);
  const children = useMemo(() => filtered.map(t => <TodoItem key={t.id} todo={t} />), [filtered]);
  return <ul>{children}</ul>;
}
```

React Compiler works with Expo SDK 52+ and Next.js. If you're starting a new project, lean on the compiler instead of manual memoization. But understand the concepts — the compiler makes the same decisions you would, just automatically.

---

## 8. REDUCERS WITH useReducer

### 8.1 When useState Isn't Enough

When state logic involves multiple related values or complex transitions, `useState` becomes unwieldy:

```jsx
// Messy with multiple useState
function ShoppingCart() {
  const [items, setItems] = useState([]);
  const [total, setTotal] = useState(0);
  const [discount, setDiscount] = useState(0);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  
  function addItem(item) {
    setItems([...items, item]);
    setTotal(total + item.price);
    setIsLoading(false);
    setError(null);
    // Easy to forget one of these updates...
  }
}
```

### 8.2 The Reducer Pattern

A reducer is a function that takes the current state and an action, and returns the new state:

```javascript
function reducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM":
      return {
        ...state,
        items: [...state.items, action.item],
        total: state.total + action.item.price,
        error: null,
      };
    case "REMOVE_ITEM":
      const item = state.items.find(i => i.id === action.id);
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.id),
        total: state.total - item.price,
      };
    case "APPLY_DISCOUNT":
      return {
        ...state,
        discount: action.percentage,
        total: state.total * (1 - action.percentage / 100),
      };
    case "SET_ERROR":
      return { ...state, error: action.message, isLoading: false };
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}
```

### 8.3 Using useReducer

```jsx
const initialState = {
  items: [],
  total: 0,
  discount: 0,
  isLoading: false,
  error: null,
};

function ShoppingCart() {
  const [state, dispatch] = useReducer(reducer, initialState);
  
  function addItem(item) {
    dispatch({ type: "ADD_ITEM", item });
  }
  
  function removeItem(id) {
    dispatch({ type: "REMOVE_ITEM", id });
  }
  
  return (
    <div>
      <p>Total: ${state.total}</p>
      {state.items.map(item => (
        <CartItem
          key={item.id}
          item={item}
          onRemove={() => removeItem(item.id)}
        />
      ))}
    </div>
  );
}
```

### 8.4 Why Reducers Are Better for Complex State

1. **All state transitions in one place.** The reducer function contains every possible state change. You can read it and understand every way the state can change.

2. **Impossible invalid states.** The reducer controls what combinations of state are valid. You can't accidentally set `isLoading: true` and `error: "something"` at the same time if the reducer doesn't allow it.

3. **Testable.** A reducer is a pure function: `(state, action) => newState`. You can test every transition independently without rendering a component.

4. **Type-safe with discriminated unions (TypeScript):**

```typescript
type CartAction =
  | { type: "ADD_ITEM"; item: CartItem }
  | { type: "REMOVE_ITEM"; id: string }
  | { type: "APPLY_DISCOUNT"; percentage: number }
  | { type: "SET_ERROR"; message: string };

function reducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM":
      // TypeScript knows action.item exists here
      return { ...state, items: [...state.items, action.item] };
    // ...
  }
}
```

TypeScript's discriminated union narrowing ensures you can only access properties that exist for each action type. Dispatch calls with wrong properties are compile-time errors.

### 8.5 useReducer vs useState

| Use useState | Use useReducer |
|-------------|----------------|
| 1-3 independent values | Related values that update together |
| Simple updates (set a value) | Complex transitions (add item + update total + clear error) |
| Local component state | State shared via context |
| No complex validation | State transitions need validation |

---

## 9. CONTEXT

### 9.1 What Context Solves

Context provides a way to pass data to deeply nested components without prop drilling:

```jsx
const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("dark");
  
  return (
    <ThemeContext.Provider value={theme}>
      <Layout>
        <Main>
          <Sidebar>
            <ThemeToggle onToggle={() => setTheme(t => t === "dark" ? "light" : "dark")} />
          </Sidebar>
        </Main>
      </Layout>
    </ThemeContext.Provider>
  );
}

function ThemeToggle({ onToggle }) {
  const theme = useContext(ThemeContext); // reads theme without prop drilling
  return <button onClick={onToggle}>Current: {theme}</button>;
}
```

`Layout`, `Main`, and `Sidebar` don't need to know about `theme`. Only `ThemeToggle` reads it, and it gets it directly from context.

### 9.2 How Context Works

Context uses a provider-consumer pattern:

1. `createContext(defaultValue)` — creates a context object
2. `<Context.Provider value={...}>` — provides a value to all descendants
3. `useContext(Context)` — reads the nearest ancestor provider's value

When the provider's `value` changes, ALL consumers re-render. This is context's biggest performance gotcha.

### 9.3 The Re-Render Problem

```jsx
function App() {
  const [user, setUser] = useState({ name: "Alice", theme: "dark" });
  
  return (
    <AppContext.Provider value={user}>
      <Header />      {/* re-renders when user changes */}
      <ExpensiveTree /> {/* re-renders when user changes, even if it only uses theme */}
      <Footer />      {/* re-renders when user changes */}
    </AppContext.Provider>
  );
}
```

When `user` changes (even just `user.name`), EVERY component that calls `useContext(AppContext)` re-renders — even if it only uses `user.theme` and `theme` didn't change.

### 9.4 Solutions to the Context Re-Render Problem

**Split contexts:**

```jsx
const UserContext = createContext(null);
const ThemeContext = createContext("light");

function App() {
  const [user, setUser] = useState({ name: "Alice" });
  const [theme, setTheme] = useState("dark");
  
  return (
    <UserContext.Provider value={user}>
      <ThemeContext.Provider value={theme}>
        {/* Components that only use theme don't re-render when user changes */}
        <Children />
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}
```

**Memoize the value:**

```jsx
function App() {
  const [user, setUser] = useState({ name: "Alice" });
  
  const contextValue = useMemo(() => ({
    user,
    updateUser: setUser,
  }), [user]);
  
  return (
    <AppContext.Provider value={contextValue}>
      <Children />
    </AppContext.Provider>
  );
}
```

Without `useMemo`, the object literal `{ user, updateUser: setUser }` creates a new object on every render, causing all consumers to re-render even if `user` hasn't changed.

### 9.5 When to Use Context vs State Management Libraries

**Use Context for:**
- Theme, locale, auth status — values that change infrequently
- Dependency injection (passing services or configuration)
- Small apps where the re-render cost is negligible

**Use Zustand, Jotai, or another state library for:**
- Frequently changing values (typing in a search box, real-time data)
- Complex state with many consumers
- State that needs to be accessed outside React (in utility functions, middleware)
- When you need selector-based subscriptions (only re-render when specific parts of state change)

Chapter 9 covers this topic in depth. The short version: context is a great tool for the right job, but it's not a general-purpose state management solution.

---

## 10. CUSTOM HOOKS

### 10.1 What Custom Hooks Are

A custom hook is a function that starts with `use` and calls other hooks. That's the only requirement.

```jsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });
  
  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);
  
  return size;
}
```

Usage:

```jsx
function ResponsiveLayout() {
  const { width } = useWindowSize();
  return width > 768 ? <DesktopLayout /> : <MobileLayout />;
}
```

### 10.2 Why Custom Hooks Matter

Custom hooks extract stateful logic from components. The component becomes simpler, and the logic becomes reusable:

```jsx
// Before: stateful logic mixed with rendering
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetchUser(userId)
      .then(data => { if (!cancelled) { setUser(data); setLoading(false); } })
      .catch(err => { if (!cancelled) { setError(err); setLoading(false); } });
    return () => { cancelled = true; };
  }, [userId]);
  
  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{user.name}</div>;
}

// After: stateful logic in a custom hook
function useUser(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetchUser(userId)
      .then(data => { if (!cancelled) { setUser(data); setLoading(false); } })
      .catch(err => { if (!cancelled) { setError(err); setLoading(false); } });
    return () => { cancelled = true; };
  }, [userId]);
  
  return { user, loading, error };
}

function UserProfile({ userId }) {
  const { user, loading, error } = useUser(userId);
  
  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{user.name}</div>;
}
```

The component now has zero data-fetching logic. `useUser` can be tested independently and reused in any component that needs user data.

(In practice, you'd use TanStack Query instead of writing this yourself — see Chapter 10. But the pattern is the same.)

### 10.3 Practical Custom Hooks

**useDebounce:**

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage:
function SearchBox() {
  const [input, setInput] = useState("");
  const debouncedInput = useDebounce(input, 300);
  
  useEffect(() => {
    if (debouncedInput) {
      search(debouncedInput);
    }
  }, [debouncedInput]);
  
  return <input value={input} onChange={e => setInput(e.target.value)} />;
}
```

**useLocalStorage:**

```jsx
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });
  
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);
  
  return [value, setValue];
}

// Usage:
const [theme, setTheme] = useLocalStorage("theme", "light");
```

**useMediaQuery:**

```jsx
function useMediaQuery(query) {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches
  );
  
  useEffect(() => {
    const mql = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);
    mql.addEventListener("change", handler);
    return () => mql.removeEventListener("change", handler);
  }, [query]);
  
  return matches;
}

// Usage:
const isDarkMode = useMediaQuery("(prefers-color-scheme: dark)");
const isMobile = useMediaQuery("(max-width: 768px)");
```

### 10.4 Custom Hook Composition

Hooks compose — custom hooks can call other custom hooks:

```jsx
function useResponsiveTheme() {
  const prefersDark = useMediaQuery("(prefers-color-scheme: dark)");
  const [userOverride, setUserOverride] = useLocalStorage("theme-override", null);
  
  const theme = userOverride ?? (prefersDark ? "dark" : "light");
  
  return { theme, setTheme: setUserOverride };
}
```

This hook composes `useMediaQuery` and `useLocalStorage` to create a theme system that respects the user's OS preference but allows manual override, persisted across sessions. Each hook maintains its own state independently, and the composition function combines them.

---

## 11. REACT 19 FEATURES

React 19 introduced several features that change how you think about data fetching, form handling, and server-client boundaries. These aren't incremental improvements — they represent a shift in React's mental model.

### 11.1 The use() Hook

`use()` is a new hook that reads the value of a resource (a Promise or a Context) during render:

```jsx
function UserProfile({ userPromise }) {
  const user = use(userPromise); // suspends until promise resolves
  return <div>{user.name}</div>;
}
```

**How it differs from useEffect for data fetching:**

With `useEffect`, you render first (showing a loading state), then fetch, then re-render with data:
```jsx
// Old pattern
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => { fetchUser(userId).then(setUser); }, [userId]);
  if (!user) return <Loading />;
  return <div>{user.name}</div>;
}
```

With `use()`, the component suspends — it pauses rendering until the data is ready:
```jsx
// New pattern
function UserProfile({ userPromise }) {
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// Parent decides what to show while loading:
<Suspense fallback={<Loading />}>
  <UserProfile userPromise={fetchUser(userId)} />
</Suspense>
```

The key difference: the loading state is handled by `Suspense` boundaries, not by the component itself. This separates the data requirement from the loading UI.

**use() with Context:**

```jsx
function ThemeButton() {
  // Can be called conditionally — unlike useContext!
  if (someCondition) {
    const theme = use(ThemeContext);
    return <button className={theme}>Click</button>;
  }
  return <button>Default</button>;
}
```

Unlike `useContext`, `use()` can be called inside conditionals and loops. This is because `use()` is designed to work with React's concurrent rendering model — it doesn't rely on the hook call order.

### 11.2 Server Components

Server Components are components that run on the server and send rendered HTML (and a serialized component tree) to the client. They never re-render on the client. They cannot have state or effects.

```jsx
// This component runs on the server only
async function ProductList() {
  const products = await db.query("SELECT * FROM products"); // direct DB access
  
  return (
    <ul>
      {products.map(p => (
        <li key={p.id}>
          {p.name} - ${p.price}
          <AddToCartButton productId={p.id} /> {/* client component */}
        </li>
      ))}
    </ul>
  );
}
```

**Why this matters:**
- **Zero client-side JavaScript** for server components. The `ProductList` component's code is never sent to the browser. This reduces bundle size.
- **Direct backend access.** Server Components can query databases, read files, call internal APIs — without an API route in between.
- **Async components.** Server Components can be `async` functions that `await` data. No `useEffect`, no loading states (at this level).

**The mental model shift:** Before Server Components, all React code ran on the client. The server sent HTML for the initial page, then the client hydrated and took over. With Server Components, some components permanently live on the server. They render once, send their output, and never execute on the client.

### 11.3 Server Actions

Server Actions are functions that run on the server, called from the client:

```jsx
// actions.js
"use server";

export async function addToCart(productId) {
  const session = await getSession();
  await db.cartItems.create({
    userId: session.userId,
    productId,
  });
  revalidatePath("/cart");
}
```

```jsx
// Client component
"use client";

import { addToCart } from "./actions";

function AddToCartButton({ productId }) {
  return (
    <form action={() => addToCart(productId)}>
      <button type="submit">Add to Cart</button>
    </form>
  );
}
```

When the user clicks the button, the function `addToCart` executes on the server. React handles the serialization, the network call, and the revalidation. You don't write fetch calls or API routes — the function call IS the API.

**The `"use server"` directive** marks a function as a server action. The bundler replaces the function body with a network call on the client side. On the server side, it creates an endpoint that executes the function.

### 11.4 useActionState

`useActionState` (previously `useFormState`) manages the state of a server action:

```jsx
"use client";

import { createUser } from "./actions";

function SignupForm() {
  const [state, formAction, isPending] = useActionState(createUser, {
    errors: null,
    success: false,
  });
  
  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      {state.errors && <p className="error">{state.errors.email}</p>}
      <button type="submit" disabled={isPending}>
        {isPending ? "Creating..." : "Sign Up"}
      </button>
      {state.success && <p>Account created!</p>}
    </form>
  );
}
```

```jsx
// actions.js
"use server";

export async function createUser(prevState, formData) {
  const email = formData.get("email");
  const password = formData.get("password");
  
  if (!email.includes("@")) {
    return { errors: { email: "Invalid email" }, success: false };
  }
  
  await db.users.create({ email, password: hash(password) });
  return { errors: null, success: true };
}
```

`useActionState` gives you:
- `state`: the return value of the last action call
- `formAction`: a wrapped version of the action to use as the form's `action` prop
- `isPending`: whether the action is currently running

### 11.5 useFormStatus

Reads the status of the parent form — useful for submit buttons:

```jsx
"use client";

import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}
```

`useFormStatus` must be called from a component rendered inside a `<form>`. It reads the form's submission state without needing to pass `isPending` through props.

### 11.6 useOptimistic

Shows optimistic UI while an async action is pending:

```jsx
"use client";

function TodoList({ todos, addTodoAction }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (currentTodos, newTodo) => [...currentTodos, { ...newTodo, pending: true }]
  );
  
  async function handleSubmit(formData) {
    const newTodo = { id: crypto.randomUUID(), text: formData.get("text") };
    addOptimisticTodo(newTodo); // immediately show the new todo
    await addTodoAction(newTodo); // server action — may take time
    // When the action completes, `todos` prop updates and optimistic state is replaced
  }
  
  return (
    <div>
      <form action={handleSubmit}>
        <input name="text" />
        <button type="submit">Add</button>
      </form>
      <ul>
        {optimisticTodos.map(todo => (
          <li key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
            {todo.text}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`useOptimistic` takes the real state and an update function. When you call `addOptimisticTodo`, it immediately updates the displayed list (the optimistic state). When the async action completes and the `todos` prop updates with the real data, the optimistic state is automatically replaced.

### 11.7 The New Mental Model

React 19's features introduce a clear boundary between server and client:

```
Server                          Client
+-----------------------------+  +-----------------------------+
| Server Components           |  | Client Components           |
| - Run once on server        |  | - Run in browser            |
| - Can be async              |  | - Can have state, effects   |
| - Direct DB/file access     |  | - Handle user interactions  |
| - Zero client JS            |  | - Use hooks                 |
|                             |  |                             |
| Server Actions              |  | useActionState              |
| - Mutate server state       |  | useFormStatus               |
| - Called from client forms   |  | useOptimistic               |
| - Automatic serialization   |  | use() for promises/context  |
+-----------------------------+  +-----------------------------+
```

The previous model: everything runs on the client, data comes from API routes, loading states are managed manually.

The new model: read operations happen on the server (Server Components), write operations happen on the server (Server Actions), and the client handles interactions and optimistic UI. React manages the boundary.

This is a significant simplification for many apps. Instead of `useEffect` + fetch + loading state + error state + cache management, you get `async function` on the server + Suspense boundaries for loading. Not every app benefits (purely client-side apps, mobile apps via React Native), but for web apps with server-rendered pages, this is the direction React is moving.

---

## SUMMARY: THE REACT MENTAL MODEL

1. **JSX is function calls.** `<div>` becomes `jsx("div", {...})`. Understanding this explains className, expressions, fragments, map, and every other JSX question.

2. **Components are functions** that take props and return React elements. Every render is a new function call. Every local variable, every closure is brand new.

3. **Props flow down, events flow up.** This one-way data flow is React's core architecture decision. It makes apps predictable and debuggable.

4. **useState is a closure** over an array index. Functional updates avoid stale closures. Immutable updates are required for change detection.

5. **useEffect is for synchronization,** not for responding to events. It runs after paint. Cleanup runs before the next effect and on unmount. Dependency arrays control re-runs.

6. **useRef is a mutable box** that persists across renders without triggering re-renders. Use for DOM access, timer IDs, and values that don't affect the UI.

7. **Memoization (React.memo, useMemo, useCallback)** prevents unnecessary re-renders and recomputations. Use when the cost of re-rendering/recomputing exceeds the cost of memoizing. React Compiler automates this.

8. **useReducer** centralizes complex state transitions. Reducers are pure functions, testable independently, and type-safe with discriminated unions.

9. **Context** provides data to deeply nested components. Split contexts and memoize values to prevent unnecessary re-renders. Use state management libraries for frequently changing data.

10. **Custom hooks** extract and compose stateful logic. They're the composition mechanism that replaced HOCs and render props.

11. **React 19** introduces Server Components (read on server), Server Actions (write on server), use() (suspending data read), and form primitives (useActionState, useFormStatus, useOptimistic). These move data operations to the server and simplify client-side code.

You now have the complete foundation: JavaScript fundamentals (Chapter 0), UI development concepts (Chapter 0b), and React's API understood through those fundamentals (this chapter). Everything that follows in this guide — React Native internals, state management, performance optimization, architecture at scale — builds on this foundation.

---

> **Next:** [Chapter 1: React Native Architecture & Internals](../part-1-foundations/01-react-native-internals.md) — How React runs in a native context: JSI, Fabric, TurboModules, Hermes, and the threading model.
