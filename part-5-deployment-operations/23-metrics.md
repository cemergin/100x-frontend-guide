<!--
  CHAPTER: 25
  TITLE: Mobile Metrics That Matter
  PART: V — Deployment & Operations
  PREREQS: Chapter 20
  KEY_TOPICS: TTI, FPS, memory, Play Console vitals, App Store Connect, Datadog RUM, dashboards, alerts
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 23: Mobile Metrics That Matter

> **Part V — Deployment & Operations** | Prerequisites: Chapter 20 | Difficulty: Intermediate

> "Not everything that counts can be counted, and not everything that can be counted counts." — William Bruce Cameron
>
> "But in mobile, the things that count CAN be counted. So count them." — Me, after a P0 incident caused by a metric nobody was watching

---

<details>
<summary><strong>TL;DR</strong></summary>

- Build a metrics dashboard on day one that covers cold start, FPS, memory, app size, crash-free rate, and network latency; if nobody is watching a metric, it will degrade silently
- Google Play Console Vitals has hard thresholds (ANR > 0.47%, crash rate > 1.09%) that trigger search ranking penalties; monitor these weekly
- App size directly impacts install conversion; apps over 150MB trigger a download warning on mobile data that can kill 25% of new installs
- Use Datadog RUM (or similar) for real user monitoring that captures performance data from actual devices in the field, not just your test lab
- Set alerts with meaningful thresholds and clear ownership; a dashboard nobody looks at is the same as no dashboard at all

</details>

## The Metric That Almost Killed an App

Let me tell you about a team that shipped a beautiful React Native app. It looked gorgeous. The animations were smooth. The feature set was complete. The reviews were 4.5 stars.

Then their Android installs started dropping. Not crashing — dropping. Fewer new installs every week, even though their marketing spend was the same. The PM spent three weeks blaming the ad creative. The designer redesigned the app store screenshots. Nothing helped.

It took them six weeks to discover the actual cause: their app was 180MB. On the Google Play Store, apps over 150MB show a warning to users on mobile data. That warning was killing their conversion rate by an estimated 25%. The Play Console had been showing this data the entire time — under "App size" in the statistics tab. Nobody was looking.

Six weeks. Tens of thousands of lost installs. Because nobody set up a dashboard that included app size as a metric.

This chapter is the dashboard you should have built on day one. We'll cover every metric that matters for a mobile app, where to find it, what the thresholds are, and how to set up alerts so you never have to learn lessons the hard way.

### In This Chapter
- Cold Start / Time to Interactive (TTI)
- FPS and Jank Detection
- Memory Usage Tracking
- Network Latency Metrics
- App Size and Download Impact
- Google Play Console Vitals (with thresholds)
- App Store Connect Metrics
- Datadog RUM for Mobile
- Building Dashboards That People Actually Look At
- Alert Thresholds and On-Call

### Related Chapters
- [Ch 20: Mobile Monitoring & Observability] — crash tracking, error reporting, Sentry/Crashlytics
- [Ch 13: Performance Optimization] — how to fix the problems these metrics reveal
- [Ch 6: EAS Mastery] — build configuration that affects app size
- [Ch 21: Firebase] — Firebase Performance Monitoring integration

---

## 1. COLD START AND TIME TO INTERACTIVE

Cold start is the first impression your app makes. It's the time from when the user taps the app icon to when they can interact with the first meaningful screen. Get this wrong and you've lost users before they've even seen your UI.

### 1.1 What "Cold Start" Actually Means

There are three types of app start on both iOS and Android:

| Start Type | What Happens | Typical Time |
|-----------|--------------|-------------|
| **Cold Start** | App process doesn't exist. OS creates process, loads binary, runs initialization, renders first frame. | 1-5 seconds |
| **Warm Start** | App process exists but activity was destroyed. Recreates activity, may need to reload some state. | 0.5-2 seconds |
| **Hot Start** | App is in background, still in memory. Just brings it to foreground. | < 500ms |

Cold start is the one you optimize for because it's the worst case — and it's what new users experience.

### 1.2 Measuring Cold Start in React Native

React Native cold start has multiple phases. You need to instrument each one:

```typescript
// src/performance/startup.ts

import { PerformanceObserver } from 'react-native-performance';

class StartupTracker {
  private marks: Record<string, number> = {};
  private readonly startTime: number;
  
  constructor() {
    // global.__startTime should be set in index.js, as early as possible
    this.startTime = (global as any).__startTime || Date.now();
  }
  
  mark(name: string) {
    this.marks[name] = Date.now();
  }
  
  getMetrics() {
    const now = Date.now();
    return {
      // Total cold start time
      totalColdStart: (this.marks['interactive'] || now) - this.startTime,
      
      // Phase breakdown
      nativeInit: (this.marks['js_bundle_start'] || now) - this.startTime,
      bundleLoad: (this.marks['js_bundle_end'] || 0) - (this.marks['js_bundle_start'] || 0),
      appInit: (this.marks['app_render_start'] || 0) - (this.marks['js_bundle_end'] || 0),
      firstRender: (this.marks['first_render'] || 0) - (this.marks['app_render_start'] || 0),
      interactive: (this.marks['interactive'] || 0) - (this.marks['first_render'] || 0),
    };
  }
}

export const startupTracker = new StartupTracker();
```

```typescript
// index.js — the VERY FIRST LINE of your app
(global as any).__startTime = Date.now();

import { AppRegistry } from 'react-native';
import { startupTracker } from './src/performance/startup';

startupTracker.mark('js_bundle_start');

// ... after imports are done
startupTracker.mark('js_bundle_end');

import App from './App';
AppRegistry.registerComponent('MyApp', () => App);
```

```tsx
// App.tsx
import { startupTracker } from './src/performance/startup';

function App() {
  startupTracker.mark('app_render_start');
  
  useEffect(() => {
    startupTracker.mark('first_render');
    
    // Wait for the first frame to be painted
    requestAnimationFrame(() => {
      startupTracker.mark('interactive');
      
      const metrics = startupTracker.getMetrics();
      console.log('Startup metrics:', metrics);
      
      // Report to your analytics
      analytics.track('app_cold_start', {
        total_ms: metrics.totalColdStart,
        native_init_ms: metrics.nativeInit,
        bundle_load_ms: metrics.bundleLoad,
        app_init_ms: metrics.appInit,
        first_render_ms: metrics.firstRender,
        interactive_ms: metrics.interactive,
      });
    });
  }, []);
  
  return <RootNavigator />;
}
```

### 1.3 Cold Start Thresholds

Here are the numbers you should be targeting:

| Metric | Good | Acceptable | Bad | Critical |
|--------|------|-----------|-----|----------|
| **Total cold start** | < 1s | 1-2s | 2-4s | > 4s |
| **Native init** | < 200ms | 200-500ms | 500ms-1s | > 1s |
| **JS bundle load** | < 300ms | 300-600ms | 600ms-1s | > 1s |
| **First render** | < 300ms | 300-500ms | 500ms-1s | > 1s |
| **Time to interactive** | < 1.5s | 1.5-3s | 3-5s | > 5s |

**Where these numbers come from:**

- Google's research shows that **53% of mobile users abandon a site/app that takes longer than 3 seconds to load.** This applies to apps too.
- Apple's App Launch Time guideline says your app should be interactive within **400ms of the user tapping the icon** — though in practice, this is aspirational for React Native apps.
- Google Play Console flags apps with cold start times above **5 seconds** as having a startup time issue.

### 1.4 Common Cold Start Killers in React Native

