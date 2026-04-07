<!--
  TYPE: index
  TITLE: The 100x Frontend Architect Guide
  CHAPTERS: 31
  PARTS: 7
  APPENDICES: 3
  UPDATED: 2026-04-07
  STRUCTURE: narrative-hierarchy
  AI_NOTE: Each chapter file contains an HTML comment metadata block with CHAPTER, TITLE, PART, PREREQS, KEY_TOPICS, DIFFICULTY, UPDATED fields. Use these for filtering, prerequisite graphs, and topic search. Files are organized into subdirectories by Part.
-->

# The 100x Frontend Architect Guide
### JavaScript, React, React Native, Expo, Vercel & Modern Frontend Mastery

A comprehensive, opinionated mega-guide covering every pattern, philosophy, and hard skill needed to become a frontend architect who ships production mobile and web apps that millions of people use — organized as a narrative that builds from first principles to deployment to operational excellence.

**31 chapters** · **7 parts + appendices** · **Read this to become a frontend beast.**

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
| **Ship a mobile app fast** | Ch 4 (Expo), Ch 5 (EAS) | Ch 8 (state), Ch 13 (monitoring) |
| **Fix performance problems NOW** | Ch 3 (rendering), Ch 11 (performance) | Ch 12 (profiling), Ch 6 (lists) |
| **Architect a new project** | Ch 15 (monorepo), Ch 16 (design systems) | Ch 7 (navigation), Ch 8 (state) |
| **Level up to staff+** | Ch 18 (FE architecture), Ch 24 (beast mode) | Ch 17 (CI/CD), Ch 14 (security) |
| **Set up monitoring** | Ch 13 (Firebase/Sentry) | Ch 12 (profiling), Ch 14 (security) |
| **Master Vercel & Next.js** | Ch 19 (Vercel platform), Ch 20 (Next.js) | Ch 21 (AI SDK), Ch 22 (edge) |
| **Prepare for on-call** | Ch 24 (beast mode), Ch 13 (monitoring) | Ch 12 (profiling), Ch 11 (performance) |
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
| 5 | [Expo: The Modern React Native Platform](./part-2-react-native-expo/05-expo-platform.md) | Beg→Inter | Expo SDK 54+, CNG (Continuous Native Generation), prebuild, config plugins, Expo Modules API, managed vs bare, React Compiler |
| 6 | [EAS: Build, Submit, Update](./part-2-react-native-expo/06-eas-mastery.md) | Inter→Adv | EAS Build (profiles, resource classes, hooks), EAS Submit (store automation), EAS Update (OTA, channels, fingerprinting), EAS Workflows, pricing |
| 7 | [Navigation Architecture](./part-2-react-native-expo/07-navigation.md) | Intermediate | Expo Router v4 (file-based routing, deep linking, type safety), React Navigation 7, navigation performance, lazy loading, prefetching |
| 8 | [Styling & Animation](./part-2-react-native-expo/08-styling-animation.md) | Intermediate | NativeWind/Tailwind, Tamagui, Unistyles, Reanimated v3 (worklets, shared values, layout animations), Gesture Handler, expo-image |

> **Read order:** 5 → 6 (Expo ecosystem), then 7 → 8 in any order.

---

## [Part III — State, Data & Communication](./part-3-state-data-communication/)
*Managing state without losing your mind, fetching data without losing your users.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 9 | [State Management at Scale](./part-3-state-data-communication/09-state-management.md) | Inter→Adv | Three-category model (server/client/form), Zustand, Jotai, Legend State, Redux when to keep/migrate, anti-patterns, finite states, type states, XState Store |
| 10 | [Data Fetching & Server Communication](./part-3-state-data-communication/10-data-fetching.md) | Inter→Adv | TanStack Query (caching, mutations, optimistic updates, infinite queries), tRPC, REST vs GraphQL, WebSockets, SSE, API layer architecture, retry/backoff |
| 11 | [Caching Strategies — Mobile & Web](./part-3-state-data-communication/11-caching.md) | Intermediate | MMKV vs AsyncStorage, TanStack Query cache, HTTP caching, CDN architecture, edge caching, Vercel Edge Config, Redis/Upstash, offline-first patterns |
| 12 | [Offline-First & Real-Time Patterns](./part-3-state-data-communication/12-offline-realtime.md) | Advanced | Legend State sync, conflict resolution, CRDT basics, WebSocket architecture, background sync, queue-based offline, optimistic UI with rollback |

> **Read order:** 9 → 10 (core), then 11 → 12 for advanced patterns.

---

