<!--
  CHAPTER: 0
  TITLE: JavaScript — The Hard Parts
  PART: 0 — JavaScript & React Fundamentals
  PREREQS: None
  KEY_TOPICS: execution context, call stack, event loop, callback queue, microtask queue, closures, higher-order functions, prototypes, this keyword, async/await, promises, generators, iterators
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 0: JavaScript — The Hard Parts

> **Part 0 — JavaScript & React Fundamentals** | Prerequisites: None | Difficulty: Beginner to Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- JavaScript runs on a single thread with one call stack; understanding execution contexts, closures, and the event loop is the foundation of every React performance bug you will ever debug
- Closures are not an interview trick -- they are the mechanism behind `useState`, `useEffect`, and every custom hook you will write
- The event loop processes microtasks (Promises) before macrotasks (setTimeout); getting this wrong causes stale state and race conditions
- Async/await is syntactic sugar over generators and promises; understanding the desugaring lets you predict execution order in complex flows
- Prototypes and `this` still matter in React Native internals, native module bindings, and library code you will need to read

</details>

Here's an uncomfortable truth that most frontend engineers never confront: **you don't actually know how JavaScript works.**

You know how to *use* JavaScript. You know how to write React components, handle events, fetch data, and push code to production. But if I asked you to trace exactly what happens when the JavaScript engine encounters `setTimeout(() => console.log('hello'), 0)` — the full picture, including which C++ subsystem the browser delegates to, which queue the callback enters, and why it prints *after* a synchronous `console.log('world')` that appears later in the code — most engineers stumble.

This isn't a trivia question. This is the foundation of every performance bug you'll ever debug, every race condition you'll ever encounter, every stale closure in a React hook that costs your team a week of investigation. The engineers who understand JavaScript at the engine level don't just write better code — they *see* problems before they happen. They read a useEffect and immediately know it has a stale closure. They look at a Promise chain and know the exact execution order. They understand why `async/await` isn't magic — it's generators and microtasks under the hood.

Will Sentance calls these "the hard parts" because they're the concepts that separate engineers who *use* JavaScript from engineers who *understand* JavaScript. Every tutorial teaches you the syntax. Almost none of them teach you the execution model. This chapter fixes that.

We're going to trace JavaScript execution the way the engine does it — step by step, line by line, building execution contexts, pushing and popping the call stack, handing off work to the browser, and watching callbacks flow through queues. By the end of this chapter, you won't just know what closures are — you'll understand *why* `useState` works. You won't just know that promises are asynchronous — you'll understand the exact mechanism by which a `.then()` callback enters the microtask queue and why it runs before a `setTimeout` callback.

This is the chapter that makes everything else in this guide click.

### In This Chapter
- The JavaScript execution model: execution contexts, thread of execution, memory
- The call stack: how function invocations create and destroy execution contexts
- Closures: the most powerful feature in JavaScript, and why they power React hooks
- Asynchronous JavaScript: the full picture (call stack, Web APIs, callback queue, microtask queue, event loop)
- Promises under the hood: what a Promise object actually looks like internally
- Async/await: syntactic sugar over generators + promises
- Iterators and generators: functions that can pause
- The event loop in depth: priority, ordering, and complex interleaving
- Prototypes and `this`: JavaScript's object system and its quirks

### Related Chapters
- [Ch 0b: UI Development — The Hard Parts] — applying these JS fundamentals to DOM manipulation and UI frameworks
- [Ch 0c: React — From First Principles] — how React leverages closures, the event loop, and composition
- [Ch 1: React Native Architecture & Internals] — how the JS engine (Hermes) runs your code in a native context
- [Ch 3: The Rendering Pipeline] — React reconciliation, Fiber, and concurrent features

---

## 1. THE JAVASCRIPT EXECUTION MODEL

Before you write a single line of React, you need to understand what happens when JavaScript *runs*. Not at the "it interprets your code" level — at the actual, mechanical, step-by-step level.

JavaScript has three core components at runtime:

1. **A thread of execution** — goes through the code line by line, executing each statement
2. **Memory (Variable Environment)** — stores data as it goes (variables, function definitions)
3. **A call stack** — tracks where the thread of execution currently is

That's it. JavaScript is, at its core, remarkably simple. One thread. One piece of code executing at a time. Memory to store stuff. A stack to keep track of where you are.

### 1.1 Global Execution Context

When your JavaScript file first loads, the engine creates a **Global Execution Context**. This is the top-level environment where your code runs. It has two components:

- **Thread of execution:** starts at line 1, goes down
- **Memory (Variable Environment):** starts empty, fills up as the engine encounters declarations

Let's trace through a simple program:

```javascript
const name = "frontend architect";
const score = 100;

function addBonus(value) {
  const bonus = 50;
  return value + bonus;
}

const finalScore = addBonus(score);
```

Here's what the engine does, step by step:

**Line 1:** Store `name` in memory with value `"frontend architect"`
```
Memory:
  name: "frontend architect"
```

**Line 2:** Store `score` in memory with value `100`
```
Memory:
  name: "frontend architect"
  score: 100
```

**Lines 4-7:** Store `addBonus` in memory. The value is the *entire function definition* — not the result of calling it. The function is stored as a blob of code, ready to be executed later.
```
Memory:
  name: "frontend architect"
  score: 100
  addBonus: function
```

**Line 9:** Now things get interesting. The engine encounters `addBonus(score)`. This is a **function invocation**. When JavaScript sees those parentheses `()`, it does something specific: it creates a brand new execution context.

### 1.2 Function Execution Contexts

Every function invocation creates a new execution context with its own:
- Thread of execution (goes through the function body line by line)
- Memory (local variable environment — parameters and local variables)

For `addBonus(score)`:

```
Global Execution Context:
  Memory: { name: "frontend architect", score: 100, addBonus: fn, finalScore: undefined }
  
  └─ addBonus Execution Context:
       Memory: { value: 100, bonus: 50 }
       Thread: executing lines 5-6
```

**Inside the function:**
1. Parameter `value` is stored in local memory with value `100` (the argument passed in)
2. `bonus` is stored in local memory with value `50`
3. `return value + bonus` evaluates to `150` and is returned to the calling context
4. The execution context is destroyed (its memory is garbage collected)

**Back in global:** `finalScore` is assigned the returned value `150`.

```
Memory:
  name: "frontend architect"
  score: 100
  addBonus: function
  finalScore: 150
```

This might seem trivially simple. It's not. This exact mechanism — creating execution contexts with their own memory — is the foundation of closures, which is the foundation of React hooks. Stay with me.

### 1.3 The Call Stack

How does the engine keep track of which execution context it's currently in? The **call stack**.

The call stack is a stack data structure (last in, first out) that tracks execution contexts:

```
When program starts:
  ┌─────────────────────┐
  │  Global Execution    │  ← currently executing
  └─────────────────────┘

When addBonus(score) is called:
  ┌─────────────────────┐
  │  addBonus()          │  ← currently executing
  ├─────────────────────┤
  │  Global Execution    │  ← paused, waiting
  └─────────────────────┘

When addBonus returns:
  ┌─────────────────────┐
  │  Global Execution    │  ← resumes executing
  └─────────────────────┘
```

The engine always executes whatever is on top of the call stack. When a function is called, its execution context is pushed onto the stack. When it returns, the context is popped off, and the engine resumes the context below.

### 1.4 Nested Function Calls

```javascript
function outer() {
  function inner() {
    return "done";
  }
  return inner();
}

const result = outer();
```

Call stack trace:

```
Step 1: Global
  ┌──────────┐
  │  Global   │
  └──────────┘

Step 2: outer() is called
  ┌──────────┐
  │  outer()  │
  ├──────────┤
  │  Global   │
  └──────────┘

Step 3: inner() is called inside outer
  ┌──────────┐
  │  inner()  │
  ├──────────┤
  │  outer()  │
  ├──────────┤
  │  Global   │
  └──────────┘

Step 4: inner() returns "done"
  ┌──────────┐
  │  outer()  │  ← receives "done", returns it
  ├──────────┤
  │  Global   │
  └──────────┘

Step 5: outer() returns "done"
  ┌──────────┐
  │  Global   │  ← result = "done"
  └──────────┘
```

**Why this matters:** When you see a stack trace in your error console, you are literally looking at the call stack at the moment the error occurred. Each line is an execution context. Understanding the call stack is understanding how to read error traces — the most basic debugging skill there is.

### 1.5 Stack Overflow

The call stack has a finite size (varies by engine — V8 allows roughly 10,000-15,000 frames). Recursive functions that don't have a base case will keep pushing execution contexts until the stack overflows:

```javascript
function recurse() {
  return recurse(); // no base case — infinite recursion
}
recurse(); // RangeError: Maximum call stack size exceeded
```

This is literally what "stack overflow" means — the call stack ran out of space.

### 1.6 Higher-Order Functions

A higher-order function is a function that takes a function as an argument or returns a function. This is not a fancy concept — it falls directly out of the execution model we just learned.

```javascript
function runTwice(fn) {
  fn();
  fn();
}

runTwice(function() {
  console.log("hello");
});
```

When `runTwice` is called:
1. `fn` is stored in `runTwice`'s local memory, pointing to the anonymous function
2. `fn()` invokes it — creates a new execution context, runs `console.log("hello")`, pops off
3. `fn()` invokes it again — same thing

The function passed in is just data stored in memory, like any other value. JavaScript treats functions as **first-class objects** — they can be stored in variables, passed as arguments, and returned from other functions.

This is not a language curiosity. This is the foundation of:
- Array methods (`map`, `filter`, `reduce` all take functions as arguments)
- Event handlers (`addEventListener` takes a callback function)
- React components (a component is a function passed to React's renderer)
- Hooks (the function you pass to `useEffect` is a callback)

```javascript
const numbers = [1, 2, 3, 4, 5];

const doubled = numbers.map(function(num) {
  return num * 2;
});
// doubled: [2, 4, 6, 8, 10]
```

`map` is a higher-order function. It takes your function, calls it once for each element with that element as the argument, and collects the return values into a new array. The execution model is the same — each call to your function creates a new execution context with `num` in its local memory.

---

## 2. CLOSURES

Closures are the single most important concept in JavaScript. They are the mechanism behind React hooks, module patterns, data privacy, memoization, and half the patterns in functional programming. Most engineers can give a textbook definition of closures. Very few can trace exactly what's happening in memory.

Let's fix that.

### 2.1 The Fundamental Mechanism

When a function is defined, it gets a hidden property called `[[Environment]]` (the spec name — you can't access it directly). This property is a reference to the variable environment of the execution context where the function was defined.

That's it. That's closures. A function remembers the variables that were in scope when it was created.

```javascript
function createCounter() {
  let count = 0;
  
  function increment() {
    count++;
    return count;
  }
  
  return increment;
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

Let's trace this in excruciating detail:

**Step 1: Global execution begins**
```
Global Memory:
  createCounter: function
  counter: undefined (not yet assigned)
```

**Step 2: `createCounter()` is called**

A new execution context is created and pushed onto the call stack.

```
createCounter Execution Context:
  Local Memory:
    count: 0
    increment: function (with [[Environment]] → createCounter's memory)
```

The critical moment: when `increment` is defined inside `createCounter`, the engine attaches a `[[Environment]]` reference pointing back to `createCounter`'s variable environment. This reference includes `count`.

**Step 3: `createCounter` returns `increment`**

The execution context for `createCounter` is popped off the call stack. Its local memory would normally be garbage collected. But `increment` still has a reference to it through `[[Environment]]`. So the garbage collector keeps it alive.

```
Global Memory:
  createCounter: function
  counter: function increment (with [[Environment]] → { count: 0 })
```

The function stored in `counter` is the `increment` function, carrying with it a backpack of data from its birth environment. That backpack contains `count`.

**Step 4: `counter()` is called (first time)**

A new execution context is created. The engine looks for `count` — it's not in the local memory of this execution context. So it follows the `[[Environment]]` reference and finds `count` in the closed-over scope. `count` is incremented from 0 to 1. Returns 1.

```
After first call:
  counter's [[Environment]]: { count: 1 }
```

**Step 5: `counter()` is called again**

Same process. `count` is found in the closure. Incremented from 1 to 2. Returns 2.

**Step 6: `counter()` is called a third time**

`count` goes from 2 to 3. Returns 3.

The variable `count` persists between calls because it lives in the closure — the `[[Environment]]` backpack attached to the function. It's not a global variable. It's not stored on the function object in any accessible way. It's a private, persistent variable that only `increment` can access.

### 2.2 The Scope Chain

When JavaScript looks up a variable, it follows a chain:

1. **Local memory** (the current execution context's variable environment)
2. **Closed-over scope** (the `[[Environment]]` reference — the enclosing function's variable environment)
3. **Outer scope** (if there are nested closures, keep going outward)
4. **Global scope** (the global execution context's variable environment)

If the variable isn't found anywhere in this chain, you get a `ReferenceError`.

```javascript
const globalVar = "global";

function outer() {
  const outerVar = "outer";
  
  function middle() {
    const middleVar = "middle";
    
    function inner() {
      console.log(middleVar); // found in middle's scope
      console.log(outerVar);  // found in outer's scope
      console.log(globalVar); // found in global scope
    }
    
    return inner;
  }
  
  return middle;
}

const middleFn = outer();
const innerFn = middleFn();
innerFn(); // logs: "middle", "outer", "global"
```

Each function carries its own `[[Environment]]` backpack, which chains to the outer scope's backpack. This is the **scope chain** — a linked list of variable environments from the innermost to the outermost scope.

### 2.3 Closures and Data Privacy

Closures give JavaScript something it doesn't natively have: private variables.

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance;
  
  return {
    deposit(amount) {
      if (amount <= 0) throw new Error("Deposit must be positive");
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > balance) throw new Error("Insufficient funds");
      balance -= amount;
      return balance;
    },
    getBalance() {
      return balance;
    }
  };
}

const account = createBankAccount(1000);
account.deposit(500);    // 1500
account.withdraw(200);   // 1300
account.getBalance();    // 1300

// There is NO way to access `balance` directly.
// account.balance → undefined
// The only way to interact with it is through the methods.
```

`balance` is permanently closed over by the three methods. It cannot be accessed or modified from outside. This is encapsulation through closures — the module pattern that dominated JavaScript before ES6 classes.

### 2.4 The Classic Loop Problem

This is the closure question that appears in every JavaScript interview, and it reveals whether someone truly understands closures or just memorized a definition:

```javascript
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
// Output: 3, 3, 3 (not 0, 1, 2)
```

Why? Let's trace it:

1. `var i` is function-scoped (or global-scoped here). There is ONE `i` variable in memory.
2. The loop runs three times, scheduling three `setTimeout` callbacks. Each callback closes over the SAME `i`.
3. The loop finishes. `i` is now 3 (the condition `i < 3` failed).
4. One second later, all three callbacks execute. They each look up `i` in their closure. They all find the SAME `i`, which is 3.

The fix with `let`:
```javascript
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
// Output: 0, 1, 2
```

`let` is block-scoped. Each iteration of the loop creates a new `i` in a new block scope. Each callback closes over a DIFFERENT `i`. This is not magic — it's the closure mechanism working with block scoping instead of function scoping.

The fix with an IIFE (how we did it before `let` existed):
```javascript
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(function() {
      console.log(j);
    }, 1000);
  })(i);
}
// Output: 0, 1, 2
```

The IIFE creates a new execution context for each iteration, with `j` as a local variable capturing the current value of `i`. Each callback closes over its own `j`.

### 2.5 Why Closures Matter for React

This is where the theoretical becomes extremely practical. React hooks are closures. When you write:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  
  function handleClick() {
    setCount(count + 1);
  }
  
  return <button onClick={handleClick}>{count}</button>;
}
```

Every time `Counter` renders, it creates a new execution context. `count` and `setCount` are stored in that execution context's local memory. `handleClick` is defined inside that execution context, so it closes over `count`.

**This is why you get stale closures:**

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // always logs 0!
      setCount(count + 1); // always sets to 1!
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty dependency array — effect only runs on mount
  
  return <div>{count}</div>;
}
```

Why does this break? Let's trace the closure:

1. First render: `count` is `0`. The effect runs and creates an interval. The interval callback closes over `count` from this render's execution context — which is `0`.
2. `setCount(count + 1)` sets count to `0 + 1 = 1`.
3. Second render: `count` is `1`. But the effect doesn't re-run (empty dependency array). The interval still has the OLD callback, which closes over the OLD `count` (`0`).
4. Every tick: `count` in the closure is still `0`. `setCount(0 + 1)` keeps setting it to `1`.

The fix — use the functional updater form:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // doesn't read count from closure
  }, 1000);
  return () => clearInterval(id);
}, []);
```

`prev => prev + 1` doesn't close over `count`. It receives the current state as an argument from React. No stale closure.

**This is not a React quirk. This is closures.** Once you understand the execution model, stale closures become obvious, not mysterious.

### 2.6 Closures and Memoization

Closures enable caching (memoization) — storing computed results so you don't recompute them:

```javascript
function memoize(fn) {
  const cache = {};
  
  return function(...args) {
    const key = JSON.stringify(args);
    if (key in cache) {
      return cache[key];
    }
    const result = fn(...args);
    cache[key] = result;
    return result;
  };
}

