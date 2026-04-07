<!--
  CHAPTER: 15
  TITLE: Monorepo Architecture — Mobile + Web from One Repo
  PART: IV — Architecture at Scale
  PREREQS: Chapters 4, 5
  KEY_TOPICS: Turborepo, pnpm workspaces, monorepo structure, code sharing RN + Next.js, build ordering, remote caching, next-forge, dependency management, catalogs
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 15: Monorepo Architecture — Mobile + Web from One Repo

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 4, 5 | Difficulty: Intermediate to Advanced

Here's a story I've seen play out at least a dozen times. A startup launches with a React Native app. Six months later, they need a web dashboard. Someone creates a new repo. A year later, there are four repos: mobile, web, API, and a "shared" package that nobody trusts because it's always out of date. The Zod schemas in the API repo don't match the types in the web repo. Someone renamed a field in the mobile repo and forgot to update the API client. The intern spent three days tracking down a bug that turned out to be a version mismatch between the validation library in mobile and web.

**This is what happens when you don't use a monorepo.** You pay a coordination tax on every change that touches more than one surface. And in a modern product company — where a single feature might require changes to the mobile app, the web app, the API, and the shared types — that's almost every change.

This chapter makes the case for monorepo architecture, then shows you exactly how to set one up with Turborepo and pnpm workspaces. We'll cover the folder structure, the build pipeline, code sharing between React Native and Next.js, dependency management, remote caching, and the real-world gotchas that documentation doesn't warn you about.

### In This Chapter
- The Case for Monorepos — why one repo is the right default
- Turborepo — task pipeline, caching, partial rebuilds
- pnpm Workspaces — the package manager that makes it practical
- The Monorepo Structure — a production-grade folder layout
- Sharing Code Between React Native and Next.js — what works, what doesn't
- Build Ordering and Dependencies — getting the dependency graph right
- next-forge — Vercel's production-grade monorepo starter
- Dependency Management — catalogs, phantom dependencies, version consistency
- Self-Hosted Turborepo Cache — for teams off Vercel

### Related Chapters
- [Ch 4: TypeScript Patterns] — the type system that makes shared code safe
- [Ch 5: Expo Platform] — how Expo works within a monorepo
- [Ch 9: State Management] — shared state patterns across platforms
- [Ch 10: Data Fetching] — API client generation and sharing
- [Ch 16: CI/CD Pipeline] — automating builds in a monorepo

---

## 1. THE CASE FOR MONOREPOS

Let me be direct: **for most teams building mobile + web products, a monorepo is the right default.** Not because it's trendy (Google, Meta, Microsoft, and Uber use them), but because the alternative — multiple repos with manual coordination — creates a category of problems that only gets worse as your team and product grow.

### 1.1 The Multi-Repo Tax

Here's what life looks like with separate repos for mobile, web, and API:

**Scenario: Add a `middleName` field to user profiles.**

In a multi-repo world:
1. Update the API schema in the API repo. Open a PR. Wait for review. Merge.
2. Publish a new version of the API types package to npm. Wait for CI.
3. Update the web repo to bump the types package. Update the form. Open a PR. Wait for review. Merge.
4. Update the mobile repo to bump the types package. Update the form. Open a PR. Wait for review. Merge.
5. Pray that all three PRs land in the right order and nobody merges something in between.

Total time: 2-5 days. PRs reviewed: 3-4. Risk of version mismatch: high. Risk of partial deployment (API deployed before clients are ready): real.

In a monorepo:
1. Update the Zod schema in `packages/shared`. Update the API handler, the web form, and the mobile form. Open one PR.
2. Reviewers see the entire change in context. CI runs type checking across all affected packages. Everything deploys together.

Total time: hours. PR reviewed: 1. Risk of mismatch: zero — the types are the same file.

**That's the atomic change advantage.** In a monorepo, a change that spans multiple packages is a single commit, a single PR, a single review. There's no version to bump, no package to publish, no downstream repo to update.

### 1.2 Shared Types Are the Killer Feature

This is the single biggest win, and I want to be explicit about why.

When your mobile app and your web app consume the same API, they need to agree on the shape of the data. In a multi-repo world, you keep these in sync one of three ways:

1. **Copy-paste the types.** Works for a week. Then they diverge.
2. **Publish a shared types package to npm.** Works until someone forgets to bump the version, or two PRs land that change the types in incompatible ways, or the published version is three versions behind the actual API.
3. **Generate types from an OpenAPI spec.** Better, but you still need to regenerate and update in each repo, and the generation step adds friction.

In a monorepo, the types live in `packages/shared/src/types.ts`. Every app imports from `@repo/shared`. When the types change, TypeScript immediately catches every consumer that needs to update. Not eventually. Not after a publish. **Immediately, in the same `tsc` run.**

```typescript
// packages/shared/src/schemas/user.ts
import { z } from 'zod';

export const userSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  firstName: z.string().min(1),
  middleName: z.string().optional(), // Add this once...
  lastName: z.string().min(1),
  role: z.enum(['admin', 'member', 'viewer']),
  createdAt: z.string().datetime(),
});

export type User = z.infer<typeof userSchema>;

// packages/shared/src/schemas/index.ts
export { userSchema, type User } from './user';
```

Now every app — mobile, web, API — imports from the same source:

```typescript
// apps/web/src/app/profile/page.tsx
import { userSchema, type User } from '@repo/shared/schemas';

// apps/mobile/src/screens/ProfileScreen.tsx
import { userSchema, type User } from '@repo/shared/schemas';

// apps/api/src/routes/users.ts
import { userSchema, type User } from '@repo/shared/schemas';
```

One schema. One type. Three consumers. Zero drift.

### 1.3 When a Monorepo Is the Wrong Choice

I said monorepos are the right *default*, not the right choice for every situation. They're the wrong choice when:

- **Your projects are genuinely independent.** If your company has a billing service written in Go, a machine learning pipeline in Python, and a React Native app, forcing them into one repo creates coupling where none exists. A monorepo should contain things that change together.
- **Your team is very large and can't agree on tooling.** Google's monorepo works because they invested millions in custom tooling (Blaze/Bazel, Critique, Piper). If you have 500 engineers and no monorepo tooling budget, you'll have a bad time.
- **You're using languages with fundamentally different build systems.** A JavaScript/TypeScript monorepo with Turborepo is elegant. A monorepo that contains Rust, Python, Go, and TypeScript needs Bazel or Nx, and that's a different level of complexity.

For the 95% of teams building products with React Native + Next.js + a Node/TypeScript API? Monorepo. Every time.

### 1.4 What Companies Actually Do

| Company | Approach | Stack | Why |
|---------|----------|-------|-----|
| **Vercel** | Monorepo (Turborepo) | Next.js, shared packages | They literally built Turborepo for this |
| **Shopify** | Monorepo | React Native, web, internal tools | Atomic changes across mobile + web |
| **Coinbase** | Monorepo | React Native, Next.js | Shared crypto types, validation |
| **Airbnb** | Monorepo (historical) | Web packages | Shared component library |
| **Expo** | Monorepo | React Native, web tools | Their own SDK packages |
| **Cal.com** | Monorepo (Turborepo) | Next.js, packages | Open source, great reference |
| **Meta** | Monorepo | Everything | The original monorepo at scale |

The trend is clear. Teams that ship across multiple surfaces converge on monorepos because the coordination cost of multi-repo is a tax that compounds with every feature, every engineer, and every platform.

---

## 2. TURBOREPO

Turborepo is a build system for JavaScript/TypeScript monorepos. It understands your package dependency graph, runs tasks in the correct order, caches results, and skips work that hasn't changed. It was created by Jared Palmer, acquired by Vercel in 2021, and has become the default choice for TypeScript monorepos.

**Why not Nx?** Nx is the other major player. It's more feature-rich (project graph visualization, code generators, plugin ecosystem) but also more complex. For a TypeScript-first team that wants to get a monorepo running in an afternoon, Turborepo is the right choice. If you're managing 200+ packages with multiple languages and need advanced affected-project analysis, Nx might be worth the complexity. For most teams reading this book? Turborepo.

### 2.1 The Task Pipeline

Turborepo's core concept is a **task pipeline** defined in `turbo.json`. You tell Turborepo what tasks exist, what their dependencies are, and what their inputs and outputs are. Turborepo figures out the rest.

Here's a production-grade `turbo.json`:

```jsonc
// turbo.json (at the repo root)
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [
    "**/.env.*local",
    ".env"
  ],
  "globalPassThroughEnv": [
    "NODE_ENV",
    "VERCEL_URL",
    "CI"
  ],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": [
        "src/**",
        "tsconfig.json",
        "package.json",
        "next.config.*",
        "tailwind.config.*"
      ],
      "outputs": [
        ".next/**",
        "!.next/cache/**",
        "dist/**",
        "build/**"
      ]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"],
      "inputs": [
        "src/**",
        "eslint.config.*",
        "tsconfig.json"
      ]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "inputs": [
        "src/**",
        "tsconfig.json",
        "package.json"
      ]
    },
    "test": {
      "dependsOn": ["^build"],
      "inputs": [
        "src/**",
        "__tests__/**",
        "vitest.config.*",
        "jest.config.*"
      ],
      "outputs": ["coverage/**"]
    },
    "clean": {
      "cache": false
    }
  }
}
```

Let me break down the key concepts here:

### 2.2 The `^` Operator (Topological Dependencies)

The `^` in `"dependsOn": ["^build"]` is the most important character in the entire file. It means: **before running this task in a package, first run `build` in all of that package's dependencies.**

Here's why this matters. Say your dependency graph looks like this:

```
apps/web → packages/ui → packages/shared
apps/mobile → packages/ui → packages/shared
```

When you run `turbo run build`:

1. Turborepo sees that `apps/web` depends on `packages/ui`, which depends on `packages/shared`.
2. It builds `packages/shared` first (no dependencies).
3. Then `packages/ui` (depends on `packages/shared`, which is now built).
4. Then `apps/web` and `apps/mobile` in parallel (both depend on `packages/ui`, which is now built).

Without the `^`, Turborepo would try to build everything in parallel, and `packages/ui` would fail because `packages/shared` hasn't produced its output yet.

```
Build Order (topological):

Step 1: packages/shared     (no deps)
Step 2: packages/ui         (depends on shared)
Step 3: apps/web ┐          (both depend on ui)
        apps/mobile ┘       (run in parallel)
```

### 2.3 Inputs and Outputs — The Cache Contract

Turborepo's caching is based on a simple contract: **given the same inputs, a task will produce the same outputs.**

**Inputs** are the files that, when changed, should invalidate the cache:
- `src/**` — your source code
- `tsconfig.json` — TypeScript configuration affects compilation
- `package.json` — dependency changes affect the build
- `next.config.*` — framework config affects the build output

**Outputs** are the files that the task produces:
- `.next/**` — Next.js build output
- `dist/**` — compiled library output
- `coverage/**` — test coverage reports

When you run `turbo run build`, Turborepo:
1. Hashes all the inputs (file contents, env vars, dependency versions)
2. Checks if it has a cached output for that exact hash
3. If yes: skips the build entirely and restores the cached output
4. If no: runs the build, then caches the output keyed by the hash

**This is why builds get fast.** If you only changed `apps/web/src/app/page.tsx`, Turborepo will:
- Skip `packages/shared` (inputs unchanged, restore from cache)
- Skip `packages/ui` (inputs unchanged, restore from cache)
- Skip `apps/mobile` (inputs unchanged, restore from cache)
- Only build `apps/web` (input changed)

On a monorepo with 10 packages, this can turn a 5-minute build into a 30-second build.

### 2.4 Cache Behavior Explained

```
$ turbo run build

• Packages in scope: @repo/api, @repo/mobile, @repo/shared, @repo/ui, @repo/web
• Running build in 5 packages
• Remote caching enabled

 @repo/shared:build  cache hit, replaying logs ████████████ 0.2s
 @repo/ui:build      cache hit, replaying logs ████████████ 0.3s
 @repo/mobile:build  cache hit, replaying logs ████████████ 0.1s
 @repo/api:build     cache hit, replaying logs ████████████ 0.2s
 @repo/web:build     cache miss, executing     ████████████ 12.4s

 Tasks:    5 successful, 5 total
 Cached:   4 cached, 5 total
   Time:   13.2s
```

Four of five packages were restored from cache. Only the one that actually changed needed to rebuild. That's Turborepo's value proposition in a single terminal output.

### 2.5 Environment Variables and Cache Safety

One of the most common Turborepo footguns is environment variables. If your build output depends on an env var (like `NEXT_PUBLIC_API_URL`), but that env var isn't listed in the inputs, Turborepo might serve a cached build from staging when you're building for production.

There are three levels of env var configuration:

```jsonc
{
  // Global: affect ALL tasks across ALL packages
  "globalPassThroughEnv": ["CI", "NODE_ENV"],
  "globalDependencies": [".env"],

  "tasks": {
    "build": {
      // Task-level: affect this task's cache key
      "env": [
        "NEXT_PUBLIC_API_URL",
        "NEXT_PUBLIC_STRIPE_KEY",
        "DATABASE_URL"
      ],
      // Pass-through: available but don't affect cache
      "passThroughEnv": [
        "TERM",
        "PATH"
      ]
    }
  }
}
```

**Rule of thumb:** If an env var affects the build output, put it in `env`. If it's needed at runtime but doesn't affect the compiled output, put it in `passThroughEnv`. If you're not sure, put it in `env` — a cache miss is better than a stale cache hit.

### 2.6 Filtering — Run Tasks for Specific Packages

You don't always want to build everything. Turborepo's `--filter` flag lets you target specific packages:

```bash
# Build only the web app (and its dependencies)
turbo run build --filter=@repo/web

# Build only things that changed since main
turbo run build --filter=...[main]

# Build everything in the apps/ directory
turbo run build --filter="./apps/*"

# Build the web app and everything it depends on
turbo run build --filter=@repo/web...

# Build everything that depends on the shared package
turbo run build --filter=...@repo/shared

# Lint only the packages that changed in this PR
turbo run lint --filter="[HEAD^1]"
```

The filter syntax is powerful once you learn it:

| Filter | Meaning |
|--------|---------|
| `--filter=@repo/web` | Only the web package |
| `--filter=@repo/web...` | Web and all its dependencies (downstream) |
| `--filter=...@repo/shared` | Shared and everything that depends on it (upstream) |
| `--filter="[main]"` | Packages changed since main branch |
| `--filter="./apps/*"` | All packages in the apps directory |

This is critical for CI. You don't want to rebuild everything on every PR — you want to rebuild the things affected by the changes in the PR.

### 2.7 The Commands You'll Run Daily

Here's what a typical developer's day looks like with Turborepo:

```bash
# Start all dev servers in parallel
turbo run dev

# Build everything (with caching)
turbo run build

# Run lint + typecheck across the entire repo
turbo run lint typecheck

# Run tests for packages affected by your changes
turbo run test --filter="[main]"

# Build only the web app for deployment
turbo run build --filter=@repo/web

# Clean all build artifacts
turbo run clean

# See the task graph (what depends on what)
turbo run build --graph

# Dry run — see what would run without running it
turbo run build --dry-run
```

---

## 3. PNPM WORKSPACES

If Turborepo is the brain of your monorepo (figuring out what to build and in what order), pnpm is the skeleton (managing dependencies, linking packages, and keeping `node_modules` sane).

**Why pnpm over npm or yarn?**

The short answer: pnpm is the only package manager that makes monorepos genuinely practical. npm workspaces are basic and error-prone. Yarn Berry (PnP) works but has compatibility issues with React Native. pnpm gives you strict dependency isolation, efficient disk usage, and the `workspace:` protocol — all of which matter enormously in a monorepo.

### 3.1 Content-Addressed Storage

Every other package manager copies packages into each project's `node_modules`. If you have 10 packages that all use `lodash@4.17.21`, npm puts 10 copies of lodash on disk. pnpm stores one copy in a global content-addressed store and creates hard links from each `node_modules` to that single copy.

```
# npm: 10 copies of lodash across your monorepo
apps/web/node_modules/lodash/          → 1.4 MB
apps/mobile/node_modules/lodash/       → 1.4 MB
packages/ui/node_modules/lodash/       → 1.4 MB
packages/shared/node_modules/lodash/   → 1.4 MB
...
Total: 14 MB

# pnpm: 1 copy, hard-linked everywhere
~/.pnpm-store/lodash@4.17.21/         → 1.4 MB (single copy)
apps/web/node_modules/.pnpm/lodash@4.17.21/  → hard link
apps/mobile/node_modules/.pnpm/lodash@4.17.21/ → hard link
...
Total: 1.4 MB + negligible hard links
```

On a monorepo with 15 packages, this can save gigabytes of disk space and make `pnpm install` significantly faster than `npm install`.

### 3.2 Strict Dependency Isolation

This is the feature that prevents a class of bugs that will ruin your afternoon.

npm and yarn use a flat `node_modules` structure. This means that if `packages/ui` depends on `react`, and `apps/web` depends on `packages/ui`, then `apps/web` can accidentally import `react` from `packages/ui`'s `node_modules` — even if `apps/web` didn't declare `react` as its own dependency. This is called a **phantom dependency.**

pnpm uses a nested `node_modules` structure where each package can only access its own declared dependencies. If `apps/web` doesn't list `zod` in its `package.json`, it can't import `zod` — even if another package in the monorepo has it installed.

```
# pnpm's node_modules structure (simplified)
apps/web/node_modules/
  @repo/shared → symlink to packages/shared
  @repo/ui → symlink to packages/ui
  react → hard link to store
  next → hard link to store
  # zod is NOT here, even though @repo/shared has it
  # apps/web must declare its own dependency on zod to use it

packages/shared/node_modules/
  zod → hard link to store
  # Only packages/shared can import zod
```

**Why this matters:** Phantom dependencies cause builds that work on your machine but fail in CI. They cause "works in development but crashes in production" bugs. They make it impossible to know what a package actually depends on by reading its `package.json`. pnpm eliminates this entire class of problems.

### 3.3 Workspace Configuration

The `pnpm-workspace.yaml` file tells pnpm where your packages live:

```yaml
# pnpm-workspace.yaml (at the repo root)
packages:
  - "apps/*"
  - "packages/*"
```

That's it. pnpm will discover every directory under `apps/` and `packages/` that contains a `package.json` and treat it as a workspace package.

### 3.4 The `workspace:` Protocol

When one package in your monorepo depends on another, you use the `workspace:` protocol:

```jsonc
// apps/web/package.json
{
  "name": "@repo/web",
  "dependencies": {
    "@repo/shared": "workspace:*",
    "@repo/ui": "workspace:*",
    "next": "^15.3.0",
    "react": "^19.1.0"
  }
}
```

`"workspace:*"` means "use whatever version of `@repo/shared` exists in this monorepo." pnpm resolves this to a symlink — `apps/web/node_modules/@repo/shared` points directly to `packages/shared`. No publishing. No version bumping. Changes in `packages/shared` are immediately visible to `apps/web`.

You can also use `workspace:^` or `workspace:~` if you want to enforce semver ranges, but for most internal monorepos, `workspace:*` is the right choice.

### 3.5 pnpm Catalogs — Version Management at Scale

