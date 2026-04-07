<!--
  CHAPTER: 10
  TITLE: App Lifecycle, Background Tasks & Push Notifications
  PART: II — React Native & Expo
  PREREQS: Chapters 6, 7
  KEY_TOPICS: AppState, splash screen, background fetch, background processing, push notifications, FCM, APNs, Expo Notifications, local notifications, notification channels, deep link from notification, foreground/background/killed handling
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 10: App Lifecycle, Background Tasks & Push Notifications

> **Part II — React Native & Expo** | Prerequisites: Chapters 6, 7 | Difficulty: Intermediate → Advanced

<details>
<summary><strong>TL;DR — App Lifecycle</strong></summary>

- `AppState` has three states: `active`, `background`, and `inactive` (iOS only, during transitions). Listen to changes to pause timers, refresh stale data on foreground, and disconnect sockets on background.
- `expo-splash-screen` with `SplashScreen.preventAutoHideAsync()` keeps the splash visible until your data loads; hiding it before your UI is ready causes a white flash that makes your app feel broken.
- The app startup sequence (tap → native init → JS bundle load → React render → first paint) has optimization points at every stage; lazy initialization and deferred work can cut startup time by 40-60%.
- Background tasks on mobile are fundamentally hostile territory: iOS kills background tasks aggressively (you get ~30 seconds), Android Doze mode throttles everything, and both platforms lie to you about when your code will actually run.
- `expo-background-fetch` and `expo-task-manager` give you periodic background execution, but expect your minimum interval to be 15+ minutes and accept that the OS will ignore your schedule whenever it feels like it.
- Force-update patterns are a safety net every production app needs; ship the mechanism before you need it, because when you need it, it's already too late to ship it.

</details>

<details>
<summary><strong>TL;DR — Push Notifications</strong></summary>

- The push pipeline is: your server → FCM/APNs → device OS → your app. Every hop can fail silently, so instrument every step.
- Expo Push Service simplifies sending by abstracting FCM and APNs behind one API, but you trade control for convenience; at scale, you may want direct FCM/APNs access.
- Every app must handle three notification states: foreground (app is open — show custom UI or suppress), background (OS shows the notification), and killed/cold start (user taps notification to open app — you must navigate to the right screen).
- Android notification channels (required since Android 8) are permanent once created; users can disable them individually, and you cannot change importance level after creation. Plan your channel strategy before your first release.
- Deep linking from notifications is the #1 source of "it works on my machine" bugs: cold start deep links follow a completely different code path than warm deep links, and if you only test one, the other is broken.
- Token management is a silent reliability killer: tokens expire, users reinstall, devices change. If you're not pruning invalid tokens, you're sending 10-30% of your pushes into the void.

</details>

---

There are two moments that define whether your mobile app feels professional or amateur. The first is what happens in the three seconds between tapping the icon and seeing content. The second is what happens when a notification arrives and the user taps it.

Get the lifecycle right and your app feels instant, alive, and trustworthy. Get notifications right and you have the most powerful re-engagement channel in existence — one that lives in the user's pocket, on their wrist, and on their lock screen.

Get either of them wrong and... well, I've watched a 4.8-star app drop to 3.2 stars in two weeks because a bad update created a splash screen loop. I've watched a billion-dollar company lose 15% of their daily active users because push notifications deep-linked to a blank screen. These aren't edge cases. These are the things that separate apps that grow from apps that churn.

This chapter covers both topics in depth because they're deeply intertwined. Background tasks power silent push updates. App state determines how notifications are displayed. The startup sequence determines whether a cold-start deep link works or crashes. You can't understand one without the other.

### In This Chapter

**Part A — App Lifecycle:**
- AppState API: Active, Background, Inactive
- Splash Screen Management with expo-splash-screen
- App Startup Sequence and Optimization
- Background Tasks with expo-background-fetch and expo-task-manager
- Keep-Alive Patterns: What iOS and Android Actually Allow
- App Updates: Force Update Patterns and Version Checking

**Part B — Push Notifications:**
- Push Architecture: FCM, APNs, and the End-to-End Pipeline
- Expo Notifications Setup: Permissions, Tokens, Configuration
- Handling the Three Notification States
- Notification Payloads: Title, Body, Data, Actions, Images
- Local and Scheduled Notifications
- Android Notification Channels
- Deep Linking from Notifications
- Testing Notifications: Simulators, Devices, and Gotchas
- Server-Side Sending: Expo Push API vs Direct FCM/APNs

### Related Chapters
- [Ch 6: EAS Mastery] — Build profiles and OTA updates that interact with force-update logic
- [Ch 7: Navigation Architecture] — Deep link routing that notification taps depend on
- [Ch 5: Expo Platform] — Expo config plugins for notification setup
- [Ch 13: Performance Optimization] — Startup performance and bundle optimization

---

# PART A: APP LIFECYCLE

---

## 1. APPSTATE API: ACTIVE, BACKGROUND, INACTIVE

The `AppState` API is deceptively simple — three states, one event listener — but it's the foundation for everything else in this chapter. Every pattern we'll discuss (splash screens, background tasks, notification handling, data refresh) depends on knowing what state your app is in.

### 1.1 The Three States

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   ┌──────────┐    ┌──────────┐    ┌──────────────┐  │
│   │  active   │◄──►│ inactive │──►│  background  │  │
│   │          │    │ (iOS only)│    │              │  │
│   └──────────┘    └──────────┘    └──────────────┘  │
│        ▲                               │            │
│        │                               │            │
│        └───────────────────────────────┘            │
│           (user returns to app)                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

- **`active`** — The app is in the foreground and receiving events. This is the normal state when the user is interacting with your app.
- **`inactive`** (iOS only) — A brief transitional state. Happens when pulling down the notification center, when the app switcher is open, or during an incoming phone call. On Android, you go straight from `active` to `background`.
- **`background`** — The app is in the background. It may still be running code briefly (iOS gives you ~30 seconds), but the OS can suspend or kill it at any time.

**The critical insight:** On iOS, `inactive` is a transitional state, not a destination. Don't treat it as "the user left" — they might just be swiping down the notification shade and will return in 200ms. On Android, there is no `inactive` state; you go directly from `active` to `background`.

### 1.2 Listening to State Changes

```tsx
// hooks/useAppState.ts
import { useEffect, useRef, useState } from 'react';
import { AppState, AppStateStatus } from 'react-native';

/**
 * Hook that returns the current AppState and provides
 * callbacks for foreground/background transitions.
 */
export function useAppState() {
  const appState = useRef(AppState.currentState);
  const [currentState, setCurrentState] = useState<AppStateStatus>(
    AppState.currentState
  );

  useEffect(() => {
    const subscription = AppState.addEventListener(
      'change',
      (nextAppState: AppStateStatus) => {
        appState.current = nextAppState;
        setCurrentState(nextAppState);
      }
    );

    return () => {
      subscription.remove();
    };
  }, []);

  return currentState;
}
```

```tsx
// hooks/useOnForeground.ts
import { useEffect, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';

/**
 * Runs a callback when the app returns to the foreground.
 * This is the pattern you'll use most often.
 */
export function useOnForeground(callback: () => void) {
  const appState = useRef(AppState.currentState);

  useEffect(() => {
    const subscription = AppState.addEventListener(
      'change',
      (nextAppState: AppStateStatus) => {
        // Coming from background/inactive TO active
        if (
          appState.current.match(/inactive|background/) &&
          nextAppState === 'active'
        ) {
          callback();
        }
        appState.current = nextAppState;
      }
    );

    return () => {
      subscription.remove();
    };
  }, [callback]);
}
```

```tsx
// hooks/useOnBackground.ts
import { useEffect, useRef } from 'react';
import { AppState, AppStateStatus } from 'react-native';

/**
 * Runs a callback when the app moves to the background.
 */
export function useOnBackground(callback: () => void) {
  const appState = useRef(AppState.currentState);

  useEffect(() => {
    const subscription = AppState.addEventListener(
      'change',
      (nextAppState: AppStateStatus) => {
        // Going from active TO background
        if (
          appState.current === 'active' &&
          nextAppState.match(/inactive|background/)
        ) {
          // On iOS, we get 'inactive' first, then 'background'
          // We fire on the first transition away from 'active'
          if (appState.current === 'active') {
            callback();
          }
        }
        appState.current = nextAppState;
      }
    );

    return () => {
      subscription.remove();
    };
  }, [callback]);
}
```

### 1.3 Common AppState Patterns

**Pattern 1: Refresh Data on Foreground**

This is the single most important AppState pattern. When a user returns to your app after being away, the data on screen is probably stale. Refresh it.

```tsx
// screens/DashboardScreen.tsx
import { useCallback } from 'react';
import { useOnForeground } from '../hooks/useOnForeground';
import { useQuery, useQueryClient } from '@tanstack/react-query';

function DashboardScreen() {
  const queryClient = useQueryClient();
  
  const { data, isLoading } = useQuery({
    queryKey: ['dashboard'],
    queryFn: fetchDashboardData,
    // Don't refetch on window focus automatically —
    // we control this ourselves with AppState
    refetchOnWindowFocus: false,
  });

  // Refresh dashboard data when app comes to foreground
  const handleForeground = useCallback(() => {
    queryClient.invalidateQueries({ queryKey: ['dashboard'] });
  }, [queryClient]);

  useOnForeground(handleForeground);

  // ... render
}
```

**Pattern 2: Pause/Resume Timers**

If you have countdown timers, polling intervals, or animation loops, pause them when the app goes to background. Otherwise you're wasting CPU cycles (and battery) while the user isn't looking.

```tsx
// hooks/usePausableInterval.ts
import { useEffect, useRef, useCallback } from 'react';
import { AppState, AppStateStatus } from 'react-native';

export function usePausableInterval(
  callback: () => void,
  intervalMs: number
) {
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
  const callbackRef = useRef(callback);
  callbackRef.current = callback;

  const start = useCallback(() => {
    if (intervalRef.current) return; // Already running
    intervalRef.current = setInterval(() => {
      callbackRef.current();
    }, intervalMs);
  }, [intervalMs]);

  const stop = useCallback(() => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }, []);

  useEffect(() => {
    // Start the interval
    start();

    // Listen for app state changes
    const subscription = AppState.addEventListener(
      'change',
      (state: AppStateStatus) => {
        if (state === 'active') {
          start();
        } else {
          stop();
        }
      }
    );

    return () => {
      stop();
      subscription.remove();
    };
  }, [start, stop]);

  return { start, stop };
}
```

**Pattern 3: WebSocket Connection Management**

Don't keep WebSocket connections open in the background. The OS will throttle or kill them anyway, and you're draining battery for nothing.

```tsx
// hooks/useBackgroundAwareWebSocket.ts
import { useEffect, useRef, useCallback } from 'react';
import { AppState, AppStateStatus } from 'react-native';

interface UseWebSocketOptions {
  url: string;
  onMessage: (data: any) => void;
  onError?: (error: Event) => void;
}

export function useBackgroundAwareWebSocket({
  url,
  onMessage,
  onError,
}: UseWebSocketOptions) {
  const wsRef = useRef<WebSocket | null>(null);
  const appStateRef = useRef(AppState.currentState);

  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    const ws = new WebSocket(url);
    ws.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        onMessage(data);
      } catch {
        onMessage(event.data);
      }
    };
    ws.onerror = (error) => onError?.(error);
    ws.onclose = () => {
      wsRef.current = null;
    };
    wsRef.current = ws;
  }, [url, onMessage, onError]);

  const disconnect = useCallback(() => {
    if (wsRef.current) {
      wsRef.current.close();
      wsRef.current = null;
    }
  }, []);

  useEffect(() => {
    connect();

    const subscription = AppState.addEventListener(
      'change',
      (nextState: AppStateStatus) => {
        const previous = appStateRef.current;
        appStateRef.current = nextState;

        if (
          previous.match(/inactive|background/) &&
          nextState === 'active'
        ) {
          // Returning to foreground — reconnect
          connect();
        } else if (
          previous === 'active' &&
          nextState.match(/inactive|background/)
        ) {
          // Going to background — disconnect
          disconnect();
        }
      }
    );

    return () => {
      disconnect();
      subscription.remove();
    };
  }, [connect, disconnect]);

  return { disconnect, reconnect: connect };
}
```

**Pattern 4: Track Time in Background**

