<!--
  CHAPTER: 19
  TITLE: Frontend Architecture Patterns
  PART: IV — Architecture at Scale
  PREREQS: Chapters 4, 15
  KEY_TOPICS: monoliths, microfrontends, module federation, BFF, island architecture, feature-based structure, architectural linting, ESLint boundaries, dependency-cruiser, runtime vs build-time composition
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 19: Frontend Architecture Patterns

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 4, 15 | Difficulty: Advanced

Here's the uncomfortable truth about frontend architecture: **there is no correct architecture.** There is only the architecture that your team can ship with, maintain without losing their minds, and evolve as requirements change. And that architecture is almost certainly not the one that looks coolest in a conference talk.

I've watched teams spend six months migrating to microfrontends because a VP saw a talk from Spotify, only to discover that their three-person team now had the operational overhead of a platform engineering org. I've also watched teams cling to a monolith until their 200-person engineering org couldn't ship a button change without coordinating across four teams and waiting three hours for CI.

Both paths lead to pain. The difference between a 1x architect and a 100x architect is knowing *when* each approach starts making sense — and more importantly, *when it doesn't.*

This chapter is not going to tell you "use microfrontends" or "stay monolith." It's going to give you the decision framework, the trade-offs, the implementation details, and the escape hatches so you can make the right call for your situation, defend that call with evidence, and change course when the situation changes.

### In This Chapter
- The Architecture Decision Space (it's not a spectrum, it's a matrix)
- The Frontend Monolith — the right default and its limits
- Microfrontends — runtime and build-time composition
- Module Federation with Rsbuild/Rspack
- Build-Time Composition in Monorepos
- Island Architecture and Partial Hydration
- Backend for Frontend (BFF) Pattern
- Feature-Based File Structure
- Architectural Linting (ESLint boundaries, dependency-cruiser)
- The Frontend Architect's Playbook

### Related Chapters
- [Ch 4: TypeScript for Architects] — type-safe module boundaries
- [Ch 15: Build Systems & Monorepos] — the tooling foundation for these patterns
- [Ch 16: CI/CD Pipelines] — deployment strategies that enable (or constrain) architecture choices
- [Ch 20: Design Systems at Scale] — component architecture across teams

---

## 1. THE ARCHITECTURE DECISION SPACE

Most architecture discussions start wrong. Someone asks "should we use microfrontends?" as if it's a yes/no question. It isn't. Frontend architecture exists along *multiple independent axes*, and the combinations matter more than any single choice.

Here are the axes:

```
┌─────────────────────────────────────────────────────────────────────┐
│                  THE ARCHITECTURE DECISION MATRIX                    │
│                                                                      │
│  AXIS 1: REPO TOPOLOGY                                              │
│  ├── Single repo (monolith)                                         │
│  ├── Monorepo (multiple packages, one repo)                         │
│  └── Polyrepo (multiple repos)                                      │
│                                                                      │
│  AXIS 2: BUILD STRATEGY                                             │
│  ├── Single build (one artifact)                                    │
│  ├── Coordinated builds (multiple artifacts, one pipeline)          │
│  └── Independent builds (each app builds on its own)                │
│                                                                      │
│  AXIS 3: DEPLOYMENT STRATEGY                                        │
│  ├── Lockstep deployment (everything ships together)                │
│  ├── Coordinated deployment (deploy in a specific order)            │
│  └── Independent deployment (each piece deploys on its own)         │
│                                                                      │
│  AXIS 4: RUNTIME COMPOSITION                                        │
│  ├── None (single SPA)                                              │
│  ├── Route-level (different apps at different routes)               │
│  ├── Component-level (remote components loaded into host)           │
│  └── Island-level (interactive islands in static HTML)              │
│                                                                      │
│  AXIS 5: SHARED CODE STRATEGY                                       │
│  ├── Shared packages in monorepo                                    │
│  ├── Published npm packages                                         │
│  ├── Runtime shared dependencies (Module Federation)                │
│  └── Copy-paste (don't laugh — sometimes it's right)               │
│                                                                      │
│  AXIS 6: TEAM TOPOLOGY                                              │
│  ├── Single team owns everything                                    │
│  ├── Feature teams with shared ownership                            │
│  ├── Autonomous teams with clear boundaries                         │
│  └── Separate orgs or vendors                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Combinations That Actually Exist in the Wild

Here's the thing: most teams don't live at the extremes. They live in hybrid configurations that combine elements from different axes. Let me show you the most common real-world setups:

**Configuration A: "The Sensible Default"**
```
Repo:        Monorepo (Turborepo or Nx)
Build:       Coordinated builds, shared cache
Deploy:      Lockstep (or near-lockstep via affected-only deploys)
Runtime:     Single SPA (Next.js / Remix / React Router)
Shared:      Monorepo packages (@company/ui, @company/utils)
Team:        1-5 teams, some shared ownership
```
This is where 80% of companies should be. It scales to surprisingly large teams — Vercel, Shopify, and many mid-size companies run variations of this for years before needing more.

**Configuration B: "The Micro-Monorepo"**
```
Repo:        Monorepo
Build:       Independent builds per app
Deploy:      Independent per app
Runtime:     Route-level composition (reverse proxy / CDN routing)
Shared:      Monorepo packages, published internally
Team:        5-15 teams, clear ownership boundaries
```
This is the step between monolith and full microfrontends. You get independent deployment without the full complexity of runtime composition. The monorepo gives you the ability to make cross-cutting changes with a single PR.

**Configuration C: "Full Microfrontends"**
```
Repo:        Polyrepo (or monorepo, but independently buildable)
Build:       Fully independent
Deploy:      Fully independent
Runtime:     Component-level composition (Module Federation)
Shared:      Runtime shared deps + published packages
Team:        10+ teams, autonomous, possibly different tech stacks
```
This is the big-company setup. Amazon, IKEA, Zalando. The cost is real: you need a platform team, a shell app, shared contracts, version management, and a much higher baseline of operational maturity. If you're not at 50+ frontend engineers, you're probably not here.

**Configuration D: "The Content-First Island"**
```
Repo:        Monorepo (or single repo)
Build:       Single build (Astro, Eleventy)
Deploy:      Single deployment
Runtime:     Island-level (interactive components in static HTML)
Shared:      Monorepo packages or local components
Team:        1-3 teams, content + engineering split
```
Perfect for marketing sites, documentation, e-commerce storefronts. Most of the page is static. Interactive parts (search, cart, configurators) hydrate independently.

### The Key Insight: Architecture Follows Team Topology

Conway's Law is real and it's not optional. Your architecture will eventually mirror your team structure — or your team structure will bend to match your architecture. The 100x architect designs with this in mind.

> **Steve Kinney puts it perfectly:** "The architecture you choose is a bet on your organization's future. Autonomy increases complexity. Every time you give a team the ability to deploy independently, you're also giving them the ability to break things independently. That's a trade-off, not a free lunch."

The question isn't "what's the best architecture?" The question is "what's the architecture that matches how my teams need to work, with the least accidental complexity?"

---

## 2. THE FRONTEND MONOLITH

Let me be clear: **the monolith is the right default for most teams.** And I don't mean "for small teams" — I mean for most teams, period. If you have fewer than 50 frontend engineers, you should have a very compelling reason to move beyond a modular monolith.

### Why the Monolith Gets a Bad Rap

The word "monolith" carries baggage. Backend engineers hear it and think "a 500K-line Java application with circular dependencies and 45-minute builds." But a frontend monolith in 2026 — a well-structured Next.js or React Router app in a monorepo with shared packages — is a fundamentally different beast.

**What a good frontend monolith looks like:**
```
apps/
  web/                          # The monolith
    src/
      features/                 # Feature-based structure (Section 8)
        auth/
        dashboard/
        settings/
        billing/
      shared/
        ui/                     # App-specific shared components
        hooks/
        utils/
packages/
  @company/ui/                  # Design system
  @company/api-client/          # Generated API client
  @company/config/              # Shared configs (ESLint, TSConfig)
```

One build. One deployment. One bundle. Shared types across every feature. Refactoring works with Find & Replace. Your IDE understands the entire application graph.

### The Monolith's Superpowers

**1. Atomic Changes**

In a monolith, a single PR can change the API response type, update the data fetching hook, modify the component that displays the data, and adjust the test — all atomically. In a microfrontend world, that same change might touch three repos, require coordinated merges, and need a specific deployment order.

```typescript
// One PR, one review, one merge, one deploy.
// Try doing this across three microfrontend repos.

// 1. Update the shared type
interface UserProfile {
  name: string;
  email: string;
  avatarUrl: string;      // new field
  avatarBlurHash: string;  // new field
}

// 2. Update the API client
const fetchUser = async (id: string): Promise<UserProfile> => {
  const res = await api.get(`/users/${id}`);
  return userProfileSchema.parse(res.data);
};

