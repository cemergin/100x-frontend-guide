<!--
  CHAPTER: 44
  TITLE: Performance Budgets & Web Vitals — Keeping Apps Fast Forever
  PART: IV — Architecture at Scale
  PREREQS: Chapters 13, 18, 25
  KEY_TOPICS: performance budgets, Core Web Vitals, LCP, INP, CLS, Lighthouse, bundle size budgets, CI enforcement, performance regression detection, RUM vs synthetic, speed index, TTFB, performance culture
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 44: Performance Budgets & Web Vitals — Keeping Apps Fast Forever

> "Performance is not a feature you add. It is a constraint you enforce. The moment you stop enforcing it, it disappears."
> — Alex Russell, Chrome team

---

<details>
<summary><strong>TL;DR</strong></summary>

- Without enforced performance budgets, every sprint adds weight; in 6 months your bundle is 2x the size and your LCP has drifted from 1.8s to 4.2s -- nobody noticed because it happened 5KB at a time
- Core Web Vitals are LCP (<2.5s), INP (<200ms), and CLS (<0.1); Google uses these for search ranking, but more importantly your users use them to decide whether your app is worth their time
- Set explicit budgets per route (web) and per screen (mobile): bundle size, LCP, INP, CLS, TTFB, memory, FPS -- and enforce them in CI so the build fails before the regression ships
- You need both synthetic testing (Lighthouse in CI, controlled and reproducible) and Real User Monitoring (what actual users on real devices and networks experience); neither alone tells the full story
- Performance culture means making performance everyone's job: PR templates with performance checkboxes, weekly performance reviews, bundle size in the PR comment, and OKRs that include speed

</details>

Here is a story I have seen play out at four different companies. It always follows the same script.

Month 1: The app launches. Lighthouse score is 96. LCP is 1.4 seconds. The team is proud. The PM posts a screenshot in Slack.

Month 3: A few new features land. A charting library here, a rich text editor there. Nobody notices the bundle grew by 180KB. Lighthouse is now 89. LCP is 1.9 seconds. Still green in Chrome DevTools.

Month 6: The marketing team wants animations. The product team wants real-time updates. The bundle is now 1.2MB of JavaScript. LCP is 3.1 seconds on mobile. A few users complain. The team files a "performance sprint" ticket and puts it in the backlog.

Month 9: The performance sprint never happened because there was always something more urgent. The bundle is 1.6MB. LCP is 4.2 seconds. Bounce rate is up 23%. Conversion is down 15%. The CEO asks "why is the site slow?" and now it is a fire drill.

Month 12: The team spends six weeks doing a performance overhaul. They remove three libraries, code-split eight routes, and optimize images. They get LCP back to 2.4 seconds. It took six weeks to fix what would have taken six minutes to prevent -- if they had budgets.

This chapter is about those six minutes. It is about building the systems, processes, and culture that prevent performance from degrading in the first place. Not heroic optimization sprints. Not clever tricks. Systems.

---

### In This Chapter

- Why Performance Degrades -- the physics of entropy in codebases
- Core Web Vitals Deep Dive -- LCP, INP, CLS: what they measure, how to optimize each
- Mobile Performance Budgets -- cold start, FPS, bundle size, memory per screen
- Web Performance Budgets -- bundle per route, LCP, INP, TTFB, CLS, total page weight
- Enforcing Budgets in CI -- Lighthouse CI, size-limit, bundlesize, failing builds on regression
- RUM vs Synthetic Monitoring -- why you need both, and how to set them up
- Performance Regression Detection -- catching regressions across deploys before users do
- Bundle Analysis -- finding what is eating your bundle and verifying tree shaking works
- Performance Culture -- making speed everyone's responsibility, not just the "performance person"
- Complete CI Setup -- full GitHub Actions workflow with budget checks and PR comments

### Related Chapters

- [Ch 13: Mobile Performance] -- target metrics, cold start optimization, memory management
- [Ch 18: CI/CD for Mobile & Web] -- the pipeline these budget checks plug into
- [Ch 25: Web Performance] -- rendering strategies, code splitting, image optimization

---

## 44.1 Why Performance Degrades

Performance degradation is not a bug. It is the natural state of any codebase under active development. Understanding *why* it happens is the first step to preventing it.

### The 5KB Problem

Every feature adds weight. A new modal with a date picker: 12KB. An analytics event with a new property: 2KB. A toast notification library because the PM saw one they liked: 35KB. Individually, none of these are alarming. In code review, nobody is going to block a PR over 12KB.

But here is the math:

```
Average JS added per PR:        ~5KB (conservative)
PRs merged per week:             15 (small team)
Weeks in 6 months:               26

Total JS added: 5KB x 15 x 26 = 1,950KB = ~1.9MB

Starting bundle:                 300KB
Bundle after 6 months:           2,200KB = 2.2MB

That is a 7.3x increase.
```

And that assumes zero new dependencies. In reality, new libraries get added regularly. A charting library (200KB min-gzipped). A rich text editor (150KB). An animation library (45KB). A date formatting library because someone did not know about `Intl.DateTimeFormat` (32KB of moment.js or its descendants).

### The Ratchet Effect

Performance is a ratchet -- it only moves in one direction unless you actively prevent it.

```
                    Performance Over Time (No Budgets)

Score  100 | *
            |  *
        90 |   **
            |     **
        80 |       **
            |         ***
        70 |            **
            |              ***
        60 |                 ****
            |                    *****
        50 |                         *********
            |
        40 |                                    <-- "Why is the site slow?"
            |
            +--------------------------------------
              M1   M2   M3   M4   M5   M6   M7

                    Performance Over Time (With Budgets)

Score  100 | *
            |  *
        90 |   **  **   **  **   **  **   **
            |     **  **   **  **   **  **
        80 |  <-- Budget threshold (build fails below this)
            |
        70 |
            |
        60 |
            |
        50 |
            +--------------------------------------
              M1   M2   M3   M4   M5   M6   M7
```

With budgets, performance dips slightly as features land, then recovers when the budget check forces the team to optimize or split before merging. The result is a stable sawtooth pattern instead of a monotonic decline.

### Why "We'll Fix It Later" Never Works

1. **Performance debt compounds.** Unlike tech debt in code quality, performance debt affects users immediately. Every day your LCP is 4 seconds is a day you are losing conversions.

2. **It gets harder to fix over time.** Removing a library that 15 features depend on is 100x harder than not adding it in the first place. Splitting a monolithic bundle after a year of development requires touching every route.

3. **The team loses context.** The person who added the 200KB charting library left six months ago. Nobody knows if the tree shaking is configured correctly. Nobody knows which features use which parts of the library.

4. **There is always something more urgent.** Performance work competes with features for sprint capacity. Features have PMs advocating for them. Performance does not -- until the CEO notices.

The only reliable solution is automation. Make the build fail. Make the CI bot comment on the PR with the size increase. Make it impossible to merge a PR that busts the budget. Humans forget. CI does not.

---

## 44.2 Core Web Vitals — The Metrics That Matter

Google's Core Web Vitals are the industry standard for measuring user-perceived web performance. They are not perfect, but they are the best widely-adopted framework we have. As of 2026, there are three Core Web Vitals:

```
+----------------------------------------------------------------------+
|                        CORE WEB VITALS                               |
|                                                                      |
|  +------------------+  +------------------+  +------------------+    |
|  |       LCP        |  |       INP        |  |       CLS        |   |
|  |                  |  |                  |  |                  |    |
|  |  Largest         |  |  Interaction to  |  |  Cumulative      |   |
|  |  Contentful      |  |  Next Paint      |  |  Layout Shift    |   |
|  |  Paint           |  |                  |  |                  |    |
|  |                  |  |                  |  |                  |    |
|  |  Loading         |  |  Interactivity   |  |  Visual          |   |
|  |  Performance     |  |  Performance     |  |  Stability       |   |
|  |                  |  |                  |  |                  |    |
|  |  Good: < 2.5s    |  |  Good: < 200ms   |  |  Good: < 0.1     |   |
|  |  Poor: > 4.0s    |  |  Poor: > 500ms   |  |  Poor: > 0.25    |   |
|  +------------------+  +------------------+  +------------------+    |
|                                                                      |
|  Measured at 75th percentile of real user page loads                 |
|  Used by Google Search as a ranking signal                           |
+----------------------------------------------------------------------+
```

### LCP — Largest Contentful Paint

**What it measures:** The time from when the user starts loading the page to when the largest image or text block finishes rendering in the viewport. This is a proxy for "when does the user see the main content?"

**Thresholds:**
- Good: < 2.5 seconds
- Needs Improvement: 2.5 - 4.0 seconds
- Poor: > 4.0 seconds

**What counts as the "largest contentful paint":**
- `<img>` elements
- `<image>` elements inside `<svg>`
- `<video>` elements (poster image or first frame)
- Elements with `background-image` loaded via CSS
- Block-level text elements (`<p>`, `<h1>`, `<div>` with text, etc.)

**How it is measured:**

The browser uses the Performance Observer API to track paint events. Each time a larger content element finishes rendering, the LCP entry is updated. The final LCP value is the timestamp of the last entry before the user first interacts with the page (click, scroll, keypress).

```typescript
// Measuring LCP programmatically
const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];

  console.log('LCP:', lastEntry.startTime, 'ms');
  console.log('LCP element:', lastEntry.element);
  console.log('LCP size:', lastEntry.size);
  console.log('LCP URL:', lastEntry.url); // If it is an image
});

observer.observe({ type: 'largest-contentful-paint', buffered: true });
```

**Common causes of bad LCP:**

1. **Render-blocking resources.** CSS and synchronous JS in `<head>` blocks rendering entirely. The browser cannot paint anything until these resources are downloaded, parsed, and executed.

```html
<!-- BAD: This blocks LCP -->
<head>
  <link rel="stylesheet" href="/styles/main.css" />        <!-- Blocks rendering -->
  <link rel="stylesheet" href="/styles/fonts.css" />        <!-- Blocks rendering -->
  <script src="/scripts/analytics.js"></script>              <!-- Blocks rendering -->
  <link rel="stylesheet" href="/styles/above-the-fold.css" /> <!-- Blocks rendering -->
</head>

<!-- BETTER: Critical CSS inline, rest deferred -->
<head>
  <style>
    /* Inline only above-the-fold critical CSS (~14KB max) */
    .hero { display: flex; min-height: 80vh; }
    .hero-title { font-size: 3rem; font-weight: 700; }
    .nav { display: flex; height: 64px; align-items: center; }
  </style>
  <link rel="preload" href="/styles/main.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="/styles/main.css" /></noscript>
  <script src="/scripts/analytics.js" defer></script>
</head>
```

2. **Slow server response (high TTFB).** If the server takes 1.5 seconds to respond, your LCP cannot possibly be under 2.5 seconds on a slow connection. The LCP clock starts when the user navigates.

3. **Unoptimized hero images.** The LCP element is often a hero image. If that image is 2MB, served from a different origin, and not preloaded, it will be slow.

```html
<!-- BAD: Large image, no preload, different origin -->
<img src="https://cdn.example.com/hero-4000x2000.jpg" alt="Hero" />

<!-- GOOD: Optimized image with preload, proper sizing, modern format -->
<head>
  <link rel="preload" as="image" href="/hero-800w.webp"
        imagesrcset="/hero-400w.webp 400w, /hero-800w.webp 800w, /hero-1200w.webp 1200w"
        imagesizes="100vw" fetchpriority="high" />
</head>
<body>
  <img src="/hero-800w.webp"
       srcset="/hero-400w.webp 400w, /hero-800w.webp 800w, /hero-1200w.webp 1200w"
       sizes="100vw"
       alt="Hero"
       fetchpriority="high"
       decoding="async"
       width="1200" height="600" />
</body>
```

