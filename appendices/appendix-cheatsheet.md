<!--
  TYPE: appendix
  TITLE: Cheat Sheet
  APPENDIX: B
  UPDATED: 2026-04-07
-->

# Appendix B: Cheat Sheet

> One-page quick reference for the 100x frontend architect. Print it, pin it, live it.

---

## The 2026 Default Stack

| Layer | Default Choice | Alternative | When to Switch |
|-------|---------------|-------------|----------------|
| **Framework** | React Native + Expo (mobile) | Flutter | Never, if you're reading this guide |
| **Web Framework** | Next.js (App Router) | Remix, Astro | Content-heavy static sites (Astro) |
| **Language** | TypeScript (strict mode) | — | Never disable strict |
| **JS Engine** | Hermes | JSC, V8 | JSC only for specific debugging |
| **Routing (mobile)** | Expo Router v4 | React Navigation bare | When you need drawer+tab combos Expo Router doesn't support |
| **State (client)** | Zustand | Jotai | Many independent atoms of derived state |
| **State (server)** | TanStack Query v5 | SWR | Never — TQ is strictly better |
| **State (forms)** | React Hook Form + Zod | Formik | Never — RHF won |
| **Styling (mobile)** | NativeWind v4 (Tailwind) | Tamagui, StyleSheet | Need native perf primitives (Tamagui) |
| **Styling (web)** | Tailwind CSS v4 | CSS Modules | Team preference only |
| **Animation** | Reanimated 3 + Gesture Handler | Moti | Simpler declarative API (Moti wraps Reanimated) |
| **Navigation** | Expo Router (file-based) | React Navigation | Complex modal flows |
| **Data Fetching** | TanStack Query | tRPC + TQ | Full-stack TypeScript monorepo |
| **API Layer** | REST + Zod validation | GraphQL (Relay) | Complex relational data needs |
| **Bundler (mobile)** | Metro | — | No choice here |
| **Bundler (web)** | Turbopack (Next.js) | Vite | Non-Next.js projects |
| **Monorepo** | Turborepo + pnpm workspaces | Nx | Enterprise with many teams (Nx) |
| **Design System** | Custom + Radix/Headless UI | shadcn/ui | Rapid prototyping (shadcn/ui) |
| **Testing (unit)** | Vitest + RNTL | Jest | Legacy projects only |
| **Testing (E2E mobile)** | Maestro | Detox | Already invested in Detox |
| **Testing (E2E web)** | Playwright | Cypress | Already invested in Cypress |
| **Testing (API mock)** | MSW v2 | Nock | Never — MSW is network-level |
| **CI/CD** | GitHub Actions + EAS Build | Bitrise | iOS-specific needs |
| **Deployment (mobile)** | EAS Build + EAS Submit | Fastlane | Complex signing requirements |
| **Deployment (web)** | Vercel | Cloudflare Pages | Edge-first architecture (CF) |
| **OTA Updates** | EAS Update | CodePush | Never — CodePush is deprecated |
| **Monitoring (crashes)** | Sentry | Crashlytics | Budget-constrained (Crashlytics is free) |
| **Monitoring (web)** | Vercel Analytics + Sentry | Datadog RUM | Large-scale observability needs |
| **Analytics** | PostHog | Amplitude, Mixpanel | Enterprise needs (Amplitude) |
| **Auth** | Clerk | Auth0, Supabase Auth | Self-hosted needs (Supabase) |
| **Payments** | Stripe + RevenueCat | — | RevenueCat for subscriptions, Stripe for one-time |
| **Backend** | Supabase | Firebase | Google ecosystem lock-in preference |
| **Versioning** | Changesets | semantic-release | Simpler single-package repos |
| **Linting** | ESLint v9 (flat config) + Prettier | Biome | All-in-one preference (Biome) |

---

## Decision Matrices

### State Management

| Need | Solution | Why |
|------|----------|-----|
| Server data (API responses) | TanStack Query | Caching, dedup, background refetch, optimistic updates |
| Global client state (theme, auth) | Zustand | Simple API, works outside React, tiny bundle |
| Many independent atoms | Jotai | Bottom-up, derived state, no single store |
| Complex async workflows | XState | Formal state machines prevent impossible states |
| Form state | React Hook Form + Zod | Uncontrolled inputs, schema validation, minimal rerenders |
| URL state | Expo Router / Next.js params | Single source of truth for navigation state |
| Per-component state | useState / useReducer | Keep it simple, co-locate with component |

### Styling Approach