Sometimes you need to know how long the user was away. Did they leave for 5 seconds (probably don't need to refresh) or 30 minutes (definitely refresh, maybe re-authenticate)?

```tsx
// hooks/useBackgroundDuration.ts
import { useEffect, useRef, useCallback } from 'react';
import { AppState, AppStateStatus } from 'react-native';

interface BackgroundDurationOptions {
  /** Called when returning to foreground with duration in ms */
  onReturn: (durationMs: number) => void;
  /** If set, only calls onReturn when background duration exceeds this threshold */
  thresholdMs?: number;
}

export function useBackgroundDuration({
  onReturn,
  thresholdMs = 0,
}: BackgroundDurationOptions) {
  const backgroundTimestamp = useRef<number | null>(null);
  const appStateRef = useRef(AppState.currentState);

  useEffect(() => {
    const subscription = AppState.addEventListener(
      'change',
      (nextState: AppStateStatus) => {
        const previous = appStateRef.current;
        appStateRef.current = nextState;

        if (previous === 'active' && nextState !== 'active') {
          // Going to background — record timestamp
          backgroundTimestamp.current = Date.now();
        }

        if (
          previous.match(/inactive|background/) &&
          nextState === 'active' &&
          backgroundTimestamp.current
        ) {
          // Returning to foreground — calculate duration
          const duration = Date.now() - backgroundTimestamp.current;
          backgroundTimestamp.current = null;

          if (duration >= thresholdMs) {
            onReturn(duration);
          }
        }
      }
    );

    return () => subscription.remove();
  }, [onReturn, thresholdMs]);
}

// Usage: Re-authenticate if away for more than 5 minutes
function App() {
  useBackgroundDuration({
    onReturn: (durationMs) => {
      console.log(`User was away for ${durationMs / 1000}s — re-authenticating`);
      authStore.refreshSession();
    },
    thresholdMs: 5 * 60 * 1000, // 5 minutes
  });
  
  // ...
}
```

### 1.4 AppState Gotchas

**Gotcha 1: iOS `inactive` fires for everything.**
Pulling down the notification shade, incoming call overlay, Face ID prompt — all trigger `inactive`. Don't treat it as "user left the app."

**Gotcha 2: `background` doesn't mean "the app is still running."**
On iOS, after ~30 seconds in background, the OS suspends your app. Your JS code stops executing entirely. On Android, with Doze mode and battery optimization, background execution is even more restricted.

**Gotcha 3: Android has no `inactive` state.**
On Android, you go directly from `active` to `background`. If you write logic that depends on the `inactive` intermediate state, it will never fire on Android.

**Gotcha 4: The initial AppState might not be `active`.**
If your app is opened via a notification or deep link while backgrounded, the initial state might be `background` before transitioning to `active`. Always check `AppState.currentState` at mount time.

```tsx
// Don't do this:
const appState = useRef('active'); // Assuming initial state

// Do this:
const appState = useRef(AppState.currentState); // Read actual state
```

---

## 2. SPLASH SCREEN MANAGEMENT

The splash screen is the first thing users see after tapping your app icon. It's also one of the most common sources of perceived performance problems. A bad splash screen experience (too long, white flash between splash and content, flicker) makes your app feel slow even if it isn't.

### 2.1 How Splash Screens Work in Expo

Expo's splash screen system has two parts:

1. **Native splash screen** — A static image configured in `app.json`/`app.config.ts`. This is shown by the OS itself the instant the app launches, before any JavaScript runs. It's fast because it's native.
2. **expo-splash-screen API** — JavaScript control over when the native splash screen hides. By default, it hides as soon as the root component renders. You can override this to keep it visible until your data loads.

```
Tap icon → OS shows native splash → JS bundle loads → React mounts
         → Your code calls hideAsync() → Splash hides → User sees app
```

### 2.2 Configuration in app.json

```json
{
  "expo": {
    "splash": {
      "image": "./assets/splash.png",
      "resizeMode": "contain",
      "backgroundColor": "#1A1A2E"
    },
    "ios": {
      "splash": {
        "image": "./assets/splash-ios.png",
        "resizeMode": "cover",
        "backgroundColor": "#1A1A2E",
        "dark": {
          "image": "./assets/splash-ios-dark.png",
          "backgroundColor": "#0A0A14"
        }
      }
    },
    "android": {
      "splash": {
        "image": "./assets/splash-android.png",
        "resizeMode": "cover",
        "backgroundColor": "#1A1A2E",
        "dark": {
          "image": "./assets/splash-android-dark.png",
          "backgroundColor": "#0A0A14"
        }
      }
    }
  }
}
```

**Important sizing notes:**
- iOS splash images should be 2732x2732px (covers all iPad and iPhone sizes)
- Android splash images should be 1920x1920px
- Use `contain` if your splash is a centered logo, `cover` if it's a full-bleed design
- Always set `backgroundColor` to match the dominant color of your splash image — this prevents a white border on devices with different aspect ratios

### 2.3 The preventAutoHideAsync Pattern

This is the pattern every production app uses. You tell Expo "don't hide the splash screen automatically" and then manually hide it when your app is ready.

```tsx
// app/_layout.tsx
import { useEffect, useState, useCallback } from 'react';
import { View } from 'react-native';
import * as SplashScreen from 'expo-splash-screen';
import * as Font from 'expo-font';
import { useAuthStore } from '../stores/auth';

// Keep the splash screen visible while we fetch resources
// This MUST be called before the component mounts — call it at module scope
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [appIsReady, setAppIsReady] = useState(false);
  const { restoreSession } = useAuthStore();

  useEffect(() => {
    async function prepare() {
      try {
        // Do all your startup work here, in parallel where possible
        await Promise.all([
          // Load fonts
          Font.loadAsync({
            'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
            'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
          }),
          // Restore auth session from secure storage
          restoreSession(),
          // Pre-fetch critical data
          prefetchCriticalData(),
        ]);
      } catch (e) {
        // If preparation fails, still show the app
        // Better to show a degraded app than an infinite splash
        console.warn('App preparation failed:', e);
      } finally {
        setAppIsReady(true);
      }
    }

    prepare();
  }, []);

  const onLayoutRootView = useCallback(async () => {
    if (appIsReady) {
      // This tells the splash screen to hide immediately.
      // We call this after the root view has performed layout —
      // this ensures the app's content is visible before hiding splash.
      await SplashScreen.hideAsync();
    }
  }, [appIsReady]);

  if (!appIsReady) {
    return null; // Keep splash screen visible
  }

  return (
    <View style={{ flex: 1 }} onLayout={onLayoutRootView}>
      {/* Your app content */}
      <Slot />
    </View>
  );
}
```

**Why `onLayout` and not just calling `hideAsync()` immediately?**

If you call `hideAsync()` the moment `appIsReady` becomes true, there's a frame where React has started rendering but the native view hasn't laid out yet. The user sees a white flash. By waiting for `onLayout`, you ensure the native views are actually painted on screen before the splash hides.

### 2.4 Animated Splash Transitions

The native splash screen is a static image. If you want an animated transition (logo animation, fade, etc.), you need a two-layer approach: keep the native splash visible, render a React Native "animated splash" on top, then hide the native splash and play your animation.

```tsx
// components/AnimatedSplash.tsx
import { useEffect, useRef, useState } from 'react';
import { View, StyleSheet, Image } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSequence,
  withDelay,
  runOnJS,
  Easing,
} from 'react-native-reanimated';
import * as SplashScreen from 'expo-splash-screen';

interface AnimatedSplashProps {
  isReady: boolean;
  onAnimationComplete: () => void;
  children: React.ReactNode;
}

export function AnimatedSplash({
  isReady,
  onAnimationComplete,
  children,
}: AnimatedSplashProps) {
  const [showSplash, setShowSplash] = useState(true);
  
  const logoScale = useSharedValue(1);
  const logoOpacity = useSharedValue(1);
  const splashOpacity = useSharedValue(1);

  useEffect(() => {
    if (isReady) {
      // First, hide the native splash screen
      SplashScreen.hideAsync();

      // Then animate our React Native splash
      // Step 1: Pulse the logo
      logoScale.value = withSequence(
        withTiming(1.1, { duration: 300, easing: Easing.out(Easing.ease) }),
        withTiming(0.9, { duration: 200, easing: Easing.in(Easing.ease) }),
        withTiming(50, { duration: 500, easing: Easing.in(Easing.ease) })
      );

      // Step 2: Fade out logo
      logoOpacity.value = withDelay(
        400,
        withTiming(0, { duration: 300 })
      );

      // Step 3: Fade out splash background
      splashOpacity.value = withDelay(
        500,
        withTiming(0, { duration: 400 }, (finished) => {
          if (finished) {
            runOnJS(setShowSplash)(false);
            runOnJS(onAnimationComplete)();
          }
        })
      );
    }
  }, [isReady]);

  const logoStyle = useAnimatedStyle(() => ({
    transform: [{ scale: logoScale.value }],
    opacity: logoOpacity.value,
  }));

  const splashStyle = useAnimatedStyle(() => ({
    opacity: splashOpacity.value,
  }));

  return (
    <View style={styles.container}>
      {children}
      {showSplash && (
        <Animated.View
          style={[StyleSheet.absoluteFill, styles.splashContainer, splashStyle]}
        >
          <Animated.Image
            source={require('../assets/splash-logo.png')}
            style={[styles.logo, logoStyle]}
            resizeMode="contain"
          />
        </Animated.View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  splashContainer: {
    flex: 1,
    backgroundColor: '#1A1A2E',
    alignItems: 'center',
    justifyContent: 'center',
  },
  logo: {
    width: 150,
    height: 150,
  },
});
```

```tsx
// app/_layout.tsx — using the animated splash
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [appIsReady, setAppIsReady] = useState(false);
  const [splashAnimationComplete, setSplashAnimationComplete] = useState(false);

  useEffect(() => {
    async function prepare() {
      try {
        await Promise.all([loadFonts(), restoreAuth(), prefetchData()]);
      } catch (e) {
        console.warn(e);
      } finally {
        setAppIsReady(true);
      }
    }
    prepare();
  }, []);

  return (
    <AnimatedSplash
      isReady={appIsReady}
      onAnimationComplete={() => setSplashAnimationComplete(true)}
    >
      {appIsReady && <Slot />}
    </AnimatedSplash>
  );
}
```

### 2.5 Splash Screen Anti-Patterns

**Anti-pattern 1: Infinite splash on error.**
If your auth check or data fetch fails, don't leave the user on the splash screen forever. Always set a timeout and show an error state.

```tsx
// BAD: If restoreSession() hangs, user sees splash forever
await restoreSession();
setAppIsReady(true);

// GOOD: Timeout after 10 seconds
const timeout = new Promise((_, reject) =>
  setTimeout(() => reject(new Error('Startup timeout')), 10000)
);

try {
  await Promise.race([restoreSession(), timeout]);
} catch (e) {
  console.warn('Startup failed or timed out:', e);
  // Show the app anyway — maybe with a "retry" banner
} finally {
  setAppIsReady(true);
}
```

**Anti-pattern 2: Loading non-critical data during splash.**
Only block on truly critical resources: fonts, auth session, maybe feature flags. Everything else can load after the app is visible.

**Anti-pattern 3: Not matching splash and app background colors.**
If your splash has a dark background but your app's initial screen is white, the transition will flash. Match them.

---

## 3. APP STARTUP SEQUENCE

Understanding what happens between the user tapping your icon and seeing your first screen is essential for optimizing perceived performance. Every millisecond here is felt directly by the user.

### 3.1 The Full Sequence

```
1. User taps app icon
      ↓
2. OS loads the native binary
      ↓
3. Native splash screen appears (from app.json config)
      ↓
4. Native modules initialize (Expo modules, React Native bridge/JSI)
      ↓
5. JavaScript bundle loads into memory
      ↓
6. JavaScript bundle executes (module-level code runs)
      ↓
7. React root component mounts
      ↓
8. Your useEffect/preparation code runs (fonts, auth, data)
      ↓
9. SplashScreen.hideAsync() called
      ↓
10. User sees your app
```

**Where the time goes (typical breakdown):**
| Phase | Typical Duration | You Control? |
|-------|-----------------|-------------|
| OS + native binary | 200-500ms | No (mostly) |
| Native module init | 100-300ms | Partially (reduce native modules) |
| JS bundle load | 200-800ms | Yes (bundle size) |
| JS execution | 100-500ms | Yes (module-level code) |
| React mount | 50-200ms | Yes (component tree) |
| Data fetching | 200-3000ms | Yes (what you block on) |
| **Total** | **850-5300ms** | |

### 3.2 Optimization Strategies

**Strategy 1: Minimize the JS Bundle**

The biggest startup win is usually bundle size. Every kilobyte of JavaScript needs to be parsed and executed before React can mount.

```tsx
// BAD: Importing an entire library for one function
import _ from 'lodash';
const sorted = _.sortBy(items, 'name');

// GOOD: Import only what you need (tree-shakeable)
import sortBy from 'lodash/sortBy';
const sorted = sortBy(items, 'name');

// BETTER: Do you even need lodash?
const sorted = [...items].sort((a, b) => a.name.localeCompare(b.name));
```

**Strategy 2: Defer Non-Critical Initialization**

```tsx
// BAD: Initializing analytics, crash reporting, and feature flags
// all at module level — blocks JS execution
import * as Sentry from '@sentry/react-native';
import analytics from '@segment/analytics-react-native';
import { initFeatureFlags } from './feature-flags';

Sentry.init({ dsn: '...' });
analytics.setup('...');
initFeatureFlags(); // This makes a network call!

// GOOD: Initialize after first render
export default function RootLayout() {
  useEffect(() => {
    // These run AFTER the first render, not blocking startup
    requestAnimationFrame(() => {
      Sentry.init({ dsn: '...' });
      analytics.setup('...');
    });

    // Feature flags can load in background
    initFeatureFlags().catch(console.warn);
  }, []);

  return <Slot />;
}
```

**Strategy 3: Parallelize Critical Work**

```tsx
// BAD: Sequential — each waits for the previous to complete
const fonts = await Font.loadAsync({ ... });
const session = await restoreSession();
const config = await fetchAppConfig();

// GOOD: Parallel — all three run simultaneously
const [fonts, session, config] = await Promise.all([
  Font.loadAsync({ ... }),
  restoreSession(),
  fetchAppConfig(),
]);
```

**Strategy 4: Lazy Load Heavy Screens**

Don't import every screen at the top of your navigation file. Expo Router does this by default with file-based routing, but if you're using React Navigation directly:

```tsx
// BAD: Every screen is imported eagerly
import HomeScreen from './screens/HomeScreen';
import ProfileScreen from './screens/ProfileScreen';
import SettingsScreen from './screens/SettingsScreen';
import AnalyticsScreen from './screens/AnalyticsScreen';

// GOOD: Lazy load screens that aren't immediately needed
import HomeScreen from './screens/HomeScreen'; // Only eager-load the first screen
const ProfileScreen = React.lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = React.lazy(() => import('./screens/SettingsScreen'));
const AnalyticsScreen = React.lazy(() => import('./screens/AnalyticsScreen'));
```

### 3.3 Measuring Startup Performance

You can't optimize what you don't measure. Here's how to instrument your startup sequence:

```tsx
// utils/startupMetrics.ts

class StartupMetrics {
  private marks: Map<string, number> = new Map();
  private startTime: number;

  constructor() {
    this.startTime = Date.now();
    this.mark('js_start');
  }

  mark(name: string) {
    this.marks.set(name, Date.now());
  }

  measure(name: string, startMark: string, endMark: string): number {
    const start = this.marks.get(startMark);
    const end = this.marks.get(endMark);
    if (!start || !end) return -1;
    const duration = end - start;
    console.log(`[Startup] ${name}: ${duration}ms`);
    return duration;
  }

  report() {
    const totalTime = Date.now() - this.startTime;
    console.log('=== Startup Metrics ===');

    const phases = [
      ['JS Init → Fonts Loaded', 'js_start', 'fonts_loaded'],
      ['JS Init → Auth Restored', 'js_start', 'auth_restored'],
      ['JS Init → Data Prefetched', 'js_start', 'data_prefetched'],
      ['JS Init → First Render', 'js_start', 'first_render'],
      ['JS Init → Splash Hidden', 'js_start', 'splash_hidden'],
    ];

    for (const [name, start, end] of phases) {
      this.measure(name, start, end);
    }

    console.log(`[Startup] Total: ${totalTime}ms`);

    // Send to analytics
    analytics.track('app_startup', {
      totalMs: totalTime,
      ...Object.fromEntries(this.marks),
    });
  }
}

export const startupMetrics = new StartupMetrics();
```

```tsx
// Usage in _layout.tsx
import { startupMetrics } from '../utils/startupMetrics';

export default function RootLayout() {
  useEffect(() => {
    async function prepare() {
      await Promise.all([
        Font.loadAsync({ ... }).then(() => startupMetrics.mark('fonts_loaded')),
        restoreSession().then(() => startupMetrics.mark('auth_restored')),
        prefetchData().then(() => startupMetrics.mark('data_prefetched')),
      ]);
    }
    prepare();
  }, []);

  const onLayout = useCallback(() => {
    startupMetrics.mark('first_render');
    SplashScreen.hideAsync().then(() => {
      startupMetrics.mark('splash_hidden');
      startupMetrics.report();
    });
  }, []);

  // ...
}
```

---

## 4. BACKGROUND TASKS

Here's the uncomfortable truth about background tasks on mobile: **the OS does not want your app running in the background.** Both iOS and Android have become increasingly aggressive about killing background processes to save battery. Your background task is a suggestion, not a guarantee.

That said, there are legitimate use cases: syncing data, processing uploads, updating location, refreshing content. Let's cover what's actually possible.

### 4.1 expo-task-manager: The Foundation

`expo-task-manager` is the base layer that registers and manages background tasks. `expo-background-fetch` builds on top of it for periodic execution.

```bash
npx expo install expo-task-manager expo-background-fetch
```

```tsx
// tasks/backgroundSync.ts
import * as TaskManager from 'expo-task-manager';
import * as BackgroundFetch from 'expo-background-fetch';

const BACKGROUND_SYNC_TASK = 'background-sync-task';

// Step 1: Define the task
// This MUST be called at module level (outside any component)
// and it MUST be imported early in your app's entry point
TaskManager.defineTask(BACKGROUND_SYNC_TASK, async () => {
  const now = Date.now();
  console.log(`[BackgroundSync] Task started at ${new Date(now).toISOString()}`);

  try {
    // Do your background work here
    const pendingChanges = await getPendingOfflineChanges();
    
    if (pendingChanges.length === 0) {
      // Nothing to sync — tell the OS we got new data (or not)
      return BackgroundFetch.BackgroundFetchResult.NoData;
    }

    await syncChangesToServer(pendingChanges);
    await markChangesSynced(pendingChanges);

    console.log(`[BackgroundSync] Synced ${pendingChanges.length} changes`);
    return BackgroundFetch.BackgroundFetchResult.NewData;
  } catch (error) {
    console.error('[BackgroundSync] Failed:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

// Step 2: Register the task for periodic execution
export async function registerBackgroundSync() {
  const isRegistered = await TaskManager.isTaskRegisteredAsync(
    BACKGROUND_SYNC_TASK
  );

  if (isRegistered) {
    console.log('[BackgroundSync] Task already registered');
    return;
  }

  await BackgroundFetch.registerTaskAsync(BACKGROUND_SYNC_TASK, {
    minimumInterval: 15 * 60, // 15 minutes (minimum on iOS)
    stopOnTerminate: false,   // Android: keep running after app is killed
    startOnBoot: true,        // Android: restart after device reboot
  });

  console.log('[BackgroundSync] Task registered');
}

// Step 3: Check status and unregister if needed
export async function unregisterBackgroundSync() {
  const isRegistered = await TaskManager.isTaskRegisteredAsync(
    BACKGROUND_SYNC_TASK
  );

  if (isRegistered) {
    await TaskManager.unregisterTaskAsync(BACKGROUND_SYNC_TASK);
    console.log('[BackgroundSync] Task unregistered');
  }
}

export async function getBackgroundSyncStatus() {
  const status = await BackgroundFetch.getStatusAsync();
  const isRegistered = await TaskManager.isTaskRegisteredAsync(
    BACKGROUND_SYNC_TASK
  );

  return {
    status: BackgroundFetch.BackgroundFetchStatus[status],
    isRegistered,
    // status values:
    // BackgroundFetch.BackgroundFetchStatus.Restricted — user disabled background refresh
    // BackgroundFetch.BackgroundFetchStatus.Denied — background refresh not available
    // BackgroundFetch.BackgroundFetchStatus.Available — good to go
  };
}
```

```tsx
// app/_layout.tsx — Register during app startup
import { registerBackgroundSync } from '../tasks/backgroundSync';

export default function RootLayout() {
  useEffect(() => {
    registerBackgroundSync().catch(console.warn);
  }, []);

  // ...
}
```

### 4.2 Platform Limitations: The Ugly Truth

**iOS Background Fetch Limitations:**

| Limitation | Detail |
|-----------|--------|
| Minimum interval | 15 minutes (and iOS will likely run it less often) |
| Execution time | ~30 seconds before iOS kills the task |
| Scheduling | iOS uses machine learning to predict when the user opens the app and runs your task before that — great for UX, terrible for predictability |
| User control | Users can disable Background App Refresh per-app in Settings |
| Cold start | If iOS has killed your app, it will cold-start it in the background — this is slow |
| Battery | iOS monitors CPU/battery usage and will reduce frequency if your task is expensive |

**Android Background Limitations:**

| Limitation | Detail |
|-----------|--------|
| Doze mode | When the device is stationary and unplugged, Android batches all background work into infrequent maintenance windows |
| App Standby | Apps the user hasn't opened recently get bucketed into lower-priority tiers |
| Battery optimization | Enabled by default; users must manually exempt your app |
| Minimum interval | Android respects the interval more than iOS, but Doze can still delay it significantly |
| `stopOnTerminate: false` | Works, but some OEMs (Samsung, Xiaomi, Huawei) have their own aggressive task killers |

**The honest assessment:** If your feature requires reliable, time-sensitive background execution, push notifications (silent push) or a foreground service (Android) are more reliable than background fetch. Background fetch is for "nice to have" optimizations like pre-fetching data.

### 4.3 Background Task Patterns

**Pattern: Upload Queue**

Upload photos or files from a background queue:

```tsx
// tasks/backgroundUpload.ts
import * as TaskManager from 'expo-task-manager';
import * as BackgroundFetch from 'expo-background-fetch';
import * as FileSystem from 'expo-file-system';

const UPLOAD_TASK = 'background-upload-task';

TaskManager.defineTask(UPLOAD_TASK, async () => {
  try {
    const queue = await getUploadQueue(); // Read from AsyncStorage/MMKV

    if (queue.length === 0) {
      return BackgroundFetch.BackgroundFetchResult.NoData;
    }

    // Process uploads — but be mindful of the ~30s time limit
    // Only process a few items per invocation
    const batch = queue.slice(0, 3);

    for (const item of batch) {
      try {
        const uploadResult = await FileSystem.uploadAsync(
          'https://api.example.com/uploads',
          item.localUri,
          {
            httpMethod: 'POST',
            uploadType: FileSystem.FileSystemUploadType.MULTIPART,
            fieldName: 'file',
            headers: {
              Authorization: `Bearer ${await getStoredToken()}`,
            },
          }
        );

        if (uploadResult.status === 200) {
          await removeFromUploadQueue(item.id);
        }
      } catch (uploadError) {
        // Don't fail the whole task for one failed upload
        console.warn(`Upload failed for ${item.id}:`, uploadError);
        await incrementRetryCount(item.id);
      }
    }

    return BackgroundFetch.BackgroundFetchResult.NewData;
  } catch (error) {
    console.error('[BackgroundUpload] Task failed:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});
```

### 4.4 The Task Definition Rule

This is the single most common mistake with background tasks in Expo:

**`TaskManager.defineTask()` MUST be called at the module level (top-level scope), and the file MUST be imported in your app's entry point.**

```tsx
// BAD: Defined inside a component — won't work
function MyComponent() {
  useEffect(() => {
    TaskManager.defineTask('my-task', async () => { ... }); // WRONG
  }, []);
}

// BAD: Defined in a file that's lazy-loaded — won't be available when the OS wakes your app
const LazyModule = React.lazy(() => import('./tasks')); // WRONG

// GOOD: Defined at module level, imported in entry point
// tasks/index.ts
TaskManager.defineTask('my-task', async () => { ... }); // RIGHT

// app/_layout.tsx (or your entry point)
import '../tasks'; // Ensure task definitions are loaded
```

Why? When iOS or Android wakes your app in the background to run a task, they need the task to already be defined when the JavaScript initializes. If it's behind a lazy import or inside a component that hasn't mounted, the OS will find no task registered and give up.

---

## 5. KEEP-ALIVE PATTERNS

Sometimes you genuinely need your app to stay alive in the background for extended periods. Here's what each platform actually allows.

### 5.1 iOS Background Modes

iOS is strict about what can run in the background. You must declare specific "background modes" in your `Info.plist` (via Expo config plugin), and Apple will reject your app if you declare a mode you don't genuinely use.

| Background Mode | Use Case | Duration |
|----------------|----------|----------|
| `audio` | Music playback, podcast, audio recording | Indefinite while audio plays |
| `location` | GPS tracking, geofencing | Indefinite (with user consent) |
| `voip` | Voice/video call signaling | Indefinite (PushKit) |
| `fetch` | Periodic background fetch | ~30 seconds per invocation |
| `remote-notification` | Silent push processing | ~30 seconds per push |
| `processing` | Heavy work (ML, data processing) | Minutes (system decides) |

```ts
// app.config.ts — Declaring background modes
export default {
  expo: {
    ios: {
      infoPlist: {
        UIBackgroundModes: ['audio', 'location', 'fetch', 'remote-notification'],
      },
    },
  },
};
```

**The "audio trick" for keep-alive:**
Some apps play a silent audio file to keep themselves alive in the background. This works, but Apple has gotten very good at detecting it and will reject your app during review. Don't try it unless you have a legitimate audio use case.

### 5.2 Android Foreground Services

Android's answer to long-running background work is the Foreground Service — a process that shows a persistent notification to the user, signaling "this app is doing something."

```tsx
// For Android foreground services in Expo, you'll need a config plugin
// or a custom native module. Here's the pattern with expo-location
// for continuous location tracking:

import * as Location from 'expo-location';

async function startLocationTracking() {
  // Request foreground permission first
  const { status: foregroundStatus } = 
    await Location.requestForegroundPermissionsAsync();
  
  if (foregroundStatus !== 'granted') {
    throw new Error('Foreground location permission denied');
  }

  // Then request background permission (separate prompt on Android 10+)
  const { status: backgroundStatus } = 
    await Location.requestBackgroundPermissionsAsync();
  
  if (backgroundStatus !== 'granted') {
    throw new Error('Background location permission denied');
  }

  // Start background location tracking
  // This creates an Android foreground service automatically
  await Location.startLocationUpdatesAsync('location-tracking-task', {
    accuracy: Location.Accuracy.Balanced,
    timeInterval: 10000,      // Every 10 seconds
    distanceInterval: 50,      // Or every 50 meters
    showsBackgroundLocationIndicator: true, // iOS: blue bar
    foregroundService: {
      notificationTitle: 'Tracking your run',
      notificationBody: 'Tap to return to the app',
      notificationColor: '#007AFF',
    },
  });
}
```

### 5.3 What You Can't Do

Let me be blunt about what mobile OSes will not let you do:

- **Keep a WebSocket permanently open in the background.** The OS will kill it. Use push notifications instead.
- **Run arbitrary code every N minutes reliably.** Background fetch intervals are suggestions. The OS will batch, delay, or skip them.
- **Prevent the OS from killing your app.** You can reduce the likelihood with foreground services (Android) or background modes (iOS), but the OS has final say.
- **Run background tasks on OEM-skinned Android phones.** Samsung, Xiaomi, Huawei, Oppo all have their own battery optimization that is more aggressive than stock Android. Your users will need to manually exempt your app.

**The pragmatic approach:** Design your app to work gracefully whether background tasks run or not. Use them as a performance optimization (pre-fetch data so it's ready when the user opens the app), not as a critical feature. For anything mission-critical, use push notifications.

---

## 6. APP UPDATES: FORCE UPDATE AND VERSION CHECKING

Every production app needs a mechanism to force users to update when a critical bug or breaking API change is deployed. Ship this mechanism in your first release, because when you need it, it's too late to ship it.

### 6.1 The Force Update Pattern

```tsx
// services/appUpdate.ts
import { Platform, Linking } from 'react-native';
import Constants from 'expo-constants';
import * as Application from 'expo-application';

interface AppVersionConfig {
  minimumVersion: string;       // Below this: force update (blocking modal)
  recommendedVersion: string;   // Below this: suggest update (dismissible)
  currentVersion: string;       // Latest version available
  updateUrl: {
    ios: string;
    android: string;
  };
  maintenanceMode: boolean;     // Kill switch for the entire app
  maintenanceMessage?: string;
}

/**
 * Compare semantic version strings.
 * Returns: -1 if a < b, 0 if a === b, 1 if a > b
 */
function compareVersions(a: string, b: string): number {
  const partsA = a.split('.').map(Number);
  const partsB = b.split('.').map(Number);

  for (let i = 0; i < Math.max(partsA.length, partsB.length); i++) {
    const numA = partsA[i] || 0;
    const numB = partsB[i] || 0;

    if (numA < numB) return -1;
    if (numA > numB) return 1;
  }

  return 0;
}

export async function checkForUpdate(): Promise<{
  status: 'up-to-date' | 'update-recommended' | 'update-required' | 'maintenance';
  message?: string;
  storeUrl?: string;
}> {
  try {
    // Fetch version config from your server
    // This should be a lightweight, highly-cached endpoint
    const response = await fetch('https://api.example.com/app/version-config');
    const config: AppVersionConfig = await response.json();

    // Maintenance mode check
    if (config.maintenanceMode) {
      return {
        status: 'maintenance',
        message: config.maintenanceMessage || 'App is under maintenance.',
      };
    }

    const currentVersion = Application.nativeApplicationVersion || '0.0.0';
    const storeUrl = Platform.OS === 'ios'
      ? config.updateUrl.ios
      : config.updateUrl.android;

    // Force update check
    if (compareVersions(currentVersion, config.minimumVersion) < 0) {
      return {
        status: 'update-required',
        message: `Please update to continue using the app. You have v${currentVersion}, minimum required is v${config.minimumVersion}.`,
        storeUrl,
      };
    }

    // Recommended update check
    if (compareVersions(currentVersion, config.recommendedVersion) < 0) {
      return {
        status: 'update-recommended',
        message: `A new version (v${config.currentVersion}) is available with improvements and bug fixes.`,
        storeUrl,
      };
    }

    return { status: 'up-to-date' };
  } catch (error) {
    // If the version check fails, don't block the user
    console.warn('Version check failed:', error);
    return { status: 'up-to-date' };
  }
}

export function openStore(storeUrl: string) {
  Linking.openURL(storeUrl).catch(console.warn);
}
```

### 6.2 Force Update UI Component

```tsx
// components/ForceUpdateGate.tsx
import { useEffect, useState, useCallback } from 'react';
import { View, Text, StyleSheet, Modal, Pressable, Linking } from 'react-native';
import { checkForUpdate, openStore } from '../services/appUpdate';
import { useOnForeground } from '../hooks/useOnForeground';

interface UpdateStatus {
  status: 'up-to-date' | 'update-recommended' | 'update-required' | 'maintenance';
  message?: string;
  storeUrl?: string;
}

export function ForceUpdateGate({ children }: { children: React.ReactNode }) {
  const [updateStatus, setUpdateStatus] = useState<UpdateStatus | null>(null);
  const [dismissed, setDismissed] = useState(false);

  const performCheck = useCallback(async () => {
    const status = await checkForUpdate();
    setUpdateStatus(status);
  }, []);

  // Check on mount
  useEffect(() => {
    performCheck();
  }, [performCheck]);

  // Re-check when returning to foreground
  // (user might have updated and come back)
  useOnForeground(performCheck);

  // Maintenance mode — full block
  if (updateStatus?.status === 'maintenance') {
    return (
      <View style={styles.fullScreen}>
        <Text style={styles.title}>Under Maintenance</Text>
        <Text style={styles.message}>{updateStatus.message}</Text>
        <Pressable style={styles.button} onPress={performCheck}>
          <Text style={styles.buttonText}>Try Again</Text>
        </Pressable>
      </View>
    );
  }

  // Force update — full block (no dismiss)
  if (updateStatus?.status === 'update-required') {
    return (
      <View style={styles.fullScreen}>
        <Text style={styles.title}>Update Required</Text>
        <Text style={styles.message}>{updateStatus.message}</Text>
        <Pressable
          style={styles.button}
          onPress={() => openStore(updateStatus.storeUrl!)}
        >
          <Text style={styles.buttonText}>Update Now</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <>
      {children}

      {/* Recommended update — dismissible modal */}
      <Modal
        visible={updateStatus?.status === 'update-recommended' && !dismissed}
        transparent
        animationType="fade"
      >
        <View style={styles.modalOverlay}>
          <View style={styles.modalContent}>
            <Text style={styles.title}>Update Available</Text>
            <Text style={styles.message}>{updateStatus?.message}</Text>
            <View style={styles.modalButtons}>
              <Pressable
                style={[styles.button, styles.secondaryButton]}
                onPress={() => setDismissed(true)}
              >
                <Text style={styles.secondaryButtonText}>Later</Text>
              </Pressable>
              <Pressable
                style={styles.button}
                onPress={() => openStore(updateStatus?.storeUrl!)}
              >
                <Text style={styles.buttonText}>Update</Text>
              </Pressable>
            </View>
          </View>
        </View>
      </Modal>
    </>
  );
}

const styles = StyleSheet.create({
  fullScreen: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 32,
    backgroundColor: '#FFFFFF',
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#1A1A1A',
    marginBottom: 12,
    textAlign: 'center',
  },
  message: {
    fontSize: 16,
    color: '#666666',
    textAlign: 'center',
    lineHeight: 24,
    marginBottom: 32,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingVertical: 14,
    paddingHorizontal: 32,
    borderRadius: 12,
    minWidth: 120,
    alignItems: 'center',
  },
  buttonText: {
    color: '#FFFFFF',
    fontSize: 16,
    fontWeight: '600',
  },
  secondaryButton: {
    backgroundColor: 'transparent',
    borderWidth: 1,
    borderColor: '#CCCCCC',
  },
  secondaryButtonText: {
    color: '#666666',
    fontSize: 16,
    fontWeight: '600',
  },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
    padding: 32,
  },
  modalContent: {
    backgroundColor: '#FFFFFF',
    borderRadius: 20,
    padding: 24,
    width: '100%',
    alignItems: 'center',
  },
  modalButtons: {
    flexDirection: 'row',
    gap: 12,
  },
});
```

### 6.3 OTA Updates with EAS Update

For Expo apps, you also have OTA (over-the-air) updates via EAS Update. These update the JavaScript bundle without going through the app store. They're instant, but they can only change JavaScript — native code changes still require a store release.

```tsx
// services/otaUpdate.ts
import * as Updates from 'expo-updates';

export async function checkForOTAUpdate() {
  if (__DEV__) {
    // OTA updates don't work in development
    return { available: false };
  }

  try {
    const update = await Updates.checkForUpdateAsync();

    if (update.isAvailable) {
      // Download the update
      await Updates.fetchUpdateAsync();

      // Prompt user to restart (or restart silently)
      return { available: true };
    }

    return { available: false };
  } catch (error) {
    console.warn('OTA update check failed:', error);
    return { available: false };
  }
}

export async function applyOTAUpdate() {
  await Updates.reloadAsync(); // Restarts the app with the new bundle
}
```

```tsx
// Using OTA updates with the foreground hook
function App() {
  useOnForeground(async () => {
    const { available } = await checkForOTAUpdate();
    if (available) {
      // Option 1: Silent restart (use for critical fixes)
      // await applyOTAUpdate();

      // Option 2: Prompt the user
      Alert.alert(
        'Update Available',
        'A new version is ready. Restart to apply?',
        [
          { text: 'Later', style: 'cancel' },
          { text: 'Restart', onPress: () => applyOTAUpdate() },
        ]
      );
    }
  });
}
```

### 6.4 The Version Check Architecture

Here's how I recommend structuring version checks for a production app:

```
┌──────────────────────────────────────────────────────┐
│ Your Server (or Firebase Remote Config)              │
│                                                      │
│  {                                                   │
│    "minimumVersion": "2.1.0",                        │
│    "recommendedVersion": "2.3.0",                    │
│    "currentVersion": "2.3.2",                        │
│    "maintenanceMode": false,                         │
│    "updateUrl": {                                    │
│      "ios": "https://apps.apple.com/app/id123",      │
│      "android": "https://play.google.com/store/..."   │
│    },                                                │
│    "featureFlags": { ... }                           │
│  }                                                   │
└───────────────────────┬──────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────┐
│ App Startup                                          │
│                                                      │
│  1. Show splash screen                               │
│  2. Fetch version config (with 5s timeout)           │
│  3. Compare installed version vs minimumVersion      │
│  4. If maintenance mode → show maintenance screen    │
│  5. If force update needed → show blocking modal     │
│  6. If recommended update → queue dismissible modal  │
│  7. Continue normal app startup                      │
│  8. Check for OTA update in background               │
└──────────────────────────────────────────────────────┘
```

**Pro tip:** Combine your version config endpoint with feature flags and other app configuration. One network request at startup that gives you everything you need. Cache it aggressively (but not so aggressively that a force-update takes hours to propagate).

---

# PART B: PUSH NOTIFICATIONS

---

## 7. PUSH NOTIFICATION ARCHITECTURE

Before we write any code, you need to understand the pipeline. Push notifications involve multiple systems, and debugging problems requires knowing where in the chain things went wrong.

### 7.1 The End-to-End Pipeline

```
┌──────────┐     ┌─────────────────────┐     ┌──────────┐
│          │     │                     │     │          │
│  Your    │────►│  FCM or APNs        │────►│  Device  │
│  Server  │     │  (Google / Apple)   │     │  OS      │
│          │     │                     │     │          │
└──────────┘     └─────────────────────┘     └────┬─────┘
                                                   │
                                                   ▼
                                             ┌──────────┐
                                             │  Your    │
                                             │  App     │
                                             └──────────┘
```

**The detailed flow:**

1. **Your server** sends a push notification payload to FCM (Firebase Cloud Messaging) or APNs (Apple Push Notification service).
2. **FCM/APNs** validates the payload, looks up the device token, and routes the notification to the correct device.
3. **The device OS** receives the notification and decides what to do:
   - If the app is in the **foreground**: delivers the event to your app (you decide whether to show it)
   - If the app is in the **background**: shows the notification in the system tray/notification center
   - If the app is **killed**: shows the notification; if the user taps it, cold-starts the app
4. **Your app** handles the notification event (display custom UI, navigate to a screen, update data).

### 7.2 FCM vs APNs

**FCM (Firebase Cloud Messaging):**
- Works on **both Android and iOS**
- On Android: delivers directly to the device via Google Play Services
- On iOS: acts as a proxy to APNs (FCM sends to APNs on your behalf)
- Provides a unified API for both platforms
- Free, no message limits (but there are rate limits)

**APNs (Apple Push Notification service):**
- iOS and macOS only
- Required for iOS push notifications (even if you use FCM, it goes through APNs)
- Requires an APNs authentication key or certificate
- Two environments: sandbox (development) and production

**Expo Push Service:**
- A third option that sits in front of both FCM and APNs
- You send to Expo's API with an Expo Push Token
- Expo routes to the correct service (FCM or APNs)
- Simpler setup, but adds a hop (and a dependency on Expo's infrastructure)

```
                    ┌─────────────────────┐
                    │                     │
              ┌────►│  APNs (Apple)       │────► iOS Device
              │     │                     │
              │     └─────────────────────┘
┌──────────┐  │
│  Your    │──┤
│  Server  │  │     ┌─────────────────────┐
│          │──┤     │                     │
└──────────┘  └────►│  FCM (Google)       │────► Android Device
                    │                     │
                    │         │           │
                    └─────────┼───────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  APNs (for iOS      │────► iOS Device
                    │  via FCM proxy)     │      (when using FCM)
                    └─────────────────────┘

Alternative with Expo Push Service:

┌──────────┐     ┌───────────────┐     ┌────────┐     ┌────────┐
│  Your    │────►│  Expo Push    │────►│FCM/APNs│────►│ Device │
│  Server  │     │  Service      │     │        │     │        │
└──────────┘     └───────────────┘     └────────┘     └────────┘
```

### 7.3 Push Tokens

Every device that wants to receive push notifications needs a **push token** — a unique identifier that tells FCM/APNs which device to deliver to.

- **FCM token**: ~150 characters, looks like `dH4bR8kS9...`. Same token works for sending to Android directly and iOS via FCM proxy.
- **APNs device token**: 64 hex characters. iOS-specific.
- **Expo Push Token**: Looks like `ExponentPushToken[xxxxxxxxxxxxxx]`. An Expo-specific wrapper that maps to the underlying FCM/APNs token.

**Token lifecycle:**
- Tokens are generated when the app first requests notification permissions
- Tokens can change: app reinstall, OS update, token refresh by FCM
- You must store tokens server-side and associate them with the user
- You must handle invalid/expired tokens (FCM and APNs will tell you)

---

## 8. EXPO NOTIFICATIONS SETUP

`expo-notifications` is the comprehensive notification library for Expo apps. It handles permissions, tokens, receiving notifications, and local/scheduled notifications.

### 8.1 Installation and Configuration

```bash
npx expo install expo-notifications expo-device expo-constants
```

```ts
// app.config.ts
export default {
  expo: {
    // ...
    plugins: [
      [
        'expo-notifications',
        {
          // iOS notification icon (optional — defaults to app icon)
          icon: './assets/notification-icon.png',
          // Android notification color
          color: '#007AFF',
          // Android default notification channel
          defaultChannel: 'default',
          // Enable sounds
          sounds: [
            './assets/sounds/notification.wav',
            './assets/sounds/urgent.wav',
          ],
        },
      ],
    ],
    android: {
      // FCM is configured automatically if you have a google-services.json
      googleServicesFile: './google-services.json',
    },
    ios: {
      // APNs is configured automatically by Expo
      // But you need to set up your APNs key in your Expo account
      // (or in your Apple Developer account if using FCM directly)
    },
  },
};
```

### 8.2 Requesting Permissions

```tsx
// services/notifications.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import { Platform } from 'react-native';
import Constants from 'expo-constants';

/**
 * Request notification permissions and return the push token.
 * Returns null if permissions are denied or device doesn't support push.
 */
export async function registerForPushNotifications(): Promise<string | null> {
  // Push notifications don't work on simulators/emulators
  // (Android emulator works with FCM, iOS simulator does NOT work with APNs)
  if (!Device.isDevice) {
    console.warn('Push notifications require a physical device (iOS)');
    // On Android emulator, we can still proceed
    if (Platform.OS === 'ios') return null;
  }

  // Check existing permission status
  const { status: existingStatus } =
    await Notifications.getPermissionsAsync();

  let finalStatus = existingStatus;

  // If not determined, request permission
  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync({
      ios: {
        allowAlert: true,
        allowBadge: true,
        allowSound: true,
        allowCriticalAlerts: false, // Requires special Apple entitlement
        allowProvisional: false,    // Deliver quietly (no prompt)
      },
    });
    finalStatus = status;
  }

  if (finalStatus !== 'granted') {
    console.log('Notification permission denied');
    return null;
  }

  // Get the push token
  try {
    // Option 1: Expo Push Token (use with Expo Push Service)
    const expoPushToken = await Notifications.getExpoPushTokenAsync({
      projectId: Constants.expoConfig?.extra?.eas?.projectId,
    });
    console.log('Expo Push Token:', expoPushToken.data);
    return expoPushToken.data;

    // Option 2: Device Push Token (use with direct FCM/APNs)
    // const deviceToken = await Notifications.getDevicePushTokenAsync();
    // console.log('Device Token:', deviceToken.data);
    // return deviceToken.data;
  } catch (error) {
    console.error('Failed to get push token:', error);
    return null;
  }
}

/**
 * Send the push token to your server.
 * Call this after getting a token and whenever it changes.
 */
export async function savePushTokenToServer(
  token: string,
  userId: string
) {
  await fetch('https://api.example.com/users/push-token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      token,
      userId,
      platform: Platform.OS,
      deviceName: Device.deviceName,
    }),
  });
}
```

### 8.3 Setting Default Notification Behavior

This is critical and often overlooked: you need to tell `expo-notifications` how to handle notifications that arrive while the app is in the foreground.

```tsx
// services/notifications.ts (continued)

/**
 * Configure how notifications behave when the app is in the foreground.
 * Call this once at app startup.
 */
export function configureNotificationHandler() {
  Notifications.setNotificationHandler({
    handleNotification: async (notification) => {
      // This function is called for EVERY notification that arrives
      // while the app is in the foreground.
      // Return an object that tells the OS what to do.

      const data = notification.request.content.data;

      // Example: Suppress chat notifications if the user is
      // already viewing that chat
      if (data?.type === 'chat_message' && isViewingChat(data.chatId)) {
        return {
          shouldShowAlert: false,  // Don't show the banner
          shouldPlaySound: false,  // Don't play sound
          shouldSetBadge: false,   // Don't update badge
        };
      }

      // For all other notifications, show them
      return {
        shouldShowAlert: true,
        shouldPlaySound: true,
        shouldSetBadge: true,
      };
    },
  });
}
```

### 8.4 Full Setup in Root Layout

```tsx
// app/_layout.tsx
import { useEffect, useRef } from 'react';
import { Platform } from 'react-native';
import * as Notifications from 'expo-notifications';
import { router } from 'expo-router';
import {
  registerForPushNotifications,
  savePushTokenToServer,
  configureNotificationHandler,
} from '../services/notifications';
import { useAuthStore } from '../stores/auth';

// Configure notification handling at module level
configureNotificationHandler();

// Android: Set up notification channel (required Android 8+)
if (Platform.OS === 'android') {
  Notifications.setNotificationChannelAsync('default', {
    name: 'Default',
    importance: Notifications.AndroidImportance.MAX,
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#007AFF',
    sound: 'default',
  });
}

export default function RootLayout() {
  const notificationListener = useRef<Notifications.Subscription>();
  const responseListener = useRef<Notifications.Subscription>();
  const { user } = useAuthStore();

  useEffect(() => {
    // Register for push notifications when user is logged in
    if (user) {
      registerForPushNotifications().then((token) => {
        if (token) {
          savePushTokenToServer(token, user.id);
        }
      });
    }

    // Listen for incoming notifications (foreground)
    notificationListener.current =
      Notifications.addNotificationReceivedListener((notification) => {
        console.log('Notification received (foreground):', notification);
        // You can update UI, show in-app banner, etc.
      });

    // Listen for notification taps (user interacted with notification)
    responseListener.current =
      Notifications.addNotificationResponseReceivedListener((response) => {
        console.log('Notification tapped:', response);
        const data = response.notification.request.content.data;
        handleNotificationNavigation(data);
      });

    return () => {
      if (notificationListener.current) {
        Notifications.removeNotificationSubscription(
          notificationListener.current
        );
      }
      if (responseListener.current) {
        Notifications.removeNotificationSubscription(
          responseListener.current
        );
      }
    };
  }, [user]);

  return <Slot />;
}

function handleNotificationNavigation(data: any) {
  // Navigate based on notification data
  if (data?.screen) {
    // e.g., data = { screen: '/chat/[chatId]', chatId: '123' }
    router.push(data.screen, data.params || {});
  } else if (data?.url) {
    // Deep link URL
    router.push(data.url);
  }
}
```

---

## 9. HANDLING THE THREE NOTIFICATION STATES

This is the most important section in the push notifications part of this chapter. Every app must handle three distinct states, and most apps only handle one correctly.

### 9.1 The Three States

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  STATE 1: FOREGROUND                                     │
│  ─────────────────                                       │
│  App is open and active. Notification arrives.           │
│  → OS does NOT automatically show a banner               │
│  → Your handler decides: show alert? play sound?         │
│  → You might show a custom in-app notification UI        │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  STATE 2: BACKGROUND                                     │
│  ────────────────────                                    │
│  App is in the background (or inactive).                 │
│  → OS automatically shows the notification               │
│  → Your code is NOT running (usually)                    │
│  → If user taps: app comes to foreground,                │
│    response listener fires                               │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  STATE 3: KILLED / COLD START                            │
│  ──────────────────────────────                          │
│  App is not running at all.                              │
│  → OS shows the notification                             │
│  → If user taps: OS cold-starts the app                  │
│  → The response listener fires, BUT your navigation      │
│    might not be ready yet                                │
│  → This is where 90% of notification bugs live           │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 9.2 State 1: Foreground Notifications

When a notification arrives while the app is in the foreground, the default behavior is... nothing. The OS does not show a banner. You need to either configure `setNotificationHandler` (as we did above) to show the system banner, or build custom UI.

**Custom In-App Notification Banner:**

```tsx
// components/InAppNotificationBanner.tsx
import { useEffect, useRef, useState } from 'react';
import { View, Text, StyleSheet, Pressable, Platform } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withDelay,
  withTiming,
  runOnJS,
} from 'react-native-reanimated';
import * as Notifications from 'expo-notifications';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

interface InAppNotification {
  id: string;
  title: string;
  body: string;
  data: any;
}

export function InAppNotificationBanner() {
  const [notification, setNotification] = useState<InAppNotification | null>(null);
  const insets = useSafeAreaInsets();
  const translateY = useSharedValue(-200);
  const opacity = useSharedValue(0);

  useEffect(() => {
    const subscription = Notifications.addNotificationReceivedListener(
      (event) => {
        const content = event.request.content;
        setNotification({
          id: event.request.identifier,
          title: content.title || '',
          body: content.body || '',
          data: content.data,
        });
      }
    );

    return () => subscription.remove();
  }, []);

  useEffect(() => {
    if (notification) {
      // Slide in
      translateY.value = withSpring(0, {
        damping: 15,
        stiffness: 150,
      });
      opacity.value = withTiming(1, { duration: 200 });

      // Auto-dismiss after 4 seconds
      translateY.value = withDelay(
        4000,
        withTiming(-200, { duration: 300 }, (finished) => {
          if (finished) {
            runOnJS(setNotification)(null);
          }
        })
      );
      opacity.value = withDelay(4000, withTiming(0, { duration: 300 }));
    }
  }, [notification]);

  const bannerStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
    opacity: opacity.value,
  }));

  if (!notification) return null;

  return (
    <Animated.View
      style={[
        styles.banner,
        { paddingTop: insets.top + 8 },
        bannerStyle,
      ]}
    >
      <Pressable
        style={styles.bannerContent}
        onPress={() => {
          // Handle tap — navigate to relevant screen
          handleNotificationNavigation(notification.data);
          setNotification(null);
        }}
      >
        <Text style={styles.bannerTitle} numberOfLines={1}>
          {notification.title}
        </Text>
        <Text style={styles.bannerBody} numberOfLines={2}>
          {notification.body}
        </Text>
      </Pressable>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  banner: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    zIndex: 9999,
    paddingHorizontal: 12,
    paddingBottom: 12,
  },
  bannerContent: {
    backgroundColor: '#1A1A2E',
    borderRadius: 16,
    padding: 16,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.2,
    shadowRadius: 12,
    elevation: 8,
  },
  bannerTitle: {
    fontSize: 15,
    fontWeight: '700',
    color: '#FFFFFF',
    marginBottom: 4,
  },
  bannerBody: {
    fontSize: 14,
    color: '#CCCCCC',
    lineHeight: 20,
  },
});
```

### 9.3 State 2: Background Notifications

When the app is in the background, the OS handles displaying the notification. You don't need to do anything special for display. The important part is handling the **response** when the user taps the notification.

The `addNotificationResponseReceivedListener` we set up in the root layout handles this. When the user taps a notification while the app is backgrounded (but still in memory), the app comes to the foreground and the response listener fires immediately.

### 9.4 State 3: Cold Start — The Hard Part

When the app is killed and the user taps a notification, the app cold-starts. The problem: your response listener isn't set up yet because your React components haven't mounted. The notification response is delivered before your listener is registered.

Expo handles this with `Notifications.getLastNotificationResponseAsync()`:

```tsx
// app/_layout.tsx
export default function RootLayout() {
  const navigationReady = useRef(false);
  const pendingNotification = useRef<any>(null);

  useEffect(() => {
    // Handle cold start notification
    // This catches the notification that launched the app
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        const data = response.notification.request.content.data;
        
        if (navigationReady.current) {
          // Navigation is ready — navigate immediately
          handleNotificationNavigation(data);
        } else {
          // Navigation not ready — queue it
          pendingNotification.current = data;
        }
      }
    });

    // Listen for future notification taps
    const subscription =
      Notifications.addNotificationResponseReceivedListener((response) => {
        const data = response.notification.request.content.data;
        handleNotificationNavigation(data);
      });

    return () => subscription.remove();
  }, []);

  // Call this when navigation is ready
  const onNavigationReady = useCallback(() => {
    navigationReady.current = true;

    // Process any pending notification from cold start
    if (pendingNotification.current) {
      // Small delay to ensure navigation is truly ready
      setTimeout(() => {
        handleNotificationNavigation(pendingNotification.current);
        pendingNotification.current = null;
      }, 100);
    }
  }, []);

  return (
    <NavigationContainer onReady={onNavigationReady}>
      <Slot />
    </NavigationContainer>
  );
}
```

### 9.5 The Complete Notification Handler

Here's a production-ready notification handler that covers all three states:

```tsx
// services/notificationHandler.ts
import * as Notifications from 'expo-notifications';
import { router } from 'expo-router';

type NotificationData = {
  type?: string;
  screen?: string;
  url?: string;
  chatId?: string;
  orderId?: string;
  [key: string]: any;
};

/**
 * Route the user to the correct screen based on notification data.
 * Handles all notification types your app supports.
 */
export function handleNotificationNavigation(data: NotificationData) {
  if (!data) return;

  switch (data.type) {
    case 'chat_message':
      router.push(`/chat/${data.chatId}`);
      break;

    case 'order_update':
      router.push(`/orders/${data.orderId}`);
      break;

    case 'new_follower':
      router.push(`/profile/${data.userId}`);
      break;

    case 'promo':
      if (data.url) {
        router.push(data.url);
      }
      break;

    default:
      // If there's a generic screen or URL, use it
      if (data.screen) {
        router.push(data.screen);
      } else if (data.url) {
        router.push(data.url);
      }
      // Otherwise, do nothing — app opens to the default screen
      break;
  }
}

/**
 * Complete notification setup function.
 * Call once in your root layout.
 */
export function setupNotifications(
  onNavigationReady: () => void
) {
  let isNavigationReady = false;
  let pendingNavigation: NotificationData | null = null;

  // Configure foreground behavior
  Notifications.setNotificationHandler({
    handleNotification: async () => ({
      shouldShowAlert: true,
      shouldPlaySound: true,
      shouldSetBadge: true,
    }),
  });

  // Handle cold start — check for notification that launched the app
  Notifications.getLastNotificationResponseAsync().then((response) => {
    if (!response) return;

    const data = response.notification.request.content.data as NotificationData;

    if (isNavigationReady) {
      handleNotificationNavigation(data);
    } else {
      pendingNavigation = data;
    }
  });

  // Handle warm start / background taps
  const responseSubscription =
    Notifications.addNotificationResponseReceivedListener((response) => {
      const data = response.notification.request.content.data as NotificationData;
      handleNotificationNavigation(data);
    });

  // Return a function to call when navigation is ready
  const markNavigationReady = () => {
    isNavigationReady = true;
    if (pendingNavigation) {
      setTimeout(() => {
        handleNotificationNavigation(pendingNavigation!);
        pendingNavigation = null;
      }, 100);
    }
  };

  return {
    markNavigationReady,
    cleanup: () => {
      responseSubscription.remove();
    },
  };
}
```

---

## 10. NOTIFICATION PAYLOADS

Understanding what you can put in a notification payload determines what kind of experiences you can build. The payload structure differs between FCM and APNs, but Expo abstracts most of this.

### 10.1 Basic Payload Structure

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxx]",
  "title": "New Message",
  "body": "Hey! Are you free for lunch?",
  "data": {
    "type": "chat_message",
    "chatId": "abc123",
    "senderId": "user456",
    "screen": "/chat/abc123"
  },
  "sound": "default",
  "badge": 3,
  "categoryId": "chat_message",
  "channelId": "messages"
}
```

**Field breakdown:**

| Field | Type | Description |
|-------|------|-------------|
| `to` | string | Push token (Expo, FCM, or APNs) |
| `title` | string | Bold headline text |
| `subtitle` | string | Secondary text (iOS only, below title) |
| `body` | string | Main message content |
| `data` | object | Custom data payload (not displayed, for your app logic) |
| `sound` | string | `"default"` or custom sound filename |
| `badge` | number | App icon badge count (iOS; Android uses channels) |
| `categoryId` | string | Notification category for action buttons |
| `channelId` | string | Android notification channel ID |
| `priority` | string | `"default"`, `"normal"`, or `"high"` |
| `ttl` | number | Time-to-live in seconds (how long FCM/APNs should retry) |

### 10.2 Rich Notifications: Images and Attachments

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxx]",
  "title": "New Photo",
  "body": "Sarah shared a photo in the group",
  "data": {
    "type": "photo",
    "groupId": "group123"
  },
  "richContent": {
    "image": "https://example.com/photo.jpg"
  }
}
```

For Expo Push API, image attachments are available through the notification content:

```tsx
// Sending with image (server-side, using Expo Push API)
const message = {
  to: pushToken,
  title: 'New Photo',
  body: 'Sarah shared a photo',
  data: { type: 'photo', groupId: 'group123' },
  // iOS: Rich notification with image
  // Requires a Notification Service Extension for custom rendering
};
```

### 10.3 Action Buttons (Notification Categories)

Action buttons let users respond to notifications without opening the app.

```tsx
// services/notificationCategories.ts
import * as Notifications from 'expo-notifications';

export async function setupNotificationCategories() {
  await Notifications.setNotificationCategoryAsync('chat_message', [
    {
      identifier: 'reply',
      buttonTitle: 'Reply',
      options: {
        opensAppToForeground: false, // Handle without opening app
      },
      textInput: {
        submitButtonTitle: 'Send',
        placeholder: 'Type a reply...',
      },
    },
    {
      identifier: 'mark_read',
      buttonTitle: 'Mark as Read',
      options: {
        opensAppToForeground: false,
        isDestructive: false,
      },
    },
  ]);

  await Notifications.setNotificationCategoryAsync('friend_request', [
    {
      identifier: 'accept',
      buttonTitle: 'Accept',
      options: {
        opensAppToForeground: false,
      },
    },
    {
      identifier: 'decline',
      buttonTitle: 'Decline',
      options: {
        opensAppToForeground: false,
        isDestructive: true,
      },
    },
  ]);
}

// Handle category action responses
export function setupCategoryActionHandler() {
  Notifications.addNotificationResponseReceivedListener(async (response) => {
    const actionId = response.actionIdentifier;
    const data = response.notification.request.content.data;

    // Check if it's a text input action (like reply)
    const userText = response.userText;

    switch (actionId) {
      case 'reply':
        if (userText && data.chatId) {
          await sendChatReply(data.chatId, userText);
        }
        break;

      case 'mark_read':
        if (data.chatId) {
          await markChatAsRead(data.chatId);
        }
        break;

      case 'accept':
        if (data.requestId) {
          await acceptFriendRequest(data.requestId);
        }
        break;

      case 'decline':
        if (data.requestId) {
          await declineFriendRequest(data.requestId);
        }
        break;

      case Notifications.DEFAULT_ACTION_IDENTIFIER:
        // User tapped the notification itself (not an action button)
        handleNotificationNavigation(data);
        break;
    }
  });
}
```

### 10.4 Silent Notifications (Data-Only)

Silent notifications don't show any UI — they deliver data to your app in the background. Use them to trigger background sync, update cached data, or invalidate stale content.

```json
{
  "to": "ExponentPushToken[xxxxxxxxxxxxxx]",
  "data": {
    "type": "silent_sync",
    "action": "invalidate_cache",
    "key": "user_profile"
  },
  "_contentAvailable": true
}
```

```tsx
// On iOS, silent notifications require the 'remote-notification'
// background mode in your Info.plist:
// app.config.ts
export default {
  expo: {
    ios: {
      infoPlist: {
        UIBackgroundModes: ['remote-notification'],
      },
    },
  },
};
```

**Silent notification gotchas:**
- iOS limits silent notifications: if you send too many that the user doesn't interact with, iOS throttles or stops delivering them.
- iOS requires `content-available: 1` in the APNs payload.
- The app gets ~30 seconds to process a silent notification.
- If the device is in Low Power Mode, silent notifications may not be delivered.

---

## 11. LOCAL AND SCHEDULED NOTIFICATIONS

Not all notifications come from your server. Local notifications are triggered by the app itself and never touch FCM or APNs. They're perfect for reminders, alarms, and time-based alerts.

### 11.1 Immediate Local Notification

```tsx
import * as Notifications from 'expo-notifications';

async function showLocalNotification() {
  await Notifications.scheduleNotificationAsync({
    content: {
      title: 'Download Complete',
      body: 'Your export is ready to view.',
      data: { type: 'download', fileId: 'file123' },
      sound: 'default',
    },
    trigger: null, // null = show immediately
  });
}
```

### 11.2 Scheduled Notifications

```tsx
// Schedule a notification for a specific time
async function scheduleReminder(
  title: string,
  body: string,
  date: Date,
  data: any = {}
) {
  const id = await Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      data,
      sound: 'default',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.DATE,
      date,
    },
  });

  console.log(`Scheduled notification ${id} for ${date.toISOString()}`);
  return id;
}

// Schedule a notification after a delay
async function scheduleDelayedNotification(
  title: string,
  body: string,
  delaySeconds: number
) {
  return Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      sound: 'default',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
      seconds: delaySeconds,
      repeats: false,
    },
  });
}

// Schedule a daily recurring notification
async function scheduleDailyReminder(
  title: string,
  body: string,
  hour: number,
  minute: number
) {
  return Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      sound: 'default',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
      hour,
      minute,
      repeats: true,
    },
  });
}

// Schedule a weekly notification (e.g., every Monday at 9 AM)
async function scheduleWeeklyReminder(
  title: string,
  body: string,
  weekday: number, // 1 = Sunday, 2 = Monday, ..., 7 = Saturday
  hour: number,
  minute: number
) {
  return Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      sound: 'default',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.CALENDAR,
      weekday,
      hour,
      minute,
      repeats: true,
    },
  });
}
```

### 11.3 Managing Scheduled Notifications

```tsx
// List all scheduled notifications
async function listScheduledNotifications() {
  const scheduled =
    await Notifications.getAllScheduledNotificationsAsync();
  
  console.log(`${scheduled.length} notifications scheduled:`);
  for (const notif of scheduled) {
    console.log(
      `  ${notif.identifier}: "${notif.content.title}" — ` +
      `trigger: ${JSON.stringify(notif.trigger)}`
    );
  }
  
  return scheduled;
}

// Cancel a specific notification
async function cancelNotification(notificationId: string) {
  await Notifications.cancelScheduledNotificationAsync(notificationId);
}

// Cancel all scheduled notifications
async function cancelAllNotifications() {
  await Notifications.cancelAllScheduledNotificationsAsync();
}

// Practical example: Reminder system
class ReminderService {
  private storageKey = 'scheduled_reminders';

  async addReminder(
    title: string,
    body: string,
    date: Date,
    metadata: any = {}
  ) {
    const notificationId = await Notifications.scheduleNotificationAsync({
      content: {
        title,
        body,
        data: { type: 'reminder', ...metadata },
        sound: 'default',
      },
      trigger: {
        type: Notifications.SchedulableTriggerInputTypes.DATE,
        date,
      },
    });

    // Store the mapping so we can cancel later
    const reminders = await this.getReminders();
    reminders.push({
      id: notificationId,
      title,
      body,
      date: date.toISOString(),
      metadata,
    });
    await AsyncStorage.setItem(this.storageKey, JSON.stringify(reminders));

    return notificationId;
  }

  async removeReminder(notificationId: string) {
    await Notifications.cancelScheduledNotificationAsync(notificationId);

    const reminders = await this.getReminders();
    const filtered = reminders.filter((r) => r.id !== notificationId);
    await AsyncStorage.setItem(this.storageKey, JSON.stringify(filtered));
  }

  async getReminders() {
    const stored = await AsyncStorage.getItem(this.storageKey);
    return stored ? JSON.parse(stored) : [];
  }

  async clearAll() {
    await Notifications.cancelAllScheduledNotificationsAsync();
    await AsyncStorage.removeItem(this.storageKey);
  }
}

export const reminderService = new ReminderService();
```

---

## 12. ANDROID NOTIFICATION CHANNELS

Android 8.0 (API 26) introduced notification channels. They're required — if you don't create a channel, your notifications won't show on Android 8+. More importantly, channels are **permanent user-facing settings** that users can customize individually.

### 12.1 Why Channels Matter

Channels let users control notification preferences at a granular level. A user might want to receive chat message notifications with sound but delivery notifications silently. Channels make this possible without building the settings UI yourself — Android handles it natively.

**The catch:** Once a channel is created, you cannot change its importance level programmatically. The user owns the channel. You can update the name and description, but importance, sound, and vibration are locked after creation. Plan your channels carefully before your first release.

### 12.2 Channel Configuration

```tsx
// services/notificationChannels.ts
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

interface ChannelConfig {
  id: string;
  name: string;
  description: string;
  importance: Notifications.AndroidImportance;
  sound?: string;
  vibrationPattern?: number[];
  lightColor?: string;
  bypassDnd?: boolean;
  lockscreenVisibility?: Notifications.AndroidNotificationVisibility;
  enableLights?: boolean;
  enableVibrate?: boolean;
  showBadge?: boolean;
  groupId?: string;
}

const CHANNELS: ChannelConfig[] = [
  {
    id: 'messages',
    name: 'Messages',
    description: 'Chat messages and direct messages',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
    vibrationPattern: [0, 250, 250, 250],
    enableLights: true,
    lightColor: '#007AFF',
    showBadge: true,
  },
  {
    id: 'orders',
    name: 'Order Updates',
    description: 'Order status changes, delivery updates',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'default',
    enableVibrate: true,
    showBadge: true,
  },
  {
    id: 'promotions',
    name: 'Promotions & Offers',
    description: 'Deals, discounts, and special offers',
    importance: Notifications.AndroidImportance.DEFAULT,
    sound: 'default',
    enableVibrate: false,
    showBadge: false,
  },
  {
    id: 'reminders',
    name: 'Reminders',
    description: 'Scheduled reminders and alerts',
    importance: Notifications.AndroidImportance.HIGH,
    sound: 'notification.wav', // Custom sound file
    vibrationPattern: [0, 500, 200, 500],
    showBadge: true,
  },
  {
    id: 'system',
    name: 'System',
    description: 'App updates, account alerts, security notices',
    importance: Notifications.AndroidImportance.DEFAULT,
    sound: 'default',
    showBadge: false,
  },
  {
    id: 'silent',
    name: 'Background Updates',
    description: 'Silent data sync and background processing',
    importance: Notifications.AndroidImportance.MIN,
    enableVibrate: false,
    showBadge: false,
  },
];

/**
 * Create all notification channels.
 * Call once at app startup. Safe to call multiple times —
 * existing channels won't be recreated.
 */
export async function setupNotificationChannels() {
  if (Platform.OS !== 'android') return;

  // Optionally create channel groups
  await Notifications.setNotificationChannelGroupAsync('communication', {
    name: 'Communication',
    description: 'Messages and social notifications',
  });

  await Notifications.setNotificationChannelGroupAsync('commerce', {
    name: 'Shopping',
    description: 'Orders and promotions',
  });

  // Create each channel
  for (const channel of CHANNELS) {
    await Notifications.setNotificationChannelAsync(channel.id, {
      name: channel.name,
      description: channel.description,
      importance: channel.importance,
      sound: channel.sound || undefined,
      vibrationPattern: channel.vibrationPattern || undefined,
      lightColor: channel.lightColor || undefined,
      bypassDnd: channel.bypassDnd || false,
      lockscreenVisibility:
        channel.lockscreenVisibility ||
        Notifications.AndroidNotificationVisibility.PUBLIC,
      enableLights: channel.enableLights ?? false,
      enableVibrate: channel.enableVibrate ?? true,
      showBadge: channel.showBadge ?? true,
      groupId: channel.groupId || undefined,
    });
  }

  console.log(`[Channels] ${CHANNELS.length} notification channels configured`);
}

/**
 * Delete a channel (e.g., when deprecating a notification type).
 * The channel will reappear if you create it again, but with fresh defaults.
 */
export async function deleteChannel(channelId: string) {
  if (Platform.OS !== 'android') return;
  await Notifications.deleteNotificationChannelAsync(channelId);
}

/**
 * List all existing channels (useful for debugging).
 */
export async function listChannels() {
  if (Platform.OS !== 'android') return [];
  return Notifications.getNotificationChannelsAsync();
}
```

### 12.3 Channel Migration Strategy

Since you can't change a channel's importance after creation, you sometimes need to migrate to a new channel:

```tsx
// Channel migration example:
// v1.0 had 'alerts' channel with DEFAULT importance
// v2.0 needs 'alerts' channel with HIGH importance

async function migrateChannels() {
  if (Platform.OS !== 'android') return;

  const existingChannels = await Notifications.getNotificationChannelsAsync();
  const alertsChannel = existingChannels.find((c) => c.id === 'alerts');

  if (alertsChannel && alertsChannel.importance !== Notifications.AndroidImportance.HIGH) {
    // Can't change importance — delete old, create new with different ID
    await Notifications.deleteNotificationChannelAsync('alerts');
    await Notifications.setNotificationChannelAsync('alerts_v2', {
      name: 'Alerts',
      description: 'Important alerts and warnings',
      importance: Notifications.AndroidImportance.HIGH,
      sound: 'default',
    });
    // Update your server to send to 'alerts_v2' channel for this device
  }
}
```

### 12.4 Importance Levels Explained

| Importance | Behavior |
|-----------|----------|
| `MAX` | Makes sound, shows as heads-up notification (pops over current app) |
| `HIGH` | Makes sound, may show as heads-up |
| `DEFAULT` | Makes sound, shows in notification shade |
| `LOW` | No sound, shows in notification shade |
| `MIN` | No sound, no visual interruption, may not show in shade |
| `NONE` | Channel is disabled |

**My recommendation:** Start with fewer channels at higher importance levels. It's easier to add channels later than to deal with users who have disabled a channel you now need.

---

## 13. DEEP LINKING FROM NOTIFICATIONS

Deep linking from notifications means: user taps notification, app opens to the specific relevant screen. It sounds simple, but it's one of the most bug-prone features in mobile development because of the different code paths for warm vs cold starts.

### 13.1 The Two Code Paths

```
WARM START (app in memory):
  User taps notification
  → App comes to foreground
  → Response listener fires immediately
  → Navigation state exists
  → router.push() works
  → User sees correct screen ✓

COLD START (app killed):
  User taps notification
  → OS launches app from scratch
  → JS bundle loads
  → React mounts root component
  → Navigation initializes
  → THEN response data becomes available
  → But navigation might not be ready yet...
  → Timing bug! ✗
```

### 13.2 The Robust Solution

```tsx
// services/notificationDeepLink.ts
import * as Notifications from 'expo-notifications';
import { router } from 'expo-router';

class NotificationDeepLinkHandler {
  private isNavigationReady = false;
  private pendingDeepLink: any = null;
  private subscription: Notifications.Subscription | null = null;

  /**
   * Initialize the handler. Call in your root layout's useEffect.
   */
  initialize() {
    // 1. Check for cold-start notification
    this.checkInitialNotification();

    // 2. Listen for future notification taps
    this.subscription =
      Notifications.addNotificationResponseReceivedListener((response) => {
        this.handleResponse(response);
      });
  }

  /**
   * Call when your navigation is fully ready.
   */
  onNavigationReady() {
    this.isNavigationReady = true;
    this.processPendingDeepLink();
  }

  /**
   * Clean up listeners.
   */
  cleanup() {
    this.subscription?.remove();
  }

  private async checkInitialNotification() {
    const response =
      await Notifications.getLastNotificationResponseAsync();

    if (response) {
      // Validate that this notification is recent (not a stale one from hours ago)
      const timestamp = response.notification.date;
      const ageMs = Date.now() - timestamp;
      const MAX_AGE = 30 * 1000; // 30 seconds

      if (ageMs < MAX_AGE) {
        this.handleResponse(response);
      }
    }
  }

  private handleResponse(
    response: Notifications.NotificationResponse
  ) {
    const data = response.notification.request.content.data;

    if (this.isNavigationReady) {
      this.navigateToScreen(data);
    } else {
      // Queue it — navigation isn't ready yet
      this.pendingDeepLink = data;
    }
  }

  private processPendingDeepLink() {
    if (this.pendingDeepLink) {
      // Give navigation one more tick to fully initialize
      setTimeout(() => {
        this.navigateToScreen(this.pendingDeepLink);
        this.pendingDeepLink = null;
      }, 150);
    }
  }

  private navigateToScreen(data: any) {
    if (!data) return;

    try {
      // Strategy 1: Explicit screen mapping
      if (data.type && data.screen) {
        router.push(data.screen);
        return;
      }

      // Strategy 2: Type-based routing
      switch (data.type) {
        case 'chat_message':
          router.push(`/chat/${data.chatId}`);
          break;
        case 'order_update':
          router.push(`/orders/${data.orderId}`);
          break;
        case 'profile_view':
          router.push(`/profile/${data.userId}`);
          break;
        default:
          // Strategy 3: Generic URL deep link
          if (data.url) {
            router.push(data.url);
          }
          break;
      }
    } catch (error) {
      console.error('Notification deep link navigation failed:', error);
      // Don't crash — the user will just see the home screen
    }
  }
}

export const notificationDeepLink = new NotificationDeepLinkHandler();
```

```tsx
// app/_layout.tsx — using the deep link handler
import { notificationDeepLink } from '../services/notificationDeepLink';

export default function RootLayout() {
  useEffect(() => {
    notificationDeepLink.initialize();
    return () => notificationDeepLink.cleanup();
  }, []);

  return (
    <View
      style={{ flex: 1 }}
      onLayout={() => {
        // Navigation is ready after first layout
        notificationDeepLink.onNavigationReady();
      }}
    >
      <Slot />
    </View>
  );
}
```

### 13.3 Testing Deep Links

```tsx
// __tests__/notificationDeepLink.test.ts
// Test your navigation mapping without needing a real notification

import { handleNotificationNavigation } from '../services/notificationHandler';

// Mock the router
jest.mock('expo-router', () => ({
  router: {
    push: jest.fn(),
    replace: jest.fn(),
  },
}));

import { router } from 'expo-router';

describe('Notification Deep Links', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('navigates to chat screen for chat_message type', () => {
    handleNotificationNavigation({
      type: 'chat_message',
      chatId: 'abc123',
    });

    expect(router.push).toHaveBeenCalledWith('/chat/abc123');
  });

  it('navigates to order screen for order_update type', () => {
    handleNotificationNavigation({
      type: 'order_update',
      orderId: 'order456',
    });

    expect(router.push).toHaveBeenCalledWith('/orders/order456');
  });

  it('uses generic URL when provided', () => {
    handleNotificationNavigation({
      type: 'promo',
      url: '/deals/summer-sale',
    });

    expect(router.push).toHaveBeenCalledWith('/deals/summer-sale');
  });

  it('does nothing for empty data', () => {
    handleNotificationNavigation({});
    expect(router.push).not.toHaveBeenCalled();
  });

  it('does nothing for null data', () => {
    handleNotificationNavigation(null as any);
    expect(router.push).not.toHaveBeenCalled();
  });
});
```

---

## 14. TESTING NOTIFICATIONS

Push notification testing is uniquely painful because of platform restrictions, device requirements, and the number of states to test. Here's your testing strategy.

### 14.1 The Testing Matrix

| Scenario | iOS Simulator | iOS Device | Android Emulator | Android Device |
|----------|:------------:|:----------:|:---------------:|:-------------:|
| Local notifications | Yes | Yes | Yes | Yes |
| Push (remote) notifications | **No** | Yes | Yes (FCM) | Yes |
| Notification permissions prompt | No | Yes | Yes | Yes |
| Notification categories/actions | Partial | Yes | Yes | Yes |
| Notification channels | N/A | N/A | Yes | Yes |
| Cold start from notification | Difficult | Yes | Yes | Yes |
| Background notification delivery | No | Yes | Yes | Yes |

**The bottom line:** You cannot fully test push notifications on iOS without a physical device. APNs simply does not work on the simulator. Android emulators with Google Play Services do support FCM.

### 14.2 Using the Expo Push Tool

Expo provides a web-based tool for sending test notifications: https://expo.dev/notifications

```tsx
// Get your token and send a test
async function printPushToken() {
  const token = await Notifications.getExpoPushTokenAsync({
    projectId: Constants.expoConfig?.extra?.eas?.projectId,
  });
  console.log('Send test notifications to:', token.data);
  // Copy this token and paste it into https://expo.dev/notifications
}
```

### 14.3 Testing with curl

```bash
# Send a test notification via Expo Push API
curl -X POST https://exp.host/--/api/v2/push/send \
  -H "Content-Type: application/json" \
  -d '{
    "to": "ExponentPushToken[xxxxxxxxxxxxxx]",
    "title": "Test Notification",
    "body": "This is a test from curl",
    "data": {
      "type": "chat_message",
      "chatId": "test123"
    },
    "sound": "default",
    "channelId": "messages"
  }'
```

```bash
# Send multiple notifications at once
curl -X POST https://exp.host/--/api/v2/push/send \
  -H "Content-Type: application/json" \
  -d '[
    {
      "to": "ExponentPushToken[token1]",
      "title": "Test 1",
      "body": "First notification"
    },
    {
      "to": "ExponentPushToken[token2]",
      "title": "Test 2",
      "body": "Second notification"
    }
  ]'
```

### 14.4 Testing Local Notifications Programmatically

For development, add a debug screen that triggers notifications:

```tsx
// screens/DevNotificationTest.tsx (only include in dev builds)
import { View, Text, Pressable, StyleSheet, ScrollView } from 'react-native';
import * as Notifications from 'expo-notifications';

export function DevNotificationTestScreen() {
  const tests = [
    {
      name: 'Basic Notification',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            title: 'Basic Test',
            body: 'This is a basic notification',
          },
          trigger: null,
        }),
    },
    {
      name: 'With Data (Chat)',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            title: 'New Message',
            body: 'Hey, are you there?',
            data: { type: 'chat_message', chatId: 'test123' },
          },
          trigger: null,
        }),
    },
    {
      name: 'With Action Buttons',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            title: 'Friend Request',
            body: 'Alex wants to be your friend',
            data: { type: 'friend_request', requestId: 'req456' },
            categoryIdentifier: 'friend_request',
          },
          trigger: null,
        }),
    },
    {
      name: 'Delayed (5 seconds)',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            title: 'Delayed Notification',
            body: 'This arrived after 5 seconds',
          },
          trigger: {
            type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
            seconds: 5,
          },
        }),
    },
    {
      name: 'Silent (Data Only)',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            data: { type: 'silent_sync', action: 'refresh' },
          },
          trigger: null,
        }),
    },
    {
      name: 'Badge Count = 5',
      fire: () =>
        Notifications.scheduleNotificationAsync({
          content: {
            title: 'Badge Test',
            body: 'Check the app icon',
            badge: 5,
          },
          trigger: null,
        }),
    },
    {
      name: 'Clear Badge',
      fire: () => Notifications.setBadgeCountAsync(0),
    },
    {
      name: 'List Scheduled',
      fire: async () => {
        const scheduled =
          await Notifications.getAllScheduledNotificationsAsync();
        console.log('Scheduled:', JSON.stringify(scheduled, null, 2));
        alert(`${scheduled.length} notifications scheduled`);
      },
    },
    {
      name: 'Cancel All Scheduled',
      fire: () => Notifications.cancelAllScheduledNotificationsAsync(),
    },
  ];

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.header}>Notification Testing</Text>
      {tests.map((test) => (
        <Pressable
          key={test.name}
          style={styles.button}
          onPress={test.fire}
        >
          <Text style={styles.buttonText}>{test.name}</Text>
        </Pressable>
      ))}
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#FFFFFF',
  },
  header: {
    fontSize: 24,
    fontWeight: '700',
    color: '#1A1A1A',
    marginBottom: 24,
  },
  button: {
    backgroundColor: '#007AFF',
    padding: 16,
    borderRadius: 12,
    marginBottom: 12,
  },
  buttonText: {
    color: '#FFFFFF',
    fontSize: 16,
    fontWeight: '600',
    textAlign: 'center',
  },
});
```

### 14.5 Common Testing Gotchas

**Gotcha 1: iOS simulator doesn't receive push notifications.**
APNs requires a real device. Use local notifications for simulator testing, and a physical device for push testing. Android emulators with Google Play Services can receive FCM pushes.

**Gotcha 2: The Expo Go app has its own push token.**
When developing with Expo Go, the push token belongs to the Expo Go app, not your app. Notifications will work for testing, but the behavior (channels, categories, etc.) may differ from a standalone build.

**Gotcha 3: APNs has separate sandbox and production environments.**
Development builds use the APNs sandbox. Production builds use APNs production. If you're sending from your server, make sure you're hitting the right environment. EAS Build handles this automatically based on your build profile.

**Gotcha 4: Token changes silently.**
The push token can change without warning (app reinstall, iOS version update, token refresh). Your app should re-register the token at every launch and send it to your server.

**Gotcha 5: Cold start testing is manual and tedious.**
To test cold-start notification handling:
1. Kill the app completely (swipe up from app switcher)
2. Send a push notification
3. Wait for it to appear
4. Tap the notification
5. Verify the app opens to the correct screen

There's no good way to automate this in a test suite. Build it right, test it manually, and add unit tests for the navigation mapping (see section 13.3).

**Gotcha 6: Android 13+ requires runtime notification permission.**
Starting with Android 13 (API 33), the `POST_NOTIFICATIONS` permission must be requested at runtime, similar to iOS. `expo-notifications` handles this through `requestPermissionsAsync()`, but be aware that on older Android versions, permission is granted by default at install time.

---

## 15. SERVER-SIDE SENDING

You've set up the client. Now let's cover the server side — how to actually send notifications to your users.

### 15.1 Using the Expo Push API

The simplest option. Send notifications to Expo Push Tokens, and Expo handles routing to FCM/APNs.

```typescript
// server/services/pushNotifications.ts (Node.js server)
import { Expo, ExpoPushMessage, ExpoPushTicket } from 'expo-server-sdk';

const expo = new Expo();

/**
 * Send push notifications to one or more Expo Push Tokens.
 */
export async function sendPushNotifications(
  messages: ExpoPushMessage[]
): Promise<void> {
  // Filter out invalid tokens
  const validMessages = messages.filter((msg) => {
    if (typeof msg.to === 'string') {
      return Expo.isExpoPushToken(msg.to);
    }
    // msg.to can be an array of tokens
    return true;
  });

  if (validMessages.length === 0) {
    console.log('No valid push tokens to send to');
    return;
  }

  // Chunk messages — Expo API has a limit per request
  const chunks = expo.chunkPushNotifications(validMessages);
  const tickets: ExpoPushTicket[] = [];

  for (const chunk of chunks) {
    try {
      const ticketChunk = await expo.sendPushNotificationsAsync(chunk);
      tickets.push(...ticketChunk);
    } catch (error) {
      console.error('Error sending push notification chunk:', error);
    }
  }

  // Process tickets to check for errors
  await processTickets(tickets, validMessages);
}

/**
 * Process push notification tickets to handle errors.
 * Some errors (like invalid tokens) should trigger token cleanup.
 */
async function processTickets(
  tickets: ExpoPushTicket[],
  messages: ExpoPushMessage[]
) {
  const receiptIds: string[] = [];

  for (let i = 0; i < tickets.length; i++) {
    const ticket = tickets[i];
    const message = messages[i];

    if (ticket.status === 'ok') {
      receiptIds.push(ticket.id);
    } else if (ticket.status === 'error') {
      console.error(
        `Push notification error: ${ticket.message}`,
        ticket.details
      );

      // Handle specific error types
      if (ticket.details?.error === 'DeviceNotRegistered') {
        // The token is no longer valid — remove it from your database
        const token = typeof message.to === 'string' ? message.to : message.to[0];
        await removeInvalidPushToken(token);
      }
    }
  }

  // Check receipts after a delay (Expo recommends 15+ minutes)
  // In practice, schedule this as a background job
  if (receiptIds.length > 0) {
    // Store receipt IDs for later checking
    await scheduleReceiptCheck(receiptIds);
  }
}

/**
 * Check push notification receipts.
 * Call this 15+ minutes after sending.
 */
export async function checkReceipts(receiptIds: string[]) {
  const chunks = expo.chunkPushNotificationReceiptIds(receiptIds);

  for (const chunk of chunks) {
    try {
      const receipts = await expo.getPushNotificationReceiptsAsync(chunk);

      for (const [receiptId, receipt] of Object.entries(receipts)) {
        if (receipt.status === 'ok') {
          // Delivered successfully
          continue;
        }

        if (receipt.status === 'error') {
          console.error(
            `Receipt error for ${receiptId}: ${receipt.message}`,
            receipt.details
          );

          if (receipt.details?.error === 'DeviceNotRegistered') {
            // Token is invalid — clean up
            // You'll need to look up which token this receipt corresponds to
            await handleInvalidToken(receiptId);
          }
        }
      }
    } catch (error) {
      console.error('Error checking receipts:', error);
    }
  }
}
```

### 15.2 Practical Sending Patterns

```typescript
// server/services/notificationSender.ts

interface NotificationTarget {
  userId: string;
  pushTokens: string[]; // A user may have multiple devices
}

/**
 * Send a notification to a specific user (all their devices).
 */
export async function notifyUser(
  userId: string,
  notification: {
    title: string;
    body: string;
    data?: Record<string, any>;
    channelId?: string;
    badge?: number;
    sound?: string;
    categoryId?: string;
  }
) {
  // Get all push tokens for this user
  const tokens = await db.pushTokens.findMany({
    where: { userId },
    select: { token: true },
  });

  if (tokens.length === 0) {
    console.log(`User ${userId} has no push tokens`);
    return;
  }

  const messages: ExpoPushMessage[] = tokens.map(({ token }) => ({
    to: token,
    title: notification.title,
    body: notification.body,
    data: notification.data || {},
    channelId: notification.channelId || 'default',
    badge: notification.badge,
    sound: notification.sound || 'default',
    categoryId: notification.categoryId,
  }));

  await sendPushNotifications(messages);
}

/**
 * Send a notification to multiple users.
 */
export async function notifyUsers(
  userIds: string[],
  notification: {
    title: string;
    body: string;
    data?: Record<string, any>;
    channelId?: string;
  }
) {
  // Batch load all tokens
  const tokens = await db.pushTokens.findMany({
    where: { userId: { in: userIds } },
    select: { token: true, userId: true },
  });

  const messages: ExpoPushMessage[] = tokens.map(({ token }) => ({
    to: token,
    title: notification.title,
    body: notification.body,
    data: notification.data || {},
    channelId: notification.channelId || 'default',
    sound: 'default',
  }));

  await sendPushNotifications(messages);
}

/**
 * Send chat message notification.
 */
export async function sendChatNotification(
  recipientUserId: string,
  senderName: string,
  messagePreview: string,
  chatId: string
) {
  await notifyUser(recipientUserId, {
    title: senderName,
    body: messagePreview,
    data: {
      type: 'chat_message',
      chatId,
      screen: `/chat/${chatId}`,
    },
    channelId: 'messages',
    categoryId: 'chat_message',
    badge: await getUnreadCount(recipientUserId),
  });
}

/**
 * Send order update notification.
 */
export async function sendOrderNotification(
  userId: string,
  orderId: string,
  status: 'confirmed' | 'shipped' | 'delivered'
) {
  const statusMessages = {
    confirmed: 'Your order has been confirmed!',
    shipped: 'Your order is on its way!',
    delivered: 'Your order has been delivered!',
  };

  await notifyUser(userId, {
    title: 'Order Update',
    body: statusMessages[status],
    data: {
      type: 'order_update',
      orderId,
      status,
      screen: `/orders/${orderId}`,
    },
    channelId: 'orders',
  });
}
```

### 15.3 Direct FCM/APNs (Without Expo Push Service)

For production apps that need more control, you may want to send directly to FCM/APNs instead of going through Expo's Push Service.

**Direct FCM (v1 HTTP API):**

```typescript
// server/services/fcmSender.ts
import { google } from 'googleapis';

const PROJECT_ID = 'your-firebase-project-id';
const FCM_ENDPOINT = `https://fcm.googleapis.com/v1/projects/${PROJECT_ID}/messages:send`;

/**
 * Get OAuth2 access token for FCM.
 * Requires a Firebase service account key JSON file.
 */
async function getAccessToken(): Promise<string> {
  const auth = new google.auth.GoogleAuth({
    keyFile: './firebase-service-account.json',
    scopes: ['https://www.googleapis.com/auth/firebase.messaging'],
  });

  const client = await auth.getClient();
  const accessToken = await client.getAccessToken();
  return accessToken.token!;
}

/**
 * Send a push notification via FCM v1 API.
 */
export async function sendFCMNotification(
  deviceToken: string,
  notification: {
    title: string;
    body: string;
    data?: Record<string, string>; // FCM data values must be strings
    channelId?: string;
    imageUrl?: string;
  }
) {
  const accessToken = await getAccessToken();

  const message = {
    message: {
      token: deviceToken,
      notification: {
        title: notification.title,
        body: notification.body,
        image: notification.imageUrl,
      },
      data: notification.data || {},
      android: {
        notification: {
          channelId: notification.channelId || 'default',
          priority: 'HIGH' as const,
          defaultSound: true,
        },
      },
      apns: {
        payload: {
          aps: {
            alert: {
              title: notification.title,
              body: notification.body,
            },
            sound: 'default',
            badge: 1,
          },
        },
      },
    },
  };

  const response = await fetch(FCM_ENDPOINT, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(message),
  });

  if (!response.ok) {
    const error = await response.json();
    
    // Handle specific FCM errors
    if (error.error?.details?.[0]?.errorCode === 'UNREGISTERED') {
      // Token is no longer valid
      await removeInvalidFCMToken(deviceToken);
    }
    
    throw new Error(`FCM send failed: ${JSON.stringify(error)}`);
  }

  return response.json();
}
```

### 15.4 Token Management

Token management is the silent reliability killer. If you're not actively managing tokens, a significant percentage of your push notifications are being sent to tokens that no longer exist.

```typescript
// server/services/tokenManager.ts