```typescript
// KILLER 1: Importing everything at the top level
// Every import in your entry file runs during startup

// BAD — all of this runs before the first frame
import { HeavyChartLibrary } from 'heavy-chart-library';
import { PDFGenerator } from 'pdf-generator';
import { VideoPlayer } from 'video-player';
import { ARView } from 'ar-module';

// GOOD — lazy import what you don't need immediately
const HeavyChartLibrary = lazy(() => import('heavy-chart-library'));
const PDFGenerator = lazy(() => import('pdf-generator'));

// KILLER 2: Synchronous storage reads during init
// BAD
const token = MMKV.getString('auth_token');  // Blocks the JS thread
const user = JSON.parse(MMKV.getString('user') || '{}');  // More blocking
const settings = JSON.parse(MMKV.getString('settings') || '{}');

// GOOD — defer non-critical reads
const token = MMKV.getString('auth_token');  // Only read what's critical for first screen
// Read the rest after first render

// KILLER 3: Too many providers wrapping your app
// Each provider's initialization runs before your first screen renders
// BAD
<Provider1>
  <Provider2>
    <Provider3>
      <Provider4>
        <Provider5>
          <Provider6>
            <App />  // This doesn't render until all 6 providers initialize
          </Provider6>
        </Provider5>
      </Provider4>
    </Provider3>
  </Provider2>
</Provider1>

// GOOD — only wrap with providers needed for the first screen
// Defer others with lazy loading
```

### 1.5 Measuring TTI in Production

For real production measurement, use Firebase Performance Monitoring or a custom trace:

```typescript
import perf from '@react-native-firebase/perf';

// Create a custom trace for cold start
async function trackColdStart() {
  const trace = await perf().newTrace('cold_start');
  await trace.start();
  
  // Add metrics for each phase
  trace.putMetric('native_init_ms', startupTracker.getMetrics().nativeInit);
  trace.putMetric('bundle_load_ms', startupTracker.getMetrics().bundleLoad);
  trace.putMetric('first_render_ms', startupTracker.getMetrics().firstRender);
  
  // Add attributes for segmentation
  trace.putAttribute('device_tier', getDeviceTier());  // 'low', 'mid', 'high'
  trace.putAttribute('app_version', getAppVersion());
  trace.putAttribute('js_engine', global.HermesInternal ? 'hermes' : 'jsc');
  
  await trace.stop();
}
```

---

## 2. FPS AND JANK

Your app runs at 60 frames per second (or 120fps on ProMotion devices). That means each frame has a budget of **16.67ms** (or 8.33ms at 120fps). Miss that budget and you get jank — visible stuttering that makes your app feel broken.

### 2.1 Understanding the Frame Budget

```
60fps = 1000ms / 60 = 16.67ms per frame

In that 16.67ms, the system needs to:
1. Run your JS code (event handlers, state updates, rendering)
2. Run layout calculations
3. Run the UI thread (Yoga layout, view flattening)
4. Composite and render to the screen

Your JS code gets roughly 12ms of that budget.
The rest goes to system overhead.

At 120fps (ProMotion):
1000ms / 120 = 8.33ms per frame
Your JS code gets roughly 5-6ms.
```

### 2.2 Measuring FPS

```typescript
// src/performance/fps-monitor.ts

class FPSMonitor {
  private frameCount = 0;
  private lastTime = 0;
  private fps = 60;
  private droppedFrames = 0;
  private isRunning = false;
  private readonly callback: (fps: number, dropped: number) => void;
  
  constructor(callback: (fps: number, dropped: number) => void) {
    this.callback = callback;
  }
  
  start() {
    this.isRunning = true;
    this.lastTime = performance.now();
    this.frameCount = 0;
    this.tick();
  }
  
  stop() {
    this.isRunning = false;
  }
  
  private tick = () => {
    if (!this.isRunning) return;
    
    const now = performance.now();
    this.frameCount++;
    
    const elapsed = now - this.lastTime;
    
    // Report every second
    if (elapsed >= 1000) {
      this.fps = Math.round((this.frameCount * 1000) / elapsed);
      this.droppedFrames = Math.max(0, 60 - this.fps);
      
      this.callback(this.fps, this.droppedFrames);
      
      this.frameCount = 0;
      this.lastTime = now;
    }
    
    requestAnimationFrame(this.tick);
  };
}

// Usage in development
if (__DEV__) {
  const monitor = new FPSMonitor((fps, dropped) => {
    if (fps < 45) {
      console.warn(`Low FPS: ${fps} (${dropped} dropped frames)`);
    }
  });
  monitor.start();
}
```

### 2.3 Using React Native Performance Monitor

In development, you can enable the built-in performance monitor:

```typescript
// Enable in dev builds
if (__DEV__) {
  // Shake the device → Dev Menu → Show Perf Monitor
  // Or programmatically:
  const { NativeModules } = require('react-native');
  NativeModules.DevSettings?.setIsDebuggingRemotely?.(false);
  // The perf monitor shows:
  // - JS thread FPS
  // - UI thread FPS
  // - RAM usage
  // - Views count
}
```

### 2.4 FPS Thresholds and What They Mean

| FPS Range | User Perception | Action Required |
|-----------|----------------|-----------------|
| **58-60** | Buttery smooth | None — you're golden |
| **50-57** | Slight micro-jank, most users won't notice | Monitor, optimize if trending down |
| **40-49** | Noticeable stuttering during animations/scroll | Investigate — check JS thread blocking |
| **30-39** | Clearly janky, feels broken | P1 fix — users will leave reviews about this |
| **< 30** | Unusable, slideshow | P0 — stop everything and fix this |

### 2.5 Common Jank Causes in React Native

```typescript
// CAUSE 1: Heavy computation on the JS thread
// BAD — sorting 10,000 items on every render
function ProductList({ products }: { products: Product[] }) {
  const sorted = products.sort((a, b) => a.price - b.price);  // Runs every render!
  return <FlatList data={sorted} ... />;
}

// GOOD — memoize expensive computations
function ProductList({ products }: { products: Product[] }) {
  const sorted = useMemo(
    () => [...products].sort((a, b) => a.price - b.price),
    [products]
  );
  return <FlatList data={sorted} ... />;
}

// CAUSE 2: Re-rendering too many components
// BAD — context change re-renders everything
const AppContext = createContext({ user: null, theme: 'light', locale: 'en' });
// Changing theme re-renders EVERY component that reads user or locale

// GOOD — split contexts by update frequency
const ThemeContext = createContext('light');
const UserContext = createContext(null);
const LocaleContext = createContext('en');

// CAUSE 3: Images without fixed dimensions in lists
// BAD — image dimensions calculated during scroll
<FlatList
  renderItem={({ item }) => (
    <Image source={{ uri: item.image }} style={{ width: '100%' }} />
    // Height is unknown → layout thrash during scroll
  )}
/>

// GOOD — fixed dimensions
<FlatList
  renderItem={({ item }) => (
    <Image 
      source={{ uri: item.image }} 
      style={{ width: SCREEN_WIDTH, height: SCREEN_WIDTH * 0.75 }}
    />
  )}
  getItemLayout={(data, index) => ({
    length: SCREEN_WIDTH * 0.75 + 16,  // item height + margin
    offset: (SCREEN_WIDTH * 0.75 + 16) * index,
    index,
  })}
/>

// CAUSE 4: console.log in production
// Every console.log serializes its arguments on the JS thread
// In a FlatList with 1000 items, that's 1000 serializations per scroll
// ALWAYS strip console.log in production builds:
// babel.config.js:
// plugins: ['transform-remove-console']
```

### 2.6 Measuring Jank in Production

For production jank measurement, report dropped frames to your analytics:

```typescript
// src/performance/jank-reporter.ts

interface JankEvent {
  timestamp: number;
  droppedFrames: number;
  duration: number;
  screen: string;
  interaction: string;  // 'scroll', 'animation', 'transition', 'idle'
}

class JankReporter {
  private events: JankEvent[] = [];
  private readonly threshold = 3; // Report if > 3 frames dropped in a window
  
  report(event: JankEvent) {
    if (event.droppedFrames > this.threshold) {
      this.events.push(event);
      
      // Batch send every 30 seconds
      if (this.events.length >= 10) {
        this.flush();
      }
    }
  }
  
  flush() {
    if (this.events.length === 0) return;
    
    const batch = [...this.events];
    this.events = [];
    
    analytics.track('jank_events', {
      events: batch,
      device_tier: getDeviceTier(),
      app_version: getAppVersion(),
      free_memory_mb: getAvailableMemory(),
    });
  }
}
```