This is a feature that landed in pnpm 9.5 and it's a game-changer for monorepos. **Catalogs** let you define dependency versions in one place and reference them across all packages.

The problem they solve: in a monorepo with 10 packages, you might have `react@19.1.0` declared in 8 different `package.json` files. When you want to upgrade React, you need to update all 8 files. Inevitably, someone misses one, and you end up with two versions of React in your bundle.

Catalogs fix this:

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"

catalog:
  react: ^19.1.0
  react-dom: ^19.1.0
  react-native: ^0.78.0
  typescript: ^5.8.0
  zod: ^3.24.0
  next: ^15.3.0
  "@tanstack/react-query": ^5.75.0
  vitest: ^3.1.0
  tailwindcss: ^4.1.0
  eslint: ^9.25.0
```

Then in each package's `package.json`, reference the catalog:

```jsonc
// apps/web/package.json
{
  "name": "@repo/web",
  "dependencies": {
    "@repo/shared": "workspace:*",
    "@repo/ui": "workspace:*",
    "next": "catalog:",
    "react": "catalog:",
    "react-dom": "catalog:",
    "zod": "catalog:",
    "@tanstack/react-query": "catalog:"
  },
  "devDependencies": {
    "typescript": "catalog:",
    "vitest": "catalog:",
    "eslint": "catalog:"
  }
}
```

```jsonc
// apps/mobile/package.json
{
  "name": "@repo/mobile",
  "dependencies": {
    "@repo/shared": "workspace:*",
    "@repo/ui": "workspace:*",
    "react": "catalog:",
    "react-native": "catalog:",
    "zod": "catalog:",
    "@tanstack/react-query": "catalog:"
  },
  "devDependencies": {
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

**One source of truth for every dependency version.** When you upgrade React, you change one line in `pnpm-workspace.yaml`. Every package that uses `catalog:` picks up the new version on the next `pnpm install`.

You can also define named catalogs for different purposes:

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"

# Default catalog
catalog:
  react: ^19.1.0
  typescript: ^5.8.0

# Named catalogs for specific concerns
catalogs:
  testing:
    vitest: ^3.1.0
    "@testing-library/react": ^16.3.0
    "@testing-library/react-native": ^13.2.0
  linting:
    eslint: ^9.25.0
    prettier: ^3.5.0
```

Reference named catalogs with `catalog:testing` or `catalog:linting`:

```jsonc
{
  "devDependencies": {
    "vitest": "catalog:testing",
    "eslint": "catalog:linting"
  }
}
```

### 3.6 Common pnpm Commands in a Monorepo

```bash
# Install all dependencies across all workspaces
pnpm install

# Add a dependency to a specific package
pnpm add zod --filter @repo/shared

# Add a dev dependency to a specific package
pnpm add -D vitest --filter @repo/web

# Add a dependency to the root (for repo-level tooling)
pnpm add -D turbo -w

# Remove a dependency from a specific package
pnpm remove lodash --filter @repo/shared

# Run a script in a specific package
pnpm --filter @repo/web dev

# Run a script in all packages that have it
pnpm -r run build

# Update all packages to latest versions matching ranges
pnpm update -r

# See why a package is installed (dependency tree)
pnpm why react --filter @repo/web

# List all workspace packages
pnpm ls --depth -1 -r
```

---

## 4. THE MONOREPO STRUCTURE

Here's the folder structure I recommend for a production React Native + Next.js monorepo. This isn't theoretical — it's based on real structures used by teams shipping to millions of users.

```
my-app/
├── apps/
│   ├── mobile/                    # React Native / Expo app
│   │   ├── app/                   # Expo Router file-based routing
│   │   │   ├── (tabs)/
│   │   │   │   ├── index.tsx
│   │   │   │   ├── profile.tsx
│   │   │   │   └── settings.tsx
│   │   │   ├── _layout.tsx
│   │   │   └── +not-found.tsx
│   │   ├── src/
│   │   │   ├── components/        # Mobile-only components
│   │   │   ├── hooks/             # Mobile-only hooks
│   │   │   ├── lib/               # Mobile-specific utilities
│   │   │   └── styles/            # Mobile-specific styles
│   │   ├── assets/
│   │   ├── app.json
│   │   ├── babel.config.js
│   │   ├── metro.config.js
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── web/                       # Next.js app
│   │   ├── src/
│   │   │   ├── app/               # App Router
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── profile/
│   │   │   │   └── settings/
│   │   │   ├── components/        # Web-only components
│   │   │   ├── hooks/             # Web-only hooks
│   │   │   └── lib/               # Web-specific utilities
│   │   ├── public/
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── api/                       # API server (optional)
│       ├── src/
│       │   ├── routes/
│       │   ├── middleware/
│       │   ├── services/
│       │   └── index.ts
│       ├── tsconfig.json
│       └── package.json
│
├── packages/
│   ├── ui/                        # Cross-platform UI components
│   │   ├── src/
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── card.tsx
│   │   │   ├── text.tsx
│   │   │   └── index.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── shared/                    # Business logic, types, validation
│   │   ├── src/
│   │   │   ├── schemas/           # Zod schemas (shared validation)
│   │   │   │   ├── user.ts
│   │   │   │   ├── product.ts
│   │   │   │   ├── order.ts
│   │   │   │   └── index.ts
│   │   │   ├── types/             # TypeScript types
│   │   │   │   ├── api.ts
│   │   │   │   ├── domain.ts
│   │   │   │   └── index.ts
│   │   │   ├── utils/             # Pure utility functions
│   │   │   │   ├── format.ts
│   │   │   │   ├── math.ts
│   │   │   │   ├── date.ts
│   │   │   │   └── index.ts
│   │   │   ├── constants/
│   │   │   │   └── index.ts
│   │   │   └── index.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── api-client/                # Generated/typed API client
│   │   ├── src/
│   │   │   ├── client.ts          # Fetch wrapper or tRPC client
│   │   │   ├── routes.ts          # Route definitions
│   │   │   └── index.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── config-eslint/             # Shared ESLint configuration
│   │   ├── base.js
│   │   ├── react.js
│   │   ├── next.js
│   │   ├── react-native.js
│   │   └── package.json
│   │
│   └── config-ts/                 # Shared TypeScript configuration
│       ├── base.json
│       ├── react-library.json
│       ├── nextjs.json
│       ├── react-native.json
│       └── package.json
│
├── turbo.json                     # Turborepo configuration
├── pnpm-workspace.yaml            # Workspace definition
├── package.json                   # Root package.json
├── pnpm-lock.yaml
├── .gitignore
├── .npmrc
└── .env.example
```

### 4.1 The Root package.json

The root `package.json` is the entry point. It doesn't contain application code — it's for repo-level tooling and scripts.

```jsonc
// package.json (repo root)
{
  "name": "my-app",
  "private": true,
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "dev:web": "turbo run dev --filter=@repo/web",
    "dev:mobile": "turbo run dev --filter=@repo/mobile",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "test:affected": "turbo run test --filter='[main]'",
    "clean": "turbo run clean",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\"",
    "format:check": "prettier --check \"**/*.{ts,tsx,js,jsx,json,md}\""
  },
  "devDependencies": {
    "turbo": "^2.5.0",
    "prettier": "^3.5.0"
  },
  "packageManager": "pnpm@9.15.0",
  "engines": {
    "node": ">=20"
  }
}
```

Note the `"packageManager"` field — this is a Corepack feature that ensures everyone on the team uses the same version of pnpm. Combined with `engines`, it prevents the "works on my machine" class of problems caused by tool version mismatches.

### 4.2 The .npmrc File

```ini
# .npmrc
# Hoist packages that need to be at the root
# (some tools don't work well with pnpm's strict structure)
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*

# Don't hoist everything — keep strict isolation
shamefully-hoist=false

# Use the lockfile
frozen-lockfile=true

# Prefer workspace packages
prefer-workspace-packages=true
```

The `public-hoist-pattern` is important. Some tools (ESLint, Prettier) expect their plugins to be at the root `node_modules`. pnpm's strict isolation would otherwise prevent them from finding their plugins.

### 4.3 Package Configuration: packages/shared

Let's look at how an internal package is configured:

```jsonc
// packages/shared/package.json
{
  "name": "@repo/shared",
  "version": "0.0.0",
  "private": true,
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./schemas": {
      "types": "./src/schemas/index.ts",
      "default": "./src/schemas/index.ts"
    },
    "./utils": {
      "types": "./src/utils/index.ts",
      "default": "./src/utils/index.ts"
    },
    "./types": {
      "types": "./src/types/index.ts",
      "default": "./src/types/index.ts"
    }
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/",
    "test": "vitest run",
    "clean": "rm -rf dist node_modules .turbo"
  },
  "dependencies": {
    "zod": "catalog:"
  },
  "devDependencies": {
    "@repo/config-ts": "workspace:*",
    "typescript": "catalog:",
    "vitest": "catalog:"
  }
}
```

**Key detail: no build step.** Notice that the `exports` field points to TypeScript source files (`./src/index.ts`), not compiled output. For internal packages in a monorepo, you often don't need to compile them separately — the consuming app's bundler (Next.js's webpack/turbopack, Metro for React Native) handles the compilation.

This is a deliberate choice. It eliminates an entire build step for internal packages, making the dev loop faster. The consuming apps already have TypeScript compilation in their pipeline.

For packages that DO need a build step (if they're published to npm, or if the consuming bundler can't handle raw TypeScript), you'd use `tsup`:

```jsonc
// packages/ui/package.json (if it needs compilation)
{
  "name": "@repo/ui",
  "version": "0.0.0",
  "private": true,
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/",
    "clean": "rm -rf dist node_modules .turbo"
  }
}
```

### 4.4 TypeScript Configuration Sharing

Shared TypeScript configs prevent drift and ensure consistent compiler behavior:

```jsonc
// packages/config-ts/base.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "bundler",
    "module": "esnext",
    "target": "es2022",
    "lib": ["es2022"],
    "resolveJsonModule": true,
    "isolatedModules": true,
    "incremental": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "exclude": ["node_modules", "dist", "build", ".turbo"]
}

