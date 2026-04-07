# Part 0 — JavaScript & React Fundamentals

> The hard parts. What JavaScript actually does under the hood, how the DOM works, React from first principles, and TypeScript from zero to enterprise mastery.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| [0](./00-javascript-hard-parts.md) | JavaScript — The Hard Parts | 1703 | Beginner to Intermediate | Execution context, call stack, event loop, closures, promises, async/await, generators, prototypes, `this` |
| [0b](./00b-ui-development-hard-parts.md) | UI Development — The Hard Parts | 1498 | Beginner to Intermediate | DOM, CSSOM, data binding, virtual DOM, diffing, reconciliation, composition, declarative UI |
| [0c](./00c-react-fundamentals.md) | React — From First Principles | 1590 | Beginner to Intermediate | JSX, components, props, state, hooks, useState, useEffect, useRef, memoization, useReducer, context, custom hooks, React 19 |
| [0d](./00d-typescript-mastery.md) | TypeScript — From Fundamentals to Enterprise Mastery | 3296 | Beginner to Advanced | Type system, generics, unions, discriminated unions, type guards, utility types, conditional types, branded types, Zod, tsconfig |

## Reading Order

**0** (JavaScript) --> **0b** (UI Development) --> **0c** (React) --> **0d** (TypeScript)

Chapter 0 is the foundation everything else builds on. Chapter 0b applies those JS fundamentals to the DOM and UI frameworks. Chapter 0c connects both into React's actual API. Chapter 0d can be read after Chapter 0 independently, but benefits from the full sequence.

## Prerequisites

- Basic programming literacy (variables, functions, loops, conditionals)
- A text editor and Node.js installed
- No prior JavaScript framework experience required — Part 0 starts from scratch

## What You'll Be Able to Do After This Part

- Trace JavaScript execution step by step through the call stack, event loop, and microtask queue
- Explain exactly why a `setTimeout(..., 0)` callback runs after synchronous code
- Build a virtual DOM and reconciliation algorithm from scratch
- Understand every React hook as the closure it actually is
- Debug stale closures, infinite re-renders, and unnecessary renders with confidence
- Write TypeScript that makes impossible states unrepresentable
- Read and author advanced generic types, conditional types, and utility types
- Configure `tsconfig.json` for production applications and monorepos
