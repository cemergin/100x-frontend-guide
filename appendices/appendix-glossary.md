<!--
  TYPE: appendix
  TITLE: Glossary
  APPENDIX: A
  UPDATED: 2026-04-07
-->

# Appendix A: Glossary

> 200+ frontend engineering terms — from ANR to zero-config.

Cross-references point to the chapter where the concept is covered in depth. Terms are grouped alphabetically for quick lookup.

---

## A

**Abstract Syntax Tree (AST)** — Tree representation of source code structure produced by a parser. Babel, ESLint, and TypeScript all operate on ASTs. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Accessibility (a11y)** — Practices ensuring apps are usable by people with disabilities. Includes screen readers, semantic markup, ARIA roles, and contrast ratios. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**ANR (App Not Responding)** — Android system dialog triggered when the main thread is blocked for 5+ seconds. A critical mobile vitals metric. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

**API Route** — Server-side endpoint defined inside a Next.js app. In App Router, these are `route.ts` files. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**App Store Connect** — Apple's portal for managing iOS app submissions, TestFlight builds, and metadata. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Atomic Design** — Design methodology (atoms, molecules, organisms, templates, pages) for building component libraries systematically. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Auto-linking** — React Native mechanism that automatically links native dependencies without manual `pod install` or Gradle configuration. Expo handles this via CNG. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

---

## B

**Babel** — JavaScript transpiler that converts modern JS/TS/JSX into backwards-compatible code. Being replaced by SWC and Hermes in many RN workflows. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Barrel File** — An `index.ts` that re-exports from multiple modules. Convenient but can destroy tree shaking if not configured carefully. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Bitcode** — Intermediate representation used by Apple's compiler. Deprecated since Xcode 14 but still referenced in legacy build configs. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Branded Type** — TypeScript pattern using intersection with a unique symbol to create nominally distinct types from structural equivalents (e.g., `UserId` vs `OrderId` both being strings). See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Branch (EAS Update)** — A named stream of OTA updates in EAS. Maps to a release channel. A branch receives updates and is pointed to by a channel. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Bridge (React Native)** — The old architecture's async JSON serialization layer between JS and native threads. Replaced by JSI in the New Architecture. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Bridgeless Mode** — React Native configuration that removes the Bridge entirely, requiring all modules to use JSI/TurboModules. The final evolution of the New Architecture. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Build Profile (EAS)** — Named configuration in `eas.json` defining how a build is created (development, preview, production). Controls signing, environment variables, and native build settings. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Bundler** — Tool that combines source files into optimized output bundles. Metro (RN), Webpack, Turbopack, Vite, esbuild. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

---

## C

**Cache-Control** — HTTP header controlling how responses are cached by browsers, CDNs, and proxies. Critical for ISR and static assets. See [Ch 11](../part-3-state-data-communication/11-caching.md).

**Certificate Pinning** — Security technique that hardcodes expected server certificates in the app, preventing MITM attacks even if a CA is compromised. See [Ch 22](../part-5-deployment-operations/22-security.md).

**Channel (EAS Update)** — A pointer to a branch in EAS Update. Apps are built pointing to a channel; you control which branch a channel points to. Enables staged rollouts. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Changesets** — Versioning tool for monorepos. Developers add changeset files describing their changes; CI aggregates them into version bumps and changelogs. See [Ch 18](../part-4-architecture-at-scale/18-cicd.md).

**Client Component** — React component that runs in the browser (marked with `"use client"` in Next.js App Router). Has access to state, effects, and browser APIs. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**CLS (Cumulative Layout Shift)** — Core Web Vital measuring visual stability. Sum of all unexpected layout shifts during page lifetime. Target: < 0.1. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**CNG (Continuous Native Generation)** — Expo's approach where native `ios/` and `android/` directories are generated from `app.json` and config plugins at build time, not checked into source control. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Code Splitting** — Technique of breaking a bundle into smaller chunks loaded on demand. Reduces initial load time. `React.lazy()` and dynamic `import()`. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Codegen (React Native)** — Build-time code generation from TypeScript/Flow specs that creates type-safe C++ bindings for TurboModules and Fabric components. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Cold Start** — The full startup sequence of a React Native app: native init → JS engine init → bundle load → React render → native mount. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Composition Layer** — GPU-accelerated layer created by the browser compositor for elements with transforms, opacity, or `will-change`. Avoids reflow/repaint. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Conditional Type** — TypeScript type that selects between two types based on a condition: `T extends U ? X : Y`. Enables type-level programming. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Config Plugin (Expo)** — JavaScript function that modifies native project files (Info.plist, AndroidManifest.xml, Podfile, etc.) during prebuild. The mechanism for customizing native code in CNG. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Core Web Vitals** — Google's three key metrics: LCP (loading), INP (interactivity), CLS (visual stability). Affect search ranking and user experience. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Crash-Free Rate** — Percentage of user sessions without a crash. Industry target: 99.9%+. Primary mobile reliability metric. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

