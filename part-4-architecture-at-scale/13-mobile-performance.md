<!--
  CHAPTER: 14
  TITLE: Performance Optimization — Mobile
  PART: IV — Architecture at Scale
  PREREQS: Chapters 1, 3
  KEY_TOPICS: cold start, FlashList, virtualization, bundle size, memory management, React Compiler, re-renders, animations, Reanimated, image optimization, target metrics
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 13: Performance Optimization — Mobile

> "Users don't care about your architecture. They care that the app opens fast, scrolls smooth, and doesn't drain their battery."
> — Every PM you've ever worked with, and they're right

---

<details>
<summary><strong>TL;DR</strong></summary>

- Target cold start under 1.5s, 60fps scrolling, JS thread frame time under 16ms, and memory under 200MB; these are not aspirational -- they are the floor for a production mobile app
- FlashList replaces FlatList for virtualized lists; it recycles cell components instead of mounting/unmounting, cutting list rendering time dramatically on long lists
- Bundle size directly affects cold start and app store conversion; set a 15MB compressed budget, measure it in CI, and block PRs that exceed it
- The React Compiler auto-memoizes renders, but you still need to understand re-render triggers to avoid unnecessary work in hot paths
- Always profile on a low-end Android device (not your development iPhone); the 10x hardware gap between a $50 phone and a $1200 phone is where your performance bugs live

</details>

Let me tell you about a moment that changed how I think about mobile performance. We shipped a feature-complete React Native app to production. Beautiful UI. Clean architecture. Comprehensive test coverage. The client loved it in demo.

Then real users opened it on a three-year-old Samsung Galaxy A13 on a 3G connection in Jakarta. Cold start: 4.7 seconds. Scrolling through the product list felt like dragging a boulder through mud. The app crashed after 15 minutes of browsing because it ate 380MB of RAM.

We had built a desktop app wearing a mobile costume.

This chapter is about not making that mistake. We are going to go deep on every axis of mobile performance — not just the "what" but the "why it matters at a physics level" and the "how companies that serve hundreds of millions of users actually solve it."

By the end, you will have a mental model for mobile performance that goes beyond tips and tricks. You will understand the constraints of the runtime, the hardware, and the user — and you will know exactly which levers to pull.

---

## 13.1 Target Metrics — What "Fast" Actually Means

Before you optimize anything, you need numbers. Not vibes. Not "it feels snappy." Numbers.

Here is the performance budget I recommend for any serious React Native application in 2026:

| Metric | Target | Why This Number |
|--------|--------|-----------------|
| **Cold Start** | < 1.5s | Users perceive >2s as "broken." 1.5s gives you headroom. |
| **Frame Rate** | 60fps (16.6ms per frame) | The human eye perceives jank at ~45fps. 60fps is the floor. |
| **Time to Interactive (TTI)** | < 2s | The screen renders, but can the user actually *do* anything? |
| **Bundle Size** | < 15MB (compressed) | App store download thresholds. Cellular data costs in emerging markets. |
| **Memory Usage** | < 200MB | Low-end Android devices have 2-3GB total. The OS needs half of that. |
| **JS Thread Frame Time** | < 16ms | If JS takes longer, animations stutter and touches feel delayed. |
| **UI Thread Frame Time** | < 8ms | The UI thread has to composite, layout, and paint. Give it room. |

### Why These Specific Numbers?

**The 1.5-second cold start target** comes from Google's research on mobile app abandonment. At 1 second, users feel the app is responsive. At 2 seconds, they notice the delay. At 3 seconds, 53% of users abandon the app entirely. The 1.5-second target gives you a margin of safety.

**The 60fps target** is non-negotiable. Your phone's display refreshes at 60Hz (or 90Hz, or 120Hz on newer devices). Every frame has exactly 16.6 milliseconds to render. If you miss that window, the frame gets dropped. Drop enough frames and users feel it as "jank" — that stuttery, laggy feeling that makes an app feel cheap.

Here is the math:

```
1 second = 1000ms
1000ms / 60 frames = 16.6ms per frame

Your JS thread work must complete in < 16ms
Your UI thread work must complete in the remaining time
Layout + paint + compositing needs ~4-8ms
```

**The 15MB bundle target** matters more than you think. In India, the average mobile data plan costs about $0.09/MB. A 50MB app costs $4.50 to download — that is a meaningful fraction of a daily wage for many users. Shopify learned this the hard way and invested heavily in shrinking their app from 50MB to 11MB. More on that later.

**The 200MB memory target** exists because Android will kill your app. Not gracefully — just kill it. Android's Low Memory Killer (LMK) daemon monitors memory pressure and terminates background apps when the system needs resources. On a device with 3GB of RAM, the OS and system services use ~1.5GB. If the user has a browser and a messaging app open, you might only have 300-400MB available. Stay under 200MB and you will survive in the background.

### How to Measure These

You cannot improve what you do not measure. Here is the toolchain:

```typescript
// Cold start measurement
// Add this to your app entry point (index.js or App.tsx)
const APP_START_TIME = global.__APP_START_TIME__ || Date.now();

export default function App() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    if (isReady) {
      const coldStartTime = Date.now() - APP_START_TIME;
      analytics.track('cold_start', { duration: coldStartTime });
      
      if (__DEV__) {
        console.log(`Cold start: ${coldStartTime}ms`);
      }
    }
  }, [isReady]);

  return (
    <AppProviders onReady={() => setIsReady(true)}>
      <RootNavigator />
    </AppProviders>
  );
}
```

For frame rate monitoring in development:

```typescript
// useFrameRate.ts — Development-only frame rate monitor
import { useEffect, useRef } from 'react';

export function useFrameRateMonitor() {
  const frameTimestamps = useRef<number[]>([]);
  const rafId = useRef<number>();

  useEffect(() => {
    if (!__DEV__) return;

    const measure = (timestamp: number) => {
      frameTimestamps.current.push(timestamp);

      // Keep last 60 timestamps (1 second of data)
      if (frameTimestamps.current.length > 60) {
        frameTimestamps.current.shift();
      }

      // Calculate FPS every 30 frames
      if (frameTimestamps.current.length >= 30) {
        const frames = frameTimestamps.current;
        const elapsed = frames[frames.length - 1] - frames[0];
        const fps = ((frames.length - 1) / elapsed) * 1000;

        if (fps < 50) {
          console.warn(`Low FPS detected: ${fps.toFixed(1)}`);
        }
      }

      rafId.current = requestAnimationFrame(measure);
    };

    rafId.current = requestAnimationFrame(measure);

    return () => {
      if (rafId.current) {
        cancelAnimationFrame(rafId.current);
      }
    };
  }, []);
}
```

For production monitoring, use Flipper's performance plugin, React Native Performance (by Shopify), or integrate with your APM tool (Datadog, Sentry, New Relic). The key is that you have a dashboard showing these numbers for real devices, not just your M3 MacBook running a simulator.

### The Performance Testing Matrix

Test on these device tiers or you are lying to yourself:

| Tier | Example Devices | Why |
|------|----------------|-----|
| **High-end** | iPhone 15 Pro, Pixel 8 Pro | Your designers and PM test here. Confirm it is buttery. |
| **Mid-range** | Samsung Galaxy A54, Pixel 7a | The most common device class globally. Your real baseline. |
| **Low-end** | Samsung Galaxy A13, Redmi Note 11 | Emerging markets. If it works here, it works everywhere. |

If you only test on high-end devices, you are building for 15% of your users.

---

## 13.2 Cold Start Optimization

Cold start is the single most impactful metric you can improve. It is the user's first impression every single time they open your app. Let me break down exactly what happens during a cold start and where the time goes.

### Anatomy of a React Native Cold Start

```
Time 0ms     → OS launches process, loads native binary
~100ms       → Native modules initialize (TurboModules or Bridge)
~200ms       → JavaScript engine (Hermes) starts up
~300ms       → JS bundle loads into memory
~400-800ms   → JS bundle parses and executes
~800-1200ms  → React tree mounts, first render
~1200-1500ms → Layout, paint, first meaningful content visible
~1500-2000ms → Data fetches complete, interactive
```

Every one of these phases is an opportunity for optimization. Let us attack each one.

### Hermes Bytecode — The Single Biggest Win

If you are not using Hermes with ahead-of-time (AOT) bytecode compilation, stop reading and go enable it. This is not optional.

Traditional JavaScript engines (like JSC or V8) parse JavaScript source code at runtime, build an AST, and then compile to bytecode or machine code. Hermes flips this: it compiles your JavaScript to bytecode at *build time*. When the app starts, Hermes just loads the pre-compiled bytecode directly into memory. No parsing. No compilation.

The impact is enormous:

| Engine | Parse + Compile Time | TTI Impact |
|--------|---------------------|------------|
| JSC (old default) | 800-1500ms | Slow cold starts on low-end devices |
| Hermes (bytecode) | 50-150ms | 5-10x faster startup |

Hermes has been the default in Expo SDK 47+ and React Native 0.70+. If you are on a modern stack, you already have it. But verify:

```typescript
// Check if Hermes is active
const isHermes = () => !!global.HermesInternal;

// In your app startup
if (__DEV__) {
  console.log(`JS Engine: ${isHermes() ? 'Hermes' : 'Other'}`);
}
```

### Lazy Imports — Stop Loading the Entire App Upfront

Here is a pattern I see in nearly every React Native codebase:

```typescript
// BAD: Every import executes at startup
import { HomeScreen } from './screens/HomeScreen';
import { ProfileScreen } from './screens/ProfileScreen';
import { SettingsScreen } from './screens/SettingsScreen';
import { AdminDashboard } from './screens/AdminDashboard';
import { AnalyticsScreen } from './screens/AnalyticsScreen';
import { ChatScreen } from './screens/ChatScreen';
import { SearchScreen } from './screens/SearchScreen';
import { NotificationsScreen } from './screens/NotificationsScreen';
import { OnboardingFlow } from './screens/OnboardingFlow';
import { PaymentScreen } from './screens/PaymentScreen';
// ... 40 more screens
```

Every one of these `import` statements executes at module load time. If `AdminDashboard` imports a charting library, a date picker, a rich text editor, and a data grid — all of that code runs before the user sees a single pixel.