4. **Client-side rendering without SSR.** If your app renders a loading spinner server-side and then fetches data client-side to render the actual content, LCP does not fire until the real content appears. The spinner is not the "largest contentful paint" -- the actual content is.

```
Client-Side Rendering LCP Timeline:
+-- HTML loads (spinner renders)         ~500ms
+-- JS bundle downloads                   ~800ms
+-- JS parses and executes               ~400ms
+-- API call for page data               ~600ms
+-- React renders with real data         ~200ms
+-- LCP fires (content visible)          ~2500ms  <-- Too slow

Server-Side Rendering LCP Timeline:
+-- Server renders HTML with data        ~200ms  (TTFB)
+-- HTML streams to browser              ~100ms
+-- Browser paints content               ~200ms
+-- LCP fires (content visible)          ~500ms   <-- Fast
```

**How to optimize LCP:**

```typescript
// Next.js example: Optimize LCP with SSR + image priority

// app/page.tsx — Server Component (no "use client")
import Image from 'next/image';

export default async function HomePage() {
  // Data fetched server-side, included in initial HTML
  const hero = await getHeroContent();

  return (
    <main>
      <section className="hero">
        <h1>{hero.title}</h1>
        {/* priority prop adds fetchpriority="high" and preload link */}
        <Image
          src={hero.imageUrl}
          alt={hero.imageAlt}
          width={1200}
          height={600}
          priority  // This is the LCP element -- tell Next.js to prioritize it
          sizes="100vw"
        />
      </section>
    </main>
  );
}
```

### INP — Interaction to Next Paint

**What it measures:** The latency from when the user interacts (click, tap, keypress) to when the browser paints the next frame reflecting that interaction. This replaced FID (First Input Delay) in March 2024 because FID only measured the *first* interaction, while INP measures *all* interactions throughout the page's lifecycle.

**Thresholds:**
- Good: < 200 milliseconds
- Needs Improvement: 200 - 500 milliseconds
- Poor: > 500 milliseconds

**Why 200ms?** Research on human perception shows that responses under 100ms feel instant. Between 100-300ms, users notice a slight delay but it feels responsive. Above 300ms, users perceive the interface as sluggish. 200ms is the boundary of "responsive."

**How INP is calculated:**

INP is not the worst interaction, nor the average. It is the highest interaction latency, ignoring one interaction per every 50 interactions. For a page session with fewer than 50 interactions, it is the single worst interaction. This makes it resilient to one-off outliers while still capturing real problems.

```
User Session (23 interactions):
  Click "Add to Cart":    85ms
  Type in search:         42ms
  Click filter dropdown:  320ms   <-- Worst interaction
  Select filter option:   180ms
  Scroll:                 12ms
  Click "Buy Now":        95ms
  ... (17 more interactions averaging 60ms)

INP = 320ms (worst interaction, since < 50 total interactions)
```

**Common causes of bad INP:**

1. **Long tasks on the main thread.** Any JavaScript that runs for more than 50ms blocks the main thread. During that time, the browser cannot process user input or paint frames.

```typescript
// BAD: Synchronous heavy computation blocks interactions
function handleFilterChange(filters: FilterState) {
  // This takes 400ms on a mid-range phone
  const filtered = products.filter(product => {
    return matchesAllFilters(product, filters); // Complex matching logic
  });

  const sorted = filtered.sort((a, b) => {
    return complexSortFunction(a, b, sortConfig);
  });

  const grouped = groupByCategory(sorted);

  setDisplayProducts(grouped);
  // User's click does not get visual feedback for 400ms
}

// GOOD: Break up work with React transitions
import { startTransition } from 'react';

function handleFilterChange(filters: FilterState) {
  // Immediate visual feedback
  setIsFiltering(true);

  // Yield to the browser so it can paint the loading state
  // then do the heavy work
  startTransition(() => {
    const filtered = products.filter(product =>
      matchesAllFilters(product, filters)
    );
    const sorted = filtered.sort((a, b) =>
      complexSortFunction(a, b, sortConfig)
    );
    const grouped = groupByCategory(sorted);
    setDisplayProducts(grouped);
    setIsFiltering(false);
  });
}
```

2. **Large DOM size.** When the browser needs to update the layout after an interaction, a large DOM tree means more layout work. Pages with 3,000+ DOM nodes frequently have INP problems.

3. **Excessive re-renders.** If clicking a button triggers a re-render of 500 components because state is stored too high in the tree, the browser has to reconcile all of those, then paint.

4. **Third-party scripts.** Analytics, ads, chat widgets, and A/B testing scripts routinely add long tasks. They compete with your code for main thread time.

**How to optimize INP:**

```typescript
// Strategy 1: Use React transitions for non-urgent updates
import { useState, useTransition } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    // Urgent: update the input value immediately
    setQuery(e.target.value);

    // Non-urgent: update results can be deferred
    startTransition(() => {
      setResults(filterProducts(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleSearch} />
      <div style={{ opacity: isPending ? 0.7 : 1 }}>
        {results.map(r => <ProductCard key={r.id} product={r} />)}
      </div>
    </>
  );
}

// Strategy 2: Virtualize long lists so interactions do not
// trigger layout recalculation for thousands of DOM nodes
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualizedList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 64, // Estimated row height
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <div
            key={virtualRow.key}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              height: virtualRow.size,
              width: '100%',
            }}
          >
            <ListItem item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}

// Strategy 3: Debounce expensive event handlers
import { useDebouncedCallback } from 'use-debounce';

function MapView() {
  const handleViewportChange = useDebouncedCallback(
    (viewport: Viewport) => {
      // Expensive: re-queries markers, recalculates clusters
      updateVisibleMarkers(viewport);
    },
    150, // 150ms debounce
    { leading: true } // Fire immediately on first call
  );

  return <Map onViewportChange={handleViewportChange} />;
}
```

### CLS — Cumulative Layout Shift

**What it measures:** The total of all unexpected layout shifts that occur during the entire lifespan of the page. A layout shift happens when a visible element changes position from one frame to the next without being triggered by user input.

**Thresholds:**
- Good: < 0.1
- Needs Improvement: 0.1 - 0.25
- Poor: > 0.25

**How it is calculated:**

```
Layout Shift Score = Impact Fraction x Distance Fraction

Impact Fraction:  The area of the viewport affected by the shift
Distance Fraction: The distance elements moved, as a fraction of viewport

Example:
  An ad banner loads and pushes content down by 20% of the viewport
  Impact Fraction = 0.6 (60% of viewport is affected)
  Distance Fraction = 0.2 (content moved 20% of viewport height)
  Layout Shift Score = 0.6 x 0.2 = 0.12

  CLS = Sum of all such shifts (excluding those within 500ms of user input)
```

**Common causes of bad CLS:**

1. **Images without dimensions.** When an image loads, the browser does not know how much space it needs until the image headers arrive. Without explicit width/height, the browser allocates zero space, then shifts everything when the image loads.

```html
<!-- BAD: No dimensions, causes layout shift -->
<img src="/product.jpg" alt="Product" />

<!-- GOOD: Dimensions prevent layout shift -->
<img src="/product.jpg" alt="Product" width="400" height="300" />

<!-- GOOD: CSS aspect ratio for responsive images -->
<img src="/product.jpg" alt="Product"
     style="aspect-ratio: 4/3; width: 100%; height: auto;" />
```

2. **Dynamically injected content.** Ads, cookie banners, email signup bars, and notification banners that push content down after initial render.

```typescript
// BAD: Banner appears after load, pushes everything down
function App() {
  const [showBanner, setShowBanner] = useState(false);

  useEffect(() => {
    checkIfShouldShowBanner().then(setShowBanner);
  }, []);

  return (
    <div>
      {showBanner && <PromoBanner />}  {/* Layout shift! */}
      <MainContent />
    </div>
  );
}

// GOOD: Reserve space for the banner to prevent shift
function App() {
  const [showBanner, setShowBanner] = useState(false);
  const [bannerResolved, setBannerResolved] = useState(false);

  useEffect(() => {
    checkIfShouldShowBanner().then((show) => {
      setShowBanner(show);
      setBannerResolved(true);
    });
  }, []);

  return (
    <div>
      {/* Reserve fixed height until we know whether to show the banner */}
      <div style={{ minHeight: bannerResolved ? 'auto' : '60px' }}>
        {showBanner && <PromoBanner />}
      </div>
      <MainContent />
    </div>
  );
}
```

3. **Web fonts causing text reflow.** When a web font loads and replaces the fallback font, text dimensions change, causing layout shifts.

```css
/* BAD: Font swap causes visible text reflow */
@font-face {
  font-family: 'Brand Font';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: swap; /* Good for LCP, but causes CLS */
}

/* BETTER: Use font-display: optional to prevent layout shift */
@font-face {
  font-family: 'Brand Font';
  src: url('/fonts/brand.woff2') format('woff2');
  font-display: optional; /* No layout shift; font used if loaded in time */
}

/* BEST: Match fallback font metrics to web font */
@font-face {
  font-family: 'Brand Font Fallback';
  src: local('Arial');
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
  size-adjust: 107%;
}

body {
  font-family: 'Brand Font', 'Brand Font Fallback', Arial, sans-serif;
}
```

In Next.js, the `next/font` module handles this automatically:

```typescript
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  // Next.js automatically generates a fallback font with matching metrics
  // This eliminates CLS from font loading
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>{children}</body>
    </html>
  );
}
```

4. **Client-side navigation with skeleton mismatches.** If your skeleton/placeholder is a different height than the actual content, the page shifts when data loads.

### Supporting Metrics

Beyond the three Core Web Vitals, these metrics provide important context:

**TTFB (Time to First Byte):** How long it takes for the server to respond with the first byte. Target: < 200ms for static content, < 500ms for dynamic pages. A slow TTFB puts a floor under your LCP -- if the server takes 2 seconds to respond, your LCP cannot be under 2 seconds.

**Speed Index:** How quickly content is visually populated. Measured by capturing video of the page load and computing how much of the viewport is filled at each point. Useful for comparing overall visual loading experience.

**Total Blocking Time (TBT):** The total time between FCP and TTI during which the main thread was blocked for long enough (>50ms) to prevent input responsiveness. This is the lab equivalent of INP. Target: < 200ms.

**First Contentful Paint (FCP):** When the first piece of DOM content renders. Important as a leading indicator -- if FCP is slow, LCP will be slow.

```
Page Load Timeline:
|
+-- Navigation Start
|
+-- TTFB --------- Server responds with first byte        (target: < 200ms)
|
+-- FCP ---------- First content renders                   (target: < 1.8s)
|
+-- LCP ---------- Largest content element renders          (target: < 2.5s)
|
+-- TTI ---------- Page is fully interactive               (target: < 3.8s)
|
+-- User interacts... INP measured per interaction          (target: < 200ms)

Throughout: CLS accumulates from layout shifts               (target: < 0.1)
```

---

## 44.3 Mobile Performance Budgets

Mobile performance budgets differ from web budgets because the constraints are fundamentally different. You are dealing with constrained CPU, limited memory, battery concerns, and users who expect native-level responsiveness.

### Budget Categories

| Metric | Budget | Measurement Method |
|--------|--------|--------------------|
| **Cold Start** | < 1.5s | Time from app process creation to first meaningful content |
| **Time to Interactive** | < 2.0s | Time until user can actually interact with the screen |
| **Frame Rate** | 60fps sustained | No frame drops during scroll/animation (Flipper, Perf Monitor) |
| **Bundle Size** | < 15MB compressed | The OTA download size on App Store / Play Store |
| **Memory** | < 200MB peak | Maximum RSS during typical usage session |
| **JS Thread Frame Time** | < 16ms | P95 JS thread frame time during active usage |
| **API Response Rendering** | < 100ms | Time from API response to UI update |