| Need | Solution | Why |
|------|----------|-----|
| Cross-platform (RN + Web) | NativeWind v4 | Tailwind classes compile to StyleSheet |
| Maximum native performance | Tamagui | Compile-time optimization, themes |
| Quick prototype | StyleSheet.create | Zero dependencies, built-in |
| Web only | Tailwind CSS v4 | Utility-first, tree-shakeable |
| Design tokens | Shared theme config | Export from design system package |

### Testing Tool

| Type | Tool | What It Tests | Speed |
|------|------|---------------|-------|
| Unit (logic) | Vitest | Pure functions, hooks, utils | < 1s |
| Component | RNTL / Testing Library | Render + interaction behavior | 1-5s |
| API mock | MSW v2 | Network layer interception | N/A |
| E2E mobile | Maestro | Full user flows on device/sim | 30-120s |
| E2E web | Playwright | Cross-browser user flows | 5-30s |
| Visual regression | Chromatic / Percy | Screenshot diff | 10-60s |
| Performance | Reassure | Render count/time regression | 5-15s |

### Build Profile (EAS)

| Profile | Use Case | Distribution | Signing |
|---------|----------|-------------|---------|
| `development` | Dev client with Expo DevTools | Internal | Debug |
| `preview` | QA / stakeholder testing | Internal (ad-hoc) | Debug/Release |
| `preview:device` | Physical device testing (iOS) | Device (ad-hoc provisioning) | Release |
| `production` | App store submission | Store | Release |
| `production:simulator` | CI testing on sim | Internal | Debug |

```json
// eas.json — recommended profiles
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal",
      "channel": "preview"
    },
    "production": {
      "autoIncrement": true,
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": { "ascAppId": "YOUR_APP_ID" },
      "android": { "track": "internal" }
    }
  }
}
```

### Deployment Strategy

| Strategy | When | Rollback Time | Risk |
|----------|------|---------------|------|
| **OTA (EAS Update)** | JS-only changes | Instant (point channel to previous branch) | Low |
| **Native build (EAS Build)** | Native dependency changes | Requires new store submission | Medium |
| **Staged rollout (Play Console)** | Production Android | Halt rollout immediately | Low |
| **TestFlight (App Store Connect)** | Production iOS beta | Remove build from testing | Low |
| **Vercel Preview** | Web feature branches | Delete deployment | None |
| **Vercel Production** | Web production | Instant rollback to previous | Low |
| **Vercel Rolling Release** | Gradual web production | Automatic on error spike | Very low |

### Auth Provider

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Clerk** | Drop-in UI components, fast integration, multi-tenant SaaS | You need deep customization of auth flows or self-hosting | Free up to 10K MAU; Pro $25/mo + $0.02/MAU |
| **Supabase Auth** | Full-stack Supabase projects, self-hosted requirements | You need advanced MFA, org management, or enterprise SSO | Free tier included with Supabase; Pro $25/mo |
| **Firebase Auth** | Google ecosystem, mobile-first apps, anonymous auth | You need server-side sessions or non-Google identity providers | Free up to 50K MAU; Blaze pay-as-you-go |
| **Auth.js (NextAuth)** | Open-source, framework-agnostic, full control | You need hosted infrastructure, pre-built UI, or enterprise support | Free (open-source); self-hosted |
| **Custom** | Unique compliance requirements, full ownership | Almost never -- prefer a managed solution unless legally required | Engineering time (most expensive long-term) |

### Database

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Postgres (Neon)** | Serverless apps, branching workflows, Next.js/Vercel | You need MySQL compatibility or embedded/edge DB | Free tier; Pro from $19/mo based on compute |
| **MySQL (PlanetScale)** | High-throughput OLTP, schema branching, Vitess-backed scale | You need JSON ops, full-text search, or PostGIS | Scaler from $39/mo; free tier available |
| **SQLite (Turso)** | Edge-first apps, embedded, low-latency reads, libSQL | You need complex joins, concurrent writes, or large datasets | Free 500 DBs; Scaler $29/mo |
| **Supabase** | Full-stack Postgres + Auth + Realtime + Storage in one | You need non-Postgres DB or want to avoid platform lock-in | Free tier; Pro $25/mo per project |

### ORM

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Drizzle** | SQL-first, edge runtimes, maximum type safety, lightweight | You need a mature migration GUI or extensive relation modeling | Free (open-source) |
| **Prisma** | Rapid prototyping, schema-first design, rich ecosystem | Bundle size matters (edge), you need raw SQL perf, or serverless cold starts hurt | Free (open-source); Accelerate/Pulse paid |

