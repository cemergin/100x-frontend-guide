<!--
  CHAPTER: 22
  TITLE: Mobile Monitoring & Observability
  PART: V — Deployment & Operations
  PREREQS: Chapters 1, 5
  KEY_TOPICS: Sentry, Crashlytics, crash-free rate, ANR, source maps, custom traces, RUM, dashboards, alerts, performance monitoring, error boundaries
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 20: Mobile Monitoring & Observability

> "If you can't measure it, you can't improve it." — Peter Drucker
>
> "If you didn't set up monitoring, you didn't ship it." — Every senior engineer who's been woken up at 3 AM

---

<details>
<summary><strong>TL;DR</strong></summary>

- Set up Sentry (or Crashlytics) before your first deploy, not after your first incident; monitoring is a pre-launch requirement, not a post-launch activity
- Track crash-free rate (target 99.5%+), ANR rate, and custom performance traces; set alerts on regressions so you find problems before users report them
- Upload source maps and dSYMs to your error tracker in CI so crash reports show real file names and line numbers, not minified gibberish
- Use React error boundaries to catch JS-layer crashes gracefully, log them with context (breadcrumbs, user ID, app version), and show a recovery UI instead of a white screen
- Monitor every release for at least 48 hours after deploy; the person who wrote the code is the best person to triage its crashes

</details>

## The Story That Changes How You Think About Monitoring

Let me tell you about a Tuesday morning at a company I worked with. The team had just shipped a big update to their React Native app — new onboarding flow, fresh animations, the works. QA had signed off. The PM was already writing the blog post. Everyone felt great.

By Wednesday afternoon, their crash-free rate had dropped from 99.7% to 97.2%. That's not a small number. That means roughly 3 out of every 100 user sessions were crashing. The app store rating started tanking. One-star reviews piled up: "App keeps crashing," "Can't even open the app anymore," "Uninstalled."

The worst part? Nobody on the team noticed until a customer success manager pinged the engineering channel on Slack. There were no alerts. No dashboards. No one was watching.

It took them four days to identify the root cause — a null pointer exception in the new onboarding flow that only triggered on Android devices running API level 28 with a specific locale setting. Four days. Thousands of affected users. A measurable impact on revenue.

This chapter exists so that never happens to you.

---

## Part 1: Why Monitoring Is Your Job

### The Ownership Mindset

Let me be blunt: if you wrote the code, you own the monitoring. Not QA. Not DevOps. Not some mythical "platform team" that's supposed to handle it. You.

This isn't about blame. It's about effectiveness. The person who wrote the code is the person best positioned to:

1. **Know what to monitor** — You know the risky parts. You know where you made trade-offs. You know that one function where you thought "this probably won't fail, but..."
2. **Understand the crash reports** — Stack traces mean something to you. You can read them and immediately know whether it's your code, a dependency, or a platform issue.
3. **Fix it fast** — You don't need a handoff meeting. You don't need to "ramp up on the codebase." You're already there.

At Shopify, every engineer who ships code to Shop (their React Native app used by millions) is expected to monitor their changes for at least 48 hours after release. They don't wait for someone else to tell them something broke. They're watching.

At Airbnb, before they migrated away from React Native, their mobile teams had a rotation called "crash duty." One engineer per team, per week, whose primary job was to triage incoming crash reports. Not a separate QA team. An engineer from the team that owned the code.

### Set Up Monitoring Before Your First Deploy

Here's a rule I want you to internalize: **monitoring is not a post-launch activity.** It's a pre-launch requirement.

Think about it this way. Would you ship a car without a dashboard? No speedometer, no fuel gauge, no check engine light? Of course not. But that's exactly what you're doing when you deploy a mobile app without monitoring.

Your very first pull request for any new mobile project should include:

```typescript
// This should be in your first PR. Not your tenth. Not "when we get to it."
// Your FIRST PR.

import * as Sentry from '@sentry/react-native';
import crashlytics from '@react-native-firebase/crashlytics';

// Initialize monitoring before anything else in your app
Sentry.init({
  dsn: 'https://your-dsn@sentry.io/your-project-id',
  tracesSampleRate: 1.0, // 100% in dev, tune down in production
  environment: __DEV__ ? 'development' : 'production',
});

// Crashlytics initializes automatically with Firebase,
// but you should explicitly enable/disable collection
crashlytics().setCrashlyticsCollectionEnabled(!__DEV__);
```

I'm serious about this. If you're reading this chapter after you've already shipped without monitoring, stop reading and go set it up. Right now. I'll wait.

### The Cost of Flying Blind

Let me quantify what "no monitoring" actually costs:

| Scenario | With Monitoring | Without Monitoring |
|----------|----------------|-------------------|
| Time to detect a crash spike | 5 minutes (alert fires) | 2-4 days (user complaints) |
| Time to identify root cause | 30 minutes (stack trace + context) | 4-8 hours (reproducing from scratch) |
| Time to ship a fix | 2-4 hours (hotfix + deploy) | 1-2 days (after you finally understand the problem) |
| Users affected | Hundreds | Tens of thousands |
| Revenue impact | Minimal | Significant |
| App store rating impact | Negligible | Measurable drop |

The math is simple. A $50/month Sentry bill or free Crashlytics setup saves you days of engineering time and potentially millions in revenue. There is no rational argument against monitoring. None.

### The Three Pillars of Mobile Observability

Before we dive into specific tools, let's establish the framework. Mobile observability rests on three pillars:

1. **Error Tracking** — What's crashing? What's throwing exceptions? What errors are users hitting?
2. **Performance Monitoring** — How fast are screens loading? How responsive is the app? Are API calls slow?
3. **User Experience Monitoring** — What's the real user experience? Are sessions healthy? Where do users drop off?

Most teams get pillar #1 partially right (they set up Crashlytics and forget about it). Few teams do pillar #2 well. Almost nobody does pillar #3 in the mobile world.

By the end of this chapter, you'll have all three.

---

## Part 2: Sentry vs Crashlytics — The Deep Comparison

### The Lay of the Land

If you've been in the React Native ecosystem for more than a week, you've heard two names: **Sentry** and **Firebase Crashlytics**. They're both error tracking tools, but they're very different in philosophy, capability, and cost.

Here's the uncomfortable truth that most "Sentry vs Crashlytics" blog posts won't tell you: **many production apps use both.** And there are good reasons for that.

Let me break down exactly when you'd use each, and when you'd use both.

### Firebase Crashlytics: The Deep Dive

Crashlytics started as a standalone company, got acquired by Twitter (as part of Fabric), then got acquired by Google and folded into Firebase. Despite this tumultuous history, it remains one of the best crash reporting tools for mobile — period.

#### What Crashlytics Does Exceptionally Well

**1. Native Crash Reporting**

Crashlytics' native crash reporting is best-in-class. When your app hits a segfault, a null pointer dereference in native code, or an unhandled Objective-C/Kotlin exception, Crashlytics captures it with remarkable fidelity. It understands the native stack, it groups crashes intelligently, and it provides context that's actually useful.

```typescript
// Setting up Crashlytics in React Native
// First, install the package
// npm install @react-native-firebase/app @react-native-firebase/crashlytics

import crashlytics from '@react-native-firebase/crashlytics';

// In your app initialization
async function initCrashlytics() {
  // Enable collection (disable in dev)
  await crashlytics().setCrashlyticsCollectionEnabled(!__DEV__);

  // Set user identifier for tracking
  await crashlytics().setUserId('user-123');

  // Set custom attributes for filtering
  await crashlytics().setAttribute('subscription_tier', 'premium');
  await crashlytics().setAttribute('app_variant', 'experiment_b');

  // Set multiple attributes at once
  await crashlytics().setAttributes({
    role: 'admin',
    onboarding_complete: 'true',
    build_channel: 'internal',
  });
}
```

**2. Crash-Free Rate Tracking**

Crashlytics invented the concept of "crash-free users" and "crash-free sessions" as a metric. The Firebase console gives you this front and center, and it's become the industry standard metric for mobile app health.

**3. Integration with Firebase Ecosystem**

If you're using Firebase for anything — analytics, remote config, A/B testing, push notifications — Crashlytics integrates seamlessly. You can see crashes correlated with Firebase Analytics events, which is incredibly powerful for debugging.

**4. It's Free**

This matters. Crashlytics has no usage limits, no session caps, no hidden costs. For a startup or a team that's cost-conscious, this is a major advantage.

#### Setting Up Crashlytics Properly

Most tutorials show you the basic setup and call it a day. Here's what a production setup actually looks like:

```typescript
// src/monitoring/crashlytics.ts

import crashlytics from '@react-native-firebase/crashlytics';
import { Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';

interface CrashlyticsConfig {
  enabled: boolean;
  userId?: string;
  customAttributes?: Record<string, string>;
}

class CrashlyticsService {
  private initialized = false;

  async init(config: CrashlyticsConfig): Promise<void> {
    if (this.initialized) return;

    await crashlytics().setCrashlyticsCollectionEnabled(config.enabled);

    if (config.userId) {
      await crashlytics().setUserId(config.userId);
    }

    // Always set device context
    await crashlytics().setAttributes({
      platform: Platform.OS,
      os_version: Platform.Version.toString(),
      app_version: DeviceInfo.getVersion(),
      build_number: DeviceInfo.getBuildNumber(),
      device_model: await DeviceInfo.getModel(),
      is_emulator: (await DeviceInfo.isEmulator()).toString(),
      available_memory: (await DeviceInfo.getTotalMemory()).toString(),
      ...config.customAttributes,
    });

    this.initialized = true;
  }

  // Log non-fatal errors — this is where most of your value comes from
  recordError(error: Error, context?: Record<string, string>): void {
    if (!this.initialized) return;

    if (context) {
      // Set context before recording
      Object.entries(context).forEach(([key, value]) => {
        crashlytics().setAttribute(key, value);
      });
    }

    crashlytics().recordError(error);
  }

  // Breadcrumbs — leave a trail so you can understand what happened before the crash
  log(message: string): void {
    if (!this.initialized) return;
    crashlytics().log(message);
  }

  // Track screen navigation
  logScreenView(screenName: string): void {
    this.log(`Screen: ${screenName}`);
  }

  // Track user actions
  logAction(action: string, params?: Record<string, string>): void {
    const paramString = params
      ? ` | ${Object.entries(params).map(([k, v]) => `${k}=${v}`).join(', ')}`
      : '';
    this.log(`Action: ${action}${paramString}`);
  }

  // Force a test crash — use this to verify your setup
  testCrash(): void {
    crashlytics().crash();
  }

  // Update user ID when they log in
  async setUser(userId: string, attributes?: Record<string, string>): Promise<void> {
    await crashlytics().setUserId(userId);
    if (attributes) {
      await crashlytics().setAttributes(attributes);
    }
  }

  // Clear user on logout
  async clearUser(): Promise<void> {
    await crashlytics().setUserId('');
  }
}

export const crashlyticsService = new CrashlyticsService();
```

**The key insight most people miss:** The `recordError` method for non-fatal errors is where 80% of the value comes from. Fatal crashes are dramatic, but non-fatal errors — API failures, JSON parsing errors, unexpected null values — these are the silent killers of user experience.

#### Crashlytics Breadcrumbs Pattern

Breadcrumbs are chronological log entries that show what happened before a crash. They're your forensic trail. Here's how to use them effectively:

```typescript
// src/navigation/NavigationTracker.tsx

import React, { useEffect } from 'react';
import { useNavigationState, useRoute } from '@react-navigation/native';
import { crashlyticsService } from '../monitoring/crashlytics';

export function useNavigationTracking() {
  const route = useRoute();

  useEffect(() => {
    crashlyticsService.logScreenView(route.name);
  }, [route.name]);
}

// In your API layer
export async function apiCall<T>(
  endpoint: string,
  options: RequestInit
): Promise<T> {
  crashlyticsService.log(`API Request: ${options.method} ${endpoint}`);

  try {
    const response = await fetch(endpoint, options);

    crashlyticsService.log(
      `API Response: ${options.method} ${endpoint} — ${response.status}`
    );

    if (!response.ok) {
      const error = new Error(`API Error: ${response.status} ${response.statusText}`);
      crashlyticsService.recordError(error, {
        endpoint,
        method: options.method || 'GET',
        status: response.status.toString(),
      });
      throw error;
    }

    return response.json();
  } catch (error) {
    crashlyticsService.log(`API Failed: ${options.method} ${endpoint} — ${error}`);
    throw error;
  }
}

// In user action handlers
function handlePurchase(productId: string) {
  crashlyticsService.logAction('purchase_initiated', { productId });

  try {
    // ... purchase logic
    crashlyticsService.logAction('purchase_completed', { productId });
  } catch (error) {
    crashlyticsService.logAction('purchase_failed', { productId });
    crashlyticsService.recordError(error as Error, {
      action: 'purchase',
      product_id: productId,
    });
    throw error;
  }
}
```

### Sentry: The Deep Dive

Sentry is a different beast. Where Crashlytics is focused primarily on crash reporting, Sentry is a full observability platform. It does error tracking, performance monitoring, session replay, and more.

#### What Sentry Does Exceptionally Well

**1. Performance Monitoring**

This is Sentry's killer feature for mobile. You get distributed tracing, custom spans, automatic instrumentation of HTTP requests and navigation, and detailed performance dashboards. Crashlytics has Firebase Performance Monitoring as a separate product, but Sentry bakes it all into one SDK.

**2. Session Replay (Mobile)**

Sentry's mobile session replay lets you literally watch what the user was doing when an error occurred. It's like having a screen recording of every session, but built with privacy in mind (it redacts sensitive content by default). This is a game-changer for debugging.

**3. Issue Grouping and Triage**

Sentry's issue grouping is sophisticated. It uses fingerprinting algorithms to group similar errors together, so you're not drowning in 10,000 individual instances of the same bug. You see one issue with 10,000 events. This makes triage manageable.