---

## 3. MEMORY USAGE

Memory management in mobile is not like web. Browsers can swap to disk. Mobile apps get killed. If your app uses too much memory, the OS will terminate it — no warning, no error, no crash report. The user just sees the app disappear.

### 3.1 Understanding Mobile Memory

```
Typical memory budgets (approximate):

iPhone SE (3GB RAM):      ~300-400MB for your app
iPhone 15 (6GB RAM):      ~600-800MB for your app
iPhone 15 Pro (8GB RAM):  ~800MB-1GB for your app

Samsung Galaxy A14 (4GB): ~200-300MB for your app
Samsung Galaxy S24 (8GB): ~600-800MB for your app

The OS gives your app a fraction of total RAM.
Background apps, the OS itself, and system services take the rest.
```

### 3.2 Memory Warning Thresholds

| Platform | Warning Level | Action |
|----------|--------------|--------|
| **iOS** | `didReceiveMemoryWarning` | OS is running low. Free caches, drop non-essential data. |
| **iOS** | Jetsam kill | Your app exceeded its memory limit. Terminated immediately. No crash report from Crashlytics. |
| **Android** | `onTrimMemory(RUNNING_LOW)` | System is low on memory. Free what you can. |
| **Android** | `onTrimMemory(RUNNING_CRITICAL)` | System is about to kill background apps. |
| **Android** | OOM Kill | Terminated. May show as ANR in Play Console. |

### 3.3 Tracking Memory in React Native

```typescript
// src/performance/memory-tracker.ts

import { NativeModules, Platform } from 'react-native';

interface MemoryInfo {
  usedJSHeap: number;     // MB
  totalJSHeap: number;    // MB
  nativeHeap: number;     // MB (Android only)
  freeMemory: number;     // MB (system-wide)
}

async function getMemoryInfo(): Promise<MemoryInfo> {
  if (Platform.OS === 'android') {
    // Android: Use the Performance API or a native module
    const info = await NativeModules.PerformanceModule.getMemoryInfo();
    return {
      usedJSHeap: info.jsHeapUsed / (1024 * 1024),
      totalJSHeap: info.jsHeapTotal / (1024 * 1024),
      nativeHeap: info.nativeHeap / (1024 * 1024),
      freeMemory: info.freeMemory / (1024 * 1024),
    };
  }
  
  // iOS: Use performance.memory if available (Hermes)
  if (global.performance?.memory) {
    const mem = global.performance.memory;
    return {
      usedJSHeap: mem.usedJSHeapSize / (1024 * 1024),
      totalJSHeap: mem.totalJSHeapSize / (1024 * 1024),
      nativeHeap: 0,  // Not available via JS on iOS
      freeMemory: 0,
    };
  }
  
  return { usedJSHeap: 0, totalJSHeap: 0, nativeHeap: 0, freeMemory: 0 };
}

// Periodic memory monitoring
class MemoryMonitor {
  private intervalId: NodeJS.Timeout | null = null;
  private readonly samples: MemoryInfo[] = [];
  
  start(intervalMs = 10000) {  // Sample every 10 seconds
    this.intervalId = setInterval(async () => {
      const info = await getMemoryInfo();
      this.samples.push(info);
      
      // Alert on high memory usage
      if (info.usedJSHeap > 200) {
        console.warn(`High JS heap usage: ${info.usedJSHeap.toFixed(1)}MB`);
        this.reportHighMemory(info);
      }
      
      // Keep only last 60 samples (10 minutes)
      if (this.samples.length > 60) {
        this.samples.shift();
      }
      
      // Check for memory leaks (steadily increasing over time)
      this.checkForLeaks();
    }, intervalMs);
  }
  
  private checkForLeaks() {
    if (this.samples.length < 12) return;  // Need at least 2 minutes of data
    
    const recent = this.samples.slice(-12);
    const trend = recent[recent.length - 1].usedJSHeap - recent[0].usedJSHeap;
    
    // If memory grew by more than 50MB in 2 minutes, something's leaking
    if (trend > 50) {
      analytics.track('memory_leak_suspected', {
        growth_mb: trend,
        current_mb: recent[recent.length - 1].usedJSHeap,
        screen: getCurrentScreen(),
      });
    }
  }
  
  private reportHighMemory(info: MemoryInfo) {
    analytics.track('high_memory_usage', {
      js_heap_mb: info.usedJSHeap,
      native_heap_mb: info.nativeHeap,
      screen: getCurrentScreen(),
      device: getDeviceModel(),
    });
  }
  
  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }
}
```

### 3.4 Memory Thresholds

| Metric | Good | Watch | Bad | Critical |
|--------|------|-------|-----|----------|
| **JS Heap** | < 100MB | 100-200MB | 200-400MB | > 400MB |
| **Total app memory** | < 200MB | 200-400MB | 400-600MB | > 600MB |
| **Memory growth rate** | Stable | < 10MB/min | 10-50MB/min | > 50MB/min |

### 3.5 Common Memory Leaks in React Native

```typescript
// LEAK 1: Event listeners not cleaned up
function ChatScreen() {
  useEffect(() => {
    const socket = io('wss://api.example.com');
    socket.on('message', handleMessage);
    
    // BAD: No cleanup — socket stays open when screen unmounts
    // GOOD:
    return () => {
      socket.off('message', handleMessage);
      socket.disconnect();
    };
  }, []);
}

// LEAK 2: Reanimated shared values holding references
// Shared values persist on the UI thread even after the component unmounts.
// This is usually fine, but if they hold large objects, it can leak.

// LEAK 3: Image cache growing unbounded
// expo-image handles this well by default, but if you're using
// react-native-fast-image or the built-in Image, monitor disk usage

// LEAK 4: FlatList with getItemLayout returning wrong values
// This causes the list to keep more items in memory than necessary

// LEAK 5: Global state that grows
// Zustand stores, Redux stores, or MobX stores that accumulate data
// without pruning old entries
const useChatStore = create((set) => ({
  messages: [],
  addMessage: (msg) => set((state) => ({ 
    messages: [...state.messages, msg]  // This array grows forever!
  })),
}));

// FIX: Cap the array
const useChatStore = create((set) => ({
  messages: [],
  addMessage: (msg) => set((state) => ({
    messages: [...state.messages, msg].slice(-500),  // Keep only last 500
  })),
}));
```

---

## 4. NETWORK LATENCY METRICS

Your app is only as fast as your API. Network latency directly impacts perceived performance, and mobile networks are unreliable in ways that wifi-connected development machines never show you.

### 4.1 What to Measure

```typescript
// src/performance/network-tracker.ts

interface NetworkMetric {
  url: string;
  method: string;
  statusCode: number;
  requestStart: number;    // When the request was initiated
  dnsLookup: number;       // DNS resolution time (ms)
  tcpConnect: number;      // TCP handshake time (ms)
  tlsHandshake: number;    // TLS negotiation time (ms)
  firstByte: number;       // Time to first byte (TTFB) (ms)
  contentDownload: number; // Time to download response body (ms)
  total: number;           // Total request time (ms)
  responseSize: number;    // Response body size (bytes)
  connectionType: string;  // 'wifi', '4g', '3g', 'offline'
}

// Wrap your fetch/axios to capture metrics
function createTrackedFetch(baseFetch: typeof fetch): typeof fetch {
  return async (input, init) => {
    const url = typeof input === 'string' ? input : input.url;
    const method = init?.method || 'GET';
    const start = performance.now();
    
    try {
      const response = await baseFetch(input, init);
      const end = performance.now();
      
      const metric: Partial<NetworkMetric> = {
        url: sanitizeUrl(url),      // Strip PII from URLs
        method,
        statusCode: response.status,
        total: Math.round(end - start),
        connectionType: getConnectionType(),
      };
      
      reportNetworkMetric(metric);
      return response;
    } catch (error) {
      const end = performance.now();
      
      reportNetworkMetric({
        url: sanitizeUrl(url),
        method,
        statusCode: 0,  // Network error
        total: Math.round(end - start),
        connectionType: getConnectionType(),
      });
      
      throw error;
    }
  };
}

function sanitizeUrl(url: string): string {
  // Remove IDs and tokens from URLs for aggregation
  return url
    .replace(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/g, '/:uuid')
    .replace(/\/\d+/g, '/:id')
    .replace(/token=[^&]+/, 'token=REDACTED');
}
```