The fix: lazy imports.

```typescript
// GOOD: Screens load only when navigated to
import { lazy } from 'react';

const HomeScreen = lazy(() => import('./screens/HomeScreen'));
const ProfileScreen = lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = lazy(() => import('./screens/SettingsScreen'));
const AdminDashboard = lazy(() => import('./screens/AdminDashboard'));
```

With React Navigation, you can integrate this directly:

```typescript
import { lazy, Suspense } from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { ActivityIndicator, View } from 'react-native';

const Stack = createNativeStackNavigator();

// Lazy-loaded screens
const HomeScreen = lazy(() => import('./screens/HomeScreen'));
const ProfileScreen = lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = lazy(() => import('./screens/SettingsScreen'));

function ScreenFallback() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <ActivityIndicator size="large" />
    </View>
  );
}

function withSuspense(LazyComponent: React.LazyExoticComponent<any>) {
  return function SuspenseWrapper(props: any) {
    return (
      <Suspense fallback={<ScreenFallback />}>
        <LazyComponent {...props} />
      </Suspense>
    );
  };
}

export function RootNavigator() {
  return (
    <Stack.Navigator>
      {/* Home loads eagerly — it's the first screen */}
      <Stack.Screen name="Home" component={HomeScreenEager} />
      
      {/* Everything else loads on demand */}
      <Stack.Screen name="Profile" component={withSuspense(ProfileScreen)} />
      <Stack.Screen name="Settings" component={withSuspense(SettingsScreen)} />
    </Stack.Navigator>
  );
}
```

### Async Module Loading with `require()`

For modules that are expensive to initialize but not needed at startup, use inline `require()`:

```typescript
// BAD: Heavy library loads at import time
import Markdown from 'react-native-markdown-display';
import { format, parseISO, differenceInDays } from 'date-fns';
import { decode } from 'html-entities';

// GOOD: Load only when needed
function ArticleScreen({ article }) {
  const [MarkdownComponent, setMarkdown] = useState(null);

  useEffect(() => {
    // Load the markdown renderer only when this screen mounts
    const Markdown = require('react-native-markdown-display').default;
    setMarkdown(() => Markdown);
  }, []);

  if (!MarkdownComponent) {
    return <ArticleSkeleton />;
  }

  return <MarkdownComponent>{article.body}</MarkdownComponent>;
}
```

Or more elegantly with a custom hook:

```typescript
// useLazyModule.ts
function useLazyModule<T>(loader: () => T): T | null {
  const [mod, setMod] = useState<T | null>(null);

  useEffect(() => {
    // Use InteractionManager to defer loading until animations complete
    const handle = InteractionManager.runAfterInteractions(() => {
      setMod(loader());
    });

    return () => handle.cancel();
  }, []);

  return mod;
}

// Usage
function RichEditor() {
  const Editor = useLazyModule(() => 
    require('react-native-rich-editor').default
  );

  if (!Editor) return <EditorSkeleton />;
  return <Editor />;
}
```

### TurboModules — Lazy Native Module Initialization

The old React Native bridge loaded *all* native modules at startup, whether you needed them or not. Camera module? Loaded. Bluetooth module? Loaded. In-app purchase module? Loaded. Even if the user never opens the camera screen.

TurboModules (part of the New Architecture) solve this with lazy initialization. Native modules only load when JavaScript first accesses them:

```typescript
// Old architecture: All native modules load at startup
// Camera, Bluetooth, IAP, Biometrics... everything initializes

// New Architecture with TurboModules: 
// Native modules initialize on first access
import { TurboModuleRegistry } from 'react-native';

// This does NOT initialize the camera module
// It only initializes when you actually call a method
const CameraModule = TurboModuleRegistry.get('CameraModule');

// NOW the camera module initializes
const devices = await CameraModule?.getAvailableDevices();
```

If you are on Expo SDK 51+ or React Native 0.74+, you likely have TurboModules enabled by default. The impact on cold start depends on how many native modules your app uses, but I have seen 200-400ms improvements on apps with 15+ native modules.

### Reducing the Initial Render Tree

Every component in your initial render tree costs time. React has to create the element, reconcile it, lay it out, and paint it. The deeper and wider your tree, the longer first render takes.

```typescript
// BAD: Rendering everything immediately
function HomeScreen() {
  return (
    <ScrollView>
      <Header />
      <HeroBanner />           {/* User sees this */}
      <CategoryGrid />         {/* User sees this */}
      <TrendingProducts />     {/* Below the fold — user scrolls to this */}
      <RecentlyViewed />       {/* Below the fold */}
      <RecommendedForYou />    {/* Below the fold */}
      <PopularBrands />        {/* Below the fold */}
      <NewsletterSignup />     {/* Way below the fold */}
      <Footer />               {/* Nobody ever sees this */}
    </ScrollView>
  );
}
```

The fix: defer below-fold content.

```typescript
// GOOD: Only render what the user can see immediately
function HomeScreen() {
  const [showBelowFold, setShowBelowFold] = useState(false);

  useEffect(() => {
    // Defer below-fold content until after first paint
    const handle = InteractionManager.runAfterInteractions(() => {
      setShowBelowFold(true);
    });
    return () => handle.cancel();
  }, []);

  return (
    <ScrollView>
      <Header />
      <HeroBanner />
      <CategoryGrid />
      
      {showBelowFold && (
        <>
          <TrendingProducts />
          <RecentlyViewed />
          <RecommendedForYou />
          <PopularBrands />
          <NewsletterSignup />
          <Footer />
        </>
      )}
    </ScrollView>
  );
}
```

A more sophisticated approach uses `onScroll` to progressively reveal content:

```typescript
function HomeScreen() {
  const [visibleSections, setVisibleSections] = useState(2); // Start with 2
  const sections = [
    HeroBanner,
    CategoryGrid, 
    TrendingProducts,
    RecentlyViewed,
    RecommendedForYou,
    PopularBrands,
  ];

  const handleScroll = useCallback((event: NativeSyntheticEvent<NativeScrollEvent>) => {
    const { contentOffset, layoutMeasurement, contentSize } = event.nativeEvent;
    const scrollPercentage = 
      (contentOffset.y + layoutMeasurement.height) / contentSize.height;

    // Load more sections as user scrolls
    if (scrollPercentage > 0.6 && visibleSections < sections.length) {
      setVisibleSections(prev => Math.min(prev + 2, sections.length));
    }
  }, [visibleSections]);

  return (
    <ScrollView onScroll={handleScroll} scrollEventThrottle={16}>
      {sections.slice(0, visibleSections).map((Section, index) => (
        <Section key={index} />
      ))}
    </ScrollView>
  );
}
```

### Cold Start Optimization Checklist

Here is the checklist I use for every app:

```
[ ] Hermes enabled with bytecode compilation
[ ] Lazy screen loading via React.lazy + Suspense
[ ] TurboModules enabled (New Architecture)
[ ] Below-fold content deferred via InteractionManager
[ ] Heavy libraries loaded via inline require()
[ ] Splash screen covers initialization (expo-splash-screen)
[ ] No synchronous storage reads at startup (use async)
[ ] Minimal provider nesting in root component
[ ] Font loading is async, not blocking
[ ] No large JSON files imported at module level
```

The combined effect of these optimizations typically brings cold start from 3-4 seconds down to 1-1.5 seconds on mid-range devices.

---

## 13.3 List Performance — The Battle You Will Fight Every Day

Lists are the most performance-critical component in any mobile app. Your feed, your product catalog, your chat messages, your search results — they are all lists. And lists are where React Native apps most commonly fall apart.

Let me explain why, starting from first principles.

### Why Lists Are Hard

A typical social media feed might have 10,000 items. Each item is a complex component: avatar, text, images, action buttons, maybe a video thumbnail. If you naively render all 10,000 items, you are asking the device to:

1. Create 10,000 React element trees
2. Create 10,000 native views
3. Lay out 10,000 components
4. Keep 10,000 views in memory

On a phone with 4GB of RAM, this is not going to work. The solution is **virtualization** — only rendering the items that are currently visible on screen, plus a small buffer above and below.

### FlatList: The Built-In Solution and Its Limitations

React Native's `FlatList` is a virtualized list. It renders items that are in or near the viewport and destroys items that scroll far enough away. This is fundamentally correct, but the implementation has problems.

FlatList's key issue: it **creates and destroys** views as items scroll in and out of the viewport. Creating a new native view is expensive — it involves:

1. Creating the React element (JS thread)
2. Sending the create command across the bridge/JSI (communication overhead)
3. Creating the native view (UI thread)
4. Layout calculation (UI thread)
5. Painting (GPU)

When you scroll fast, FlatList has to create many views quickly. If it cannot keep up, you see blank space — the dreaded "blank area" problem.

```typescript
// Basic FlatList — works for simple cases
<FlatList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  keyExtractor={(item) => item.id}
  // These help, but FlatList's core architecture limits it
  maxToRenderPerBatch={10}
  windowSize={5}
  removeClippedSubviews={true}
  initialNumToRender={10}
/>
```

### FlashList v2: The Real Solution

FlashList, built by Shopify, takes a fundamentally different approach inspired by how native iOS (`UITableView` / `UICollectionView`) and Android (`RecyclerView`) handle lists. Instead of creating and destroying views, FlashList **recycles** them.

Here is the key insight: when a list item scrolls off the top of the screen, its native view is not destroyed. Instead, it is moved to a recycling pool. When a new item needs to appear at the bottom of the screen, FlashList grabs a view from the pool and re-binds it with new data.

```
Visible viewport: Items 5-15

Scrolling down...

Item 5 scrolls off the top
  → Its native view goes to the recycling pool

Item 16 needs to appear at the bottom
  → FlashList grabs the recycled view from the pool
  → Updates it with Item 16's data
  → No native view creation needed!
```

This is dramatically faster because updating an existing view's properties is much cheaper than creating a new view from scratch.

```typescript
import { FlashList } from '@shopify/flash-list';

// Drop-in replacement for FlatList with recycling
<FlashList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  estimatedItemSize={120}  // REQUIRED: approximate height of each item
  keyExtractor={(item) => item.id}
/>
```

