# Part IV — Architecture at Scale

> Monorepos, design systems, testing, CI/CD, caching, performance budgets, and the patterns that let large teams move fast.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| 14 | [Performance Optimization — Mobile](./13-mobile-performance.md) | 2,739 | Intermediate to Advanced | Cold start, FlashList, virtualization, bundle size, memory management, React Compiler, re-renders, animations, Reanimated, image optimization |
| 15 | [Profiling & Debugging](./14-profiling-debugging.md) | 1,441 | Intermediate to Advanced | Hermes profiling, Chrome DevTools for RN, Android Studio profiler, Xcode Instruments, Perfetto, Reactotron, memory leaks, crash analysis |
| 16 | [Monorepo Architecture — Mobile + Web from One Repo](./15-monorepo.md) | 2,714 | Intermediate to Advanced | Turborepo, pnpm workspaces, monorepo structure, code sharing RN + Next.js, build ordering, remote caching, next-forge, dependency management |
| 17 | [Design Systems & Component Libraries](./16-design-systems.md) | 1,612 | Intermediate | Design tokens, shadcn/ui, Tamagui, NativeWind, Storybook, CVA, accessibility, compound components |
| 18 | [Testing Strategy — What to Test, How to Test, When to Stop](./17-testing.md) | 3,572 | Intermediate | Testing trophy, Jest, RNTL, Maestro, Playwright, MSW, testing hooks, snapshot testing, visual regression, performance testing |
| 19 | [CI/CD for Mobile & Web](./18-cicd.md) | 2,789 | Intermediate to Advanced | GitHub Actions, EAS Build CI, preview deployments, performance budgets, semantic versioning, Changesets, Fastlane, self-hosted runners |
| 20 | [Frontend Architecture Patterns](./19-fe-architecture.md) | 2,289 | Advanced | Monoliths, microfrontends, module federation, BFF, island architecture, feature-based structure, architectural linting, ESLint boundaries |
| 21 | [Codebase Management, DX & Editor Mastery](./20-codebase-management.md) | 2,345 | Beginner to Intermediate | VS Code, extensions, keybindings, ESLint, Biome, Prettier, EditorConfig, git hooks, Husky, lint-staged, dependency updates, Renovate |
| 22 | [Full-Stack Caching Architecture — From Database to Pixels](./22-caching-architecture.md) | 3,568 | Advanced | Caching pyramid, Redis/Upstash, CDN edge cache, HTTP caching, TanStack Query, MMKV persistence, image CDN, cache warming, thundering herd, ISR, stale-while-revalidate |
| 23 | [Image & Content Optimization — Every Screen, Every Size](./23-image-content-optimization.md) | 4,014 | Intermediate to Advanced | Responsive images, expo-image, next/image, image CDN, Cloudinary, Imgix, WebP, AVIF, blurhash, thumbhash, responsive layouts, breakpoints, safe areas, adaptive components |
| 34 | [Database & ORM Patterns — The Server-Side Data Layer](./34-database-orm.md) | 4,192 | Intermediate to Advanced | Drizzle ORM, Prisma, Neon Postgres, PlanetScale, Turso, Supabase, database schema design, migrations, type-safe queries, connection pooling, serverless databases |
| 36 | [Error Handling Architecture — Graceful Failures Everywhere](./36-error-handling.md) | 2,708 | Intermediate | Error boundaries, API error standardization, user-facing error UX, retry patterns, graceful degradation, error reporting, toast notifications, timeout handling |
| 38 | [Search Implementation — From Input to Results](./38-search.md) | 3,501 | Intermediate | Algolia, Typesense, Meilisearch, full-text search, search UX, debounce, autocomplete, faceted search, recent searches, SQLite FTS, Postgres full-text search |
| 44 | [Performance Budgets & Web Vitals — Keeping Apps Fast Forever](./44-performance-budgets.md) | 2,854 | Intermediate | Performance budgets, Core Web Vitals, LCP, INP, CLS, Lighthouse, bundle size budgets, CI enforcement, performance regression detection, RUM vs synthetic |
| 47 | [UI/UX Paradigms — Making Apps Come to Life](./47-ux-paradigms.md) | 2,555 | Intermediate | Design thinking, mobile UX patterns, gesture-first design, bottom navigation, haptic feedback, skeleton screens, optimistic UI, progressive disclosure, dark mode, platform conventions |
| 48 | [Component Library Architecture — Building UI at Scale](./48-component-architecture.md) | 4,709 | Advanced | Atomic design, compound components, headless UI, Radix primitives, composition patterns, slot pattern, polymorphic components, forwardRef, generic components, Storybook driven development |