### Search

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Algolia** | Instant search, e-commerce, typo tolerance, analytics | Budget-constrained or need self-hosted | Free 10K requests/mo; Pay-as-you-go from $1/1K requests |
| **Typesense** | Self-hosted, open-source, fast and simple setup | You need AI-powered search or advanced analytics | Free (open-source); Typesense Cloud from $29.99/mo |
| **Meilisearch** | Developer-friendly, great defaults, multi-tenancy | You need geo-search or complex filtering at scale | Free (open-source); Cloud from $30/mo |
| **Postgres FTS** | Already on Postgres, simple search, no extra infra | You need typo tolerance, faceting, or sub-10ms results | Free (built into Postgres) |

### Feature Flag Provider

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Statsig** | Product experimentation, A/B testing, feature gates | You need on-prem or have <1ms latency requirements | Free up to 1M events; Pro usage-based |
| **LaunchDarkly** | Enterprise feature management, complex targeting rules | Budget-constrained or you need simpler setup | Starter from $10/seat/mo; Enterprise custom |
| **Unleash** | Self-hosted, open-source, GDPR-compliant | You need built-in experimentation or analytics | Free (open-source); Pro from $80/mo |
| **Vercel Flags** | Next.js/Vercel projects, edge evaluation, zero config | You need cross-platform flags beyond Vercel | Included with Vercel Pro; limited free tier |

### Real-time

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Socket.io** | Custom real-time logic, self-hosted, full control | You need managed infrastructure or global edge delivery | Free (open-source); self-hosted |
| **Ably** | Global edge messaging, guaranteed delivery, presence | Budget-constrained or simple pub/sub needs | Free 6M messages/mo; Pro from $29/mo |
| **Pusher** | Simple channels/events, quick integration, triggers | You need complex presence, history, or guaranteed ordering | Free 200K messages/day; Pro from $49/mo |
| **Supabase Realtime** | Already on Supabase, DB change streams, presence | You need custom protocols or non-Postgres sources | Included with Supabase plan |

### Email

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Resend** | Developer experience, React Email, modern API | You need advanced analytics or high-volume marketing | Free 3K emails/mo; Pro from $20/mo |
| **SendGrid** | High volume, marketing + transactional, legacy support | You want better DX or simpler API | Free 100/day; Essentials from $19.95/mo |
| **Postmark** | Transactional email, deliverability, speed | You need marketing emails or complex templates | Free 100/mo; $15/mo for 10K emails |
| **SES** | AWS ecosystem, lowest cost at scale, raw SMTP | You need managed templates, analytics, or quick setup | $0.10 per 1K emails (cheapest at scale) |

### Analytics

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **PostHog** | Open-source, product analytics + session replay + flags | You need enterprise support or advanced BI integrations | Free 1M events/mo; paid usage-based |
| **Mixpanel** | Event-based analytics, funnels, retention, user flows | You need session replay or feature flags built-in | Free 20M events/mo; Growth from $28/mo |
| **Firebase Analytics** | Mobile-first, Google ecosystem, free unlimited events | You need advanced funnels or non-Google integrations | Free (unlimited); BigQuery export included |
| **Amplitude** | Enterprise product analytics, behavioral cohorts, AI | Budget-constrained or need simpler setup | Free 100K MAU; Growth from $49/mo |

### Image CDN

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Cloudinary** | Rich transformations, video, AI-powered cropping | You need lower cost at scale or simpler API | Free 25 credits/mo; Plus from $89/mo |
| **Imgix** | Performance-focused, real-time params, simple CDN | You need video processing or advanced AI features | Free trial; Starter from $10/mo |
| **Cloudflare Images** | Budget, global CDN, simple resize/optimize | You need complex transformations or video | $5/mo for 100K images stored + $1/100K delivered |
| **Vercel (next/image)** | Next.js projects, zero config, automatic optimization | You need transformations beyond resize/format | Included with Vercel plan; 1K optimizations free |

### GraphQL Client

| Tool | Best For | When to Switch | Pricing |
|------|----------|----------------|---------|
| **Apollo Client** | Complex caching, local state, devtools, large teams | Bundle size matters or you need simpler setup | Free (open-source); Apollo Studio paid |
| **urql** | Lightweight, extensible via exchanges, SSR-friendly | You need normalized caching or advanced local state | Free (open-source) |

---

## Essential Commands

### Expo / EAS

