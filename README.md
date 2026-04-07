<!--
  TYPE: index
  TITLE: The 100x Frontend Architect Guide
  CHAPTERS: 35
  PARTS: 7
  APPENDICES: 3
  UPDATED: 2026-04-07
  STRUCTURE: narrative-hierarchy
  AI_NOTE: Each chapter file contains an HTML comment metadata block with CHAPTER, TITLE, PART, PREREQS, KEY_TOPICS, DIFFICULTY, UPDATED fields. Use these for filtering, prerequisite graphs, and topic search. Files are organized into subdirectories by Part.
-->

# The 100x Frontend Architect Guide
### JavaScript, React, React Native, Expo, Vercel & Modern Frontend Mastery

A comprehensive, opinionated mega-guide covering every pattern, philosophy, and hard skill needed to become a frontend architect who ships production mobile and web apps that millions of people use — organized as a narrative that builds from first principles to deployment to operational excellence.

**35 chapters** · **7 parts + appendices** · **Read this to become a frontend beast.**

```
100x-frontend-guide/
├── part-0-fundamentals/             ← The Hard Parts: JavaScript engine, async, DOM, React from scratch
├── part-1-foundations/              ← Internals: RN architecture, rendering, browser, TypeScript at scale
├── part-2-react-native-expo/       ← Mobile: Expo, EAS, navigation, styling, animations, native modules
├── part-3-state-data-communication/← Data: state management, fetching, caching, real-time, offline-first
├── part-4-architecture-at-scale/   ← Scale: monorepos, design systems, microfrontends, testing, CI/CD
├── part-5-deployment-operations/   ← Ship: EAS deployments, monitoring, Firebase, metrics, security
├── part-6-vercel-web/              ← Web: Next.js, Vercel platform, AI SDK, edge, analytics
├── appendices/                     ← Reference: glossary, reading list, cheat sheets
└── README.md                       ← You are here
```

---

## Who This Guide Is For

You're a frontend engineer who's tired of building things that break in production, slow down at scale, and make your teammates' lives harder. Maybe you're the senior engineer who gets pulled into every performance fire drill. Maybe you're the tech lead who needs to make architecture decisions that'll outlast this quarter's sprint. Maybe you're the ambitious mid-level who wants to understand *why* things work, not just *how* to make them work.

This guide is for you if:
- You build with React Native and Expo and want to master the internals, not just the APIs
- You deploy to Vercel and want to understand the platform deeply, not just `git push`
- You want EAS builds, OTA updates, and app store submissions to be second nature
- You believe state management has a correct answer for each use case, and you want to know what it is
- You want your CI/CD pipeline to catch bugs before your users do
- You want to understand monitoring, metrics, and crash analysis well enough to maintain 99.9% crash-free rates
- You want to architect systems that make other engineers more productive, not just ship your own features

If you want to be the person your team calls when things are on fire — and the person who prevents fires in the first place — keep reading.

---

## Quick Start — Reading Paths

| Your Goal | Start Here | Then |
|-----------|-----------|------|
| **Learn from zero** | Part I: Ch 1 → 2 → 3 | Part II, then Part III |
| **Ship a mobile app fast** | Ch 6 (Expo), Ch 7 (EAS) | Ch 10 (state), Ch 22 (monitoring) |
| **Fix performance problems NOW** | Ch 3 (rendering), Ch 14 (performance) | Ch 15 (profiling), Ch 8 (lists) |
| **Architect a new project** | Ch 16 (monorepo), Ch 17 (design systems) | Ch 8 (navigation), Ch 10 (state) |
| **Level up to staff+** | Ch 20 (FE architecture), Ch 31 (beast mode) | Ch 19 (CI/CD), Ch 24 (security) |
| **Set up monitoring** | Ch 22 (Firebase/Sentry) | Ch 15 (profiling), Ch 24 (security) |
| **Master Vercel & Next.js** | Ch 27 (Vercel platform), Ch 28 (Next.js) | Ch 29 (AI SDK), Ch 30 (edge) |
| **Prepare for on-call** | Ch 31 (beast mode), Ch 22 (monitoring) | Ch 15 (profiling), Ch 14 (performance) |
| **Quick lookup** | [Glossary](./appendices/appendix-glossary.md) | [Cheat Sheet](./appendices/appendix-cheatsheet.md) |

