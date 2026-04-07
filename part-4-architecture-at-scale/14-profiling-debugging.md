<!--
  CHAPTER: 14
  TITLE: Profiling & Debugging
  PART: IV — Architecture at Scale
  PREREQS: Chapters 1, 13
  KEY_TOPICS: Hermes profiling, Chrome DevTools for RN, Android Studio profiler, Xcode Instruments, Perfetto, Reactotron, memory leaks, Hermes heap snapshots, crash analysis, SIGABRT, SIGSEGV, native-layer debugging, Flipper alternatives
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 14: Profiling & Debugging

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 1, 13 | Difficulty: Intermediate to Advanced

There's a moment in every React Native developer's career — usually at 2 AM, usually the night before a release — when something goes wrong and `console.log` is not going to cut it. The app is crashing on Android but not iOS. Or there's a memory leak that only manifests after 15 minutes of use. Or the animations are dropping frames on a Samsung Galaxy A13 but running smoothly on your iPhone 15. Or, the worst of all, you're getting a native crash with a stack trace full of symbols you don't recognize — `SIGABRT`, `SIGSEGV`, something in `libc++abi.dylib`.

This is the chapter that prepares you for that moment. Not by teaching you a single tool, but by giving you a mental model for diagnosis and a toolkit for every layer of the React Native stack — from the JavaScript you wrote, through the bridge, down to the native code you didn't write but are responsible for debugging.

The best engineers I've worked with share a common trait: they don't guess. They measure. They don't say "I think the list is slow" — they say "the JS thread is blocked for 45ms per frame during scroll because the cell renderer is triggering a layout recalculation." That precision comes from knowing which profiler to open, what to measure, and how to interpret the results.

### In This Chapter
- The Debugging Mental Model — which tool for which layer
- Hermes Profiling — CPU profiling on the actual JS engine
- Chrome DevTools for React Native
- Reactotron — the React Native-specific inspector
- Android Studio Profiler — CPU, memory, network, energy
- Xcode Instruments — Time Profiler, Allocations, Leaks
- Perfetto — system-level trace analysis
- Memory Leak Detection with Hermes Heap Snapshots
- Crash Analysis — SIGABRT, SIGSEGV, and the native layer
- Native-Layer Debugging — when the problem isn't in JavaScript
- Flipper Alternatives — the post-Flipper debugging landscape
- Building a Debugging Workflow

### Related Chapters
- [Ch 1: React Native Architecture & Internals] — understanding the layers you're debugging
- [Ch 3: The Rendering Pipeline] — understanding what the profiler is showing you
- [Ch 13: Performance Optimization] — applying profiling insights to fix problems
- [Ch 23: Mobile Metrics That Matter] — production metrics that tell you where to profile

---

## 1. THE DEBUGGING MENTAL MODEL

React Native is a multi-layered system. The first step in debugging is identifying which layer the problem is in. This determines which tool you reach for.

```
┌──────────────────────────────────────────────────────────┐
│  LAYER 5: REACT COMPONENTS                                │
│  Symptoms: Unnecessary re-renders, slow lists,            │
│            incorrect state                                │
│  Tools: React DevTools, why-did-you-render, Reactotron    │
├──────────────────────────────────────────────────────────┤
│  LAYER 4: JAVASCRIPT ENGINE (Hermes)                      │
│  Symptoms: JS thread blocking, slow computations,         │
│            excessive GC pauses                            │
│  Tools: Hermes profiler, Chrome DevTools (CPU profile)    │
├──────────────────────────────────────────────────────────┤
│  LAYER 3: BRIDGE / JSI                                    │
│  Symptoms: Serialization overhead, async delays,          │
│            crossing-boundary costs                        │
│  Tools: Systrace, custom Bridge logging                   │
├──────────────────────────────────────────────────────────┤
│  LAYER 2: NATIVE UI (Fabric / UIKit / Android Views)      │
│  Symptoms: Layout thrashing, overdraw, slow native        │
│            animations, GPU bottlenecks                    │
│  Tools: Xcode Instruments, Android Studio Profiler,       │
│         GPU Overdraw, Layout Inspector                    │
├──────────────────────────────────────────────────────────┤
│  LAYER 1: PLATFORM (OS, memory, threads, signals)         │
│  Symptoms: Crashes (SIGABRT, SIGSEGV), OOM kills,        │
│            ANRs, thread deadlocks                         │
│  Tools: Xcode crash logs, Android logcat, Perfetto,       │
│         native debuggers (lldb, gdb)                      │
└──────────────────────────────────────────────────────────┘
```

### The Diagnostic Decision Tree

```
Problem: App is slow
  │
  ├── During scrolling/animation?
  │     ├── JS thread busy? → Hermes profiler → find the expensive function
  │     ├── UI thread busy? → Android Studio / Xcode Instruments → check layout
  │     └── Both idle? → GPU issue → check overdraw, image sizes
  │
  ├── During data loading?
  │     ├── Network slow? → Reactotron network inspector → check payloads
  │     ├── JSON parsing slow? → Hermes profiler → consider streaming parser
  │     └── Too many re-renders? → React DevTools → check memoization
  │
  └── On cold start?
        ├── Hermes bytecode compilation? → Check pre-compiled bundles
        ├── Large JS bundle? → Bundle analyzer → split/lazy-load
        └── Native module initialization? → Xcode/Android profiler → check startup

Problem: App is crashing
  │
  ├── JavaScript error? → Error boundary + React DevTools
  ├── Native crash? → See "Crash Analysis" section
  ├── OOM kill? → Memory profiler → heap snapshots
  └── ANR (Android)? → Thread analysis → find the blocked thread
```

---

## 2. HERMES PROFILING

