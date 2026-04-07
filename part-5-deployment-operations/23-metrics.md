<!--
  CHAPTER: 23
  TITLE: Mobile Metrics That Matter
  PART: V — Deployment & Operations
  PREREQS: Chapter 20
  KEY_TOPICS: TTI, cold start, FPS, jank measurement, memory usage, network latency, app size, Play Console vitals, ANR rate, crash rate, App Store Connect, Datadog RUM, dashboards, alert thresholds
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 23: Mobile Metrics That Matter

> **Part V — Deployment & Operations** | Prerequisites: Chapter 20 | Difficulty: Intermediate

Here's a scenario that plays out at every mobile-focused company: the team ships a feature, the product manager says "great, it looks good," and everyone moves on. Two weeks later, the app's rating drops from 4.5 to 4.2 stars. One-star reviews mention "the app is getting slower" and "it crashes when I open the camera." The team is confused — nothing changed, right? Nothing changed except 200ms of added cold start time from the new feature's initialization, a memory leak in the camera module that only triggers after 10 photos, and an ANR spike on Samsung Galaxy A-series phones because of a synchronous disk write on the main thread.

**You cannot fix what you cannot measure.** And in mobile development, the gap between what you *think* your app is doing and what it's *actually* doing on real devices, on real networks, with real battery levels, is enormous.

This chapter is about closing that gap. We're going to cover every metric that matters for a production mobile app, where to find those metrics, how to set up dashboards that give you visibility, and what thresholds should trigger alerts. By the end, you'll have a monitoring setup that tells you about problems before your users do.

### In This Chapter
- The Metrics Mental Model — what to measure and why
- Cold Start Time and TTI (Time to Interactive)
- FPS and Jank Measurement
- Memory Usage Tracking
- Network Latency and Payload Size
- App Size Optimization
- Google Play Console Vitals
- App Store Connect Metrics
- Datadog RUM for Mobile
- Setting Up Dashboards
- Alert Thresholds and On-Call Practices

### Related Chapters
- [Ch 13: Performance Optimization] — fixing the problems these metrics reveal
- [Ch 14: Profiling & Debugging] — deep-diving into specific performance issues
- [Ch 20: CI/CD for Mobile] — automating metric collection in your pipeline
- [Ch 21: Error Tracking & Monitoring] — crash and error tracking

---

## 1. THE METRICS MENTAL MODEL

Not all metrics are equal. Some tell you about the user's experience directly (how fast the app feels). Others tell you about the health of the system (how much memory is being used). Both matter, but for different reasons.

```
┌─────────────────────────────────────────────────────────────┐
│  USER-FACING METRICS (UX)                                    │
│  These directly affect how the user perceives your app       │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Cold Start / TTI  — "How long until I can use it?"     ││
│  │  FPS / Jank        — "Does it feel smooth?"             ││
│  │  Network Latency   — "Does data load fast?"             ││
│  │  Crash Rate        — "Does it crash?"                   ││
│  │  ANR Rate          — "Does it freeze?"                  ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  SYSTEM HEALTH METRICS                                       │
│  These predict future problems before users feel them        │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Memory Usage      — Trending up? Leak incoming.        ││
│  │  App Size          — Growing? Users stop updating.      ││
│  │  Battery Impact    — High? Users uninstall.             ││
│  │  Network Usage     — Excessive? Users on metered plans. ││
│  │  JS Bundle Size    — Growing? Cold start slowing.       ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  BUSINESS METRICS                                            │
│  These connect technical performance to business outcomes    │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  Session Duration   — Are users sticking around?        ││
│  │  Funnel Conversion  — Are drops correlated with perf?   ││
│  │  App Store Rating   — Is performance affecting reviews? ││
│  │  Uninstall Rate     — Post-update uninstalls?           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### The Golden Rule of Mobile Metrics

**Segment everything by device class.** A p50 cold start of 1.2 seconds means nothing if your p90 on low-end Android is 4.5 seconds. Your users don't experience averages — they experience their specific device on their specific network.

Segment by:
- **Device tier**: High-end (iPhone 15, Pixel 8), Mid-range (Galaxy A54, Pixel 7a), Low-end (Galaxy A13, Redmi devices)
- **OS version**: iOS 17 vs 16 vs 15, Android 14 vs 13 vs 12
- **Network type**: WiFi, 5G, LTE, 3G
- **Geography**: Network conditions vary dramatically by region
- **App version**: Compare versions to detect regressions

---

## 2. COLD START TIME AND TTI

Cold start is the single most impactful performance metric for mobile apps. It's the first impression every time. Studies consistently show that every 100ms of startup time costs measurable engagement.

### What Actually Happens During Cold Start

```
┌──────────────────────────────────────────────────────────────┐
│  COLD START TIMELINE (React Native, New Architecture)         │
│                                                               │
│  0ms    Process created by OS                                │
│         ├── Native libraries loaded (libhermes, libjsi, etc) │
│         ├── Application class initialized                    │
│         └── Native modules registered                        │
│                                                               │
│  ~100ms Hermes engine initialized                            │
│         ├── Bytecode bundle loaded (mmap from disk)          │
│         └── Global scope executed                            │
│                                                               │
│  ~200ms React tree begins rendering                          │
│         ├── Root component mounted                           │
│         ├── Navigation initialized                           │
│         └── Initial data fetched (or loaded from cache)      │
│                                                               │
│  ~400ms First meaningful paint                               │
│         ├── Initial screen visible                           │
│         └── Images begin loading                             │
│                                                               │
│  ~600ms Time to Interactive (TTI)                            │
│         ├── Touch handlers registered                        │
│         ├── Navigation functional                            │
│         └── User can interact                                │
│                                                               │
│  Target: TTI < 1000ms on mid-range devices                   │
│  Good:   TTI < 500ms on high-end devices                     │
│  Bad:    TTI > 2000ms on any device                          │
└──────────────────────────────────────────────────────────────┘
```

### Measuring Cold Start Time

```typescript
// lib/startup-metrics.ts
import { PerformanceObserver } from 'react-native-performance';