```bash
# Create new project
npx create-expo-app@latest my-app --template tabs

# Start dev server
npx expo start

# Prebuild native directories (CNG)
npx expo prebuild --clean

# Run on device/simulator
npx expo run:ios
npx expo run:android

# Install Expo-compatible package
npx expo install react-native-reanimated

# EAS Build
eas build --platform ios --profile production
eas build --platform android --profile preview
eas build --platform all --profile production --auto-submit

# EAS Submit
eas submit --platform ios --latest
eas submit --platform android --latest

# EAS Update (OTA)
eas update --branch production --message "Fix checkout crash"
eas update --branch preview --message "Add dark mode toggle"

# Check fingerprint (native compatibility)
npx expo-fingerprint
eas update --check-fingerprint

# EAS Channel management
eas channel:create preview
eas channel:edit preview --branch preview-v2

# View build logs
eas build:list --limit 5
eas build:view <build-id>
```

### Vercel CLI

```bash
# Login
vercel login

# Link to existing project
vercel link

# Deploy preview
vercel

# Deploy production
vercel --prod

# Pull environment variables
vercel env pull .env.local

# List deployments
vercel ls

# Inspect deployment
vercel inspect <url>

# Promote preview to production
vercel promote <deployment-url>

# Rollback
vercel rollback

# View logs
vercel logs <deployment-url>

# Domain management
vercel domains add example.com
vercel domains ls
```

### Git / GitHub

```bash
# Feature branch workflow
git checkout -b feat/add-checkout-screen
git add -p                              # Stage hunks interactively
git commit -m "feat: add checkout screen with Stripe integration"
git push -u origin feat/add-checkout-screen

# Stash changes
git stash push -m "WIP: checkout styling"
git stash pop

# Interactive rebase (squash before merge)
git rebase -i HEAD~3

# Cherry-pick a fix
git cherry-pick <commit-sha>

# View CI status
gh pr checks

# Create PR
gh pr create --title "feat: checkout screen" --body "Adds Stripe checkout flow"

# Review PR
gh pr review --approve
gh pr merge --squash

# View PR diff
gh pr diff 123
```

### Package Management (pnpm)

```bash
# Install all dependencies
pnpm install

# Add dependency to specific workspace
pnpm --filter @repo/mobile add react-native-reanimated
pnpm --filter @repo/web add next

# Add dev dependency to root
pnpm add -Dw turbo

# Run script in specific workspace
pnpm --filter @repo/mobile start
pnpm --filter @repo/shared-ui build

# Run script in all workspaces
pnpm -r build

# Update dependencies interactively
pnpm update -i -r

# Check for outdated
pnpm outdated -r

# Why is this installed?
pnpm why react-native
```

### Turborepo

```bash
# Run build across all packages (with caching)
turbo build

# Run dev in specific app and its dependencies
turbo dev --filter=@repo/mobile

# Run tests affected by changes
turbo test --filter=...[HEAD~1]

# Run lint in parallel
turbo lint

# View task graph
turbo build --graph

# Clear cache
turbo clean

# Run with env passthrough
turbo build --env-mode=loose

# Remote caching (Vercel)
turbo login
turbo link
turbo build               # Now caches remotely
```

### Drizzle ORM

```bash
# Generate SQL migrations from schema changes
npx drizzle-kit generate

# Push schema directly to database (dev only, no migration files)
npx drizzle-kit push

# Open Drizzle Studio (database GUI)
npx drizzle-kit studio

# Run seed script
npx tsx src/db/seed.ts
# or with drizzle-seed
npx drizzle-seed
```

### Stripe CLI

```bash
# Listen for webhooks locally (forward to your dev server)
stripe listen --forward-to localhost:3000/api/webhooks/stripe

# Trigger a test event
stripe trigger payment_intent.succeeded
stripe trigger customer.subscription.created

# View recent logs
stripe logs tail

# View recent events
stripe events list --limit 5

# Create a test customer
stripe customers create --email test@example.com
```

### Changesets

```bash
# Add a changeset (interactive prompt for packages + semver bump)
npx changeset add

# Version packages (consumes changesets, updates package.json + CHANGELOG)
npx changeset version

# Publish changed packages to npm
npx changeset publish

# Check changeset status
npx changeset status
```

---

## Key Thresholds

### Web Performance (Core Web Vitals)

| Metric | Good | Needs Improvement | Poor | Source |
|--------|------|-------------------|------|--------|
| **LCP** | < 2.5s | 2.5-4.0s | > 4.0s | web.dev |
| **INP** | < 200ms | 200-500ms | > 500ms | web.dev |
| **CLS** | < 0.1 | 0.1-0.25 | > 0.25 | web.dev |
| **FCP** | < 1.8s | 1.8-3.0s | > 3.0s | web.dev |
| **TTFB** | < 800ms | 800-1800ms | > 1800ms | web.dev |

### Mobile Performance