// 3. Update the component
function UserAvatar({ userId }: { userId: string }) {
  const { data: user } = useUser(userId);
  return (
    <Avatar
      src={user.avatarUrl}
      blurHash={user.avatarBlurHash}
      alt={user.name}
    />
  );
}
```

**2. Type Safety Across the Entire App**

TypeScript's power scales with scope. In a monolith, a type change in one feature ripples through the entire application at compile time. You know immediately if something breaks. In a microfrontend world, type contracts exist at boundaries and they can drift.

**3. Performance Optimization Is Straightforward**

Bundle splitting, tree-shaking, shared chunks — the bundler sees the entire application graph and can optimize globally. In a microfrontend world, each app bundles its own dependencies, and shared dependency management becomes a manual coordination problem.

**4. Developer Experience**

One `npm install`. One `npm run dev`. Hot reload works everywhere. You can cmd-click through any import in the entire app. Tests run against the real code, not mocked interfaces.

### When the Monolith Starts Hurting

The monolith has real limits, and you'll feel them before you hit them. Here are the signals:

**Signal 1: Team Collision**

When two teams need to change the same file, you have a problem. Not because of git merge conflicts (those are just annoying) — but because of implicit coupling. Team A changes a shared utility and breaks Team B's feature. Team B's tests fail on Team A's PR. Now both teams are blocked.

```
Merge conflict frequency:     Low → High  (🔴 when > 5 conflicts/week)
PR review wait time:          < 1 day → > 3 days (🔴)
"Who owns this?" questions:   Rare → Daily (🔴)
Cross-team PR dependencies:   Rare → Most PRs (🔴)
```

**Signal 2: CI/CD Bottleneck**

The entire app builds and tests on every PR. At 50K lines, this is 2 minutes. At 500K lines, this is 20 minutes. Engineers start batching changes to avoid waiting, which makes PRs larger, which makes reviews slower, which makes merges riskier.

```
CI time per PR:         < 5 min → > 15 min (🔴)
Flaky test frequency:   < 1% → > 5% (🔴)
Deploy frequency:       Multiple/day → Weekly (🔴)
Rollback scope:         "just my feature" → "everything" (🔴)
```

**Signal 3: Deploy Coordination**

Team A's feature is ready but Team B's half-finished work is on main. Team A waits. Or Team A uses a feature flag. Or Team A branches off main and cherry-picks. All of these are workarounds for a fundamental problem: **lockstep deployment forces lockstep development.**

**Signal 4: Onboarding Overhead**

New engineers need to understand the entire application to work on any part of it. The mental model is too large. They're afraid to change anything because they don't know what might break.

### The Shopify Modular Monolith Lesson

Shopify is the canonical example of doing the monolith *right* at massive scale. Their web platform started as a Rails monolith and they applied the same thinking to their frontend: rather than splitting into microfrontends, they invested in **making the monolith modular.**

The key ideas:
- **Strong module boundaries within the monolith.** Each domain (orders, products, customers, analytics) is a self-contained module with explicit public APIs.
- **Automated boundary enforcement.** Linting rules that prevent imports across module boundaries (we'll cover this in Section 9).
- **Eventual extraction path.** Each module is structured so it *could* be extracted into an independent service, but it doesn't have to be.

```
# Shopify-style modular monolith boundaries
src/
  modules/
    orders/
      public/              # The module's public API
        index.ts           # Only these exports can be imported by other modules
        types.ts
      internal/            # Hidden from other modules
        components/
        hooks/
        utils/
      orders.module.ts     # Module definition (routes, providers)
    products/
      public/
        index.ts
      internal/
        ...
    shared/                # Genuinely shared code (design system, auth, etc.)
      ...
```

**The lesson:** You can get 80% of microfrontends' benefits (clear ownership, independent development, reduced cognitive load) with 20% of the complexity by investing in module boundaries within a monolith.

### The Monolith Decision Checklist

Before you leave the monolith, ask these questions:

| Question | If Yes | If No |
|----------|--------|-------|
| Do you have > 50 frontend engineers? | Consider splitting | Stay monolith |
| Do teams need different tech stacks? | Consider microfrontends | Stay monolith |
| Is CI > 15 minutes even with caching? | Optimize first, then consider splitting | Stay monolith |
| Do teams need independent deploy cadences? | Consider route-level splitting | Stay monolith |
| Are merge conflicts blocking teams daily? | Enforce module boundaries first | Stay monolith |
| Are you migrating from legacy tech? | Strangler fig pattern (Section 3) | Stay monolith |
| Did a VP see a conference talk? | Run. | Stay monolith |

---

## 3. MICROFRONTENDS

Microfrontends are not "microservices for the frontend." They're a different beast with different trade-offs, and treating them like backend microservices is how teams get into trouble.

A microfrontend is an independently deliverable frontend application that composes with other frontend applications to form a complete user experience. The key word is *independently* — independent repo (optionally), independent build, independent deployment, independent team.

### The Two Composition Models

There are fundamentally two ways to compose microfrontends:

```
┌──────────────────────────────────────────────────────────────┐
│                 COMPOSITION MODELS                            │
│                                                               │
│  BUILD-TIME COMPOSITION                                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │ Package  │  │ Package │  │ Package │                      │
│  │    A     │  │    B    │  │    C    │                      │
│  └────┬─────┘  └────┬────┘  └────┬────┘                     │
│       │             │            │                            │
│       └──────┬──────┘────────────┘                           │
│              ▼                                                │
│  ┌───────────────────┐                                       │
│  │   Single Bundle   │  ← One build, one deploy              │
│  │   (Host App)      │                                       │
│  └───────────────────┘                                       │
│                                                               │
│  Pros: type-safe, optimized bundle, simple deployment        │
│  Cons: no independent deployment, all teams on same cadence  │
│                                                               │
│  ─────────────────────────────────────────────────────────── │
│                                                               │
│  RUNTIME COMPOSITION                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │ Remote  │  │ Remote  │  │ Remote  │                      │
│  │   A     │  │    B    │  │    C    │                      │
│  │ (CDN)   │  │ (CDN)   │  │ (CDN)   │                      │
│  └────┬─────┘  └────┬────┘  └────┬────┘                     │
│       │             │            │                            │
│       └──────┬──────┘────────────┘                           │
│              ▼  (loaded at runtime)                           │
│  ┌───────────────────┐                                       │
│  │    Host Shell     │  ← Loads remotes dynamically          │
│  │   (Thin App)      │                                       │
│  └───────────────────┘                                       │
│                                                               │
│  Pros: independent deployment, team autonomy, tech agnostic  │
│  Cons: runtime overhead, version drift, harder debugging     │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### When Microfrontends Genuinely Make Sense

I'll be specific. Microfrontends are the right call in these situations:

**1. Multiple Autonomous Teams (10+ frontend engineers, 3+ teams)**

Each team owns a business domain end-to-end. The orders team shouldn't need to coordinate with the analytics team to ship a change. They have different roadmaps, different priorities, different deploy cadences.

**2. Legacy Migration (The Strangler Fig)**

You have a jQuery/Angular.js/Backbone application and you can't rewrite it all at once. You wrap the legacy app in a shell, and new features are built as microfrontends that gradually replace the old code.

```
Year 1:
┌──────────────────────────────────┐
│        Legacy App (Angular.js)    │
│  ┌──────────┐                    │
│  │ New React │  ← First MFE      │
│  │ Dashboard │                    │
│  └──────────┘                    │
└──────────────────────────────────┘

Year 2:
┌──────────────────────────────────┐
│  ┌──────────┐  ┌──────────────┐  │
│  │  React   │  │    React     │  │
│  │Dashboard │  │   Settings   │  │
│  └──────────┘  └──────────────┘  │
│        Legacy (shrinking)         │
└──────────────────────────────────┘

Year 3:
┌──────────────────────────────────┐
│  ┌──────────┐┌──────┐┌────────┐ │
│  │Dashboard ││Orders││Settings│ │
│  └──────────┘└──────┘└────────┘ │
│  Legacy: just auth (being replaced)│
└──────────────────────────────────┘
```

This is probably the single best use case for microfrontends. You're not choosing complexity for its own sake — you're managing the reality of a multi-year migration.

**3. Different Tech Stack Requirements**

One team is React, another is Svelte, a third maintains a legacy Vue app. This happens more in large organizations through acquisitions, agency work, or team preferences. Microfrontends let each team use its preferred stack.

**4. Third-Party Embeddable Widgets**

You're building a platform where third parties embed your UI into their applications. Each widget needs to be independently loadable, version-controllable, and isolated.

### When Microfrontends Are Overkill

Let me be equally specific:

- **You have fewer than 3 frontend teams.** The coordination cost of microfrontends exceeds the coordination cost of working in a monolith.
- **All teams use the same tech stack and version.** You're paying for heterogeneity you don't need.
- **Your deploy cadence is weekly or less.** If you're not shipping multiple times per day, independent deployment doesn't buy you much.
- **You don't have a platform team.** Microfrontends require shared infrastructure — shell apps, CDN configuration, shared dependency management, monitoring. Someone has to own this. If that's "everyone, kinda," it's nobody.
- **You want it because it sounds cool.** I've seen this more than I'd like to admit.

### The Hidden Costs

Every microfrontend architecture pays these costs. Non-negotiable.

| Cost | Description |
|------|-------------|
| **Duplicate dependencies** | Even with shared deps, you'll ship some code twice. React, your design system, utility libraries — version mismatches mean duplicates. |
| **Inconsistent UX** | When teams deploy independently, the settings page might use v2 of your design system while the dashboard uses v3. Users notice. |
| **Cross-MFE communication** | The dashboard needs to know what the user selected in the sidebar. In a monolith, that's a prop or a store. Across MFEs, it's a message bus, shared state, or URL params. |
| **Auth duplication** | Each MFE needs auth tokens. Shared auth is solvable but non-trivial (cookie-based auth helps, token refresh across MFEs is fun). |
| **Testing complexity** | Integration testing across MFEs requires running multiple apps simultaneously. E2E tests become slower and flakier. |
| **Debugging difficulty** | A bug might span two MFEs. Source maps might not work across boundaries. Error tracking needs correlation IDs. |
| **Performance overhead** | Multiple framework instances, multiple fetch waterfalls, larger total bundle size. The first meaningful paint often gets worse, not better. |

