<!--
  CHAPTER: 43
  TITLE: Analytics, Attribution & Data-Driven Development
  PART: V — Deployment & Operations
  PREREQS: Chapters 21, 25
  KEY_TOPICS: Firebase Analytics, Mixpanel, Amplitude, PostHog, Segment, analytics architecture, event taxonomy, user properties, funnels, cohorts, retention, attribution, install tracking, UTM, App Tracking Transparency, GDPR consent, privacy-first analytics
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 43: Analytics, Attribution & Data-Driven Development

> **Part V — Deployment & Operations** | Prerequisites: Chapters 21, 25 | Difficulty: Intermediate

> "Without data, you're just another person with an opinion." — W. Edwards Deming
>
> "With bad data, you're a person with misplaced confidence." — Every PM who shipped a feature based on a broken funnel

---

<details>
<summary><strong>TL;DR</strong></summary>

- Build your analytics architecture as a first-class system: typed events, consistent naming conventions (noun_verb like `screen_viewed`, `purchase_completed`), a centralized analytics service that abstracts providers, and consent management baked in from day one
- Choose your provider based on actual needs: Firebase Analytics for free Google ecosystem basics, Mixpanel/Amplitude for product analytics and funnels, PostHog for open-source self-hosted with session replay, Segment/RudderStack when you need to fan out to multiple destinations
- Implement the analytics abstraction layer pattern: one interface your app calls, multiple providers behind it; this lets you switch providers, add destinations, and respect consent without touching feature code
- Design your event taxonomy before writing a single `track()` call; a bad taxonomy is worse than no analytics because it creates false confidence in garbage data
- Attribution in the post-ATT world requires fundamentally different thinking: probabilistic modeling, SKAdNetwork, aggregated measurement, and honest acceptance that you will never have the per-user precision of 2019 again
- Privacy is not a compliance checkbox; it is an architecture decision that affects every layer from event collection to data storage to provider selection

</details>

Every app collects analytics. Almost no app collects analytics well.

The typical pattern goes like this: someone adds Firebase Analytics in week two. A few `logEvent` calls get sprinkled into button handlers. Six months later, the PM asks "what's our onboarding completion rate?" and the engineer stares at a dashboard full of events named `btn_click`, `screen1`, `thing_happened`, and `test_event_please_ignore`. Nobody can answer the question. The team spends two sprints re-instrumenting everything. Half the historical data is useless.

I have seen this pattern at startups and at companies with hundreds of engineers. The problem is never the analytics tool. The problem is that analytics is treated as an afterthought -- something you bolt on after the features are built -- instead of a system that requires the same architectural thinking as your state management or your API layer.

This chapter treats analytics as a system. We will design event taxonomies that scale, build abstraction layers that let you swap providers without touching feature code, implement attribution tracking in the post-ATT world, handle privacy regulations properly, and build the feedback loops that actually make your team data-driven instead of data-adjacent.

### In This Chapter
- Analytics architecture -- event models, identity resolution, the analytics service pattern
- Provider comparison -- Firebase Analytics, Mixpanel, Amplitude, PostHog, Segment
- Implementing analytics in React Native with Expo
- Event taxonomy design that scales
- Funnels, cohorts, and retention analysis
- Segment as a data pipeline (and alternatives)
- Attribution and install tracking
- App Tracking Transparency on iOS
- Privacy-first analytics and GDPR consent
- Web analytics with Vercel and Next.js
- Data-driven development and the build-measure-learn cycle
- Complete implementation with typed events and consent management

### Related Chapters
- [Ch 21: Firebase Console Mastery] -- Firebase Analytics setup and native SDK integration
- [Ch 23: Mobile Metrics That Matter] -- performance metrics that complement product analytics
- [Ch 22: Security & Data Protection] -- securing analytics data and API keys
- [Ch 25: Next.js App Router] -- server-side analytics in web applications
- [Ch 11: Deep Linking] -- deep link attribution and deferred deep links

---

## 1. ANALYTICS ARCHITECTURE

### 1.1 The Event-Based Tracking Model

Modern analytics is event-based. Every meaningful user interaction becomes a structured event with a name and a set of properties. This replaced the old pageview-centric model that made sense for websites in 2005 but falls apart for mobile apps and single-page applications.

An event has three components:

```
EVENT
├── Name:       "purchase_completed"
├── Properties:  { amount: 49.99, currency: "USD", item_count: 3, payment_method: "apple_pay" }
└── Context:     { user_id: "u_abc123", device: "iPhone 15", os: "iOS 18.2", app_version: "2.4.1", timestamp: "2026-04-07T10:30:00Z" }
```

**Name** identifies what happened. **Properties** describe the specifics. **Context** is automatically attached metadata about who, where, when, and on what device.

The power of event-based analytics is composability. Once you have a stream of well-structured events, you can:

- Build funnels: Did users who did A also do B?
- Measure retention: Are users who did X in week 1 coming back in week 4?
- Create cohorts: How do users who entered via campaign Y behave differently?
- Run A/B analysis: Did variant A produce more Z events than variant B?

All of this falls apart if your events are poorly named, inconsistently structured, or missing key properties. Which brings us to the most important decision you will make.

### 1.2 Event Taxonomy Design

Your event taxonomy is the schema of your analytics system. Get it wrong and every downstream analysis is compromised. Here are the rules:

**Rule 1: Use noun_verb (object_action) naming, not verb_noun (action_object).**

```
GOOD (noun_verb / object_action):
  screen_viewed
  button_tapped
  purchase_completed
  subscription_started
  onboarding_step_completed
  search_performed
  item_added_to_cart

BAD (verb_noun / action_object):
  viewed_screen
  tap_button
  complete_purchase
  startSubscription
  onboardingComplete
```

Why noun_verb? Because when you have hundreds of events, they sort and group naturally. All `purchase_*` events cluster together. All `screen_*` events cluster together. With verb_noun, you get `completed_purchase`, `completed_onboarding`, `completed_signup` scattered across your event list instead of `purchase_completed`, `onboarding_completed`, `signup_completed` which all sit next to their related events.

**Rule 2: Use snake_case. Always. No exceptions.**

Not camelCase, not PascalCase, not kebab-case. Snake_case is the universal standard across Firebase Analytics, Mixpanel, Amplitude, Segment, and PostHog. It also reads clearly in dashboards and SQL queries.

**Rule 3: Every event has a consistent set of required properties plus event-specific properties.**

```typescript
// Common properties attached to EVERY event
interface CommonEventProperties {
  screen_name: string;           // Where did this happen?
  session_id: string;            // Which session?
  app_version: string;           // Which release?
  platform: 'ios' | 'android' | 'web';
}

// Event-specific properties
interface PurchaseCompletedProperties extends CommonEventProperties {
  amount: number;
  currency: string;
  item_count: number;
  payment_method: string;
  is_first_purchase: boolean;
}
```

**Rule 4: Be specific enough to be useful, generic enough to be queryable.**

Bad: `button_tapped` with no properties (what button?)
Bad: `home_screen_top_left_blue_settings_gear_icon_tapped` (too specific, impossible to query)
Good: `button_tapped` with properties `{ button_name: "settings", screen_name: "home" }`

### 1.3 User Properties vs Event Properties

There are two kinds of properties in analytics, and confusing them is a common mistake:

**Event properties** describe the event. They change per event occurrence.
```typescript
// Each purchase has different properties
analytics.track('purchase_completed', {
  amount: 29.99,          // Different each time
  item_count: 2,          // Different each time
  payment_method: 'card', // May vary
});
```

**User properties** describe the user. They are set once and persist across all future events.
```typescript
// Set once, applies to all future events from this user
analytics.setUserProperties({
  account_type: 'premium',
  signup_date: '2026-01-15',
  company_size: '50-200',
  preferred_language: 'en',
});
```

User properties are incredibly powerful for segmentation. When you set `account_type: 'premium'`, every future event from that user carries that property. You can then filter any event -- `screen_viewed`, `feature_used`, `error_occurred` -- by account type without having to pass `account_type` as a property on every single event.

### 1.4 Identity Resolution

Users interact with your product across devices and across authentication states. Identity resolution is the process of stitching these interactions into a single user profile.

The typical flow:

```
Anonymous Session (device A)  ──→  Sign Up  ──→  Authenticated User
                                                        ↕
Anonymous Session (device B)  ──→  Log In   ──→  Same Authenticated User
```

Before login, the user has an anonymous ID (usually a device-generated UUID). After login, they have a known user ID. The `identify` call links these:

```typescript
// Before signup/login: events tracked with anonymous ID
analytics.track('screen_viewed', { screen_name: 'onboarding' });
analytics.track('signup_form_submitted', { method: 'email' });

// After signup: identify links anonymous → known
analytics.identify('user_12345', {
  email: 'user@example.com',
  name: 'Jane Doe',
  created_at: '2026-04-07',
});

// Now all previous anonymous events are attributed to user_12345
analytics.track('onboarding_completed', { steps_completed: 5 });
```

Different providers handle identity resolution differently:

| Provider | Anonymous ID | Identify Behavior | Cross-Device |
|----------|-------------|-------------------|--------------|
| **Firebase** | `app_instance_id` (auto) | `setUserId()` links to instance | Limited (Google Signals) |
| **Mixpanel** | `$device_id` (auto) | `identify()` merges profiles | Yes, via `$device_id` merge |
| **Amplitude** | `device_id` (auto) | `setUserId()` + `identify()` | Yes, merge by `user_id` |
| **PostHog** | Random UUID (auto) | `identify()` merges persons | Yes, alias-based merging |
| **Segment** | `anonymousId` (auto) | `identify()` passes to all destinations | Yes, via Segment identity graph |

**The critical rule:** Call `identify` at signup and at every login. If you only call it at signup, returning users who clear their app data or switch devices become new anonymous users and your retention numbers are wrong.

---

## 2. PROVIDER COMPARISON

### 2.1 Firebase Analytics (Google Analytics for Firebase)

**What it is:** Free analytics from Google, deeply integrated with the Firebase ecosystem. Automatically collects screen views, session starts, first opens, and other standard events. The data flows into BigQuery for custom analysis if you are on the Blaze plan.

**Strengths:**
- Free. Unlimited events, unlimited users. No pricing tiers.
- Native mobile SDK through `@react-native-firebase/analytics` -- events are batched and sent efficiently with minimal battery impact.
- Automatic events: `first_open`, `session_start`, `screen_view`, `app_update`, `os_update` without any code.
- Deep integration with other Firebase services (Remote Config, A/B Testing, Cloud Messaging audience targeting).
- BigQuery export for raw event data -- this is the killer feature. You get your raw event stream in a SQL-queryable warehouse at no additional cost (BigQuery charges for queries, but the export itself is free).
- Integration with Google Ads for attribution and audience building.

**Weaknesses:**
- The Firebase Console analytics UI is mediocre. Funnels are clunky. Cohort analysis is basic. If you want sophisticated product analytics, you will be writing BigQuery SQL.
- Data latency: events can take 24-48 hours to appear in the console. The real-time view shows only a subset.
- Limited event parameters: you can log up to 25 custom parameters per event, and the parameter name length is limited to 40 characters. Event names are limited to 500 unique names.
- No session replay. No heatmaps.
- You are sending all your data to Google. For some companies and some jurisdictions, this is a dealbreaker.

**Best for:** Teams that want free, reliable event tracking with BigQuery export for custom analysis. Apps already using Firebase for other services (Crashlytics, Remote Config, FCM).

### 2.2 Mixpanel

**What it is:** Product analytics platform focused on user behavior, funnels, and retention. One of the original event-based analytics platforms.

**Strengths:**
- Excellent funnel analysis with conversion rates, time-to-convert, and drop-off visualization.
- Powerful retention analysis with customizable retention events.
- Flows: see the actual paths users take through your app, not just predefined funnels.
- Signal: automated insights that surface statistically significant behavior patterns.
- JQL (JavaScript Query Language) for custom analysis beyond the UI.
- Group analytics for B2B (track accounts, not just users).
- Free tier: 20 million events per month. That is generous.

**Weaknesses:**
- Gets expensive fast at scale. The Growth plan starts around $28/month but scales with data volume.
- React Native SDK (`mixpanel-react-native`) works but is less polished than the Firebase native SDK.
- No session replay (though they have added screen capture as of 2025).
- Data governance features require higher-tier plans.

**Best for:** Product teams that live in funnels and retention charts. B2B apps with group/account analytics needs.

### 2.3 Amplitude

**What it is:** Behavioral analytics platform with strong cohort analysis, experimentation, and data management features.

**Strengths:**
- Behavioral cohorts: "users who did X but not Y within Z days" -- this is Amplitude's signature feature and it is genuinely best-in-class.
- Experiment (A/B testing) built into the platform with statistical rigor.
- Data taxonomy management with Govern -- enforce naming conventions, block bad events.
- Excellent retention analysis and lifecycle analysis.
- Free tier: 50 million events per month. Most generous free tier in the space.
- Session replay added in 2024-2025, now mature.
- React Native SDK (`@amplitude/analytics-react-native`) is solid.

**Weaknesses:**
- UI can be overwhelming. The learning curve is steeper than Mixpanel.
- Advanced features (Experiment, Audiences, CDP) are paid add-ons.
- Custom queries require their own query language (not SQL).
- Can be slow with very large datasets on the free tier.

**Best for:** Teams that want sophisticated cohort analysis and built-in experimentation. Companies that need the most generous free tier.

### 2.4 PostHog

**What it is:** Open-source product analytics platform. Can be self-hosted or used as a cloud service. Includes analytics, session replay, feature flags, A/B testing, and surveys in one platform.

**Strengths:**
- Open source (MIT license). You can self-host for complete data ownership.
- All-in-one: analytics, session replay, feature flags, A/B testing, surveys, and a data warehouse. This replaces 4-5 separate tools.
- Session replay works on mobile (React Native support added in 2025). This is a massive differentiator -- seeing exactly what the user did is worth a thousand event logs.
- SQL access to your data. Write queries, build dashboards, export.
- Privacy-friendly: self-host in your own infrastructure, data never leaves your servers.
- React Native SDK (`posthog-react-native`) with autocapture.
- Generous free tier on cloud: 1 million events/month, 5,000 session recordings/month.
- Feature flags included -- no separate LaunchDarkly subscription.

