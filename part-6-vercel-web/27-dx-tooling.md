<!--
  CHAPTER: 30
  TITLE: Developer Experience & Tooling
  PART: VI — Vercel & Web
  PREREQS: None
  KEY_TOPICS: Biome, Turbopack, Metro, Expo Dev Client, Reactotron, Claude Code, Storybook, pnpm, Changeset
  DIFFICULTY: Beginner to Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 27: Developer Experience & Tooling

> **Part VI — Vercel & Web** | Prerequisites: None | Difficulty: Beginner to Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- The tools you use every day (linter, bundler, package manager, debugger) have more cumulative impact on team velocity than your framework choice
- Biome replaces ESLint + Prettier with a single Rust-based tool that is 20-100x faster; one config file, zero conflicts, 200+ lint rules enabled by default
- Use pnpm for package management (strict node_modules, fast installs, workspace support); Turbopack for Next.js bundling; Metro for React Native
- Claude Code accelerates frontend development for architecture decisions, debugging, code generation, and refactoring; treat it as a senior pair programmer, not a code autocomplete
- The ideal DX setup in 2026: Biome for lint+format, pnpm for packages, Turbopack/Metro for bundling, Reactotron for RN debugging, Storybook for component development, and Changesets for versioning

</details>

I want to tell you something that took me years to understand: **the tools you use every day matter more than the framework you choose.** A team on a mediocre framework with excellent tooling will outship a team on the best framework with terrible tooling, every single time.

Think about it. You interact with your linter hundreds of times a day. Your bundler runs on every save. Your package manager runs on every install, every CI build, every deploy. Your debugger is the difference between a 5-minute fix and a 5-hour mystery. The cumulative impact of these tools — measured in seconds saved per interaction, multiplied by hundreds of interactions per day, multiplied by every developer on your team, multiplied by every working day of the year — is staggering.

And yet, most teams treat tooling as an afterthought. They copy a config from a blog post in 2022, never update it, and live with whatever friction it creates. They use `console.log` debugging because "the debugger is too hard to set up." They run `npm install` and wait 45 seconds when `pnpm install` would take 8. They manually format code and argue about semicolons in code reviews.

This chapter is about the tooling stack that makes frontend development fast, pleasant, and reliable in 2026. Not just what tools to use, but how to configure them, how they fit together, and what the ideal setup looks like when everything is dialed in.

### In This Chapter
- Biome: Linting + Formatting in One Tool
- Turbopack: The Next.js Bundler
- Metro: React Native's Bundler
- Expo Dev Client: Custom Native Debugging
- React DevTools and Reactotron
- Claude Code for Frontend Development
- Storybook for Component Development
- Package Managers: pnpm vs. npm vs. yarn vs. bun
- tsx: Running TypeScript Directly
- Changesets: Versioning and Changelogs
- The Ideal Frontend DX Setup in 2026

### Related Chapters
- [Ch 5: Expo Platform] — Expo-specific tooling and development workflow
- [Ch 14: Profiling & Debugging] — deep dive on performance profiling tools
- [Ch 26: The AI-Powered Frontend] — AI SDK and building AI features
- [Ch 22: Monorepo Architecture] — how tooling fits into monorepo setups

---

## 1. BIOME: LINTING + FORMATTING IN ONE TOOL

For years, the frontend world ran ESLint for linting and Prettier for formatting. Two tools, two configs, two sets of plugins, two passes over your code, and an endless stream of conflicts between them. ESLint wanted one thing, Prettier wanted another, and you spent time configuring them to not fight each other instead of writing code.

Biome ends this. It is a single tool, written in Rust, that handles both linting and formatting. It is fast — not "a little faster than ESLint" fast, but "processes your entire codebase in under a second" fast. And it ships with sane defaults that eliminate the configuration bikeshedding that plagues ESLint setups.

### 1.1 Why Biome Over ESLint + Prettier

**Speed.** Biome is 20-100x faster than ESLint + Prettier. On a large codebase (thousands of files), ESLint might take 30-60 seconds. Biome takes under a second. This is not a benchmark curiosity — it is the difference between running your linter on every save (instant feedback) and running it only in CI (delayed feedback).

**Single config.** One file — `biome.json` — configures everything: formatting rules, linting rules, import sorting, and file patterns. No `.eslintrc.js`, no `.prettierrc`, no `.eslintignore`, no `.prettierignore`, no `eslint-config-prettier` to make them play nice.

**Better defaults.** Biome ships with over 200 lint rules enabled by default, covering correctness, suspicious patterns, accessibility, and complexity. You get comprehensive linting out of the box without installing 15 plugins.

**Consistency.** Because formatting and linting are one tool, there are no conflicts. The formatter and linter are aware of each other. What one does, the other respects.

### 1.2 Setting Up Biome

```bash
# Install
pnpm add -D @biomejs/biome

# Initialize config
pnpm biome init
```

This creates a `biome.json`:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "trailingCommas": "all",
      "semicolons": "always"
    }
  }
}
```

That is the entire configuration. Compare this to a typical ESLint setup with `@typescript-eslint/parser`, `@typescript-eslint/eslint-plugin`, `eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-import`, `eslint-config-prettier`, and their combined hundreds of lines of configuration.

### 1.3 Running Biome

```bash
# Format files
pnpm biome format --write .

# Lint files
pnpm biome lint .

# Lint and format (the common case)
pnpm biome check --write .

# Check without writing (CI)
pnpm biome check .
```

### 1.4 Package.json Scripts

```json
{
  "scripts": {
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "format": "biome format --write ."
  }
}
```

### 1.5 Editor Integration

Install the Biome extension for VS Code. It provides:
- Format on save (replaces Prettier extension)
- Inline lint errors (replaces ESLint extension)
- Quick fixes for auto-fixable issues
- Import sorting on save

```jsonc
// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  }
}
```

### 1.6 Customizing Rules

You can enable, disable, or change the severity of individual rules:

```json
{
  "linter": {
    "rules": {
      "recommended": true,
      "correctness": {
        "noUnusedVariables": "error",
        "noUnusedImports": "error"
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsole": "warn"
      },
      "complexity": {
        "noForEach": "off"
      },
      "style": {
        "useConst": "error",
        "noNonNullAssertion": "warn"
      },
      "a11y": {
        "useAltText": "error",
        "useAriaProps": "error"
      }
    }
  }
}
```

### 1.7 Migration from ESLint + Prettier

Biome provides a migration command:

```bash
# Migrate ESLint config
pnpm biome migrate eslint --write