interface StoredPushToken {
  id: string;
  userId: string;
  token: string;
  platform: 'ios' | 'android';
  deviceName: string;
  lastUsed: Date;
  createdAt: Date;
}

/**
 * Register or update a push token for a user.
 */
export async function registerPushToken(
  userId: string,
  token: string,
  platform: 'ios' | 'android',
  deviceName: string
) {
  // Upsert: if this token already exists (same device, maybe different user),
  // update it. If it's new, create it.
  await db.pushTokens.upsert({
    where: { token },
    update: {
      userId,       // Token might have moved to a different user (re-login)
      platform,
      deviceName,
      lastUsed: new Date(),
    },
    create: {
      userId,
      token,
      platform,
      deviceName,
      lastUsed: new Date(),
    },
  });

  // Also clean up old tokens for this user+device combination
  // (prevents accumulating stale tokens)
  await db.pushTokens.deleteMany({
    where: {
      userId,
      deviceName,
      token: { not: token }, // Keep the current token, delete others from same device
    },
  });
}

/**
 * Remove an invalid token.
 * Called when FCM/APNs/Expo reports the token as invalid.
 */
export async function removeInvalidPushToken(token: string) {
  const deleted = await db.pushTokens.delete({
    where: { token },
  });

  if (deleted) {
    console.log(
      `Removed invalid push token for user ${deleted.userId}: ${token.slice(0, 20)}...`
    );
  }
}

