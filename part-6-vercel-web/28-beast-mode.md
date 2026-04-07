<!--
  CHAPTER: 31
  TITLE: Beast Mode — Frontend Operational Readiness
  PART: VI — Vercel & the Web
  PREREQS: None (all chapters helpful)
  KEY_TOPICS: operational readiness, day-one checklist, access matrix, dashboards, incident readiness, codebase navigation, tribal knowledge, frontend architect playbook
  DIFFICULTY: All levels
  UPDATED: 2026-04-07
-->

# Chapter 28: Beast Mode — Frontend Operational Readiness

> **Part VI — Vercel & the Web** | Prerequisites: None (all chapters helpful) | Difficulty: All levels

<details>
<summary><strong>TL;DR</strong></summary>

- Beast Mode means knowing how everything connects -- component library, state architecture, deployment pipeline, monitoring, performance budgets -- so you can operate the system under pressure
- On day one at a new codebase: get access to every console (Vercel, Sentry, Firebase, App Store Connect, Play Console), map the monorepo structure, identify the CI/CD pipeline, and set your baselines
- Build a frontend health dashboard covering crash-free rate, cold start, bundle size, build time, and Core Web Vitals; review it weekly
- Incident readiness means you can symbolicate a crash, push an OTA hotfix, communicate a staged rollout, and confirm resolution -- all within hours, not days
- Elevate your team by documenting architecture decisions, establishing component patterns, automating quality checks, and making the right thing the easy thing

</details>

> *"You need to be ready — your setup, synths, drum machines, sounds, presets. So when you show up at someone's studio you are ready to throw down."* — Nick Hook

The frontend architect equivalent: your Vercel dashboard is bookmarked, your Sentry filters are saved, your bundle analyzer is one command away, your performance budgets are memorized, and when someone asks "why is the app slow?" you don't google — you're already reading the profiler. When the app crashes in production on a Saturday morning, you don't scramble for the Crashlytics URL — you're already looking at the stack trace, you already know the OTA update path, and you're drafting the staged rollout before anyone else has opened their laptop.

This chapter gives you a repeatable system for becoming operationally dangerous as a frontend architect — fast. Whether you're joining a new company, inheriting a codebase, taking over a React Native app, or stepping into the architect role for the first time. This is the capstone. Everything in the previous 27 chapters converges here. The internals knowledge from Part I, the infrastructure patterns from Part II, the state architecture from Part III, the performance discipline from Part IV, the team patterns from Part V, and the deployment mastery from Part VI — all of it collapses into a single question: **can you operate this system under pressure?**

Beast Mode is the answer.

### In This Chapter
- The Frontend Architect's Beast Mode — what it means, why it matters
- Priority Triage — the "2 hours, 5 things" quick-hit adapted for frontend
- The Frontend Access Matrix — every console, every dashboard, every tool
- Codebase Navigation — monorepo structure, entry points, state architecture, CI/CD
- The Frontend Health Dashboard — crash-free rate, cold start, bundle size, build time
- Incident Readiness — crashes, symbolicating, OTA fixes, staged rollouts, communication
- The Frontend Architect's Playbook — mental models that multiply your impact
- Tribal Knowledge Extraction — the questions nobody documents
- Elevating Your Team — tooling, patterns, decision documents, architecture reviews
- The Beast Mode Checklist — the comprehensive operational readiness checklist

### Related Chapters
- [Ch 1: React Native Architecture & Internals] — the foundation of understanding crashes and performance
- [Ch 5: Expo Platform] — EAS, OTA updates, the deployment pipeline you need to master
- [Ch 9: State Management at Scale] — the architecture decisions you'll need to explain
- [Ch 13: Performance Optimization] — the profiling skills you'll use during incidents
- [Ch 14: Profiling & Debugging] — the tools behind incident diagnosis
- [Ch 20: Design Systems] — the component library that multiplies your team
- [Ch 25: Vercel & Next.js Deployment] — the web deployment pipeline

---

## 1. THE FRONTEND ARCHITECT'S BEAST MODE

Let's be direct about what a frontend architect actually is. Not the title on your LinkedIn. The job.

**You are the person who knows how everything connects.**

The component library. The state architecture. The deployment pipeline. The monitoring dashboards. The performance budgets. The design system. The build configuration. The navigation structure. The API layer. The caching strategy. The offline behavior. The accessibility compliance. The platform-specific quirks on iOS and Android. The CI/CD pipeline that turns a PR into a production deployment. The error tracking system that tells you what broke. The analytics pipeline that tells you what users actually do.

When someone asks "why is the app slow?" — you don't google. You're already reading the profiler. You know which screen they're talking about. You know the component tree for that screen. You know whether it's a rendering issue, a data fetching issue, a bundle size issue, or a platform issue. You know because you've already mapped the system, you've already set your baselines, and you've already identified the hot paths.

When the designer says "can we add a parallax scroll effect to the product page?" — you don't say "let me look into it." You already know the answer. You know the frame budget (16ms for 60fps), you know the threading model (JS thread vs. UI thread), you know whether the current scroll handler is already saturated, and you know whether `react-native-reanimated` worklets can handle it on the UI thread without blocking JS.

When the PM asks "how long to add dark mode?" — you don't guess. You know the design system architecture. You know whether the color tokens are centralized or scattered across 400 files. You know whether the theme provider is already wired through the component tree. You know the answer is "two days" or "two sprints" and you know *why*.

**That's Beast Mode for a frontend architect.** Not knowing everything — knowing where to look, what to check first, and how to connect the dots faster than anyone else in the room.

### Why Operational Readiness Matters More for Frontend

Here's the thing that backend engineers sometimes miss: **frontend is where users live.** A backend service can degrade gracefully — a slow API response makes the loading spinner spin longer, but the user waits. A frontend crash means a white screen. A blank page. The app force-closes. The user opens the App Store and writes a one-star review. The user tweets about it. The user switches to your competitor.

Frontend failures are **visceral.** They happen in the user's hands. There is no retry logic between a user's eyes and a white screen. There is no circuit breaker between a user's thumb and an unresponsive button. The frontend is the last line of defense, and when it fails, it fails loudly, publicly, and personally.

This is why frontend operational readiness isn't optional. It's not "nice to have." It's the difference between "our app is stable" and "did you see the Twitter thread?"

### The Architect's Superpower: Pattern Recognition

The most valuable thing a frontend architect brings to an incident isn't code. It's pattern recognition.

When you've mapped the system, built the mental models, and internalized the baselines — you develop an almost unconscious ability to spot anomalies. You glance at the crash-free rate and something feels wrong before you even read the number. You look at the bundle size chart and notice the slope changed. You read an error report and recognize the shape of a race condition you've seen before.

This isn't magic. It's the result of deliberate preparation. It's the compound interest of every dashboard you bookmarked, every baseline you recorded, every postmortem you read, every architecture diagram you sketched. Beast Mode is the system for building that preparation, fast.

```
The Frontend Architect's Truth Hierarchy:

  1. Production metrics      — what IS happening right now
  2. Error tracking data     — what IS breaking right now
  3. CI/CD pipeline state    — what WILL be deployed next
  4. The codebase            — what SHOULD happen (in theory)
  5. The design specs        — what someone INTENDED to happen
  6. The documentation       — what someone REMEMBERS intending (maybe)

When the Figma file and the production metrics disagree,
the production metrics are right.
```

---

## 2. PRIORITY TRIAGE — THE "2 HOURS, 5 THINGS" QUICK-HIT

You just joined the team. Maybe it's day one at a new company. Maybe you're taking over a frontend codebase that nobody has owned for six months. Maybe you volunteered to lead the mobile app after the previous architect left. You don't have two weeks to ramp up. You have two hours before the next standup. Here's how you spend them.

### The 5-Thing Rule

In your first two hours, you need exactly five things:

| # | Thing | Time | Why |
|---|-------|------|-----|
| 1 | **Local dev running** | 30 min | Clone, install, run. See the app on a simulator or device. If you can't run it, you can't reason about it. If it doesn't work in 30 minutes, that's a finding — file it. |
| 2 | **Deployment pipeline understood** | 20 min | Find the CI/CD config. Read the last 5 builds. Understand what "deploy" means — is it a merge to `main`? An EAS build? A Vercel push? A manual App Store submission? |
| 3 | **Monitoring dashboard bookmarked** | 15 min | Find the primary monitoring dashboard — Sentry, Crashlytics, Vercel Analytics, whatever the team uses. Bookmark it. Know what "healthy" looks like. |
| 4 | **Most recent incident postmortem read** | 15 min | Find the last time something broke. Read the postmortem. This tells you what *actually* breaks, not what theoretically could. |
| 5 | **Go-to person identified** | 10 min | Find your phone-a-friend. The engineer who's been here longest. Send them a message now. Don't wait until you're stuck at 2 AM. |

**Total: ~90 minutes.** The remaining 30 minutes are buffer for the inevitable: npm install failing, CocoaPods refusing to cooperate, Xcode needing an update, the Android emulator crashing, environment variables missing from `.env.example`.

### The Frontend-Specific Twist

Backend engineers can usually clone and run in one command. Frontend — especially React Native — has a dependency tree that reaches into native land. Here's the realistic version:

```bash
# Step 1: Clone and install
git clone git@github.com:org/mobile-app.git
cd mobile-app
npm install  # or yarn, or pnpm — check the lockfile to know which

# Step 2: Check for Expo or bare React Native
ls app.json expo.json 2>/dev/null  # Expo project?
ls ios/ android/ 2>/dev/null        # Bare RN or Expo prebuild?

# Step 3: Expo project — the happy path
npx expo start
# Press 'i' for iOS simulator, 'a' for Android emulator

# Step 3 (alt): Bare React Native
cd ios && pod install && cd ..
npx react-native run-ios
# or
npx react-native run-android

# Step 4: Next.js / web project
npm run dev
# Open http://localhost:3000

# Step 5: Monorepo? Check for workspaces
cat package.json | grep workspaces
ls packages/ apps/ 2>/dev/null
# Turborepo?
ls turbo.json 2>/dev/null
npx turbo run dev --filter=mobile
```

**If the build fails:** Do NOT spend more than 30 minutes debugging it solo. Ask your phone-a-friend. A broken local dev setup is a team problem, not a you problem — and fixing it is your first contribution. Update the README when you figure it out.

### The Anti-Pattern: "I'll Read the Architecture Docs First"