> **The Steve Kinney Rule:** "Every time you draw a box on a whiteboard, you're also drawing lines between boxes. Those lines are where bugs live. The more lines you draw, the more bugs you get. Draw lines intentionally."

---

## 4. MODULE FEDERATION

Module Federation is the runtime composition mechanism that makes component-level microfrontends possible. It was introduced in Webpack 5, and its successor (Module Federation 2.0) works with Rspack, Rsbuild, and Webpack. It's the most powerful — and most complex — approach to microfrontends.

### How Module Federation Works

The mental model is simple: **one application (the host) loads modules from other applications (remotes) at runtime, as if they were local imports.**

```
┌─────────────────────────────────────────────────────────────┐
│                    HOW MODULE FEDERATION WORKS                │
│                                                              │
│  BUILD TIME:                                                 │
│  ┌─────────────┐      ┌─────────────┐                       │
│  │  Host App   │      │  Remote App │                       │
│  │  (Shell)    │      │  (Dashboard)│                       │
│  │             │      │             │                       │
│  │  Exposes:   │      │  Exposes:   │                       │
│  │  - Layout   │      │  - Charts   │                       │
│  │  - Nav      │      │  - KPIs     │                       │
│  │             │      │  - DataGrid │                       │
│  │  Consumes:  │      │             │                       │
│  │  - Dashboard│      │  Shared:    │                       │
│  │    /Charts  │      │  - react    │                       │
│  │  - Settings │      │  - react-dom│                       │
│  │    /Panel   │      │             │                       │
│  └──────┬──────┘      └──────┬──────┘                       │
│         │                     │                              │
│         ▼                     ▼                              │
│  host-app.js           remoteEntry.js                        │
│  (+ chunks)            (+ chunks)                            │
│                                                              │
│  RUNTIME:                                                    │
│  1. Browser loads host-app.js                                │
│  2. Host needs <Charts /> from Dashboard                     │
│  3. Host fetches remoteEntry.js from Dashboard CDN           │
│  4. remoteEntry.js tells host where chunk files are          │
│  5. Host loads the specific chunk for Charts                 │
│  6. Charts component renders as if it were local             │
│                                                              │
│  Shared deps (react, react-dom) loaded once by whoever       │
│  initializes first. Others reuse the same instance.          │
└─────────────────────────────────────────────────────────────┘
```

### Configuration with Rsbuild (Module Federation 2.0)

Rspack and Rsbuild have first-class Module Federation support. Here's a real configuration:

**Remote App (Dashboard)** — `rsbuild.config.ts`:

```typescript
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';
import { pluginModuleFederation } from '@module-federation/rsbuild-plugin';

export default defineConfig({
  plugins: [pluginReact()],
  pluginModuleFederation({
    name: 'dashboard',
    // What this app exposes to consumers
    exposes: {
      './Charts': './src/components/Charts.tsx',
      './KPIPanel': './src/components/KPIPanel.tsx',
      './DataGrid': './src/components/DataGrid.tsx',
    },
    // Dependencies shared with the host
    shared: {
      react: {
        singleton: true,       // Only one instance of React allowed
        requiredVersion: '^19.0.0',
      },
      'react-dom': {
        singleton: true,
        requiredVersion: '^19.0.0',
      },
      '@company/design-system': {
        singleton: true,
        requiredVersion: '^3.0.0',
      },
    },
  }),
});
```

**Host App (Shell)** — `rsbuild.config.ts`:

```typescript
import { defineConfig } from '@rsbuild/core';
import { pluginReact } from '@rsbuild/plugin-react';
import { pluginModuleFederation } from '@module-federation/rsbuild-plugin';

export default defineConfig({
  plugins: [pluginReact()],
  pluginModuleFederation({
    name: 'shell',
    // Where to find remotes
    remotes: {
      dashboard: 'dashboard@https://cdn.company.com/dashboard/remoteEntry.js',
      settings: 'settings@https://cdn.company.com/settings/remoteEntry.js',
    },
    shared: {
      react: { singleton: true, requiredVersion: '^19.0.0' },
      'react-dom': { singleton: true, requiredVersion: '^19.0.0' },
      '@company/design-system': { singleton: true, requiredVersion: '^3.0.0' },
    },
  }),
});
```

**Using Remote Components in the Host:**

```typescript
// src/pages/DashboardPage.tsx
import { lazy, Suspense } from 'react';
import { ErrorBoundary } from '@company/ui';
import { DashboardSkeleton } from './DashboardSkeleton';

// These imports resolve at RUNTIME to the remote app's CDN
const Charts = lazy(() => import('dashboard/Charts'));
const KPIPanel = lazy(() => import('dashboard/KPIPanel'));

export function DashboardPage() {
  return (
    <div className="grid grid-cols-12 gap-6">
      <div className="col-span-8">
        <ErrorBoundary
          fallback={<DashboardError message="Charts failed to load" />}
        >
          <Suspense fallback={<DashboardSkeleton variant="charts" />}>
            <Charts
              dateRange={dateRange}
              onDrillDown={handleDrillDown}
            />
          </Suspense>
        </ErrorBoundary>
      </div>
      <div className="col-span-4">
        <ErrorBoundary
          fallback={<DashboardError message="KPIs unavailable" />}
        >
          <Suspense fallback={<DashboardSkeleton variant="kpi" />}>
            <KPIPanel metrics={selectedMetrics} />
          </Suspense>
        </ErrorBoundary>
      </div>
    </div>
  );
}
```

**Critical:** Always wrap remote components in both `ErrorBoundary` and `Suspense`. The remote CDN might be down. The remote might have shipped a breaking change. The network might be slow. Your app needs to handle all of these gracefully.

### Type Safety Across Federation Boundaries

This is where things get interesting — and where most tutorials stop. In a monolith, TypeScript covers you. Across Module Federation boundaries, you need explicit type contracts.

**Module Federation 2.0 has a built-in solution:** the `@module-federation/enhanced` package generates type definitions that remotes publish and hosts consume.

```typescript
// dashboard/federation.config.ts
export default {
  name: 'dashboard',
  exposes: {
    './Charts': './src/components/Charts.tsx',
  },
  // Generates .d.ts files and serves them
  dts: {
    generateTypes: true,
    consumeTypes: true,
    // Host fetches types from this URL
    remoteTypesFolder: '@mf-types',
  },
};
```

In the host, you get autocomplete and type checking for remote components — they feel like local imports.

But here's the reality: **type generation has a lag.** The remote deploys new code, the types update, but the host hasn't rebuilt yet. During that window, the runtime types don't match the compile-time types. This is a fundamental tension of runtime composition and it's one you need to plan for.

**Defensive patterns for type drift:**

```typescript
// Don't trust remote data implicitly. Validate at the boundary.
import { z } from 'zod';

const ChartPropsSchema = z.object({
  dateRange: z.object({
    start: z.string().datetime(),
    end: z.string().datetime(),
  }),
  onDrillDown: z.function().optional(),
});

// Wrapper that validates props before passing to remote
function SafeCharts(props: z.infer<typeof ChartPropsSchema>) {
  const validated = ChartPropsSchema.parse(props);
  return <Charts {...validated} />;
}
```

### Shared Dependencies: The Singleton Problem

The `singleton: true` flag in shared config means "there can be only one instance of this package in the runtime." This is critical for React (multiple React instances break hooks) and your design system (ThemeProvider needs to be a single context).

But singletons create version coupling:

```
Scenario: Host uses React 19.0.0, Remote uses React 19.1.0

With singleton: true
  → Only one React loads (host's version wins, typically)
  → Remote might break if it depends on 19.1.0 features
  → You get a runtime warning, not a build error

With singleton: false
  → Both React versions load
  → 100KB+ of duplicate React in the bundle
  → Hooks break because they see different React instances
  → Everything crashes
```

**The rule:** React, ReactDOM, and your design system must be singletons. Everything else, evaluate case by case. Lodash? Let each remote bundle its own — it's small and stateless. A state management library? Singleton if MFEs share state, otherwise let them own their own.

### Communication Between Microfrontends

This is the most underestimated complexity of microfrontends. In a monolith, components communicate through props, context, and stores. Across MFE boundaries, you need explicit communication channels.

**Option 1: Custom Events (simplest, lowest coupling)**

```typescript
// publisher (in Dashboard MFE)
function publishMetricSelected(metric: string) {
  window.dispatchEvent(
    new CustomEvent('mfe:metric-selected', {
      detail: { metric, timestamp: Date.now() },
    })
  );
}

// subscriber (in Sidebar MFE)
function useMetricSelection() {
  const [metric, setMetric] = useState<string | null>(null);

  useEffect(() => {
    const handler = (e: CustomEvent) => {
      setMetric(e.detail.metric);
    };
    window.addEventListener('mfe:metric-selected', handler);
    return () => window.removeEventListener('mfe:metric-selected', handler);
  }, []);

  return metric;
}
```

Pros: Zero dependencies, works across any framework. Cons: No type safety, no guaranteed delivery, debugging is `console.log` archaeology.

**Option 2: Nanostores (recommended for React-to-React)**

Nanostores is a tiny (< 1KB) state manager that works across framework boundaries. It's perfect for MFE communication because it's framework-agnostic and supports multiple subscribers.

```typescript
// shared/stores/user-selection.ts (published as a shared package)
import { atom, map } from 'nanostores';

export const $selectedMetric = atom<string>('revenue');
export const $dateRange = map({
  start: new Date().toISOString(),
  end: new Date().toISOString(),
});

// In Dashboard MFE (React)
import { useStore } from '@nanostores/react';
import { $selectedMetric } from '@company/shared-stores';

function MetricSelector() {
  const metric = useStore($selectedMetric);
  return (
    <Select
      value={metric}
      onChange={(val) => $selectedMetric.set(val)}
    />
  );
}

// In Sidebar MFE (could even be Svelte or Vue)
import { $selectedMetric } from '@company/shared-stores';

// Vanilla subscription
$selectedMetric.subscribe((metric) => {
  console.log('Metric changed:', metric);
});
```