| Metric | Target | Why |
|--------|--------|-----|
| **Cold start** | < 1.5s (iOS), < 2s (Android) | User abandonment threshold |
| **Frame rate** | 60fps (16.67ms/frame) | Perceptible jank below this |
| **JS thread** | < 16ms per frame | Blocks gesture response |
| **TTI (app)** | < 3s | Interactive before user gives up |
| **Bundle size (JS)** | < 2MB compressed | Download time on slow networks |
| **ANR rate** | < 0.47% | Google Play vitals threshold |
| **Crash-free rate** | > 99.9% | Industry standard for production |

### App Store / Play Store

| Metric | Threshold | Consequence |
|--------|-----------|-------------|
| **Crash rate (iOS)** | > 1% | App Review flags your app |
| **ANR rate (Android)** | > 0.47% | Play Console bad behavior warning |
| **Startup crash rate** | > 2% | Risk of app removal |
| **Binary size (iOS)** | > 200MB | Can't download over cellular |
| **Binary size (Android)** | > 150MB AAB | Play Store warning |
| **Memory (iOS)** | > 1.5GB | Jetsam kill (OOM) |

### Mobile Budgets

| Metric | Target | Context |
|--------|--------|---------|
| **Cold start** | < 1.5s | Users abandon after 1.5s on mobile |
| **JS bundle (compressed)** | < 15MB | Over-the-air download ceiling on slow 3G |
| **Memory usage** | < 200MB | Safe headroom before OS background kills |
| **Frame budget** | 16.67ms | 60fps target; jank visible above this |
| **First meaningful paint** | < 2s | Perceived loading threshold |

### Play Store Vitals

| Metric | Threshold | Consequence |
|--------|-----------|-------------|
| **ANR rate** | < 0.47% | Exceeding triggers bad behavior warning |
| **Crash rate** | < 1.09% | Exceeding triggers bad behavior warning |
| **Excessive wakeups** | < 10 per hour | Battery drain warning |
| **Stuck partial wakelock** | < 0.30% | Background energy warning |
| **Excessive background Wi-Fi** | < 0.10% | Network usage warning |

### Cookie and Storage Limits

| Limit | Value | Notes |
|-------|-------|-------|
| **Cookie size** | 4KB per cookie | Name + value + attributes combined |
| **Cookies per domain** | ~50 | Browser-dependent (Chrome ~180, Safari ~50) |
| **localStorage** | 5-10MB per origin | Synchronous, blocks main thread |
| **sessionStorage** | 5-10MB per origin | Cleared on tab close |
| **IndexedDB** | Varies (GB range) | Async, best for large structured data |

### TanStack Query Defaults

| Option | Default | Recommended | Why |
|--------|---------|-------------|-----|
| `staleTime` | 0 | 30s-5min | Prevents unnecessary refetches |
| `gcTime` | 5min | 10-30min | Keeps cache warm for back-nav |
| `refetchOnWindowFocus` | true | true (web), false (mobile) | Mobile has different lifecycle |
| `refetchOnReconnect` | true | true | Always refresh stale data |
| `retry` | 3 | 3 (queries), 0 (mutations) | Don't retry side effects |
| `retryDelay` | exponential | exponential | Back off on failures |

### Vercel Limits (Pro Plan)

| Resource | Limit |
|----------|-------|
| Serverless Function duration | 60s (Fluid: 300s) |
| Edge Function duration | 30s |
| Serverless Function size | 250MB (compressed) |
| Edge Function size | 4MB |
| Middleware execution | 30s |
| Build duration | 45min |
| Deployments per day | Unlimited |
| Bandwidth | 1TB/month |
| Edge Config size | 512KB |
| ISR revalidation | No limit |

---

## Architecture Decision Records (Quick Reference)

| Decision | Our Answer | Rationale |
|----------|-----------|-----------|
| Monorepo or polyrepo? | Monorepo | Shared types, atomic changes, single CI |
| pnpm or yarn? | pnpm | Strict deps, disk efficient, fast |
| Expo managed or bare? | Managed (CNG) | Prebuild when needed, no native dirs in git |
| REST or GraphQL? | REST + TanStack Query | Simpler, cacheable, sufficient for most apps |
| CSS-in-JS or utility? | Utility (NativeWind/Tailwind) | Compile-time, no runtime cost |
| Jest or Vitest? | Vitest | Faster, ESM-native, same API |
| Sentry or Crashlytics? | Sentry (primary) + Crashlytics (free backup) | Better symbolication, breadcrumbs, web+mobile |
| Feature flags service? | Edge Config (simple) / LaunchDarkly (complex) | Start simple, migrate when needed |

---

*Last updated: 2026-04-07.*

> **See also:** [Appendix A: Glossary](./appendix-glossary.md) | [Appendix C: Reading List](./appendix-reading-list.md)