**CSSOM (CSS Object Model)** — Browser's internal representation of CSS rules. Combined with DOM to produce the render tree. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Custom Hook** — A JavaScript function starting with `use` that encapsulates reusable stateful logic by composing built-in hooks. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

---

## D

**Dead Code Elimination** — Compiler optimization that removes code proven unreachable. Related to but distinct from tree shaking. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Debounce** — Technique that delays function execution until a pause in invocations. Used for search inputs, resize handlers. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Deep Linking** — Opening a specific screen in a mobile app via a URL. Requires navigation configuration and native URL scheme/universal link setup. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**Design Token** — Named value (color, spacing, font size) that represents a design decision. The atomic unit of a design system. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Discriminated Union** — TypeScript pattern where union members share a literal-typed discriminant field, enabling exhaustive `switch` statements with type narrowing. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**DOM (Document Object Model)** — Tree of C++ objects representing HTML elements, exposed to JavaScript via WebIDL bindings. See [Ch 0b](../part-0-fundamentals/00b-ui-development-hard-parts.md).

**dSYM (Debug Symbol File)** — macOS/iOS file containing debug symbols for symbolication of crash reports. Must be uploaded to Sentry/Crashlytics for readable stack traces. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

**Dynamic Import** — `import()` expression that loads a module at runtime, returning a Promise. The mechanism behind code splitting. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

---

## E

**EAS Build** — Expo Application Services cloud build infrastructure. Compiles native iOS/Android binaries from your project without local Xcode/Android Studio. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**EAS Submit** — Expo service that uploads built binaries to App Store Connect and Google Play Console. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**EAS Update** — Expo's OTA (over-the-air) update system. Pushes JS bundle changes to users without app store review. Uses branches and channels for staged rollouts. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Edge Config (Vercel)** — Ultra-low-latency (< 1ms) key-value data store read at the edge. Used for feature flags, A/B tests, and redirects. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Edge Function** — Serverless function that runs at CDN edge locations (close to users). Limited runtime but extremely low latency. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Edge Runtime** — Lightweight V8 isolate-based runtime available in Vercel/Next.js. No Node.js APIs, but fast cold starts and global distribution. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Error Boundary** — React component that catches JavaScript errors in its child tree, preventing full-app crashes. Class component using `componentDidCatch`. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**ESLint** — Pluggable JavaScript/TypeScript linter. Essential for enforcing code standards in teams. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Event Loop** — JavaScript's concurrency mechanism. Single-threaded execution with a message queue, microtask queue, and macrotask queue. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Execution Context** — JavaScript's internal mechanism for tracking function execution: variable environment, scope chain, and `this` binding. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Expo** — Platform built on React Native providing managed workflow, native module ecosystem, and cloud services (EAS). See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Expo Fingerprint** — Hash of all native dependencies and configuration. When the fingerprint changes, a new native build is required (OTA update won't suffice). See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Expo Router** — File-based routing for React Native (and web) built on React Navigation. Supports deep linking, typed routes, and universal apps. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

---

## F

**Fabric** — React Native's new rendering system. Replaces the old async bridge-based renderer with synchronous C++ rendering. Enables concurrent features. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**FCP (First Contentful Paint)** — Web performance metric: time until the first text or image is rendered. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Feature Flag** — Configuration toggle that enables/disables functionality without deploying new code. Often stored in Edge Config or LaunchDarkly. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Fiber** — React's internal reconciliation engine (since React 16). Represents work as a linked list of fiber nodes, enabling incremental rendering, priorities, and interruption. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**Flipper** — Meta's debugging platform for React Native. Being deprecated in favor of Chrome DevTools and Expo Dev Tools. See [Ch 14](../part-4-architecture-at-scale/14-profiling-debugging.md).