/**
 * Clean up stale tokens periodically.
 * Tokens that haven't been refreshed in 60+ days are likely invalid.
 * Run this as a daily cron job.
 */
export async function cleanupStaleTokens() {
  const cutoffDate = new Date();
  cutoffDate.setDate(cutoffDate.getDate() - 60);

  const result = await db.pushTokens.deleteMany({
    where: {
      lastUsed: { lt: cutoffDate },
    },
  });

  console.log(`Cleaned up ${result.count} stale push tokens`);
}

/**
 * Remove all tokens for a user (e.g., on account deletion or logout-from-all-devices).
 */
export async function removeAllTokensForUser(userId: string) {
  await db.pushTokens.deleteMany({
    where: { userId },
  });
}
```

### 15.5 Expo Push API vs Direct FCM/APNs: When to Use Which

| Factor | Expo Push API | Direct FCM/APNs |
|--------|:------------:|:---------------:|
| **Setup complexity** | Easy | Moderate-Hard |
| **Token format** | Expo Push Token | Device token (FCM/APNs) |
| **Extra hop** | Yes (Expo servers) | No |
| **Cost** | Free (Expo's infra) | Free (FCM/APNs) |
| **Rate limits** | Expo limits (generous) | FCM/APNs limits |
| **Reliability** | Depends on Expo uptime | Direct to platform |
| **Rich features** | Good coverage | Full platform capabilities |
| **Analytics** | Basic (receipts) | Full platform analytics |
| **When to use** | MVP, small-to-medium scale | Large scale, complex requirements |

**My recommendation:** Start with Expo Push API. It's significantly simpler to set up and works well for most apps. Switch to direct FCM/APNs when you need:
- Delivery guarantees that can't depend on a third party
- Advanced FCM features (topic messaging, condition targeting)
- FCM Analytics integration
- More than 600 notifications per second (Expo's rate limit)
- Custom APNs configurations

### 15.6 Batch Sending for Large Audiences

When sending notifications to large groups (thousands or millions of users), you need to be smart about it:

```typescript
// server/services/batchNotification.ts

