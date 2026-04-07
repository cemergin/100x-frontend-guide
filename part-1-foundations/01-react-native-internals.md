<!--
  CHAPTER: 1
  TITLE: React Native Architecture & Internals
  PART: I — Foundations
  PREREQS: None
  KEY_TOPICS: New Architecture, JSI, Fabric, TurboModules, Hermes, threading model, bridgeless mode, cold start, rendering pipeline
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 1: React Native Architecture & Internals

> **Part I — Foundations** | Prerequisites: None | Difficulty: Intermediate to Advanced

Here's the thing about React Native that most tutorials skip entirely: **you are not writing a web app that runs on a phone.** You are writing JavaScript that orchestrates a native rendering engine, communicates across thread boundaries through a serialization layer, and manages memory in a garbage-collected VM running on hardware that can range from a $1,200 iPhone to a $50 Android phone with 2GB of RAM.

Every performance bug you'll ever chase, every crash that makes no sense from the JavaScript layer, every animation that janks on low-end devices — the root cause lives in this chapter. The engineers who understand React Native's internals debug in hours what others spend weeks on. They make architecture decisions that prevent entire categories of problems. They look at a frame drop and know whether it's the JS thread, the UI thread, or the Shadow thread before they even open the profiler.

This chapter gives you that understanding. We're going to trace a component from JSX to pixels — through the JavaScript engine, across thread boundaries, through the layout system, and into the native rendering pipeline. By the end, you'll know how React Native *actually* works, not just how it appears to work.

### In This Chapter
- The Old Architecture (Bridge) — what it was, why it was slow
- The New Architecture — JSI, Fabric, TurboModules, Codegen
- Hermes: React Native's JavaScript Engine
- The Threading Model — UI, JS, and Shadow threads
- Bridgeless Mode — the final evolution
- Cold Start: What Actually Happens
- The Rendering Pipeline End-to-End
- Production Metrics: What Companies Actually Measured

### Related Chapters
- [Ch 3: The Rendering Pipeline] — React reconciliation and Fiber
- [Ch 5: Expo Platform] — how Expo builds on this architecture
- [Ch 13: Performance Optimization] — applying these internals to real performance problems
- [Ch 14: Profiling & Debugging] — using these internals to diagnose issues

---

## 1. THE OLD ARCHITECTURE: THE BRIDGE

Before we talk about where React Native is, we need to talk about where it was — because understanding the old architecture tells you exactly *why* the new one exists.

React Native launched in 2015 with an architecture centered on a concept called **the Bridge.** The Bridge was an asynchronous, serialized, batched message-passing system between two worlds: JavaScript (where your React code runs) and Native (where the actual UI components live).

Here's how it worked: your JavaScript code would describe UI changes — "create a View with these styles, put a Text inside it with this content" — and those descriptions would be serialized into JSON, queued into a batch, and sent across the Bridge to the native side. The native side would deserialize the JSON, create the corresponding native views (UIView on iOS, android.view.View on Android), lay them out, and render them.

Going the other direction: when a user tapped a button, the native side would serialize the touch event into JSON, send it across the Bridge to JavaScript, and your event handler would run.

**Every single interaction between JS and Native went through JSON serialization.**

This had several consequences:

### The Serialization Tax

Every data transfer — props, layout calculations, events, images — was serialized to JSON and back. For a simple screen, this was negligible. For a complex list with hundreds of items, or an animation that needs 60fps updates, the Bridge became the bottleneck. The serialization overhead added latency to every frame, and because the Bridge was asynchronous, there was no way to guarantee that a touch event and its visual response would happen in the same frame.

### The Async Gap

Because the Bridge was asynchronous, JavaScript couldn't synchronously read native values. Want to know the current scroll position? You send a message across the Bridge, wait for the response, and by the time you get it, the user has scrolled somewhere else. This made scroll-driven animations, gesture-based interactions, and layout measurements fundamentally difficult. Hacks like `measure()` callbacks and the `Animated` API's `useNativeDriver` option were workarounds for this fundamental limitation.

### The Single-Threaded Bottleneck