Hermes is the JavaScript engine that ships with React Native (since 0.70 by default). It's optimized for mobile: it compiles JS to bytecode ahead of time, uses less memory than V8, and starts faster. But it also has different performance characteristics than V8, which means profiling techniques from web development don't always apply.

### Capturing a Hermes CPU Profile

```typescript
// Method 1: Programmatic (in code)
import { HermesInternal } from 'react-native';

// Start profiling
HermesInternal?.enableSampling?.();

// ... do the thing you want to profile ...

// Stop profiling and save the trace
const profilePath = HermesInternal?.disableSampling?.();
console.log('Profile saved to:', profilePath);
```

```bash
# Method 2: Via CLI (recommended)
# Start your app with profiling enabled
npx react-native start --experimental-debugger

# Or use Expo
npx expo start --dev-client

# In the Dev Menu (shake device or Cmd+D):
# → "Start/Stop Sampling Profiler"
```

### Reading the Hermes Profile

The profile is saved as a `.cpuprofile` file that you can open in Chrome DevTools:

1. Open Chrome → `chrome://inspect` → Open dedicated DevTools for Node.js
2. Go to the "Performance" tab
3. Load the `.cpuprofile` file

Or use the Hermes profile transformer:

```bash
npx hermes-profile-transformer <profile.cpuprofile> -o chrome-profile.json
```

### What to Look For

```
┌─────────────────────────────────────────────────────────┐
│  FLAME CHART ANALYSIS                                    │
│                                                          │
│  Wide bars at the top → Functions called frequently      │
│  Tall stacks → Deep call chains (possible recursion)     │
│  Gaps between bars → Idle time (good during animations)  │
│                                                          │
│  RED FLAGS:                                               │
│  ─────────                                               │
│  • Any function taking >16ms (blocks one frame at 60fps) │
│  • JSON.parse on large strings                           │
│  • Array.sort on large arrays during scroll              │
│  • Regular expressions on long strings                   │
│  • Object spread (...obj) in tight loops                 │
│  • Hermes GC pauses >5ms                                 │
│                                                          │
│  HERMES-SPECIFIC:                                        │
│  • Look for "GC" markers — pauses for garbage collection │
│  • Hermes uses a generational GC: short Gen0 pauses are  │
│    normal, long Gen1+ pauses indicate memory pressure    │
│  • Function compilation time (first call is slower)      │
└─────────────────────────────────────────────────────────┘
```

### Hermes-Specific Performance Gotchas

```typescript
// GOTCHA 1: Hermes is slower at certain operations than V8
// Object.keys() on large objects is slower in Hermes
// Workaround: use for...in loops

// Slow in Hermes
const keys = Object.keys(largeObject);
for (const key of keys) { /* ... */ }

// Faster in Hermes
for (const key in largeObject) {
  if (largeObject.hasOwnProperty(key)) { /* ... */ }
}

// GOTCHA 2: Regex performance differs
// Hermes uses its own regex engine, which can be slower
// for certain patterns. Pre-compile regexes outside loops.

// Slow
function search(items: string[], query: string) {
  return items.filter(item => new RegExp(query, 'i').test(item));
}

// Fast
function search(items: string[], query: string) {
  const regex = new RegExp(query, 'i');
  return items.filter(item => regex.test(item));
}

// GOTCHA 3: Proxy performance
// Hermes supports Proxies but they're slower than in V8
// Libraries that use heavy Proxy usage (some state managers)
// may be slower on Hermes
```

---

## 3. CHROME DEVTOOLS FOR REACT NATIVE

With the New Architecture and Hermes, Chrome DevTools connects directly to your React Native app via CDP (Chrome DevTools Protocol).

### Connecting

```bash
# For Expo dev builds
npx expo start --dev-client
# Press 'j' in the terminal to open Chrome DevTools

# For bare React Native
npx react-native start --experimental-debugger
# The debugger URL is printed in the terminal
```

### The Console

Beyond `console.log`, Chrome DevTools gives you:

```typescript
// Structured logging
console.table(arrayOfObjects); // Renders as a table
console.group('Network Request');
console.log('URL:', url);
console.log('Status:', response.status);
console.log('Time:', duration);
console.groupEnd();

// Timing
console.time('fetchProducts');
const products = await fetchProducts();
console.timeEnd('fetchProducts'); // "fetchProducts: 234ms"

// Count occurrences
function MyComponent() {
  console.count('MyComponent rendered'); // Tracks cumulative renders
  // ...
}

// Assert (log only if condition is false)
console.assert(items.length > 0, 'Items array is empty!');
```

### The Sources Tab

Set breakpoints, step through code, inspect variables — all the standard debugger features. But there's a key difference from web: **you're debugging Hermes bytecode, not V8 JavaScript.** Source maps are essential for readable stack traces.

### The Performance Tab

Record a performance trace to see:
- JavaScript execution timeline
- Function call frequency and duration
- GC pauses
- Long tasks (>50ms)

```
Pro tip: Record BEFORE you start the interaction, stop AFTER it completes.
Don't record for more than 10-15 seconds — the trace becomes unwieldy.
```

### The Memory Tab

Take heap snapshots to analyze memory usage:

1. Click "Take snapshot"
2. Use the app for a while
3. Click "Take snapshot" again
4. Compare snapshots to find growing allocations
5. Filter by "Objects allocated between Snapshot 1 and Snapshot 2"

---

## 4. REACTOTRON

Reactotron is an inspector for React Native (and React) apps. It's not a replacement for Chrome DevTools — it's a complement, focused on the specific things React Native developers need to inspect.

### Setup

```bash
npm install -D reactotron-react-native reactotron-apisauce
```