/**
 * Send a notification to all users of a specific segment.
 * Handles batching, rate limiting, and error tracking.
 */
export async function sendBroadcastNotification(
  segment: {
    where: Record<string, any>; // Database filter for users
  },
  notification: {
    title: string;
    body: string;
    data?: Record<string, any>;
    channelId?: string;
  }
) {
  const BATCH_SIZE = 100;
  let offset = 0;
  let totalSent = 0;
  let totalFailed = 0;

  const expo = new Expo();

  while (true) {
    // Fetch a batch of tokens
    const tokens = await db.pushTokens.findMany({
      where: { user: segment.where },
      select: { token: true },
      take: BATCH_SIZE,
      skip: offset,
    });

    if (tokens.length === 0) break;

    // Build messages
    const messages: ExpoPushMessage[] = tokens
      .filter(({ token }) => Expo.isExpoPushToken(token))
      .map(({ token }) => ({
        to: token,
        title: notification.title,
        body: notification.body,
        data: notification.data || {},
        channelId: notification.channelId || 'default',
        sound: 'default' as const,
      }));

    // Send in chunks (Expo's chunk size)
    const chunks = expo.chunkPushNotifications(messages);

    for (const chunk of chunks) {
      try {
        const tickets = await expo.sendPushNotificationsAsync(chunk);
        totalSent += tickets.filter((t) => t.status === 'ok').length;
        totalFailed += tickets.filter((t) => t.status === 'error').length;
      } catch (error) {
        console.error('Batch send error:', error);
        totalFailed += chunk.length;
      }

      // Rate limit: don't overwhelm Expo's API
      await sleep(100); // 100ms between chunks
    }

    offset += BATCH_SIZE;
    console.log(`Progress: ${offset} tokens processed, ${totalSent} sent, ${totalFailed} failed`);
  }

  console.log(
    `Broadcast complete: ${totalSent} sent, ${totalFailed} failed`
  );

  return { totalSent, totalFailed };
}