const expensiveSquare = memoize(function(n) {
  console.log("computing...");
  return n * n;
});

expensiveSquare(4); // logs "computing...", returns 16
expensiveSquare(4); // returns 16 (no log — cache hit)
expensiveSquare(5); // logs "computing...", returns 25
```

The returned function closes over `cache` and `fn`. The cache persists across calls because it lives in the closure. This is exactly the same pattern React uses for `useMemo` and `useCallback` — closures that persist data across renders.

---

## 3. ASYNCHRONOUS JAVASCRIPT

Here is the single most important thing to understand about JavaScript and the browser:

**JavaScript is single-threaded. The browser is not.**

JavaScript can only do one thing at a time. It has one thread of execution, one call stack. But the browser (or Node.js runtime) is a multi-threaded C++ application with networking capabilities, timers, a rendering engine, a file system interface, and more.

When JavaScript needs something that takes time — a network request, a timer, reading a file — it doesn't wait. It hands the work off to the browser (a C++ subsystem), and the browser does that work on a separate thread. When the work is done, the browser puts the result into a queue, and JavaScript picks it up when the call stack is empty.

This is the entire model. Let's build it up piece by piece.

### 3.1 The Components

```
┌──────────────────────────────────────────────────────────────────┐
│                        JAVASCRIPT ENGINE                         │
│                                                                  │
│   ┌──────────────────┐    ┌──────────────────────────────────┐  │
│   │    CALL STACK     │    │     MEMORY (HEAP)                │  │
│   │                   │    │                                  │  │
│   │  [current frame]  │    │  variables, objects, functions   │  │
│   │  [caller frame]   │    │                                  │  │
│   │  [global]         │    │                                  │  │
│   └──────────────────┘    └──────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              │ (JavaScript can trigger browser features)
                              │
┌─────────────────────────────▼────────────────────────────────────┐
│                    BROWSER / WEB APIs (C++)                       │
│                                                                  │
│   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐ │
│   │   Timer     │  │  Network   │  │    DOM      │  │  Console │ │
│   │ (setTimeout)│  │  (fetch)   │  │ (C++ tree)  │  │          │ │
│   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────────┘ │
│         │               │               │                        │
└─────────┼───────────────┼───────────────┼────────────────────────┘
          │               │               │
          ▼               ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                         QUEUES                                    │
│                                                                  │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │  MICROTASK QUEUE (Promises, queueMicrotask, MutationObserver)│
│   │  Priority: HIGH — drained completely before any macrotask   │ │
│   └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │  CALLBACK QUEUE / MACROTASK QUEUE (setTimeout, setInterval, │ │
│   │  DOM events, I/O callbacks)                                 │ │
│   │  Priority: LOW — one at a time, after microtask queue empty │ │
│   └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
          │
          │ EVENT LOOP: "Is the call stack empty? 
          │              If yes, take from microtask queue (all of them).
          │              Then take ONE from callback queue.
          │              Repeat."
          ▼
```

### 3.2 Tracing setTimeout

```javascript
console.log("first");

setTimeout(function callback() {
  console.log("second");
}, 0);