### 4.2 Network Latency Thresholds

| Metric | Good | Acceptable | Bad | Alert |
|--------|------|-----------|-----|-------|
| **API response (p50)** | < 200ms | 200-500ms | 500ms-1s | > 1s |
| **API response (p95)** | < 500ms | 500ms-1s | 1-3s | > 3s |
| **API response (p99)** | < 1s | 1-2s | 2-5s | > 5s |
| **Error rate** | < 0.1% | 0.1-1% | 1-5% | > 5% |
| **Timeout rate** | < 0.01% | 0.01-0.1% | 0.1-1% | > 1% |

### 4.3 Connection-Aware Metrics

Mobile networks are wildly variable. An API that responds in 50ms on wifi takes 800ms on 3G. Always segment your metrics by connection type:

```typescript
import NetInfo from '@react-native-community/netinfo';

async function getConnectionType(): Promise<string> {
  const state = await NetInfo.fetch();
  
  if (!state.isConnected) return 'offline';
  
  if (state.type === 'wifi') return 'wifi';
  
  if (state.type === 'cellular') {
    switch (state.details?.cellularGeneration) {
      case '2g': return '2g';
      case '3g': return '3g';
      case '4g': return '4g';
      case '5g': return '5g';
      default: return 'cellular-unknown';
    }
  }
  
  return state.type;
}

// Expected latency by connection type (for your dashboards)
// wifi:  50-200ms
// 5g:    50-100ms
// 4g:    100-400ms
// 3g:    300-1500ms
// 2g:    1000-5000ms
```

---

## 5. APP SIZE AND ITS IMPACT ON DOWNLOADS

App size is the most underestimated metric in mobile development. It directly impacts:

1. **Download conversion** — Google Play shows warnings for large apps on mobile data
2. **Install time** — Larger apps take longer to install, increasing drop-off
3. **Update adoption** — Users on limited data plans defer large updates
4. **Storage pressure** — Users uninstall large apps first when they run out of space
5. **Store placement** — Both stores factor app size into their ranking algorithms

### 5.1 Size Thresholds

| Platform | Size | Impact |
|----------|------|--------|
| **Android** | < 50MB | Optimal. No warnings. Fast installs. |
| **Android** | 50-150MB | Acceptable. Consider using Play App Bundles. |
| **Android** | > 150MB | Play Store shows "large app" warning on mobile data. This kills your conversion rate. |
| **Android** | > 200MB | Must use Play Asset Delivery or expansion files. |
| **iOS** | < 50MB | Can download over cellular without user permission. |
| **iOS** | 50-200MB | iOS prompts user to confirm download on cellular. Many users tap "cancel." |
| **iOS** | > 200MB | Requires WiFi download (this threshold was raised from 150MB in 2019, then 200MB in 2022). Can be changed in iOS Settings. |
| **iOS** | > 4GB | Hard limit from App Store. |

### 5.2 Measuring App Size

Your build system should track app size on every commit. Here's how:

```bash
# Android: Measure the AAB (App Bundle) size
# This is what gets uploaded to Play Store
ls -la android/app/build/outputs/bundle/release/app-release.aab

# Android: See the estimated download size
# bundletool gives you per-device estimates
bundletool build-apks --bundle=app-release.aab --output=app.apks
bundletool get-size total --apks=app.apks
# Output: "Min: 25MB, Max: 45MB" (varies by device)

# iOS: After archiving, check the .ipa size
ls -la build/MyApp.ipa

# iOS: More accurate — use App Store Connect's size report
# After uploading, App Store Connect shows the actual download size
# per device (iPhone, iPad, etc.)
```

```typescript
// CI/CD: Track size over time
// In your CI pipeline (GitHub Actions example):

// .github/workflows/track-size.yml
// - name: Measure Android size
//   run: |
//     SIZE=$(stat -f%z android/app/build/outputs/bundle/release/app-release.aab)
//     SIZE_MB=$(echo "scale=2; $SIZE / 1048576" | bc)
//     echo "Android bundle size: ${SIZE_MB}MB"
//     curl -X POST $METRICS_ENDPOINT -d "{\"metric\":\"app_size_android\",\"value\":$SIZE_MB}"

// - name: Fail if size exceeds limit
//   run: |
//     SIZE=$(stat -f%z android/app/build/outputs/bundle/release/app-release.aab)
//     SIZE_MB=$(echo "scale=2; $SIZE / 1048576" | bc)
//     if (( $(echo "$SIZE_MB > 100" | bc -l) )); then
//       echo "::error::Android bundle size ($SIZE_MB MB) exceeds 100MB limit"
//       exit 1
//     fi
```

### 5.3 What Makes React Native Apps Large

| Component | Typical Size | Notes |
|-----------|-------------|-------|
| **Hermes bytecode** | 5-15MB | Your JS bundle, compiled |
| **Native libraries** | 10-30MB | Per-architecture. Use app bundles! |
| **Assets (images)** | 5-50MB+ | The biggest variable |
| **Fonts** | 1-5MB | Each font file is ~200KB-1MB |
| **Native modules** | 5-20MB | Each native dependency adds weight |
| **React Native core** | ~5MB | The framework itself |

### 5.4 Size Reduction Strategies

```typescript
// 1. Use Hermes (you should already be doing this)
// Hermes compiles JS to bytecode, which is smaller than raw JS
// In app.json:
// "jsEngine": "hermes"

// 2. Enable ProGuard/R8 on Android
// android/app/build.gradle:
// def enableProguardInReleaseBuilds = true

// 3. Use App Bundles (AAB) instead of APKs
// Play Store splits the bundle per device — users only download
// the native libraries for their architecture

// 4. Optimize images
// - Use WebP instead of PNG (40-60% smaller)
// - Remove @3x images if you don't need them
// - Consider loading large images from CDN instead of bundling

// 5. Audit your dependencies
// npx react-native-bundle-visualizer
// This shows you what's in your JS bundle

// 6. Tree shake unused code
// Hermes doesn't tree shake by default.
// Use Metro's tree shaking or re-export only what you need:

// BAD
import { format } from 'date-fns';  // Imports ALL of date-fns

// GOOD
import { format } from 'date-fns/format';  // Imports only format
```

---

## 6. GOOGLE PLAY CONSOLE VITALS

Google Play Console Vitals is the most important dashboard you're not looking at. Google uses these metrics to rank your app in search results and decide whether to show it in recommendations. Bad vitals = less visibility = fewer installs. It's that direct.

### 6.1 The Metrics Google Watches

#### ANR Rate (Application Not Responding)

An ANR occurs when your app's main thread is blocked for more than 5 seconds. On Android, the user sees the "App isn't responding" dialog.

| Metric | Good | Bad Behavior Threshold |
|--------|------|----------------------|
| **User-perceived ANR rate** | < 0.47% | >= 0.47% |
| **ANR rate (all occurrences)** | < 2% | >= 8% |

**What Google does:** If your user-perceived ANR rate exceeds **0.47%**, Google marks your app with a "bad behavior" badge. This reduces your visibility in Play Store search and recommendations.