**FlashList** — High-performance list component by Shopify for React Native. Replaces FlatList with cell recycling architecture. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Fluid Compute (Vercel)** — Vercel's serverless execution model that keeps functions warm between requests, streams responses, and avoids cold starts. Replaces traditional request-response model. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**FlatList** — React Native's built-in virtualized list. Renders only visible items. Being superseded by FlashList for performance-critical use cases. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

---

## G

**Garbage Collection (GC)** — Automatic memory reclamation. Hermes uses a generational GC optimized for mobile. V8 uses Orinoco. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**gcTime** — TanStack Query option defining how long inactive query data stays in the garbage collection queue before being removed. Default: 5 minutes. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**Gesture Handler (react-native-gesture-handler)** — Library that moves gesture recognition to the native thread, avoiding JS thread bottlenecks. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**GitHub Actions** — CI/CD platform built into GitHub. Workflows defined in YAML, triggered by events (push, PR, schedule). See [Ch 18](../part-4-architecture-at-scale/18-cicd.md).

**Google Play Console** — Google's portal for managing Android app submissions, staged rollouts, and crash analytics. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

---

## H

**Hermes** — JavaScript engine built by Meta specifically for React Native. Features ahead-of-time bytecode compilation, optimized GC, and small binary size. Default engine since RN 0.70. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**HMR (Hot Module Replacement)** — Development feature that updates changed modules in a running app without full reload, preserving state. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Hooks** — React functions (`useState`, `useEffect`, `useMemo`, etc.) that let function components use state and lifecycle features. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Hydration** — Process of attaching event handlers and React state to server-rendered HTML, making it interactive. Mismatch causes hydration errors. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

---

## I

**Idempotency** — Property where repeating an operation produces the same result. Critical for payment webhooks and API retries. Stripe uses idempotency keys. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Idempotency Key** — Unique identifier sent with payment requests to prevent duplicate charges on retry. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Image Optimization** — Automatic resizing, format conversion (WebP/AVIF), and lazy loading of images. Next.js `<Image>` and Expo Image handle this. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**INP (Interaction to Next Paint)** — Core Web Vital measuring responsiveness. Time from user interaction to the next visual update. Target: < 200ms. Replaced FID in 2024. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Incremental Static Regeneration (ISR)** — Next.js feature that regenerates static pages after deployment on a time-based or on-demand basis. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Intersection Observer** — Browser API that asynchronously observes element visibility. Used for lazy loading, infinite scroll, and analytics. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

---

## J

**Jank** — Visible stutter or frame drop in UI animation, caused by frames taking longer than 16.67ms (60fps). See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Jotai** — Atomic state management library for React. Bottom-up approach: state is composed from independent atoms. Minimal API, excellent for derived state. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

**JSI (JavaScript Interface)** — C++ API enabling synchronous, direct communication between JavaScript and native code in React Native. Replaces the async JSON bridge. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**JSX** — Syntax extension that lets you write HTML-like markup in JavaScript. Compiled to `React.createElement()` (or `jsx()`) calls. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**JIT (Just-In-Time Compilation)** — V8's optimization strategy: frequently-run code is compiled to machine code at runtime. Hermes uses AOT instead. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

---

## K

**Keychain (iOS) / Keystore (Android)** — Secure OS-level storage for credentials, tokens, and certificates. Used via `expo-secure-store`. See [Ch 22](../part-5-deployment-operations/22-security.md).

**KeyExtractor** — Function prop on FlatList/FlashList that returns a unique key for each item. Critical for efficient list reconciliation. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

---

## L

**Layout Animation** — React Native's `LayoutAnimation` API for animating layout changes declaratively. Simpler but less controllable than Reanimated. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Lazy Loading** — Deferring resource loading until needed. `React.lazy()` for components, native lazy loading for images. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**LCP (Largest Contentful Paint)** — Core Web Vital measuring perceived load speed. Time until the largest visible element renders. Target: < 2.5s. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Linting** — Static analysis to catch bugs and enforce code style. ESLint for JS/TS, Prettier for formatting. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

---

## M