### Setting Budgets Per Screen

Not all screens are equal. Your home screen needs to be the fastest thing in the app -- it is the first thing users see. A settings screen buried three levels deep can have a slightly more relaxed budget.

```typescript
// performance-budgets.mobile.ts
// Screen-level performance budgets for CI and monitoring

export const SCREEN_BUDGETS = {
  // Critical path -- users see these first
  'home': {
    timeToContent: 800,       // ms -- must show real content, not skeleton
    timeToInteractive: 1200,  // ms -- user can scroll, tap
    memoryDelta: 30,          // MB -- memory added by this screen
    jsFrameTime: 12,          // ms -- tighter budget for the main screen
  },

  // High-traffic screens
  'product-list': {
    timeToContent: 600,       // ms -- list should render fast (FlashList)
    timeToInteractive: 1000,  // ms
    memoryDelta: 50,          // MB -- lists load images
    jsFrameTime: 14,          // ms -- scrolling must be smooth
    scrollFps: 58,            // minimum FPS during rapid scroll
  },

  'product-detail': {
    timeToContent: 500,       // ms -- hero image + title
    timeToInteractive: 800,   // ms
    memoryDelta: 40,          // MB
    jsFrameTime: 14,          // ms
  },

  // Transaction screens -- correctness over speed, but still fast
  'checkout': {
    timeToContent: 1000,      // ms -- some latency acceptable
    timeToInteractive: 1500,  // ms
    memoryDelta: 25,          // MB
    jsFrameTime: 16,          // ms -- standard budget
  },

  // Low-traffic screens -- relaxed budgets
  'settings': {
    timeToContent: 1500,      // ms
    timeToInteractive: 2000,  // ms
    memoryDelta: 15,          // MB
    jsFrameTime: 16,          // ms
  },
} as const;

export const GLOBAL_BUDGETS = {
  totalBundleSize: 15 * 1024 * 1024,      // 15MB compressed
  totalMemoryPeak: 200 * 1024 * 1024,     // 200MB peak RSS
  coldStartTime: 1500,                     // 1.5 seconds
  warmStartTime: 500,                      // 0.5 seconds
  backgroundMemory: 50 * 1024 * 1024,     // 50MB when backgrounded
  jsBundleParseTime: 500,                  // 500ms for JS bundle parse
};
```

### React Native Bundle Size Budget

The bundle size directly affects cold start time on mobile. Every megabyte of JavaScript needs to be loaded from disk, parsed by Hermes, and executed. On a low-end device, parsing 1MB of JavaScript takes roughly 200-400ms.

```typescript
// scripts/check-mobile-bundle-size.ts
import * as fs from 'fs';
import { execFileSync } from 'child_process';

const BUDGET = {
  android: {
    totalApk: 25 * 1024 * 1024,        // 25MB APK (uncompressed)
    jsBundleRaw: 8 * 1024 * 1024,       // 8MB raw JS bundle
    jsBundleHbc: 6 * 1024 * 1024,       // 6MB Hermes bytecode
    assets: 10 * 1024 * 1024,           // 10MB assets
  },
  ios: {
    totalIpa: 30 * 1024 * 1024,         // 30MB IPA
    jsBundleRaw: 8 * 1024 * 1024,       // 8MB raw JS bundle
    jsBundleHbc: 6 * 1024 * 1024,       // 6MB Hermes bytecode
    assets: 12 * 1024 * 1024,           // 12MB assets
  },
};

function checkBundleSize() {
  // Generate the Android JS bundle
  execFileSync('npx', [
    'react-native', 'bundle',
    '--platform', 'android',
    '--dev', 'false',
    '--entry-file', 'index.js',
    '--bundle-output', '/tmp/android-bundle.js',
    '--assets-dest', '/tmp/android-assets',
  ], { stdio: 'inherit' });

  const bundleSize = fs.statSync('/tmp/android-bundle.js').size;
  const budget = BUDGET.android.jsBundleRaw;
  const usage = (bundleSize / budget * 100).toFixed(1);

  console.log(`\nJS Bundle Size: ${(bundleSize / 1024 / 1024).toFixed(2)}MB`);
  console.log(`Budget: ${(budget / 1024 / 1024).toFixed(2)}MB`);
  console.log(`Usage: ${usage}%`);

  if (bundleSize > budget) {
    console.error(`\n BUDGET EXCEEDED by ${((bundleSize - budget) / 1024).toFixed(0)}KB`);
    console.error('Run `npx react-native-bundle-visualizer` to find large dependencies.');
    process.exit(1);
  }

  if (bundleSize > budget * 0.9) {
    console.warn(`\n WARNING: Bundle is at ${usage}% of budget. Consider optimizing.`);
  }

  console.log('\n Bundle size within budget.');
}

checkBundleSize();
```

### Memory Budget Enforcement

Memory is harder to enforce in CI because it requires running the app on a real device or emulator. But you can set up automated tests that measure memory:

```typescript
// e2e/performance/memory-budget.test.ts
// Uses Detox or Maestro for mobile E2E performance testing

describe('Memory Budgets', () => {
  it('should stay under 200MB during typical usage', async () => {
    // Simulate a typical user session
    await device.launchApp({ newInstance: true });

    // Navigate through high-traffic screens
    await element(by.id('home-tab')).tap();
    await waitFor(element(by.id('product-list'))).toBeVisible();

    // Scroll through products (triggers image loading)
    await element(by.id('product-list')).scroll(2000, 'down');
    await element(by.id('product-list')).scroll(2000, 'down');

    // Navigate to product detail
    await element(by.id('product-card-0')).tap();
    await waitFor(element(by.id('product-detail'))).toBeVisible();

    // Go back and repeat (tests for memory leaks)
    for (let i = 0; i < 5; i++) {
      await device.pressBack();
      await element(by.id('product-list')).scroll(1000, 'down');
      await element(by.id(`product-card-${i + 1}`)).tap();
      await waitFor(element(by.id('product-detail'))).toBeVisible();
    }

    // Check memory usage
    const memoryInfo = await device.getMemoryInfo();

    expect(memoryInfo.usedMemory).toBeLessThan(200 * 1024 * 1024);

    // Check for memory growth (leak indicator)
    // Memory should not grow significantly during repeated navigation
    expect(memoryInfo.memoryGrowthRate).toBeLessThan(5 * 1024 * 1024); // < 5MB growth
  });
});
```

---

## 44.4 Web Performance Budgets

Web budgets are route-level. A marketing homepage and a dashboard app have fundamentally different performance profiles. Set budgets per route, not globally.

### Budget Per Route

```typescript
// performance-budgets.web.ts
// Route-level performance budgets for web applications

export const ROUTE_BUDGETS = {
  // Marketing / SEO pages -- must be extremely fast
  '/': {
    jsBundle: 100 * 1024,         // 100KB JS (compressed/gzipped)
    cssBundle: 30 * 1024,         // 30KB CSS
    totalPageWeight: 500 * 1024,  // 500KB total (HTML + CSS + JS + images)
    lcp: 2000,                    // 2.0s -- aim below the 2.5s threshold
    inp: 150,                     // 150ms
    cls: 0.05,                    // Very low -- no shifts on landing page
    ttfb: 200,                    // 200ms -- edge-cached or SSG
    fcp: 1200,                    // 1.2s
  },

  // Product pages -- need good SEO + fast loads
  '/products/[id]': {
    jsBundle: 150 * 1024,         // 150KB JS
    cssBundle: 40 * 1024,         // 40KB CSS
    totalPageWeight: 800 * 1024,  // 800KB total
    lcp: 2500,                    // 2.5s -- at the threshold
    inp: 200,                     // 200ms
    cls: 0.1,                     // Standard threshold
    ttfb: 300,                    // 300ms -- SSR with caching
    fcp: 1500,                    // 1.5s
  },

  // Dashboard -- authenticated, complex UI, more relaxed
  '/dashboard': {
    jsBundle: 250 * 1024,         // 250KB JS (charting, tables, etc.)
    cssBundle: 60 * 1024,         // 60KB CSS
    totalPageWeight: 1200 * 1024, // 1.2MB total
    lcp: 3000,                    // 3.0s -- authenticated, more complex
    inp: 200,                     // 200ms -- interactions must still be snappy
    cls: 0.1,                     // Standard
    ttfb: 500,                    // 500ms -- may need auth + data
    fcp: 2000,                    // 2.0s
  },

  // Checkout -- conversion critical
  '/checkout': {
    jsBundle: 120 * 1024,         // 120KB JS -- keep it lean
    cssBundle: 25 * 1024,         // 25KB CSS
    totalPageWeight: 400 * 1024,  // 400KB total -- speed = money here
    lcp: 2000,                    // 2.0s
    inp: 100,                     // 100ms -- form interactions must be instant
    cls: 0.02,                    // Near zero -- shifting on checkout = abandoned carts
    ttfb: 200,                    // 200ms
    fcp: 1000,                    // 1.0s
  },
};

// Global web budgets
export const GLOBAL_WEB_BUDGETS = {
  // Total JS across all chunks (shared + route)
  totalJsSize: 400 * 1024,            // 400KB gzipped

  // Individual chunk limits
  maxChunkSize: 150 * 1024,           // No single chunk > 150KB gzipped

  // Third-party JS
  thirdPartyJsSize: 100 * 1024,       // 100KB gzipped for all 3rd party

  // Images
  maxImageSize: 200 * 1024,           // No single image > 200KB
  maxImagesPerPage: 20,               // Lazy load beyond this

  // Fonts
  totalFontSize: 100 * 1024,          // 100KB for all font files
  maxFontFiles: 4,                     // Limit font variants

  // Total page weight
  totalPageWeight: 1500 * 1024,        // 1.5MB max for any page
};
```

### Why These Numbers

**100KB JS for the homepage** might seem aggressive. Here is the reasoning: On a mid-range mobile phone (Snapdragon 600 series) over a 4G connection, 100KB of gzipped JavaScript expands to roughly 300KB uncompressed. Parsing and executing 300KB of JavaScript on a mid-range phone takes about 200-400ms. That leaves you 2 seconds of your 2.5s LCP budget for the network round trip, server processing, and rendering.

At 300KB of gzipped JavaScript (expanding to ~900KB), parse time alone on a mid-range phone is 800ms-1.2s. You have almost no budget left for anything else. This is why code splitting per route is not optional -- it is the only way to keep individual route bundles small enough.

**CLS of 0.02 on checkout** is because layout shifts during checkout directly cause abandoned purchases. A user filling in their credit card number does not want the form to jump. Amazon found that even a 100ms delay in checkout reduced conversions by 1%. A visible layout shift is worse.

### Implementing Route-Level Bundle Budgets with size-limit

```json
// .size-limit.json
[
  {
    "name": "Homepage JS",
    "path": ".next/static/chunks/pages/index-*.js",
    "limit": "100 KB",
    "gzip": true
  },
  {
    "name": "Product Page JS",
    "path": ".next/static/chunks/pages/products/[id]-*.js",
    "limit": "150 KB",
    "gzip": true
  },
  {
    "name": "Dashboard JS",
    "path": ".next/static/chunks/pages/dashboard-*.js",
    "limit": "250 KB",
    "gzip": true
  },
  {
    "name": "Checkout JS",
    "path": ".next/static/chunks/pages/checkout-*.js",
    "limit": "120 KB",
    "gzip": true
  },
  {
    "name": "Shared Framework JS",
    "path": ".next/static/chunks/framework-*.js",
    "limit": "100 KB",
    "gzip": true
  },
  {
    "name": "Total First-Load JS (Homepage)",
    "path": [
      ".next/static/chunks/main-*.js",
      ".next/static/chunks/webpack-*.js",
      ".next/static/chunks/framework-*.js",
      ".next/static/chunks/pages/index-*.js"
    ],
    "limit": "200 KB",
    "gzip": true
  }
]
```