### FlashList Configuration That Actually Matters

#### `estimatedItemSize` (Required)

This is the one prop you must get right. FlashList uses this estimate to calculate how many items to render initially and how much buffer space to maintain.

```typescript
// Measure your actual item height, then use that
// Too small = too few items rendered = blank space when scrolling
// Too large = too many items rendered = wasted memory

// For a social feed with variable height posts
<FlashList
  estimatedItemSize={280}  // Measure average post height
  // ...
/>

// For a simple list with fixed-height rows
<FlashList
  estimatedItemSize={72}  // Exact height if all items are the same
  // ...
/>
```

How to find the right value: in development, FlashList logs a warning with the actual average item size after initial render. Use that number.

#### `drawDistance` — The Scroll Buffer

`drawDistance` controls how far outside the visible area FlashList pre-renders items. Higher values mean less blank space during fast scrolling, but more memory usage.

```typescript
// Default is 250 (pixels)
<FlashList
  drawDistance={250}   // Default: good for most cases
  // ...
/>

// For very fast scrolling (e.g., momentum scroll in a long list)
<FlashList
  drawDistance={500}   // Render further ahead
  // ...
/>

// For memory-constrained scenarios
<FlashList
  drawDistance={100}   // Render less, save memory
  // ...
/>
```

#### `overrideItemLayout` — When Items Have Different Sizes

If your list has items of varying heights (and they do — think a social feed with text posts, image posts, and video posts), `overrideItemLayout` lets you tell FlashList the exact size of each item type:

```typescript
<FlashList
  data={feedItems}
  renderItem={({ item }) => {
    switch (item.type) {
      case 'text': return <TextPost item={item} />;
      case 'image': return <ImagePost item={item} />;
      case 'video': return <VideoPost item={item} />;
    }
  }}
  // Tell FlashList about different item types for better recycling
  getItemType={(item) => item.type}
  estimatedItemSize={200}
  overrideItemLayout={(layout, item) => {
    switch (item.type) {
      case 'text':
        layout.size = 120;
        break;
      case 'image':
        layout.size = 350;
        break;
      case 'video':
        layout.size = 280;
        break;
    }
  }}
/>
```

The `getItemType` prop is crucial here. FlashList maintains separate recycling pools for each item type. A text post view gets recycled into another text post view, not an image post view. This prevents the costly layout recalculations that would happen if a simple 120px text view had to transform into a 350px image view.

### When to Use SectionList

Use `SectionList` (or FlashList with sticky headers) when you need grouped data with headers:

```typescript
// Contacts list with alphabetical sections
import { FlashList } from '@shopify/flash-list';

const sections = [
  { title: 'A', data: [{ name: 'Alice' }, { name: 'Alex' }] },
  { title: 'B', data: [{ name: 'Bob' }, { name: 'Barbara' }] },
  // ...
];

// Flatten for FlashList but track section boundaries
const flatData = sections.flatMap(section => [
  { type: 'header', title: section.title },
  ...section.data.map(item => ({ type: 'item', ...item })),
]);

<FlashList
  data={flatData}
  renderItem={({ item }) => {
    if (item.type === 'header') {
      return <SectionHeader title={item.title} />;
    }
    return <ContactRow name={item.name} />;
  }}
  getItemType={(item) => item.type}
  estimatedItemSize={56}
  stickyHeaderIndices={
    flatData
      .map((item, index) => (item.type === 'header' ? index : null))
      .filter(Boolean) as number[]
  }
/>
```

### Virtualization from Scratch — Understanding the Mechanics

To truly understand list performance, let me walk you through how virtualization works at the fundamental level. This is what FlatList and FlashList do under the hood.

```typescript
// Simplified virtualization concept
// This is educational — use FlashList in production

interface VirtualizedListProps<T> {
  data: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  itemHeight: number;  // Simplified: fixed height
  containerHeight: number;
}

function SimpleVirtualizedList<T>({
  data,
  renderItem,
  itemHeight,
  containerHeight,
}: VirtualizedListProps<T>) {
  const [scrollTop, setScrollTop] = useState(0);

  // Total height of all items (for scroll container sizing)
  const totalHeight = data.length * itemHeight;

  // Calculate which items are visible
  const overscan = 3; // Extra items above and below
  const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
  const endIndex = Math.min(
    data.length - 1,
    Math.ceil((scrollTop + containerHeight) / itemHeight) + overscan
  );

  // Only render visible items
  const visibleItems = [];
  for (let i = startIndex; i <= endIndex; i++) {
    visibleItems.push(
      <View
        key={data[i].id}
        style={{
          position: 'absolute',
          top: i * itemHeight,
          height: itemHeight,
          left: 0,
          right: 0,
        }}
      >
        {renderItem(data[i], i)}
      </View>
    );
  }

  return (
    <ScrollView
      onScroll={(e) => setScrollTop(e.nativeEvent.contentOffset.y)}
      scrollEventThrottle={16}
      style={{ height: containerHeight }}
    >
      {/* Spacer to maintain correct scroll height */}
      <View style={{ height: totalHeight, position: 'relative' }}>
        {visibleItems}
      </View>
    </ScrollView>
  );
}
```

The key concept: a `ScrollView` with a spacer element that is the full height of all items combined. Inside that spacer, only the visible items are rendered with absolute positioning. As the user scrolls, we calculate new visible items and React reconciles the difference.

FlashList improves on this by:
1. **Recycling views** instead of creating/destroying them
2. **Using native cell recycling** (UICollectionView/RecyclerView) under the hood
3. **Handling variable-height items** with layout estimation
4. **Maintaining separate type pools** for heterogeneous lists

### Performance Comparison: Real Numbers

Here are benchmarks from a production app with a feed of 1,000 items, each containing an avatar, text, and an image. Measured on a Samsung Galaxy A54:

| Metric | FlatList | FlashList v2 |
|--------|----------|-------------|
| **Initial render** | 380ms | 190ms |
| **Scroll FPS (slow)** | 58 fps | 60 fps |
| **Scroll FPS (fast momentum)** | 42 fps | 57 fps |
| **Blank area (fast scroll)** | Frequent | Rare |
| **Memory (1K items loaded)** | 280MB | 160MB |
| **JS thread usage (scrolling)** | 12ms/frame | 4ms/frame |

FlashList is not marginally better — it is fundamentally better for large lists.

### List Performance Optimization Patterns

**1. Keep renderItem components lightweight**

```typescript
// BAD: Heavy computation in renderItem
const renderItem = ({ item }) => {
  const formattedDate = new Intl.DateTimeFormat('en-US', {
    year: 'numeric', month: 'long', day: 'numeric',
    hour: '2-digit', minute: '2-digit',
  }).format(new Date(item.createdAt));

  const excerpt = item.body.slice(0, 200).replace(/\s+/g, ' ').trim() + '...';

  return (
    <View>
      <Text>{formattedDate}</Text>
      <Text>{excerpt}</Text>
    </View>
  );
};

// GOOD: Pre-compute in data transformation, not in render
const processedItems = useMemo(() => 
  items.map(item => ({
    ...item,
    formattedDate: formatDate(item.createdAt),  // Computed once
    excerpt: truncateText(item.body, 200),       // Computed once
  })),
  [items]
);
```

**2. Avoid inline functions and objects in renderItem**

```typescript
// BAD: New function and style object created every render
const renderItem = ({ item }) => (
  <Pressable onPress={() => navigate('Detail', { id: item.id })}>
    <View style={{ padding: 16, backgroundColor: '#fff' }}>
      <Text>{item.title}</Text>
    </View>
  </Pressable>
);

// GOOD: Stable references
const styles = StyleSheet.create({
  card: { padding: 16, backgroundColor: '#fff' },
});

const MemoizedItem = React.memo(function FeedItem({ 
  item, 
  onPress 
}: { 
  item: FeedItem; 
  onPress: (id: string) => void;
}) {
  const handlePress = useCallback(() => onPress(item.id), [item.id, onPress]);

  return (
    <Pressable onPress={handlePress}>
      <View style={styles.card}>
        <Text>{item.title}</Text>
      </View>
    </Pressable>
  );
});
```

**3. Use `CellRendererComponent` for complex cell layouts**

```typescript
// Wrap each cell in a custom container for consistent styling
// without affecting recycling performance
<FlashList
  CellRendererComponent={({ children, index, style }) => (
    <View style={[style, { paddingHorizontal: 16, marginBottom: 8 }]}>
      {children}
    </View>
  )}
  // ...
/>
```

---

## 13.4 Re-Render Elimination

Every unnecessary re-render is stolen time. On a 60fps budget, you have 16 milliseconds per frame. If a parent component re-renders and triggers re-renders in 50 child components, you have burned through your frame budget before the user even scrolled.

### Understanding When React Re-Renders

React re-renders a component when:

1. Its state changes (`useState`, `useReducer`)
2. Its parent re-renders (unless memoized)
3. A context it consumes changes
4. A store it subscribes to dispatches

The problem is #2. By default, when a parent re-renders, *all* of its children re-render, even if their props have not changed. This cascade is where performance dies.

```typescript
// This is the pattern that kills performance
function FeedScreen() {
  const [refreshing, setRefreshing] = useState(false);
  // When refreshing changes, ALL children re-render

  return (
    <View>
      <Header />          {/* Re-renders unnecessarily */}
      <FilterBar />       {/* Re-renders unnecessarily */}
      <FeedList />         {/* Re-renders unnecessarily */}
      <FloatingButton />   {/* Re-renders unnecessarily */}
    </View>
  );
}
```

### React Compiler — The New Default (SDK 54+)

The React Compiler (formerly React Forget) is the biggest change to React performance in years. As of Expo SDK 54, it is enabled by default, and 83% of EAS builds have it active. Here is what it does.

The React Compiler analyzes your component code at build time and automatically inserts memoization. It effectively auto-generates `React.memo`, `useMemo`, and `useCallback` where they are beneficial.

**Before the compiler, you had to write this:**