console.log("third");
```

Output: `first`, `third`, `second`

Even with a 0ms delay. Let's trace exactly why:

**Step 1:** `console.log("first")` — synchronous, runs immediately. Output: `first`.

**Step 2:** `setTimeout(callback, 0)` — JavaScript does NOT set a timer itself. It calls a **Web API** (a browser feature written in C++). The browser starts a timer on a separate thread. JavaScript continues immediately — it does not wait.

```
Call Stack: [global]
Browser Timer: callback (0ms remaining)
Callback Queue: (empty)
```

**Step 3:** The browser's timer completes (basically instantly, since delay is 0ms). The browser puts `callback` into the **Callback Queue**. But it does NOT execute yet — the call stack is not empty (global is still executing).

```
Call Stack: [global]
Callback Queue: [callback]
```

**Step 4:** `console.log("third")` — synchronous, runs immediately. Output: `third`.

**Step 5:** Global execution finishes. The call stack is empty.

**Step 6:** The **Event Loop** checks: "Is the call stack empty? Yes. Is there anything in the microtask queue? No. Is there anything in the callback queue? Yes." It moves `callback` from the queue to the call stack.

**Step 7:** `callback` executes. `console.log("second")`. Output: `second`.

**The key insight:** `setTimeout(fn, 0)` does not mean "run this in 0 milliseconds." It means "run this as soon as possible AFTER all synchronous code has finished." The 0ms is the minimum delay before the callback is eligible to enter the queue, not a guarantee of when it will execute.

### 3.3 Tracing fetch

```javascript
console.log("start");

fetch("https://api.example.com/data")
  .then(response => response.json())
  .then(data => console.log("data:", data));

console.log("end");
```

Output: `start`, `end`, then `data: ...` (when the network request completes)

**Step 1:** `console.log("start")` — synchronous. Output: `start`.

**Step 2:** `fetch(url)` — this does TWO things simultaneously:

1. **In JavaScript:** Creates a Promise object in memory. This object has:
   - A `state` property: `"pending"`
   - A `value` property: `undefined`
   - An internal `onFulfilled` array: `[]`

2. **In the browser:** Triggers the browser's networking C++ code (XMLHttpRequest under the hood, or the Fetch API's native implementation) to make an HTTP request on a background thread. JavaScript does NOT wait for this — it continues immediately.

```
Call Stack: [global]
JS Memory: { fetchPromise: { state: "pending", value: undefined, onFulfilled: [] } }
Browser Network: GET https://api.example.com/data (in progress on C++ thread)
```

**Step 3:** `.then(response => response.json())` — this pushes the function `response => response.json()` into the Promise's `onFulfilled` array. It does not execute the function. It just registers it.

**Step 4:** `.then(data => console.log("data:", data))` — the second `.then` creates a new Promise that's chained to the first. When the first resolves, the second's `onFulfilled` runs.

**Step 5:** `console.log("end")` — synchronous. Output: `end`.

**Step 6:** Global execution finishes. Call stack is empty.

**Step 7:** (Some time later) The browser's networking code receives the HTTP response. It sets the Promise's `value` to the Response object and changes `state` to `"fulfilled"`. The functions in `onFulfilled` are placed into the **Microtask Queue** (not the callback queue — this is critical).

**Step 8:** The Event Loop checks: "Call stack empty? Yes. Microtask queue? Yes — there's a function." It runs `response => response.json()`.

**Step 9:** `response.json()` returns ANOTHER Promise. The second `.then`'s function is registered on this new Promise.

**Step 10:** When JSON parsing completes, `data => console.log("data:", data)` enters the microtask queue. Event loop runs it. Output: `data: {...}`.

### 3.4 Why Fetch Uses the Microtask Queue

This is a design decision, not an accident. Promises use the microtask queue, which has **higher priority** than the callback queue. The microtask queue is completely drained before the event loop looks at the callback queue.

This means promise callbacks run before setTimeout callbacks, even if the setTimeout was registered first and its delay has already elapsed:

```javascript
setTimeout(() => console.log("timeout"), 0);
Promise.resolve().then(() => console.log("promise"));
console.log("sync");
```

Output: `sync`, `promise`, `timeout`

Why? After synchronous code finishes:
1. Event loop checks microtask queue first: `promise` callback is there. Runs it.
2. Microtask queue is empty. Event loop checks callback queue: `timeout` callback is there. Runs it.

Promises always jump the line ahead of timers. This is by design — it keeps promise chains fast and predictable.

---

## 4. PROMISES UNDER THE HOOD

Most engineers learn promises as "a thing that represents a future value." That's true but useless for understanding how they actually work. Let's build a mental model of a Promise as a JavaScript object with specific properties and behaviors.

### 4.1 The Promise Object

When you create a Promise, the engine creates an object that looks conceptually like this:

```javascript
// Conceptual model — not actual implementation
{
  state: "pending",       // "pending" | "fulfilled" | "rejected"
  value: undefined,       // the resolved value (set when fulfilled)
  onFulfilled: [],        // array of functions to call when fulfilled
  onRejected: [],         // array of functions to call when rejected
}
```

### 4.2 How .then() Works

`.then(fn)` does one simple thing: it pushes `fn` into the Promise's `onFulfilled` array.

```javascript
const promise = new Promise((resolve) => {
  // resolve will be called later
  setTimeout(() => resolve(42), 1000);
});

promise.then(value => console.log(value));
```

Step by step:

1. `new Promise(executor)` creates the Promise object. Immediately calls `executor` with a `resolve` function.
2. Inside the executor: `setTimeout` hands off to the browser. JavaScript continues.
3. `promise.then(fn)` — pushes `fn` into the Promise's `onFulfilled` array. The Promise is still pending, so the function just sits there, waiting.

```
Promise: { state: "pending", value: undefined, onFulfilled: [fn] }
```

4. One second later: the browser's timer fires. `resolve(42)` is called. This does:
   - Sets `state` to `"fulfilled"`
   - Sets `value` to `42`
   - For each function in `onFulfilled`: schedules it to run in the **microtask queue** with `value` as the argument

5. Event loop picks up the microtask: `fn(42)` runs. Output: `42`.

### 4.3 Promise Chaining — The Mechanics

`.then()` doesn't just register a callback — it also returns a NEW Promise. This is what enables chaining.

```javascript
const p = Promise.resolve(1)
  .then(x => x + 1)     // returns a new Promise that resolves to 2
  .then(x => x * 3)     // returns a new Promise that resolves to 6
  .then(x => console.log(x)); // 6
```

Each `.then()`:
1. Creates a new Promise (let's call it `p2`)
2. Pushes a wrapper function into the previous Promise's `onFulfilled` array
3. When the wrapper runs, it calls your function, takes the return value, and resolves `p2` with it
4. Resolving `p2` triggers ITS `onFulfilled` callbacks, which were registered by the next `.then()`

It's Promises all the way down. Each `.then()` creates a new Promise, and the chain is a linked list of Promises where each one's resolution triggers the next one's callbacks.

### 4.4 Error Handling — .catch() and Rejection

`.catch(fn)` is identical to `.then(null, fn)`. It pushes `fn` into the `onRejected` array.

When a Promise rejects (either through `reject()` or an unhandled exception in a `.then()` callback), the engine:
1. Sets `state` to `"rejected"`
2. Sets `value` to the rejection reason (usually an Error)
3. Looks for the nearest `onRejected` handler in the chain
4. Skips all `onFulfilled` handlers until it finds a `catch`

```javascript
Promise.resolve(1)
  .then(x => { throw new Error("boom"); })
  .then(x => console.log("skipped"))      // skipped
  .then(x => console.log("also skipped")) // skipped
  .catch(err => console.log(err.message)) // "boom"
  .then(x => console.log("recovered"));   // "recovered"