**4. Source Maps and Stack Traces**

Sentry's source map support for JavaScript is excellent. When properly configured, you get fully symbolicated stack traces with the exact line of your original TypeScript code — not minified garbage.

**5. Release Tracking**

Sentry lets you associate errors with specific releases, see regression data, and track which commits introduced which bugs. This is invaluable for large teams.

#### Setting Up Sentry Properly

Here's a production-grade Sentry setup for React Native:

```typescript
// src/monitoring/sentry.ts

import * as Sentry from '@sentry/react-native';
import { Platform } from 'react-native';
import DeviceInfo from 'react-native-device-info';

interface SentryConfig {
  dsn: string;
  environment: 'development' | 'staging' | 'production';
  release: string;
  dist: string;
  tracesSampleRate: number;
  profilesSampleRate: number;
  replaysSessionSampleRate: number;
  replaysOnErrorSampleRate: number;
}

export function initSentry(config: SentryConfig): void {
  Sentry.init({
    dsn: config.dsn,
    environment: config.environment,
    release: config.release,
    dist: config.dist,

    // Performance Monitoring
    tracesSampleRate: config.tracesSampleRate,
    profilesSampleRate: config.profilesSampleRate,

    // Session Replay
    replaysSessionSampleRate: config.replaysSessionSampleRate,
    replaysOnErrorSampleRate: config.replaysOnErrorSampleRate,

    // Integrations
    integrations: [
      Sentry.reactNativeTracingIntegration({
        // Automatic screen tracking
        routingInstrumentation:
          Sentry.reactNavigationIntegration(),
        // Track all XMLHttpRequest and fetch requests
        enableHTTPTimings: true,
      }),
      Sentry.mobileReplayIntegration({
        // Mask all text by default for privacy
        maskAllText: true,
        // Mask all images by default
        maskAllImages: false,
        // Mask specific views
        maskAllVectors: false,
      }),
      Sentry.httpClientIntegration({
        // Capture request/response bodies for failed requests
        failedRequestStatusCodes: [[400, 599]],
      }),
    ],

    // Before sending an event, you can modify or drop it
    beforeSend(event, hint) {
      // Don't send events in development
      if (__DEV__) {
        console.log('[Sentry] Would have sent:', event);
        return null;
      }

      // Strip PII from events
      if (event.user) {
        delete event.user.email;
        delete event.user.ip_address;
      }

      return event;
    },

    // Before sending a breadcrumb
    beforeBreadcrumb(breadcrumb) {
      // Filter out noisy breadcrumbs
      if (breadcrumb.category === 'console' && breadcrumb.level === 'debug') {
        return null;
      }
      return breadcrumb;
    },
  });
}

// Production configuration
export function initProductionSentry(): void {
  initSentry({
    dsn: 'https://your-dsn@sentry.io/your-project-id',
    environment: 'production',
    release: `com.yourapp@${DeviceInfo.getVersion()}+${DeviceInfo.getBuildNumber()}`,
    dist: DeviceInfo.getBuildNumber(),
    tracesSampleRate: 0.2,        // 20% of transactions
    profilesSampleRate: 0.1,       // 10% of profiled transactions
    replaysSessionSampleRate: 0.1, // 10% of sessions get replay
    replaysOnErrorSampleRate: 1.0, // 100% of error sessions get replay
  });
}
```

#### Sentry's React Navigation Integration

One of Sentry's best features is automatic screen tracking with React Navigation:

```typescript
// App.tsx

import * as Sentry from '@sentry/react-native';
import { NavigationContainer } from '@react-navigation/native';
import React, { useRef } from 'react';

const reactNavigationIntegration = Sentry.reactNavigationIntegration();

// Wrap your app with Sentry
const App = Sentry.wrap(() => {
  const navigationRef = useRef(null);

  return (
    <NavigationContainer
      ref={navigationRef}
      onReady={() => {
        reactNavigationIntegration.registerNavigationContainer(navigationRef);
      }}
    >
      {/* Your navigation structure */}
    </NavigationContainer>
  );
});

export default App;
```

This automatically creates transactions for every screen navigation, measuring:
- Time to initial display (TTID)
- Time to full display (TTFD)
- Screen render time
- JS frame rate during transitions

#### Sentry Custom Spans and Transactions

This is where Sentry really shines for performance monitoring:

```typescript
// Measuring a complex operation with custom spans

import * as Sentry from '@sentry/react-native';

async function loadProductScreen(productId: string): Promise<Product> {
  return Sentry.startSpan(
    {
      name: 'product.screen.load',
      op: 'ui.load',
      attributes: {
        'product.id': productId,
      },
    },
    async (span) => {
      // Fetch product data
      const product = await Sentry.startSpan(
        { name: 'product.fetch', op: 'http.client' },
        async () => {
          return api.getProduct(productId);
        }
      );

      // Fetch reviews in parallel with recommendations
      const [reviews, recommendations] = await Promise.all([
        Sentry.startSpan(
          { name: 'reviews.fetch', op: 'http.client' },
          () => api.getReviews(productId)
        ),
        Sentry.startSpan(
          { name: 'recommendations.fetch', op: 'http.client' },
          () => api.getRecommendations(productId)
        ),
      ]);

      // Image preloading
      await Sentry.startSpan(
        { name: 'images.preload', op: 'resource.image' },
        async () => {
          await Promise.all(
            product.images.map((url) => Image.prefetch(url))
          );
        }
      );

      return { ...product, reviews, recommendations };
    }
  );
}
```

In Sentry's performance dashboard, you'd see this as a waterfall:

```
product.screen.load [============================================] 1200ms
  product.fetch      [===============]                              450ms
  reviews.fetch                       [========]                    300ms
  recommendations.fetch               [=====]                      200ms
  images.preload                                 [=================] 550ms
```

This kind of visibility is transformative. You can immediately see that image preloading is the bottleneck and focus your optimization efforts there.

### The Head-to-Head Comparison

| Feature | Sentry | Crashlytics |
|---------|--------|-------------|
| **Price** | Free tier (5K errors/mo), paid plans from $26/mo | Completely free, no limits |
| **Crash Reporting** | Excellent | Excellent |
| **Non-Fatal Error Tracking** | Excellent | Good |
| **Performance Monitoring** | Built-in, excellent | Separate product (Firebase Performance) |
| **Session Replay** | Yes (mobile) | No |
| **Source Map Support** | Excellent | N/A (different approach) |
| **Native Symbolication** | Good | Excellent |
| **Issue Grouping** | Sophisticated fingerprinting | Good, but simpler |
| **Release Tracking** | Excellent, with commit integration | Basic |
| **Breadcrumbs** | Automatic + manual | Manual only |
| **User Feedback Widget** | Yes | No |
| **Alert Configuration** | Highly configurable | Basic |
| **Dashboard Customization** | Extensive | Limited to Firebase Console |
| **Self-Hosted Option** | Yes | No |
| **React Native SDK** | First-class | Via react-native-firebase |
| **Distributed Tracing** | Yes | No |
| **CI/CD Integration** | Excellent (source maps, releases) | Good (dSYM upload) |
| **On-Call Integration** | PagerDuty, Opsgenie, Slack | Firebase Alerts (limited) |
| **Data Retention** | 30-90 days (plan dependent) | 90 days |
| **GDPR Compliance** | EU data residency available | Google's privacy terms |

### When to Use Crashlytics

Use Crashlytics when:

- You're already in the Firebase ecosystem (Analytics, Remote Config, etc.)
- You're cost-sensitive and need free, unlimited crash reporting
- Your primary concern is native crash analysis
- You want tight integration with Google Play Console's vitals
- Your team is small and doesn't need advanced triage workflows

### When to Use Sentry

Use Sentry when:

- You need performance monitoring alongside error tracking
- You want session replay for debugging
- You need sophisticated alerting and on-call integration
- Your team is large and needs advanced issue triage workflows
- You want release health tracking with commit association
- You need self-hosted options for compliance reasons
- You want distributed tracing across your mobile app and backend

### When to Use Both (The Production Answer)

Here's what many mature mobile teams actually do:

```
Crashlytics:  Native crash reporting + Firebase ecosystem integration
Sentry:       JS error tracking + performance monitoring + session replay
```

This gives you the best of both worlds:

- Crashlytics excels at native crash analysis (OOM kills, signal crashes, native library crashes)
- Sentry excels at JavaScript-level error tracking and performance monitoring
- You get redundancy — if one tool misses something, the other catches it

```typescript
// src/monitoring/index.ts — Unified monitoring service

import { crashlyticsService } from './crashlytics';
import { sentryService } from './sentry';

class MonitoringService {
  async init(userId?: string): Promise<void> {
    await Promise.all([
      crashlyticsService.init({
        enabled: !__DEV__,
        userId,
      }),
      sentryService.init({
        environment: __DEV__ ? 'development' : 'production',
        userId,
      }),
    ]);
  }

  // Report errors to both services
  captureError(error: Error, context?: Record<string, string>): void {
    // Sentry for JS-level tracking with full context
    sentryService.captureException(error, context);

    // Crashlytics for the Firebase ecosystem view
    crashlyticsService.recordError(error, context);
  }

  // Breadcrumbs go to both
  addBreadcrumb(message: string, category: string, data?: Record<string, string>): void {
    sentryService.addBreadcrumb(message, category, data);
    crashlyticsService.log(`[${category}] ${message}`);
  }

  // Performance traces only in Sentry (it's built for this)
  startTransaction(name: string, op: string) {
    return sentryService.startTransaction(name, op);
  }

  // User identification in both
  async identifyUser(userId: string, attributes?: Record<string, string>): Promise<void> {
    sentryService.setUser(userId, attributes);
    await crashlyticsService.setUser(userId, attributes);
  }

  // Clear user from both on logout
  async clearUser(): Promise<void> {
    sentryService.clearUser();
    await crashlyticsService.clearUser();
  }
}

export const monitoring = new MonitoringService();
```

This unified interface means your app code never needs to know which monitoring tool it's talking to. You just call `monitoring.captureError()` and both tools get the data.

---

## Part 3: Crash-Free Rate — The Single Most Important Mobile Metric

### What Crash-Free Rate Actually Means

Crash-free rate is the percentage of user sessions that didn't experience a crash. It sounds simple, but understanding the nuances is critical.

**Crash-free sessions** = (Total sessions - Sessions with crashes) / Total sessions * 100

So if you have 100,000 sessions and 100 of them crashed:

```
Crash-free rate = (100,000 - 100) / 100,000 * 100 = 99.9%
```

99.9% sounds great, right? But let's put it in perspective:

| Crash-Free Rate | What It Means | Practical Impact |
|-----------------|---------------|------------------|
| 99.99% | 1 crash per 10,000 sessions | Elite. Best-in-class apps. |
| 99.9% | 1 crash per 1,000 sessions | Good. Industry target for top apps. |
| 99.5% | 5 crashes per 1,000 sessions | Acceptable. You should be working to improve. |
| 99.0% | 10 crashes per 1,000 sessions | Concerning. Users are noticing. |
| 98.0% | 20 crashes per 1,000 sessions | Bad. App store reviews are suffering. |
| 95.0% | 50 crashes per 1,000 sessions | Critical. Fix this before anything else. |

Google Play Console uses crash-free rate as a key signal for app quality. If your crash-free rate drops below certain thresholds, Google will:
- Reduce your app's visibility in search results
- Show warning labels to users considering installing your app
- In extreme cases, flag your app for review

Apple doesn't publish specific thresholds, but they track crash rates in App Store Connect and use them in editorial decisions.

### The Two Flavors: Crash-Free Users vs Crash-Free Sessions

There's an important distinction:

- **Crash-free users**: What percentage of your users experienced zero crashes in a given period?
- **Crash-free sessions**: What percentage of individual sessions were crash-free?

These numbers can diverge significantly. Consider a scenario where one user crashes 50 times because they keep retrying:

- Crash-free users: 99.99% (only 1 user affected out of 10,000)
- Crash-free sessions: 99.5% (50 crash sessions out of 10,000)

Both metrics matter, but for different reasons:
- **Crash-free users** tells you how many people are having a bad experience
- **Crash-free sessions** tells you how unreliable your app is overall

Most teams track crash-free sessions as their primary metric and crash-free users as a secondary metric.

### How Shopify Maintains 99.9%+ Crash-Free Rate

Shopify's Shop app is one of the most widely-used React Native apps in production, with millions of daily active users. Here's what they do (based on their public engineering blog posts and conference talks):

**1. Crash Budget**

They treat crash-free rate like an error budget in SRE. If the crash-free rate drops below 99.9%, that becomes the team's top priority. Feature work stops. Everything else is secondary.

**2. Automated Regression Detection**

They have automated systems that compare crash rates between app versions. If a new version shows a statistically significant increase in crashes — even if the absolute number is small — it gets flagged automatically.

**3. Canary Releases**

New versions roll out to 1% of users first. They monitor crash-free rate for that 1% for 24-48 hours before expanding. If the canary shows elevated crash rates, the rollout stops.

**4. Weekly Crash Triage**

Every week, the top 10 crashes by frequency are reviewed by the engineering team. Each crash gets assigned an owner, a severity, and a timeline for resolution.

**5. Crash-Free Rate in CI**

They run crash detection in their CI pipeline. Certain test suites specifically check for crash scenarios, and failing these tests blocks deployment.

### Tracking Crash-Free Rate

Here's how to track it yourself:

```typescript
// Using Sentry's release health
// Sentry automatically tracks session data when you initialize the SDK

import * as Sentry from '@sentry/react-native';

// Sessions are tracked automatically, but you can also manually manage them
// This is useful for apps that have long-running sessions

// Start a session when the app comes to the foreground
import { AppState, AppStateStatus } from 'react-native';

let appState = AppState.currentState;

AppState.addEventListener('change', (nextAppState: AppStateStatus) => {
  if (appState.match(/inactive|background/) && nextAppState === 'active') {
    // App came to foreground — Sentry handles this automatically
    // but you might want to track this in your own analytics too
    monitoring.addBreadcrumb('App foregrounded', 'lifecycle');
  }

  if (nextAppState === 'background') {
    monitoring.addBreadcrumb('App backgrounded', 'lifecycle');
  }

  appState = nextAppState;
});
```