```typescript
const MemoizedHeader = React.memo(function Header({ title, onBack }) {
  return (
    <View style={styles.header}>
      <Pressable onPress={onBack}>
        <Icon name="back" />
      </Pressable>
      <Text>{title}</Text>
    </View>
  );
});

function FeedScreen() {
  const [refreshing, setRefreshing] = useState(false);
  
  // Manual memoization to prevent re-creating this function
  const handleBack = useCallback(() => {
    navigation.goBack();
  }, [navigation]);

  // Manual memoization of the title
  const title = useMemo(() => `Feed (${items.length})`, [items.length]);

  return (
    <View>
      <MemoizedHeader title={title} onBack={handleBack} />
      {/* ... */}
    </View>
  );
}
```

**With the React Compiler, you just write normal code:**

```typescript
function Header({ title, onBack }) {
  return (
    <View style={styles.header}>
      <Pressable onPress={onBack}>
        <Icon name="back" />
      </Pressable>
      <Text>{title}</Text>
    </View>
  );
}

function FeedScreen() {
  const [refreshing, setRefreshing] = useState(false);
  const handleBack = () => navigation.goBack();
  const title = `Feed (${items.length})`;

  return (
    <View>
      <Header title={title} onBack={handleBack} />
      {/* ... */}
    </View>
  );
}

// The compiler automatically memoizes Header, handleBack, and title
// No manual React.memo, useCallback, or useMemo needed
```

The compiler figures out the dependencies and inserts memoization at the right granularity. It actually does a better job than most developers because it memoizes at the JSX element level, not just the component level.

### How to Enable the React Compiler

If you are on Expo SDK 54+, it is likely already enabled. To explicitly configure it:

```javascript
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      ['babel-plugin-react-compiler', {
        // Options
        runtimeModule: 'react/compiler-runtime',
      }],
    ],
  };
};
```

To verify it is working, look for the `react/compiler-runtime` import in your compiled output, or use the React DevTools Profiler — compiled components show a small compiler badge.

### When You Still Need Manual Memoization

The React Compiler is not magic. There are cases where you still need manual intervention:

**1. External store subscriptions with expensive selectors**

```typescript
// The compiler cannot optimize Zustand selector patterns
// because it does not understand the store's update mechanics

// BAD: Re-renders on any store change
function UserProfile() {
  const store = useAppStore();
  // This re-renders when ANY part of the store changes
  return <Text>{store.user.name}</Text>;
}

// GOOD: Selector prevents unnecessary re-renders
function UserProfile() {
  const userName = useAppStore((state) => state.user.name);
  // Only re-renders when user.name actually changes
  return <Text>{userName}</Text>;
}

// ALSO GOOD: Shallow equality for objects
import { shallow } from 'zustand/shallow';

function UserCard() {
  const { name, avatar, bio } = useAppStore(
    (state) => ({
      name: state.user.name,
      avatar: state.user.avatar,
      bio: state.user.bio,
    }),
    shallow // Compare object shallowly, not by reference
  );

  return (
    <View>
      <Image source={{ uri: avatar }} />
      <Text>{name}</Text>
      <Text>{bio}</Text>
    </View>
  );
}
```

**2. Expensive computations that depend on large data sets**

```typescript
// The compiler will memoize this, but it cannot know that
// the computation is O(n log n) and should really be debounced

function SearchResults({ query, items }) {
  // This is correct memoization, but the compiler will add it anyway
  // The real optimization is algorithmic: use a search index
  const results = useMemo(() => {
    if (!query) return items;
    
    return items
      .filter(item => 
        item.title.toLowerCase().includes(query.toLowerCase()) ||
        item.description.toLowerCase().includes(query.toLowerCase())
      )
      .sort((a, b) => {
        // Relevance scoring
        const aScore = getRelevanceScore(a, query);
        const bScore = getRelevanceScore(b, query);
        return bScore - aScore;
      });
  }, [query, items]);

  return <FlashList data={results} /* ... */ />;
}
```

**3. Components receiving non-primitive props from non-compiled libraries**

```typescript
// If a third-party library passes new object references every render,
// you need manual memoization at the boundary

function MapWithMarkers({ locations }) {
  // Third-party map library creates new marker objects every render
  const markers = useMemo(() => 
    locations.map(loc => ({
      coordinate: { latitude: loc.lat, longitude: loc.lng },
      title: loc.name,
    })),
    [locations]
  );

  return <MapView markers={markers} />;
}
```

### The `why-did-you-render` Tool

When you have performance problems and you are not sure what is re-rendering, `why-did-you-render` is invaluable:

```typescript
// wdyr.ts — Import this FIRST in your entry file
import React from 'react';

if (__DEV__) {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    logOnDifferentValues: true,
    // Track specific components by name
    trackExtraHooks: [
      [require('zustand'), 'useStore'],
    ],
  });
}
```

```typescript
// Then on any component you want to investigate:
function ExpensiveList({ items, filter }) {
  // ... component logic
}

ExpensiveList.whyDidYouRender = true;

// Console output will show:
// ExpensiveList - Re-rendered because:
// Props: { filter: "new value" !== "old value" }
// or
// Props: { items: [Array(100)] !== [Array(100)] } (same content, different reference!)
```

### Zustand Selectors — The Key to Store Performance

Zustand is the state management library I recommend for React Native (see Chapter 8). But using it incorrectly can cause as many re-renders as Context. The key is selectors.

```typescript
// store.ts
interface AppState {
  user: { name: string; email: string; avatar: string };
  feed: { items: FeedItem[]; loading: boolean; page: number };
  notifications: { unread: number; items: Notification[] };
  settings: { theme: 'light' | 'dark'; language: string };
}

const useAppStore = create<AppState>((set) => ({
  user: { name: '', email: '', avatar: '' },
  feed: { items: [], loading: false, page: 1 },
  notifications: { unread: 0, items: [] },
  settings: { theme: 'light', language: 'en' },
}));

// BAD: Subscribes to entire store. Re-renders on ANY change.
function NotificationBadge() {
  const state = useAppStore();
  return <Badge count={state.notifications.unread} />;
}
// When user changes theme → NotificationBadge re-renders. Why?

// GOOD: Subscribe to exactly what you need
function NotificationBadge() {
  const unreadCount = useAppStore((state) => state.notifications.unread);
  return <Badge count={unreadCount} />;
}
// When user changes theme → NotificationBadge does NOT re-render.
```

The performance impact is dramatic. In an app with 50 components subscribed to the store, a single state update could trigger 50 re-renders with the naive approach. With proper selectors, it triggers re-renders only in the 2-3 components that actually use the changed data.

### Re-Render Audit Process

Here is my process for hunting down unnecessary re-renders:

```
1. Enable React DevTools Profiler
2. Record a typical user interaction (scroll, tap, type)
3. Look for components that render >2x during the interaction
4. For each:
   a. Check if it receives new object/array references as props
   b. Check if it subscribes to a store without selectors
   c. Check if it consumes a Context that updates frequently
5. Apply the appropriate fix:
   a. New references → memoize at the parent level
   b. Store without selectors → add selectors
   c. Frequent Context → split into multiple Contexts or use Zustand
```

---

## 13.5 Animation Performance

Smooth animations are what separate a good mobile app from a great one. But animations are also where React Native apps most commonly feel "not native." Let me explain why and how to fix it.

### The Two-Thread Problem

React Native runs on (at minimum) two threads:

1. **JS Thread** — Runs your JavaScript/React code
2. **UI Thread** — Handles rendering, layout, gestures, and native animations

When you drive animations from JavaScript, every frame of the animation requires:

```
JS Thread: Calculate new position → 
  Bridge/JSI: Send update to native → 
    UI Thread: Apply the update → 
      GPU: Render the frame
```

If the JS thread is busy (processing a network response, running business logic, re-rendering components), the animation frame gets delayed. The result: janky, stuttering animations.

The solution: run animations on the UI thread, where they cannot be blocked by JavaScript.

### `useNativeDriver` — The Minimum Viable Solution

React Native's `Animated` API has a `useNativeDriver` option that offloads the animation to the UI thread:

```typescript
import { Animated } from 'react-native';

function FadeInView({ children }) {
  const opacity = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    Animated.timing(opacity, {
      toValue: 1,
      duration: 300,
      useNativeDriver: true,  // Animation runs on UI thread
    }).start();
  }, []);

  return (
    <Animated.View style={{ opacity }}>
      {children}
    </Animated.View>
  );
}
```

The limitation: `useNativeDriver` only works with a subset of properties — specifically `transform` and `opacity`. You cannot use it for `width`, `height`, `backgroundColor`, `borderRadius`, or any layout-affecting property. This is because layout changes require the Yoga layout engine, which runs on the JS thread.

```typescript
// WORKS with useNativeDriver:
// - opacity
// - transform: [{ translateX }, { translateY }, { scale }, { rotate }]

// DOES NOT WORK with useNativeDriver:
// - width, height
// - backgroundColor
// - borderRadius
// - margin, padding
// - Any layout property
```

For simple fade/slide/scale animations, `useNativeDriver: true` is sufficient. For anything more complex, you need Reanimated.

### Reanimated v3 — The Real Animation Engine

Reanimated v3 is a fundamentally different approach to animations. It runs animation logic in **worklets** — small JavaScript functions that execute on the UI thread, completely independent of the JS thread.

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

function SwipeableCard() {
  // Shared values live on the UI thread
  const translateX = useSharedValue(0);
  const opacity = useSharedValue(1);

  // Animated styles are computed on the UI thread as worklets
  const cardStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
    opacity: opacity.value,
  }));

  const handleSwipe = () => {
    // These animations run entirely on the UI thread
    translateX.value = withSpring(300, { damping: 15, stiffness: 150 });
    opacity.value = withTiming(0, { duration: 200 });
  };

  return (
    <Animated.View style={[styles.card, cardStyle]}>
      <Text>Swipe me</Text>
      <Button onPress={handleSwipe} title="Swipe" />
    </Animated.View>
  );
}
```

The key difference: `useAnimatedStyle` returns a worklet function that runs on the UI thread. When `translateX.value` changes, the style updates happen on the UI thread *without crossing the bridge*. The JS thread is completely uninvolved.

### Shared Values — The Foundation

Shared values are Reanimated's core primitive. They are values that can be read and written from both the JS thread and the UI thread.

```typescript
import { useSharedValue, runOnJS, runOnUI } from 'react-native-reanimated';