**Weaknesses:**
- Self-hosting requires infrastructure management (ClickHouse, Kafka, PostgreSQL). Not trivial.
- Cloud version costs add up if you have high volume (but still competitive).
- Mobile session replay is newer and less polished than web.
- Smaller ecosystem of integrations compared to Segment.
- Community support is good but not the 24/7 enterprise support of Amplitude or Mixpanel.

**Best for:** Teams that value data ownership, want an all-in-one platform, or operate in regulated industries where data cannot leave their infrastructure.

### 2.5 Segment

**What it is:** Not an analytics tool. Segment is a Customer Data Platform (CDP) -- a data pipeline that collects events from your app and routes them to multiple destinations (analytics, CRM, email, advertising, data warehouse).

**Strengths:**
- One SDK, many destinations. Instrument once, send to Firebase, Mixpanel, Amplitude, your data warehouse, your email tool, your CRM, and your ad platforms simultaneously.
- 400+ destination integrations. If a tool exists, Segment probably has an integration.
- Protocols: schema enforcement that validates events before they reach destinations. This is governance at the pipeline level.
- Identity resolution across sources (mobile, web, server).
- Replay: re-send historical data to new destinations. Add Amplitude six months from now and backfill it with six months of events.
- React Native SDK (`@segment/analytics-react-native`) with plugin architecture.

**Weaknesses:**
- Expensive. The free tier is 1,000 tracked users/month (not events -- users). The Team plan starts at $120/month for 10,000 users. At scale, you are looking at serious money.
- Adds a layer of indirection. Debugging "why isn't this event showing up in Amplitude?" now involves debugging Segment's pipeline too.
- The SDK adds another dependency and initialization step.
- You are trusting a third party with ALL your user data flowing through their servers.

**Best for:** Companies that use 3+ analytics/marketing destinations and want to instrument once. Teams that anticipate switching or adding providers.

### 2.6 Decision Matrix

| Criteria | Firebase | Mixpanel | Amplitude | PostHog | Segment |
|----------|----------|----------|-----------|---------|---------|
| **Cost at 10M events/mo** | Free | ~$300+ | Free | Free (cloud) | $$$$ |
| **Cost at 100M events/mo** | Free | $$$$ | $$$ | $$ (cloud) or self-host | $$$$$ |
| **Funnel analysis** | Basic | Excellent | Excellent | Good | N/A (destination dependent) |
| **Cohort analysis** | Basic | Good | Excellent | Good | N/A |
| **Session replay** | No | Limited | Yes | Yes (including mobile) | No |
| **Feature flags** | Via Remote Config | No | Yes (paid) | Yes (included) | No |
| **A/B testing** | Via Firebase A/B | No | Yes (paid) | Yes (included) | No |
| **Self-hostable** | No | No | No | Yes | No (RudderStack alternative) |
| **React Native SDK quality** | Excellent (native) | Good | Good | Good | Good |
| **Data warehouse export** | BigQuery (free) | BigQuery/S3 (paid) | Various (paid) | Built-in | Warehouse destinations |
| **Privacy/GDPR** | Data in Google | Data in Mixpanel | Data in Amplitude | Self-host option | Data flows through Segment |
| **Multi-destination** | No | No | No | No | Yes (this is the point) |

**My recommendation for most React Native teams:**

1. **Starting out, budget-conscious:** Firebase Analytics + PostHog free tier. Firebase for basics and BigQuery export, PostHog for funnels, session replay, and feature flags.

2. **Product-led growth, need deep analytics:** Amplitude free tier. The 50M events/month free tier with strong cohort analysis is hard to beat.

3. **Regulated industry, data sovereignty required:** PostHog self-hosted. Full control, no data leaving your infrastructure.

4. **Multiple marketing/analytics tools:** Segment + your analytics tool of choice. Pay the Segment tax to avoid instrumenting 5 different SDKs.

5. **Enterprise, money is not the constraint:** Segment + Amplitude + whatever else you need. Segment as the pipeline, Amplitude for product analytics, plus whatever marketing tools your growth team demands.

---

## 3. IMPLEMENTING ANALYTICS IN REACT NATIVE

### 3.1 Firebase Analytics Setup with Expo

If you followed Chapter 21, you already have `@react-native-firebase/app` configured. Adding analytics:

```bash
npx expo install @react-native-firebase/analytics
```

The config plugin handles native setup. No manual iOS/Android file editing.

```typescript
// app.config.ts
export default {
  // ... your existing config
  plugins: [
    '@react-native-firebase/app',
    '@react-native-firebase/analytics',
    // ... other plugins
  ],
};
```

Basic usage:

```typescript
// src/analytics/firebase.ts
import analytics from '@react-native-firebase/analytics';

// Log a custom event
await analytics().logEvent('purchase_completed', {
  amount: 49.99,
  currency: 'USD',
  item_count: 3,
  payment_method: 'apple_pay',
});

// Log a screen view
await analytics().logScreenView({
  screen_name: 'ProductDetail',
  screen_class: 'ProductDetailScreen',
});

// Set user properties
await analytics().setUserProperties({
  account_type: 'premium',
  preferred_language: 'en',
});

// Set user ID for identity resolution
await analytics().setUserId('user_12345');

// Log standard ecommerce events (Firebase has predefined schemas)
await analytics().logPurchase({
  value: 49.99,
  currency: 'USD',
  items: [
    {
      item_id: 'sku_001',
      item_name: 'Premium Widget',
      item_category: 'widgets',
      quantity: 1,
      price: 49.99,
    },
  ],
});
```

### 3.2 PostHog Setup in React Native

```bash
npx expo install posthog-react-native expo-file-system expo-application expo-device expo-localization
```

```typescript
// src/analytics/posthog.ts
import { PostHogProvider } from 'posthog-react-native';

// In your App root:
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <PostHogProvider
      apiKey="phc_your_api_key_here"
      options={{
        host: 'https://us.i.posthog.com', // or your self-hosted URL
        enableSessionReplay: true,
        sessionReplayConfig: {
          maskAllTextInputs: true,    // Privacy: mask text inputs in replay
          maskAllImages: false,
          captureNetworkTelemetry: true,
        },
        // Respect user consent (we will wire this up later)
        persistence: 'file',
      }}
    >
      {children}
    </PostHogProvider>
  );
}
```

Using PostHog in components:

```typescript
// src/hooks/useAnalytics.ts (PostHog-specific, before we build the abstraction)
import { usePostHog } from 'posthog-react-native';

export function useProductAnalytics() {
  const posthog = usePostHog();

  return {
    trackEvent: (name: string, properties?: Record<string, unknown>) => {
      posthog.capture(name, properties);
    },
    identifyUser: (userId: string, traits?: Record<string, unknown>) => {
      posthog.identify(userId, traits);
    },
    trackScreen: (screenName: string, properties?: Record<string, unknown>) => {
      posthog.screen(screenName, properties);
    },
    setUserProperties: (properties: Record<string, unknown>) => {
      posthog.identify(undefined, undefined, properties);
    },
    resetUser: () => {
      posthog.reset();
    },
    optOut: () => {
      posthog.optOut();
    },
    optIn: () => {
      posthog.optIn();
    },
  };
}
```

### 3.3 Automatic Screen Tracking with Expo Router

One of the most valuable automatic tracking capabilities is screen views. With Expo Router, you can track every screen transition without adding manual `logScreenView` calls to each screen:

```typescript
// src/analytics/screen-tracking.ts
import { usePathname, useSegments } from 'expo-router';
import { useEffect, useRef } from 'react';
import { analyticsService } from './analytics-service';

/**
 * Hook that automatically tracks screen views when the route changes.
 * Place this in your root layout component.
 */
export function useScreenTracking() {
  const pathname = usePathname();
  const segments = useSegments();
  const previousPath = useRef<string | null>(null);
  const screenStartTime = useRef<number>(Date.now());

  useEffect(() => {
    // Skip if the path has not changed
    if (pathname === previousPath.current) return;

    // Track time spent on previous screen
    if (previousPath.current !== null) {
      const timeSpent = Date.now() - screenStartTime.current;
      analyticsService.track('screen_exited', {
        screen_name: previousPath.current,
        time_spent_ms: timeSpent,
        time_spent_seconds: Math.round(timeSpent / 1000),
      });
    }

    // Track new screen view
    const screenName = formatScreenName(pathname, segments);
    analyticsService.track('screen_viewed', {
      screen_name: screenName,
      screen_path: pathname,
      screen_segments: segments.join('/'),
    });

    previousPath.current = pathname;
    screenStartTime.current = Date.now();
  }, [pathname, segments]);
}

/**
 * Convert Expo Router path to a readable screen name.
 * /home -> "Home"
 * /products/[id] -> "ProductDetail"
 * /(tabs)/profile/settings -> "ProfileSettings"
 */
function formatScreenName(pathname: string, segments: string[]): string {
  // Remove group segments like (tabs), (auth)
  const meaningful = segments.filter(
    (s) => !s.startsWith('(') && !s.endsWith(')')
  );

  if (meaningful.length === 0) return 'Home';

  return meaningful
    .map((segment) => {
      // Replace dynamic segments: [id] -> "Detail"
      if (segment.startsWith('[') && segment.endsWith(']')) {
        return 'Detail';
      }
      // Capitalize each segment
      return segment.charAt(0).toUpperCase() + segment.slice(1);
    })
    .join('');
}
```

Wire it into your root layout:

```typescript
// app/_layout.tsx
import { Stack } from 'expo-router';
import { useScreenTracking } from '@/analytics/screen-tracking';
import { AppProviders } from '@/providers';

export default function RootLayout() {
  useScreenTracking();

  return (
    <AppProviders>
      <Stack />
    </AppProviders>
  );
}
```

### 3.4 Super Properties (Properties Attached to Every Event)