// packages/config-ts/nextjs.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["dom", "dom.iterable", "es2022"],
    "jsx": "preserve",
    "module": "esnext",
    "moduleResolution": "bundler",
    "plugins": [{ "name": "next" }],
    "allowJs": true
  }
}

// packages/config-ts/react-native.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["es2022"],
    "jsx": "react-jsx",
    "module": "esnext",
    "moduleResolution": "bundler",
    "types": ["react-native", "jest"]
  }
}

// packages/config-ts/react-library.json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "extends": "./base.json",
  "compilerOptions": {
    "lib": ["es2022"],
    "jsx": "react-jsx"
  }
}
```

Then each app extends the appropriate config:

```jsonc
// apps/web/tsconfig.json
{
  "extends": "@repo/config-ts/nextjs.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}

// apps/mobile/tsconfig.json
{
  "extends": "@repo/config-ts/react-native.json",
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}

// packages/shared/tsconfig.json
{
  "extends": "@repo/config-ts/react-library.json",
  "compilerOptions": {
    "baseUrl": ".",
    "outDir": "./dist"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### 4.5 ESLint Configuration Sharing

```javascript
// packages/config-eslint/base.js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import prettierConfig from 'eslint-config-prettier';

/** @type {import("eslint").Linter.Config[]} */
export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  prettierConfig,
  {
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/no-explicit-any': 'warn',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
  },
  {
    ignores: ['dist/', 'build/', '.next/', '.turbo/', 'node_modules/'],
  },
];

// packages/config-eslint/react.js
import baseConfig from './base.js';
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';

/** @type {import("eslint").Linter.Config[]} */
export default [
  ...baseConfig,
  {
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooksPlugin,
    },
    rules: {
      ...reactPlugin.configs.recommended.rules,
      ...reactHooksPlugin.configs.recommended.rules,
      'react/react-in-jsx-scope': 'off',
      'react/prop-types': 'off',
    },
    settings: {
      react: { version: 'detect' },
    },
  },
];

// packages/config-eslint/next.js
import reactConfig from './react.js';
import nextPlugin from '@next/eslint-plugin-next';

/** @type {import("eslint").Linter.Config[]} */
export default [
  ...reactConfig,
  {
    plugins: {
      '@next/next': nextPlugin,
    },
    rules: {
      ...nextPlugin.configs.recommended.rules,
      ...nextPlugin.configs['core-web-vitals'].rules,
    },
  },
];
```

---

## 5. SHARING CODE BETWEEN REACT NATIVE AND NEXT.JS

This is where monorepos go from "nice organizational pattern" to "actual productivity multiplier." But you need to be precise about what can and cannot be shared, or you'll create packages that break in ways that are confusing and hard to debug.

### 5.1 The Sharing Spectrum

Not all code is equally shareable. Here's the complete taxonomy:

```
┌────────────────────────────────────────────────────────────┐
│  FULLY SHAREABLE (100% platform-agnostic)                  │
│                                                            │
│  ✅ TypeScript types and interfaces                        │
│  ✅ Zod schemas and validation logic                       │
│  ✅ Pure utility functions (formatCurrency, parseDate)     │
│  ✅ Business logic (calculateTax, validateOrder)           │
│  ✅ Constants and enums                                    │
│  ✅ API client logic (fetch wrappers, tRPC client)         │
│  ✅ Zustand/Jotai stores (state logic without UI)          │
│  ✅ TanStack Query hooks (data fetching)                   │
│  ✅ Math, string, and array utilities                      │
│  ✅ Date formatting (using Intl or date-fns)               │
├────────────────────────────────────────────────────────────┤
│  PARTIALLY SHAREABLE (shared interface, different impl)    │
│                                                            │
│  ⚠️  UI components (shared props, different rendering)     │
│  ⚠️  Storage (AsyncStorage vs localStorage — same API)     │
│  ⚠️  Analytics (same events, different SDK)                │
│  ⚠️  Authentication (same flow, different storage)         │
│  ⚠️  Haptics/feedback (web vibration vs native haptics)    │
├────────────────────────────────────────────────────────────┤
│  NOT SHAREABLE (platform-specific by nature)               │
│                                                            │
│  ❌ Navigation (Expo Router vs Next.js routing)            │
│  ❌ Native modules (camera, biometrics, NFC)               │
│  ❌ Push notifications (APNS/FCM vs Web Push)              │
│  ❌ Styling implementation (StyleSheet vs CSS/Tailwind)    │
│  ❌ File system access (different APIs per platform)       │
│  ❌ Deep linking (different schemes per platform)          │
│  ❌ App lifecycle (backgrounding, foregrounding)           │
│  ❌ Web-specific APIs (document, window, CSS animations)   │
│  ❌ Native-specific APIs (Animated, LayoutAnimation)       │
└────────────────────────────────────────────────────────────┘
```

### 5.2 The packages/shared Pattern

The `packages/shared` directory is where your fully shareable code lives. The rule is simple: **nothing in this package can import from `react-native`, `next`, or any platform-specific API.**

```typescript
// packages/shared/src/schemas/order.ts
import { z } from 'zod';
import { userSchema } from './user';

export const orderItemSchema = z.object({
  productId: z.string().uuid(),
  name: z.string(),
  quantity: z.number().int().positive(),
  unitPrice: z.number().positive(),
  currency: z.enum(['USD', 'EUR', 'GBP']),
});

export const orderSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  items: z.array(orderItemSchema).min(1),
  status: z.enum(['pending', 'confirmed', 'shipped', 'delivered', 'cancelled']),
  shippingAddress: z.object({
    street: z.string(),
    city: z.string(),
    state: z.string(),
    zip: z.string(),
    country: z.string().length(2), // ISO 3166-1 alpha-2
  }),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export type Order = z.infer<typeof orderSchema>;
export type OrderItem = z.infer<typeof orderItemSchema>;
export type OrderStatus = Order['status'];

// Pure business logic — no platform code
export function calculateOrderTotal(items: OrderItem[]): number {
  return items.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0);
}

export function canCancelOrder(order: Order): boolean {
  return order.status === 'pending' || order.status === 'confirmed';
}

export function getOrderStatusLabel(status: OrderStatus): string {
  const labels: Record<OrderStatus, string> = {
    pending: 'Pending',
    confirmed: 'Confirmed',
    shipped: 'Shipped',
    delivered: 'Delivered',
    cancelled: 'Cancelled',
  };
  return labels[status];
}
```

```typescript
// packages/shared/src/utils/format.ts

export function formatCurrency(
  amount: number,
  currency: string = 'USD',
  locale: string = 'en-US'
): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
}

export function formatRelativeTime(date: Date | string): string {
  const d = typeof date === 'string' ? new Date(date) : date;
  const now = new Date();
  const diffMs = now.getTime() - d.getTime();
  const diffMins = Math.floor(diffMs / 60000);
  const diffHours = Math.floor(diffMs / 3600000);
  const diffDays = Math.floor(diffMs / 86400000);

  if (diffMins < 1) return 'just now';
  if (diffMins < 60) return `${diffMins}m ago`;
  if (diffHours < 24) return `${diffHours}h ago`;
  if (diffDays < 7) return `${diffDays}d ago`;

  return d.toLocaleDateString();
}

export function truncate(str: string, maxLength: number): string {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength - 3) + '...';
}

export function slugify(str: string): string {
  return str
    .toLowerCase()
    .trim()
    .replace(/[^\w\s-]/g, '')
    .replace(/[\s_-]+/g, '-')
    .replace(/^-+|-+$/g, '');
}
```

### 5.3 The packages/api-client Pattern

The API client is another highly shareable package. Whether you're using tRPC, REST with typed fetch, or a generated client, the calling code is the same on mobile and web.

```typescript
// packages/api-client/src/client.ts
import { type Order, type User, orderSchema, userSchema } from '@repo/shared/schemas';

type RequestConfig = {
  baseUrl: string;
  getToken: () => Promise<string | null>;
};

let config: RequestConfig | null = null;

export function initApiClient(cfg: RequestConfig) {
  config = cfg;
}

async function request<T>(
  path: string,
  options: RequestInit = {},
  schema?: { parse: (data: unknown) => T }
): Promise<T> {
  if (!config) throw new Error('API client not initialized. Call initApiClient first.');

  const token = await config.getToken();

  const response = await fetch(`${config.baseUrl}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ message: 'Unknown error' }));
    throw new ApiError(response.status, error.message ?? 'Request failed');
  }

  const data = await response.json();
  return schema ? schema.parse(data) : (data as T);
}