**Maestro** — Mobile UI testing framework. Uses YAML-based flows for E2E testing across iOS and Android. No flaky selectors. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Mapped Type** — TypeScript type that transforms properties of an existing type: `{ [K in keyof T]: NewType }`. Powers utility types like `Partial`, `Required`, `Readonly`. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Memoization** — Caching expensive computation results. `React.memo()` for components, `useMemo()` for values, `useCallback()` for functions. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Metro** — React Native's JavaScript bundler. Handles module resolution, transformation, and serialization. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Microtask Queue** — JavaScript queue for Promise callbacks and `queueMicrotask()`. Processed before the next macrotask. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Middleware (Next.js)** — Code that runs before a request is completed. Executes at the edge for redirects, rewrites, authentication, and A/B testing. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Monorepo** — Single repository containing multiple packages/apps. Managed with Turborepo, pnpm workspaces, or Nx. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**MSW (Mock Service Worker)** — API mocking library that intercepts requests at the network level. Works in browser and Node.js. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Mutation** — In TanStack Query, an operation that changes server-side data (POST/PUT/DELETE). `useMutation` handles loading, error, and cache invalidation. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

---

## N

**Native Module** — Platform-specific code (Swift/Kotlin) exposed to React Native JavaScript via JSI/TurboModules. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Navigation Container** — Root component in React Navigation that manages navigation state and links to deep linking configuration. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**New Architecture (React Native)** — Umbrella term for JSI, Fabric, TurboModules, and Codegen. Replaces the Bridge with synchronous, type-safe native interop. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Next.js App Router** — Next.js routing paradigm using the `app/` directory with React Server Components, nested layouts, and streaming. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Node.js Runtime** — Server-side JavaScript runtime. In Vercel, the default runtime for Serverless Functions (as opposed to Edge Runtime). See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

---

## O

**Offline-First** — Architecture where the app works without a network connection, syncing when connectivity returns. See [Ch 12](../part-3-state-data-communication/12-offline-realtime.md).

**Optimistic Update** — UI pattern that immediately reflects a mutation's expected result before server confirmation. Rolled back on failure. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**OTA (Over-the-Air) Update** — Pushing JavaScript bundle updates to users without going through app store review. EAS Update is Expo's OTA system. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**OWASP** — Open Web Application Security Project. Publishes the OWASP Top 10 (web) and OWASP Mobile Top 10 security risk lists. See [Ch 22](../part-5-deployment-operations/22-security.md).

---

## P

**Partial Prerendering (PPR)** — Next.js feature that combines static shell with streaming dynamic content in a single request. Static parts served from CDN, dynamic parts stream in. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**PaymentIntent** — Stripe object representing a payment lifecycle from creation to completion. Tracks amount, currency, status, and payment method. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**PCI DSS (Payment Card Industry Data Security Standard)** — Security standard for handling cardholder data. Stripe handles most PCI compliance; you handle secure integration. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**PKCE (Proof Key for Code Exchange)** — OAuth 2.0 extension for public clients (mobile apps). Prevents authorization code interception attacks. See [Ch 22](../part-5-deployment-operations/22-security.md).

**Playwright** — Cross-browser E2E testing framework by Microsoft. Supports Chromium, Firefox, and WebKit. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**pnpm** — Fast, disk-efficient package manager using content-addressable storage and symlinks. Enforces strict dependency isolation. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Prebuild (Expo)** — Command (`npx expo prebuild`) that generates native `ios/` and `android/` directories from `app.json` and config plugins. The CNG mechanism. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Prefetch** — Loading resources/data before they're needed. TanStack Query's `prefetchQuery`, Next.js link prefetching, DNS prefetch. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**Progressive Web App (PWA)** — Web app with offline support, installability, and native-like features via service workers and web manifests. See [Ch 12](../part-3-state-data-communication/12-offline-realtime.md).

**Project References (TypeScript)** — `tsconfig.json` feature enabling incremental builds across packages in a monorepo. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**ProGuard / R8** — Android build tools that shrink, optimize, and obfuscate Java/Kotlin bytecode. R8 is Google's replacement for ProGuard. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Promise** — JavaScript object representing an eventual async result. States: pending, fulfilled, rejected. Foundation of async/await. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

---

## Q