function sleep(ms: number) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

---

## PUTTING IT ALL TOGETHER

Here's the complete integration — a root layout that handles splash screen, app state, notifications, deep links, and updates:

```tsx
// app/_layout.tsx — The production root layout
import { useEffect, useRef, useState, useCallback } from 'react';
import { View, Platform } from 'react-native';
import * as SplashScreen from 'expo-splash-screen';
import * as Font from 'expo-font';
import * as Notifications from 'expo-notifications';
import { Slot, router } from 'expo-router';
import { useAuthStore } from '../stores/auth';
import { registerForPushNotifications, savePushTokenToServer } from '../services/notifications';
import { notificationDeepLink } from '../services/notificationDeepLink';
import { setupNotificationChannels } from '../services/notificationChannels';
import { setupNotificationCategories } from '../services/notificationCategories';
import { checkForUpdate } from '../services/appUpdate';
import { checkForOTAUpdate } from '../services/otaUpdate';
import { registerBackgroundSync } from '../tasks/backgroundSync';
import { ForceUpdateGate } from '../components/ForceUpdateGate';
import { InAppNotificationBanner } from '../components/InAppNotificationBanner';
import { useOnForeground } from '../hooks/useOnForeground';
import { startupMetrics } from '../utils/startupMetrics';

// 1. Prevent splash screen from auto-hiding
SplashScreen.preventAutoHideAsync();

// 2. Configure notification foreground behavior
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

// 3. Set up Android notification channels
if (Platform.OS === 'android') {
  setupNotificationChannels();
}

export default function RootLayout() {
  const [appIsReady, setAppIsReady] = useState(false);
  const { user, restoreSession } = useAuthStore();

  // === STARTUP ===
  useEffect(() => {
    async function prepare() {
      try {
        await Promise.all([
          // Critical: block splash on these
          Font.loadAsync({
            'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
            'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
          }).then(() => startupMetrics.mark('fonts_loaded')),

          restoreSession()
            .then(() => startupMetrics.mark('auth_restored')),

          // Non-critical: don't block, but start early
          setupNotificationCategories().catch(console.warn),
        ]);
      } catch (e) {
        console.warn('Startup preparation failed:', e);
      } finally {
        setAppIsReady(true);
      }
    }

    prepare();

    // Initialize notification deep link handler
    notificationDeepLink.initialize();

    // Register background tasks
    registerBackgroundSync().catch(console.warn);

    return () => {
      notificationDeepLink.cleanup();
    };
  }, []);

  // === PUSH TOKEN REGISTRATION ===
  useEffect(() => {
    if (user) {
      registerForPushNotifications().then((token) => {
        if (token) {
          savePushTokenToServer(token, user.id);
        }
      });
    }
  }, [user]);

  // === FOREGROUND REFRESH ===
  useOnForeground(() => {
    // Check for OTA updates when returning to foreground
    checkForOTAUpdate().catch(console.warn);
  });

  // === LAYOUT CALLBACK ===
  const onLayoutRootView = useCallback(async () => {
    if (appIsReady) {
      startupMetrics.mark('first_render');
      await SplashScreen.hideAsync();
      startupMetrics.mark('splash_hidden');
      startupMetrics.report();

      // Mark navigation as ready for deep link handling
      notificationDeepLink.onNavigationReady();
    }
  }, [appIsReady]);

  if (!appIsReady) {
    return null;
  }

  return (
    <View style={{ flex: 1 }} onLayout={onLayoutRootView}>
      <ForceUpdateGate>
        <Slot />
        <InAppNotificationBanner />
      </ForceUpdateGate>
    </View>
  );
}
```