Do NOT spend your first two hours reading a 30-page architecture document or watching a recorded "team onboarding" video from 2024. Here's why:

- Architecture docs are frequently outdated. The team migrated from Redux to Zustand six months ago but the doc still says Redux.
- Watching a video is passive. You'll forget 80% by tomorrow.
- You learn frontend codebases by running them, breaking them, and fixing them. Not by reading about them.

The docs are valuable — but read them *after* you have the app running on your device and a mental model forming. They'll make ten times more sense.

### Quick-Hit Checklist

```markdown
## Day-One Quick-Hit (copy this into your notes)

- [ ] Cloned the primary repo and ran the app locally
- [ ] Located the CI/CD pipeline (EAS, Vercel, GitHub Actions) and read last 5 runs
- [ ] Bookmarked the main health dashboard (Sentry, Crashlytics, Vercel Analytics)
- [ ] Read one recent incident doc or postmortem
- [ ] Identified my go-to person for questions
- [ ] Noted the 3 biggest "I don't understand this yet" items
```

That last item — the "I don't understand this yet" list — is your learning roadmap for the next two weeks. Write it down now while everything is unfamiliar. In a week you'll have normalized the confusion and forgotten what puzzled you.

### Speed Matters, But Not for the Reason You Think

Getting combat-ready fast isn't about impressing your new team. It's about **reducing your mean-time-to-usefulness.** Every hour you spend unable to run the app, unable to find the error tracker, unable to understand the deployment pipeline — that's an hour where you can't help when something goes wrong.

The fastest way to earn trust on a new frontend team is to be the person who, during the first production crash, can say "I see the crash in Sentry, it's a null reference in the PaymentScreen component, started with the deploy at 2 PM" instead of "how do I access Sentry?"

---

## 3. THE FRONTEND ACCESS MATRIX

Access is the single biggest blocker for new engineers. You can be the best debugger in the world, but if you can't reach the crash logs, you're useless. Treat access acquisition as a project, not an afterthought.

### Build This Table on Day One

Fill in every cell. Empty cells are action items.

| System | URL / Entry Point | Access Level | Status | Ticket/Contact |
|--------|-------------------|-------------|--------|----------------|
| **Source Code** (GitHub/GitLab) | `github.com/org/...` | Write | ☐ | — |
| **CI/CD** (GitHub Actions / EAS / Vercel) | `github.com/org/.../actions` | Read | ☐ | — |
| **Expo Dashboard** (EAS) | `expo.dev/accounts/org` | Member | ☐ | — |
| **Vercel Dashboard** | `vercel.com/org` | Member | ☐ | — |
| **Error Tracking** (Sentry) | `sentry.io/organizations/org/` | Read | ☐ | — |
| **Crash Reporting** (Crashlytics) | `console.firebase.google.com` | Read | ☐ | — |
| **Firebase Console** | `console.firebase.google.com` | Editor | ☐ | — |
| **App Store Connect** | `appstoreconnect.apple.com` | Developer | ☐ | — |
| **Google Play Console** | `play.google.com/console` | Release Mgr | ☐ | — |
| **Analytics** (Amplitude/Mixpanel/PostHog) | varies | Read | ☐ | — |
| **Design** (Figma) | `figma.com/org` | Editor | ☐ | — |
| **Storybook** | `storybook.company.com` | — (public) | ☐ | — |
| **Communication** (Slack) | channels listed below | Member | ☐ | — |
| **Documentation** (Notion/Confluence) | varies | Read | ☐ | — |
| **Feature Flags** (LaunchDarkly/Statsig) | varies | Read | ☐ | — |
| **APM / Performance** (Datadog RUM / Vercel Speed Insights) | varies | Read | ☐ | — |
| **CDN / Edge Config** (Vercel Edge Config / CloudFront) | varies | Read | ☐ | — |
| **Backend API docs** (Swagger/OpenAPI) | varies | Read | ☐ | — |

**Pro tip:** Copy this table into a personal doc. Update it as access comes through. When the next frontend engineer joins, hand it to them. You just saved them a full day.

### Apple & Google — The Gatekeepers

These two deserve special attention because they are **uniquely painful** for frontend engineers:

**App Store Connect:**
- You need a personal Apple ID invited to the team's App Store Connect organization
- Different roles grant different access: Admin, App Manager, Developer, Marketing, Finance
- As an architect, you need at minimum "Developer" to view crash reports and TestFlight feedback
- Request "App Manager" if you'll be managing builds, metadata, and submissions
- **Critical:** Ask about the team's signing certificates and provisioning profiles. Who owns them? Where are they stored? If the answer is "on Dave's laptop" — that's a finding.

**Google Play Console:**
- You need a Google account invited to the developer console
- Roles: Owner, Admin, Release Manager, etc.
- As an architect, you need "Release Manager" at minimum for viewing crash data and managing releases
- **Critical:** Ask about the upload key vs. the app signing key. Google Play App Signing should be managing the signing key. If the team is managing it manually — that's a risk.

**EAS (Expo Application Services):**
- You need an Expo account added to the organization
- EAS Build, EAS Submit, and EAS Update are the three services you need
- Verify you can trigger a build: `eas build --profile preview --platform ios`
- Verify you can push an OTA update: `eas update --branch production --message "test"`
- **Critical:** Understand the update channels and branches. Which branch maps to production? Which to staging?

```bash
# Verify your EAS access
eas whoami
eas project:info

# Check the build profiles
cat eas.json

# Check recent builds
eas build:list --limit 5

# Check recent OTA updates
eas update:list --limit 5
```

### Vercel — The Web Deployment Layer

If the team has a Next.js web app or a marketing site on Vercel:

```bash
# Verify Vercel CLI access
vercel whoami

# Check linked project
vercel project ls

# Check recent deployments
vercel ls --limit 10

# Check environment variables (names only, not values)
vercel env ls
```

**Bookmark immediately:**
- Production deployment: `https://vercel.com/org/project`
- Analytics: `https://vercel.com/org/project/analytics`
- Speed Insights: `https://vercel.com/org/project/speed-insights`
- Logs: `https://vercel.com/org/project/logs`
- Environment variables: `https://vercel.com/org/project/settings/environment-variables`

### Communication Channels

Join these Slack channels immediately:

| Channel | Purpose |
|---------|---------|
| `#team-frontend` or `#team-mobile` | Day-to-day team communication |
| `#incidents` or `#ops` | Active incident coordination |
| `#releases` or `#deploys` | Build and deployment notifications |
| `#design-system` | Component library discussions |
| `#app-reviews` | App Store and Google Play review monitoring |
| `#frontend-guild` or `#engineering` | Cross-team frontend discussions |
| `#alerts` | Automated alert notifications from Sentry/Crashlytics |
| `#product-feedback` | User feedback and bug reports |

**Mute strategically.** Join everything, but mute the noisy channels. Check `#deploys` when you need it, not when it notifies you.

### End-to-End Verification

Once you have access to everything, verify the full chain works:

```bash
# 1. Can you read the code?
git log --oneline -5

# 2. Can you build it?
npm run build  # or: eas build --profile preview --platform ios

# 3. Can you run it locally?
npx expo start  # or: npm run dev

# 4. Can you see the errors?
# Open Sentry, check for recent issues on your project

# 5. Can you see a deployment?
# Open EAS dashboard or Vercel dashboard, check last deployment

# 6. Can you see production metrics?
# Open Crashlytics or Vercel Analytics, verify data is flowing

# 7. Can you access the design files?
# Open Figma, verify you can see the current design system

# 8. Can you push an OTA update to a staging channel?
# eas update --branch staging --message "access test"
```

If all 8 pass, you are tooled up. If any fail, you now have a specific list of access gaps to close.

---

## 4. CODEBASE NAVIGATION

You have access. You can build and run the app. Now you need to understand what you're looking at. The goal is not to understand every component — it's to build a **working mental model** good enough to reason about failures, trace user interactions, and know where to look when something breaks.

### 4.1 Understanding the Monorepo Structure

Modern frontend projects almost always live in a monorepo. Understanding the structure is step one.

```bash
# What's the package manager?
ls package-lock.json yarn.lock pnpm-lock.yaml bun.lock 2>/dev/null

# Is it a monorepo?
cat package.json | grep -A 5 workspaces
ls turbo.json 2>/dev/null    # Turborepo
ls nx.json 2>/dev/null       # Nx
ls lerna.json 2>/dev/null    # Lerna (legacy, but still around)

# What packages/apps exist?
ls packages/ apps/ 2>/dev/null

# Common monorepo structure:
# apps/
#   mobile/          ← React Native / Expo app
#   web/             ← Next.js web app
#   admin/           ← Admin dashboard
# packages/
#   ui/              ← Shared component library
#   config/          ← Shared ESLint, TypeScript configs
#   api-client/      ← Generated or handwritten API client
#   utils/           ← Shared utilities
#   types/           ← Shared TypeScript types
```

**Document the structure for yourself:**

```markdown
## Monorepo Map — [Project Name]

**Build system:** Turborepo + pnpm workspaces
**Apps:**
- apps/mobile — React Native / Expo (SDK 52)
- apps/web — Next.js 15 (App Router)
- apps/admin — Next.js 15 (internal tools)

**Packages:**
- packages/ui — Shared component library (Tamagui-based)
- packages/api — tRPC client + React Query hooks
- packages/config — Shared ESLint, TypeScript, Tailwind configs
- packages/types — Shared TypeScript types (generated from Prisma)

**Key commands:**
- `turbo run dev --filter=mobile` — run mobile app
- `turbo run dev --filter=web` — run web app
- `turbo run build` — build everything
- `turbo run lint` — lint everything
- `turbo run test` — run all tests
```

### 4.2 Finding the Entry Points

Every frontend app has entry points. These are your anchor points — everything fans out from them.

**React Native / Expo:**

```bash
# The app entry point
cat app.json | grep -i main
# Usually: index.js or App.tsx

# If using Expo Router (file-based routing):
ls app/ 2>/dev/null
ls app/_layout.tsx 2>/dev/null     # Root layout
ls app/(tabs)/ 2>/dev/null         # Tab navigation
ls app/+not-found.tsx 2>/dev/null  # 404 screen

# If using React Navigation (manual routing):
grep -r "createNativeStackNavigator\|createBottomTabNavigator\|NavigationContainer" \
  --include="*.tsx" --include="*.ts" -l src/

# Find all screen components
grep -r "Screen\|screen" --include="*.tsx" -l src/navigation/ src/screens/
```

