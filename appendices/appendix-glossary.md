<!--
  TYPE: appendix
  TITLE: Glossary
  APPENDIX: A
  UPDATED: 2026-04-07
-->

# Appendix A: Glossary

> 300+ frontend engineering terms — from A/B test to Zustand.

Cross-references point to the chapter where the concept is covered in depth. Terms are grouped alphabetically for quick lookup.

---

## A

**A/B Test** — Experiment comparing two or more variants to determine which performs better against a metric. Requires statistical significance to declare a winner. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**Abstract Syntax Tree (AST)** — Tree representation of source code structure produced by a parser. Babel, ESLint, and TypeScript all operate on ASTs. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Access Token** — Short-lived credential (often a JWT) authorizing API requests on behalf of a user. Typically paired with a longer-lived refresh token. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Accessibility (a11y)** — Practices ensuring apps are usable by people with disabilities. Includes screen readers, semantic markup, ARIA roles, and contrast ratios. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Algolia** — Hosted search-as-a-service platform providing sub-50ms full-text search, faceting, and typo tolerance via a REST/JS API. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

**Amplitude** — Product analytics platform for tracking user behavior, funnels, and retention. Competes with Mixpanel and PostHog. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**ANR (App Not Responding)** — Android system dialog triggered when the main thread is blocked for 5+ seconds. A critical mobile vitals metric. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

**API Route** — Server-side endpoint defined inside a Next.js app. In App Router, these are `route.ts` files. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Apollo Client** — Feature-rich GraphQL client for React with normalized caching, local state management, and devtools. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**App Store Connect** — Apple's portal for managing iOS app submissions, TestFlight builds, and metadata. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Art Direction** — Serving different image crops or compositions based on viewport size using `<picture>` and `<source>` elements, rather than simply scaling the same image. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**Atomic Design** — Design methodology (atoms, molecules, organisms, templates, pages) for building component libraries systematically. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**ATT (App Tracking Transparency)** — Apple's iOS 14.5+ framework requiring apps to request user permission before tracking across other apps and websites. Significantly impacts attribution. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Attribution** — Identifying which marketing channel, campaign, or touchpoint led a user to convert. Models include first-touch, last-touch, and multi-touch. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Auth.js** — Open-source authentication library for Next.js (formerly NextAuth.js). Supports OAuth providers, credentials, database sessions, and JWT. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Auto-linking** — React Native mechanism that automatically links native dependencies without manual `pod install` or Gradle configuration. Expo handles this via CNG. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Autosave** — Form pattern that persists user input automatically (via debounced mutations or local storage) without requiring an explicit save action. See [Ch 35](../part-3-state-data-communication/35-forms-at-scale.md).

**AVIF** — Next-generation image format based on AV1 video codec offering ~50% smaller files than JPEG at equivalent quality. Supported by modern browsers. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

---

## B

**Babel** — JavaScript transpiler that converts modern JS/TS/JSX into backwards-compatible code. Being replaced by SWC and Hermes in many RN workflows. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Barrel File** — An `index.ts` that re-exports from multiple modules. Convenient but can destroy tree shaking if not configured carefully. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Bitcode** — Intermediate representation used by Apple's compiler. Deprecated since Xcode 14 but still referenced in legacy build configs. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Blurhash** — Compact string representation of a blurred image placeholder. Decoded client-side to show a colored blur while the full image loads. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

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

**Changesets** — Versioning tool for monorepos. Developers add changeset files describing their changes; CI aggregates them into version bumps and changelogs. See [Ch 45](../part-5-deployment-operations/45-release-management.md).

**Circuit Breaker** — Resilience pattern that stops calling a failing service after a threshold of errors, returning a fallback instead. Prevents cascade failures. See [Ch 36](../part-4-architecture-at-scale/36-error-handling.md).

**Claude Code** — Anthropic's agentic coding tool that operates directly in the terminal, reading/writing files and executing commands to implement features autonomously. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**CLAUDE.md** — Convention file placed at the repository root providing project context, coding standards, and instructions to agentic coding tools like Claude Code. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**Clerk** — Drop-in authentication and user management platform with prebuilt UI components, multi-factor auth, and organization support. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Client Component** — React component that runs in the browser (marked with `"use client"` in Next.js App Router). Has access to state, effects, and browser APIs. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**CLS (Cumulative Layout Shift)** — Core Web Vital measuring visual stability. Sum of all unexpected layout shifts during page lifetime. Target: < 0.1. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**CNG (Continuous Native Generation)** — Expo's approach where native `ios/` and `android/` directories are generated from `app.json` and config plugins at build time, not checked into source control. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Code Splitting** — Technique of breaking a bundle into smaller chunks loaded on demand. Reduces initial load time. `React.lazy()` and dynamic `import()`. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Codegen (GraphQL)** — Build-time tool (e.g., GraphQL Code Generator) that produces TypeScript types and hooks from GraphQL schemas and operations, ensuring end-to-end type safety. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**Codegen (React Native)** — Build-time code generation from TypeScript/Flow specs that creates type-safe C++ bindings for TurboModules and Fabric components. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Cohort Analysis** — Analytics technique grouping users by a shared characteristic (e.g., signup week) to compare behavior and retention over time. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Cold Start** — The full startup sequence of a React Native app: native init → JS engine init → bundle load → React render → native mount. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Composition Layer** — GPU-accelerated layer created by the browser compositor for elements with transforms, opacity, or `will-change`. Avoids reflow/repaint. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Compound Component** — React pattern where a parent component shares implicit state with child components via context, enabling flexible composition (e.g., `<Select>`, `<Select.Option>`). See [Ch 48](../part-4-architecture-at-scale/48-component-architecture.md).