// Mark the earliest point you control
// This goes in your entry file (index.js / App.tsx)
const APP_START_TIME = global.performance?.now?.() ?? Date.now();

export function measureStartupTime() {
  // Native module launch time (if you have a custom native module)
  const nativeLaunchTime = NativeModules.StartupModule?.getLaunchTimestamp?.();

  return {
    // Time from native app launch to JS execution start
    nativeToJs: APP_START_TIME - (nativeLaunchTime ?? APP_START_TIME),
    // Will be populated when we mark interactive
    jsToInteractive: 0,
    total: 0,
  };
}

// Call this when your main screen is interactive
export function markInteractive() {
  const now = global.performance?.now?.() ?? Date.now();
  const tti = now - APP_START_TIME;

  analytics.track('app_tti', {
    ttiMs: tti,
    deviceModel: DeviceInfo.getModel(),
    osVersion: DeviceInfo.getSystemVersion(),
    appVersion: DeviceInfo.getVersion(),
    isLowEndDevice: isLowEndDevice(),
  });

  console.log(`TTI: ${tti.toFixed(0)}ms`);
}

// In your main screen component:
function HomeScreen() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    if (isReady) {
      markInteractive();
    }
  }, [isReady]);

  // Mark as ready when critical content is loaded
  const { data } = useQuery({
    queryKey: ['home-data'],
    queryFn: fetchHomeData,
    onSuccess: () => setIsReady(true),
  });

  // ...
}
```

### What Affects Cold Start (and How to Fix It)

```
┌─────────────────────────────────────────────────────────────┐
│  COLD START FACTORS                                          │
│                                                              │
│  JS BUNDLE SIZE                                              │
│  Impact: +1MB ≈ +100-200ms on mid-range devices             │
│  Fix: Code splitting, lazy loading, tree shaking             │
│  Measure: npx react-native-bundle-visualizer                │
│                                                              │
│  NATIVE MODULE INITIALIZATION                                │
│  Impact: Each eager module adds 10-50ms                      │
│  Fix: Lazy-load native modules (TurboModules)               │
│  Measure: Trace NativeModule.init in Android Studio          │
│                                                              │
│  HERMES BYTECODE COMPILATION                                 │
│  Impact: Pre-compiled bytecode saves 200-500ms              │
│  Fix: Ensure Hermes is enabled + bytecode is precompiled    │
│  Measure: Check for .hbc files in your bundle               │
│                                                              │
│  STATE REHYDRATION                                           │
│  Impact: Loading persisted state from disk                   │
│  Fix: Use MMKV (sync) instead of AsyncStorage (async)       │
│  Measure: Time storage.get() calls on startup               │
│                                                              │
│  INITIAL DATA FETCH                                          │
│  Impact: Network request before first render                 │
│  Fix: Persist TanStack Query cache, show cached data first  │
│  Measure: Time from render to data available                 │
│                                                              │
│  SPLASH SCREEN DURATION                                      │
│  Impact: Perceived vs actual start time                      │
│  Fix: Show splash until TTI, not until JS loads             │
│  Measure: Time from splash hide to interactive              │
└─────────────────────────────────────────────────────────────┘
```

### Tracking Cold Start Trends Over Time

```typescript
// This runs in your CI pipeline for every build
// Using a performance testing device farm (Firebase Test Lab, AWS Device Farm)

interface StartupBenchmark {
  version: string;
  buildNumber: number;
  device: string;
  coldStartMs: number;
  ttiMs: number;
  jsBundleSizeBytes: number;
  timestamp: string;
}

// Store benchmarks and alert on regressions
async function reportStartupBenchmark(benchmark: StartupBenchmark) {
  // Compare with the previous version
  const previousBenchmark = await getLastBenchmark(benchmark.device);

  if (previousBenchmark) {
    const regression = benchmark.ttiMs - previousBenchmark.ttiMs;
    const regressionPercent = (regression / previousBenchmark.ttiMs) * 100;

    if (regressionPercent > 10) {
      await sendAlert({
        severity: 'warning',
        message: `TTI regression of ${regression.toFixed(0)}ms (${regressionPercent.toFixed(1)}%) on ${benchmark.device}`,
        build: benchmark.buildNumber,
      });
    }

    if (regressionPercent > 25) {
      await sendAlert({
        severity: 'critical',
        message: `Major TTI regression: ${regression.toFixed(0)}ms on ${benchmark.device}. Consider blocking release.`,
        build: benchmark.buildNumber,
      });
    }
  }

  await saveBenchmark(benchmark);
}
```

---

## 3. FPS AND JANK MEASUREMENT

Frame rate is what makes apps feel "smooth" or "janky." On most devices, the target is 60 FPS (16.6ms per frame). On ProMotion iPhones and high-refresh Android displays, the target is 120 FPS (8.3ms per frame).

### What Causes Jank

```
┌──────────────────────────────────────────────────────────┐
│  A FRAME AT 60 FPS                                        │
│                                                           │
│  You have 16.6ms to:                                      │
│  1. Run JavaScript (event handlers, state updates)        │
│  2. Run React reconciliation (diffing)                    │
│  3. Commit layout changes (Yoga layout calculation)       │
│  4. Paint native views (platform rendering)               │
│  5. Composite and display (GPU)                           │
│                                                           │
│  If any of these exceed 16.6ms → frame drop → jank        │
│                                                           │
│  COMMON CAUSES:                                            │
│  • JS thread blocked (complex computation during scroll)  │
│  • Too many re-renders (unnecessary React reconciliation) │
│  • Heavy layout (deeply nested views)                     │
│  • Large images (decoding on UI thread)                   │
│  • Garbage collection pauses (Hermes GC)                  │
└──────────────────────────────────────────────────────────┘
```

### Measuring FPS in Production

```typescript
// lib/fps-monitor.ts
import { FrameCallbackRegistration } from 'react-native-reanimated';