```typescript
// reactotron-config.ts (only included in development)
import Reactotron from 'reactotron-react-native';
import { QueryClientManager, reactotronReactQuery } from 'reactotron-react-query';

const queryClientManager = new QueryClientManager({});

Reactotron.configure({
  name: 'MyApp',
  // For physical devices, use your computer's IP
  host: '192.168.1.100',
})
  .useReactNative({
    asyncStorage: false, // We use MMKV
    networking: {
      ignoreUrls: /symbolicate|generate_204/,
    },
    editor: false,
    errors: { veto: () => false },
    overlay: false,
  })
  .use(reactotronReactQuery(queryClientManager))
  .connect();

// Make it available globally in dev
if (__DEV__) {
  console.tron = Reactotron;
}

export default Reactotron;
```

### What Reactotron Does Well

**1. Network Inspection:**
See every API request with timing, headers, body, and response — without any proxy setup.

**2. State Inspection:**
Subscribe to your Zustand/Redux stores and see every state change with diffs.

**3. TanStack Query Inspection:**
With the React Query plugin, see every query, its status, cache state, and refetch timing.

**4. Benchmark:**
```typescript
// Time a specific operation
Reactotron.benchmark('parseProducts')
  .step('fetch')
  .step('parse JSON')
  .step('transform')
  .stop('done');
```

**5. Custom Commands:**
```typescript
// Create a custom button in Reactotron that triggers app behavior
Reactotron.onCustomCommand({
  command: 'clearCache',
  handler: () => {
    queryClient.clear();
    storage.clearAll();
  },
  title: 'Clear All Caches',
  description: 'Clears TanStack Query and MMKV caches',
});

Reactotron.onCustomCommand({
  command: 'simulateOffline',
  handler: () => {
    onlineManager.setOnline(false);
  },
  title: 'Go Offline',
  description: 'Simulates offline mode',
});
```

**6. Overlay:**
Display performance metrics or debug info directly on the app screen.

### When to Use Reactotron vs Chrome DevTools

```
┌──────────────────────────────┬────────────────────────────────┐
│  REACTOTRON                   │  CHROME DEVTOOLS               │
├──────────────────────────────┼────────────────────────────────┤
│  Network request inspection   │  JavaScript debugging          │
│  State change timeline        │  CPU profiling                 │
│  TanStack Query monitoring    │  Memory heap snapshots         │
│  Custom dev commands          │  Console (advanced)            │
│  Benchmarking                 │  Source maps / breakpoints     │
│  Image overlay                │  Performance timeline          │
└──────────────────────────────┴────────────────────────────────┘

Use both. They complement each other.
```

---

## 5. ANDROID STUDIO PROFILER

When the problem is on the native side — or when you need to understand CPU, memory, network, and energy usage at the OS level — Android Studio's profiler is indispensable.

### Opening the Profiler

1. Open your project's `android/` folder in Android Studio
2. Run the app on a device or emulator
3. View → Tool Windows → Profiler (or: View → App Inspection)
4. Click the `+` button to start a profiling session

### CPU Profiler

The CPU profiler shows you what every thread is doing:

```
┌──────────────────────────────────────────────────────────┐
│  KEY THREADS TO MONITOR:                                  │
│                                                          │
│  mqt_js  — The JavaScript thread (Hermes)                │
│  mqt_native_modules — Native module calls                │
│  main (UI thread) — UI rendering and layout              │
│  RenderThread — GPU rendering                            │
│  OkHttp — Network requests                               │
│  MMKV — MMKV I/O operations                              │
│                                                          │
│  WHAT TO LOOK FOR:                                        │
│  • mqt_js blocking for >16ms (frame drops)               │
│  • main thread blocking (ANR if >5s)                     │
│  • Excessive context switching between threads           │
│  • GC pauses on the JS thread                            │
└──────────────────────────────────────────────────────────┘
```

### Recording a Method Trace

```
1. In the CPU profiler, click "Record"
2. Select "Java/Kotlin Method Trace" for native code
   or "Sample Java Methods" for lower overhead
3. Perform the problematic action in the app
4. Click "Stop"
5. Analyze the flame chart
```

For React Native specifically, look for:

```
// Common native bottlenecks in React Native apps:

1. View creation during fast scroll
   → Look for: android.view.View.<init> in hot path
   → Fix: RecyclerView / FlashList instead of ScrollView

2. Image decoding on UI thread
   → Look for: BitmapFactory.decode* on main thread
   → Fix: Use expo-image (decodes on background thread)

3. Layout calculation storms
   → Look for: android.view.View.layout called hundreds of times
   → Fix: Reduce nested views, use Fabric's flat view hierarchy

4. Bridge serialization (old architecture)
   → Look for: ReadableNativeMap, WritableNativeMap in traces
   → Fix: Migrate to New Architecture (JSI)
```

### Memory Profiler

The memory profiler shows real-time heap usage, allocations, and GC events:

```
┌──────────────────────────────────────────────────────────┐
│  MEMORY PROFILER GUIDE                                    │
│                                                          │
│  Java/Kotlin Heap — Native objects (Views, Bitmaps)      │
│  Native Heap — C++ allocations (Hermes, Yoga, Fabric)    │
│  Graphics — GPU memory for rendering                     │
│  Stack — Thread stacks                                   │
│  Code — Loaded DEX files and native libraries            │
│                                                          │
│  RED FLAGS:                                               │
│  • Continuously increasing Java heap (memory leak)       │
│  • Large Native heap growth (Hermes leak or image leak)  │
│  • Graphics memory spikes (large images, many layers)    │
│  • Java heap approaching device limit (OOM incoming)     │
│                                                          │
│  RECORDING ALLOCATIONS:                                   │
│  1. Click "Record" in memory profiler                    │
│  2. Navigate through your app                            │
│  3. Click "Stop"                                         │
│  4. Sort by "Remaining Size" to find leaks              │
│  5. Look for objects with many instances that should     │
│     have been collected                                  │
└──────────────────────────────────────────────────────────┘
```