**Option 3: Broadcast Channel API (cross-tab, cross-iframe)**

```typescript
// For communication across iframes or browser tabs
const channel = new BroadcastChannel('mfe-events');

// Send
channel.postMessage({
  type: 'METRIC_SELECTED',
  payload: { metric: 'revenue' },
});

// Receive
channel.addEventListener('message', (event) => {
  if (event.data.type === 'METRIC_SELECTED') {
    handleMetricChange(event.data.payload.metric);
  }
});
```

**Option 4: Comlink (for Web Worker-based MFEs)**

If you're running MFEs in Web Workers for isolation (advanced pattern), Comlink makes RPC-style communication feel like function calls:

```typescript
// worker.ts (running the MFE in a Web Worker)
import { expose } from 'comlink';

const dashboardAPI = {
  getMetrics: async (range: DateRange) => {
    const data = await fetchMetrics(range);
    return data;
  },
  subscribe: (callback: (metric: string) => void) => {
    metricEmitter.on('change', callback);
  },
};

expose(dashboardAPI);

// host.ts (main thread)
import { wrap } from 'comlink';

const worker = new Worker('./worker.ts', { type: 'module' });
const dashboard = wrap<typeof dashboardAPI>(worker);

// Feels like a local function call, runs in a Worker
const metrics = await dashboard.getMetrics({ start, end });
```

### Communication Pattern Decision Table

| Pattern | Coupling | Type Safety | Cross-Framework | Performance | Complexity |
|---------|----------|-------------|-----------------|-------------|------------|
| Custom Events | Lowest | None (manual) | Yes | Best | Lowest |
| Nanostores | Low | Good | Yes | Great | Low |
| Shared Zustand | Medium | Excellent | React only | Great | Medium |
| Broadcast Channel | Lowest | None (manual) | Yes | Good | Low |
| Comlink (Workers) | Low | Good | Yes | Overhead | High |
| URL params | Lowest | None | Yes | N/A | Lowest |

**My recommendation:** Start with Nanostores for React-to-React MFE communication. Fall back to Custom Events when crossing framework boundaries. Use Broadcast Channel only for cross-tab scenarios. Use Comlink only when you genuinely need Worker isolation.

### The Complexity Cost: An Honest Assessment

Let me lay out what Module Federation actually costs your team:

```
ONE-TIME COSTS:
  - Shell app setup and routing                    2-4 weeks
  - Shared dependency configuration                1-2 weeks
  - CI/CD pipeline for independent deploys         2-4 weeks
  - Type contract generation and sharing           1-2 weeks
  - Local development environment (all MFEs)       1-2 weeks
  - Monitoring and error boundary setup            1 week
                                        TOTAL:     8-15 weeks

ONGOING COSTS:
  - Shared dependency version management           2-4 hours/week
  - Shell app maintenance                          1-2 hours/week
  - Cross-MFE integration testing                  4-8 hours/sprint
  - Debugging cross-boundary issues                Unpredictable
  - Onboarding new engineers to the architecture   Extra 1-2 weeks

WHAT YOU GET:
  - Independent deployment per team                ✅
  - Different tech stacks per team                 ✅
  - Reduced blast radius per deploy                ✅
  - Team autonomy in tooling choices               ✅
  - Faster CI per team (smaller codebase)          ✅

WHAT YOU LOSE:
  - Atomic cross-cutting changes                   ❌
  - Full-app type safety                           ❌
  - Simple local development                       ❌
  - Optimal bundle size                            ❌
  - Easy debugging                                 ❌
```

If the "What You Get" list doesn't address your top 3 pain points, don't do this.

---

## 5. BUILD-TIME COMPOSITION

Build-time composition is the pragmatic middle ground: you organize your code like microfrontends (separate packages, clear boundaries, team ownership) but compose them at build time into a single application. It's microfrontends' organizational benefits without the runtime complexity.

### How It Works

```
packages/
  @company/shell/           # Host app — routes, layout, providers
    src/
      App.tsx
      routes.tsx            # Imports features as packages
    package.json            # Dependencies: @company/dashboard, @company/settings

  @company/dashboard/       # "Microfrontend" as a package
    src/
      index.ts              # Public API
      DashboardPage.tsx
      components/
      hooks/
    package.json            # Own dependencies, own tests

  @company/settings/        # Another "microfrontend" package
    src/
      index.ts
      SettingsPage.tsx
    package.json

  @company/shared-ui/       # Shared design system
    src/
      Button.tsx
      Card.tsx
    package.json
```

**The shell app imports features as packages:**

```typescript
// packages/@company/shell/src/routes.tsx
import { lazy } from 'react';

// These are monorepo packages, resolved at BUILD TIME
const DashboardPage = lazy(() =>
  import('@company/dashboard').then((m) => ({ default: m.DashboardPage }))
);
const SettingsPage = lazy(() =>
  import('@company/settings').then((m) => ({ default: m.SettingsPage }))
);

export const routes = [
  { path: '/dashboard', element: <DashboardPage /> },
  { path: '/settings', element: <SettingsPage /> },
];
```

### The Package Contract

Each feature package exports a clean public API:

```typescript
// packages/@company/dashboard/src/index.ts

// Public components (what the shell can import)
export { DashboardPage } from './DashboardPage';
export { DashboardWidget } from './components/DashboardWidget';

// Public types (for cross-package type safety)
export type { DashboardConfig, MetricDefinition } from './types';

// Public hooks (if other packages need dashboard data)
export { useDashboardMetrics } from './hooks/useDashboardMetrics';

// Everything else is INTERNAL — not exported, not importable
// (enforced by ESLint boundaries — Section 9)
```

### Build-Time vs. Runtime: The Trade-Off Table

| Dimension | Build-Time | Runtime (Module Fed) |
|-----------|------------|---------------------|
| **Independent deploy** | No — single build artifact | Yes — each remote deploys alone |
| **Build errors** | Caught at compile time | Caught at runtime (or never) |
| **Type safety** | Full TypeScript coverage | Needs type generation / contracts |
| **Bundle optimization** | Bundler sees entire graph, optimal splitting | Each remote bundles independently, some duplication |
| **Dev experience** | Standard monorepo tooling | Custom dev server that loads remotes |
| **Team autonomy** | Medium — same build, same CI | High — own build, own CI, own tools |
| **Performance** | Optimal — single optimized bundle | Overhead — remote entry fetching, shared dep negotiation |
| **Debugging** | Normal — single source map | Harder — multiple source maps, remote origins |
| **Complexity** | Low — familiar package model | High — federation config, shared deps, communication |

### When Build-Time Composition Wins

1. **Your teams want module ownership but don't need independent deploy.** This is most teams. They want clear boundaries, separate test suites, and code ownership — but shipping everything together is fine.

2. **You want a stepping stone to runtime composition.** Structure your packages with clean APIs now. If you need runtime composition later, the extraction is straightforward because the boundaries already exist.

3. **You need maximum performance.** The bundler can tree-shake across package boundaries, share chunks optimally, and produce the smallest possible output.

4. **You're building a library or platform with pluggable features.**

```typescript
// A feature flag system that includes/excludes entire packages
const enabledFeatures = [
  '@company/dashboard',
  '@company/settings',
  // '@company/analytics', // disabled — won't even be in the bundle
];

const routes = enabledFeatures.flatMap((pkg) => {
  // Dynamic import — if the package is excluded, the bundler tree-shakes it
  return require(pkg).routes;
});
```

### Monorepo Tooling for Build-Time Composition

**Turborepo** handles the build orchestration:

```jsonc
// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],      // Build dependencies first
      "outputs": ["dist/**"],
      "cache": true                  // Cache build outputs
    },
    "test": {
      "dependsOn": ["^build"],
      "cache": true
    },
    "lint": {
      "cache": true
    }
  }
}
```

With this setup:
- `turbo run build` builds packages in dependency order, with full caching
- Change `@company/dashboard`? Only `@company/dashboard` and `@company/shell` rebuild
- Change `@company/shared-ui`? Everything that depends on it rebuilds
- Second run with no changes? Cached — instant

**Nx** provides the same with `affected` commands:

```bash
# Only build/test what changed since main
nx affected --target=build
nx affected --target=test
```

### The Package Boundary Anti-Pattern

The most common mistake with build-time composition: **packages that import each other's internals.**

```typescript
// BAD — Dashboard reaches into Settings' internal structure
import { useSettingsAPI } from '@company/settings/src/internal/hooks/useSettingsAPI';

// GOOD — Dashboard uses Settings' public API
import { getUserPreferences } from '@company/settings';
```

We'll enforce this automatically in Section 9. For now, the rule is: **every package has a public API (its `index.ts` exports), and nothing outside the package imports anything else.**

---

## 6. ISLAND ARCHITECTURE

Island Architecture flips the frontend paradigm. Instead of "JavaScript application that renders HTML," it's "HTML document with JavaScript islands." The page is static by default. Only the interactive parts (search bars, carousels, dynamic forms) load JavaScript and hydrate.

### The Mental Model