# Migrate Prettier config
pnpm biome migrate prettier --write
```

This reads your existing ESLint and Prettier configs and generates an equivalent `biome.json`. It is not perfect — some ESLint plugins have no Biome equivalent — but it gets you 90% of the way there.

After migration, you can remove:
- `eslint`, `@typescript-eslint/*`, all `eslint-plugin-*` packages
- `prettier`, `eslint-config-prettier`
- `.eslintrc.*`, `.prettierrc.*`, `.eslintignore`, `.prettierignore`

The dependency count reduction alone is worth it. A typical ESLint + Prettier setup adds 50+ transitive dependencies. Biome adds one.

### 1.8 When to Still Use ESLint

Biome does not cover every ESLint plugin. If you rely on:
- `eslint-plugin-testing-library` — Biome has some equivalent rules but not all
- Custom ESLint rules specific to your team — you would need to port them
- Framework-specific plugins with deep integration (like some Angular-specific rules)

In these cases, you can run Biome for formatting and most linting, and ESLint for the specialized rules you cannot replace. But for most React / React Native / Next.js projects, Biome covers everything you need.

---

## 2. TURBOPACK: THE NEXT.JS BUNDLER

Turbopack is the bundler built into Next.js, designed to replace Webpack for development (and increasingly, for production builds). If you are using Next.js 15 or later, you already have access to it.

### 2.1 Why Turbopack Exists

Webpack is slow. Everyone knows this. A large Next.js application with Webpack might take 30-60 seconds for a cold start and 1-5 seconds for HMR (Hot Module Replacement) updates. Turbopack was built from scratch in Rust to fix this.

The key innovation is **incremental computation.** Turbopack tracks the dependency graph of your entire application at a granular level. When you change a file, it does not rebuild everything — it recomputes only the parts of the graph that are affected by your change. This is fundamentally different from Webpack's approach of invalidating chunks and rebuilding them.

### 2.2 Enabling Turbopack

```bash
# Development (default in Next.js 15+)
next dev --turbopack

# Or in next.config.ts (always enabled for dev)
# Turbopack is the default dev bundler in Next.js 15+
```

For most projects, you do not need to do anything. If you are on Next.js 15 or later, `next dev` uses Turbopack by default.

### 2.3 What Turbopack Does Better

**Cold start:** Where Webpack might take 30 seconds to start a large application, Turbopack starts in 2-5 seconds. It does not eagerly compile your entire application — it compiles routes on demand, so you only pay for what you visit.

**HMR (Hot Module Replacement):** This is where the difference is most felt. Webpack HMR on a large app can take 1-5 seconds. Turbopack HMR is typically under 200 milliseconds. You save a file, and the change is visible almost instantly. This sounds like a small thing until you realize you save files hundreds of times a day.

**Memory usage:** Turbopack uses significantly less memory than Webpack because of its incremental architecture. It does not hold the entire compiled application in memory — only the parts that are currently needed.

### 2.4 Turbopack Configuration

Turbopack configuration lives in `next.config.ts` under the `turbopack` key:

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  turbopack: {
    // Custom resolve aliases (like Webpack's resolve.alias)
    resolveAlias: {
      '@components': './src/components',
      '@utils': './src/utils',
    },
    // Custom resolve extensions
    resolveExtensions: ['.ts', '.tsx', '.js', '.jsx', '.json'],
    // Module rules for custom loaders
    rules: {
      '*.svg': {
        loaders: ['@svgr/webpack'],
        as: '*.js',
      },
    },
  },
};

export default nextConfig;
```

### 2.5 Webpack Compatibility

Not everything from Webpack works in Turbopack yet. The main gaps:

- **Custom Webpack plugins**: Most Webpack plugins do not work with Turbopack. Check the Next.js documentation for Turbopack equivalents.
- **Module federation**: Not yet supported in Turbopack.
- **Some loaders**: Most common loaders work, but exotic ones may not.

If you hit a compatibility issue, you can fall back to Webpack for specific builds:

```bash
# Force Webpack for development
next dev --webpack
```

But the trajectory is clear: Turbopack is the future, Webpack is the past. Migrate your custom Webpack configuration to Turbopack-compatible patterns as soon as you can.

---

## 3. METRO: REACT NATIVE'S BUNDLER

If Turbopack is the bundler for the web side of your stack, Metro is the bundler for the mobile side. It ships with React Native and Expo, and understanding how it works will save you from a category of confusing build errors.

### 3.1 What Metro Does

Metro takes your JavaScript/TypeScript source code and bundles it into a single JavaScript file that can run on a phone. It handles:

- **Module resolution**: Finding and loading all your imports
- **Transformation**: Compiling TypeScript, JSX, and modern JS to something Hermes can run
- **Bundling**: Combining everything into a single file (or multiple files with code splitting)
- **Hot Module Replacement**: Updating your running app when you save a file

### 3.2 Metro Configuration

Metro configuration lives in `metro.config.js` at the root of your project. For Expo projects:

```javascript
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

// Customize resolver
config.resolver.sourceExts = ['tsx', 'ts', 'jsx', 'js', 'json', 'cjs', 'mjs'];

// Add asset extensions
config.resolver.assetExts = [
  ...config.resolver.assetExts,
  'db', 'sql', 'ttf', 'otf', 'png', 'jpg',
];

// Support for monorepos
config.watchFolders = [
  // Watch the shared packages directory
  require('path').resolve(__dirname, '../../packages'),
];

module.exports = config;
```

### 3.3 Common Metro Issues

**"Module not found" errors in monorepos:** Metro's resolver does not follow symlinks the same way Node does. In a monorepo with pnpm (which uses symlinks extensively), you need to configure `watchFolders` to point to the actual package locations.

```javascript
// metro.config.js for pnpm monorepos
const { getDefaultConfig } = require('expo/metro-config');
const path = require('path');

const projectRoot = __dirname;
const monorepoRoot = path.resolve(projectRoot, '../..');

const config = getDefaultConfig(projectRoot);

// Watch all packages in the monorepo
config.watchFolders = [monorepoRoot];

// Let Metro know where to resolve packages
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(monorepoRoot, 'node_modules'),
];

module.exports = config;
```

**Slow bundle times:** Metro caches aggressively. If your bundles are slow, first try clearing the cache:

```bash
# Clear Metro cache
npx expo start --clear

# Or manually
rm -rf node_modules/.cache/metro
```

**Platform-specific modules:** Metro supports platform extensions out of the box. If you have `Button.ios.tsx` and `Button.android.tsx`, Metro will automatically pick the right one based on the target platform.

### 3.4 Metro vs. Turbopack

These are not competing tools — they serve different purposes:

| | Metro | Turbopack |
|---|---|---|
| **Purpose** | React Native bundling | Next.js web bundling |
| **Target** | Mobile (iOS, Android) | Web (browsers) |
| **Output** | Single JS bundle for Hermes | Optimized chunks for browsers |
| **HMR** | Over device/simulator connection | Over localhost WebSocket |
| **Written in** | JavaScript | Rust |

If you are building a cross-platform app with Expo and Next.js, you use both: Metro for the mobile app, Turbopack for the web app.

---

## 4. EXPO DEV CLIENT: CUSTOM NATIVE DEBUGGING

The Expo Dev Client is a custom development build of your app that includes all the debugging tools you need. Think of it as a development-only version of your app that has superpowers.

### 4.1 Why You Need It

Expo Go (the pre-built Expo app from the App Store) is great for getting started, but it has limitations: it only includes the native modules that Expo bundles by default. The moment you add a custom native module, a library that requires native code not in Expo Go, or custom native configuration, you need a dev client.

The dev client is your app, compiled with debug tools baked in. It connects to Metro, supports HMR, and includes:

- An in-app developer menu (shake to open)
- Network inspector
- Performance overlay
- Element inspector
- Console log viewer

### 4.2 Creating a Dev Client

```bash
# Build a development client
npx expo run:ios
# or
npx expo run:android

# Or use EAS Build for cloud-built dev clients
eas build --profile development --platform ios
```

The dev client build includes all your native dependencies and Expo config plugins, so everything works exactly as it will in production — but with development tools attached.

### 4.3 The Development Workflow

```
1. Build dev client once (eas build --profile development)
2. Install it on your device/simulator
3. Start Metro: npx expo start --dev-client
4. Open the dev client app -> it connects to Metro
5. Code, save, see changes via HMR
6. Rebuild dev client only when native dependencies change
```

The key insight: **you only rebuild the dev client when your native dependencies change.** If you are just writing TypeScript/JavaScript, Metro handles the updates via HMR. This means you rebuild the native shell maybe once a week, not on every change.

### 4.4 EAS Build Profiles for Dev Client

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    },
    "development-device": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": false
      }
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

Two development profiles: one for simulator (faster, no signing needed) and one for physical devices (requires signing, but lets you test on real hardware).

---

## 5. REACT DEVTOOLS AND REACTOTRON

### 5.1 React DevTools

React DevTools is the official debugging tool for React applications. It works for both web (browser extension) and React Native (standalone app).

**For Web (Browser Extension):**
Install the React DevTools browser extension. It adds two tabs to your browser dev tools:
- **Components**: Inspect the React component tree, view props, state, and hooks
- **Profiler**: Record rendering performance, identify unnecessary re-renders

**For React Native (Standalone):**
```bash
# Install globally
npm install -g react-devtools

# Run it
react-devtools
```

This opens a standalone window that connects to your React Native app. Same capabilities as the browser extension — component tree, props, state, hooks, profiler.

**What to look for in the Profiler:**
- Components that re-render when they should not (highlighted in the flame chart)
- Long render times (wide bars in the flame chart)
- The "Why did this render?" feature that tells you exactly which prop or state change triggered a re-render

### 5.2 Reactotron

Reactotron is a desktop app for inspecting React Native (and React) applications. It is particularly useful for React Native because the browser dev tools are not available.

```bash
# Install the Reactotron desktop app from:
# https://github.com/infinitered/reactotron/releases

# Install the client library
pnpm add -D reactotron-react-native
```

**Configuration:**

```typescript
// src/devtools/reactotron.ts
import Reactotron from 'reactotron-react-native';
import { QueryClientManager, reactotronReactQuery } from 'reactotron-react-query';
import { queryClient } from '../lib/queryClient';

const queryClientManager = new QueryClientManager({ queryClient });

Reactotron.configure({
  name: 'MyApp',
  // host: '192.168.1.100', // Use your machine's IP for device debugging
})
  .useReactNative({
    networking: {
      ignoreUrls: /symbolicate|logs/, // Ignore noisy requests
    },
    storybook: false,
    errors: { veto: () => false },
  })
  .use(reactotronReactQuery(queryClientManager))
  .connect();

// Patch console.log to also show in Reactotron
if (__DEV__) {
  console.tron = Reactotron;
}
```

**Load it early in your app:**

```typescript
// app/_layout.tsx (Expo Router) or index.ts
if (__DEV__) {
  require('./devtools/reactotron');
}
```

**What Reactotron gives you:**
- **Timeline**: Every action, API call, and log in chronological order
- **API monitoring**: See every network request with headers, body, and response
- **State inspection**: View and modify Zustand stores, Redux state, or AsyncStorage in real-time
- **React Query integration**: See all queries, their status, and cache state
- **Benchmarking**: Measure how long operations take
- **Custom commands**: Send commands to your app from the Reactotron desktop

**The killer feature: API monitoring.** In React Native, you do not have the browser's Network tab. Reactotron gives you the equivalent — every API call, with request/response bodies, headers, timing, and status codes. This alone makes it worth installing.

### 5.3 Debugging Workflow Comparison

```
Web Debugging:
  Browser DevTools  ->  Components tab (React DevTools)
                    ->  Network tab (built-in)
                    ->  Console (built-in)
                    ->  Performance tab (built-in)
                    ->  Profiler tab (React DevTools)

React Native Debugging:
  React DevTools (standalone)  ->  Components, Profiler
  Reactotron                   ->  Network, State, Logs, Timeline
  Expo Dev Client              ->  Element inspector, Performance overlay
  Flipper (optional)           ->  Layout inspector, Database inspector
```

---

## 6. CLAUDE CODE FOR FRONTEND DEVELOPMENT

Let me be direct: AI-assisted development is not optional in 2026. Claude Code specifically has become a critical tool in the frontend development workflow. Not as a replacement for understanding your code, but as a multiplier for the understanding you already have.

### 6.1 What Claude Code Is

Claude Code is a CLI tool that gives Claude access to your codebase, your terminal, and your development environment. It can read files, write files, run commands, search code, and interact with Git. It runs in your terminal, in your project directory, with your permissions.

The difference between Claude Code and a chatbot is the difference between a consultant who can look at your code and a consultant who can only hear your description of your code. Claude Code sees the actual files, understands the actual structure, and makes changes in the actual codebase.

### 6.2 CLAUDE.md — Teaching Claude Your Project

The most important file for Claude Code is `CLAUDE.md` at the root of your project. This file teaches Claude how your specific project works — conventions, patterns, architecture decisions, and things it should know before making changes.

```markdown
# CLAUDE.md

## Project Overview
Cross-platform app built with Expo (mobile) and Next.js (web).
Monorepo managed with pnpm workspaces and Turborepo.

## Architecture
- /apps/mobile — Expo Router app (iOS + Android)
- /apps/web — Next.js App Router
- /packages/ui — Shared component library (Tamagui)
- /packages/api — tRPC router definitions
- /packages/db — Drizzle ORM schema and migrations

## Key Conventions
- All components use named exports, not default exports
- State management: Zustand for client state, TanStack Query for server state
- Styling: Tamagui for cross-platform, Tailwind for web-only pages
- API layer: tRPC with Zod validation on all inputs
- Tests: Vitest for unit tests, Maestro for mobile E2E

## Commands
- `pnpm dev` — Start all apps in development
- `pnpm test` — Run all tests
- `pnpm lint` — Run Biome check
- `pnpm db:push` — Push schema changes to database
- `pnpm db:generate` — Generate Drizzle migrations

## Important Patterns
- All API routes require authentication via middleware
- Use `useCallback` and `useMemo` only when there is a measured perf issue
- Prefer server components for data fetching on web
- Mobile navigation uses Expo Router file-based routing
- Error boundaries wrap every screen in mobile app

## Things Claude Should Know
- The app uses Expo SDK 52 with the New Architecture enabled
- We use Drizzle, not Prisma
- Our CI runs on GitHub Actions, deploy previews on Vercel
- Never import from `next/server` in shared packages
```

A good `CLAUDE.md` saves you from repeating the same context every time you start a conversation. Claude reads it automatically and applies the conventions to every change it makes.

### 6.3 Skills

Skills are reusable instructions that Claude Code can invoke for specific tasks. They are like recipes that encode your team's best practices:

```markdown
<!-- .claude/skills/create-component.md -->
# Create Component

When creating a new shared UI component:

1. Create the file in `packages/ui/src/components/`
2. Use Tamagui for styling (not raw StyleSheet or Tailwind)
3. Export from `packages/ui/src/index.ts`
4. Add a Storybook story in `packages/ui/src/stories/`
5. Include TypeScript props interface with JSDoc comments
6. Add basic accessibility props (accessibilityRole, accessibilityLabel)
7. Write a Vitest test in `packages/ui/src/__tests__/`

Example structure:
- ComponentName.tsx (component)
- ComponentName.test.tsx (tests)
- ComponentName.stories.tsx (storybook)
```

### 6.4 Hooks

Hooks let you run automated checks before and after Claude Code takes actions. They are like Git hooks but for AI actions:

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm biome check --write $CLAUDE_FILE_PATH"
      }
    ],
    "Stop": [
      {
        "command": "pnpm typecheck"
      }
    ]
  }
}
```

This automatically runs Biome formatting on any file Claude writes or edits, and runs type checking when Claude finishes a task. You get guardrails without manual intervention.

### 6.5 MCP Servers

MCP (Model Context Protocol) servers give Claude Code access to external tools and data sources. For frontend development, useful MCP servers include:

- **Figma MCP**: Claude can read Figma designs and implement them as components
- **GitHub MCP**: Claude can read issues, PRs, and code reviews
- **Database MCP**: Claude can query your database to understand data shapes
- **Storybook MCP**: Claude can see your component library

### 6.6 Practical Claude Code Workflows

**Workflow 1: Implement a feature from a GitHub issue**
```
You: "Implement the feature described in GitHub issue #234"
Claude Code: [reads the issue] [reads related code] [implements the feature]
             [runs tests] [creates a commit]
```

**Workflow 2: Debug a production error**
```
You: "We're seeing this error in production: [paste error]"
Claude Code: [searches codebase for related code] [traces the execution path]
             [identifies the root cause] [proposes and implements a fix]
             [writes a regression test]
```

**Workflow 3: Refactor a module**
```
You: "The user auth module is too complex. Refactor it."
Claude Code: [reads the entire module] [identifies complexity]
             [proposes a refactoring plan] [implements incrementally]
             [runs tests after each change] [verifies nothing broke]
```

**Workflow 4: Create components from design**
```
You: "Implement this Figma design as a React component" [provides Figma link]
Claude Code: [reads the Figma design via MCP] [creates the component]
             [adds Storybook story] [writes tests]
```

The pattern across all of these: Claude Code is most effective when it has full context (via CLAUDE.md), clear conventions (via skills), and guardrails (via hooks). Invest in these, and Claude Code becomes dramatically more useful.

---

## 7. STORYBOOK FOR COMPONENT DEVELOPMENT

Storybook is a tool for developing UI components in isolation. You build each component independently, outside your application, with different states and variations visible at a glance.

### 7.1 Why Storybook Matters

**Component isolation.** When you develop a component inside your app, you are constantly fighting with the context: routing, authentication, data fetching, state management. Storybook strips all of that away. You render the component with explicit props and see exactly what it looks like.

**Visual testing.** Every state of every component is visible: loading, error, empty, full, disabled, focused, hovered. You catch visual bugs that you would never think to test manually.

**Documentation.** Storybook automatically generates documentation for your components. New team members can browse the component library and understand what is available without reading code.

**Design system development.** If you are building a shared component library (and you should be — see Chapter 22 on monorepos), Storybook is where that library lives and breathes.

### 7.2 Setting Up Storybook

```bash
# Initialize Storybook (auto-detects your framework)
pnpm dlx storybook@latest init
```

This sets up Storybook with the right configuration for your framework (React, React Native, Next.js, etc.).

### 7.3 Writing Stories

A "story" is a specific state of a component:

```tsx
// components/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  component: Button,
  title: 'UI/Button',
  tags: ['autodocs'], // Auto-generate documentation
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'danger', 'ghost'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
    disabled: { control: 'boolean' },
    loading: { control: 'boolean' },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

// Each export is a story (a specific state)
export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Click me',
  },
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Click me',
  },
};

export const Danger: Story = {
  args: {
    variant: 'danger',
    children: 'Delete',
  },
};

export const Loading: Story = {
  args: {
    variant: 'primary',
    children: 'Saving...',
    loading: true,
  },
};

export const Disabled: Story = {
  args: {
    variant: 'primary',
    children: 'Submit',
    disabled: true,
  },
};

export const AllSizes: Story = {
  render: () => (
    <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
      <Button size="sm">Small</Button>
      <Button size="md">Medium</Button>
      <Button size="lg">Large</Button>
    </div>
  ),
};
```

### 7.4 Storybook for React Native

React Native Storybook works differently — it runs inside your app, not in a browser:

```bash
# Install for React Native
pnpm add -D @storybook/react-native @storybook/addon-ondevice-controls
```

You can set up a special entry point that shows Storybook instead of your app during development:

```typescript
// app.config.ts
export default {
  // ...
  extra: {
    storybookEnabled: process.env.STORYBOOK_ENABLED === 'true',
  },
};
```

```tsx
// App.tsx
import Constants from 'expo-constants';

function App() {
  if (Constants.expoConfig?.extra?.storybookEnabled) {
    const StorybookUI = require('./.storybook').default;
    return <StorybookUI />;
  }

  return <MainApp />;
}
```

There is also `@storybook/react-native-web` which renders React Native components in a browser using `react-native-web`, giving you the standard Storybook web experience with React Native components.

### 7.5 Visual Regression Testing with Storybook

Storybook integrates with visual testing tools like Chromatic:

```bash
# Install Chromatic
pnpm add -D chromatic

# Run visual tests
pnpm chromatic --project-token=your-token
```

Chromatic screenshots every story and compares them to the previous version. If a component changes visually, you get a diff. This catches unintended visual regressions that unit tests cannot detect.

---

## 8. PACKAGE MANAGERS: PNPM VS. NPM VS. YARN VS. BUN

This is one of those topics where people have strong opinions. Let me give you the pragmatic breakdown.

### 8.1 The Comparison

```
                    npm       yarn      pnpm      bun
Speed (install)     Slow      Medium    Fast      Fastest
Disk usage          High      High      Low       Medium
Monorepo support    Basic     Good      Excellent Good
Lock file           package-  yarn.lock pnpm-     bun.lock
                    lock.json           lock.yaml
Strictness          Loose     Loose     Strict    Loose
Node.js compat      Perfect   Perfect   Perfect   Good*
Maturity            Oldest    Mature    Mature    Young
```

*Bun has occasional compatibility issues with some Node.js APIs and native modules.

### 8.2 Why pnpm Wins for Most Teams

**Disk space.** pnpm uses a content-addressable store. If ten projects use `react@19.0.0`, pnpm stores it once on disk and hardlinks to it from each project. npm and yarn store a full copy in each project's `node_modules`. On a machine with multiple projects, this saves gigabytes.

**Strictness.** pnpm creates a non-flat `node_modules` structure. If your `package.json` does not list a dependency, you cannot import it — even if it exists as a transitive dependency. This catches phantom dependency bugs that would only surface in production.

**Speed.** pnpm is 2-3x faster than npm for most operations. `pnpm install` on a warm cache is nearly instant.

**Monorepo support.** pnpm workspaces are the best in the business. The `pnpm -r` (recursive) command, workspace protocol (`workspace:*`), and filtering (`--filter`) make monorepo management straightforward.

```bash
# Install dependencies for all packages
pnpm install

# Run build in all packages
pnpm -r build

# Run tests only in packages that changed
pnpm -r --filter="...[origin/main]" test

# Add a dependency to a specific package
pnpm --filter @myapp/mobile add react-native-reanimated

# Run a script in a specific workspace
pnpm --filter @myapp/web dev
```

### 8.3 When to Use the Others

**npm**: When you are working on a simple project and do not want to install another tool. npm ships with Node.js. It works. It is just slower and uses more disk space.

**yarn**: If your team is already on Yarn and everything works, there is no urgent reason to switch. Yarn 4 (Berry) is fast and has good features. But if you are starting fresh, pnpm is the better choice.

**bun**: When speed is your top priority and you are willing to accept occasional compatibility issues. Bun's install speed is genuinely impressive — it can install a large project's dependencies in 1-2 seconds. But bun's Node.js compatibility is not perfect, and you may hit edge cases with native modules, especially in React Native projects.

### 8.4 Setting Up pnpm

```bash
# Install pnpm (recommended: via corepack)
corepack enable
corepack prepare pnpm@latest --activate

# Or standalone install
curl -fsSL https://get.pnpm.io/install.sh | sh -
```

```yaml
# pnpm-workspace.yaml (for monorepos)
packages:
  - 'apps/*'
  - 'packages/*'
```

```json
// .npmrc (recommended pnpm settings)
// auto-install-peers=true
// strict-peer-dependencies=false
// shamefully-hoist=false
```

Note on `shamefully-hoist`: Some React Native tools expect a flat `node_modules` structure. If you hit module resolution issues with Metro, you may need `shamefully-hoist=true` or targeted `public-hoist-pattern` entries. This is a Metro limitation, not a pnpm problem.

---

## 9. TSX: RUNNING TYPESCRIPT DIRECTLY

`tsx` is a tool that lets you run TypeScript files directly, without a build step. It is the spiritual successor to `ts-node`, but faster and with zero configuration.

### 9.1 Why tsx

Every frontend project has scripts: database seeds, data migrations, code generators, API testing scripts, utility scripts. These should be in TypeScript (same language as your codebase, same types, same imports). But running TypeScript traditionally requires either a build step or `ts-node` with its slow startup and configuration headaches.

`tsx` solves this:

```bash
# Install
pnpm add -D tsx

# Run any TypeScript file
pnpm tsx scripts/seed-database.ts
pnpm tsx scripts/generate-types.ts
pnpm tsx scripts/migrate-data.ts
```

No `tsconfig.json` configuration needed. No `--loader` flags. No `--experimental-*` flags. It just works.

### 9.2 Watch Mode

```bash
# Re-run on file changes
pnpm tsx watch scripts/dev-server.ts
```

### 9.3 In package.json Scripts

```json
{
  "scripts": {
    "seed": "tsx scripts/seed.ts",
    "generate": "tsx scripts/generate-api-types.ts",
    "migrate": "tsx scripts/run-migrations.ts",
    "check-env": "tsx scripts/validate-env.ts"
  }
}
```

### 9.4 tsx vs. Alternatives

| Tool | Speed | Config needed | ESM support | CJS support |
|------|-------|--------------|-------------|-------------|
| tsx | Fast | None | Yes | Yes |
| ts-node | Slow | tsconfig.json paths, --loader flags | Complicated | Yes |
| bun | Fastest | None | Yes | Yes |
| node --loader | Medium | Flags, configuration | Yes | Yes |

If you are already using bun as your runtime, `bun run script.ts` works similarly. But if you are using Node.js (which most production apps are), `tsx` is the best option.

---

## 10. CHANGESETS: VERSIONING AND CHANGELOGS

If you are building a monorepo with shared packages (and you should be), you need a way to version those packages and communicate changes to consumers. Changesets is the tool for this.

### 10.1 The Problem Changesets Solves

Imagine you have this monorepo:

```
packages/
  ui/         (v1.2.0)
  api-client/ (v2.0.1)
  utils/      (v0.5.3)
apps/
  mobile/
  web/
```

You make a change to `packages/ui` that adds a new component. You also fix a bug in `packages/utils`. Now you need to:

1. Bump the version of `packages/ui` (minor bump: 1.2.0 -> 1.3.0)
2. Bump the version of `packages/utils` (patch bump: 0.5.3 -> 0.5.4)
3. Update the changelog for both packages
4. Publish both packages
5. If `packages/api-client` depends on `packages/utils`, bump its dependency too

Doing this manually is error-prone. Doing it for 20 packages in a large monorepo is impossible. Changesets automates the entire flow.

### 10.2 Setting Up Changesets

```bash
# Install
pnpm add -D @changesets/cli

# Initialize
pnpm changeset init
```

This creates a `.changeset` directory with a config file:

```json
// .changeset/config.json
{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

### 10.3 The Workflow

**Step 1: When you make a change, create a changeset:**

```bash
pnpm changeset
```

This walks you through an interactive prompt:

```
Which packages would you like to include? 
  [x] @myapp/ui
  [ ] @myapp/api-client
  [x] @myapp/utils

Which packages should have a major bump?
  (none)

Which packages should have a minor bump?
  [x] @myapp/ui

The following packages will be patch bumped:
  @myapp/utils

Please enter a summary for this change:
  Added DatePicker component to UI library. Fixed timezone bug in formatDate utility.
```

This creates a Markdown file in `.changeset/`:

```markdown
---
'@myapp/ui': minor
'@myapp/utils': patch
---

Added DatePicker component to UI library. Fixed timezone bug in formatDate utility.
```

**Step 2: Commit the changeset file with your code changes.**

The changeset file is just a Markdown file — it gets committed alongside your code changes. Code reviewers can see exactly what version bump you are proposing and why.

**Step 3: When ready to release, consume the changesets:**

```bash
pnpm changeset version
```

This:
- Bumps version numbers in `package.json` files
- Updates `CHANGELOG.md` for each package
- Removes the consumed changeset files

**Step 4: Publish:**

```bash
pnpm changeset publish
```

### 10.4 Changesets in CI

The real power of Changesets is the GitHub bot and CI automation:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install
      - run: pnpm build

      - name: Create Release Pull Request or Publish
        uses: changesets/action@v1
        with:
          publish: pnpm changeset publish
          version: pnpm changeset version
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

This workflow:
1. Detects pending changesets on every push to main
2. Creates a "Version Packages" PR that bumps versions and updates changelogs
3. When that PR is merged, publishes the packages to npm

### 10.5 Changesets for Internal Packages

Even if you are not publishing to npm (your packages are only used within your monorepo), Changesets is still valuable for:
- **Changelogs**: Knowing what changed in each package and when
- **Version tracking**: Knowing which version of a shared package each app uses
- **Release communication**: The changelog serves as documentation for your team

For internal-only packages, set `access: "restricted"` and skip the publish step.

---

## 11. THE IDEAL FRONTEND DX SETUP IN 2026

Let me bring everything together. Here is what the ideal frontend development setup looks like when all the tools are dialed in correctly.

### 11.1 The Stack

```
Package Manager:    pnpm (with corepack)
Monorepo:           Turborepo + pnpm workspaces
Linting/Formatting: Biome
Web Bundler:        Turbopack (via Next.js)
Mobile Bundler:     Metro (via Expo)
Type Checking:      TypeScript (strict mode)
Testing:            Vitest (unit) + Playwright (web E2E) + Maestro (mobile E2E)
Component Dev:      Storybook
Debugging (Web):    Browser DevTools + React DevTools
Debugging (Mobile): Expo Dev Client + Reactotron
Scripts:            tsx
Versioning:         Changesets
AI Assistant:       Claude Code (with CLAUDE.md, skills, hooks)
CI/CD:              GitHub Actions + Vercel + EAS Build
```

### 11.2 The Developer's Day

Here is what a typical development session looks like with this stack:

```
8:30 AM — Start work
  $ pnpm dev                          # Starts all apps (Turborepo)
                                       # Next.js starts in 2s (Turbopack)
                                       # Expo starts in 3s (Metro)
  
8:32 AM — Pick up a feature ticket
  Open Claude Code
  "Implement the notification preferences screen from issue #456"
  Claude Code reads CLAUDE.md, the issue, the design, relevant code
  Proposes an implementation plan
  
8:35 AM — Start coding
  Save a file -> Biome formats it instantly (< 50ms)
  Save a file -> Turbopack HMR updates browser (< 200ms)
  Save a file -> Metro HMR updates simulator (< 500ms)
  
  Component development in Storybook:
  $ pnpm storybook                    # See all component states
  Build the component in isolation
  Verify all states: loading, error, empty, populated
  
9:00 AM — Debug a rendering issue on mobile
  Shake phone -> Expo Dev Client menu
  Toggle element inspector
  Check Reactotron for API responses
  Open React DevTools -> Profiler -> record
  Find the unnecessary re-render -> fix it
  
9:30 AM — Run tests
  $ pnpm test                         # Vitest runs in < 5s
  $ pnpm test:e2e                     # Playwright for web
  
10:00 AM — Ready for review
  $ pnpm changeset                    # Document what changed
  $ git add . && git commit           # Biome runs in pre-commit hook
  $ git push                          # CI runs: types, lint, test
                                       # Vercel creates preview deploy
                                       # PR created with changeset summary
```

### 11.3 The Configuration Files

Here is the minimal set of configuration files for this stack:

```
project-root/
  biome.json                  # Linting + formatting
  turbo.json                  # Turborepo pipeline
  pnpm-workspace.yaml         # Workspace definitions
  .changeset/config.json      # Versioning config
  tsconfig.json               # Base TypeScript config
  CLAUDE.md                   # AI assistant context
  .claude/
    settings.json             # Claude Code hooks
    skills/                   # Reusable AI instructions
  .vscode/
    settings.json             # Editor config (Biome formatter)
    extensions.json           # Recommended extensions
  .github/
    workflows/
      ci.yml                  # Lint, type-check, test
      release.yml             # Changeset versioning + publish
```

### 11.4 The CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install

      # All of these are fast because of the tools we chose
      - name: Lint + Format Check
        run: pnpm biome check .          # < 2s for entire monorepo

      - name: Type Check
        run: pnpm -r typecheck            # Parallel across packages

      - name: Unit Tests
        run: pnpm -r test                 # Vitest, parallel

      - name: Build
        run: pnpm -r build               # Turborepo caches unchanged packages
```

### 11.5 Pre-commit Hooks

Use `lefthook` (fast, written in Go) instead of `husky` + `lint-staged`:

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    biome:
      glob: "*.{ts,tsx,js,jsx,json}"
      run: pnpm biome check --write {staged_files}
      stage_fixed: true
    typecheck:
      run: pnpm -r typecheck
```

### 11.6 VS Code Settings for the Team

```jsonc
// .vscode/settings.json
{
  // Biome handles formatting
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit",
    "source.organizeImports.biome": "explicit"
  },

  // TypeScript
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.preferences.importModuleSpecifier": "non-relative",

  // Disable conflicting formatters
  "prettier.enable": false,
  "eslint.enable": false,

  // File nesting (cleaner file tree)
  "explorer.fileNesting.enabled": true,
  "explorer.fileNesting.patterns": {
    "*.ts": "${capture}.test.ts, ${capture}.stories.tsx",
    "*.tsx": "${capture}.test.tsx, ${capture}.stories.tsx",
    "package.json": "pnpm-lock.yaml, .npmrc, turbo.json"
  }
}
```

```jsonc
// .vscode/extensions.json
{
  "recommendations": [
    "biomejs.biome",
    "ms-vscode.vscode-typescript-next",
    "bradlc.vscode-tailwindcss",
    "styled-components.vscode-styled-components"
  ],
  "unwantedRecommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint"
  ]
}
```

---

## 12. TOOLING ANTI-PATTERNS

Let me save you from the mistakes I see teams make repeatedly.

### 12.1 The Config Sprawl Anti-Pattern

```
project/
  .eslintrc.js
  .eslintrc.json           # Wait, which one is used?
  .prettierrc
  .prettierrc.js           # This overrides .prettierrc? Or vice versa?
  .prettierignore
  .eslintignore
  prettier.config.js       # Oh no, there's a third one?
  stylelint.config.js
  commitlint.config.js
  lint-staged.config.js
  .lintstagedrc
```

**Fix:** Use Biome (one config for linting + formatting). Use Lefthook (one config for Git hooks). Eliminate duplicate and conflicting configs.

### 12.2 The Slow CI Anti-Pattern

```yaml
# BAD: Sequential, no caching, installs everything
steps:
  - run: npm install            # 60s
  - run: npm run lint            # 30s
  - run: npm run typecheck       # 20s
  - run: npm run test            # 45s
  - run: npm run build           # 60s
  # Total: ~3.5 minutes
```

```yaml
# GOOD: Parallel where possible, cached, fast tools
steps:
  - run: pnpm install            # 8s (cached)
  - run: pnpm biome check .      # 2s
  # Type check and test run in parallel
  - run: pnpm -r typecheck & pnpm -r test & wait   # 15s
  - run: pnpm -r build           # 20s (Turborepo cached)
  # Total: ~45 seconds
```

### 12.3 The "It Works on My Machine" Anti-Pattern

```json
// BAD: No pinned tool versions
{
  "dependencies": {
    "react": "^19.0.0"    // Different version on every install
  }
}
```

```json
// GOOD: Pin versions, use lockfile, pin Node version
{
  "engines": {
    "node": ">=22.0.0",
    "pnpm": ">=9.0.0"
  },
  "packageManager": "pnpm@9.15.0"
}
```

Also pin your Node.js version:

```
# .node-version
22.12.0
```

And your tool versions via corepack:

```json
// package.json
{
  "packageManager": "pnpm@9.15.0+sha512.abc123..."
}
```

### 12.4 The Over-Tooling Anti-Pattern

I have seen teams with:
- ESLint + Prettier + Stylelint + Commitlint + lint-staged + Husky
- TypeScript + Flow (yes, both)
- Jest + Mocha + Cypress + Playwright + Testing Library
- Webpack + esbuild + SWC (all configured, none working properly)

More tools does not mean better DX. It means more configuration, more conflicts, more things to update, and more cognitive overhead. Use the minimal set of tools that covers your needs.

### 12.5 The Ignored Warnings Anti-Pattern

```typescript
// The codebase has 847 TypeScript warnings
// Everyone ignores them
// New warnings get buried in the noise
// A real issue hides among the ignored warnings for 3 months
```

**Fix:** Treat warnings as errors in CI. Fix them or explicitly suppress them with a comment explaining why. A codebase with zero warnings means every new warning is visible and actionable.

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

---

## 13. THE TOOLING DECISION FRAMEWORK

When evaluating a new tool, ask these questions in order:

```
1. Does it solve a real problem we have today?
   (Not a hypothetical future problem)
   NO  -> Don't add it
   YES -> Continue

2. Can an existing tool do this?
   (Even if imperfectly)
   YES -> Use the existing tool
   NO  -> Continue

3. Is it actively maintained?
   (Check: last release, open issues, contributor count)
   NO  -> Find an alternative
   YES -> Continue

4. Does the team need to learn something new?
   YES -> Is the learning cost justified by the benefit?
         NO  -> Don't add it
         YES -> Continue
   NO  -> Continue

5. Does it integrate with our existing tools?
   (Editor, CI, other tools in the stack)
   NO  -> Strong signal to skip it
   YES -> Add it

6. Can we remove it later if it doesn't work out?
   NO  -> Be very cautious
   YES -> Try it in a branch first
```

The best tool is the one your team actually uses correctly. A "worse" tool that everyone understands beats a "better" tool that half the team ignores or misconfigures.

---

## 14. LOOKING AHEAD: TOOLING TRENDS

**AI-native tooling.** Tools are increasingly built with AI assistance in mind. Biome's structured error output is easier for AI tools to parse and fix than ESLint's. Storybook stories serve as context for AI code generation. CLAUDE.md files are becoming standard in codebases. Expect this trend to accelerate.

**Rust-based tools.** The rewrite-it-in-Rust trend is not a fad. Biome, Turbopack, SWC, and Oxc are all Rust-based, and they are all dramatically faster than their JavaScript predecessors. This is a permanent shift: performance-critical tooling is moving to compiled languages.

**Fewer, better tools.** The JavaScript ecosystem is consolidating. Instead of 5 tools that each do one thing (often conflicting), we are moving toward fewer tools that each do more. Biome replacing ESLint + Prettier is the clearest example. Expect more consolidation.

**Remote development environments.** Tools like GitHub Codespaces and Gitpod mean your development environment is defined in code and spun up in seconds. This eliminates "works on my machine" and makes onboarding a new developer take minutes instead of days.

**Type-safe everything.** TypeScript won. The entire stack — from database schema (Drizzle) to API definitions (tRPC) to form validation (Zod) to environment variables (t3-env) — is typed end-to-end. Tools that do not support TypeScript are increasingly irrelevant.

---

## CHAPTER SUMMARY

- **Biome** replaces ESLint + Prettier with a single, fast tool. One config, 20-100x faster, better defaults.
- **Turbopack** is the default Next.js dev bundler. Sub-200ms HMR, 2-5 second cold starts. Use it.
- **Metro** bundles React Native apps. Understand `watchFolders` and platform extensions for monorepo setups.
- **Expo Dev Client** is your custom debug build. Rebuild only when native deps change, not on every code change.
- **React DevTools** (Profiler) and **Reactotron** (API monitoring, state inspection) are essential for React Native debugging.
- **Claude Code** with a good `CLAUDE.md`, skills, and hooks is a development multiplier. Invest in the configuration.
- **Storybook** develops components in isolation. Every state visible, documented, and testable.
- **pnpm** is the best package manager for most teams: fast, strict, excellent monorepo support, low disk usage.
- **tsx** runs TypeScript files directly with zero config. Use it for all your scripts.
- **Changesets** automates versioning and changelogs in monorepos. Essential for shared packages.
- The ideal DX setup prioritizes **fast feedback loops**: instant formatting, sub-second HMR, fast tests, and fast CI.

---

*Next chapter: [Chapter 28: Testing Strategies](/part-6-vercel-web/28-testing-strategies.md)*