function AnimatedCounter() {
  // Create a shared value
  const progress = useSharedValue(0);
  const [displayValue, setDisplayValue] = useState(0);

  // Worklet: runs on UI thread
  const updateProgress = (newValue: number) => {
    'worklet';  // This directive tells Reanimated to run on UI thread
    progress.value = withTiming(newValue, { duration: 500 });
    
    // Need to update React state? Use runOnJS to cross back
    runOnJS(setDisplayValue)(newValue);
  };

  const animatedStyle = useAnimatedStyle(() => ({
    width: `${progress.value}%`,
  }));

  return (
    <View>
      <Animated.View style={[styles.progressBar, animatedStyle]} />
      <Text>Progress: {displayValue}%</Text>
      <Button onPress={() => runOnUI(updateProgress)(progress.value + 10)} />
    </View>
  );
}
```

### Layout Animations — Automatic Transitions

Reanimated v3 includes layout animations that automatically animate components when they enter, exit, or change position in the layout:

```typescript
import Animated, {
  FadeIn,
  FadeOut,
  SlideInRight,
  SlideOutLeft,
  Layout,
} from 'react-native-reanimated';

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([]);

  const removeTodo = (id: string) => {
    setTodos(prev => prev.filter(t => t.id !== id));
  };

  return (
    <View>
      {todos.map((todo) => (
        <Animated.View
          key={todo.id}
          entering={SlideInRight.duration(300).springify()}
          exiting={SlideOutLeft.duration(200)}
          layout={Layout.springify()}  // Animate position changes
        >
          <TodoItem
            todo={todo}
            onRemove={() => removeTodo(todo.id)}
          />
        </Animated.View>
      ))}
    </View>
  );
}
```

When you remove a todo, the item slides out to the left, and all remaining items smoothly animate up to fill the gap. All of this runs on the UI thread. No jank, even if the JS thread is completely blocked.

### Gesture Handler + Reanimated — The Power Combo

The real magic happens when you combine Reanimated with React Native Gesture Handler. Gestures (pan, pinch, rotation) are detected on the native thread, and the animation response runs on the UI thread. JavaScript is never involved.

```typescript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withDecay,
} from 'react-native-reanimated';

