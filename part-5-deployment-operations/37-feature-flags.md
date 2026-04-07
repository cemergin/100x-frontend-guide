<!--
  CHAPTER: 37
  TITLE: Feature Flags, A/B Testing & Experimentation
  PART: V — Deployment & Operations
  PREREQS: Chapters 19, 22
  KEY_TOPICS: LaunchDarkly, Statsig, Unleash, Vercel feature flags, Firebase Remote Config, A/B testing, gradual rollouts, kill switches, feature gates, experiment-driven development, statistical significance
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 37: Feature Flags, A/B Testing & Experimentation

> "If you're not experimenting, you're guessing." — Every data-driven PM you've ever worked with
>
> "Deploy on Friday. Turn it on on Monday." — The feature flag promise

---

<details>
<summary><strong>TL;DR</strong></summary>

- Feature flags decouple deployment from release — ship code behind a flag, turn it on for 1% of users, monitor, ramp up, and kill it instantly if something goes wrong, no redeployment needed
- Statsig offers the best free tier with built-in A/B testing and excellent DX; LaunchDarkly is the enterprise standard with the most features (and highest price); Unleash is the best self-hosted open-source option; Vercel Feature Flags integrate beautifully with Next.js middleware and Edge Config
- Every critical feature in production should have a kill switch — a flag that can disable it in seconds without a deployment
- A/B testing requires statistical discipline: define your hypothesis before the experiment, calculate sample size upfront, don't peek at results, and commit to acting on the data even when it contradicts your intuition
- Stale feature flags are technical debt; implement automated detection and schedule cleanup sprints, or your codebase will become an archaeological dig site of abandoned experiments

</details>

---

## 1. WHY FEATURE FLAGS CHANGE EVERYTHING

### Deploy !== Release

Here's a mental model that will change how you think about shipping software forever: **deployment and release are two completely different things.**

Deployment is putting code on a server. Release is making that code available to users. For most of software engineering history, these happened at the same time. You merged to main, CI built and deployed, and users got the new code. All or nothing. Ship and pray.

Feature flags break this coupling. With feature flags, you deploy code to production that's wrapped in a conditional. The code is *there* — it's on the server, it's in the bundle, it's running — but it's behind a gate. Nobody sees it until you flip a switch.

```typescript
// Without feature flags: deploy === release
// You merge this, it's live. For everyone. Immediately.
function CheckoutPage() {
  return (
    <div>
      <NewPaymentForm />  {/* Hope this works! */}
    </div>
  );
}

// With feature flags: deploy !== release
// You merge this, nobody sees it until you're ready
function CheckoutPage() {
  const showNewPaymentForm = useFeatureFlag('new_payment_form');

  return (
    <div>
      {showNewPaymentForm ? <NewPaymentForm /> : <LegacyPaymentForm />}
    </div>
  );
}
```

This seems simple — it's just an `if` statement, right? But the implications are profound.

### The Five Superpowers of Feature Flags

**Superpower 1: Ship incomplete features to production.**

You don't need to finish a feature in one sprint. You can merge partially-built features behind a flag. No more long-lived feature branches that drift from main for weeks. No more merge conflicts that make you question your career choices. You're always shipping to main, always keeping the codebase in a deployable state, and the half-finished feature is invisible to users.

```typescript
// Sprint 1: Ship the skeleton behind a flag
function NewDashboard() {
  return (
    <div>
      <DashboardHeader />
      {/* TODO: Add charts in sprint 2 */}
      {/* TODO: Add filters in sprint 3 */}
      <Text>Dashboard coming soon</Text>
    </div>
  );
}

// In your main component:
const showNewDashboard = useFeatureFlag('new_dashboard_v2');
// Only internal team members see it. Ship it. Move on.
```

**Superpower 2: Gradual rollouts eliminate big-bang risk.**

Instead of releasing to 100% of users at once, you roll out to 1%, then 5%, then 25%, then 100%. At each stage, you monitor metrics. If something looks wrong at 5%, you roll back to 0% — no deployment needed, no hotfix, no App Store review. Just flip the flag.

**Superpower 3: Kill switches for every critical feature.**

Your new checkout flow has a bug that's causing failed payments? Kill it. In seconds. Not "merge a revert PR, wait for CI, deploy to production" — *seconds*. The flag goes to `false`, the old checkout flow comes back, and you've stopped the bleeding while you figure out what went wrong.

**Superpower 4: User-segment targeting.**

Want to test a feature with your internal team first? Then beta users? Then premium subscribers? Then everyone? Feature flags let you target features at specific user segments without building separate app versions.

**Superpower 5: Data-driven decisions through experimentation.**

A/B testing is just feature flags with metrics. Show variant A to half your users, variant B to the other half, measure which one performs better. No more arguing about whether the blue button or the green button converts better. Just test it.

### The Deployment Risk Matrix

Here's how feature flags change your risk profile:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT RISK MATRIX                            │
│                                                                      │
│                    Without Flags        With Flags                   │
│                    ─────────────        ──────────                   │
│  Deploy freq:      Weekly/biweekly      Multiple times/day          │
│  Blast radius:     100% of users        1% → 5% → 25% → 100%      │
│  Rollback time:    15-60 minutes        < 1 second                  │
│  Rollback method:  Redeploy             Flip a switch               │
│  Feature testing:  Staging only         Production (real data!)     │
│  Incomplete work:  Feature branches     Main branch (flagged)       │
│  Experimentation:  Gut feeling          Statistical evidence        │
│  Deploy on Friday: Absolutely not       Sure, it's behind a flag    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Real-World Impact

Let me tell you about a team I worked with. They shipped a redesigned onboarding flow. Without feature flags, they would have deployed it to all users, discovered on Monday that the drop-off rate had increased by 12%, spent two days reverting and hotfixing, and lost an estimated $40K in new signups during that window.

With feature flags, here's what actually happened: they deployed behind a flag, ramped to 5% of new users, noticed the drop-off increase within 24 hours on a small sample, killed the experiment, iterated on the design, re-ran the experiment, and this time saw a 7% *improvement* in onboarding completion. The total cost of the initial failure? Almost zero — only 5% of users were affected, and only for 24 hours.

That's the difference. Feature flags don't just change how you deploy. They change how you build products.

---

## 2. FEATURE FLAG PROVIDERS — THE DECISION MATRIX

### The Landscape in 2026

The feature flag market has matured significantly. Here are the five providers that matter for frontend teams, and I'm going to be very opinionated about when to use each.

### Statsig — The Best for Most Teams

**What it is:** A feature flagging and experimentation platform with an incredibly generous free tier, built-in A/B testing, and excellent developer experience.