**Common ANR causes in React Native:**
```typescript
// ANR CAUSE 1: Large synchronous operations on the main thread
// JSON.parse of a 5MB response
const data = JSON.parse(hugeJsonString);  // Blocks main thread

// FIX: Use a worker or break it up
// Or better: paginate your API responses

// ANR CAUSE 2: SQLite operations on the main thread
// Watermelon DB / realm queries that return thousands of records

// ANR CAUSE 3: Network calls in onCreate
// If the Android activity initialization makes a blocking network call

// ANR CAUSE 4: Deadlocks in native modules
// A native module that waits for the JS thread while the JS thread
// waits for the native module
```

#### Crash Rate

| Metric | Good | Bad Behavior Threshold |
|--------|------|----------------------|
| **User-perceived crash rate** | < 1.09% | >= 1.09% |
| **Crash rate (all occurrences)** | < 2% | >= 8% |

**What Google does:** If your user-perceived crash rate exceeds **1.09%**, bad behavior badge. Same visibility penalty.

#### Startup Time

| Metric | Good | Slow |
|--------|------|------|
| **Cold start** | < 3s | > 5s |
| **Warm start** | < 1.5s | > 3s |
| **Hot start** | < 1s | > 1.5s |

Google tracks startup time percentiles (p50, p90, p99) in Play Console under "Android vitals → App startup time."

**What Google does:** Slow startup doesn't get a "bad behavior" badge, but it does affect your Core metrics in the Play Console. And Google has stated that performance metrics factor into Play Store ranking.

#### Permission Denials

| Metric | What It Measures |
|--------|-----------------|
| **Permission denial rate** | How often users deny runtime permissions |

**Why it matters:** A high denial rate (> 30%) for a permission suggests you're requesting it at the wrong time or without adequate explanation. Google doesn't penalize you directly for this, but it affects user trust metrics.

```typescript
// BAD: Requesting camera permission on app launch
// Users don't know why you need it, so they deny
useEffect(() => {
  Permissions.request('camera');  // 60% denial rate
}, []);

// GOOD: Request in context, with explanation
function ScanQRButton() {
  const handlePress = async () => {
    // Show explanation first
    Alert.alert(
      'Camera Access',
      'We need camera access to scan the QR code on your membership card.',
      [
        { text: 'Not Now', style: 'cancel' },
        { 
          text: 'Allow Camera', 
          onPress: () => Permissions.request('camera'),  // 85% acceptance rate
        },
      ]
    );
  };
  
  return <Button onPress={handlePress}>Scan QR Code</Button>;
}
```

#### Excessive Wakeups

| Metric | Good | Bad Behavior Threshold |
|--------|------|----------------------|
| **Stuck partial wakelocks** | < 0.3% of sessions | >= 0.3% |
| **Excessive wakeups** | < 0.1% of sessions | >= 0.1% |

This happens when your app's background processes (push notification handlers, background tasks) wake the device too frequently. It drains battery and Google penalizes it.

### 6.2 Play Console Dashboard Setup

Navigate to **Play Console → Your App → Android vitals → Overview**. You should see:

1. **Core vitals** — ANR rate, crash rate (the ones with bad behavior thresholds)
2. **Other vitals** — Startup time, permission denials, battery usage
3. **Trends** — 28-day rolling averages

**Set up email alerts:**
Play Console → Settings → Notifications → Enable "Performance alerts." You'll get emailed when your app crosses a bad behavior threshold.

### 6.3 Comparing Against Peers

Play Console shows you how your vitals compare to peer apps in the same category. This is gold. If your crash rate is 0.8% and the peer median is 0.5%, you know you're behind. If your startup time p90 is 4s and the peer median is 3s, that's actionable.

Look at: Play Console → Android vitals → Benchmarks.

---

## 7. APP STORE CONNECT METRICS

Apple's equivalent of Play Console Vitals is spread across App Store Connect and Xcode Organizer. It's less centralized than Google's but equally important.

### 7.1 Crashes (App Store Connect)

Navigate to **App Store Connect → Your App → Analytics → Metrics → Crashes**.

Apple reports:
- **Crash count** — absolute number per day
- **Crash rate** — crashes per 1,000 sessions
- **Unique affected devices** — how many devices experienced at least one crash

| Metric | Good | Watch | Bad |
|--------|------|-------|-----|
| **Crashes per 1K sessions** | < 5 | 5-15 | > 15 |
| **Crash-free device rate** | > 99.5% | 99-99.5% | < 99% |

**Apple's enforcement:** Unlike Google's automated "bad behavior" badge, Apple reviews are more manual. But apps with high crash rates get flagged during App Review, and extreme cases can lead to removal.

### 7.2 Disk Writes (Xcode Organizer)

This is one Apple tracks that most developers don't know about. Your app should minimize disk writes because:
- SSDs have limited write cycles
- Excessive writes drain battery
- Apple measures this and shows it in Xcode Organizer

**Xcode → Window → Organizer → Disk Writes**

| Metric | Good | Watch | Bad |
|--------|------|-------|-----|
| **Disk writes (daily)** | < 100MB/day | 100-500MB/day | > 500MB/day |
| **Logical writes per session** | < 50MB | 50-200MB | > 200MB |

Common culprits in React Native:
- Excessive AsyncStorage writes (use MMKV instead)
- Uncompressed log files
- Image caching without size limits
- SQLite WAL files growing without checkpointing

### 7.3 Memory (Xcode Organizer)

**Xcode → Window → Organizer → Memory**

Apple shows your app's memory footprint and compares it to the device's capabilities. They also track **memory pressure terminations** (jetsam kills) — when the OS kills your app because it's using too much memory.

| Metric | Good | Watch | Bad |
|--------|------|-------|-----|
| **Peak memory** | < 200MB | 200-400MB | > 400MB |
| **Memory at suspend** | < 100MB | 100-200MB | > 200MB |
| **Jetsam terminations** | 0 | < 0.1% of sessions | > 0.1% |

### 7.4 Launch Time (Xcode Organizer)

**Xcode → Window → Organizer → Launch Time**

Apple tracks cold launch time and breaks it down by device type. They show you:
- **Typical** (p50) — What most users experience
- **Slow** (p90) — What 10% of users experience

| Metric | Good | Watch | Bad |
|--------|------|-------|-----|
| **Cold launch (p50)** | < 1s | 1-2s | > 2s |
| **Cold launch (p90)** | < 2s | 2-3s | > 3s |

### 7.5 Hang Rate (iOS 16+)

A "hang" is when the main thread is blocked for more than 250ms. This is more granular than Android's 5-second ANR threshold.

**Xcode → Window → Organizer → Hangs**

| Metric | Good | Watch | Bad |
|--------|------|-------|-----|
| **Hang rate** | < 1 per hour | 1-5 per hour | > 5 per hour |
| **Critical hangs (> 2s)** | 0 per session | 0-1 per session | > 1 per session |

### 7.6 Battery Usage (Xcode Organizer)

**Xcode → Window → Organizer → Battery Usage**

Apple tracks:
- **CPU time** — how much CPU your app uses in foreground and background
- **Location usage** — are you using GPS when you don't need to?
- **Networking** — how much data are you transferring?
- **Display** — are you preventing the screen from sleeping?

There are no hard thresholds, but Apple compares your app to peer apps. If you're significantly worse, they'll flag it.

---

## 8. DATADOG RUM FOR MOBILE

While Play Console and App Store Connect give you platform-specific metrics, Datadog Real User Monitoring (RUM) gives you a unified, cross-platform view of your app's performance. It's the tool that ties everything together.

### 8.1 Setting Up Datadog RUM for React Native

```bash
npm install @datadog/mobile-react-native
```