function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);
  const savedX = useSharedValue(0);
  const savedY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onStart(() => {
      // Save current position
      savedX.value = translateX.value;
      savedY.value = translateY.value;
      scale.value = withSpring(1.05); // Slight scale up when grabbed
    })
    .onUpdate((event) => {
      // Directly update position — no JS thread involved
      translateX.value = savedX.value + event.translationX;
      translateY.value = savedY.value + event.translationY;
    })
    .onEnd((event) => {
      scale.value = withSpring(1); // Scale back to normal
      
      // Apply velocity-based decay for natural feel
      translateX.value = withDecay({
        velocity: event.velocityX,
        deceleration: 0.997,
      });
      translateY.value = withDecay({
        velocity: event.velocityY,
        deceleration: 0.997,
      });
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Drag me!</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

This drag interaction is 60fps on a low-end Android device because the gesture recognition and the animation response both happen on the native/UI thread. The JS thread could be running a complex API call and the drag would still be butter-smooth.

### JS-Driven vs Native-Driven: A Decision Framework

| Factor | JS-Driven (Animated API) | Native-Driven (useNativeDriver) | Reanimated Worklets |
|--------|-------------------------|-------------------------------|-------------------|
| **Thread** | JS thread | UI thread | UI thread |
| **Properties** | All | opacity, transform only | All |
| **Gesture response** | 1-3 frame delay | Not applicable | Immediate |
| **JS thread blocking** | Blocks animations | No effect | No effect |
| **Complexity** | Simple | Simple | Moderate |
| **Use when** | Prototyping, simple animations | Fade/slide/scale only | Production animations, gestures |

**My recommendation:** Default to Reanimated for everything. The API is actually simpler than `Animated` once you learn shared values. The performance is categorically better. There is no reason to use JS-driven animations in a production app.

### Common Animation Anti-Patterns

**1. Animating layout properties without Reanimated**

```typescript
// BAD: Triggers layout recalculation every frame
Animated.timing(width, {
  toValue: 200,
  duration: 300,
  useNativeDriver: false, // Cannot use native driver for width
}).start();

// GOOD: Use Reanimated which handles layout animations on UI thread
const animatedStyle = useAnimatedStyle(() => ({
  width: withTiming(isExpanded.value ? 200 : 100, { duration: 300 }),
}));
```

**2. Animating too many elements simultaneously**

```typescript
// BAD: 100 independent animations overwhelming the UI thread
{items.map((item, index) => (
  <Animated.View
    key={item.id}
    entering={FadeIn.delay(index * 50)} // 100 staggered animations
  />
))}

// GOOD: Limit simultaneous animations
{items.slice(0, visibleCount).map((item, index) => (
  <Animated.View
    key={item.id}
    entering={index < 10 ? FadeIn.delay(index * 50) : FadeIn}
  />
))}
```

**3. Using `setState` in gesture handlers**

```typescript
// BAD: Crossing the bridge every gesture frame
const panGesture = Gesture.Pan().onUpdate((event) => {
  setPosition({ x: event.translationX, y: event.translationY });
  // Every frame: UI → JS → setState → re-render → bridge → UI
});

// GOOD: Stay on the UI thread
const panGesture = Gesture.Pan().onUpdate((event) => {
  translateX.value = event.translationX;
  translateY.value = event.translationY;
  // Every frame: UI → update shared value → UI renders
});
```

---

## 13.6 Bundle Size Optimization

Bundle size directly impacts two things: download time (which affects installs) and startup time (which affects every session). Every byte in your bundle has to be downloaded over cellular, stored on device, and parsed by Hermes.

### Understanding Your Bundle

Before optimizing, understand what is in your bundle:

```bash
# Generate a Metro bundle stats file
npx react-native bundle \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js \
  --sourcemap-output /tmp/bundle.js.map

# Visualize with source-map-explorer
npx source-map-explorer /tmp/bundle.js /tmp/bundle.js.map
```

This generates an interactive treemap showing exactly which packages consume the most space. In my experience, the usual suspects are:

| Library | Typical Size | Common Culprit |
|---------|-------------|----------------|
| `moment.js` | 300KB+ | Includes all locales |
| `lodash` (full) | 530KB | Importing the entire library |
| `date-fns` (full) | 200KB+ | Importing all functions |
| Icon libraries | 2-5MB | Including all icon sets |
| `aws-sdk` v2 | 3MB+ | The entire AWS SDK |
| Lottie animations | Varies | Unoptimized JSON files |

### Tree Shaking and Dead Code Elimination

Metro (React Native's bundler) supports tree shaking as of React Native 0.73+. But tree shaking only works if the library uses ES modules (`import`/`export`), not CommonJS (`require`/`module.exports`).

```typescript
// Tree-shakeable — Metro can eliminate unused exports
import { format, parseISO } from 'date-fns';
// Only format and parseISO are included in the bundle

// NOT tree-shakeable — entire library included
const _ = require('lodash');
// Even if you only use _.get, the entire 530KB is bundled
```

### Replacing Heavy Libraries

This is the highest-impact optimization you can make:

**moment.js -> day.js**

```typescript
// BEFORE: moment.js — 300KB+
import moment from 'moment';
const formatted = moment(date).format('MMMM Do, YYYY');
const relative = moment(date).fromNow();

// AFTER: day.js — 7KB (with plugins)
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
dayjs.extend(relativeTime);

const formatted = dayjs(date).format('MMMM D, YYYY');
const relative = dayjs(date).fromNow();
```

Savings: ~293KB. For one API change.

**lodash -> individual imports or native methods**

```typescript
// BEFORE: Full lodash — 530KB
import _ from 'lodash';
const result = _.groupBy(items, 'category');
const unique = _.uniqBy(items, 'id');
const deep = _.cloneDeep(obj);

// AFTER: Individual imports — only what you use (~5KB each)
import groupBy from 'lodash/groupBy';
import uniqBy from 'lodash/uniqBy';

// Or better yet, use native methods
const result = Object.groupBy(items, item => item.category); // ES2024
const unique = [...new Map(items.map(i => [i.id, i])).values()];
const deep = structuredClone(obj); // Built-in
```

**Full icon libraries -> individual icons**

```typescript
// BEFORE: Imports entire icon set — 2-5MB
import { MaterialIcons } from '@expo/vector-icons';
<MaterialIcons name="home" size={24} />;

// AFTER: Import only the icons you use
// With expo-vector-icons, use the specific icon component
import MaterialIcons from '@expo/vector-icons/MaterialIcons';

// Or even better: use SVG icons directly
import HomeSvg from './assets/icons/home.svg';
<HomeSvg width={24} height={24} fill="#000" />;
```

### Code Splitting with React.lazy

Metro now supports code splitting, which means you can split your bundle into chunks that load on demand:

```typescript
// app.config.js (Expo)
module.exports = {
  expo: {
    // Enable Metro code splitting
    experiments: {
      treeShaking: true,
    },
  },
};
```

```typescript
// Split heavy features into separate chunks
const AdminPanel = lazy(() => import('./features/admin/AdminPanel'));
const Analytics = lazy(() => import('./features/analytics/Analytics'));
const RichEditor = lazy(() => import('./features/editor/RichEditor'));

// These chunks only download when the user navigates to them
function AppNavigator() {
  return (
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen 
        name="Admin" 
        component={withSuspense(AdminPanel)} 
      />
      <Stack.Screen 
        name="Analytics" 
        component={withSuspense(Analytics)} 
      />
    </Stack.Navigator>
  );
}
```

### Import Cost Analysis

Install the "Import Cost" VS Code extension and be horrified. Then add bundle size checks to your CI pipeline:

```typescript
// .github/workflows/bundle-check.yml concept
// Track bundle size on every PR

// package.json
{
  "scripts": {
    "bundle:ios": "npx react-native bundle --platform ios --dev false --entry-file index.js --bundle-output /tmp/ios-bundle.js",
    "bundle:size": "wc -c < /tmp/ios-bundle.js",
    "bundle:check": "node scripts/check-bundle-size.js"
  }
}
```

```typescript
// scripts/check-bundle-size.js
const fs = require('fs');
const path = require('path');

const MAX_BUNDLE_SIZE = 15 * 1024 * 1024; // 15MB
const bundlePath = '/tmp/ios-bundle.js';

const stats = fs.statSync(bundlePath);
const sizeMB = (stats.size / (1024 * 1024)).toFixed(2);

console.log(`Bundle size: ${sizeMB}MB`);

if (stats.size > MAX_BUNDLE_SIZE) {
  console.error(`Bundle size ${sizeMB}MB exceeds limit of 15MB!`);
  process.exit(1);
}

console.log('Bundle size check passed.');
```

### Shopify's 50MB -> 11MB Journey

Shopify's mobile app went from 50MB to 11MB over 18 months. Here is what they did (from their public engineering blog and conference talks):

1. **Replaced moment.js with date-fns** (tree-shakeable): -2MB
2. **Removed unused native modules**: -8MB (camera, maps, and AR modules that were no longer used)
3. **Compressed assets** (images, fonts, animations): -12MB
4. **Code-split admin features** into downloadable bundles: -7MB
5. **Replaced Lottie animations** with Reanimated equivalents: -4MB
6. **Switched from AWS SDK v2 to v3** (modular): -3MB
7. **Removed polyfills** no longer needed with Hermes: -3MB

The key lesson: bundle size is usually not a single library problem. It is death by a thousand cuts, and you have to attack it from every angle.

### Metro Configuration for Smaller Bundles

```javascript
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

// Enable tree shaking (React Native 0.73+)
config.transformer.minifierConfig = {
  compress: {
    // Remove console.log in production
    drop_console: true,
    // Remove debugger statements
    drop_debugger: true,
    // Reduce size with these optimizations
    ecma: 2020,
    passes: 2,
  },
  mangle: {
    toplevel: true,
  },
};

// Exclude unnecessary file types from the bundle
config.resolver.assetExts = config.resolver.assetExts.filter(
  ext => !['mp4', 'mov', 'avi'].includes(ext)
);

module.exports = config;
```

---

## 13.7 Image Optimization

Images are typically the largest assets in a mobile app and the most common source of memory issues. A single unoptimized image can use more memory than the rest of your component tree combined.

### The Math of Image Memory

Here is something most developers do not realize: the file size of an image (what you see in your asset folder) has almost nothing to do with how much memory it uses when rendered.

A 200KB JPEG file that is 4000x3000 pixels uses this much memory when decoded:

```
4000 x 3000 x 4 bytes (RGBA) = 48,000,000 bytes = ~48MB
```

That is 48 megabytes of RAM for a single image. If you have a list showing 10 product photos at full resolution, you are using **480MB of RAM** just for images. Your app will be killed by the OS.

### expo-image — The Modern Image Component

`expo-image` is built on native SDKs (SDWebImage on iOS, Coil on Android) and handles all the hard problems: caching, progressive loading, format support, and memory management.

```typescript
import { Image } from 'expo-image';

// Basic usage with automatic caching
function ProductCard({ product }) {
  return (
    <Image
      source={{ uri: product.imageUrl }}
      style={{ width: 200, height: 200 }}
      contentFit="cover"
      transition={200}  // Smooth fade-in when loaded
    />
  );
}
```

### Why You Should Never Load Full-Resolution Images in Lists

```typescript
// BAD: Loading 4000x3000 images in a product grid
function ProductGrid({ products }) {
  return (
    <FlashList
      data={products}
      renderItem={({ item }) => (
        <Image
          source={{ uri: item.originalImageUrl }} // 4000x3000px!
          style={{ width: 180, height: 180 }}
          // The image is decoded at full resolution, then scaled down
          // Memory: 48MB per image x 10 visible = 480MB
        />
      )}
    />
  );
}

// GOOD: Request appropriately sized images from your CDN
function ProductGrid({ products }) {
  return (
    <FlashList
      data={products}
      renderItem={({ item }) => (
        <Image
          source={{ 
            uri: getResizedImageUrl(item.imageUrl, { 
              width: 360,  // 2x for retina (180pt x 2)
              height: 360,
              quality: 80,
            })
          }}
          style={{ width: 180, height: 180 }}
          contentFit="cover"
          // Memory: 0.5MB per image x 10 visible = 5MB
        />
      )}
    />
  );
}

// Helper: Most CDN services support URL-based resizing
function getResizedImageUrl(
  url: string, 
  opts: { width: number; height: number; quality?: number }
): string {
  // Cloudinary example
  const baseUrl = url.replace('/upload/', 
    `/upload/w_${opts.width},h_${opts.height},q_${opts.quality || 80},f_auto/`
  );
  return baseUrl;

  // Imgix example:
  // return `${url}?w=${opts.width}&h=${opts.height}&q=${opts.quality || 80}&auto=format`;
  
  // Cloudflare Images example:
  // return `${url}/w=${opts.width},h=${opts.height},quality=${opts.quality || 80},format=auto`;
}
```

The `f_auto` or `format=auto` parameter is important — it tells the CDN to serve WebP to devices that support it (which is all modern devices), which is 25-35% smaller than JPEG at the same quality.

### Caching Strategies

`expo-image` handles caching automatically, but you can control it:

```typescript
import { Image } from 'expo-image';

// Memory cache: recently viewed images, fast access
// Disk cache: persisted between sessions, slower access

function Avatar({ user }) {
  return (
    <Image
      source={{ uri: user.avatarUrl }}
      style={{ width: 48, height: 48, borderRadius: 24 }}
      // Cache policy
      cachePolicy="memory-disk"  // Default: cache in both memory and disk
      // Or "memory" for transient images
      // Or "disk" for images that rarely change
      // Or "none" for one-time-use images
      
      // Placeholder while loading
      placeholder={user.avatarBlurhash}
      placeholderContentFit="cover"
    />
  );
}
```

### Blurhash and Thumbhash Placeholders

Instead of showing a blank space or a spinner while images load, use blurhash or thumbhash to show a blurred preview. The hash is a tiny string (20-30 bytes) that can be stored with your data and renders instantly.

```typescript
// Your API returns blurhash strings with image data
// {
//   "imageUrl": "https://cdn.example.com/photo.jpg",
//   "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj"
// }

function FeedImage({ imageUrl, blurhash }) {
  return (
    <Image
      source={{ uri: imageUrl }}
      placeholder={{ blurhash }}  // Shows immediately while image loads
      contentFit="cover"
      transition={300}  // Smooth transition from blur to full image
      style={{ width: '100%', aspectRatio: 16 / 9 }}
    />
  );
}
```

The user experience difference is huge. Instead of content jumping around as images load (layout shift), the user sees a pleasing blurred placeholder that smoothly transitions to the full image.

### Progressive Loading Pattern

For hero images or detail screens where quality matters:

```typescript
function ProductDetailImage({ product }) {
  const [highResLoaded, setHighResLoaded] = useState(false);

  return (
    <View style={{ width: '100%', aspectRatio: 1 }}>
      {/* Low-res thumbnail loads instantly (already cached from list view) */}
      <Image
        source={{ 
          uri: getResizedImageUrl(product.imageUrl, { 
            width: 100, height: 100, quality: 30 
          })
        }}
        style={StyleSheet.absoluteFill}
        contentFit="cover"
        blurRadius={highResLoaded ? 0 : 2}
      />
      
      {/* High-res version loads on top */}
      <Image
        source={{ 
          uri: getResizedImageUrl(product.imageUrl, { 
            width: 800, height: 800, quality: 85 
          })
        }}
        style={StyleSheet.absoluteFill}
        contentFit="cover"
        onLoad={() => setHighResLoaded(true)}
        transition={300}
      />
    </View>
  );
}
```

### WebP Format

WebP provides 25-35% smaller files than JPEG at the same visual quality, and supports transparency (unlike JPEG). All modern mobile devices support WebP.

```typescript
// Configure your image CDN to serve WebP
// Most CDNs do this automatically with "auto format" options:
// - Cloudinary: f_auto
// - Imgix: auto=format
// - Cloudflare: format=auto
// - Fastly: auto=webp

// If you're bundling local images, convert them at build time:
// npx sharp-cli --input ./assets/images/*.png --output ./assets/images/ --format webp --quality 80
```

### Image Optimization Checklist

```
[ ] Using expo-image (not React Native's Image component)
[ ] Images are resized server-side to match display size x device pixel ratio
[ ] WebP format served where supported
[ ] Blurhash/thumbhash placeholders for all network images
[ ] Disk caching enabled for images that persist
[ ] No full-resolution images in lists (max 2x display size)
[ ] Lazy loading for below-fold images
[ ] Progressive loading for hero/detail images
[ ] Image memory monitored in production (Hermes heap snapshots)
```

---

## 13.8 Memory Management

Memory leaks are the silent killers of mobile apps. Your app works fine for 5 minutes. After 20 minutes of browsing, it starts getting sluggish. After 45 minutes, it crashes. The user blames your app, not the leak.

### Common Leak Patterns

**1. Closures Over State in Event Listeners**

```typescript
// LEAK: The interval closure captures `messages` state
// Every new message creates a new closure, but the interval
// holds a reference to the old closure (and its old state)
function ChatScreen() {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const interval = setInterval(() => {
      // This closure captures the initial `messages` array
      // As messages grow, old references are never freed
      fetchNewMessages().then(newMessages => {
        setMessages([...messages, ...newMessages]); // Bug: stale closure
      });
    }, 5000);

    // LEAK: No cleanup!
  }, []); // Empty deps = closure never updates

  return <MessageList messages={messages} />;
}

// FIXED: Proper cleanup and functional state updates
function ChatScreen() {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const interval = setInterval(() => {
      fetchNewMessages().then(newMessages => {
        // Functional update: no closure over state needed
        setMessages(prev => [...prev, ...newMessages]);
      });
    }, 5000);

    return () => clearInterval(interval); // Cleanup!
  }, []);

  return <MessageList messages={messages} />;
}
```

**2. Uncleared Listeners and Subscriptions**

```typescript
// LEAK: WebSocket connection never closed
function LiveFeed() {
  const [updates, setUpdates] = useState([]);

  useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/feed');
    
    ws.onmessage = (event) => {
      setUpdates(prev => [...prev, JSON.parse(event.data)]);
    };

    // LEAK: WebSocket stays open when component unmounts!
    // If the user navigates away and back 5 times,
    // there are now 5 open WebSocket connections
  }, []);

  return <FeedList data={updates} />;
}

// FIXED: Close WebSocket on unmount
function LiveFeed() {
  const [updates, setUpdates] = useState([]);

  useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/feed');
    
    ws.onmessage = (event) => {
      setUpdates(prev => [...prev, JSON.parse(event.data)]);
    };

    return () => {
      ws.close(); // Clean up!
    };
  }, []);

  return <FeedList data={updates} />;
}
```

**3. Navigation Stacks Holding References**

This is a subtle one. In a stack navigator, when you navigate from Screen A to Screen B, Screen A is not unmounted — it stays in the stack. If Screen A holds a large data set in state, that memory is retained even though the user is looking at Screen B.

```typescript
// LEAK: Navigation stack retains screen state
function ProductListScreen() {
  const [products, setProducts] = useState<Product[]>([]); // 5,000 products
  const [images, setImages] = useState<Map<string, ImageData>>(); // Cached images

  // User navigates to ProductDetail → this screen stays mounted
  // 5,000 products + cached images stay in memory
  // User navigates to another ProductList → another copy in memory
  // After 5 navigation levels deep, you've got 25,000 products in memory
}

// FIXED: Clean up when screen loses focus
import { useFocusEffect } from '@react-navigation/native';

function ProductListScreen() {
  const [products, setProducts] = useState<Product[]>([]);
  const scrollPosition = useRef(0);

  useFocusEffect(
    useCallback(() => {
      // Load data when screen focuses
      loadProducts().then(setProducts);

      return () => {
        // Clear heavy data when screen loses focus
        // Keep scroll position for when user comes back
        setProducts([]);
      };
    }, [])
  );

  return (
    <FlashList 
      data={products} 
      onScroll={(e) => {
        scrollPosition.current = e.nativeEvent.contentOffset.y;
      }}
      // ... 
    />
  );
}
```

**4. Accumulated Arrays That Never Shrink**

```typescript
// LEAK: Analytics events accumulate forever
const analyticsQueue: AnalyticsEvent[] = [];

function trackEvent(event: AnalyticsEvent) {
  analyticsQueue.push(event);
  // These are never flushed!
  // After an hour of use, you might have 10,000+ events in memory
}

// FIXED: Flush periodically and cap the queue
const MAX_QUEUE_SIZE = 100;
let analyticsQueue: AnalyticsEvent[] = [];

function trackEvent(event: AnalyticsEvent) {
  analyticsQueue.push(event);
  
  if (analyticsQueue.length >= MAX_QUEUE_SIZE) {
    flushAnalytics();
  }
}

async function flushAnalytics() {
  const events = analyticsQueue;
  analyticsQueue = []; // Clear immediately
  
  try {
    await api.sendAnalytics(events);
  } catch (error) {
    // Re-queue failed events (with a cap)
    analyticsQueue = [...events.slice(-50), ...analyticsQueue];
  }
}

// Flush on a timer and on app background
setInterval(flushAnalytics, 30000);
AppState.addEventListener('change', (state) => {
  if (state === 'background') flushAnalytics();
});
```

### Detecting Leaks with Hermes Heap Snapshots

Hermes (the JavaScript engine in React Native) provides heap snapshot tools that let you see exactly what is in memory:

```typescript
// Take a heap snapshot programmatically
import { HermesInternal } from 'react-native';

function takeHeapSnapshot() {
  if (__DEV__ && global.HermesInternal) {
    // This creates a .heapsnapshot file you can load in Chrome DevTools
    const filename = `/tmp/heap-${Date.now()}.heapsnapshot`;
    global.HermesInternal.takeHeapSnapshot(filename);
    console.log(`Heap snapshot saved to ${filename}`);
  }
}

// Take snapshots at key moments to compare
// 1. After app startup
// 2. After navigating to a heavy screen
// 3. After navigating away from that screen
// 4. After navigating back and forth 5 times
// If memory keeps growing after step 4, you have a leak
```

To analyze the snapshots:

1. Open Chrome DevTools
2. Go to the Memory tab
3. Load the `.heapsnapshot` file
4. Compare snapshots: look for objects that grow between snapshots
5. Use "Retained Size" to find the biggest memory consumers
6. Follow the retainer chain to find what is keeping objects alive

### useEffect Cleanup Pattern — The Complete Version

Here is the cleanup pattern I use for every `useEffect` that creates subscriptions or resources:

```typescript
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');

  useEffect(() => {
    let isActive = true; // Guard against updates after unmount
    let ws: WebSocket | null = null;
    let reconnectTimer: ReturnType<typeof setTimeout> | null = null;

    async function connect() {
      try {
        ws = new WebSocket(`wss://api.example.com/rooms/${roomId}`);

        ws.onopen = () => {
          if (isActive) setConnectionStatus('connected');
        };

        ws.onmessage = (event) => {
          if (!isActive) return; // Don't update state if unmounted
          
          const message = JSON.parse(event.data);
          setMessages(prev => {
            // Cap at 500 messages to prevent memory growth
            const updated = [...prev, message];
            return updated.length > 500 ? updated.slice(-500) : updated;
          });
        };

        ws.onclose = () => {
          if (isActive) {
            setConnectionStatus('disconnected');
            // Reconnect after 3 seconds
            reconnectTimer = setTimeout(connect, 3000);
          }
        };

        ws.onerror = () => {
          ws?.close();
        };
      } catch (error) {
        if (isActive) {
          setConnectionStatus('disconnected');
          reconnectTimer = setTimeout(connect, 3000);
        }
      }
    }

    connect();

    // Cleanup: close everything, prevent state updates
    return () => {
      isActive = false;
      
      if (ws) {
        ws.onopen = null;
        ws.onmessage = null;
        ws.onclose = null;
        ws.onerror = null;
        ws.close();
      }
      
      if (reconnectTimer) {
        clearTimeout(reconnectTimer);
      }
    };
  }, [roomId]);

  return (
    <View>
      <ConnectionIndicator status={connectionStatus} />
      <MessageList messages={messages} />
    </View>
  );
}
```

The `isActive` flag is critical. Without it, if the component unmounts while an async operation is in flight, the callback will try to call `setState` on an unmounted component. This does not crash (React ignores it), but it means the closure is keeping the component's state in memory longer than necessary.

### Memory Monitoring in Production

```typescript
// utils/memoryMonitor.ts
import { AppState, Platform } from 'react-native';