```json
// package.json -- add size-limit scripts
{
  "scripts": {
    "build": "next build",
    "size": "size-limit",
    "size:check": "size-limit --json > .size-limit-report.json"
  },
  "devDependencies": {
    "size-limit": "^11.0.0",
    "@size-limit/file": "^11.0.0",
    "@size-limit/webpack": "^11.0.0"
  }
}
```

---

## 44.5 Enforcing Budgets in CI

Budgets without enforcement are suggestions. Suggestions get ignored when the deadline is tomorrow. The build must fail when a budget is exceeded. No exceptions, no overrides (except explicit ones that require a comment explaining why).

### Lighthouse CI

Lighthouse CI runs Lighthouse in your CI pipeline and compares results against budgets. It can store historical results, compare across builds, and block PRs that regress.

**Installation and Configuration:**

```bash
npm install -D @lhci/cli
```

```javascript
// lighthouserc.js -- Lighthouse CI configuration
module.exports = {
  ci: {
    collect: {
      // URLs to test
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/products/example-product',
        'http://localhost:3000/checkout',
      ],

      // Start your server before testing
      startServerCommand: 'npm run start',
      startServerReadyPattern: 'ready on',
      startServerReadyTimeout: 30000,

      // Run multiple times for stability
      numberOfRuns: 3,

      // Chrome settings
      settings: {
        // Simulate a mid-range mobile device
        preset: 'desktop', // or 'perf' for mobile throttling

        // Custom throttling for more realistic mobile simulation
        throttling: {
          rttMs: 150,                    // 4G round-trip time
          throughputKbps: 1638.4,        // 4G throughput
          cpuSlowdownMultiplier: 4,      // Simulate mid-range CPU
          requestLatencyMs: 0,
          downloadThroughputKbps: 1638.4,
          uploadThroughputKbps: 675,
        },

        // Screen emulation
        screenEmulation: {
          mobile: true,
          width: 412,
          height: 823,
          deviceScaleFactor: 1.75,
        },
      },
    },

    assert: {
      // Assertions that must pass or the build fails
      assertions: {
        // Core Web Vitals
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'interactive': ['error', { maxNumericValue: 3800 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 200 }],

        // Performance score
        'categories:performance': ['error', { minScore: 0.85 }],

        // Accessibility
        'categories:accessibility': ['error', { minScore: 0.9 }],

        // Best practices
        'categories:best-practices': ['warn', { minScore: 0.9 }],

        // SEO
        'categories:seo': ['warn', { minScore: 0.9 }],

        // Additional useful audits
        'first-contentful-paint': ['warn', { maxNumericValue: 1800 }],
        'speed-index': ['warn', { maxNumericValue: 3400 }],
        'server-response-time': ['warn', { maxNumericValue: 500 }],

        // Resource budgets
        'resource-summary:script:size': ['error', { maxNumericValue: 400000 }],
        'resource-summary:stylesheet:size': ['warn', { maxNumericValue: 100000 }],
        'resource-summary:image:size': ['warn', { maxNumericValue: 500000 }],
        'resource-summary:total:size': ['error', { maxNumericValue: 1500000 }],

        // Specific audit results
        'uses-responsive-images': ['warn', { minScore: 0.9 }],
        'unused-javascript': ['warn', { maxNumericValue: 100000 }],
        'unminified-javascript': ['error', { minScore: 1 }],
        'unminified-css': ['error', { minScore: 1 }],
        'uses-text-compression': ['error', { minScore: 1 }],
      },
    },

    upload: {
      // Store results for historical comparison
      target: 'temporary-public-storage', // Free, 7-day retention

      // For self-hosted LHCI server (recommended for teams):
      // target: 'lhci',
      // serverBaseUrl: 'https://lhci.yourcompany.com',
      // token: process.env.LHCI_TOKEN,
    },
  },
};
```

**Running Lighthouse CI:**

```bash
# Build and then run LHCI
npm run build
npx lhci autorun

# Or run steps individually for more control
npx lhci collect     # Run Lighthouse
npx lhci assert      # Check against budgets
npx lhci upload      # Store results
```

### Bundle Size Checks with size-limit

size-limit is simpler than Lighthouse but laser-focused on bundle size. It is faster to run and easier to configure.

```json
// .size-limit.json -- comprehensive bundle budgets
[
  {
    "name": "Total Client JS (gzip)",
    "path": ".next/static/**/*.js",
    "limit": "400 KB",
    "gzip": true
  },
  {
    "name": "Homepage Route",
    "path": ".next/static/chunks/pages/index-*.js",
    "limit": "100 KB",
    "gzip": true
  },
  {
    "name": "Shared Components",
    "path": "packages/ui/dist/**/*.js",
    "limit": "80 KB",
    "gzip": true
  },
  {
    "name": "API Client",
    "path": "packages/api-client/dist/**/*.js",
    "limit": "30 KB",
    "gzip": true
  }
]
```

### React Native Bundle Size Checks

For React Native, you cannot use size-limit directly on the built bundle. Instead, generate the bundle and check its size:

```bash
#!/bin/bash
# scripts/check-rn-bundle-size.sh

set -e

BUDGET_MB=8  # 8MB raw JS bundle budget
BUDGET_BYTES=$((BUDGET_MB * 1024 * 1024))

echo "Generating Android JS bundle..."
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/rn-bundle.js \
  --assets-dest /tmp/rn-assets \
  --minify true

BUNDLE_SIZE=$(stat -f%z /tmp/rn-bundle.js 2>/dev/null || stat -c%s /tmp/rn-bundle.js)
BUNDLE_MB=$(echo "scale=2; $BUNDLE_SIZE / 1024 / 1024" | bc)

echo ""
echo "Bundle size: ${BUNDLE_MB}MB"
echo "Budget: ${BUDGET_MB}MB"
echo "Usage: $(echo "scale=1; $BUNDLE_SIZE * 100 / $BUDGET_BYTES" | bc)%"

if [ "$BUNDLE_SIZE" -gt "$BUDGET_BYTES" ]; then
  OVER=$(echo "scale=0; ($BUNDLE_SIZE - $BUDGET_BYTES) / 1024" | bc)
  echo ""
  echo "ERROR: Bundle exceeds budget by ${OVER}KB"
  echo "Run 'npx react-native-bundle-visualizer' to identify large dependencies."
  exit 1
fi

# Check for previous bundle size (for delta reporting)
if [ -f /tmp/rn-bundle-prev-size.txt ]; then
  PREV_SIZE=$(cat /tmp/rn-bundle-prev-size.txt)
  DELTA=$((BUNDLE_SIZE - PREV_SIZE))
  DELTA_KB=$(echo "scale=1; $DELTA / 1024" | bc)

  if [ "$DELTA" -gt 0 ]; then
    echo "Delta: +${DELTA_KB}KB from previous build"
  else
    echo "Delta: ${DELTA_KB}KB from previous build (smaller!)"
  fi
fi

echo "$BUNDLE_SIZE" > /tmp/rn-bundle-prev-size.txt
echo ""
echo "Bundle size within budget."
```

### Failing Builds on Regression

The key design principle: **budget violations are build failures, not warnings.** If you make them warnings, they will be ignored.

```yaml
# In your CI pipeline (GitHub Actions example):
- name: Check bundle size
  run: npx size-limit
  # size-limit exits with code 1 if any budget is exceeded
  # This fails the GitHub Actions step, which blocks the PR

- name: Run Lighthouse CI
  run: npx lhci autorun
  # LHCI exits with code 1 if any assertion fails
  # Combined with branch protection rules, this blocks merging
```

However, you need an escape hatch for intentional increases. Sometimes a feature genuinely requires more JavaScript. The process should be:

1. The PR fails the size check
2. The developer adds a comment to the PR explaining *why* the increase is justified
3. A performance-aware reviewer approves the size increase
4. The budget is updated in the same PR

```json
// .size-limit.json -- the budget update happens in the same PR
// Before (the PR that adds the charting feature):
// {
//   "name": "Dashboard JS",
//   "limit": "250 KB"
// }

// After (with justification in PR description):
[
  {
    "name": "Dashboard JS",
    "path": ".next/static/chunks/pages/dashboard-*.js",
    "limit": "310 KB",
    "gzip": true
  }
]
// Increased for recharts (60KB gzip). See PR #1234 for alternatives analysis.
```

This creates a paper trail. Six months later, when someone asks "why is the dashboard bundle 310KB?", you can trace it back to a specific PR with a specific justification.

---

## 44.6 RUM vs Synthetic Monitoring

You need both Real User Monitoring and synthetic testing. They answer different questions and catch different problems.

### Synthetic Testing (Lab Data)

**What it is:** Running Lighthouse or WebPageTest in a controlled environment with consistent hardware, network conditions, and configuration. You get the same results every time (within a small margin).

**When to use it:**
- CI pipelines (every PR, every deploy)
- Comparing builds (did this PR make things faster or slower?)
- Debugging specific performance issues (reproducible environment)
- Establishing baselines before launch

**Tools:**
- Lighthouse CI (in your CI pipeline)
- WebPageTest (detailed waterfall analysis)
- Chrome DevTools Performance panel (local development)
- Unlighthouse (bulk Lighthouse scans across your entire site)

**Limitations:**
- Does not reflect real user conditions (network variance, device diversity, extensions)
- Simulated throttling is not the same as real slow networks
- Cannot capture INP well (no real user interactions)
- Does not account for CDN cache behavior, geographic latency, etc.

```typescript
// Example: Running Lighthouse programmatically for custom analysis
import lighthouse from 'lighthouse';
import * as chromeLauncher from 'chrome-launcher';

async function runLighthouse(url: string) {
  const chrome = await chromeLauncher.launch({ chromeFlags: ['--headless'] });

  const result = await lighthouse(url, {
    port: chrome.port,
    output: 'json',
    onlyCategories: ['performance'],
    settings: {
      formFactor: 'mobile',
      throttling: {
        rttMs: 150,
        throughputKbps: 1638.4,
        cpuSlowdownMultiplier: 4,
      },
    },
  });

  await chrome.kill();

  const { audits, categories } = result.lhr;

  return {
    performanceScore: categories.performance.score * 100,
    lcp: audits['largest-contentful-paint'].numericValue,
    tbt: audits['total-blocking-time'].numericValue,
    cls: audits['cumulative-layout-shift'].numericValue,
    fcp: audits['first-contentful-paint'].numericValue,
    speedIndex: audits['speed-index'].numericValue,
    ttfb: audits['server-response-time'].numericValue,
  };
}

// Compare two URLs (e.g., production vs. preview deployment)
async function comparePerformance(prodUrl: string, previewUrl: string) {
  const [prod, preview] = await Promise.all([
    runLighthouse(prodUrl),
    runLighthouse(previewUrl),
  ]);

  const regressions: string[] = [];

  if (preview.lcp > prod.lcp * 1.1) {
    regressions.push(
      `LCP regressed: ${prod.lcp.toFixed(0)}ms -> ${preview.lcp.toFixed(0)}ms (+${((preview.lcp / prod.lcp - 1) * 100).toFixed(1)}%)`
    );
  }

  if (preview.cls > prod.cls + 0.05) {
    regressions.push(
      `CLS regressed: ${prod.cls.toFixed(3)} -> ${preview.cls.toFixed(3)}`
    );
  }

  return { prod, preview, regressions };
}
```