```

After a `.catch()` handles the error, the chain continues with a fulfilled Promise. The `.catch()` return value (or `undefined` if it doesn't return anything) becomes the resolved value for the next `.then()`.

### 4.5 Promise.all, Promise.race, Promise.allSettled

These are higher-order functions that operate on arrays of Promises:

**`Promise.all(promises)`** — creates a Promise that fulfills when ALL input promises fulfill, or rejects when ANY one rejects. Returns an array of results in the same order.

```javascript
const results = await Promise.all([
  fetch("/api/users"),
  fetch("/api/posts"),
  fetch("/api/comments"),
]);
// results[0] = users response, results[1] = posts, results[2] = comments
// If ANY fetch fails, the whole Promise.all rejects
```

**`Promise.race(promises)`** — resolves or rejects with the FIRST promise to settle.

```javascript
const result = await Promise.race([
  fetch("/api/data"),
  new Promise((_, reject) => setTimeout(() => reject(new Error("timeout")), 5000)),
]);
// Either gets data or times out after 5 seconds
```

**`Promise.allSettled(promises)`** — waits for ALL promises to settle (fulfill or reject), never short-circuits. Returns an array of `{ status, value/reason }` objects.

```javascript
const results = await Promise.allSettled([
  fetch("/api/critical"),
  fetch("/api/optional"),
]);
// results: [{ status: "fulfilled", value: Response }, { status: "rejected", reason: Error }]
// Never rejects — you get the status of every promise
```

### 4.6 Promise.withResolvers()

ES2024 added `Promise.withResolvers()`, which gives you the resolve and reject functions externally:

```javascript
const { promise, resolve, reject } = Promise.withResolvers();

// Now you can resolve/reject from outside the executor
setTimeout(() => resolve("done"), 1000);

const result = await promise; // "done"
```

This is useful when you need to resolve a Promise from a different scope — event handlers, callback-based APIs, or any situation where the Promise constructor's executor pattern is awkward.

---

## 5. ASYNC/AWAIT

`async/await` is the modern syntax for working with Promises. But it is NOT a new feature — it's syntactic sugar over generators and promises. Understanding this is crucial because it explains exactly when your code pauses, resumes, and what the event loop is doing.

### 5.1 What async Does

The `async` keyword before a function does one thing: it makes the function return a Promise. Whatever value you return is automatically wrapped in `Promise.resolve()`.

```javascript
async function getNumber() {
  return 42;
}

// Identical to:
function getNumber() {
  return Promise.resolve(42);
}

getNumber().then(n => console.log(n)); // 42
```

### 5.2 What await Does

`await` pauses the execution of the async function and waits for the Promise to settle. But here's the crucial part: **it doesn't block the call stack.** When the engine hits `await`, it:

1. Evaluates the expression after `await` (which should be a Promise)
2. Suspends the async function — removes it from the call stack
3. Returns control to the caller
4. When the Promise resolves, the async function is scheduled to resume in the **microtask queue**

```javascript
async function fetchData() {
  console.log("before fetch");
  const response = await fetch("/api/data");  // pauses HERE
  console.log("after fetch");                  // resumes here when fetch resolves
  return response.json();
}

console.log("start");
fetchData();
console.log("end");
```

Output: `start`, `before fetch`, `end`, `after fetch`

Let's trace this:

1. `console.log("start")` — synchronous. Output: `start`.
2. `fetchData()` is called. Enters the function.
3. `console.log("before fetch")` — synchronous inside the function. Output: `before fetch`.
4. `await fetch("/api/data")` — `fetch` starts the network request (browser C++ thread). The `await` **suspends `fetchData`** and returns a Promise to the caller.
5. Back in global: `console.log("end")` — synchronous. Output: `end`.
6. Call stack is empty. Event loop runs.
7. When the fetch completes, the Promise resolves. `fetchData` resumes from where it paused. `console.log("after fetch")`. Output: `after fetch`.

**The key insight:** `await` doesn't block the entire program. It only pauses the async function. Everything outside that function continues executing. This is what makes async/await non-blocking — the call stack is free for other work while the async function is suspended.

### 5.3 Sequential vs Parallel Awaits

A common mistake — awaiting in sequence when you could await in parallel:

```javascript
// SLOW: Sequential — each fetch waits for the previous one
async function getDataSequential() {
  const users = await fetch("/api/users");     // waits...
  const posts = await fetch("/api/posts");     // THEN starts this
  const comments = await fetch("/api/comments"); // THEN starts this
  return { users, posts, comments };
}
// Total time: users + posts + comments (e.g., 300ms + 200ms + 100ms = 600ms)

// FAST: Parallel — all fetches start at once
async function getDataParallel() {
  const [users, posts, comments] = await Promise.all([
    fetch("/api/users"),
    fetch("/api/posts"),
    fetch("/api/comments"),
  ]);
  return { users, posts, comments };
}
// Total time: max(users, posts, comments) (e.g., max(300, 200, 100) = 300ms)
```

This is a 2x speedup for free. If the requests don't depend on each other, start them all at once.

### 5.4 Error Handling with Async/Await

`try/catch` works naturally with `await`:

```javascript
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (error) {
    console.error("Failed to fetch user:", error.message);
    throw error; // re-throw to let the caller handle it
  }
}
```

When an awaited Promise rejects, the rejection is thrown as an exception in the async function. `try/catch` catches it just like any synchronous exception. This is one of the biggest ergonomic wins of async/await over raw `.then()/.catch()` chains.

### 5.5 The Generator Connection

Under the hood, `async/await` is implemented using generators (or a similar mechanism, depending on the engine). Here's the conceptual transformation:

```javascript
// What you write:
async function fetchData() {
  const a = await fetch("/api/a");
  const b = await fetch("/api/b");
  return [a, b];
}

