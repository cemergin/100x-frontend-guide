<!--
  CHAPTER: 21
  TITLE: Firebase Console Mastery
  PART: V — Deployment & Operations
  PREREQS: Chapter 20
  KEY_TOPICS: Crashlytics, Analytics, Performance Monitoring, Remote Config, Cloud Messaging, App Distribution, @react-native-firebase
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 21: Firebase Console Mastery

> **Part V — Deployment & Operations** | Prerequisites: Chapter 20 | Difficulty: Intermediate

Firebase is one of those platforms that teams either use brilliantly or terribly. There is almost no middle ground. The teams using it brilliantly have Crashlytics alerts wired to Slack, custom analytics events that actually inform product decisions, Remote Config driving feature flags without app updates, and performance monitoring catching regressions before users complain. The teams using it terribly installed it once for push notifications, never looked at the console, and have 47,000 unread crash reports.

The difference is not the tool. It is whether someone on the team took the time to set each service up properly, instrument the right things, and build the feedback loops that make the data actionable. That is what this chapter is about.

We are going to go through every Firebase service that matters for React Native apps -- not as a docs walkthrough, but as a senior engineer showing you how to set them up in a production app, what to instrument, what to ignore, and how to build the operational awareness that separates teams who ship confidently from teams who ship and pray.

### In This Chapter
- Firebase for React Native -- @react-native-firebase setup, modular installs, Expo config plugins
- Crashlytics -- crash reporting, breadcrumbs, custom keys, non-fatal errors, debug symbols with EAS
- Analytics -- event tracking, user properties, funnels, retention, conversions
- Performance Monitoring -- startup traces, network monitoring, custom traces, screen rendering
- Remote Config -- feature flags, A/B testing, gradual rollouts, real-time updates
- Cloud Messaging (FCM) -- push notification setup, channels, foreground/background handling, topics
- App Distribution -- beta testing, tester management, EAS integration

### Related Chapters
- [Ch 20: EAS & App Store Submission] -- build pipeline that feeds into Firebase services
- [Ch 22: Security & Data Protection] -- securing Firebase credentials and API keys
- [Ch 14: Profiling & Debugging] -- using Firebase performance data alongside profiler output
- [Ch 13: Performance Optimization] -- acting on Firebase performance monitoring insights

---

## 1. FIREBASE FOR REACT NATIVE

### 1.1 Why @react-native-firebase

Let us clear up the confusion first. There are two ways to use Firebase in React Native:

1. **The Firebase JS SDK** (`firebase` npm package) -- the web SDK. Works in React Native because React Native has a JavaScript runtime. But it is designed for web browsers, not mobile apps.
2. **React Native Firebase** (`@react-native-firebase/*`) -- native SDKs for iOS and Android, wrapped with a JavaScript API. Uses the actual Firebase iOS SDK and Firebase Android SDK under the hood.

**Always use @react-native-firebase.** Here is why:

| Aspect | Firebase JS SDK | @react-native-firebase |
|--------|----------------|----------------------|
| Crashlytics | Not available | Full native integration |
| Analytics | Limited (web-style) | Full native integration |
| Performance | Not available | Full native integration |
| Push Notifications | Not available | Full FCM support |
| Offline persistence | Limited | Full native support |
| App startup impact | Heavier (JS-based init) | Native init (faster) |
| Bundle size | Larger (includes web code) | Smaller (native modules) |

The JS SDK literally cannot do half the things we are covering in this chapter. Crashlytics, performance monitoring, and push notifications require native code. There is no JavaScript-only path to catching native crashes or receiving background push notifications.

### 1.2 The Modular Approach

React Native Firebase is modular by design. You install only what you need:

```bash
# Core -- always required
npx expo install @react-native-firebase/app

# Individual services -- install only what you use
npx expo install @react-native-firebase/crashlytics
npx expo install @react-native-firebase/analytics
npx expo install @react-native-firebase/perf
npx expo install @react-native-firebase/remote-config
npx expo install @react-native-firebase/messaging
```

Each package adds its native SDK dependency. `@react-native-firebase/crashlytics` adds the Firebase Crashlytics iOS SDK and the Firebase Crashlytics Android SDK. If you do not install it, the native SDK is not linked, and your binary is smaller.

**This matters.** The full Firebase SDK suite is enormous. If you only need Crashlytics and Analytics, do not install Remote Config, Cloud Firestore, Realtime Database, and everything else "just in case." Every package adds to your binary size and cold start time.

Here is a realistic setup for a production app:

```bash
# The essentials for most production apps
npx expo install @react-native-firebase/app
npx expo install @react-native-firebase/crashlytics
npx expo install @react-native-firebase/analytics
npx expo install @react-native-firebase/perf
npx expo install @react-native-firebase/messaging
npx expo install @react-native-firebase/remote-config

# Only if you actually use these
# npx expo install @react-native-firebase/firestore
# npx expo install @react-native-firebase/auth
# npx expo install @react-native-firebase/storage
```

### 1.3 Expo Config Plugin Integration

If you are using Expo (and you should be -- see Chapter 5), React Native Firebase integrates through Expo config plugins. This means you do not need to manually edit `AppDelegate.m`, `MainApplication.java`, or any native files. The config plugin handles all native configuration during the prebuild step.

**app.json / app.config.ts:**

```typescript
// app.config.ts
import { ExpoConfig } from 'expo/config';

const config: ExpoConfig = {
  name: 'MyApp',
  slug: 'my-app',
  // ... other config

  ios: {
    bundleIdentifier: 'com.mycompany.myapp',
    googleServicesFile: './GoogleService-Info.plist',
  },
  android: {
    package: 'com.mycompany.myapp',
    googleServicesFile: './google-services.json',
  },

  plugins: [
    '@react-native-firebase/app',
    '@react-native-firebase/crashlytics',
    [
      'expo-build-properties',
      {
        ios: {
          useFrameworks: 'static',
        },
      },
    ],
  ],
};

export default config;
```

**The `useFrameworks: 'static'` bit is critical.** Firebase iOS SDKs require static frameworks. Without this, you will get build errors on iOS that are confusing and hard to debug. Set it once and forget about it.

### 1.4 Firebase Project Setup

Before any code works, you need Firebase configuration files:

1. Go to the [Firebase Console](https://console.firebase.google.com)
2. Create a project (or use an existing one)
3. Add an iOS app with your `bundleIdentifier`
4. Add an Android app with your `package` name
5. Download `GoogleService-Info.plist` (iOS) and `google-services.json` (Android)
6. Place them in your project root

```
my-app/
├── app.config.ts
├── GoogleService-Info.plist    <- iOS config
├── google-services.json        <- Android config
├── src/
│   └── ...
└── package.json
```

**Multiple environments?** You will need separate Firebase projects for development, staging, and production. The config files will differ per environment.

```typescript
// app.config.ts -- environment-aware Firebase config
const IS_PRODUCTION = process.env.APP_ENV === 'production';
const IS_STAGING = process.env.APP_ENV === 'staging';

const getGoogleServicesFile = () => {
  if (IS_PRODUCTION) return './firebase/production/google-services.json';
  if (IS_STAGING) return './firebase/staging/google-services.json';
  return './firebase/development/google-services.json';
};

const getGoogleServiceInfoPlist = () => {
  if (IS_PRODUCTION) return './firebase/production/GoogleService-Info.plist';
  if (IS_STAGING) return './firebase/staging/GoogleService-Info.plist';
  return './firebase/development/GoogleService-Info.plist';
};

const config: ExpoConfig = {
  // ...
  ios: {
    bundleIdentifier: IS_PRODUCTION
      ? 'com.mycompany.myapp'
      : IS_STAGING
        ? 'com.mycompany.myapp.staging'
        : 'com.mycompany.myapp.dev',
    googleServicesFile: getGoogleServiceInfoPlist(),
  },
  android: {
    package: IS_PRODUCTION
      ? 'com.mycompany.myapp'
      : IS_STAGING
        ? 'com.mycompany.myapp.staging'
        : 'com.mycompany.myapp.dev',
    googleServicesFile: getGoogleServicesFile(),
  },
};
```

**Directory structure for multi-environment:**

```
my-app/
├── firebase/
│   ├── development/
│   │   ├── GoogleService-Info.plist
│   │   └── google-services.json
│   ├── staging/
│   │   ├── GoogleService-Info.plist
│   │   └── google-services.json
│   └── production/
│       ├── GoogleService-Info.plist
│       └── google-services.json
├── app.config.ts
└── ...
```

### 1.5 Initialization

Unlike the JS SDK, `@react-native-firebase/app` initializes automatically from the native config files. You do not need to call `initializeApp()` with a config object. The native SDKs read `GoogleService-Info.plist` (iOS) and `google-services.json` (Android) at app startup.

```typescript
// You do NOT need this:
// firebase.initializeApp({ apiKey: '...', projectId: '...' });

// Just import and use:
import crashlytics from '@react-native-firebase/crashlytics';
import analytics from '@react-native-firebase/analytics';

// They are ready to use immediately
crashlytics().log('App started');
analytics().logEvent('app_open');
```

This is one of the benefits of the native approach -- initialization is handled by the platform, not your JavaScript. It is faster and more reliable.

### 1.6 Verifying the Setup

After installation and configuration, verify everything is working:

```typescript
// src/utils/firebase-check.ts
import firebase from '@react-native-firebase/app';

export function verifyFirebaseSetup(): void {
  // Check that Firebase is initialized
  const app = firebase.app();

  console.log('Firebase app name:', app.name);
  console.log('Firebase options:', {
    appId: app.options.appId,
    projectId: app.options.projectId,
    // Don't log the API key in production!
    ...(__DEV__ && { apiKey: app.options.apiKey }),
  });

  // Verify each service is available
  try {
    const modules: string[] = [];

    if (firebase.app().crashlytics) modules.push('Crashlytics');
    if (firebase.app().analytics) modules.push('Analytics');
    if (firebase.app().perf) modules.push('Performance');
    if (firebase.app().messaging) modules.push('Messaging');
    if (firebase.app().remoteConfig) modules.push('Remote Config');

    console.log('Available Firebase modules:', modules.join(', '));
  } catch (error) {
    console.error('Firebase module check failed:', error);
  }
}
```

Run this once during development to make sure everything is wired up correctly. Then remove it.

---

## 2. CRASHLYTICS

Crashlytics is, dollar for dollar, the most valuable Firebase service for a React Native team. A good Crashlytics setup is the difference between "we think the app is stable" and "we know the app is stable, and here is the data."

### 2.1 Setup

If you followed section 1, you already have `@react-native-firebase/crashlytics` installed and the config plugin registered. There is one more thing you need for meaningful crash reports: **debug symbols.**

For Android, the Crashlytics Gradle plugin needs to upload mapping files (for ProGuard/R8) and native symbol files:

```typescript
// app.config.ts -- Crashlytics plugin configuration
plugins: [
  '@react-native-firebase/app',
  [
    '@react-native-firebase/crashlytics',
    {
      // Upload native symbols for better stack traces
      android: {
        enableNativeSymbolUpload: true,
      },
    },
  ],
],
```

For iOS, dSYM files are uploaded automatically if your build process includes the Crashlytics run script. The Expo config plugin handles this, but you need to verify it is working (more on this in the EAS section).

### 2.2 Crash Reporting Fundamentals

Crashlytics automatically captures:
- **Native crashes** (SIGSEGV, SIGABRT, etc.) -- when the native code crashes
- **Unhandled JavaScript exceptions** -- when your JS throws an uncaught error
- **ANRs (Application Not Responding)** -- when the main thread is blocked for too long (Android)

You do not need to write any code for these. Install the package, build your app, and crashes are reported. But the default reports are only the beginning.

### 2.3 Breadcrumbs

Breadcrumbs are log messages that provide context for a crash. When a crash happens, Crashlytics includes the last breadcrumbs leading up to it. This is invaluable for understanding *what the user was doing* when the crash occurred.

```typescript
// src/services/crashlytics.ts
import crashlytics from '@react-native-firebase/crashlytics';

class CrashlyticsService {
  /**
   * Log a breadcrumb. These appear in crash reports
   * to show what happened before the crash.
   */
  log(message: string): void {
    crashlytics().log(message);
  }

  /**
   * Log navigation events as breadcrumbs.
   * Connect this to your navigation state listener.
   */
  logNavigation(screenName: string, params?: Record<string, unknown>): void {
    crashlytics().log(
      `Navigate: ${screenName}${params ? ` (${JSON.stringify(params)})` : ''}`
    );
  }

  /**
   * Log API calls as breadcrumbs.
   * Connect this to your API client interceptor.
   */
  logApiCall(method: string, url: string, status?: number): void {
    const statusStr = status ? ` -> ${status}` : '';
    crashlytics().log(`API: ${method} ${url}${statusStr}`);
  }

  /**
   * Log user actions as breadcrumbs.
   */
  logAction(action: string, details?: string): void {
    crashlytics().log(`Action: ${action}${details ? ` (${details})` : ''}`);
  }
}

export const crashlyticsService = new CrashlyticsService();
```

**Integrating breadcrumbs with navigation:**

```typescript
// src/navigation/NavigationContainer.tsx
import {
  NavigationContainer,
  NavigationState,
} from '@react-navigation/native';
import { useRef } from 'react';
import { crashlyticsService } from '../services/crashlytics';
import crashlytics from '@react-native-firebase/crashlytics';

function getActiveRouteName(state: NavigationState): string {
  const route = state.routes[state.index];
  if (route.state) {
    return getActiveRouteName(route.state as NavigationState);
  }
  return route.name;
}

export function AppNavigationContainer({
  children,
}: {
  children: React.ReactNode;
}) {
  const routeNameRef = useRef<string>();

  const onStateChange = (state: NavigationState | undefined) => {
    if (!state) return;

    const currentRouteName = getActiveRouteName(state);
    const previousRouteName = routeNameRef.current;

    if (previousRouteName !== currentRouteName) {
      // Log navigation breadcrumb
      crashlyticsService.logNavigation(currentRouteName);

      // Also set the current screen for Crashlytics grouping
      crashlytics().setAttribute('last_screen', currentRouteName);
    }

    routeNameRef.current = currentRouteName;
  };

  return (
    <NavigationContainer onStateChange={onStateChange}>
      {children}
    </NavigationContainer>
  );
}
```

**Integrating breadcrumbs with your API client:**

```typescript
// src/api/client.ts
import { crashlyticsService } from '../services/crashlytics';

// If using axios:
api.interceptors.request.use((config) => {
  crashlyticsService.logApiCall(
    config.method?.toUpperCase() ?? 'GET',
    config.url ?? 'unknown'
  );
  return config;
});

api.interceptors.response.use(
  (response) => {
    crashlyticsService.logApiCall(
      response.config.method?.toUpperCase() ?? 'GET',
      response.config.url ?? 'unknown',
      response.status
    );
    return response;
  },
  (error) => {
    if (error.response) {
      crashlyticsService.logApiCall(
        error.config?.method?.toUpperCase() ?? 'GET',
        error.config?.url ?? 'unknown',
        error.response.status
      );
    }
    return Promise.reject(error);
  }
);
```

### 2.4 Custom Keys

Custom keys are metadata that gets attached to crash reports. Unlike breadcrumbs (which are time-ordered logs), custom keys are key-value pairs that describe the state of the app at the time of the crash.

```typescript
// src/services/crashlytics.ts (continued)
class CrashlyticsService {
  // ... previous methods

  /**
   * Set the current user ID. Appears on all crash reports.
   */
  setUserId(userId: string): void {
    crashlytics().setUserId(userId);
  }

  /**
   * Set custom key-value pairs that appear on crash reports.
   */
  setCustomKeys(keys: Record<string, string | number | boolean>): void {
    Object.entries(keys).forEach(([key, value]) => {
      crashlytics().setAttribute(key, String(value));
    });
  }

  /**
   * Set multiple attributes at once (more efficient).
   */
  setAttributes(attributes: Record<string, string>): void {
    crashlytics().setAttributes(attributes);
  }

  /**
   * Set user context. Call this after login.
   */
  setUserContext(user: {
    id: string;
    email?: string;
    plan?: string;
    appVersion?: string;
  }): void {
    crashlytics().setUserId(user.id);
    crashlytics().setAttributes({
      email: user.email ?? 'unknown',
      plan: user.plan ?? 'free',
      app_version: user.appVersion ?? 'unknown',
    });
  }

  /**
   * Clear user context. Call this after logout.
   */
  clearUserContext(): void {
    crashlytics().setUserId('');
    crashlytics().setAttributes({
      email: '',
      plan: '',
    });
  }
}
```

**What custom keys should you set?**

```typescript
// After login:
crashlyticsService.setUserContext({
  id: user.id,
  email: user.email,
  plan: user.subscription.plan,
  appVersion: Application.nativeApplicationVersion ?? 'unknown',
});

// When app state changes:
crashlyticsService.setCustomKeys({
  is_onboarded: user.hasCompletedOnboarding,
  feature_flag_new_ui: remoteConfig.getValue('new_ui_enabled').asBoolean(),
  total_items_in_cart: cart.items.length,
  last_sync_timestamp: lastSync.toISOString(),
  connectivity: netInfo.isConnected ? 'online' : 'offline',
});
```

When a crash report comes in, you will see these values alongside the stack trace. "Oh, this crash only happens for users on the free plan who are offline" -- that is the kind of insight that turns a three-day debugging session into a thirty-minute fix.

### 2.5 Non-Fatal Error Logging

Not every error is a crash. Network timeouts, JSON parsing failures, unexpected API responses -- these are errors that your app handles gracefully, but you still want to know about them.

```typescript
// src/services/crashlytics.ts (continued)
class CrashlyticsService {
  // ... previous methods

  /**
   * Record a non-fatal error. The app did not crash,
   * but something went wrong that we should track.
   */
  recordError(error: Error, context?: string): void {
    if (context) {
      crashlytics().log(`Error context: ${context}`);
    }
    crashlytics().recordError(error);
  }

  /**
   * Record an error with custom attributes for categorization.
   */
  recordErrorWithContext(
    error: Error,
    context: {
      screen?: string;
      action?: string;
      category?: string;
      metadata?: Record<string, string>;
    }
  ): void {
    if (context.screen) {
      crashlytics().log(`Error on screen: ${context.screen}`);
    }
    if (context.action) {
      crashlytics().log(`Error during action: ${context.action}`);
    }
    if (context.metadata) {
      Object.entries(context.metadata).forEach(([key, value]) => {
        crashlytics().setAttribute(`error_${key}`, value);
      });
    }
    crashlytics().recordError(error);
  }
}
```

**Practical usage in your app:**

```typescript
// In a data fetching function:
async function fetchUserProfile(userId: string): Promise<UserProfile> {
  try {
    const response = await api.get(`/users/${userId}`);
    return response.data;
  } catch (error) {
    // Non-fatal: the app will show an error state, not crash
    crashlyticsService.recordErrorWithContext(
      error instanceof Error ? error : new Error(String(error)),
      {
        screen: 'ProfileScreen',
        action: 'fetchUserProfile',
        category: 'api_error',
        metadata: {
          userId,
          statusCode: String(error?.response?.status ?? 'unknown'),
        },
      }
    );
    throw error; // Re-throw so the UI can handle it
  }
}
```

```typescript
// In an error boundary:
class AppErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    crashlyticsService.recordErrorWithContext(error, {
      category: 'react_error_boundary',
      metadata: {
        componentStack: errorInfo.componentStack ?? 'unknown',
      },
    });
  }

  render(): React.ReactNode {
    if (this.state.hasError) {
      return <ErrorFallbackScreen error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

### 2.6 Crash-Free Rate Tracking

The crash-free rate is your single most important stability metric. It measures the percentage of user sessions that do not experience a crash.

**Targets:**
- **99.9%+** -- excellent, industry-leading stability
- **99.5%-99.9%** -- good, most users are happy
- **99.0%-99.5%** -- concerning, investigate top crashes
- **< 99.0%** -- critical, your app has a stability problem

Firebase Crashlytics shows this on the dashboard, but you should set up alerting:

1. Go to Firebase Console > Crashlytics
2. Click the gear icon > Alerts
3. Set up alerts for:
   - New fatal issues (always)
   - Regressed issues (issues that were closed but reappeared)
   - Velocity alerts (when a crash suddenly affects many users)

**Integrate with Slack:**

Firebase supports direct Slack integration for Crashlytics alerts. Go to Project Settings > Integrations > Slack. Set up a dedicated `#mobile-crashes` channel. Every new crash, regression, and velocity alert goes there.

### 2.7 ANR Analysis (Android)

ANRs (Application Not Responding) happen when the Android main thread is blocked for more than 5 seconds. They are one of the top reasons for poor Google Play Store ratings, and they are notoriously hard to debug without proper tooling.

Crashlytics captures ANRs automatically on Android. In the Firebase Console:

1. Go to Crashlytics
2. Filter by "ANRs" using the issue type filter
3. Look at the stack traces -- they show what was blocking the main thread

**Common ANR causes in React Native:**

```
+--------------------------------+----------------------------------------+
| ANR Cause                      | Fix                                    |
+--------------------------------+----------------------------------------+
| Large synchronous storage read | Use MMKV, or async-load with splash    |
| Heavy JS computation on start  | Defer work, use InteractionManager     |
| Synchronous native module call | Make the call async                    |
| Large image decode on UI thread| Use FastImage with proper caching      |
| Blocking network call          | Never block the main thread            |
| Database query on main thread  | Use background thread for queries      |
+--------------------------------+----------------------------------------+
```

### 2.8 Debug Symbol Upload with EAS

When you build with EAS Build, your crash reports need debug symbols to show meaningful stack traces instead of memory addresses. Without symbols, native crash reports are useless -- you will see `0x7fff5fbff8c0` instead of `MyViewController.handleButtonTap()`.

**For Android (ProGuard/R8 mapping files):**

The Crashlytics Gradle plugin handles this automatically during the build. Make sure the plugin is enabled:

```typescript
// app.config.ts
plugins: [
  [
    '@react-native-firebase/crashlytics',
    {
      android: {
        enableNativeSymbolUpload: true,
      },
    },
  ],
],
```

**For iOS (dSYM files):**

EAS Build produces dSYM files, but they need to be uploaded to Crashlytics. There are two approaches:

**Approach 1: Automatic upload during build (recommended)**

The `@react-native-firebase/crashlytics` config plugin adds a build phase script that uploads dSYMs automatically. This works out of the box with EAS Build.

**Approach 2: Manual upload with the Firebase CLI**

If automatic upload fails (it sometimes does with custom build configurations):

```bash
# Download dSYMs from EAS
eas build:list --platform ios --status finished --limit 1
# Get the build ID, then:
eas build:download --id <build-id> --output ./build.ipa

# Upload to Firebase using the CLI
firebase crashlytics:symbols:upload \
  --app=YOUR_IOS_APP_ID \
  ./path/to/dSYMs
```

**Verifying dSYM upload:**

Go to Firebase Console > Crashlytics > Settings > Missing dSYMs. If you see entries here, your dSYMs are not being uploaded correctly. Fix this immediately -- without dSYMs, your iOS crash reports are unreadable.

### 2.9 The Complete Crashlytics Setup

Putting it all together, here is the full service with everything we have discussed:

```typescript
// src/services/crashlytics.ts
import crashlytics from '@react-native-firebase/crashlytics';
import * as Application from 'expo-application';

interface UserContext {
  id: string;
  email?: string;
  plan?: string;
}

interface ErrorContext {
  screen?: string;
  action?: string;
  category?: string;
  metadata?: Record<string, string>;
}

class CrashlyticsService {
  private initialized = false;

  /**
   * Initialize Crashlytics with app-level context.
   * Call once at app startup.
   */
  async initialize(): Promise<void> {
    if (this.initialized) return;

    // Enable Crashlytics collection
    // In dev, you might want to disable it:
    await crashlytics().setCrashlyticsCollectionEnabled(!__DEV__);

    // Set app-level attributes
    crashlytics().setAttributes({
      app_version: Application.nativeApplicationVersion ?? 'unknown',
      build_number: Application.nativeBuildVersion ?? 'unknown',
    });

    this.initialized = true;
    this.log('Crashlytics initialized');
  }

  // --- Breadcrumbs ---

  log(message: string): void {
    crashlytics().log(message);
  }

  logNavigation(screenName: string): void {
    crashlytics().log(`Navigate: ${screenName}`);
    crashlytics().setAttribute('last_screen', screenName);
  }

  logApiCall(method: string, url: string, status?: number): void {
    const statusStr = status !== undefined ? ` -> ${status}` : '';
    crashlytics().log(`API: ${method} ${url}${statusStr}`);
  }

  logAction(action: string, details?: string): void {
    crashlytics().log(
      `Action: ${action}${details ? ` (${details})` : ''}`
    );
  }

  // --- User Context ---

  setUserContext(user: UserContext): void {
    crashlytics().setUserId(user.id);
    crashlytics().setAttributes({
      email: user.email ?? 'unknown',
      plan: user.plan ?? 'free',
    });
  }

  clearUserContext(): void {
    crashlytics().setUserId('');
    crashlytics().setAttributes({
      email: '',
      plan: '',
    });
  }

  // --- Custom Keys ---

  setCustomKeys(keys: Record<string, string | number | boolean>): void {
    const stringKeys: Record<string, string> = {};
    Object.entries(keys).forEach(([key, value]) => {
      stringKeys[key] = String(value);
    });
    crashlytics().setAttributes(stringKeys);
  }

  // --- Error Recording ---

  recordError(error: Error, context?: ErrorContext): void {
    if (context?.screen) {
      this.log(`Error on screen: ${context.screen}`);
    }
    if (context?.action) {
      this.log(`Error during action: ${context.action}`);
    }
    if (context?.category) {
      crashlytics().setAttribute('error_category', context.category);
    }
    if (context?.metadata) {
      const prefixed: Record<string, string> = {};
      Object.entries(context.metadata).forEach(([key, value]) => {
        prefixed[`error_${key}`] = value;
      });
      crashlytics().setAttributes(prefixed);
    }
    crashlytics().recordError(error);
  }

  /**
   * Force a test crash. Use during development
   * to verify Crashlytics is working.
   */
  testCrash(): void {
    crashlytics().crash();
  }
}

export const crashlyticsService = new CrashlyticsService();
```

**Testing your setup:**

```typescript
// In a dev-only screen or debug menu:
import { View } from 'react-native';
import { Button } from 'react-native';
import { crashlyticsService } from '../services/crashlytics';

function DebugScreen() {
  return (
    <View>
      <Button
        title="Test Crash"
        onPress={() => crashlyticsService.testCrash()}
      />
      <Button
        title="Test Non-Fatal"
        onPress={() => {
          crashlyticsService.recordError(
            new Error('Test non-fatal error'),
            { category: 'test', screen: 'DebugScreen' }
          );
        }}
      />
      <Button
        title="Test Breadcrumbs + Crash"
        onPress={() => {
          crashlyticsService.log('User pressed test button');
          crashlyticsService.log('About to crash intentionally');
          crashlyticsService.testCrash();
        }}
      />
    </View>
  );
}
```

> **Important:** Test crashes will not show up immediately. Crashlytics batches reports and sends them when the app next starts. After triggering a test crash, close the app, reopen it, wait a few minutes, and check the Firebase Console.

---

## 3. ANALYTICS

Analytics tells you what users *do*. Crashlytics tells you when things break. Together, they give you a complete picture of your app's health and usage.

### 3.1 Event Tracking Fundamentals

Firebase Analytics tracks two kinds of events:

1. **Automatically collected events** -- `first_open`, `session_start`, `app_update`, `os_update`, `screen_view`, etc. You get these for free.
2. **Custom events** -- anything you log with `logEvent()`.

```typescript
import analytics from '@react-native-firebase/analytics';

// Log a custom event
await analytics().logEvent('add_to_cart', {
  item_id: 'SKU_123',
  item_name: 'Premium Widget',
  item_category: 'widgets',
  price: 29.99,
  currency: 'USD',
});
```

### 3.2 Recommended Events vs Custom Events

Firebase has a set of **recommended events** with predefined parameters. Use these whenever possible because they unlock built-in reports and integrations (especially with Google Ads).

```typescript
// src/services/analytics.ts
import analytics from '@react-native-firebase/analytics';

class AnalyticsService {
  // --- Recommended E-Commerce Events ---

  async logViewItem(item: {
    id: string;
    name: string;
    category: string;
    price: number;
  }): Promise<void> {
    await analytics().logViewItem({
      items: [
        {
          item_id: item.id,
          item_name: item.name,
          item_category: item.category,
          price: item.price,
        },
      ],
    });
  }

  async logAddToCart(item: {
    id: string;
    name: string;
    category: string;
    price: number;
    quantity: number;
  }): Promise<void> {
    await analytics().logAddToCart({
      items: [
        {
          item_id: item.id,
          item_name: item.name,
          item_category: item.category,
          price: item.price,
          quantity: item.quantity,
        },
      ],
      value: item.price * item.quantity,
      currency: 'USD',
    });
  }

  async logPurchase(purchase: {
    transactionId: string;
    value: number;
    currency: string;
    items: Array<{
      id: string;
      name: string;
      category: string;
      price: number;
      quantity: number;
    }>;
  }): Promise<void> {
    await analytics().logPurchase({
      transaction_id: purchase.transactionId,
      value: purchase.value,
      currency: purchase.currency,
      items: purchase.items.map((item) => ({
        item_id: item.id,
        item_name: item.name,
        item_category: item.category,
        price: item.price,
        quantity: item.quantity,
      })),
    });
  }

  // --- Recommended Engagement Events ---

  async logSignUp(method: string): Promise<void> {
    await analytics().logSignUp({ method });
  }

  async logLogin(method: string): Promise<void> {
    await analytics().logLogin({ method });
  }

  async logSearch(searchTerm: string): Promise<void> {
    await analytics().logSearch({ search_term: searchTerm });
  }

  async logShare(contentType: string, itemId: string): Promise<void> {
    await analytics().logShare({
      content_type: contentType,
      item_id: itemId,
      method: 'in_app',
    });
  }

  // --- Custom Events ---

  async logCustomEvent(
    name: string,
    params?: Record<string, string | number | boolean>
  ): Promise<void> {
    await analytics().logEvent(name, params);
  }

  // --- Screen Tracking ---

  async logScreenView(
    screenName: string,
    screenClass?: string
  ): Promise<void> {
    await analytics().logScreenView({
      screen_name: screenName,
      screen_class: screenClass ?? screenName,
    });
  }
}

export const analyticsService = new AnalyticsService();
```

**Here is the full recommended events table for common app types:**

```
+----------------------+---------------------------------------------+
| Event Name           | When to Log                                 |
+----------------------+---------------------------------------------+
| sign_up              | User creates an account                     |
| login                | User logs in                                |
| tutorial_begin       | User starts onboarding                      |
| tutorial_complete    | User finishes onboarding                    |
| view_item            | User views a product/content detail         |
| view_item_list       | User views a list of products/content       |
| add_to_cart          | User adds item to cart                      |
| remove_from_cart     | User removes item from cart                 |
| begin_checkout       | User starts checkout                        |
| purchase             | User completes purchase                     |
| refund               | Purchase is refunded                        |
| search               | User performs a search                      |
| share                | User shares content                         |
| select_content       | User selects a content item                 |
| view_promotion       | User views a promotion/banner               |
| select_promotion     | User taps on a promotion/banner             |
+----------------------+---------------------------------------------+
```

### 3.3 User Properties

User properties are attributes you assign to segments of your user base. They are different from event parameters -- user properties persist across sessions and apply to all subsequent events.

```typescript
// src/services/analytics.ts (continued)
class AnalyticsService {
  // ... previous methods

  /**
   * Set user properties for segmentation.
   * These persist across sessions.
   */
  async setUserProperties(properties: {
    subscriptionPlan?: string;
    accountAge?: string;
    preferredLanguage?: string;
    notificationsEnabled?: string;
    darkMode?: string;
  }): Promise<void> {
    const entries = Object.entries(properties).filter(
      ([, value]) => value !== undefined
    );

    for (const [key, value] of entries) {
      await analytics().setUserProperty(key, value ?? null);
    }
  }

  /**
   * Set user ID for cross-device tracking.
   */
  async setUserId(userId: string | null): Promise<void> {
    await analytics().setUserId(userId);
  }
}
```

**What user properties should you set?**

```typescript
// After login or profile load:
analyticsService.setUserId(user.id);
analyticsService.setUserProperties({
  subscriptionPlan: user.plan, // 'free', 'pro', 'enterprise'
  accountAge: calculateAccountAge(user.createdAt),
  preferredLanguage: user.locale,
  notificationsEnabled: String(user.notificationsEnabled),
});
```

These properties are then available in the Firebase Console for:
- Filtering analytics reports ("show me events only from Pro users")
- Creating audiences ("users who are on the free plan AND have been active for 3+ months")
- A/B testing ("show this variant to enterprise users only")

### 3.4 Screen Tracking

Screen views are critical for understanding user journeys. There are two approaches:

**Approach 1: Manual screen tracking (recommended)**

```typescript
// src/navigation/NavigationContainer.tsx
import analytics from '@react-native-firebase/analytics';
import {
  NavigationContainer,
  NavigationState,
} from '@react-navigation/native';
import { useRef } from 'react';

function getActiveRouteName(state: NavigationState): string {
  const route = state.routes[state.index];
  if (route.state) {
    return getActiveRouteName(route.state as NavigationState);
  }
  return route.name;
}

export function AppNavigationContainer({
  children,
}: {
  children: React.ReactNode;
}) {
  const routeNameRef = useRef<string>();

  const onStateChange = async (state: NavigationState | undefined) => {
    if (!state) return;

    const currentRouteName = getActiveRouteName(state);
    const previousRouteName = routeNameRef.current;

    if (previousRouteName !== currentRouteName) {
      await analytics().logScreenView({
        screen_name: currentRouteName,
        screen_class: currentRouteName,
      });
    }

    routeNameRef.current = currentRouteName;
  };

  return (
    <NavigationContainer onStateChange={onStateChange}>
      {children}
    </NavigationContainer>
  );
}
```

**Approach 2: Per-screen tracking with a hook**

```typescript
// src/hooks/useScreenTracking.ts
import { useEffect } from 'react';
import { useIsFocused } from '@react-navigation/native';
import { analyticsService } from '../services/analytics';

export function useScreenTracking(screenName: string): void {
  const isFocused = useIsFocused();

  useEffect(() => {
    if (isFocused) {
      analyticsService.logScreenView(screenName);
    }
  }, [isFocused, screenName]);
}

// Usage in a screen:
function ProfileScreen() {
  useScreenTracking('Profile');
  // ...
}
```

I prefer Approach 1 for most apps -- it is centralized, does not require remembering to add a hook to every screen, and captures all navigation changes including deep links and push notification opens.

### 3.5 Funnels and Conversion Tracking

Funnels are sequences of events that track user progress through a flow. The classic example is an e-commerce funnel:

```
view_item -> add_to_cart -> begin_checkout -> purchase
```

To build meaningful funnels, you need to log events at each step:

```typescript
// src/features/checkout/hooks/useCheckoutTracking.ts

export function useCheckoutTracking() {
  const trackViewItem = (item: Product) => {
    analyticsService.logViewItem({
      id: item.id,
      name: item.name,
      category: item.category,
      price: item.price,
    });
  };

  const trackAddToCart = (item: Product, quantity: number) => {
    analyticsService.logAddToCart({
      id: item.id,
      name: item.name,
      category: item.category,
      price: item.price,
      quantity,
    });
  };

  const trackBeginCheckout = (cart: CartState) => {
    analyticsService.logCustomEvent('begin_checkout', {
      value: cart.total,
      currency: 'USD',
      items_count: cart.items.length,
    });
  };

  const trackPurchaseComplete = (order: Order) => {
    analyticsService.logPurchase({
      transactionId: order.id,
      value: order.total,
      currency: order.currency,
      items: order.items.map((item) => ({
        id: item.productId,
        name: item.name,
        category: item.category,
        price: item.price,
        quantity: item.quantity,
      })),
    });
  };

  return {
    trackViewItem,
    trackAddToCart,
    trackBeginCheckout,
    trackPurchaseComplete,
  };
}
```

In the Firebase Console, go to Analytics > Funnels to create a funnel visualization. You will see where users drop off, which is often more valuable than knowing what they complete.

### 3.6 Retention Analysis

Firebase Analytics has built-in cohort analysis for retention. Go to Analytics > Retention in the Firebase Console.

But the default retention view only shows you "did the user come back?" You can make it more meaningful by tracking specific retention events:

```typescript
// Track engagement quality, not just presence
analyticsService.logCustomEvent('meaningful_engagement', {
  engagement_type: 'content_consumed',
  session_duration_seconds: sessionDuration,
  content_count: contentViewed,
});
```

### 3.7 Consent and Privacy

You must handle analytics consent properly. GDPR, CCPA, and Apple's App Tracking Transparency require it.

```typescript
// src/services/analytics.ts (continued)
class AnalyticsService {
  // ... previous methods

  /**
   * Set analytics collection based on user consent.
   * Call this after the user responds to your consent prompt.
   */
  async setConsent(granted: boolean): Promise<void> {
    await analytics().setAnalyticsCollectionEnabled(granted);

    if (!granted) {
      // Also disable Crashlytics if consent is not granted
      // (depending on your privacy policy)
      await crashlytics().setCrashlyticsCollectionEnabled(false);
    }
  }

  /**
   * Set consent for specific purposes (iOS 14+ ATT).
   */
  async setConsentV2(consent: {
    analyticsStorage: 'granted' | 'denied';
    adStorage: 'granted' | 'denied';
    adUserData: 'granted' | 'denied';
    adPersonalization: 'granted' | 'denied';
  }): Promise<void> {
    await analytics().setConsent(consent);
  }
}
```

```typescript
// In your consent flow:
import { requestTrackingPermissionsAsync } from 'expo-tracking-transparency';

async function handleTrackingConsent(): Promise<void> {
  const { status } = await requestTrackingPermissionsAsync();
  const granted = status === 'granted';

  await analyticsService.setConsent(granted);
  await analyticsService.setConsentV2({
    analyticsStorage: granted ? 'granted' : 'denied',
    adStorage: granted ? 'granted' : 'denied',
    adUserData: granted ? 'granted' : 'denied',
    adPersonalization: granted ? 'granted' : 'denied',
  });
}
```

---

## 4. PERFORMANCE MONITORING

Firebase Performance Monitoring gives you real-world performance data from actual user devices. This is fundamentally different from profiling on your development machine -- your M3 MacBook does not represent the experience of a user on a 3-year-old Samsung Galaxy.

### 4.1 Setup

Performance monitoring is part of the `@react-native-firebase/perf` package:

```bash
npx expo install @react-native-firebase/perf
```

```typescript
// app.config.ts
plugins: [
  '@react-native-firebase/app',
  '@react-native-firebase/perf',
],
```

### 4.2 Automatic Traces

Firebase Performance automatically captures:

- **App startup trace** -- time from app launch to the first frame rendered
- **Screen rendering traces** -- slow and frozen frames per screen
- **HTTP/S network request traces** -- response time, payload size, success rate

You get these without writing any code. They appear in the Firebase Console under Performance.

### 4.3 Custom Traces

Automatic traces are a starting point. Custom traces let you measure what matters to your specific app.

```typescript
// src/services/performance.ts
import perf, { FirebasePerformanceTypes } from '@react-native-firebase/perf';

class PerformanceService {
  private traces: Map<string, FirebasePerformanceTypes.Trace> = new Map();

  /**
   * Start a custom trace. Call stopTrace() when the operation completes.
   */
  async startTrace(name: string): Promise<void> {
    try {
      const trace = await perf().startTrace(name);
      this.traces.set(name, trace);
    } catch (error) {
      // Performance monitoring should never crash the app
      console.warn(`Failed to start trace "${name}":`, error);
    }
  }

  /**
   * Stop a custom trace and optionally set metrics/attributes.
   */
  async stopTrace(
    name: string,
    options?: {
      metrics?: Record<string, number>;
      attributes?: Record<string, string>;
    }
  ): Promise<void> {
    try {
      const trace = this.traces.get(name);
      if (!trace) {
        console.warn(
          `Trace "${name}" not found. Was startTrace called?`
        );
        return;
      }

      // Set metrics (numeric values that can be aggregated)
      if (options?.metrics) {
        Object.entries(options.metrics).forEach(([key, value]) => {
          trace.putMetric(key, value);
        });
      }

      // Set attributes (string values for filtering)
      if (options?.attributes) {
        Object.entries(options.attributes).forEach(([key, value]) => {
          trace.putAttribute(key, value);
        });
      }

      await trace.stop();
      this.traces.delete(name);
    } catch (error) {
      console.warn(`Failed to stop trace "${name}":`, error);
      this.traces.delete(name);
    }
  }

  /**
   * Increment a metric on a running trace.
   */
  incrementMetric(
    traceName: string,
    metricName: string,
    value: number = 1
  ): void {
    const trace = this.traces.get(traceName);
    if (trace) {
      trace.incrementMetric(metricName, value);
    }
  }

  /**
   * Measure a function's execution time.
   */
  async measure<T>(
    name: string,
    fn: () => Promise<T>,
    options?: {
      attributes?: Record<string, string>;
    }
  ): Promise<T> {
    await this.startTrace(name);
    try {
      const result = await fn();
      await this.stopTrace(name, {
        attributes: options?.attributes,
      });
      return result;
    } catch (error) {
      await this.stopTrace(name, {
        attributes: {
          ...options?.attributes,
          error: 'true',
          error_message:
            error instanceof Error ? error.message : 'unknown',
        },
      });
      throw error;
    }
  }
}

export const performanceService = new PerformanceService();
```

### 4.4 Screen Load Traces

Measuring how long screens take to become interactive is one of the most impactful things you can instrument:

```typescript
// src/hooks/useScreenLoadTrace.ts
import { useEffect, useRef, useCallback } from 'react';
import { performanceService } from '../services/performance';

/**
 * Track how long a screen takes to load its data
 * and become interactive.
 *
 * Usage:
 *   const { markReady } = useScreenLoadTrace('ProfileScreen');
 *   // ... fetch data ...
 *   // When data is loaded and screen is ready:
 *   markReady();
 */
export function useScreenLoadTrace(screenName: string) {
  const isTracking = useRef(false);
  const startTime = useRef(Date.now());

  useEffect(() => {
    if (!isTracking.current) {
      isTracking.current = true;
      startTime.current = Date.now();
      performanceService.startTrace(`screen_load_${screenName}`);
    }

    return () => {
      // If the screen unmounts before markReady was called,
      // stop the trace with an 'abandoned' attribute
      if (isTracking.current) {
        performanceService.stopTrace(
          `screen_load_${screenName}`,
          { attributes: { completed: 'false' } }
        );
        isTracking.current = false;
      }
    };
  }, [screenName]);

  const markReady = useCallback(() => {
    if (isTracking.current) {
      const duration = Date.now() - startTime.current;
      performanceService.stopTrace(`screen_load_${screenName}`, {
        metrics: { duration_ms: duration },
        attributes: { completed: 'true' },
      });
      isTracking.current = false;
    }
  }, [screenName]);

  return { markReady };
}

// Usage in a screen component:
function ProfileScreen() {
  const { markReady } = useScreenLoadTrace('ProfileScreen');
  const { data, isLoading } = useQuery({
    queryKey: ['profile'],
    queryFn: fetchProfile,
  });

  useEffect(() => {
    if (data && !isLoading) {
      markReady();
    }
  }, [data, isLoading, markReady]);

  // ... render
}
```

### 4.5 Data Fetch Traces

Track how long your API calls take in the real world:

```typescript
// src/api/client.ts
import { performanceService } from '../services/performance';

// With axios interceptors:
api.interceptors.request.use(async (config) => {
  const traceId = `api_${config.method}_${new URL(config.url ?? '', config.baseURL).pathname}`;
  // Store trace ID on the config so we can access it in the response
  (config as any).__traceId = traceId;
  await performanceService.startTrace(traceId);
  return config;
});

api.interceptors.response.use(
  async (response) => {
    const traceId = (response.config as any).__traceId;
    if (traceId) {
      await performanceService.stopTrace(traceId, {
        metrics: {
          response_size: JSON.stringify(response.data).length,
        },
        attributes: {
          status_code: String(response.status),
          method: response.config.method?.toUpperCase() ?? 'GET',
        },
      });
    }
    return response;
  },
  async (error) => {
    const traceId = error?.config?.__traceId;
    if (traceId) {
      await performanceService.stopTrace(traceId, {
        attributes: {
          status_code: String(
            error.response?.status ?? 'network_error'
          ),
          error: 'true',
        },
      });
    }
    return Promise.reject(error);
  }
);
```

### 4.6 HTTP Metric Monitoring

Firebase Performance also supports explicit HTTP metrics for more control:

```typescript
// src/services/performance.ts (continued)
class PerformanceService {
  // ... previous methods

  /**
   * Create an HTTP metric for a network request.
   * More structured than custom traces for API calls.
   */
  async measureHttpRequest<T>(options: {
    url: string;
    method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
    fn: () => Promise<{
      data: T;
      status: number;
      contentType?: string;
    }>;
  }): Promise<T> {
    const metric = await perf().newHttpMetric(
      options.url,
      options.method
    );
    await metric.start();

    try {
      const result = await options.fn();
      metric.setHttpResponseCode(result.status);
      if (result.contentType) {
        metric.setResponseContentType(result.contentType);
      }
      await metric.stop();
      return result.data;
    } catch (error) {
      metric.setHttpResponseCode(0); // Network error
      await metric.stop();
      throw error;
    }
  }
}
```

### 4.7 Screen Rendering Performance

Firebase Performance automatically tracks slow and frozen frames. A **slow frame** takes more than 16ms to render (below 60fps). A **frozen frame** takes more than 700ms -- the user perceives the app as frozen.

These metrics appear per screen in the Firebase Console. But you can add more context:

```typescript
// src/hooks/useFrameRateMonitor.ts
import { useEffect } from 'react';
import { performanceService } from '../services/performance';

/**
 * Monitor frame drops on a specific screen.
 * Uses requestAnimationFrame to detect jank.
 */
export function useFrameRateMonitor(screenName: string) {
  useEffect(() => {
    let frameCount = 0;
    let lastTime = performance.now();
    let rafId: number;
    let slowFrames = 0;

    const measureFrame = () => {
      const now = performance.now();
      const delta = now - lastTime;
      lastTime = now;
      frameCount++;

      if (delta > 16.67) {
        slowFrames++;
      }

      rafId = requestAnimationFrame(measureFrame);
    };

    // Start measuring after mount
    rafId = requestAnimationFrame(measureFrame);

    // Report every 10 seconds
    const interval = setInterval(() => {
      if (frameCount > 0) {
        const slowFrameRatio = slowFrames / frameCount;
        if (slowFrameRatio > 0.1) {
          // More than 10% slow frames -- worth logging
          performanceService
            .startTrace(`jank_${screenName}`)
            .then(() => {
              performanceService.stopTrace(
                `jank_${screenName}`,
                {
                  metrics: {
                    slow_frame_ratio: Math.round(
                      slowFrameRatio * 100
                    ),
                    total_frames: frameCount,
                    slow_frames: slowFrames,
                  },
                }
              );
            });
        }
        frameCount = 0;
        slowFrames = 0;
      }
    }, 10000);

    return () => {
      cancelAnimationFrame(rafId);
      clearInterval(interval);
    };
  }, [screenName]);
}
```

### 4.8 Performance Monitoring Dashboard Best Practices

Once you have data flowing, here is how to read it:

```
+----------------------------------+--------------+------------------+
| Metric                           | Good         | Investigate      |
+----------------------------------+--------------+------------------+
| App start time (cold)            | < 2s         | > 4s             |
| App start time (warm)            | < 1s         | > 2s             |
| Screen load (data-dependent)     | < 1.5s       | > 3s             |
| API response time (p50)          | < 300ms      | > 1s             |
| API response time (p95)          | < 1s         | > 3s             |
| Slow rendering (% of frames)    | < 5%         | > 15%            |
| Frozen frames (% of frames)     | < 1%         | > 3%             |
| HTTP success rate                | > 99%        | < 95%            |
+----------------------------------+--------------+------------------+
```

---

## 5. REMOTE CONFIG

Remote Config is one of those Firebase services that seems simple but is incredibly powerful when used well. It lets you change your app's behavior and appearance without publishing an app update. Feature flags, A/B tests, gradual rollouts -- all controlled from the Firebase Console.

### 5.1 Setup

```bash
npx expo install @react-native-firebase/remote-config
```

### 5.2 Defaults and Fetching

Remote Config works with a defaults-first approach: you define default values in your app, then override them from the Firebase Console. If the fetch fails (user is offline, Firebase is down), the defaults are used.

```typescript
// src/services/remote-config.ts
import remoteConfig from '@react-native-firebase/remote-config';

// Define your defaults -- these are used before the first fetch
// and as fallbacks if fetch fails
const DEFAULTS = {
  // Feature flags
  new_checkout_flow_enabled: false,
  onboarding_v2_enabled: false,
  dark_mode_enabled: true,

  // Configuration values
  max_upload_size_mb: 10,
  api_timeout_seconds: 30,
  min_app_version: '1.0.0',

  // A/B test variants
  home_screen_layout: 'default',
  onboarding_steps_count: 3,

  // Feature rollout percentages (handled by Firebase conditions)
  premium_feature_preview: false,

  // Maintenance / kill switch
  maintenance_mode: false,
  maintenance_message:
    'We are performing scheduled maintenance. Please try again later.',
  force_update_enabled: false,
  force_update_message:
    'Please update to the latest version to continue.',
} as const;

type RemoteConfigDefaults = typeof DEFAULTS;
type RemoteConfigKey = keyof RemoteConfigDefaults;

class RemoteConfigService {
  private initialized = false;

  /**
   * Initialize Remote Config with defaults and fetch.
   * Call once at app startup.
   */
  async initialize(): Promise<void> {
    if (this.initialized) return;

    // Set defaults
    await remoteConfig().setDefaults(DEFAULTS);

    // Set minimum fetch interval
    // In development, use 0 for instant updates
    // In production, use 3600 (1 hour) or higher
    await remoteConfig().setConfigSettings({
      minimumFetchIntervalMillis: __DEV__ ? 0 : 3600000,
    });

    // Fetch and activate
    try {
      await remoteConfig().fetchAndActivate();
    } catch (error) {
      // Fetch failed -- defaults will be used
      console.warn(
        'Remote Config fetch failed, using defaults:',
        error
      );
    }

    this.initialized = true;
  }

  /**
   * Get a boolean value.
   */
  getBoolean(key: RemoteConfigKey): boolean {
    return remoteConfig().getValue(String(key)).asBoolean();
  }

  /**
   * Get a string value.
   */
  getString(key: RemoteConfigKey): string {
    return remoteConfig().getValue(String(key)).asString();
  }

  /**
   * Get a number value.
   */
  getNumber(key: RemoteConfigKey): number {
    return remoteConfig().getValue(String(key)).asNumber();
  }

  /**
   * Get the source of a value (default, remote, or static).
   * Useful for debugging.
   */
  getValueSource(
    key: RemoteConfigKey
  ): 'default' | 'remote' | 'static' {
    return remoteConfig().getValue(String(key)).getSource();
  }

  /**
   * Force a fetch (ignores minimum fetch interval).
   * Use sparingly -- there are quotas.
   */
  async forceRefresh(): Promise<void> {
    try {
      await remoteConfig().fetch(0);
      await remoteConfig().activate();
    } catch (error) {
      console.warn('Remote Config force refresh failed:', error);
    }
  }

  /**
   * Get all values as a typed object.
   * Useful for debugging or logging.
   */
  getAllValues(): Record<RemoteConfigKey, string> {
    const all = remoteConfig().getAll();
    const result: Record<string, string> = {};
    Object.entries(all).forEach(([key, value]) => {
      result[key] = value.asString();
    });
    return result as Record<RemoteConfigKey, string>;
  }
}

export const remoteConfigService = new RemoteConfigService();
```

### 5.3 Feature Flags

Feature flags are the most common use of Remote Config. Here is a clean pattern:

```typescript
// src/hooks/useFeatureFlags.ts
import { useEffect, useState } from 'react';
import { remoteConfigService } from '../services/remote-config';

export interface FeatureFlags {
  newCheckoutFlow: boolean;
  onboardingV2: boolean;
  darkMode: boolean;
  premiumFeaturePreview: boolean;
  maintenanceMode: boolean;
  forceUpdate: boolean;
  homeScreenLayout: 'default' | 'grid' | 'list';
}

export function useFeatureFlags(): FeatureFlags {
  const [flags, setFlags] = useState<FeatureFlags>(getFlags());

  useEffect(() => {
    // Re-read flags when the component mounts
    // (in case Remote Config was activated in the background)
    setFlags(getFlags());
  }, []);

  return flags;
}

function getFlags(): FeatureFlags {
  return {
    newCheckoutFlow: remoteConfigService.getBoolean(
      'new_checkout_flow_enabled'
    ),
    onboardingV2: remoteConfigService.getBoolean(
      'onboarding_v2_enabled'
    ),
    darkMode: remoteConfigService.getBoolean('dark_mode_enabled'),
    premiumFeaturePreview: remoteConfigService.getBoolean(
      'premium_feature_preview'
    ),
    maintenanceMode: remoteConfigService.getBoolean(
      'maintenance_mode'
    ),
    forceUpdate: remoteConfigService.getBoolean(
      'force_update_enabled'
    ),
    homeScreenLayout: remoteConfigService.getString(
      'home_screen_layout'
    ) as FeatureFlags['homeScreenLayout'],
  };
}
```

**Using feature flags in components:**

```typescript
function CheckoutScreen() {
  const { newCheckoutFlow } = useFeatureFlags();

  if (newCheckoutFlow) {
    return <NewCheckoutFlow />;
  }
  return <LegacyCheckoutFlow />;
}
```

### 5.4 A/B Testing

Firebase Remote Config integrates with Firebase A/B Testing. You create an experiment in the Firebase Console, and Remote Config delivers different values to different user groups.

**Setting up an A/B test:**

1. Go to Firebase Console > Remote Config
2. Click "Add parameter" or edit an existing one
3. Click "Add condition" > "A/B experiment"
4. Define your variants:
   - Control: `home_screen_layout = 'default'`
   - Variant A: `home_screen_layout = 'grid'`
   - Variant B: `home_screen_layout = 'list'`
5. Set your goal metric (e.g., `purchase` event)
6. Set the audience percentage (e.g., 20% of users in the experiment)
7. Publish

**On the client side, you do not need to do anything special.** The same code that reads `home_screen_layout` will get different values for different users. Firebase handles the randomization and tracking.

```typescript
function HomeScreen() {
  const { homeScreenLayout } = useFeatureFlags();

  // Each user in the experiment gets a consistent variant
  switch (homeScreenLayout) {
    case 'grid':
      return <HomeGridLayout />;
    case 'list':
      return <HomeListLayout />;
    default:
      return <HomeDefaultLayout />;
  }
}
```

### 5.5 Gradual Rollouts

Gradual rollouts use Firebase conditions to release a feature to an increasing percentage of users:

1. Create a condition: "10% of users" (using "User in random percentile")
2. Set the parameter to `true` for that condition
3. Monitor crash-free rate and performance
4. If stable, increase to 25%, then 50%, then 100%

```
Week 1: 10% of users -> monitor Crashlytics, monitor performance
Week 2: 25% of users -> still stable? Continue
Week 3: 50% of users -> check support tickets
Week 4: 100% of users -> full rollout
```

This is especially powerful for risky changes. If the 10% cohort shows a spike in crashes, you flip the flag to `false` and the feature is instantly disabled for everyone. No app update needed.

### 5.6 Real-Time Updates

By default, Remote Config caches values and re-fetches on a schedule. But you can listen for real-time updates:

```typescript
// src/services/remote-config.ts (continued)
class RemoteConfigService {
  // ... previous methods

  /**
   * Listen for real-time Remote Config updates.
   * Returns an unsubscribe function.
   */
  onConfigUpdated(callback: () => void): () => void {
    const unsubscribe = remoteConfig().onConfigUpdated(
      async () => {
        try {
          // Activate the new values
          await remoteConfig().activate();
          callback();
        } catch (error) {
          console.warn(
            'Failed to activate updated config:',
            error
          );
        }
      }
    );

    return unsubscribe;
  }
}
```

```typescript
// In your app root:
import { useEffect } from 'react';
import { remoteConfigService } from './src/services/remote-config';

function App() {
  useEffect(() => {
    const unsubscribe = remoteConfigService.onConfigUpdated(() => {
      // Config was updated -- re-read flags
      console.log('Remote Config updated');
    });

    return unsubscribe;
  }, []);

  // ...
}
```

**Real-time updates are particularly useful for:**
- Emergency kill switches (maintenance mode)
- Force update prompts
- Disabling a feature that is causing issues

### 5.7 The Maintenance Mode / Kill Switch Pattern

Every production app should have a kill switch:

```typescript
// src/components/MaintenanceGate.tsx
import { useEffect, useState } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { remoteConfigService } from '../services/remote-config';

export function MaintenanceGate({
  children,
}: {
  children: React.ReactNode;
}) {
  const [isMaintenanceMode, setIsMaintenanceMode] = useState(false);
  const [maintenanceMessage, setMaintenanceMessage] = useState('');

  useEffect(() => {
    // Check on mount
    checkMaintenance();

    // Listen for real-time updates
    const unsubscribe = remoteConfigService.onConfigUpdated(() => {
      checkMaintenance();
    });

    return unsubscribe;
  }, []);

  function checkMaintenance() {
    const maintenance = remoteConfigService.getBoolean(
      'maintenance_mode'
    );
    const message = remoteConfigService.getString(
      'maintenance_message'
    );
    setIsMaintenanceMode(maintenance);
    setMaintenanceMessage(message);
  }

  if (isMaintenanceMode) {
    return (
      <View style={styles.container}>
        <Text style={styles.title}>Under Maintenance</Text>
        <Text style={styles.message}>{maintenanceMessage}</Text>
      </View>
    );
  }

  return <>{children}</>;
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    marginBottom: 12,
  },
  message: {
    fontSize: 16,
    textAlign: 'center',
    color: '#666',
  },
});
```

```typescript
// App.tsx
export default function App() {
  return (
    <MaintenanceGate>
      <ForceUpdateGate>
        <AppContent />
      </ForceUpdateGate>
    </MaintenanceGate>
  );
}
```

---

## 6. CLOUD MESSAGING (FCM)

Push notifications are table stakes for a mobile app. Users expect them, and when done well, they drive engagement without being annoying. Firebase Cloud Messaging (FCM) is the standard for React Native push notifications.

### 6.1 Setup

```bash
npx expo install @react-native-firebase/messaging
```

**iOS requires additional setup:** You need an APNs (Apple Push Notification service) certificate or key uploaded to Firebase.

1. Go to [Apple Developer Portal](https://developer.apple.com)
2. Create an APNs Authentication Key (recommended over certificates)
3. Go to Firebase Console > Project Settings > Cloud Messaging
4. Upload the APNs key under "Apple app configuration"

**app.config.ts additions:**

```typescript
const config: ExpoConfig = {
  // ...
  ios: {
    // ...
    infoPlist: {
      UIBackgroundModes: ['remote-notification'],
    },
  },
  android: {
    // ...
  },
  plugins: [
    '@react-native-firebase/app',
    '@react-native-firebase/messaging',
  ],
};
```

### 6.2 Requesting Permission

On iOS, you must request permission before sending push notifications. On Android 13+, you also need runtime permission.

```typescript
// src/services/push-notifications.ts
import messaging from '@react-native-firebase/messaging';
import { Platform, PermissionsAndroid } from 'react-native';

class PushNotificationService {
  /**
   * Request push notification permission.
   * Returns the FCM token if granted, null if denied.
   */
  async requestPermission(): Promise<string | null> {
    // iOS: Request permission
    if (Platform.OS === 'ios') {
      const authStatus = await messaging().requestPermission();
      const enabled =
        authStatus === messaging.AuthorizationStatus.AUTHORIZED ||
        authStatus === messaging.AuthorizationStatus.PROVISIONAL;

      if (!enabled) {
        console.log('Push notification permission denied');
        return null;
      }
    }

    // Android 13+: Request POST_NOTIFICATIONS permission
    if (Platform.OS === 'android' && Platform.Version >= 33) {
      const result = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.POST_NOTIFICATIONS
      );
      if (result !== PermissionsAndroid.RESULTS.GRANTED) {
        console.log('Push notification permission denied');
        return null;
      }
    }

    // Get the FCM token
    const token = await messaging().getToken();
    console.log('FCM Token:', token);
    return token;
  }

  /**
   * Get the current FCM token (without requesting permission).
   */
  async getToken(): Promise<string | null> {
    try {
      return await messaging().getToken();
    } catch (error) {
      console.warn('Failed to get FCM token:', error);
      return null;
    }
  }

  /**
   * Listen for token refresh events.
   * The token can change -- always send the latest to your server.
   */
  onTokenRefresh(callback: (token: string) => void): () => void {
    return messaging().onTokenRefresh(callback);
  }
}

export const pushNotificationService = new PushNotificationService();
```

### 6.3 Sending the Token to Your Server

Your backend needs the FCM token to send targeted push notifications:

```typescript
// src/services/push-notifications.ts (continued)
class PushNotificationService {
  // ... previous methods

  /**
   * Register the FCM token with your backend.
   * Call after login and on token refresh.
   */
  async registerWithServer(userId: string): Promise<void> {
    const token = await this.getToken();
    if (!token) return;

    try {
      await api.post('/users/push-token', {
        userId,
        token,
        platform: Platform.OS,
      });
    } catch (error) {
      console.warn('Failed to register push token:', error);
    }

    // Listen for token refreshes
    this.onTokenRefresh(async (newToken) => {
      try {
        await api.post('/users/push-token', {
          userId,
          token: newToken,
          platform: Platform.OS,
        });
      } catch (error) {
        console.warn('Failed to update push token:', error);
      }
    });
  }

  /**
   * Unregister the push token. Call on logout.
   */
  async unregisterFromServer(userId: string): Promise<void> {
    const token = await this.getToken();
    if (!token) return;

    try {
      await api.delete('/users/push-token', {
        data: { userId, token },
      });
    } catch (error) {
      console.warn('Failed to unregister push token:', error);
    }
  }
}
```

### 6.4 Handling Notifications

There are three scenarios for receiving notifications:

1. **App is in the foreground** -- the notification arrives while the user is actively using the app
2. **App is in the background** -- the notification arrives while the app is backgrounded but still alive
3. **App is killed/quit** -- the notification arrives when the app is not running

Each requires different handling:

```typescript
// src/services/push-notifications.ts (continued)
import { FirebaseMessagingTypes } from '@react-native-firebase/messaging';

type NotificationHandler = (
  remoteMessage: FirebaseMessagingTypes.RemoteMessage
) => void;

class PushNotificationService {
  // ... previous methods

  /**
   * Set up all notification handlers.
   * Call once at app startup.
   */
  setupHandlers(options: {
    onForegroundMessage: NotificationHandler;
    onNotificationOpened: NotificationHandler;
    onInitialNotification: NotificationHandler;
  }): () => void {
    const unsubscribers: Array<() => void> = [];

    // 1. Foreground messages
    // When the app is in the foreground, notifications are NOT
    // shown by default. You must handle them manually.
    const unsubForeground = messaging().onMessage(
      async (remoteMessage) => {
        console.log('Foreground message:', remoteMessage);
        options.onForegroundMessage(remoteMessage);
      }
    );
    unsubscribers.push(unsubForeground);

    // 2. Background notification opened
    // When the user taps a notification while the app is backgrounded
    const unsubBackground = messaging().onNotificationOpenedApp(
      (remoteMessage) => {
        console.log(
          'Notification opened (background):',
          remoteMessage
        );
        options.onNotificationOpened(remoteMessage);
      }
    );
    unsubscribers.push(unsubBackground);

    // 3. App was killed, user opened via notification
    // Check if the app was opened from a notification
    messaging()
      .getInitialNotification()
      .then((remoteMessage) => {
        if (remoteMessage) {
          console.log('Initial notification:', remoteMessage);
          options.onInitialNotification(remoteMessage);
        }
      });

    return () => {
      unsubscribers.forEach((unsub) => unsub());
    };
  }
}
```

**Background message handler (must be registered outside of React):**

```typescript
// index.js (or your app entry point)
import messaging from '@react-native-firebase/messaging';

// This runs when the app is in the background or killed
// and receives a data-only message (not a notification message)
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  console.log('Background message:', remoteMessage);

  // Do something with the message:
  // - Update a local badge count
  // - Sync data in the background
  // - Schedule a local notification

  // IMPORTANT: Do not do heavy work here.
  // On Android, you have about 10 seconds before the system kills
  // the handler.
  // On iOS, you have about 30 seconds.
});
```

### 6.5 Notification Channels (Android)

Android 8.0+ requires notification channels. Each channel has its own importance level, sound, vibration pattern, and lock screen visibility. Users can individually control each channel in their device settings.

```typescript
// src/services/push-notifications.ts (continued)
import notifee, { AndroidImportance } from '@notifee/react-native';

class PushNotificationService {
  // ... previous methods

  /**
   * Create notification channels for Android.
   * Call once at app startup (before showing any notifications).
   */
  async setupAndroidChannels(): Promise<void> {
    if (Platform.OS !== 'android') return;

    // High-priority channel for urgent messages
    await notifee.createChannel({
      id: 'urgent',
      name: 'Urgent Notifications',
      description:
        'Time-sensitive notifications that require immediate attention',
      importance: AndroidImportance.HIGH,
      sound: 'default',
      vibration: true,
    });

    // Default channel for general notifications
    await notifee.createChannel({
      id: 'default',
      name: 'General Notifications',
      description: 'General app notifications',
      importance: AndroidImportance.DEFAULT,
      sound: 'default',
    });

    // Low-priority channel for updates and promotions
    await notifee.createChannel({
      id: 'updates',
      name: 'Updates & Tips',
      description:
        'Product updates, tips, and promotional offers',
      importance: AndroidImportance.LOW,
    });

    // Silent channel for background sync notifications
    await notifee.createChannel({
      id: 'silent',
      name: 'Background Updates',
      description:
        'Silent notifications for data synchronization',
      importance: AndroidImportance.MIN,
    });
  }
}
```

### 6.6 Displaying Foreground Notifications

When the app is in the foreground, FCM delivers the message but does NOT show a notification banner. You need to display it yourself using a library like `@notifee/react-native`:

```bash
npx expo install @notifee/react-native
```

```typescript
// src/services/push-notifications.ts (continued)
import notifee from '@notifee/react-native';

class PushNotificationService {
  // ... previous methods

  /**
   * Display a local notification (for foreground messages).
   */
  async displayLocalNotification(
    remoteMessage: FirebaseMessagingTypes.RemoteMessage
  ): Promise<void> {
    const { notification, data } = remoteMessage;

    if (!notification) return;

    // Determine the channel based on the message data
    const channelId =
      (data?.channel as string) ?? 'default';

    await notifee.displayNotification({
      title: notification.title ?? 'New Notification',
      body: notification.body ?? '',
      data: data as Record<string, string>,
      android: {
        channelId,
        smallIcon: 'ic_notification',
        pressAction: {
          id: 'default',
        },
      },
      ios: {
        sound: 'default',
      },
    });
  }
}
```

### 6.7 Topic-Based Messaging

Instead of sending to individual tokens, you can subscribe users to topics and send messages to all subscribers at once:

```typescript
// src/services/push-notifications.ts (continued)
class PushNotificationService {
  // ... previous methods

  /**
   * Subscribe to a notification topic.
   */
  async subscribeToTopic(topic: string): Promise<void> {
    try {
      await messaging().subscribeToTopic(topic);
      console.log(`Subscribed to topic: ${topic}`);
    } catch (error) {
      console.warn(
        `Failed to subscribe to topic "${topic}":`,
        error
      );
    }
  }

  /**
   * Unsubscribe from a notification topic.
   */
  async unsubscribeFromTopic(topic: string): Promise<void> {
    try {
      await messaging().unsubscribeFromTopic(topic);
      console.log(`Unsubscribed from topic: ${topic}`);
    } catch (error) {
      console.warn(
        `Failed to unsubscribe from topic "${topic}":`,
        error
      );
    }
  }

  /**
   * Set up topic subscriptions based on user preferences.
   */
  async syncTopicSubscriptions(preferences: {
    marketingNotifications: boolean;
    productUpdates: boolean;
    orderUpdates: boolean;
    weeklyDigest: boolean;
  }): Promise<void> {
    const topicMap: Record<string, boolean> = {
      marketing: preferences.marketingNotifications,
      'product-updates': preferences.productUpdates,
      'order-updates': preferences.orderUpdates,
      'weekly-digest': preferences.weeklyDigest,
    };

    for (const [topic, subscribed] of Object.entries(topicMap)) {
      if (subscribed) {
        await this.subscribeToTopic(topic);
      } else {
        await this.unsubscribeFromTopic(topic);
      }
    }
  }
}
```

### 6.8 Deep Linking from Notifications

Push notifications should take users somewhere specific, not just open the app:

```typescript
// src/services/push-notifications.ts
import { Linking } from 'react-native';

class PushNotificationService {
  // ... previous methods

  /**
   * Handle navigation from a notification.
   * Extracts the deep link from the notification data
   * and navigates to the appropriate screen.
   */
  handleNotificationNavigation(
    remoteMessage: FirebaseMessagingTypes.RemoteMessage,
    navigationRef: React.RefObject<any>
  ): void {
    const { data } = remoteMessage;

    if (!data) return;

    // Option 1: Deep link URL in the notification data
    if (data.deepLink && typeof data.deepLink === 'string') {
      Linking.openURL(data.deepLink);
      return;
    }

    // Option 2: Screen name and params in the notification data
    if (data.screen && typeof data.screen === 'string') {
      const params = data.params
        ? JSON.parse(data.params as string)
        : undefined;

      navigationRef.current?.navigate(data.screen, params);
    }
  }
}
```

### 6.9 The Complete Push Notification Setup

Here is how it all comes together at the app level:

```typescript
// App.tsx
import { useEffect, useRef } from 'react';
import {
  NavigationContainer,
  NavigationContainerRef,
} from '@react-navigation/native';
import { pushNotificationService } from './src/services/push-notifications';

export default function App() {
  const navigationRef =
    useRef<NavigationContainerRef<any>>(null);

  useEffect(() => {
    // Initialize push notifications
    async function init() {
      // Set up Android notification channels
      await pushNotificationService.setupAndroidChannels();

      // Request permission and get token
      const token =
        await pushNotificationService.requestPermission();
      if (token) {
        // Register token with your backend
        // (after user is authenticated)
      }
    }
    init();

    // Set up notification handlers
    const unsubscribe = pushNotificationService.setupHandlers({
      onForegroundMessage: (message) => {
        // Show a local notification banner
        pushNotificationService.displayLocalNotification(message);
      },
      onNotificationOpened: (message) => {
        // User tapped notification while app was backgrounded
        pushNotificationService.handleNotificationNavigation(
          message,
          navigationRef
        );
      },
      onInitialNotification: (message) => {
        // App was opened from a killed state via notification
        // Navigation might not be ready yet -- defer slightly
        setTimeout(() => {
          pushNotificationService.handleNotificationNavigation(
            message,
            navigationRef
          );
        }, 1000);
      },
    });

    return unsubscribe;
  }, []);

  return (
    <NavigationContainer ref={navigationRef}>
      {/* Your app content */}
    </NavigationContainer>
  );
}
```

---

## 7. APP DISTRIBUTION

Firebase App Distribution is Google's answer to TestFlight for Android (and an alternative for iOS too). It lets you distribute pre-release builds to testers without going through the Play Store or App Store review.

### 7.1 Why Firebase App Distribution?

```
+-----------------------------+------------------+------------------+
| Feature                     | TestFlight       | Firebase App     |
|                             | (iOS only)       | Distribution     |
+-----------------------------+------------------+------------------+
| iOS distribution            | Yes              | Yes (Ad Hoc)     |
| Android distribution        | No               | Yes              |
| Tester management           | Groups           | Groups + emails  |
| Build notes                 | Yes              | Yes              |
| Tester feedback             | Via TestFlight   | In-app SDK       |
| UDID collection (iOS)       | Automatic        | Manual/Tester app|
| Crash reporting             | Separate         | Integrated       |
| CI/CD integration           | Xcode Cloud      | Firebase CLI     |
| Expiration                  | 90 days          | No expiration    |
| Max testers                 | 10,000           | Unlimited        |
+-----------------------------+------------------+------------------+
```

For most React Native teams, **use TestFlight for iOS** (better experience, automatic UDID management) and **Firebase App Distribution for Android** (since there is no TestFlight for Android). Use Firebase App Distribution for iOS only if you need cross-platform consistency or if the TestFlight review process is too slow for your iteration speed.

### 7.2 Setting Up with the Firebase CLI

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# Distribute an APK or AAB
firebase appdistribution:distribute ./build/app-release.apk \
  --app YOUR_FIREBASE_APP_ID \
  --groups "internal-testers,qa-team" \
  --release-notes "Build 42: Fixed checkout bug, added dark mode"

# Distribute an iOS IPA
firebase appdistribution:distribute ./build/app.ipa \
  --app YOUR_IOS_FIREBASE_APP_ID \
  --groups "ios-testers" \
  --release-notes "Build 42: Fixed checkout bug, added dark mode"
```

### 7.3 Tester Groups

Organize testers into groups in the Firebase Console:

- **internal** -- your engineering team, gets every build
- **qa** -- QA team, gets release candidates
- **beta** -- external beta testers, gets stable pre-release builds
- **stakeholders** -- product managers, executives, gets milestone builds

```bash
# Add testers to a group
firebase appdistribution:testers:add \
  --project YOUR_PROJECT_ID \
  --emails "tester@example.com,another@example.com"

# List testers
firebase appdistribution:testers:list \
  --project YOUR_PROJECT_ID
```

### 7.4 Integration with EAS Build

The most powerful setup combines EAS Build with Firebase App Distribution. Build with EAS, distribute with Firebase:

```json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "channel": "development"
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      },
      "channel": "preview",
      "env": {
        "APP_ENV": "staging"
      }
    },
    "production": {
      "channel": "production",
      "env": {
        "APP_ENV": "production"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@apple.id",
        "ascAppId": "YOUR_APP_STORE_CONNECT_APP_ID",
        "appleTeamId": "YOUR_TEAM_ID"
      },
      "android": {
        "serviceAccountKeyPath": "./play-store-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

**Automated distribution with EAS Build webhooks:**

Set up an EAS Build webhook that triggers Firebase App Distribution when a build completes. Here is the server-side handler:

```typescript
// server/api/eas-build-webhook.ts
// This runs on your server (or a serverless function)

import { execFile } from 'node:child_process';
import { promisify } from 'node:util';

const execFileAsync = promisify(execFile);

interface EASBuildWebhook {
  id: string;
  platform: 'ios' | 'android';
  status: 'finished' | 'errored' | 'canceled';
  artifacts: {
    buildUrl: string;
  };
  metadata: {
    appVersion: string;
    buildNumber: string;
    channel: string;
  };
}

export async function handleEASBuildWebhook(
  body: EASBuildWebhook
) {
  if (body.status !== 'finished') return;
  if (body.metadata.channel !== 'preview') return;

  // Download the build artifact
  const buildPath = await downloadBuild(
    body.artifacts.buildUrl
  );

  // Distribute via Firebase
  const firebaseAppId =
    body.platform === 'android'
      ? process.env.FIREBASE_ANDROID_APP_ID!
      : process.env.FIREBASE_IOS_APP_ID!;

  const releaseNotes = `v${body.metadata.appVersion} (${body.metadata.buildNumber})`;

  await execFileAsync('firebase', [
    'appdistribution:distribute',
    buildPath,
    '--app',
    firebaseAppId,
    '--groups',
    'internal-testers,qa-team',
    '--release-notes',
    releaseNotes,
  ]);

  // Notify the team
  await sendSlackNotification({
    channel: '#mobile-builds',
    text: `New ${body.platform} build distributed: ${releaseNotes}`,
  });
}
```

### 7.5 In-App Feedback

Firebase App Distribution includes an in-app feedback SDK that lets testers take screenshots and submit feedback without leaving the app:

```typescript
// src/utils/beta-feedback.ts
import { Platform } from 'react-native';

/**
 * For beta/preview builds only:
 * Enable the Firebase App Distribution tester feedback UI.
 *
 * This shows a floating button that testers can tap to
 * take a screenshot and submit feedback.
 */
export async function enableBetaFeedback(): Promise<void> {
  // Only enable for preview/beta builds
  // Check an EAS environment variable or build channel
  const isBeta = process.env.EXPO_PUBLIC_APP_ENV === 'staging';

  if (!isBeta) return;

  try {
    const appDistribution =
      require('@react-native-firebase/app-distribution').default;

    // Check if updates are available
    const update = await appDistribution().checkForUpdate();
    if (update.isAvailable) {
      // Prompt tester to update
      console.log('New build available:', update.version);
    }
  } catch (error) {
    // Module not installed in production builds -- expected
    console.warn('App Distribution not available:', error);
  }
}
```

### 7.6 A Complete Distribution Workflow

Here is the workflow that works for most teams:

```
Developer pushes to `develop` branch
    |
    v
CI triggers EAS Build (preview profile)
    |
    v
EAS Build completes
    |
    v
Webhook triggers Firebase App Distribution
    |
    v
Internal testers automatically get the build
    |
    v
Testers install, test, submit feedback
    |
    v
When ready, PR merged to `main`
    |
    v
CI triggers EAS Build (production profile)
    |
    v
EAS Submit -> App Store / Play Store
```

---

## 8. PUTTING IT ALL TOGETHER

Here is the initialization order for Firebase services in a production app:

```typescript
// src/services/firebase-init.ts
import { remoteConfigService } from './remote-config';
import { crashlyticsService } from './crashlytics';
import { analyticsService } from './analytics';
import { pushNotificationService } from './push-notifications';

/**
 * Initialize all Firebase services.
 * Call once at app startup, before rendering the app.
 */
export async function initializeFirebase(): Promise<void> {
  try {
    // 1. Crashlytics first -- we want to catch crashes ASAP
    await crashlyticsService.initialize();

    // 2. Remote Config -- feature flags may affect the rest
    //    of the app
    await remoteConfigService.initialize();

    // 3. Analytics -- start tracking after config is loaded
    //    (consent may be controlled by Remote Config)
    await analyticsService.logCustomEvent('app_initialized');

    // 4. Push notifications -- set up channels and handlers
    await pushNotificationService.setupAndroidChannels();

    crashlyticsService.log('Firebase services initialized');
  } catch (error) {
    // Firebase initialization should never crash the app
    console.error('Firebase initialization error:', error);
    if (error instanceof Error) {
      crashlyticsService.recordError(error, {
        category: 'firebase_init',
      });
    }
  }
}
```

```typescript
// App.tsx
import { useEffect, useState } from 'react';
import { initializeFirebase } from './src/services/firebase-init';

export default function App() {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    initializeFirebase()
      .then(() => setReady(true))
      .catch(() => setReady(true)); // Do not block app on Firebase failure

    return () => {
      // Cleanup if needed
    };
  }, []);

  if (!ready) {
    return <SplashScreen />;
  }

  return <AppContent />;
}
```

### The Decision Matrix: When to Use What

```
+------------------------------+----------------------------------------+
| I want to...                 | Use this Firebase service              |
+------------------------------+----------------------------------------+
| Know when the app crashes    | Crashlytics                            |
| Know what users do           | Analytics                              |
| Know if the app is slow      | Performance Monitoring                 |
| Change behavior without      | Remote Config                          |
|   shipping an update         |                                        |
| Send push notifications      | Cloud Messaging (FCM) + Notifee       |
| Distribute beta builds       | App Distribution                       |
| Run A/B tests                | Remote Config + A/B Testing            |
| Disable a feature instantly  | Remote Config (kill switch)            |
| Track purchase funnels       | Analytics (recommended events)         |
| Measure real-world perf      | Performance Monitoring (custom traces) |
| Debug a crash from last week | Crashlytics (breadcrumbs + keys)       |
+------------------------------+----------------------------------------+
```

---

## 9. COMMON PITFALLS AND HOW TO AVOID THEM

### 9.1 "I installed Crashlytics but I do not see any crashes"

**Causes:**
- You are testing in debug mode. Crashlytics is disabled in `__DEV__` (as it should be).
- dSYMs are not being uploaded. Check Firebase Console > Crashlytics > Settings > Missing dSYMs.
- You are looking too soon. Crashlytics batches reports and sends them on the next app launch. Wait.

### 9.2 "My analytics events are not showing up"

**Causes:**
- Analytics has a processing delay. Events can take up to 24 hours to appear in the Firebase Console.
- For real-time debugging, use Firebase DebugView: Analytics > DebugView. Enable it by adding a launch argument:
  - iOS: `-FIRDebugEnabled` in Xcode scheme arguments
  - Android: `adb shell setprop debug.firebase.analytics.app com.mycompany.myapp`

### 9.3 "Remote Config is not returning my updated values"

**Causes:**
- The minimum fetch interval has not elapsed. In production, this is typically 1 hour.
- You called `fetch()` but not `activate()`. Use `fetchAndActivate()` for both in one call.
- The condition is not matching. Check your Firebase Console conditions carefully.

### 9.4 "Push notifications work on Android but not iOS"

**Causes:**
- Missing or expired APNs key/certificate in Firebase Console.
- Missing `UIBackgroundModes: ['remote-notification']` in your Info.plist.
- You are testing on the iOS Simulator. Push notifications require a physical device.
- Permission was denied. Check `messaging().requestPermission()` return value.

### 9.5 "Firebase is making my app start slowly"

**Solutions:**
- Only install the Firebase modules you actually use.
- Use `useFrameworks: 'static'` on iOS (dynamic frameworks are slower to load).
- Do not call `fetchAndActivate()` synchronously during startup if Remote Config values are not critical for the initial render. Fetch in the background and apply on the next launch.

---

## 10. FIREBASE COSTS: THE REALISTIC PICTURE

Firebase has generous free tiers, but some services have costs that can surprise you:

```
+------------------------------+--------------------------+------------+
| Service                      | Free Tier                | Cost After |
+------------------------------+--------------------------+------------+
| Crashlytics                  | Unlimited                | Free       |
| Analytics                    | Unlimited                | Free       |
| Performance Monitoring       | Unlimited                | Free       |
| Remote Config                | Unlimited                | Free       |
| Cloud Messaging (FCM)        | Unlimited                | Free       |
| App Distribution             | Unlimited                | Free       |
| A/B Testing                  | Unlimited                | Free       |
+------------------------------+--------------------------+------------+
| Cloud Firestore              | 1 GiB storage,           | Pay per    |
|                              | 50K reads/day            | operation  |
| Authentication               | 10K/month (phone auth)   | Per SMS    |
| Cloud Functions              | 2M invocations/month     | Per call   |
| Cloud Storage                | 5 GB                     | Per GB     |
+------------------------------+--------------------------+------------+
```

Notice something? Every service we covered in this chapter is free. The costs come from backend services (Firestore, Auth, Functions, Storage) which are outside our scope. For crash reporting, analytics, performance monitoring, feature flags, push notifications, and beta distribution, **Firebase is genuinely free at any scale.**

That is a remarkable value proposition. Use it.

---

## CHAPTER SUMMARY

Firebase is not just a backend-as-a-service. For React Native teams, it is an operational intelligence platform. The services we covered -- Crashlytics, Analytics, Performance Monitoring, Remote Config, Cloud Messaging, and App Distribution -- form the operational backbone of a production mobile app.

The teams that get the most value from Firebase are the ones that:

1. **Set up Crashlytics properly** -- with breadcrumbs, custom keys, user context, and non-fatal error logging. When a crash happens, you have the full story.
2. **Track meaningful analytics events** -- not just screen views, but user actions that map to business outcomes. Funnels, conversions, retention.
3. **Instrument performance traces** -- screen load times, API response times, custom traces for critical flows. You cannot improve what you do not measure.
4. **Use Remote Config for feature flags** -- gradual rollouts, kill switches, A/B tests. Ship confidently because you can always roll back without an app update.
5. **Handle push notifications completely** -- foreground, background, and killed states. Proper channels. Deep linking.
6. **Automate beta distribution** -- build, distribute, test, feedback. A tight loop that keeps the team shipping.

None of this is rocket science. It is just discipline -- setting up each service properly and building the habits to actually look at the data.

### What's Next

In the next chapter, we will cover **Security & Data Protection** -- how to protect API keys, secure user data, implement certificate pinning, and handle the OWASP Mobile Top 10 in a React Native context. Some of what we covered here (like not logging sensitive data in Crashlytics breadcrumbs) connects directly to those security concerns.