---

## [Part 0 — JavaScript & React Fundamentals](./part-0-fundamentals/)
*The hard parts. What JavaScript actually does under the hood, how the DOM works, and React from first principles.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 0 | [JavaScript — The Hard Parts](./part-0-fundamentals/00-javascript-hard-parts.md) | Beg→Inter | Execution context, call stack, event loop, closures, promises under the hood, async/await, generators, microtask queue, prototypes |
| 0b | [UI Development — The Hard Parts](./part-0-fundamentals/00b-ui-development-hard-parts.md) | Beg→Inter | DOM as C++ (WebCore/WebIDL), data binding, one-way data flow, virtual DOM from scratch, diffing algorithm, declarative UI, UI composition |
| 0c | [React — From First Principles](./part-0-fundamentals/00c-react-fundamentals.md) | Beg→Inter | JSX compilation, components, props/state, hooks (useState/useEffect/useRef/useReducer/useContext), custom hooks, React 19 features, memoization |
| 0d | [TypeScript — From Fundamentals to Enterprise Mastery](./part-0-fundamentals/00d-typescript-mastery.md) | Beg→Adv | Type system, inference, unions/intersections, discriminated unions, generics, utility types, conditional types, Zod schemas, tRPC, tsconfig, project references, monorepo configs, enterprise patterns, type challenges |

> **Read order:** 0 → 0b → 0c → 0d (JavaScript engine → DOM/UI → React → TypeScript). If you're already comfortable with JS and React, start at 0d (TypeScript). If you're comfortable with all four, skip to Part I.

---

## [Part I — Foundations](./part-1-foundations/)
*How React Native actually works, how browsers render, how the build toolchain transforms your code, and how TypeScript scales.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 1 | [React Native Architecture & Internals](./part-1-foundations/01-react-native-internals.md) | Inter→Adv | New Architecture (JSI, Fabric, TurboModules), Hermes engine, threading model, bridgeless mode, cold start sequences |
| 2 | [Browser Rendering & Web Fundamentals](./part-1-foundations/02-browser-rendering.md) | Intermediate | Box model, formatting contexts, reflow/repaint, composition layers, GPU acceleration, Core Web Vitals, rendering pipeline |
| 3 | [The Rendering Pipeline — Mobile & Web](./part-1-foundations/03-rendering-pipeline.md) | Inter→Adv | React reconciliation, Fiber architecture, concurrent features, virtual DOM diffing, commit phase, React Compiler auto-memoization |
| 4 | [TypeScript at Scale](./part-1-foundations/04-typescript-at-scale.md) | Inter→Adv | Project references, composite builds, barrel file anti-patterns, diagnostics, monorepo tsconfig, discriminated unions, branded types, Zod schema inference |
| 5 | [The Build Toolchain — From package.json to Running App](./part-1-foundations/05-build-toolchain.md) | Inter→Adv | Bundlers (Metro, Webpack, Vite, Turbopack, Rspack), transpilers (Babel, SWC, esbuild), Bun, ESM vs CJS, tree shaking, code splitting, minification, source maps, HMR |

> **Read order:** 1 → 2 → 3 (core rendering understanding), then 4 and 5 independently.

---

## [Part II — React Native & Expo](./part-2-react-native-expo/)
*Building mobile apps that feel native, deploy cleanly, and don't crash.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 6 | [Expo: The Modern React Native Platform](./part-2-react-native-expo/05-expo-platform.md) | Beg→Inter | Expo SDK 54+, CNG (Continuous Native Generation), prebuild, config plugins, Expo Modules API, managed vs bare, React Compiler |
| 7 | [EAS: Build, Submit, Update](./part-2-react-native-expo/06-eas-mastery.md) | Inter→Adv | EAS Build (profiles, resource classes, hooks), EAS Submit (store automation), EAS Update (OTA, channels, fingerprinting), EAS Workflows, pricing |
| 8 | [Navigation Architecture](./part-2-react-native-expo/07-navigation.md) | Intermediate | Expo Router v4 (file-based routing, deep linking, type safety), React Navigation 7, navigation performance, lazy loading, prefetching |
| 9 | [Styling & Animation](./part-2-react-native-expo/08-styling-animation.md) | Intermediate | NativeWind/Tailwind, Tamagui, Unistyles, Reanimated v3 (worklets, shared values, layout animations), Gesture Handler, expo-image |