interface MemorySnapshot {
  jsHeapSize: number;
  timestamp: number;
  screen: string;
}

class MemoryMonitor {
  private snapshots: MemorySnapshot[] = [];
  private currentScreen = 'unknown';
  private warningThreshold = 150 * 1024 * 1024; // 150MB
  private criticalThreshold = 200 * 1024 * 1024; // 200MB

  setCurrentScreen(screen: string) {
    this.currentScreen = screen;
  }

  takeSnapshot() {
    if (!global.performance?.memory) return;

    const snapshot: MemorySnapshot = {
      jsHeapSize: (global.performance as any).memory?.usedJSHeapSize || 0,
      timestamp: Date.now(),
      screen: this.currentScreen,
    };

    this.snapshots.push(snapshot);

    // Keep last 100 snapshots
    if (this.snapshots.length > 100) {
      this.snapshots = this.snapshots.slice(-100);
    }

    // Check thresholds
    if (snapshot.jsHeapSize > this.criticalThreshold) {
      this.reportCriticalMemory(snapshot);
    } else if (snapshot.jsHeapSize > this.warningThreshold) {
      this.reportHighMemory(snapshot);
    }

    return snapshot;
  }

  getMemoryTrend(): 'stable' | 'growing' | 'leaking' {
    if (this.snapshots.length < 10) return 'stable';

    const recent = this.snapshots.slice(-10);
    const growthRate = 
      (recent[recent.length - 1].jsHeapSize - recent[0].jsHeapSize) / 
      (recent[recent.length - 1].timestamp - recent[0].timestamp);

    // Growing more than 1MB per minute = probable leak
    if (growthRate > 1024 * 1024 / 60000) return 'leaking';
    if (growthRate > 0) return 'growing';
    return 'stable';
  }

  private reportCriticalMemory(snapshot: MemorySnapshot) {
    analytics.track('memory_critical', {
      heapSize: snapshot.jsHeapSize,
      screen: snapshot.screen,
      trend: this.getMemoryTrend(),
    });
  }

  private reportHighMemory(snapshot: MemorySnapshot) {
    analytics.track('memory_warning', {
      heapSize: snapshot.jsHeapSize,
      screen: snapshot.screen,
    });
  }
}

export const memoryMonitor = new MemoryMonitor();

// Take snapshots periodically
setInterval(() => memoryMonitor.takeSnapshot(), 30000);

// Take snapshot on app state changes
AppState.addEventListener('change', (state) => {
  if (state === 'active') {
    memoryMonitor.takeSnapshot();
  }
});
```

### Memory Cleanup Strategies

```typescript
// Strategy 1: Clear state when navigating away
import { useFocusEffect } from '@react-navigation/native';

function HeavyScreen() {
  const [data, setData] = useState(null);

  useFocusEffect(
    useCallback(() => {
      loadData().then(setData);
      return () => setData(null); // Free memory on blur
    }, [])
  );
}

// Strategy 2: Limit cache sizes
const imageCache = new Map<string, ImageData>();
const MAX_CACHE_SIZE = 50;

function cacheImage(key: string, data: ImageData) {
  if (imageCache.size >= MAX_CACHE_SIZE) {
    // Remove oldest entry (first inserted)
    const firstKey = imageCache.keys().next().value;
    imageCache.delete(firstKey);
  }
  imageCache.set(key, data);
}

// Strategy 3: Use WeakRef for optional caches
const optionalCache = new Map<string, WeakRef<object>>();