```typescript
// src/monitoring/datadog.ts

import {
  DdSdkReactNative,
  DdSdkReactNativeConfiguration,
  SdkVerbosity,
  TrackingConsent,
} from '@datadog/mobile-react-native';

const config = new DdSdkReactNativeConfiguration(
  'YOUR_CLIENT_TOKEN',                           // Client token (not API key!)
  'production',                                    // Environment
  'app-id-from-datadog',                          // RUM Application ID
  true,                                            // Track interactions
  true,                                            // Track XHR resources
  true,                                            // Track errors
);

config.site = 'US1';                              // Or 'EU1', 'US3', etc.
config.nativeCrashReportEnabled = true;
config.sampleRate = 100;                          // 100% of sessions in production
config.resourceTracingSamplingRate = 20;          // Sample 20% for APM traces
config.verbosity = SdkVerbosity.WARN;
config.trackingConsent = TrackingConsent.GRANTED;  // Respect user consent!

export async function initDatadog() {
  await DdSdkReactNative.initialize(config);
  
  // Set user info for session tracking
  DdSdkReactNative.setUser({
    id: getUserId(),
    name: getUserName(),
    email: getUserEmail(),
    // Custom attributes
    subscription_tier: getSubscriptionTier(),
    app_version: getAppVersion(),
  });
  
  // Set global attributes that appear on every event
  DdSdkReactNative.setAttributes({
    app_version: getAppVersion(),
    build_number: getBuildNumber(),
    js_engine: global.HermesInternal ? 'hermes' : 'jsc',
    device_tier: getDeviceTier(),
  });
}
```

### 8.2 Custom Views (Screen Tracking)

```tsx
import { DdRum } from '@datadog/mobile-react-native';

// Track screen views
function useScreenTracking(screenName: string) {
  useEffect(() => {
    DdRum.startView(
      screenName,                    // View key (unique identifier)
      screenName,                    // View name (human readable)
      {
        screen_type: 'feature',      // Custom attributes
      }
    );
    
    return () => {
      DdRum.stopView(screenName);
    };
  }, [screenName]);
}

// Usage
function ProductDetailScreen({ product }: ProductDetailProps) {
  useScreenTracking(`ProductDetail-${product.id}`);
  // ...
}
```

### 8.3 Custom Actions and Timing

```typescript
import { DdRum, RumActionType } from '@datadog/mobile-react-native';

// Track user actions
function handleAddToCart(product: Product) {
  DdRum.addAction(
    RumActionType.TAP,
    'Add to Cart',
    {
      product_id: product.id,
      product_price: product.price,
      product_category: product.category,
    }
  );
  
  // ... actual logic
}

// Track custom timing
async function loadProductCatalog() {
  const startTime = Date.now();
  
  try {
    const products = await api.getProducts();
    const duration = Date.now() - startTime;
    
    DdRum.addTiming('product_catalog_load');
    
    // Or use a custom action with timing
    DdRum.addAction(
      RumActionType.CUSTOM,
      'catalog_loaded',
      {
        duration_ms: duration,
        product_count: products.length,
        connection_type: getConnectionType(),
      }
    );
    
    return products;
  } catch (error) {
    DdRum.addError(
      error.message,
      'source',
      error.stack || '',
      {
        duration_ms: Date.now() - startTime,
        connection_type: getConnectionType(),
      }
    );
    throw error;
  }
}
```

### 8.4 Resource Tracking (API Calls)

Datadog automatically tracks fetch/XHR calls as "resources." But you can add custom attributes:

```typescript
import { DdRum } from '@datadog/mobile-react-native';

// The SDK auto-instruments fetch, but you can add more context:
DdRum.addTiming('first_api_call');

// For custom resource tracking (e.g., WebSocket messages):
DdRum.startResource(
  'ws-message-123',           // Resource key
  'GET',                       // Method
  'wss://api.example.com/ws', // URL
  {
    message_type: 'chat',
  }
);

// ... when complete
DdRum.stopResource(
  'ws-message-123',
  200,                         // Status code
  'xhr',                       // Resource type
  1024,                        // Response size in bytes
  {
    message_count: 5,
  }
);
```

### 8.5 Error Tracking

```typescript
import { DdRum, ErrorSource } from '@datadog/mobile-react-native';

// Manual error tracking
try {
  await riskyOperation();
} catch (error) {
  DdRum.addError(
    error.message,
    ErrorSource.SOURCE,        // 'source', 'network', 'console', 'custom'
    error.stack || '',
    {
      operation: 'risky_operation',
      user_action: 'submit_form',
      screen: getCurrentScreen(),
    }
  );
}

// Global error handler
ErrorUtils.setGlobalHandler((error, isFatal) => {
  DdRum.addError(
    error.message,
    ErrorSource.SOURCE,
    error.stack || '',
    {
      is_fatal: isFatal,
      screen: getCurrentScreen(),
    }
  );
});

// Promise rejection handler
if (global.HermesInternal) {
  global.HermesInternal.enablePromiseRejectionTracker?.({
    allRejections: true,
    onUnhandled: (id: number, rejection: any) => {
      DdRum.addError(
        `Unhandled Promise Rejection: ${rejection}`,
        ErrorSource.SOURCE,
        rejection?.stack || '',
        { promise_id: id }
      );
    },
  });
}
```

---

## 9. BUILDING DASHBOARDS THAT PEOPLE ACTUALLY LOOK AT

Here's a hard truth: most dashboards are never looked at after the first week. The team builds them with enthusiasm, walks away, and never comes back until something breaks. That defeats the purpose.

The trick to dashboards that get used is **putting the right information in the right place at the right time.**

### 9.1 The Three Dashboard Tiers

**Tier 1: The TV Dashboard (Office TV / Team Slack Channel)**

This is what goes on the big screen in the office or gets posted to Slack every morning. It should be glanceable in 5 seconds.

What goes on it:
```
+-----------------------------------------------------+
|  APP HEALTH DASHBOARD                               |
|                                                     |
|  Crash-Free Rate      Cold Start (p50)   Active     |
|  ======== 99.7%       ==== 1.2s          Users      |
|  ^ 0.1% vs yesterday  ^ 0.1s vs last wk  12,345    |
|                                                     |
|  ANR Rate             API Latency (p50)  Error Rate |
|  ======== 0.12%       ==== 180ms         0.3%       |
|  * Below threshold    * Healthy          * Healthy  |
|                                                     |
|  --- 24h Trend ------------------------------------ |
|  [sparkline graph of crash-free rate]               |
|  [sparkline graph of cold start time]               |
|                                                     |
|  Recent Deploys                                     |
|  - v2.4.1 (3h ago) * Healthy                       |
|  - v2.4.0 (2d ago) * Healthy                       |
+-----------------------------------------------------+
```

**Rules for the TV dashboard:**
- Maximum 6-8 metrics
- Color coding: green/yellow/red
- Trend arrows: up/down vs yesterday or last week
- Recent deploys visible (so you can correlate deployments with metric changes)
- NO raw numbers without context (is 180ms good or bad? The color tells you)

**Tier 2: The Investigation Dashboard**

This is what you open when the TV dashboard turns yellow or red. It has more detail.

What goes on it:
```
Breakdown by:
  - Platform (iOS / Android)
  - App version
  - Device tier (low / mid / high)
  - Connection type (wifi / 4g / 3g)
  - Country
  - Screen / feature

Time series graphs:
  - Cold start time by percentile (p50, p90, p99)
  - FPS distribution during scroll
  - Memory usage over session lifetime
  - API latency by endpoint
  - Error rate by error type

Heatmaps:
  - Crash rate by device model
  - ANR rate by Android API level
  - Startup time by app version
```

**Tier 3: The Deep Dive Dashboard**

This is for the engineer who's actively debugging an issue. It has everything:

```
Individual session replays (Datadog Session Replay)
Full request waterfall for a specific session
Flame charts for JS thread performance
Memory allocation timeline
Specific crash stack traces with symbolication
Native crash reports (tombstones / crash logs)
```

### 9.2 Building the TV Dashboard in Datadog