> **Read order:** 6 → 7 (Expo ecosystem), then 8 → 9 in any order.

---

## [Part III — State, Data & Communication](./part-3-state-data-communication/)
*Managing state without losing your mind, fetching data without losing your users.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 10 | [State Management at Scale](./part-3-state-data-communication/09-state-management.md) | Inter→Adv | Three-category model (server/client/form), Zustand, Jotai, Legend State, Redux when to keep/migrate, anti-patterns, finite states, type states, XState Store |
| 11 | [Data Fetching & Server Communication](./part-3-state-data-communication/10-data-fetching.md) | Inter→Adv | TanStack Query (caching, mutations, optimistic updates, infinite queries), tRPC, REST vs GraphQL, WebSockets, SSE, API layer architecture, retry/backoff |
| 12 | [Caching Strategies — Mobile & Web](./part-3-state-data-communication/11-caching.md) | Intermediate | MMKV vs AsyncStorage, TanStack Query cache, HTTP caching, CDN architecture, edge caching, Vercel Edge Config, Redis/Upstash, offline-first patterns |
| 13 | [Offline-First & Real-Time Patterns](./part-3-state-data-communication/12-offline-realtime.md) | Advanced | Legend State sync, conflict resolution, CRDT basics, WebSocket architecture, background sync, queue-based offline, optimistic UI with rollback |

> **Read order:** 10 → 11 (core), then 12 → 13 for advanced patterns.

---

## [Part IV — Architecture at Scale](./part-4-architecture-at-scale/)
*Monorepos, design systems, testing, and CI/CD that actually work.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 14 | [Performance Optimization — Mobile](./part-4-architecture-at-scale/13-mobile-performance.md) | Inter→Adv | Cold start optimization, FlashList, list virtualization, bundle size, memory management, React Compiler, re-render elimination, target metrics |
| 15 | [Profiling & Debugging](./part-4-architecture-at-scale/14-profiling-debugging.md) | Inter→Adv | Hermes profiling, Android Studio profiler, Xcode Instruments, Perfetto, Flipper alternatives, Reactotron, memory leak detection, crash analysis |
| 16 | [Monorepo Architecture](./part-4-architecture-at-scale/15-monorepo.md) | Inter→Adv | Turborepo (task pipeline, caching, partial rebuilds), pnpm workspaces, sharing code between RN and Next.js, next-forge, dependency management |
| 17 | [Design Systems & Component Libraries](./part-4-architecture-at-scale/16-design-systems.md) | Intermediate | Cross-platform tokens, shadcn/ui for web, Tamagui/NativeWind for mobile, Storybook, CVA variants, compound components, accessibility |
| 18 | [Testing Strategy](./part-4-architecture-at-scale/17-testing.md) | Intermediate | Testing trophy, Jest + RNTL, Maestro for E2E, Playwright for web, MSW for API mocking, testing TanStack Query, testing Zustand, visual regression |
| 19 | [CI/CD for Mobile & Web](./part-4-architecture-at-scale/18-cicd.md) | Inter→Adv | GitHub Actions patterns, EAS Build in CI, preview deployments, performance budgets, semantic versioning, Changesets, mobile+web from one monorepo |
| 20 | [Frontend Architecture Patterns](./part-4-architecture-at-scale/19-fe-architecture.md) | Advanced | Monoliths vs microfrontends, module federation, BFF pattern, island architecture, feature-based structure, architectural linting, ESLint boundaries |
| 21 | [Codebase Management, DX & Editor Mastery](./part-4-architecture-at-scale/20-codebase-management.md) | Beg→Inter | VS Code, extensions, keybindings, ESLint, Biome, Prettier, EditorConfig, git hooks, Husky, lint-staged, dependency updates, Renovate |

> **Read order:** 14 → 15 (performance), 16 → 17 → 18 → 19 (project structure), 20 (capstone), 21 (anytime).

---