---

## CHAPTER CHECKLIST

Before shipping, verify you've handled:

- [ ] **AppState transitions** — data refreshes on foreground, sockets disconnect on background
- [ ] **Splash screen** — `preventAutoHideAsync()` called, splash hides after data loads, no white flash
- [ ] **Startup optimization** — critical work parallelized, non-critical work deferred
- [ ] **Background tasks** — defined at module level, imported in entry point, handles both platforms
- [ ] **Force update** — version check endpoint, blocking modal for minimum version, dismissible for recommended
- [ ] **OTA updates** — EAS Update configured, checks on foreground return
- [ ] **Push permissions** — requested at the right time (not on first launch), handles denied gracefully
- [ ] **Push token management** — token sent to server, re-registered on every launch, invalid tokens cleaned up
- [ ] **Foreground notifications** — handler configured, custom in-app banner or system alert
- [ ] **Background notifications** — response listener for notification taps
- [ ] **Cold start notifications** — `getLastNotificationResponseAsync()` checked, navigation readiness gated
- [ ] **Notification payload** — data field includes type and screen/URL for deep linking
- [ ] **Android channels** — created at startup, appropriate importance levels, migration strategy for changes
- [ ] **Deep linking from notifications** — works for warm start AND cold start, tested manually on both platforms
- [ ] **Local notifications** — scheduled correctly, managed (list, cancel), IDs stored for reference
- [ ] **Server-side sending** — batch support, error handling, receipt checking, token cleanup
- [ ] **Testing** — local notification test screen in dev, curl scripts for push testing, deep link unit tests

---

> **Next up:** [Chapter 11: Device APIs & Native Features] — Camera, file system, biometrics, and bridging the gap between JavaScript and native capabilities.