### Real User Monitoring (Field Data)

**What it is:** Collecting performance data from actual users in production. Every page load, every interaction, from every device, on every network. This is the truth about your app's performance.

**When to use it:**
- Understanding what real users experience (the whole point)
- Identifying geographic or device-specific issues
- Tracking performance over time (did the last deploy help or hurt?)
- Measuring INP accurately (requires real user interactions)
- Catching issues that synthetic testing misses (third-party script interference, CDN issues, etc.)

**Tools:**
- Vercel Speed Insights (built into Vercel, zero-config for Next.js)
- Google Chrome User Experience Report (CrUX) (anonymized real Chrome user data)
- web-vitals library (collect metrics yourself)
- Datadog RUM, Sentry Performance, New Relic Browser

**Setting up RUM with the web-vitals library:**

```typescript
// lib/web-vitals.ts
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals';

type MetricName = 'CLS' | 'INP' | 'LCP' | 'FCP' | 'TTFB';

interface PerformanceMetric {
  name: MetricName;
  value: number;
  rating: 'good' | 'needs-improvement' | 'poor';
  delta: number;
  navigationType: string;
  url: string;
  timestamp: number;
}

function sendMetric(metric: PerformanceMetric) {
  // Use sendBeacon for reliability (fires even if page is closing)
  const body = JSON.stringify(metric);

  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/analytics/web-vitals', body);
  } else {
    fetch('/api/analytics/web-vitals', {
      method: 'POST',
      body,
      keepalive: true, // Allows request to outlive the page
    });
  }
}

function createMetricHandler(name: MetricName) {
  return (metric: {
    value: number;
    rating: string;
    delta: number;
    navigationType: string;
  }) => {
    sendMetric({
      name,
      value: metric.value,
      rating: metric.rating as PerformanceMetric['rating'],
      delta: metric.delta,
      navigationType: metric.navigationType,
      url: window.location.href,
      timestamp: Date.now(),
    });
  };
}

export function initWebVitals() {
  onCLS(createMetricHandler('CLS'));
  onINP(createMetricHandler('INP'));
  onLCP(createMetricHandler('LCP'));
  onFCP(createMetricHandler('FCP'));
  onTTFB(createMetricHandler('TTFB'));
}
```

```typescript
// app/web-vitals-reporter.tsx -- Client Component for initializing web vitals
'use client';

import { useEffect } from 'react';

export function WebVitalsReporter() {
  useEffect(() => {
    import('../lib/web-vitals').then(({ initWebVitals }) => {
      initWebVitals();
    });
  }, []);

  return null;
}
```

**Setting up Vercel Speed Insights (if you deploy on Vercel):**

```typescript
// app/layout.tsx
import { SpeedInsights } from '@vercel/speed-insights/next';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {children}
        <SpeedInsights />
      </body>
    </html>
  );
}
// That is it. Vercel Speed Insights automatically collects
// CWV data from real users and displays it in the Vercel dashboard.
// Zero configuration beyond adding the component.
```

### Why You Need Both

```
+--------------------------------------------------------------------+
|                    SYNTHETIC vs RUM                                 |
|                                                                    |
|  Synthetic (Lab)              |  RUM (Field)                       |
|  -----------------------------+----------------------------------  |
|  Controlled environment       |  Real user conditions              |
|  Reproducible results         |  Statistically significant         |
|  Runs in CI on every PR       |  Runs in production                |
|  Catches regressions early    |  Shows actual user experience      |
|  Cannot measure INP well      |  Best source for INP               |
|  Simulated throttling         |  Real network conditions           |
|  Single device profile        |  Full device diversity             |
|  Fast feedback (minutes)      |  Slow feedback (hours/days)        |
|  Free                         |  Needs traffic volume              |
|                               |                                    |
|  Use for: CI gates,           |  Use for: Dashboards, alerts,      |
|  regression prevention,       |  understanding real experience,    |
|  debugging specific issues    |  geographic/device analysis        |
|                                                                    |
|  Together: Synthetic catches problems before deploy.               |
|           RUM confirms whether the fix actually helped real users.  |
+--------------------------------------------------------------------+
```

A common mistake is relying only on synthetic testing. Your Lighthouse score is 95, so performance must be great, right? Then you look at RUM data and discover that 30% of your users in India have an LCP over 6 seconds because your CDN does not have a PoP in Mumbai and the images are served from us-east-1.

Another common mistake is relying only on RUM. You notice a performance regression in production, but you cannot reproduce it locally because your development machine is too fast. Synthetic testing with throttling lets you reproduce the issue consistently.

### Building a Performance Dashboard

```typescript
// api/analytics/web-vitals/route.ts -- Server endpoint for collecting metrics
import { NextRequest, NextResponse } from 'next/server';

// Store in your preferred database
// (Postgres, ClickHouse, BigQuery are all good choices)
async function storeMetric(metric: {
  name: string;
  value: number;
  rating: string;
  url: string;
  timestamp: number;
  userAgent: string;
  country: string;
}) {
  // Example: store in a time-series table
  await db.insert('web_vitals', {
    ...metric,
    created_at: new Date(metric.timestamp),
    // Parse useful dimensions from userAgent
    device_type: parseDeviceType(metric.userAgent),
    browser: parseBrowser(metric.userAgent),
    os: parseOS(metric.userAgent),
  });
}

export async function POST(request: NextRequest) {
  const metric = await request.json();

  await storeMetric({
    ...metric,
    userAgent: request.headers.get('user-agent') || '',
    country: request.headers.get('x-vercel-ip-country') || 'unknown',
  });

  return NextResponse.json({ ok: true });
}
```

```sql
-- Example queries for a performance dashboard

-- P75 LCP by day (the metric Google uses)
SELECT
  DATE(created_at) as day,
  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY value) as p75_lcp,
  PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY value) as p50_lcp,
  COUNT(*) as samples
FROM web_vitals
WHERE name = 'LCP'
  AND created_at > NOW() - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY day;

-- CWV pass rate by route
SELECT
  url,
  COUNT(*) as total,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE rating = 'good') / COUNT(*), 1
  ) as pct_good,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE rating = 'poor') / COUNT(*), 1
  ) as pct_poor
FROM web_vitals
WHERE name = 'LCP'
  AND created_at > NOW() - INTERVAL '7 days'
GROUP BY url
ORDER BY pct_poor DESC;

-- INP by country (find geographic hotspots)
SELECT
  country,
  PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY value) as p75_inp,
  COUNT(*) as samples
FROM web_vitals
WHERE name = 'INP'
  AND created_at > NOW() - INTERVAL '7 days'
GROUP BY country
HAVING COUNT(*) > 100  -- Minimum sample size
ORDER BY p75_inp DESC;
```

---

## 44.7 Performance Regression Detection

Detecting regressions across deploys is the core purpose of performance budgets. You need to know, before the code reaches production, whether this change makes things faster or slower.

### Comparing Performance Across Deploys

The ideal workflow:

```
+--------------------------------------------------------------+
|                  PERFORMANCE REGRESSION FLOW                  |
|                                                              |
|  1. Developer opens PR                                       |
|     |                                                        |
|  2. CI builds the app                                        |
|     |                                                        |
|  3. CI deploys to preview environment                        |
|     |                                                        |
|  4. Lighthouse CI runs against preview URL                   |
|     |                                                        |
|  5. Bundle size check runs against build output              |
|     |                                                        |
|  6. Results compared against:                                |
|     +-- Fixed budgets (hard limits)                          |
|     +-- Production baseline (regression detection)           |
|     |                                                        |
|  7. Bot posts results as PR comment:                         |
|     +-- "Bundle size: 187KB -> 192KB (+5KB, +2.7%)"         |
|     +-- "LCP: 1.8s -> 1.9s (+100ms, +5.6%)"                |
|     +-- "Status: PASS (within budget)"                       |
|     |                                                        |
|  8. If budget exceeded -> build fails -> PR blocked          |
|     If within budget -> build passes -> ready for review     |
|                                                              |
+--------------------------------------------------------------+
```

### Lighthouse CI Historical Comparison

Lighthouse CI can store results and compare across builds:

```javascript
// lighthouserc.js -- with comparison enabled
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/', 'http://localhost:3000/products/1'],
      numberOfRuns: 5, // More runs = more stable comparison
      startServerCommand: 'npm run start',
    },
    assert: {
      assertions: {
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 200 }],
        'categories:performance': ['error', { minScore: 0.85 }],
      },
    },
    upload: {
      target: 'lhci',
      serverBaseUrl: 'https://lhci.yourcompany.com',
      token: process.env.LHCI_TOKEN,

      // GitHub status check integration
      githubToken: process.env.GITHUB_TOKEN,
      githubApiHost: 'https://api.github.com',
    },
  },
};
```

The LHCI server provides:

1. **Historical trends** -- LCP, CLS, TBT over time for each URL
2. **Build comparison** -- side-by-side diff of any two builds
3. **GitHub integration** -- status checks on PRs with links to comparison

### Flashlight for Mobile Render Time Regression

Flashlight (formerly Reassure) measures React Native component render times and compares across commits:

```typescript
// __perf__/ProductList.perf.tsx
import { measureRenders } from 'reassure';
import { ProductList } from '../src/screens/ProductList';

const mockProducts = generateMockProducts(100);

test('ProductList renders within budget', async () => {
  const scenario = async (screen: RenderAPI) => {
    // Simulate scroll to trigger virtualization
    const list = screen.getByTestId('product-list');
    fireEvent.scroll(list, {
      nativeEvent: {
        contentOffset: { y: 2000 },
        contentSize: { height: 10000, width: 400 },
        layoutMeasurement: { height: 800, width: 400 },
      },
    });
  };

  await measureRenders(
    <ProductList products={mockProducts} />,
    { scenario, runs: 20 }
  );
});
```

```json
// .reassurerc.js
{
  "testMatch": "**/__perf__/**/*.perf.tsx"
}
```

```bash
# Run performance tests and generate comparison
npx reassure --baseline  # Run on main branch (stores baseline)
npx reassure             # Run on PR branch (compares to baseline)
```

Flashlight generates a markdown report showing render time changes:

```
| Component      | Baseline  | Current   | Change       | Status |
|----------------|-----------|-----------|--------------|--------|
| ProductList    | 12.3ms    | 13.1ms    | +0.8ms (6%)  | PASS   |
| ProductCard    | 2.1ms     | 2.0ms     | -0.1ms (5%)  | PASS   |
| SearchBar      | 4.2ms     | 8.7ms     | +4.5ms(107%) | FAIL   |
| CheckoutForm   | 6.8ms     | 6.9ms     | +0.1ms (1%)  | PASS   |
```

### Alerting on Regressions

For production monitoring, set up alerts when RUM data shows regression:

```typescript
// lib/performance-alerts.ts
// Run this as a cron job or on a schedule

interface PerformanceAlert {
  metric: string;
  route: string;
  currentP75: number;
  previousP75: number;
  changePercent: number;
  severity: 'warning' | 'critical';
}

async function checkForRegressions(): Promise<PerformanceAlert[]> {
  const alerts: PerformanceAlert[] = [];

  const metrics = ['LCP', 'INP', 'CLS'];
  const routes = ['/', '/products', '/checkout', '/dashboard'];

  for (const metric of metrics) {
    for (const route of routes) {
      // Compare last 24h to previous 24h
      const current = await getP75(metric, route, '24h');
      const previous = await getP75(metric, route, '24h', '24h'); // offset by 24h

      if (!current || !previous) continue;

      const changePercent = ((current - previous) / previous) * 100;

      // Thresholds for alerting
      const thresholds: Record<string, { warning: number; critical: number }> = {
        LCP: { warning: 15, critical: 30 },      // % increase
        INP: { warning: 20, critical: 40 },
        CLS: { warning: 25, critical: 50 },
      };

      const threshold = thresholds[metric];

      if (changePercent > threshold.critical) {
        alerts.push({
          metric,
          route,
          currentP75: current,
          previousP75: previous,
          changePercent,
          severity: 'critical',
        });
      } else if (changePercent > threshold.warning) {
        alerts.push({
          metric,
          route,
          currentP75: current,
          previousP75: previous,
          changePercent,
          severity: 'warning',
        });
      }
    }
  }

  return alerts;
}

async function notifyRegressions() {
  const alerts = await checkForRegressions();

  if (alerts.length === 0) return;

  const criticals = alerts.filter(a => a.severity === 'critical');
  const warnings = alerts.filter(a => a.severity === 'warning');

  if (criticals.length > 0) {
    await sendSlackAlert({
      channel: '#engineering-alerts',
      text: `Performance Regression Detected (CRITICAL)\n` +
        criticals.map(a =>
          `${a.metric} on ${a.route}: ` +
          `${a.previousP75.toFixed(0)} -> ${a.currentP75.toFixed(0)} ` +
          `(+${a.changePercent.toFixed(1)}%)`
        ).join('\n'),
    });
  }

  if (warnings.length > 0) {
    await sendSlackAlert({
      channel: '#performance',
      text: `Performance Regression Warning\n` +
        warnings.map(a =>
          `${a.metric} on ${a.route}: ` +
          `${a.previousP75.toFixed(0)} -> ${a.currentP75.toFixed(0)} ` +
          `(+${a.changePercent.toFixed(1)}%)`
        ).join('\n'),
    });
  }
}
```

---

## 44.8 Bundle Analysis

When a budget check fails, you need to figure out *what* is eating your bundle. This is where bundle analysis tools come in.

### webpack-bundle-analyzer

The classic tool for webpack-based projects (including Next.js with webpack):

```bash
# For Next.js (using webpack)
npm install -D @next/bundle-analyzer

# For standalone webpack projects
npm install -D webpack-bundle-analyzer
```

```typescript
// next.config.ts -- with bundle analyzer
import type { NextConfig } from 'next';
import bundleAnalyzer from '@next/bundle-analyzer';

const withBundleAnalyzer = bundleAnalyzer({
  enabled: process.env.ANALYZE === 'true',
  openAnalyzer: true, // Opens the treemap visualization in your browser
});

const nextConfig: NextConfig = {
  // ... your config
};

export default withBundleAnalyzer(nextConfig);
```

```bash
# Run the analyzer
ANALYZE=true npm run build
# Opens an interactive treemap showing every module and its size
```

The treemap visualization shows:

```
+------------------------------------------------------------------+
| Total Bundle: 387KB (gzipped)                                    |
|                                                                  |
| +----------------------+ +-----------------+ +---------------+   |
| | node_modules (262KB) | | src (98KB)      | | other (27KB)  |  |
| |                      | |                 | |               |   |
| | +----------+ +-----+ | | +-----+ +-----+ | | +---+ +-----+ | |
| | | recharts | |react | | | |pages| |comps| | | |css| |fonts| | |
| | |  (89KB)  | |dom   | | | |     | |     | | | |   | |     | | |
| | |          | |(42KB)| | | |(52K)| |(31K)| | | |(8K| |(12K)| | |
| | |  <- HERE | |      | | | |     | |     | | | |)  | |     | | |
| | +----------+ +------+ | | +-----+ +-----+ | | +---+ +-----+ | |
| | +------+ +--------+   | | +-------------+  | |               | |
| | |lodash| |date-fns|   | | |   utils     |  | |               | |
| | |(72KB)| | (38KB) |   | | |   (15KB)    |  | |               | |
| | +------+ +--------+   | | +-------------+  | |               | |
| +------------------------+ +-----------------+ +---------------+ |
|                                                                  |
| RED FLAGS: lodash (72KB) -- use lodash-es or native methods      |
|           recharts (89KB) -- consider lighter alternative        |
|           date-fns (38KB) -- check if all locales are imported   |
+------------------------------------------------------------------+
```

### react-native-bundle-visualizer

The equivalent for React Native:

```bash
npx react-native-bundle-visualizer

# Or for Expo:
npx react-native-bundle-visualizer --expo
```

This generates a similar treemap for your React Native bundle, showing which native modules and JS libraries are taking space.

### Finding the Large Dependencies

After visualizing, here is the systematic approach to reducing bundle size:

```typescript
// scripts/analyze-dependencies.ts
// Identify the biggest dependencies and suggest alternatives

const KNOWN_ALTERNATIVES: Record<string, string> = {
  'moment':
    'Use date-fns or dayjs (87% smaller). Or use Intl.DateTimeFormat for formatting.',
  'lodash':
    'Use lodash-es with tree shaking, or native Array/Object methods (90% smaller).',
  'axios':
    'Use native fetch (0KB). Axios is 14KB gzipped for what fetch does natively.',
  'uuid':
    'Use crypto.randomUUID() (0KB). Built into all modern runtimes.',
  'classnames':
    'Use clsx (0.5KB vs 1.8KB) or template literals (0KB).',
  'underscore':
    'Use native methods. Underscore is a 2010s library.',
  'numeral':
    'Use Intl.NumberFormat (0KB). Built into all modern runtimes.',
  'chalk':
    'Only needed in Node.js scripts, not client bundles. Check your imports.',
  'validator':
    'Use zod for validation (13KB vs 48KB). Better TypeScript support too.',
};

function checkDependencies() {
  // Read package.json to get production dependencies
  const fs = require('fs');
  const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf-8'));

  const deps = Object.keys(packageJson.dependencies || {});

  console.log('Checking dependency sizes...\n');
  console.log('Top dependencies with known lighter alternatives:\n');

  for (const dep of deps) {
    if (KNOWN_ALTERNATIVES[dep]) {
      console.log(`  ${dep}`);
      console.log(`    -> ${KNOWN_ALTERNATIVES[dep]}`);
      console.log('');
    }
  }

  console.log(
    'For full size analysis, run: ANALYZE=true npm run build'
  );
  console.log(
    'Or check sizes at: https://bundlephobia.com'
  );
}

checkDependencies();
```

### Tree Shaking Verification

Tree shaking eliminates unused code from your bundle. But it only works under specific conditions:

1. The library must use ES modules (`import`/`export`), not CommonJS (`require`/`module.exports`)
2. The code must be free of side effects (or properly marked with `sideEffects: false` in package.json)
3. Your bundler must be configured to tree shake

```typescript
// Verify tree shaking is working for a specific dependency

// BAD: This imports the entire lodash library (72KB)
import { debounce } from 'lodash';

// GOOD: This imports only debounce (~1KB)
import debounce from 'lodash/debounce';

// BEST: This tree-shakes correctly with lodash-es
import { debounce } from 'lodash-es';

// Check if your imports are tree-shakeable:
// 1. Run your build with ANALYZE=true
// 2. Search for the library in the treemap
// 3. If you see the ENTIRE library, tree shaking is not working
// 4. Common fix: switch from CJS version to ESM version
//    - lodash -> lodash-es
//    - aws-sdk -> @aws-sdk/client-* (v3)
//    - rxjs -> rxjs (v7+ is ESM)
```

```javascript
// next.config.js -- verify tree shaking with webpack stats
module.exports = {
  webpack: (config, { isServer }) => {
    if (!isServer && process.env.ANALYZE_TREE_SHAKING === 'true') {
      config.optimization.usedExports = true;
      config.optimization.sideEffects = true;

      // Log which modules are tree-shaken
      config.stats = {
        ...config.stats,
        usedExports: true,
        optimizationBailout: true, // Shows WHY tree shaking failed
      };
    }
    return config;
  },
};
```

### Common Bundle Bloat Patterns

```
+------------------------------------------------------------------+
|              COMMON BUNDLE BLOAT PATTERNS                        |
|                                                                  |
|  Pattern                     |  Size Impact  |  Fix              |
|  ----------------------------+--------------+------------------  |
|  moment.js with all locales  |  +232KB       |  date-fns / dayjs |
|  Full lodash import          |  +72KB        |  lodash-es / ESM  |
|  All of @mui/icons-material  |  +100KB+      |  Named imports    |
|  Unminified dev build        |  +200%        |  Check NODE_ENV   |
|  Source maps in production   |  +300%        |  devtool: false   |
|  Polyfills for dead browsers |  +50KB        |  Update targets   |
|  Duplicate React versions    |  +40KB        |  Dedupe in lock   |
|  Full Intl polyfill          |  +200KB       |  Use runtime Intl |
|  CSS-in-JS runtime           |  +30-50KB     |  Tailwind or CSS  |
|  Full AWS SDK v2             |  +500KB       |  @aws-sdk v3      |
|                                                                  |
|  Pro tip: Run `npx depcheck` to find unused dependencies.        |
|  Remove them. They might still end up in your bundle.            |
+------------------------------------------------------------------+
```

---

## 44.9 Performance Culture

Tools and CI checks are necessary but not sufficient. If the team does not *care* about performance, they will find ways around the checks. They will bump the budget numbers without analysis. They will skip the performance section of the PR template. They will not look at the RUM dashboard.

Performance culture is what makes the difference between "we have performance checks" and "we have a fast app."

### Making Performance Everyone's Job

The biggest anti-pattern is having a "performance person" -- one engineer who is responsible for performance. What happens? Everyone else stops thinking about performance because "that is Sarah's job." When Sarah goes on vacation or leaves the company, performance degrades rapidly.

Instead, make performance a shared responsibility:

**1. Performance Section in PR Template**

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Changes
<!-- Describe what this PR does -->

## Performance Impact
<!-- Required for all PRs. Check all that apply: -->

- [ ] This PR adds new dependencies (list them and their sizes below)
- [ ] This PR adds new API calls on page load
- [ ] This PR adds new images/media
- [ ] This PR changes a component rendered on every page (layout, nav, etc.)
- [ ] This PR has no meaningful performance impact

### If any boxes are checked above:
- **New dependencies:** <!-- name (size gzipped) -- why needed -- alternatives considered -->
- **Bundle size change:** <!-- will be auto-filled by CI bot -->
- **Expected CWV impact:** <!-- any expected change to LCP, INP, CLS? -->

## Testing
- [ ] Tested on low-end device / throttled connection
- [ ] Checked bundle size impact (CI will verify)
```

**2. Performance Review in Code Review**

Train your team to ask these questions in code review:

```
Performance Code Review Checklist:

LOADING
[ ] Does this add JS to the critical path?
[ ] Could this component be lazy-loaded?
[ ] Are images optimized and properly sized?
[ ] Is there a loading state that will not cause CLS?

RUNTIME
[ ] Does this cause unnecessary re-renders?
[ ] Is there work on the main thread that could be deferred?
[ ] Are expensive computations memoized where appropriate?
[ ] Is the list virtualized if it could be long?

DEPENDENCIES
[ ] Is the new dependency necessary? Could we use a native API?
[ ] Is it tree-shakeable? Check the package.json for "module" or "exports".
[ ] What is the gzipped size? (Check bundlephobia.com)
[ ] Are we importing the minimum needed?