## [Part V — Deployment & Operations](./part-5-deployment-operations/)
*Shipping, monitoring, and keeping your app alive at scale.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 22 | [Mobile Monitoring & Observability](./part-5-deployment-operations/20-monitoring.md) | Inter→Adv | Sentry vs Crashlytics, crash-free rate, ANR analysis, source maps, custom traces, real user monitoring, dashboards, alerts, maintaining 99.9% |
| 23 | [Firebase Console Mastery](./part-5-deployment-operations/21-firebase.md) | Intermediate | Crashlytics, Analytics (events, funnels, retention), Performance Monitoring, Remote Config, Cloud Messaging, App Distribution |
| 24 | [Security & Data Protection](./part-5-deployment-operations/22-security.md) | Intermediate | Secure storage, API key protection, certificate pinning, biometrics, OWASP Mobile Top 10, OAuth2 PKCE, token management, code obfuscation |
| 25 | [Mobile Metrics That Matter](./part-5-deployment-operations/23-metrics.md) | Intermediate | Cold start time, TTI, FPS/jank, memory usage, network latency, app size, Play Console vitals, App Store Connect, Datadog RUM |
| 26 | [Payments & Money Handling](./part-5-deployment-operations/24-payments.md) | Inter→Adv | Stripe (React Native + Next.js), PaymentIntents, subscriptions, Apple Pay, Google Pay, In-App Purchases, RevenueCat, webhooks, PCI compliance, testing |

> **Narrative:** How to monitor (Ch 22) → Firebase deep dive (Ch 23) → How to secure (Ch 24) → What to measure (Ch 25).

---

## [Part VI — Vercel & the Web](./part-6-vercel-web/)
*The web side of the frontend architect's toolkit.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 27 | [Vercel Platform Mastery](./part-6-vercel-web/24-vercel-platform.md) | Intermediate | Fluid Compute, Functions, Blob, Edge Config, AI Gateway, Queues, CLI, vercel.ts, Rolling Releases, Analytics, Speed Insights, BotID |
| 28 | [Next.js App Router — The Complete Picture](./part-6-vercel-web/25-nextjs-app-router.md) | Inter→Adv | Server/Client Components, Server Actions, Route Handlers, Streaming, ISR, PPR, caching (use cache/cacheLife/cacheTag), middleware, layouts |
| 29 | [The AI-Powered Frontend](./part-6-vercel-web/26-ai-frontend.md) | Intermediate | Vercel AI SDK v6, useChat/useCompletion, streaming, tool calling, structured output, AI Gateway multi-provider, building AI features in mobile+web |
| 30 | [Developer Experience & Tooling](./part-6-vercel-web/27-dx-tooling.md) | Beg→Inter | Biome, Turbopack, Metro, Expo Dev Client, React DevTools, Reactotron, Claude Code for FE, Storybook, pnpm vs bun, changeset |
| 31 | [Beast Mode — Frontend Operational Readiness](./part-6-vercel-web/28-beast-mode.md) | All levels | Day-one checklist, access matrix, dashboard setup, incident readiness, codebase navigation, tribal knowledge, the frontend architect's playbook |

> **Read order:** 27 → 28 (Vercel + Next.js), 29 → 30 (AI + DX), 31 (capstone).

---

## [Appendices](./appendices/)
*Reference material. Look things up as needed.*

| Ch | Title | What's Inside |
|----|-------|--------------|
| A | [Glossary](./appendices/appendix-glossary.md) | 200+ frontend terms — from ANR to zero-config, JSI to ISR |
| B | [Cheat Sheet](./appendices/appendix-cheatsheet.md) | One-page reference — stack decisions, commands, decision matrices |
| C | [Reading List](./appendices/appendix-reading-list.md) | 50+ essential articles, talks, and papers with summaries |

---

## Chapter Dependency Graph