class FPSMonitor {
  private frameCount = 0;
  private droppedFrames = 0;
  private lastTimestamp = 0;
  private isRunning = false;
  private samples: { fps: number; dropped: number; timestamp: number }[] = [];

  start() {
    this.isRunning = true;
    this.lastTimestamp = performance.now();
    this.tick();
  }

  stop() {
    this.isRunning = false;
    return this.getSummary();
  }

  private tick() {
    if (!this.isRunning) return;

    requestAnimationFrame(() => {
      const now = performance.now();
      const delta = now - this.lastTimestamp;

      this.frameCount++;

      // If frame took longer than 16.6ms, it's a dropped frame
      if (delta > 16.7) {
        const droppedCount = Math.floor(delta / 16.7) - 1;
        this.droppedFrames += droppedCount;
      }

      // Record a sample every second
      if (delta > 1000) {
        const fps = Math.round((this.frameCount / delta) * 1000);
        this.samples.push({
          fps,
          dropped: this.droppedFrames,
          timestamp: now,
        });

        this.frameCount = 0;
        this.droppedFrames = 0;
        this.lastTimestamp = now;
      }

      this.tick();
    });
  }

  getSummary() {
    if (this.samples.length === 0) return null;

    const fpsValues = this.samples.map(s => s.fps);
    const sortedFps = [...fpsValues].sort((a, b) => a - b);

    return {
      avgFps: Math.round(fpsValues.reduce((a, b) => a + b, 0) / fpsValues.length),
      p5Fps: sortedFps[Math.floor(sortedFps.length * 0.05)],    // Worst 5%
      p50Fps: sortedFps[Math.floor(sortedFps.length * 0.5)],    // Median
      p95Fps: sortedFps[Math.floor(sortedFps.length * 0.95)],   // Best 95%
      minFps: sortedFps[0],
      totalDroppedFrames: this.samples.reduce((a, s) => a + s.dropped, 0),
      duration: this.samples.length, // seconds
    };
  }
}

export const fpsMonitor = new FPSMonitor();
```

### Using the FPS Monitor for Specific Interactions

```typescript
// Measure FPS during scroll
function ProductList() {
  const handleScrollBegin = () => {
    fpsMonitor.start();
  };

  const handleScrollEnd = () => {
    const summary = fpsMonitor.stop();
    if (summary) {
      analytics.track('scroll_performance', {
        screen: 'ProductList',
        ...summary,
      });

      // Alert if scroll performance is bad
      if (summary.p5Fps < 30) {
        console.warn(
          `Scroll jank detected: p5 FPS = ${summary.p5Fps}`,
          summary
        );
      }
    }
  };

  return (
    <FlashList
      data={products}
      renderItem={renderProduct}
      onScrollBeginDrag={handleScrollBegin}
      onMomentumScrollEnd={handleScrollEnd}
      estimatedItemSize={120}
    />
  );
}
```

### FPS Thresholds

```
┌──────────────────────────────────────────────────────────┐
│  FPS QUALITY LEVELS                                       │
│                                                           │
│  60 FPS (16.6ms/frame)  — Smooth (target for most apps)  │
│  45-59 FPS              — Acceptable (minor jank)         │
│  30-44 FPS              — Noticeable jank (user will feel)│
│  < 30 FPS               — Severe jank (unacceptable)     │
│                                                           │
│  For 120Hz displays:                                      │
│  120 FPS (8.3ms/frame)  — Butter smooth                  │
│  90-119 FPS             — Smooth                          │
│  60-89 FPS              — Acceptable                      │
│  < 60 FPS               — Noticeable                     │
│                                                           │
│  KEY: Measure p5 (worst 5%), not average                  │
│  A p5 < 30 FPS means 1 in 20 seconds is severely janky   │
└──────────────────────────────────────────────────────────┘
```

---

## 4. MEMORY USAGE TRACKING

Memory is the silent killer of mobile apps. Unlike crashes (which are loud and obvious), memory issues degrade the experience gradually — the app gets slower as the garbage collector works harder, the OS kills background tabs and cached data more aggressively, and eventually the OS kills your app entirely with no crash report.

### Measuring Memory in React Native

```typescript
// lib/memory-monitor.ts
import { NativeModules, Platform } from 'react-native';

interface MemoryInfo {
  jsHeapSizeBytes: number;    // JavaScript heap (Hermes)
  nativeHeapSizeBytes: number; // Native heap (Java/ObjC)
  totalMemoryBytes: number;    // Total app memory
  availableMemoryBytes: number; // Device available memory
}

export async function getMemoryInfo(): Promise<MemoryInfo> {
  if (Platform.OS === 'android') {
    // Android: use Debug.getMemoryInfo() via native module
    return NativeModules.MemoryModule.getMemoryInfo();
  }

  // iOS: use task_info() via native module
  return NativeModules.MemoryModule.getMemoryInfo();
}

// Hermes-specific heap info
export function getHermesMemoryInfo() {
  if (global.HermesInternal) {
    const heapInfo = global.HermesInternal.getInstrumentedStats?.();
    return {
      jsHeapSize: heapInfo?.js_heapSize ?? 0,
      jsHeapAllocated: heapInfo?.js_totalAllocatedBytes ?? 0,
      jsGcCount: heapInfo?.js_numGCs ?? 0,
    };
  }
  return null;
}
```

### Continuous Memory Monitoring

```typescript
// lib/memory-reporter.ts

class MemoryReporter {
  private intervalId: ReturnType<typeof setInterval> | null = null;
  private samples: { timestamp: number; heapSize: number; total: number }[] = [];
  private baselineHeap: number = 0;