// What the engine conceptually does (simplified):
function fetchData() {
  return new Promise((resolve, reject) => {
    const gen = function*() {
      const a = yield fetch("/api/a");
      const b = yield fetch("/api/b");
      return [a, b];
    }();
    
    function step(value) {
      const result = gen.next(value);
      if (result.done) {
        resolve(result.value);
      } else {
        Promise.resolve(result.value).then(step, reject);
      }
    }
    
    step();
  });
}
```

Each `await` becomes a `yield`. The generator pauses at each `yield`, and a runner function waits for the yielded Promise to resolve before calling `gen.next(resolvedValue)` to resume the generator with the result.

You don't need to write this manually. But understanding it explains WHY async/await works: it's generators (pausable functions) + Promises (future values) + microtask queue (scheduling).

---

## 6. ITERATORS AND GENERATORS

Generators are functions that can **pause**. They maintain their execution context across pauses, returning values incrementally instead of all at once. This is a fundamentally different execution model from normal functions.

### 6.1 Iterators — The Protocol

Before generators, let's understand the iteration protocol. An **iterator** is any object with a `next()` method that returns `{ value, done }`:

```javascript
function createRangeIterator(start, end) {
  let current = start;
  
  return {
    next() {
      if (current <= end) {
        return { value: current++, done: false };
      }
      return { value: undefined, done: true };
    }
  };
}

const iter = createRangeIterator(1, 3);
iter.next(); // { value: 1, done: false }
iter.next(); // { value: 2, done: false }
iter.next(); // { value: 3, done: false }
iter.next(); // { value: undefined, done: true }
```

An **iterable** is any object with a `[Symbol.iterator]()` method that returns an iterator. Arrays, strings, Maps, Sets are all iterables. `for...of` works on any iterable.

```javascript
const arr = [10, 20, 30];
const iter = arr[Symbol.iterator]();
iter.next(); // { value: 10, done: false }
iter.next(); // { value: 20, done: false }
iter.next(); // { value: 30, done: false }
iter.next(); // { value: undefined, done: true }
```

### 6.2 Generators — Pausable Functions

A generator function is declared with `function*`. When called, it doesn't execute its body — it returns a generator object (which is both an iterator and an iterable).

```javascript
function* generateNumbers() {
  console.log("start");
  yield 1;
  console.log("after first yield");
  yield 2;
  console.log("after second yield");
  yield 3;
  console.log("done");
}

const gen = generateNumbers();
// Nothing is logged yet! The function body hasn't started executing.

gen.next(); // logs "start", returns { value: 1, done: false }
// Function pauses at `yield 1`

gen.next(); // logs "after first yield", returns { value: 2, done: false }
// Function pauses at `yield 2`

gen.next(); // logs "after second yield", returns { value: 3, done: false }
// Function pauses at `yield 3`

gen.next(); // logs "done", returns { value: undefined, done: true }
// Function completes
```

**The crucial mechanism:** When a generator hits `yield`, it:
1. Returns the yielded value as `{ value, done: false }`
2. **Saves its entire execution context** (local variables, where it was in the code)
3. Pops off the call stack

When `next()` is called again, the generator:
1. Restores its saved execution context
2. Resumes from exactly where it paused
3. Continues until the next `yield` or the function ends

This is NOT the same as closures. Closures preserve access to variables but the function re-executes from the top each time. Generators preserve the entire execution state and resume mid-function.

### 6.3 Passing Values INTO Generators

`next(value)` sends a value back into the generator. The sent value becomes the result of the `yield` expression:

```javascript
function* conversation() {
  const name = yield "What is your name?";
  const age = yield `Hello ${name}! How old are you?`;
  return `${name} is ${age} years old.`;
}

const gen = conversation();
console.log(gen.next().value);          // "What is your name?"
console.log(gen.next("Alice").value);   // "Hello Alice! How old are you?"
console.log(gen.next(30).value);        // "Alice is 30 years old."
```

Step by step:
1. `gen.next()` — starts the generator. Runs until `yield "What is your name?"`. Returns `"What is your name?"`.
2. `gen.next("Alice")` — resumes the generator. The value `"Alice"` becomes the result of the first `yield`. So `name = "Alice"`. Continues to the next `yield`, returns `"Hello Alice! How old are you?"`.
3. `gen.next(30)` — resumes. `30` becomes the result of the second `yield`. `age = 30`. The function returns. Returns `{ value: "Alice is 30 years old.", done: true }`.

### 6.4 Infinite Sequences with Generators

Because generators are lazy (they only compute the next value when asked), they can represent infinite sequences:

```javascript
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
fib.next().value; // 0
fib.next().value; // 1
fib.next().value; // 1
fib.next().value; // 2
fib.next().value; // 3
fib.next().value; // 5
fib.next().value; // 8
```

The `while(true)` loop runs forever, but it only advances one step per `next()` call. The generator pauses at each `yield`, so it never actually creates an infinite loop.

This is lazy evaluation — values are computed on demand, not eagerly. It's memory-efficient (no giant array) and composable.

### 6.5 Generator Delegation with yield*

`yield*` delegates to another iterable or generator:

```javascript
function* inner() {
  yield 1;
  yield 2;
}

function* outer() {
  yield 0;
  yield* inner(); // delegates to inner
  yield 3;
}

const gen = outer();
gen.next(); // { value: 0, done: false }
gen.next(); // { value: 1, done: false } — from inner
gen.next(); // { value: 2, done: false } — from inner
gen.next(); // { value: 3, done: false } — back to outer
gen.next(); // { value: undefined, done: true }
```

### 6.6 Generators for Async Control Flow (Historical)

Before `async/await`, generators were used for async control flow with libraries like `co`:

```javascript
// The co pattern (historical, pre-async/await):
function* fetchUserFlow() {
  const response = yield fetch("/api/user");
  const data = yield response.json();
  console.log(data);
}

// A runner that makes this work:
function run(genFn) {
  const gen = genFn();
  
  function step(value) {
    const result = gen.next(value);
    if (result.done) return Promise.resolve(result.value);
    return Promise.resolve(result.value).then(step);
  }
  
  return step();
}

run(fetchUserFlow);
```

This is literally what `async/await` compiles to. `async function` is a generator. `await` is `yield`. The engine has a built-in runner. Understanding this connection demystifies async/await entirely.

---

## 7. THE EVENT LOOP IN DEPTH

Now that we understand all the pieces — the call stack, Web APIs, the microtask queue, the callback queue, promises, and async/await — let's put them all together with a complex example that interleaves all of them.

### 7.1 The Priority Model

The event loop follows a strict priority:

1. **Synchronous code** (call stack) — always runs first, to completion
2. **Microtask queue** — drained completely after each macrotask (all microtasks run before any callback queue task)
3. **Callback queue (macrotask queue)** — one task at a time, after the microtask queue is empty

Within the microtask queue, tasks run in FIFO order. Same for the callback queue.

**What goes where:**

| Microtask Queue | Callback Queue (Macrotask) |
|----------------|---------------------------|
| Promise `.then()` / `.catch()` / `.finally()` | `setTimeout` / `setInterval` |
| `async/await` continuations | DOM event callbacks |
| `queueMicrotask()` | `requestAnimationFrame` (special — runs before repaint) |
| `MutationObserver` | I/O callbacks (Node.js) |
| | `MessageChannel` |

### 7.2 The Complex Example

```javascript
console.log("1: sync");

setTimeout(() => {
  console.log("2: timeout 1");
  Promise.resolve().then(() => {
    console.log("3: promise inside timeout 1");
  });
}, 0);