**Why I lead with Statsig:** For 90% of teams reading this book, Statsig is the right choice. The free tier covers up to 500M events per month (which is a lot — most Series A startups won't hit that). It includes feature flags, experiments, and analytics in one platform. The SDK is lightweight, the DX is great, and you don't need to glue together three different tools.

**Best for:**
- Startups and growth-stage companies
- Teams that want experimentation built in (not bolted on)
- React Native and Next.js applications (first-class SDKs)
- Teams that don't want to self-host

**Pricing:** Free tier is genuinely usable for production. Pro starts at reasonable per-seat pricing. Enterprise is custom.

```typescript
// Statsig's DX is clean
import { useGate, useExperiment } from '@statsig/react-bindings';

function PricingPage() {
  // Feature gate (boolean flag)
  const { value: showNewPricing } = useGate('new_pricing_page');

  // Experiment (A/B test with parameters)
  const { config } = useExperiment('pricing_layout_test');
  const layout = config.get('layout', 'grid'); // 'grid' or 'list'
  const ctaText = config.get('cta_text', 'Get Started');

  if (!showNewPricing) return <LegacyPricingPage />;

  return <NewPricingPage layout={layout} ctaText={ctaText} />;
}
```

### LaunchDarkly — The Enterprise Standard

**What it is:** The original feature flag platform. Most features, most integrations, most battle-tested. Also the most expensive.

**Why you'd pick it:** You're a large enterprise with complex targeting needs, regulatory requirements, audit trails, and a budget. LaunchDarkly has the most sophisticated targeting rules, the best audit logging, the deepest integrations with enterprise tools (Jira, ServiceNow, Datadog), and the most mature SDK ecosystem.

**Why you might not:** It's expensive. The per-seat pricing adds up fast, and the experimentation features (LaunchDarkly Experimentation) are an add-on product, not built into the core platform. For a 10-person startup, you're spending thousands per month on something you could get from Statsig for free.

**Best for:**
- Large enterprises (100+ engineers)
- Regulated industries (finance, healthcare) that need audit trails
- Teams already deep in the enterprise tool ecosystem
- Organizations with dedicated release management teams

```typescript
// LaunchDarkly React SDK
import { useFlags, useLDClient } from 'launchdarkly-react-client-sdk';

function FeaturePage() {
  const { newCheckoutFlow, experimentVariant } = useFlags();
  const ldClient = useLDClient();

  // Track custom events for experimentation
  const handlePurchase = (amount: number) => {
    ldClient?.track('purchase', { amount });
  };

  return newCheckoutFlow
    ? <NewCheckout variant={experimentVariant} onPurchase={handlePurchase} />
    : <LegacyCheckout onPurchase={handlePurchase} />;
}
```

### Unleash — The Self-Hosted Champion

**What it is:** An open-source feature flag platform you can self-host. The OSS version is surprisingly capable, and the paid version (Unleash Pro/Enterprise) adds a nicer UI and more advanced features.

**Why you'd pick it:** You need to keep data on your own infrastructure. Maybe you're in a regulated industry, maybe you're in a geography with strict data residency requirements, maybe your security team won't approve a third-party SaaS for feature flag evaluation. Unleash lets you run the entire stack on your own servers.

**Why you might not:** Self-hosting means you own the uptime. Feature flags are mission-critical infrastructure — if your flag service goes down, either all your flags resolve to defaults (which might mean features disappear) or your app can't evaluate flags at all. You need to be confident in your ops capabilities before self-hosting.

**Best for:**
- Teams with strict data residency or compliance requirements
- Organizations that prefer open-source
- Teams with strong DevOps capabilities
- Companies that want to avoid vendor lock-in

```typescript
// Unleash React SDK
import { useFlag, useVariant } from '@unleash/proxy-client-react';

function Dashboard() {
  const newDashboardEnabled = useFlag('new-dashboard');
  const dashboardVariant = useVariant('new-dashboard');

  if (!newDashboardEnabled) return <LegacyDashboard />;

  return <NewDashboard layout={dashboardVariant.payload?.value} />;
}
```

### Vercel Feature Flags — The Next.js Native Option

**What it is:** Feature flag tooling built directly into the Vercel platform, integrated with Edge Config for ultra-fast reads and Next.js middleware for server-side flag evaluation.

**Why you'd pick it:** You're already on Vercel, you're building with Next.js, and you want the simplest possible integration. Vercel Feature Flags evaluate at the edge — before your page even renders — which means you can serve entirely different pages to different users without any client-side flicker. Combined with Edge Config (which has single-digit millisecond reads globally), it's the fastest way to do server-side feature flagging.

**Why you might not:** It only works with Vercel and Next.js. If you're also building a React Native app, you need a second flag provider for mobile. And the experimentation/A/B testing features are more basic than Statsig.

**Best for:**
- Next.js apps deployed on Vercel
- Teams that want zero-flicker server-side flags
- Simple flag use cases (on/off, percentage rollouts)
- Projects where speed of integration matters

```typescript
// Vercel Flags SDK + Edge Config
// flags.ts
import { flag } from '@vercel/flags/next';

export const newHomepage = flag({
  key: 'new-homepage',
  decide() {
    // Simple boolean flag
    return false; // default
  },
});

export const pricingVariant = flag({
  key: 'pricing-variant',
  decide() {
    return 'control';
  },
  options: ['control', 'variant-a', 'variant-b'] as const,
});
```

### Firebase Remote Config — The Free Mobile Option

**What it is:** A free feature flag and remote configuration service from Firebase. Great for mobile apps, integrates with Firebase Analytics and Firebase A/B Testing.

**Why you'd pick it:** You're building a mobile app, you're already using Firebase for other things (analytics, crashlytics, auth), and you want feature flags without adding another vendor. Remote Config is free with no usage limits.

**Why you might not:** The developer experience is clunky compared to Statsig. The SDK is heavier. The console UI is functional but not great. And the flag evaluation happens on the client after an async fetch, which means you need a caching strategy to avoid showing the wrong UI on first render.

**Best for:**
- Mobile apps already using Firebase
- Budget-constrained teams that need free tooling
- Simple remote configuration (not complex experiments)
- Teams that want to A/B test with Firebase A/B Testing

```typescript
// Firebase Remote Config in React Native
import remoteConfig from '@react-native-firebase/remote-config';

// Set defaults (shown before first fetch)
await remoteConfig().setDefaults({
  new_onboarding: false,
  checkout_version: 'v1',
  max_items_in_cart: 50,
});

// Fetch and activate
await remoteConfig().setConfigSettings({
  minimumFetchIntervalMillis: 300000, // 5 minutes in prod
});
await remoteConfig().fetchAndActivate();

// Read values
const showNewOnboarding = remoteConfig().getValue('new_onboarding').asBoolean();
const checkoutVersion = remoteConfig().getValue('checkout_version').asString();
```

### The Decision Matrix

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                     FEATURE FLAG PROVIDER DECISION MATRIX                        │
│                                                                                  │
│  Factor              Statsig    LaunchDarkly  Unleash    Vercel     Firebase RC  │
│  ──────              ───────    ────────────  ───────    ──────     ───────────  │
│  Free tier           Excellent  None          OSS free   Limited    Free         │
│  Pricing             $$         $$$$          $-$$$      $$         Free         │
│  Built-in A/B test   Yes        Add-on        No         Basic      Via FB A/B   │
│  React Native SDK    Excellent  Good          Good       N/A        Good         │
│  Next.js support     Good       Good          Good       Excellent  N/A          │
│  Self-hostable       No         No            Yes        No         No           │
│  Edge evaluation     Yes        Yes           Via proxy  Yes        No           │
│  Targeting rules     Advanced   Most advanced Basic      Basic      Basic        │
│  Audit logging       Yes        Extensive     Basic      Basic      Basic        │
│  Analytics           Built-in   Add-on        None       Basic      Firebase     │
│  Learning curve      Low        Medium        Medium     Very low   Low          │
│  Enterprise ready    Growing    Yes           With Pro   Yes        Limited      │
│                                                                                  │
│  MY RECOMMENDATION:                                                              │
│  ─────────────────                                                               │
│  • Default choice for most teams:     Statsig                                    │
│  • Large enterprise with budget:      LaunchDarkly                               │
│  • Must self-host:                    Unleash                                    │
│  • Next.js on Vercel (web only):      Vercel Flags + Statsig for experiments    │
│  • Mobile-only, already on Firebase:  Firebase Remote Config                     │
│  • Monorepo with RN + Next.js:        Statsig (unified across platforms)        │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### The Uncomfortable Truth About Cost

Let me say something that the feature flag vendors don't want you to hear: **for simple use cases, you can build your own feature flags with a JSON file in Edge Config or a database table.** I'm not saying you *should* — the analytics, targeting, and experiment features of a real platform are worth paying for — but I've seen startups spend $2,000/month on LaunchDarkly when their total flag count was 12 and none of them had targeting rules more complex than "on" or "off."

Start with Statsig's free tier. When you outgrow it, you'll know exactly what features you need, and you can make an informed decision about whether to upgrade to Statsig Pro, switch to LaunchDarkly, or go a different direction.

---

## 3. IMPLEMENTING FLAGS IN REACT NATIVE

Let's build a real feature flag system for a React Native app using Statsig. I'm using Statsig as the primary example because it's the best fit for most teams, but the patterns apply regardless of provider.

### SDK Setup

```bash
# Install Statsig React Native SDK
npx expo install @statsig/expo-bindings @statsig/js-client
```

```typescript
// src/providers/statsig-provider.tsx

import { StatsigProviderExpo } from '@statsig/expo-bindings';
import { StatsigClient } from '@statsig/js-client';
import { useAuth } from '../hooks/use-auth';
import { storage } from '../lib/storage'; // Your MMKV instance

// Initialize the client outside the component tree
const statsigClient = new StatsigClient(
  process.env.EXPO_PUBLIC_STATSIG_CLIENT_KEY!,
  // User object — this is how Statsig knows who the user is
  { userID: '' }, // We'll update this after auth
  {
    // Performance: use a custom storage adapter for local caching
    overrideAdapter: createMMKVStorageAdapter(),
  }
);

// Custom MMKV storage adapter for faster flag reads
function createMMKVStorageAdapter() {
  return {
    getItem(key: string): string | null {
      return storage.getString(`statsig:${key}`) ?? null;
    },
    setItem(key: string, value: string): void {
      storage.set(`statsig:${key}`, value);
    },
    removeItem(key: string): void {
      storage.delete(`statsig:${key}`);
    },
  };
}

export function StatsigProvider({ children }: { children: React.ReactNode }) {
  const { user } = useAuth();

  return (
    <StatsigProviderExpo
      client={statsigClient}
      user={{
        userID: user?.id ?? '',
        email: user?.email,
        custom: {
          plan: user?.subscriptionPlan ?? 'free',
          signupDate: user?.createdAt,
          platform: 'mobile',
        },
      }}
    >
      {children}
    </StatsigProviderExpo>
  );
}
```

### App Integration

```typescript
// App.tsx — wrap your app with the provider

import { StatsigProvider } from './src/providers/statsig-provider';

export default function App() {
  return (
    <StatsigProvider>
      <AuthProvider>
        <NavigationContainer>
          <RootNavigator />
        </NavigationContainer>
      </AuthProvider>
    </StatsigProvider>
  );
}
```

### Type-Safe Flag Definitions

Here's where most tutorials stop and most production apps start having problems. If your flags aren't type-safe, you'll typo a flag name, deploy it, and it'll silently evaluate to `false`. You won't know until someone asks "hey, why isn't the new feature showing up?"

```typescript
// src/flags/definitions.ts

/**
 * Central registry of all feature flags in the app.
 * 
 * RULES:
 * 1. Every flag must be defined here before use
 * 2. Include a description, owner, and expected cleanup date
 * 3. Flags older than their cleanup date get flagged in CI (see Section 9)
 */

export const FLAGS = {
  // ─── Feature Gates (boolean on/off) ───
  new_onboarding_flow: {
    key: 'new_onboarding_flow',
    type: 'gate' as const,
    description: 'Redesigned onboarding with progressive profiling',
    owner: 'growth-team',
    created: '2026-03-01',
    cleanupBy: '2026-06-01',
  },
  new_payment_form: {
    key: 'new_payment_form',
    type: 'gate' as const,
    description: 'Stripe Elements v2 payment form',
    owner: 'payments-team',
    created: '2026-03-15',
    cleanupBy: '2026-05-15',
  },
  dark_mode_v2: {
    key: 'dark_mode_v2',
    type: 'gate' as const,
    description: 'Updated dark mode with OLED black option',
    owner: 'design-systems',
    created: '2026-02-01',
    cleanupBy: '2026-04-30',
  },

  // ─── Experiments (A/B tests with parameters) ───
  checkout_flow_experiment: {
    key: 'checkout_flow_experiment',
    type: 'experiment' as const,
    description: 'Testing single-page vs multi-step checkout',
    owner: 'growth-team',
    created: '2026-03-20',
    cleanupBy: '2026-05-20',
    variants: ['control', 'single_page', 'multi_step'] as const,
  },
  pricing_page_test: {
    key: 'pricing_page_test',
    type: 'experiment' as const,
    description: 'Testing pricing page layouts and CTA copy',
    owner: 'marketing',
    created: '2026-04-01',
    cleanupBy: '2026-06-01',
    variants: ['control', 'variant_a', 'variant_b'] as const,
  },

  // ─── Kill Switches (emergency disable) ───
  kill_live_chat: {
    key: 'kill_live_chat',
    type: 'gate' as const,
    description: 'KILL SWITCH: Disable live chat if provider is down',
    owner: 'platform-team',
    created: '2026-01-01',
    cleanupBy: 'never', // Kill switches are permanent
  },
  kill_push_notifications: {
    key: 'kill_push_notifications',
    type: 'gate' as const,
    description: 'KILL SWITCH: Disable push notifications',
    owner: 'platform-team',
    created: '2026-01-01',
    cleanupBy: 'never',
  },
} as const;

// Type helpers
export type FlagKey = keyof typeof FLAGS;
export type GateKey = {
  [K in FlagKey]: (typeof FLAGS)[K]['type'] extends 'gate' ? K : never;
}[FlagKey];
export type ExperimentKey = {
  [K in FlagKey]: (typeof FLAGS)[K]['type'] extends 'experiment' ? K : never;
}[FlagKey];
```

### Type-Safe Hooks

```typescript
// src/flags/hooks.ts

import { useGate, useExperiment } from '@statsig/react-bindings';
import { FLAGS, type GateKey, type ExperimentKey } from './definitions';

/**
 * Type-safe feature gate hook.
 * Only accepts keys that are defined as 'gate' type in FLAGS.
 */
export function useFeatureFlag(flagKey: GateKey): boolean {
  // Validate the flag exists in our registry (dev-time safety)
  if (__DEV__ && !(flagKey in FLAGS)) {
    console.warn(`[FeatureFlags] Unknown flag key: "${flagKey}". Did you add it to FLAGS?`);
  }

  const { value } = useGate(FLAGS[flagKey].key);
  return value;
}

/**
 * Type-safe experiment hook.
 * Returns the experiment config with typed parameter access.
 */
export function useFeatureExperiment(experimentKey: ExperimentKey) {
  if (__DEV__ && !(experimentKey in FLAGS)) {
    console.warn(`[FeatureFlags] Unknown experiment key: "${experimentKey}". Did you add it to FLAGS?`);
  }

  const { config } = useExperiment(FLAGS[experimentKey].key);

  return {
    variant: config.get('variant', 'control') as string,
    getParam: <T>(key: string, defaultValue: T): T => {
      return config.get(key, defaultValue) as T;
    },
  };
}

/**
 * Kill switch hook — semantically inverted.
 * Returns true when the feature IS available (kill switch is NOT active).
 * Returns false when the feature should be killed.
 */
export function useKillSwitch(flagKey: GateKey): boolean {
  const isKilled = useFeatureFlag(flagKey);
  // Kill switches are "on" when the feature should be DISABLED
  // So we invert: if the kill switch gate is ON, feature is OFF
  return !isKilled;
}
```

### Using Flags in Components

```typescript
// src/features/onboarding/onboarding-screen.tsx

import { useFeatureFlag } from '../../flags/hooks';

export function OnboardingScreen() {
  const showNewOnboarding = useFeatureFlag('new_onboarding_flow');

  if (showNewOnboarding) {
    return <NewOnboardingFlow />;
  }

  return <LegacyOnboardingFlow />;
}

// src/features/checkout/checkout-screen.tsx

import { useFeatureExperiment, useKillSwitch } from '../../flags/hooks';

export function CheckoutScreen() {
  const { variant, getParam } = useFeatureExperiment('checkout_flow_experiment');
  const paymentFormAvailable = useKillSwitch('kill_live_chat');

  const buttonColor = getParam('button_color', '#007AFF');
  const showTrustBadges = getParam('show_trust_badges', true);

  // Render based on experiment variant
  switch (variant) {
    case 'single_page':
      return (
        <SinglePageCheckout
          buttonColor={buttonColor}
          showTrustBadges={showTrustBadges}
        />
      );
    case 'multi_step':
      return (
        <MultiStepCheckout
          buttonColor={buttonColor}
          showTrustBadges={showTrustBadges}
        />
      );
    default:
      return <LegacyCheckout />;
  }
}
```

### Caching Flags Locally with MMKV

This is critical for mobile. Network requests are unreliable on mobile — the user might be in a subway, on a flaky connection, or opening the app for the first time with no cached data. You need flags to resolve instantly from a local cache, not after a network round trip.

```typescript
// src/flags/cache.ts

import { MMKV } from 'react-native-mmkv';

const flagStorage = new MMKV({ id: 'feature-flags' });

interface CachedFlags {
  gates: Record<string, boolean>;
  experiments: Record<string, Record<string, unknown>>;
  fetchedAt: number;
}

const CACHE_KEY = 'cached_flag_values';
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

export function getCachedFlags(): CachedFlags | null {
  const raw = flagStorage.getString(CACHE_KEY);
  if (!raw) return null;

  try {
    const cached: CachedFlags = JSON.parse(raw);

    // Check if cache is still fresh
    if (Date.now() - cached.fetchedAt > CACHE_TTL) {
      return null; // Stale cache — will be refreshed
    }

    return cached;
  } catch {
    return null;
  }
}

export function setCachedFlags(flags: CachedFlags): void {
  flagStorage.set(CACHE_KEY, JSON.stringify({
    ...flags,
    fetchedAt: Date.now(),
  }));
}

export function clearFlagCache(): void {
  flagStorage.delete(CACHE_KEY);
}
```

### Handling Flag Updates in Real-Time

Statsig supports real-time flag updates through their SDK, but for mobile apps, you need to be smart about *when* you re-evaluate flags. Changing a flag while the user is mid-action can be jarring.

```typescript
// src/flags/flag-sync.ts

import { useEffect, useRef } from 'react';
import { AppState, type AppStateStatus } from 'react-native';
import { useStatsigClient } from '@statsig/react-bindings';

/**
 * Syncs feature flags when the app comes to the foreground.
 * 
 * We DON'T update flags in real-time while the user is actively
 * using the app — that would cause jarring UI changes. Instead,
 * we refresh flags on foreground transitions, which is the natural
 * point where users expect the app might look slightly different.
 */
export function useFlagSync() {
  const { client } = useStatsigClient();
  const appState = useRef(AppState.currentState);

  useEffect(() => {
    const subscription = AppState.addEventListener(
      'change',
      (nextState: AppStateStatus) => {
        // Only refresh when coming FROM background TO foreground
        if (
          appState.current.match(/inactive|background/) &&
          nextState === 'active'
        ) {
          // Refresh flags in the background — don't block the UI
          client.updateUserAsync().catch((error) => {
            console.warn('[FeatureFlags] Failed to refresh flags:', error);
            // Non-fatal — we'll use cached values
          });
        }

        appState.current = nextState;
      }
    );

    return () => subscription.remove();
  }, [client]);
}

// Use it in your root component:
// function App() {
//   useFlagSync();
//   return <RootNavigator />;
// }
```

---

## 4. IMPLEMENTING FLAGS IN NEXT.JS

Next.js has a unique advantage for feature flags: you can evaluate flags on the server, before the page ever renders. No flicker. No loading states. The user sees the correct variant from the first paint. This is a superpower that client-side flag evaluation can't match.

### Vercel Feature Flags with the Flags SDK

The `@vercel/flags` SDK integrates with Next.js middleware and Edge Config for blazing-fast server-side flag evaluation.

```bash
npm install @vercel/flags @vercel/edge-config
```

### Defining Flags

```typescript
// flags.ts (root of your Next.js app)

import { flag, dedupe } from '@vercel/flags/next';
import { getEdgeConfigValue } from './lib/edge-config';
import { getCurrentUser } from './lib/auth';

// Deduplicate the user fetch across multiple flag evaluations
const getUser = dedupe(async () => getCurrentUser());

/**
 * Simple boolean flag — read from Edge Config
 * Edge Config reads are < 1ms at the edge
 */
export const showNewHomepage = flag({
  key: 'new-homepage',
  description: 'Show the redesigned homepage',
  async decide() {
    const value = await getEdgeConfigValue('new-homepage');
    return value === true;
  },
  defaultValue: false,
});

/**
 * Percentage rollout flag
 */
export const showNewCheckout = flag({
  key: 'new-checkout',
  description: 'New checkout flow (gradual rollout)',
  async decide() {
    const user = await getUser();
    if (!user) return false;

    const rolloutPercentage = await getEdgeConfigValue('new-checkout-rollout');

    // Deterministic: same user always gets the same result
    const hash = simpleHash(user.id + 'new-checkout');
    const bucket = hash % 100;

    return bucket < (rolloutPercentage ?? 0);
  },
  defaultValue: false,
});

/**
 * Multi-variant experiment flag
 */
export const pricingVariant = flag({
  key: 'pricing-variant',
  description: 'Pricing page A/B test',
  options: ['control', 'annual-first', 'social-proof'] as const,
  async decide() {
    const user = await getUser();
    if (!user) return 'control';

    const hash = simpleHash(user.id + 'pricing-variant');
    const bucket = hash % 100;

    if (bucket < 33) return 'control';
    if (bucket < 66) return 'annual-first';
    return 'social-proof';
  },
  defaultValue: 'control',
});

// Simple deterministic hash for consistent bucketing
function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash |= 0;
  }
  return Math.abs(hash);
}
```

### Middleware-Based Flag Evaluation

This is the killer feature. By evaluating flags in middleware, you can rewrite requests *before* they hit your page. Different users see different pages, with zero client-side JavaScript, zero flicker, and zero layout shift.

```typescript
// middleware.ts

import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { showNewHomepage, showNewCheckout } from './flags';

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Evaluate flags at the edge
  if (pathname === '/') {
    const useNewHomepage = await showNewHomepage();

    if (useNewHomepage) {
      // Rewrite to the new homepage — the URL doesn't change,
      // but the user sees a completely different page
      return NextResponse.rewrite(new URL('/new-homepage', request.url));
    }
  }

  if (pathname === '/checkout') {
    const useNewCheckout = await showNewCheckout();

    if (useNewCheckout) {
      return NextResponse.rewrite(new URL('/new-checkout', request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/', '/checkout', '/pricing'],
};
```

### Server Component Flag Checks

For more granular control within a page, you can evaluate flags directly in Server Components:

```typescript
// app/pricing/page.tsx

import { pricingVariant } from '../../flags';

export default async function PricingPage() {
  const variant = await pricingVariant();

  return (
    <main>
      <h1>Pricing</h1>
      {variant === 'control' && <StandardPricingGrid />}
      {variant === 'annual-first' && <AnnualFirstPricingGrid />}
      {variant === 'social-proof' && <SocialProofPricingGrid />}
    </main>
  );
}

// Each variant is a separate component — clean separation
function StandardPricingGrid() {
  return (
    <div className="grid grid-cols-3 gap-8">
      <PricingCard plan="starter" price={9} period="monthly" />
      <PricingCard plan="pro" price={29} period="monthly" featured />
      <PricingCard plan="enterprise" price={99} period="monthly" />
    </div>
  );
}

function AnnualFirstPricingGrid() {
  // Show annual prices first (hypothesis: higher conversion to annual)
  return (
    <div className="grid grid-cols-3 gap-8">
      <PricingCard plan="starter" price={90} period="annual" discount="Save 17%" />
      <PricingCard plan="pro" price={290} period="annual" discount="Save 17%" featured />
      <PricingCard plan="enterprise" price={990} period="annual" discount="Save 17%" />
    </div>
  );
}

function SocialProofPricingGrid() {
  // Add social proof elements (hypothesis: trust increases conversion)
  return (
    <div className="grid grid-cols-3 gap-8">
      <PricingCard plan="starter" price={9} period="monthly" userCount="2,340 teams" />
      <PricingCard plan="pro" price={29} period="monthly" userCount="8,120 teams" featured />
      <PricingCard plan="enterprise" price={99} period="monthly" userCount="890 teams" />
    </div>
  );
}
```

### Edge Config for Ultra-Fast Flag Reads

Edge Config is Vercel's globally distributed key-value store optimized for reads. It's not a database — it's a config store with single-digit millisecond reads from every edge location worldwide. Perfect for feature flags.

```typescript
// lib/edge-config.ts

import { createClient } from '@vercel/edge-config';

const edgeConfig = createClient(process.env.EDGE_CONFIG);

/**
 * Read a feature flag value from Edge Config.
 * These reads are < 1ms at the edge — faster than a database query
 * by orders of magnitude.
 */
export async function getEdgeConfigValue<T>(key: string): Promise<T | undefined> {
  try {
    return await edgeConfig.get<T>(key);
  } catch (error) {
    console.error(`[EdgeConfig] Failed to read key "${key}":`, error);
    return undefined;
  }
}

/**
 * Read multiple flag values in a single request.
 * More efficient than multiple individual reads.
 */
export async function getEdgeConfigValues<T extends Record<string, unknown>>(
  keys: string[]
): Promise<Partial<T>> {
  try {
    const items = await edgeConfig.getAll<T>(keys);
    return items ?? {};
  } catch (error) {
    console.error('[EdgeConfig] Failed to read keys:', error);
    return {};
  }
}
```

### Combining Vercel Flags with Statsig

For the best of both worlds — Vercel's edge evaluation speed with Statsig's experimentation engine:

```typescript
// flags.ts — hybrid approach

import { flag } from '@vercel/flags/next';
import Statsig from 'statsig-node';

// Initialize Statsig server SDK (for Next.js server-side)
let statsigInitialized = false;

async function ensureStatsigInit() {
  if (!statsigInitialized) {
    await Statsig.initialize(process.env.STATSIG_SERVER_KEY!);
    statsigInitialized = true;
  }
}

/**
 * Use Vercel Flags as the evaluation framework,
 * but pull targeting/experiment data from Statsig.
 */
export const checkoutExperiment = flag({
  key: 'checkout-experiment',
  description: 'Checkout flow A/B test powered by Statsig',
  options: ['control', 'single-page', 'express'] as const,
  async decide() {
    await ensureStatsigInit();

    const user = await getCurrentUser();
    if (!user) return 'control';

    const statsigUser = {
      userID: user.id,
      email: user.email,
      custom: { plan: user.plan },
    };

    const experiment = Statsig.getExperimentSync(
      statsigUser,
      'checkout_flow_experiment'
    );

    return experiment.get('variant', 'control');
  },
  defaultValue: 'control',
});
```

---

## 5. GRADUAL ROLLOUTS

### The Golden Rule of Rollouts

**Never go from 0% to 100%.** Ever. I don't care how confident you are. I don't care how much testing you've done. I don't care if the CEO is breathing down your neck. You roll out gradually, you monitor at each stage, and you only proceed when the data says it's safe.

Here's why: staging environments lie. They lie about performance under load. They lie about edge cases in user data. They lie about interactions with other features. They lie about device diversity. The only environment that tells the truth is production, with real users, real data, and real devices.

### The Standard Rollout Ladder

```
┌──────────────────────────────────────────────────────────────────────┐
│                    THE STANDARD ROLLOUT LADDER                       │
│                                                                      │
│  Stage 0: Internal Team (dogfooding)                                │
│  ├── Who: Your engineering and product team                         │
│  ├── Duration: 2-5 days                                             │
│  ├── Monitor: Errors, crashes, functional correctness               │
│  ├── Gate: No critical bugs found by team                           │
│  └── Next: If clean, proceed to Stage 1                             │
│                                                                      │
│  Stage 1: 1% of users                                               │
│  ├── Who: Random 1% (deterministic by user ID)                     │
│  ├── Duration: 24-48 hours                                          │
│  ├── Monitor: Error rates, performance metrics, key business metrics│
│  ├── Gate: Error rate within 10% of baseline                        │
│  └── Next: If metrics stable, proceed to Stage 2                    │
│                                                                      │
│  Stage 2: 5% of users                                               │
│  ├── Who: Random 5%                                                 │
│  ├── Duration: 24-48 hours                                          │
│  ├── Monitor: All Stage 1 metrics + conversion funnels              │
│  ├── Gate: No statistically significant metric regressions          │
│  └── Next: If metrics stable, proceed to Stage 3                    │
│                                                                      │
│  Stage 3: 25% of users                                              │
│  ├── Who: Random 25%                                                │
│  ├── Duration: 48-72 hours                                          │
│  ├── Monitor: All metrics + load impact                             │
│  ├── Gate: No performance degradation at scale                      │
│  └── Next: If metrics stable, proceed to Stage 4                    │
│                                                                      │
│  Stage 4: 50% of users                                              │
│  ├── Who: Random 50%                                                │
│  ├── Duration: 48-72 hours                                          │
│  ├── Monitor: All metrics (this is your A/B test window!)           │
│  ├── Gate: Feature metrics ≥ baseline                               │
│  └── Next: If positive or neutral, proceed to Stage 5               │
│                                                                      │
│  Stage 5: 100% of users                                             │
│  ├── Who: Everyone                                                  │
│  ├── Duration: 1-2 weeks                                            │
│  ├── Monitor: All metrics                                           │
│  ├── Gate: Stable for 1+ week                                       │
│  └── Next: Clean up the flag (Section 9)                            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Implementing Percentage-Based Rollouts

The key to percentage-based rollouts is **deterministic bucketing**. A user must always land in the same bucket — if I'm in the 1% group today, I should still be in it tomorrow. Otherwise, users get a jarring experience where features appear and disappear randomly.

```typescript
// src/flags/rollout.ts

import { createHash } from 'crypto';

/**
 * Deterministic percentage-based rollout.
 * 
 * Given a user ID and a flag key, returns a number 0-99.
 * The same user + flag combination always returns the same number.
 * Different flags give different buckets (so being in 1% for flag A
 * doesn't mean you're in 1% for flag B).
 */
function getUserBucket(userId: string, flagKey: string): number {
  const hash = createHash('sha256')
    .update(`${userId}:${flagKey}`)
    .digest('hex');

  // Take the first 8 hex chars and convert to a number
  const num = parseInt(hash.substring(0, 8), 16);

  // Modulo 100 gives us a number 0-99
  return num % 100;
}

/**
 * Check if a user should see a feature at a given rollout percentage.
 */
export function isUserInRollout(
  userId: string,
  flagKey: string,
  rolloutPercentage: number
): boolean {
  if (rolloutPercentage <= 0) return false;
  if (rolloutPercentage >= 100) return true;

  const bucket = getUserBucket(userId, flagKey);
  return bucket < rolloutPercentage;
}

// Usage:
// isUserInRollout('user-123', 'new_checkout', 5)  // true if user is in 5%
// isUserInRollout('user-123', 'new_checkout', 25) // true if user is in 25%
// The 5% group is always a SUBSET of the 25% group
```

### User-Segment Rollouts

Sometimes you want more control than just random percentages:

```typescript
// src/flags/segments.ts

type UserSegment = 
  | 'internal'     // Your team
  | 'beta'         // Beta testers who opted in
  | 'premium'      // Paying customers
  | 'free'         // Free tier users
  | 'new'          // Signed up in last 30 days
  | 'all';         // Everyone

interface SegmentRolloutConfig {
  segments: UserSegment[];
  percentage?: number; // Optional: percentage within segment
}

/**
 * Segment-based rollout with optional percentage within segments.
 */
export function isUserInSegmentRollout(
  user: {
    id: string;
    isInternal: boolean;
    isBeta: boolean;
    plan: 'free' | 'premium' | 'enterprise';
    createdAt: Date;
  },
  flagKey: string,
  config: SegmentRolloutConfig
): boolean {
  // Check if user is in any of the target segments
  const isInSegment = config.segments.some((segment) => {
    switch (segment) {
      case 'internal':
        return user.isInternal;
      case 'beta':
        return user.isBeta;
      case 'premium':
        return user.plan === 'premium' || user.plan === 'enterprise';
      case 'free':
        return user.plan === 'free';
      case 'new':
        const thirtyDaysAgo = Date.now() - 30 * 24 * 60 * 60 * 1000;
        return user.createdAt.getTime() > thirtyDaysAgo;
      case 'all':
        return true;
      default:
        return false;
    }
  });

  if (!isInSegment) return false;

  // If no percentage specified, all users in segment see it
  if (config.percentage === undefined || config.percentage >= 100) return true;

  // Otherwise, apply percentage-based rollout within the segment
  return isUserInRollout(user.id, flagKey, config.percentage);
}

// Example rollout strategy:
// Week 1: Internal team only
// { segments: ['internal'] }
//
// Week 2: Internal + beta testers
// { segments: ['internal', 'beta'] }
//
// Week 3: All premium users, 10% of free users
// { segments: ['premium'] } + { segments: ['free'], percentage: 10 }
//
// Week 4: Everyone
// { segments: ['all'] }
```

### Geographic Rollouts

For apps serving multiple regions, geographic rollouts let you limit blast radius by geography:

```typescript
// In your flag evaluation (server-side)
export const newFeature = flag({
  key: 'new-feature',
  async decide() {
    const user = await getUser();
    const geoData = await getGeoFromRequest(); // From Vercel headers or IP lookup

    // Roll out to US first, then EU, then rest of world
    const regionRollout: Record<string, number> = {
      US: 100,   // Fully rolled out in US
      EU: 25,    // 25% in EU
      APAC: 0,   // Not yet in APAC
    };

    const region = getRegion(geoData.country);
    const percentage = regionRollout[region] ?? 0;

    if (!user) return percentage >= 100;
    return isUserInRollout(user.id, 'new-feature', percentage);
  },
  defaultValue: false,
});
```

### Monitoring at Each Stage

At every rollout stage, you should be watching:

```typescript
// src/flags/rollout-monitoring.ts

/**
 * Metrics to monitor at each rollout stage.
 * Set up alerts in your monitoring tool (Sentry, Datadog, etc.)
 * for each of these when ramping up a flag.
 */
export const ROLLOUT_MONITORING_CHECKLIST = {
  stage1_1percent: {
    duration: '24-48 hours',
    metrics: [
      'Error rate (should be within 10% of baseline)',
      'Crash-free rate (should not drop)',
      'API latency for affected endpoints',
      'Client-side performance (TTI, FCP for affected screens)',
    ],
    alertThresholds: {
      errorRateIncrease: '> 10% above baseline',
      crashFreeRateDrop: '> 0.1% drop',
      latencyIncrease: '> 200ms p95 increase',
    },
  },
  stage2_5percent: {
    duration: '24-48 hours',
    metrics: [
      'All Stage 1 metrics',
      'Conversion funnel drop-off rates',
      'Feature-specific engagement metrics',
      'Support ticket volume',
    ],
    alertThresholds: {
      conversionDrop: '> 5% drop in primary conversion',
      supportTicketSpike: '> 2x normal volume',
    },
  },
  stage3_25percent: {
    duration: '48-72 hours',
    metrics: [
      'All previous metrics',
      'Server load / infrastructure costs',
      'Database query patterns (new queries? slow queries?)',
      'Third-party API rate limits',
    ],
    alertThresholds: {
      serverLoadIncrease: '> 25% CPU/memory increase',
      newSlowQueries: 'Any query > 1s',
    },
  },
  stage4_50percent: {
    duration: '48-72 hours',
    metrics: [
      'All previous metrics',
      'A/B comparison: feature vs control group',
      'Revenue impact (if applicable)',
      'User retention (if long enough window)',
    ],
    decisionCriteria: {
      proceed: 'Feature metrics >= control, no regressions',
      pause: 'Mixed signals, need more data',
      rollback: 'Clear regression in any critical metric',
    },
  },
} as const;
```

---

## 6. A/B TESTING

### The Scientific Method for Product Decisions

A/B testing is not "show two things and pick the one you like." It's the scientific method applied to product development. And like the scientific method, it requires discipline, patience, and a willingness to be wrong.

Here's the process:

```
┌──────────────────────────────────────────────────────────────────────┐
│                   THE A/B TESTING PROCESS                            │
│                                                                      │
│  1. HYPOTHESIS                                                       │
│     "If we [change X], then [metric Y] will [improve/decrease]     │
│      because [reasoning Z]."                                         │
│                                                                      │
│  2. DESIGN                                                           │
│     - Define control and variant(s)                                 │
│     - Choose primary metric (ONE — not three)                       │
│     - Choose guardrail metrics (things that must NOT regress)       │
│     - Calculate required sample size                                │
│     - Define experiment duration                                    │
│                                                                      │
│  3. IMPLEMENT                                                        │
│     - Build the variant behind a feature flag                       │
│     - Set up event tracking for all metrics                         │
│     - QA both control and variant paths                             │
│                                                                      │
│  4. RUN                                                              │
│     - Start the experiment                                          │
│     - DO NOT PEEK AT RESULTS (seriously)                            │
│     - Wait for the predetermined duration                           │
│     - Monitor guardrail metrics only                                │
│                                                                      │
│  5. ANALYZE                                                          │
│     - Is the result statistically significant?                      │
│     - What's the confidence interval?                               │
│     - Did any guardrail metrics regress?                            │
│                                                                      │
│  6. DECIDE                                                           │
│     - Significant positive: Ship it                                 │
│     - Significant negative: Kill it, learn from it                  │
│     - Not significant: Consider if the change is worth the code     │
│       complexity, or kill it                                        │
│                                                                      │
│  7. CLEAN UP                                                         │
│     - Remove the flag and losing variant code                       │
│     - Document the result                                           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Writing a Good Hypothesis

Bad hypotheses lead to useless experiments. Here's the difference:

**Bad hypothesis:** "The new checkout page will be better."
- Better than what? Better how? What does "better" mean?

**Good hypothesis:** "If we reduce the checkout flow from 3 steps to 1 step, then checkout completion rate will increase by at least 5%, because we're reducing the number of abandonment points."

A good hypothesis has:
1. A specific change (what you're doing)
2. A specific metric (what you're measuring)
3. A specific target (how much improvement you expect)
4. A rationale (why you believe this will work)

### Setting Up an Experiment with Statsig

```typescript
// 1. Define the experiment in code

// src/experiments/checkout-experiment.ts

import { useExperiment } from '@statsig/react-bindings';

/**
 * EXPERIMENT: Checkout Flow Simplification
 * 
 * HYPOTHESIS: Reducing checkout from 3 steps to 1 step will increase
 * checkout completion rate by ≥ 5%.
 * 
 * PRIMARY METRIC: checkout_completed (conversion event)
 * GUARDRAIL METRICS: 
 *   - payment_errors (must not increase)
 *   - average_order_value (must not decrease by > 3%)
 *   - customer_support_tickets (must not increase by > 10%)
 * 
 * SAMPLE SIZE: ~10,000 users per variant (calculated below)
 * DURATION: 14 days minimum
 * 
 * OWNER: growth-team
 * START DATE: 2026-04-07
 * END DATE: 2026-04-21
 */
export function useCheckoutExperiment() {
  const { config } = useExperiment('checkout_simplification_v1');

  return {
    variant: config.get('variant', 'control') as 'control' | 'single_page',
    // Experiment parameters that might differ between variants
    showProgressBar: config.get('show_progress_bar', true) as boolean,
    showOrderSummary: config.get('show_order_summary', 'sidebar') as
      | 'sidebar'
      | 'inline'
      | 'collapsible',
  };
}
```

```typescript
// 2. Track events that feed into metrics

// src/analytics/experiment-events.ts

import Statsig from '@statsig/js-client';

/**
 * Track experiment-related events.
 * These events power the metrics that determine if the experiment wins.
 */
export const experimentEvents = {
  /** User started the checkout process */
  checkoutStarted: (metadata: { itemCount: number; cartValue: number }) => {
    Statsig.logEvent('checkout_started', undefined, metadata);
  },

  /** User completed a checkout step (for multi-step variant) */
  checkoutStepCompleted: (step: number, totalSteps: number) => {
    Statsig.logEvent('checkout_step_completed', String(step), {
      step,
      totalSteps,
    });
  },

  /** User completed checkout successfully */
  checkoutCompleted: (metadata: {
    orderValue: number;
    itemCount: number;
    paymentMethod: string;
    timeToCompleteMs: number;
  }) => {
    Statsig.logEvent('checkout_completed', undefined, {
      ...metadata,
      value: metadata.orderValue, // Statsig uses 'value' for metric calculations
    });
  },

  /** User abandoned checkout */
  checkoutAbandoned: (metadata: {
    step: number;
    reason?: string;
    cartValue: number;
  }) => {
    Statsig.logEvent('checkout_abandoned', undefined, metadata);
  },

  /** Payment error occurred */
  paymentError: (metadata: { errorCode: string; errorMessage: string }) => {
    Statsig.logEvent('payment_error', metadata.errorCode, metadata);
  },
};
```

```typescript
// 3. Implement the experiment in the UI

// src/features/checkout/checkout-screen.tsx

import { useCheckoutExperiment } from '../../experiments/checkout-experiment';
import { experimentEvents } from '../../analytics/experiment-events';
import { useEffect, useRef } from 'react';

export function CheckoutScreen() {
  const { variant, showProgressBar, showOrderSummary } = useCheckoutExperiment();
  const startTime = useRef(Date.now());

  useEffect(() => {
    experimentEvents.checkoutStarted({
      itemCount: cart.items.length,
      cartValue: cart.total,
    });
  }, []);

  const handleComplete = (orderValue: number) => {
    experimentEvents.checkoutCompleted({
      orderValue,
      itemCount: cart.items.length,
      paymentMethod: selectedPaymentMethod,
      timeToCompleteMs: Date.now() - startTime.current,
    });
  };

  if (variant === 'single_page') {
    return (
      <SinglePageCheckout
        showOrderSummary={showOrderSummary}
        onComplete={handleComplete}
      />
    );
  }

  return (
    <MultiStepCheckout
      showProgressBar={showProgressBar}
      showOrderSummary={showOrderSummary}
      onStepComplete={(step, total) =>
        experimentEvents.checkoutStepCompleted(step, total)
      }
      onComplete={handleComplete}
    />
  );
}
```

### Sample Size Calculations

This is where most teams get sloppy. You can't run an experiment for two days on 200 users and declare a winner. You need statistical rigor.

```typescript
// src/experiments/sample-size-calculator.ts

/**
 * Calculate the required sample size for an A/B test.
 * 
 * Uses the standard formula for comparing two proportions.
 * This is a simplified version — for complex experiments,
 * use Statsig's built-in power calculator or an online tool
 * like Evan Miller's sample size calculator.
 */
export function calculateSampleSize(params: {
  /** Current conversion rate (e.g., 0.10 for 10%) */
  baselineConversionRate: number;
  /** Minimum detectable effect (e.g., 0.05 for 5% relative improvement) */
  minimumDetectableEffect: number;
  /** Statistical significance level (typically 0.05) */
  alpha?: number;
  /** Statistical power (typically 0.80) */
  power?: number;
}): { perVariant: number; total: number; estimatedDays: number } {
  const {
    baselineConversionRate: p1,
    minimumDetectableEffect: mde,
    alpha = 0.05,
    power = 0.80,
  } = params;

  // Target conversion rate
  const p2 = p1 * (1 + mde);

  // Z-scores for alpha and power
  const zAlpha = getZScore(1 - alpha / 2); // 1.96 for alpha = 0.05
  const zBeta = getZScore(power);          // 0.84 for power = 0.80

  // Pooled standard deviation
  const pBar = (p1 + p2) / 2;
  const numerator = (zAlpha * Math.sqrt(2 * pBar * (1 - pBar)) +
    zBeta * Math.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2;
  const denominator = (p2 - p1) ** 2;

  const perVariant = Math.ceil(numerator / denominator);

  return {
    perVariant,
    total: perVariant * 2,
    // Rough estimate assuming 1000 daily active users hitting the feature
    estimatedDays: Math.ceil((perVariant * 2) / 1000),
  };
}

function getZScore(p: number): number {
  // Approximation of inverse normal CDF
  // Good enough for sample size calculations
  if (p === 0.975) return 1.96;
  if (p === 0.80) return 0.84;
  if (p === 0.90) return 1.28;
  if (p === 0.95) return 1.645;
  if (p === 0.99) return 2.326;

  // Rational approximation for other values
  const a = [
    -3.969683028665376e1, 2.209460984245205e2, -2.759285104469687e2,
    1.383577518672690e2, -3.066479806614716e1, 2.506628277459239e0,
  ];
  const b = [
    -5.447609879822406e1, 1.615858368580409e2, -1.556989798598866e2,
    6.680131188771972e1, -1.328068155288572e1,
  ];
  
  const t = p < 0.5 ? p : 1 - p;
  const s = Math.sqrt(-2 * Math.log(t));
  
  let z = (((((a[0] * s + a[1]) * s + a[2]) * s + a[3]) * s + a[4]) * s + a[5]) /
    ((((b[0] * s + b[1]) * s + b[2]) * s + b[3]) * s + b[4]) * s + 1);
  
  return p < 0.5 ? -z : z;
}

// Example calculations:
//
// Scenario: Checkout conversion rate is 10%, want to detect 5% relative improvement
// calculateSampleSize({
//   baselineConversionRate: 0.10,
//   minimumDetectableEffect: 0.05,
// })
// → { perVariant: ~31,000, total: ~62,000, estimatedDays: ~62 }
//
// Scenario: CTA click rate is 3%, want to detect 10% relative improvement
// calculateSampleSize({
//   baselineConversionRate: 0.03,
//   minimumDetectableEffect: 0.10,
// })
// → { perVariant: ~115,000, total: ~230,000, estimatedDays: ~230 }
//
// Key insight: small baseline rates and small effects need MUCH bigger samples.
// This is why "should the button be blue or green" tests often don't reach significance.
```

### The Cardinal Sin: Peeking

Let me explain the single biggest mistake teams make with A/B testing: **peeking at results before the experiment is complete.**

Here's why it's a problem. Statistical significance calculations assume you look at the data *once*, at the end of the experiment. If you look every day — "oh, variant B is up 8% today!" — you're running multiple comparisons, which inflates your false positive rate dramatically.

The math: if you peek at results daily for 14 days with a p < 0.05 threshold, your actual false positive rate isn't 5%. It's closer to 25-30%. That means you'll ship a "winning" variant that actually has no effect roughly one in four times.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     THE PEEKING PROBLEM                              │
│                                                                      │
│  Day  │ P-value │ "Significant?" │ What you should do               │
│  ─────│─────────│────────────────│──────────────────                 │
│   1   │  0.12   │  No            │  Don't look                      │
│   2   │  0.03   │  "Yes!" (*)    │  Don't look                      │
│   3   │  0.08   │  No            │  Don't look                      │
│   4   │  0.04   │  "Yes!" (*)    │  Don't look                      │
│   5   │  0.15   │  No            │  Don't look                      │
│   ...                                                                │
│  14   │  0.07   │  No            │  NOW you can look. Result: null. │
│                                                                      │
│  (*) If you had called the experiment on Day 2 or Day 4, you        │
│      would have shipped a false positive. The early "significance"   │
│      was just noise.                                                 │
│                                                                      │
│  SOLUTION: Use sequential testing (Statsig does this automatically) │
│  or just DON'T LOOK until the experiment window is complete.         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Statsig and most modern experimentation platforms use **sequential testing** methods that account for continuous monitoring, which somewhat mitigates the peeking problem. But even with sequential testing, you should define your experiment duration upfront and resist the urge to call it early.

### Guardrail Metrics

Your primary metric tells you if the experiment is winning. Guardrail metrics tell you if it's *safe* to ship. An experiment can improve conversion by 10% but if it also increases payment errors by 50%, you can't ship it.

```typescript
// Common guardrail metrics for different experiment types:

const GUARDRAIL_METRICS = {
  // For any UI change
  general: [
    'crash_free_rate',       // Must not drop below 99.5%
    'error_rate',            // Must not increase by more than 10%
    'page_load_time_p95',    // Must not increase by more than 200ms
    'api_error_rate',        // Must not increase
  ],

  // For checkout/payment experiments
  checkout: [
    'payment_error_rate',    // Must not increase at all
    'average_order_value',   // Must not decrease by more than 3%
    'refund_rate',           // Must not increase
    'support_ticket_rate',   // Must not increase by more than 10%
  ],

  // For engagement experiments
  engagement: [
    'session_length',        // Must not decrease by more than 10%
    'daily_active_users',    // Must not decrease
    'retention_d7',          // Must not decrease (needs longer experiment)
    'uninstall_rate',        // Must not increase
  ],

  // For performance experiments
  performance: [
    'time_to_interactive',   // Primary metric (should improve)
    'memory_usage_p95',      // Must not increase by more than 10%
    'battery_drain_rate',    // Must not increase
    'data_usage_mb',         // Must not increase by more than 20%
  ],
};
```

---

## 7. KILL SWITCHES

### Every Critical Feature Needs One

A kill switch is a feature flag with one specific purpose: disable a feature instantly in production without deploying new code. It's your emergency brake. It's the thing that lets you sleep at night after shipping a big change.

Here's my rule: **any feature that touches payments, authentication, data mutation, or third-party integrations must have a kill switch.** No exceptions.

```typescript
// src/flags/kill-switches.ts

/**
 * Kill switch definitions.
 * 
 * These are separate from feature flags because they have different semantics:
 * - Feature flags default to OFF (new feature is hidden until enabled)
 * - Kill switches default to ON (feature is available unless explicitly killed)
 * 
 * A kill switch being "active" means the feature is DISABLED.
 */

export const KILL_SWITCHES = {
  // Payment and financial features
  payments: {
    key: 'kill_payments',
    description: 'Disables all payment processing. Use when Stripe is having an outage.',
    fallbackBehavior: 'Show maintenance message, disable checkout button.',
    owner: 'payments-team',
    oncallChannel: '#payments-oncall',
  },
  
  subscriptions: {
    key: 'kill_subscriptions',
    description: 'Disables subscription changes (upgrade/downgrade/cancel).',
    fallbackBehavior: 'Hide plan change buttons, show "temporarily unavailable" message.',
    owner: 'payments-team',
    oncallChannel: '#payments-oncall',
  },

  // Communication features
  liveChat: {
    key: 'kill_live_chat',
    description: 'Disables live chat widget. Use when Intercom/Zendesk is down.',
    fallbackBehavior: 'Hide chat bubble, show email support link instead.',
    owner: 'platform-team',
    oncallChannel: '#platform-oncall',
  },

  pushNotifications: {
    key: 'kill_push_notifications',
    description: 'Disables all push notification sending.',
    fallbackBehavior: 'Silently skip notification delivery.',
    owner: 'platform-team',
    oncallChannel: '#platform-oncall',
  },

  // Core features
  search: {
    key: 'kill_search',
    description: 'Disables search functionality. Use when Algolia/search provider is down.',
    fallbackBehavior: 'Show cached/popular results, disable search input.',
    owner: 'search-team',
    oncallChannel: '#search-oncall',
  },

  imageUpload: {
    key: 'kill_image_upload',
    description: 'Disables image upload. Use when S3/CDN is having issues.',
    fallbackBehavior: 'Disable upload button, show "temporarily unavailable".',
    owner: 'platform-team',
    oncallChannel: '#platform-oncall',
  },

  // Third-party integrations
  oauthLogin: {
    key: 'kill_oauth_login',
    description: 'Disables social/OAuth login (Google, Apple, etc).',
    fallbackBehavior: 'Hide OAuth buttons, show email/password only.',
    owner: 'auth-team',
    oncallChannel: '#auth-oncall',
  },

  analytics: {
    key: 'kill_analytics',
    description: 'Disables analytics event tracking. Use if analytics provider is causing performance issues.',
    fallbackBehavior: 'Silently drop analytics events.',
    owner: 'platform-team',
    oncallChannel: '#platform-oncall',
  },
} as const;
```

### Implementation Pattern

```typescript
// src/hooks/use-kill-switch.ts

import { useFeatureFlag } from '../flags/hooks';
import { KILL_SWITCHES } from '../flags/kill-switches';

type KillSwitchKey = keyof typeof KILL_SWITCHES;

/**
 * Hook to check if a kill switch is active.
 * 
 * Returns:
 * - isAvailable: true if the feature is available (kill switch is NOT active)
 * - isKilled: true if the feature is disabled (kill switch IS active)
 * - fallbackMessage: what to show the user when the feature is killed
 */
export function useKillSwitch(key: KillSwitchKey) {
  const killSwitch = KILL_SWITCHES[key];
  
  // The flag being TRUE means the feature is KILLED
  const isKilled = useFeatureFlag(killSwitch.key as any);

  return {
    isAvailable: !isKilled,
    isKilled,
    fallbackMessage: killSwitch.description,
    fallbackBehavior: killSwitch.fallbackBehavior,
  };
}
```

```typescript
// src/features/checkout/checkout-button.tsx

import { useKillSwitch } from '../../hooks/use-kill-switch';

export function CheckoutButton({ onPress }: { onPress: () => void }) {
  const { isAvailable, isKilled } = useKillSwitch('payments');

  if (isKilled) {
    return (
      <View style={styles.maintenanceContainer}>
        <Icon name="wrench" size={24} color={colors.warning} />
        <Text style={styles.maintenanceText}>
          Payments are temporarily unavailable. Please try again in a few minutes.
        </Text>
      </View>
    );
  }

  return (
    <Pressable
      style={styles.checkoutButton}
      onPress={onPress}
      accessibilityRole="button"
    >
      <Text style={styles.checkoutButtonText}>Complete Purchase</Text>
    </Pressable>
  );
}
```

```typescript
// src/features/chat/chat-widget.tsx

import { useKillSwitch } from '../../hooks/use-kill-switch';

export function ChatWidget() {
  const { isAvailable } = useKillSwitch('liveChat');

  if (!isAvailable) {
    // Don't render the chat widget at all
    // User can still reach support via email in the settings screen
    return null;
  }

  return <IntercomChatWidget />;
}
```

### Kill Switch Response Plan

Having kill switches is useless if nobody knows when to flip them. Document your response plan:

```typescript
// docs/kill-switch-runbook.ts (or in your team wiki)

/**
 * KILL SWITCH ACTIVATION PROTOCOL
 * 
 * WHEN TO ACTIVATE:
 * 1. Third-party service outage (Stripe, Intercom, etc.)
 *    → Kill the specific integration
 * 2. Bug causing data corruption
 *    → Kill the affected feature immediately
 * 3. Performance degradation traced to a specific feature
 *    → Kill the feature, investigate
 * 4. Security vulnerability discovered
 *    → Kill the affected feature, page security team
 * 
 * WHO CAN ACTIVATE:
 * - Any on-call engineer
 * - Any team lead
 * - Any engineering manager
 * - (In an emergency, ANYONE — ask forgiveness, not permission)
 * 
 * ACTIVATION STEPS:
 * 1. Go to Statsig console (or your flag provider)
 * 2. Find the kill switch flag
 * 3. Enable it (this DISABLES the feature)
 * 4. Post in #incidents: "Activated kill switch [X] because [Y]"
 * 5. Page the feature owner if they're not already aware
 * 
 * DEACTIVATION STEPS:
 * 1. Confirm the underlying issue is resolved
 * 2. Disable the kill switch flag
 * 3. Monitor for 30 minutes
 * 4. Post in #incidents: "Deactivated kill switch [X], issue resolved"
 * 
 * TESTING:
 * Kill switches should be tested monthly. If you've never flipped
 * a kill switch, you don't know if it works. Schedule a "chaos day"
 * where you activate each kill switch in a staging environment
 * and verify the fallback behavior is correct.
 */
```

---

## 8. EXPERIMENTATION CULTURE

### The HiPPO Problem

HiPPO stands for "Highest Paid Person's Opinion." It's when product decisions are made by the most senior person in the room, based on intuition, rather than by data. And it's *rampant* in engineering organizations.

"I think users want a sidebar navigation." Did you test it? "No, but I've been building products for 20 years." That's great, but your 20 years of experience didn't include *this* product with *these* users in *this* market. The data might surprise you.

I've seen experiments where the CEO's preferred design lost by 15%. I've seen experiments where the intern's "crazy idea" won by 20%. I've seen experiments where the thing everyone was *sure* would win... had zero measurable impact. That's the humbling reality of experimentation. Your intuition is a hypothesis, not a conclusion.

### Building the Culture

```
┌──────────────────────────────────────────────────────────────────────┐
│            EXPERIMENTATION MATURITY MODEL                            │
│                                                                      │
│  Level 0: "We don't test, we just ship"                             │
│  ├── Decisions: Based on opinion                                     │
│  ├── Flag usage: None                                                │
│  └── Outcome: Arguments about features, no one knows what works     │
│                                                                      │
│  Level 1: "We have feature flags"                                    │
│  ├── Decisions: Ship it, flag it, roll it out                       │
│  ├── Flag usage: On/off toggles for rollouts                        │
│  └── Outcome: Safer deployments, but still guessing on effectiveness│
│                                                                      │
│  Level 2: "We run experiments sometimes"                             │
│  ├── Decisions: Big changes get A/B tested                          │
│  ├── Flag usage: Experiments for major features                     │
│  └── Outcome: Data for big decisions, gut for small ones            │
│                                                                      │
│  Level 3: "We experiment by default"                                 │
│  ├── Decisions: Every customer-facing change is an experiment       │
│  ├── Flag usage: Systematic, with hypothesis and metrics            │
│  └── Outcome: Data-driven product, faster iteration                 │
│                                                                      │
│  Level 4: "We're an experimentation-first org"                       │
│  ├── Decisions: Culture of testing, shared experiment library       │
│  ├── Flag usage: Platform team supports self-serve experiments      │
│  └── Outcome: Compounding wins, evidence-based roadmap              │
│                                                                      │
│  Most teams are at Level 0-1.                                        │
│  This chapter aims to get you to Level 2-3.                          │
│  Level 4 requires organizational commitment.                         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Running Experiments Across Your Team

Here's a practical system for running experiments at scale:

```typescript
// Experiment tracking template (use in your project management tool)

interface ExperimentProposal {
  // Identity
  name: string;
  owner: string;
  team: string;
  
  // Hypothesis
  hypothesis: string;
  rationale: string;
  
  // Design
  primaryMetric: string;
  secondaryMetrics: string[];
  guardrailMetrics: string[];
  variants: {
    name: string;
    description: string;
  }[];
  
  // Sizing
  minimumDetectableEffect: number;
  requiredSampleSize: number;
  estimatedDurationDays: number;
  trafficAllocation: number; // Percentage of eligible users
  
  // Timeline
  proposedStartDate: string;
  proposedEndDate: string;
  
  // Review
  status: 'proposed' | 'approved' | 'running' | 'analyzing' | 'concluded';
  result?: 'winner' | 'loser' | 'inconclusive';
  learnings?: string;
}

// Example filled in:
const checkoutExperiment: ExperimentProposal = {
  name: 'Checkout Flow Simplification Q2 2026',
  owner: 'Sarah Chen',
  team: 'growth',
  
  hypothesis: 'Reducing checkout from 3 steps to 1 step will increase ' +
    'checkout completion rate by at least 5%, because fewer steps mean ' +
    'fewer abandonment points.',
  rationale: 'Our funnel data shows 12% drop-off between step 1 and step 2, ' +
    'and 8% between step 2 and step 3. If we can eliminate those transitions, ' +
    'we recover most of that drop-off.',
  
  primaryMetric: 'checkout_completion_rate',
  secondaryMetrics: [
    'time_to_checkout_completion',
    'checkout_error_rate',
  ],
  guardrailMetrics: [
    'payment_error_rate',
    'average_order_value',
    'refund_rate_7d',
  ],
  variants: [
    { name: 'control', description: '3-step checkout (current)' },
    { name: 'single_page', description: '1-step checkout (all fields on one page)' },
  ],
  
  minimumDetectableEffect: 0.05, // 5% relative
  requiredSampleSize: 31000,     // per variant
  estimatedDurationDays: 14,
  trafficAllocation: 50,         // 50% of checkout users enter experiment
  
  proposedStartDate: '2026-04-07',
  proposedEndDate: '2026-04-21',
  
  status: 'proposed',
};
```

### Documenting Results

Every experiment should produce a brief document. This is your institutional memory — it prevents future teams from re-running the same experiments and making the same mistakes.

```markdown
## Experiment Report: Checkout Flow Simplification

**Date:** April 7-21, 2026
**Owner:** Sarah Chen (Growth Team)
**Status:** WINNER — shipping single-page variant

### Hypothesis
Reducing checkout from 3 steps to 1 step will increase checkout 
completion rate by >= 5%.

### Results
| Metric                    | Control | Single Page | Delta   | Significant? |
|---------------------------|---------|-------------|---------|-------------|
| Checkout completion rate  | 68.2%   | 73.1%       | +7.2%   | Yes (p<0.01)|
| Time to complete (median) | 142s    | 89s         | -37.3%  | Yes (p<0.01)|
| Payment error rate        | 1.2%    | 1.3%        | +0.1pp  | No          |
| Average order value       | $47.20  | $46.80      | -0.8%   | No          |
| Refund rate (7-day)       | 2.1%    | 2.0%        | -0.1pp  | No          |

### Decision
Ship the single-page checkout. Primary metric improved by 7.2% 
(exceeding our 5% target), no guardrail metrics regressed.

### Learnings
1. The biggest win was eliminating the step 1 → 2 transition (12% drop-off → 0%)
2. Users spent less time overall but the same time per field — they weren't rushing
3. Mobile users saw an even bigger improvement (+9.4%) than desktop (+5.1%)
4. Consider further experiments on mobile-specific checkout optimizations

### Next Steps
- [ ] Remove experiment code and old checkout flow
- [ ] Clean up feature flag by May 5, 2026
- [ ] Propose follow-up experiment for mobile checkout optimization
```

### Avoiding Common Experimentation Pitfalls

**Pitfall 1: Too many experiments running simultaneously.**
If a user is in 10 experiments at once, interactions between experiments can pollute your results. Limit concurrent experiments per user to 3-5.

**Pitfall 2: Underpowered experiments.**
Running an experiment with too few users is worse than not running one at all, because you'll get noise that *looks* like signal. Always calculate sample size first.

**Pitfall 3: Testing things that don't matter.**
"Should the icon be 24px or 28px?" Unless your hypothesis includes a specific metric that this change will impact, don't waste an experiment slot on it. Reserve experiments for changes that could meaningfully impact business metrics.

**Pitfall 4: Not acting on results.**
If the experiment says the new thing lost, *kill it*. Don't rationalize. Don't say "well, the data was probably wrong." If you don't trust the data enough to act on it when it disagrees with you, you didn't trust it when it agreed with you either.

**Pitfall 5: Survivorship bias in experiment selection.**
Teams tend to only experiment on things they're excited about. But the most valuable experiments are often the ones that test whether a beloved feature is actually working. "What happens if we *remove* the recommendation carousel?" Sometimes the answer is "nothing," and you just simplified your codebase.

---

## 9. FLAG LIFECYCLE

### The Flag Lifecycle

Every feature flag should follow a predictable lifecycle:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    THE FLAG LIFECYCLE                                 │
│                                                                      │
│  CREATE ──→ TEST ──→ ENABLE ──→ MONITOR ──→ FULL ROLLOUT ──→ CLEAN │
│                                                                      │
│  Create:    Define in code + flag provider                          │
│  Test:      Verify both paths work (flag on AND off)                │
│  Enable:    Start gradual rollout                                    │
│  Monitor:   Watch metrics at each rollout stage                      │
│  Rollout:   Reach 100% of users                                     │
│  Clean up:  Remove the flag, delete the old code path               │
│                                                                      │
│  TYPICAL TIMELINE:                                                   │
│  Week 0:  Create flag, deploy behind it                              │
│  Week 1:  Internal testing (dogfooding)                              │
│  Week 2:  1% → 5% → 25% rollout                                    │
│  Week 3:  50% → 100% rollout                                        │
│  Week 4:  Confirm stable at 100%                                     │
│  Week 5:  Clean up flag (remove code, remove from provider)          │
│                                                                      │
│  REALITY: Most flags never reach "clean up." This is a problem.     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Technical Debt from Abandoned Flags

Here's a scenario I see constantly: a team creates a feature flag, rolls out the feature, confirms it works, and then... never removes the flag. Six months later, the codebase is littered with `if (useFeatureFlag('new_checkout'))` checks where the flag has been at 100% for months. Nobody remembers which flags are active, which are stale, and which are load-bearing (removing them would break something).

This is real technical debt, and it compounds:

1. **Code complexity:** Every flag is a branching point. Two flags mean 4 possible code paths. Ten flags mean 1,024 possible paths. Good luck testing all of those.
2. **Confusion:** New team members don't know if a flag check is intentional (still being experimented on) or abandoned (should be cleaned up).
3. **Performance:** Each flag evaluation is a function call that may involve network I/O or cache lookups. Unnecessary flags add unnecessary overhead.
4. **Provider costs:** Most flag providers charge by flag count or evaluation count. Dead flags still cost money.

### Automated Stale Flag Detection

```typescript
// scripts/check-stale-flags.ts
// Run this in CI to detect flags that need cleanup

import { FLAGS } from '../src/flags/definitions';

const TODAY = new Date();
const WARNING_DAYS = 14; // Warn if cleanup date is within 14 days
const ERROR_DAYS = 0;    // Error if cleanup date has passed

interface FlagStatus {
  key: string;
  cleanupBy: string;
  daysUntilCleanup: number;
  status: 'ok' | 'warning' | 'overdue' | 'permanent';
}

function checkStaleness(): FlagStatus[] {
  return Object.entries(FLAGS).map(([key, flag]) => {
    if (flag.cleanupBy === 'never') {
      return { key, cleanupBy: 'never', daysUntilCleanup: Infinity, status: 'permanent' as const };
    }

    const cleanupDate = new Date(flag.cleanupBy);
    const daysUntil = Math.ceil(
      (cleanupDate.getTime() - TODAY.getTime()) / (1000 * 60 * 60 * 24)
    );

    let status: 'ok' | 'warning' | 'overdue';
    if (daysUntil < ERROR_DAYS) {
      status = 'overdue';
    } else if (daysUntil < WARNING_DAYS) {
      status = 'warning';
    } else {
      status = 'ok';
    }

    return { key, cleanupBy: flag.cleanupBy, daysUntilCleanup: daysUntil, status };
  });
}

// Run the check
const results = checkStaleness();

const overdue = results.filter((r) => r.status === 'overdue');
const warnings = results.filter((r) => r.status === 'warning');

if (warnings.length > 0) {
  console.warn('\n⚠️  Feature flags approaching cleanup date:');
  warnings.forEach((w) => {
    console.warn(`  - ${w.key}: cleanup by ${w.cleanupBy} (${w.daysUntilCleanup} days)`);
  });
}

if (overdue.length > 0) {
  console.error('\n❌ Feature flags OVERDUE for cleanup:');
  overdue.forEach((o) => {
    console.error(`  - ${o.key}: was due ${o.cleanupBy} (${Math.abs(o.daysUntilCleanup)} days ago)`);
  });

  // Fail CI if flags are overdue
  console.error(
    `\n${overdue.length} flag(s) overdue for cleanup. ` +
    'Either clean up the flags or update the cleanupBy date with justification.'
  );
  process.exit(1);
}

console.log('\n✅ All feature flags are within their lifecycle.');
```

### ESLint Rule for Flag Hygiene

```typescript
// eslint-rules/no-direct-flag-access.ts
// Enforce that all flag access goes through the typed hooks

import { AST_NODE_TYPES, ESLintUtils } from '@typescript-eslint/utils';

const createRule = ESLintUtils.RuleCreator(
  (name) => `https://your-docs.com/rules/${name}`
);

export const noDirectFlagAccess = createRule({
  name: 'no-direct-flag-access',
  meta: {
    type: 'problem',
    docs: {
      description: 'Enforce using typed flag hooks instead of raw SDK calls',
    },
    messages: {
      useTypedHook:
        'Use useFeatureFlag() or useFeatureExperiment() from src/flags/hooks ' +
        'instead of calling the SDK directly. This ensures type safety and ' +
        'centralized flag management.',
    },
    schema: [],
  },
  defaultOptions: [],
  create(context) {
    return {
      ImportDeclaration(node) {
        const source = node.source.value;
        // Flag if someone imports directly from the SDK in non-flag files
        if (
          typeof source === 'string' &&
          (source.includes('@statsig/') ||
            source.includes('launchdarkly') ||
            source.includes('@unleash/')) &&
          !context.getFilename().includes('/flags/')
        ) {
          context.report({
            node,
            messageId: 'useTypedHook',
          });
        }
      },
    };
  },
});
```

### The Cleanup Sprint

Every quarter, schedule a "flag cleanup sprint" — a dedicated period where the team cleans up stale flags. Here's a process:

1. **Run the stale flag detector** to get the list of overdue flags.
2. **For each flag, answer:** Is it at 100%? Has it been stable for 2+ weeks? If yes, it's time to clean up.
3. **Cleaning up a flag means:**
   - Removing the flag check from the code (keep the winning path, delete the losing path)
   - Removing the flag definition from your flag registry
   - Deleting the flag from your provider's console
   - Deleting any associated experiment configuration
4. **PR review:** Flag cleanup PRs should be reviewed carefully — you're deleting code paths, and you want to make sure you're deleting the *right* ones.
5. **Track your progress:** Celebrate the team that cleans up the most flags. Make it a positive thing, not a chore.

---

## 10. COMPLETE IMPLEMENTATION

### Full Feature Flag Setup for a Monorepo

Let's tie everything together. Here's a complete feature flag architecture for a monorepo with a React Native app and a Next.js web app, using Statsig as the shared provider and Vercel Flags for Next.js-specific edge evaluation.

### Shared Flag Definitions

```typescript
// packages/feature-flags/src/definitions.ts

/**
 * Shared feature flag definitions.
 * 
 * This package is used by both the React Native app and the Next.js app.
 * All flags must be defined here — no ad-hoc flag usage.
 */

export interface FlagDefinition {
  key: string;
  type: 'gate' | 'experiment';
  description: string;
  owner: string;
  created: string;
  cleanupBy: string;
  platforms: ('mobile' | 'web')[];
}

export interface ExperimentDefinition extends FlagDefinition {
  type: 'experiment';
  variants: readonly string[];
  primaryMetric: string;
  guardrailMetrics: string[];
}

export const FLAGS = {
  // ─── Gates (boolean flags) ───
  new_onboarding: {
    key: 'new_onboarding',
    type: 'gate' as const,
    description: 'Redesigned onboarding flow with progressive profiling',
    owner: 'growth-team',
    created: '2026-03-01',
    cleanupBy: '2026-06-01',
    platforms: ['mobile', 'web'] as const,
  },

  new_profile_page: {
    key: 'new_profile_page',
    type: 'gate' as const,
    description: 'Redesigned user profile page',
    owner: 'profile-team',
    created: '2026-03-15',
    cleanupBy: '2026-05-15',
    platforms: ['mobile', 'web'] as const,
  },

  native_image_picker: {
    key: 'native_image_picker',
    type: 'gate' as const,
    description: 'Use native image picker instead of web-based',
    owner: 'media-team',
    created: '2026-04-01',
    cleanupBy: '2026-06-01',
    platforms: ['mobile'] as const, // Mobile only
  },

  edge_caching: {
    key: 'edge_caching',
    type: 'gate' as const,
    description: 'Enable edge caching for API responses',
    owner: 'platform-team',
    created: '2026-04-01',
    cleanupBy: '2026-06-01',
    platforms: ['web'] as const, // Web only
  },

  // ─── Experiments ───
  pricing_layout: {
    key: 'pricing_layout',
    type: 'experiment' as const,
    description: 'Testing pricing page layouts',
    owner: 'growth-team',
    created: '2026-04-01',
    cleanupBy: '2026-06-01',
    platforms: ['web'] as const,
    variants: ['control', 'annual_first', 'social_proof'] as const,
    primaryMetric: 'plan_selection_rate',
    guardrailMetrics: ['bounce_rate', 'support_ticket_rate'],
  },

  notification_frequency: {
    key: 'notification_frequency',
    type: 'experiment' as const,
    description: 'Testing notification frequency impact on retention',
    owner: 'engagement-team',
    created: '2026-04-01',
    cleanupBy: '2026-06-01',
    platforms: ['mobile'] as const,
    variants: ['control', 'reduced', 'batched'] as const,
    primaryMetric: 'd7_retention',
    guardrailMetrics: ['notification_opt_out_rate', 'daily_active_rate'],
  },

  // ─── Kill Switches (permanent) ───
  kill_payments: {
    key: 'kill_payments',
    type: 'gate' as const,
    description: 'KILL SWITCH: Disable payment processing',
    owner: 'payments-team',
    created: '2026-01-01',
    cleanupBy: 'never',
    platforms: ['mobile', 'web'] as const,
  },

  kill_search: {
    key: 'kill_search',
    type: 'gate' as const,
    description: 'KILL SWITCH: Disable search',
    owner: 'search-team',
    created: '2026-01-01',
    cleanupBy: 'never',
    platforms: ['mobile', 'web'] as const,
  },
} as const;

// ─── Type Exports ───

export type FlagKey = keyof typeof FLAGS;

export type GateKey = {
  [K in FlagKey]: (typeof FLAGS)[K]['type'] extends 'gate' ? K : never;
}[FlagKey];

export type ExperimentKey = {
  [K in FlagKey]: (typeof FLAGS)[K]['type'] extends 'experiment' ? K : never;
}[FlagKey];

export type MobileFlagKey = {
  [K in FlagKey]: 'mobile' extends (typeof FLAGS)[K]['platforms'][number] ? K : never;
}[FlagKey];

export type WebFlagKey = {
  [K in FlagKey]: 'web' extends (typeof FLAGS)[K]['platforms'][number] ? K : never;
}[FlagKey];
```

### Shared Statsig User Builder

```typescript
// packages/feature-flags/src/user.ts

/**
 * Build a consistent Statsig user object across platforms.
 * Both mobile and web should send the same user shape.
 */

export interface AppUser {
  id: string;
  email?: string;
  plan?: 'free' | 'pro' | 'enterprise';
  createdAt?: string;
  country?: string;
  locale?: string;
  appVersion?: string;
  isInternal?: boolean;
}

export function buildStatsigUser(user: AppUser, platform: 'mobile' | 'web') {
  return {
    userID: user.id,
    email: user.email,
    country: user.country,
    custom: {
      plan: user.plan ?? 'free',
      signup_date: user.createdAt,
      locale: user.locale,
      app_version: user.appVersion,
      is_internal: user.isInternal ? 'true' : 'false',
      platform,
    },
  };
}
```

### React Native Integration

```typescript
// apps/mobile/src/providers/feature-flags.tsx

import { StatsigProviderExpo } from '@statsig/expo-bindings';
import { StatsigClient } from '@statsig/js-client';
import { useGate, useExperiment } from '@statsig/react-bindings';
import { MMKV } from 'react-native-mmkv';
import {
  FLAGS,
  type GateKey,
  type ExperimentKey,
  type MobileFlagKey,
  buildStatsigUser,
} from '@company/feature-flags';
import { useAuth } from '../hooks/use-auth';
import { getAppVersion } from '../lib/device-info';

const flagStorage = new MMKV({ id: 'statsig-cache' });

const statsigClient = new StatsigClient(
  process.env.EXPO_PUBLIC_STATSIG_CLIENT_KEY!,
  { userID: '' },
  {
    overrideAdapter: {
      getItem: (key) => flagStorage.getString(key) ?? null,
      setItem: (key, value) => flagStorage.set(key, value),
      removeItem: (key) => flagStorage.delete(key),
    },
  }
);

// Initialize the client
statsigClient.initializeAsync();

export function FeatureFlagProvider({ children }: { children: React.ReactNode }) {
  const { user } = useAuth();

  const statsigUser = user
    ? buildStatsigUser(
        {
          id: user.id,
          email: user.email,
          plan: user.subscription?.plan,
          createdAt: user.createdAt,
          country: user.country,
          locale: user.locale,
          appVersion: getAppVersion(),
          isInternal: user.isInternal,
        },
        'mobile'
      )
    : { userID: '' };

  return (
    <StatsigProviderExpo client={statsigClient} user={statsigUser}>
      {children}
    </StatsigProviderExpo>
  );
}

// ─── Type-safe hooks for mobile ───

/**
 * Only allows gates that are defined for the 'mobile' platform.
 */
export function useFeatureFlag(flagKey: MobileFlagKey & GateKey): boolean {
  const flag = FLAGS[flagKey];

  if (__DEV__ && !flag.platforms.includes('mobile')) {
    console.error(
      `[FeatureFlags] Flag "${flagKey}" is not defined for mobile platform. ` +
      `Platforms: ${flag.platforms.join(', ')}`
    );
  }

  const { value } = useGate(flag.key);
  return value;
}

/**
 * Only allows experiments that are defined for the 'mobile' platform.
 */
export function useFeatureExperiment(experimentKey: MobileFlagKey & ExperimentKey) {
  const flag = FLAGS[experimentKey];

  if (__DEV__ && !flag.platforms.includes('mobile')) {
    console.error(
      `[FeatureFlags] Experiment "${experimentKey}" is not defined for mobile. ` +
      `Platforms: ${flag.platforms.join(', ')}`
    );
  }

  const { config } = useExperiment(flag.key);

  return {
    variant: config.get('variant', 'control') as string,
    getParam: <T>(key: string, defaultValue: T): T => {
      return config.get(key, defaultValue) as T;
    },
  };
}
```

### Next.js Integration

```typescript
// apps/web/flags.ts

import { flag, dedupe } from '@vercel/flags/next';
import Statsig from 'statsig-node';
import {
  FLAGS,
  type WebFlagKey,
  type GateKey,
  type ExperimentKey,
  buildStatsigUser,
} from '@company/feature-flags';
import { getCurrentUser } from './lib/auth';

// Deduplicate auth calls across flag evaluations
const getUser = dedupe(async () => getCurrentUser());

// Initialize Statsig server SDK
let statsigReady = false;
async function ensureStatsig() {
  if (!statsigReady) {
    await Statsig.initialize(process.env.STATSIG_SERVER_KEY!);
    statsigReady = true;
  }
}

/**
 * Dynamically create Vercel flags backed by Statsig evaluation.
 * This gives us Vercel's edge speed with Statsig's targeting and analytics.
 */
function createGateFlag(flagKey: WebFlagKey & GateKey) {
  const flagDef = FLAGS[flagKey];

  return flag({
    key: flagDef.key,
    description: flagDef.description,
    async decide() {
      await ensureStatsig();
      const user = await getUser();
      if (!user) return false;

      const statsigUser = buildStatsigUser(
        {
          id: user.id,
          email: user.email,
          plan: user.subscription?.plan,
          createdAt: user.createdAt,
          country: user.country,
          locale: user.locale,
          isInternal: user.isInternal,
        },
        'web'
      );

      return Statsig.checkGateSync(statsigUser, flagDef.key);
    },
    defaultValue: false,
  });
}

function createExperimentFlag(
  flagKey: WebFlagKey & ExperimentKey
) {
  const flagDef = FLAGS[flagKey] as (typeof FLAGS)[WebFlagKey & ExperimentKey];

  return flag({
    key: flagDef.key,
    description: flagDef.description,
    options: [...flagDef.variants] as string[],
    async decide() {
      await ensureStatsig();
      const user = await getUser();
      if (!user) return 'control';

      const statsigUser = buildStatsigUser(
        {
          id: user.id,
          email: user.email,
          plan: user.subscription?.plan,
          createdAt: user.createdAt,
          country: user.country,
          locale: user.locale,
          isInternal: user.isInternal,
        },
        'web'
      );

      const experiment = Statsig.getExperimentSync(statsigUser, flagDef.key);
      return experiment.get('variant', 'control') as string;
    },
    defaultValue: 'control',
  });
}

// ─── Export all web flags ───

export const newOnboarding = createGateFlag('new_onboarding');
export const newProfilePage = createGateFlag('new_profile_page');
export const edgeCaching = createGateFlag('edge_caching');
export const killPayments = createGateFlag('kill_payments');
export const killSearch = createGateFlag('kill_search');
export const pricingLayout = createExperimentFlag('pricing_layout');
```

### Using Flags in Next.js Pages

```typescript
// apps/web/app/onboarding/page.tsx

import { newOnboarding } from '../../flags';

export default async function OnboardingPage() {
  const showNew = await newOnboarding();

  return showNew ? <NewOnboarding /> : <LegacyOnboarding />;
}
```

```typescript
// apps/web/app/pricing/page.tsx

import { pricingLayout } from '../../flags';

export default async function PricingPage() {
  const variant = await pricingLayout();

  switch (variant) {
    case 'annual_first':
      return <AnnualFirstPricing />;
    case 'social_proof':
      return <SocialProofPricing />;
    default:
      return <StandardPricing />;
  }
}
```

### Next.js Middleware for Edge Evaluation

```typescript
// apps/web/middleware.ts

import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { newOnboarding, killPayments } from './flags';

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Kill switch: redirect away from checkout if payments are killed
  if (pathname.startsWith('/checkout')) {
    const paymentsKilled = await killPayments();
    if (paymentsKilled) {
      return NextResponse.redirect(new URL('/maintenance', request.url));
    }
  }

  // Feature flag: rewrite to new onboarding
  if (pathname === '/onboarding') {
    const showNew = await newOnboarding();
    if (showNew) {
      return NextResponse.rewrite(new URL('/new-onboarding', request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/checkout/:path*', '/onboarding'],
};
```

### Package Structure

```
packages/
  feature-flags/
    package.json
    src/
      definitions.ts      # Shared flag definitions (both platforms)
      user.ts             # Shared Statsig user builder
      index.ts            # Re-exports everything
    tsconfig.json

apps/
  mobile/
    src/
      providers/
        feature-flags.tsx  # RN Statsig provider + typed hooks
      flags/
        kill-switches.ts   # Kill switch definitions
        cache.ts           # MMKV caching layer
        sync.ts            # Background flag refresh

  web/
    flags.ts               # Vercel Flags backed by Statsig
    middleware.ts           # Edge flag evaluation
    lib/
      edge-config.ts       # Edge Config reader

scripts/
  check-stale-flags.ts     # CI script for stale flag detection
```

```json
// packages/feature-flags/package.json
{
  "name": "@company/feature-flags",
  "version": "1.0.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "dependencies": {},
  "devDependencies": {
    "typescript": "^5.5.0"
  }
}
```

### Testing Flag Behavior

```typescript
// packages/feature-flags/src/__tests__/definitions.test.ts

import { FLAGS } from '../definitions';

describe('Feature Flag Definitions', () => {
  test('every flag has required fields', () => {
    Object.entries(FLAGS).forEach(([key, flag]) => {
      expect(flag.key).toBeTruthy();
      expect(flag.type).toMatch(/^(gate|experiment)$/);
      expect(flag.description).toBeTruthy();
      expect(flag.owner).toBeTruthy();
      expect(flag.created).toMatch(/^\d{4}-\d{2}-\d{2}$/);
      expect(flag.platforms.length).toBeGreaterThan(0);
      
      // Cleanup date must be a valid date or 'never'
      if (flag.cleanupBy !== 'never') {
        expect(flag.cleanupBy).toMatch(/^\d{4}-\d{2}-\d{2}$/);
        expect(new Date(flag.cleanupBy).getTime()).not.toBeNaN();
      }
    });
  });

  test('experiment flags have variants defined', () => {
    Object.entries(FLAGS)
      .filter(([, flag]) => flag.type === 'experiment')
      .forEach(([key, flag]) => {
        const experimentFlag = flag as any;
        expect(experimentFlag.variants).toBeDefined();
        expect(experimentFlag.variants.length).toBeGreaterThanOrEqual(2);
        expect(experimentFlag.primaryMetric).toBeTruthy();
        expect(experimentFlag.guardrailMetrics).toBeDefined();
      });
  });

  test('flag keys match their object keys', () => {
    Object.entries(FLAGS).forEach(([objectKey, flag]) => {
      expect(flag.key).toBe(objectKey);
    });
  });

  test('kill switches are marked as permanent', () => {
    Object.entries(FLAGS)
      .filter(([key]) => key.startsWith('kill_'))
      .forEach(([key, flag]) => {
        expect(flag.cleanupBy).toBe('never');
      });
  });

  test('no duplicate flag keys', () => {
    const keys = Object.values(FLAGS).map((f) => f.key);
    const uniqueKeys = new Set(keys);
    expect(keys.length).toBe(uniqueKeys.size);
  });
});
```

```typescript
// apps/mobile/src/__tests__/feature-flags.test.tsx

import React from 'react';
import { render, screen } from '@testing-library/react-native';

// Mock the Statsig SDK
jest.mock('@statsig/react-bindings', () => ({
  useGate: jest.fn(),
  useExperiment: jest.fn(),
}));

import { useGate, useExperiment } from '@statsig/react-bindings';
import { CheckoutScreen } from '../features/checkout/checkout-screen';

describe('CheckoutScreen with feature flags', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('shows legacy checkout when experiment is control', () => {
    (useGate as jest.Mock).mockReturnValue({ value: false });
    (useExperiment as jest.Mock).mockReturnValue({
      config: {
        get: (key: string, defaultValue: any) => defaultValue,
      },
    });

    render(<CheckoutScreen />);
    expect(screen.getByTestId('legacy-checkout')).toBeTruthy();
  });

  test('shows single-page checkout for single_page variant', () => {
    (useGate as jest.Mock).mockReturnValue({ value: false });
    (useExperiment as jest.Mock).mockReturnValue({
      config: {
        get: (key: string, defaultValue: any) => {
          if (key === 'variant') return 'single_page';
          return defaultValue;
        },
      },
    });

    render(<CheckoutScreen />);
    expect(screen.getByTestId('single-page-checkout')).toBeTruthy();
  });

  test('shows maintenance message when payments are killed', () => {
    // Kill switch active
    (useGate as jest.Mock).mockReturnValue({ value: true });
    (useExperiment as jest.Mock).mockReturnValue({
      config: {
        get: (key: string, defaultValue: any) => defaultValue,
      },
    });

    render(<CheckoutScreen />);
    expect(screen.getByText(/temporarily unavailable/i)).toBeTruthy();
  });
});
```

### The Complete Mental Model

Let me leave you with the mental model that ties this all together:

```
┌──────────────────────────────────────────────────────────────────────┐
│               THE FEATURE FLAG MENTAL MODEL                          │
│                                                                      │
│  CODE is always deployed to production.                              │
│  FEATURES are controlled by flags.                                   │
│  FLAGS are evaluated by the flag provider.                           │
│  EXPERIMENTS are flags with metrics.                                 │
│  KILL SWITCHES are flags with emergency semantics.                   │
│                                                                      │
│  The result:                                                         │
│  - You deploy code multiple times a day (low risk)                  │
│  - You release features gradually (measured risk)                    │
│  - You experiment with everything (data-driven)                      │
│  - You can kill anything instantly (recoverable)                     │
│  - You clean up flags regularly (maintainable)                       │
│                                                                      │
│  This is how the best product teams in the world ship software.      │
│  It's not magic. It's discipline and tooling.                        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

Feature flags are not a nice-to-have. They're infrastructure. They belong in your first PR, not your tenth. Set them up, use them for every customer-facing change, run experiments, maintain kill switches, and clean up your flags. Do this, and you'll ship faster, break less, and build better products.

---

**Next:** [Chapter 38: Search Implementation — From Input to Results](../part-4-architecture-at-scale/38-search.md)

**Previous:** [Chapter 36: Error Recovery & Offline Resilience](./36-error-recovery.md)