**Next.js (App Router):**

```bash
# The app entry point is the file system
ls app/ 2>/dev/null
ls app/layout.tsx 2>/dev/null      # Root layout
ls app/page.tsx 2>/dev/null        # Home page
ls app/api/ 2>/dev/null            # API routes

# Find all pages
find app/ -name "page.tsx" -o -name "page.ts" 2>/dev/null

# Find all layouts
find app/ -name "layout.tsx" -o -name "layout.ts" 2>/dev/null

# Find all API routes
find app/api/ -name "route.tsx" -o -name "route.ts" 2>/dev/null

# Find Server Actions
grep -r "use server" --include="*.ts" --include="*.tsx" -l app/
```

**Build a quick entry point map:**

```markdown
## Entry Points — Mobile App

### Navigation Structure
- app/_layout.tsx — Root layout (auth check, providers)
  - app/(auth)/_layout.tsx — Auth flow (login, signup, forgot password)
  - app/(tabs)/_layout.tsx — Main tab bar
    - app/(tabs)/home/index.tsx — Home screen
    - app/(tabs)/search/index.tsx — Search screen
    - app/(tabs)/profile/index.tsx — Profile screen
  - app/product/[id].tsx — Product detail (deep linked)
  - app/checkout/index.tsx — Checkout flow

### Key Providers (wrapped in root layout)
- ThemeProvider (design system tokens)
- QueryClientProvider (TanStack Query)
- AuthProvider (user session)
- AnalyticsProvider (event tracking)
```

### 4.3 Tracing a User Action End-to-End

Pick the most important user action — "user adds item to cart" or "user completes purchase" — and trace it from tap to API to response to screen update. This single exercise teaches you more about the codebase than reading 50 files.

**The trace:**

```
User taps "Add to Cart" button
  → ButtonComponent (packages/ui/src/Button.tsx)
    → onPress handler (app/product/[id].tsx)
      → addToCart mutation (packages/api/src/cart.ts)
        → TanStack Query useMutation
          → POST /api/cart/items (backend API)
            → Response: { success: true, cartCount: 3 }
          → onSuccess callback
            → Invalidate cart query (triggers refetch)
            → Zustand store update (optimistic UI)
              → Cart badge re-renders with new count
            → Toast notification ("Added to cart!")
            → Analytics event (track("add_to_cart", { productId }))
```

**How to trace it yourself:**

```bash
# Step 1: Find the button
grep -r "Add to Cart\|addToCart\|add-to-cart" --include="*.tsx" -l

# Step 2: Find the handler
grep -r "addToCart\|useAddToCart" --include="*.tsx" --include="*.ts" -l

# Step 3: Find the API call
grep -r "cart" --include="*.ts" -l packages/api/

# Step 4: Find the state update
grep -r "cart" --include="*.ts" -l src/store/ packages/store/

# Step 5: Find the UI that reacts to the state
grep -r "useCart\|cartCount\|cartItems" --include="*.tsx" -l
```

### 4.4 Understanding the State Architecture

This is where Chapter 9 (State Management at Scale) meets reality. You need to know, for this specific codebase, where state lives.

```bash
# What state management library?
grep -r "zustand\|jotai\|redux\|recoil\|legend-state\|valtio\|xstate" package.json

# Where are the stores defined?
find . -path "*/store/*" -o -path "*/stores/*" -o -path "*/state/*" \
  | grep -v node_modules | head -20

# Where is TanStack Query configured?
grep -r "QueryClient\|queryClient\|QueryClientProvider" \
  --include="*.tsx" --include="*.ts" -l | grep -v node_modules

# Where is form state managed?
grep -r "useForm\|react-hook-form\|formik" \
  --include="*.tsx" --include="*.ts" -l | grep -v node_modules | head -10

# Where does data persist?
grep -r "AsyncStorage\|MMKV\|SecureStore\|localStorage\|sessionStorage" \
  --include="*.tsx" --include="*.ts" -l | grep -v node_modules
```

**Document it:**

```markdown
## State Architecture — [App Name]

**Server state:** TanStack Query v5
  - Query client configured in: app/_layout.tsx
  - API hooks in: packages/api/src/hooks/
  - Cache time: 5 min stale, 30 min garbage collection

**Client state:** Zustand v5
  - Stores in: packages/store/src/
  - Auth store: user session, tokens (persisted to MMKV)
  - UI store: theme, sidebar, active filters (not persisted)
  - Cart store: optimistic cart state (persisted to MMKV)

**Form state:** React Hook Form + Zod
  - Used in: checkout flow, profile editing, auth forms
  - Schemas in: packages/types/src/schemas/

**Persistence:** MMKV (React Native), localStorage (web)
  - Auth tokens: SecureStore (RN) / httpOnly cookies (web)
  - User preferences: MMKV / localStorage
  - Cache: TanStack Query built-in
```

### 4.5 Identify the Load-Bearing Code

Every frontend codebase has a handful of files that everything depends on. Touch them carefully, test them thoroughly, know them well.

**How to find them:**

```bash
# Most frequently changed files (hot spots)
git log --pretty=format: --name-only --since="6 months ago" \
  | grep -E '\.(tsx?|jsx?)$' | sort | uniq -c | sort -rn | head -20

# Most authors per file (many people touch it = shared dependency)
for f in $(git ls-files "*.tsx" "*.ts" | head -50); do
  authors=$(git log --format='%ae' -- "$f" | sort -u | wc -l)
  echo "$authors $f"
done | sort -rn | head -15

# Largest component files (complexity accumulates in large files)
find . -name "*.tsx" -not -path "*/node_modules/*" -exec wc -l {} + \
  | sort -rn | head -20

# Most imported module (everything depends on it)
grep -rh "from ['\"]" --include="*.ts" --include="*.tsx" \
  | grep -v node_modules \
  | sed "s/.*from ['\"]//; s/['\"].*//g" \
  | sort | uniq -c | sort -rn | head -20
```

**What load-bearing frontend code looks like:**

| File | Why It's Load-Bearing |
|------|----------------------|
| Root layout (`app/_layout.tsx`) | Every screen inherits from it. Providers, auth, theme. |
| Navigation configuration | Changing it can break deep links, back behavior, and screen transitions |
| Auth middleware / provider | Every authenticated screen depends on it |
| API client configuration | Every data fetch goes through it |
| Design system tokens | Every component references them |
| The "utils" or "helpers" grab bag | Everything imports from it, 2000+ lines, 15 authors |
| Error boundary wrapper | Determines crash recovery behavior for every screen |
| Build configuration (`next.config.js`, `app.config.ts`) | Changes here affect everything |

**Real-world example:** An architect runs the most-changed-files command and discovers that `packages/ui/src/Button.tsx` has been modified 89 times in 6 months by 11 different authors. It's the most-used component in the entire app — every screen imports it. This means any change to Button's API is a breaking change for dozens of screens. Knowing this on day one means she'll treat Button changes with extra caution, require thorough visual regression testing, and advocate for a stable, well-documented API.

### 4.6 Reading the CI/CD Pipeline

You need to understand how code becomes a production app before you need to deploy under pressure.

```bash
# Find the CI/CD config
ls .github/workflows/ 2>/dev/null
cat eas.json 2>/dev/null
cat vercel.json 2>/dev/null

# Read the main workflow
cat .github/workflows/ci.yml
cat .github/workflows/deploy.yml

# Check recent workflow runs
gh run list --limit 10
```

**Key questions to answer:**

| Question | Where to Find It |
|----------|------------------|
| What triggers a build? | `.github/workflows/*.yml` — `on:` section |
| How long does a build take? | EAS dashboard or GitHub Actions run history |
| How do you deploy to staging? | Usually a branch push or manual trigger |
| How do you deploy to production? | Merge to `main`? Tag? Manual approval? |
| How do you push an OTA update? | `eas update` command and branch configuration |
| How do you submit to the App Store? | `eas submit` or manual from App Store Connect |
| How do you roll back? | OTA update to previous version, or redeploy previous build |
| What's the test suite? | `npm test`, `turbo run test`, check CI config |
| What's the lint/type-check setup? | `npm run lint`, `npm run typecheck` |

**Document the deployment pipeline:**

```markdown
## Deployment Pipeline — [App Name]

### Mobile (React Native / Expo)
1. PR opened → GitHub Actions: lint + typecheck + test
2. PR merged to `main` → EAS Build (preview profile) → TestFlight + Internal Track
3. Manual trigger → EAS Build (production profile) → App Store + Play Store submission
4. Hotfix → `eas update --branch production` → OTA update (no store review)

### Web (Next.js / Vercel)
1. PR opened → Vercel Preview Deployment (automatic)
2. PR merged to `main` → Vercel Production Deployment (automatic)
3. Rollback → Vercel Dashboard → Promote previous deployment

### Build times (baseline):
- CI (lint + test): ~4 min
- EAS Build (iOS): ~15 min
- EAS Build (Android): ~12 min
- Vercel deploy: ~2 min
```

---

## 5. THE FRONTEND HEALTH DASHBOARD

This is the single biggest force multiplier for a frontend architect. Every healthy frontend project should have a dashboard covering these metrics. If your team doesn't have one, building it is your highest-leverage first contribution.

### 5.1 The Essential Metrics

| Metric | What It Measures | Why It Matters | Target |
|--------|-----------------|----------------|--------|
| **Crash-Free Rate** | % of sessions without a crash | The single most important quality metric | > 99.5% (mobile), > 99.9% (web) |
| **ANR Rate** (Android) | Application Not Responding events | User perceives app as frozen | < 0.5% |
| **Cold Start P50 / P95** | Time from app launch to interactive | First impression, affects retention | P50 < 1s, P95 < 2s |
| **TTI (Time to Interactive)** | Time to usable on web | Core Web Vital proxy | < 3.8s (web) |
| **Error Rate** | JS errors per session | Code quality signal | < 1% of sessions |
| **Active Users** (DAU / WAU / MAU) | User engagement | Business health proxy | Trending up or stable |
| **Bundle Size** | JS bundle + assets | Download time, parse time, memory | Trending down or stable |
| **Build Time** | CI/CD pipeline duration | Developer velocity | < 15 min (mobile), < 5 min (web) |
| **OTA Update Adoption** | % of users on latest update | Rollout health | > 90% within 24h |
| **API Latency (client-perceived)** | Time from request to render | User experience | P95 < 2s |
| **Frame Rate** | FPS during animations/scrolling | Smoothness perception | > 55 FPS on target devices |
| **LCP / CLS / INP** (web) | Core Web Vitals | SEO and user experience | Green thresholds per Google |

