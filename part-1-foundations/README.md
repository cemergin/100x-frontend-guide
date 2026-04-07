# Part I — Foundations

> How React Native actually works, how browsers render, how the build toolchain transforms your code, and how TypeScript scales.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|-------|-----------|------------|
| [1](./01-react-native-internals.md) | React Native Architecture & Internals | 461 | Intermediate to Advanced | New Architecture, JSI, Fabric, TurboModules, Hermes, threading model, bridgeless mode, cold start, rendering pipeline |
| [2](./02-browser-rendering.md) | Browser Rendering & Web Fundamentals | 1,859 | Intermediate | Box model, formatting context, reflow, repaint, composition layers, GPU, Core Web Vitals, LCP, INP, CLS |
| [3](./03-rendering-pipeline.md) | The Rendering Pipeline — Mobile & Web | 1,584 | Intermediate to Advanced | React reconciliation, Fiber, concurrent features, React Compiler, useTransition, useDeferredValue, Suspense |
| [4](./04-typescript-at-scale.md) | TypeScript at Scale | 1,530 | Intermediate to Advanced | Project references, barrel files, diagnostics, monorepo tsconfig, discriminated unions, branded types, Zod, ESLint boundaries |
| [5](./05-build-toolchain.md) | The Build Toolchain — From package.json to Running App | 2,646 | Intermediate to Advanced | Bundlers, transpilers, Babel, SWC, esbuild, Metro, Webpack, Vite, Turbopack, Rspack, Bun, ESM vs CJS, tree shaking, code splitting |

## Reading Order

1. **Chapter 1** and **Chapter 2** can be read in parallel -- one covers mobile internals, the other covers browser internals.
2. **Chapter 3** builds on both, unifying the React rendering pipeline across platforms. Read it after 1 and 2.
3. **Chapter 4** (TypeScript) is self-contained and can be read at any point.
4. **Chapter 5** (Build Toolchain) ties everything together -- how source code becomes a running app.

## Prerequisites

None. This part is the starting point for the guide.

## What You'll Be Able to Do After This Part

- Trace a React Native component from JSX to native pixels and explain every layer in between.
- Diagnose browser rendering bottlenecks by identifying which pipeline stage (layout, paint, composite) is the problem.
- Structure TypeScript projects that stay fast at 500k+ lines with clean dependency graphs.
- Read and write bundler configurations, understand tree shaking, code splitting, and source maps.
- Make informed architecture decisions grounded in how the platform actually works, not how it appears to work.