## [Part IV — Architecture at Scale](./part-4-architecture-at-scale/)
*Monorepos, design systems, testing, and CI/CD that actually work.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 13 | [Performance Optimization — Mobile](./part-4-architecture-at-scale/13-mobile-performance.md) | Inter→Adv | Cold start optimization, FlashList, list virtualization, bundle size, memory management, React Compiler, re-render elimination, target metrics |
| 14 | [Profiling & Debugging](./part-4-architecture-at-scale/14-profiling-debugging.md) | Inter→Adv | Hermes profiling, Android Studio profiler, Xcode Instruments, Perfetto, Flipper alternatives, Reactotron, memory leak detection, crash analysis |
| 15 | [Monorepo Architecture](./part-4-architecture-at-scale/15-monorepo.md) | Inter→Adv | Turborepo (task pipeline, caching, partial rebuilds), pnpm workspaces, sharing code between RN and Next.js, next-forge, dependency management |
| 16 | [Design Systems & Component Libraries](./part-4-architecture-at-scale/16-design-systems.md) | Intermediate | Cross-platform tokens, shadcn/ui for web, Tamagui/NativeWind for mobile, Storybook, CVA variants, compound components, accessibility |
| 17 | [Testing Strategy](./part-4-architecture-at-scale/17-testing.md) | Intermediate | Testing trophy, Jest + RNTL, Maestro for E2E, Playwright for web, MSW for API mocking, testing TanStack Query, testing Zustand, visual regression |
| 18 | [CI/CD for Mobile & Web](./part-4-architecture-at-scale/18-cicd.md) | Inter→Adv | GitHub Actions patterns, EAS Build in CI, preview deployments, performance budgets, semantic versioning, Changesets, mobile+web from one monorepo |
| 19 | [Frontend Architecture Patterns](./part-4-architecture-at-scale/19-fe-architecture.md) | Advanced | Monoliths vs microfrontends, module federation, BFF pattern, island architecture, feature-based structure, architectural linting, ESLint boundaries |

> **Read order:** 13 → 14 (performance), 15 → 16 → 17 → 18 (project structure), 19 (capstone).

---

## [Part V — Deployment & Operations](./part-5-deployment-operations/)
*Shipping, monitoring, and keeping your app alive at scale.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 20 | [Mobile Monitoring & Observability](./part-5-deployment-operations/20-monitoring.md) | Inter→Adv | Sentry vs Crashlytics, crash-free rate, ANR analysis, source maps, custom traces, real user monitoring, dashboards, alerts, maintaining 99.9% |
| 21 | [Firebase Console Mastery](./part-5-deployment-operations/21-firebase.md) | Intermediate | Crashlytics, Analytics (events, funnels, retention), Performance Monitoring, Remote Config, Cloud Messaging, App Distribution |
| 22 | [Security & Data Protection](./part-5-deployment-operations/22-security.md) | Intermediate | Secure storage, API key protection, certificate pinning, biometrics, OWASP Mobile Top 10, OAuth2 PKCE, token management, code obfuscation |
| 23 | [Mobile Metrics That Matter](./part-5-deployment-operations/23-metrics.md) | Intermediate | Cold start time, TTI, FPS/jank, memory usage, network latency, app size, Play Console vitals, App Store Connect, Datadog RUM |
| 24 | [Payments & Money Handling](./part-5-deployment-operations/24-payments.md) | Inter→Adv | Stripe (React Native + Next.js), PaymentIntents, subscriptions, Apple Pay, Google Pay, In-App Purchases, RevenueCat, webhooks, PCI compliance, testing |

> **Narrative:** How to monitor (Ch 20) → Firebase deep dive (Ch 21) → How to secure (Ch 22) → What to measure (Ch 23).

---

## [Part VI — Vercel & the Web](./part-6-vercel-web/)
*The web side of the frontend architect's toolkit.*

| Ch | Title | Difficulty | What You'll Learn |
|----|-------|-----------|-------------------|
| 24 | [Vercel Platform Mastery](./part-6-vercel-web/24-vercel-platform.md) | Intermediate | Fluid Compute, Functions, Blob, Edge Config, AI Gateway, Queues, CLI, vercel.ts, Rolling Releases, Analytics, Speed Insights, BotID |
| 25 | [Next.js App Router — The Complete Picture](./part-6-vercel-web/25-nextjs-app-router.md) | Inter→Adv | Server/Client Components, Server Actions, Route Handlers, Streaming, ISR, PPR, caching (use cache/cacheLife/cacheTag), middleware, layouts |
| 26 | [The AI-Powered Frontend](./part-6-vercel-web/26-ai-frontend.md) | Intermediate | Vercel AI SDK v6, useChat/useCompletion, streaming, tool calling, structured output, AI Gateway multi-provider, building AI features in mobile+web |
| 27 | [Developer Experience & Tooling](./part-6-vercel-web/27-dx-tooling.md) | Beg→Inter | Biome, Turbopack, Metro, Expo Dev Client, React DevTools, Reactotron, Claude Code for FE, Storybook, pnpm vs bun, changeset |
| 28 | [Beast Mode — Frontend Operational Readiness](./part-6-vercel-web/28-beast-mode.md) | All levels | Day-one checklist, access matrix, dashboard setup, incident readiness, codebase navigation, tribal knowledge, the frontend architect's playbook |

> **Read order:** 24 → 25 (Vercel + Next.js), 26 → 27 (AI + DX), 28 (capstone).

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

Part II — React Native & Expo (after Part I):
  Ch 5 → Ch 6                      (Expo + EAS pipeline)
  Ch 7, Ch 8                       (standalone, benefits from Ch 1)

Part III — State & Data (after Part I, any order):
  Ch 9 → Ch 10                     (state + data core)
  Ch 11 → Ch 12                    (caching + offline, advanced)

Part IV — Architecture at Scale (after Parts I-III):
  Ch 13 → Ch 14                    (performance + profiling)
  Ch 15 → Ch 16 → Ch 17 → Ch 18   (project structure pipeline)
  Ch 19                             (capstone, benefits from all)

Part V — Deployment & Operations (after Part II):
  Ch 20 → Ch 21                    (monitoring + Firebase)
  Ch 22, Ch 23                     (standalone)

Part VI — Vercel & Web (anytime after Part I):
  Ch 24 → Ch 25                    (Vercel + Next.js)
  Ch 26, Ch 27                     (standalone)
  Ch 28                             (capstone, benefits from all)

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