  start(intervalMs: number = 30000) {
    // Record baseline
    const initial = getHermesMemoryInfo();
    this.baselineHeap = initial?.jsHeapSize ?? 0;

    this.intervalId = setInterval(async () => {
      const hermesInfo = getHermesMemoryInfo();
      const memInfo = await getMemoryInfo();

      const sample = {
        timestamp: Date.now(),
        heapSize: hermesInfo?.jsHeapSize ?? 0,
        total: memInfo.totalMemoryBytes,
      };

      this.samples.push(sample);

      // Check for memory leak pattern
      if (this.samples.length >= 10) {
        this.checkForLeaks();
      }

      // Check for memory pressure
      const usagePercent = memInfo.totalMemoryBytes /
        (memInfo.totalMemoryBytes + memInfo.availableMemoryBytes) * 100;

      if (usagePercent > 80) {
        console.warn(`Memory pressure: ${usagePercent.toFixed(0)}% used`);
        analytics.track('memory_pressure', {
          usagePercent,
          jsHeapMB: (hermesInfo?.jsHeapSize ?? 0) / 1024 / 1024,
        });
      }
    }, intervalMs);
  }

  stop() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  private checkForLeaks() {
    // Simple linear regression on heap size
    // If it's consistently increasing, we might have a leak
    const recentSamples = this.samples.slice(-10);
    const firstHeap = recentSamples[0].heapSize;
    const lastHeap = recentSamples[recentSamples.length - 1].heapSize;
    const growthRate = (lastHeap - firstHeap) / recentSamples.length;

    // If heap is growing by more than 1MB per sample interval, flag it
    if (growthRate > 1024 * 1024) {
      console.warn(
        `Possible memory leak: heap growing at ${(growthRate / 1024 / 1024).toFixed(1)}MB per interval`
      );
      analytics.track('possible_memory_leak', {
        growthRateMBPerInterval: growthRate / 1024 / 1024,
        currentHeapMB: lastHeap / 1024 / 1024,
        baselineHeapMB: this.baselineHeap / 1024 / 1024,
      });
    }
  }
}

export const memoryReporter = new MemoryReporter();
```

### Memory Thresholds

```
┌──────────────────────────────────────────────────────────┐
│  MEMORY THRESHOLDS BY DEVICE CLASS                        │
│                                                           │
│  LOW-END (2-3GB RAM):                                     │
│    JS heap warning:  > 80MB                               │
│    Total app warning: > 200MB                             │
│    OOM risk:          > 300MB                             │
│                                                           │
│  MID-RANGE (4-6GB RAM):                                   │
│    JS heap warning:  > 120MB                              │
│    Total app warning: > 350MB                             │
│    OOM risk:          > 500MB                             │
│                                                           │
│  HIGH-END (8GB+ RAM):                                     │
│    JS heap warning:  > 200MB                              │
│    Total app warning: > 500MB                             │
│    OOM risk:          > 800MB                             │
│                                                           │
│  TREND ALERTS:                                             │
│    Heap growth > 1MB/minute sustained → Likely leak       │
│    GC frequency > 10/second → Memory pressure             │
│    GC pause > 100ms → User-visible jank                   │
└──────────────────────────────────────────────────────────┘
```

---

## 5. NETWORK LATENCY AND PAYLOAD SIZE

Mobile networks are unpredictable. Your API might respond in 50ms from your office, but 500ms for a user on a train. Understanding your actual network performance profile is critical.

### Measuring API Latency

```typescript
// lib/network-metrics.ts

interface NetworkMetric {
  url: string;
  method: string;
  statusCode: number;
  requestSizeBytes: number;
  responseSizeBytes: number;
  dnsLookupMs: number;
  tlsHandshakeMs: number;
  ttfbMs: number;        // Time to first byte
  downloadMs: number;
  totalMs: number;
  connectionType: string; // wifi, cellular, etc.
}

// Wrap fetch to automatically collect metrics
export function instrumentedFetch(input: RequestInfo, init?: RequestInit): Promise<Response> {
  const startTime = performance.now();
  const url = typeof input === 'string' ? input : input.url;
  const method = init?.method ?? 'GET';

  return fetch(input, init).then(async (response) => {
    const endTime = performance.now();
    const totalMs = endTime - startTime;

    // Get response size
    const clone = response.clone();
    const body = await clone.arrayBuffer();
    const responseSizeBytes = body.byteLength;

    // Get request size
    const requestSizeBytes = init?.body
      ? new Blob([init.body]).size
      : 0;

    const metric: NetworkMetric = {
      url: new URL(url).pathname, // Don't log query params (could contain PII)
      method,
      statusCode: response.status,
      requestSizeBytes,
      responseSizeBytes,
      dnsLookupMs: 0,      // Not directly available from fetch
      tlsHandshakeMs: 0,   // Would need native module
      ttfbMs: totalMs * 0.6, // Approximate (use native module for real)
      downloadMs: totalMs * 0.4,
      totalMs,
      connectionType: await getConnectionType(),
    };

    // Report metric
    reportNetworkMetric(metric);

    // Warn on slow requests
    if (totalMs > 3000) {
      console.warn(`Slow request: ${method} ${url} took ${totalMs.toFixed(0)}ms`);
    }

    // Warn on large payloads
    if (responseSizeBytes > 500 * 1024) { // > 500KB
      console.warn(
        `Large response: ${method} ${url} was ${(responseSizeBytes / 1024).toFixed(0)}KB`
      );
    }

    return response;
  });
}
```

### Network Thresholds

```
┌──────────────────────────────────────────────────────────┐
│  NETWORK PERFORMANCE TARGETS                              │
│                                                           │
│  API LATENCY (p50 / p95):                                 │
│    WiFi:     < 200ms / < 500ms                            │
│    LTE/5G:   < 300ms / < 800ms                            │
│    3G:       < 500ms / < 2000ms                           │
│                                                           │
│  PAYLOAD SIZE:                                             │
│    API response: < 100KB (ideal), < 500KB (acceptable)    │
│    Image: < 200KB (list), < 500KB (detail)                │
│    Initial data bundle: < 50KB                            │
│                                                           │
│  ERROR RATES:                                              │
│    Network errors: < 1% (healthy), > 5% (investigate)     │
│    Timeout rate: < 0.5%                                   │
│    5xx errors: < 0.1%                                     │
└──────────────────────────────────────────────────────────┘
```

---

## 6. APP SIZE OPTIMIZATION

App size directly affects download rates, update rates, and storage-constrained devices. Google reports that for every 6MB increase in app size, install conversion drops by 1%.

### Measuring App Size

```bash
# iOS: Xcode → Product → Archive → Distribute App → App Thinning Size Report
# This shows the actual download size per device type