export class ApiError extends Error {
  constructor(
    public status: number,
    message: string
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// Typed API methods — same on mobile and web
export const api = {
  users: {
    me: () => request<User>('/api/users/me', {}, userSchema),
    update: (data: Partial<User>) =>
      request<User>('/api/users/me', {
        method: 'PATCH',
        body: JSON.stringify(data),
      }, userSchema),
  },
  orders: {
    list: () => request<Order[]>('/api/orders'),
    get: (id: string) => request<Order>(`/api/orders/${id}`, {}, orderSchema),
    create: (data: Omit<Order, 'id' | 'createdAt' | 'updatedAt'>) =>
      request<Order>('/api/orders', {
        method: 'POST',
        body: JSON.stringify(data),
      }, orderSchema),
    cancel: (id: string) =>
      request<Order>(`/api/orders/${id}/cancel`, { method: 'POST' }, orderSchema),
  },
};
```

Then in each app, you initialize with platform-specific config:

```typescript
// apps/web/src/lib/api.ts
import { initApiClient } from '@repo/api-client';
import { getSession } from 'next-auth/react';

initApiClient({
  baseUrl: process.env.NEXT_PUBLIC_API_URL!,
  getToken: async () => {
    const session = await getSession();
    return session?.accessToken ?? null;
  },
});

export { api } from '@repo/api-client';

// apps/mobile/src/lib/api.ts
import { initApiClient } from '@repo/api-client';
import * as SecureStore from 'expo-secure-store';

initApiClient({
  baseUrl: process.env.EXPO_PUBLIC_API_URL!,
  getToken: async () => {
    return SecureStore.getItemAsync('auth_token');
  },
});

export { api } from '@repo/api-client';
```

Same API client. Same type safety. Different auth mechanisms.

### 5.4 Shared React Hooks (Platform-Agnostic)

Hooks that don't touch the DOM or native APIs can be shared:

```typescript
// packages/shared/src/hooks/use-debounce.ts
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);

  return debouncedValue;
}

// packages/shared/src/hooks/use-order.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { api } from '@repo/api-client';
import type { Order } from '@repo/shared/schemas';

export function useOrder(orderId: string) {
  return useQuery({
    queryKey: ['order', orderId],
    queryFn: () => api.orders.get(orderId),
  });
}

export function useOrders() {
  return useQuery({
    queryKey: ['orders'],
    queryFn: () => api.orders.list(),
  });
}

export function useCancelOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (orderId: string) => api.orders.cancel(orderId),
    onSuccess: (updatedOrder) => {
      queryClient.setQueryData(['order', updatedOrder.id], updatedOrder);
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    },
  });
}
```

These hooks work identically on mobile and web because React, TanStack Query, and the API client are all platform-agnostic.

### 5.5 The UI Package — Where It Gets Tricky

The `packages/ui` package is where most teams struggle. You want shared components, but React Native and Next.js have fundamentally different rendering targets.

There are three approaches, ranked from simplest to most ambitious:

**Approach 1: Shared Props, Separate Implementations (Recommended)**

Define the component interface in the shared package, implement separately:

```typescript
// packages/ui/src/button.tsx
// This works if your bundler resolves platform extensions
// (.native.tsx for RN, .web.tsx for web)

// button.types.ts — shared props
export interface ButtonProps {
  label: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  disabled?: boolean;
  loading?: boolean;
}

// button.web.tsx — web implementation
import type { ButtonProps } from './button.types';

export function Button({
  label,
  onPress,
  variant = 'primary',
  size = 'md',
  disabled = false,
  loading = false,
}: ButtonProps) {
  return (
    <button
      onClick={onPress}
      disabled={disabled || loading}
      className={cn(
        'rounded-lg font-medium transition-colors',
        variants[variant],
        sizes[size],
        disabled && 'opacity-50 cursor-not-allowed'
      )}
    >
      {loading ? <Spinner /> : label}
    </button>
  );
}

// button.native.tsx — React Native implementation
import { Pressable, Text, ActivityIndicator, StyleSheet } from 'react-native';
import type { ButtonProps } from './button.types';

export function Button({
  label,
  onPress,
  variant = 'primary',
  size = 'md',
  disabled = false,
  loading = false,
}: ButtonProps) {
  return (
    <Pressable
      onPress={onPress}
      disabled={disabled || loading}
      style={({ pressed }) => [
        styles.base,
        variantStyles[variant],
        sizeStyles[size],
        (disabled || loading) && styles.disabled,
        pressed && styles.pressed,
      ]}
    >
      {loading ? (
        <ActivityIndicator color="#fff" />
      ) : (
        <Text style={[styles.label, variantTextStyles[variant]]}>
          {label}
        </Text>
      )}
    </Pressable>
  );
}
```

**Approach 2: React Native Web (Unified)**

Use React Native Web to render RN components in the browser. This works but comes with trade-offs:

```typescript
// packages/ui/src/button.tsx
// Single implementation using React Native primitives
import { Pressable, Text, ActivityIndicator, StyleSheet } from 'react-native';

export interface ButtonProps {
  label: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary' | 'ghost';
  disabled?: boolean;
  loading?: boolean;
}

export function Button({
  label,
  onPress,
  variant = 'primary',
  disabled = false,
  loading = false,
}: ButtonProps) {
  return (
    <Pressable
      onPress={onPress}
      disabled={disabled || loading}
      style={({ pressed }) => [
        styles.base,
        variantStyles[variant],
        pressed && styles.pressed,
      ]}
    >
      {loading ? (
        <ActivityIndicator color="#fff" />
      ) : (
        <Text style={[styles.label, labelStyles[variant]]}>
          {label}
        </Text>
      )}
    </Pressable>
  );
}

const styles = StyleSheet.create({
  base: {
    paddingHorizontal: 16,
    paddingVertical: 10,
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
  },
  pressed: { opacity: 0.8 },
  label: { fontSize: 16, fontWeight: '600' },
});
```

The trade-off: you lose access to web-native features (CSS animations, Tailwind, HTML semantics). Your web app renders `<div>` elements styled with inline styles instead of semantic HTML. For internal tools, that's fine. For a consumer-facing web app where SEO, accessibility, and web performance matter? Approach 1 is better.

**Approach 3: Tamagui / Gluestack / NativeWind (Cross-Platform Styling)**

These libraries provide a unified styling system that compiles to native styles on mobile and CSS on web:

```typescript
// Using NativeWind (Tailwind for React Native)
import { View, Text, Pressable } from 'react-native';

export function Button({ label, onPress, variant = 'primary' }: ButtonProps) {
  return (
    <Pressable
      onPress={onPress}
      className={cn(
        'flex-row items-center justify-center rounded-lg px-4 py-2.5',
        variant === 'primary' && 'bg-blue-600 active:bg-blue-700',
        variant === 'secondary' && 'bg-gray-200 active:bg-gray-300',
      )}
    >
      <Text
        className={cn(
          'text-base font-semibold',
          variant === 'primary' && 'text-white',
          variant === 'secondary' && 'text-gray-900',
        )}
      >
        {label}
      </Text>
    </Pressable>
  );
}
```

NativeWind compiles Tailwind classes to React Native `StyleSheet` objects on mobile and to actual CSS on web. It's the closest to "write once, run everywhere" for styling.

### 5.6 Metro Configuration for Monorepo Packages

React Native's Metro bundler needs to know about your monorepo structure:

```javascript
// apps/mobile/metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

// Find the project and workspace directories
const projectRoot = __dirname;
const monorepoRoot = path.resolve(projectRoot, '../..');

const config = getDefaultConfig(projectRoot);

// 1. Watch all files within the monorepo
config.watchFolders = [monorepoRoot];

// 2. Let Metro know where to resolve packages and in what order
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(monorepoRoot, 'node_modules'),
];

// 3. Force Metro to resolve (sub)dependencies only from the `nodeModulesPaths`
config.resolver.disableHierarchicalLookup = true;

module.exports = config;
```

Without this configuration, Metro won't find packages in the `packages/` directory because it only looks in the app's own `node_modules` by default.

### 5.7 Next.js Configuration for Monorepo Packages

Next.js needs to know to transpile your internal packages:

```typescript
// apps/web/next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Transpile internal packages
  transpilePackages: [
    '@repo/shared',
    '@repo/ui',
    '@repo/api-client',
  ],

  // If using React Native Web (Approach 2)
  // webpack: (config) => {
  //   config.resolve.alias = {
  //     ...(config.resolve.alias || {}),
  //     'react-native$': 'react-native-web',
  //   };
  //   return config;
  // },
};

export default nextConfig;
```

The `transpilePackages` array tells Next.js to run these packages through its compiler instead of treating them as pre-built node modules. This is what allows you to use raw TypeScript source files in your internal packages.

---

## 6. BUILD ORDERING AND DEPENDENCIES

Getting the dependency graph right is the difference between a monorepo that "just works" and one that breaks in CI every other PR. Turborepo is only as smart as the dependency information you give it.

### 6.1 How Turborepo Resolves Build Order

Turborepo reads every `package.json` in your workspace and builds a dependency graph from the `dependencies` and `devDependencies` fields. When a task has `"dependsOn": ["^build"]`, Turborepo traverses this graph to determine the order.

Here's a concrete example. Given these package.json files:

```jsonc
// packages/shared/package.json
{ "name": "@repo/shared", "dependencies": { "zod": "catalog:" } }

// packages/ui/package.json
{ "name": "@repo/ui", "dependencies": { "@repo/shared": "workspace:*", "react": "catalog:" } }

// packages/api-client/package.json
{ "name": "@repo/api-client", "dependencies": { "@repo/shared": "workspace:*" } }

// apps/web/package.json
{ "name": "@repo/web", "dependencies": { "@repo/ui": "workspace:*", "@repo/api-client": "workspace:*", "@repo/shared": "workspace:*" } }

// apps/mobile/package.json
{ "name": "@repo/mobile", "dependencies": { "@repo/ui": "workspace:*", "@repo/api-client": "workspace:*", "@repo/shared": "workspace:*" } }
```

Turborepo constructs this graph:

```
                    @repo/shared
                   /      |       \
            @repo/ui  @repo/api-client
                |    \     |      /
                |     \    |     /
              @repo/web  @repo/mobile
