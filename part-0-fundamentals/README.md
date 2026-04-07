# Part 0 — JavaScript & React Fundamentals

> The hard parts. What JavaScript actually does under the hood, how the DOM works, React from first principles, and TypeScript from zero to enterprise mastery.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| 0 | [JavaScript — The Hard Parts](./00-javascript-hard-parts.md) | 1,715 | Beginner to Intermediate | Execution context, call stack, event loop, callback queue, microtask queue, closures, higher-order functions, prototypes, `this`, async/await, promises, generators, iterators |
| 0b | [UI Development — The Hard Parts](./00b-ui-development-hard-parts.md) | 1,510 | Beginner to Intermediate | DOM, CSSOM, WebIDL, data binding, one-way data binding, virtual DOM, diffing, reconciliation, composition, declarative UI |
| 0c | [React — From First Principles](./00c-react-fundamentals.md) | 1,602 | Beginner to Intermediate | JSX, components, props, state, hooks, useState, useEffect, useRef, useCallback, useMemo, useReducer, useContext, custom hooks, React 19 |
| 0d | [TypeScript — From Fundamentals to Enterprise Mastery](./00d-typescript-mastery.md) | 3,309 | Beginner to Advanced | Type system, inference, unions, intersections, discriminated unions, generics, type guards, utility types, conditional types, template literal types, branded types, Zod, tsconfig |

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