# Android: Use bundletool
bundletool build-apks \
  --bundle=app.aab \
  --output=output.apks \
  --device-spec=device-spec.json

# Or check Play Console → App size
```

### What Contributes to App Size

```
┌──────────────────────────────────────────────────────────┐
│  REACT NATIVE APP SIZE BREAKDOWN (typical)                │
│                                                           │
│  JavaScript bundle         10-25 MB  (compressed: 2-5MB) │
│  Native libraries          15-30 MB                      │
│    ├── Hermes engine        ~3 MB                        │
│    ├── React Native core    ~8 MB                        │
│    ├── Third-party native   ~5-15 MB (varies)            │
│    └── Your native code     ~1-3 MB                      │
│  Assets (images, fonts)     5-20 MB                      │
│  Resources                  1-5 MB                       │
│                                                           │
│  Total (before thinning):   30-80 MB                     │
│  Download size (thinned):   15-40 MB                     │
│                                                           │
│  TARGETS:                                                 │
│    Download size < 30MB  — Good                          │
│    Download size < 50MB  — Acceptable                    │
│    Download size > 100MB — Users may skip on cellular    │
│    Download size > 200MB — Requires WiFi on iOS          │
└──────────────────────────────────────────────────────────┘
```

### Size Optimization Checklist

```
┌─────────────────────────────────────────────────────────────┐
│  APP SIZE OPTIMIZATION CHECKLIST                             │
│                                                              │
│  JS BUNDLE:                                                  │
│  ☐ Enable Hermes (compiles to bytecode, smaller than JS)    │
│  ☐ Enable Proguard/R8 (Android) for dead code elimination   │
│  ☐ Use import() for lazy loading non-critical screens       │
│  ☐ Audit dependencies: npx react-native-bundle-visualizer   │
│  ☐ Remove unused dependencies (check with depcheck)         │
│                                                              │
│  ASSETS:                                                     │
│  ☐ Compress images (use WebP/AVIF instead of PNG/JPEG)      │
│  ☐ Use vector icons instead of image assets                 │
│  ☐ Subset fonts (include only needed character ranges)      │
│  ☐ Consider downloading large assets on-demand              │
│                                                              │
│  NATIVE:                                                     │
│  ☐ Enable App Thinning (iOS) — sends only relevant slices   │
│  ☐ Use Android App Bundle (AAB) not APK                     │
│  ☐ Audit native dependencies for size impact                │
│  ☐ Enable bitcode (iOS) for compiler optimization           │
│  ☐ Strip debug symbols from release builds                  │
│                                                              │
│  MONITORING:                                                  │
│  ☐ Track app size in CI (fail on >5% increase)              │
│  ☐ Compare against size budget                              │
│  ☐ Review size impact of every new dependency               │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. GOOGLE PLAY CONSOLE VITALS

Play Console's Android Vitals is the most important dashboard for Android app health. Google uses these metrics to determine your app's quality tier, which affects store ranking, featuring, and user trust signals.

### The Core Vitals

```
┌─────────────────────────────────────────────────────────────┐
│  ANDROID VITALS — CORE METRICS                               │
│                                                              │
│  CRASH RATE (user-perceived)                                 │
│  Bad threshold:  > 1.09%  (daily sessions with crashes)     │
│  Target:         < 0.5%                                      │
│  Google's bar:   Bottom 25% of peer apps → bad rating       │
│  Note: JavaScript crashes that show error screens count!     │
│                                                              │
│  ANR RATE (Application Not Responding)                        │
│  Bad threshold:  > 0.47%  (daily sessions with ANRs)        │
│  Target:         < 0.1%                                      │
│  Trigger:        Main thread blocked > 5 seconds            │
│  Common cause:   Synchronous I/O, heavy computation on main │
│  RN-specific:    Long Bridge serialization, sync native calls│
│                                                              │
│  STARTUP TIME                                                │
│  Bad threshold:  Cold start > 5 seconds                      │
│  Target:         < 2 seconds cold start                      │
│  Measured:       From process creation to first frame drawn  │
│                                                              │
│  PERMISSION DENIALS                                          │
│  Bad threshold:  Varies by permission type                   │
│  Note: Excessive denials suggest poor permission UX          │
│                                                              │
│  BATTERY (Background)                                        │
│  Excessive wakeups, stuck wake locks, excessive WiFi scans   │
│  These get your app flagged as battery-draining              │
└─────────────────────────────────────────────────────────────┘
```

### Reading Play Console Vitals

```
Navigation: Play Console → Your app → Android vitals → Overview

KEY SECTIONS:
1. Core vitals summary — green/yellow/red status for each metric
2. Crash rate — daily trend, breakdown by device, OS, version
3. ANR rate — same breakdowns
4. Startup time — distribution histogram
5. Technical quality — grouped by issue type

WHAT TO DO:
1. Check vitals after every release (24-48 hour window)
2. Set up email alerts for threshold violations
3. Segment by:
   - App version (detect regressions)
   - Device model (find device-specific issues)
   - Android version (find OS-specific issues)
4. Prioritize fixes by user impact (sessions affected)
```

### Common ANR Causes in React Native