### 5.2 Where to Find These Metrics

**Crash-Free Rate and ANR Rate:**
- **Firebase Crashlytics:** `console.firebase.google.com` → Your project → Crashlytics
- **Sentry:** `sentry.io` → Your project → Releases → select a release → crash-free sessions
- **App Store Connect:** App Analytics → Metrics → Crashes
- **Google Play Console:** Android Vitals → Crashes & ANRs

**Cold Start / TTI:**
- **Custom instrumentation:** You almost certainly need to measure this yourself
- **React Native:** Measure from `AppRegistry.registerComponent` to first meaningful paint
- **Expo:** Use `expo-splash-screen` timing + custom performance marks
- **Next.js / Web:** Vercel Speed Insights, or `performance.mark()` / `performance.measure()`

```typescript
// React Native cold start measurement pattern
import { PerformanceObserver } from 'react-native-performance';

const appStartTime = global.__APP_START_TIME__;  // Set in index.js

function HomeScreen() {
  useEffect(() => {
    const coldStartMs = Date.now() - appStartTime;
    analytics.track('cold_start', { duration_ms: coldStartMs });
  }, []);
}
```

**Bundle Size:**
```bash
# React Native — analyze bundle
npx react-native-bundle-visualizer

# Expo
npx expo export --dump-sourcemap
# Then use source-map-explorer on the output

# Next.js
ANALYZE=true npm run build
# Requires @next/bundle-analyzer in next.config.js

# Track over time — add to CI
npx bundlesize --config bundlesize.config.json
```

**Build Time:**
- GitHub Actions: Check run duration in the Actions tab
- EAS Build: Check build duration on `expo.dev`
- Vercel: Check deployment duration on the Vercel dashboard

**Core Web Vitals (web only):**
- **Vercel Speed Insights:** Automatic if deployed on Vercel
- **Google Search Console:** Core Web Vitals report
- **web-vitals library:** `npm install web-vitals` for custom reporting

### 5.3 Building Your Dashboard

If the team has no unified frontend health dashboard, build one. Here's what to include:

```markdown
## Frontend Health Dashboard — [App Name]
## (Create in Grafana, Datadog, or even a Notion page updated weekly)

### Row 1: Stability
- Crash-free rate (iOS) — line chart, 7-day rolling
- Crash-free rate (Android) — line chart, 7-day rolling
- ANR rate (Android) — line chart, 7-day rolling
- JS error rate — line chart, 7-day rolling

### Row 2: Performance
- Cold start P50 (iOS) — line chart, 7-day rolling
- Cold start P50 (Android) — line chart, 7-day rolling
- API latency P95 (client-side) — line chart
- Frame rate P10 during scroll — line chart

### Row 3: Growth & Engagement
- DAU — bar chart, 30-day trend
- Session duration P50 — line chart
- OTA update adoption rate — bar chart per release

### Row 4: Build & Deploy
- Bundle size (JS) — line chart, trend over last 30 builds
- Build time (CI) — line chart, trend over last 30 builds
- Deployment frequency — bar chart, deploys per week

### Row 5: Web Vitals (if applicable)
- LCP P75 — line chart
- CLS P75 — line chart
- INP P75 — line chart
```

### 5.4 Learn What "Normal" Looks Like

You cannot spot anomalies without knowing the baseline. This is non-negotiable.

**The 15-minute baseline exercise:** During a calm period, open each data source and capture these numbers:

| Metric | Baseline Value | Date Observed | Source |
|--------|---------------|---------------|--------|
| Crash-free rate (iOS) | _____% | _____ | Crashlytics |
| Crash-free rate (Android) | _____% | _____ | Crashlytics |
| ANR rate (Android) | _____% | _____ | Play Console |
| Cold start P50 (iOS) | _____ ms | _____ | Custom metric |
| Cold start P50 (Android) | _____ ms | _____ | Custom metric |
| JS error rate | _____% | _____ | Sentry |
| Bundle size (JS) | _____ KB | _____ | Build output |
| Build time (CI) | _____ min | _____ | GitHub Actions |
| DAU | _____ | _____ | Analytics |
| OTA update adoption (24h) | _____% | _____ | EAS |

**Write these numbers down.** Tape them to your monitor. When the crash-free rate drops from 99.7% to 98.2%, you'll know instantly that something is wrong — because you know what 99.7% looks like.

### 5.5 The "Metrics Don't Lie" Principle for Frontend

| What the Docs Say | What the Metrics Show | Who's Right? |
|--------------------|-----------------------|--------------|
| "App launches in under 1 second" | Cold start P95 is 3.4 seconds on Android | The metrics |
| "We support iOS 15+" | Crashlytics shows 12% of crashes on iOS 15 | Investigate — might be a compatibility issue |
| "Bundle is optimized" | Bundle grew 400KB in the last 3 weeks | The bundle analyzer |
| "We have 99.9% crash-free rate" | Crashlytics says 99.2% over last 7 days | Crashlytics |
| "OTA updates reach everyone in minutes" | EAS shows 67% adoption after 24 hours | EAS dashboard |
| "The app is fast" | Frame rate drops to 22 FPS on the product list | The profiler |

Trust observable reality over stated intention. Always.

---

## 6. INCIDENT READINESS — WHEN THE APP CRASHES IN PRODUCTION

The whole chapter builds to this moment: something breaks in production. The crash-free rate is plummeting. Users are leaving one-star reviews. The PM is panicking. Slack is lighting up.

You have two options. Freeze and pray. Or execute a script you've already rehearsed.

### 6.1 The First 5 Minutes — Frontend Incident Script

**Step 1: Check the crash report.**

Read the actual alert. What platform? iOS, Android, or web? What's the crash type? Native crash, JS error, ANR? When did it start? Don't skip this — half of all incident confusion starts with someone reacting to an alert they didn't read.

**Step 2: Open the error tracker.**

Open Sentry or Crashlytics. Sort by "first seen" or "new in this release." Is this a new crash or a spike in an existing one?

```
# Sentry — quick triage
1. Go to Issues → filter by "is:unresolved firstSeen:-1h"
2. Sort by "Events" (most frequent first)
3. Look for: new issues that correlate with the alert time

# Crashlytics — quick triage
1. Go to Crashlytics → Issues
2. Filter by "version" (latest release)
3. Sort by "Users impacted"
4. Look for: new issues with high user count
```

**Step 3: Check recent deployments / releases.**

Was there a new app version released recently? Was there an OTA update pushed? Was there a backend API change?

```bash
# Check recent EAS updates
eas update:list --limit 5

# Check recent EAS builds
eas build:list --limit 5

# Check recent Vercel deployments
vercel ls --limit 5

# Check recent merges to main
git log --oneline --merges --since="24 hours ago"
```

**Step 4: Identify the blast radius.**

How many users are affected? Is it all users or a specific segment?

| Question | Where to Check |
|----------|----------------|
| All platforms or one? | Crashlytics / Sentry — filter by platform |
| All versions or one? | Crashlytics / Sentry — filter by version |
| All devices or specific? | Crashlytics — filter by device model |
| All OS versions or specific? | Crashlytics — filter by OS version |
| Geographic pattern? | Analytics — check by region |
| Behind a feature flag? | Feature flag dashboard — check flag status |

**Step 5: Determine severity and communicate.**

```markdown
## Frontend Incident Severity Guide

**SEV-1 — Production Down**
- App crashes on launch for all users
- White screen / blank page on web
- Payment flow completely broken
→ All hands. OTA fix or rollback immediately.

**SEV-2 — Major Feature Broken**
- A primary flow (checkout, login, search) is broken for significant users
- Crash-free rate dropped below 99%
→ Focused response. Fix within hours.

**SEV-3 — Degraded Experience**
- Performance degradation (slow screens, janky scrolling)
- Minor feature broken, workaround exists
- Crash-free rate dropped but still above 99%
→ Normal priority. Fix in next release.

**SEV-4 — Cosmetic / Edge Case**
- Visual glitch on specific device
- Error in non-critical feature
- Only affects < 0.1% of users
→ Log it. Fix when convenient.
```

### 6.2 Reading Crash Reports — Crashlytics and Sentry

This is a skill. Not a glance-and-hope skill — a deliberate, practiced skill.

**Reading a Crashlytics crash report:**

```
Fatal Exception: java.lang.NullPointerException
  at com.myapp.MainApplication.getReactNativeHost
  (MainApplication.java:32)

Non-fatal Exception: Error
  TypeError: Cannot read properties of undefined (reading 'map')
  at ProductList (ProductList.tsx:47)
  at renderWithHooks (react-dom.development.js:14985)
```

**What to look for:**

| Element | What It Tells You |
|---------|-------------------|
| **Exception type** | `NullPointerException` = native crash. `TypeError` = JS error |
| **Stack trace** | The call chain — trace it back to your code (ignore framework frames) |
| **File and line number** | The exact location — but only if sourcemaps are properly configured |
| **Affected versions** | Which app version introduced the crash |
| **Device / OS breakdown** | Is it device-specific? OS-specific? |
| **Frequency** | How many times? How many users? Is it growing? |
| **Breadcrumbs** | What the user did before the crash (navigation, taps, API calls) |

**Symbolicating a crash (React Native):**

When you see a crash with obfuscated stack traces (minified JavaScript or native code without symbols), you need to symbolicate it.

```bash
# For JavaScript crashes — ensure sourcemaps are uploaded
# Sentry:
npx sentry-cli releases files <release> upload-sourcemaps \
  --dist <dist> ./dist

# For native iOS crashes — ensure dSYMs are uploaded
# Sentry:
npx sentry-cli upload-dsym

# Crashlytics does this automatically if configured in the build
# Check: ios/Podfile should have the Crashlytics upload script
# Check: android/app/build.gradle should have the Crashlytics plugin

# EAS builds upload sourcemaps automatically if sentry-expo is configured
```

**The most common "I can't read the crash" problems:**