DATA
[ ] Are we fetching more data than we display?
[ ] Is the API response cached appropriately?
[ ] Is there pagination for large datasets?
```

**3. Weekly Performance Check-ins**

Add a 15-minute performance review to your weekly team meeting:

```
Weekly Performance Review Agenda (15 minutes):

1. DASHBOARD CHECK (5 min)
   - Review P75 LCP, INP, CLS for key routes
   - Compare to last week
   - Flag any route that regressed > 10%

2. BUNDLE SIZE TREND (3 min)
   - Total bundle size this week vs. last week
   - Any new dependencies added?
   - Any dependencies removed or replaced?

3. REGRESSIONS (5 min)
   - Review any performance alerts from the past week
   - Root cause of any regression
   - Action items assigned

4. UPCOMING RISKS (2 min)
   - Any large features about to land that might affect performance?
   - Proactive planning (code splitting, lazy loading)
```

**4. Performance OKRs**

If performance matters to the business (and it always does for user-facing applications), it should be in the team's OKRs:

```
Example Performance OKRs:

Objective: Deliver a fast, responsive user experience
  KR1: P75 LCP under 2.0s for all marketing pages (currently 2.8s)
  KR2: P75 INP under 150ms across all routes (currently 220ms)
  KR3: Total JS bundle under 300KB gzipped (currently 387KB)
  KR4: Zero critical performance regressions shipped to production

Objective: Build performance into our development process
  KR1: 100% of PRs pass automated performance checks before merge
  KR2: Bundle size delta posted on every PR
  KR3: Performance dashboard reviewed weekly in team meeting
  KR4: All engineers complete performance training module
```

**5. Performance Error Budget**

Borrow from SRE's concept of error budgets. Allocate a "performance budget" for the quarter:

```
Q2 Performance Budget:

Starting LCP: 1.8s
Maximum allowed LCP regression: 0.5s (to 2.3s)
Remaining budget: 0.5s

Each PR that regresses LCP "spends" from this budget.
When the budget is exhausted, no new features until performance
is improved back to baseline.

This prevents the "death by a thousand cuts" problem.
```

### What to Do When Performance Culture Fails

Sometimes, despite your best efforts, a team will override the checks and ship a slow feature. When that happens:

1. **Do not blame.** The person who shipped the slow code was probably under deadline pressure. Fix the process, not the person.

2. **Do a performance retrospective.** Ask: Why did the checks not catch this? Was the budget too generous? Did someone bypass CI? Was the RUM alerting too slow?

3. **Tighten the budget.** If the team keeps hitting the budget ceiling, the budget is probably too generous. Tighten it so there is less room for drift.

4. **Make performance visible.** Put the RUM dashboard on a TV in the office. Show LCP next to revenue on the business dashboard. When people see the correlation between performance and business metrics, they start caring.

5. **Celebrate improvements.** When someone reduces the bundle by 50KB or improves LCP by 300ms, call it out in the team channel. Performance work is often invisible and thankless. Make it visible and thanked.

---

## 44.10 Complete CI Setup

This is the whole thing. A complete GitHub Actions workflow that checks bundle size, runs Lighthouse CI, compares against budgets, and posts results as a PR comment.

### Project Configuration Files

First, the configuration files that the CI workflow references:

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/products/example',
        'http://localhost:3000/checkout',
      ],
      startServerCommand: 'npm run start',
      startServerReadyPattern: 'ready on',
      startServerReadyTimeout: 30000,
      numberOfRuns: 3,
      settings: {
        preset: 'desktop',
        throttling: {
          rttMs: 150,
          throughputKbps: 1638.4,
          cpuSlowdownMultiplier: 4,
        },
        screenEmulation: {
          mobile: true,
          width: 412,
          height: 823,
          deviceScaleFactor: 1.75,
        },
      },
    },
    assert: {
      assertions: {
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 200 }],
        'categories:performance': ['error', { minScore: 0.85 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['warn', { maxNumericValue: 1800 }],
        'speed-index': ['warn', { maxNumericValue: 3400 }],
        'resource-summary:script:size': ['error', { maxNumericValue: 400000 }],
        'resource-summary:total:size': ['error', { maxNumericValue: 1500000 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

```json
// .size-limit.json
[
  {
    "name": "Homepage JS",
    "path": ".next/static/chunks/pages/index-*.js",
    "limit": "100 KB",
    "gzip": true
  },
  {
    "name": "Product Page JS",
    "path": ".next/static/chunks/pages/products/[id]-*.js",
    "limit": "150 KB",
    "gzip": true
  },
  {
    "name": "Checkout JS",
    "path": ".next/static/chunks/pages/checkout-*.js",
    "limit": "120 KB",
    "gzip": true
  },
  {
    "name": "Shared Framework",
    "path": ".next/static/chunks/framework-*.js",
    "limit": "100 KB",
    "gzip": true
  },
  {
    "name": "Total First-Load JS",
    "path": [
      ".next/static/chunks/main-*.js",
      ".next/static/chunks/webpack-*.js",
      ".next/static/chunks/framework-*.js",
      ".next/static/chunks/pages/index-*.js"
    ],
    "limit": "200 KB",
    "gzip": true
  }
]
```

### The Complete GitHub Actions Workflow

```yaml
# .github/workflows/performance.yml
# Complete performance budget enforcement workflow
# Runs on every PR and posts results as a comment

name: Performance Budgets

on:
  pull_request:
    branches: [main]
    # Only run when code or config changes -- skip docs-only PRs
    paths:
      - 'apps/**'
      - 'packages/**'
      - 'package.json'
      - 'pnpm-lock.yaml'
      - '.size-limit.json'
      - 'lighthouserc.js'

