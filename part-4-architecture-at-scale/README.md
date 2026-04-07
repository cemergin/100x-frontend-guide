# Part IV — Architecture at Scale

> Monorepos, design systems, testing, CI/CD, and the patterns that let large teams move fast.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|-------|-----------|------------|
| [13](./13-mobile-performance.md) | Performance Optimization — Mobile | 2728 | Intermediate to Advanced | cold start, FlashList, virtualization, bundle size, memory management, React Compiler, re-renders, animations, Reanimated, image optimization |
| [14](./14-profiling-debugging.md) | Profiling & Debugging | 1430 | Intermediate to Advanced | Hermes profiling, Chrome DevTools for RN, Android Studio profiler, Xcode Instruments, Perfetto, Reactotron, memory leaks, crash analysis |
| [15](./15-monorepo.md) | Monorepo Architecture — Mobile + Web from One Repo | 2703 | Intermediate to Advanced | Turborepo, pnpm workspaces, monorepo structure, code sharing RN + Next.js, build ordering, remote caching, next-forge, dependency management |
| [16](./16-design-systems.md) | Design Systems & Component Libraries | 1601 | Intermediate | design tokens, shadcn/ui, Tamagui, NativeWind, Storybook, CVA, accessibility, compound components |
| [17](./17-testing.md) | Testing Strategy — What to Test, How to Test, When to Stop | 3561 | Intermediate | testing trophy, Jest, RNTL, Maestro, Playwright, MSW, testing hooks, snapshot testing, visual regression, performance testing |
| [18](./18-cicd.md) | CI/CD for Mobile & Web | 2778 | Intermediate to Advanced | GitHub Actions, EAS Build CI, preview deployments, performance budgets, semantic versioning, Changesets, Fastlane, self-hosted runners |
| [19](./19-fe-architecture.md) | Frontend Architecture Patterns | 2278 | Advanced | monoliths, microfrontends, module federation, BFF, island architecture, feature-based structure, architectural linting, ESLint boundaries |
| [20](./20-codebase-management.md) | Codebase Management, DX & Editor Mastery | 2334 | Beginner to Intermediate | VS Code, extensions, keybindings, ESLint, Biome, Prettier, EditorConfig, git hooks, Husky, lint-staged, dependency updates, Renovate |

## Reading Order

1. **Ch 13** and **Ch 14** form a pair: performance optimization then profiling/debugging.
2. **Ch 15** (Monorepo) is foundational for the rest of the part — read it before 16-18.
3. **Ch 16** (Design Systems) builds on the monorepo structure from Ch 15.
4. **Ch 17** (Testing) and **Ch 18** (CI/CD) are best read in sequence.
5. **Ch 19** (Architecture Patterns) is the most advanced — read after absorbing 15-18.
6. **Ch 20** (Codebase Management) can be read at any point; it is the most beginner-friendly chapter here.

## Prerequisites

- **Ch 13**: Chapters 1, 3
- **Ch 14**: Chapters 1, 13
- **Ch 15**: Chapters 4, 5
- **Ch 16**: Chapter 15
- **Ch 17**: Chapters 5, 9
- **Ch 18**: Chapters 5, 6, 15
- **Ch 19**: Chapters 4, 15
- **Ch 20**: Chapters 0d, 15

## What You'll Be Able to Do After This Part

- Profile and fix performance bottlenecks on real mobile devices (not just simulators).
- Set up a Turborepo + pnpm monorepo that shares code between React Native and Next.js.
- Build a cross-platform design system with design tokens, component variants, and accessibility baked in.
- Write a testing strategy that balances coverage, speed, and confidence using the testing trophy model.
- Design a CI/CD pipeline that builds only what changed, tests only what is affected, and deploys automatically.
- Choose the right frontend architecture pattern (monolith, microfrontends, BFF, islands) for your team size and product stage.
- Configure your editor, linter, formatter, and git hooks into a frictionless developer experience.