In the Sentry dashboard, you can view:
- **Release Health** → Shows crash-free rate per release
- **Session data** → Shows total sessions, errored sessions, crashed sessions
- **Trends** → Shows how crash-free rate changes over time

In Firebase Crashlytics:
- The main dashboard shows crash-free users/sessions front and center
- You can filter by app version, OS version, device type
- The "Trends" tab shows crash-free rate over time

### What to Do When Crash-Free Rate Drops

When you see the crash-free rate drop, here's your playbook:

**Step 1: Assess Severity (5 minutes)**

```
Is the drop:
  > 1% (e.g., 99.9% → 98.5%)?  → CRITICAL. All hands on deck.
  > 0.5% (e.g., 99.9% → 99.3%)? → HIGH. Drop current work.
  > 0.1% (e.g., 99.9% → 99.7%)? → MEDIUM. Schedule fix this sprint.
  < 0.1%?                         → LOW. Add to backlog, fix within 2 weeks.
```

**Step 2: Identify the Cause (30-60 minutes)**

1. Check Sentry/Crashlytics for new crash groups that appeared around the time of the drop
2. Correlate with recent releases — did the drop coincide with a new version?
3. Check if it's platform-specific (Android only? iOS only? Specific OS version?)
4. Check if it's device-specific (low-memory devices? specific manufacturers?)

**Step 3: Triage and Fix**

For CRITICAL drops:
1. If the crash is in the latest release, prepare a hotfix immediately
2. If possible, use feature flags to disable the offending feature
3. If the crash is widespread, consider rolling back the release
4. Communicate to stakeholders: "We've identified a crash regression affecting X% of users. Fix ETA: Y hours."

For HIGH/MEDIUM drops:
1. Create a ticket with full reproduction steps
2. Assign it to the engineer who owns the affected code
3. Set a deadline based on severity
4. Monitor the crash-free rate after the fix ships

### Real-World Crash-Free Rate Dashboard

Here's what I recommend putting on your team's primary dashboard:

```
+--------------------------------------------------+
|  MOBILE HEALTH DASHBOARD                          |
+--------------------------------------------------+
|                                                    |
|  Crash-Free Sessions (7 day)                       |
|  iOS:     99.92% [============================] |
|  Android: 99.87% [===========================]   |
|                                                    |
|  Crash-Free Sessions (24 hour)                     |
|  iOS:     99.95% [============================] |
|  Android: 99.83% [==========================] !!  |
|                                                    |
|  Top Crashes (Last 24h)                            |
|  1. NullPointerException in CartScreen    (142)    |
|  2. NetworkError in PaymentService        (87)     |
|  3. OutOfMemoryError in ImageGallery      (34)     |
|                                                    |
|  Version Comparison                                |
|  v3.2.1: 99.92%                                    |
|  v3.2.0: 99.89%                                    |
|  v3.1.9: 99.91%                                    |
+--------------------------------------------------+
```

The `!!` next to Android's 24-hour crash-free rate is there because it dropped below the 7-day average, suggesting a regression. That's a signal worth investigating immediately.

---

## Part 4: ANR (App Not Responding) — The Silent Killer

### What ANR Is and Why It Matters

ANR stands for "App Not Responding." It's an Android-specific concept (iOS has its own equivalent called "watchdog terminations," but we'll focus on Android first since the tooling is more explicit).

An ANR occurs when the main thread of an Android app is blocked for too long:

- **5 seconds** for foreground activities (the user sees a "app isn't responding" dialog)
- **5 seconds** for foreground broadcast receivers
- **10 seconds** for background services (varies by Android version)

When an ANR happens, the user sees a dialog asking if they want to wait or force-close the app. In most cases, they force-close. And Google Play tracks your ANR rate — it's one of the Android vitals that affects your store listing.

**Google Play's thresholds:**
- ANR rate > 0.47% (user-perceived) — "Bad behavior" threshold
- ANR rate > 8% — Possible store visibility reduction

### What Causes ANRs in React Native

In React Native, ANRs can come from several sources:

**1. JS Thread Blocking**

The most common cause. When your JavaScript code runs a long synchronous operation, it blocks the JS thread. While the JS thread itself isn't the main thread, it prevents React Native from processing touch events and updating the UI, which eventually causes the Android system to detect an unresponsive state.

```typescript
// BAD: This will cause ANR on large datasets
function processLargeDataset(data: Item[]) {
  // Synchronous processing of 10,000 items
  const results = data.map(item => {
    return expensiveComputation(item);
  });
  setResults(results);
}

// GOOD: Process in chunks using InteractionManager
import { InteractionManager } from 'react-native';

async function processLargeDataset(data: Item[]) {
  // Wait for animations to complete
  await InteractionManager.runAfterInteractions();

  const CHUNK_SIZE = 100;
  const results: ProcessedItem[] = [];

  for (let i = 0; i < data.length; i += CHUNK_SIZE) {
    const chunk = data.slice(i, i + CHUNK_SIZE);
    const chunkResults = chunk.map(item => expensiveComputation(item));
    results.push(...chunkResults);

    // Yield to the event loop between chunks
    await new Promise(resolve => setTimeout(resolve, 0));
  }

  setResults(results);
}
```

**2. Bridge Congestion (Old Architecture)**

In the old React Native architecture (before the New Architecture / JSI), heavy bridge traffic could cause ANRs. Sending too many messages across the bridge in a short time would queue up on the native side, blocking the main thread.

```typescript
// BAD: Sending hundreds of bridge calls in a loop
items.forEach(item => {
  NativeModule.processItem(item); // Each call goes across the bridge
});

// GOOD: Batch operations
NativeModule.processItems(items); // Single bridge call with all data
```

**3. Synchronous Native Calls**

Some native modules perform synchronous operations on the main thread. If you're calling these from your JS code and they take too long, ANR.

```typescript
// BAD: Synchronous file read on main thread
const data = RNFS.readFileSync('/path/to/large/file');

// GOOD: Asynchronous file read
const data = await RNFS.readFile('/path/to/large/file');
```

**4. Large Layout Computations**

Rendering extremely complex layouts — deeply nested views, large lists without virtualization — can block the main thread during layout computation.

```typescript
// BAD: Rendering 1000 items in a ScrollView
<ScrollView>
  {items.map(item => (
    <ComplexItemCard key={item.id} item={item} />
  ))}
</ScrollView>

// GOOD: Using FlatList with proper optimization
<FlatList
  data={items}
  renderItem={({ item }) => <ComplexItemCard item={item} />}
  keyExtractor={item => item.id}
  windowSize={5}
  maxToRenderPerBatch={10}
  initialNumToRender={10}
  removeClippedSubviews={true}
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
/>
```

**5. Large Shared Preferences / AsyncStorage Operations**

Storing or retrieving large amounts of data from AsyncStorage can block the main thread on Android:

```typescript
// BAD: Storing a huge object
await AsyncStorage.setItem('user_data', JSON.stringify(hugeObject));

// GOOD: Store incrementally or use a proper database
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();
storage.set('user_data', JSON.stringify(hugeObject)); // MMKV is much faster
```

### Detecting ANRs

**In Sentry:**

Sentry's React Native SDK automatically detects ANRs on Android. You'll see them in the Issues dashboard with the `mechanism:anr` tag.

```typescript
// Sentry ANR detection is automatic, but you can configure it
Sentry.init({
  dsn: 'your-dsn',
  enableAnrTracking: true, // Enabled by default
  anrTimeout: 5000, // Default: 5 seconds (matches Android's threshold)
});
```

**In Crashlytics:**

Firebase Crashlytics also captures ANRs automatically on Android. In the Firebase Console, you'll see a separate "ANRs" section alongside crashes.

**In Google Play Console:**

Google Play Console → Android Vitals → ANR rate gives you the most authoritative ANR data, since it comes directly from the OS.

### Fixing ANRs: A Systematic Approach

When you identify an ANR, here's how to approach fixing it:

**Step 1: Read the ANR trace**

ANR reports include a stack trace showing what every thread was doing at the moment the ANR was detected. Look for the main thread:

```
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x71c6e720 self=0x7f8e8c8000
  | sysTid=12345 nice=-10 cgrp=default sched=0/0 handle=0x7f9219c548
  | state=S schedstat=( 12345678 9876543 210 ) utm=15 stm=3 core=0 HZ=100
  | stack=0x7ff7b00000-0x7ff7b02000 stackSize=8388608
  at com.facebook.react.bridge.queue.MessageQueueThreadImpl.m(MessageQueueThreadImpl.java:XX)
  - waiting to lock <0x0d4c5a70> held by thread 42
```

This tells you the main thread is waiting to acquire a lock that's held by thread 42. Now go look at what thread 42 is doing.

**Step 2: Categorize the ANR**

ANRs fall into categories:

| Category | Typical Cause | Solution |
|----------|---------------|----------|
| JS Thread Blocking | Heavy computation | Move to worker thread or chunk the work |
| Bridge Congestion | Too many bridge calls | Batch operations, use JSI |
| Disk I/O on Main Thread | File reads, database queries | Make async, move off main thread |
| Network on Main Thread | Synchronous HTTP calls | Always use async networking |
| Deadlock | Thread synchronization issues | Fix locking order, use timeouts |
| Layout Thrashing | Complex layout recalculations | Simplify layout, use virtualization |

**Step 3: Apply the Fix**

Here's a real-world example of fixing a common ANR in React Native:

```typescript
// BEFORE: ANR caused by JSON parsing a large API response on the JS thread

async function loadFeed(): Promise<FeedItem[]> {
  const response = await fetch('https://api.example.com/feed');
  const text = await response.text();
  // This JSON.parse call blocks the JS thread for 200-500ms on large payloads
  const data = JSON.parse(text);
  return data.items;
}

// AFTER: Move heavy parsing off the main JS thread

// Option 1: Use a native JSON parser that runs on a background thread
import { parseJSON } from 'react-native-fast-json';

async function loadFeed(): Promise<FeedItem[]> {
  const response = await fetch('https://api.example.com/feed');
  const text = await response.text();
  const data = await parseJSON(text); // Runs on native background thread
  return data.items;
}

// Option 2: Paginate your API so you never get huge payloads
async function loadFeed(page: number = 1): Promise<FeedItem[]> {
  const response = await fetch(
    `https://api.example.com/feed?page=${page}&limit=20`
  );
  return response.json(); // Small payloads parse quickly
}

// Option 3: Use streaming JSON parsing for very large datasets
import { createJsonStreamParser } from './utils/streaming-json';

async function loadFeed(): Promise<FeedItem[]> {
  const response = await fetch('https://api.example.com/feed');
  const reader = response.body?.getReader();
  if (!reader) throw new Error('No response body');

  const items: FeedItem[] = [];
  const parser = createJsonStreamParser({
    onItem: (item: FeedItem) => {
      items.push(item);
      // Update UI incrementally as items arrive
      if (items.length % 10 === 0) {
        setPartialResults([...items]);
      }
    },
  });

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    parser.write(value);
  }

  return items;
}
```

### iOS Equivalent: Watchdog Terminations

iOS doesn't have the same ANR dialog, but it has something worse — the watchdog timer. If your app's main thread is unresponsive for approximately 20 seconds, iOS will terminate your app. No dialog. No warning to the user. The app just disappears.

These show up in Xcode Organizer as "watchdog terminations" and in Crashlytics as a specific crash type. They're harder to debug because the stack trace is often less informative.

The prevention strategies are the same: keep the main thread free, use background threads for heavy work, and profile your app regularly.

---

## Part 5: Source Maps and Symbolication — Making Crash Reports Readable

### The Problem

Here's what a crash report looks like without source maps:

```
TypeError: Cannot read property 'name' of undefined
  at a (main.jsbundle:1:234567)
  at s (main.jsbundle:1:345678)
  at u (main.jsbundle:1:456789)
  at Object.v (main.jsbundle:1:567890)
```

This is useless. You can't tell what `a`, `s`, `u`, or `v` are. You can't find the file. You can't see the line of code. You're debugging blind.

Here's what the same crash looks like WITH source maps:

```
TypeError: Cannot read property 'name' of undefined
  at getUserDisplayName (src/utils/user.ts:42:18)
  at ProfileHeader (src/components/ProfileHeader.tsx:23:25)
  at renderScreen (src/screens/ProfileScreen.tsx:67:12)
  at AppNavigator (src/navigation/AppNavigator.tsx:134:8)
```

Now you know exactly where to look. You can open `src/utils/user.ts` at line 42 and immediately see the bug.

Source maps are the translation layer between your minified/bundled production code and your original source code. Without them, your crash reports are nearly worthless.

### How Source Maps Work

When your React Native app is built for production, the JavaScript code goes through several transformations:

1. **TypeScript compilation** → Converts TS to JS
2. **Babel transformation** → Converts modern JS to compatible JS
3. **Metro bundling** → Combines all modules into a single bundle
4. **Minification** → Reduces bundle size by renaming variables, removing whitespace

Each step produces a source map that maps the output back to the input. The final source map maps the minified bundle back to your original TypeScript files.

### Uploading Source Maps to Sentry

This is one of the most critical setup steps and one that teams frequently get wrong.

#### Method 1: Using EAS Build (Recommended)

If you're using Expo Application Services (EAS), Sentry integration is straightforward:

```javascript
// app.json or app.config.js
{
  "expo": {
    "plugins": [
      [
        "@sentry/react-native/expo",
        {
          "organization": "your-org",
          "project": "your-project",
          "url": "https://sentry.io/"
        }
      ]
    ]
  }
}
```

```bash
# .env or eas.json
SENTRY_AUTH_TOKEN=your-auth-token
SENTRY_ORG=your-org
SENTRY_PROJECT=your-project
```

With this configuration, EAS Build automatically uploads source maps after every build. No manual steps required.

#### Method 2: Manual Upload with Sentry CLI

For custom build pipelines or bare React Native projects:

```bash
#!/bin/bash
# scripts/upload-source-maps.sh