**Query Key** — Unique identifier array for a TanStack Query cache entry. Determines cache hits, invalidation scope, and refetch behavior. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**Query Client** — TanStack Query's central cache store. Holds all query data, manages garbage collection, and coordinates invalidation. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

---

## R

**React Navigation** — Standard navigation library for React Native. Provides stack, tab, drawer navigators with native-like transitions. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**Reanimated (react-native-reanimated)** — Animation library running on the UI thread via worklets. Enables 60fps animations independent of JS thread. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Reconciliation** — React's diffing algorithm that compares virtual DOM trees to determine the minimal set of DOM/native updates needed. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**Reflow** — Browser layout recalculation triggered by DOM changes affecting geometry (width, height, position). Expensive operation. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Render Tree** — Combined DOM + CSSOM structure the browser uses to compute layout and paint. Excludes `display: none` elements. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Repaint** — Browser redrawing pixels for visual-only changes (color, shadow) that don't affect layout. Cheaper than reflow. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**RevenueCat** — Third-party SDK for managing in-app purchases and subscriptions across iOS and Android. Handles receipt validation, entitlements, and analytics. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**RNTL (React Native Testing Library)** — Testing library encouraging testing components the way users interact with them. Built on `@testing-library` principles. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Rolling Releases (Vercel)** — Vercel deployment strategy that gradually shifts traffic from old to new deployment, monitoring for errors. Automatic rollback on failure. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Route Handler** — Next.js App Router's server-side API endpoints defined as `route.ts` files. Support GET, POST, PUT, DELETE, etc. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**RUM (Real User Monitoring)** — Collecting performance and error data from actual user sessions (not synthetic tests). Sentry, Datadog, Vercel Analytics. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

---

## S