### Network Profiler

See every network request, response, and timing — including requests from native modules that don't go through JS fetch:

```
What the network profiler tells you that JavaScript logging doesn't:
1. Total data transferred (including headers, compression)
2. Connection reuse (HTTP/2 multiplexing vs new connections)
3. DNS resolution time
4. TLS handshake time
5. Time to first byte (TTFB)
6. Native-initiated requests (image loads, analytics, crash reporters)
```

### Energy Profiler

Mobile-specific but critical for user experience:

```
┌──────────────────────────────────────────────────────────┐
│  ENERGY PROFILER INSIGHTS                                 │
│                                                          │
│  CPU usage → Is your app doing unnecessary background    │
│              computation?                                │
│                                                          │
│  Network activity → Are you polling too frequently?      │
│                     Can you batch requests?              │
│                                                          │
│  Location → Are you using fine location when coarse      │
│             would suffice?                               │
│                                                          │
│  Wake locks → Is your app preventing the device from     │
│               sleeping?                                  │
│                                                          │
│  Alarms → Are you scheduling unnecessary wake-ups?      │
│                                                          │
│  A well-behaved app should show minimal energy usage     │
│  when in the background.                                 │
└──────────────────────────────────────────────────────────┘
```

---

## 6. XCODE INSTRUMENTS

Instruments is Apple's profiling suite, and it's remarkably powerful once you know your way around it. For React Native iOS debugging, it's the definitive tool.

### Launching Instruments

```
1. Open your project's ios/ folder in Xcode
2. Product → Profile (Cmd + I) to build and launch Instruments
3. Or: Open Instruments directly from /Applications/Xcode.app → Open Developer Tool → Instruments
```

### Time Profiler

The Time Profiler shows you where CPU time is spent, across all threads:

```
KEY SETTINGS:
  • Recording Mode: Deferred (lower overhead)
  • Sample interval: 1ms for detailed, 10ms for overview
  • Check "Record Waiting Threads" to see thread blocking

FILTERING FOR REACT NATIVE:
  • In the call tree, search for "hermes" to see JS engine activity
  • Search for "RCT" to see React Native framework code
  • Search for your app name to see your native modules
  • Invert the call tree to see "heaviest" functions first
```

### Allocations Instrument

Track every memory allocation and find leaks:

```
HEAP GROWTH ANALYSIS:
1. Start recording
2. Perform an action (e.g., navigate to a screen)
3. Undo the action (navigate back)
4. Repeat 5-10 times
5. Stop recording
6. Filter by "Heap Growth" — these are potential leaks

If navigating to Screen A and back increases the heap each time,
something on Screen A is leaking. Look at the allocations that
appear during navigation and never get deallocated.
```

### Leaks Instrument

Specifically detects retain cycles and leaked objects:

```
COMMON REACT NATIVE LEAKS ON iOS:

1. Timer leaks
   → setInterval/setTimeout not cleared on unmount
   → Fix: useEffect cleanup function

2. EventEmitter leaks
   → addListener without corresponding removeListener
   → Fix: Return cleanup from useEffect

3. Native module retain cycles
   → Objective-C blocks capturing self strongly
   → Fix: Use [weak self] in closures

4. Image cache over-retention
   → Large images staying in memory cache
   → Fix: expo-image with proper cache policy

5. WebView leaks
   → WebView instances not properly destroyed
   → Fix: Unmount WebView when not visible
```

### Core Animation Instrument

For animation performance:

```
WHAT TO MEASURE:
  • FPS (should be 60 or 120 on ProMotion devices)
  • Offscreen rendering (yellow overlay in simulator)
  • Blended layers (red overlay)
  • Hit testing

COMMON ANIMATION ISSUES:
  • Shadow rendering without rasterization
    → Add: layer.shouldRasterize = YES on static views with shadows
  • Large image scaling
    → Resize images to display size before rendering
  • Transparent overlapping views
    → Reduce opacity layers, use opaque backgrounds
```

### System Trace (Metal System Trace)

For GPU-level debugging:

```
When to use System Trace:
  • Frame drops that aren't caused by CPU bottlenecks
  • Complex layout hierarchies with many views
  • Heavy use of blur effects, shadows, or transparency
  • Custom native rendering (Canvas, OpenGL, Metal)
```

---

## 7. PERFETTO

Perfetto is Google's system-level tracing tool. It captures traces from the Android kernel, system services, and your app simultaneously, giving you the most complete picture of what's happening at every level.

### Capturing a Perfetto Trace

```bash
# On a connected Android device
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.perfetto-trace \
<<EOF
buffers: {
  size_kb: 63488
  fill_policy: RING_BUFFER
}
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "power/suspend_resume"
      ftrace_events: "power/cpu_frequency"
    }
  }
}
data_sources: {
  config {
    name: "linux.process_stats"
    process_stats_config {
      scan_all_processes_on_start: true
    }
  }
}
duration_ms: 10000
EOF

# Pull the trace
adb pull /data/misc/perfetto-traces/trace.perfetto-trace .
```

### Viewing Perfetto Traces

Open `https://ui.perfetto.dev` and drag your trace file in.

### What Perfetto Shows That Other Tools Don't

```
┌──────────────────────────────────────────────────────────┐
│  PERFETTO UNIQUE INSIGHTS                                 │
│                                                          │
│  1. THREAD SCHEDULING                                     │
│     See exactly when each thread was running, sleeping,  │
│     or waiting. Find cases where the JS thread is ready  │
│     but the CPU scheduler isn't giving it time.          │
│                                                          │
│  2. CPU FREQUENCY SCALING                                 │
│     Mobile CPUs dynamically adjust frequency to save     │
│     battery. Your app might be slow because the CPU is   │
│     in a low-power state, not because your code is slow. │
│                                                          │
│  3. CROSS-PROCESS INTERACTIONS                            │
│     See how your app interacts with system services      │
│     (SurfaceFlinger for rendering, Binder for IPC).      │
│                                                          │
│  4. FRAME TIMELINE                                        │
│     Track each frame from when it was requested to when  │
│     it was displayed. See exactly which phase took too   │
│     long: input handling, animation, layout, draw, GPU.  │
│                                                          │
│  5. MEMORY EVENTS                                         │
│     Low-memory killer events, page faults, memory        │
│     pressure — things that can cause unexpected slowdowns│
│     or kills that don't show up in application profilers.│
└──────────────────────────────────────────────────────────┘
```