function getCached(key: string): object | undefined {
  const ref = optionalCache.get(key);
  if (ref) {
    const value = ref.deref();
    if (value) return value;
    // Reference was garbage collected, clean up
    optionalCache.delete(key);
  }
  return undefined;
}
```

---

## 13.9 React Compiler Deep Dive

We touched on the React Compiler in the re-render section, but it deserves deeper coverage because it fundamentally changes how you think about React Native performance.

### What the React Compiler Does

The React Compiler is a build-time transform that analyzes your React code and automatically inserts memoization. It understands React's rules (components are pure functions of props, hooks follow the rules of hooks) and uses that understanding to determine what can be safely memoized.

**Input (what you write):**

```typescript
function UserProfile({ userId }) {
  const [isEditing, setIsEditing] = useState(false);
  const user = useUser(userId);

  const fullName = `${user.firstName} ${user.lastName}`;
  const initials = user.firstName[0] + user.lastName[0];

  const handleEdit = () => setIsEditing(true);
  const handleSave = (data) => {
    updateUser(userId, data);
    setIsEditing(false);
  };

  return (
    <View>
      <Avatar initials={initials} imageUrl={user.avatar} />
      <Text>{fullName}</Text>
      {isEditing ? (
        <EditForm user={user} onSave={handleSave} />
      ) : (
        <Button onPress={handleEdit} title="Edit" />
      )}
    </View>
  );
}
```

**Output (what the compiler produces, conceptually):**

```typescript
function UserProfile({ userId }) {
  const [isEditing, setIsEditing] = useState(false);
  const user = useUser(userId);

  // Compiler memoizes derived values
  const fullName = useMemo(
    () => `${user.firstName} ${user.lastName}`,
    [user.firstName, user.lastName]
  );
  const initials = useMemo(
    () => user.firstName[0] + user.lastName[0],
    [user.firstName, user.lastName]
  );

  // Compiler memoizes callbacks
  const handleEdit = useCallback(
    () => setIsEditing(true),
    []
  );
  const handleSave = useCallback(
    (data) => {
      updateUser(userId, data);
      setIsEditing(false);
    },
    [userId]
  );

  // Compiler memoizes JSX elements
  const avatarElement = useMemo(
    () => <Avatar initials={initials} imageUrl={user.avatar} />,
    [initials, user.avatar]
  );

  const nameElement = useMemo(
    () => <Text>{fullName}</Text>,
    [fullName]
  );

  return (
    <View>
      {avatarElement}
      {nameElement}
      {isEditing ? (
        <EditForm user={user} onSave={handleSave} />
      ) : (
        <Button onPress={handleEdit} title="Edit" />
      )}
    </View>
  );
}
```

Notice that the compiler memoizes at a finer granularity than you would manually. It does not just wrap the whole component in `React.memo` — it memoizes individual JSX elements. This means when `isEditing` changes, only the conditional part re-renders. The `Avatar` and `Text` elements are untouched because their memoized dependencies have not changed.

### What the Compiler Does NOT Do

This is just as important as what it does:

**1. It does not fix algorithmic complexity.**

```typescript
// The compiler will memoize this, but it's still O(n^2)
function SlowComponent({ items }) {
  // Compiler adds: useMemo(..., [items])
  // But if items changes, this still runs O(n^2)
  const pairs = items.flatMap((a, i) =>
    items.slice(i + 1).map(b => [a, b])
  );

  return <PairList pairs={pairs} />;
}

// The fix is algorithmic, not memoization
function FastComponent({ items }) {
  const pairs = useMemo(() => {
    // Use a more efficient algorithm or pre-compute on the server
    return computePairsEfficiently(items);
  }, [items]);

  return <PairList pairs={pairs} />;
}
```

**2. It does not optimize data fetching or network calls.**

```typescript
// Compiler cannot help here — the problem is making too many requests
function BadComponent({ ids }) {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    // N+1 query problem — one request per ID
    Promise.all(ids.map(id => fetchItem(id))).then(setData);
  }, [ids]);
}

// Fix: Batch requests
function GoodComponent({ ids }) {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    fetchItemsBatch(ids).then(setData); // Single batched request
  }, [ids]);
}
```

**3. It does not reduce the size of your component tree.**

If you render 1,000 components, the compiler makes each one faster to re-render, but the initial render still creates 1,000 components. Virtualization is still necessary for large lists.

**4. It does not help with native thread bottlenecks.**

If your animation stutters because the UI thread is overloaded with layout calculations, the compiler cannot help — it only optimizes the JS thread.

### How to Verify the Compiler Is Working

**Method 1: React DevTools**

Open React DevTools and look for the compiler badge (a small icon) next to component names. Compiled components show this badge.

**Method 2: Build output inspection**

```bash
# Check for compiler runtime import in the bundle
npx react-native bundle \
  --platform ios \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js

# Search for the compiler runtime
grep -c "react/compiler-runtime" /tmp/bundle.js
# If > 0, the compiler is active
```

**Method 3: Performance comparison**

```typescript
// Create a stress test component
function CompilerStressTest() {
  const [count, setCount] = useState(0);
  const startTime = useRef(Date.now());
  const renderCount = useRef(0);

  renderCount.current++;

  // This creates many derived values and JSX elements
  // that the compiler should memoize
  const items = Array.from({ length: 100 }, (_, i) => ({
    id: i,
    label: `Item ${i}`,
    isEven: i % 2 === 0,
  }));

  const evenItems = items.filter(i => i.isEven);
  const oddItems = items.filter(i => !i.isEven);

  return (
    <View>
      <Text>Renders: {renderCount.current}</Text>
      <Text>
        Render time: {Date.now() - startTime.current}ms
      </Text>
      <Button 
        title="Trigger re-render" 
        onPress={() => setCount(c => c + 1)} 
      />
      {/* With compiler: evenItems and oddItems are memoized
          Without: they're recreated every render */}
      <ItemList items={evenItems} label="Even" />
      <ItemList items={oddItems} label="Odd" />
    </View>
  );
}
```

### Compiler Compatibility

The React Compiler works with standard React patterns. It does not work with:

- **Non-standard React patterns** (mutating props, reading `this` in function components)
- **Escape hatches** like `useRef` for storing render-relevant data (the compiler treats refs as opaque)
- **Libraries that break React's rules** (some older animation libraries, certain form libraries)

If a component cannot be compiled, the compiler silently skips it — your app still works, you just do not get the optimization for that component. Check the compiler's diagnostic output for skipped components:

```javascript
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      ['babel-plugin-react-compiler', {
        // Enable diagnostics to see what the compiler skips
        panicThreshold: 'NONE', // Report all issues, don't bail out
      }],
    ],
  };
};
```

---

## Putting It All Together — A Performance Optimization Playbook

Here is the order I attack performance in any React Native app. This is not theory — this is the sequence that gives you the biggest impact for the least effort.

### Phase 1: Measure (Day 1)

```
1. Set up performance monitoring (cold start, FPS, memory)
2. Identify your bottom-of-the-barrel device (test on it)
3. Record baseline metrics
4. Generate a bundle analysis
```

### Phase 2: Quick Wins (Days 2-3)

```
1. Enable Hermes bytecode (if not already)
2. Enable React Compiler (if not already)
3. Replace FlatList with FlashList
4. Replace moment.js with day.js
5. Replace full lodash with individual imports
6. Add useNativeDriver: true to existing Animated animations
7. Add image resizing to your CDN URLs
```

These six changes typically yield a 30-50% improvement across all metrics.

### Phase 3: Targeted Optimization (Week 2)

```
1. Implement lazy loading for non-critical screens
2. Defer below-fold content on key screens
3. Add blurhash placeholders for images
4. Fix Zustand store subscriptions (add selectors)
5. Audit and fix useEffect cleanup patterns
6. Switch heavy animations to Reanimated
```

### Phase 4: Deep Optimization (Weeks 3-4)

```
1. Implement code splitting for large features
2. Profile memory usage across navigation paths
3. Optimize FlashList configuration (estimatedItemSize, getItemType)
4. Pre-compute data transformations outside renderItem
5. Implement progressive image loading
6. Add memory monitoring to production analytics
```

### Performance Budget Enforcement

Add these checks to your CI pipeline:

```typescript
// performance-budget.ts
const BUDGETS = {
  bundleSize: 15 * 1024 * 1024,     // 15MB
  coldStart: 1500,                    // 1.5s (measured on CI device farm)
  jsThreadFrameTime: 16,              // 16ms
  memoryBaseline: 200 * 1024 * 1024,  // 200MB
};

// In your PR checks:
// 1. Bundle size check (automated)
// 2. Startup time check (device farm)
// 3. Scroll performance check (device farm)
// 4. Memory growth check (automated scenario test)
```

---

## Key Takeaways

1. **Measure first.** Without numbers, you are guessing. Set up monitoring before you optimize.

2. **The React Compiler is not optional.** Enable it. It eliminates an entire class of performance bugs (unnecessary re-renders) automatically.

3. **FlashList is not optional.** If you have lists with more than 50 items, use FlashList. The recycling architecture is fundamentally superior to FlatList's create-destroy model.

4. **Animations must run on the UI thread.** Use Reanimated for anything beyond simple opacity/transform animations. Combine with Gesture Handler for gesture-driven interactions.

5. **Images are the #1 memory consumer.** Never load full-resolution images in lists. Use CDN resizing, WebP format, and blurhash placeholders.

6. **Memory leaks are cumulative.** A small leak is invisible for 5 minutes and catastrophic after 30 minutes. Every `useEffect` that creates a resource needs a cleanup function.

7. **Test on low-end devices.** If it runs smooth on a Galaxy A13, it runs smooth everywhere. If you only test on an iPhone 15 Pro, you are lying to yourself.

8. **Bundle size compounds.** Every unnecessary library you add costs download time, parse time, and memory. Be ruthless about what goes into your bundle.

The difference between a "React Native app" and an app that "happens to be built with React Native" is performance. Users cannot see your architecture. They cannot see your clean code. They can *feel* whether your app is fast. Make it fast.

---

*Next up: Chapter 14 — Testing Strategy, where we cover how to test these performance characteristics automatically and prevent regressions before they ship.*