```
┌─────────────────────────────────────────────────────────────┐
│  ANR DIAGNOSIS IN REACT NATIVE                               │
│                                                              │
│  CAUSE: Synchronous native module calls on main thread       │
│  Fix: Make the native method async, or use TurboModules      │
│  Check: Look for "main" in ANR stack traces                  │
│                                                              │
│  CAUSE: Large Bridge serialization (old architecture)        │
│  Fix: Migrate to New Architecture (JSI)                      │
│  Check: Look for ReadableNativeMap in stack traces           │
│                                                              │
│  CAUSE: Synchronous disk I/O on main thread                  │
│  Fix: Use async I/O, move to background thread               │
│  Check: Look for "write" or "read" syscalls in traces        │
│                                                              │
│  CAUSE: SQLite operations on main thread                     │
│  Fix: Use MMKV or move SQLite to background thread           │
│  Check: Look for "sqlite3_" in stack traces                  │
│                                                              │
│  CAUSE: Layout thrashing (many synchronous measure calls)    │
│  Fix: Reduce nested views, batch layout operations           │
│  Check: Look for "onMeasure" in stack traces                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. APP STORE CONNECT METRICS

Apple provides a different set of metrics through App Store Connect and Xcode Organizer.

### Key Metrics in App Store Connect

```
┌─────────────────────────────────────────────────────────────┐
│  APP STORE CONNECT → App Analytics                           │
│                                                              │
│  PERFORMANCE:                                                │
│  Crash Rate        — Crashes per session (similar to Play)  │
│  Disk Writes       — Excessive I/O can trigger warnings     │
│  Launch Time       — p50 and p95 cold start times           │
│  Hang Rate         — Main thread blocked > 250ms            │
│  Memory (peak)     — Maximum memory usage per session       │
│                                                              │
│  ENGAGEMENT:                                                 │
│  Sessions per Active Device                                  │
│  Average Session Duration                                    │
│  Retention (Day 1, Day 7, Day 28)                           │
│  Active Devices (daily, monthly)                             │
│                                                              │
│  APPLE'S QUALITY BAR:                                        │
│  - Apps with high hang rates get warning emails              │
│  - Apps with high crash rates can be removed from the store  │
│  - Battery usage complaints trigger review flags             │
└─────────────────────────────────────────────────────────────┘
```

### Xcode Organizer Metrics

```
Xcode → Window → Organizer → Metrics tab

AVAILABLE METRICS:
1. Battery Usage (CPU time, networking, location, display)
2. Launch Time (histograms, trends, percentiles)
3. Hang Rate (by duration: 250ms-500ms, 500ms-1s, 1s-2s, 2s+)
4. Memory (peak, suspended memory, memory graph)
5. Disk Writes (logical, excessive write diagnostics)
6. Scrolling (hitch rate during scroll events)

These metrics are collected from real users via MetricKit,
aggregated by Apple, and presented with 24-hour delay.
```

### MetricKit for Custom Metrics

```swift
// AppDelegate.swift (or a native module)
import MetricKit

class MetricManager: NSObject, MXMetricManagerSubscriber {
    static let shared = MetricManager()

    func start() {
        MXMetricManager.shared.add(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            // Launch metrics
            if let launchMetrics = payload.applicationLaunchMetrics {
                let coldStart = launchMetrics.histogrammedTimeToFirstDraw
                // Send to your analytics
            }

            // Hang metrics
            if let hangMetrics = payload.applicationResponsivenessMetrics {
                let hangTime = hangMetrics.histogrammedApplicationHangTime
                // Alert on regression
            }

            // Memory metrics
            if let memoryMetrics = payload.memoryMetrics {
                let peakMemory = memoryMetrics.peakMemoryUsage
                // Track trends
            }
        }
    }

    // Diagnostic reports (crash/hang details)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            if let crashDiagnostics = payload.crashDiagnostics {
                for diagnostic in crashDiagnostics {
                    let callStack = diagnostic.callStackTree
                    // Symbolicate and analyze
                }
            }
        }
    }
}
```

---

## 9. DATADOG RUM FOR MOBILE

Real User Monitoring (RUM) gives you production telemetry from actual user sessions. While Play Console and App Store Connect provide aggregate metrics, RUM gives you individual session traces — you can see exactly what one user did, how long each screen took, what errors they encountered, and what the app's performance was at each step.

### Setting Up Datadog RUM

```typescript
// lib/datadog.ts
import {
  DdSdkReactNative,
  DdSdkReactNativeConfiguration,
  SdkVerbosity,
  TrackingConsent,
} from '@datadog/mobile-react-native';

const config = new DdSdkReactNativeConfiguration(
  process.env.DATADOG_CLIENT_TOKEN!,
  process.env.DATADOG_ENVIRONMENT!, // 'production', 'staging'
  process.env.DATADOG_APPLICATION_ID!,
  true, // Track interactions
  true, // Track XHR
  true  // Track errors
);

config.site = 'US1'; // or 'EU1', 'US3', etc.
config.nativeCrashReportEnabled = true;
config.sampleRate = 100; // 100% of sessions in production
config.serviceName = 'my-mobile-app';
config.verbosity = __DEV__ ? SdkVerbosity.DEBUG : SdkVerbosity.WARN;

export async function initializeDatadog() {
  await DdSdkReactNative.initialize(config);

  // Set user info when authenticated
  const user = await getAuthenticatedUser();
  if (user) {
    DdSdkReactNative.setUser({
      id: user.id,
      name: user.name,
      email: user.email,
    });
  }

  // Set global attributes
  DdSdkReactNative.setAttributes({
    app_version: DeviceInfo.getVersion(),
    build_number: DeviceInfo.getBuildNumber(),
    device_model: DeviceInfo.getModel(),
    os_version: DeviceInfo.getSystemVersion(),
    is_low_end_device: isLowEndDevice(),
  });
}
```

### Tracking Views (Screens)

```typescript
// lib/datadog-navigation.ts
import { DdRumReactNavigationTracking } from '@datadog/mobile-react-native-navigation';

// In your navigation setup
export function NavigationContainer({ children }: { children: React.ReactNode }) {
  const navigationRef = useNavigationContainerRef();

  return (
    <RNNavigationContainer
      ref={navigationRef}
      onReady={() => {
        DdRumReactNavigationTracking.startTrackingViews(navigationRef);
      }}
    >
      {children}
    </RNNavigationContainer>
  );
}
```

### Custom Performance Metrics

```typescript
import { DdRum } from '@datadog/mobile-react-native';