| Problem | Cause | Fix |
|---------|-------|-----|
| Stack trace is minified JS | Sourcemaps not uploaded | Upload sourcemaps as part of build pipeline |
| Native crash with no symbols | dSYMs not uploaded (iOS) or ProGuard mapping missing (Android) | Configure upload in CI/CD |
| "Unknown" file names | Source map mismatch (wrong version) | Ensure release/dist tags match |
| No breadcrumbs | Sentry/Crashlytics not properly initialized | Check SDK initialization in root layout |

### 6.3 Pushing an OTA Fix

This is the frontend architect's superpower: **you can fix production without going through the App Store review process.** OTA (Over-The-Air) updates let you push JavaScript bundle changes directly to users' devices.

**When to use OTA vs. a new build:**

| Scenario | OTA Update? | New Build? |
|----------|-------------|------------|
| JS bug fix (logic error, typo, wrong API URL) | Yes | No |
| New UI component using existing native modules | Yes | No |
| Updated native module version | No | Yes |
| New native module added | No | Yes |
| Changed `app.json` configuration | No | Yes |
| iOS/Android permission changes | No | Yes |
| Asset changes (images, fonts) already bundled | Yes | No |

**Pushing an OTA fix with EAS Update:**

```bash
# Step 1: Fix the bug on a hotfix branch
git checkout -b hotfix/crash-fix
# Make the fix...
git add -A && git commit -m "fix: null check in ProductList"

# Step 2: Push the update to the production branch
eas update --branch production \
  --message "Fix null reference crash in ProductList" \
  --platform all

# Step 3: Verify the update
eas update:list --branch production --limit 3

# Step 4: Monitor adoption
# Check EAS dashboard for update adoption rate
# Check Crashlytics for crash rate change
```

**Staged rollout for OTA updates:**

EAS Update doesn't have built-in percentage rollout (as of early 2026), but you can implement staged rollouts using branches and runtime version targeting:

```bash
# Strategy 1: Canary branch
# Push to a canary branch first, monitored by internal users
eas update --branch canary --message "Fix: ProductList crash"

# After validation, promote to production
eas update --branch production --message "Fix: ProductList crash (validated)"

# Strategy 2: Feature flag gate
# Deploy the fix behind a feature flag, gradually increase rollout
# This requires the fix to be flag-gated in the code
```

**For web (Vercel):**

```bash
# Vercel deployments are instant — no OTA concept needed
# Just merge the fix and it deploys automatically

# If you need to rollback:
# Option 1: Promote a previous deployment
vercel promote <deployment-url>

# Option 2: Revert the commit and let CI redeploy
git revert HEAD --no-edit && git push origin main
```

### 6.4 Coordinating a Staged App Store Rollout

When the fix requires a new native build (not just a JS change), you need the App Store and Google Play Store.

**Google Play staged rollout:**

1. Build and submit: `eas submit --platform android`
2. In Google Play Console → Production → Releases → create new release
3. Set rollout percentage: start at 5%, increase to 20%, 50%, 100%
4. Monitor crash-free rate at each stage before increasing

```
Staged Rollout Timeline (typical):
  Day 1: 5% rollout  → Monitor crash rate for 24h
  Day 2: 20% rollout → Monitor for 24h
  Day 3: 50% rollout → Monitor for 24h
  Day 4: 100% rollout

If crash rate increases at any stage → HALT rollout → investigate
```

**App Store (iOS):**

Apple supports phased release:
1. Build and submit: `eas submit --platform ios`
2. In App Store Connect → select the version → Phased Release
3. Apple's phased release is automatic: 1% → 2% → 5% → 10% → 20% → 50% → 100% over 7 days
4. You can pause and resume at any time

**Critical: App Store review takes time.** For urgent fixes:
- If the fix is JS-only → use OTA update (minutes, not days)
- If the fix requires native changes → submit for expedited review (Apple supports this)
- Always have OTA as your first line of defense

### 6.5 The "What Changed?" Reflex for Frontend

Here's the single most important mental habit for frontend incident response:

> **Ask "what changed?" before you ask "what's broken?"**

Roughly 90% of frontend production incidents are caused by a change. Not a spontaneous failure — a change. Your job is to find which one.

**The Frontend Change Checklist:**

| Change Type | Where to Check | Typical Signal |
|---|---|---|
| **OTA update** | EAS dashboard, `eas update:list` | Errors correlate with update timestamp |
| **New app version** | App Store Connect, Play Console | Errors only in latest version |
| **Backend API change** | Backend deploy channel, API changelog | Data shape errors, 4xx/5xx spikes |
| **Feature flag change** | LaunchDarkly/Statsig audit log | Errors only for flagged user segment |
| **Config / env change** | Vercel env vars, Firebase Remote Config | Behavior change without new code |
| **Third-party SDK update** | package.json diff, lockfile changes | New error types from vendor code |
| **Device / OS update** | Crashlytics device filter | Errors spike on specific OS version |
| **CDN / edge config** | Vercel Edge Config, CloudFront | Asset loading failures, CORS errors |

**Train yourself to run through this checklist reflexively.** When someone posts "the app is crashing," your first question is not "what screen?" — it's "what shipped in the last hour?"

**The correlation technique:** Open Sentry or Crashlytics. Look at the crash timeline graph. Now overlay the deployment timeline. If the crash spike starts within minutes of a deploy or OTA push, that's your leading hypothesis. You don't need to prove it yet — you need to rollback first and investigate after.

### 6.6 Communicating During a Frontend Incident

When you're the frontend architect and the app is crashing:

**Internal communication (Slack):**

```
🔴 FRONTEND INCIDENT — [App Name] [iOS/Android/Web]

Impact: Crash-free rate dropped from 99.7% to 97.2%
Affected: [version], [platform], [user segment]
Started: [time]
Likely cause: [recent deploy / OTA update / backend change / unknown]

Current status: Investigating
Next update: [time]

Dashboard: [link]
Sentry: [link]
```

**External communication (if user-facing status page exists):**

```
[App Name] — Some users may experience crashes
We're aware of an issue affecting [platform] users and are actively
working on a fix. We expect to resolve this within [timeframe].
```

**Timeline notes (for the postmortem):**

Keep a running timeline in the incident channel. This is the single highest-value thing you can do during an incident if you're not the one writing the fix.

```markdown
| Time (UTC) | Event | Source |
|---|---|---|
| 14:30 | OTA update pushed to production branch | EAS Dashboard |
| 14:45 | First crash reports appearing in Sentry | Sentry |
| 14:52 | Crash-free rate drops below 99% | Crashlytics |
| 14:55 | Root cause identified: null reference in ProductList | Sentry stack trace |
| 15:02 | Fix merged, OTA update pushed | EAS Dashboard |
| 15:15 | Crash rate stabilizing, new update at 40% adoption | EAS + Crashlytics |
| 15:45 | Crash-free rate back to 99.5%, monitoring | Crashlytics |
```

---

## 7. THE FRONTEND ARCHITECT'S PLAYBOOK

This is the mental model layer. The previous sections gave you the mechanics — the dashboards, the commands, the checklists. This section gives you the **thinking frameworks** that separate a senior frontend engineer from a frontend architect.

### 7.1 The Component Library as a Force Multiplier

**The mental model:** A well-built component library doesn't just save time on individual features. It creates a **multiplicative effect** on team velocity. Every component you add to the library is a component that N engineers never have to build again. The library is not a cost center — it's a force multiplier.

```
Without a component library:
  Team of 5 engineers × 10 features = 50 instances of building buttons,
  cards, modals, inputs, and forms from scratch. Each slightly different.

With a component library:
  1 engineer builds Button, Card, Modal, Input, Form (2 weeks)
  → 5 engineers × 10 features = 50 features built with consistent
  components (in half the time, with half the bugs)

The ROI isn't linear. It's multiplicative.
  - First month: investment > return (building the library)
  - Second month: investment = return (library is used, saves time)
  - Months 3-12: investment << return (every new hire is productive faster,
    every new feature is built faster, every design change is applied once)
```

**The architect's role:** You don't build every component. You design the API, set the patterns, review the implementations, and ensure consistency. You're the gardener, not the flower.

**What makes a component library succeed:**
- **Minimal API surface.** Every prop you add is a prop you maintain forever.
- **Composition over configuration.** `<Card><Card.Header>...</Card.Header><Card.Body>...</Card.Body></Card>` beats `<Card title="..." subtitle="..." body="..." footer="..." />`.
- **Design tokens, not hardcoded values.** Colors, spacing, typography — all from tokens.
- **Storybook for every component.** If it doesn't have a story, it doesn't exist.
- **Tested.** Not just "it renders" — tested for accessibility, edge cases, theming.

### 7.2 Performance Budgets as Guardrails

**The mental model:** Performance budgets are not goals. They're **guardrails** — automated constraints that prevent the team from accidentally degrading the user experience. A goal says "we should be fast." A budget says "if the bundle exceeds 500KB, the build fails."

```
Performance Budget Architecture:

  ┌─────────────────────────────────────────────┐
  │  BUDGET                                      │
  │  Bundle: < 500KB (JS)                        │
  │  Cold Start: < 2s (P95)                      │
  │  LCP: < 2.5s (P75)                           │
  │  TTI: < 3.8s (P75)                           │
  │  CLS: < 0.1 (P75)                            │
  │  INP: < 200ms (P75)                          │
  ├─────────────────────────────────────────────┤
  │  ENFORCEMENT                                 │
  │  CI: Bundle size check → fail build if over  │
  │  CI: Lighthouse CI → fail if scores drop     │
  │  Dashboard: Alert if cold start P95 exceeds  │
  │  PR: Bot comments with size diff             │
  └─────────────────────────────────────────────┘
```

**Implementation:**

```bash
# bundlesize in CI
# package.json
{
  "bundlesize": [
    { "path": "dist/main.*.js", "maxSize": "250 kB" },
    { "path": "dist/vendor.*.js", "maxSize": "300 kB" }
  ]
}

# Lighthouse CI
# lighthouserc.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      },
    },
  },
};
```

**The architect's role:** You set the budgets. You wire them into CI. You educate the team on why they exist. And when a budget is exceeded, you help the team fix it — not by removing the budget.

### 7.3 Monitoring as Insurance

**The mental model:** Monitoring is not a feature you build after launch. It's **insurance** you pay for before the incident. The cost of monitoring setup is fixed and small. The cost of a production incident without monitoring is variable and potentially catastrophic.