```

Build order for `turbo run build`:

```
Level 0 (no deps):   @repo/shared
Level 1 (dep on 0):  @repo/ui, @repo/api-client    (parallel)
Level 2 (dep on 1):  @repo/web, @repo/mobile        (parallel)
```

Turborepo maximizes parallelism. Packages at the same level run concurrently. This is why it's faster than running `pnpm -r run build`, which runs sequentially.

### 6.2 The Most Common Mistake: Missing Dependencies

If `packages/ui` uses types from `@repo/shared` but doesn't declare it as a dependency in `package.json`, Turborepo won't know to build `packages/shared` first. The build will work locally (because pnpm hoists things in a way that makes it accidentally available) but fail in CI or when cached outputs are involved.

**Rule: If package A imports from package B, package A MUST list package B in its dependencies.**

```jsonc
// ❌ BAD: packages/ui uses @repo/shared but doesn't declare it
{
  "name": "@repo/ui",
  "dependencies": {
    "react": "catalog:"
    // @repo/shared is imported in code but missing here!
  }
}

// ✅ GOOD: explicitly declare the dependency
{
  "name": "@repo/ui",
  "dependencies": {
    "@repo/shared": "workspace:*",
    "react": "catalog:"
  }
}
```

### 6.3 Circular Dependencies

Turborepo will error if it detects a circular dependency. If `packages/ui` depends on `packages/shared` and `packages/shared` depends on `packages/ui`, there's no valid build order.

The fix is almost always to extract the shared code into a third package:

```
Before (circular):
  packages/ui ←→ packages/shared

After (resolved):
  packages/types (no internal deps)
  packages/ui → packages/types
  packages/shared → packages/types
```

### 6.4 Internal Packages vs. Published Packages

In a monorepo, you have two kinds of packages:

**Internal packages** (`"private": true`) are consumed only within the monorepo. They use `workspace:*` and don't need to be published. They can export raw TypeScript. They don't need a build step if consumers compile them.

**Published packages** are packages you distribute on npm for external consumption. They need a build step, proper `exports` fields pointing to compiled output, `types` fields, and semantic versioning.

Most teams should start with all internal packages. Only publish when you have external consumers.

```jsonc
// Internal package — simple, no build needed
{
  "name": "@repo/shared",
  "version": "0.0.0",
  "private": true,
  "exports": {
    ".": "./src/index.ts"
  }
}

// Published package — needs build step, proper exports
{
  "name": "@acme/design-system",
  "version": "2.4.1",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.js"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts"
  }
}
```

### 6.5 Visualizing the Dependency Graph

Turborepo can generate a visual graph of your task dependencies:

```bash
# Generate a graph visualization (opens in browser)
turbo run build --graph

# Generate a DOT file for custom rendering
turbo run build --graph=graph.dot

# Generate a JSON representation
turbo run build --graph=graph.json
```

This is invaluable for debugging build order issues. If a package is building before its dependencies, the graph will show you why.

---

## 7. NEXT-FORGE

next-forge is Vercel's production-grade Turborepo monorepo starter. It's not a framework — it's an opinionated starting point that includes the tooling, configuration, and patterns that most SaaS applications need. Think of it as "create-next-app but for serious monorepos."

### 7.1 What It Includes

When you run `npx next-forge init`, you get:

```
my-app/
├── apps/
│   ├── app/              # Main Next.js application
│   ├── web/              # Marketing site (Next.js)
│   ├── api/              # API routes (Hono on Vercel Functions)
│   └── email/            # Email templates (React Email)
├── packages/
│   ├── analytics/        # Vercel Analytics + custom events
│   ├── auth/             # Better Auth / Clerk integration
│   ├── cms/              # Basehub CMS integration
│   ├── database/         # Prisma + Neon Postgres
│   ├── design-system/    # shadcn/ui components
│   ├── email/            # React Email templates
│   ├── feature-flags/    # Feature flag management
│   ├── logging/          # Structured logging
│   ├── observability/    # Sentry error tracking
│   ├── payments/         # Stripe integration
│   ├── security/         # Arcjet rate limiting + bot protection
│   ├── seo/              # Meta tags, sitemap, robots.txt
│   ├── notifications/    # Knock notification system
│   └── webhooks/         # Webhook handling
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

### 7.2 When to Use next-forge

**Use next-forge when:**
- You're building a SaaS product with Next.js
- You want production-grade infrastructure from day one (auth, payments, database, analytics, error tracking)
- You're comfortable with Vercel's ecosystem (it's optimized for Vercel deployment)
- You want a reference architecture for how to structure a monorepo

**Don't use next-forge when:**
- You're building a React Native + Next.js app (it doesn't include mobile)
- You want to self-host everything (it's Vercel-native)
- You need a minimal starting point (it includes a LOT of integrations)
- Your stack doesn't align (you use Firebase instead of Prisma, for example)

### 7.3 What You Can Learn from next-forge

Even if you don't use next-forge directly, it's a masterclass in monorepo architecture. Here's what to study:

**Package boundaries.** Each integration (auth, payments, analytics) is its own package with a clean API. The app imports from `@repo/auth`, not directly from `better-auth`. This means you can swap auth providers without touching app code.

```typescript
// packages/auth/src/index.ts — clean abstraction
export { auth, signIn, signOut } from './server';
export { useSession, useUser } from './client';

// The app doesn't know or care that this uses Better Auth
// If you switch to Clerk, you change this file, not the app
```

**Environment variable management.** next-forge uses a pattern where each package declares its required env vars, and a root-level check validates that all required vars are set:

```typescript
// packages/database/src/env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
```

**Shared configuration.** TypeScript configs, ESLint configs, and Tailwind configs are all shared packages — exactly the pattern we described in Section 4.

### 7.4 Extending next-forge for Mobile

If you want to use next-forge AND add a React Native app, the approach is:

1. Start with `npx next-forge init`
2. Add `apps/mobile/` with an Expo project
3. Move shared types and validation to a `packages/shared` package
4. Configure the mobile app to consume shared packages (Metro config, tsconfig)
5. Add `react-native` to the pnpm catalog

The shared packages from next-forge (types, validation, API client) work as-is on mobile. The platform-specific packages (design-system, SEO, CMS) stay web-only.

---

## 8. DEPENDENCY MANAGEMENT

Dependency management in a monorepo is where most teams experience the most friction — and where the right patterns save the most time. The core challenge: you have N packages, each with their own `package.json`, and you need to keep their dependencies consistent, up-to-date, and conflict-free.

### 8.1 The Phantom Dependency Problem

I mentioned this briefly in the pnpm section, but it deserves a deeper explanation because it's one of the most insidious bugs in JavaScript monorepos.

**What it is:** A phantom dependency is a package that your code imports but doesn't declare in its `package.json`. It works because another package in the monorepo (or the package manager's hoisting) makes it available in `node_modules`.

**How it happens:**

```typescript
// packages/ui/src/button.tsx
import { clsx } from 'clsx'; // packages/ui doesn't depend on clsx!

// This works because apps/web depends on clsx,
// and npm/yarn hoists clsx to the root node_modules.
```

**Why it's dangerous:**
1. The build works locally but fails in CI (different hoisting)
2. If `apps/web` removes `clsx`, `packages/ui` suddenly breaks
3. You can't deploy `packages/ui` independently because its actual dependencies aren't declared
4. Turborepo's caching might not invalidate correctly because the dependency isn't tracked

**How pnpm solves it:** pnpm's non-flat `node_modules` structure means each package can only access its own declared dependencies. If `packages/ui` doesn't list `clsx` in its `package.json`, importing `clsx` will fail with a module-not-found error. This is annoying the first time it happens, but it forces correct dependency declarations and prevents phantom dependency bugs.

### 8.2 Version Consistency

Even without phantom dependencies, version inconsistency is a problem. If `apps/web` uses `zod@3.23.0` and `apps/mobile` uses `zod@3.24.0`, you might get subtle runtime differences or duplicate packages in your bundle.

**Solution 1: pnpm Catalogs (Recommended)**

As shown in Section 3.5, define versions in `pnpm-workspace.yaml` and use `catalog:` in package.json files. One source of truth.

**Solution 2: syncpack**

If you can't use catalogs (older pnpm version), `syncpack` is a tool that checks and fixes version mismatches:

```bash
# Install syncpack
pnpm add -D syncpack -w

# Check for mismatches
npx syncpack list-mismatches

# Fix mismatches (set all to the highest version)
npx syncpack fix-mismatches
```

```jsonc
// .syncpackrc.json
{
  "versionGroups": [
    {
      "label": "Use workspace protocol for internal packages",
      "dependencies": ["@repo/*"],
      "dependencyTypes": ["dev", "prod"],
      "pinVersion": "workspace:*"
    },
    {
      "label": "React versions must match",
      "dependencies": ["react", "react-dom"],
      "policy": "same"
    }
  ]
}
```

### 8.3 Updating Dependencies

Updating dependencies across a monorepo needs to be deliberate:

```bash
# Check for outdated packages across all workspaces
pnpm outdated -r

# Update a specific package everywhere
pnpm update react -r

# Update all packages to latest within their semver range
pnpm update -r

# Interactive update (shows each package and lets you choose)
pnpm update -r --interactive

# Update a catalog entry (edit pnpm-workspace.yaml, then)
pnpm install
```

**Best practice:** Automate dependency updates with Renovate or Dependabot, configured to group related packages:

```jsonc
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "description": "Group React packages",
      "matchPackageNames": ["react", "react-dom", "@types/react", "@types/react-dom"],
      "groupName": "react"
    },
    {
      "description": "Group React Native packages",
      "matchPackageNames": ["react-native", "expo", "expo-*"],
      "groupName": "react-native"
    },
    {
      "description": "Group Next.js packages",
      "matchPackageNames": ["next", "@next/*", "eslint-config-next"],
      "groupName": "nextjs"
    },
    {
      "description": "Group TypeScript packages",
      "matchPackageNames": ["typescript", "ts-node", "tsup", "tslib"],
      "groupName": "typescript"
    }
  ]
}
```