```
┌────────────────────────────────────────────────────────┐
│                  TRADITIONAL SPA                        │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │                                                    │ │
│  │          ENTIRE PAGE IS JAVASCRIPT                 │ │
│  │                                                    │ │
│  │  Header ─── Nav ─── Content ─── Sidebar ─── Footer│ │
│  │  (JS)      (JS)     (JS)       (JS)       (JS)   │ │
│  │                                                    │ │
│  │  Bundle: 350KB JS                                  │ │
│  │  TTI: 3.2s                                         │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                  ISLAND ARCHITECTURE                    │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │                                                    │ │
│  │  Header     Nav       Content     Sidebar   Footer │ │
│  │  (HTML)    (HTML)    (HTML)      (HTML)    (HTML)  │ │
│  │                                                    │ │
│  │        ┌──────────┐        ┌──────────┐           │ │
│  │        │  Search  │        │  Cart    │           │ │
│  │        │  Island  │        │  Island  │           │ │
│  │        │  (JS)    │        │  (JS)    │           │ │
│  │        └──────────┘        └──────────┘           │ │
│  │                                                    │ │
│  │  Bundle: 45KB JS (only interactive parts)          │ │
│  │  TTI: 0.8s                                         │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### Astro: The Island Architecture Framework

Astro is the leading implementation of Island Architecture. It renders everything to static HTML at build time, and only hydrates the components you explicitly mark as interactive.

```astro
---
// src/pages/product/[id].astro
// This runs at BUILD TIME (or server-side) — zero JS sent to client
import Layout from '@layouts/Layout.astro';
import ProductInfo from '@components/ProductInfo.astro';  // Static
import Reviews from '@components/Reviews.astro';          // Static
import AddToCart from '@components/AddToCart';             // React island
import ImageGallery from '@components/ImageGallery';      // React island

const { id } = Astro.params;
const product = await getProduct(id);
const reviews = await getReviews(id);
---

<Layout title={product.name}>
  <!-- Static HTML — no JS loaded for these -->
  <ProductInfo product={product} />
  <Reviews reviews={reviews} />

  <!-- Interactive Islands — JS loads only for these -->
  <AddToCart client:visible productId={product.id} price={product.price} />
  <ImageGallery client:idle images={product.images} />
</Layout>
```

### Astro's Hydration Directives

The `client:*` directives control *when* an island hydrates:

```astro
<!-- Hydrate immediately on page load -->
<SearchBar client:load />

<!-- Hydrate when the browser is idle (requestIdleCallback) -->
<RecommendationCarousel client:idle />

<!-- Hydrate when the component scrolls into view (IntersectionObserver) -->
<ReviewForm client:visible />

<!-- Hydrate only when viewport width matches -->
<MobileMenu client:media="(max-width: 768px)" />

<!-- Never hydrate — render on server, send as static HTML -->
<ProductSpecs />
```

**The performance impact is dramatic.** A marketing page that was 350KB of JavaScript becomes 45KB. Time to Interactive drops from 3 seconds to under 1 second. Core Web Vitals go from "needs improvement" to "good" overnight.

### When Island Architecture Is Perfect

**Content-Heavy Sites**
- Marketing pages, landing pages, blogs
- Documentation sites (Astro's Starlight is built for this)
- E-commerce storefronts where most content is static product info
- News sites, media sites, portfolio sites

**The Common Pattern: 90/10 Sites**

If 90% of your page is content that doesn't change based on user interaction, and 10% is interactive (search, forms, carousels, carts), Island Architecture is almost certainly the right choice. You're shipping 100% of the JavaScript for the 10% that needs it, and 0% for the 90% that doesn't.

### When Island Architecture Doesn't Fit

**Highly Interactive Applications**
- Dashboards where every component responds to user input
- Real-time collaboration tools (Figma, Google Docs)
- Complex forms with interdependent fields
- Apps where the "page" is really a single interactive surface

If your page is 80%+ interactive, the island model adds overhead without benefit — you'll end up hydrating almost everything anyway, and the island boundaries create communication complexity.

### Islands + React: A Practical Example

Astro lets you use React (or Svelte, Vue, Solid, Preact) components as islands:

```tsx
// src/components/AddToCart.tsx (React component used as an Astro island)
import { useState } from 'react';
import { useCartStore } from '@stores/cart';

interface AddToCartProps {
  productId: string;
  price: number;
  variants?: Array<{ id: string; label: string }>;
}

export default function AddToCart({ productId, price, variants }: AddToCartProps) {
  const [selectedVariant, setSelectedVariant] = useState(variants?.[0]?.id);
  const [quantity, setQuantity] = useState(1);
  const addItem = useCartStore((s) => s.addItem);

  const handleAdd = () => {
    addItem({
      productId,
      variantId: selectedVariant,
      quantity,
      price,
    });
  };

  return (
    <div className="flex flex-col gap-3">
      {variants && variants.length > 1 && (
        <select
          value={selectedVariant}
          onChange={(e) => setSelectedVariant(e.target.value)}
          className="rounded-md border px-3 py-2"
        >
          {variants.map((v) => (
            <option key={v.id} value={v.id}>{v.label}</option>
          ))}
        </select>
      )}
      <div className="flex items-center gap-2">
        <button
          onClick={() => setQuantity(Math.max(1, quantity - 1))}
          className="rounded-md bg-gray-100 px-3 py-1"
        >
          -
        </button>
        <span className="w-8 text-center">{quantity}</span>
        <button
          onClick={() => setQuantity(quantity + 1)}
          className="rounded-md bg-gray-100 px-3 py-1"
        >
          +
        </button>
      </div>
      <button
        onClick={handleAdd}
        className="rounded-lg bg-blue-600 px-6 py-3 text-white font-semibold
                   hover:bg-blue-700 transition-colors"
      >
        Add to Cart — ${(price * quantity).toFixed(2)}
      </button>
    </div>
  );
}
```

This component ships zero JavaScript until the user scrolls to it (with `client:visible`). When it hydrates, it loads only the React runtime, this component, and the cart store. The rest of the page — product images, descriptions, reviews — remains static HTML.

### Partial Hydration: The Underlying Concept

Island Architecture is one implementation of a broader concept called **partial hydration** — the idea that not every component on a page needs to be interactive, and the ones that don't shouldn't cost the user JavaScript bytes.

Other implementations:
- **React Server Components** — server-rendered components that never hydrate on the client. The "island" is any component marked `'use client'`.
- **Qwik** — "resumability" instead of hydration. The framework serializes component state into HTML and resumes execution on interaction, without replaying the component tree.
- **Fresh (Deno)** — Preact-based island architecture similar to Astro.

The convergence is clear: **the industry is moving toward "server by default, client when needed."** Whether you use Astro, Next.js with Server Components, or Qwik, the principle is the same: ship less JavaScript, hydrate only what's interactive.

---

## 7. BACKEND FOR FRONTEND (BFF)

The BFF pattern answers a simple question: **who transforms backend API responses into the shape the frontend needs?**

In a naive architecture, the frontend fetches from multiple backend services and assembles the data:

```
┌─────────────────────────────────────────────────┐
│                  NAIVE: NO BFF                   │
│                                                  │
│  ┌──────────┐                                    │
│  │ Frontend │──── GET /users/123                 │
│  │          │──── GET /users/123/orders           │
│  │          │──── GET /products?ids=a,b,c         │
│  │          │──── GET /reviews?user=123           │
│  │          │──── GET /recommendations?user=123   │
│  └──────────┘                                    │
│       │                                          │
│       │  5 requests, waterfall, over-fetching,   │
│       │  client assembles data, handles errors   │
│       │  for each independently                   │
│                                                  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│                  WITH BFF                        │
│                                                  │
│  ┌──────────┐      ┌─────────┐                  │
│  │ Frontend │──────│   BFF   │── Users Service   │
│  │          │ GET  │         │── Orders Service   │
│  │          │ /    │         │── Products Service  │
│  │          │ dash │         │── Reviews Service   │
│  │          │      │         │── Reco Service      │
│  └──────────┘      └─────────┘                   │
│       │                 │                         │
│       │  1 request     Aggregates, transforms,   │
│       │  exact shape   caches, handles failures   │
│       │  frontend needs for downstream services   │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Why Different Clients Need Different APIs

This is the core insight behind BFF. Your web app, mobile app, and smartwatch all consume the same backend services — but they need dramatically different data shapes:

```typescript
// What the User Service returns
interface UserServiceResponse {
  id: string;
  email: string;
  name: string;
  avatarUrl: string;
  preferences: Record<string, unknown>;
  metadata: {
    createdAt: string;
    lastLogin: string;
    loginCount: number;
    deviceHistory: Array<{ device: string; date: string }>;
  };
  addresses: Address[];
  paymentMethods: PaymentMethod[];
  // ... 30 more fields
}

// What the WEB dashboard needs
interface WebDashboardUser {
  name: string;
  avatarUrl: string;
  recentOrders: OrderSummary[];    // From Orders Service
  recommendations: Product[];       // From Reco Service
  unreadNotifications: number;      // From Notifications Service
}

// What the MOBILE app needs
interface MobileUser {
  name: string;
  avatarUrl: string;               // Transformed to a smaller resolution
  lastOrderStatus: string;          // Just the most recent order
  pushEnabled: boolean;             // From Preferences Service
}
```

The web app needs a rich dashboard. The mobile app needs minimal data to render fast on 3G. Making either client do the aggregation and transformation wastes bandwidth, battery, and developer time.

### Next.js as a BFF

Here's where modern frameworks blur the line. Next.js with Server Components and Route Handlers is effectively a BFF built into your frontend framework.