setTimeout(() => {
  console.log("4: timeout 2");
}, 0);

Promise.resolve().then(() => {
  console.log("5: promise 1");
}).then(() => {
  console.log("6: promise 2");
});

Promise.resolve().then(() => {
  console.log("7: promise 3");
});

queueMicrotask(() => {
  console.log("8: microtask");
});

console.log("9: sync end");
```

**Before looking at the answer, try to work out the order yourself.** This is the exercise that builds real understanding.

---

**The answer:** `1`, `9`, `5`, `7`, `8`, `6`, `2`, `3`, `4`

Let's trace it:

**Phase 1: Synchronous execution (call stack)**
- `console.log("1: sync")` — Output: `1: sync`
- `setTimeout(cb1, 0)` — hands off to browser timer. `cb1` will go to callback queue.
- `setTimeout(cb2, 0)` — hands off to browser timer. `cb2` will go to callback queue.
- `Promise.resolve().then(fn5)` — Promise resolves immediately. `fn5` → microtask queue.
- `.then(fn6)` — creates a new Promise. `fn6` is registered but not queued yet (waiting for `fn5`'s Promise to resolve).
- `Promise.resolve().then(fn7)` — `fn7` → microtask queue.
- `queueMicrotask(fn8)` — `fn8` → microtask queue.
- `console.log("9: sync end")` — Output: `9: sync end`

State after synchronous code:
```
Call Stack: empty
Microtask Queue: [fn5, fn7, fn8]
Callback Queue: [cb1, cb2]
```

**Phase 2: Drain microtask queue**
- `fn5` runs: `console.log("5: promise 1")` — Output: `5: promise 1`. Its return value resolves the chained Promise, so `fn6` → microtask queue.
- `fn7` runs: `console.log("7: promise 3")` — Output: `7: promise 3`.
- `fn8` runs: `console.log("8: microtask")` — Output: `8: microtask`.
- `fn6` runs (it was added during this drain): `console.log("6: promise 2")` — Output: `6: promise 2`.
- Microtask queue is now empty.

**Phase 3: One macrotask from callback queue**
- `cb1` runs: `console.log("2: timeout 1")` — Output: `2: timeout 1`.
  - Inside `cb1`: `Promise.resolve().then(fn3)` — `fn3` → microtask queue.
  
**Phase 4: Drain microtask queue (again, before next macrotask)**
- `fn3` runs: `console.log("3: promise inside timeout 1")` — Output: `3: promise inside timeout 1`.

**Phase 5: Next macrotask**
- `cb2` runs: `console.log("4: timeout 2")` — Output: `4: timeout 2`.

Final output:
```
1: sync
9: sync end
5: promise 1
7: promise 3
8: microtask
6: promise 2
2: timeout 1
3: promise inside timeout 1
4: timeout 2
```

**The critical observations:**
1. ALL synchronous code runs first (1, 9)
2. ALL microtasks run next, including any microtasks added during the drain (5, 7, 8, 6)
3. Then ONE macrotask (2), followed by draining any new microtasks it created (3)
4. Then the next macrotask (4)

### 7.3 Microtask Starvation

Because the microtask queue is drained completely before any macrotask runs, you can starve the callback queue:

```javascript
// DON'T DO THIS — infinite microtask loop
function recursiveMicrotask() {
  queueMicrotask(() => {
    console.log("microtask");
    recursiveMicrotask(); // adds another microtask
  });
}

recursiveMicrotask();
setTimeout(() => console.log("timeout"), 0); // NEVER runs
```

Each microtask queues another microtask. The microtask queue never empties, so the event loop never gets to the callback queue. The `setTimeout` callback is starved. The browser becomes unresponsive because it can't process rendering or user input.

This is the same principle that makes long synchronous code dangerous — but more insidious because it looks asynchronous. The call stack IS empty between microtasks, but the event loop never moves past the microtask drain phase.

### 7.4 requestAnimationFrame

`requestAnimationFrame` (rAF) has its own timing that's distinct from both microtasks and macrotasks. It runs before the browser paints the next frame, typically at 60Hz (every ~16.67ms):

```
Event Loop Cycle:
  1. Run synchronous code (call stack)
  2. Drain microtask queue
  3. Is it time to render?
     Yes → Run requestAnimationFrame callbacks
         → Style calculation → Layout → Paint → Composite
     No  → Skip to step 4
  4. Run one macrotask from callback queue
  5. Go to step 2
```

rAF is the correct place for visual updates that need to be synchronized with the browser's repaint cycle. Using `setTimeout` for animations means your update might happen mid-frame or right after a paint, causing visual stuttering.

### 7.5 Why This Matters for React

React's state updates are batched and scheduled using microtasks (in React 18+, all updates are automatically batched). When you call `setState`:

1. React doesn't re-render immediately
2. It marks the component as "needs update"
3. It schedules the re-render as a microtask (or uses its own scheduler for concurrent features)
4. After all your event handler code runs, the microtask fires and React processes all pending updates in one batch

This is why `setState` is "asynchronous" — it's not a network request, it's a microtask. And it's why you can call `setState` three times in an event handler and only get one re-render.

---

## 8. PROTOTYPES AND THIS

JavaScript's object system is prototype-based, not class-based. Even with the `class` syntax, under the hood it's prototypes. Understanding this explains several React patterns and common bugs.

### 8.1 The Prototype Chain

Every object in JavaScript has a hidden link to another object called its **prototype**. When you access a property on an object, the engine:

1. Looks for the property on the object itself
2. If not found, looks on the object's prototype
3. If not found, looks on the prototype's prototype
4. ... continues up the chain until it reaches `null`

```javascript
const animal = {
  alive: true,
  breathe() {
    return "breathing";
  }
};

const dog = Object.create(animal);
dog.bark = function() {
  return "woof";
};

dog.bark();    // "woof" — found on dog itself
dog.breathe(); // "breathing" — not on dog, found on animal (dog's prototype)
dog.alive;     // true — found on animal
dog.fly;       // undefined — not found anywhere in the chain
```

```
dog: { bark: fn }
  └─ __proto__ → animal: { alive: true, breathe: fn }
       └─ __proto__ → Object.prototype: { toString: fn, ... }
            └─ __proto__ → null
```

### 8.2 Constructor Functions and new

Before ES6 classes, objects were created with constructor functions:

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}`;
};