### 8.4 Peer Dependencies in a Monorepo

Peer dependencies are particularly tricky in monorepos. If `packages/ui` has React as a peer dependency, each app that uses `packages/ui` must provide its own React.

```jsonc
// packages/ui/package.json
{
  "name": "@repo/ui",
  "peerDependencies": {
    "react": "^19.0.0",
    "react-native": "^0.78.0"
  },
  "peerDependenciesMeta": {
    "react-native": {
      "optional": true
    }
  },
  "devDependencies": {
    "react": "catalog:",
    "react-native": "catalog:"
  }
}
```

Note that `react-native` is marked as optional — the web app doesn't have it, and that's fine. The `devDependencies` include the packages so that development, testing, and type-checking work within `packages/ui` itself.

### 8.5 The Lock File

One more critical point: **commit your lock file.** The `pnpm-lock.yaml` is what ensures reproducible installs. Without it, `pnpm install` might resolve to different versions on different machines or in CI.

```gitignore
# .gitignore — do NOT ignore the lock file
# pnpm-lock.yaml  ← Don't put this here!

# DO ignore node_modules
node_modules/

# Turbo cache
.turbo/

# Build outputs
dist/
build/
.next/
```

Run `pnpm install --frozen-lockfile` in CI to ensure the lock file is respected and catch any discrepancies.

---

## 9. REMOTE CACHING

Local caching is great — Turborepo saves time on your machine. But the real productivity unlock is **remote caching**, where cache artifacts are shared across your entire team and CI pipeline.

Here's the scenario: you push a PR. CI runs `turbo run build`. All packages build from scratch. Your teammate pulls the same branch and runs `turbo run build`. Without remote caching, all packages build from scratch again. With remote caching, they get the cached outputs from CI instantly.

### 9.1 Vercel Remote Cache

If you deploy on Vercel, remote caching is built in and free:

```bash
# Login to Vercel (one time)
npx turbo login

# Link your repo to Vercel (one time)
npx turbo link

# That's it. Builds are now cached remotely.
turbo run build
```

After linking, every `turbo run` checks the remote cache before building. If anyone on your team (or CI) has already built that exact combination of inputs, you get the cached output.

The numbers are impressive. From Vercel's own data: remote caching typically reduces CI build times by 40-80%, and local build times by 60-90% (because most builds are cache hits for unchanged packages).

### 9.2 Self-Hosted Turborepo Cache

Not every team uses Vercel, and not every team wants to send build artifacts to a third-party service. Turborepo supports custom remote cache endpoints — you just need a server that implements the cache API.

**Option 1: turborepo-remote-cache (Open Source)**

The most popular self-hosted option is `turborepo-remote-cache`, a Node.js server that stores cache artifacts in S3, GCS, or local disk:

```bash
# Clone and deploy the cache server
git clone https://github.com/ducktors/turborepo-remote-cache.git
cd turborepo-remote-cache

# Configure with environment variables
export STORAGE_PROVIDER=s3
export S3_ACCESS_KEY=your-access-key
export S3_SECRET_KEY=your-secret-key
export S3_REGION=us-east-1
export S3_BUCKET=turborepo-cache
export TURBO_TOKEN=your-secret-token

# Run with Docker
docker build -t turborepo-cache .
docker run -p 3000:3000 turborepo-cache
```

Then configure your monorepo to use it:

```jsonc
// .turbo/config.json (or set via CLI)
{
  "teamId": "team_my-team",
  "apiUrl": "https://cache.your-domain.com"
}
```

Or set environment variables in CI:

```bash
export TURBO_API="https://cache.your-domain.com"
export TURBO_TOKEN="your-secret-token"
export TURBO_TEAM="team_my-team"

turbo run build
```

**Option 2: Docker Compose for Local Teams**

For teams that want a quick self-hosted setup:

```yaml
# docker-compose.yml
version: '3.8'

services:
  turborepo-cache:
    image: ghcr.io/ducktors/turborepo-remote-cache:latest
    ports:
      - "3000:3000"
    environment:
      - STORAGE_PROVIDER=local
      - STORAGE_PATH=/cache
      - TURBO_TOKEN=${TURBO_TOKEN}
    volumes:
      - cache-data:/cache
    restart: unless-stopped

volumes:
  cache-data:
```

**Option 3: S3-Compatible Object Storage**

If you're already running MinIO, R2 (Cloudflare), or DigitalOcean Spaces, you can point `turborepo-remote-cache` at any S3-compatible endpoint:

```bash
STORAGE_PROVIDER=s3
S3_ENDPOINT=https://your-minio-instance.com  # or R2, DO Spaces, etc.
S3_ACCESS_KEY=your-key
S3_SECRET_KEY=your-secret
S3_BUCKET=turborepo-cache
S3_REGION=auto
```

### 9.3 Cache Security Considerations

Remote caching means build artifacts leave your machine. Be aware of:

- **Secrets in build output.** If your build inlines API keys or secrets into the bundle, those will be in the cache. Use runtime environment variables instead of build-time injection where possible.
- **Access control.** Ensure only your team can read/write the cache. Use the `TURBO_TOKEN` for authentication.
- **Cache poisoning.** If someone pushes a compromised cache artifact, downstream builds will use it. Self-hosted solutions should run on trusted infrastructure.
- **Artifact size.** Cache artifacts can be large (hundreds of MB for Next.js builds). Monitor your storage costs.

### 9.4 Disabling Cache for Specific Tasks

Some tasks shouldn't be cached:

```jsonc
{
  "tasks": {
    "dev": {
      "cache": false,         // Dev servers are persistent, not cacheable
      "persistent": true
    },
    "deploy": {
      "cache": false         // Deployments should always execute
    },
    "db:migrate": {
      "cache": false         // Database migrations must always run
    },
    "lint": {
      "cache": true          // Linting is deterministic — cache it
    }
  }
}
```

---

## 10. PUTTING IT ALL TOGETHER: THE COMPLETE SETUP

Let me walk you through setting up a complete monorepo from scratch. This is the recipe I recommend for teams starting a new project with React Native and Next.js.

### 10.1 Initial Setup

```bash
# Create the monorepo structure
mkdir my-app && cd my-app
git init

# Initialize pnpm
pnpm init

# Create the workspace configuration
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - "apps/*"
  - "packages/*"

catalog:
  react: ^19.1.0
  react-dom: ^19.1.0
  react-native: ^0.78.0
  typescript: ^5.8.0
  zod: ^3.24.0
  next: ^15.3.0
  "@tanstack/react-query": ^5.75.0
  vitest: ^3.1.0
  eslint: ^9.25.0
EOF

# Create the directory structure
mkdir -p apps/{web,mobile} packages/{shared,ui,api-client,config-ts,config-eslint}

# Install Turborepo at the root
pnpm add -D turbo -w

# Create turbo.json (use the configuration from Section 2.1)

# Create .npmrc
cat > .npmrc << 'EOF'
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*
shamefully-hoist=false
prefer-workspace-packages=true
EOF

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
.turbo/
dist/
build/
.next/
.expo/
*.tsbuildinfo
.env*.local
.env
EOF
```

### 10.2 Initialize Each App

```bash
# Web app (Next.js)
cd apps/web
pnpm create next-app . --typescript --tailwind --eslint --app --src-dir

# Mobile app (Expo)
cd ../mobile
npx create-expo-app . --template blank-typescript

# Go back to root
cd ../..
```

### 10.3 Connect the Packages

Update each app's `package.json` to reference workspace packages:

```bash
# Add workspace dependencies to the web app
pnpm add @repo/shared@workspace:* @repo/ui@workspace:* @repo/api-client@workspace:* --filter @repo/web

# Add workspace dependencies to the mobile app
pnpm add @repo/shared@workspace:* @repo/ui@workspace:* @repo/api-client@workspace:* --filter @repo/mobile
```

### 10.4 Verify Everything Works

```bash
# Install all dependencies
pnpm install

# Type check everything
turbo run typecheck

# Build everything
turbo run build

# Start dev servers
turbo run dev
```

### 10.5 CI Configuration (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2  # Needed for turbo's change detection

      - name: Setup pnpm
        uses: pnpm/action-setup@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: turbo run lint

      - name: Type check
        run: turbo run typecheck

      - name: Test
        run: turbo run test

      - name: Build
        run: turbo run build
        env:
          TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
          TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

---

## 11. COMMON PITFALLS AND HOW TO AVOID THEM

After working with dozens of teams on monorepo setups, here are the problems that come up over and over:

### 11.1 "It Works Locally but Fails in CI"

**Cause:** Phantom dependencies. Your code imports something that's not in its `package.json`, and it works locally because pnpm hoists it (or it's available from another package's `node_modules`).

**Fix:** Use `pnpm --strict-peer-dependencies` in CI. Ensure every import has a corresponding `package.json` entry.

### 11.2 "Turbo Keeps Rebuilding Everything"

**Cause:** Your `inputs` in `turbo.json` are too broad. If you include `**/*` as an input, changing a README will invalidate the build cache.

**Fix:** Be specific about inputs. Only include files that actually affect the build output:

```jsonc
// ❌ Too broad
"inputs": ["**/*"]

// ✅ Specific
"inputs": ["src/**", "tsconfig.json", "package.json"]
```

### 11.3 "Two Versions of React in the Bundle"

**Cause:** Different packages depend on different React versions, or a package doesn't use `workspace:*` for an internal dependency and accidentally installs a published version.

**Fix:** Use pnpm catalogs to enforce a single version. Run `pnpm why react` to find duplicate installations. Ensure internal packages use `workspace:*`.

### 11.4 "Metro Can't Find My Monorepo Packages"