# Configuration
SENTRY_ORG="your-org"
SENTRY_PROJECT="your-project"
VERSION=$(node -e "console.log(require('./package.json').version)")
BUILD_NUMBER=$(cat ios/build_number.txt 2>/dev/null || echo "1")
RELEASE="${SENTRY_PROJECT}@${VERSION}+${BUILD_NUMBER}"

echo "Uploading source maps for release: ${RELEASE}"

# Create a new release in Sentry
npx sentry-cli releases new "${RELEASE}"

# Upload iOS source maps
npx sentry-cli releases files "${RELEASE}" upload-sourcemaps \
  --dist "${BUILD_NUMBER}" \
  --strip-prefix /path/to/project \
  --rewrite \
  ios/build/Build/Products/Release-iphoneos/main.jsbundle \
  ios/build/Build/Products/Release-iphoneos/main.jsbundle.map

# Upload Android source maps
npx sentry-cli releases files "${RELEASE}" upload-sourcemaps \
  --dist "${BUILD_NUMBER}" \
  --strip-prefix /path/to/project \
  --rewrite \
  android/app/build/generated/assets/createBundleReleaseJsAndAssets/index.android.bundle \
  android/app/build/generated/sourcemaps/react/release/index.android.bundle.map

# Associate commits with the release
npx sentry-cli releases set-commits "${RELEASE}" --auto

# Finalize the release
npx sentry-cli releases finalize "${RELEASE}"

echo "Source maps uploaded successfully!"
```

#### Method 3: Using @sentry/react-native's Built-in Upload

The `@sentry/react-native` package includes build phase scripts that handle source map upload:

**For iOS (Xcode Build Phase):**

In Xcode, add a new "Run Script" build phase after the "Bundle React Native code and images" phase:

```bash
# Upload source maps to Sentry
export SENTRY_PROPERTIES=sentry.properties
../node_modules/@sentry/cli/bin/sentry-cli react-native xcode \
  ../node_modules/react-native/scripts/react-native-xcode.sh
```

**For Android (Gradle):**

```groovy
// android/app/build.gradle

apply from: "../../node_modules/@sentry/react-native/sentry.gradle"

// Configure Sentry plugin
sentry {
    autoUploadSourceMap = true
    autoUploadNativeSymbols = true
    includeSourceContext = true
    org = "your-org"
    projectName = "your-project"
}
```

### Uploading dSYMs for iOS Native Crashes

dSYMs (debug symbols) are to native iOS code what source maps are to JavaScript. Without them, native crash stack traces show memory addresses instead of function names.

```bash
#!/bin/bash
# scripts/upload-dsyms.sh

# Automatic dSYM upload after Xcode build
DSYM_PATH="${DWARF_DSYM_FOLDER_PATH}/${DWARF_DSYM_FILE_NAME}"

if [ -d "${DSYM_PATH}" ]; then
  echo "Uploading dSYMs from: ${DSYM_PATH}"

  npx sentry-cli debug-files upload \
    --org "your-org" \
    --project "your-project" \
    "${DSYM_PATH}"

  echo "dSYMs uploaded successfully!"
else
  echo "Warning: dSYM path not found: ${DSYM_PATH}"
fi
```

For apps distributed through App Store Connect with bitcode enabled, you'll also need to download the dSYMs from App Store Connect:

```bash
# Download dSYMs from App Store Connect
npx sentry-cli difutil find /path/to/your.app.dSYM

# Or use the App Store Connect API
npx sentry-cli debug-files upload --derived-data \
  --org "your-org" \
  --project "your-project"
```

### ProGuard/R8 Mappings for Android Native Crashes

Android uses ProGuard (or R8) to obfuscate native code. You need to upload the mapping files:

```groovy
// android/app/build.gradle

// If using Sentry's Gradle plugin (recommended)
apply from: "../../node_modules/@sentry/react-native/sentry.gradle"

// The plugin automatically uploads ProGuard/R8 mapping files
// Make sure you have these in your sentry.properties:
// defaults.org=your-org
// defaults.project=your-project
// auth.token=your-auth-token
```

```bash
# Manual upload of ProGuard mapping
npx sentry-cli debug-files upload \
  --org "your-org" \
  --project "your-project" \
  --type proguard \
  android/app/build/outputs/mapping/release/mapping.txt
```

### Uploading dSYMs/Mappings to Crashlytics

Crashlytics handles symbolication differently. For iOS, it requires a build phase script:

```bash
# In Xcode Build Phases, add this Run Script:
"${BUILD_DIR%/Build/*}/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/run"

# Or if using CocoaPods:
"${PODS_ROOT}/FirebaseCrashlytics/run"
```

For Android, the Crashlytics Gradle plugin handles it automatically:

```groovy
// android/app/build.gradle

apply plugin: 'com.google.firebase.crashlytics'

android {
    buildTypes {
        release {
            // Enable Crashlytics mapping upload
            firebaseCrashlytics {
                mappingFileUploadEnabled true
                nativeSymbolUploadEnabled true
            }
        }
    }
}
```

### Verifying Your Source Maps Work

This is a step most people skip, and then they wonder why their crash reports are still unreadable. Always verify:

```bash
# Verify source maps are uploaded to Sentry
npx sentry-cli releases files "your-project@1.0.0+1" list

# Expected output:
# +---------------------------+------+-------------------+
# | Name                      | Size | Source Map        |
# +---------------------------+------+-------------------+
# | main.jsbundle             | 2.1M | main.jsbundle.map |
# | main.jsbundle.map         | 5.4M |                   |
# +---------------------------+------+-------------------+

# Test symbolication with a specific event
npx sentry-cli events get <event-id> --org "your-org" --project "your-project"
```

And here's a simple test you can add to your CI:

```typescript
// e2e/source-maps.test.ts

import * as Sentry from '@sentry/react-native';

test('source maps are configured correctly', () => {
  try {
    throw new Error('Source map test error');
  } catch (error) {
    // In a real test, you'd check that the error was sent to Sentry
    // and that the stack trace is properly symbolicated
    expect(error.stack).toContain('source-maps.test.ts');
  }
});
```

### Common Source Map Pitfalls

**1. Mismatched release/dist values**

The most common issue. The `release` and `dist` values in your Sentry SDK initialization MUST exactly match the values used when uploading source maps.

```typescript
// In your app
Sentry.init({
  release: 'com.myapp@1.2.3+45',  // Must match
  dist: '45',                       // Must match
});
```

```bash
# When uploading
npx sentry-cli releases files "com.myapp@1.2.3+45" upload-sourcemaps \
  --dist "45" \  # Must match
  ./path/to/bundle.js ./path/to/bundle.js.map
```

**2. Source map file paths don't match**

The file paths in the source map must match what the runtime reports. Use `--strip-prefix` and `--rewrite` flags to fix path mismatches.

**3. Forgetting to upload after every build**

Source maps are specific to each build. If you build v1.2.3 today and v1.2.3 tomorrow (with code changes but same version), the second build's crashes will be symbolicated incorrectly with the first build's source maps. Always use a unique `dist` value (like the build number) for each build.

**4. Source maps not generated**

Make sure your Metro config actually generates source maps for production builds:

```javascript
// metro.config.js
module.exports = {
  transformer: {
    // Ensure source maps are generated
    minifierConfig: {
      sourceMap: {
        includeSources: true,
      },
    },
  },
};
```

---

## Part 6: Error Boundaries — Crash Containment in React Native

### What Error Boundaries Are

Error boundaries are React's mechanism for catching JavaScript errors in the component tree and displaying a fallback UI instead of crashing the entire app. Think of them as `try/catch` for your component tree.

Without error boundaries, a single `TypeError` in one component brings down your entire app. With error boundaries, the error is contained to that component's subtree, and the rest of the app continues to function.

### Why Error Boundaries Are Critical in Mobile

On the web, a crash means the user sees a white screen and refreshes. Annoying but recoverable.

On mobile, a crash means the app closes. The user has to find the app icon, tap it, wait for it to launch, and navigate back to where they were. Many users don't bother. They uninstall.

Error boundaries are your last line of defense before a crash becomes a force-close.

### Implementing Error Boundaries in React Native

React doesn't provide a hook-based API for error boundaries (as of React 19), so you still need a class component:

```typescript
// src/components/ErrorBoundary.tsx

import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Image } from 'react-native';
import * as Sentry from '@sentry/react-native';
import crashlytics from '@react-native-firebase/crashlytics';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  /** A human-readable name for this boundary (shown in reports) */
  boundaryName: string;
  /** Called when an error is caught — use for custom recovery logic */
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  /** If true, show a retry button instead of a generic error message */
  retryable?: boolean;
}