**Conditional Type** — TypeScript type that selects between two types based on a condition: `T extends U ? X : Y`. Enables type-level programming. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Config Plugin (Expo)** — JavaScript function that modifies native project files (Info.plist, AndroidManifest.xml, Podfile, etc.) during prebuild. The mechanism for customizing native code in CNG. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Connection Pooling** — Reusing database connections across requests instead of creating new ones. Essential for serverless environments where each invocation would otherwise open a fresh connection. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Contentful** — Headless CMS providing structured content via REST and GraphQL APIs with a visual editing interface and CDN delivery. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**Cookie Management (SSR)** — Handling cookies in server-rendered environments where `document.cookie` is unavailable. Requires reading from request headers and setting via response headers. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Core Web Vitals** — Google's three key metrics: LCP (loading), INP (interactivity), CLS (visual stability). Affect search ranking and user experience. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Crash-Free Rate** — Percentage of user sessions without a crash. Industry target: 99.9%+. Primary mobile reliability metric. See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

**CRDT (Conflict-free Replicated Data Type)** — Data structure that can be independently updated on multiple replicas and merged without conflicts. Foundation of real-time collaboration libraries like Yjs. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**CSRF (Cross-Site Request Forgery)** — Attack where a malicious site tricks a user's browser into making authenticated requests to another site. Prevented with CSRF tokens and `sameSite` cookies. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**CSSOM (CSS Object Model)** — Browser's internal representation of CSS rules. Combined with DOM to produce the render tree. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Custom Hook** — A JavaScript function starting with `use` that encapsulates reusable stateful logic by composing built-in hooks. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

---

## D

**Dead Code Elimination** — Compiler optimization that removes code proven unreachable. Related to but distinct from tree shaking. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Dead Letter Queue** — Storage for messages or events that repeatedly fail processing. Enables investigation and replay without blocking the main processing pipeline. See [Ch 36](../part-4-architecture-at-scale/36-error-handling.md).

**Debounce** — Technique that delays function execution until a pause in invocations. Used for search inputs, resize handlers. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Deep Linking** — Opening a specific screen in a mobile app via a URL. Requires navigation configuration and native URL scheme/universal link setup. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**Dehydrate/Hydrate** — TanStack Query pattern where server-fetched query data is serialized (dehydrated) into HTML, then restored (hydrated) on the client to avoid redundant fetches. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Design Token** — Named value (color, spacing, font size) that represents a design decision. The atomic unit of a design system. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Design-to-Code** — Workflow converting visual designs (Figma, screenshots) into production code, increasingly automated by AI tools like v0, Vercel, and Claude Code. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**Discriminated Union** — TypeScript pattern where union members share a literal-typed discriminant field, enabling exhaustive `switch` statements with type narrowing. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**DKIM (DomainKeys Identified Mail)** — Email authentication method that attaches a digital signature to outgoing messages, allowing receiving servers to verify the sender's domain. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**DMARC (Domain-based Message Authentication, Reporting & Conformance)** — Email authentication policy that builds on SPF and DKIM to tell receivers how to handle unauthenticated mail from your domain. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**DOM (Document Object Model)** — Tree of C++ objects representing HTML elements, exposed to JavaScript via WebIDL bindings. See [Ch 0b](../part-0-fundamentals/00b-ui-development-hard-parts.md).

**Drizzle ORM** — Lightweight TypeScript ORM with SQL-like syntax, zero runtime overhead, and excellent type inference. Supports Postgres, MySQL, and SQLite. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

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

**Email Deliverability** — The ability to land emails in recipients' inboxes rather than spam folders. Depends on SPF, DKIM, DMARC, sender reputation, and content quality. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**Error Boundary** — React component that catches JavaScript errors in its child tree, preventing full-app crashes. Class component using `componentDidCatch`. See [Ch 36](../part-4-architecture-at-scale/36-error-handling.md).