## Reading Order

1. **Ch 14** and **Ch 15** form a pair: performance optimization then profiling/debugging.
2. **Ch 16** (Monorepo) is foundational for the rest of the part -- read it before 17-19.
3. **Ch 17** (Design Systems) builds on the monorepo structure from Ch 16.
4. **Ch 18** (Testing) and **Ch 19** (CI/CD) are best read in sequence.
5. **Ch 20** (Architecture Patterns) is the most advanced -- read after absorbing 16-19.
6. **Ch 21** (Codebase Management) can be read at any point; it is the most beginner-friendly chapter here.
7. **Ch 22** (Caching Architecture) after Chapters 11, 16, 27 -- full-stack caching from DB to pixels.
8. **Ch 23** (Image & Content Optimization) after Chapters 9, 11 -- responsive images and content adaptation.
9. **Ch 34** (Database & ORM) after Chapters 11, 27, 28 -- the server-side data layer.
10. **Ch 36** (Error Handling) after Chapters 10, 20 -- graceful failure patterns across the stack.
11. **Ch 38** (Search) after Chapters 11, 34 -- search UX and backend integration.
12. **Ch 44** (Performance Budgets) after Chapters 13, 18, 25 -- enforcing performance in CI.
13. **Ch 47** (UI/UX Paradigms) after Chapters 9, 17 -- platform conventions and design patterns.
14. **Ch 48** (Component Architecture) after Chapters 17, 16 -- building component libraries at scale.

## Prerequisites

- **Ch 14**: Chapters 1, 3
- **Ch 15**: Chapters 1, 14
- **Ch 16**: Chapters 4, 5
- **Ch 17**: Chapter 16
- **Ch 18**: Chapters 5, 9
- **Ch 19**: Chapters 5, 6, 16
- **Ch 20**: Chapters 4, 16
- **Ch 21**: Chapters 0d, 16
- **Ch 22**: Chapters 11, 16, 27
- **Ch 23**: Chapters 9, 11
- **Ch 34**: Chapters 11, 27, 28
- **Ch 36**: Chapters 10, 20
- **Ch 38**: Chapters 11, 34
- **Ch 44**: Chapters 13, 18, 25
- **Ch 47**: Chapters 9, 17
- **Ch 48**: Chapters 17, 16

## What You'll Be Able to Do After This Part

- Profile and fix performance bottlenecks on real mobile devices (not just simulators).
- Set up a Turborepo + pnpm monorepo that shares code between React Native and Next.js.
- Build a cross-platform design system with design tokens, component variants, and accessibility baked in.
- Write a testing strategy that balances coverage, speed, and confidence using the testing trophy model.
- Design a CI/CD pipeline that builds only what changed, tests only what is affected, and deploys automatically.
- Choose the right frontend architecture pattern (monolith, microfrontends, BFF, islands) for your team size and product stage.
- Architect full-stack caching from database through CDN edge with cache invalidation strategies.
- Implement responsive images, content optimization, and adaptive layouts across every screen size.
- Design database schemas and type-safe ORM layers for serverless environments.
- Build robust error handling, search, and performance budget enforcement into your architecture.
