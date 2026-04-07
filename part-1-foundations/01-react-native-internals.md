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

<details>
<summary><strong>TL;DR</strong></summary>

- The old Bridge architecture serialized every JS-to-native call as JSON; the New Architecture (JSI, Fabric, TurboModules) eliminates the bridge entirely with synchronous C++ bindings
- JSI lets JavaScript hold direct references to C++ host objects, enabling synchronous native calls -- this is what makes Reanimated worklets and fast gesture handling possible
- Hermes compiles your JS to bytecode ahead of time, cutting cold start time nearly in half compared to JavaScriptCore
- React Native uses three threads (JS, UI, Shadow); knowing which thread owns a bottleneck is the first step to fixing any performance problem
- Bridgeless mode is the final evolution -- zero JSON serialization, zero async gaps, and the default for new projects since RN 0.76

</details>

Here's the thing about React Native that most tutorials skip entirely: **you are not writing a web app that runs on a phone.** You are writing JavaScript that orchestrates a native rendering engine, communicates across thread boundaries through a serialization layer, and manages memory in a garbage-collected VM running on hardware that can range from a $1,200 iPhone to a $50 Android phone with 2GB of RAM.

Every performance bug you'll ever chase, every crash that makes no sense from the JavaScript layer, every animation that janks on low-end devices — the root cause lives in this chapter. The engineers who understand React Native's internals debug in hours what others spend weeks on. They make architecture decisions that prevent entire categories of problems. They look at a frame drop and know whether it's the JS thread, the UI thread, or the Shadow thread before they even open the profiler.

This chapter gives you that understanding. We're going to trace a component from JSX to pixels — through the JavaScript engine, across thread boundaries, through the layout system, and into the native rendering pipeline. By the end, you'll know how React Native *actually* works, not just how it appears to work.

### In This Chapter
- The Old Architecture (Bridge) — what it was, why it was slow, and the JSON tax in code
- The New Architecture — JSI, Fabric, TurboModules, Codegen
- Hermes: React Native's JavaScript Engine — bytecode format, GC internals, memory budgets
- The Threading Model — UI, JS, and Shadow threads, with profiler interpretation
- Bridgeless Mode — the final evolution
- Cold Start: What Actually Happens — millisecond-level breakdown
- The Rendering Pipeline End-to-End — a single tap traced through every layer
- Old Architecture vs New Architecture — comprehensive comparison
- Production War Stories — real debugging scenarios where internals saved the day
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

### 1.1 The Bridge Message Format

To appreciate why this was slow, you need to see what actually went over the wire. When JavaScript told the native side to create a view, the Bridge didn't send a compact binary message. It sent JSON. Real JSON. Every single frame.

Here's a simplified version of what a single `UIManager.createView` call looked like on the Bridge:

```json
{
  "module": "UIManager",
  "method": "createView",
  "args": [
    42,
    "RCTView",
    1,
    {
      "backgroundColor": -1,
      "flex": 1,
      "flexDirection": "column",
      "padding": 16,
      "marginTop": 8,
      "borderRadius": 4,
      "shadowColor": -16777216,
      "shadowOffset": { "width": 0, "height": 2 },
      "shadowOpacity": 0.25,
      "shadowRadius": 3.84,
      "elevation": 5
    }
  ]
}
```

That's one view. A single card component. Now imagine a FlatList rendering 50 items on screen, each with a card containing an image, title, subtitle, and two action buttons. That's roughly:

- 50 cards x ~8 views per card = 400 `createView` calls
- Each call serialized to JSON (averaging ~200-500 bytes per message)
- All batched and sent across the Bridge
- All deserialized on the native side

Let's look at the actual serialization cost in JavaScript:

```javascript
// This is approximately what the Bridge did internally for EACH native call.
// In production, RN batched these, but the serialization cost was the same.

function sendBridgeMessage(module, method, args) {
  // Step 1: Serialize to JSON — this is the expensive part
  const jsonString = JSON.stringify([module, method, args]);
  
  // Step 2: The JSON string gets copied to a shared buffer
  // that the native side reads from
  nativeBridge.enqueue(jsonString);
  
  // Step 3: At the end of the JS frame, flush all queued messages
  // The native side JSON.parse()s every single one
}

// A single style update triggered this chain:
sendBridgeMessage('UIManager', 'updateView', [
  42,               // viewTag
  'RCTView',        // viewName
  { opacity: 0.5 }  // changed props — serialized to JSON
]);
```

The problem wasn't any single message. The problem was the *volume*. During a scroll on a complex list, the Bridge could process hundreds of messages per frame. Let's measure the actual cost:

```javascript
// Measuring Bridge serialization overhead (simplified benchmark)

const singleViewProps = {
  backgroundColor: '#FFFFFF',
  flex: 1,
  flexDirection: 'column',
  padding: 16,
  marginTop: 8,
  borderRadius: 4,
  overflow: 'hidden',
};

// Time to serialize a single view's props
console.time('single');
JSON.stringify(singleViewProps);
console.timeEnd('single');
// ~0.002ms on iPhone 14, ~0.01ms on budget Android

// But during a FlatList scroll rendering 50 new items:
const messages = [];
for (let i = 0; i < 400; i++) {
  messages.push({
    module: 'UIManager',
    method: 'createView',
    args: [i, 'RCTView', 1, singleViewProps],
  });
}

console.time('batch');
const serialized = JSON.stringify(messages);
console.timeEnd('batch');
// ~2-4ms on iPhone 14, ~8-15ms on budget Android
// That's JUST serialization. Deserialization on native side costs the same.
// Total: 4-8ms on iPhone, 16-30ms on budget Android
// That's 25-50% of your 16ms frame budget, JUST for serialization.
```

And that was the *optimistic* case. The Bridge batched messages, which amortized the overhead. But if you had a large batch (a new screen rendering for the first time), the serialization cost could blow past your frame budget entirely.

### 1.2 The Async Gap

Because the Bridge was asynchronous, JavaScript couldn't synchronously read native values. This created a class of problems that no amount of optimization could solve:

```javascript
// THE ASYNC GAP PROBLEM — Old Architecture

// You want to know the scroll position to decide what to render
scrollView.measure((x, y, width, height, pageX, pageY) => {
  // By the time this callback fires, the user has scrolled further.
  // This measurement is STALE. You're always one frame behind.
  
  // Even worse: if you setState based on this measurement,
  // that triggers ANOTHER async Bridge round-trip for the re-render.
  // You're now TWO frames behind the actual scroll position.
});

// THE ANIMATION PROBLEM
// JS-driven animations had to send updates across the Bridge every frame:
//   Frame 1: JS calculates opacity: 0.0, sends to Bridge
//   Frame 2: Bridge delivers 0.0 to native, JS calculates 0.033, sends to Bridge
//   Frame 3: Bridge delivers 0.033 to native, JS calculates 0.066, sends to Bridge
//   ...
// If the JS thread was busy (computing business logic, parsing JSON),
// the animation frames would queue up and you'd see visible stuttering.

// THE WORKAROUND: useNativeDriver
Animated.timing(opacity, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true,  // "Please run this animation on native, not JS"
}).start();
// This worked for simple animations (opacity, transform)
// But it couldn't animate layout properties (width, height, marginTop)
// because Yoga layout ran on a different thread with Bridge-mediated results.
```

The async gap was the root cause of React Native's reputation for "almost native but not quite" animations. The fundamental architecture made it impossible to synchronize JavaScript logic with native rendering in the same frame.

### 1.3 The Single-Threaded Bridge Queue

All Bridge messages went through a single queue. This meant that unrelated operations competed for the same pipe:

```
Frame N timeline (old architecture, problematic case):

JS Thread:
  [0ms]   API response arrives, parse 200KB JSON          (8ms)
  [8ms]   Process data, update 3 state values              (2ms)
  [10ms]  Re-render list with 50 items                     (6ms)
  [16ms]  ← FRAME DEADLINE PASSED

Bridge Queue:
  [0ms]   Backed up with 50 createView messages from previous frame
  [8ms]   New batch of 150 messages from list render arriving
  [16ms]  Touch event from user tap WAITING IN LINE behind view updates
  [24ms]  Touch event finally delivered to JS thread
  [24ms]  ← User perceives 24ms delay on their button tap

Native Thread:
  [0ms]   Processing previous frame's view updates
  [16ms]  Still processing — 150 messages to deserialize
  [24ms]  Finally idle, but next batch already arriving
```

The Bridge was a funnel. Everything had to pass through it, and when it backed up, the entire app felt unresponsive. Touch events waited behind data processing. Animations competed with network response handling. Layout measurements queued behind image loading.

### 1.4 The Shadow Tree Overhead