```
The Monitoring Insurance Matrix:

| Investment          | Cost    | What It Prevents                    | ROI              |
|---------------------|---------|-------------------------------------|------------------|
| Sentry setup        | 2 hours | Blind debugging during crashes      | 100x first time  |
| Crashlytics setup   | 1 hour  | Unknown native crashes in prod      | 100x first time  |
| Performance marks   | 4 hours | Invisible performance degradation   | 10x per quarter  |
| Bundle size CI      | 1 hour  | Gradual bundle bloat                | 5x per month     |
| Error boundary setup| 2 hours | White screens reaching users        | 50x first time   |
| Analytics events    | 8 hours | Building features nobody uses       | 10x per decision |
```

**Error boundaries are your frontend circuit breaker.** Every screen should be wrapped in an error boundary that catches JS errors and shows a fallback UI instead of crashing the entire app.

```typescript
// The minimum viable error boundary for every screen
function ScreenErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary
      fallback={<ScreenErrorFallback />}
      onError={(error, info) => {
        Sentry.captureException(error, { extra: info });
        analytics.track('screen_error', {
          error: error.message,
          componentStack: info.componentStack,
        });
      }}
    >
      {children}
    </ErrorBoundary>
  );
}
```

### 7.4 CI/CD as a Time Machine

**The mental model:** A good CI/CD pipeline doesn't just deploy your code. It's a **time machine** — it lets you go back to any previous state of the application with confidence. Every build is a snapshot. Every deployment is a bookmark. The ability to roll back to a known-good state in under 5 minutes is worth more than the ability to deploy new features faster.

```
The CI/CD Time Machine:

  Past                    Present               Future
  ──────────────────────────────────────────────────────
  v2.3.1 ← v2.3.2 ← v2.4.0 ← v2.4.1 (current)
           ↑                     ↑
           │                     │
     "last known good"     "probably good"
     (use if v2.4.1           (roll back here first)
      also fails)

  OTA: Instant rollback to any previous JS bundle
  Build: 15-min rollback to any previous native build
  Web: 30-sec rollback to any previous Vercel deployment
```

**The architect's role:** Ensure the pipeline is reliable, fast, and reversible. A pipeline that takes 45 minutes to deploy is a pipeline that takes 45 minutes to roll back. That's unacceptable for a production incident.

### 7.5 Design Systems as Team Velocity

**The mental model:** A design system is not a Figma file. It's a **contract** between design and engineering that eliminates an entire category of work: the translation from design intent to implementation detail. When the design system is strong, designers design with components that exist, engineers build with components that are designed, and nobody spends time in a meeting arguing about the border-radius of a card.

```
Without a design system:
  Designer: "Make the button blue, rounded, with a shadow"
  Engineer: "What blue? What radius? What shadow?"
  Designer: "Like the other button"
  Engineer: "Which other button? There are 7 different buttons"
  → 2 hours of Slack messages, 3 design reviews, 1 frustrated PM

With a design system:
  Designer: "Use Button/Primary/Large"
  Engineer: <Button variant="primary" size="lg">Submit</Button>
  → Done. No conversation needed.

  Time saved per component instance: ~30 minutes
  Component instances per sprint: ~50
  Time saved per sprint: ~25 hours
  That's 3 full engineering days. Every sprint. Forever.
```

**The architect's role:** You own the design system as a technical product. You define the token structure, the component API patterns, the theming architecture, and the contribution model. You bridge the gap between the design team's Figma library and the engineering team's component library.

### 7.6 The Mental Model Summary

```
┌─────────────────────────────────────────────────────────────┐
│  THE FRONTEND ARCHITECT'S MENTAL MODELS                      │
│                                                              │
│  Component Library  → Force Multiplier (invest once, N×)     │
│  Performance Budget → Guardrails (automated, enforced)       │
│  Monitoring         → Insurance (cheap upfront, saves later) │
│  CI/CD Pipeline     → Time Machine (reversibility > speed)   │
│  Design System      → Contract (eliminates translation)      │
│  Error Boundaries   → Circuit Breakers (graceful failure)    │
│  Type Safety        → Documentation (self-documenting code)  │
│  Code Review        → Quality Gate (catch > fix)             │
│  Architecture Docs  → Memory (decisions outlive engineers)   │
│  Tooling            → Leverage (your code writes code)       │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. TRIBAL KNOWLEDGE EXTRACTION

Every frontend codebase has an oral history. Decisions that never made it into an ADR. The unwritten rule about not using `useLayoutEffect` because it causes a flash on Android. The reason the navigation structure has that weird nested stack. The engineer who built the custom animation library, left a year ago, and took all the context with them.

This knowledge exists in Slack threads, in old PR descriptions, in commit messages, and in the heads of your teammates. Your job is to extract it before you need it during an incident.

### 8.1 The Three Conversations

In your first two weeks, have these three specific conversations. Not casual chats — deliberate, prepared conversations with specific questions.

**Conversation 1: The Tech Lead / Previous Architect**

*Purpose: Strategic view — the big decisions and their rationale*

```markdown
## Tech Lead Conversation — Frontend Architecture

1. "Can you draw the app's architecture on a whiteboard in 5 minutes?"
   → What they include tells you what matters. What they skip tells you
     what they consider stable (or forgotten).

2. "Why did we choose [state management library] over alternatives?"
   → Reveals the constraints that existed at decision time. "We chose
     Zustand because Redux was too boilerplate-heavy for our team size"
     is context you can't get from the code.

3. "What's the biggest technical risk in the frontend right now?"
   → Might be: bundle size growing, performance regression on Android,
     design system divergence, dependency on a deprecated library.

4. "If you had a free sprint, what would you fix?"
   → Reveals the tech debt they're aware of but can't prioritize.

5. "What should I definitely NOT refactor without talking to someone first?"
   → Reveals the landmines. The component that looks simple but has 50
     edge cases. The navigation structure that breaks if you change it.

6. "Why is [specific architectural choice] the way it is?"
   → Ask about things that surprised you in codebase exploration:
     - "Why do we have two navigation libraries?"
     - "Why is this component server-rendered but this one isn't?"
     - "Why does the auth flow use a custom solution instead of Auth0?"
```

**Conversation 2: The Longest-Tenured Frontend Engineer**

*Purpose: Historical truth — the scars, the regrets, the context*

```markdown
## Veteran Engineer Conversation — Frontend History

1. "What's the oldest component in the codebase, and why hasn't it been rewritten?"
   → Reveals the code that's too critical, too complex, or too scary to touch.

2. "What breaks most often?"
   → The answer is almost never what you'd expect. It might be "the deep link
     handler" or "the image caching layer" or "anything involving the keyboard
     on Android."

3. "Is there a screen or flow that everyone is afraid to modify?"
   → This is the haunted graveyard. You need to know where it is so you don't
     accidentally wander in.

4. "What's a decision you'd make differently if starting the app from scratch?"
   → "I'd use Expo from day one." "I'd invest in a design system before building
     50 screens." "I'd never have used that custom networking layer."

5. "What's the story behind [the thing that surprised you in the code]?"
   → Every weird pattern has a story. The custom bridge module that wraps a
     native SDK. The helper function with 15 parameters. The navigation structure
     that doesn't match the Figma.
```

**Conversation 3: The QA Engineer or Most Active Bug Reporter**

*Purpose: User truth — what actually hurts users*

```markdown
## QA / Bug Reporter Conversation — User Pain

1. "What are the top 3 bugs users report most often?"
   → Not crashes (those are in Crashlytics) — bugs. "The app forgets my
     preferences." "Images don't load sometimes." "The search is slow."

2. "Which screen or flow has the most bugs filed against it?"
   → That's your highest-risk area.

3. "What are the devices / OS versions that cause the most issues?"
   → "Everything works fine on iPhone 15 but the Galaxy A13 is a disaster"
     is information worth gold.

4. "What's the testing gap you wish was covered?"
   → "We don't test offline mode." "We don't test RTL languages." "Nobody
     tests on tablets."

5. "What would make your job easier?"
   → This question surfaces tooling opportunities — Storybook visual testing,
     Detox E2E tests, better error messages, test account management.
```

### 8.2 The Questions Nobody Documents

Beyond the three conversations, here are specific questions to ask whenever you encounter something surprising in the codebase:

**Architecture decisions:**
- "Why Zustand and not Redux / Jotai / MobX?"
- "Why did we build a custom design system instead of using an existing one?"
- "Why is this component class-based when everything else uses hooks?"
- "Why do we have both REST and GraphQL endpoints?"
- "Why does this module have its own bundler configuration?"

**Performance decisions:**
- "Why is this list using FlatList with a custom implementation instead of FlashList?"
- "Why is this screen not using React.memo when it clearly re-renders frequently?"
- "Why are we importing the entire lodash library instead of individual functions?"
- "Why is this image stored locally instead of loaded from a CDN?"

**Platform decisions:**
- "Why do we have platform-specific code (.ios.tsx / .android.tsx) for this screen?"
- "Why are we using a native module for this when a JS solution exists?"
- "Why does the Android build have a different feature set than iOS?"
- "Why do we support iOS 15 when only 3% of users are on it?"

**Infrastructure decisions:**
- "Why is the EAS build profile configured this way?"
- "Why do we have separate staging and preview environments?"
- "Why is the OTA update channel separate from the build channel?"
- "Why did we choose Sentry over Crashlytics (or vice versa)?"

**Document every answer.** The person who told you might leave tomorrow. The decision rationale needs to live in the codebase, not in someone's head.

### 8.3 Capture Format: The Architecture Decision Record

When you uncover a tribal knowledge item that's actually an architecture decision, write it down as an ADR:

```markdown
# ADR-007: Why We Use Zustand Over Redux

## Status: Accepted (retroactive documentation)
## Date: 2026-04-07 (documenting existing decision made ~2024)
## Context
The app was originally built with Redux Toolkit. As the team grew from 2 to 8
frontend engineers, we found that:
- Redux boilerplate slowed feature development
- New engineers needed 2+ weeks to learn the Redux patterns
- Server state management was awkward in Redux (before RTK Query matured)

## Decision
Migrate from Redux to Zustand for client state, and adopt TanStack Query
for server state. The migration was done incrementally over 3 sprints.

## Consequences
- Positive: Onboarding time for new engineers dropped from 2 weeks to 3 days
- Positive: Client state code reduced by ~60% (LOC)
- Positive: Server state caching is now automatic via TanStack Query
- Negative: Some older screens still have Redux connected components (migration incomplete)
- Negative: Team needs to maintain knowledge of both libraries during transition