**Cause:** Metro's default resolver doesn't look outside the app directory.

**Fix:** Configure `watchFolders` and `nodeModulesPaths` in `metro.config.js` (see Section 5.6).

### 11.5 "TypeScript Can't Resolve Internal Package Types"

**Cause:** The `exports` field in the internal package doesn't include a `types` condition, or the consuming app's `tsconfig.json` doesn't use `"moduleResolution": "bundler"`.

**Fix:**

```jsonc
// packages/shared/package.json
{
  "exports": {
    ".": {
      "types": "./src/index.ts",   // ← Must include this
      "default": "./src/index.ts"
    }
  }
}

// apps/web/tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "bundler"   // ← Must be "bundler", not "node"
  }
}
```

### 11.6 "CI Is Slow Because It Installs All Dependencies"

**Fix:** Use pnpm's `--filter` with `--frozen-lockfile` and caching:

```yaml
# Cache pnpm store
- uses: actions/cache@v4
  with:
    path: ~/.pnpm-store
    key: pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: pnpm-

# Cache turbo outputs
- uses: actions/cache@v4
  with:
    path: .turbo
    key: turbo-${{ github.sha }}
    restore-keys: turbo-
```

### 11.7 "My IDE Is Slow in the Monorepo"

**Cause:** TypeScript language server is indexing the entire monorepo.

**Fix:** Use project references and the `"references"` field in `tsconfig.json`:

```jsonc
// apps/web/tsconfig.json
{
  "extends": "@repo/config-ts/nextjs.json",
  "references": [
    { "path": "../../packages/shared" },
    { "path": "../../packages/ui" }
  ]
}
```

Also add a root-level `tsconfig.json` for the IDE:

```jsonc
// tsconfig.json (repo root — for IDE only, not for building)
{
  "files": [],
  "references": [
    { "path": "apps/web" },
    { "path": "apps/mobile" },
    { "path": "packages/shared" },
    { "path": "packages/ui" },
    { "path": "packages/api-client" }
  ]
}
```

---

## 12. THE DECISION FRAMEWORK

When should you use what? Here's the decision matrix:

### Monorepo vs. Multi-Repo

| Factor | Monorepo | Multi-Repo |
|--------|----------|------------|
| Team size < 30 | Recommended | Overhead not justified |
| Mobile + Web + API | Strong fit | Coordination tax too high |
| Shared types/validation | Critical advantage | Manual sync required |
| Independent services (different languages) | Bad fit | Natural fit |
| Open source library + internal app | Depends | Often cleaner |
| Startup velocity | Faster iteration | Slower coordination |

### Turborepo vs. Nx vs. Bazel

| Factor | Turborepo | Nx | Bazel |
|--------|-----------|-----|-------|
| Setup complexity | 10 min | 30 min | Days |
| TypeScript-first | Yes | Yes | No (generic) |
| Package count < 50 | Sweet spot | Works | Overkill |
| Package count > 200 | Starts to strain | Better | Designed for this |
| Code generation | No | Yes (generators) | Yes (rules) |
| Multi-language | No (JS/TS only) | Plugin-based | Yes (any language) |
| Remote caching | Vercel or self-host | Nx Cloud or self-host | Built-in |
| Learning curve | Low | Medium | High |

### pnpm vs. npm vs. yarn

| Factor | pnpm | npm | Yarn (Berry) |
|--------|------|-----|--------------|
| Workspace support | Excellent | Basic | Excellent |
| Strict isolation | Yes | No | Yes (PnP) |
| Catalogs | Yes | No | No (has constraints) |
| Disk efficiency | Best (content-addressed) | Worst (copies) | Good (PnP) |
| RN compatibility | Excellent | Excellent | Requires workarounds |
| Speed | Fastest | Slowest | Fast |
| Ecosystem support | Mainstream | Universal | Some compatibility issues |

---

## 13. REAL-WORLD MONOREPO PATTERNS

Let me close with some patterns that experienced teams use but rarely document.

### 13.1 The Feature Package Pattern

Instead of organizing by technical layer (ui, shared, api-client), some teams organize by feature:

```
packages/
  feature-auth/        # Everything for authentication
    src/
      schemas.ts       # Auth-related Zod schemas
      hooks.ts         # useAuth, useSession
      api.ts           # Auth API calls
      types.ts         # Auth types
  feature-payments/    # Everything for payments
    src/
      schemas.ts
      hooks.ts
      api.ts
      types.ts
  feature-orders/
    ...
```

**When this works:** Teams organized around product areas (squad model). Each squad owns a feature package and the corresponding screens in the apps.

**When this doesn't work:** When features have high coupling. If orders depend on auth and payments, you're back to a tangled dependency graph.

### 13.2 The Platform Abstraction Pattern

For code that needs to work differently on each platform but present the same API:

```typescript
// packages/platform/src/storage.ts
export interface StorageAdapter {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
  remove(key: string): Promise<void>;
}

// packages/platform/src/storage.web.ts
export const storage: StorageAdapter = {
  get: async (key) => localStorage.getItem(key),
  set: async (key, value) => localStorage.setItem(key, value),
  remove: async (key) => localStorage.removeItem(key),
};

// packages/platform/src/storage.native.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

export const storage: StorageAdapter = {
  get: (key) => AsyncStorage.getItem(key),
  set: (key, value) => AsyncStorage.setItem(key, value),
  remove: (key) => AsyncStorage.removeItem(key),
};
```

### 13.3 The Generator Pattern

As your monorepo grows, creating new packages becomes repetitive. Use a generator script:

```bash
#!/bin/bash
# scripts/new-package.sh

PACKAGE_NAME=$1

if [ -z "$PACKAGE_NAME" ]; then
  echo "Usage: ./scripts/new-package.sh <package-name>"
  exit 1
fi

mkdir -p "packages/$PACKAGE_NAME/src"

cat > "packages/$PACKAGE_NAME/package.json" << EOF
{
  "name": "@repo/$PACKAGE_NAME",
  "version": "0.0.0",
  "private": true,
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    }
  },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/",
    "test": "vitest run",
    "clean": "rm -rf dist node_modules .turbo"
  },
  "devDependencies": {
    "@repo/config-ts": "workspace:*",
    "typescript": "catalog:"
  }
}
EOF

cat > "packages/$PACKAGE_NAME/tsconfig.json" << EOF
{
  "extends": "@repo/config-ts/react-library.json",
  "compilerOptions": {
    "baseUrl": ".",
    "outDir": "./dist"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
EOF

cat > "packages/$PACKAGE_NAME/src/index.ts" << EOF
// @repo/$PACKAGE_NAME
export {};
EOF

echo "Created packages/$PACKAGE_NAME"
echo "Run 'pnpm install' to link the new package"
```

### 13.4 Testing in a Monorepo

Each package should be independently testable:

```typescript
// packages/shared/src/utils/__tests__/format.test.ts
import { describe, it, expect } from 'vitest';
import { formatCurrency, truncate, slugify } from '../format';

describe('formatCurrency', () => {
  it('formats USD correctly', () => {
    expect(formatCurrency(42.5, 'USD')).toBe('$42.50');
  });

  it('formats EUR correctly', () => {
    expect(formatCurrency(42.5, 'EUR', 'de-DE')).toBe('42,50\u00a0\u20ac');
  });
});

describe('truncate', () => {
  it('does not truncate short strings', () => {
    expect(truncate('hello', 10)).toBe('hello');
  });

  it('truncates long strings with ellipsis', () => {
    expect(truncate('hello world', 8)).toBe('hello...');
  });
});

describe('slugify', () => {
  it('converts to URL-safe slug', () => {
    expect(slugify('Hello World!')).toBe('hello-world');
  });
});
```

Run tests across the entire monorepo:

```bash
# Test everything
turbo run test

# Test only affected packages
turbo run test --filter="[main]"

# Test a specific package
turbo run test --filter=@repo/shared

# Watch mode for a specific package (during development)
pnpm --filter @repo/shared test -- --watch
```

---

## CHAPTER SUMMARY

The monorepo with Turborepo and pnpm is the right default for teams building across mobile and web. Not because it's fashionable, but because it eliminates the coordination overhead that slows teams down as they scale.

**Key takeaways:**

1. **Monorepo = atomic changes.** One PR can update the Zod schema, the API handler, the web form, and the mobile screen. No version bumping, no publishing, no downstream repo to update.

2. **Turborepo handles the hard parts.** Task ordering, caching, partial rebuilds, remote caching. Configure it once in `turbo.json` and it keeps getting faster as your cache hit rate improves.

3. **pnpm makes it safe.** Strict dependency isolation prevents phantom dependencies. The `workspace:` protocol links internal packages without publishing. Catalogs keep versions consistent.

4. **Share what's shareable, don't force what's not.** Types, schemas, validation, business logic, API clients, and TanStack Query hooks are fully shareable. Navigation, styling implementation, and native modules are not. Be precise about the boundary.

5. **Internal packages don't need build steps.** Point `exports` to TypeScript source and let the consuming app's bundler compile. One fewer thing to maintain, one faster dev loop.

6. **Get the dependency graph right.** Every import needs a corresponding `package.json` entry. Turborepo can only optimize what it can see.

7. **Remote caching is the multiplier.** Local caching saves you minutes. Remote caching saves your entire team hours every day.

The tools are mature. The patterns are proven. The companies that ship fastest across mobile and web have converged on this architecture not because someone on Hacker News said it was cool, but because it's the simplest way to maintain consistency and velocity when your product spans multiple platforms.

Start with the structure in this chapter. Adjust as you learn your specific needs. But start with a monorepo. Future you will be grateful.

---

> **Next:** [Chapter 16: CI/CD Pipeline — From Push to Production] — how to automate testing, building, and deploying from your monorepo.