```
# Datadog Dashboard JSON (simplified)
# Import this as a new dashboard

Widgets:
1. Query Value: "Crash-Free Rate"
   query: 100 - (count(rum.error{type:crash}) / count(rum.session{}) * 100)
   thresholds: green < 0.5%, yellow 0.5-1%, red > 1%

2. Query Value: "Cold Start p50"
   query: p50:rum.view.loading_time{view.name:AppRoot}
   thresholds: green < 1500ms, yellow 1500-3000ms, red > 3000ms

3. Query Value: "API Latency p50"
   query: p50:rum.resource.duration{resource.type:xhr}
   thresholds: green < 300ms, yellow 300-500ms, red > 500ms

4. Query Value: "ANR Rate"
   query: count(rum.long_task{duration:>5000}) / count(rum.session{})
   thresholds: green < 0.47%, yellow 0.47-1%, red > 1%

5. Timeseries: "Crash-Free Rate (7 days)"
   query: crash_free_rate by {version}

6. Timeseries: "Cold Start by Version"
   query: p50:rum.view.loading_time{view.name:AppRoot} by {version}
```

### 9.3 Daily Metrics Slack Report

Automate a daily report to your team's Slack channel:

```typescript
// functions/daily-metrics-report.ts (runs on a cron job)

interface DailyReport {
  crashFreeRate: number;
  coldStartP50: number;
  coldStartP90: number;
  apiLatencyP50: number;
  errorRate: number;
  activeUsers: number;
  anrRate: number;
  appSizeAndroid: number;
  appSizeIos: number;
}

async function generateDailyReport(): Promise<DailyReport> {
  // Pull from Datadog API, Play Console API, App Store Connect API
  const [ddMetrics, playMetrics, appStoreMetrics] = await Promise.all([
    fetchDatadogMetrics(),
    fetchPlayConsoleMetrics(),
    fetchAppStoreMetrics(),
  ]);
  
  return {
    crashFreeRate: ddMetrics.crashFreeRate,
    coldStartP50: ddMetrics.coldStartP50,
    coldStartP90: ddMetrics.coldStartP90,
    apiLatencyP50: ddMetrics.apiLatencyP50,
    errorRate: ddMetrics.errorRate,
    activeUsers: ddMetrics.activeUsers,
    anrRate: playMetrics.anrRate,
    appSizeAndroid: playMetrics.downloadSize,
    appSizeIos: appStoreMetrics.downloadSize,
  };
}

function formatSlackMessage(report: DailyReport): string {
  const emoji = (value: number, good: number, bad: number) => {
    if (value <= good) return ':large_green_circle:';
    if (value <= bad) return ':large_yellow_circle:';
    return ':red_circle:';
  };
  
  return `
*Daily App Health Report* — ${new Date().toLocaleDateString()}

${emoji(100 - report.crashFreeRate, 0.5, 1)} *Crash-Free Rate:* ${report.crashFreeRate.toFixed(2)}%
${emoji(report.coldStartP50, 1500, 3000)} *Cold Start (p50):* ${report.coldStartP50}ms
${emoji(report.coldStartP90, 3000, 5000)} *Cold Start (p90):* ${report.coldStartP90}ms
${emoji(report.apiLatencyP50, 300, 500)} *API Latency (p50):* ${report.apiLatencyP50}ms
${emoji(report.errorRate, 0.5, 1)} *Error Rate:* ${report.errorRate.toFixed(2)}%
${emoji(report.anrRate, 0.47, 1)} *ANR Rate:* ${report.anrRate.toFixed(2)}%

:bust_in_silhouette: *Active Users:* ${report.activeUsers.toLocaleString()}
:package: *App Size:* Android ${report.appSizeAndroid}MB / iOS ${report.appSizeIos}MB

_Dashboard: <https://app.datadoghq.com/dashboard/your-dashboard|View Full Dashboard>_
  `.trim();
}
```

---

## 10. ALERT THRESHOLDS AND ON-CALL

Alerts are the automated safety net between "something went wrong" and "a human investigates." Get them right and you catch issues in minutes. Get them wrong and you either miss real problems (too few alerts) or drown in noise (too many alerts).

### 10.1 The Alert Hierarchy

```
SEVERITY 1 (P0) — Page on-call immediately, wake people up
  - Crash-free rate drops below 98% (was 99.5%+)
  - ANR rate exceeds 2%
  - API error rate exceeds 10%
  - App completely unresponsive (all health checks failing)

SEVERITY 2 (P1) — Notify in Slack, investigate within 1 hour
  - Crash-free rate drops below 99%
  - ANR rate exceeds 0.47% (Play Console bad behavior threshold)
  - Cold start p90 exceeds 5 seconds
  - API latency p95 exceeds 3 seconds
  - Memory warnings spike above 5% of sessions

SEVERITY 3 (P2) — Notify in Slack, investigate within 1 business day
  - Crash-free rate drops below 99.5%
  - Cold start p50 exceeds 2 seconds
  - API latency p50 exceeds 500ms
  - JS error rate exceeds 1%
  - App size exceeds your target by 20%+

SEVERITY 4 (P3) — Track in dashboard, address in sprint planning
  - Cold start trending up over 7 days
  - Memory usage trending up (potential slow leak)
  - API latency trending up
  - Bundle size growing faster than features justify
```

### 10.2 Setting Up Alerts in Datadog

```
# Datadog Monitor: Crash-Free Rate P0
# Type: Metric Monitor
# Query: 100 - (count(rum.error{type:crash}).as_count() / count(rum.session{}).as_count() * 100)
# Evaluation window: Last 15 minutes
# Alert threshold: < 98
# Warning threshold: < 99
# Notify: @pagerduty-mobile-oncall @slack-mobile-alerts
# Message: |
#   ## P0: Crash-Free Rate Critical
#   
#   Current crash-free rate: {{value}}%
#   Threshold: 98%
#   
#   **Immediate actions:**
#   1. Check recent deployments: {{dashboard_link}}
#   2. Check Sentry for new crash groups
#   3. If caused by new version, prepare hotfix or rollback
#   
#   **Recent deploys:** Check the releases dashboard
```

```
# Datadog Monitor: ANR Rate P1
# Type: Metric Monitor
# Query: count(rum.long_task{duration:>5000}).as_count() / count(rum.session{}).as_count() * 100
# Evaluation window: Last 1 hour
# Alert threshold: > 0.47
# Warning threshold: > 0.3
# Notify: @slack-mobile-alerts
# Message: |
#   ## P1: ANR Rate Exceeds Play Console Threshold
#   
#   Current ANR rate: {{value}}%
#   Play Console bad behavior threshold: 0.47%
#   
#   This affects Play Store visibility. Investigate within 1 hour.
#   
#   Common causes:
#   - Heavy JS thread work during app startup
#   - Synchronous storage reads
#   - Large JSON parsing on main thread
```

### 10.3 Alert Anti-Patterns

**Anti-pattern 1: Alerting on absolute counts instead of rates.**

```
# BAD: "Alert when errors > 100"
# Why it's bad: 100 errors when you have 10 users = catastrophe
#               100 errors when you have 1M users = statistical noise

# GOOD: "Alert when error RATE > 1%"
```

**Anti-pattern 2: No recovery notification.**

```
# BAD: Only sending alerts when things break
# The on-call person doesn't know when it resolves

# GOOD: Configure "recovery" notifications
# Datadog → Monitor → "Notify on recovery" → enabled
# This sends a green "resolved" message when the metric returns to normal
```

**Anti-pattern 3: Alerting on every individual error.**

```
# BAD: An alert for every crash
# At 10,000 DAU with 0.5% crash rate, that's 50 alerts per day

# GOOD: Alert on rate changes and new crash groups
# - Alert when crash-free rate drops
# - Alert when a NEW crash signature appears (via Sentry)
# - Don't alert on every individual crash
```

**Anti-pattern 4: Not including context in alert messages.**