Layout calculations happened on a dedicated Shadow Thread using Yoga (Facebook's cross-platform Flexbox implementation). But the results had to be sent back through the Bridge:

```
Layout calculation path (old architecture):

1. JS Thread: "Create a View with flex: 1, padding: 16"
      ↓ JSON.stringify + Bridge
2. Shadow Thread: Yoga calculates { x: 0, y: 0, width: 390, height: 844 }
      ↓ JSON.stringify + Bridge  
3. Native Thread: Create UIView, set frame to calculated values

That's TWO Bridge crossings for a single layout operation.
For a screen with 200 views, that's 400 Bridge crossings.
Each crossing: serialize, queue, deserialize.
```

**The result:** React Native apps had a reputation for feeling "almost native but not quite." Animations stuttered. Lists were slower than their native counterparts. Cold starts were slower because the Bridge needed to bootstrap. The JavaScript-to-native boundary was a tax on every operation, and that tax compounded.

> **Discord's experience:** Discord's React Native app served millions of users, but the Bridge was a constant source of performance challenges. Their engineering team measured that on an iPhone 6, the serialization overhead for their message list added measurable latency to every scroll frame. Their message list was particularly pathological: each message had an avatar, username, timestamp, text content, and potentially reactions, embeds, and attachments — 10-15 native views per message. Scrolling through a busy channel meant creating and destroying hundreds of views per second, each one serialized across the Bridge. This was one of the driving forces behind their investment in performance optimization — and eventually, the New Architecture.

---

## 2. THE NEW ARCHITECTURE

The New Architecture is not a minor upgrade. It is a ground-up reimagining of how JavaScript talks to native code. Meta started this effort in 2018 (internally called "Fabric" and "TurboModules"), and it's been the default for new React Native projects since 0.76 (late 2024).

The core insight: **eliminate the Bridge entirely.** Instead of serializing data to JSON and passing it through an asynchronous queue, let JavaScript call native functions directly — synchronously when needed — through a shared C++ layer.

Four pieces make this work:

### 2.1 JSI (JavaScript Interface)

JSI is the foundation of everything. It's a C++ API that lets JavaScript hold references to C++ objects and call methods on them directly. No serialization. No Bridge. No JSON.

Think of it this way: in the old architecture, JavaScript would say "please serialize this message and put it in the Bridge queue and when the native side gets around to it, deserialize it and create a View." With JSI, JavaScript directly calls `nativeView.setProps({ backgroundColor: 'red' })` and the call goes straight to the native layer through a C++ binding.

#### How JSI Works Under the Hood

JSI is not magic — it's a well-defined C++ abstraction layer. Let's look at the actual concepts, simplified but accurate:

```cpp
// JSI defines a "Runtime" — an abstraction over ANY JavaScript engine.
// This is the key insight: the Runtime interface is engine-agnostic.

// Hermes implements this interface. So does JSC. So could V8.
// Your JavaScript code doesn't know or care which engine is underneath.

class Runtime {
public:
  // Evaluate JavaScript code
  virtual Value evaluateJavaScript(
    const std::shared_ptr<const Buffer>& buffer,
    const std::string& sourceURL) = 0;
    
  // Create a JavaScript object from C++
  virtual Object createObject() = 0;
  
  // Create a function callable from JavaScript
  virtual Function createFunctionFromHostFunction(
    const PropNameID& name,
    unsigned int paramCount,
    HostFunctionType func) = 0;
    
  // ... more methods
};
```

The critical concept is the **HostObject**. A HostObject is a C++ object that JavaScript can hold a reference to and call methods on:

```cpp
// A simplified HostObject — this is what makes JSI powerful.
// JavaScript holds a REFERENCE to this C++ object.
// No serialization. No copying. Direct access.

class NativeStorage : public jsi::HostObject {
public:
  // Called when JavaScript accesses a property: nativeStorage.someKey
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& name) override {
    std::string key = name.utf8(rt);
    
    if (key == "getItem") {
      // Return a function that JS can call directly
      return jsi::Function::createFromHostFunction(
        rt,
        jsi::PropNameID::forAscii(rt, "getItem"),
        1, // paramCount
        [this](jsi::Runtime& rt,
               const jsi::Value& thisVal,
               const jsi::Value* args,
               size_t count) -> jsi::Value {
          
          // This C++ code runs SYNCHRONOUSLY when JS calls getItem()
          std::string key = args[0].asString(rt).utf8(rt);
          std::string value = this->storage_.get(key); // Direct C++ call
          return jsi::String::createFromUtf8(rt, value);
        });
    }
    
    if (key == "setItem") {
      return jsi::Function::createFromHostFunction(
        rt,
        jsi::PropNameID::forAscii(rt, "setItem"),
        2,
        [this](jsi::Runtime& rt,
               const jsi::Value& thisVal,
               const jsi::Value* args,
               size_t count) -> jsi::Value {
          
          std::string key = args[0].asString(rt).utf8(rt);
          std::string value = args[1].asString(rt).utf8(rt);
          this->storage_.set(key, value); // Direct C++ call
          return jsi::Value::undefined();
        });
    }
    
    return jsi::Value::undefined();
  }

private:
  StorageEngine storage_; // Some native storage implementation
};
```

And from the JavaScript side, this HostObject is used as if it were a normal object:

```javascript
// JavaScript side — using a JSI HostObject
// The 'nativeStorage' object is a C++ HostObject installed on the global scope

const value = global.nativeStorage.getItem('user_token');
// This call goes DIRECTLY to C++. No Bridge. No JSON. No async.
// The C++ getItem() method runs synchronously and returns the value.

global.nativeStorage.setItem('user_token', 'abc123');
// Same — direct synchronous call to C++.

// Compare to the old Bridge approach:
AsyncStorage.getItem('user_token').then(value => {
  // This was ASYNC because it went through the Bridge:
  // JS → JSON.stringify → Bridge → Native → AsyncStorage → Bridge → JSON.parse → JS
  // Minimum 2 frame round-trip.
});
```

This is the fundamental shift. JavaScript objects can now **hold references** to native objects. Not serialized copies. Not JSON descriptions. Actual references. When JavaScript calls a method on that reference, the call drops directly into C++ code running in the same process.

#### What JSI Enables

This direct binding is what makes the following possible:

1. **Reanimated worklets**: Reanimated uses JSI to install "worklets" — small JavaScript functions — directly on the UI thread. When an animation runs, the worklet executes on the UI thread through a JSI-powered Hermes runtime, without ever touching the JS thread or the Bridge.

2. **Synchronous native calls**: Need the current scroll offset for a computation? With JSI, you call a synchronous function and get the value back in the same JavaScript execution frame. No callbacks, no promises, no stale data.

3. **Shared memory**: JSI enables ArrayBuffer sharing between JavaScript and native code. Both sides can read and write to the same memory. This is how high-performance image processing and audio libraries work in the New Architecture.

4. **Multiple JS engines**: Because JSI is an abstraction over the engine, React Native can support Hermes, JSC, or any conforming engine without changing the native module code.

```
Old Architecture:
  JS → JSON.stringify → Bridge Queue → JSON.parse → Native
  Native → JSON.stringify → Bridge Queue → JSON.parse → JS
  
  Cost: O(data_size) serialization on both sides, minimum 1 frame latency

New Architecture (JSI):
  JS → C++ binding → Native  (direct call, synchronous or async by choice)
  Native → C++ binding → JS  (direct callback)
  
  Cost: O(1) function call overhead, zero-frame latency for sync calls
```

### 2.2 Fabric (New Rendering System)

Fabric is the new rendering system, built on top of JSI. It replaces the old "UIManager" that communicated through the Bridge.

In the old system, when you rendered a `<View>`, JavaScript would send a create command through the Bridge, the native side would create the view, and any subsequent updates went through the same serialized path. Layout was calculated on the Shadow Thread and results were sent back through the Bridge.

Fabric changes this fundamentally.

#### Old Rendering Flow vs Fabric

Here's the old rendering flow, step by step:

```
OLD RENDERING FLOW (Bridge-Based UIManager):

Step 1: JS Thread
  React reconciler produces effect list: "Create View #42, set props {flex: 1}"
      │
      ▼
Step 2: JS Thread → Bridge (ASYNC)
  JSON.stringify({
    module: "UIManager",
    method: "createView",
    args: [42, "RCTView", 1, {flex: 1}]
  })
  Message queued in Bridge buffer
      │
      ▼
Step 3: Bridge → Shadow Thread (ASYNC)
  JSON.parse(message)
  Create Shadow Node #42
  Yoga calculates layout: {x: 0, y: 0, w: 390, h: 200}
      │
      ▼
Step 4: Shadow Thread → Bridge (ASYNC)
  JSON.stringify({
    module: "UIManager", 
    method: "updateLayout",
    args: [42, 0, 0, 390, 200]
  })
      │
      ▼
Step 5: Bridge → UI Thread (ASYNC)
  JSON.parse(message)
  Create UIView, set frame = (0, 0, 390, 200)
  
  MINIMUM LATENCY: 3 async hops × ~1 frame each = 3 frames (50ms at 60fps)
  WORST CASE: Bridge backed up = 5-10 frames (83-166ms)
```

Now here's the same operation under Fabric:

```
NEW RENDERING FLOW (Fabric + JSI):

Step 1: JS Thread
  React reconciler produces effect list: "Create View #42, set props {flex: 1}"
      │
      ▼
Step 2: JS Thread → C++ Shadow Tree (SYNCHRONOUS via JSI)
  Direct C++ call: shadowTree.createNode(42, "View", {flex: 1})
  Shadow Node created in C++ memory (no serialization)
  Yoga calculates layout immediately: {x: 0, y: 0, w: 390, h: 200}
      │
      ▼
Step 3: C++ Shadow Tree → UI Thread (DIRECT via JSI)
  Layout results applied to native view hierarchy
  Create UIView, set frame = (0, 0, 390, 200)
  
  LATENCY: 1-2 frames (16-33ms at 60fps)
  The layout result is available synchronously if needed
```

The difference is stark: three async Bridge hops reduced to direct C++ calls. No JSON anywhere.

#### Key Fabric Concepts

**The Shadow Tree is now in C++.** Both JavaScript and native code can read and write the Shadow Tree directly through JSI. No serialization needed for layout calculations.

```
                    ┌─────────────────────────────┐
                    │      C++ Shadow Tree         │
                    │  (shared between all threads) │
                    │                               │
                    │   ShadowNode #1 (View)       │
                    │     ├── ShadowNode #2 (Text) │
                    │     └── ShadowNode #3 (View) │
                    │           └── #4 (Image)     │
                    └───────┬──────────┬───────────┘
                            │          │
                    ┌───────▼──┐  ┌───▼──────────┐
                    │ JS Thread │  │  UI Thread    │
                    │ (read/    │  │  (read/       │
                    │  write    │  │   commit to   │
                    │  via JSI) │  │   native views)│
                    └──────────┘  └───────────────┘
```

**Synchronous layout.** JavaScript can request a layout measurement and get the result in the same frame. This is what makes features like `onLayout` reliable and what enables smooth, synchronized animations.

```javascript
// With Fabric, this is synchronous through JSI:
const measurement = viewRef.current.measure();
// Returns immediately: { x, y, width, height, pageX, pageY }
// No callback. No promise. No stale data.

// Old architecture equivalent (async, unreliable):
viewRef.current.measure((x, y, width, height, pageX, pageY) => {
  // This fires NEXT frame at best. Data is already stale.
});
```

**Priority-based rendering.** Fabric supports React's concurrent features — different parts of the UI can be rendered at different priorities. A user interaction can interrupt a low-priority background render, keeping the app responsive.

```javascript
// Fabric enables React concurrent features in React Native:

// High-priority update (user typed in input) — processed immediately
const [query, setQuery] = useState('');

// Low-priority update (search results) — can be interrupted
const [results, setResults] = useTransition();

function handleSearch(text) {
  setQuery(text);                    // High priority: update input immediately
  startTransition(() => {
    setResults(filterData(text));    // Low priority: can yield to user input
  });
}

// This was IMPOSSIBLE with the old Bridge architecture because:
// 1. All updates went through the same Bridge queue (no priorities)
// 2. You couldn't interrupt an in-progress render
// 3. The async nature meant you couldn't synchronize priorities
```

**Immutable tree operations.** Fabric uses an immutable tree structure where updates create new tree nodes rather than mutating existing ones. This enables safe concurrent access from multiple threads.

```
Immutable Tree Update:

Before update:
  Tree A: Root → [View1, View2, View3]
  
After View2 changes color:
  Tree A: Root → [View1, View2, View3]         (still valid, read-safe)
  Tree B: Root' → [View1, View2', View3]        (new tree with changed node)
  
  Only View2' and Root' are new allocations.
  View1 and View3 are shared references.
  Tree A can still be safely read by another thread.
  
  Once Tree B is committed, Tree A is garbage collected.
```

This is the same structural sharing concept used by persistent data structures in Clojure and Immutable.js, but applied at the rendering layer. It means the UI thread can read the current tree while the JS thread is building the next one, without locks or data races.

**The measurable impact:** Shopify measured a 17-27% improvement in cold start times after migrating to the New Architecture. The removal of the serialization layer alone accounted for a significant portion of this improvement.

### 2.3 TurboModules

TurboModules are the New Architecture's replacement for Native Modules. Where old Native Modules were loaded eagerly at app startup (every registered module was initialized, whether you needed it or not), TurboModules are loaded lazily — initialized only when first accessed.

This has a direct impact on cold start time. If your app registers 40 native modules (a realistic number for a production app with analytics, crash reporting, maps, camera, push notifications, etc.), the old architecture initialized all 40 at startup. TurboModules initialize only the ones you actually use during startup — maybe 8-10 — and defer the rest.

#### A Complete TurboModule Example

Let's walk through a complete TurboModule from spec to implementation. This is the real workflow — understanding this is essential if you ever need to write a custom native module or debug one.

**Step 1: Define the spec in TypeScript**

```typescript
// specs/NativeDeviceInfo.ts
// This file is the SINGLE SOURCE OF TRUTH for the module interface.
// Codegen reads this to generate C++ bindings.

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // Synchronous methods — return values directly via JSI
  getDeviceName(): string;
  getSystemVersion(): string;
  getTotalMemory(): number;
  isLowEndDevice(): boolean;

  // Asynchronous methods — return promises
  getBatteryLevel(): Promise<number>;
  getStorageUsage(): Promise<{
    total: number;
    used: number;
    available: number;
  }>;

  // Methods with complex parameters
  logEvent(
    eventName: string,
    params: { [key: string]: string | number | boolean }
  ): void;

  // Constants — evaluated once at module initialization
  getConstants(): {
    brand: string;
    model: string;
    deviceId: string;
    isEmulator: boolean;
    apiLevel: number;
  };
}

// getEnforcing will throw if the module isn't available
// (as opposed to get() which returns null)
export default TurboModuleRegistry.getEnforcing<Spec>('NativeDeviceInfo');
```

**Step 2: Codegen generates C++ bindings**

When you build, Codegen reads that TypeScript spec and generates C++ header files:

```cpp
// Auto-generated by Codegen — DO NOT EDIT
// NativeDeviceInfoSpec.h

#pragma once

#include <ReactCommon/TurboModule.h>
#include <react/bridging/Bridging.h>

namespace facebook::react {

class JSI_EXPORT NativeDeviceInfoCxxSpecJSI : public TurboModule {
protected:
  NativeDeviceInfoCxxSpecJSI(std::shared_ptr<CallInvoker> jsInvoker);

public:
  // These are the methods YOUR native code must implement
  virtual jsi::String getDeviceName(jsi::Runtime& rt) = 0;
  virtual jsi::String getSystemVersion(jsi::Runtime& rt) = 0;
  virtual double getTotalMemory(jsi::Runtime& rt) = 0;
  virtual bool isLowEndDevice(jsi::Runtime& rt) = 0;
  virtual jsi::Value getBatteryLevel(jsi::Runtime& rt) = 0;
  virtual jsi::Value getStorageUsage(jsi::Runtime& rt) = 0;
  virtual void logEvent(
    jsi::Runtime& rt,
    jsi::String eventName,
    jsi::Object params) = 0;
  virtual jsi::Object getConstants(jsi::Runtime& rt) = 0;
};

} // namespace facebook::react
```

**Step 3: Implement the native module (iOS example)**

```objc
// NativeDeviceInfo.mm (Objective-C++)

#import "NativeDeviceInfoSpec.h"  // Codegen-generated header
#import <React/RCTBridge.h>
#import <UIKit/UIKit.h>
#import <sys/utsname.h>
#import <mach/mach.h>

@interface NativeDeviceInfo : NSObject <NativeDeviceInfoSpec>
@end

@implementation NativeDeviceInfo

RCT_EXPORT_MODULE()

- (NSString *)getDeviceName {
  return [[UIDevice currentDevice] name];
}

- (NSString *)getSystemVersion {
  return [[UIDevice currentDevice] systemVersion];
}

- (NSNumber *)getTotalMemory {
  return @([NSProcessInfo processInfo].physicalMemory);
}

- (NSNumber *)isLowEndDevice {
  // Devices with less than 3GB are "low-end" for our purposes
  uint64_t totalMemory = [NSProcessInfo processInfo].physicalMemory;
  return @(totalMemory < 3ULL * 1024 * 1024 * 1024);
}

- (void)getBatteryLevel:(RCTPromiseResolveBlock)resolve
                 reject:(RCTPromiseRejectBlock)reject {
  [UIDevice currentDevice].batteryMonitoringEnabled = YES;
  float level = [UIDevice currentDevice].batteryLevel;
  resolve(@(level));
}

- (void)getStorageUsage:(RCTPromiseResolveBlock)resolve
                 reject:(RCTPromiseRejectBlock)reject {
  NSDictionary *attrs = [[NSFileManager defaultManager]
    attributesOfFileSystemForPath:NSHomeDirectory() error:nil];
  
  uint64_t total = [attrs[NSFileSystemSize] unsignedLongLongValue];
  uint64_t free = [attrs[NSFileSystemFreeSize] unsignedLongLongValue];
  
  resolve(@{
    @"total": @(total),
    @"used": @(total - free),
    @"available": @(free),
  });
}

- (void)logEvent:(NSString *)eventName params:(NSDictionary *)params {
  // Forward to your analytics SDK
  [AnalyticsSDK logEvent:eventName parameters:params];
}

- (NSDictionary *)getConstants {
  struct utsname systemInfo;
  uname(&systemInfo);
  
  return @{
    @"brand": @"Apple",
    @"model": [NSString stringWithCString:systemInfo.machine
                                 encoding:NSUTF8StringEncoding],
    @"deviceId": [[[UIDevice currentDevice] identifierForVendor] UUIDString],
    @"isEmulator": @(TARGET_IPHONE_SIMULATOR != 0),
    @"apiLevel": @(0),  // Not applicable on iOS
  };
}

- (std::shared_ptr<facebook::react::TurboModule>)getTurboModule:
    (const facebook::react::ObjCTurboModule::InitParams &)params {
  return std::make_shared<
    facebook::react::NativeDeviceInfoSpecJSI>(params);
}

@end
```

**Step 4: Use from JavaScript**

```typescript
// In your React Native code — using the TurboModule

import NativeDeviceInfo from './specs/NativeDeviceInfo';

function DeviceInfoScreen() {
  // Synchronous calls — instant, no async overhead
  const deviceName = NativeDeviceInfo.getDeviceName();
  const isLowEnd = NativeDeviceInfo.isLowEndDevice();
  const constants = NativeDeviceInfo.getConstants();

  // Constants are available immediately (no bridge round-trip)
  console.log(`Device: ${constants.brand} ${constants.model}`);
  console.log(`Emulator: ${constants.isEmulator}`);

  // Async calls for I/O-bound operations
  const [battery, setBattery] = useState<number | null>(null);
  
  useEffect(() => {
    NativeDeviceInfo.getBatteryLevel().then(setBattery);
  }, []);

  // Use device info for architecture decisions
  if (isLowEnd) {
    // Reduce animation complexity on low-end devices
    // Use smaller image sizes
    // Disable expensive visual effects
  }

  return (
    <View>
      <Text>Device: {deviceName}</Text>
      <Text>Memory: {(NativeDeviceInfo.getTotalMemory() / 1e9).toFixed(1)} GB</Text>
      <Text>Battery: {battery !== null ? `${(battery * 100).toFixed(0)}%` : 'Loading...'}</Text>
    </View>
  );
}
```

#### TurboModule vs Old Native Module — Key Differences

```
OLD NATIVE MODULE:
  ┌─────────────────────────────────────────┐
  │ App startup                              │
  │  → Initialize ALL 40 registered modules  │
  │  → Each module: alloc, init, register    │
  │  → Total: ~300ms on budget Android       │
  │  → 32 of 40 modules unused at startup   │
  └─────────────────────────────────────────┘

TURBO MODULE:
  ┌─────────────────────────────────────────┐
  │ App startup                              │
  │  → Register module SPECS only (fast)     │
  │  → Initialize 0 modules                  │
  │  → Total: ~20ms on budget Android        │
  │                                          │
  │ First use of NativeDeviceInfo:           │
  │  → Initialize NativeDeviceInfo (~3ms)    │
  │  → Cache for subsequent calls            │
  │                                          │
  │ First use of NativeCamera:               │
  │  → Initialize NativeCamera (~5ms)        │
  │  → Cache for subsequent calls            │
  │                                          │
  │ NativeMaps never used this session:      │
  │  → Never initialized. Zero cost.         │
  └─────────────────────────────────────────┘
```

**Additional TurboModule benefits:**
- **Type-safe interfaces.** The Codegen-generated C++ enforces the TypeScript contract at compile time. Pass the wrong types and you get a compile error, not a runtime crash.
- **Synchronous method calls.** TurboModules can expose synchronous methods that JavaScript calls directly through JSI. Critical for performance-sensitive operations like reading device sensors or checking feature flags.
- **Shared C++ code.** The C++ implementation can be shared across iOS and Android, reducing platform-specific code. Write once, run on both platforms.

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

Codegen reads this and generates the entire C++ to Objective-C/Java bridge at build time. Zero runtime overhead for the type checking. Zero JSON serialization.

#### Codegen in Practice: What Gets Generated

When you run a build, Codegen produces a set of files that most developers never look at but should understand:

```
codegen output/
├── react/
│   └── renderer/
│       └── components/
│           └── MyModuleSpec/
│               ├── ComponentDescriptors.h    ← Component metadata
│               ├── EventEmitters.h           ← Type-safe event definitions  
│               ├── Props.h                   ← Typed prop structs
│               ├── ShadowNodes.h             ← Shadow tree node types
│               └── States.h                  ← Component state types
└── rncore/
    └── NativeMyModuleSpec.h                  ← TurboModule interface
```

The generated code is verbose but fast. It converts JavaScript values to C++ types at the boundary, validates types, and calls through to your native implementation. The conversion happens once, at the call site, instead of serializing an entire message to JSON.

```cpp
// What Codegen generates for a method call (simplified):

// Instead of this (old Bridge):
//   1. JS: JSON.stringify(["fetchUserData", ["user_123"]])
//   2. Bridge: copy string to native
//   3. Native: JSON.parse → extract module → extract method → extract args
//   4. Native: validate types at runtime (maybe crash if wrong)

// Codegen generates this (new Architecture):
jsi::Value NativeMyModuleCxxSpecJSI::fetchUserData(
    jsi::Runtime& rt,
    jsi::String userId) {
  // userId is ALREADY a C++ string — no parsing needed.
  // Type is enforced at compile time — wrong types = build error.
  return static_cast<NativeMyModuleCxxSpec*>(this)
    ->fetchUserData(rt, std::move(userId));
}
```

---

## 3. HERMES: REACT NATIVE'S JAVASCRIPT ENGINE

Hermes is Meta's JavaScript engine, purpose-built for React Native. It replaced JavaScriptCore as the default engine starting with React Native 0.70, and understanding how it differs explains several performance characteristics you'll observe in production.

### 3.1 Why a Custom Engine?

JavaScriptCore (JSC) was designed for Safari — a desktop/mobile browser where the JS engine has time to do JIT (Just-In-Time) compilation and where startup time is amortized across a long browser session. React Native apps have different constraints:

- **Cold start matters enormously.** Users expect your app to launch in under 2 seconds. Every millisecond of engine initialization is felt directly.
- **Memory is scarce.** A $50 Android phone has 2-3GB of total RAM, shared across the OS, background apps, and your app. JSC's JIT compiler consumes significant memory for compiled code caches.
- **Predictable performance > peak performance.** Users care more about consistent 60fps than occasional 120fps with dropped frames.
- **Battery life matters.** JIT compilation uses CPU cycles that drain the battery. AOT compilation moves that cost to build time (on a wall-powered build server).

The tradeoff Hermes makes is explicit: **sacrifice peak throughput for faster startup, lower memory usage, and more predictable performance.** A JIT engine might execute a hot loop 3x faster than Hermes, but Hermes starts executing code 2x sooner and uses half the memory. For a mobile app (not a compute-heavy server), that tradeoff is overwhelmingly correct.

### 3.2 How Hermes Works: The Compilation Pipeline

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

### 3.3 Hermes Bytecode Format Deep Dive

Hermes bytecode files (`.hbc`) have a specific binary format that's worth understanding, especially when you're debugging bundle size or analyzing build output.

```
Hermes Bytecode File Structure (.hbc):
  
  ┌───────────────────────────────────┐
  │          FILE HEADER              │  (64 bytes)
  │  Magic number: C6 1F BC 03       │  ← Identifies file as Hermes bytecode
  │  Bytecode version: 96            │  ← Must match runtime Hermes version
  │  Source hash (SHA1)               │  ← For integrity verification
  │  Global code index                │  ← Entry point for execution
  │  Function count                   │  ← Number of functions in bundle
  │  String count                     │  ← Number of string literals
  │  Overflow string count            │  ← Strings > 255 bytes
  │  String storage size              │  ← Total bytes for string table
  │  RegExp count                     │  ← Number of compiled RegExps
  │  RegExp storage size              │  ← Total bytes for RegExp table
  │  Array buffer size                │  ← Size of array literal data
  │  Obj key buffer size              │  ← Size of object key data
  │  Obj value buffer size            │  ← Size of object value data
  │  CJS module count                 │  ← CommonJS module count
  │  CJS module count (static)       │  ← Statically resolved modules
  │  Debug info offset                │  ← For source maps / debugging
  └───────────────────────────────────┘
  ┌───────────────────────────────────┐
  │       FUNCTION HEADERS            │  (variable size)
  │  For each function:               │
  │    - Bytecode offset              │
  │    - Bytecode size                │
  │    - Parameter count              │
  │    - Frame size (local variables) │
  │    - Environment size (closures)  │
  │    - Highest read/write cache idx │
  │    - Flags (strictMode, async,    │
  │            generator, etc.)       │
  └───────────────────────────────────┘
  ┌───────────────────────────────────┐
  │        STRING TABLE               │  (variable size)
  │  All string literals, identifiers │
  │  Deduplicated and compressed      │
  │  Referenced by numeric index      │
  │  (not by name — saves space)      │
  └───────────────────────────────────┘
  ┌───────────────────────────────────┐
  │        BYTECODE INSTRUCTIONS      │  (variable size)
  │  The actual executable code       │
  │  Register-based VM instructions   │
  │  Compact encoding (1-4 bytes/op)  │
  └───────────────────────────────────┘
  ┌───────────────────────────────────┐
  │        REGEXP BYTECODE            │  (variable size)
  │  Pre-compiled regular expressions │
  └───────────────────────────────────┘
  ┌───────────────────────────────────┐
  │        DEBUG INFO (optional)      │  (variable size)
  │  Source maps, function names      │
  │  Line/column mappings             │
  │  Only in debug builds             │
  └───────────────────────────────────┘
```

You can inspect Hermes bytecode with the `hermes` CLI tools:

```bash
# Compile JavaScript to Hermes bytecode
npx hermes -emit-binary -out bundle.hbc bundle.js

# Disassemble bytecode to see the instructions
npx hermes -dump-bytecode bundle.hbc

# Example disassembly output for a simple function:
# Function<greet>(2 params, 4 registers, 0 symbols):
#   LoadParam       r0, 1          ; Load first parameter
#   LoadConstString r1, "Hello, "  ; Load string literal
#   Add             r2, r1, r0     ; String concatenation
#   Ret             r2             ; Return result
```

**Hermes bytecode is typically 20-30% smaller than equivalent JavaScript bundles** because:
- Comments and whitespace are eliminated during compilation (not just minification — actual compilation)
- Identifiers are converted to numeric references in the string table
- The bytecode format is more compact than minified JS text
- Repeated patterns are encoded more efficiently in binary

This means your OTA updates (EAS Update) ship smaller payloads, and your app's initial download is smaller.

**Key implication for debugging:** Hermes bytecode version must match the Hermes runtime version. If you build bytecode with Hermes 0.12 but the app runs Hermes 0.11, you get a crash at startup — not a helpful error, but an opaque "invalid bytecode version" crash. This is a common issue when Expo SDK versions and manually installed Hermes versions get out of sync.

### 3.4 Hermes Memory Model and Garbage Collection

Hermes uses a **generational garbage collector** specifically tuned for mobile workloads. Understanding its behavior is critical for debugging memory issues and designing data structures that cooperate with (rather than fight against) the GC.

#### The Three Generations

```
HERMES HEAP LAYOUT:

┌──────────────────────────────────────────────────────────────┐
│                     YOUNG GENERATION (YG)                     │
│                                                               │
│  Size: ~2-4 MB (configurable)                                │
│  Collection frequency: every few hundred ms to seconds        │
│  Collection time: ~1-5ms (very fast)                         │
│  Strategy: Semi-space copying collector                       │
│                                                               │
│  What goes here:                                              │
│  - ALL new object allocations                                 │
│  - Short-lived objects (event handlers, render intermediates) │
│  - Temporary strings, arrays, closures                        │
│                                                               │
│  Most objects die here. A well-written app has 90%+ of        │
│  allocations collected in YG without ever being promoted.     │
└───────────────────────────────┬──────────────────────────────┘
                                │ Objects that survive 
                                │ multiple YG collections
                                │ get PROMOTED to OG
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                     OLD GENERATION (OG)                       │
│                                                               │
│  Size: grows as needed (up to heap limit)                     │
│  Collection frequency: when OG usage hits threshold           │
│  Collection time: ~10-50ms (incremental, broken into slices) │
│  Strategy: Mark-compact with incremental collection           │
│                                                               │
│  What goes here:                                              │
│  - Long-lived objects (module-scope variables, caches)        │
│  - Promoted objects from YG                                   │
│  - Persistent data structures (Redux store, navigation state) │
│                                                               │
│  Objects here are expensive to collect.                        │
│  If OG grows unbounded, you have a memory leak.              │
└───────────────────────────────┬──────────────────────────────┘
                                │ Objects larger than
                                │ ~128KB threshold
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                   LARGE OBJECT SPACE                          │
│                                                               │
│  Individual allocations for very large objects                │
│  Not moved during compaction (too expensive to copy)          │
│  Collected independently when unreachable                     │
│                                                               │
│  What goes here:                                              │
│  - Large ArrayBuffers (image data, audio buffers)             │
│  - Very large strings (JSON API responses before parsing)     │
│  - Large typed arrays                                         │
└──────────────────────────────────────────────────────────────┘
```

#### GC Phases in Detail

When Hermes runs a garbage collection, here's what happens at each phase:

```
YOUNG GENERATION COLLECTION:

Phase 1: Root Scanning (~0.1-0.5ms)
  Scan the stack, registers, and globals for references into YG.
  
Phase 2: Copy Live Objects (~0.5-3ms)
  Copy all reachable YG objects to either:
    - The other semi-space (if they're still young)
    - Old Generation (if they've survived enough collections)
  Unreachable objects are implicitly freed (their space is reclaimed
  when the entire semi-space is marked as empty).

Phase 3: Update References (~0.2-1ms)
  Fix all pointers that referenced moved objects.

Total YG collection: ~1-5ms
Impact: Usually invisible. Might cause a ~1ms hitch if it happens
during a critical animation frame.
```

```
OLD GENERATION COLLECTION (incremental):

Phase 1: Initial Mark (~2-5ms)
  Mark all objects directly reachable from roots.
  This is a STOP-THE-WORLD pause — JS execution halts.

Phase 2: Incremental Marking (~5-20ms, broken into ~1ms slices)
  Mark all transitively reachable objects.
  This runs CONCURRENTLY with JS execution:
    - Run ~1ms of marking
    - Yield to JS execution
    - Run ~1ms of marking
    - Yield to JS execution
    - ...
  Write barrier tracks mutations during marking.

Phase 3: Remark (~1-3ms)
  Final mark pass to catch objects mutated during incremental phase.
  Another STOP-THE-WORLD pause.

Phase 4: Sweep (~2-10ms, can be incremental)
  Free all unmarked objects.
  Optionally compact memory to reduce fragmentation.

Total OG collection: ~10-50ms elapsed (but only ~5-15ms of JS pause time)
Impact: CAN cause visible jank if it hits during an animation.
```

#### Memory Budget Recommendations by Device Tier

This is one of the most actionable tables in this entire guide. Exceeding these budgets triggers aggressive GC, which causes jank.

```
DEVICE TIER CLASSIFICATION AND MEMORY BUDGETS:

┌─────────────────┬──────────────┬───────────────┬──────────────────────────┐
│ Device Tier     │ RAM          │ JS Heap Budget│ Examples                  │
├─────────────────┼──────────────┼───────────────┼──────────────────────────┤
│ Ultra-low-end   │ ≤2 GB        │ 64-96 MB      │ Samsung A03, Nokia C01,  │
│                 │              │               │ Redmi 9A, most $50-80    │
│                 │              │               │ Android phones            │
├─────────────────┼──────────────┼───────────────┼──────────────────────────┤
│ Low-end         │ 3-4 GB       │ 96-150 MB     │ Samsung A14, Pixel 4a,   │
│                 │              │               │ Redmi Note 11, most      │
│                 │              │               │ $100-200 Android phones   │
├─────────────────┼──────────────┼───────────────┼──────────────────────────┤
│ Mid-range       │ 4-6 GB       │ 150-256 MB    │ Samsung A54, Pixel 7a,   │
│                 │              │               │ iPhone 11/12, most       │
│                 │              │               │ $200-400 phones           │
├─────────────────┼──────────────┼───────────────┼──────────────────────────┤
│ High-end        │ 6-8 GB       │ 256-384 MB    │ Samsung S23, Pixel 8,    │
│                 │              │               │ iPhone 13/14, most       │
│                 │              │               │ $400-800 phones           │
├─────────────────┼──────────────┼───────────────┼──────────────────────────┤
│ Flagship        │ 8-16 GB      │ 384-512 MB    │ Samsung S24 Ultra,       │
│                 │              │               │ iPhone 15 Pro Max,       │
│                 │              │               │ Pixel 8 Pro, $800+       │
└─────────────────┴──────────────┴───────────────┴──────────────────────────┘

PRACTICAL RULES:
- Your JS heap should NEVER exceed 50% of available RAM
- Target 150 MB JS heap for a "works everywhere" app
- If your heap exceeds 256 MB, you have a memory problem regardless of device
- Monitor heap growth over time — linear growth = leak
- Each React re-render allocates temporary objects; more components = more GC pressure
```

#### Configuring Hermes GC

You can configure Hermes GC behavior at build time for your specific use case:

```javascript
// In your React Native app config or native code, you can set Hermes flags:

// react-native.config.js (or via native code)
// These flags control Hermes GC behavior:

// gcInitHeapSize: Initial heap size in bytes (default: 32MB)
//   Lower = less initial memory, but more early GC
//   Higher = faster startup (fewer early GCs), but more memory

// gcMaxHeapSize: Maximum heap size in bytes (default: 512MB)
//   This is the HARD LIMIT. Hermes OOMs if exceeded.
//   Set lower on memory-constrained apps to fail fast.

// gcSanitizeRate: How aggressively Hermes sanitizes freed memory
//   0 = no sanitization (fast, default for release)
//   1 = full sanitization (slow, useful for debugging use-after-free)
```

**Why this matters for you:** If you see your app's memory growing steadily over time, the issue is almost certainly objects leaking from YG to OG — closures capturing references they shouldn't, state that accumulates without cleanup, event listeners that are never removed. The profiling chapter (Ch 14) covers how to diagnose this using Hermes's built-in heap snapshots.

### 3.5 Hermes vs JSC vs V8: When Hermes Loses

Hermes is not universally faster. Understanding where it's slower is as important as understanding where it's faster:

```
PERFORMANCE COMPARISON: HERMES vs JSC vs V8

                     Hermes    JSC       V8 (Chrome)
                     -------   -------   -----------
Cold start time      ★★★★★     ★★☆☆☆    ★★★☆☆
  (Hermes wins by 2x due to AOT bytecode)
  
Memory usage         ★★★★★     ★★☆☆☆    ★★★☆☆
  (No JIT code cache = less memory)

Peak CPU throughput  ★★☆☆☆     ★★★★☆    ★★★★★
  (No JIT = slower hot loops and heavy computation)

Predictability       ★★★★★     ★★★☆☆    ★★☆☆☆
  (No JIT warmup = no performance cliffs)

GC pause time        ★★★★☆     ★★★☆☆    ★★★★★
  (Incremental GC is good; V8's Orinoco GC is better)

Binary size          ★★★★★     ★★★★☆    ★★☆☆☆
  (Hermes runtime is ~2MB; V8 is ~7MB+)

ES2022+ features     ★★★☆☆     ★★★★★    ★★★★★
  (Hermes lags slightly on bleeding-edge features)

PRACTICAL IMPLICATIONS:
- JSON.parse of a 1MB payload: Hermes ~40ms, JSC ~25ms, V8 ~15ms
  → For heavy JSON parsing, consider native parsing via TurboModule
  
- Image processing loop (100K iterations): Hermes ~200ms, JSC ~80ms, V8 ~50ms
  → For CPU-heavy work, ALWAYS use native modules, not JS
  
- App startup to first meaningful frame: Hermes ~350ms, JSC ~650ms
  → Hermes wins where it matters most for mobile
  
- Regex matching on large strings: Hermes ~60ms, JSC ~30ms
  → Hermes is slower on regex; pre-filter on native if possible
```

The bottom line: Hermes is optimized for the mobile use case. If your app does heavy computation in JavaScript (which it shouldn't — that belongs in native modules), Hermes will be slower than a JIT engine. But for the 95% of mobile app work that's UI rendering, event handling, and state management, Hermes's startup and memory advantages dominate.

---

## 4. THE THREADING MODEL

React Native operates on three primary threads, with a few additional specialized threads that come into play for specific operations. Understanding what runs where — and what happens when they compete — is the difference between apps that feel native and apps that stutter.

This is the single most important section for debugging performance problems. Every jank, every stutter, every "the app feels slow" complaint traces back to one of these threads being overloaded.

### 4.1 The JS Thread

This is where your React code executes. Component rendering, state updates, event handlers, business logic, TanStack Query's cache management — all of it runs on the JS thread.

**The critical constraint:** The JS thread is single-threaded. JavaScript has no true parallelism (Web Workers exist in RN but are rarely used and have their own overhead). If your event handler does 100ms of computation, the JS thread is blocked for 100ms. During that time:
- No other event handlers can fire
- No state updates can be processed
- No new renders can begin
- If an animation is driven from JS, it drops frames
- Timer callbacks are delayed
- Network response handlers are queued

**What lives on the JS thread (exhaustive list):**
```
┌─────────────────────────────────────────────────────────┐
│                    JS THREAD WORK                        │
├─────────────────────────────────────────────────────────┤
│ React reconciliation (diffing old vs new tree)          │
│ Component render functions                               │
│ useState/useReducer state calculations                   │
│ useEffect/useLayoutEffect callbacks                      │
│ useMemo/useCallback computations                         │
│ Event handler functions (onPress, onScroll, etc.)       │
│ TanStack Query data fetching + cache management          │
│ Redux/Zustand/Jotai state updates                        │
│ Navigation state transitions (React Navigation)          │
│ JSON parsing (API responses)                             │
│ Business logic (validation, formatting, transformations) │
│ Timer callbacks (setTimeout, setInterval, rAF)           │
│ Promise resolution chains                                │
│ Module initialization (require/import)                   │
│ Hermes GC pauses (YG collections)                        │
└─────────────────────────────────────────────────────────┘
```

**Rule of thumb:** Keep JS thread work under 16ms per frame (60fps target) or 8ms per frame (120fps target on ProMotion displays). Anything longer and you're dropping frames.

#### Reading JS Thread Profiles

When you capture a performance trace (via Flipper, React DevTools, or Hermes sampling profiler), the JS thread flame graph tells you exactly where time is going:

```
JS Thread Flame Graph — Reading Guide:

Time →
├────────────────────────────────────────────────────┤
│                  commitRoot (12ms)                   │  ← React commit phase
│  ┌──────────────────────┐┌────────────────────────┐│
│  │ flushPassiveEffects  ││ flushSyncEffects       ││  ← Effect execution
│  │ (4ms)                ││ (3ms)                   ││
│  │  ┌──────────────┐   ││  ┌───────────────────┐ ││
│  │  │ useEffect cb │   ││  │ useLayoutEffect cb│ ││
│  │  │ (2ms)        │   ││  │ (1.5ms)           │ ││
│  │  └──────────────┘   ││  └───────────────────┘ ││
│  └──────────────────────┘└────────────────────────┘│
├────────────────────────────────────────────────────┤

HOW TO READ THIS:
- Width = time. Wider bars = more time spent.
- Depth = call stack. Deeper = more specific.
- Colors typically: blue = React internal, green = your code, red = native calls

WHAT TO LOOK FOR:
1. Wide bars at the top level → overall phase is slow
   Fix: reduce work in that phase (fewer effects, simpler renders)

2. Single wide bar deep in the stack → one function is the bottleneck
   Fix: optimize that specific function (memoize, defer, move to native)

3. Many thin bars at the same level → death by a thousand cuts
   Fix: reduce the number of components/effects, not individual cost

4. Bars labeled "GC" → garbage collection pause
   Fix: reduce allocations (fewer object spreads, fewer closures per render)
```

### 4.2 The UI Thread (Main Thread)

The UI thread is the native main thread — it's where UIKit (iOS) and Android's View system do their work. Touch event dispatch, native animations, and final pixel rendering all happen here.

**Key insight:** Native animations (Reanimated worklets, `useNativeDriver: true`) run on the UI thread, independent of JavaScript. This is why a Reanimated animation stays smooth even when the JS thread is busy — the animation runs entirely on the UI thread in a worklet, never crossing the thread boundary.

**What lives on the UI thread:**
```
┌─────────────────────────────────────────────────────────┐
│                    UI THREAD WORK                        │
├─────────────────────────────────────────────────────────┤
│ Touch event detection and dispatch                       │
│ Native gesture recognition                               │
│ Reanimated worklet execution                             │
│ Native Animated driver updates (useNativeDriver: true)   │
│ Core Animation / RenderThread frame composition          │
│ Native view creation and property updates                │
│ Native navigation transitions (screen push/pop)          │
│ StatusBar, keyboard, and system UI changes               │
│ Accessibility announcements                              │
│ Fabric commit phase (applying Shadow Tree to native)     │
│ Native module main-thread callbacks                      │
└─────────────────────────────────────────────────────────┘
```

**The critical constraint:** If the UI thread is blocked, the entire app freezes. The user can't scroll, can't tap, can't interact at all. Even the system back gesture stops working. A blocked UI thread is the worst possible performance failure — it looks like a crash.

```
UI Thread blocked — what the user experiences:

[0ms]   User taps a button
[0ms]   UI Thread starts handling the tap
[0ms]   Tap handler calls a synchronous native method that takes 200ms
        (e.g., a poorly implemented TurboModule sync method)
[0ms]   ← UI Thread is now BLOCKED
...
[200ms] UI Thread unblocks
[200ms] Touch feedback (button highlight) finally appears
[200ms] ← User felt a 200ms delay. App feels "laggy" or "frozen".

// Even worse: if UI Thread is blocked for 5+ seconds on Android,
// the OS shows "App Not Responding" (ANR) dialog.
// On iOS, the watchdog can KILL your app (watchdog timeout crash).
```

### 4.3 The Shadow Thread (Background Thread)

The Shadow Thread handles layout calculations using Yoga (the Flexbox engine). When your component tree changes, the new layout is calculated on this thread.

In the New Architecture, the Shadow Thread's work is more tightly integrated with both JS and UI threads through JSI, reducing the latency of layout results reaching the native rendering.

**What lives on the Shadow Thread:**
```
┌─────────────────────────────────────────────────────────┐
│                  SHADOW THREAD WORK                      │
├─────────────────────────────────────────────────────────┤
│ Yoga layout calculations (Flexbox algorithm)             │
│ Shadow Tree node creation and updates                    │
│ Text measurement (calculating text layout dimensions)    │
│ Image size resolution                                    │
│ Layout diffing (what changed between old and new tree)   │
│ Fabric tree cloning (immutable updates)                  │
└─────────────────────────────────────────────────────────┘
```

The Shadow Thread is typically the least likely to be your bottleneck, but it can be if you have deeply nested Flexbox layouts (>15 levels deep) or very large flat lists with complex item layouts. Yoga is fast, but the Flexbox algorithm is O(n * depth) — more nodes and more nesting means more computation.

### 4.4 Additional Threads You Should Know About

Beyond the big three, React Native uses several other threads:

```
┌─────────────────────────────────────────────────────────┐
│              ADDITIONAL THREADS                           │
├─────────────────────────────────────────────────────────┤
│ NativeModules Thread (iOS)                               │
│   Default thread for async native module methods         │
│   Prevents native work from blocking the UI thread       │
│                                                          │
│ Networking Thread                                        │
│   HTTP requests, WebSocket connections, fetch() calls    │
│   Results dispatched back to JS thread for handling      │
│                                                          │
│ Image Decoding Thread (per-platform)                     │
│   JPEG/PNG/WebP decoding happens off-main-thread         │
│   Decoded pixels sent to UI thread for display           │
│                                                          │
│ Reanimated UI Worklet Thread                             │
│   Runs a separate Hermes instance for worklets           │
│   Has its own JS event loop, isolated from main JS       │
│                                                          │
│ Android RenderThread                                     │
│   Hardware-accelerated drawing (Android only)            │
│   Handles GPU command submission                         │
│   Separate from the UI thread on Android 5.0+           │
└─────────────────────────────────────────────────────────┘
```

### 4.5 Thread Interaction: A Complete Picture

```
User taps a button — full thread trace:

UI Thread: Touch event dispatched by iOS/Android
    │
    ├──→ Gesture system evaluates: is this a tap, scroll, or long-press?
    │    (runs on UI thread, ~1-2ms)
    │
    ├──→ If using Gesture Handler: gesture evaluated in native (~0.5ms)
    │
    ▼
JS Thread: onPress handler executes
    │
    ├──→ Your event handler code runs (~1-5ms typically)
    │
    ├──→ setState called → React schedules re-render
    │
    ├──→ React reconciler runs: diff old tree vs new tree (~2-10ms)
    │    
    ├──→ React produces "effect list" of changes
    │
    ▼
Shadow Thread (via JSI): Layout recalculation
    │
    ├──→ New Shadow Nodes created for changed components
    │
    ├──→ Yoga runs Flexbox algorithm on changed subtree (~1-3ms)
    │
    ├──→ Absolute positions calculated for all affected nodes
    │
    ▼
UI Thread: Commit phase
    │
    ├──→ New/updated native views created
    │
    ├──→ View properties set (colors, text, images)
    │
    ├──→ Views positioned according to Yoga results
    │
    ├──→ Core Animation / RenderThread composites layers
    │
    ▼
GPU: Pixels composited and sent to display
    │
    └──→ User sees updated UI
    
TOTAL TIME (well-optimized, New Architecture):
  Gesture recognition:  ~1ms
  JS handler + render:  ~5-10ms
  Layout:               ~2ms
  Commit + composite:   ~2ms
  ─────────────────────────────
  Total:                ~10-15ms  (fits in one 16ms frame at 60fps)
  
TOTAL TIME (unoptimized, expensive render):
  Gesture recognition:  ~1ms
  JS handler + render:  ~50ms (expensive computation in handler)
  Layout:               ~10ms (deep nesting)
  Commit + composite:   ~5ms
  ─────────────────────────────
  Total:                ~66ms  (drops 3-4 frames, visible stutter)
```

### 4.6 Where Jank Comes From — A Diagnostic Guide

```
SYMPTOM → THREAD → CAUSE → FIX

Animations stutter during data loading
  → JS Thread overloaded
  → Network response parsing + state updates block the thread
  → Fix: Use native-driven animations (Reanimated worklets)
         Parse JSON in smaller chunks or on a native thread
         Use startTransition for low-priority updates

Scroll feels jerky on complex lists
  → JS Thread or Shadow Thread
  → Too many components rendering on scroll
  → Fix: Use FlashList instead of FlatList
         Reduce per-item component complexity
         Use getItemType for heterogeneous lists
         Increase windowSize if items are simple

Tap response is delayed (100ms+)
  → UI Thread or JS Thread
  → If UI thread: synchronous native work blocking touch delivery
  → If JS thread: heavy computation blocking event handler execution
  → Fix: Profile to identify which thread. Move work off that thread.

App freezes for 1-5 seconds
  → UI Thread blocked
  → Synchronous file I/O, database query, or heavy native computation on main thread
  → Fix: Move to async. NEVER do synchronous I/O on UI thread.

Periodic micro-stutters (~5ms hitches every few seconds)
  → JS Thread — GC pauses
  → Too many object allocations triggering frequent YG collections
  → Fix: Reduce allocations per render (fewer object spreads, fewer inline objects)
         Use useRef for mutable values that don't affect rendering
         Profile with Hermes heap timeline

Screen transition is slow
  → JS Thread
  → New screen's component tree is expensive to mount
  → Fix: Use react-native-screens (native stack)
         Lazy-load heavy components with React.lazy
         Use InteractionManager.runAfterInteractions for deferred work
```

**The golden rule:** Keep heavy work off the JS thread. Use `InteractionManager.runAfterInteractions()` for deferred work. Use Reanimated worklets for animations. Use native modules for CPU-intensive computation. Use `requestAnimationFrame` for frame-aligned updates.

---

## 5. BRIDGELESS MODE

Bridgeless mode is the final piece of the New Architecture puzzle. Starting with React Native 0.76, new projects have no Bridge at all — the entire communication layer is JSI.

What this means:
- **No fallback path.** In earlier versions of the New Architecture, the Bridge was still present as a fallback for modules that hadn't been migrated. Bridgeless mode removes it entirely.
- **Faster interop.** Without the Bridge overhead, even old-style modules that are wrapped with a compatibility layer perform better.
- **Smaller binary.** The Bridge code is no longer compiled into the app, slightly reducing the binary size.
- **Simpler mental model.** There is one communication path: JSI. No need to reason about whether a given operation goes through the Bridge or JSI.

### 5.1 What Bridgeless Mode Changes Internally

```
COMMUNICATION PATHS — BEFORE AND AFTER:

WITH BRIDGE (RN < 0.76 or New Arch with bridge fallback):

  TurboModule call:   JS → JSI → C++ → Native         (fast, direct)
  Legacy Module call: JS → Bridge → JSON → Native      (slow, serialized)
  Fabric render:      JS → JSI → C++ Shadow Tree       (fast, direct)
  Event from native:  Native → JSI → JS                (fast, direct)
  Legacy event:       Native → Bridge → JSON → JS      (slow, serialized)

  Problem: You had TWO communication paths in the same app.
  Some calls were fast (JSI), some were slow (Bridge).
  Hard to predict which path a given library used.
  
BRIDGELESS (RN >= 0.76):

  ALL module calls:   JS → JSI → C++ → Native          (fast, direct)
  ALL rendering:      JS → JSI → C++ Shadow Tree        (fast, direct)
  ALL events:         Native → JSI → JS                 (fast, direct)
  Legacy modules:     JS → JSI → Interop Layer → Native (fast, no JSON)

  One path. Predictable. Fast.
```

The interop layer is key: even legacy native modules that were written for the old Bridge get wrapped in a JSI-based compatibility layer. They don't get the full benefits of TurboModules (no lazy loading, no type safety), but they do avoid JSON serialization. It's a ~30-40% improvement for legacy modules just from removing the serialization overhead.

### 5.2 Migration Reality

If you're on Expo SDK 52+, you're already on the New Architecture with bridgeless mode by default. Expo's managed workflow handles the migration transparently — your JavaScript code doesn't change. The migration challenge is primarily for native modules that haven't been updated to support TurboModules.

Common libraries and their New Architecture status (as of early 2026):
- **react-native-reanimated** — fully supported since v3
- **react-native-gesture-handler** — fully supported
- **react-native-screens** — fully supported
- **@react-native-firebase** — fully supported since v19
- **react-native-maps** — supported via community fork
- **react-native-camera** (old) — not supported; use `expo-camera` instead
- **react-native-mmkv** — fully supported (uses JSI natively)
- **react-native-skia** — fully supported (built on JSI from day one)
- **@shopify/flash-list** — fully supported

### 5.3 Detecting Architecture at Runtime

Sometimes you need to know which architecture you're running on, especially during migration:

```typescript
// Detecting the architecture at runtime
import { Platform } from 'react-native';

// Check if Fabric is enabled
const isFabric = global.__fabric !== undefined 
  || (global.nativeFabricUIManager !== undefined);

// Check if TurboModules are available
const hasTurboModules = global.__turboModuleProxy !== undefined;

// Check if Bridgeless mode is active
const isBridgeless = global.RN$Bridgeless === true;

// Check if Hermes is the JS engine
const isHermes = () => !!global.HermesInternal;

// Practical usage:
if (__DEV__) {
  console.log('Architecture:', {
    fabric: isFabric,
    turboModules: hasTurboModules,
    bridgeless: isBridgeless,
    hermes: isHermes(),
    engine: global.HermesInternal 
      ? `Hermes ${global.HermesInternal.getRuntimeProperties()['OSS Release Version']}`
      : 'JSC',
  });
}
```

---

## 6. COLD START: WHAT ACTUALLY HAPPENS

When a user taps your app icon, here's the sequence — and where the time goes. This is the most important performance metric for user perception. A 2-second cold start feels "fast." A 4-second cold start feels "slow." A 6-second cold start gets your app uninstalled.

### 6.1 iOS Cold Start Sequence — Millisecond Breakdown

```
PHASE 1: PROCESS CREATION                              ~50-100ms
├── OS creates app process                              ~10ms
├── dyld loads executable binary                        ~15-30ms
├── dyld loads dynamic libraries (frameworks)           ~20-40ms
│   ├── UIKit.framework                                 ~8ms
│   ├── CoreFoundation.framework                        ~5ms
│   ├── libswiftCore.dylib                              ~4ms
│   ├── Hermes.framework                                ~3ms
│   └── ... (15-30 more frameworks)                     ~10-20ms
├── Objective-C runtime: register classes, categories   ~5-15ms
└── C++ static initializers execute                     ~2-5ms

PHASE 2: APP DELEGATE INITIALIZATION                    ~20-60ms
├── UIApplicationMain() called                          ~5ms
├── AppDelegate.init()                                  ~2ms
├── application:didFinishLaunchingWithOptions:           ~15-50ms
│   ├── React Native bridge/runtime initialization      ~10-30ms
│   ├── TurboModule registry setup                      ~2-5ms
│   │   (only REGISTERS specs; does NOT initialize modules)
│   └── Third-party SDK initialization                  ~5-20ms
│       ├── Firebase.configure()                        ~3-8ms
│       ├── Crash reporting setup                       ~1-3ms
│       └── Analytics setup                             ~1-5ms
└── Root view controller presented                      ~3-5ms

PHASE 3: HERMES ENGINE STARTUP                          ~30-60ms
├── Create Hermes Runtime instance                      ~5-10ms
├── Load bytecode from disk (.hbc file)                 ~10-25ms
│   ├── File I/O: read from app bundle                  ~5-15ms
│   └── Memory map bytecode (lazy loading)              ~3-8ms
├── Initialize global scope                             ~5-10ms
│   ├── Install JSI host objects                        ~2-4ms
│   ├── Install TurboModule proxy                       ~1-2ms
│   └── Install Fabric UIManager                        ~1-3ms
└── Hermes ready to execute bytecode                    ~2-5ms

PHASE 4: JS BUNDLE EXECUTION                            ~100-500ms
├── Execute top-level module code                       ~50-200ms
│   ├── React library initialization                    ~10-20ms
│   ├── React Native core module initialization         ~10-20ms
│   ├── Navigation library initialization               ~5-15ms
│   ├── State management setup                          ~3-8ms
│   ├── YOUR top-level imports execute                  ~20-100ms
│   │   ├── Each import runs module factory             
│   │   ├── Transitive imports cascade                  
│   │   └── Heavy libraries (date-fns, lodash) add up  
│   └── Metro require() resolution                     ~5-15ms
├── First TurboModule accesses                          ~10-50ms
│   ├── NativeDeviceInfo.getConstants()                 ~3-5ms
│   │   (module initialized on first access)
│   ├── NativeAppearance.getColorScheme()               ~2-3ms
│   ├── NativeStatusBar.getHeight()                     ~1-2ms
│   └── Other accessed modules (~5-8 typically)         ~5-30ms
└── AppRegistry.registerComponent()                     ~2-5ms

PHASE 5: REACT FIRST RENDER                             ~50-200ms
├── React.createElement for root component              ~1-2ms
├── Reconciler creates initial Fiber tree               ~20-80ms
│   ├── Each component's render function executes       
│   ├── useState initializers run                       
│   ├── useMemo computations for initial values         
│   └── Context provider values computed                
├── Effects scheduled (not yet executed)                ~1ms
├── Commit phase begins                                 ~10-40ms
│   ├── Shadow Tree nodes created via JSI               ~5-15ms
│   ├── Yoga layout calculation                         ~5-20ms
│   └── Layout results available                        ~1-2ms
└── useLayoutEffect callbacks execute                   ~5-20ms

PHASE 6: NATIVE VIEW MOUNTING                           ~30-80ms
├── Create native views (UIView hierarchy)              ~10-30ms
├── Set view properties (colors, text, images)          ~5-15ms
├── Position views according to Yoga layout             ~3-8ms
├── Core Animation layer tree composition               ~5-10ms
├── GPU renders first frame                             ~3-8ms
└── ★ FIRST MEANINGFUL PAINT ★                          

PHASE 7: POST-MOUNT (Time-to-Interactive)               ~50-200ms
├── useEffect callbacks execute                         ~20-100ms
│   ├── Data fetching initiated                         
│   ├── Analytics screen view tracked                   
│   ├── Listeners registered                            
│   └── Lazy initialization of deferred modules         
├── Network responses arrive → re-renders               ~30-100ms
└── ★ APP IS FULLY INTERACTIVE ★                        

─────────────────────────────────────────────────────────────────
TOTAL TIME TO FIRST MEANINGFUL PAINT:
  Well-optimized app, modern iPhone:     400-600ms
  Average app, modern iPhone:            600-1000ms
  Unoptimized app, modern iPhone:        1000-2000ms
  
  Well-optimized app, budget Android:    800-1200ms
  Average app, budget Android:           1500-3000ms
  Unoptimized app, budget Android:       3000-6000ms
─────────────────────────────────────────────────────────────────
```

### 6.2 What You Can Control (And What You Can't)

```
┌─────────────────────────────┬──────────────────────┬────────────────────┐
│ Phase                       │ Can You Optimize?    │ How                │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ Process creation + dyld     │ Minimal              │ Reduce frameworks, │
│                             │                      │ use fewer pods     │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ App delegate init           │ Yes, moderate        │ Defer SDK init,    │
│                             │                      │ async where possible│
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ Hermes startup              │ Minimal (already     │ Already optimized  │
│                             │ fast with AOT)       │ by Hermes          │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ JS bundle execution         │ YES — BIGGEST WIN    │ Reduce bundle size,│
│                             │                      │ lazy imports,      │
│                             │                      │ tree shaking       │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ TurboModule init            │ Yes (use TurboModules│ Already lazy with  │
│                             │ not legacy modules)  │ New Architecture   │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ React first render          │ YES — SECOND WIN     │ Minimize initial   │
│                             │                      │ tree, defer below- │
│                             │                      │ fold content       │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ Native view mounting        │ Moderate             │ Fewer views, use   │
│                             │                      │ Suspense           │
├─────────────────────────────┼──────────────────────┼────────────────────┤
│ Post-mount effects          │ Yes                  │ Defer non-critical │
│                             │                      │ effects            │
└─────────────────────────────┴──────────────────────┴────────────────────┘
```

### 6.3 Bundle Size and Cold Start: The Linear Relationship

There is an approximately linear relationship between JS bundle size and cold start time. More code = more bytecode = more time to load and execute.

```
BUNDLE SIZE → COLD START IMPACT (approximate, modern iPhone):

  Bundle Size     Load + Execute Time    Typical App
  ───────────     ─────────────────      ──────────────
  500 KB          ~50ms                  Minimal app
  1 MB            ~80ms                  Small app  
  2 MB            ~150ms                 Medium app
  5 MB            ~300ms                 Large app
  10 MB           ~550ms                 Very large app
  20 MB           ~1000ms                Needs optimization

  On budget Android, multiply these by 2-4x.

  EVERY MEGABYTE COSTS ~50-100ms ON iPhone, ~150-300ms ON BUDGET ANDROID.
  
  A library you import but barely use? It still costs you cold start time
  because its module factory runs during bundle execution.
```

### 6.4 Cold Start Optimization Checklist

Here's the practical checklist, ordered by impact:

```
HIGH IMPACT (save 100-500ms):
  □ Use Hermes (default since RN 0.70 — verify it's enabled)
  □ Use New Architecture / TurboModules (default since RN 0.76)
  □ Audit bundle size: remove unused dependencies
  □ Lazy-load screens with React.lazy + Suspense
  □ Defer non-critical SDK initialization (analytics, crash reporting)

MEDIUM IMPACT (save 50-200ms):
  □ Use re.pack or metro tree shaking to eliminate dead code
  □ Replace heavy libraries with lighter alternatives
    (moment → date-fns, lodash → individual lodash-es functions)
  □ Minimize the initial component tree (render placeholder, then fill)
  □ Use InteractionManager.runAfterInteractions for post-mount work

LOWER IMPACT (save 10-50ms):
  □ Optimize images: use WebP, right-size for device
  □ Pre-warm Hermes with inline requires for critical paths
  □ Minimize useLayoutEffect work (it blocks paint)
  □ Use React Suspense for progressive rendering
```

### 6.5 Case Studies

**Shopify:** Measured a 17-27% reduction in cold start after migrating to the New Architecture. The primary gains came from TurboModule lazy loading (eliminated 200ms of eager module initialization) and Fabric's removal of Bridge serialization during initial render.

**A Million Monkeys:** Measured 24% Android cold start reduction. Migration took 12 days with 2 developers — primarily auditing and updating third-party native modules. No JavaScript code changes required.

**Discord:** Reduced TTI from ~8s to ~4.5s on iPhone 6 through a combination of bundle size optimization (custom serialization for message data), deferred module loading, and reducing the initial render tree. The 90% improvement in parse time came from bypassing JSON.parse entirely for message data and using a binary format that was decoded in native code.

---

## 7. THE RENDERING PIPELINE END-TO-END

Let's trace a state update from `setState` to pixels on screen. This is the sequence that runs 60 times per second when your UI is animating.

### Step 1: State Update (JS Thread)

```tsx
const [count, setCount] = useState(0);
// User taps → setCount(1)
```

React queues the state update. In concurrent mode, this update is assigned a priority (user interactions are high priority, background data fetches are low priority).

When `setCount(1)` is called, React does NOT immediately re-render. It schedules a re-render via `scheduleMicrotask` or `MessageChannel` (depending on the React version). Multiple `setState` calls in the same synchronous execution batch into a single render.

```javascript
// These three calls produce ONE re-render, not three:
function handlePress() {
  setName('Alice');     // Queued
  setAge(30);           // Queued
  setActive(true);      // Queued
}
// React batches all three into a single reconciliation pass.
// This was already true in React 18+ (automatic batching).
```

### Step 2: Reconciliation (JS Thread)

React's Fiber reconciler walks the component tree, comparing the previous render with the new render:
- Same component type, same key → update existing fiber
- Different type or key → unmount old, mount new
- Props changed → mark for update

The reconciler produces a list of changes (the "effect list") — which components need to update, which need to mount/unmount, which need layout effects.

```
Reconciliation example — Counter component:

Previous tree:
  <View>
    <Text>"Count: 0"</Text>
    <Button title="Increment" />
  </View>

New tree (after setCount(1)):
  <View>                          ← Same type, same key → reuse
    <Text>"Count: 1"</Text>       ← Same type → check props → text changed → UPDATE
    <Button title="Increment" />  ← Same type → check props → unchanged → SKIP
  </View>

Effect list: [{ type: UPDATE, fiber: Text, changes: { text: "Count: 1" } }]
```

With **concurrent rendering** (enabled by Fabric), this reconciliation can be interrupted:

```
Concurrent reconciliation:

  [0ms]  Start reconciling a large list update (low priority)
  [4ms]  User taps a button (high priority event arrives)
  [4ms]  INTERRUPT: pause list reconciliation
  [4ms]  Start reconciling button press response (high priority)
  [6ms]  Finish button response → commit to screen
  [6ms]  RESUME: continue list reconciliation from where we left off
  [12ms] Finish list reconciliation → commit to screen
  
  Without concurrent rendering:
  [0ms]  Start reconciling list (blocking)
  [12ms] Finish list → commit
  [12ms] NOW handle button press (12ms delay — user notices)
```

### Step 3: Shadow Tree Update (Shadow Thread via JSI)

The effect list is applied to the Shadow Tree (C++ in-memory representation of the layout):
- New nodes are created for new components
- Existing nodes are updated with new props/styles
- Removed components' nodes are deleted

Yoga runs the Flexbox layout algorithm on the updated Shadow Tree, calculating the absolute position and size of every element.

```
Shadow Tree update for our counter:

1. JSI call from JS thread to C++ Shadow Tree:
   shadowTree.updateNode(textNodeId, { text: "Count: 1" })
   
2. Text node needs re-measurement (text changed → dimensions may change):
   Yoga marks the node as "dirty"
   
3. Yoga runs layout on the dirty subtree:
   - Text node: measure text "Count: 1" → { width: 72, height: 20 }
   - Parent View: recalculate if child size changed
   - If size unchanged: layout is complete (fast path)
   - If size changed: propagate up the tree (expensive path)
   
4. Final layout results:
   Text node: { x: 16, y: 16, width: 72, height: 20 }
   (absolute position within the screen coordinate system)
```

### Step 4: Commit (UI Thread)

The calculated layout is committed to the native view hierarchy:
- New native views are created (`UIView`, `android.view.View`)
- Existing views are updated with new props (text content, colors, sizes)
- Views are positioned according to the Yoga layout results
- Removed views are detached from the hierarchy

```
Commit phase for our counter:

iOS:
  UILabel *textLabel = existingViews[textNodeId];
  textLabel.text = @"Count: 1";
  textLabel.frame = CGRectMake(16, 16, 72, 20);  // From Yoga
  // No new view creation needed — just property update
  // ~0.1ms for this simple case

Android:
  TextView textView = existingViews.get(textNodeId);
  textView.setText("Count: 1");
  textView.layout(16, 16, 88, 36);  // From Yoga (left, top, right, bottom)
  // ~0.1ms for this simple case
```

### Step 5: Composite (GPU)

The native rendering system composites all the layers:
- iOS: Core Animation composites CALayers into a single frame buffer
- Android: RenderThread (hardware-accelerated pipeline) composites View hierarchy

The final frame is sent to the display.

```
Compositing:

iOS Core Animation:
  ┌──────────────────────────────┐
  │ Root CALayer                  │
  │  ├── View CALayer             │  (background color, border)
  │  │    ├── Text CALayer        │  (text content, rasterized)
  │  │    └── Button CALayer      │  (background + title sublayers)
  │  └── StatusBar overlay        │
  └──────────────────────────────┘
  
  Core Animation diffs the layer tree from last frame.
  Only the Text layer changed → only that layer re-rasterizes.
  Everything else is cached GPU textures from previous frame.
  
  GPU composites all layers into frame buffer → display.
  
  Total compositing time: ~1-2ms (GPU is FAST at this)
```

### Step 6: Display

The frame buffer is sent to the display controller:
- 60Hz display: new frame every 16.67ms
- 120Hz display (ProMotion): new frame every 8.33ms
- Variable refresh rate: display syncs to content update rate

If your entire pipeline (Steps 1-5) completes in less than one frame period, the update appears in the very next frame. If it takes longer, you skip one or more frames (jank).

### What Makes This Fast (New Architecture)

In the old architecture, steps 2-to-3 and 3-to-4 each crossed the Bridge with JSON serialization. In the New Architecture:
- Step 2 to 3: Direct JSI call from JS to the C++ Shadow Tree. No serialization.
- Step 3 to 4: Direct JSI call from C++ to native views. No serialization.

The serialization tax was applied twice per render cycle in the old architecture. With JSI, it's zero. At 60fps, that's 120 serialization operations per second that no longer happen.

### 7.1 A Single Tap Event: Complete Thread-by-Thread Trace

Let's trace a single tap event through the entire system, with specific timing on each thread. This is the most detailed view you'll find anywhere of what happens when a user interacts with a React Native app.

```
TIME    THREAD          ACTION                                    DURATION
─────   ──────          ──────                                    ────────
0.0ms   UI Thread       Screen hardware detects finger contact    —
0.2ms   UI Thread       IOKit delivers touch to UIApplication     0.2ms
0.3ms   UI Thread       Hit testing: which view was touched?      0.1ms
                         (Walk the native view hierarchy,
                          find the deepest view at touch coords)
0.5ms   UI Thread       UIGestureRecognizer begins tracking       0.2ms
0.7ms   UI Thread       Gesture Handler evaluates recognizers     0.2ms
                         (is this a tap? scroll? long press?)

        ── Wait for gesture to be recognized ──
        (tap recognized after finger lifts, ~100-150ms later)

150ms   UI Thread       Tap recognized → dispatch to JS           0.1ms
150.1ms UI Thread→JS    JSI calls into JS event system            0.2ms
150.3ms JS Thread       React event system receives touch event   0.1ms
150.4ms JS Thread       Synthetic event created (SyntheticEvent)  0.1ms
150.5ms JS Thread       Event bubbles through React tree          0.2ms
                         (capture phase → target → bubble phase)
150.7ms JS Thread       onPress handler executes                  0.5ms
                         setCount(prev => prev + 1)
151.2ms JS Thread       React schedules re-render                 0.1ms
151.3ms JS Thread       Reconciler begins                         —
151.3ms JS Thread       Counter component re-renders              0.3ms
                         (run function body, compute new JSX)
151.6ms JS Thread       Reconciler diffs: Text content changed    0.2ms
151.8ms JS Thread       Commit phase: effects scheduled           0.1ms

151.9ms JS→Shadow       JSI call: update Shadow Tree              0.3ms
152.2ms Shadow Thread   Yoga re-measures Text node                0.4ms
152.6ms Shadow Thread   Yoga recalculates layout (if needed)      0.2ms
152.8ms Shadow Thread   Layout result ready                       —

152.8ms Shadow→UI       JSI call: commit to native views          0.2ms
153.0ms UI Thread       Update UILabel text property              0.1ms
153.1ms UI Thread       Update frame if layout changed            0.1ms
153.2ms UI Thread       Core Animation marks layer dirty          0.05ms
153.25ms UI Thread      CATransaction committed                   0.05ms

153.3ms GPU             Rasterize changed Text layer              0.2ms
153.5ms GPU             Composite all layers into frame buffer    0.3ms
153.8ms Display         VSync — frame appears on screen           —

TOTAL: From touch to pixels = ~3.8ms of actual processing time
       From tap recognition to pixels = ~3.5ms
       User-perceived latency = ~153ms (dominated by gesture recognition wait)
       
NOTE: The 150ms gesture recognition delay is INHERENT to tap gestures.
      The system must wait for the finger to lift to distinguish
      tap from scroll from long-press.
      This is why Pressable has delayLongPress and why
      scroll views have a directionalLockEnabled property.
```

---

## 8. OLD ARCHITECTURE VS NEW ARCHITECTURE: COMPREHENSIVE COMPARISON

Here is the definitive comparison table. Bookmark this for architecture discussions and migration planning.

```
┌────────────────────────────┬──────────────────────────┬──────────────────────────┐
│ Dimension                  │ Old Architecture         │ New Architecture          │
│                            │ (Bridge)                 │ (JSI/Fabric/TurboModules) │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ JS-to-Native communication │ Async JSON serialization │ Sync/async C++ via JSI   │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Serialization format       │ JSON (text-based)        │ None (direct references) │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Call latency               │ 1-3 frames minimum       │ Same-frame possible      │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Synchronous native calls   │ Impossible               │ Supported via JSI        │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Native module loading      │ Eager (all at startup)   │ Lazy (on first use)      │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Type safety                │ Runtime only             │ Compile-time (Codegen)   │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Layout measurement         │ Async callback           │ Synchronous              │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Shadow Tree location       │ Java/ObjC per platform   │ Shared C++               │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Concurrent rendering       │ Not possible             │ Full React 18+ support   │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ JS engine flexibility      │ JSC only (effectively)   │ Any JSI-compliant engine │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Shared C++ native code     │ Not practical            │ First-class support      │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Animation architecture     │ JS-driven (jank-prone)   │ UI-thread worklets       │
│                            │ or useNativeDriver       │ (Reanimated 3+)          │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Event handling             │ Async (1+ frame delay)   │ Sync dispatch possible   │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Memory overhead            │ JSON copies + Bridge     │ Shared references,       │
│                            │ buffers                  │ no duplication           │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Cold start (typical)       │ Baseline                 │ 17-27% faster            │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Scroll performance (FPS)   │ 45-55 FPS on complex     │ 58-60 FPS on same list   │
│                            │ lists (budget Android)   │ (budget Android)         │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Default since              │ RN 0.1 (2015)           │ RN 0.76 (late 2024)      │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Expo default since         │ SDK < 52                 │ SDK 52+ (2025)           │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Library compatibility      │ All legacy libraries     │ Most major libraries     │
│                            │                          │ (interop layer for rest) │
├────────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Debugging experience       │ Chrome DevTools          │ Hermes debugger, Flipper,│
│                            │ (remote debugging)       │ React DevTools (direct)  │
└────────────────────────────┴──────────────────────────┴──────────────────────────┘
```

### Migration Decision Framework

```
SHOULD YOU MIGRATE TO THE NEW ARCHITECTURE?

IF you're starting a new project:
  → YES. Use New Architecture by default. No reason not to.
  → Expo SDK 52+ does this automatically.

IF you have an existing app on Old Architecture:
  → CHECK your native module dependencies first.
  → If ALL your native modules support New Arch → migrate now.
  → If 1-2 modules don't support it → check if alternatives exist.
  → If critical modules don't support it → wait or contribute PRs.

IF you have a brownfield app (RN embedded in native app):
  → Migration is more complex. Plan for 2-4 weeks of native work.
  → Test thoroughly — the interop layer handles most cases but not all.

MIGRATION EFFORT (typical):
  Expo managed workflow:  ~1-2 days (mostly testing)
  Expo bare workflow:     ~3-5 days (native config + testing)
  Pure React Native:      ~5-15 days (depends on native modules)
  Brownfield app:         ~10-30 days (depends on integration depth)
```

---

## 9. PRODUCTION WAR STORIES

These are real debugging scenarios where understanding React Native's internals was the difference between a quick fix and weeks of frustration. Each story illustrates a specific internal mechanism and how to diagnose it.

### War Story #1: The Phantom Frame Drops

**The symptom:** A fintech app's main dashboard screen showed consistent 5-8ms frame drops every 3-4 seconds, even when the screen was completely idle. No user interaction, no animations, no network activity. Just periodic stutters visible in the frame rate graph.

**The investigation:**

The team first suspected background timers or re-renders. They added `React.Profiler` wrappers around every component — no re-renders were happening during the drops. They checked network activity — nothing. They checked native module calls — nothing.

Finally, they captured a Hermes sampling profile and found the culprit: **garbage collection.**

```
Hermes Profiler Output (simplified):

[3247ms] GC: YG collection triggered
         Reason: allocation failure (YG full)
         Duration: 4.2ms
         Objects scanned: 12,847
         Objects freed: 11,203
         Objects promoted to OG: 1,644
         YG usage before: 3.8 MB / 4.0 MB
         YG usage after: 0.4 MB / 4.0 MB

[6891ms] GC: YG collection triggered
         Reason: allocation failure (YG full)
         Duration: 5.1ms
         Objects scanned: 14,102
         Objects freed: 12,388
         Objects promoted to OG: 1,714
```

The YG was filling up every 3-4 seconds and triggering a collection. But the screen was idle — where were the allocations coming from?

**The root cause:** A `setInterval` running every 1 second to check the user's authentication token expiry. The check function was creating closures and temporary objects:

```javascript
// THE PROBLEM CODE
useEffect(() => {
  const interval = setInterval(() => {
    const token = authStore.getState().token;        // Object access
    const decoded = jwtDecode(token);                 // Creates new object
    const now = Date.now();                           // Creates Number object (boxed)
    const timeLeft = decoded.exp * 1000 - now;        // More allocations
    
    if (timeLeft < 300000) {                          // 5 minutes
      refreshToken();
    }
    
    // Each tick creates ~50 intermediate objects
    // 1 tick/sec × 50 objects × 3-4 seconds = 150-200 objects
    // These fill the YG and trigger collection
  }, 1000);
  
  return () => clearInterval(interval);
}, []);
```

**The fix:**

```javascript
// THE FIX: minimize allocations, check less frequently
useEffect(() => {
  // Check every 60 seconds instead of every 1 second
  // Token expiry doesn't need second-level precision
  const interval = setInterval(() => {
    // Use a native module for token expiry check (zero JS allocations)
    const timeLeftMs = NativeTokenManager.getExpiryTimeLeft();
    if (timeLeftMs < 300000) {
      refreshToken();
    }
  }, 60000);
  
  return () => clearInterval(interval);
}, []);
```

**The lesson:** Hermes GC is fast but not free. On idle screens, any periodic allocation pattern will eventually trigger visible GC pauses. The fix is always: allocate less, not collect faster.

---

### War Story #2: The Android Navigation Freeze

**The symptom:** On Android (specifically Samsung devices with Android 12-13), navigating from the home screen to a detail screen occasionally caused a 2-3 second freeze. The app didn't crash — it just hung, then suddenly the detail screen appeared. It happened about 10% of the time, was not reproducible on iOS, and was not reproducible on Pixel devices.

**The investigation:**

The team suspected it was a Samsung-specific rendering issue. They added performance marks around the navigation:

```javascript
performance.mark('nav-start');
navigation.navigate('Detail', { id: item.id });

// In the Detail screen:
useEffect(() => {
  performance.mark('nav-end');
  performance.measure('nav-time', 'nav-start', 'nav-end');
  const measure = performance.getEntriesByName('nav-time')[0];
  console.log(`Navigation took: ${measure.duration}ms`);
}, []);
```

The measurements showed: normal navigation took ~200ms. The freeze cases showed ~2500-3000ms. But the JS thread profiler showed normal execution times — the JS work was completing in ~200ms. The extra 2300ms was not in JavaScript.

They enabled Android systrace and found the problem: **the UI thread was blocked by a synchronous `SharedPreferences.commit()` call.**

```
Android Systrace output:

UI Thread:
  [0ms]     dispatchTouchEvent
  [0.5ms]   React Navigation triggers screen transition
  [1ms]     NativeScreenManager.pushScreen()
  [1.5ms]   AnalyticsModule.trackScreenView()  ← THIS IS THE PROBLEM
  [1.5ms]     SharedPreferences.edit().commit()  ← SYNCHRONOUS DISK I/O
  [2500ms]  SharedPreferences.commit() returns   ← 2.5 SECONDS of disk I/O
  [2500ms]  Screen transition continues
  [2700ms]  Screen visible

The cause: Samsung's implementation of SharedPreferences.commit() 
on Android 12-13 had a known performance issue where commit() 
could block for seconds if the system was doing a background 
flush of pending writes from other apps.
```

**The root cause:** A third-party analytics library was calling `SharedPreferences.commit()` (synchronous) instead of `SharedPreferences.apply()` (asynchronous) on the UI thread. On most devices, this took <1ms. On Samsung devices under specific conditions (low storage, background flushes), it could take seconds.

**The fix:**

```java
// In the native analytics module:

// BEFORE (blocking):
SharedPreferences.Editor editor = prefs.edit();
editor.putString("last_screen", screenName);
editor.commit();  // BLOCKS UI THREAD until disk write completes

// AFTER (non-blocking):
SharedPreferences.Editor editor = prefs.edit();
editor.putString("last_screen", screenName);
editor.apply();   // Queues write to background thread, returns immediately
```

**The lesson:** Native thread blocking is invisible from JavaScript. Your JS profiler will show nothing wrong. You need platform-specific profiling tools (systrace on Android, Instruments on iOS) to find native-side bottlenecks. And synchronous disk I/O on the UI thread is NEVER acceptable, even if it's "usually fast."

---

### War Story #3: The Memory Leak That Wasn't

**The symptom:** A social media app's memory grew steadily over time during a session. After 30 minutes of use, the app would crash on low-end Android devices with an OOM (Out of Memory) error. The Hermes heap showed linear growth: ~150 MB at startup, growing to ~400 MB+ over 30 minutes.

**The investigation:**

The team's first instinct was to look for JavaScript memory leaks — event listeners not cleaned up, closures holding references, state that accumulated without pruning. They took Hermes heap snapshots at 5-minute intervals and compared them:

```
Heap Snapshot Comparison:

  T+0min:   150 MB total, 45 MB reachable
  T+5min:   185 MB total, 78 MB reachable
  T+10min:  220 MB total, 112 MB reachable
  T+15min:  260 MB total, 148 MB reachable
  T+20min:  295 MB total, 180 MB reachable
  
  Growth rate: ~7.5 MB/min
  
  Top growing object types:
    1. ArrayBuffer — growing by ~4 MB/min
    2. Object — growing by ~2 MB/min
    3. String — growing by ~1 MB/min
```

ArrayBuffers were the main culprit. The team traced them to their image caching system. They were using a JavaScript-side LRU cache for decoded image data:

```javascript
// THE PROBLEM: JavaScript-side image cache
class ImageCache {
  private cache = new Map<string, ArrayBuffer>();
  private maxSize = 100; // max 100 images

  set(url: string, data: ArrayBuffer) {
    if (this.cache.size >= this.maxSize) {
      // Evict oldest entry
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(url, data);
  }
}

// The problem: each ArrayBuffer held decoded image data
// A typical social media image: 1080x1080 pixels × 4 bytes (RGBA) = ~4.7 MB
// Cache of 100 images = ~470 MB of ArrayBuffers in the JS heap
// Even with eviction, the peak was too high for low-end devices
```

But here was the twist: the LRU cache was working correctly — it was evicting old entries. The Map never had more than 100 entries. So why was memory growing?

**The actual root cause:** The evicted ArrayBuffers were being promoted to Hermes's Old Generation before eviction. Once in OG, they were collected during OG collection — but OG collections are infrequent and incremental. The JS heap reported 180 MB "reachable" because the OG collection hadn't run yet to reclaim the evicted buffers.

The "leak" was actually **GC pressure from large objects that survived long enough to be promoted to OG, then were evicted but not yet collected.**

**The fix:**

```javascript
// FIX 1: Don't cache decoded image data in JS at all.
// Use the native image cache (NSURLCache on iOS, OkHttp cache on Android).
// These caches are outside the JS heap and managed by the OS.

// FIX 2: If you must cache in JS, reduce the cache size and use
// a native module for the actual storage:
const MAX_CACHE_SIZE = 20; // Not 100

// FIX 3: Force Hermes to do an OG collection when memory is high:
// (Available via Hermes debug APIs)
if (__DEV__) {
  if (global.HermesInternal?.collectGarbage) {
    global.HermesInternal.collectGarbage();
  }
}

// FIX 4 (best): Move image caching to a native TurboModule
// that uses platform-appropriate caching (no JS heap impact):
NativeImageCache.cacheImage(url, imageData);
// Native side handles LRU eviction, disk caching, memory management
// Completely outside the Hermes heap
```

**The lesson:** Not every memory growth pattern is a "leak." Sometimes it's GC scheduling — objects are unreachable but haven't been collected yet. Hermes's incremental OG collector trades prompt collection for lower pause times. If you're working with large objects (images, audio buffers, large JSON payloads), keep them out of the JS heap entirely. Use native modules for large data management.

---

### War Story #4: The Bridge Queue Starvation

**The symptom (old architecture):** A messaging app experienced a specific failure mode: when a user received a burst of messages while scrolling, the scroll would completely freeze for 500ms-1s, then suddenly "catch up" with a visible jump.

**The investigation:**

The team captured Bridge traffic during the issue. The Bridge was processing messages in this order:

```
Bridge Queue at time of freeze:

Position  | Message Type          | Payload Size
─────────────────────────────────────────────────
1         | updateView (scroll)   | 120 bytes
2         | updateView (scroll)   | 120 bytes
3         | createView (message)  | 1,800 bytes
4         | createView (avatar)   | 450 bytes
5         | createView (text)     | 2,200 bytes
6         | updateView (scroll)   | 120 bytes    ← scroll event, stuck behind messages
7         | createView (message)  | 1,800 bytes
8         | createView (avatar)   | 450 bytes
9         | createView (text)     | 2,400 bytes
10        | createView (message)  | 1,800 bytes
... (50 more message creation calls)
62        | updateView (scroll)   | 120 bytes    ← another scroll event, buried
```

The Bridge was FIFO (First In, First Out). New message views were being created between scroll events. The scroll events were trapped behind expensive view creation operations. The native side was spending all its time deserializing and creating message views, and the scroll position updates were delayed by hundreds of milliseconds.

**The root cause:** The Bridge had no priority system. A scroll event (which the user is actively watching) had the same priority as a background message render (which could be deferred). The old architecture treated all Bridge messages equally.

**The fix (at the time):** The team implemented a custom batching strategy that separated scroll events from render events and processed them on different schedules. This was a complex, fragile hack.

**Why this doesn't happen in the New Architecture:** Fabric supports priority-based rendering via React's concurrent features. User interactions (scroll events) are high priority and processed immediately. Background updates (new messages arriving) are lower priority and can be interrupted or deferred. There is no single queue bottleneck.

```javascript
// New Architecture: concurrent rendering handles this naturally
function MessageList() {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    socket.on('new-messages', (batch) => {
      // startTransition marks this as low-priority
      startTransition(() => {
        setMessages(prev => [...prev, ...batch]);
      });
      // Even during this render, scroll events are processed immediately
      // because they're high-priority user interactions.
    });
  }, []);

  return <FlatList data={messages} /* ... */ />;
}
```

**The lesson:** The Bridge's single-queue architecture made it fundamentally impossible to prioritize user-visible updates over background work. This was an architectural limitation, not a bug. The New Architecture's concurrent rendering model solves this at the framework level. If you're still on the old architecture and hitting this pattern, the only real fix is migration.

---

### War Story #5: The Hermes Bytecode Version Mismatch

**The symptom:** After an OTA update via EAS Update, a subset of users (~2% of the user base) experienced immediate crashes on app launch. The crashes were native-level (no JavaScript stack trace), and Crashlytics showed the crash originating in Hermes's bytecode loader.

**The investigation:**

The crash signature was:
```
HermesVM: Bytecode version mismatch.
  Expected: 96
  Got: 93
  
Fatal error in hermes::vm::RuntimeConfig::loadBytecodeFile
```

The app had been built with Hermes 0.12 (bytecode version 96), but the OTA update's JavaScript bundle was compiled with Hermes 0.11 (bytecode version 93). How?

The team had two CI pipelines:
1. **App builds** (eas build) — used Expo SDK 52, which ships Hermes 0.12
2. **OTA updates** (eas update) — ran locally on a developer's machine, which had an older Expo CLI that used Hermes 0.11

The OTA update compiled the JS bundle to bytecode format 93 and pushed it. Users whose app binary contained Hermes 0.12 (format 96) received the update, tried to load the mismatched bytecode, and crashed.

The 2% crash rate corresponded to users who had the latest app binary AND received the OTA update before the CI pipeline was fixed.

**The fix:**
```bash
# Ensure OTA updates use the SAME Hermes version as the app binary

# Option 1: Pin Expo CLI version in CI
npx expo@~52.0.0 export  # Matches Expo SDK 52's Hermes

# Option 2: Build OTA updates in the same CI environment as app builds
# (use eas update with the same EAS Build environment)
eas update --branch production --channel production

# Option 3: Version check in the update manifest
# EAS Update includes runtimeVersion that prevents this mismatch
# ALWAYS set runtimeVersion in app.json:
{
  "expo": {
    "runtimeVersion": {
      "policy": "fingerprint"  // Automatically matches native binary
    }
  }
}
```

**The lesson:** Hermes bytecode is not forward-compatible or backward-compatible. The bytecode version must exactly match the Hermes runtime version. When using OTA updates, your CI pipeline for updates must use the exact same Hermes compiler version as your app builds. The `runtimeVersion` field in Expo's configuration exists specifically to prevent this class of error — always use it.

---

## 10. PRODUCTION METRICS: WHAT COMPANIES ACTUALLY MEASURED

These aren't theoretical — they're from engineering blog posts, conference talks, and published case studies.

| Company | Migration | Metric | Before | After | Improvement |
|---------|-----------|--------|--------|-------|-------------|
| Shopify | New Architecture | Cold start | Baseline | -17-27% | TurboModules + Fabric |
| Meta | Hermes engine | TTI (Android) | Baseline | ~2x faster | AOT bytecode |
| Discord | Various optimizations | TTI (iPhone 6) | ~8s | ~4.5s | -3,500ms |
| Discord | Message list optimization | Parse time | Baseline | 90% faster | Custom serialization |
| A Million Monkeys | New Architecture | Android cold start | Baseline | -24% | 12-day migration |
| Shopify | Bundle optimization | Bundle size | 50MB | 11MB | -78% |
| Microsoft (Teams) | Hermes adoption | Memory (Android) | ~180 MB | ~90 MB | -50% |
| Coinbase | New Architecture | List scroll FPS | 47 FPS avg | 58 FPS avg | +23% |
| Wix | Fabric migration | TTI (low-end) | 3.2s | 2.1s | -34% |

### The Meta-Lesson

Every company that published performance improvements reported the same pattern: **the biggest wins came from understanding the internals, not from applying surface-level tips.** Shopify didn't speed up their cold start by tweaking React components — they migrated to TurboModules. Discord didn't fix their scroll performance by adding `React.memo` everywhere — they understood the Bridge bottleneck and worked around it. Meta didn't improve TTI by code-splitting — they built an entirely new JavaScript engine.

The internals aren't academic. They're the debugging superpower that separates engineers who fix symptoms from engineers who fix causes.

---

## 11. PUTTING IT ALL TOGETHER

Here's the mental model you should carry with you for the rest of this guide:

```
┌──────────────────────────────────────────────────────────┐
│                     YOUR APP CODE                         │
│              (TypeScript, React Components)                │
└───────────────────────────┬──────────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │  HERMES (JS)  │  ← AOT bytecode, generational GC,
                    │               │    single-threaded, register-based VM
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │      JSI      │  ← C++ bridge-less interface
                    │               │    HostObjects, synchronous calls,
                    │               │    engine-agnostic abstraction
                    └───┬───────┬───┘
                        │       │
              ┌─────────▼─┐ ┌──▼──────────┐
              │  FABRIC    │ │ TURBOMODULES│
              │ (Rendering)│ │ (Native API)│
              │            │ │             │
              │ C++ Shadow │ │ Lazy loaded │
              │ Tree +     │ │ Type-safe   │
              │ Yoga +     │ │ Codegen     │
              │ Concurrent │ │ Sync calls  │
              └─────┬──────┘ └──────┬──────┘
                    │               │
              ┌─────▼───────────────▼──────┐
              │    NATIVE VIEW HIERARCHY    │
              │  (UIKit / Android Views)    │
              │                             │
              │  Three threads:             │
              │  - JS Thread (your code)    │
              │  - UI Thread (rendering)    │
              │  - Shadow Thread (layout)   │
              └─────────────┬──────────────┘
                            │
                    ┌───────▼───────┐
                    │      GPU      │  → Pixels on screen
                    │               │    Core Animation (iOS)
                    │               │    RenderThread (Android)
                    └───────────────┘
```

### The Key Takeaways

1. **JSI eliminated the Bridge.** Everything is direct C++ calls now. This is the single biggest architectural improvement in React Native's history. No more JSON serialization tax. No more async-only communication. No more single-queue bottleneck.

2. **Hermes compiles ahead of time.** Your JS is bytecode before it reaches the device. This is why cold starts are fast. Hermes trades peak CPU throughput for startup speed, memory efficiency, and predictable performance — the right tradeoff for mobile.

3. **Three threads, one orchestration.** JS thread for logic, Shadow thread for layout, UI thread for rendering. Keep them unblocked. Know which thread owns the bottleneck. Profile on the right thread.

4. **TurboModules are lazy.** They load when needed, not at startup. This directly reduces cold start time. Type safety at compile time, not runtime.

5. **Fabric is synchronous.** Layout measurements are available in the same frame. Concurrent rendering enables priority-based updates. Immutable tree operations enable safe multi-thread access.

6. **Memory has a budget.** Hermes GC is generational. YG collections are fast but frequent. OG collections are infrequent but can cause jank. Keep large data out of the JS heap. Target 150 MB for universal compatibility.

7. **The architecture determines the ceiling.** No amount of React optimization can overcome a fundamental architecture limitation. The shift from Bridge to JSI didn't just make things faster — it made entire categories of previously-impossible things possible (synchronous native calls, concurrent rendering, worklet-based animations).

With this foundation, everything else in this guide — Expo's abstractions, state management patterns, performance optimization, profiling — will make visceral sense instead of being arbitrary rules to memorize. When Chapter 13 tells you to use Reanimated worklets instead of JS-driven animations, you'll know *why*: because worklets run on the UI thread via JSI, independent of JS thread congestion. When Chapter 14 tells you to profile on the right thread, you'll know *which* threads exist and what runs on each one.

The internals are not trivia. They're the lens through which every performance decision becomes obvious.

---

*Next: [Chapter 2 — Browser Rendering & Web Fundamentals](./02-browser-rendering.md) — the web side of the rendering equation.*