```
Part I — Foundations (start here):
  Ch 1 → Ch 2 → Ch 3              (rendering understanding)
  Ch 4                              (TypeScript, standalone)
  Ch 5                              (Build Toolchain, standalone)

Part II — React Native & Expo (after Part I):
  Ch 6 → Ch 7                      (Expo + EAS pipeline)
  Ch 8, Ch 9                       (standalone, benefits from Ch 1)

Part III — State & Data (after Part I, any order):
  Ch 10 → Ch 11                    (state + data core)
  Ch 12 → Ch 13                    (caching + offline, advanced)

Part IV — Architecture at Scale (after Parts I-III):
  Ch 14 → Ch 15                    (performance + profiling)
  Ch 16 → Ch 17 → Ch 18 → Ch 19   (project structure pipeline)
  Ch 20                             (capstone, benefits from all)
  Ch 21                             (codebase management, anytime)

Part V — Deployment & Operations (after Part II):
  Ch 22 → Ch 23                    (monitoring + Firebase)
  Ch 24, Ch 25                     (standalone)
  Ch 26                             (payments, standalone)

Part VI — Vercel & Web (anytime after Part I):
  Ch 27 → Ch 28                    (Vercel + Next.js)
  Ch 29, Ch 30                     (standalone)
  Ch 31                             (capstone, benefits from all)

Appendices — anytime:
  A, B, C
```

---

## How Each Chapter Is Structured

Every chapter follows a consistent, AI-scannable format:

```
<!-- HTML metadata: CHAPTER, TITLE, PART, PREREQS, KEY_TOPICS, DIFFICULTY, UPDATED -->

# Chapter N: Title

> Part · Prerequisites · Difficulty

Opening narrative — why this matters, a real-world story.

### In This Chapter        ← section index
### Related Chapters       ← cross-references

---

## 1. MAJOR SECTION
### 1.1 Subsection
**What it is:** ...
**When to use:** ...
**Trade-offs:** ...
**Real-world example:** ...
**Code example:** ...
```

---

## Production Stats

Real numbers from production apps referenced across these guides:

| Company | Metric | Value | Guide |
|---------|--------|-------|-------|
| Shopify | Crash-free sessions | 99.9% | [Monitoring](./part-5-deployment-operations/20-monitoring.md) |
| Shopify | Screen load P75 | <500ms | [Performance](./part-4-architecture-at-scale/13-mobile-performance.md) |
| Shopify | Bundle size reduction | 78% (50→11MB) | [Performance](./part-4-architecture-at-scale/13-mobile-performance.md) |
| Discord | TTI improvement (iPhone 6) | -3,500ms | [Profiling](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| Discord | Message parse optimization | 90% faster | [Profiling](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| Discord | Memory reduction (lists) | -14% | [Performance](./part-4-architecture-at-scale/13-mobile-performance.md) |
| A Million Monkeys | Android cold start | -24% | [RN Internals](./part-1-foundations/01-react-native-internals.md) |
| Moment→Day.js | Size reduction | 99% (232→2KB) | [Performance](./part-4-architecture-at-scale/13-mobile-performance.md) |

---

## Concept Cross-Index

Find where each concept is covered across all guides:

| Concept | Primary Guide | Also Covered In |
|---------|--------------|----------------|
| **JSI (JavaScript Interface)** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **Fabric Renderer** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **TurboModules** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Hermes Engine** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **New Architecture (Bridgeless)** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Cold Start Optimization** | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) | [Ch 1](./part-1-foundations/01-react-native-internals.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |
| **Threading Model (RN)** | [Ch 1](./part-1-foundations/01-react-native-internals.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 9](./part-2-react-native-expo/08-styling-animation.md) |
| **CNG (Continuous Native Generation)** | [Ch 6](./part-2-react-native-expo/05-expo-platform.md) | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md), [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **Expo Prebuild** | [Ch 6](./part-2-react-native-expo/05-expo-platform.md) | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md), [Ch 19](./part-4-architecture-at-scale/18-cicd.md) |
| **Config Plugins** | [Ch 6](./part-2-react-native-expo/05-expo-platform.md) | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md), [Ch 24](./part-5-deployment-operations/22-security.md) |
| **EAS Build** | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md), [Ch 6](./part-2-react-native-expo/05-expo-platform.md) |
| **EAS Update (OTA)** | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md), [Ch 22](./part-5-deployment-operations/20-monitoring.md) |
| **Channels & Branches (EAS)** | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md) |
| **Fingerprint (EAS)** | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md) |
| **Error Recovery (OTA)** | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) | [Ch 22](./part-5-deployment-operations/20-monitoring.md) |
| **React Server Components** | [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 0c](./part-0-fundamentals/00c-react-fundamentals.md) |
| **Hooks (useState, useEffect, etc.)** | [Ch 0c](./part-0-fundamentals/00c-react-fundamentals.md) | [Ch 10](./part-3-state-data-communication/09-state-management.md), [Ch 3](./part-1-foundations/03-rendering-pipeline.md) |
| **Fiber Architecture** | [Ch 3](./part-1-foundations/03-rendering-pipeline.md) | [Ch 1](./part-1-foundations/01-react-native-internals.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Reconciliation & Diffing** | [Ch 3](./part-1-foundations/03-rendering-pipeline.md) | [Ch 0b](./part-0-fundamentals/00b-ui-development-hard-parts.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **React Compiler** | [Ch 3](./part-1-foundations/03-rendering-pipeline.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Suspense & Streaming** | [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 11](./part-3-state-data-communication/10-data-fetching.md) |
| **Discriminated Unions** | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md) | [Ch 4](./part-1-foundations/04-typescript-at-scale.md), [Ch 10](./part-3-state-data-communication/09-state-management.md) |
| **Generics (TypeScript)** | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md) | [Ch 4](./part-1-foundations/04-typescript-at-scale.md), [Ch 11](./part-3-state-data-communication/10-data-fetching.md) |
| **Branded Types** | [Ch 4](./part-1-foundations/04-typescript-at-scale.md) | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md) |
| **Zod Schema Validation** | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md) | [Ch 4](./part-1-foundations/04-typescript-at-scale.md), [Ch 11](./part-3-state-data-communication/10-data-fetching.md) |
| **tRPC** | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md), [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **Project References (TS)** | [Ch 4](./part-1-foundations/04-typescript-at-scale.md) | [Ch 0d](./part-0-fundamentals/00d-typescript-mastery.md), [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **Zustand** | [Ch 10](./part-3-state-data-communication/09-state-management.md) | [Ch 18](./part-4-architecture-at-scale/17-testing.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **TanStack Query** | [Ch 11](./part-3-state-data-communication/10-data-fetching.md) | [Ch 10](./part-3-state-data-communication/09-state-management.md), [Ch 18](./part-4-architecture-at-scale/17-testing.md) |
| **Jotai** | [Ch 10](./part-3-state-data-communication/09-state-management.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **MMKV** | [Ch 12](./part-3-state-data-communication/11-caching.md) | [Ch 10](./part-3-state-data-communication/09-state-management.md), [Ch 24](./part-5-deployment-operations/22-security.md) |
| **Optimistic Updates** | [Ch 11](./part-3-state-data-communication/10-data-fetching.md) | [Ch 13](./part-3-state-data-communication/12-offline-realtime.md) |
| **Three-Category State Model** | [Ch 10](./part-3-state-data-communication/09-state-management.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md), [Appendix B](./appendices/appendix-cheatsheet.md) |
| **FlashList** | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) | [Ch 9](./part-2-react-native-expo/08-styling-animation.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **Reanimated** | [Ch 9](./part-2-react-native-expo/08-styling-animation.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md), [Ch 8](./part-2-react-native-expo/07-navigation.md) |
| **Bundle Size Optimization** | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) | [Ch 5](./part-1-foundations/05-build-toolchain.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |
| **Memory Leaks** | [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |
| **FPS & Jank** | [Ch 25](./part-5-deployment-operations/23-metrics.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **Monorepo (Turborepo)** | [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md), [Ch 4](./part-1-foundations/04-typescript-at-scale.md) |
| **Microfrontends** | [Ch 20](./part-4-architecture-at-scale/19-fe-architecture.md) | [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **BFF (Backend for Frontend)** | [Ch 20](./part-4-architecture-at-scale/19-fe-architecture.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Design Systems** | [Ch 17](./part-4-architecture-at-scale/16-design-systems.md) | [Ch 9](./part-2-react-native-expo/08-styling-animation.md), [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **Feature-Based Structure** | [Ch 20](./part-4-architecture-at-scale/19-fe-architecture.md) | [Ch 16](./part-4-architecture-at-scale/15-monorepo.md), [Ch 8](./part-2-react-native-expo/07-navigation.md) |
| **Vercel Platform** | [Ch 27](./part-6-vercel-web/24-vercel-platform.md) | [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md), [Ch 19](./part-4-architecture-at-scale/18-cicd.md) |
| **Next.js App Router** | [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) | [Ch 27](./part-6-vercel-web/24-vercel-platform.md), [Ch 20](./part-4-architecture-at-scale/19-fe-architecture.md) |
| **Self-Hosting (AWS)** | [Ch 20](./part-4-architecture-at-scale/19-fe-architecture.md) | [Ch 27](./part-6-vercel-web/24-vercel-platform.md) |
| **CI/CD & GitHub Actions** | [Ch 19](./part-4-architecture-at-scale/18-cicd.md) | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md), [Ch 16](./part-4-architecture-at-scale/15-monorepo.md) |
| **Sentry** | [Ch 22](./part-5-deployment-operations/20-monitoring.md) | [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md), [Ch 31](./part-6-vercel-web/28-beast-mode.md) |
| **Crashlytics** | [Ch 22](./part-5-deployment-operations/20-monitoring.md) | [Ch 23](./part-5-deployment-operations/21-firebase.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |
| **Crash-Free Rate** | [Ch 22](./part-5-deployment-operations/20-monitoring.md) | [Ch 25](./part-5-deployment-operations/23-metrics.md), [Ch 31](./part-6-vercel-web/28-beast-mode.md) |
| **ANR (App Not Responding)** | [Ch 22](./part-5-deployment-operations/20-monitoring.md) | [Ch 25](./part-5-deployment-operations/23-metrics.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **Source Maps** | [Ch 22](./part-5-deployment-operations/20-monitoring.md) | [Ch 5](./part-1-foundations/05-build-toolchain.md), [Ch 15](./part-4-architecture-at-scale/14-profiling-debugging.md) |
| **OWASP Mobile Top 10** | [Ch 24](./part-5-deployment-operations/22-security.md) | [Ch 26](./part-5-deployment-operations/24-payments.md) |
| **OAuth2 PKCE** | [Ch 24](./part-5-deployment-operations/22-security.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md) |
| **Certificate Pinning** | [Ch 24](./part-5-deployment-operations/22-security.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md) |
| **Secure Storage** | [Ch 24](./part-5-deployment-operations/22-security.md) | [Ch 12](./part-3-state-data-communication/11-caching.md), [Ch 26](./part-5-deployment-operations/24-payments.md) |
| **Stripe (PaymentIntents)** | [Ch 26](./part-5-deployment-operations/24-payments.md) | [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **In-App Purchases (IAP)** | [Ch 26](./part-5-deployment-operations/24-payments.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md) |
| **RevenueCat** | [Ch 26](./part-5-deployment-operations/24-payments.md) | [Ch 7](./part-2-react-native-expo/06-eas-mastery.md) |
| **Webhooks** | [Ch 26](./part-5-deployment-operations/24-payments.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Event Loop & Microtasks** | [Ch 0](./part-0-fundamentals/00-javascript-hard-parts.md) | [Ch 1](./part-1-foundations/01-react-native-internals.md), [Ch 3](./part-1-foundations/03-rendering-pipeline.md) |
| **Closures & Prototypes** | [Ch 0](./part-0-fundamentals/00-javascript-hard-parts.md) | [Ch 0c](./part-0-fundamentals/00c-react-fundamentals.md) |
| **Virtual DOM & Diffing** | [Ch 0b](./part-0-fundamentals/00b-ui-development-hard-parts.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Bundlers (Metro, Vite, Turbopack)** | [Ch 5](./part-1-foundations/05-build-toolchain.md) | [Ch 16](./part-4-architecture-at-scale/15-monorepo.md), [Ch 30](./part-6-vercel-web/27-dx-tooling.md) |
| **Tree Shaking & Code Splitting** | [Ch 5](./part-1-foundations/05-build-toolchain.md) | [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Expo Router (File-Based Routing)** | [Ch 8](./part-2-react-native-expo/07-navigation.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Deep Linking** | [Ch 8](./part-2-react-native-expo/07-navigation.md) | [Ch 6](./part-2-react-native-expo/05-expo-platform.md), [Ch 24](./part-5-deployment-operations/22-security.md) |
| **NativeWind / Tailwind** | [Ch 9](./part-2-react-native-expo/08-styling-animation.md) | [Ch 17](./part-4-architecture-at-scale/16-design-systems.md) |
| **Offline-First & CRDTs** | [Ch 13](./part-3-state-data-communication/12-offline-realtime.md) | [Ch 12](./part-3-state-data-communication/11-caching.md), [Ch 10](./part-3-state-data-communication/09-state-management.md) |
| **WebSockets & Real-Time** | [Ch 13](./part-3-state-data-communication/12-offline-realtime.md) | [Ch 11](./part-3-state-data-communication/10-data-fetching.md), [Ch 29](./part-6-vercel-web/26-ai-frontend.md) |
| **Testing (Jest, Maestro, Playwright)** | [Ch 18](./part-4-architecture-at-scale/17-testing.md) | [Ch 19](./part-4-architecture-at-scale/18-cicd.md), [Ch 10](./part-3-state-data-communication/09-state-management.md) |
| **AI SDK & AI Features** | [Ch 29](./part-6-vercel-web/26-ai-frontend.md) | [Ch 27](./part-6-vercel-web/24-vercel-platform.md), [Ch 30](./part-6-vercel-web/27-dx-tooling.md) |
| **Edge Computing & CDN** | [Ch 12](./part-3-state-data-communication/11-caching.md) | [Ch 27](./part-6-vercel-web/24-vercel-platform.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Firebase (Analytics, Remote Config)** | [Ch 23](./part-5-deployment-operations/21-firebase.md) | [Ch 22](./part-5-deployment-operations/20-monitoring.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |
| **Core Web Vitals** | [Ch 2](./part-1-foundations/02-browser-rendering.md) | [Ch 25](./part-5-deployment-operations/23-metrics.md), [Ch 28](./part-6-vercel-web/25-nextjs-app-router.md) |
| **Browser Rendering Pipeline** | [Ch 2](./part-1-foundations/02-browser-rendering.md) | [Ch 3](./part-1-foundations/03-rendering-pipeline.md), [Ch 14](./part-4-architecture-at-scale/13-mobile-performance.md) |
| **Beast Mode (Operational Readiness)** | [Ch 31](./part-6-vercel-web/28-beast-mode.md) | [Ch 22](./part-5-deployment-operations/20-monitoring.md), [Ch 25](./part-5-deployment-operations/23-metrics.md) |

---

## Interactive Learning with AI

The [`course/`](./course/) directory contains **60 hands-on modules** organized in 3 progressive loops that build on this guide:

| Loop | Modules | Focus | TicketFlow Arc |
|------|---------|-------|----------------|
| **Loop 1: Foundation** | L1-M01 to L1-M20 | Core mental models, heavy scaffolding | Empty folder to simple Expo app with basic screens, navigation, and first EAS build |
| **Loop 2: Practice** | L2-M01 to L2-M20 | Intermediate patterns, moderate scaffolding | Basic app to full-featured app with API integration, state management, testing, CI/CD, and monitoring |
| **Loop 3: Mastery** | L3-M01 to L3-M20 | Architect-level decisions, minimal scaffolding | Single app to production-ready monorepo with web dashboard, design system, Vercel deployment, AI features, and operational readiness |

**Running project:** Throughout all 60 modules, you build **TicketFlow** -- an event discovery and ticketing app (think Resident Advisor meets Eventbrite) with React Native (Expo) for mobile and a Next.js web dashboard, deployed via EAS and Vercel from a Turborepo monorepo.

**How to start:** Use the [tech-skill-builder](https://github.com/anthropics/claude-code-plugins) plugin for Claude Code:
```bash
# Start from the beginning
/learn course/course.yaml

# Jump to a specific module
/learn course/course.yaml --module L2-M05
```

---

## Key Principles

1. **The best architecture is the simplest one that solves actual problems** — don't microfrontend a two-person team's codebase
2. **Separate your state into three categories or suffer the consequences** — server, client, form. Mixing them is the #1 architecture mistake
3. **Measure before optimizing** — your intuition about what's slow is wrong until Hermes tells you otherwise
4. **Ship the smallest possible binary** — every MB you add is a user who didn't download your app on cellular
5. **Monitor before you need to** — set up Crashlytics on day one, not after your first production crash
6. **Make other engineers faster** — the frontend architect's job isn't just to ship features, it's to build the platform that lets everyone else ship features
7. **Composition over invention** — compose TanStack Query + Zustand + MMKV rather than building a custom state layer from scratch

---

*Built with [Claude Code](https://claude.ai/claude-code). Contributions welcome.*