## Related
- Migration tracker: JIRA-4521
- Original discussion: Slack #frontend-architecture, 2024-06-15
```

---

## 9. ELEVATING YOUR TEAM

Here's the uncomfortable truth about being a frontend architect: **your individual output doesn't matter as much as your team's collective output.** The best frontend architect isn't the one who writes the most code. It's the one who makes everyone else write better code, faster.

### 9.1 Building Tooling That Makes Others Faster

**Code generators:**

```bash
# Instead of every engineer creating components from scratch:
npx turbo gen component --name ProductCard

# This generates:
# packages/ui/src/ProductCard/
#   ProductCard.tsx          ← Component skeleton with proper types
#   ProductCard.stories.tsx  ← Storybook story
#   ProductCard.test.tsx     ← Test file with basic rendering test
#   index.ts                 ← Export barrel
```

Write the generator once, and every engineer on the team creates consistent, properly-structured components without thinking about it.

**Custom ESLint rules:**

```typescript
// Instead of writing "don't use inline styles" in a Confluence doc
// that nobody reads, write a lint rule that enforces it:
module.exports = {
  rules: {
    'no-inline-styles': {
      create(context) {
        return {
          JSXAttribute(node) {
            if (node.name.name === 'style' &&
                node.value.type === 'JSXExpressionContainer') {
              context.report({
                node,
                message: 'Use design tokens from the theme instead of inline styles. ' +
                         'See: packages/ui/CONTRIBUTING.md#styling',
              });
            }
          },
        };
      },
    },
  },
};
```

**Pre-commit hooks:**

```bash
# .husky/pre-commit
npx lint-staged

# lint-staged.config.js
module.exports = {
  '*.{ts,tsx}': ['eslint --fix', 'prettier --write'],
  '*.{json,md}': ['prettier --write'],
  'packages/ui/**/*.tsx': ['npm run test:ui -- --bail'],
};
```

**The tooling philosophy:** Every manual process that engineers repeat is a candidate for automation. Every review comment you make more than three times is a candidate for a lint rule. Every pattern you explain more than twice is a candidate for a code generator or template.

### 9.2 Creating Shared Patterns

**Shared patterns > documentation.** A pattern is a living, executable example. Documentation is a promise that may or may not be kept.

```
Pattern Library (create in your monorepo under docs/patterns/):

patterns/
  data-fetching.md        ← How to fetch data with TanStack Query
  form-handling.md        ← How to build forms with React Hook Form + Zod
  error-handling.md       ← Error boundaries, try/catch, fallback UI
  navigation.md           ← Adding a new screen, deep linking
  animation.md            ← Reanimated worklets, shared transitions
  testing.md              ← What to test, how to test, test IDs
  performance.md          ← Memoization, lazy loading, image optimization
  accessibility.md        ← Required a11y props, testing with screen readers
  new-feature-checklist.md ← The checklist for shipping any new feature
```

Each pattern file should contain:
1. **When to use this pattern** (and when not to)
2. **The pattern** (copy-paste code that works)
3. **Common mistakes** (what goes wrong when you deviate)
4. **Real examples** (links to PRs where this pattern was used correctly)

### 9.3 Writing Decision Documents

When you face an architectural decision, write it down *before* you decide. This seems slow. It's actually faster, because:

1. Writing forces clarity. Half the time, the answer becomes obvious while writing.
2. Async review. The team can review on their schedule instead of in a meeting.
3. Permanent record. Six months later, "why did we do this?" has an answer.

**Decision Document Template:**

```markdown
# [DECISION] Title of the Decision

## Status: Proposed / Accepted / Deprecated
## Author: [name]
## Date: [date]
## Reviewers: [names]

## Problem
What problem are we solving? Be specific. Include data if available.
"Users report the product list screen is slow on Android. P95 render time
is 3.2 seconds. The component re-renders 47 times per list scroll."

## Options Considered

### Option A: [Name]
- Description
- Pros: ...
- Cons: ...
- Effort: S / M / L
- Risk: Low / Medium / High

### Option B: [Name]
- Description
- Pros: ...
- Cons: ...
- Effort: S / M / L
- Risk: Low / Medium / High

### Option C: [Name] (recommended)
- Description
- Pros: ...
- Cons: ...
- Effort: S / M / L
- Risk: Low / Medium / High

## Recommendation
Option C because [reasons]. The main tradeoff is [X] but we accept this
because [Y].

## Implementation Plan
1. [Step 1] — [owner] — [timeline]
2. [Step 2] — [owner] — [timeline]
3. [Step 3] — [owner] — [timeline]

## Success Criteria
How will we know this worked?
- P95 render time drops to < 500ms
- Re-render count during scroll drops to < 5
- No regression in crash-free rate
```

### 9.4 Running Architecture Reviews

**The Architecture Review Meeting:**

Hold this bi-weekly. One hour. Structured.

```
Architecture Review Agenda:
  1. [10 min] Metrics review — dashboard walkthrough
     - Crash-free rate trend
     - Bundle size trend
     - Build time trend
     - Any new performance regressions

  2. [15 min] Design system sync
     - New components in the pipeline
     - Design-to-code gaps identified
     - Upcoming design changes that affect architecture

  3. [20 min] Decision review
     - Open decisions that need resolution
     - Completed decisions that need a checkpoint
     - New decisions that need to be written up

  4. [10 min] Tech debt triage
     - Top 3 tech debt items by impact
     - Can we dedicate time this sprint?
     - Any debt that's becoming urgent?

  5. [5 min] Open floor
     - Questions, concerns, ideas
```

**The architect's role in the review is not to decide everything.** It's to:
- Ensure decisions are written down
- Ensure tradeoffs are explicit
- Ensure the team is aligned on direction
- Ensure metrics are trending the right way
- Flag risks before they become incidents

### 9.5 Onboarding as Architecture

**The mental model:** How you onboard new engineers IS your architecture. A codebase that takes 2 days to set up locally is telling you something about its complexity. A codebase where new engineers can ship their first PR on day 2 is telling you something about its design.

**The architect's onboarding investment:**

```
The Onboarding Multiplier:

  Time to first PR (current):  5 days
  Engineers hired per year:     6
  Total onboarding cost:        30 engineer-days

  Investment: 2 weeks of architect time improving onboarding
  Time to first PR (after):     2 days
  Total onboarding cost:        12 engineer-days

  Net savings: 18 engineer-days per year
  Plus: better first impression, higher retention, faster ramp
```

**Concrete onboarding improvements an architect can make:**

1. **One-command setup.** If getting the app running takes more than one command after cloning, fix it. Write a setup script.

```bash
#!/bin/bash
# scripts/setup.sh — one command to rule them all
set -e

echo "Installing dependencies..."
pnpm install

echo "Setting up environment..."
cp .env.example .env.local
echo "→ Edit .env.local with your API keys (see docs/SETUP.md)"

echo "Installing iOS pods..."
cd apps/mobile/ios && pod install && cd ../../..

echo "Checking EAS login..."
eas whoami || echo "→ Run 'eas login' to authenticate with Expo"

echo "Setup complete! Run 'pnpm dev:mobile' to start the mobile app."
```

2. **First-PR template.** Create a curated list of "good first issues" that let new engineers ship quickly without needing deep context. Typo fixes, adding test coverage, small UI polish, documentation updates.

3. **Architecture walkthrough video.** Record a 15-minute screen share walking through the codebase structure, the key files, and the development workflow. Update it every quarter.

4. **Pair programming slots.** Block 30 minutes on your calendar twice a week for "pair with the architect." New engineers book these slots when they're stuck.

### 9.6 The Architect's Multiplier Effect

```
Individual contributor:
  Your code × 1 = your output

Senior engineer:
  Your code + your code reviews + your mentoring × 1 = team output

Architect:
  (Team code quality × team velocity × system reliability)
  ÷ incidents prevented
  = organizational output

The shift:
  IC thinks: "How do I build this feature?"
  Senior thinks: "How do I build this feature well?"
  Architect thinks: "How do I make the team build every feature well?"