### React Native-Specific Perfetto Analysis

```
Look for these in your Perfetto traces:

1. JS Thread (mqt_js) Scheduling:
   If the JS thread is frequently preempted (interrupted by the scheduler),
   your frame timing will be inconsistent even if your code is fast.

   Solution: On rooted devices, set CPU affinity to keep the JS thread
   on a performance core. In production, this is a device limitation.

2. Choreographer Frame Misses:
   Search for "Choreographer" to see frame timing. Each frame should
   complete within 16.6ms (60fps) or 8.3ms (120fps).

3. SurfaceFlinger Composition:
   If frames are being composed by SurfaceFlinger slowly, the issue is
   GPU-side, not CPU-side. Look for composition taking >4ms.

4. Binder Transactions:
   Heavy Binder IPC can block the UI thread. If you see binder_transaction
   on the main thread, a native module might be doing synchronous IPC.
```

---

## 8. MEMORY LEAK DETECTION WITH HERMES HEAP SNAPSHOTS

Memory leaks in React Native are sneaky. They don't crash your app immediately — they slowly consume memory over minutes or hours until the OS kills your app. The user just sees the app "randomly" crash, which is the worst kind of bug.

### Taking Hermes Heap Snapshots

```typescript
// In your development build:
if (__DEV__ && global.HermesInternal) {
  // Take a heap snapshot
  const filename = global.HermesInternal.takeHeapSnapshot();
  console.log('Heap snapshot saved:', filename);
}
```

Or from the Chrome DevTools Memory tab when connected to Hermes.

### The Three-Snapshot Technique

This is the most reliable method for finding leaks:

```
STEP 1: Take Snapshot A (baseline)
  - App is in a known state (e.g., home screen)

STEP 2: Perform the suspicious action
  - Navigate to a screen, open a modal, start a process

STEP 3: Undo the action
  - Navigate back, close the modal, stop the process

STEP 4: Force garbage collection
  - In Chrome DevTools, click the trash can icon
  - Or: global.gc() if exposed

STEP 5: Take Snapshot B

STEP 6: Repeat steps 2-5 several times

STEP 7: Take Snapshot C

COMPARE: Objects in C that weren't in A are likely leaked.
The more times you repeat, the more obvious the leak
(leaked objects accumulate linearly).
```

### Common React Native Memory Leaks

**Leak 1: Unmounted Component Subscriptions**

```typescript
// LEAKS: The interval keeps running after unmount
function BadComponent() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setInterval(() => {
      setCount(c => c + 1); // Closure keeps component in memory
    }, 1000);
  }, []);

  return <Text>{count}</Text>;
}

// FIXED: Clean up the interval
function GoodComponent() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);

    return () => clearInterval(id); // Cleanup!
  }, []);

  return <Text>{count}</Text>;
}
```

**Leak 2: Event Listener Accumulation**

```typescript
// LEAKS: New listener added on every render, never removed
function BadListener() {
  const handleResize = () => { /* ... */ };
  Dimensions.addEventListener('change', handleResize);
  // ...
}

// FIXED: Add in useEffect, remove in cleanup
function GoodListener() {
  useEffect(() => {
    const subscription = Dimensions.addEventListener('change', handleResize);
    return () => subscription.remove();
  }, []);
  // ...
}
```

**Leak 3: Closure Captures**

```typescript
// LEAKS: The closure captures `largeData` and prevents GC
function processData() {
  const largeData = fetchLargeDataset(); // 50MB
  const summary = computeSummary(largeData);

  return {
    getSummary: () => summary,
    // This closure captures `largeData` even though it doesn't use it!
    // JavaScript closures capture the entire scope, not just used variables
    // (though some engines optimize this — Hermes doesn't always)
  };
}

// FIXED: Explicitly null out large references
function processData() {
  let largeData = fetchLargeDataset();
  const summary = computeSummary(largeData);
  largeData = null; // Allow GC

  return {
    getSummary: () => summary,
  };
}
```

**Leak 4: React Navigation Screen Cache**

```typescript
// React Navigation keeps screens in memory when navigating forward
// This is by design (for back navigation), but can accumulate

// Mitigation: Use the `unmountOnBlur` option for heavy screens
<Stack.Screen
  name="HeavyScreen"
  component={HeavyScreen}
  options={{
    // Unmount when not focused (saves memory, but loses state)
    unmountOnBlur: true,
  }}
/>

// Better: Clean up heavy resources when screen loses focus
function HeavyScreen() {
  const isFocused = useIsFocused();

  useEffect(() => {
    if (!isFocused) {
      // Release heavy resources (image caches, video players, etc.)
      releaseHeavyResources();
    }
  }, [isFocused]);
}
```

**Leak 5: Native Module Leaks**

```typescript
// Native modules can hold references to JS objects
// If the native module isn't properly cleaning up, objects leak

// Common culprits:
// - Camera modules holding frame buffers
// - Map views holding marker data
// - WebView holding page content
// - Audio/Video players holding media buffers

// Always check that native module cleanup runs:
useEffect(() => {
  camera.start();
  return () => {
    camera.stop();
    camera.release(); // <-- Don't forget native resource cleanup
  };
}, []);
```