const alice = new Person("Alice", 30);
alice.greet(); // "Hi, I'm Alice"
```

What `new` does:
1. Creates a new empty object
2. Sets the object's `__proto__` to `Person.prototype`
3. Calls `Person` with `this` bound to the new object
4. Returns the new object (unless the constructor explicitly returns a different object)

Methods on `prototype` are shared by all instances — they exist in one place in memory, not copied to each instance. This is memory-efficient but can be confusing.

### 8.3 ES6 Classes — Syntactic Sugar

```javascript
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
  
  greet() {
    return `Hi, I'm ${this.name}`;
  }
}
```

This is exactly equivalent to the constructor function pattern above. `class` is syntactic sugar over prototypes. `greet` goes on `Person.prototype`. `new Person(...)` does the same four steps.

### 8.4 The this Keyword

`this` is the most confusing keyword in JavaScript because its value depends on **how the function is called**, not where it's defined. There are four rules:

**Rule 1: Default binding** — In a regular function call, `this` is `undefined` (strict mode) or the global object (sloppy mode).

```javascript
function showThis() {
  console.log(this);
}
showThis(); // undefined (strict) or window/global (sloppy)
```

**Rule 2: Implicit binding** — When a function is called as a method on an object, `this` is that object.

```javascript
const obj = {
  name: "Alice",
  greet() {
    console.log(this.name);
  }
};
obj.greet(); // "Alice" — this is obj
```

But watch out:
```javascript
const greetFn = obj.greet;
greetFn(); // undefined — this is NOT obj anymore!
```

When you extract the method from the object, you lose the implicit binding. `this` reverts to default binding. This is a VERY common source of bugs.

**Rule 3: Explicit binding** — `call`, `apply`, and `bind` explicitly set `this`.

```javascript
function greet() {
  console.log(this.name);
}

const alice = { name: "Alice" };
const bob = { name: "Bob" };

greet.call(alice);  // "Alice"
greet.call(bob);    // "Bob"

const greetAlice = greet.bind(alice);
greetAlice(); // "Alice" — permanently bound
```

**Rule 4: new binding** — When a function is called with `new`, `this` is the newly created object.

```javascript
function Person(name) {
  this.name = name; // this = the new object
}
const p = new Person("Alice"); // this = p
```

**Priority:** new > explicit (bind/call/apply) > implicit (method call) > default

### 8.5 Arrow Functions and Lexical this

Arrow functions don't have their own `this`. They inherit `this` from their enclosing scope at the time they're defined. This is called **lexical this**.

```javascript
const obj = {
  name: "Alice",
  greet: function() {
    // Regular function — `this` is `obj` (implicit binding)
    
    setTimeout(function() {
      console.log(this.name); // undefined! `this` is NOT obj
      // setTimeout calls the callback as a regular function (default binding)
    }, 100);
    
    setTimeout(() => {
      console.log(this.name); // "Alice"! Arrow function uses lexical `this`
      // `this` is inherited from `greet`, where `this` is `obj`
    }, 100);
  }
};
```

This is why arrow functions "fixed" the `this` problem in callbacks. Before arrow functions, you had to write `const self = this;` or use `.bind(this)`.

### 8.6 Why This Matters for React

**Class components (historical but still in codebases):**

```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    // Without this bind, `this` in handleClick would be undefined
    this.handleClick = this.handleClick.bind(this);
  }
  
  handleClick() {
    // Without bind: `this` is undefined (default binding in strict mode)
    // Because React calls this as a regular function, not a method
    this.setState({ count: this.state.count + 1 });
  }
  
  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}
```

When React calls `onClick`, it calls the function directly (not as `this.handleClick()` on the instance). Without `.bind(this)` in the constructor, `this` inside `handleClick` would be `undefined`.

The class field syntax (which uses arrow functions) fixed this:

```jsx
class Counter extends React.Component {
  state = { count: 0 };
  
  // Arrow function — lexical this, automatically bound to the instance
  handleClick = () => {
    this.setState({ count: this.state.count + 1 });
  };
  
  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}
```

**Function components and hooks eliminated this problem entirely:**

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  
  // No `this` anywhere. Just closures.
  const handleClick = () => setCount(count + 1);
  
  return <button onClick={handleClick}>{count}</button>;
}
```

Hooks replaced `this` with closures. Instead of `this.state.count`, you have `count` from the closure. Instead of `this.setState`, you have `setCount` from the closure. The execution model is simpler, the bugs are fewer, and the code is more readable.

This is one of the core reasons function components with hooks won over class components — they leverage closures (a fundamental JavaScript mechanism) instead of `this` (a confusing, error-prone mechanism).

### 8.7 Prototypal Inheritance vs Composition

JavaScript's prototype chain enables inheritance:

```javascript
class Animal {
  breathe() { return "breathing"; }
}

class Dog extends Animal {
  bark() { return "woof"; }
}

const dog = new Dog();
dog.bark();    // "woof" — on Dog.prototype
dog.breathe(); // "breathing" — on Animal.prototype (via prototype chain)
```

But React (and modern JavaScript) strongly favors **composition over inheritance**:

```javascript
// Composition — preferred
function useBreathe() {
  return { breathe: () => "breathing" };
}

function useBark() {
  return { bark: () => "woof" };
}

function createDog() {
  return {
    ...useBreathe(),
    ...useBark(),
  };
}
```

React hooks ARE the composition pattern:

```jsx
function useAuth() { /* ... */ }
function useTheme() { /* ... */ }
function useAnalytics() { /* ... */ }

function UserProfile() {
  const auth = useAuth();
  const theme = useTheme();
  const analytics = useAnalytics();
  // Composed from three independent pieces of logic
}
```

No inheritance hierarchy. No `this`. Just functions composing closures. This is why understanding closures is more important than understanding prototypes for modern React development — but you need to understand both to work with the full JavaScript ecosystem.

---

## SUMMARY: THE MENTAL MODEL

Here is the complete mental model for JavaScript execution:

1. **JavaScript is single-threaded.** One call stack, one thread of execution. Everything runs to completion before the next thing starts.

2. **The browser is multi-threaded.** Network requests, timers, DOM operations — these are C++ features running on separate threads. JavaScript triggers them and continues.

3. **Functions create execution contexts** with their own memory. The call stack tracks which context is active.

4. **Closures** are functions that carry a reference to their birth scope's variables. This is the mechanism behind data privacy, memoization, and React hooks.

5. **Promises** are objects with a state, a value, and arrays of callback functions. `.then()` pushes into the array. Resolution triggers the callbacks via the microtask queue.

6. **Async/await** is generators + promises. `await` pauses the generator, and the microtask queue resumes it when the promise resolves.

7. **The event loop** has a strict priority: synchronous code > microtasks (all of them) > one macrotask > repeat. Microtasks can starve macrotasks.

8. **`this`** depends on how a function is called, not where it's defined. Arrow functions use lexical `this`. Hooks eliminated `this` in favor of closures.

Every concept in this chapter will appear again in the React chapters. Closures power hooks. The event loop explains React's batching. Promises power data fetching. Composition replaces inheritance. You don't just need to know these concepts — you need to be able to trace execution through them step by step, the way the engine does.

The next chapter applies this foundation to UI development: the DOM, data binding, the virtual DOM, and why React's declarative model was a breakthrough.

---

> **Next:** [Chapter 0b: UI Development — The Hard Parts](./00b-ui-development-hard-parts.md) — How the DOM works, why data binding is hard, and the virtual DOM from scratch.