```

This is the fundamental mindset shift. Your job is not to write the best component. Your job is to create the conditions where every engineer writes good components. The component library. The patterns. The tooling. The reviews. The budgets. The monitoring. The documentation.

**You succeed when the team succeeds without needing you for every decision.**

---

## 10. THE BEAST MODE CHECKLIST

Everything in sections 1 through 9, distilled into a single checklist. Print this out. Tape it to your monitor. Execute it in your first two weeks. Not everything will be possible on day one — some items (like running an architecture review) require a few weeks of context. But every box should be checked by the end of your first month.

### Priority Triage (Day 1 — First 2 Hours)

- [ ] App running locally (simulator/device or `localhost:3000`)
- [ ] CI/CD pipeline located and last 5 builds reviewed
- [ ] Primary monitoring dashboard bookmarked (Sentry, Crashlytics, Vercel Analytics)
- [ ] Most recent incident postmortem or bug report read
- [ ] Go-to person identified and messaged
- [ ] "I don't understand this yet" list started (3+ items)

### Access Matrix (Days 1-3)

- [ ] Source code: repo cloned, can push to a branch
- [ ] CI/CD: GitHub Actions / EAS / Vercel — can view builds and deployments
- [ ] Expo Dashboard (EAS): can view builds, updates, and credentials
- [ ] Vercel Dashboard: can view deployments, analytics, and logs
- [ ] Error tracking (Sentry): can view issues, filter by release and platform
- [ ] Crash reporting (Crashlytics): can view crash reports and ANR data
- [ ] Firebase Console: can view project settings and analytics
- [ ] App Store Connect: can view app status, crash reports, TestFlight
- [ ] Google Play Console: can view app status, Android Vitals, releases
- [ ] Figma: can view and comment on design files
- [ ] Storybook: can access and browse the component library
- [ ] Analytics (Amplitude/Mixpanel/PostHog): can view dashboards and events
- [ ] Feature flags: can view flag status and targeting rules
- [ ] Slack channels: all relevant channels joined
- [ ] Documentation: Notion/Confluence access confirmed

### Codebase Navigation (Days 2-5)

- [ ] Monorepo structure mapped (apps, packages, build system)
- [ ] Entry points identified (root layout, navigation structure, screens)
- [ ] State architecture understood (server state, client state, form state, persistence)
- [ ] CI/CD pipeline read and understood (build, test, deploy, rollback)
- [ ] One user action traced end-to-end (tap → handler → API → response → UI update)
- [ ] Design system / component library explored (Storybook, packages/ui)
- [ ] Load-bearing files identified (most changed, most imported, most authors)
- [ ] Legacy code boundaries noted (old patterns vs. new patterns)
- [ ] Package.json scripts understood (dev, build, test, lint, typecheck)

### Frontend Health Dashboard (Days 3-7)

- [ ] Crash-free rate baseline captured (iOS and Android)
- [ ] ANR rate baseline captured (Android)
- [ ] Cold start P50/P95 baseline captured (or measurement instrumented)
- [ ] Bundle size baseline captured
- [ ] Build time baseline captured
- [ ] Error rate baseline captured
- [ ] Core Web Vitals baseline captured (web, if applicable)
- [ ] All baselines written down and taped to monitor
- [ ] Personal dashboard/hotlinks page created and bookmarked

### Incident Readiness (Days 5-10)

- [ ] First 5 minutes script written (the exact steps for when something crashes)
- [ ] Crash report reading practiced (Sentry + Crashlytics — can symbolicate)
- [ ] OTA update process understood and tested on staging
- [ ] App Store / Play Store rollback process understood
- [ ] Rollback procedure documented with exact commands
- [ ] Incident communication template saved
- [ ] "What changed?" reflex practiced (deploy history → config → traffic → dependencies)

### Architecture & Mental Models (Days 5-14)

- [ ] Architecture sketch drawn (navigation tree, data flow, API layer)
- [ ] Frontend Architect's Playbook mental models internalized:
  - [ ] Component Library = Force Multiplier
  - [ ] Performance Budgets = Guardrails
  - [ ] Monitoring = Insurance
  - [ ] CI/CD = Time Machine
  - [ ] Design System = Contract
- [ ] Performance budget defined (or existing budget verified)
- [ ] Error boundary strategy verified (every screen wrapped?)
- [ ] Bundle splitting strategy understood (code splitting, lazy routes)

### Tribal Knowledge (Days 7-14)

- [ ] Conversation 1 completed: Tech Lead / Previous Architect
- [ ] Conversation 2 completed: Longest-Tenured Frontend Engineer
- [ ] Conversation 3 completed: QA Engineer / Bug Reporter
- [ ] Key architecture decisions documented as ADRs
- [ ] "Known unknowns" list collected from team conversations
- [ ] At least one documentation gap fixed (README, setup guide, pattern doc)
- [ ] Newcomer FAQ started (questions I had, answers I found)

### Team Elevation (Days 14-30)

- [ ] One shared pattern documented and shared with team
- [ ] One tooling improvement identified (code generator, lint rule, script)
- [ ] One decision document written for an upcoming architectural choice
- [ ] Architecture review meeting proposed or attended
- [ ] Component library contribution made (new component or improvement)
- [ ] Build/deploy improvement identified (faster builds, better caching, cleaner pipeline)

---

## CLOSING: IT'S YOUR FIRST WEEK AND THE APP IS CRASHING

It's Wednesday afternoon. Day three at the new company.

You're sitting at your desk, earbuds in, tracing through the checkout flow in the codebase. Your Beast Mode checklist is open in a Notion doc on your second monitor. Most of the "Days 1-3" boxes are checked. You got the app running on your simulator Monday afternoon — took 45 minutes because the README didn't mention you need to run `eas login` first (you already submitted a PR to fix that). You bookmarked the Sentry project, the EAS dashboard, and the Vercel deployment page yesterday morning. You captured your baselines: 99.6% crash-free rate on iOS, 99.3% on Android, 1.8MB JS bundle, 14-minute EAS build time.

You're reading through the navigation structure — it uses Expo Router with a nested `(tabs)` group — when your phone buzzes with a Slack notification.

```
#alerts
🔴 Sentry: New issue with 847 events in the last 5 minutes
   TypeError: Cannot read properties of undefined (reading 'price')
   ProductDetailScreen.tsx:142
   Release: 2.4.1 (1024)
```

Your stomach drops. Three days in. You barely know anyone's name.

But you open the incident channel anyway. Nobody has responded yet. It's 3:15 PM on a Wednesday and the team is in sprint planning.

You remember section 6. The first five minutes matter more than the next five hours. You remember the script you wrote for yourself yesterday afternoon.

**Minute 0-1: Open the error tracker.**

You click the Sentry bookmark you set up on Tuesday. Three clicks and you're on the issue page. You see it immediately:

- **TypeError: Cannot read properties of undefined (reading 'price')**
- **First seen:** 3:10 PM — five minutes ago
- **Events:** 847 and climbing
- **Users affected:** 312
- **Release:** 2.4.1 (1024)
- **Platform:** iOS and Android — both affected

The breadcrumbs show: users navigate to a product, the product data loads, the component tries to read `product.price` but `product` is undefined.

**Minute 1-2: What changed?**

You check the EAS dashboard — it's bookmarked. No new builds today. But there's an OTA update:

```
EAS Updates — production branch
  Update: "Add price formatting for new currencies"
  Published: 3:05 PM by @alex
  Runtime version: 2.4.1
```

Five minutes before the first crash. An OTA update to the production branch.

**Minute 2-3: Read the update diff.**

You check the GitHub commit linked to the update:

```typescript
// Before (working):
const formattedPrice = formatPrice(product.price, product.currency);

// After (broken):
const formattedPrice = formatPrice(product.pricing.amount, product.pricing.currency);
```

The update changed the property access from `product.price` to `product.pricing.amount`. But the API still returns `product.price` — the backend migration for the new pricing structure hasn't shipped yet. The frontend update was deployed before the backend was ready.

**Minute 3-4: Communicate.**

You switch to the incident channel:

```
Joining. I'm [your name], new frontend engineer.

Issue: TypeError at ProductDetailScreen.tsx:142 — 'Cannot read
properties of undefined (reading "price")'

847 events, 312 users affected, both platforms. Started at 3:10 PM.

Cause: OTA update published at 3:05 PM by @alex changed product.price
to product.pricing.amount, but the backend API still returns the old
schema.

This is a JS-only change, so OTA rollback should fix it immediately.

Should I push an update reverting to the previous JS bundle?
```

Fifteen seconds later:

```
@alex: Oh no, I thought the backend migration was already live.
Yes, please roll back the OTA update.
```

**Minute 4-5: Roll back.**

You practiced this yesterday. You know the command.

```bash
# Push the previous bundle to the production branch
eas update --branch production \
  --message "Rollback: revert pricing format change (API not ready)" \
  --platform all
```

You watch the Sentry dashboard. New events slow from a flood to a trickle. 847... 852... 853... 854... stopped. The new bundle is propagating. Users who restart the app get the fixed version.

```
#incidents
@you: OTA rollback published at 3:20 PM. Sentry events have stopped
climbing. Monitoring for the next 15 minutes.

Timeline:
- 3:05 PM: OTA update "price formatting" pushed by @alex
- 3:10 PM: First crash reported in Sentry
- 3:15 PM: Alert fired, 847 events
- 3:20 PM: Rollback OTA update pushed
- 3:25 PM: New events stopped, crash rate stabilizing

Action items:
1. Coordinate frontend/backend deploy order for pricing migration
2. Add API response validation to prevent this class of error
3. Consider adding error boundary around ProductDetailScreen
```

The tech lead, who joined silently:

```
@sarah: Really solid response for day 3. Clear diagnosis, clean rollback,
good action items. Exactly how it should work.
```

---

Think about what you had that a typical day-three engineer wouldn't:

- **A bookmarked Sentry project** so you didn't waste time searching for it
- **Baseline numbers** written down, so you could immediately see the crash-free rate was abnormal
- **An EAS dashboard bookmark** that got you to the recent updates in seconds
- **Knowledge of the OTA update process** so you could roll back in under a minute
- **An incident communication template** so your message was structured, data-rich, and actionable — not "something seems broken"
- **A PR fixing the README** already submitted — earning trust before the incident even happened

None of this required deep system knowledge. You didn't architect the pricing system. You didn't write the product detail screen. You didn't know the history of why the pricing migration was planned this way. Alex knew that. Sarah knew that.

What *you* knew was where to look, what to check first, and how to act.

---

You didn't know everything. You didn't need to.

You knew where to look. You knew what to check first. You knew how to communicate what you found. And you knew how to fix it.

That's Beast Mode.

It's not about being the smartest person in the room. It's not about having three years of context on the codebase. It's about doing the work *before* the moment arrives, so that when the moment arrives — and it always does — you're ready.

The engineers who do this don't just survive their first month. They change the trajectory of their team. They raise the bar for what "operational readiness" means. They prove that the frontend architect role isn't about gatekeeping — it's about enabling. Enabling the team to build faster, ship safer, and recover quicker.

Your first week isn't a grace period. It's a launchpad.

> **"The best frontend architects I've worked with weren't the ones who knew the most about React on day one. They were the ones who built the scaffolding to be dangerous the fastest. They bookmarked the dashboards, captured the baselines, traced the critical paths, and learned the rollback procedures — all before their first incident. When the crash came, they didn't panic. They executed. That's not experience. That's preparation. And preparation is a choice you make on day one."**
>
> — The Beast Mode philosophy

---

## TRY IT YOURSELF

1. **Build your personal access matrix for your current project** — list every system in the table from Section 3, verify your access, and note the gaps. Share it with your team lead.

2. **Capture your baselines right now** — open every monitoring tool you have access to and fill in the baseline table from Section 5. Write the numbers on a sticky note and put it on your monitor.

3. **Trace one critical user action end-to-end** — pick the most important flow in your app (login, purchase, search) and trace it from user tap to API call to response to screen update. Document it in the format from Section 4.

4. **Practice your OTA rollback** — push a trivial OTA update to your staging branch, then roll it back. Do this when nothing is on fire, so you're not learning the process during an incident.

5. **Have one tribal knowledge conversation this week** — pick one of the three conversations from Section 8 and have it. Document what you learn as an ADR or a FAQ entry.

6. **Build one piece of team tooling** — a code generator, a lint rule, a CI check, a dashboard. Something that makes the whole team faster, not just you.

The checklist is your compass. The conversations are your map. The dashboards are your radar. The rollback procedure is your ejection seat.

You have everything you need. Go be dangerous.