### Memory Budget Guidelines

```
┌──────────────────────────────────────────────────────────┐
│  MOBILE MEMORY BUDGETS (JavaScript heap)                  │
│                                                          │
│  Low-end Android (2-3GB RAM):                            │
│    Target: <100MB JS heap                                │
│    Warning: >150MB                                       │
│    OOM likely: >200MB                                    │
│                                                          │
│  Mid-range Android (4-6GB RAM):                          │
│    Target: <150MB JS heap                                │
│    Warning: >250MB                                       │
│    OOM likely: >350MB                                    │
│                                                          │
│  High-end Android / iOS (8GB+ RAM):                      │
│    Target: <200MB JS heap                                │
│    Warning: >400MB                                       │
│    OOM likely: >600MB                                    │
│                                                          │
│  Note: These are JS heap only. Total app memory includes │
│  native views, images, native modules, and GPU memory.   │
└──────────────────────────────────────────────────────────┘
```

---

## 9. CRASH ANALYSIS: SIGABRT, SIGSEGV, AND THE NATIVE LAYER

Native crashes are the most intimidating bugs in React Native development. The stack trace is full of symbols you didn't write, the error message is cryptic, and `console.log` can't help you. But native crashes follow patterns, and once you recognize the patterns, they become diagnosable.

### Understanding Signal Types

```
┌──────────────────────────────────────────────────────────┐
│  SIGNAL     MEANING                   COMMON CAUSE        │
│                                                          │
│  SIGABRT    Abort (intentional crash) Assertion failed,   │
│             The app asked to crash.   unhandled exception, │
│                                       abort() called      │
│                                                          │
│  SIGSEGV    Segmentation fault        Null pointer deref, │
│             Bad memory access.        use-after-free,     │
│                                       buffer overflow     │
│                                                          │
│  SIGBUS     Bus error                 Misaligned memory   │
│             Bad memory alignment.     access, corrupted   │
│                                       mmap'd file         │
│                                                          │
│  SIGFPE     Floating point exception  Division by zero    │
│             Arithmetic error.         (in native code)    │
│                                                          │
│  SIGKILL    Killed by OS              OOM (out of memory),│
│             Can't be caught.          watchdog timeout,   │
│                                       thermal shutdown    │
│                                                          │
│  SIGTRAP    Trap (debugger)           Breakpoint hit,     │
│             Debug break.              __builtin_trap()    │
└──────────────────────────────────────────────────────────┘
```

### Reading a Native Crash Log (iOS)

```
Exception Type:  EXC_CRASH (SIGABRT)
Exception Codes: 0x0000000000000000, 0x0000000000000000

Triggered by Thread:  0

Thread 0 Crashed:
0   libsystem_kernel.dylib          __pthread_kill + 8
1   libsystem_pthread.dylib         pthread_kill + 268
2   libsystem_c.dylib               abort + 180
3   libc++abi.dylib                 __cxa_bad_cast + 0
4   libc++abi.dylib                 demangling_terminate_handler() + 320
5   libobjc.A.dylib                 _objc_terminate() + 160
6   libc++abi.dylib                 std::__terminate(void (*)()) + 16
7   libc++abi.dylib                 __cxxabiv1::failed_throw(...) + 36
8   libc++abi.dylib                 __cxa_throw + 140
9   hermes                          facebook::hermes::HermesRuntime::... + 256
10  MyApp                           -[RCTBridge handleError:] + 128
```

**How to read this:**

```
Frame 0-2: The system is aborting the process (this is the mechanism, not the cause)
Frame 3-8: C++ exception handling (an exception was thrown and not caught)
Frame 9: Hermes (the JS engine) threw an exception
Frame 10: React Native's bridge tried to handle it

→ This is likely an unhandled JavaScript exception that propagated to native.
→ Look at your JavaScript error logs for the actual error message.
```

### Reading a Native Crash Log (Android)

```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'samsung/a13/a13:13/TP1A.220624.014/A135...'
pid: 12345, tid: 12367, name: mqt_js  >>> com.myapp <<<
uid: 10234
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
    r0  00000000  r1  00000001  r2  00000000  r3  00000000
backtrace:
    #00 pc 001a2f4c  /data/app/.../lib/arm64-v8a/libhermes.so (hermes::vm::..+124)
    #01 pc 001b3e28  /data/app/.../lib/arm64-v8a/libhermes.so (hermes::vm::..+456)
    #02 pc 00043210  /data/app/.../lib/arm64-v8a/libjsi.so
    #03 pc 00012abc  /data/app/.../lib/arm64-v8a/libfabrjni.so
```

**How to read this:**

```
signal 11 (SIGSEGV) → Segmentation fault (bad memory access)
code 1 (SEGV_MAPERR) → The address 0x0 was accessed → NULL pointer dereference
Thread: mqt_js → The JavaScript thread crashed

Frame #00-#01: Inside Hermes VM → JS engine crashed
Frame #02: JSI layer → The JS-to-native interface
Frame #03: Fabric JNI → Native UI rendering

→ This is likely a Hermes bug or a native module passing bad data to Hermes.
→ Check: recent native module updates, Hermes version, device-specific issues.
```

### Symbolication

Crash logs are useless without symbols. Here's how to symbolicate:

```bash
# iOS: Use atos (Address to Symbol)
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x104000000 0x10401a2f4c

# iOS: Or use Xcode Organizer
# Window → Organizer → Crashes → Select crash → Symbolicate

# Android: Use ndk-stack
adb logcat | ndk-stack -sym path/to/unstripped/libs

# Android: Or addr2line
$ANDROID_NDK/toolchains/aarch64-linux-android-4.9/prebuilt/darwin-x86_64/bin/aarch64-linux-android-addr2line \
  -f -C -e path/to/libhermes.so 0x001a2f4c
```