// Track a custom timing
function useTrackScreenLoad(screenName: string) {
  const startTime = useRef(performance.now());

  useEffect(() => {
    return () => {
      const duration = performance.now() - startTime.current;
      DdRum.addTiming(`${screenName}_load_time`);
    };
  }, [screenName]);

  // Call when content is visible
  const markContentReady = useCallback(() => {
    const duration = performance.now() - startTime.current;
    DdRum.addTiming(`${screenName}_content_ready`);

    // Also track as a custom metric
    DdRum.addAction('CUSTOM', `${screenName}_content_ready`, {
      duration_ms: duration,
    });
  }, [screenName]);

  return { markContentReady };
}

// Usage
function ProductDetailScreen({ productId }: { productId: string }) {
  const { markContentReady } = useTrackScreenLoad('ProductDetail');

  const { data: product } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
    onSuccess: () => markContentReady(),
  });

  // ...
}
```

### Custom Error Tracking

```typescript
// Track business logic errors (not just crashes)
function handlePaymentError(error: PaymentError) {
  DdRum.addError(
    error.message,
    'SOURCE', // or 'NETWORK', 'CONSOLE'
    error.stack ?? '',
    {
      payment_provider: error.provider,
      error_code: error.code,
      amount: error.amount,
      currency: error.currency,
    }
  );
}

// Track API errors with context
function instrumentApiClient(baseClient: ApiClient) {
  return new Proxy(baseClient, {
    get(target, prop) {
      const original = target[prop as keyof ApiClient];
      if (typeof original !== 'function') return original;

      return async (...args: any[]) => {
        try {
          return await original.apply(target, args);
        } catch (error) {
          DdRum.addError(
            `API Error: ${String(prop)}`,
            'NETWORK',
            error instanceof Error ? error.stack ?? '' : '',
            {
              endpoint: String(prop),
              status_code: error.statusCode,
              args: JSON.stringify(args).slice(0, 200),
            }
          );
          throw error;
        }
      };
    },
  });
}
```

---

## 10. SETTING UP DASHBOARDS

Metrics are useless if nobody looks at them. Dashboards make metrics visible, and visibility drives action.

### The Three Essential Dashboards

```
┌─────────────────────────────────────────────────────────────┐
│  DASHBOARD 1: RELEASE HEALTH                                 │
│  Who looks: Engineering lead, on-call engineer               │
│  When: After every release, daily check                      │
│                                                              │
│  Metrics:                                                    │
│  • Crash rate (current vs previous version)                  │
│  • ANR rate (current vs previous version)                    │
│  • Cold start p50 and p95 (current vs previous)              │
│  • Error rate by type                                        │
│  • App store rating (rolling 7-day average)                  │
│  • Active users (to detect mass uninstalls)                  │
│                                                              │
│  Layout:                                                     │
│  Top row: Big numbers with red/green comparison arrows       │
│  Middle: Time series showing last 14 days                    │
│  Bottom: Top 5 crashes/errors by frequency                   │
├─────────────────────────────────────────────────────────────┤
│  DASHBOARD 2: PERFORMANCE                                    │
│  Who looks: Performance team, engineering leads               │
│  When: Weekly review, after optimizations                    │
│                                                              │
│  Metrics:                                                    │
│  • TTI by device class (high/mid/low)                       │
│  • FPS p5 during scroll (by screen)                         │
│  • Memory usage trend (7-day, 30-day)                       │
│  • JS bundle size trend                                      │
│  • API latency p50/p95 (by endpoint)                        │
│  • Image load time p50/p95                                   │
│                                                              │
│  Layout:                                                     │
│  Top row: Performance scores by device class                 │
│  Middle: Histograms for TTI, FPS, memory                    │
│  Bottom: Slowest screens and endpoints                      │
├─────────────────────────────────────────────────────────────┤
│  DASHBOARD 3: BUSINESS IMPACT                                │
│  Who looks: Product, engineering leadership                   │
│  When: Weekly business review, quarterly planning            │
│                                                              │
│  Metrics:                                                    │
│  • Session duration (trending)                               │
│  • Funnel conversion with performance overlay               │
│  • Checkout completion rate vs crash rate                    │
│  • User retention (D1, D7, D30) vs app performance          │
│  • Store rating trend                                        │
│  • Performance impact on revenue (correlation)               │
│                                                              │
│  Layout:                                                     │
│  Correlation charts: performance metric vs business metric   │
│  Goal tracking: current vs target for key metrics            │
└─────────────────────────────────────────────────────────────┘
```

### Datadog Dashboard Configuration

```typescript
// Example Datadog dashboard definition (via API or Terraform)
const performanceDashboard = {
  title: 'Mobile Performance',
  widgets: [
    {
      title: 'Cold Start Time (p50) by Device Class',
      type: 'timeseries',
      requests: [
        {
          q: 'p50:rum.action.loading_time{@type:view_loading,@device.class:high}',
          display_type: 'line',
          style: { palette: 'green' },
        },
        {
          q: 'p50:rum.action.loading_time{@type:view_loading,@device.class:mid}',
          display_type: 'line',
          style: { palette: 'yellow' },
        },
        {
          q: 'p50:rum.action.loading_time{@type:view_loading,@device.class:low}',
          display_type: 'line',
          style: { palette: 'red' },
        },
      ],
    },
    {
      title: 'Crash-Free Sessions',
      type: 'query_value',
      requests: [
        {
          q: '100 - (count:rum.error{@error.source:crash} / count:rum.session{} * 100)',
          aggregator: 'avg',
        },
      ],
      conditional_formats: [
        { comparator: '>=', value: 99.5, palette: 'green_on_white' },
        { comparator: '>=', value: 99.0, palette: 'yellow_on_white' },
        { comparator: '<', value: 99.0, palette: 'red_on_white' },
      ],
    },
    // ... more widgets
  ],
};
```

---

## 11. ALERT THRESHOLDS AND ON-CALL PRACTICES

Dashboards show you the state. Alerts tell you when to act.

### Alert Configuration

```
┌─────────────────────────────────────────────────────────────┐
│  CRITICAL ALERTS (page on-call immediately)                  │
│                                                              │
│  Crash rate > 2% of sessions in last 1 hour                 │
│  ANR rate > 1% of sessions in last 1 hour                   │
│  5xx error rate > 5% of API calls in last 15 minutes        │
│  Cold start p95 > 5 seconds (sustained 30 minutes)          │
│  Memory leak detected (heap growth > 5MB/minute for 10 min) │
│                                                              │
│  SEVERITY:  PagerDuty / Opsgenie page                        │
│  RESPONSE:  Investigate within 15 minutes                    │
│  ESCALATION: If no response in 30 minutes, page team lead   │
├─────────────────────────────────────────────────────────────┤
│  WARNING ALERTS (notify Slack channel)                        │
│                                                              │
│  Crash rate > 1% of sessions (rolling 24h)                   │
│  TTI p95 > 3 seconds (rolling 24h)                           │
│  FPS p5 < 30 during scroll (rolling 24h)                     │
│  App size increased > 5% from last release                   │
│  New crash cluster (>100 occurrences, new signature)         │
│  API latency p95 > 2x normal baseline                        │
│                                                              │
│  SEVERITY:  Slack notification                               │
│  RESPONSE:  Investigate same business day                    │
│  ESCALATION: If not addressed in 48 hours, create ticket     │
├─────────────────────────────────────────────────────────────┤
│  INFORMATIONAL ALERTS (weekly digest)                         │
│                                                              │
│  JS bundle size trend (4-week growth rate)                   │
│  Memory usage trend (4-week growth rate)                     │
│  Slowest screens by TTI (weekly top 10)                      │
│  Most impactful crashes (weekly top 10 by sessions affected) │
│  Play Console / App Store vitals summary                     │
│                                                              │
│  SEVERITY:  Email / Weekly report                            │
│  RESPONSE:  Review in weekly performance meeting             │
└─────────────────────────────────────────────────────────────┘
```

### Post-Release Monitoring Checklist

```
After every release, the releasing engineer should:

□ 15 MINUTES AFTER RELEASE:
  - Check crash rate on new version vs previous
  - Verify no new crash clusters
  - Check API error rates (new version might hit new endpoints)

□ 1 HOUR AFTER RELEASE:
  - Compare crash rate with 1-hour baseline
  - Check ANR rate
  - Verify cold start time hasn't regressed
  - Check Play Console vitals pre-launch report (if available)

□ 24 HOURS AFTER RELEASE:
  - Full vitals comparison (crash, ANR, startup, memory)
  - Check app store reviews for performance complaints
  - Review performance dashboard for regressions
  - Confirm no memory leak trends

□ 1 WEEK AFTER RELEASE:
  - Compare all metrics against 7-day baseline
  - Review Play Console / App Store Connect vitals
  - Check long-running session metrics (memory leaks)
  - Share release health report with team
```

### The Performance Budget

Set explicit budgets that new features must stay within:

```typescript
// performance-budget.ts

export const PERFORMANCE_BUDGET = {
  // Cold start
  tti: {
    p50_max_ms: 800,
    p95_max_ms: 2000,
    regression_threshold_ms: 100, // Alert if TTI increases by 100ms+
  },

  // Frame rate
  fps: {
    scroll_p5_min: 45,       // Worst 5% during scroll must be > 45fps
    animation_p5_min: 50,     // Worst 5% during animation must be > 50fps
  },

  // Memory
  memory: {
    peak_js_heap_mb: 150,    // JS heap should never exceed 150MB
    growth_rate_mb_per_min: 1, // Should not grow more than 1MB/min sustained
  },

  // App size
  appSize: {
    max_download_mb: 40,      // Maximum download size
    max_increase_per_release_percent: 3, // Max 3% increase per release
  },

  // Network
  network: {
    api_p95_ms: 1000,         // API calls p95 < 1 second
    initial_data_kb: 50,      // First screen data < 50KB
  },

  // Stability
  stability: {
    crash_free_rate_min: 99.5, // 99.5%+ crash-free sessions
    anr_rate_max: 0.2,        // < 0.2% ANR rate
  },
} as const;
```

---

## CHAPTER SUMMARY

Mobile metrics that matter fall into three categories: user-facing (what users experience), system health (what predicts future problems), and business impact (what moves the needle for your product).

The essential metrics:

- **Cold start / TTI**: Target < 1s on mid-range devices. Measure with custom instrumentation and Datadog RUM.
- **FPS / jank**: Measure p5 during scroll. Target > 45 FPS on all device classes.
- **Memory**: Track JS heap and total app memory. Alert on sustained growth (leak detection).
- **Network**: Track API latency p50/p95 and payload sizes. Segment by connection type.
- **App size**: Set a budget, track in CI, gate releases on regressions.
- **Play Console vitals**: Crash rate < 1%, ANR rate < 0.5%. These affect store ranking.
- **App Store Connect**: Hang rate, launch time, memory — Apple will email you if you're bad.
- **Datadog RUM**: Session-level traces for debugging individual user experiences.

The operational practices that make metrics actionable:

- **Dashboards**: Release health, performance, and business impact — three views for three audiences
- **Alerts**: Critical (page on-call), Warning (Slack), Informational (weekly digest)
- **Post-release monitoring**: Structured checklist at 15 minutes, 1 hour, 24 hours, and 1 week
- **Performance budgets**: Explicit thresholds that new features must not exceed

The teams that build the best mobile apps are not the ones with the fastest code — they're the ones with the best visibility into how their code performs in the real world. Measure relentlessly, segment by device class, and act on regressions before your users notice them.

---

*Next: [Chapter 24: Internationalization & Localization]*
*Previous: [Chapter 22: Feature Flags & Remote Configuration]*