**ESLint** — Pluggable JavaScript/TypeScript linter. Essential for enforcing code standards in teams. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Event Loop** — JavaScript's concurrency mechanism. Single-threaded execution with a message queue, microtask queue, and macrotask queue. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Event Taxonomy** — Structured naming convention and hierarchy for analytics events (e.g., `object_action` format). Ensures consistent, queryable data across an organization. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Execution Context** — JavaScript's internal mechanism for tracking function execution: variable environment, scope chain, and `this` binding. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Expo** — Platform built on React Native providing managed workflow, native module ecosystem, and cloud services (EAS). See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Expo Fingerprint** — Hash of all native dependencies and configuration. When the fingerprint changes, a new native build is required (OTA update won't suffice). See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Expo Router** — File-based routing for React Native (and web) built on React Navigation. Supports deep linking, typed routes, and universal apps. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

---

## F

**Fabric** — React Native's new rendering system. Replaces the old async bridge-based renderer with synchronous C++ rendering. Enables concurrent features. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Faceted Search** — Search interface allowing users to filter results by multiple dimensions (category, price, date) simultaneously. Each facet shows available values and counts. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

**FCP (First Contentful Paint)** — Web performance metric: time until the first text or image is rendered. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Feature Flag** — Configuration toggle that enables/disables functionality without deploying new code. Often stored in Edge Config or LaunchDarkly. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**Feature Gate** — Boolean feature flag that gates access to a feature based on user attributes, percentage rollout, or environment. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**Fiber** — React's internal reconciliation engine (since React 16). Represents work as a linked list of fiber nodes, enabling incremental rendering, priorities, and interruption. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**Fitts's Law** — UX principle stating the time to reach a target is a function of its distance and size. Larger, closer targets are faster to click/tap. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

**Flipper** — Meta's debugging platform for React Native. Being deprecated in favor of Chrome DevTools and Expo Dev Tools. See [Ch 14](../part-4-architecture-at-scale/14-profiling-debugging.md).

**FlashList** — High-performance list component by Shopify for React Native. Replaces FlatList with cell recycling architecture. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Fluid Compute (Vercel)** — Vercel's serverless execution model that keeps functions warm between requests, streams responses, and avoids cold starts. Replaces traditional request-response model. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**FlatList** — React Native's built-in virtualized list. Renders only visible items. Being superseded by FlashList for performance-critical use cases. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Fragment (GraphQL)** — Reusable unit of GraphQL fields that can be shared across queries. Enables component-level data requirements via co-located fragments. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**Full-Text Search** — Search capability that indexes and queries natural language text across documents, supporting relevance ranking, stemming, and fuzzy matching. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

---

## G

**Garbage Collection (GC)** — Automatic memory reclamation. Hermes uses a generational GC optimized for mobile. V8 uses Orinoco. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**gcTime** — TanStack Query option defining how long inactive query data stays in the garbage collection queue before being removed. Default: 5 minutes. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**Gesture Handler (react-native-gesture-handler)** — Library that moves gesture recognition to the native thread, avoiding JS thread bottlenecks. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**GitHub Actions** — CI/CD platform built into GitHub. Workflows defined in YAML, triggered by events (push, PR, schedule). See [Ch 18](../part-4-architecture-at-scale/18-cicd.md).

**Google Play Console** — Google's portal for managing Android app submissions, staged rollouts, and crash analytics. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Graceful Degradation** — Design strategy where a system continues to function with reduced capability when a component fails, rather than crashing entirely. See [Ch 36](../part-4-architecture-at-scale/36-error-handling.md).

**Gradual Rollout** — Releasing a feature or update to an increasing percentage of users over time while monitoring metrics. Enables early detection of issues. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

---

## H

**Haptic Vocabulary** — Consistent mapping between haptic feedback patterns (light, medium, heavy, selection) and UI actions, creating a tactile language users learn intuitively. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

**Headless CMS** — Content management system that provides content via API (REST/GraphQL) without a coupled frontend. Examples: Sanity, Contentful, Strapi. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**Headless UI** — Component library providing behavior and accessibility without visual styling, giving developers full control over appearance. Examples: Radix UI, Headless UI, React Aria. See [Ch 48](../part-4-architecture-at-scale/48-component-architecture.md).

**Hermes** — JavaScript engine built by Meta specifically for React Native. Features ahead-of-time bytecode compilation, optimized GC, and small binary size. Default engine since RN 0.70. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Hick's Law** — UX principle stating decision time increases logarithmically with the number of choices. Guides progressive disclosure and menu design. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

**HMR (Hot Module Replacement)** — Development feature that updates changed modules in a running app without full reload, preserving state. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Hook (Agentic)** — In Claude Code, a user-defined script or command that runs before or after specific tool invocations, enabling automated validation, formatting, or side effects. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**Hooks** — React functions (`useState`, `useEffect`, `useMemo`, etc.) that let function components use state and lifecycle features. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Hotfix** — Emergency patch deployed outside the normal release cycle to fix a critical production issue. Typically cherry-picked to the release branch. See [Ch 45](../part-5-deployment-operations/45-release-management.md).

**httpOnly Cookie** — Cookie flag preventing JavaScript access via `document.cookie`, mitigating XSS token theft. Standard for storing session tokens securely. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Hydration** — Process of attaching event handlers and React state to server-rendered HTML, making it interactive. Mismatch causes hydration errors. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Hydration Mismatch** — Error when server-rendered HTML differs from client-rendered output, causing React to discard the server HTML and re-render. Common with dates, random values, and browser-only APIs. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

---

## I

**Idempotency** — Property where repeating an operation produces the same result. Critical for payment webhooks and API retries. Stripe uses idempotency keys. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Idempotency Key** — Unique identifier sent with payment requests to prevent duplicate charges on retry. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Image CDN** — Service that resizes, reformats, and caches images on-the-fly at edge locations via URL parameters. Examples: Cloudinary, imgix, Vercel Image Optimization. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

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

**JWT (JSON Web Token)** — Compact, self-contained token format encoding claims as a signed JSON payload. Used for stateless authentication but cannot be revoked without a denylist. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

---

## K

**Keychain (iOS) / Keystore (Android)** — Secure OS-level storage for credentials, tokens, and certificates. Used via `expo-secure-store`. See [Ch 22](../part-5-deployment-operations/22-security.md).

**KeyExtractor** — Function prop on FlatList/FlashList that returns a unique key for each item. Critical for efficient list reconciliation. See [Ch 13](../part-4-architecture-at-scale/13-mobile-performance.md).

**Kill Switch** — Feature flag designed to instantly disable a feature in production without a deploy. Essential for incident response. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

---

## L

**Layout Animation** — React Native's `LayoutAnimation` API for animating layout changes declaratively. Simpler but less controllable than Reanimated. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Lazy Loading** — Deferring resource loading until needed. `React.lazy()` for components, native lazy loading for images. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**LaunchDarkly** — Enterprise feature flag management platform with targeting rules, experimentation, and audit logs. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**LCP (Largest Contentful Paint)** — Core Web Vital measuring perceived load speed. Time until the largest visible element renders. Target: < 2.5s. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**Lighthouse CI** — Automated tool for running Google Lighthouse audits in CI pipelines, enforcing performance budgets and accessibility thresholds on every PR. See [Ch 44](../part-4-architecture-at-scale/44-performance-budgets.md).

**Linting** — Static analysis to catch bugs and enforce code style. ESLint for JS/TS, Prettier for formatting. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Live Cursors** — Real-time display of other users' cursor positions in a collaborative interface, providing awareness of where others are working. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**Liveblocks** — Real-time collaboration infrastructure providing presence, storage, and conflict resolution as a service. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**LQIP (Low-Quality Image Placeholder)** — Technique of showing a tiny, blurred version of an image while the full-resolution version loads. Implemented via blurhash, thumbhash, or tiny inline images. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

---

## M

**Maestro** — Mobile UI testing framework. Uses YAML-based flows for E2E testing across iOS and Android. No flaky selectors. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Mapped Type** — TypeScript type that transforms properties of an existing type: `{ [K in keyof T]: NewType }`. Powers utility types like `Partial`, `Required`, `Readonly`. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**MCP Server (Model Context Protocol)** — Standardized server that exposes tools, resources, and prompts to AI coding assistants, enabling them to interact with external services and APIs. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**Meilisearch** — Open-source, Rust-based search engine offering typo-tolerant, instant full-text search with faceting. Self-hostable alternative to Algolia. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

**Memoization** — Caching expensive computation results. `React.memo()` for components, `useMemo()` for values, `useCallback()` for functions. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Metro** — React Native's JavaScript bundler. Handles module resolution, transformation, and serialization. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**MFA (Multi-Factor Authentication)** — Authentication requiring two or more verification factors (password + TOTP code, password + passkey). Significantly reduces account compromise risk. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Micro-copy** — Short, contextual UI text (button labels, tooltips, error messages, empty states) that guides users through an interface. Small changes can dramatically affect conversion. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

**Microtask Queue** — JavaScript queue for Promise callbacks and `queueMicrotask()`. Processed before the next macrotask. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Middleware (Next.js)** — Code that runs before a request is completed. Executes at the edge for redirects, rewrites, authentication, and A/B testing. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Migration (Database)** — Versioned, incremental change to a database schema (adding tables, columns, indexes). Managed by ORM tools like Drizzle or Prisma. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Mixpanel** — Product analytics platform focused on event-based tracking, funnels, retention, and user segmentation. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Monorepo** — Single repository containing multiple packages/apps. Managed with Turborepo, pnpm workspaces, or Nx. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**MSW (Mock Service Worker)** — API mocking library that intercepts requests at the network level. Works in browser and Node.js. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Multi-Step Wizard** — Form pattern that breaks a long form into sequential steps with progress indication, validation per step, and the ability to navigate back. See [Ch 35](../part-3-state-data-communication/35-forms-at-scale.md).

**Mutation** — In TanStack Query, an operation that changes server-side data (POST/PUT/DELETE). `useMutation` handles loading, error, and cache invalidation. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

---

## N

**Native Module** — Platform-specific code (Swift/Kotlin) exposed to React Native JavaScript via JSI/TurboModules. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Navigation Container** — Root component in React Navigation that manages navigation state and links to deep linking configuration. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**Neon** — Serverless Postgres platform with autoscaling, branching, and a built-in connection pooler. Designed for serverless and edge environments. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**New Architecture (React Native)** — Umbrella term for JSI, Fabric, TurboModules, and Codegen. Replaces the Bridge with synchronous, type-safe native interop. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Next.js App Router** — Next.js routing paradigm using the `app/` directory with React Server Components, nested layouts, and streaming. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Node.js Runtime** — Server-side JavaScript runtime. In Vercel, the default runtime for Serverless Functions (as opposed to Edge Runtime). See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

---

## O

**OAuth2** — Industry-standard authorization framework enabling third-party applications to obtain limited access to a user's account. Flows include Authorization Code (with PKCE for public clients), Client Credentials, and Device Code. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Offline-First** — Architecture where the app works without a network connection, syncing when connectivity returns. See [Ch 12](../part-3-state-data-communication/12-offline-realtime.md).

**Operational Transformation (OT)** — Algorithm for resolving concurrent edits in collaborative editing by transforming operations against each other. Used by Google Docs; largely superseded by CRDTs. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**Optimistic Update** — UI pattern that immediately reflects a mutation's expected result before server confirmation. Rolled back on failure. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**OTA (Over-the-Air) Update** — Pushing JavaScript bundle updates to users without going through app store review. EAS Update is Expo's OTA system. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**OWASP** — Open Web Application Security Project. Publishes the OWASP Top 10 (web) and OWASP Mobile Top 10 security risk lists. See [Ch 22](../part-5-deployment-operations/22-security.md).

---

## P

**Partial Prerendering (PPR)** — Next.js feature that combines static shell with streaming dynamic content in a single request. Static parts served from CDN, dynamic parts stream in. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Passkey** — Phishing-resistant, passwordless credential based on WebAuthn public-key cryptography. Stored in platform authenticators (iCloud Keychain, Google Password Manager). See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**PaymentIntent** — Stripe object representing a payment lifecycle from creation to completion. Tracks amount, currency, status, and payment method. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**PCI DSS (Payment Card Industry Data Security Standard)** — Security standard for handling cardholder data. Stripe handles most PCI compliance; you handle secure integration. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Performance Budget** — Hard limits on metrics (bundle size, LCP, total JS weight) that trigger CI failures when exceeded. Enforced via size-limit, Lighthouse CI, or bundlesize. See [Ch 44](../part-4-architecture-at-scale/44-performance-budgets.md).

**Persisted Query** — GraphQL optimization where query strings are replaced with hashes. The server stores the query text, reducing payload size and enabling CDN caching. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**Pixel Density** — Ratio of physical pixels to CSS pixels on a display (1x, 2x, 3x). Images must serve appropriate resolutions via `srcset` to look sharp on high-density screens. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**PKCE (Proof Key for Code Exchange)** — OAuth 2.0 extension for public clients (mobile/SPA). Prevents authorization code interception attacks using a code verifier and challenge. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**PlanetScale** — Serverless MySQL platform built on Vitess with non-blocking schema changes, branching, and connection pooling. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Playwright** — Cross-browser E2E testing framework by Microsoft. Supports Chromium, Firefox, and WebKit. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**pnpm** — Fast, disk-efficient package manager using content-addressable storage and symlinks. Enforces strict dependency isolation. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Polymorphic Component** — React component that can render as different HTML elements or other components via an `as` or `asChild` prop while preserving type safety. See [Ch 48](../part-4-architecture-at-scale/48-component-architecture.md).

**Portable Text** — Sanity's structured rich text format stored as JSON, enabling fully customizable rendering across any platform (web, mobile, email). See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**PostHog** — Open-source product analytics platform combining event tracking, session replay, feature flags, and A/B testing in one tool. Self-hostable. See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Prebuild (Expo)** — Command (`npx expo prebuild`) that generates native `ios/` and `android/` directories from `app.json` and config plugins. The CNG mechanism. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Prefetch** — Loading resources/data before they're needed. TanStack Query's `prefetchQuery`, Next.js link prefetching, DNS prefetch. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**Presence** — Real-time awareness of which users are currently active in a shared space, including their cursor positions, selections, and online status. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**Prisma** — TypeScript ORM with a declarative schema language, auto-generated type-safe client, and visual database browser (Prisma Studio). See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Progressive Disclosure** — UX pattern that reveals information and options gradually as needed, reducing cognitive load. Shows basic options first with advanced settings available on demand. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

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

**RBAC (Role-Based Access Control)** — Authorization model where permissions are assigned to roles, and roles are assigned to users. Simpler than attribute-based access control but sufficient for most apps. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**React Email** — Library for building emails using React components and rendering them to cross-client HTML. Pairs with Resend for sending. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**React Navigation** — Standard navigation library for React Native. Provides stack, tab, drawer navigators with native-like transitions. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**Read Replica** — Read-only copy of a database that offloads read queries from the primary, improving throughput and reducing latency for read-heavy workloads. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Reanimated (react-native-reanimated)** — Animation library running on the UI thread via worklets. Enables 60fps animations independent of JS thread. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Reconciliation** — React's diffing algorithm that compares virtual DOM trees to determine the minimal set of DOM/native updates needed. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**Refresh Token** — Long-lived credential used to obtain new access tokens without re-authenticating. Stored securely (httpOnly cookie or secure storage) and rotated on each use. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Reflow** — Browser layout recalculation triggered by DOM changes affecting geometry (width, height, position). Expensive operation. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Release Train** — Scheduled release cadence (e.g., weekly) where all completed features ship together, reducing coordination overhead and improving predictability. See [Ch 45](../part-5-deployment-operations/45-release-management.md).

**Render Tree** — Combined DOM + CSSOM structure the browser uses to compute layout and paint. Excludes `display: none` elements. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**Resend** — Modern email API for developers. Provides a simple REST API for sending transactional emails, with React Email integration for templating. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**Repaint** — Browser redrawing pixels for visual-only changes (color, shadow) that don't affect layout. Cheaper than reflow. See [Ch 2](../part-1-foundations/02-browser-rendering.md).

**RevenueCat** — Third-party SDK for managing in-app purchases and subscriptions across iOS and Android. Handles receipt validation, entitlements, and analytics. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**RNTL (React Native Testing Library)** — Testing library encouraging testing components the way users interact with them. Built on `@testing-library` principles. See [Ch 17](../part-4-architecture-at-scale/17-testing.md).

**Rolling Releases (Vercel)** — Vercel deployment strategy that gradually shifts traffic from old to new deployment, monitoring for errors. Automatic rollback on failure. See [Ch 24](../part-6-vercel-web/24-vercel-platform.md).

**Route Handler** — Next.js App Router's server-side API endpoints defined as `route.ts` files. Support GET, POST, PUT, DELETE, etc. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Row Level Security (RLS)** — Database-level access control (primarily Postgres/Supabase) where policies filter rows based on the current user's identity, enforcing authorization at the data layer. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**RUM (Real User Monitoring)** — Collecting performance and error data from actual user sessions (not synthetic tests). Sentry, Datadog, Vercel Analytics. See [Ch 44](../part-4-architecture-at-scale/44-performance-budgets.md).

---

## S

**Safe Area** — Screen region not obscured by device hardware (notch, home indicator, status bar). Managed with `SafeAreaView` or `useSafeAreaInsets`. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**sameSite** — Cookie attribute controlling when cookies are sent with cross-site requests. Values: `Strict`, `Lax`, `None`. Key defense against CSRF attacks. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Sanity** — Headless CMS with a real-time, customizable editing studio and structured content stored as Portable Text. Content delivered via GROQ or GraphQL. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**Segment** — Customer data platform that collects analytics events from multiple sources, transforms them, and routes them to hundreds of downstream tools (Amplitude, Mixpanel, data warehouses). See [Ch 43](../part-5-deployment-operations/43-analytics-attribution.md).

**Semver (Semantic Versioning)** — Versioning convention `MAJOR.MINOR.PATCH` where major = breaking changes, minor = new features, patch = bug fixes. Standard for npm packages. See [Ch 45](../part-5-deployment-operations/45-release-management.md).

**Server Action** — Next.js feature: async function marked with `"use server"` that runs on the server when called from client. Replaces API routes for mutations. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Server Component** — React component that runs only on the server. Zero client JS, direct database access, async by default. Default in Next.js App Router. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Server Timing** — HTTP header (`Server-Timing`) that exposes backend timing metrics (db query, auth check) to the browser's Performance API, enabling end-to-end performance debugging. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Server-Side Validation** — Validating form data on the server regardless of client-side checks. The authoritative validation layer since client-side checks can be bypassed. See [Ch 35](../part-3-state-data-communication/35-forms-at-scale.md).

**Session Management** — Tracking authenticated user sessions across requests via cookies, tokens, or server-side session stores. Includes expiry, rotation, and revocation. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Shadow Thread** — React Native thread running Yoga layout calculations. In Fabric, layout can be computed synchronously on any thread. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Size-limit** — Build-time tool that checks JavaScript bundle size against a performance budget and fails CI if limits are exceeded. See [Ch 44](../part-4-architecture-at-scale/44-performance-budgets.md).

**Skeleton Screen** — Loading placeholder showing the approximate shape of content before it loads, reducing perceived wait time compared to spinners. See [Ch 47](../part-4-architecture-at-scale/47-ux-paradigms.md).

**Skill (Agentic)** — In Claude Code, a reusable instruction file loaded on-demand that provides specialized knowledge or workflows to the AI agent. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

**Skia (react-native-skia)** — 2D graphics library bringing Skia (Chrome/Android's rendering engine) to React Native for custom drawing, shaders, and image processing. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Slot Pattern** — Component architecture pattern where a parent defines named insertion points (slots) that children fill with content, enabling flexible layout composition. See [Ch 48](../part-4-architecture-at-scale/48-component-architecture.md).

**Socket.io** — Library providing real-time bidirectional event-based communication with automatic transport fallback (WebSocket to long-polling). Includes rooms, namespaces, and reconnection. See [Ch 42](../part-3-state-data-communication/42-realtime-transport.md).

**Source Map** — File mapping bundled/minified code back to original source. Essential for debugging production crashes. Must be uploaded to error tracking services. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**SPF (Sender Policy Framework)** — Email authentication DNS record specifying which mail servers are authorized to send email on behalf of your domain. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**Srcset** — HTML `<img>` attribute listing multiple image sources at different widths or pixel densities, enabling the browser to choose the optimal image for the viewport. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**SSG (Static Site Generation)** — Pre-rendering pages at build time. Fastest possible TTFB but data can be stale. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**SSR (Server-Side Rendering)** — Rendering HTML on the server per request. Fresh data but slower TTFB than static. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Staged Rollout** — Releasing an update to a percentage of users, gradually increasing if metrics are healthy. Supported by EAS Update channels, feature flags, and app stores. See [Ch 45](../part-5-deployment-operations/45-release-management.md).

**staleTime** — TanStack Query option defining how long data is considered fresh. During staleTime, no background refetch occurs. Default: 0. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**State Machine** — Formal model of states and transitions. Libraries like XState prevent impossible states. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

**Statistical Significance** — The probability that an A/B test result is not due to random chance. Typically requires a p-value < 0.05 before declaring a winner. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**Statsig** — Feature flag and experimentation platform with built-in statistical analysis, auto-exposure logging, and a free tier. See [Ch 37](../part-5-deployment-operations/37-feature-flags.md).

**Streaming (SSR)** — Sending HTML to the browser in chunks as it's rendered on the server. Used with React Suspense for progressive page loading. See [Ch 25](../part-6-vercel-web/25-nextjs-app-router.md).

**Strict Mode** — React development wrapper that double-invokes effects and renders to catch side effect bugs. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**Storybook** — Component development environment for building, testing, and documenting UI components in isolation. See [Ch 16](../part-4-architecture-at-scale/16-design-systems.md).

**Subscription (GraphQL)** — GraphQL operation that maintains a persistent connection (typically via WebSocket) to receive real-time data updates from the server. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**Suspense** — React component for declarative loading states. Wraps components that suspend (lazy, data fetching) and shows a fallback. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

**SWC** — Rust-based JavaScript/TypeScript compiler. 20-70x faster than Babel. Used by Next.js and Turbopack. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Symbolication** — Process of translating memory addresses in crash reports back to human-readable function names and line numbers using debug symbols (dSYM/source maps). See [Ch 20](../part-5-deployment-operations/20-monitoring.md).

---

## T

**TanStack Query (React Query)** — Async state management library for server state. Handles caching, deduplication, background refetch, pagination, and optimistic updates. See [Ch 10](../part-3-state-data-communication/10-data-fetching.md).

**TestFlight** — Apple's beta testing platform. Distributes pre-release iOS builds to testers. EAS Submit can upload directly. See [Ch 6](../part-2-react-native-expo/06-eas-mastery.md).

**Throttle** — Technique that limits function execution to at most once per time interval. Used for scroll handlers and API rate limiting. See [Ch 0](../part-0-fundamentals/00-javascript-hard-parts.md).

**Thumbhash** — Compact image placeholder encoding (similar to blurhash) that also preserves aspect ratio and alpha channel information. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**Token (Design)** — See *Design Token*.

**TOTP (Time-based One-Time Password)** — MFA method generating a 6-digit code that rotates every 30 seconds based on a shared secret. Used by authenticator apps (Google Authenticator, Authy). See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Transactional Email** — Automated email triggered by a user action (password reset, order confirmation, invite). Must have high deliverability and fast delivery. See [Ch 39](../part-5-deployment-operations/39-email-comms.md).

**Transpiler** — Tool that converts code from one language/version to another (e.g., TypeScript → JavaScript, modern JS → ES5). Babel, SWC, tsc. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Tree Shaking** — Bundler optimization that eliminates unused exports from the final bundle. Requires ES modules (static `import`/`export`). See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Trigram Similarity** — Search technique comparing overlapping 3-character sequences between strings to find fuzzy matches. Built into PostgreSQL via `pg_trgm`. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

**tRPC** — End-to-end typesafe API framework. Client and server share TypeScript types with zero code generation. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**TTI (Time to Interactive)** — Time until the page is fully interactive (all event handlers attached, main thread idle). See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**TTFB (Time to First Byte)** — Time from request start to first byte of response. Measures server/CDN response speed. See [Ch 23](../part-5-deployment-operations/23-metrics.md).

**TurboModules** — React Native's new native module system using JSI for synchronous, lazy-loaded, type-safe native method calls. Replaces the Bridge-based NativeModules. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

**Turbopack** — Rust-based bundler by Vercel. Successor to Webpack for Next.js development. Incremental by design. See [Ch 5](../part-1-foundations/05-build-toolchain.md).

**Turborepo** — High-performance build system for monorepos. Content-addressed caching, parallel execution, task pipelines. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

**Turso** — SQLite-based edge database using libSQL. Provides embedded replicas for zero-latency reads at the edge with a central primary. See [Ch 34](../part-4-architecture-at-scale/34-database-orm.md).

**Type Guard** — TypeScript function or expression that narrows a type within a conditional block. `typeof`, `instanceof`, `in`, or custom `is` predicates. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Type Inference** — TypeScript's ability to determine types without explicit annotations. Reduces verbosity while maintaining safety. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Typesense** — Open-source, typo-tolerant search engine with built-in faceting and geo-search. Simpler alternative to Elasticsearch. See [Ch 38](../part-4-architecture-at-scale/38-search.md).

---

## U

**Universal Link (iOS) / App Link (Android)** — OS-level mechanism that opens a URL directly in your app instead of the browser. Requires server-side verification file. See [Ch 7](../part-2-react-native-expo/07-navigation.md).

**urql** — Lightweight, extensible GraphQL client for React with a plugin-based architecture (exchanges) for caching, auth, and subscriptions. See [Ch 46](../part-3-state-data-communication/46-graphql.md).

**useActionState** — React hook (React 19+) for handling form actions with pending state and error handling. Replaces `useFormState` and simplifies server action integration. See [Ch 35](../part-3-state-data-communication/35-forms-at-scale.md).

**useCallback** — React hook that memoizes a function reference. Prevents unnecessary re-renders when passed as a prop. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useEffect** — React hook for side effects: data fetching, subscriptions, DOM manipulation. Runs after render. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useFieldArray** — React Hook Form hook for managing dynamic lists of form fields (add, remove, reorder) while maintaining validation and registration. See [Ch 35](../part-3-state-data-communication/35-forms-at-scale.md).

**useMemo** — React hook that memoizes a computed value. Only recomputes when dependencies change. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useReducer** — React hook for complex state logic. Dispatch actions to a reducer function. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useRef** — React hook that creates a mutable ref object persisting across renders. Used for DOM refs and instance variables. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useState** — React hook for component-level state. Returns `[value, setter]` tuple. See [Ch 0c](../part-0-fundamentals/00c-react-fundamentals.md).

**useTransition** — React 18+ hook that marks state updates as non-urgent, allowing urgent updates (typing) to interrupt slow renders. See [Ch 3](../part-1-foundations/03-rendering-pipeline.md).

---

## V

**v0** — Vercel's AI-powered UI generation tool that creates React components from natural language prompts and screenshots. See [Ch 49](../part-6-vercel-web/49-agentic-ui-dev.md).

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

**WebAuthn** — W3C standard for passwordless authentication using public-key cryptography. Browsers expose it via `navigator.credentials`. Underpins passkeys. See [Ch 33](../part-5-deployment-operations/33-authentication.md).

**Webhook** — HTTP callback sent by a service when an event occurs. Stripe webhooks notify your server of payment events. Must be verified and handled idempotently. See [Ch 24](../part-5-deployment-operations/24-payments.md).

**Webhook-Triggered ISR** — Pattern where a CMS sends a webhook on content change, triggering on-demand ISR revalidation in Next.js so pages update without a full rebuild. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**WebP** — Image format developed by Google offering 25-35% smaller files than JPEG/PNG with support for transparency and animation. Broadly supported in modern browsers. See [Ch 23](../part-4-architecture-at-scale/23-image-content-optimization.md).

**WebSocket** — Full-duplex communication protocol over a single TCP connection. Used for real-time features (chat, live updates). See [Ch 12](../part-3-state-data-communication/12-offline-realtime.md).

**Worklet (Reanimated)** — JavaScript function that runs on the UI thread via a separate JS runtime. Enables 60fps animations without crossing the bridge. See [Ch 8](../part-2-react-native-expo/08-styling-animation.md).

**Workspace (pnpm/npm)** — Monorepo feature that links local packages together and hoists shared dependencies. See [Ch 15](../part-4-architecture-at-scale/15-monorepo.md).

---

## X

**XState** — State machine and statechart library for JavaScript. Prevents impossible states through formal modeling. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

---

## Y

**Yjs** — High-performance CRDT framework for building real-time collaborative applications. Supports shared types (text, array, map) with network-agnostic sync. See [Ch 41](../part-3-state-data-communication/41-realtime-collaboration.md).

**Yoga** — Cross-platform layout engine by Meta implementing a subset of CSS Flexbox. Used by React Native to compute layouts in C++. See [Ch 1](../part-1-foundations/01-react-native-internals.md).

---

## Z

**Zod** — TypeScript-first schema validation library. Runtime type checking that infers static types. Used with tRPC, React Hook Form, and API validation. See [Ch 4](../part-0-fundamentals/00d-typescript-mastery.md).

**Zero-Config** — Tools or frameworks that work without configuration files. Expo, Vercel, and Turbopack aim for zero-config defaults with escape hatches. See [Ch 5](../part-2-react-native-expo/05-expo-platform.md).

**Zustand** — Lightweight state management library for React. Single store, no boilerplate, works outside React. Ideal for client-side global state. See [Ch 9](../part-3-state-data-communication/09-state-management.md).

---

*Total: 301 terms. Last updated: 2026-04-07.*

> **See also:** [Appendix B: Cheat Sheet](./appendix-cheatsheet.md) | [Appendix C: Reading List](./appendix-reading-list.md)