### Common Native Crash Patterns

```
┌─────────────────────────────────────────────────────────────┐
│  PATTERN: SIGABRT in libc++abi.dylib                         │
│  Cause: Uncaught C++ exception                               │
│  Common triggers:                                            │
│    - NSInternalInconsistencyException (UIKit thread violation)│
│    - Attempting to update UI from background thread           │
│    - Invalid layout calculation (NaN values in Yoga)         │
│  Fix: Check for runOnUI / dispatch_async(dispatch_get_main_) │
│                                                              │
│  PATTERN: SIGSEGV in libhermes.so                            │
│  Cause: Hermes accessed invalid memory                       │
│  Common triggers:                                            │
│    - Corrupted bytecode bundle                               │
│    - Native module returning unexpected types via JSI        │
│    - Memory pressure causing Hermes mmap to fail             │
│  Fix: Update Hermes, check native module JSI calls           │
│                                                              │
│  PATTERN: SIGKILL (no crash log)                             │
│  Cause: OS killed the app                                    │
│  Common triggers:                                            │
│    - Out of memory (check Jetsam log on iOS)                 │
│    - Watchdog timeout (main thread blocked >20s)             │
│    - Background execution time exceeded                      │
│  Fix: Profile memory usage, check for main thread blocking   │
│                                                              │
│  PATTERN: SIGABRT with "Thread 1: signal SIGABRT"            │
│  in RCTFatal / RCTLogError                                   │
│  Cause: React Native caught a fatal error                    │
│  Common triggers:                                            │
│    - Invalid props passed to native component                │
│    - Native module method called with wrong arguments        │
│    - TurboModule codegen mismatch                            │
│  Fix: Check Metro logs for the actual JS error message       │
│                                                              │
│  PATTERN: EXC_BAD_ACCESS in objc_msgSend                     │
│  Cause: Message sent to deallocated object                   │
│  Common triggers:                                            │
│    - Race condition in native module lifecycle                │
│    - View accessed after being removed from hierarchy        │
│    - Delegate/callback pointing to deallocated object        │
│  Fix: Enable Zombie Objects in Xcode scheme diagnostics      │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. NATIVE-LAYER DEBUGGING

Sometimes you need to step into the native layer — Objective-C/Swift on iOS, Java/Kotlin on Android — to understand what's happening.

### Debugging Native Code in Xcode

```
1. Open ios/MyApp.xcworkspace in Xcode
2. Set breakpoints in Objective-C/Swift files
3. Run the app from Xcode (not from Metro)
4. When the breakpoint hits, you can:
   - Inspect native variables
   - Step through native code
   - Examine the view hierarchy (Debug → View Debugging → Capture View Hierarchy)
   - Check thread states
```

### Debugging Native Code in Android Studio

```
1. Open the android/ folder in Android Studio
2. Add breakpoints in Java/Kotlin files
3. Run → Attach Debugger to Android Process → Select your app
4. When the breakpoint hits, you can:
   - Inspect Java objects
   - Evaluate expressions
   - Use the Layout Inspector (Tools → Layout Inspector)
   - Check thread dumps (Run → Get Thread Dump)
```

### Common Native Debugging Scenarios

**Scenario 1: Native Module Not Working**

```
Symptoms: Native module method returns undefined or doesn't call back
Debug approach:
1. Add a breakpoint at the native method entry point
2. Verify the method is being called
3. Check parameter types (JSI types → native types)
4. Step through the native implementation
5. Check the return value conversion
```

**Scenario 2: View Not Rendering**

```
Symptoms: Component renders in JS but nothing appears on screen
Debug approach:
1. Xcode: Debug → View Debugging → Capture View Hierarchy
   Android: Tools → Layout Inspector
2. Check if the native view exists in the hierarchy
3. Check its frame (is it 0x0? Is it offscreen?)
4. Check its hidden/visibility property
5. Check its z-order (is it behind another view?)
```

**Scenario 3: Native Crash in Third-Party Library**

```
Symptoms: Crash in a native library you didn't write
Debug approach:
1. Get the symbolicated crash log
2. Identify the library (e.g., libhermes.so, RNReanimated)
3. Check the library's GitHub issues for the crash signature
4. Try reproducing with the library's debug build
5. If reproducible, file a detailed issue with:
   - Device model and OS version
   - Library version
   - Symbolicated crash log
   - Minimal reproduction
```

### The lldb Cheat Sheet (iOS)

```
# At an lldb prompt in Xcode:

# Print a variable
p myVariable
po myObject  # Print with description (Objective-C objects)

# Print the view hierarchy
expr -l objc++ -O -- [UIApplication.sharedApplication.keyWindow recursiveDescription]

# Print React Native view hierarchy
expr -l objc++ -O -- [RCTUIManager viewRegistry]

# Find the address of a crashed object
image lookup --address 0x10401a2f4c

# Get all threads
thread list

# Get the backtrace of a specific thread
thread backtrace 3

# Break on all Objective-C exceptions
breakpoint set -E objc

# Break on a specific C++ exception type
breakpoint set -E c++ -O "NSException"
```

---

## 11. FLIPPER ALTERNATIVES: THE POST-FLIPPER LANDSCAPE

Flipper was React Native's official debugging tool for several years. Meta deprecated it in favor of the built-in Chrome DevTools debugger (available since React Native 0.73). If you're starting a new project, don't use Flipper. If you're maintaining an existing project with Flipper, here's the migration path:

### What Replaced Flipper

```
┌────────────────────────────┬────────────────────────────────┐
│  FLIPPER FEATURE            │  REPLACEMENT                   │
├────────────────────────────┼────────────────────────────────┤
│  JS Debugger                │  Chrome DevTools (built-in)    │
│  Network Inspector          │  Reactotron / Charles Proxy    │
│  Layout Inspector           │  Xcode View Debugging /        │
│                             │  Android Layout Inspector      │
│  React DevTools             │  Standalone React DevTools     │
│  Database Inspector         │  Reactotron / native tools     │
│  Shared Preferences Viewer  │  MMKV Viewer (Reactotron)     │
│  Performance Monitor        │  Xcode Instruments /           │
│                             │  Android Studio Profiler       │
│  Custom Plugins             │  Reactotron custom commands    │
└────────────────────────────┴────────────────────────────────┘
```

### Setting Up the Modern Debugging Stack

```bash
# 1. Chrome DevTools (built-in, no setup needed)
npx expo start --dev-client
# Press 'j' to open