**Safe Area** — Screen region not obscured by device hardware (notch, home indicator, status bar). Managed with `SafeAreaView` or `useSafeAreaInsets`. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Server Action** — Next.js feature: async function marked with `"use server"` that runs on the server when called from client. Replaces API routes for mutations. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Server Component** — React component that runs only on the server. Zero client JS, direct database access, async by default. Default in Next.js App Router. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Shadow Thread** — React Native thread running Yoga layout calculations. In Fabric, layout can be computed synchronously on any thread. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Skia (react-native-skia)** — 2D graphics library bringing Skia (Chrome/Android's rendering engine) to React Native for custom drawing, shaders, and image processing. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Source Map** — File mapping bundled/minified code back to original source. Essential for debugging production crashes. Must be uploaded to error tracking services. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**SSG (Static Site Generation)** — Pre-rendering pages at build time. Fastest possible TTFB but data can be stale. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**SSR (Server-Side Rendering)** — Rendering HTML on the server per request. Fresh data but slower TTFB than static. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Staged Rollout** — Releasing an update to a percentage of users, gradually increasing if metrics are healthy. Supported by EAS Update channels and Google Play. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**staleTime** — TanStack Query option defining how long data is considered fresh. During staleTime, no background refetch occurs. Default: 0. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**State Machine** — Formal model of states and transitions. Libraries like XState prevent impossible states. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

**Streaming (SSR)** — Sending HTML to the browser in chunks as it's rendered on the server. Used with React Suspense for progressive page loading. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Strict Mode** — React development wrapper that double-invokes effects and renders to catch side effect bugs. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Storybook** — Component development environment for building, testing, and documenting UI components in isolation. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Suspense** — React component for declarative loading states. Wraps components that suspend (lazy, data fetching) and shows a fallback. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**SWC** — Rust-based JavaScript/TypeScript compiler. 20-70x faster than Babel. Used by Next.js and Turbopack. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Symbolication** — Process of translating memory addresses in crash reports back to human-readable function names and line numbers using debug symbols (dSYM/source maps). See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

---

## T

**TanStack Query (React Query)** — Async state management library for server state. Handles caching, deduplication, background refetch, pagination, and optimistic updates. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**TestFlight** — Apple's beta testing platform. Distributes pre-release iOS builds to testers. EAS Submit can upload directly. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Throttle** — Technique that limits function execution to at most once per time interval. Used for scroll handlers and API rate limiting. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Token (Design)** — See *Design Token*.

**Transpiler** — Tool that converts code from one language/version to another (e.g., TypeScript → JavaScript, modern JS → ES5). Babel, SWC, tsc. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Tree Shaking** — Bundler optimization that eliminates unused exports from the final bundle. Requires ES modules (static `import`/`export`). See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**tRPC** — End-to-end typesafe API framework. Client and server share TypeScript types with zero code generation. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**TTI (Time to Interactive)** — Time until the page is fully interactive (all event handlers attached, main thread idle). See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**TTFB (Time to First Byte)** — Time from request start to first byte of response. Measures server/CDN response speed. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**TurboModules** — React Native's new native module system using JSI for synchronous, lazy-loaded, type-safe native method calls. Replaces the Bridge-based NativeModules. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Turbopack** — Rust-based bundler by Vercel. Successor to Webpack for Next.js development. Incremental by design. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Turborepo** — High-performance build system for monorepos. Content-addressed caching, parallel execution, task pipelines. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Type Guard** — TypeScript function or expression that narrows a type within a conditional block. `typeof`, `instanceof`, `in`, or custom `is` predicates. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Type Inference** — TypeScript's ability to determine types without explicit annotations. Reduces verbosity while maintaining safety. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

---

## U

**Universal Link (iOS) / App Link (Android)** — OS-level mechanism that opens a URL directly in your app instead of the browser. Requires server-side verification file. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**useCallback** — React hook that memoizes a function reference. Prevents unnecessary re-renders when passed as a prop. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useEffect** — React hook for side effects: data fetching, subscriptions, DOM manipulation. Runs after render. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useMemo** — React hook that memoizes a computed value. Only recomputes when dependencies change. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useReducer** — React hook for complex state logic. Dispatch actions to a reducer function. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useRef** — React hook that creates a mutable ref object persisting across renders. Used for DOM refs and instance variables. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useState** — React hook for component-level state. Returns `[value, setter]` tuple. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useTransition** — React 18+ hook that marks state updates as non-urgent, allowing urgent updates (typing) to interrupt slow renders. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

---

## V

**V8** — Google's JavaScript engine (Chrome, Node.js). Uses JIT compilation. Not used in React Native (which uses Hermes). See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Vercel** — Cloud platform for frontend deployment. Provides serverless functions, edge network, analytics, and framework-aware builds. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Vercel Analytics** — Real user performance monitoring built into Vercel. Tracks Core Web Vitals, page views, and custom events. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Virtual DOM** — In-memory representation of the UI tree. React diffs old vs new virtual DOM to compute minimal updates. See [Ch 0b](../part-0-fundamentals/00b-ui-development-hard-parts.md).

**Virtualization** — Rendering only visible list items, recycling off-screen views. FlatList, FlashList, and `react-window` use this pattern. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Vitest** — Vite-powered test runner. Fast, ESM-native, compatible with Jest API. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

---

## W

**Warm Start** — App resuming from a backgrounded state (memory retained). Much faster than cold start. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Web Vitals** — See *Core Web Vitals*.

**Webhook** — HTTP callback sent by a service when an event occurs. Stripe webhooks notify your server of payment events. Must be verified and handled idempotently. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**WebSocket** — Full-duplex communication protocol over a single TCP connection. Used for real-time features (chat, live updates). See [Ch 12](../part-3-state-data-communication/12-offline-realtime.md).

**Worklet (Reanimated)** — JavaScript function that runs on the UI thread via a separate JS runtime. Enables 60fps animations without crossing the bridge. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Workspace (pnpm/npm)** — Monorepo feature that links local packages together and hoists shared dependencies. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

---

## X

**XState** — State machine and statechart library for JavaScript. Prevents impossible states through formal modeling. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

---

## Y

**Yoga** — Cross-platform layout engine by Meta implementing a subset of CSS Flexbox. Used by React Native to compute layouts in C++. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

---

## Z

**Zod** — TypeScript-first schema validation library. Runtime type checking that infers static types. Used with tRPC, React Hook Form, and API validation. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Zero-Config** — Tools or frameworks that work without configuration files. Expo, Vercel, and Turbopack aim for zero-config defaults with escape hatches. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Zustand** — Lightweight state management library for React. Single store, no boilerplate, works outside React. Ideal for client-side global state. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

---

*Total: 200+ terms. Last updated: 2026-04-07.*

> **See also:** [Appendix B: Cheat Sheet](./appendix-cheatsheet.md) | [Appendix C: Reading List](./appendix-reading-list.md)