concurrency:
  # Cancel superseded runs for the same PR
  group: perf-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  # ---------------------------------------------------
  # Job 1: Bundle Size Check
  # ---------------------------------------------------
  bundle-size:
    name: Bundle Size
    runs-on: ubuntu-latest
    timeout-minutes: 15
    outputs:
      comment: ${{ steps.size-comparison.outputs.comment }}

    steps:
      - name: Checkout PR
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build
        run: pnpm build
        env:
          NODE_ENV: production
          NEXT_TELEMETRY_DISABLED: 1

      - name: Check bundle size against budgets
        run: pnpm size-limit --json > size-limit-report.json
        continue-on-error: true
        id: size-check

      - name: Upload size report
        uses: actions/upload-artifact@v4
        with:
          name: size-limit-report
          path: size-limit-report.json

      # Compare against main branch
      - name: Checkout main for comparison
        uses: actions/checkout@v4
        with:
          ref: main
          path: main-branch

      - name: Build main branch
        run: |
          cd main-branch
          pnpm install --frozen-lockfile
          pnpm build
          pnpm size-limit --json > ../size-limit-main.json || true
        env:
          NODE_ENV: production
          NEXT_TELEMETRY_DISABLED: 1

      - name: Generate size comparison
        uses: actions/github-script@v7
        id: size-comparison
        with:
          script: |
            const fs = require('fs');

            let current, baseline;
            try {
              current = JSON.parse(fs.readFileSync('size-limit-report.json', 'utf-8'));
              baseline = JSON.parse(fs.readFileSync('size-limit-main.json', 'utf-8'));
            } catch (e) {
              core.setOutput('comment', '**Bundle Size:** Unable to generate comparison.');
              return;
            }

            let table = '| Bundle | Main | PR | Delta | Status |\n';
            table += '|--------|------|----|-------|--------|\n';

            let anyFailed = false;

            for (let i = 0; i < current.length; i++) {
              const curr = current[i];
              const base = baseline[i] || { size: 0 };

              const currKB = (curr.size / 1024).toFixed(1);
              const baseKB = (base.size / 1024).toFixed(1);
              const deltaBytes = curr.size - (base.size || 0);
              const deltaKB = (deltaBytes / 1024).toFixed(1);
              const deltaPercent = base.size
                ? ((deltaBytes / base.size) * 100).toFixed(1)
                : 'N/A';

              let status = 'PASS';
              if (curr.passed === false) {
                status = 'OVER BUDGET';
                anyFailed = true;
              } else if (deltaBytes > 5120) {
                status = 'WARNING +' + deltaKB + 'KB';
              }

              const sign = deltaBytes >= 0 ? '+' : '';
              table += `| ${curr.name} | ${baseKB}KB | ${currKB}KB | ${sign}${deltaKB}KB (${sign}${deltaPercent}%) | ${status} |\n`;
            }

            const header = anyFailed
              ? '### Bundle Size -- Budget Exceeded\n\n'
              : '### Bundle Size -- Within Budget\n\n';

            core.setOutput('comment', header + table);
            core.setOutput('failed', anyFailed);

      - name: Fail if budget exceeded
        if: steps.size-check.outcome == 'failure'
        run: |
          echo "Bundle size budget exceeded. See the PR comment for details."
          exit 1

  # ---------------------------------------------------
  # Job 2: Lighthouse CI
  # ---------------------------------------------------
  lighthouse:
    name: Lighthouse
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      comment: ${{ steps.lighthouse-results.outputs.comment }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build
        run: pnpm build
        env:
          NODE_ENV: production
          NEXT_TELEMETRY_DISABLED: 1

      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun 2>&1 | tee lighthouse-output.txt
        continue-on-error: true
        id: lhci
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Parse Lighthouse results
        uses: actions/github-script@v7
        id: lighthouse-results
        with:
          script: |
            const fs = require('fs');
            const path = require('path');

            // Find the latest Lighthouse JSON result
            const resultsDir = '.lighthouseci';
            let results = [];

            try {
              const files = fs.readdirSync(resultsDir)
                .filter(f => f.endsWith('.json') && !f.startsWith('manifest'));

              for (const file of files) {
                const data = JSON.parse(
                  fs.readFileSync(path.join(resultsDir, file), 'utf-8')
                );
                if (!data.categories) continue;
                results.push({
                  url: data.finalUrl || data.requestedUrl,
                  performance: (data.categories.performance.score * 100).toFixed(0),
                  accessibility: (data.categories.accessibility.score * 100).toFixed(0),
                  bestPractices: (data.categories['best-practices'].score * 100).toFixed(0),
                  seo: (data.categories.seo.score * 100).toFixed(0),
                  lcp: data.audits['largest-contentful-paint'].numericValue.toFixed(0),
                  tbt: data.audits['total-blocking-time'].numericValue.toFixed(0),
                  cls: data.audits['cumulative-layout-shift'].numericValue.toFixed(3),
                  fcp: data.audits['first-contentful-paint'].numericValue.toFixed(0),
                  si: data.audits['speed-index'].numericValue.toFixed(0),
                });
              }
            } catch (e) {
              core.setOutput('comment', '### Lighthouse\nUnable to parse results.');
              return;
            }

            if (results.length === 0) {
              core.setOutput('comment', '### Lighthouse\nNo results found.');
              return;
            }

            let comment = '### Lighthouse Results\n\n';
            comment += '| URL | Perf | A11y | BP | SEO | LCP | TBT | CLS |\n';
            comment += '|-----|------|------|----|-----|-----|-----|-----|\n';

            // Deduplicate by URL (take the median run)
            const byUrl = {};
            for (const r of results) {
              const urlPath = new URL(r.url).pathname;
              if (!byUrl[urlPath]) byUrl[urlPath] = [];
              byUrl[urlPath].push(r);
            }

            let anyFailed = false;

            for (const [urlPath, runs] of Object.entries(byUrl)) {
              // Take the median result
              runs.sort((a, b) => Number(a.performance) - Number(b.performance));
              const median = runs[Math.floor(runs.length / 2)];

              const perfScore = Number(median.performance);
              const perfLabel = perfScore >= 90 ? 'GOOD'
                : perfScore >= 70 ? 'OK' : 'POOR';

              if (perfScore < 85) anyFailed = true;
              if (Number(median.lcp) > 2500) anyFailed = true;

              comment += `| ${urlPath} | ${perfLabel} ${median.performance} | ${median.accessibility} | ${median.bestPractices} | ${median.seo} | ${median.lcp}ms | ${median.tbt}ms | ${median.cls} |\n`;
            }

            comment += '\n';

            if (anyFailed) {
              comment += '> One or more pages failed performance thresholds. See Lighthouse CI output for details.\n';
            }

            core.setOutput('comment', comment);
            core.setOutput('failed', anyFailed);

      - name: Upload Lighthouse artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: lighthouse-results
          path: .lighthouseci/

      - name: Fail if Lighthouse assertions failed
        if: steps.lhci.outcome == 'failure'
        run: |
          echo "Lighthouse CI assertions failed. See the PR comment and artifacts for details."
          exit 1

  # ---------------------------------------------------
  # Job 3: React Native Bundle Size (if applicable)
  # ---------------------------------------------------
  mobile-bundle:
    name: Mobile Bundle Size
    runs-on: ubuntu-latest
    timeout-minutes: 15
    outputs:
      comment: ${{ steps.mobile-size.outputs.comment }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Generate React Native bundle
        run: |
          cd apps/mobile
          npx react-native bundle \
            --platform android \
            --dev false \
            --entry-file index.js \
            --bundle-output /tmp/rn-bundle.js \
            --assets-dest /tmp/rn-assets \
            --minify true

      - name: Check mobile bundle size
        uses: actions/github-script@v7
        id: mobile-size
        with:
          script: |
            const fs = require('fs');

            const BUDGET = 8 * 1024 * 1024; // 8MB
            const bundleSize = fs.statSync('/tmp/rn-bundle.js').size;
            const budgetMB = (BUDGET / 1024 / 1024).toFixed(1);
            const sizeMB = (bundleSize / 1024 / 1024).toFixed(2);
            const usage = ((bundleSize / BUDGET) * 100).toFixed(1);

            const passed = bundleSize <= BUDGET;
            const status = passed ? 'PASS' : 'FAIL';

            const comment = `### Mobile Bundle Size -- ${status}\n\n` +
              `| Metric | Value |\n` +
              `|--------|-------|\n` +
              `| Bundle Size | ${sizeMB}MB |\n` +
              `| Budget | ${budgetMB}MB |\n` +
              `| Usage | ${usage}% |\n`;

            core.setOutput('comment', comment);

            if (!passed) {
              core.setFailed(
                `Mobile bundle (${sizeMB}MB) exceeds budget (${budgetMB}MB)`
              );
            }

  # ---------------------------------------------------
  # Job 4: Post Combined PR Comment
  # ---------------------------------------------------
  comment:
    name: Post Results
    runs-on: ubuntu-latest
    needs: [bundle-size, lighthouse, mobile-bundle]
    if: always() && github.event_name == 'pull_request'

    permissions:
      pull-requests: write

    steps:
      - name: Post performance report as PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const bundleComment = '${{ needs.bundle-size.outputs.comment }}'
              .replace(/'/g, "'");
            const lighthouseComment = '${{ needs.lighthouse.outputs.comment }}'
              .replace(/'/g, "'");
            const mobileComment = '${{ needs.mobile-bundle.outputs.comment }}'
              .replace(/'/g, "'");

            let body = '## Performance Report\n\n';

            if (bundleComment) body += bundleComment + '\n\n';
            if (lighthouseComment) body += lighthouseComment + '\n\n';
            if (mobileComment) body += mobileComment + '\n\n';

            body += '---\n';
            body += '*Generated by Performance Budget CI*\n';

            // Find existing comment to update (avoid spamming the PR)
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            const botComment = comments.find(c =>
              c.user.type === 'Bot' &&
              c.body.includes('## Performance Report')
            );

            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body,
              });
            }
```

### What the PR Comment Looks Like

When this workflow runs on a PR, the developer sees a comment like this:

```
## Performance Report

### Bundle Size -- Within Budget

| Bundle            | Main    | PR      | Delta          | Status |
|-------------------|---------|---------|----------------|--------|
| Homepage JS       | 87.3KB  | 89.1KB  | +1.8KB (+2.1%) | PASS   |
| Product Page JS   | 132.5KB | 131.9KB | -0.6KB (-0.5%) | PASS   |
| Checkout JS       | 98.2KB  | 98.2KB  | +0.0KB (+0.0%) | PASS   |
| Shared Framework  | 89.7KB  | 89.7KB  | +0.0KB (+0.0%) | PASS   |
| Total First-Load  | 178.2KB | 180.0KB | +1.8KB (+1.0%) | PASS   |

### Lighthouse Results

| URL        | Perf     | A11y | BP  | SEO | LCP    | TBT   | CLS   |
|------------|----------|------|-----|-----|--------|-------|-------|
| /          | GOOD 94  | 98   | 100 | 100 | 1420ms | 120ms | 0.002 |
| /products  | GOOD 91  | 96   | 100 | 98  | 1890ms | 180ms | 0.015 |
| /checkout  | GOOD 93  | 100  | 100 | 95  | 1650ms | 95ms  | 0.000 |

### Mobile Bundle Size -- PASS

| Metric      | Value  |
|-------------|--------|
| Bundle Size | 6.23MB |
| Budget      | 8.0MB  |
| Usage       | 77.9%  |

---
*Generated by Performance Budget CI*
```

And if a budget is exceeded:

```
## Performance Report

### Bundle Size -- Budget Exceeded

| Bundle            | Main    | PR       | Delta            | Status      |
|-------------------|---------|----------|------------------|-------------|
| Homepage JS       | 87.3KB  | 142.8KB  | +55.5KB (+63.6%) | OVER BUDGET |
| Product Page JS   | 132.5KB | 131.9KB  | -0.6KB (-0.5%)   | PASS        |
| Checkout JS       | 98.2KB  | 98.2KB   | +0.0KB (+0.0%)   | PASS        |
| Shared Framework  | 89.7KB  | 89.7KB   | +0.0KB (+0.0%)   | PASS        |
| Total First-Load  | 178.2KB | 233.7KB  | +55.5KB (+31.1%) | OVER BUDGET |

> Homepage JS exceeds its 100KB budget (142.8KB).
> Total First-Load JS exceeds its 200KB budget (233.7KB).
>
> Action required: Reduce bundle size or update the budget with justification.
> Run `ANALYZE=true pnpm build` to identify large dependencies.
```

The build fails. The PR is blocked. The developer has clear information about what went wrong and how to investigate.

### Branch Protection Rules

The workflow alone is not enough -- you need branch protection rules to prevent merging when it fails:

```
Repository Settings -> Branches -> Branch protection rules -> main:

[x] Require status checks to pass before merging
  [x] Require branches to be up to date before merging

  Required status checks:
  [x] Bundle Size
  [x] Lighthouse
  [x] Mobile Bundle Size (if applicable)
```

---

## 44.11 The Escape Hatch — Intentional Budget Increases

Not every budget violation is a problem. Sometimes a feature genuinely needs more JavaScript. The process for intentional increases:

```
+--------------------------------------------------------------+
|               BUDGET INCREASE PROCESS                        |
|                                                              |
|  1. PR fails bundle size check                               |
|     |                                                        |
|  2. Developer investigates:                                  |
|     +-- Is this a real increase or a measurement artifact?   |
|     +-- Can the new code be lazy-loaded?                     |
|     +-- Can an existing dependency be replaced?              |
|     +-- Is the dependency tree-shakeable?                    |
|     |                                                        |
|  3. If the increase is justified:                            |
|     +-- Update .size-limit.json in the same PR              |
|     +-- Add a comment in the config explaining WHY          |
|     +-- Add an analysis to the PR description:              |
|         * What was added and why                             |
|         * Alternatives considered                            |
|         * Bundle analyzer screenshot                         |
|         * Plan to offset the increase (if any)              |
|     |                                                        |
|  4. Reviewer with performance expertise approves             |
|     |                                                        |
|  5. PR merges with the updated budget                        |
|                                                              |
|  Paper trail: Every budget increase is traceable to a PR     |
|  with a justification. No silent drift.                      |
+--------------------------------------------------------------+
```

### Documenting Budget Decisions

JSON does not support comments, so store the documentation in a companion file:

```typescript
// .size-limit-docs.ts -- companion documentation for budget decisions
export const BUDGET_DECISIONS = {
  'Dashboard JS': {
    currentLimit: '310 KB',
    history: [
      {
        date: '2026-01-15',
        pr: '#987',
        limit: '200 KB',
        reason: 'Initial budget based on route complexity analysis',
      },
      {
        date: '2026-02-20',
        pr: '#1089',
        limit: '250 KB',
        reason:
          'Added data table with sorting/filtering (ag-grid-react, 50KB)',
      },
      {
        date: '2026-03-10',
        pr: '#1234',
        limit: '310 KB',
        reason:
          'Added recharts for analytics charts (60KB). ' +
          'Evaluated Chart.js (45KB, worse React DX) and raw D3 (80KB+ estimated).',
      },
    ],
  },
};
```

This creates a paper trail. Six months later, when someone asks "why is the dashboard bundle 310KB?", you can trace it back to a specific PR with a specific justification. And when someone proposes removing recharts, they can see exactly which PR introduced it and what alternatives were already evaluated.

---

## Key Takeaways

1. **Performance is a ratchet.** Without enforced budgets, every sprint makes your app slower. The degradation is invisible in individual PRs and devastating over months. Automate the enforcement.

2. **Know your Core Web Vitals.** LCP (<2.5s) measures loading speed. INP (<200ms) measures interactivity. CLS (<0.1) measures visual stability. Optimize each differently. Measure at the 75th percentile.

3. **Set budgets per route, not globally.** Your homepage and your admin dashboard have different performance profiles. A global "400KB JS" budget is too coarse. Per-route budgets catch regressions that global budgets miss.

4. **Make CI the enforcer.** The build must fail when a budget is exceeded. Warnings get ignored. Use Lighthouse CI for web performance, size-limit for bundle sizes, and custom scripts for mobile bundle sizes.

5. **You need both synthetic and RUM.** Synthetic testing (Lighthouse in CI) catches regressions before they ship. Real User Monitoring catches problems that synthetic testing misses (geographic issues, device diversity, third-party script interference). Neither alone is sufficient.

6. **Bundle analysis is your debugging tool.** When a budget check fails, webpack-bundle-analyzer (or its equivalents) shows you exactly what is eating your bundle. Most bloat comes from a few large dependencies that could be replaced, tree-shaken, or lazy-loaded.

7. **Performance culture beats performance tools.** The CI checks are necessary but not sufficient. If the team does not care about performance -- if there are no performance OKRs, no weekly dashboard reviews, no celebration of improvements -- the checks will be bypassed or the budgets will be inflated until they are meaningless.

8. **Document every budget increase.** When a budget is intentionally raised, the justification must be in the PR. Six months later, you need to know why the dashboard budget is 310KB and whether the offset plan was ever executed. Without paper trails, budgets inflate silently.

The difference between a team that has a fast app and a team that used to have a fast app is not talent or technology. It is systems. Build the systems -- the budgets, the CI checks, the dashboards, the culture -- and performance takes care of itself. Skip the systems and you will be doing a "performance sprint" every quarter, fighting the same entropy you could have prevented with a six-line size-limit config.

---

*Next up: Chapter 45 — Observability & Error Tracking, where we cover how to monitor application health in production, from error tracking with Sentry to logging, alerting, and building dashboards that tell you what is broken before your users do.*