```typescript
// app/api/dashboard/route.ts — Next.js Route Handler as BFF endpoint

import { NextResponse } from 'next/server';
import { auth } from '@/lib/auth';
import { userService, orderService, recoService } from '@/lib/services';

export async function GET() {
  const session = await auth();
  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Parallel fetches to backend services
  const [user, recentOrders, recommendations] = await Promise.all([
    userService.getUser(session.userId),
    orderService.getRecent(session.userId, { limit: 5 }),
    recoService.getForUser(session.userId, { limit: 10 }),
  ]);

  // Transform into exactly the shape the frontend needs
  const dashboardData = {
    user: {
      name: user.name,
      avatarUrl: user.avatarUrl,
    },
    orders: recentOrders.map((order) => ({
      id: order.id,
      status: order.status,
      total: order.totalCents / 100,
      date: order.createdAt,
    })),
    recommendations: recommendations.map((product) => ({
      id: product.id,
      name: product.name,
      imageUrl: product.images[0]?.url,
      price: product.priceCents / 100,
    })),
  };

  return NextResponse.json(dashboardData, {
    headers: {
      'Cache-Control': 'private, max-age=60',  // Cache for 1 minute
    },
  });
}
```

Or even better — **Server Components as an implicit BFF:**

```typescript
// app/dashboard/page.tsx — Server Component
// This runs on the server. It IS the BFF.

import { auth } from '@/lib/auth';
import { userService, orderService, recoService } from '@/lib/services';
import { DashboardView } from './DashboardView';

export default async function DashboardPage() {
  const session = await auth();

  const [user, orders, recommendations] = await Promise.all([
    userService.getUser(session.userId),
    orderService.getRecent(session.userId, { limit: 5 }),
    recoService.getForUser(session.userId, { limit: 10 }),
  ]);

  // No API endpoint needed — the server component fetches and renders
  return (
    <DashboardView
      user={{ name: user.name, avatarUrl: user.avatarUrl }}
      orders={orders}
      recommendations={recommendations}
    />
  );
}
```

With Server Components, the BFF isn't a separate service — it's the server-side part of your component tree. Data fetching, transformation, and rendering happen in one place. No separate API to maintain, no client-side fetch waterfall, no over-fetching.

### GraphQL as a BFF Alternative

GraphQL solves the "different clients need different shapes" problem differently: instead of a dedicated BFF per client, you expose a single flexible API that each client queries for exactly what it needs.

```graphql
# Web dashboard query — gets everything
query WebDashboard {
  me {
    name
    avatarUrl
    recentOrders(limit: 5) {
      id
      status
      total
      date
    }
    recommendations(limit: 10) {
      id
      name
      imageUrl
      price
    }
  }
}

# Mobile app query — gets minimal data
query MobileDashboard {
  me {
    name
    avatarUrl(size: THUMBNAIL)
    lastOrder {
      status
    }
  }
}
```

**GraphQL vs. Dedicated BFF:**

| Dimension | GraphQL | Dedicated BFF |
|-----------|---------|---------------|
| Client flexibility | Clients query what they need | BFF returns fixed shapes |
| Backend complexity | Schema, resolvers, data loader | Simple route handlers |
| Performance | N+1 queries (without DataLoader), query complexity | Optimized per-endpoint |
| Caching | Complex (normalized cache) | Simple (HTTP caching) |
| Tooling | Heavy (codegen, schema management) | Light (REST conventions) |
| When it shines | Many clients, rapidly changing needs | Few clients, stable shapes |

### BFF Patterns and Anti-Patterns

**Pattern: One BFF per client type**
```
Mobile App → Mobile BFF → Backend Services
Web App    → Web BFF    → Backend Services
Partner API → Public BFF → Backend Services
```
Each BFF is owned by the team that builds the client. They can evolve independently.

**Anti-Pattern: The "God BFF"**
```
All Clients → ONE BFF → Backend Services
```
This becomes a bottleneck. Every client change requires a BFF change. The BFF grows into a monolith that's harder to change than the backend services it was supposed to simplify.

**Anti-Pattern: Business logic in the BFF**
```typescript
// BAD — the BFF is doing business logic
export async function GET() {
  const orders = await orderService.getAll(userId);

  // This belongs in the Orders Service, not the BFF
  const eligibleForReturn = orders.filter((o) =>
    o.status === 'delivered' &&
    daysSince(o.deliveredAt) < 30 &&
    !o.items.some((i) => i.category === 'final-sale')
  );

  return NextResponse.json({ eligibleForReturn });
}
```

The BFF should aggregate, transform, and cache — not make business decisions. If you find yourself writing business rules in the BFF, that logic belongs in a backend service.

**Pattern: BFF with caching**
```typescript
// Good — BFF caches aggregated data to avoid hitting backend services on every request
import { cache } from 'react';  // React's request-level cache

const getDashboardData = cache(async (userId: string) => {
  const [user, orders, recs] = await Promise.all([
    userService.getUser(userId),
    orderService.getRecent(userId, { limit: 5 }),
    recoService.getForUser(userId, { limit: 10 }),
  ]);

  return { user, orders, recs };
});
```

---

## 8. FEATURE-BASED FILE STRUCTURE

This section is about the one decision that affects your daily developer experience more than any architecture pattern: **how you organize your files.**

### The Two Approaches

**Approach A: Group by Type (the default everyone starts with)**

```
src/
  components/
    Button.tsx
    Card.tsx
    UserAvatar.tsx
    OrderCard.tsx
    DashboardChart.tsx
    SettingsForm.tsx
    ... 200 more components
  hooks/
    useAuth.ts
    useOrders.ts
    useDashboard.ts
    useSettings.ts
    ... 50 more hooks
  utils/
    formatDate.ts
    formatCurrency.ts
    validateEmail.ts
    ... 80 more utils
  types/
    user.ts
    order.ts
    dashboard.ts
    ... 40 more type files
  services/
    userService.ts
    orderService.ts
    ... 20 more services
```

**Approach B: Group by Feature (what actually scales)**

```
src/
  features/
    auth/
      components/
        LoginForm.tsx
        SignupForm.tsx
        AuthGuard.tsx
      hooks/
        useAuth.ts
        useSession.ts
      utils/
        validateCredentials.ts
      types.ts
      index.ts              # Public API

    dashboard/
      components/
        DashboardChart.tsx
        KPIPanel.tsx
        ActivityFeed.tsx
      hooks/
        useDashboardMetrics.ts
        useDateRange.ts
      utils/
        aggregateMetrics.ts
      types.ts
      index.ts

    orders/
      components/
        OrderList.tsx
        OrderDetail.tsx
        OrderCard.tsx
      hooks/
        useOrders.ts
        useOrderFilters.ts
      services/
        orderService.ts
      types.ts
      index.ts

  shared/                   # Genuinely shared across features
    ui/                     # Design system components
      Button.tsx
      Card.tsx
      Modal.tsx
    hooks/
      useDebounce.ts
      useMediaQuery.ts
    utils/
      formatDate.ts
      formatCurrency.ts
```

### Why Feature-Based Wins

**1. Locality of Change**

When you work on the dashboard, everything you need is in `features/dashboard/`. You don't bounce between `components/`, `hooks/`, `utils/`, and `types/` to understand one feature. In a type-based structure, a change to "orders" touches files across 5 directories.

```
Type-based:   Change to "orders" touches
              components/OrderCard.tsx
              components/OrderList.tsx
              hooks/useOrders.ts
              services/orderService.ts
              types/order.ts
              utils/formatOrderStatus.ts
              → 6 directories, scattered

Feature-based: Change to "orders" touches
              features/orders/
              → 1 directory, everything right there
```

**2. Ownership Mapping**

Feature folders map directly to teams. The orders team owns `features/orders/`. The dashboard team owns `features/dashboard/`. In a type-based structure, every team touches `components/` and `hooks/`, creating collision.

**3. Deletion is Easy**

Removing a feature? Delete the folder. In a type-based structure, removing a feature means hunting through every directory for files related to that feature, hoping you got them all.

**4. Cohesion Over Coupling**

Feature-based structure keeps related code together (high cohesion) and keeps unrelated code apart (low coupling). Type-based structure groups by technical role, which creates coupling between unrelated features that happen to share a technology (both are "components," both are "hooks").

### The Feature Module Contract

Every feature should have a clean public API via its `index.ts`:

```typescript
// features/orders/index.ts

// Components that other features can use
export { OrderSummaryCard } from './components/OrderSummaryCard';

// Hooks that other features can use
export { useRecentOrders } from './hooks/useRecentOrders';

// Types that other features need
export type { Order, OrderStatus, OrderSummary } from './types';

// Everything else is PRIVATE to this feature
// - Internal components (OrderFilterDropdown, OrderStatusBadge)
// - Internal hooks (useOrderFilters, useOrderSort)
// - Internal utils (calculateOrderTotal, formatOrderDate)
// - Internal services (orderService)
```

**Rule: Features import from other features' public APIs only. Never reach into another feature's internal files.**

```typescript
// GOOD
import { OrderSummaryCard } from '@/features/orders';

// BAD — reaching into another feature's internals
import { OrderStatusBadge } from '@/features/orders/components/OrderStatusBadge';
```

### The Shared Directory Rules

The `shared/` directory is for code that is genuinely used across multiple features. But it needs guardrails, because `shared/` tends to become a dumping ground.

**Rules for `shared/`:**

1. **Must be used by 3+ features.** If only two features use it, it lives in one feature and the other imports it. If a third feature needs it, promote it to `shared/`.

2. **Must be generic.** `shared/ui/Button.tsx` is fine. `shared/utils/calculateOrderDiscount.ts` is not — that's order-specific logic masquerading as shared code.

3. **No business logic.** Shared code is infrastructure: UI components, generic hooks (`useDebounce`, `useLocalStorage`), formatting utilities, type helpers.