Some properties should be on every event without explicitly passing them each time. These are called super properties (Mixpanel's term) or global properties:

```typescript
// src/analytics/super-properties.ts
import * as Application from 'expo-application';
import * as Device from 'expo-device';
import { Platform } from 'react-native';

interface SuperProperties {
  app_version: string;
  build_number: string;
  platform: 'ios' | 'android' | 'web';
  device_model: string | null;
  os_version: string;
  is_emulator: boolean;
}

let cachedSuperProperties: SuperProperties | null = null;

export function getSuperProperties(): SuperProperties {
  if (cachedSuperProperties) return cachedSuperProperties;

  cachedSuperProperties = {
    app_version: Application.nativeApplicationVersion ?? 'unknown',
    build_number: Application.nativeBuildVersion ?? 'unknown',
    platform: Platform.OS as 'ios' | 'android' | 'web',
    device_model: Device.modelName,
    os_version: `${Platform.OS} ${Platform.Version}`,
    is_emulator: !Device.isDevice,
  };

  return cachedSuperProperties;
}

/**
 * Merge super properties with event-specific properties.
 * Event properties take precedence if there is a conflict.
 */
export function withSuperProperties(
  eventProperties?: Record<string, unknown>
): Record<string, unknown> {
  return {
    ...getSuperProperties(),
    ...eventProperties,
  };
}
```

---

## 4. EVENT TAXONOMY

### 4.1 Standard Event Categories

A well-designed taxonomy groups events into categories. Here is a taxonomy that works for most mobile apps:

```typescript
// src/analytics/events.ts

/**
 * Standard event taxonomy.
 *
 * Naming convention: object_action (noun_verb)
 * All names use snake_case.
 *
 * Categories:
 * - Lifecycle:    app-level and session-level events
 * - Navigation:   screen views and navigation actions
 * - Auth:         signup, login, logout
 * - Onboarding:   onboarding funnel events
 * - Content:      viewing, searching, interacting with content
 * - Commerce:     cart, checkout, purchase
 * - Social:       sharing, reactions, comments
 * - Engagement:   feature usage, notifications
 * - Error:        errors visible to the user
 */

// ============================================================
// LIFECYCLE EVENTS (mostly automatic)
// ============================================================
export type LifecycleEvents = {
  app_opened: {
    is_cold_start: boolean;
    time_since_last_open_hours?: number;
  };
  app_backgrounded: {
    session_duration_seconds: number;
  };
  app_updated: {
    previous_version: string;
    new_version: string;
  };
  session_started: {
    session_number: number;
  };
};

// ============================================================
// NAVIGATION EVENTS
// ============================================================
export type NavigationEvents = {
  screen_viewed: {
    screen_name: string;
    screen_path: string;
    screen_segments?: string;
    referrer_screen?: string;
  };
  screen_exited: {
    screen_name: string;
    time_spent_ms: number;
    time_spent_seconds: number;
  };
  tab_switched: {
    from_tab: string;
    to_tab: string;
  };
  deep_link_opened: {
    url: string;
    source?: string;
    campaign?: string;
  };
};

// ============================================================
// AUTHENTICATION EVENTS
// ============================================================
export type AuthEvents = {
  signup_started: {
    method: 'email' | 'google' | 'apple' | 'phone';
  };
  signup_completed: {
    method: 'email' | 'google' | 'apple' | 'phone';
    time_to_complete_seconds: number;
  };
  signup_failed: {
    method: 'email' | 'google' | 'apple' | 'phone';
    error_code: string;
    error_message: string;
  };
  login_completed: {
    method: 'email' | 'google' | 'apple' | 'phone' | 'biometric';
  };
  login_failed: {
    method: string;
    error_code: string;
  };
  logout_completed: {
    reason: 'manual' | 'session_expired' | 'account_deleted';
  };
  password_reset_requested: {
    method: 'email' | 'phone';
  };
};

// ============================================================
// ONBOARDING EVENTS
// ============================================================
export type OnboardingEvents = {
  onboarding_started: Record<string, never>;
  onboarding_step_completed: {
    step_number: number;
    step_name: string;
    time_on_step_seconds: number;
  };
  onboarding_step_skipped: {
    step_number: number;
    step_name: string;
  };
  onboarding_completed: {
    total_time_seconds: number;
    steps_completed: number;
    steps_skipped: number;
  };
  onboarding_abandoned: {
    last_step_number: number;
    last_step_name: string;
    total_time_seconds: number;
  };
};

// ============================================================
// CONTENT EVENTS
// ============================================================
export type ContentEvents = {
  content_viewed: {
    content_type: string;
    content_id: string;
    content_name?: string;
    source: 'feed' | 'search' | 'deep_link' | 'push' | 'share' | 'direct';
  };
  content_shared: {
    content_type: string;
    content_id: string;
    share_method: 'native_share' | 'copy_link' | 'social';
    share_platform?: string;
  };
  search_performed: {
    query: string;
    results_count: number;
    search_type: 'global' | 'category' | 'filter';
    filters_applied?: string[];
  };
  search_result_tapped: {
    query: string;
    result_position: number;
    result_id: string;
    result_type: string;
  };
};

// ============================================================
// COMMERCE EVENTS
// ============================================================
export type CommerceEvents = {
  product_viewed: {
    product_id: string;
    product_name: string;
    product_category: string;
    price: number;
    currency: string;
  };
  item_added_to_cart: {
    product_id: string;
    product_name: string;
    quantity: number;
    price: number;
    currency: string;
    cart_total: number;
  };
  item_removed_from_cart: {
    product_id: string;
    quantity: number;
    cart_total: number;
  };
  checkout_started: {
    cart_total: number;
    currency: string;
    item_count: number;
    coupon_applied?: string;
  };
  checkout_step_completed: {
    step_number: number;
    step_name: 'shipping' | 'payment' | 'review';
  };
  purchase_completed: {
    order_id: string;
    amount: number;
    currency: string;
    item_count: number;
    payment_method: string;
    is_first_purchase: boolean;
    coupon_applied?: string;
    discount_amount?: number;
  };
  purchase_failed: {
    error_code: string;
    error_message: string;
    payment_method: string;
    amount: number;
  };
  subscription_started: {
    plan_id: string;
    plan_name: string;
    billing_period: 'monthly' | 'annual';
    amount: number;
    currency: string;
    is_trial: boolean;
  };
  subscription_renewed: {
    plan_id: string;
    billing_period: 'monthly' | 'annual';
    amount: number;
    renewal_count: number;
  };
  subscription_cancelled: {
    plan_id: string;
    reason?: string;
    time_subscribed_days: number;
  };
};

// ============================================================
// ENGAGEMENT EVENTS
// ============================================================
export type EngagementEvents = {
  feature_used: {
    feature_name: string;
    feature_variant?: string;
    is_first_use: boolean;
  };
  notification_received: {
    notification_type: string;
    notification_id: string;
    channel: 'push' | 'in_app' | 'email';
  };
  notification_tapped: {
    notification_type: string;
    notification_id: string;
    time_to_tap_seconds?: number;
  };
  notification_permission_prompted: {
    result: 'granted' | 'denied' | 'not_determined';
  };
  rating_prompted: {
    trigger: string;
    session_count: number;
  };
  rating_submitted: {
    rating: number;
    platform: 'ios' | 'android';
  };
  feedback_submitted: {
    feedback_type: 'bug' | 'feature_request' | 'general';
    source_screen: string;
  };
  button_tapped: {
    button_name: string;
    screen_name: string;
    section?: string;
  };
};

// ============================================================
// ERROR EVENTS
// ============================================================
export type ErrorEvents = {
  error_displayed: {
    error_type: 'network' | 'validation' | 'server' | 'permission' | 'unknown';
    error_code?: string;
    error_message: string;
    screen_name: string;
    is_recoverable: boolean;
  };
  error_boundary_triggered: {
    component_name: string;
    error_message: string;
    stack_trace?: string;
  };
};

// ============================================================
// UNION TYPE: All events
// ============================================================
export type AnalyticsEvents = LifecycleEvents &
  NavigationEvents &
  AuthEvents &
  OnboardingEvents &
  ContentEvents &
  CommerceEvents &
  EngagementEvents &
  ErrorEvents;

// Type-safe event name
export type EventName = keyof AnalyticsEvents;

// Type-safe properties for a given event
export type EventProperties<E extends EventName> = AnalyticsEvents[E];
```

### 4.2 Why This Taxonomy Works

This taxonomy works because:

1. **Events group naturally.** All purchase-related events start with `purchase_` or related commerce nouns. All authentication events start with `signup_`, `login_`, or `logout_`. When you have 50 events in your dashboard, this grouping is the difference between "I can find what I need in 5 seconds" and "I am scrolling through an alphabetical list of chaos."

2. **Funnels map directly to events.** The onboarding funnel is `onboarding_started` -> `onboarding_step_completed` (x N) -> `onboarding_completed`. The purchase funnel is `product_viewed` -> `item_added_to_cart` -> `checkout_started` -> `checkout_step_completed` (x N) -> `purchase_completed`. No ambiguity.

3. **Every event has typed properties.** You cannot log `purchase_completed` without `order_id`, `amount`, and `currency`. TypeScript catches this at compile time, not in a failed dashboard query six weeks later.

4. **Error events are first-class.** Most analytics implementations ignore errors. When something goes wrong, you want to know: what error, where (which screen), and was it recoverable? The `error_displayed` event tells you things Crashlytics cannot -- because Crashlytics tracks crashes and non-fatal exceptions, but not every validation error or server 400 response that the user sees.

### 4.3 Event Property Standards

Every event property should follow these standards:

```typescript
/**
 * EVENT PROPERTY STANDARDS
 *
 * 1. snake_case names: screen_name, not screenName
 * 2. Specific types, not stringly-typed:
 *    - Use number for amounts, counts, durations
 *    - Use boolean for flags
 *    - Use string unions for known categories
 * 3. Monetary values: always include currency alongside amount
 * 4. Time durations: include unit in name (time_spent_ms, time_spent_seconds)
 * 5. Booleans: prefix with is_ or has_ (is_first_purchase, has_coupon)
 * 6. IDs: suffix with _id (product_id, order_id, session_id)
 * 7. Counts: suffix with _count (item_count, retry_count)
 * 8. Never log PII in event properties:
 *    - No emails, phone numbers, full names
 *    - User IDs are OK (they're pseudonymous identifiers)
 *    - Search queries: be careful, they might contain PII
 */
```

---

## 5. FUNNELS

### 5.1 Defining Conversion Funnels

A funnel is a sequence of events that represents a critical user journey. If your app makes money, you have at least three funnels:

**Onboarding Funnel:**
```
app_opened (first time)
  → onboarding_started
    → onboarding_step_completed (step 1: welcome)
      → onboarding_step_completed (step 2: profile setup)
        → onboarding_step_completed (step 3: preferences)
          → onboarding_completed
```

**Purchase/Checkout Funnel:**
```
product_viewed
  → item_added_to_cart
    → checkout_started
      → checkout_step_completed (shipping)
        → checkout_step_completed (payment)
          → purchase_completed
```

**Subscription Funnel:**
```
paywall_viewed
  → plan_selected
    → checkout_started
      → subscription_started
```

### 5.2 Measuring Drop-Off

The entire point of a funnel is to find where users drop off. Here is how to think about it:

```
Onboarding Funnel Analysis (30-day window)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: app_opened            10,000  (100%)
Step 2: onboarding_started     8,200  ( 82%)  ← 18% never start onboarding
Step 3: step_1_completed       6,800  ( 68%)  ← 17% drop at first step
Step 4: step_2_completed       5,100  ( 51%)  ← 25% drop at profile setup ⚠️
Step 5: step_3_completed       4,600  ( 46%)  ← 10% drop at preferences
Step 6: onboarding_completed   4,400  ( 44%)  ←  4% drop at final step

Overall conversion: 44%
Biggest drop-off: Step 3 → Step 4 (profile setup) — 25% loss
```

That 25% drop at profile setup is your highest-leverage optimization target. You do not need to guess what to work on -- the funnel tells you.

### 5.3 Implementing Funnel Tracking

The events themselves just need to be logged correctly. The funnel analysis happens in your analytics provider's dashboard. But here is a pattern for ensuring funnel events are always complete:

```typescript
// src/analytics/funnels/onboarding-funnel.ts

import { analyticsService } from '../analytics-service';

const ONBOARDING_STEPS = [
  'welcome',
  'profile_setup',
  'preferences',
  'notifications_permission',
  'completion',
] as const;

type OnboardingStep = typeof ONBOARDING_STEPS[number];

class OnboardingFunnel {
  private startTime: number = 0;
  private stepStartTime: number = 0;
  private currentStep: number = 0;
  private stepsCompleted: number = 0;
  private stepsSkipped: number = 0;

  start() {
    this.startTime = Date.now();
    this.stepStartTime = Date.now();
    this.currentStep = 0;
    this.stepsCompleted = 0;
    this.stepsSkipped = 0;

    analyticsService.track('onboarding_started', {});
  }

  completeStep(stepName: OnboardingStep) {
    const stepIndex = ONBOARDING_STEPS.indexOf(stepName);
    const timeOnStep = Math.round((Date.now() - this.stepStartTime) / 1000);

    // Track any skipped steps
    while (this.currentStep < stepIndex) {
      analyticsService.track('onboarding_step_skipped', {
        step_number: this.currentStep + 1,
        step_name: ONBOARDING_STEPS[this.currentStep],
      });
      this.stepsSkipped++;
      this.currentStep++;
    }

    analyticsService.track('onboarding_step_completed', {
      step_number: stepIndex + 1,
      step_name: stepName,
      time_on_step_seconds: timeOnStep,
    });

    this.stepsCompleted++;
    this.currentStep = stepIndex + 1;
    this.stepStartTime = Date.now();
  }

  complete() {
    const totalTime = Math.round((Date.now() - this.startTime) / 1000);

    analyticsService.track('onboarding_completed', {
      total_time_seconds: totalTime,
      steps_completed: this.stepsCompleted,
      steps_skipped: this.stepsSkipped,
    });
  }

  abandon() {
    const totalTime = Math.round((Date.now() - this.startTime) / 1000);

    analyticsService.track('onboarding_abandoned', {
      last_step_number: this.currentStep,
      last_step_name: ONBOARDING_STEPS[Math.min(
        this.currentStep,
        ONBOARDING_STEPS.length - 1
      )],
      total_time_seconds: totalTime,
    });
  }
}

export const onboardingFunnel = new OnboardingFunnel();
```

Usage in components:

```typescript
// app/(onboarding)/welcome.tsx
import { onboardingFunnel } from '@/analytics/funnels/onboarding-funnel';

export default function WelcomeScreen() {
  useEffect(() => {
    onboardingFunnel.start();
  }, []);

  const handleContinue = () => {
    onboardingFunnel.completeStep('welcome');
    router.push('/(onboarding)/profile-setup');
  };

  return (
    <View>
      <Text>Welcome to the app!</Text>
      <Button title="Get Started" onPress={handleContinue} />
    </View>
  );
}
```

### 5.4 A/B Testing Funnel Improvements

Once you know where users drop off, A/B test improvements:

```typescript
// src/experiments/onboarding-experiment.ts
import { analyticsService } from '@/analytics/analytics-service';

/**
 * A/B test: Simplified profile setup
 *
 * Hypothesis: Reducing profile setup from 5 fields to 2 required fields
 * will increase step 3→4 conversion by 15%.
 *
 * Variant A (control): Current 5-field profile setup
 * Variant B: 2 required fields, 3 optional "complete later"
 */
export function getOnboardingVariant(userId: string): 'control' | 'simplified' {
  // Simple deterministic assignment based on user ID
  // In production, use your feature flag provider (PostHog, Amplitude, etc.)
  const hash = simpleHash(userId);
  return hash % 2 === 0 ? 'control' : 'simplified';
}

// Track which variant the user sees
export function trackExperimentExposure(variant: 'control' | 'simplified') {
  analyticsService.track('experiment_exposed', {
    experiment_name: 'onboarding_profile_simplification',
    variant_name: variant,
    experiment_id: 'exp_2026_04_onboarding_v2',
  });
}
```

**Measure the result** by comparing funnel conversion rates between variants. Most analytics providers (Amplitude, PostHog, Mixpanel) have built-in experiment analysis that calculates statistical significance for you.

---

## 6. COHORT ANALYSIS

### 6.1 What Cohorts Are and Why They Matter

A cohort is a group of users who share a common characteristic. The most common type is a time-based cohort (users who signed up in the same week), but behavioral cohorts are more powerful.

**Time-based cohorts:**
- Users who signed up in Week 1 of April 2026
- Users who made their first purchase in March 2026

**Behavioral cohorts:**
- Users who completed onboarding but never made a purchase
- Users who used feature X more than 5 times in their first week
- Users who opened the app 3+ days per week for the last month (power users)
- Users who haven't opened the app in 14+ days (at-risk)

### 6.2 Retention Curves

Retention is the most important metric for any consumer app. A retention curve shows what percentage of a cohort comes back over time:

```
Week 0 (signup):     100%  ████████████████████████████████████████
Week 1:               45%  ██████████████████
Week 2:               32%  █████████████
Week 3:               28%  ███████████
Week 4:               25%  ██████████
Week 8:               20%  ████████
Week 12:              18%  ███████
Week 26:              14%  ██████
```

**What good retention looks like:**
- The curve flattens. If it keeps dropping steeply past week 4, you have a retention problem that no amount of acquisition spend will fix.
- Day-1 retention above 40% is good for consumer apps. Above 60% is excellent.
- Week-4 retention above 20% is decent. Above 30% is strong.
- If you have a subscription app, month-6 retention is the number that determines your business viability.

### 6.3 Implementing Cohort Tracking

You do not implement cohort analysis in your app code. You implement the events and user properties that enable cohort analysis in your analytics tool.

The key is setting the right user properties at the right time:

```typescript
// src/analytics/cohort-properties.ts

import { analyticsService } from './analytics-service';

/**
 * Set cohort-defining user properties.
 * Call this at key lifecycle moments.
 */
export const cohortProperties = {
  /**
   * Called at signup. Sets permanent acquisition cohort properties.
   */
  onSignup(params: {
    signupMethod: string;
    acquisitionSource?: string;
    acquisitionCampaign?: string;
    referralCode?: string;
  }) {
    const signupWeek = getISOWeek(new Date());
    const signupMonth = new Date().toISOString().slice(0, 7); // "2026-04"

    analyticsService.setUserProperties({
      signup_week: signupWeek,
      signup_month: signupMonth,
      signup_method: params.signupMethod,
      acquisition_source: params.acquisitionSource ?? 'organic',
      acquisition_campaign: params.acquisitionCampaign ?? 'none',
      has_referral: params.referralCode ? 'true' : 'false',
    });
  },

  /**
   * Called when user completes a key action for the first time.
   * These properties enable "users who did X" cohort analysis.
   */
  onFirstPurchase(purchaseAmount: number) {
    analyticsService.setUserProperties({
      is_paying_user: 'true',
      first_purchase_date: new Date().toISOString().slice(0, 10),
      first_purchase_amount: purchaseAmount.toString(),
      days_to_first_purchase: getDaysSinceSignup().toString(),
    });
  },

  /**
   * Called periodically to update engagement tier.
   */
  updateEngagementTier(sessionsLast30Days: number) {
    let tier: string;
    if (sessionsLast30Days >= 20) tier = 'power_user';
    else if (sessionsLast30Days >= 8) tier = 'active';
    else if (sessionsLast30Days >= 3) tier = 'casual';
    else if (sessionsLast30Days >= 1) tier = 'at_risk';
    else tier = 'dormant';

    analyticsService.setUserProperties({
      engagement_tier: tier,
      sessions_last_30_days: sessionsLast30Days.toString(),
    });
  },

  /**
   * Called when user subscribes. Critical for subscription businesses.
   */
  onSubscription(params: {
    planName: string;
    billingPeriod: 'monthly' | 'annual';
    isTrial: boolean;
  }) {
    analyticsService.setUserProperties({
      subscription_plan: params.planName,
      subscription_billing: params.billingPeriod,
      is_subscriber: 'true',
      subscription_start_date: new Date().toISOString().slice(0, 10),
      started_with_trial: params.isTrial ? 'true' : 'false',
    });
  },
};

function getISOWeek(date: Date): string {
  const d = new Date(date);
  d.setHours(0, 0, 0, 0);
  d.setDate(d.getDate() + 3 - ((d.getDay() + 6) % 7));
  const week1 = new Date(d.getFullYear(), 0, 4);
  const weekNumber = Math.round(
    ((d.getTime() - week1.getTime()) / 86400000 - 3 + ((week1.getDay() + 6) % 7)) / 7
  ) + 1;
  return `${d.getFullYear()}-W${weekNumber.toString().padStart(2, '0')}`;
}

function getDaysSinceSignup(): number {
  // This would come from your user profile / auth context
  // Placeholder implementation
  return 0;
}
```

### 6.4 Using Cohorts for Targeted Actions

Once you have behavioral cohorts, you can take targeted action:

| Cohort | Signal | Action |
|--------|--------|--------|
| **Power users** (20+ sessions/month) | High engagement, likely advocates | Prompt for App Store review, offer referral program |
| **Active users** (8-19 sessions/month) | Solid engagement, room to grow | Feature discovery notifications, new feature announcements |
| **Casual users** (3-7 sessions/month) | Using app but not habit-formed | Push notifications with personalized content, email re-engagement |
| **At-risk users** (1-2 sessions/month) | Losing interest | Win-back campaign, "We miss you" notification, survey about what's missing |
| **Dormant users** (0 sessions/month) | Churned or churning | Re-engagement email sequence, special offer if applicable |
| **Trial users** (not converted) | Evaluating product | Feature highlight notifications, offer help, extend trial |
| **New users** (first 7 days) | Critical activation window | Onboarding optimization, quick wins, reduce time-to-value |

---

## 7. SEGMENT AS DATA PIPELINE

### 7.1 The Architecture

Segment's value proposition is simple: instrument once, send everywhere.

```
                        ┌──────────────────┐
                        │   Your App (SDK)  │
                        │                  │
                        │  analytics.track() │
                        │  analytics.identify()│
                        └────────┬─────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │     Segment      │
                        │                  │
                        │  ┌────────────┐  │
                        │  │  Protocols  │  │  ← Schema validation
                        │  └────────────┘  │
                        │  ┌────────────┐  │
                        │  │  Identity   │  │  ← Cross-device resolution
                        │  │  Graph      │  │
                        │  └────────────┘  │
                        └────────┬─────────┘
                                 │
              ┌──────────┬───────┼────────┬──────────┐
              ▼          ▼       ▼        ▼          ▼
        ┌──────────┐┌────────┐┌──────┐┌────────┐┌─────────┐
        │ Amplitude ││Firebase││BigQ  ││Customer││ Google  │
        │ (product ││(crash  ││uery  ││.io     ││ Ads     │
        │ analytics)││reports)││(raw  ││(email) ││(attrib) │
        └──────────┘└────────┘│data) │└────────┘└─────────┘
                              └──────┘
```

### 7.2 React Native Implementation with Segment

```bash
npx expo install @segment/analytics-react-native
# Add destination plugins
npx expo install @segment/analytics-react-native-plugin-firebase
npx expo install @segment/analytics-react-native-plugin-amplitude
```

```typescript
// src/analytics/segment-setup.ts
import {
  createClient,
  SegmentClient,
} from '@segment/analytics-react-native';
import { FirebasePlugin } from '@segment/analytics-react-native-plugin-firebase';
import { AmplitudeSessionPlugin } from '@segment/analytics-react-native-plugin-amplitude-session';

let segmentClient: SegmentClient | null = null;

export function initializeSegment(): SegmentClient {
  if (segmentClient) return segmentClient;

  segmentClient = createClient({
    writeKey: process.env.EXPO_PUBLIC_SEGMENT_WRITE_KEY!,
    trackAppLifecycleEvents: true,  // Auto-track app open, background, etc.
    trackDeepLinks: true,
    debug: __DEV__,
    flushAt: 20,          // Batch 20 events before sending
    flushInterval: 30,    // Or send every 30 seconds
  });

  // Add destination plugins
  segmentClient.add({ plugin: new FirebasePlugin() });
  segmentClient.add({ plugin: new AmplitudeSessionPlugin() });

  return segmentClient;
}

export function getSegmentClient(): SegmentClient {
  if (!segmentClient) {
    throw new Error('Segment not initialized. Call initializeSegment() first.');
  }
  return segmentClient;
}
```

### 7.3 When Segment Is Worth the Cost

Segment is expensive. Here is the honest calculation:

**Segment makes sense when:**
- You send data to 3+ destinations (analytics, CRM, email, advertising)
- You anticipate switching analytics providers in the next 12 months
- You need schema enforcement across a large team (Protocols)
- You want to replay historical data to new destinations
- The engineering cost of maintaining 3+ SDKs exceeds the Segment subscription

**Segment does NOT make sense when:**
- You only use one analytics provider
- You are a small team with under 10,000 monthly users
- Your budget is tight -- the money is better spent on the analytics tool itself
- You are in a regulated industry and cannot have data flowing through a third-party pipeline

### 7.4 Open-Source Alternatives

If you want the Segment architecture without the Segment price:

**RudderStack:**
- Open-source Segment alternative. Self-hostable or cloud.
- Compatible with Segment's API (you can swap them with minimal code changes).
- Warehouse-first: events go to your data warehouse first, then to destinations.
- React Native SDK available.
- Free tier: 25,000 events/month (cloud), unlimited (self-hosted).

**Jitsu:**
- Open-source data pipeline. Self-hosted or cloud.
- More focused on data engineers than product teams.
- Good if you primarily want events in your data warehouse and run analytics from there.

```typescript
// RudderStack is API-compatible with Segment
// If you switch from Segment, the code changes are minimal:

// Before (Segment):
import { createClient } from '@segment/analytics-react-native';
const client = createClient({ writeKey: SEGMENT_KEY });

// After (RudderStack):
import { createClient } from '@rudderstack/rudder-sdk-react-native';
const client = await createClient({
  writeKey: RUDDERSTACK_KEY,
  dataPlaneUrl: 'https://your-data-plane.rudderstack.com',
});

// The track/identify/screen API is identical
client.track('purchase_completed', { amount: 49.99 });
client.identify('user_123', { email: 'user@example.com' });
```

---

## 8. ATTRIBUTION & INSTALL TRACKING

### 8.1 The Attribution Problem

Attribution answers the most expensive question in mobile: **Where did this user come from?**

When a user installs your app, you want to know:
- Did they click a Google Ad?
- Did they follow a link from a blog post?
- Did a friend share a link?
- Did they see it featured on the App Store?
- Did they come from a TikTok campaign?

This matters because if you are spending $50,000/month on ads across Google, Meta, TikTok, and Apple Search Ads, you need to know which channel is actually driving valuable users (not just installs, but users who convert and retain).

### 8.2 How Attribution Works (Conceptually)

The attribution flow for mobile:

```
1. User sees ad / clicks link
   ↓
2. User is taken to App Store / Play Store
   ↓
3. User installs app (this breaks the tracking chain -- stores don't pass click data)
   ↓
4. User opens app for the first time
   ↓
5. Attribution SDK matches the install to the original click
   ↓
6. Attribution data is available: "This user came from Google campaign X"
```

Step 3 is the hard part. When a user goes through the App Store or Play Store, the link context is lost. Attribution providers use various techniques to match installs to clicks:

- **Device ID matching** (deprecated on iOS with ATT): If you know the device ID that clicked the ad and the device ID that opened the app, you match them.
- **Probabilistic/Fingerprinting** (increasingly restricted): Match based on IP address, device model, OS version, and timing. Less precise but works without device IDs.
- **SKAdNetwork (iOS)**: Apple's privacy-preserving attribution framework. Gives aggregated, delayed attribution data without exposing user-level data.
- **Google Play Install Referrer (Android)**: The Play Store passes referrer information directly. This is the gold standard on Android -- it is deterministic and reliable.
- **Deferred deep links**: A user clicks a link with parameters, installs the app, and on first open is taken to the specific content from the link (and the parameters are captured for attribution).

### 8.3 UTM Parameters

UTM parameters are the simplest form of attribution for web-to-app flows:

```
https://yourapp.com/download?
  utm_source=google&
  utm_medium=cpc&
  utm_campaign=spring_2026&
  utm_content=blue_banner&
  utm_term=fitness+app
```

| Parameter | Meaning | Example |
|-----------|---------|---------|
| `utm_source` | Where the traffic comes from | google, facebook, newsletter |
| `utm_medium` | Marketing medium | cpc, email, social, organic |
| `utm_campaign` | Campaign name | spring_2026, black_friday |
| `utm_content` | Differentiates similar content | blue_banner, video_ad |
| `utm_term` | Paid search keyword | fitness+app |

For mobile, UTM parameters flow through deferred deep links:

```typescript
// src/analytics/attribution.ts
import * as Linking from 'expo-linking';
import { analyticsService } from './analytics-service';

/**
 * Parse UTM parameters from the initial deep link
 * and set them as user properties for attribution.
 */
export async function captureAttribution() {
  const initialUrl = await Linking.getInitialURL();

  if (!initialUrl) return;

  const url = new URL(initialUrl);
  const utmParams = {
    utm_source: url.searchParams.get('utm_source'),
    utm_medium: url.searchParams.get('utm_medium'),
    utm_campaign: url.searchParams.get('utm_campaign'),
    utm_content: url.searchParams.get('utm_content'),
    utm_term: url.searchParams.get('utm_term'),
  };

  // Filter out null values
  const validParams = Object.fromEntries(
    Object.entries(utmParams).filter(([, v]) => v !== null)
  );

  if (Object.keys(validParams).length > 0) {
    // Set as user properties (persist across all future events)
    analyticsService.setUserProperties(validParams);

    // Also track as an event for the attribution timestamp
    analyticsService.track('attribution_captured', {
      ...validParams,
      initial_url: initialUrl,
    });
  }
}
```

### 8.4 Attribution Providers

For serious install attribution (matching ad clicks to app installs across ad networks), you need a dedicated attribution provider:

| Provider | Strengths | Pricing | Market Position |
|----------|-----------|---------|-----------------|
| **Adjust** | Privacy-focused, strong in Europe, good fraud prevention | Per-attribution pricing, starts ~$500/mo | Major player, Applovin-owned |
| **AppsFlyer** | Market leader, widest ad network integrations, strong SKAdNetwork support | Per-attribution pricing, free tier available | Largest market share |
| **Branch** | Best deep linking, good attribution, link management | Free tier (10K attributions/mo), then tiered | Deep linking leader, growing in attribution |
| **Kochava** | Strong fraud protection, unified audience platform | Custom pricing | Enterprise-focused |
| **Singular** | Cost aggregation + attribution in one platform | Custom pricing | Growing, good for ROI analysis |

**For most React Native apps, Branch is the best starting point** because it combines deep linking (which you probably need anyway -- see Chapter 11) with attribution. You get two features in one integration.

```typescript
// Basic Branch integration for React Native (conceptual)
// See Branch's official React Native docs for full setup

import branch from 'react-native-branch';

// Listen for deep link opens and capture attribution
branch.subscribe({
  onOpenStart: ({ uri, cachedInitialEvent }) => {
    console.log('Branch link opening:', uri);
  },
  onOpenComplete: ({ error, params }) => {
    if (error) {
      console.error('Branch error:', error);
      return;
    }

    if (params['+clicked_branch_link']) {
      // User came from a Branch link -- capture attribution
      analyticsService.track('attribution_captured', {
        source: params['~channel'],        // e.g., "google", "facebook"
        campaign: params['~campaign'],     // e.g., "spring_2026"
        feature: params['~feature'],       // e.g., "marketing"
        referring_link: params['~referring_link'],
        is_first_session: params['+is_first_session'],
      });

      analyticsService.setUserProperties({
        attribution_source: params['~channel'] ?? 'direct',
        attribution_campaign: params['~campaign'] ?? 'none',
      });

      // Handle deferred deep link: take user to specific content
      if (params['product_id']) {
        router.push(`/products/${params['product_id']}`);
      }
    }
  },
});
```

### 8.5 The Post-ATT World on iOS

In 2021, Apple introduced App Tracking Transparency (ATT), which requires apps to ask permission before tracking users across other apps and websites. This fundamentally changed mobile attribution. We cover ATT implementation in detail in the next section.

The practical impact on attribution:

**Before ATT (pre-iOS 14.5):**
- IDFA (Identifier for Advertisers) was available by default
- Deterministic attribution: "User with IDFA X clicked this ad and then installed the app"
- Attribution accuracy: ~95%

**After ATT (iOS 14.5+):**
- IDFA requires explicit user consent (opt-in rate: 20-35% in most apps)
- For users who deny: no IDFA, no deterministic attribution
- Relies on SKAdNetwork for aggregated, delayed attribution
- Attribution accuracy for non-consenting users: ~50-70% (probabilistic)

This is not going back. The trend across the industry is toward more privacy, not less. Android is following with its own Privacy Sandbox. Build your attribution strategy assuming you will never have user-level cross-app tracking precision again.

---

## 9. APP TRACKING TRANSPARENCY (iOS)

### 9.1 What ATT Requires

ATT (App Tracking Transparency) is Apple's framework that requires apps to request permission before tracking users across apps and websites owned by other companies. The key word is "other companies" -- tracking within your own app and your own services does not require ATT.

**You MUST show the ATT prompt if you:**
- Use the IDFA for advertising attribution
- Share user data with data brokers
- Link user data collected from your app with data from other companies' apps or websites for advertising purposes

**You do NOT need ATT if you:**
- Only track user behavior within your own app
- Use first-party analytics (Firebase Analytics for your own analysis)
- Use SKAdNetwork for attribution (it is privacy-preserving by design)
- Track conversions on your own website after in-app actions

### 9.2 Implementing ATT in React Native

```bash
npx expo install expo-tracking-transparency
```

```typescript
// src/analytics/tracking-transparency.ts
import {
  requestTrackingPermissionsAsync,
  getTrackingPermissionsAsync,
  PermissionStatus,
} from 'expo-tracking-transparency';
import { Platform } from 'react-native';

export type TrackingStatus =
  | 'authorized'
  | 'denied'
  | 'not_determined'
  | 'restricted'
  | 'unavailable'; // Android or older iOS

/**
 * Get current tracking permission status without prompting.
 */
export async function getTrackingStatus(): Promise<TrackingStatus> {
  if (Platform.OS !== 'ios') return 'unavailable';

  const { status } = await getTrackingPermissionsAsync();

  switch (status) {
    case PermissionStatus.GRANTED:
      return 'authorized';
    case PermissionStatus.DENIED:
      return 'denied';
    case PermissionStatus.UNDETERMINED:
      return 'not_determined';
    default:
      return 'restricted';
  }
}

/**
 * Request tracking permission.
 * Returns the resulting status.
 *
 * IMPORTANT: Only call this once. Calling it again after denial
 * will NOT show the prompt again -- iOS only allows one prompt per install.
 */
export async function requestTrackingPermission(): Promise<TrackingStatus> {
  if (Platform.OS !== 'ios') return 'unavailable';

  const { status } = await requestTrackingPermissionsAsync();

  switch (status) {
    case PermissionStatus.GRANTED:
      return 'authorized';
    case PermissionStatus.DENIED:
      return 'denied';
    default:
      return 'restricted';
  }
}
```

### 9.3 The Pre-Permission Dialog Strategy

Apple's ATT prompt is not customizable. It looks like this:

```
┌─────────────────────────────────────────┐
│                                         │
│   "MyApp" Would Like Permission to      │
│   Track Your Activity Across Other      │
│   Companies' Apps and Websites          │
│                                         │
│   Your data will be used to deliver     │
│   personalized ads to you.              │
│                                         │
│   ┌───────────────────────────────────┐ │
│   │        Ask App Not to Track       │ │
│   └───────────────────────────────────┘ │
│   ┌───────────────────────────────────┐ │
│   │              Allow                │ │
│   └───────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘
```

That wording is hostile to opt-in. "Track your activity across other companies' apps and websites" sounds invasive. The default opt-in rate is 15-25%.

The solution is a **pre-permission dialog** -- a custom screen shown BEFORE the ATT prompt that explains why you are asking and what the user gets in return:

```typescript
// src/components/TrackingConsentScreen.tsx
import React, { useState } from 'react';
import { View, Text, StyleSheet, TouchableOpacity, Image } from 'react-native';
import { requestTrackingPermission, TrackingStatus } from '@/analytics/tracking-transparency';

interface Props {
  onComplete: (status: TrackingStatus) => void;
}

/**
 * Pre-permission dialog shown BEFORE the native ATT prompt.
 *
 * Apple allows this as long as:
 * 1. You don't prevent the user from continuing if they decline
 * 2. You don't incentivize acceptance (no "allow tracking for a discount")
 * 3. You show the native ATT prompt regardless (not just your own dialog)
 */
export function TrackingConsentScreen({ onComplete }: Props) {
  const [isRequesting, setIsRequesting] = useState(false);

  const handleAllow = async () => {
    setIsRequesting(true);
    const status = await requestTrackingPermission();
    onComplete(status);
  };

  const handleDecline = () => {
    // Skip the ATT prompt entirely -- user has already said no
    // (You could also still show ATT, but it annoys users who already said no)
    onComplete('denied');
  };

  return (
    <View style={styles.container}>
      <Image
        source={require('@/assets/images/privacy-friendly.png')}
        style={styles.image}
      />

      <Text style={styles.title}>Help Us Improve Your Experience</Text>

      <Text style={styles.body}>
        We use anonymous data to understand how people use the app so we can make
        it better. We never sell your data.
      </Text>

      <Text style={styles.body}>
        On the next screen, iOS will ask about tracking. This helps us understand
        which features are most useful and fix issues faster.
      </Text>

      <View style={styles.benefits}>
        <BenefitRow icon="chart" text="Help us build features you actually want" />
        <BenefitRow icon="bug" text="Help us find and fix bugs faster" />
        <BenefitRow icon="lock" text="Your data stays anonymous" />
      </View>

      <TouchableOpacity style={styles.allowButton} onPress={handleAllow}>
        <Text style={styles.allowButtonText}>
          {isRequesting ? 'Requesting...' : 'Continue'}
        </Text>
      </TouchableOpacity>

      <TouchableOpacity style={styles.declineButton} onPress={handleDecline}>
        <Text style={styles.declineButtonText}>Maybe Later</Text>
      </TouchableOpacity>
    </View>
  );
}

function BenefitRow({ icon, text }: { icon: string; text: string }) {
  return (
    <View style={styles.benefitRow}>
      <Text style={styles.benefitIcon}>{icon === 'chart' ? '📊' : icon === 'bug' ? '🔧' : '🔒'}</Text>
      <Text style={styles.benefitText}>{text}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    paddingHorizontal: 32,
    backgroundColor: '#ffffff',
  },
  image: {
    width: 120,
    height: 120,
    marginBottom: 24,
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    textAlign: 'center',
    marginBottom: 16,
    color: '#1a1a1a',
  },
  body: {
    fontSize: 16,
    lineHeight: 24,
    textAlign: 'center',
    color: '#666666',
    marginBottom: 12,
  },
  benefits: {
    marginVertical: 24,
    alignSelf: 'stretch',
  },
  benefitRow: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 12,
    paddingHorizontal: 16,
  },
  benefitIcon: {
    fontSize: 20,
    marginRight: 12,
  },
  benefitText: {
    fontSize: 15,
    color: '#333333',
    flex: 1,
  },
  allowButton: {
    backgroundColor: '#007AFF',
    paddingVertical: 16,
    paddingHorizontal: 48,
    borderRadius: 12,
    width: '100%',
    alignItems: 'center',
    marginBottom: 12,
  },
  allowButtonText: {
    color: '#ffffff',
    fontSize: 17,
    fontWeight: '600',
  },
  declineButton: {
    paddingVertical: 12,
    paddingHorizontal: 48,
  },
  declineButtonText: {
    color: '#999999',
    fontSize: 15,
  },
});
```

**With a well-designed pre-permission dialog, opt-in rates typically increase from 20% to 40-50%.** The key is explaining the benefit to the user, not just "we want to track you."

### 9.4 When to Show the ATT Prompt

**Do NOT show ATT on first launch.** The user has not experienced any value from your app yet. Why would they trust you?

Best practice timing:

```typescript
// src/analytics/att-timing.ts

/**
 * Determine whether to show the ATT prompt.
 *
 * Strategy: Wait until the user has experienced value.
 * Show after:
 * - Completing onboarding (they've invested time)
 * - Using a key feature for the first time (they've seen value)
 * - 3rd session (they've chosen to come back)
 *
 * Never show:
 * - On first launch
 * - During onboarding
 * - During checkout
 * - After the user has already been prompted (iOS only allows once)
 */
export function shouldShowATTPrompt(context: {
  hasBeenPrompted: boolean;
  sessionCount: number;
  hasCompletedOnboarding: boolean;
  isInCheckout: boolean;
}): boolean {
  if (context.hasBeenPrompted) return false;
  if (context.isInCheckout) return false;
  if (!context.hasCompletedOnboarding) return false;
  if (context.sessionCount < 3) return false;

  return true;
}
```

### 9.5 What Happens When Denied

When a user denies ATT (which ~65-80% will):

1. **IDFA is all zeros.** You cannot use it for attribution or advertising.
2. **Your attribution provider falls back to probabilistic matching** -- matching based on IP, device model, OS version, and timing. Less accurate but still provides directional data.
3. **SKAdNetwork provides aggregated attribution.** You know "10 installs came from this Google campaign yesterday" but not which specific users.
4. **Your first-party analytics still work perfectly.** Firebase Analytics, Mixpanel, Amplitude, PostHog -- none of these require ATT because they track within your app, not across other apps.

```typescript
// src/analytics/post-att-strategy.ts

import { getTrackingStatus, TrackingStatus } from './tracking-transparency';

/**
 * Configure analytics based on ATT status.
 * First-party analytics always work. Cross-app attribution varies.
 */
export async function configureAnalyticsForATT() {
  const status = await getTrackingStatus();

  // First-party analytics: ALWAYS enabled (no ATT required)
  // These track behavior within YOUR app only
  await initializeFirstPartyAnalytics(); // Firebase, Mixpanel, PostHog, etc.

  switch (status) {
    case 'authorized':
      // Full attribution: enable IDFA-based attribution
      await enableFullAttribution();
      break;

    case 'denied':
    case 'restricted':
      // Limited attribution: SKAdNetwork + probabilistic
      await enablePrivacyPreservingAttribution();
      break;

    case 'not_determined':
      // Haven't asked yet. Use privacy-preserving until we ask.
      await enablePrivacyPreservingAttribution();
      break;

    case 'unavailable':
      // Android or older iOS. GAID available (for now).
      await enableFullAttribution();
      break;
  }
}

async function initializeFirstPartyAnalytics() {
  // These do NOT require ATT and always work
  // - Firebase Analytics (logEvent, setUserId, etc.)
  // - Mixpanel (track, identify, etc.)
  // - PostHog (capture, identify, etc.)
  // - Your own analytics endpoint
}

async function enableFullAttribution() {
  // User opted in:
  // - Enable IDFA collection for ad attribution
  // - Enable cross-app measurement
  // - Share device-level data with attribution providers
}

async function enablePrivacyPreservingAttribution() {
  // User denied or not yet asked:
  // - Enable SKAdNetwork (Apple's privacy-preserving attribution)
  // - Use probabilistic attribution in your attribution provider
  // - Do NOT collect or share IDFA
  // - First-party analytics still fully functional
}
```

---

## 10. PRIVACY-FIRST ANALYTICS

### 10.1 The Privacy Landscape

Privacy is not just ATT. Here is what you need to handle:

| Regulation / Framework | Scope | Key Requirements |
|----------------------|-------|------------------|
| **GDPR** (EU) | EU residents | Explicit consent for data collection, right to deletion, data portability, DPO |
| **CCPA/CPRA** (California) | California residents | Opt-out of sale, right to know/delete, sensitive data protections |
| **ATT** (Apple) | All iOS users | Opt-in for cross-app tracking |
| **Privacy Sandbox** (Google) | Android users (rolling out) | Replacement for GAID, aggregated measurement |
| **LGPD** (Brazil) | Brazilian residents | Similar to GDPR, consent-based |
| **PIPL** (China) | Chinese residents | Strict consent, data localization |
| **ePrivacy** (EU) | EU residents | Cookie/storage consent on web |

### 10.2 GDPR Consent Management

For GDPR compliance, you need explicit consent before collecting personal data. Here is a consent management system:

```typescript
// src/analytics/consent-manager.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

/**
 * Consent categories. Users can grant or deny each independently.
 */
export interface ConsentPreferences {
  /** Required for basic app functionality. Cannot be denied. */
  necessary: true;

  /** Anonymous usage analytics (Firebase, PostHog, etc.) */
  analytics: boolean;

  /** Advertising and attribution tracking */
  advertising: boolean;

  /** Third-party integrations (Segment destinations, etc.) */
  thirdParty: boolean;
}

const CONSENT_KEY = '@consent_preferences';
const CONSENT_VERSION_KEY = '@consent_version';
const CURRENT_CONSENT_VERSION = '2026-04-07'; // Update when consent categories change

class ConsentManager {
  private preferences: ConsentPreferences | null = null;
  private listeners: Array<(prefs: ConsentPreferences) => void> = [];

  /**
   * Load saved consent preferences.
   * Returns null if user has never been asked or consent version has changed.
   */
  async load(): Promise<ConsentPreferences | null> {
    try {
      const [storedPrefs, storedVersion] = await Promise.all([
        AsyncStorage.getItem(CONSENT_KEY),
        AsyncStorage.getItem(CONSENT_VERSION_KEY),
      ]);

      // If consent version changed, re-ask
      if (storedVersion !== CURRENT_CONSENT_VERSION) {
        return null;
      }

      if (storedPrefs) {
        this.preferences = JSON.parse(storedPrefs);
        return this.preferences;
      }

      return null;
    } catch {
      return null;
    }
  }

  /**
   * Save consent preferences.
   */
  async save(preferences: ConsentPreferences): Promise<void> {
    this.preferences = preferences;

    await Promise.all([
      AsyncStorage.setItem(CONSENT_KEY, JSON.stringify(preferences)),
      AsyncStorage.setItem(CONSENT_VERSION_KEY, CURRENT_CONSENT_VERSION),
    ]);

    // Notify listeners (analytics service, etc.)
    this.listeners.forEach((listener) => listener(preferences));
  }

  /**
   * Get current preferences. Returns null if not yet set.
   */
  getPreferences(): ConsentPreferences | null {
    return this.preferences;
  }

  /**
   * Check if a specific consent category is granted.
   */
  hasConsent(category: keyof ConsentPreferences): boolean {
    if (!this.preferences) return false;
    return this.preferences[category] === true;
  }

  /**
   * Subscribe to consent changes.
   */
  onChange(listener: (prefs: ConsentPreferences) => void): () => void {
    this.listeners.push(listener);
    return () => {
      this.listeners = this.listeners.filter((l) => l !== listener);
    };
  }

  /**
   * Accept all categories.
   */
  async acceptAll(): Promise<void> {
    await this.save({
      necessary: true,
      analytics: true,
      advertising: true,
      thirdParty: true,
    });
  }

  /**
   * Deny all optional categories.
   */
  async denyAll(): Promise<void> {
    await this.save({
      necessary: true,
      analytics: false,
      advertising: false,
      thirdParty: false,
    });
  }

  /**
   * Delete all stored consent and analytics data.
   * GDPR "right to be forgotten" support.
   */
  async deleteAllData(): Promise<void> {
    this.preferences = null;
    await AsyncStorage.multiRemove([CONSENT_KEY, CONSENT_VERSION_KEY]);
    // Also clear analytics IDs
    // Provider-specific: call their reset/clear methods
  }
}

export const consentManager = new ConsentManager();
```

### 10.3 The Consent UI

```typescript
// src/components/ConsentBanner.tsx
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, Switch, StyleSheet, ScrollView } from 'react-native';
import { consentManager, ConsentPreferences } from '@/analytics/consent-manager';

interface Props {
  onComplete: () => void;
}

export function ConsentBanner({ onComplete }: Props) {
  const [showDetails, setShowDetails] = useState(false);
  const [prefs, setPrefs] = useState<ConsentPreferences>({
    necessary: true,
    analytics: true,
    advertising: false,
    thirdParty: false,
  });

  const handleAcceptAll = async () => {
    await consentManager.acceptAll();
    onComplete();
  };

  const handleDenyAll = async () => {
    await consentManager.denyAll();
    onComplete();
  };

  const handleSaveChoices = async () => {
    await consentManager.save(prefs);
    onComplete();
  };

  if (!showDetails) {
    return (
      <View style={styles.banner}>
        <Text style={styles.bannerTitle}>Your Privacy Matters</Text>
        <Text style={styles.bannerText}>
          We use analytics to improve the app experience. You can choose which
          types of data collection to allow.
        </Text>
        <View style={styles.buttonRow}>
          <TouchableOpacity style={styles.denyButton} onPress={handleDenyAll}>
            <Text style={styles.denyButtonText}>Deny All</Text>
          </TouchableOpacity>
          <TouchableOpacity
            style={styles.customizeButton}
            onPress={() => setShowDetails(true)}
          >
            <Text style={styles.customizeButtonText}>Customize</Text>
          </TouchableOpacity>
          <TouchableOpacity style={styles.acceptButton} onPress={handleAcceptAll}>
            <Text style={styles.acceptButtonText}>Accept All</Text>
          </TouchableOpacity>
        </View>
      </View>
    );
  }

  return (
    <ScrollView style={styles.detailContainer}>
      <Text style={styles.detailTitle}>Privacy Preferences</Text>

      <ConsentCategory
        title="Necessary"
        description="Required for core app functionality. Cannot be disabled."
        value={true}
        locked
      />

      <ConsentCategory
        title="Analytics"
        description="Anonymous usage data to help us understand which features are used and fix bugs. No personal data is collected."
        value={prefs.analytics}
        onValueChange={(v) => setPrefs({ ...prefs, analytics: v })}
      />

      <ConsentCategory
        title="Advertising"
        description="Allows measurement of ad campaign effectiveness. Used to understand where you heard about us."
        value={prefs.advertising}
        onValueChange={(v) => setPrefs({ ...prefs, advertising: v })}
      />

      <ConsentCategory
        title="Third-Party Services"
        description="Shares anonymous data with analytics partners for deeper product insights."
        value={prefs.thirdParty}
        onValueChange={(v) => setPrefs({ ...prefs, thirdParty: v })}
      />

      <TouchableOpacity style={styles.saveButton} onPress={handleSaveChoices}>
        <Text style={styles.saveButtonText}>Save Preferences</Text>
      </TouchableOpacity>
    </ScrollView>
  );
}

function ConsentCategory({
  title,
  description,
  value,
  onValueChange,
  locked,
}: {
  title: string;
  description: string;
  value: boolean;
  onValueChange?: (value: boolean) => void;
  locked?: boolean;
}) {
  return (
    <View style={styles.category}>
      <View style={styles.categoryHeader}>
        <Text style={styles.categoryTitle}>{title}</Text>
        <Switch
          value={value}
          onValueChange={onValueChange}
          disabled={locked}
          trackColor={{ true: '#007AFF' }}
        />
      </View>
      <Text style={styles.categoryDescription}>{description}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  banner: {
    padding: 20,
    backgroundColor: '#ffffff',
    borderTopWidth: 1,
    borderTopColor: '#e0e0e0',
  },
  bannerTitle: {
    fontSize: 18,
    fontWeight: '700',
    marginBottom: 8,
    color: '#1a1a1a',
  },
  bannerText: {
    fontSize: 14,
    lineHeight: 20,
    color: '#666666',
    marginBottom: 16,
  },
  buttonRow: {
    flexDirection: 'row',
    gap: 8,
  },
  denyButton: {
    flex: 1,
    paddingVertical: 12,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#cccccc',
    alignItems: 'center',
  },
  denyButtonText: {
    color: '#666666',
    fontWeight: '600',
  },
  customizeButton: {
    flex: 1,
    paddingVertical: 12,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: '#007AFF',
    alignItems: 'center',
  },
  customizeButtonText: {
    color: '#007AFF',
    fontWeight: '600',
  },
  acceptButton: {
    flex: 1,
    paddingVertical: 12,
    borderRadius: 8,
    backgroundColor: '#007AFF',
    alignItems: 'center',
  },
  acceptButtonText: {
    color: '#ffffff',
    fontWeight: '600',
  },
  detailContainer: {
    flex: 1,
    padding: 20,
    backgroundColor: '#ffffff',
  },
  detailTitle: {
    fontSize: 22,
    fontWeight: '700',
    marginBottom: 20,
    color: '#1a1a1a',
  },
  category: {
    marginBottom: 20,
    paddingBottom: 20,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  categoryHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 8,
  },
  categoryTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#1a1a1a',
  },
  categoryDescription: {
    fontSize: 14,
    lineHeight: 20,
    color: '#888888',
  },
  saveButton: {
    backgroundColor: '#007AFF',
    paddingVertical: 16,
    borderRadius: 12,
    alignItems: 'center',
    marginTop: 12,
    marginBottom: 40,
  },
  saveButtonText: {
    color: '#ffffff',
    fontSize: 17,
    fontWeight: '600',
  },
});
```

### 10.4 Self-Hosted PostHog for Full Data Ownership

For companies in regulated industries (healthcare, fintech, government) or companies that simply want full control over their user data, self-hosted PostHog is the answer:

```yaml
# docker-compose.yml for PostHog self-hosted
# This is a simplified version. See PostHog's official docs for production config.

version: '3'

services:
  posthog:
    image: posthog/posthog:release-1.43.0
    environment:
      DATABASE_URL: postgres://posthog:posthog@db:5432/posthog
      REDIS_URL: redis://redis:6379
      SECRET_KEY: your-secret-key-here
      SITE_URL: https://analytics.yourcompany.com
      IS_BEHIND_PROXY: 'true'
    ports:
      - '8000:8000'
    depends_on:
      - db
      - redis
      - clickhouse

  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: posthog
      POSTGRES_USER: posthog
      POSTGRES_PASSWORD: posthog
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

  clickhouse:
    image: clickhouse/clickhouse-server:23.12
    volumes:
      - clickhouse-data:/var/lib/clickhouse

volumes:
  postgres-data:
  redis-data:
  clickhouse-data:
```

Then point your React Native SDK to your own instance:

```typescript
// PostHog pointing to your own server
<PostHogProvider
  apiKey="phc_your_self_hosted_key"
  options={{
    host: 'https://analytics.yourcompany.com', // Your server, your data
  }}
>
```

**The tradeoff:** You now own the infrastructure. That means you handle upgrades, scaling, backups, and uptime. ClickHouse (PostHog's analytics database) can be resource-intensive. For a team under 100K users, this is manageable. At scale, budget for dedicated DevOps time or use PostHog Cloud in an EU region if GDPR is the primary concern.

### 10.5 Minimal Data Collection Philosophy

The best privacy strategy is collecting less data:

```typescript
// src/analytics/privacy-config.ts

/**
 * Privacy-first analytics configuration.
 *
 * Principle: Collect the minimum data needed to make product decisions.
 * Everything else is noise that creates liability.
 */
export const privacyConfig = {
  // COLLECT: Anonymous behavioral events
  // These tell you WHAT users do, which is what you need for product decisions
  trackEvents: true,

  // COLLECT: Aggregate device/OS info
  // You need this to prioritize platform-specific bugs
  trackDeviceInfo: true,

  // DO NOT COLLECT: Exact location
  // City-level geo (from IP, not GPS) is enough for localization decisions
  trackExactLocation: false,

  // DO NOT COLLECT: User agent strings with full detail
  // OS + version is enough. You don't need browser fingerprint-level data.
  trackFullUserAgent: false,

  // DO NOT COLLECT: Free-form text in analytics
  // Search queries, form inputs, error messages -- these can contain PII.
  // If you must track search queries, hash them or use categories.
  trackSearchQueries: false,

  // DO NOT COLLECT: Anything from children (COPPA)
  // If your app has users under 13, you need entirely separate rules.
  trackMinors: false,

  // DATA RETENTION: Delete raw events after 90 days
  // Aggregated reports live longer. Raw events are a liability.
  rawEventRetentionDays: 90,

  // IP HANDLING: Anonymize before storage
  // Most analytics providers support this. Enable it.
  anonymizeIp: true,
} as const;
```

---

## 11. WEB ANALYTICS

### 11.1 Vercel Analytics

If you are deploying your Next.js app on Vercel (see Chapter 24-25), Vercel Analytics gives you Web Vitals tracking with zero configuration:

```bash
npm install @vercel/analytics
```

```typescript
// app/layout.tsx (Next.js App Router)
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        {children}
        <Analytics />
      </body>
    </html>
  );
}
```

That is it. You now get:
- Core Web Vitals (LCP, FID/INP, CLS) from real users
- Page views with path, referrer, and country
- Audience insights (device, browser, OS)
- Performance insights per route

**What Vercel Analytics does NOT give you:** Custom events, funnels, cohorts, user identification, session replay. It is a Web Vitals and traffic tool, not a product analytics tool.

### 11.2 PostHog for Web Analytics

For full product analytics on web, PostHog works on both React Native and Next.js:

```bash
npm install posthog-js
```

```typescript
// src/lib/posthog.ts
import posthog from 'posthog-js';

export function initPostHog() {
  if (typeof window === 'undefined') return; // SSR guard

  posthog.init(process.env.NEXT_PUBLIC_POSTHOG_KEY!, {
    api_host: process.env.NEXT_PUBLIC_POSTHOG_HOST || 'https://us.i.posthog.com',
    person_profiles: 'identified_only',
    capture_pageview: false, // We'll handle this manually for App Router
    capture_pageleave: true,
    session_recording: {
      maskAllInputs: true,
      maskTextContent: false,
    },
  });
}
```

```typescript
// app/providers.tsx
'use client';

import { useEffect } from 'react';
import { usePathname, useSearchParams } from 'next/navigation';
import posthog from 'posthog-js';
import { PostHogProvider as PHProvider } from 'posthog-js/react';
import { initPostHog } from '@/lib/posthog';

export function PostHogProvider({ children }: { children: React.ReactNode }) {
  useEffect(() => {
    initPostHog();
  }, []);

  return <PHProvider client={posthog}>{children}</PHProvider>;
}

/**
 * Track page views in App Router.
 * Place this in your root layout.
 */
export function PostHogPageView() {
  const pathname = usePathname();
  const searchParams = useSearchParams();

  useEffect(() => {
    if (pathname) {
      let url = window.origin + pathname;
      const search = searchParams.toString();
      if (search) {
        url += '?' + search;
      }
      posthog.capture('$pageview', { $current_url: url });
    }
  }, [pathname, searchParams]);

  return null;
}
```

### 11.3 Custom Events in Next.js

```typescript
// Server Component: track events server-side (useful for API routes)
// app/api/purchase/route.ts
import { PostHog } from 'posthog-node';

const posthogServer = new PostHog(process.env.POSTHOG_API_KEY!, {
  host: process.env.POSTHOG_HOST,
});

export async function POST(request: Request) {
  const { userId, amount, currency } = await request.json();

  // Process the purchase...

  // Track server-side (guaranteed delivery, not dependent on client JS)
  posthogServer.capture({
    distinctId: userId,
    event: 'purchase_completed',
    properties: {
      amount,
      currency,
      source: 'web',
    },
  });

  // Ensure events are sent before the function returns
  await posthogServer.shutdown();

  return Response.json({ success: true });
}
```

```typescript
// Client Component: track events from UI interactions
'use client';

import { usePostHog } from 'posthog-js/react';

export function AddToCartButton({ productId, productName, price }: {
  productId: string;
  productName: string;
  price: number;
}) {
  const posthog = usePostHog();

  const handleClick = () => {
    posthog.capture('item_added_to_cart', {
      product_id: productId,
      product_name: productName,
      price,
      currency: 'USD',
    });

    // ... actual add-to-cart logic
  };

  return <button onClick={handleClick}>Add to Cart</button>;
}
```

### 11.4 UTM Tracking in Server Components

```typescript
// app/page.tsx -- Server Component
import { headers } from 'next/headers';

interface UTMParams {
  utm_source?: string;
  utm_medium?: string;
  utm_campaign?: string;
  utm_content?: string;
  utm_term?: string;
}

function getUTMParams(searchParams: URLSearchParams): UTMParams {
  return {
    utm_source: searchParams.get('utm_source') ?? undefined,
    utm_medium: searchParams.get('utm_medium') ?? undefined,
    utm_campaign: searchParams.get('utm_campaign') ?? undefined,
    utm_content: searchParams.get('utm_content') ?? undefined,
    utm_term: searchParams.get('utm_term') ?? undefined,
  };
}

export default async function LandingPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | undefined>>;
}) {
  const params = await searchParams;
  const utmParams = getUTMParams(new URLSearchParams(params as Record<string, string>));

  // Pass UTM params to client component for PostHog tracking
  return (
    <div>
      <UTMTracker params={utmParams} />
      <HeroSection />
      <FeaturesSection />
      {/* ... */}
    </div>
  );
}
```

```typescript
// src/components/UTMTracker.tsx
'use client';

import { useEffect, useRef } from 'react';
import { usePostHog } from 'posthog-js/react';

export function UTMTracker({ params }: { params: Record<string, string | undefined> }) {
  const posthog = usePostHog();
  const tracked = useRef(false);

  useEffect(() => {
    if (tracked.current) return;

    const validParams = Object.fromEntries(
      Object.entries(params).filter(([, v]) => v !== undefined)
    );

    if (Object.keys(validParams).length > 0) {
      posthog.capture('utm_landing', validParams);
      posthog.setPersonProperties(validParams);
      tracked.current = true;
    }
  }, [params, posthog]);

  return null;
}
```

---

## 12. DATA-DRIVEN DEVELOPMENT

### 12.1 The Build-Measure-Learn Cycle

Data-driven development is not "look at a dashboard sometimes." It is a disciplined cycle:

```
         ┌──────────┐
         │  IDEAS    │ ← Informed by data, user feedback, market
         └────┬─────┘
              │
              ▼
         ┌──────────┐
         │  BUILD    │ ← Ship the smallest version that tests the idea
         └────┬─────┘
              │
              ▼
         ┌──────────┐
         │  MEASURE  │ ← Collect analytics, run A/B tests
         └────┬─────┘
              │
              ▼
         ┌──────────┐
         │  LEARN    │ ← Analyze data, draw conclusions, decide next action
         └────┬─────┘
              │
              └──────────→ back to IDEAS
```

### 12.2 Measuring Feature Adoption

Every feature you ship should have a hypothesis and a measurement plan:

```typescript
// src/analytics/feature-tracking.ts

/**
 * Track feature adoption following the standard pattern:
 *
 * 1. feature_exposed   - User saw the feature (could interact with it)
 * 2. feature_activated - User interacted with it for the first time
 * 3. feature_used      - User used it (including repeat usage)
 * 4. feature_retained  - User used it again in a subsequent session
 *
 * This gives you: awareness → activation → usage → retention
 */

import { analyticsService } from './analytics-service';
import AsyncStorage from '@react-native-async-storage/async-storage';

class FeatureTracker {
  private activatedFeatures = new Set<string>();

  async trackExposure(featureName: string, variant?: string) {
    analyticsService.track('feature_exposed', {
      feature_name: featureName,
      feature_variant: variant,
    });
  }

  async trackUsage(featureName: string, properties?: Record<string, unknown>) {
    const isFirstUse = !this.activatedFeatures.has(featureName);

    if (isFirstUse) {
      this.activatedFeatures.add(featureName);
      await AsyncStorage.setItem(
        `@feature_activated_${featureName}`,
        new Date().toISOString()
      );

      analyticsService.track('feature_activated', {
        feature_name: featureName,
        ...properties,
      });
    }

    analyticsService.track('feature_used', {
      feature_name: featureName,
      is_first_use: isFirstUse,
      ...properties,
    });
  }

  /**
   * Example measurement plan for a new "Smart Search" feature:
   *
   * Hypothesis: Smart Search will increase search-to-purchase conversion by 20%
   *
   * Metrics:
   * - feature_exposed: Users who saw the new search bar (should be ~100% if shipped to all)
   * - feature_activated: Users who performed at least one search (target: 40% of exposed)
   * - feature_used count per user: Average searches per active user (target: 3+/week)
   * - search → purchase_completed funnel: conversion rate (target: 15%, up from 12%)
   *
   * Timeline: Measure for 2 weeks with A/B test, then decide ship/iterate/kill
   */
}

export const featureTracker = new FeatureTracker();
```

### 12.3 A/B Test Analysis

When running A/B tests, the analytics setup is critical:

```typescript
// src/experiments/experiment-tracker.ts

import { analyticsService } from '@/analytics/analytics-service';

interface ExperimentConfig {
  experimentId: string;
  experimentName: string;
  variants: string[];
  primaryMetric: string;      // The event name that determines success
  secondaryMetrics?: string[]; // Additional events to monitor
}

/**
 * Track experiment exposure and outcomes.
 *
 * Critical rules:
 * 1. Track exposure when the user SEES the variant, not when assigned
 * 2. Only track exposure once per session to avoid inflating sample size
 * 3. Always include experiment metadata so outcomes can be attributed
 */
class ExperimentTracker {
  private exposedExperiments = new Set<string>();

  trackExposure(config: ExperimentConfig, variant: string) {
    const exposureKey = `${config.experimentId}:${variant}`;
    if (this.exposedExperiments.has(exposureKey)) return;

    this.exposedExperiments.add(exposureKey);

    analyticsService.track('experiment_exposed', {
      experiment_id: config.experimentId,
      experiment_name: config.experimentName,
      variant_name: variant,
      primary_metric: config.primaryMetric,
    });

    // Set as user property so ALL subsequent events carry the experiment context
    analyticsService.setUserProperties({
      [`experiment_${config.experimentId}`]: variant,
    });
  }

  /**
   * When analyzing results, your analytics tool should be able to answer:
   *
   * "For users exposed to experiment X:
   *  - Variant A: N users, M converted (primary metric), P% conversion
   *  - Variant B: N users, M converted, P% conversion
   *  - Statistical significance: 95% confidence that B > A by X%"
   *
   * Amplitude, PostHog, and Mixpanel all have built-in experiment analysis.
   * Firebase A/B Testing does this through Remote Config.
   */
}

export const experimentTracker = new ExperimentTracker();
```

### 12.4 The Feature Decision Framework

Use analytics to make feature decisions systematically:

```
Feature Decision Matrix
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

              High Adoption               Low Adoption
           ┌──────────────────┬──────────────────────┐
    High   │                  │                      │
    Impact │   DOUBLE DOWN    │   INVESTIGATE        │
    on     │                  │                      │
    Metric │   Users love it  │   Valuable but       │
           │   AND it moves   │   undiscovered.      │
           │   the needle.    │   Improve discovery, │
           │   Invest more.   │   onboarding, UX.    │
           │                  │                      │
           ├──────────────────┼──────────────────────┤
    Low    │                  │                      │
    Impact │   MAINTAIN       │   KILL OR PIVOT      │
    on     │                  │                      │
    Metric │   Users enjoy    │   Nobody uses it     │
           │   it but it does │   and it does not    │
           │   not move       │   move the needle.   │
           │   business       │   Remove it. Redirect│
           │   metrics. Keep  │   the effort.        │
           │   but don't      │                      │
           │   invest more.   │                      │
           └──────────────────┴──────────────────────┘
```

The hardest decision is "kill." Teams fall in love with features they built. Analytics gives you the objectivity to say "this feature took 3 weeks to build, 2% of users tried it, and it didn't change retention. We should remove it and simplify the app."

### 12.5 User Feedback Loops

Analytics tells you what users do. It does not tell you why. Combine quantitative data with qualitative feedback:

```typescript
// src/analytics/feedback-loop.ts

import { analyticsService } from './analytics-service';

/**
 * Contextual feedback collection.
 * Trigger based on analytics signals, not random timing.
 */
export const feedbackTriggers = {
  /**
   * After a user completes a key task successfully.
   * They're in a good mood and can articulate what worked.
   */
  afterTaskCompletion: (taskName: string) => {
    analyticsService.track('feedback_prompt_shown', {
      trigger: 'task_completion',
      task_name: taskName,
    });
  },

  /**
   * After a user encounters friction (detected by analytics).
   * E.g., user started checkout but came back to cart 3+ times.
   */
  afterFriction: (frictionType: string) => {
    analyticsService.track('feedback_prompt_shown', {
      trigger: 'friction_detected',
      friction_type: frictionType,
    });
  },

  /**
   * For power users who might be willing to do a detailed interview.
   * Trigger: user has been active 20+ days in the last 30.
   */
  powerUserInterview: () => {
    analyticsService.track('feedback_prompt_shown', {
      trigger: 'power_user_interview',
    });
  },

  /**
   * For at-risk users to understand why they're leaving.
   * Trigger: user's session frequency dropped by 50%+ week-over-week.
   */
  churnRisk: () => {
    analyticsService.track('feedback_prompt_shown', {
      trigger: 'churn_risk',
    });
  },
};
```

---

## 13. COMPLETE IMPLEMENTATION

### 13.1 The Analytics Service (Provider Abstraction Layer)

This is the centerpiece of your analytics architecture. One interface that all your feature code calls. The service handles provider routing, consent checking, super properties, and queuing.

```typescript
// src/analytics/analytics-service.ts

import { EventName, EventProperties, AnalyticsEvents } from './events';
import { ConsentPreferences, consentManager } from './consent-manager';
import { withSuperProperties } from './super-properties';

/**
 * Analytics provider interface.
 * Each provider (Firebase, PostHog, Mixpanel, etc.) implements this.
 */
interface AnalyticsProvider {
  name: string;
  requiresConsent: keyof ConsentPreferences;

  initialize(): Promise<void>;
  track(eventName: string, properties: Record<string, unknown>): void;
  identify(userId: string, traits?: Record<string, unknown>): void;
  setUserProperties(properties: Record<string, unknown>): void;
  screen(screenName: string, properties?: Record<string, unknown>): void;
  reset(): void;
  flush(): Promise<void>;
}

/**
 * Firebase Analytics provider implementation.
 */
class FirebaseAnalyticsProvider implements AnalyticsProvider {
  name = 'firebase';
  requiresConsent: keyof ConsentPreferences = 'analytics';

  private analytics: any = null;

  async initialize() {
    try {
      const { default: firebaseAnalytics } = await import(
        '@react-native-firebase/analytics'
      );
      this.analytics = firebaseAnalytics();
      // Enable analytics collection (can be toggled based on consent)
      await this.analytics.setAnalyticsCollectionEnabled(true);
    } catch (error) {
      console.warn('[Analytics] Firebase Analytics failed to initialize:', error);
    }
  }

  track(eventName: string, properties: Record<string, unknown>) {
    if (!this.analytics) return;

    // Firebase has a 40-char limit on event names and parameter keys
    const sanitizedName = eventName.slice(0, 40);
    const sanitizedProps = Object.fromEntries(
      Object.entries(properties)
        .slice(0, 25) // Firebase limits to 25 custom params
        .map(([key, value]) => [key.slice(0, 40), value])
    );

    this.analytics.logEvent(sanitizedName, sanitizedProps);
  }

  identify(userId: string) {
    if (!this.analytics) return;
    this.analytics.setUserId(userId);
  }

  setUserProperties(properties: Record<string, unknown>) {
    if (!this.analytics) return;

    // Firebase user properties must be strings
    const stringProps = Object.fromEntries(
      Object.entries(properties).map(([key, value]) => [
        key.slice(0, 24), // Firebase: 24 char limit on user property names
        String(value),
      ])
    );

    this.analytics.setUserProperties(stringProps);
  }

  screen(screenName: string, properties?: Record<string, unknown>) {
    if (!this.analytics) return;
    this.analytics.logScreenView({
      screen_name: screenName,
      screen_class: screenName,
      ...properties,
    });
  }

  reset() {
    if (!this.analytics) return;
    this.analytics.setUserId(null);
    this.analytics.resetAnalyticsData();
  }

  async flush() {
    // Firebase batches and sends automatically
  }
}

/**
 * PostHog provider implementation.
 */
class PostHogAnalyticsProvider implements AnalyticsProvider {
  name = 'posthog';
  requiresConsent: keyof ConsentPreferences = 'analytics';

  private posthog: any = null;

  async initialize() {
    try {
      const { default: PostHog } = await import('posthog-react-native');
      // PostHog is typically initialized via PostHogProvider in React
      // This is for imperative access outside React components
      this.posthog = PostHog;
    } catch (error) {
      console.warn('[Analytics] PostHog failed to initialize:', error);
    }
  }

  track(eventName: string, properties: Record<string, unknown>) {
    this.posthog?.capture(eventName, properties);
  }

  identify(userId: string, traits?: Record<string, unknown>) {
    this.posthog?.identify(userId, traits);
  }

  setUserProperties(properties: Record<string, unknown>) {
    this.posthog?.identify(undefined, undefined, properties);
  }

  screen(screenName: string, properties?: Record<string, unknown>) {
    this.posthog?.screen(screenName, properties);
  }

  reset() {
    this.posthog?.reset();
  }

  async flush() {
    await this.posthog?.flush();
  }
}

/**
 * Console/debug provider for development.
 */
class DebugAnalyticsProvider implements AnalyticsProvider {
  name = 'debug';
  requiresConsent: keyof ConsentPreferences = 'necessary'; // Always active

  async initialize() {
    console.log('[Analytics:Debug] Initialized');
  }

  track(eventName: string, properties: Record<string, unknown>) {
    console.log(`[Analytics:Debug] TRACK: ${eventName}`, properties);
  }

  identify(userId: string, traits?: Record<string, unknown>) {
    console.log(`[Analytics:Debug] IDENTIFY: ${userId}`, traits);
  }

  setUserProperties(properties: Record<string, unknown>) {
    console.log('[Analytics:Debug] SET USER PROPS:', properties);
  }

  screen(screenName: string, properties?: Record<string, unknown>) {
    console.log(`[Analytics:Debug] SCREEN: ${screenName}`, properties);
  }

  reset() {
    console.log('[Analytics:Debug] RESET');
  }

  async flush() {
    console.log('[Analytics:Debug] FLUSH');
  }
}

// ============================================================
// THE ANALYTICS SERVICE
// ============================================================

class AnalyticsService {
  private providers: AnalyticsProvider[] = [];
  private initialized = false;
  private eventQueue: Array<{
    type: 'track' | 'identify' | 'screen' | 'userProps';
    args: any[];
  }> = [];

  /**
   * Initialize the analytics service with the desired providers.
   * Call this once at app startup, after loading consent preferences.
   */
  async initialize() {
    if (this.initialized) return;

    // Always add debug provider in development
    if (__DEV__) {
      const debug = new DebugAnalyticsProvider();
      await debug.initialize();
      this.providers.push(debug);
    }

    // Add production providers
    if (!__DEV__) {
      const firebase = new FirebaseAnalyticsProvider();
      await firebase.initialize();
      this.providers.push(firebase);

      const posthog = new PostHogAnalyticsProvider();
      await posthog.initialize();
      this.providers.push(posthog);
    }

    this.initialized = true;

    // Flush queued events
    for (const queued of this.eventQueue) {
      switch (queued.type) {
        case 'track':
          this.track(queued.args[0], queued.args[1]);
          break;
        case 'identify':
          this.identify(queued.args[0], queued.args[1]);
          break;
        case 'screen':
          this.screen(queued.args[0], queued.args[1]);
          break;
        case 'userProps':
          this.setUserProperties(queued.args[0]);
          break;
      }
    }
    this.eventQueue = [];
  }

  /**
   * Track a typed event.
   *
   * Usage:
   *   analyticsService.track('purchase_completed', {
   *     order_id: 'ord_123',
   *     amount: 49.99,
   *     currency: 'USD',
   *     item_count: 3,
   *     payment_method: 'apple_pay',
   *     is_first_purchase: true,
   *   });
   */
  track<E extends EventName>(eventName: E, properties: EventProperties<E>) {
    if (!this.initialized) {
      this.eventQueue.push({ type: 'track', args: [eventName, properties] });
      return;
    }

    const enrichedProperties = withSuperProperties(
      properties as Record<string, unknown>
    );

    for (const provider of this.providers) {
      if (this.isProviderConsented(provider)) {
        try {
          provider.track(eventName, enrichedProperties);
        } catch (error) {
          console.warn(
            `[Analytics] Error tracking ${eventName} to ${provider.name}:`,
            error
          );
        }
      }
    }
  }

  /**
   * Identify a user after signup or login.
   */
  identify(userId: string, traits?: Record<string, unknown>) {
    if (!this.initialized) {
      this.eventQueue.push({ type: 'identify', args: [userId, traits] });
      return;
    }

    for (const provider of this.providers) {
      if (this.isProviderConsented(provider)) {
        try {
          provider.identify(userId, traits);
        } catch (error) {
          console.warn(
            `[Analytics] Error identifying to ${provider.name}:`,
            error
          );
        }
      }
    }
  }

  /**
   * Set persistent user properties.
   */
  setUserProperties(properties: Record<string, unknown>) {
    if (!this.initialized) {
      this.eventQueue.push({ type: 'userProps', args: [properties] });
      return;
    }

    for (const provider of this.providers) {
      if (this.isProviderConsented(provider)) {
        try {
          provider.setUserProperties(properties);
        } catch (error) {
          console.warn(
            `[Analytics] Error setting user props on ${provider.name}:`,
            error
          );
        }
      }
    }
  }

  /**
   * Track a screen view.
   */
  screen(screenName: string, properties?: Record<string, unknown>) {
    if (!this.initialized) {
      this.eventQueue.push({ type: 'screen', args: [screenName, properties] });
      return;
    }

    for (const provider of this.providers) {
      if (this.isProviderConsented(provider)) {
        try {
          provider.screen(screenName, properties);
        } catch (error) {
          console.warn(
            `[Analytics] Error tracking screen on ${provider.name}:`,
            error
          );
        }
      }
    }
  }

  /**
   * Reset user identity (on logout).
   */
  reset() {
    for (const provider of this.providers) {
      try {
        provider.reset();
      } catch (error) {
        console.warn(`[Analytics] Error resetting ${provider.name}:`, error);
      }
    }
  }

  /**
   * Flush all pending events (call before app close or backgrounding).
   */
  async flush() {
    await Promise.allSettled(
      this.providers.map((provider) => provider.flush())
    );
  }

  /**
   * Check if a provider is consented for its category.
   */
  private isProviderConsented(provider: AnalyticsProvider): boolean {
    // Necessary providers always run
    if (provider.requiresConsent === 'necessary') return true;

    return consentManager.hasConsent(provider.requiresConsent);
  }
}

// Singleton instance
export const analyticsService = new AnalyticsService();
```

### 13.2 Type-Safe React Hook

```typescript
// src/hooks/useAnalytics.ts

import { useCallback } from 'react';
import { analyticsService } from '@/analytics/analytics-service';
import { EventName, EventProperties } from '@/analytics/events';

/**
 * Type-safe analytics hook for React components.
 *
 * Usage:
 *   const { track, identify } = useAnalytics();
 *
 *   track('purchase_completed', {
 *     order_id: 'ord_123',     // ✅ Required by type
 *     amount: 49.99,           // ✅ Required by type
 *     currency: 'USD',         // ✅ Required by type
 *     // missing item_count    // ❌ TypeScript error!
 *   });
 */
export function useAnalytics() {
  const track = useCallback(
    <E extends EventName>(eventName: E, properties: EventProperties<E>) => {
      analyticsService.track(eventName, properties);
    },
    []
  );

  const identify = useCallback(
    (userId: string, traits?: Record<string, unknown>) => {
      analyticsService.identify(userId, traits);
    },
    []
  );

  const setUserProperties = useCallback(
    (properties: Record<string, unknown>) => {
      analyticsService.setUserProperties(properties);
    },
    []
  );

  const reset = useCallback(() => {
    analyticsService.reset();
  }, []);

  return {
    track,
    identify,
    setUserProperties,
    reset,
  };
}
```

### 13.3 Putting It All Together: App Initialization

```typescript
// src/analytics/init.ts

import { analyticsService } from './analytics-service';
import { consentManager } from './consent-manager';
import { captureAttribution } from './attribution';
import { configureAnalyticsForATT } from './post-att-strategy';
import { Platform } from 'react-native';

/**
 * Initialize the complete analytics stack.
 * Call this once in your app's root component or entry point.
 *
 * Order of operations:
 * 1. Load consent preferences from storage
 * 2. Initialize analytics service (respects consent)
 * 3. Configure ATT-based behavior (iOS)
 * 4. Capture attribution from deep links
 */
export async function initializeAnalytics() {
  // 1. Load consent
  const consent = await consentManager.load();

  if (!consent) {
    // First launch: user hasn't set preferences yet.
    // Initialize with minimal tracking until they choose.
    // The consent banner will be shown by the UI.
    await consentManager.save({
      necessary: true,
      analytics: false,   // Default: off until consented
      advertising: false,
      thirdParty: false,
    });
  }

  // 2. Initialize analytics providers
  await analyticsService.initialize();

  // 3. Configure ATT behavior (iOS only)
  if (Platform.OS === 'ios') {
    await configureAnalyticsForATT();
  }

  // 4. Capture attribution from initial URL
  await captureAttribution();

  // 5. Listen for consent changes and reconfigure
  consentManager.onChange(async (newPrefs) => {
    // When consent changes, providers automatically check consent
    // before sending events (via isProviderConsented).
    // But if analytics was denied and is now granted, we should
    // re-initialize any providers that weren't active before.
    if (newPrefs.analytics) {
      await analyticsService.initialize();
    }
  });
}
```

Usage in the app root:

```typescript
// app/_layout.tsx
import { useEffect, useState } from 'react';
import { Stack } from 'expo-router';
import { initializeAnalytics } from '@/analytics/init';
import { useScreenTracking } from '@/analytics/screen-tracking';
import { consentManager } from '@/analytics/consent-manager';
import { ConsentBanner } from '@/components/ConsentBanner';

export default function RootLayout() {
  const [analyticsReady, setAnalyticsReady] = useState(false);
  const [showConsentBanner, setShowConsentBanner] = useState(false);

  useEffect(() => {
    async function bootstrap() {
      await initializeAnalytics();
      setAnalyticsReady(true);

      // Check if we need to show consent banner
      const consent = consentManager.getPreferences();
      if (!consent || !consent.analytics) {
        setShowConsentBanner(true);
      }
    }

    bootstrap();
  }, []);

  // Screen tracking (only after analytics is ready)
  useScreenTracking();

  return (
    <>
      <Stack />
      {showConsentBanner && (
        <ConsentBanner
          onComplete={() => setShowConsentBanner(false)}
        />
      )}
    </>
  );
}
```

### 13.4 Testing Your Analytics

Analytics is code. Test it like code:

```typescript
// src/analytics/__tests__/analytics-service.test.ts

import { analyticsService } from '../analytics-service';
import { consentManager } from '../consent-manager';

// Mock providers
const mockTrack = jest.fn();
const mockIdentify = jest.fn();
const mockSetUserProperties = jest.fn();

jest.mock('@react-native-firebase/analytics', () => ({
  __esModule: true,
  default: () => ({
    logEvent: mockTrack,
    setUserId: mockIdentify,
    setUserProperties: mockSetUserProperties,
    setAnalyticsCollectionEnabled: jest.fn(),
    logScreenView: jest.fn(),
    resetAnalyticsData: jest.fn(),
  }),
}));

describe('AnalyticsService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('tracks typed events with correct properties', async () => {
    await consentManager.save({
      necessary: true,
      analytics: true,
      advertising: false,
      thirdParty: false,
    });
    await analyticsService.initialize();

    analyticsService.track('purchase_completed', {
      order_id: 'ord_123',
      amount: 49.99,
      currency: 'USD',
      item_count: 3,
      payment_method: 'apple_pay',
      is_first_purchase: true,
    });

    expect(mockTrack).toHaveBeenCalledWith(
      'purchase_completed',
      expect.objectContaining({
        order_id: 'ord_123',
        amount: 49.99,
        currency: 'USD',
      })
    );
  });

  it('does not track when analytics consent is denied', async () => {
    await consentManager.save({
      necessary: true,
      analytics: false,  // Denied
      advertising: false,
      thirdParty: false,
    });
    await analyticsService.initialize();

    analyticsService.track('screen_viewed', {
      screen_name: 'Home',
      screen_path: '/home',
    });

    expect(mockTrack).not.toHaveBeenCalled();
  });

  it('queues events before initialization and flushes after', async () => {
    // Track before init
    analyticsService.track('app_opened', {
      is_cold_start: true,
    });

    // Events should be queued, not sent
    expect(mockTrack).not.toHaveBeenCalled();

    // Now initialize
    await consentManager.save({
      necessary: true,
      analytics: true,
      advertising: false,
      thirdParty: false,
    });
    await analyticsService.initialize();

    // Queued events should now be sent
    expect(mockTrack).toHaveBeenCalledWith(
      'app_opened',
      expect.objectContaining({ is_cold_start: true })
    );
  });

  it('includes super properties with every event', async () => {
    await consentManager.save({
      necessary: true,
      analytics: true,
      advertising: false,
      thirdParty: false,
    });
    await analyticsService.initialize();

    analyticsService.track('button_tapped', {
      button_name: 'settings',
      screen_name: 'home',
    });

    expect(mockTrack).toHaveBeenCalledWith(
      'button_tapped',
      expect.objectContaining({
        button_name: 'settings',
        screen_name: 'home',
        // Super properties should be present
        platform: expect.any(String),
        app_version: expect.any(String),
      })
    );
  });
});
```

### 13.5 Analytics Debugging Checklist

When analytics is not working (and it will stop working at some point), use this checklist:

```
ANALYTICS DEBUGGING CHECKLIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ Is the analytics service initialized?
  → Check that initializeAnalytics() is called in the root layout
  → Check the console for initialization errors

□ Does the user have consent?
  → Check consentManager.getPreferences()
  → If analytics: false, no events will be sent to analytics providers

□ Is the event name correct?
  → Firebase: max 40 chars, no reserved prefixes (firebase_, google_, ga_)
  → All providers: snake_case only, no spaces

□ Are the event properties valid?
  → Firebase: max 25 custom params, 40 char param names, 100 char string values
  → PostHog/Mixpanel: check for nested objects (some providers flatten)
  → No undefined values (replace with null or omit)

□ Is the event reaching the provider?
  → Firebase: DebugView (enable with -FIRAnalyticsDebugEnabled flag)
  → PostHog: check /activity in your PostHog dashboard
  → Mixpanel: Events tab, real-time view
  → Segment: Debugger tab in Segment dashboard

□ Are you testing in production mode?
  → The debug provider only runs in __DEV__
  → Firebase Analytics batches events (can take minutes to appear)
  → Use Firebase DebugView for real-time validation

□ Is the user identified correctly?
  → Check that identify() is called after signup/login
  → Check that reset() is called on logout
  → Verify the userId matches across providers

□ Network issues?
  → Events are batched and sent periodically
  → Check device connectivity
  → Airplane mode queues events; they send when connection resumes
```

---

## SUMMARY

Analytics, attribution, and data-driven development are not bolt-on afterthoughts. They are systems that require the same architectural care as your state management, your API layer, or your navigation structure.

Here is what matters:

1. **Design your event taxonomy first.** Use noun_verb naming (`purchase_completed`, not `completePurchase`). Type every event. Make properties consistent. Do this before writing a single `track()` call.

2. **Abstract your analytics provider.** Use the analytics service pattern from section 13. Your feature code should never import Firebase or Mixpanel or PostHog directly. When you switch providers (and you will), you change one file instead of 200.

3. **Respect privacy from day one.** Build consent management into your analytics initialization, not as a retrofit. Support GDPR granular consent. Handle ATT gracefully. Collect the minimum data you need. Self-host (PostHog) if you need full data control.

4. **Build funnels that map to your business.** Every conversion path (onboarding, checkout, subscription) should be a measured funnel with typed events. When something drops, the funnel tells you exactly where to look.

5. **Attribution is probabilistic now.** The IDFA-based precision of 2019 is gone. Accept that. Build your attribution strategy on SKAdNetwork, aggregated measurement, first-party data (UTM parameters, deferred deep links), and probabilistic matching. Focus on directional accuracy, not per-user precision.

6. **Use data to make decisions.** Build-measure-learn is not a buzzword; it is a practice. Every feature ships with a hypothesis and a measurement plan. After 2 weeks, look at the data and decide: double down, iterate, or kill.

The goal is not to collect data. The goal is to make better product decisions, faster. Everything in this chapter -- the taxonomy, the abstraction layer, the consent management, the funnel tracking -- serves that goal.

If you set up analytics correctly, your team stops arguing about what to build next based on opinion and starts arguing about what to build next based on evidence. That is a much better argument to have.

---

> **Key takeaway:** Analytics is a system, not a feature. Design the taxonomy before instrumenting. Abstract the provider so you can switch later. Respect privacy from day one. Build funnels that match your business. Use the data to make decisions, not to make dashboards.

---

*Next chapter: We will look at crash monitoring and error tracking -- the operational counterpart to the product analytics covered here. Where analytics tells you what users do, crash monitoring tells you what went wrong.*