interface State {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
    };
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    this.setState({ errorInfo });

    // Report to Sentry with full context
    Sentry.withScope((scope) => {
      scope.setTag('error_boundary', this.props.boundaryName);
      scope.setExtra('componentStack', errorInfo.componentStack);
      scope.setLevel('error');
      Sentry.captureException(error);
    });

    // Report to Crashlytics
    crashlytics().setAttribute('error_boundary', this.props.boundaryName);
    crashlytics().log(`Error caught by boundary: ${this.props.boundaryName}`);
    crashlytics().recordError(error);

    // Call custom error handler if provided
    this.props.onError?.(error, errorInfo);
  }

  handleRetry = (): void => {
    this.setState({
      hasError: false,
      error: null,
      errorInfo: null,
    });
  };

  render(): ReactNode {
    if (this.state.hasError) {
      // Use custom fallback if provided
      if (this.props.fallback) {
        return this.props.fallback;
      }

      // Default fallback UI
      return (
        <View style={styles.container}>
          <Text style={styles.emoji}>!</Text>
          <Text style={styles.title}>Something went wrong</Text>
          <Text style={styles.message}>
            We've been notified and are working on a fix.
          </Text>

          {this.props.retryable && (
            <TouchableOpacity
              style={styles.retryButton}
              onPress={this.handleRetry}
            >
              <Text style={styles.retryText}>Try Again</Text>
            </TouchableOpacity>
          )}

          {__DEV__ && this.state.error && (
            <View style={styles.debugInfo}>
              <Text style={styles.debugTitle}>Debug Info:</Text>
              <Text style={styles.debugText}>
                {this.state.error.toString()}
              </Text>
              <Text style={styles.debugText}>
                {this.state.errorInfo?.componentStack}
              </Text>
            </View>
          )}
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#f8f9fa',
  },
  emoji: {
    fontSize: 48,
    fontWeight: 'bold',
    marginBottom: 16,
    color: '#dc3545',
  },
  title: {
    fontSize: 20,
    fontWeight: '600',
    color: '#212529',
    marginBottom: 8,
  },
  message: {
    fontSize: 14,
    color: '#6c757d',
    textAlign: 'center',
    marginBottom: 24,
  },
  retryButton: {
    backgroundColor: '#007bff',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  retryText: {
    color: '#ffffff',
    fontSize: 16,
    fontWeight: '600',
  },
  debugInfo: {
    marginTop: 24,
    padding: 16,
    backgroundColor: '#1e1e1e',
    borderRadius: 8,
    width: '100%',
  },
  debugTitle: {
    color: '#ff6b6b',
    fontSize: 14,
    fontWeight: '600',
    marginBottom: 8,
  },
  debugText: {
    color: '#e0e0e0',
    fontSize: 12,
    fontFamily: 'monospace',
  },
});
```

### Strategic Error Boundary Placement

Don't just wrap your entire app in one error boundary and call it a day. That's like having one `try/catch` around your entire codebase — technically correct but practically useless. You want error boundaries at strategic points:

```typescript
// App.tsx — The layered error boundary approach

import React from 'react';
import { ErrorBoundary } from './components/ErrorBoundary';

function App() {
  return (
    // Level 1: App-level boundary — catches everything, shows full-screen error
    <ErrorBoundary boundaryName="app-root" retryable={false}>

      <NavigationContainer>
        <MainNavigator />
      </NavigationContainer>

    </ErrorBoundary>
  );
}

// In your screen components
function HomeScreen() {
  return (
    // Level 2: Screen-level boundary — catches errors in this screen
    <ErrorBoundary boundaryName="home-screen" retryable={true}>
      <View style={styles.container}>
        <Header />

        {/* Level 3: Section-level boundary — contains errors to specific sections */}
        <ErrorBoundary
          boundaryName="home-featured-section"
          retryable={true}
          fallback={<FeaturedSectionPlaceholder />}
        >
          <FeaturedSection />
        </ErrorBoundary>

        <ErrorBoundary
          boundaryName="home-recommendations"
          retryable={true}
          fallback={<RecommendationsPlaceholder />}
        >
          <RecommendationsSection />
        </ErrorBoundary>

        <ErrorBoundary
          boundaryName="home-recent-orders"
          retryable={true}
          fallback={<RecentOrdersPlaceholder />}
        >
          <RecentOrders />
        </ErrorBoundary>
      </View>
    </ErrorBoundary>
  );
}
```

With this approach:
- If `FeaturedSection` crashes, the rest of `HomeScreen` still works
- If the entire `HomeScreen` crashes, the user sees a retry screen for just that screen
- If the navigation itself crashes, the app-level boundary catches it

### The Graceful Degradation Pattern

Here's an advanced pattern where error boundaries enable graceful degradation:

```typescript
// src/components/GracefulBoundary.tsx

import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import * as Sentry from '@sentry/react-native';

interface Props {
  children: ReactNode;
  boundaryName: string;
  /** What to show when the primary content fails */
  degradedComponent?: ReactNode;
  /** What to show when even the degraded component fails */
  minimalComponent?: ReactNode;
  /** If true, silently degrade without showing any error UI */
  silent?: boolean;
}

interface State {
  errorLevel: 'none' | 'degraded' | 'minimal' | 'failed';
}

export class GracefulBoundary extends Component<Props, State> {
  state: State = { errorLevel: 'none' };

  static getDerivedStateFromError(): Partial<State> {
    return {}; // We handle state in componentDidCatch
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    Sentry.withScope((scope) => {
      scope.setTag('error_boundary', this.props.boundaryName);
      scope.setTag('degradation_level', this.state.errorLevel);
      scope.setExtra('componentStack', errorInfo.componentStack);
      Sentry.captureException(error);
    });

    // Progressive degradation
    this.setState((prevState) => {
      switch (prevState.errorLevel) {
        case 'none':
          return { errorLevel: 'degraded' };
        case 'degraded':
          return { errorLevel: 'minimal' };
        default:
          return { errorLevel: 'failed' };
      }
    });
  }

  render(): ReactNode {
    switch (this.state.errorLevel) {
      case 'none':
        return this.props.children;

      case 'degraded':
        return this.props.degradedComponent || (
          <View style={styles.degraded}>
            <Text style={styles.degradedText}>
              Limited functionality available
            </Text>
          </View>
        );

      case 'minimal':
        return this.props.minimalComponent || (
          this.props.silent ? null : (
            <View style={styles.minimal}>
              <Text style={styles.minimalText}>
                This section is temporarily unavailable
              </Text>
            </View>
          )
        );

      case 'failed':
        return this.props.silent ? null : (
          <View style={styles.failed}>
            <Text style={styles.failedText}>
              Unable to load this content
            </Text>
          </View>
        );
    }
  }
}

const styles = StyleSheet.create({
  degraded: {
    padding: 16,
    backgroundColor: '#fff3cd',
    borderRadius: 8,
    margin: 8,
  },
  degradedText: {
    color: '#856404',
    textAlign: 'center',
  },
  minimal: {
    padding: 12,
    backgroundColor: '#f8f9fa',
    borderRadius: 8,
    margin: 8,
  },
  minimalText: {
    color: '#6c757d',
    textAlign: 'center',
    fontSize: 13,
  },
  failed: {
    padding: 8,
  },
  failedText: {
    color: '#adb5bd',
    textAlign: 'center',
    fontSize: 12,
  },
});
```

Usage:

```typescript
<GracefulBoundary
  boundaryName="product-reviews"
  degradedComponent={<SimpleReviewsList productId={productId} />}
  minimalComponent={<Text>See reviews on our website</Text>}
>
  <RichReviewsSection productId={productId} />
</GracefulBoundary>
```

This pattern says: "Try the rich reviews section first. If that crashes, fall back to a simple list. If that also crashes, just show a link to the website."

### Error Boundaries and Sentry's React Integration

Sentry provides its own error boundary wrapper that automatically reports errors:

```typescript
import * as Sentry from '@sentry/react-native';

// Sentry's built-in error boundary
function App() {
  return (
    <Sentry.ErrorBoundary
      fallback={({ error, resetError }) => (
        <View style={styles.errorContainer}>
          <Text style={styles.errorTitle}>Oops!</Text>
          <Text style={styles.errorMessage}>{error.toString()}</Text>
          <TouchableOpacity onPress={resetError}>
            <Text>Try Again</Text>
          </TouchableOpacity>
        </View>
      )}
      showDialog  // Shows Sentry's feedback dialog
    >
      <MainApp />
    </Sentry.ErrorBoundary>
  );
}
```

I recommend using Sentry's error boundary at the top level (for the automatic reporting) and your custom boundaries at the section level (for granular degradation).

### What Error Boundaries Don't Catch

Important limitations to be aware of:

| Error Type | Caught by Error Boundaries? | Alternative |
|------------|----------------------------|-------------|
| Render errors | Yes | - |
| Lifecycle errors | Yes | - |
| Constructor errors | Yes | - |
| Event handler errors | No | try/catch in handlers |
| Async code (promises) | No | .catch() or try/catch with await |
| setTimeout/setInterval | No | try/catch inside callbacks |
| Native module errors | No | Native crash reporting |
| Errors in the error boundary itself | No | Nested error boundaries |

For async errors, you need explicit error handling:

```typescript
// Error boundaries don't catch this
async function handlePress() {
  const data = await fetchData(); // If this throws, no boundary catches it
  setData(data);
}

// You need this instead
async function handlePress() {
  try {
    const data = await fetchData();
    setData(data);
  } catch (error) {
    Sentry.captureException(error);
    crashlytics().recordError(error as Error);
    showErrorToast('Failed to load data. Please try again.');
  }
}
```

Set up a global unhandled promise rejection handler as a safety net:

```typescript
// src/monitoring/unhandled-errors.ts

import * as Sentry from '@sentry/react-native';
import crashlytics from '@react-native-firebase/crashlytics';
import { LogBox } from 'react-native';

// Global handler for unhandled promise rejections
const originalHandler = global.ErrorUtils?.getGlobalHandler();

global.ErrorUtils?.setGlobalHandler((error: Error, isFatal?: boolean) => {
  // Report to monitoring
  Sentry.withScope((scope) => {
    scope.setTag('fatal', isFatal ? 'true' : 'false');
    scope.setTag('handler', 'global');
    Sentry.captureException(error);
  });

  crashlytics().setAttribute('fatal', isFatal ? 'true' : 'false');
  crashlytics().recordError(error);

  // Call original handler (this will show the red screen in dev)
  originalHandler?.(error, isFatal);
});

// Handle unhandled promise rejections
if (__DEV__) {
  // In dev, we want to see the warnings
  LogBox.ignoreLogs([]); // Don't ignore anything in dev
} else {
  // In production, capture and report
  const rejectionTracking = require('promise/setimmediate/rejection-tracking');
  rejectionTracking.enable({
    allRejections: true,
    onUnhandled: (id: number, error: Error) => {
      Sentry.captureException(error, {
        tags: { type: 'unhandled_promise_rejection' },
      });
      crashlytics().log('Unhandled promise rejection');
      crashlytics().recordError(error);
    },
    onHandled: () => {
      // Promise was eventually handled, no action needed
    },
  });
}
```

---

## Part 7: Custom Performance Traces — Measuring What Matters

### Beyond Crash Rates

Crash-free rate tells you if your app is stable. Performance traces tell you if your app is fast. Both matter, but performance is harder to measure because "fast" is subjective and context-dependent.

A screen that loads in 200ms feels instant. At 500ms, users notice. At 1000ms, they're impatient. At 3000ms, they're gone.

The problem is that your app's performance varies wildly across devices. What loads in 200ms on an iPhone 16 Pro might take 2000ms on a budget Android device from 2020. You need to measure actual user experience, not just what happens on your MacBook in the simulator.

### What to Measure

Here are the performance metrics that actually matter for mobile apps:

| Metric | What It Measures | Target |
|--------|------------------|--------|
| **Cold Start Time** | Time from app launch to first interactive screen | < 2 seconds |
| **Screen Load Time** | Time from navigation to screen fully rendered | < 500ms |
| **Time to Interactive (TTI)** | Time until user can interact with the screen | < 1 second |
| **API Response Time** | Time for network requests to complete | < 500ms (p95) |
| **Image Load Time** | Time for images to render on screen | < 200ms (cached) |
| **Frame Rate** | Frames per second during interactions | > 55 FPS |
| **JS Frame Rate** | JavaScript thread frame rate | > 55 FPS |
| **Memory Usage** | App memory consumption | < 200MB typical |
| **Battery Impact** | CPU usage during common operations | Low |

### Measuring Cold Start Time

Cold start is one of the most impactful metrics. It's the user's first impression of your app.

```typescript
// src/monitoring/performance.ts

import * as Sentry from '@sentry/react-native';
import performance from 'react-native-performance';

class PerformanceMonitor {
  private coldStartTimestamp: number | null = null;

  markColdStart(): void {
    this.coldStartTimestamp = performance.now();
  }

  markAppReady(): void {
    if (!this.coldStartTimestamp) return;

    const duration = performance.now() - this.coldStartTimestamp;

    // Report to Sentry
    Sentry.startSpan(
      {
        name: 'app.cold_start',
        op: 'app.start.cold',
        startTime: this.coldStartTimestamp / 1000,
      },
      (span) => {
        span.setAttribute('duration_ms', duration);
        span.end((this.coldStartTimestamp! + duration) / 1000);
      }
    );

    // Log for analytics
    console.log(`[Performance] Cold start: ${duration.toFixed(0)}ms`);
    this.coldStartTimestamp = null;
  }
}

export const performanceMonitor = new PerformanceMonitor();
```

```typescript
// index.js — Mark cold start as early as possible
import { performanceMonitor } from './src/monitoring/performance';
performanceMonitor.markColdStart();

import { AppRegistry } from 'react-native';
import App from './src/App';

AppRegistry.registerComponent('YourApp', () => App);
```

```typescript
// src/App.tsx — Mark when app is ready
import React, { useEffect } from 'react';
import { performanceMonitor } from './monitoring/performance';

function App() {
  useEffect(() => {
    // Mark app as ready after first render
    requestAnimationFrame(() => {
      performanceMonitor.markAppReady();
    });
  }, []);

  return <MainApp />;
}
```

### Measuring Screen Load Time

Every screen in your app should have its load time measured:

```typescript
// src/hooks/useScreenPerformance.ts

import { useEffect, useRef } from 'react';
import * as Sentry from '@sentry/react-native';

export function useScreenPerformance(screenName: string) {
  const startTime = useRef(Date.now());
  const reported = useRef(false);

  useEffect(() => {
    // Screen mounted — record start time
    startTime.current = Date.now();
    reported.current = false;
  }, [screenName]);

  // Call this when the screen's data is fully loaded and rendered
  const markReady = () => {
    if (reported.current) return;
    reported.current = true;

    const duration = Date.now() - startTime.current;

    Sentry.startSpan(
      {
        name: `screen.${screenName}`,
        op: 'ui.load',
      },
      (span) => {
        span.setAttribute('screen.name', screenName);
        span.setAttribute('duration_ms', duration);
      }
    );
  };

  // Call this when the screen first shows content (even before all data loads)
  const markFirstContentfulPaint = () => {
    const duration = Date.now() - startTime.current;

    Sentry.addBreadcrumb({
      category: 'performance',
      message: `${screenName} FCP: ${duration}ms`,
      level: 'info',
    });
  };

  return { markReady, markFirstContentfulPaint };
}
```

Usage in a screen:

```typescript
// src/screens/ProductScreen.tsx

import React, { useEffect, useState } from 'react';
import { useScreenPerformance } from '../hooks/useScreenPerformance';

function ProductScreen({ route }: ProductScreenProps) {
  const { productId } = route.params;
  const { markReady, markFirstContentfulPaint } = useScreenPerformance('ProductScreen');
  const [product, setProduct] = useState<Product | null>(null);
  const [reviews, setReviews] = useState<Review[]>([]);

  useEffect(() => {
    async function loadProduct() {
      // Load product data
      const productData = await api.getProduct(productId);
      setProduct(productData);

      // First contentful paint — product name and image are showing
      markFirstContentfulPaint();

      // Load secondary data
      const reviewData = await api.getReviews(productId);
      setReviews(reviewData);

      // Fully loaded — all data is rendered
      markReady();
    }

    loadProduct();
  }, [productId]);

  // ... render
}
```

### Firebase Performance Monitoring Custom Traces

If you're using Firebase, you can create custom traces that appear in the Firebase Performance dashboard:

```typescript
// src/monitoring/firebase-performance.ts

import perf from '@react-native-firebase/perf';

class FirebasePerformanceService {
  // Trace an async operation
  async traceAsync<T>(
    traceName: string,
    operation: () => Promise<T>,
    attributes?: Record<string, string>
  ): Promise<T> {
    const trace = await perf().startTrace(traceName);

    if (attributes) {
      Object.entries(attributes).forEach(([key, value]) => {
        trace.putAttribute(key, value);
      });
    }

    try {
      const result = await operation();
      trace.putAttribute('status', 'success');
      return result;
    } catch (error) {
      trace.putAttribute('status', 'error');
      trace.putAttribute('error_message', (error as Error).message);
      throw error;
    } finally {
      await trace.stop();
    }
  }

  // Trace a screen render
  async traceScreenRender(screenName: string): Promise<() => Promise<void>> {
    const trace = await perf().startTrace(`screen_render_${screenName}`);
    trace.putAttribute('screen', screenName);

    return async () => {
      await trace.stop();
    };
  }

  // Trace an API call
  async traceApiCall(
    endpoint: string,
    method: string
  ): Promise<{
    stop: (statusCode: number, responseSize: number) => Promise<void>;
  }> {
    const trace = await perf().startTrace('api_call');
    trace.putAttribute('endpoint', endpoint);
    trace.putAttribute('method', method);

    return {
      stop: async (statusCode: number, responseSize: number) => {
        trace.putAttribute('status_code', statusCode.toString());
        trace.putMetric('response_size_bytes', responseSize);
        await trace.stop();
      },
    };
  }

  // Trace an image load
  async traceImageLoad(
    imageUrl: string,
    source: 'network' | 'cache' | 'local'
  ): Promise<() => Promise<void>> {
    const trace = await perf().startTrace('image_load');
    trace.putAttribute('source', source);
    trace.putAttribute('url_hash', hashUrl(imageUrl));

    return async () => {
      await trace.stop();
    };
  }
}

function hashUrl(url: string): string {
  // Simple hash for URL identification without exposing full URL
  let hash = 0;
  for (let i = 0; i < url.length; i++) {
    const char = url.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash |= 0;
  }
  return Math.abs(hash).toString(36);
}

export const firebasePerf = new FirebasePerformanceService();
```

Usage:

```typescript
// In your API layer
async function fetchProduct(productId: string): Promise<Product> {
  return firebasePerf.traceAsync(
    'fetch_product',
    async () => {
      const response = await fetch(`/api/products/${productId}`);
      return response.json();
    },
    { product_id: productId }
  );
}

// In your screen
function ProductScreen() {
  useEffect(() => {
    let stopTrace: (() => Promise<void>) | null = null;

    async function init() {
      stopTrace = await firebasePerf.traceScreenRender('ProductScreen');
      await loadData();
      await stopTrace?.();
    }

    init();

    return () => {
      // Ensure trace is stopped even if component unmounts early
      stopTrace?.();
    };
  }, []);
}
```

### Sentry Performance Monitoring: The Full Picture

Sentry gives you a more complete picture with distributed tracing. You can trace a user action from the button tap through the API call to the backend and back:

```typescript
// src/monitoring/sentry-performance.ts

import * as Sentry from '@sentry/react-native';

// Trace a full user flow
async function traceCheckoutFlow(cartId: string): Promise<OrderConfirmation> {
  return Sentry.startSpan(
    {
      name: 'checkout.flow',
      op: 'user.action',
      attributes: { 'cart.id': cartId },
    },
    async () => {
      // Step 1: Validate cart
      const cart = await Sentry.startSpan(
        { name: 'checkout.validate_cart', op: 'function' },
        () => validateCart(cartId)
      );

      // Step 2: Process payment
      const paymentResult = await Sentry.startSpan(
        { name: 'checkout.process_payment', op: 'http.client' },
        () => processPayment(cart)
      );

      // Step 3: Create order
      const order = await Sentry.startSpan(
        { name: 'checkout.create_order', op: 'http.client' },
        () => createOrder(cart, paymentResult)
      );

      // Step 4: Send confirmation
      await Sentry.startSpan(
        { name: 'checkout.send_confirmation', op: 'function' },
        () => sendConfirmationNotification(order)
      );

      return order;
    }
  );
}
```

In Sentry's Performance dashboard, this shows up as a waterfall trace:

```
checkout.flow                [==============================================] 2450ms
  checkout.validate_cart     [=====]                                          150ms
  checkout.process_payment         [================]                         850ms
  checkout.create_order                              [========]               420ms
  checkout.send_confirmation                                   [===]          130ms
                                                                    [gap]     900ms ???
```

Wait — there's a 900ms gap between create_order ending and the total span ending. That's suspicious. Without this trace, you'd never know that gap exists. Now you can investigate what's happening in those 900ms (probably UI re-rendering with the new order data).

### Frame Rate Monitoring

Frame drops are the bane of mobile apps. Here's how to track them:

```typescript
// src/monitoring/frame-rate.ts

import { FrameRateMonitor } from 'react-native-performance';
import * as Sentry from '@sentry/react-native';

class FrameRateTracker {
  private monitor: FrameRateMonitor | null = null;

  start(screenName: string): void {
    if (__DEV__) return; // Don't track in development

    this.monitor = new FrameRateMonitor();
    this.monitor.start();

    // Sample frame rate every 5 seconds
    this.monitor.onFrameRateUpdate((data) => {
      const { uiFps, jsFps } = data;

      // Alert if FPS drops significantly
      if (uiFps < 45 || jsFps < 45) {
        Sentry.addBreadcrumb({
          category: 'performance',
          message: `Low FPS on ${screenName}: UI=${uiFps}, JS=${jsFps}`,
          level: 'warning',
          data: { uiFps, jsFps, screen: screenName },
        });
      }

      // Track extremely low FPS as a performance issue
      if (uiFps < 30 || jsFps < 30) {
        Sentry.captureMessage(`Severe frame drop on ${screenName}`, {
          level: 'warning',
          tags: {
            screen: screenName,
            ui_fps: uiFps.toString(),
            js_fps: jsFps.toString(),
          },
        });
      }
    });
  }

  stop(): void {
    this.monitor?.stop();
    this.monitor = null;
  }
}

export const frameRateTracker = new FrameRateTracker();
```

### Performance Budgets

Set explicit performance budgets and alert when they're exceeded:

```typescript
// src/monitoring/performance-budgets.ts

import * as Sentry from '@sentry/react-native';

interface PerformanceBudget {
  screenLoad: number;    // ms
  apiResponse: number;   // ms
  imageLoad: number;     // ms
  coldStart: number;     // ms
}

const BUDGETS: Record<string, PerformanceBudget> = {
  default: {
    screenLoad: 500,
    apiResponse: 1000,
    imageLoad: 200,
    coldStart: 2000,
  },
  critical: {
    screenLoad: 300,
    apiResponse: 500,
    imageLoad: 100,
    coldStart: 1500,
  },
};

export function checkBudget(
  metric: keyof PerformanceBudget,
  duration: number,
  context: string,
  budgetLevel: 'default' | 'critical' = 'default'
): void {
  const budget = BUDGETS[budgetLevel][metric];

  if (duration > budget) {
    Sentry.addBreadcrumb({
      category: 'performance.budget',
      message: `Budget exceeded: ${context} took ${duration}ms (budget: ${budget}ms)`,
      level: 'warning',
      data: { metric, duration, budget, context },
    });

    // If it's more than 2x over budget, capture as an issue
    if (duration > budget * 2) {
      Sentry.captureMessage(
        `Performance budget severely exceeded: ${context}`,
        {
          level: 'warning',
          tags: {
            metric,
            budget_level: budgetLevel,
          },
          extra: {
            duration,
            budget,
            overage_percentage: ((duration / budget - 1) * 100).toFixed(0) + '%',
          },
        }
      );
    }
  }
}
```

---

## Part 8: Real User Monitoring (RUM) — The Complete Picture

### What RUM Is and Why It Matters

Real User Monitoring measures the actual experience of real users in production. It's different from synthetic monitoring (which runs tests from bots) in a fundamental way: RUM tells you what's actually happening, not what could happen.

Think of it this way:

- **Synthetic monitoring** is like a restaurant health inspection. It happens on a schedule, with known criteria, in controlled conditions.
- **RUM** is like having a camera in the dining room. You see every meal, every wait time, every complaint in real time.

Both are valuable, but RUM is closer to truth.

### Sentry Session Tracking

Sentry's session tracking gives you RUM-like capabilities:

```typescript
// Sentry automatically tracks sessions, but you can enrich them

import * as Sentry from '@sentry/react-native';

// Set user context for session tracking
function onUserLogin(user: User) {
  Sentry.setUser({
    id: user.id,
    segment: user.subscriptionTier, // 'free', 'premium', 'enterprise'
  });

  Sentry.setTag('user_segment', user.subscriptionTier);
  Sentry.setTag('app_variant', getExperimentVariant());
}

// Track key user flows
function trackUserFlow(flowName: string, step: string) {
  Sentry.addBreadcrumb({
    category: 'user.flow',
    message: `${flowName}: ${step}`,
    level: 'info',
  });
}

// Example: tracking the checkout flow
function CheckoutScreen() {
  useEffect(() => {
    trackUserFlow('checkout', 'started');
  }, []);

  const handlePayment = async () => {
    trackUserFlow('checkout', 'payment_initiated');
    try {
      await processPayment();
      trackUserFlow('checkout', 'payment_succeeded');
    } catch (error) {
      trackUserFlow('checkout', 'payment_failed');
      Sentry.captureException(error);
    }
  };

  const handleConfirm = async () => {
    trackUserFlow('checkout', 'order_confirmed');
  };

  // ...
}
```

### Session Replay for Mobile

Sentry's mobile session replay is one of the most powerful debugging tools available. It records what the user sees (with privacy controls) so you can watch exactly what led to a crash or error:

```typescript
// Session replay configuration
Sentry.init({
  dsn: 'your-dsn',
  integrations: [
    Sentry.mobileReplayIntegration({
      // Privacy settings
      maskAllText: true,        // Mask all text content
      maskAllImages: false,      // Don't mask images (adjust for your needs)
      maskAllVectors: false,     // Don't mask vector graphics

      // Quality settings
      quality: 'medium',         // 'low' | 'medium' | 'high'
    }),
  ],

  // Record 10% of all sessions
  replaysSessionSampleRate: 0.1,

  // Record 100% of sessions that have an error
  replaysOnErrorSampleRate: 1.0,
});
```

The `replaysOnErrorSampleRate: 1.0` is crucial. It means every time a user hits an error, you get a recording of what they were doing. This is invaluable for reproducing bugs that are hard to trigger in development.

### Datadog RUM for Mobile

If your organization is already using Datadog, their mobile RUM product provides deep integration with your backend monitoring:

```typescript
// Setting up Datadog RUM for React Native
import {
  DdSdkReactNative,
  DdSdkReactNativeConfiguration,
  TrackingConsent,
} from '@datadog/mobile-react-native';

const config = new DdSdkReactNativeConfiguration(
  'your-client-token',
  'production',
  'your-rum-application-id',
  true, // Track interactions
  true, // Track XHR resources
  true  // Track errors
);

config.site = 'US1'; // or 'EU1', 'US3', etc.
config.nativeCrashReportEnabled = true;
config.sessionSampleRate = 100; // percentage of sessions to track
config.resourceTracingSamplingRate = 100;
config.firstPartyHosts = ['api.yourapp.com'];
config.telemetrySampleRate = 20;

await DdSdkReactNative.initialize(config);

// Track user
DdSdkReactNative.setUser({
  id: 'user-123',
  name: 'John Doe',
  extraInfo: {
    subscription: 'premium',
  },
});

// Track custom actions
import { DdRum } from '@datadog/mobile-react-native';

DdRum.addAction('tap', 'checkout_button', {
  cart_value: '149.99',
  item_count: '3',
});

// Track custom timing
DdRum.startResource(
  'fetch-products',
  'GET',
  'https://api.yourapp.com/products'
);
// ... after request completes
DdRum.stopResource(
  'fetch-products',
  200,
  'application/json',
  responseSize
);
```

Datadog's advantage is correlation. You can trace a user action in the mobile app through your API gateway, into your microservices, and back. If a user reports "the app was slow," you can find their exact session, see the exact API calls they made, and trace those calls through your entire backend infrastructure.

### Synthetic vs RUM: When to Use Each

| Aspect | Synthetic Monitoring | RUM |
|--------|---------------------|-----|
| **What it tests** | Known scenarios | All user behavior |
| **When it runs** | On a schedule | All the time |
| **Device coverage** | Your test devices | All user devices |
| **Network conditions** | Controlled | Real-world |
| **Cost** | Lower | Higher (proportional to traffic) |
| **Best for** | Regression detection, SLO validation | User experience measurement, debugging |
| **Blind spots** | Doesn't catch edge cases users hit | Doesn't test features nobody uses yet |

The ideal setup uses both:
- Synthetic tests run in CI and on a schedule to catch regressions before users hit them
- RUM runs in production to measure the actual user experience

### Building a RUM Dashboard

Here's what a useful RUM dashboard looks like:

```
+------------------------------------------------------------------+
|  REAL USER MONITORING DASHBOARD                                    |
+------------------------------------------------------------------+
|                                                                    |
|  Active Sessions: 14,523          Error Rate: 0.8%                |
|  Avg Session Duration: 8m 42s     Crash Rate: 0.03%              |
|                                                                    |
|  ---- Screen Load Times (p50 / p95 / p99) ----                    |
|  HomeScreen:     180ms / 450ms / 1200ms                           |
|  ProductScreen:  220ms / 580ms / 1500ms                           |
|  CartScreen:     150ms / 350ms / 800ms                            |
|  CheckoutScreen: 280ms / 650ms / 1800ms   [!]                    |
|  ProfileScreen:  120ms / 280ms / 600ms                            |
|                                                                    |
|  ---- API Performance (p50 / p95) ----                            |
|  GET /products:      85ms / 340ms                                 |
|  POST /orders:       220ms / 890ms                                |
|  GET /user/profile:  45ms / 180ms                                 |
|  GET /feed:          150ms / 620ms   [!]                          |
|                                                                    |
|  ---- Device Distribution ----                                    |
|  iOS 18:    42%    |  iPhone 16 Pro:  18%                         |
|  iOS 17:    35%    |  iPhone 15:      15%                         |
|  Android 15: 12%   |  Galaxy S24:     8%                          |
|  Android 14:  8%   |  Pixel 9:        5%                          |
|  Other:      3%    |  Other:          54%                         |
|                                                                    |
|  ---- Network Conditions ----                                     |
|  4G/LTE:   62%     | Fast (>10Mbps):    48%                      |
|  WiFi:     31%     | Medium (1-10Mbps): 35%                      |
|  5G:        5%     | Slow (<1Mbps):     17%  [!!]                |
|  3G:        2%     |                                              |
+------------------------------------------------------------------+
```

The `[!]` markers flag metrics that are above your performance budgets. The `[!!]` next to slow network connections tells you that 17% of your users are on connections under 1Mbps — that's significant and should inform your optimization strategy.

---

## Part 9: Dashboards and Alerts — Your Early Warning System

### The Four Golden Signals for Mobile

Borrowed from Google's SRE book and adapted for mobile, these are the four metrics that should be on every mobile team's primary dashboard:

1. **Crash Rate** — Percentage of sessions that crash. Target: < 0.1%
2. **ANR Rate** — Percentage of sessions with ANRs (Android). Target: < 0.2%
3. **Cold Start Time** — Time to first interactive screen. Target: < 2s (p95)
4. **Network Error Rate** — Percentage of API calls that fail. Target: < 1%

If all four of these are green, your app is probably healthy. If any one of them goes red, you have a problem worth investigating immediately.

### What to Put on Your Dashboard

Here's my recommended dashboard layout, built from experience across multiple production mobile apps:

**Tier 1: The Vitals (visible at all times)**

```
Crash-Free Rate:     99.92%  [target: >99.9%]
ANR-Free Rate:       99.84%  [target: >99.8%]
Cold Start (p95):    1.8s    [target: <2.0s]
API Error Rate:      0.6%    [target: <1.0%]
```

**Tier 2: Version Health (check after every release)**

```
Version   | Crash-Free | ANR-Free | Cold Start | Adoption
----------|------------|----------|------------|----------
v3.2.1    | 99.94%     | 99.87%   | 1.7s       | 42%
v3.2.0    | 99.91%     | 99.83%   | 1.8s       | 38%
v3.1.9    | 99.89%     | 99.80%   | 1.9s       | 15%
v3.1.8    | 99.92%     | 99.85%   | 1.8s       | 5%
```

**Tier 3: Deep Dive (investigate during triage)**

```
Top Crashes:
  1. NullPointerException in CartScreen — 142 events, 89 users
  2. NetworkTimeoutError in PaymentService — 87 events, 72 users
  3. OutOfMemoryError in ImageGallery — 34 events, 28 users

Top Slow Screens (p95):
  1. SearchResults — 2.1s (budget: 500ms) [3.2x over]
  2. OrderHistory — 1.4s (budget: 500ms) [1.8x over]
  3. ProductDetail — 680ms (budget: 500ms) [0.36x over]

Top Failing Endpoints:
  1. POST /api/orders — 3.2% error rate
  2. GET /api/search — 1.8% error rate
  3. GET /api/feed — 1.1% error rate
```

### Setting Up Alerts

Alerts should follow the "if it pages someone at 2 AM, it should be worth waking up for" principle. Here's my recommended alert configuration:

#### Critical Alerts (Page On-Call Immediately)

```yaml
# Sentry Alert Rules

- name: "Crash Rate Spike"
  trigger: "crash_free_rate < 99.5% over 15 minutes"
  action: "page_oncall"
  channel: "#mobile-critical"
  description: >
    Crash-free rate dropped below 99.5%. This affects
    approximately 5 in 1000 user sessions.

- name: "New Fatal Crash (High Volume)"
  trigger: "new_issue AND events_count > 100 in 30 minutes"
  action: "page_oncall"
  channel: "#mobile-critical"
  description: >
    A new crash type is affecting more than 100 sessions
    in 30 minutes. Likely a regression in the latest release.

- name: "API Availability Down"
  trigger: "api_error_rate > 10% for 5 minutes"
  action: "page_oncall"
  channel: "#mobile-critical"
  description: >
    More than 10% of API calls are failing. Users are likely
    unable to use core functionality.
```

#### High Priority Alerts (Slack Notification, Investigate Within 1 Hour)

```yaml
- name: "Crash Rate Elevated"
  trigger: "crash_free_rate < 99.8% over 1 hour"
  action: "slack_notification"
  channel: "#mobile-alerts"
  description: >
    Crash-free rate is below 99.8% for the past hour.
    Investigate the top crashes for this period.

- name: "ANR Rate Elevated"
  trigger: "anr_free_rate < 99.5% over 1 hour"
  action: "slack_notification"
  channel: "#mobile-alerts"
  description: >
    ANR rate is elevated. Check for JS thread blocking
    or heavy main thread operations.

- name: "Cold Start Regression"
  trigger: "cold_start_p95 > 3000ms for 30 minutes"
  action: "slack_notification"
  channel: "#mobile-alerts"
  description: >
    Cold start time has regressed significantly. Check
    for new initialization code or heavy startup operations.

- name: "New Error Cluster"
  trigger: "new_issue AND events_count > 50 in 1 hour"
  action: "slack_notification"
  channel: "#mobile-alerts"
  description: >
    A new error type is appearing frequently. May indicate
    a regression or a new edge case.
```

#### Low Priority Alerts (Daily Digest)

```yaml
- name: "Performance Budget Exceeded"
  trigger: "any_screen_p95 > budget * 1.5 over 24 hours"
  action: "daily_digest"
  channel: "#mobile-performance"
  description: >
    Screen load times are exceeding performance budgets.
    Review and optimize during the next sprint.

- name: "Memory Usage Elevated"
  trigger: "avg_memory_usage > 250MB over 24 hours"
  action: "daily_digest"
  channel: "#mobile-performance"
  description: >
    Average memory usage is high. Risk of OOM kills on
    low-memory devices.

- name: "Network Retry Rate High"
  trigger: "retry_rate > 5% over 24 hours"
  action: "daily_digest"
  channel: "#mobile-reliability"
  description: >
    More than 5% of network requests are being retried.
    Check for timeout issues or server-side problems.
```

### Configuring Sentry Alerts

Here's how to set up these alerts in Sentry:

```typescript
// This is configured in the Sentry web UI, but here's the equivalent
// configuration using Sentry's API for reproducibility

// Alert Rule: Crash Rate Spike
const crashRateAlert = {
  name: 'Crash Rate Spike - Critical',
  dataset: 'metrics',
  query: '',
  aggregate: 'percentage(sessions_crashed, sessions) AS crash_rate',
  timeWindow: 15, // minutes
  triggers: [
    {
      label: 'critical',
      alertThreshold: 0.5, // 0.5% crash rate (99.5% crash-free)
      actions: [
        {
          type: 'slack',
          targetIdentifier: '#mobile-critical',
          targetType: 'specific',
        },
        {
          type: 'pagerduty',
          targetIdentifier: 'mobile-oncall',
          severity: 'critical',
        },
      ],
    },
    {
      label: 'warning',
      alertThreshold: 0.2, // 0.2% crash rate (99.8% crash-free)
      actions: [
        {
          type: 'slack',
          targetIdentifier: '#mobile-alerts',
          targetType: 'specific',
        },
      ],
    },
  ],
};
```

### PagerDuty / Opsgenie Integration

For critical alerts that need to wake someone up:

**Sentry + PagerDuty:**

1. In PagerDuty, create a service for your mobile app
2. Get the integration key
3. In Sentry, go to Settings > Integrations > PagerDuty
4. Add the integration and configure alert rules to send to your PagerDuty service

**Sentry + Opsgenie:**

```
Sentry Settings > Integrations > Opsgenie > Configure

Map Sentry alert levels to Opsgenie priorities:
  - Critical → P1 (pages immediately)
  - High → P2 (notifies within 15 minutes)
  - Medium → P3 (notifies within 1 hour)
  - Low → P4 (adds to backlog)
```

**Firebase Alerts:**

Firebase has a simpler alert system. In the Firebase Console:

1. Go to Crashlytics
2. Click the bell icon on any issue to subscribe to notifications
3. Set up velocity alerts: "Alert me if this issue affects more than X users in Y hours"

Firebase also supports:
- Email notifications
- Slack integration (via Firebase Extensions)
- Google Cloud Pub/Sub (for custom integrations)

### The On-Call Playbook

When an alert fires, the on-call engineer should follow this playbook:

```
MOBILE INCIDENT RESPONSE PLAYBOOK

1. ACKNOWLEDGE (within 5 minutes)
   - Acknowledge the alert in PagerDuty/Opsgenie
   - Post in #mobile-incidents: "Investigating [alert name]"

2. ASSESS (within 15 minutes)
   - Open Sentry/Crashlytics dashboard
   - Determine scope: How many users? Which versions? Which platforms?
   - Determine severity:
     - P1: >1% of users affected, core functionality broken
     - P2: 0.1-1% of users affected, or non-core functionality broken
     - P3: <0.1% of users affected, workaround available

3. MITIGATE (within 30 minutes for P1)
   - Can we disable the feature via feature flag?
   - Can we halt the rollout?
   - Do we need to roll back to the previous version?

4. COMMUNICATE (throughout)
   - Update #mobile-incidents every 30 minutes
   - For P1: notify engineering leadership and product
   - For P1/P2: update the status page if applicable

5. FIX
   - Identify root cause
   - Write and test the fix
   - Ship via hotfix process (expedited review + staged rollout)

6. POST-MORTEM (within 48 hours)
   - What happened?
   - Why wasn't it caught before production?
   - What will we change to prevent this in the future?
   - Assign action items with owners and deadlines
```

---

## Part 10: Maintaining 99.9% Crash-Free — The Operational Playbook

### This Is Not a Tool Problem, It's a Culture Problem

You can set up the best monitoring in the world, but if nobody looks at the dashboards, nobody triages the crashes, and nobody owns the crash-free rate, it doesn't matter.

Maintaining 99.9% crash-free rate is an operational practice, not a technical achievement. It requires discipline, processes, and cultural buy-in.

Here's what elite mobile teams do.

### Weekly Crash Triage

Every week, your mobile team should spend 30-60 minutes triaging crashes. Here's the format:

**Participants:** Mobile engineers (required), PM (optional), QA (optional)

**Agenda:**

```
1. DASHBOARD REVIEW (5 minutes)
   - Current crash-free rate vs target
   - Trend: improving, stable, or degrading?
   - Any alerts that fired this week?

2. TOP 10 CRASHES REVIEW (20 minutes)
   - For each of the top 10 crashes by frequency:
     - Is this already assigned to someone?
     - If new: assign an owner and set a timeline
     - If existing: what's the status of the fix?
     - Any new context or reproduction steps?

3. RELEASE HEALTH (10 minutes)
   - How is the latest release performing?
   - Any regressions compared to the previous release?
   - Are there platform-specific issues?

4. PERFORMANCE REVIEW (10 minutes)
   - Any screen load times above budget?
   - Any API endpoints with elevated error rates?
   - Any new ANR patterns?

5. ACTION ITEMS (5 minutes)
   - Summarize assignments
   - Set deadlines
   - Identify any blockers
```

### Crash Severity Classification

Not all crashes are equal. Use this framework:

| Severity | Criteria | Response Time | Example |
|----------|----------|---------------|---------|
| **S1 - Critical** | >0.1% of sessions, core flow | Same day | Crash on app launch |
| **S2 - High** | >0.01% of sessions, or important flow | Within 1 sprint | Crash during checkout |
| **S3 - Medium** | <0.01% of sessions, non-critical flow | Within 2 sprints | Crash in settings page |
| **S4 - Low** | Rare, edge case, workaround available | Backlog | Crash on obscure device model |

### Regression Detection

Automated regression detection is essential for catching problems early. Here's how to implement it:

```typescript
// scripts/check-crash-regression.ts
// Run this in CI after a canary release

import * as Sentry from '@sentry/node';

interface ReleaseHealth {
  release: string;
  crashFreeRate: number;
  sessionCount: number;
}

async function checkForRegression(
  currentRelease: string,
  previousRelease: string
): Promise<{ passed: boolean; message: string }> {
  const current = await getReleaseHealth(currentRelease);
  const previous = await getReleaseHealth(previousRelease);

  // Need minimum session count for statistical significance
  if (current.sessionCount < 1000) {
    return {
      passed: true,
      message: `Insufficient data (${current.sessionCount} sessions). Need 1000+ for comparison.`,
    };
  }

  const crashRateDelta = previous.crashFreeRate - current.crashFreeRate;

  // If crash-free rate dropped by more than 0.1%, flag it
  if (crashRateDelta > 0.1) {
    return {
      passed: false,
      message: `REGRESSION DETECTED: Crash-free rate dropped from ${previous.crashFreeRate}% to ${current.crashFreeRate}% (delta: ${crashRateDelta.toFixed(2)}%)`,
    };
  }

  // If crash-free rate dropped by more than 0.05%, warn
  if (crashRateDelta > 0.05) {
    return {
      passed: true,
      message: `WARNING: Slight crash-free rate decrease from ${previous.crashFreeRate}% to ${current.crashFreeRate}% (delta: ${crashRateDelta.toFixed(2)}%). Monitor closely.`,
    };
  }

  return {
    passed: true,
    message: `OK: Crash-free rate stable at ${current.crashFreeRate}% (previous: ${previous.crashFreeRate}%)`,
  };
}

async function getReleaseHealth(release: string): Promise<ReleaseHealth> {
  // Using Sentry API to get release health data
  const response = await fetch(
    `https://sentry.io/api/0/organizations/your-org/releases/${encodeURIComponent(release)}/health/`,
    {
      headers: {
        Authorization: `Bearer ${process.env.SENTRY_API_TOKEN}`,
      },
    }
  );

  const data = await response.json();

  return {
    release,
    crashFreeRate: data.sessionsCrashFreeRate * 100,
    sessionCount: data.sessionsTotal,
  };
}
```

### Canary Releases

Canary releases are your safety net. Instead of rolling out a new version to all users at once, you roll it out to a small percentage first and monitor:

```
Release Strategy:
  Hour 0:   Release to 1% of users (canary)
  Hour 24:  If crash-free rate > 99.9%, expand to 10%
  Hour 48:  If still > 99.9%, expand to 50%
  Hour 72:  If still > 99.9%, expand to 100%

  At any point: If crash-free rate < 99.5%, HALT rollout
  At any point: If crash-free rate < 99.0%, ROLL BACK
```

On Android, you can implement this directly through Google Play Console's staged rollouts. On iOS, there's no built-in staged rollout (Apple releases to everyone), but you can use feature flags to control which users see new functionality:

```typescript
// Using feature flags for canary releases on iOS

import remoteConfig from '@react-native-firebase/remote-config';

async function shouldShowNewFeature(): Promise<boolean> {
  await remoteConfig().fetchAndActivate();

  const rolloutPercentage = remoteConfig().getValue('new_checkout_rollout').asNumber();
  const userBucket = getUserBucket(); // Consistent hash of user ID to 0-100

  return userBucket < rolloutPercentage;
}

// In Firebase Remote Config console:
// new_checkout_rollout: 1    (1% canary)
// new_checkout_rollout: 10   (10% rollout)
// new_checkout_rollout: 100  (full rollout)
```

### Staged Rollouts on Google Play

```
Google Play Console > Release Management > Production

Staged Rollout:
  Stage 1: 1% — Monitor for 24 hours
  Stage 2: 5% — Monitor for 24 hours
  Stage 3: 20% — Monitor for 24 hours
  Stage 4: 50% — Monitor for 24 hours
  Stage 5: 100% — Full release

Monitoring at each stage:
  - Crash-free rate (Crashlytics + Sentry)
  - ANR rate (Google Play Console)
  - User reviews (Google Play Console)
  - Custom metrics (your dashboards)
```

### The Pre-Release Checklist

Before any release goes out, run through this checklist:

```markdown
## Pre-Release Checklist

### Monitoring
- [ ] Source maps uploaded and verified
- [ ] dSYMs uploaded (iOS) or ProGuard mappings uploaded (Android)
- [ ] Release created in Sentry with correct version string
- [ ] Crash-free rate baseline recorded for comparison
- [ ] Alert thresholds reviewed and appropriate for this release

### Testing
- [ ] E2E tests passing on both platforms
- [ ] Manual smoke test on physical devices (at least 1 iOS, 1 Android)
- [ ] Performance benchmarks within budget
- [ ] Memory leak tests passing

### Rollout
- [ ] Staged rollout configured (Android)
- [ ] Feature flags set to appropriate levels (iOS)
- [ ] Rollback plan documented
- [ ] On-call engineer identified and available for 48 hours post-release

### Communication
- [ ] Release notes drafted
- [ ] Team notified of release timeline
- [ ] Support team briefed on known issues and workarounds
```

### Post-Release Monitoring Cadence

After a release ships, follow this monitoring cadence:

```
Hour 0-1:   Check crash-free rate every 15 minutes
Hour 1-4:   Check crash-free rate every 30 minutes
Hour 4-24:  Check crash-free rate every 2 hours
Day 1-3:    Check crash-free rate twice daily (morning + evening)
Day 3-7:    Check crash-free rate once daily
Day 7+:     Weekly crash triage covers it
```

This is aggressive for the first few hours, then relaxes as confidence grows. The key insight is that most regressions show up within the first 24 hours.

### When Things Go Wrong: The Hotfix Process

When you need to ship an emergency fix:

```
HOTFIX PROCESS

1. BRANCH
   - Create hotfix branch from the release tag (not from main)
   - git checkout -b hotfix/crash-in-cart-screen v3.2.1

2. FIX
   - Make the minimal change needed
   - No feature changes, no refactoring, no "while I'm here" fixes
   - Just the fix

3. TEST
   - Run the full test suite
   - Manual test on physical device
   - Verify the specific crash is fixed

4. BUILD
   - Increment the patch version (3.2.1 → 3.2.2)
   - Build with the same signing config as the original release
   - Upload source maps and debug symbols

5. REVIEW
   - Expedited code review (1 reviewer minimum)
   - Review focuses on: correctness, no regressions, minimal change

6. DEPLOY
   - Android: Push to 10% immediately, expand to 100% within 4 hours if stable
   - iOS: Submit for expedited review (Apple supports this for critical fixes)

7. VERIFY
   - Monitor crash-free rate for the hotfix release
   - Confirm the specific crash is no longer appearing
   - Compare crash-free rate to the pre-regression baseline

8. MERGE BACK
   - Cherry-pick the fix into main
   - Ensure the fix is in the next regular release
```

### Building a Culture of Monitoring

The hardest part isn't the technical setup. It's getting your team to actually use it. Here are practices that work:

**1. Make the Dashboard Visible**

Put your mobile health dashboard on a TV in the office (or in a Slack channel that posts daily). When the numbers are visible, people pay attention.

**2. Celebrate Improvements**

When the crash-free rate goes from 99.85% to 99.92%, celebrate it. Recognize the engineer who fixed the top crash. Make crash reduction a valued achievement, not just a chore.

**3. Include Monitoring in Sprint Planning**

Allocate 10-15% of each sprint to monitoring-related work: fixing crashes, improving performance, adding instrumentation. Don't let it be "we'll get to it when we have time" because you never will.

**4. Make It Part of the Definition of Done**

A feature isn't done when the code is merged. It's done when:
- The code is in production
- Monitoring is in place
- No new crashes or performance regressions are detected
- The feature works for 48 hours without issues

**5. Blameless Post-Mortems**

When a crash makes it to production, don't ask "who broke it?" Ask "why did our process fail to catch it?" Maybe the test coverage is insufficient. Maybe the monitoring alert thresholds are too lenient. Maybe the code review process missed something. Fix the process, not the person.

### The Monitoring Maturity Model

Where is your team on this scale?

**Level 0: Flying Blind**
- No crash reporting
- Errors discovered through user complaints
- "It works on my machine"

**Level 1: Basic Crash Reporting**
- Crashlytics or Sentry installed
- Crashes are logged but rarely investigated
- No alerts

**Level 2: Active Monitoring**
- Crash reporting + performance monitoring
- Alerts configured for critical issues
- Someone looks at crashes weekly
- Source maps uploaded

**Level 3: Proactive Monitoring**
- Full observability (errors + performance + RUM)
- Automated regression detection
- Weekly crash triage meetings
- Staged rollouts with monitoring gates
- Performance budgets enforced

**Level 4: Elite**
- Real-time dashboards visible to all engineers
- Automated canary analysis
- Crash budget tied to release decisions
- Session replay for every error session
- Custom business metric monitoring
- Post-mortems for every crash regression
- Monitoring coverage in CI/CD pipeline

Most teams are at Level 1 or 2. Getting to Level 3 is achievable within a quarter. Level 4 takes sustained investment but pays massive dividends.

---

## Putting It All Together: The Complete Setup

Here's a complete monitoring setup for a React Native app, from scratch:

```typescript
// src/monitoring/index.ts — The unified monitoring entry point

import * as Sentry from '@sentry/react-native';
import crashlytics from '@react-native-firebase/crashlytics';
import perf from '@react-native-firebase/perf';
import DeviceInfo from 'react-native-device-info';
import { Platform } from 'react-native';

// ============================================================
// CONFIGURATION
// ============================================================

const CONFIG = {
  sentry: {
    dsn: process.env.SENTRY_DSN || '',
    tracesSampleRate: __DEV__ ? 1.0 : 0.2,
    profilesSampleRate: __DEV__ ? 1.0 : 0.1,
    replaysSessionSampleRate: __DEV__ ? 0 : 0.1,
    replaysOnErrorSampleRate: __DEV__ ? 0 : 1.0,
  },
  crashlytics: {
    enabled: !__DEV__,
  },
};

// ============================================================
// INITIALIZATION
// ============================================================

export async function initMonitoring(): Promise<void> {
  const release = `${DeviceInfo.getBundleId()}@${DeviceInfo.getVersion()}+${DeviceInfo.getBuildNumber()}`;

  // Initialize Sentry
  Sentry.init({
    dsn: CONFIG.sentry.dsn,
    release,
    dist: DeviceInfo.getBuildNumber(),
    environment: __DEV__ ? 'development' : 'production',
    tracesSampleRate: CONFIG.sentry.tracesSampleRate,
    profilesSampleRate: CONFIG.sentry.profilesSampleRate,
    replaysSessionSampleRate: CONFIG.sentry.replaysSessionSampleRate,
    replaysOnErrorSampleRate: CONFIG.sentry.replaysOnErrorSampleRate,
    integrations: [
      Sentry.reactNativeTracingIntegration({
        enableHTTPTimings: true,
      }),
      Sentry.mobileReplayIntegration({
        maskAllText: true,
        maskAllImages: false,
      }),
    ],
    beforeSend(event) {
      if (__DEV__) return null;
      return event;
    },
  });

  // Initialize Crashlytics
  await crashlytics().setCrashlyticsCollectionEnabled(CONFIG.crashlytics.enabled);

  // Set common context
  const commonAttributes = {
    platform: Platform.OS,
    os_version: String(Platform.Version),
    app_version: DeviceInfo.getVersion(),
    build_number: DeviceInfo.getBuildNumber(),
    device_model: await DeviceInfo.getModel(),
  };

  await crashlytics().setAttributes(commonAttributes);
  Sentry.setContext('device_info', commonAttributes);

  console.log('[Monitoring] Initialized successfully');
}

// ============================================================
// USER IDENTIFICATION
// ============================================================

export async function identifyUser(
  userId: string,
  attributes?: Record<string, string>
): Promise<void> {
  Sentry.setUser({ id: userId, ...attributes });
  await crashlytics().setUserId(userId);

  if (attributes) {
    await crashlytics().setAttributes(attributes);
  }
}

export async function clearUser(): Promise<void> {
  Sentry.setUser(null);
  await crashlytics().setUserId('');
}

// ============================================================
// ERROR REPORTING
// ============================================================

export function captureError(
  error: Error,
  context?: {
    tags?: Record<string, string>;
    extra?: Record<string, unknown>;
    level?: 'fatal' | 'error' | 'warning' | 'info';
  }
): void {
  // Sentry
  Sentry.withScope((scope) => {
    if (context?.tags) {
      Object.entries(context.tags).forEach(([k, v]) => scope.setTag(k, v));
    }
    if (context?.extra) {
      Object.entries(context.extra).forEach(([k, v]) => scope.setExtra(k, v));
    }
    if (context?.level) {
      scope.setLevel(context.level);
    }
    Sentry.captureException(error);
  });

  // Crashlytics
  if (context?.tags) {
    Object.entries(context.tags).forEach(([k, v]) => {
      crashlytics().setAttribute(k, v);
    });
  }
  crashlytics().recordError(error);
}

// ============================================================
// BREADCRUMBS
// ============================================================

export function addBreadcrumb(
  message: string,
  category: string,
  data?: Record<string, string>,
  level: 'debug' | 'info' | 'warning' | 'error' = 'info'
): void {
  Sentry.addBreadcrumb({
    message,
    category,
    data,
    level,
  });

  crashlytics().log(`[${category}] ${message}`);
}

export function logNavigation(screenName: string): void {
  addBreadcrumb(`Navigated to ${screenName}`, 'navigation', {
    screen: screenName,
  });
}

export function logUserAction(action: string, data?: Record<string, string>): void {
  addBreadcrumb(action, 'user.action', data);
}

export function logApiCall(
  method: string,
  url: string,
  status?: number,
  duration?: number
): void {
  addBreadcrumb(
    `${method} ${url} ${status ? `→ ${status}` : ''} ${duration ? `(${duration}ms)` : ''}`,
    'http',
    {
      method,
      url,
      ...(status && { status: status.toString() }),
      ...(duration && { duration_ms: duration.toString() }),
    }
  );
}

// ============================================================
// PERFORMANCE
// ============================================================

export function startSpan<T>(
  name: string,
  op: string,
  callback: () => Promise<T> | T,
  attributes?: Record<string, string | number>
): Promise<T> {
  return Sentry.startSpan(
    {
      name,
      op,
      attributes,
    },
    callback
  );
}

export async function traceAsync<T>(
  name: string,
  operation: () => Promise<T>,
  attributes?: Record<string, string>
): Promise<T> {
  // Sentry span
  const result = await startSpan(name, 'function', operation, attributes);

  // Firebase trace (fire and forget)
  perf()
    .startTrace(name)
    .then(async (trace) => {
      if (attributes) {
        Object.entries(attributes).forEach(([k, v]) =>
          trace.putAttribute(k, v)
        );
      }
      await trace.stop();
    })
    .catch(() => {
      // Don't let perf monitoring errors affect the app
    });

  return result;
}

// ============================================================
// EXPORTS
// ============================================================

export const monitoring = {
  init: initMonitoring,
  identifyUser,
  clearUser,
  captureError,
  addBreadcrumb,
  logNavigation,
  logUserAction,
  logApiCall,
  startSpan,
  traceAsync,
};
```

Usage in your app:

```typescript
// index.js
import { monitoring } from './src/monitoring';
monitoring.init();

// After user logs in
monitoring.identifyUser('user-123', { tier: 'premium' });

// In API calls
monitoring.logApiCall('GET', '/api/products', 200, 145);

// In error handlers
try {
  await riskyOperation();
} catch (error) {
  monitoring.captureError(error as Error, {
    tags: { operation: 'risky_operation' },
    extra: { input: someData },
  });
}

// In navigation
monitoring.logNavigation('ProductScreen');

// For performance traces
const product = await monitoring.traceAsync(
  'load_product',
  () => api.getProduct(id),
  { product_id: id }
);
```

---

## Summary: The Monitoring Mindset

Let me leave you with the key takeaways:

1. **Monitoring is not optional.** It's as fundamental as testing. Ship monitoring with your first PR, not after your first incident.

2. **Use both Sentry and Crashlytics.** Sentry for JS errors, performance, and session replay. Crashlytics for native crashes and Firebase ecosystem integration. The cost of running both is negligible compared to the cost of missing a crash.

3. **Crash-free rate is your north star.** Track it daily. Alert on it. Make it visible to everyone. Target 99.9% and treat anything below that as a fire.

4. **Source maps are not optional.** Without them, your crash reports are useless. Automate the upload process so you never forget.

5. **Error boundaries are your last line of defense.** Place them strategically at the app, screen, and section level. Report errors from them. Never let a single component crash take down the whole app.

6. **Measure real user experience.** Synthetic tests are necessary but insufficient. RUM tells you what's actually happening on real devices with real network conditions.

7. **Set up alerts that mean something.** A dashboard nobody looks at is useless. An alert that fires for non-issues gets ignored. Configure alerts thoughtfully and respond to them seriously.

8. **Build operational practices around monitoring.** Weekly crash triage, staged rollouts, regression detection, post-mortems. The tools are easy; the discipline is hard.

9. **Invest in performance monitoring.** Crashes are dramatic, but slow apps kill you just as dead. Users don't write one-star reviews saying "the app is slow" — they just stop using it.

10. **Own it.** You wrote the code. You ship the monitoring. You watch the dashboards. You fix the crashes. That's what it means to be a 100x frontend architect.

---

**Next Chapter:** Chapter 21 — CI/CD Pipelines for Mobile: From Commit to App Store

**Prerequisites for next chapter:** Chapters 18 (Build Systems), 19 (Testing), and this chapter (Monitoring)