All Bridge messages went through a single queue. If your app was doing heavy JS computation (parsing a large JSON response, running business logic), the Bridge backed up. Touch events waited in line behind data processing. The UI felt unresponsive not because the native rendering was slow, but because the communication pipe was clogged.

### The Shadow Tree Overhead

Layout calculations (using Yoga, Facebook's cross-platform Flexbox implementation) happened on a dedicated Shadow Thread. But the results had to be sent back through the Bridge — again, serialized JSON. A layout pass that should take microseconds was padded with serialization overhead.

**The result:** React Native apps had a reputation for feeling "almost native but not quite." Animations stuttered. Lists were slower than their native counterparts. Cold starts were slower because the Bridge needed to bootstrap. The JavaScript-to-native boundary was a tax on every operation, and that tax compounded.

> **Discord's experience:** Discord's React Native app served millions of users, but the Bridge was a constant source of performance challenges. Their engineering team measured that on an iPhone 6, the serialization overhead for their message list added measurable latency to every scroll frame. This was one of the driving forces behind their investment in performance optimization — and eventually, the New Architecture.

---

## 2. THE NEW ARCHITECTURE

The New Architecture is not a minor upgrade. It is a ground-up reimagining of how JavaScript talks to native code. Meta started this effort in 2018 (internally called "Fabric" and "TurboModules"), and it's been the default for new React Native projects since 0.76 (late 2024).

The core insight: **eliminate the Bridge entirely.** Instead of serializing data to JSON and passing it through an asynchronous queue, let JavaScript call native functions directly — synchronously when needed — through a shared C++ layer.

Four pieces make this work:

### 2.1 JSI (JavaScript Interface)

JSI is the foundation of everything. It's a C++ API that lets JavaScript hold references to C++ objects and call methods on them directly. No serialization. No Bridge. No JSON.

Think of it this way: in the old architecture, JavaScript would say "please serialize this message and put it in the Bridge queue and when the native side gets around to it, deserialize it and create a View." With JSI, JavaScript directly calls `nativeView.setProps({ backgroundColor: 'red' })` and the call goes straight to the native layer through a C++ binding.

**Why this matters for you:**
- **Synchronous calls are now possible.** JavaScript can read native values in the same frame they're requested. Scroll positions, layout measurements, sensor data — available instantly.
- **No serialization overhead.** Data passes by reference through shared memory, not by value through JSON. This eliminates the largest single source of overhead in React Native.
- **Multiple JavaScript engines.** Because JSI is an abstraction over the JS engine, React Native is no longer tied to JavaScriptCore. This is what enabled Hermes.
- **Direct C++ interop.** Native modules can be written in C++ and shared across iOS and Android without platform-specific bridges.

```
Old Architecture:
  JS → JSON.stringify → Bridge Queue → JSON.parse → Native
  Native → JSON.stringify → Bridge Queue → JSON.parse → JS

New Architecture (JSI):
  JS → C++ binding → Native  (direct call, synchronous or async)
  Native → C++ binding → JS  (direct callback)
```

### 2.2 Fabric (New Rendering System)

Fabric is the new rendering system, built on top of JSI. It replaces the old "UIManager" that communicated through the Bridge.

In the old system, when you rendered a `<View>`, JavaScript would send a create command through the Bridge, the native side would create the view, and any subsequent updates went through the same serialized path. Layout was calculated on the Shadow Thread and results were sent back through the Bridge.

Fabric changes this fundamentally:

- **The Shadow Tree is now in C++.** Both JavaScript and native code can read and write the Shadow Tree directly through JSI. No serialization needed for layout calculations.
- **Synchronous layout.** JavaScript can request a layout measurement and get the result in the same frame. This is what makes features like `onLayout` reliable and what enables smooth, synchronized animations.
- **Priority-based rendering.** Fabric supports React's concurrent features — different parts of the UI can be rendered at different priorities. A user interaction can interrupt a low-priority background render, keeping the app responsive.
- **Immutable tree operations.** Fabric uses an immutable tree structure where updates create new tree nodes rather than mutating existing ones. This enables safe concurrent access from multiple threads.

**The measurable impact:** Shopify measured a 17-27% improvement in cold start times after migrating to the New Architecture. The removal of the serialization layer alone accounted for a significant portion of this improvement.

### 2.3 TurboModules

TurboModules are the New Architecture's replacement for Native Modules. Where old Native Modules were loaded eagerly at app startup (every registered module was initialized, whether you needed it or not), TurboModules are loaded lazily — initialized only when first accessed.

This has a direct impact on cold start time. If your app registers 40 native modules (a realistic number for a production app with analytics, crash reporting, maps, camera, push notifications, etc.), the old architecture initialized all 40 at startup. TurboModules initialize only the ones you actually use during startup — maybe 8-10 — and defer the rest.

**Additional TurboModule benefits:**
- **Type-safe interfaces.** TurboModules use Codegen to generate type-safe C++ bindings from TypeScript or Flow definitions. The interface contract is enforced at compile time, not at runtime.
- **Synchronous method calls.** TurboModules can expose synchronous methods that JavaScript calls directly through JSI, without going through the Bridge. Critical for performance-sensitive operations.
- **Shared C++ code.** The C++ implementation can be shared across iOS and Android, reducing platform-specific code.

### 2.4 Codegen

Codegen is the build-time tool that generates the C++ glue code between JavaScript and native. You define your native module interface in TypeScript (or Flow), and Codegen generates:
- C++ header files with the interface definition
- Platform-specific adapter code (Objective-C++ for iOS, Java/JNI for Android)
- Type-safe bindings that enforce the contract at compile time

This means: if your TypeScript says a native method takes `{ userId: string, count: number }`, the generated C++ code will enforce that contract. Pass the wrong types and you get a compile error, not a runtime crash.

```tsx
// MyModule.ts — the source of truth
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getDeviceName(): string;                    // synchronous
  fetchUserData(userId: string): Promise<Object>;  // async
}

export default TurboModuleRegistry.getEnforcing<Spec>('MyModule');
```

Codegen reads this and generates the entire C++ → Objective-C/Java bridge at build time. Zero runtime overhead for the type checking. Zero JSON serialization.

---

## 3. HERMES: REACT NATIVE'S JAVASCRIPT ENGINE

Hermes is Meta's JavaScript engine, purpose-built for React Native. It replaced JavaScriptCore as the default engine starting with React Native 0.70, and understanding how it differs explains several performance characteristics you'll observe in production.

### Why a Custom Engine?

JavaScriptCore (JSC) was designed for Safari — a desktop/mobile browser where the JS engine has time to do JIT (Just-In-Time) compilation and where startup time is amortized across a long browser session. React Native apps have different constraints:

- **Cold start matters enormously.** Users expect your app to launch in under 2 seconds. Every millisecond of engine initialization is felt directly.
- **Memory is scarce.** A $50 Android phone has 2-3GB of total RAM, shared across the OS, background apps, and your app. JSC's JIT compiler consumes significant memory for compiled code caches.
- **Predictable performance > peak performance.** Users care more about consistent 60fps than occasional 120fps with dropped frames.

### How Hermes Works

Hermes uses **Ahead-of-Time (AOT) compilation** instead of Just-In-Time compilation. During your build process (when you run `eas build` or `npx expo prebuild`), your JavaScript source code is compiled to Hermes bytecode — a compact, optimized binary format. At runtime, Hermes loads and executes this bytecode directly, without needing to parse and compile JavaScript.

```
Traditional JS Engine (JSC):
  App Launch → Download JS Bundle → Parse JS → Compile to Machine Code → Execute
  (all of this happens at runtime, on the user's device)

Hermes:
  Build Time: JS Source → Hermes Compiler → Bytecode (.hbc file)
  App Launch: Load Bytecode → Execute
  (parsing and compilation already happened at build time)
```

**The impact on cold start:** The expensive parse-and-compile phase moves from runtime to build time. Hermes apps start executing meaningful code significantly faster. Meta reported a 2x improvement in TTI (Time-to-Interactive) on mid-range Android devices when switching from JSC to Hermes.

### Hermes Memory Model

Hermes uses a **generational garbage collector** with three generations:
- **Young generation (YG):** Small, frequently collected. New allocations go here. Most short-lived objects (event handlers, intermediate computation results) are collected before they leave YG.
- **Old generation (OG):** Objects that survive multiple YG collections are promoted here. Collected less frequently.
- **Large object space:** Objects above a size threshold get their own allocation. Avoids fragmenting the main heap.

**Why this matters for you:** If you see your app's memory growing steadily over time, the issue is almost certainly objects leaking from YG to OG — closures capturing references they shouldn't, state that accumulates without cleanup, event listeners that are never removed. The profiling chapter (Ch 14) covers how to diagnose this using Hermes's built-in heap snapshots.

### Hermes Bytecode and Bundle Size

Hermes bytecode files (`.hbc`) are typically 20-30% smaller than equivalent JavaScript bundles because:
- Comments and whitespace are eliminated during compilation (not just minification — actual compilation)
- Identifiers are converted to numeric references
- The bytecode format is more compact than minified JS

This means your OTA updates (EAS Update) ship smaller payloads, and your app's initial download is smaller.

---

## 4. THE THREADING MODEL

React Native operates on three primary threads. Understanding what runs where — and what happens when they compete — is the difference between apps that feel native and apps that stutter.

### 4.1 The JS Thread

This is where your React code executes. Component rendering, state updates, event handlers, business logic, TanStack Query's cache management — all of it runs on the JS thread.

**The critical constraint:** The JS thread is single-threaded. JavaScript has no true parallelism (Web Workers exist but are rarely used in RN). If your event handler does 100ms of computation, the JS thread is blocked for 100ms. During that time:
- No other event handlers can fire
- No state updates can be processed
- No new renders can begin
- If an animation is driven from JS, it drops frames

**Rule of thumb:** Keep JS thread work under 16ms per frame (60fps target) or 8ms per frame (120fps target on ProMotion displays). Anything longer and you're dropping frames.

### 4.2 The UI Thread (Main Thread)

The UI thread is the native main thread — it's where UIKit (iOS) and Android's View system do their work. Touch event dispatch, native animations, and final pixel rendering all happen here.

**Key insight:** Native animations (Reanimated worklets, `useNativeDriver: true`) run on the UI thread, independent of JavaScript. This is why a Reanimated animation stays smooth even when the JS thread is busy — the animation runs entirely on the UI thread in a worklet, never crossing the thread boundary.

### 4.3 The Shadow Thread (Background Thread)

The Shadow Thread handles layout calculations using Yoga (the Flexbox engine). When your component tree changes, the new layout is calculated on this thread.

In the New Architecture, the Shadow Thread's work is more tightly integrated with both JS and UI threads through JSI, reducing the latency of layout results reaching the native rendering.

### 4.4 Thread Interaction

```
User taps a button:
  UI Thread: Touch event dispatched
      ↓
  JS Thread: onPress handler executes → setState → re-render
      ↓
  Shadow Thread: New layout calculated (Yoga)
      ↓
  UI Thread: Native views updated with new layout
      ↓
  GPU: Pixels composited to screen
```

**Where jank comes from:**
- JS Thread blocked → event handlers delayed, state updates queued, renders skipped
- UI Thread blocked → native animations stutter, touch events lost, screen frozen
- Shadow Thread blocked → layout calculations delayed, visual inconsistency between content and layout

**The golden rule:** Keep heavy work off the JS thread. Use `InteractionManager.runAfterInteractions()` for deferred work. Use Reanimated worklets for animations. Use native modules for CPU-intensive computation. Use `requestAnimationFrame` for frame-aligned updates.

---

## 5. BRIDGELESS MODE

Bridgeless mode is the final piece of the New Architecture puzzle. Starting with React Native 0.76, new projects have no Bridge at all — the entire communication layer is JSI.

What this means:
- **No fallback path.** In earlier versions of the New Architecture, the Bridge was still present as a fallback for modules that hadn't been migrated. Bridgeless mode removes it entirely.
- **Faster interop.** Without the Bridge overhead, even old-style modules that are wrapped with a compatibility layer perform better.
- **Smaller binary.** The Bridge code is no longer compiled into the app, slightly reducing the binary size.
- **Simpler mental model.** There is one communication path: JSI. No need to reason about whether a given operation goes through the Bridge or JSI.

### Migration Reality

If you're on Expo SDK 52+, you're already on the New Architecture with bridgeless mode by default. Expo's managed workflow handles the migration transparently — your JavaScript code doesn't change. The migration challenge is primarily for native modules that haven't been updated to support TurboModules.

Common libraries and their New Architecture status (as of early 2026):
- **react-native-reanimated** — fully supported since v3
- **react-native-gesture-handler** — fully supported
- **react-native-screens** — fully supported
- **@react-native-firebase** — fully supported since v19
- **react-native-maps** — supported via community fork
- **react-native-camera** (old) — not supported; use `expo-camera` instead

---

## 6. COLD START: WHAT ACTUALLY HAPPENS

When a user taps your app icon, here's the sequence — and where the time goes:

### iOS Cold Start Sequence

```
1. PROCESS CREATION          (~50-100ms)
   OS creates the app process, loads the binary, runs dyld (dynamic linker)

2. NATIVE MODULE INIT        (~100-300ms, New Arch: ~50-100ms)
   TurboModules: only used modules initialized (lazy loading)
   Old Bridge: ALL modules initialized (eager loading)

3. HERMES ENGINE STARTUP     (~30-60ms)
   Load Hermes bytecode from disk
   Initialize the runtime environment
   Set up the global scope

4. JS BUNDLE EXECUTION       (~100-500ms, varies wildly by app size)
   Execute your JavaScript bundle
   Run top-level imports and module initialization
   Create the React root and trigger first render

5. REACT FIRST RENDER        (~50-200ms)
   Reconcile the initial component tree
   Calculate layout (Yoga)
   Create native views

6. NATIVE VIEW MOUNTING      (~30-80ms)
   Mount native views to the screen
   GPU composites the first frame
   → User sees content (TTI)
```

**Total cold start (well-optimized app):** 400-800ms on modern iPhone
**Total cold start (unoptimized app):** 2-5+ seconds on mid-range Android

### What You Can Control

| Phase | Optimization | Impact |
|-------|-------------|--------|
| Native Module Init | Use TurboModules (lazy loading) | -100-200ms |
| Hermes Startup | Already using AOT bytecode via Hermes | Built-in |
| JS Bundle Execution | Reduce bundle size, lazy imports, tree shaking | -100-500ms |
| React First Render | Minimize initial component tree, defer below-fold content | -50-200ms |
| Native View Mounting | Use `<Suspense>` for progressive rendering | -20-50ms |

### A Million Monkeys Case Study

The A Million Monkeys team measured their migration to the New Architecture:
- **Android cold start reduction: 24%** — from TurboModules lazy loading and JSI elimination of Bridge serialization
- **Migration timeline: 12 days with 2 developers** — primarily auditing and updating third-party native modules
- **No JavaScript code changes required** — the migration was entirely in the native layer

---

## 7. THE RENDERING PIPELINE END-TO-END

Let's trace a state update from `setState` to pixels on screen. This is the sequence that runs 60 times per second when your UI is animating.

### Step 1: State Update (JS Thread)

```tsx
const [count, setCount] = useState(0);
// User taps → setCount(1)
```

React queues the state update. In concurrent mode, this update is assigned a priority (user interactions are high priority, background data fetches are low priority).

### Step 2: Reconciliation (JS Thread)

React's Fiber reconciler walks the component tree, comparing the previous render with the new render:
- Same component type, same key → update existing fiber
- Different type or key → unmount old, mount new
- Props changed → mark for update

The reconciler produces a list of changes (the "effect list") — which components need to update, which need to mount/unmount, which need layout effects.

### Step 3: Shadow Tree Update (Shadow Thread via JSI)

The effect list is applied to the Shadow Tree (C++ in-memory representation of the layout):
- New nodes are created for new components
- Existing nodes are updated with new props/styles
- Removed components' nodes are deleted

Yoga runs the Flexbox layout algorithm on the updated Shadow Tree, calculating the absolute position and size of every element.

### Step 4: Commit (UI Thread)

The calculated layout is committed to the native view hierarchy:
- New native views are created (`UIView`, `android.view.View`)
- Existing views are updated with new props (text content, colors, sizes)
- Views are positioned according to the Yoga layout results
- Removed views are detached from the hierarchy

### Step 5: Composite (GPU)

The native rendering system composites all the layers:
- iOS: Core Animation composites CALayers
- Android: RenderThread composites through the hardware-accelerated pipeline

The final frame is sent to the display.

### What Makes This Fast (New Architecture)

In the old architecture, steps 2 → 3 and 3 → 4 each crossed the Bridge with JSON serialization. In the New Architecture:
- Step 2 → 3: Direct JSI call from JS to the C++ Shadow Tree. No serialization.
- Step 3 → 4: Direct JSI call from C++ to native views. No serialization.

The serialization tax was applied twice per render cycle in the old architecture. With JSI, it's zero. At 60fps, that's 120 serialization operations per second that no longer happen.

---

## 8. PRODUCTION METRICS: WHAT COMPANIES ACTUALLY MEASURED

These aren't theoretical — they're from engineering blog posts, conference talks, and published case studies.

| Company | Migration | Metric | Before | After | Improvement |
|---------|-----------|--------|--------|-------|-------------|
| Shopify | New Architecture | Cold start | Baseline | -17-27% | TurboModules + Fabric |
| Meta | Hermes engine | TTI (Android) | Baseline | ~2x faster | AOT bytecode |
| Discord | Various optimizations | TTI (iPhone 6) | ~8s | ~4.5s | -3,500ms |
| Discord | Message list optimization | Parse time | Baseline | 90% faster | Custom serialization |
| A Million Monkeys | New Architecture | Android cold start | Baseline | -24% | 12-day migration |
| Shopify | Bundle optimization | Bundle size | 50MB | 11MB | -78% |

### The Meta-Lesson

Every company that published performance improvements reported the same pattern: **the biggest wins came from understanding the internals, not from applying surface-level tips.** Shopify didn't speed up their cold start by tweaking React components — they migrated to TurboModules. Discord didn't fix their scroll performance by adding `React.memo` everywhere — they understood the Bridge bottleneck and worked around it. Meta didn't improve TTI by code-splitting — they built an entirely new JavaScript engine.

The internals aren't academic. They're the debugging superpower that separates engineers who fix symptoms from engineers who fix causes.

---

## 9. PUTTING IT ALL TOGETHER

Here's the mental model you should carry with you for the rest of this guide:

```
┌──────────────────────────────────────────────────────────┐
│                     YOUR APP CODE                         │
│              (TypeScript, React Components)                │
└───────────────────────────┬──────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │  HERMES (JS)  │  ← AOT bytecode, GC, single-threaded
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │      JSI      │  ← C++ bridge-less interface
                    └───┬───────┬───┘
                        │       │
              ┌─────────▼─┐ ┌──▼──────────┐
              │  FABRIC    │ │ TURBOMODULES│
              │ (Rendering)│ │ (Native API)│
              └─────┬──────┘ └──────┬──────┘
                    │               │
              ┌─────▼───────────────▼──────┐
              │    NATIVE VIEW HIERARCHY    │
              │  (UIKit / Android Views)    │
              └─────────────┬──────────────┘
                            │
                    ┌───────▼───────┐
                    │      GPU      │  → Pixels on screen
                    └───────────────┘
```

**The key takeaways:**
1. **JSI eliminated the Bridge.** Everything is direct C++ calls now. This is the single biggest architectural improvement in React Native's history.
2. **Hermes compiles ahead of time.** Your JS is bytecode before it reaches the device. This is why cold starts are fast.
3. **Three threads, one orchestration.** JS thread for logic, Shadow thread for layout, UI thread for rendering. Keep them unblocked.
4. **TurboModules are lazy.** They load when needed, not at startup. This directly reduces cold start time.
5. **Fabric is synchronous.** Layout measurements are available in the same frame. This is what makes animations smooth.

With this foundation, everything else in this guide — Expo's abstractions, state management patterns, performance optimization, profiling — will make visceral sense instead of being arbitrary rules to memorize.

---

*Next: [Chapter 2 — Browser Rendering & Web Fundamentals](./02-browser-rendering.md) — the web side of the rendering equation.*