# 2. React DevTools (standalone)
npx react-devtools
# Components tab: inspect component tree, props, state
# Profiler tab: record and analyze render performance

# 3. Reactotron (see section 4 for setup)
brew install --cask reactotron  # macOS

# 4. Charles Proxy (for advanced network debugging)
# Download from charlesproxy.com
# Configure device to proxy through Charles
# See HTTPS traffic with SSL proxying
```

### React DevTools Profiler

The React DevTools Profiler deserves special attention because it shows you something no other tool can: **why components re-rendered.**

```
USING THE PROFILER:
1. Open React DevTools
2. Go to the Profiler tab
3. Click "Record"
4. Perform the interaction you want to profile
5. Click "Stop"

WHAT YOU SEE:
- Each commit (state update that caused a render)
- Which components re-rendered in each commit
- Why they re-rendered (state change, parent render, context change)
- How long each component took to render

SETTINGS (important!):
- Enable "Record why each component rendered while profiling"
  → Settings gear → Profiler → Check the box
- Enable "Highlight updates when components render"
  → Settings gear → General → Check the box
```

---

## 12. BUILDING A DEBUGGING WORKFLOW

Theory is useless without practice. Here's the workflow I recommend:

### The Quick Check (< 2 minutes)

When something seems wrong, start here:

```
1. Open Metro terminal — any red/yellow warnings?
2. Check console output — any error messages?
3. React DevTools → Components tab — is state what you expect?
4. Reactotron → Network tab — are requests succeeding?
```

### The Performance Investigation (15-30 minutes)

When the app is slow:

```
1. Enable "Highlight updates" in React DevTools
   → Watch for unexpected re-renders during the slow interaction

2. Record a Hermes CPU profile during the interaction
   → Look for long tasks (>16ms) on the JS thread

3. If JS thread looks fine, check native:
   → iOS: Time Profiler in Instruments
   → Android: CPU profiler in Android Studio

4. If CPU is fine, check memory:
   → Take heap snapshots before and after
   → Look for memory pressure causing GC pauses

5. If memory is fine, check GPU:
   → iOS: Core Animation instrument
   → Android: GPU rendering profiler (Developer Options → GPU overdraw)
```

### The Crash Investigation (30-60 minutes)

When the app crashes:

```
1. Get the crash log
   → iOS: Xcode Organizer → Crashes
   → Android: adb logcat or Play Console → Crashes
   → Both: Sentry / Crashlytics / Datadog

2. Symbolicate (if not already done by crash reporter)

3. Identify the crash type (signal)
   → SIGABRT: Check for unhandled exceptions
   → SIGSEGV: Check for null pointers, use-after-free
   → SIGKILL: Check memory usage, background time

4. Identify the layer
   → JavaScript: Error boundary should have caught it
   → Hermes: Check Hermes version, bytecode integrity
   → Native module: Check recent updates to the module
   → Platform: Check OS version, device model

5. Reproduce
   → Same device model? Same OS version? Same locale?
   → Try on the lowest-end device you support

6. Fix and verify
   → Add a regression test if possible
   → Monitor crash rate after fix ships
```

### The Memory Leak Investigation (1-2 hours)

When the app gets slower over time or crashes after extended use:

```
1. Establish baseline
   → Take heap snapshot on fresh app start
   → Note the total JS heap size

2. Reproduce the suspected leak
   → Navigate to a screen, then navigate back — 10 times
   → Open and close a modal — 10 times
   → Scroll through a long list — for 5 minutes

3. Take another heap snapshot

4. Compare snapshots
   → Sort by "Objects allocated between snapshots"
   → Look for growing counts of specific object types
   → Follow the retainer chain to find what's holding references

5. Common fixes
   → Missing useEffect cleanup
   → EventEmitter listener not removed
   → Timer not cleared
   → Closure capturing large objects

6. Verify the fix
   → Repeat steps 2-4 with the fix applied
   → Heap growth should stabilize
```

---

## CHAPTER SUMMARY

Profiling and debugging React Native requires comfort with multiple tools across multiple layers:

- **Hermes profiler** for JavaScript CPU profiling on the actual engine your app uses
- **Chrome DevTools** for breakpoints, console, memory snapshots, and performance timelines
- **Reactotron** for React Native-specific inspection (network, state, TanStack Query)
- **Android Studio Profiler** for native CPU, memory, network, and energy analysis
- **Xcode Instruments** for Time Profiler, Allocations, Leaks, and Core Animation
- **Perfetto** for system-level traces that show thread scheduling and kernel interactions
- **Hermes heap snapshots** for the three-snapshot memory leak detection technique
- **Crash log analysis** using signal types, symbolication, and pattern recognition

The key insight: **don't guess, measure.** Every performance claim should be backed by a number from a profiler. Every crash should be analyzed from a symbolicated stack trace. Every "it feels slow" should be traced to a specific thread, function, and millisecond count.

The engineers who debug efficiently aren't the ones who know the most tools — they're the ones who know which tool to reach for first.

---

*Next: [Chapter 15: Monorepo Architecture]*
*Previous: [Chapter 13: Performance Optimization]*