```
# BAD:
# "Alert: Crash rate is high"

# GOOD:
# "Alert: Crash rate 2.3% (threshold: 1.09%)
#  - Platform: Android
#  - Most affected version: 2.4.1 (released 3 hours ago)
#  - Top crash: NullPointerException in PaymentModule.java:142
#  - Affected users: ~450
#  - Dashboard: [link]
#  - Sentry issue: [link]"
```

### 10.4 On-Call Rotation

If your team is big enough (4+ mobile engineers), set up a rotation:

```
On-Call Structure:
- Primary on-call: 1 week rotation
- Secondary (backup): Next person in rotation
- Escalation: If primary doesn't acknowledge in 15 min, page secondary
- Business hours only: For P2-P3 alerts
- 24/7: For P0-P1 alerts only

What on-call does:
1. Acknowledge alerts within 15 minutes
2. Triage: Is this a real problem or a false alarm?
3. If real: Start investigating, update the team
4. If deploy-related: Decide whether to rollback or hotfix
5. Document the incident (even small ones)

What on-call does NOT do:
- Fix every bug during their shift
- Guarantee zero alerts
- Stay awake all night for P3 issues
```

### 10.5 The Runbook

Every alert should link to a runbook. A runbook is a document that tells the on-call person exactly what to do when an alert fires.

```markdown
# Runbook: Crash-Free Rate Drop

## Severity
P0 if < 98%, P1 if < 99%, P2 if < 99.5%

## First Response (< 5 minutes)
1. Open Datadog dashboard: [link]
2. Check if the drop correlates with a recent deploy
   - Go to Releases: [link]
   - If yes: prepare to rollback (see Rollback section below)
3. Open Sentry: [link]
   - Sort by "first seen" to find new crash groups
   - Check the top crash group for affected count and stack trace

## Investigation (5-30 minutes)
1. Identify the crash signature
2. Check affected platforms (iOS only? Android only? Both?)
3. Check affected versions
4. Check affected device models (hardware-specific?)
5. Check affected OS versions

## Rollback Decision
Rollback if:
- Crash-free rate < 97%
- The crash is in a critical path (launch, payment, auth)
- No fix is available within 2 hours

## Rollback Steps
### Android (Play Console)
1. Go to Play Console -> Release management -> App releases
2. Click "Manage" on the production track
3. Halt the rollout of the current release
4. Start a new release with the previous version

### iOS (App Store Connect)
1. Go to App Store Connect -> Your App
2. Remove the current version from sale (if needed)
3. Use EAS to build the previous version
4. Submit for expedited review

## Communication
1. Post in #mobile-incidents: "Investigating crash-free rate drop"
2. Update every 30 minutes until resolved
3. Post-incident: Write a brief post-mortem
```

---

## 11. THE METRICS LIFECYCLE

Here's how these metrics should flow through your development process:

### 11.1 Before You Ship

```
Pre-release checklist:
[ ] Cold start time measured on test devices (low/mid/high tier)
[ ] FPS profiled on lowest-tier supported device
[ ] Memory profiled with 10-minute usage session
[ ] Bundle size compared to previous release
[ ] No new ANR triggers in automated tests
[ ] API latency tested on throttled network (3G simulation)
```

### 11.2 During Rollout

```
Staged rollout monitoring (Android):
  1% rollout -> Wait 24 hours -> Check vitals
  5% rollout -> Wait 24 hours -> Check vitals
  20% rollout -> Wait 24 hours -> Check vitals
  50% rollout -> Wait 24 hours -> Check vitals
  100% rollout

At each stage, check:
[ ] Crash-free rate is within 0.1% of previous version
[ ] ANR rate hasn't increased
[ ] Cold start time hasn't regressed
[ ] No new crash signatures in Sentry
[ ] API error rate stable
```

### 11.3 After Ship (Ongoing)

```
Weekly review (in sprint retrospective):
- Review TV dashboard trends
- Compare current metrics to 4-week average
- Identify any slow degradation (boiling frog problems)
- Review alert fatigue — are we getting too many false alarms?
- Review on-call incidents from the week

Monthly review:
- Compare Play Console / App Store Connect benchmarks to peers
- Review app size growth trend
- Review cold start trend across app versions
- Plan performance improvement work for next sprint
```

---

## 12. QUICK REFERENCE: ALL THRESHOLDS IN ONE PLACE

For when you're setting up alerts and need the numbers fast:

### Cold Start / TTI
| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Cold start (p50) | < 1s | 1-2s | > 2s |
| Cold start (p90) | < 2s | 2-3s | > 3s |
| Cold start (p99) | < 3s | 3-5s | > 5s |

### FPS / Jank
| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| Scroll FPS (p10) | > 55 | 45-55 | < 45 |
| Animation FPS (p10) | > 58 | 50-58 | < 50 |
| Dropped frames/min | < 5 | 5-15 | > 15 |

### Memory
| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| JS heap | < 100MB | 100-200MB | > 200MB |
| Total app memory | < 200MB | 200-400MB | > 400MB |
| Memory growth | Stable | < 10MB/min | > 10MB/min |

### Network
| Metric | Green | Yellow | Red |
|--------|-------|--------|-----|
| API latency (p50) | < 200ms | 200-500ms | > 500ms |
| API latency (p95) | < 500ms | 500ms-1s | > 1s |
| Error rate | < 0.1% | 0.1-1% | > 1% |

### App Size
| Platform | Green | Yellow | Red |
|----------|-------|--------|-----|
| Android (AAB) | < 50MB | 50-100MB | > 150MB |
| iOS (IPA) | < 50MB | 50-150MB | > 200MB |

### Store Vitals
| Metric | Green | Bad Behavior (Google) |
|--------|-------|-----------------------|
| Crash rate (user-perceived) | < 0.5% | >= 1.09% |
| ANR rate (user-perceived) | < 0.2% | >= 0.47% |

### Alert Severity
| Severity | Response Time | Example |
|----------|--------------|---------|
| P0 | Immediate (15 min) | Crash-free < 98% |
| P1 | 1 hour | ANR > 0.47%, cold start p90 > 5s |
| P2 | 1 business day | Crash-free < 99.5%, latency p50 > 500ms |
| P3 | Next sprint | Trending metrics |

---

## Key Takeaways

1. **Metrics are not optional.** If you're not measuring cold start, FPS, memory, network latency, and app size, you're flying blind. Set up tracking before your first production deploy, not after.

2. **Know the thresholds that matter to the stores.** Google Play's 0.47% ANR rate and 1.09% crash rate thresholds directly affect your app's visibility. Apple's Xcode Organizer metrics affect your App Review relationship. These aren't vanity metrics — they're business metrics.

3. **Build three tiers of dashboards.** A TV dashboard for daily glancing (6-8 metrics, color-coded). An investigation dashboard for when things go wrong. A deep-dive dashboard for active debugging.

4. **Alerts should have context and runbooks.** An alert that says "something is broken" wastes the on-call person's time. An alert that says "crash rate spiked to 2.3% after v2.4.1 deployed 3 hours ago, top crash is NullPointerException in PaymentModule.java:142, here's the Sentry link and the rollback steps" saves hours.

5. **Segment everything.** A p50 cold start of 1.2s that averages across all devices hides the fact that it's 600ms on iPhone 15 Pro and 3.8s on a 4-year-old Samsung Galaxy A10. Always segment by platform, device tier, connection type, and app version.

6. **Watch for boiling frogs.** A cold start that increases by 50ms per release doesn't trigger alerts. But after 10 releases, it's 500ms slower. Use weekly and monthly reviews to catch slow degradation.

7. **The daily Slack report is the most important automation you'll build.** A human won't check a dashboard every day. But they'll glance at a Slack message. Make the data go where the people are.

---

**Next up:** [Chapter 24: Incident Response] — what to do when the alerts fire and the metrics are red. Because preparation is the difference between a 30-minute incident and a 3-day crisis.