4. **Review promotions to shared.** When code moves to `shared/`, it gets a broader audience and more consumers. Breaking changes now affect everyone. Treat it like a public API.

### Scaling the Feature Structure

As features grow, they can develop sub-features:

```
features/
  orders/
    features/                # Sub-features of orders
      order-creation/
        components/
        hooks/
        index.ts
      order-tracking/
        components/
        hooks/
        index.ts
      returns/
        components/
        hooks/
        index.ts
    shared/                  # Shared within orders
      components/
        OrderStatusBadge.tsx
      hooks/
        useOrderPermissions.ts
    index.ts                 # Public API for the orders feature
```

This is a fractal structure — the same pattern at every level. Features contain sub-features, each with their own public API. The orders feature's `index.ts` re-exports what the outside world needs:

```typescript
// features/orders/index.ts
export { CreateOrderFlow } from './features/order-creation';
export { OrderTracker } from './features/order-tracking';
export { ReturnRequest } from './features/returns';
export type { Order, OrderStatus } from './shared/types';
```

---

## 9. ARCHITECTURAL LINTING

Here's where good intentions become enforceable rules. You can document module boundaries all day, but without automation, someone will import from another feature's internals "just this once" and then everyone will, and your carefully designed architecture will erode into spaghetti.

Architectural linting tools make boundary violations a build error, not a code review comment.

### ESLint Plugin: Boundaries

The `eslint-plugin-boundaries` package lets you define architectural rules about which modules can import from which other modules.

**Installation:**

```bash
npm install -D eslint-plugin-boundaries
```

**Configuration:**

```typescript
// eslint.config.ts (flat config)
import boundaries from 'eslint-plugin-boundaries';

export default [
  {
    plugins: {
      boundaries,
    },
    settings: {
      'boundaries/elements': [
        {
          type: 'feature',
          pattern: 'src/features/*',
          capture: ['featureName'],
        },
        {
          type: 'shared',
          pattern: 'src/shared/*',
          capture: ['sharedModule'],
        },
        {
          type: 'app',
          pattern: 'src/app/*',
        },
      ],
      'boundaries/ignore': ['**/*.test.*', '**/*.spec.*'],
    },
    rules: {
      // Features can only import from:
      // 1. Their own internals
      // 2. Other features' PUBLIC APIs (index.ts)
      // 3. Shared modules
      'boundaries/element-types': [
        'error',
        {
          default: 'disallow',
          rules: [
            {
              // Features can import from shared
              from: ['feature'],
              allow: ['shared'],
            },
            {
              // Features can import from other features' index.ts only
              from: ['feature'],
              allow: [
                ['feature', { featureName: '!${featureName}' }],
              ],
            },
            {
              // App (routes/layout) can import from features and shared
              from: ['app'],
              allow: ['feature', 'shared'],
            },
            {
              // Shared cannot import from features (no circular deps)
              from: ['shared'],
              allow: ['shared'],
              disallow: ['feature'],
            },
          ],
        },
      ],

      // Prevent importing internal files from other features
      'boundaries/entry-point': [
        'error',
        {
          default: 'disallow',
          rules: [
            {
              // Only allow importing from a feature's index.ts
              target: ['feature'],
              allow: 'index.(ts|tsx)',
            },
            {
              // Shared modules can be imported from any file
              target: ['shared'],
              allow: '**',
            },
          ],
        },
      ],
    },
  },
];
```

**What this catches:**

```typescript
// ERROR: boundaries/entry-point
// Cannot import internal file from another feature
import { OrderStatusBadge } from '@/features/orders/components/OrderStatusBadge';
//                                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// Only '@/features/orders' (index.ts) is allowed

// ERROR: boundaries/element-types
// Shared modules cannot depend on features
// In src/shared/utils/format.ts:
import { OrderStatus } from '@/features/orders';
//       ^^^^^^^^^^^ shared cannot import from feature

// OK: importing from a feature's public API
import { OrderSummaryCard } from '@/features/orders';

// OK: importing from shared
import { Button } from '@/shared/ui';
```

### dependency-cruiser: Dependency Graph Rules

`dependency-cruiser` takes a different approach: it analyzes the entire dependency graph of your project and validates it against rules you define. It's more powerful than ESLint boundaries (it can reason about transitive dependencies, cycles, and orphans) but also more complex to configure.

**Installation:**

```bash
npm install -D dependency-cruiser
npx depcruise --init
```

**Configuration (`.dependency-cruiser.cjs`):**

```javascript
/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    // No circular dependencies
    {
      name: 'no-circular',
      severity: 'error',
      from: {},
      to: {
        circular: true,
      },
    },

    // Features cannot import other features' internals
    {
      name: 'feature-boundary-violation',
      severity: 'error',
      comment: 'Features must import from other features via index.ts only',
      from: {
        path: '^src/features/([^/]+)/',
        pathNot: '^src/features/([^/]+)/index\\.ts$',
      },
      to: {
        path: '^src/features/([^/]+)/',
        pathNot: [
          // Allow importing from own feature
          '^src/features/$1/',
          // Allow importing from other features' index.ts
          '^src/features/[^/]+/index\\.ts$',
        ],
      },
    },

    // Shared cannot depend on features
    {
      name: 'shared-no-feature-deps',
      severity: 'error',
      comment: 'Shared code must not depend on feature code',
      from: {
        path: '^src/shared/',
      },
      to: {
        path: '^src/features/',
      },
    },

    // No orphan modules (files not imported by anything)
    {
      name: 'no-orphans',
      severity: 'warn',
      from: {
        orphan: true,
        pathNot: [
          '\\.(test|spec)\\.(ts|tsx)$',
          '\\.d\\.ts$',
          'index\\.ts$',
        ],
      },
      to: {},
    },

    // Prevent importing from node_modules not in package.json
    {
      name: 'no-unlisted-deps',
      severity: 'error',
      from: {},
      to: {
        dependencyTypes: ['npm-no-pkg', 'npm-unknown'],
      },
    },

    // Limit dependency depth (prevent deep transitive chains)
    {
      name: 'max-depth',
      severity: 'warn',
      from: {
        path: '^src/features/',
      },
      to: {
        reachable: true,
        path: '^src/',
        numberOfReachable: { $gte: 50 },  // Warn if a file reaches 50+ others
      },
    },
  ],

  options: {
    doNotFollow: {
      path: 'node_modules',
    },
    tsPreCompilationDeps: true,
    tsConfig: {
      fileName: './tsconfig.json',
    },
    reporterOptions: {
      dot: {
        theme: {
          graph: { rankdir: 'LR' },
          modules: [
            {
              criteria: { source: '^src/features/' },
              attributes: { fillcolor: '#ccffcc' },
            },
            {
              criteria: { source: '^src/shared/' },
              attributes: { fillcolor: '#ccccff' },
            },
          ],
        },
      },
    },
  },
};
```

**Running dependency-cruiser:**

```bash
# Validate the dependency graph
npx depcruise src --config .dependency-cruiser.cjs

# Generate a visual dependency graph
npx depcruise src --config .dependency-cruiser.cjs --output-type dot | dot -T svg > dependency-graph.svg

# Check only what changed (for CI)
npx depcruise src --config .dependency-cruiser.cjs --reaches "src/features/orders"
```

### Putting It All Together: CI Integration

Add both tools to your CI pipeline so violations are caught before merge:

```yaml
# .github/workflows/architecture.yml
name: Architecture Checks

on:
  pull_request:
    paths:
      - 'src/**'

jobs:
  architecture:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'npm'

      - run: npm ci

      - name: ESLint Boundary Check
        run: npx eslint src/ --rule 'boundaries/element-types: error' --rule 'boundaries/entry-point: error'

      - name: Dependency Graph Validation
        run: npx depcruise src --config .dependency-cruiser.cjs --output-type err

      - name: Check for Circular Dependencies
        run: npx depcruise src --config .dependency-cruiser.cjs --output-type err-long --validate .dependency-cruiser.cjs
```

### Pre-Commit Hook for Fast Feedback

Don't wait for CI. Catch violations locally:

```javascript
// .lintstagedrc.js
module.exports = {
  'src/features/**/*.{ts,tsx}': [
    'eslint --rule "boundaries/element-types: error" --rule "boundaries/entry-point: error"',
  ],
};
```

### Visualizing Your Architecture

One of `dependency-cruiser`'s best features is visualization. Run it periodically to see your actual architecture (vs. what you think your architecture is):

```bash
# Full dependency graph as SVG
npx depcruise src \
  --config .dependency-cruiser.cjs \
  --output-type dot \
  --exclude "\\.(test|spec)" \
  | dot -T svg > architecture.svg

# Feature-level summary (collapsed)
npx depcruise src/features \
  --config .dependency-cruiser.cjs \
  --output-type ddot \
  --collapse "^src/features/([^/]+)" \
  | dot -T svg > feature-dependencies.svg
```

The resulting SVG shows you which features depend on which, where cycles exist, and which shared modules are most depended on. I recommend generating this in CI and posting it as a PR comment when the dependency graph changes.

### The Ratchet Pattern

Here's a technique for gradually improving an existing codebase's architecture without a big-bang refactor:

```javascript
// .dependency-cruiser.cjs — The Ratchet

// Step 1: Count current violations
// npx depcruise src --output-type err | wc -l
// Result: 47 violations

// Step 2: Set the ratchet at the current count
const MAX_VIOLATIONS = 47;

module.exports = {
  forbidden: [
    // ... your rules
  ],
  // In CI, count violations and fail if they increase
  // (separate script, not built into dependency-cruiser)
};
```

```bash
#!/bin/bash
# scripts/architecture-ratchet.sh

VIOLATIONS=$(npx depcruise src --config .dependency-cruiser.cjs --output-type err 2>&1 | grep -c "ERROR")
MAX_ALLOWED=47

if [ "$VIOLATIONS" -gt "$MAX_ALLOWED" ]; then
  echo "Architecture violations increased: $VIOLATIONS > $MAX_ALLOWED"
  echo "New violations must be fixed before merging."
  exit 1
fi

if [ "$VIOLATIONS" -lt "$MAX_ALLOWED" ]; then
  echo "Great! Violations decreased: $VIOLATIONS < $MAX_ALLOWED"
  echo "Update MAX_ALLOWED in this script to $VIOLATIONS to lock in progress."
fi
```

The ratchet only turns one way: violations can decrease, never increase. Every PR that fixes a violation lowers the ceiling. Over time, you converge on zero violations without blocking anyone.

---

## 10. THE FRONTEND ARCHITECT'S PLAYBOOK

Architecture decisions are not technical decisions disguised as technical decisions. They're bets about the future — about how your team will grow, how your product will evolve, and what kinds of changes will be most common. The 100x architect makes these bets deliberately, with an explicit understanding of the trade-offs.

### The Questions to Ask Before Any Architecture Decision

**1. "What's the change that happens most often?"**

If the most common change is "add a new feature" — optimize for feature isolation. If it's "change how everything looks" — optimize for shared components. If it's "update the API contract" — optimize for type safety across boundaries.

```
Most common change          → Optimize for
──────────────────────────────────────────────
New feature                 → Feature isolation, clear boundaries
Cross-cutting style change  → Shared design system, global tokens
API schema change           → Full-stack type safety, codegen
Performance tuning          → Single build, bundler visibility
Team onboarding             → Simple structure, minimal abstractions
```

**2. "What's our team topology in 18 months?"**

Don't architect for today. Don't architect for 5 years from now either. 18 months is the sweet spot — far enough to avoid immediate rewrites, close enough to predict with reasonable accuracy.

```
Current: 1 team, 5 engineers
18 months: 2 teams, 12 engineers
→ Modular monolith with feature-based structure
→ Prepare for extraction but don't extract yet

Current: 3 teams, 20 engineers
18 months: 6 teams, 40 engineers
→ Monorepo with build-time composition
→ Independent build pipelines per team
→ Evaluate runtime composition for the highest-traffic teams

Current: 8 teams, 60 engineers, legacy codebase
18 months: 10 teams, 80 engineers, ongoing migration
→ Microfrontends with Module Federation for new features
→ Strangler fig for legacy migration
→ Dedicated platform team for shell and shared infra
```

**3. "What's the blast radius of a bad deploy?"**

If one team's broken code takes down the entire app, and you deploy multiple times a day, you need isolation. If deploys are weekly and go through staging, the blast radius is already managed.

**4. "What's our operational maturity?"**

Microfrontends require: per-MFE monitoring, per-MFE error tracking, per-MFE performance budgets, shared auth infrastructure, CDN configuration per remote, rollback per MFE, integration testing across MFEs.

If your team doesn't have dedicated DevOps/platform engineering, the operational overhead of microfrontends will eat you alive.

**5. "What would it take to reverse this decision?"**

This is the most important question. Some architecture decisions are one-way doors — hard to reverse once committed. Others are two-way doors — easy to walk back if they don't work out.

```
ONE-WAY DOORS (commit carefully):
  - Polyrepo — merging repos back is painful
  - Different frameworks per team — migrating back to one is a rewrite
  - Runtime composition with external teams consuming your MFEs
  - Public API contracts that external partners depend on

TWO-WAY DOORS (experiment freely):
  - Feature-based file structure — just move files
  - Adding ESLint boundary rules — can relax later
  - Build-time composition in a monorepo — packages can be merged back
  - BFF layer — can be replaced with direct calls
  - Island architecture for a new section — doesn't affect existing pages
```

### The Trade-Off Matrix

Every architecture decision involves trading off between these properties. You can't maximize all of them. Pick your top 3 and explicitly accept lower scores on the rest.

```
┌────────────────────────────────────────────────────────────────────┐
│                    THE TRADE-OFF MATRIX                             │
│                                                                     │
│  Property          │ Mono │ Mod Mono │ Build-Time │ Runtime MFE    │
│  ──────────────────┼──────┼──────────┼────────────┼───────────     │
│  Team autonomy     │ Low  │ Medium   │ Medium     │ High           │
│  Deploy independence│ None │ Low      │ None       │ Full           │
│  Type safety       │ Full │ Full     │ Full       │ Partial        │
│  Bundle optimization│ Best │ Great    │ Great      │ Worst          │
│  Dev experience    │ Best │ Great    │ Good       │ Worst          │
│  Debugging ease    │ Best │ Great    │ Great      │ Hard           │
│  Onboarding speed  │ Worst│ Good     │ Good       │ Best (per MFE) │
│  Cross-cutting chg │ Easy │ Medium   │ Medium     │ Hard           │
│  Blast radius      │ Full │ Full     │ Full       │ Per-MFE        │
│  Operational cost  │ Low  │ Low      │ Medium     │ High           │
│  Migration escape  │ N/A  │ Easy     │ Medium     │ Hard           │
│                                                                     │
│  Mono = Monolith                                                    │
│  Mod Mono = Modular Monolith (Shopify-style)                       │
│  Build-Time = Monorepo with package composition                     │
│  Runtime MFE = Module Federation / runtime composition              │
└────────────────────────────────────────────────────────────────────┘
```

### The Decision Flowchart

```
START: You're choosing a frontend architecture
│
├─ How many frontend engineers?
│  ├─ < 15 engineers
│  │  └─ MONOLITH with feature-based structure
│  │     Add ESLint boundaries from day one
│  │
│  ├─ 15-50 engineers
│  │  ├─ Same tech stack?
│  │  │  ├─ Yes → MODULAR MONOLITH in monorepo
│  │  │  │        Build-time composition, independent CI per package
│  │  │  └─ No  → MICROFRONTENDS (build-time if possible)
│  │  │           Route-level splitting, shared design system package
│  │  │
│  │  └─ Migrating from legacy?
│  │     └─ Yes → STRANGLER FIG microfrontends
│  │              New features as MFEs, legacy in iframe/wrapper
│  │
│  └─ > 50 engineers
│     ├─ Multiple business units?
│     │  └─ Yes → RUNTIME MICROFRONTENDS (Module Federation)
│     │           Platform team, shell app, shared dep management
│     └─ Single product?
│        └─ MODULAR MONOLITH or BUILD-TIME COMPOSITION
│           You probably need less than you think
│
├─ Is this a content-heavy site?
│  ├─ > 70% static content → ISLAND ARCHITECTURE (Astro)
│  └─ < 30% static content → Continue above
│
└─ Do you need a BFF?
   ├─ Multiple client types (web, mobile, partner API)?
   │  └─ Yes → BFF PER CLIENT TYPE
   │           Or GraphQL if shapes change frequently
   ├─ Complex backend with many microservices?
   │  └─ Yes → NEXT.JS SERVER COMPONENTS AS BFF
   │           Aggregation in the server-side component tree
   └─ Simple CRUD with one backend?
      └─ No BFF needed — direct API calls are fine
```

### The Architect's Checklist (Copy This)

Before committing to any architecture decision, verify:

```
PROBLEM VALIDATION
□ I can articulate the specific pain point (not "it would be nice")
□ I've measured the pain (CI time, merge conflicts, deploy failures)
□ I've tried simpler solutions first (better structure, caching, tooling)

SOLUTION EVALUATION
□ The solution addresses my top 3 pain points
□ I've identified what I'm giving up (trade-offs are explicit)
□ I've talked to teams who've done this (not just read blog posts)
□ I have a proof of concept, not just a plan

REVERSIBILITY
□ I know whether this is a one-way or two-way door
□ If one-way, I've validated with a pilot (one team, one feature)
□ I have a rollback plan if it doesn't work

OPERATIONAL READINESS
□ My team can operate this (monitoring, debugging, deploys)
□ I have (or can hire) the platform expertise needed
□ New engineers can understand this within their first sprint

MIGRATION PATH
□ I'm not doing a big bang migration
□ There's a gradual adoption path (feature by feature)
□ The old and new can coexist during transition
□ I have a timeline (with milestones, not just "when it's done")
```

### Final Thought: The Right Answer Is Usually a Mix

The most successful architectures I've seen are not pure anything. They're hybrids that use different patterns for different parts of the system:

- A **modular monolith** for the core product, with **ESLint boundary enforcement**
- **Island architecture** (Astro) for the marketing site and docs
- **Build-time composition** for the design system and shared packages
- A **BFF layer** (Next.js Server Components) for API aggregation
- **One or two runtime microfrontends** for the parts of the app that genuinely need independent deployment (the legacy migration, the team in a different timezone, the acquired product)

The 100x architect doesn't pick one pattern and apply it everywhere. They pick the right pattern for each problem and create clear boundaries between them. They make the simple things simple and the complex things possible. And they always, always leave an escape hatch.

> **Steve Kinney's closing thought on enterprise architecture:** "The best architecture is the one your team can reason about at 2 AM when production is down. If your architecture requires a 30-minute explanation before someone can debug an issue, you've optimized for the wrong thing."

Build for clarity. Enforce with automation. Evolve with evidence.

---

*Next: [Chapter 20 — Design Systems at Scale](./20-design-systems.md) — building component libraries that serve multiple teams, products, and platforms.*
