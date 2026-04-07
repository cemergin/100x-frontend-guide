<!--
  CHAPTER: 49
  TITLE: Agentic UI Development — AI-Powered Frontend Workflows
  PART: VI — Vercel & the Web
  PREREQS: Chapters 30, 29
  KEY_TOPICS: Claude Code, Cursor, GitHub Copilot, AI code review, AI testing, AI component generation, CLAUDE.md, MCP servers, skills, hooks, prompt engineering for code, AI pair programming, Vercel v0, AI design-to-code, Figma-to-code, agentic workflows, multi-agent development
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 49: Agentic UI Development — AI-Powered Frontend Workflows

> **Part VI — Vercel & the Web** | Prerequisites: Chapters 30, 29 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- AI agents are not replacing frontend architects -- they are replacing the tedious 80% (boilerplate, tests, documentation, refactoring) so you can focus on architecture, UX decisions, and complex business logic
- Claude Code is the most capable agentic coding tool in 2026; configure it with CLAUDE.md (project context), skills (reusable workflows), hooks (automated guardrails), and MCP servers (external tool access) for maximum leverage
- The modern design-to-code pipeline -- Figma MCP + Claude Code + Storybook + tests -- reduces implementation time from days to hours while maintaining quality
- Use Cursor/Windsurf for in-flow editing and tab completion; use Claude Code for multi-step autonomous tasks, refactoring, and complex debugging; use Copilot for repetitive boilerplate
- AI-generated code needs the same review rigor you would apply to a junior engineer's PR: trust the structure, verify the logic, always check security-critical paths manually

</details>

Here is the uncomfortable truth about frontend development in 2026: the engineer who refuses to use AI tools is not principled -- they are slow. And in a world where your competitors are shipping features in hours that used to take days, slow is a strategic liability.

But here is the equally uncomfortable truth that the AI hype machine does not want you to hear: **the engineer who blindly trusts AI tools ships more bugs, introduces more security vulnerabilities, and creates more technical debt than the engineer who writes everything by hand.** The AI does not understand your business logic. It does not know why that seemingly redundant null check exists. It does not grasp that the "simpler" refactoring it suggests will break the payment flow on Android 12 devices with slow network connections.

The frontend architect who thrives in 2026 is neither the AI refuser nor the AI worshipper. They are the **AI conductor** -- someone who understands exactly what AI is good at, exactly what it is bad at, and exactly how to configure it to amplify their judgment rather than replace it.

This chapter is about becoming that conductor. Not a tutorial on prompting -- a comprehensive guide to building AI-powered workflows that make you and your team dramatically faster without sacrificing the quality, security, and architectural coherence that separates production-grade software from demo-grade software.

### In This Chapter
- The Agentic Revolution: What AI Actually Changes (and What It Does Not)
- Claude Code Deep Dive: CLAUDE.md, Skills, Hooks, MCP Servers
- Practical Claude Code Workflows for Frontend Development
- Cursor and Windsurf: IDE-Integrated AI
- GitHub Copilot: When It Helps, When It Hurts
- AI Code Review: Automated PR Analysis
- AI-Generated Tests: From Components to E2E Flows
- AI Component Generation: v0, Figma-to-Code, Design Specs
- The Design-to-Code Pipeline: Days to Hours
- Multi-Agent Development: Parallel Workflows
- Prompt Engineering for Code: What Actually Works
- AI-Assisted Debugging: Crashes, Performance, Build Failures
- AI-Assisted Documentation: JSDoc, ADRs, README Generation
- The AI-Augmented Architect: How Your Role Changes
- Risks and Boundaries: When NOT to Trust AI

### Related Chapters
- [Ch 27: Developer Experience & Tooling] -- foundational DX setup, Biome, Storybook, Claude Code basics
- [Ch 26: The AI-Powered Frontend] -- building AI features into your product (AI SDK, streaming, tool calling)
- [Ch 17: Testing Strategy] -- the testing patterns AI will help you write
- [Ch 16: Design Systems] -- the component library AI will generate into
- [Ch 18: CI/CD] -- the pipeline where AI review and AI tests run

---

## 1. THE AGENTIC REVOLUTION

Let me draw a line that too many people blur: **Chapter 26 taught you to build AI features for your users. This chapter teaches you to use AI features for yourself.**

The AI-powered frontend (Chapter 26) is about integrating LLMs into your product -- chat interfaces, content generation, structured output. Agentic UI development is about integrating LLMs into your *workflow* -- writing code, reviewing PRs, generating tests, debugging crashes, implementing designs.

These are entirely different disciplines. Building AI features requires understanding streaming protocols, token budgets, and prompt engineering for end users. Using AI for development requires understanding code generation quality, agentic tool use, and the boundary between what AI can decide and what only a human should decide.

### 1.1 What AI Actually Replaces

The frontend development workflow has always had a split:

```
THE 80/20 SPLIT IN FRONTEND ENGINEERING
========================================

THE TEDIOUS 80% (AI handles this well)        THE CREATIVE 20% (still yours)
-----------------------------------------------+------------------------------
Writing boilerplate components                  | Architecture decisions
Connecting API endpoints to UI                  | UX trade-offs
Writing unit and integration tests              | Performance budget decisions
Updating types after API changes                | Security model design
Refactoring patterns you have done before       | Navigation flow design
Writing JSDoc and documentation                 | State architecture choices
Converting designs to component code            | Complex business logic
Fixing lint and type errors                     | Accessibility judgment
Writing Storybook stories                       | Team workflow design
Creating CRUD screens                           | Novel algorithm design
Updating dependencies and migration code        | Cross-platform trade-offs
Writing CSS/Tailwind for standard layouts       | Design system philosophy
```

The 80% is not "easy" work. It is work that requires knowledge, attention, and correctness. It is just *predictable* work. There is a right answer, and an experienced engineer can recognize it. That is exactly the kind of work AI excels at -- pattern matching against enormous training data to produce the conventionally correct solution.

The 20% is where human judgment is irreplaceable. Should this component use local state or global state? Should we prefetch this data or lazy-load it? Should this screen exist at all, or should we redesign the flow? Is this API shape going to cause problems when we add the multi-tenant feature next quarter? AI cannot answer these questions because they depend on context, constraints, and vision that exists in your head and your team's heads, not in training data.

### 1.2 The Velocity Multiplier

Here is a concrete example. Before AI-assisted development:

```
MANUAL WORKFLOW: Implement a settings screen
=============================================
1. Read the Figma design                                    10 min
2. Create the screen component file                          2 min
3. Build the form layout (15 form fields)                   45 min
4. Wire up form state with React Hook Form                  20 min
5. Connect to API endpoints (3 mutations)                   30 min
6. Add loading, error, and success states                   20 min
7. Add form validation with Zod schemas                     25 min
8. Write unit tests for validation logic                    30 min
9. Write integration tests for form submission              40 min
10. Write Storybook stories for each form state             25 min
11. Add accessibility labels and roles                      15 min
12. Handle keyboard navigation and focus management         20 min
                                                    ──────────────
TOTAL                                                4 hours 42 min
```

With AI-assisted development (Claude Code with proper configuration):

```
AI-ASSISTED WORKFLOW: Same settings screen
==========================================
1. Read the Figma design (human)                            10 min
2. Prompt Claude Code with Figma link + requirements         5 min
3. Claude Code generates component, form, validation,
   API connection, loading/error states                     10 min
4. Review generated code (human)                            15 min
5. Adjust UX details Claude missed (human)                  15 min
6. Prompt Claude Code to generate tests                      3 min
7. Claude Code generates unit + integration tests            5 min
8. Review tests, adjust assertions (human)                  10 min
9. Prompt Claude Code for Storybook stories                  2 min
10. Claude Code generates stories                            3 min
11. Final review and manual QA (human)                      15 min
                                                    ──────────────
TOTAL                                                1 hour 33 min
```

That is a 3x speedup. And the human time -- the actual thinking and reviewing -- is about 70 minutes. The rest is Claude Code executing. You spend your time on judgment, not typing.

Extrapolate this across a sprint. A team of four engineers, each saving 2-3 hours per day on boilerplate, tests, and documentation. That is 40-60 extra engineering hours per week. That is the difference between shipping the feature and shipping the feature plus the performance optimization plus the accessibility audit plus the documentation update.

### 1.3 The New Skill Stack

The skills that make a frontend engineer effective are shifting:

```
OLD SKILL STACK (2020-2024)            NEW SKILL STACK (2025+)
================================       ================================
Typing speed matters                   Prompt precision matters
Memorize API surfaces                  Know what to ask for
Write code from scratch                Review and refine generated code
Manual debugging (console.log)         AI-assisted debugging + judgment
Read docs linearly                     Query docs conversationally
Build components bottom-up             Describe components top-down
Write tests after implementation       Generate tests, verify assertions
Manual code review                     AI-assisted review + human judgment
```

Notice what is NOT on the new stack: "let AI do everything." The new stack is heavily weighted toward judgment, review, and refinement. You are spending less time producing first drafts and more time evaluating them. That is a fundamentally different skill.

---

## 2. CLAUDE CODE FOR FRONTEND DEVELOPMENT

Chapter 27 introduced Claude Code as one tool among many. This section goes deep -- because Claude Code in 2026 is not just a tool. It is a development environment extension that, properly configured, transforms how you build frontend applications.

### 2.1 What Makes Claude Code Different

There are many AI coding tools. Here is why Claude Code occupies a unique position for frontend development:

**It is agentic, not reactive.** Cursor and Copilot respond to what you are doing. Claude Code *does things*. You give it a task, and it reads files, searches the codebase, writes code, runs commands, checks for errors, and iterates. It is the difference between an assistant who hands you tools and an assistant who uses the tools.

**It has full codebase context.** Claude Code does not operate on a single file. It can read any file in your project, understand the relationships between modules, trace import chains, and make changes that are consistent across the codebase. For a React Native + Next.js monorepo with hundreds of files, this context awareness is transformative.

**It runs in your terminal.** Claude Code has access to your shell environment. It can run your linter, execute your tests, start your dev server, check TypeScript types, install packages, and interact with Git. This means it can verify its own work -- generate code, run the tests, fix what fails, and repeat.

**It is configurable.** CLAUDE.md, skills, hooks, and MCP servers let you shape Claude Code's behavior to your specific project. A well-configured Claude Code instance knows your conventions, follows your patterns, and catches mistakes specific to your architecture.

### 2.2 Setup and Installation

```bash
# Install Claude Code globally
npm install -g @anthropic-ai/claude-code

# Navigate to your project
cd ~/projects/my-app

# Start Claude Code
claude

# Or start with a specific task
claude "explain the architecture of this project"
```

Claude Code reads your project structure on startup. But raw project structure is not enough. You need to tell it how your project actually works.

### 2.3 CLAUDE.md — The Project Brain

The `CLAUDE.md` file at your project root is the single most impactful configuration for Claude Code. It is the difference between Claude Code that writes generic React code and Claude Code that writes code that looks like *your team* wrote it.

Here is a comprehensive `CLAUDE.md` for a production React Native + Next.js monorepo:

```markdown
# CLAUDE.md

## Project Overview
Nelo is a cross-platform fintech application. Mobile app built with Expo SDK 53
and React Native (New Architecture enabled). Web app built with Next.js 16 App
Router. Monorepo managed with pnpm workspaces and Turborepo.

## Monorepo Structure
```
apps/
  mobile/          — Expo Router app (iOS + Android)
  web/             — Next.js 16 App Router
  admin/           — Next.js internal admin dashboard
packages/
  ui/              — Shared component library (Tamagui + React Native)
  api/             — tRPC router definitions
  db/              — Drizzle ORM schema + migrations (Neon Postgres)
  auth/            — Shared authentication logic (Clerk)
  config/          — Shared ESLint, TypeScript, Tailwind configs
  utils/           — Shared utility functions
  types/           — Shared TypeScript type definitions
```

## Tech Stack
- **Runtime**: React Native 0.78, Expo SDK 53, Next.js 16
- **Language**: TypeScript 5.7 (strict mode, no `any` allowed)
- **State**: Zustand for client state, TanStack Query v5 for server state
- **Styling**: Tamagui for cross-platform, Tailwind CSS v4 for web-only
- **API**: tRPC v11 with Zod validation on every input
- **Database**: Drizzle ORM + Neon Postgres
- **Auth**: Clerk with custom session tokens
- **Testing**: Vitest (unit), RNTL (component), Maestro (mobile E2E), Playwright (web E2E)
- **CI/CD**: GitHub Actions, EAS Build/Submit (mobile), Vercel (web)
- **Linting**: Biome (replaces ESLint + Prettier)
- **Package Manager**: pnpm 10

## Commands
- `pnpm dev` — Start all apps in development (Turborepo)
- `pnpm dev:mobile` — Start Expo dev server only
- `pnpm dev:web` — Start Next.js dev server only
- `pnpm test` — Run all Vitest tests
- `pnpm test:e2e:mobile` — Run Maestro E2E tests
- `pnpm test:e2e:web` — Run Playwright tests
- `pnpm lint` — Run Biome check across all packages
- `pnpm typecheck` — Run TypeScript type checking
- `pnpm db:push` — Push Drizzle schema to database
- `pnpm db:generate` — Generate Drizzle migrations
- `pnpm db:studio` — Open Drizzle Studio

## Key Conventions
- **Named exports only** — never use default exports except for Next.js pages
- **Barrel exports** — every package has an `index.ts` that re-exports public API
- **Component files** — one component per file, file name matches component name
- **Hooks** — custom hooks go in `hooks/` directory, prefixed with `use`
- **Zustand stores** — one store per domain (`useAuthStore`, `useCartStore`), defined in `stores/`
- **TanStack Query** — query keys are typed constants in `queryKeys.ts`, mutations use `useMutation` with `onSettled` invalidation
- **Error handling** — all screens wrapped in `ErrorBoundary`, API errors surfaced through `useQuery` error states
- **Imports** — use path aliases (`@/components`, `@repo/ui`) not relative paths beyond one level
- **No inline styles** — use Tamagui tokens or Tailwind classes, never `style={{ }}`

## Code Style
- Function components only (no class components)
- Prefer `const` over `let`, never `var`
- Use `interface` for component props, `type` for unions and intersections
- Destructure props in function signature: `function Button({ label, onPress }: ButtonProps)`
- Use early returns for guard clauses
- No `useEffect` for data fetching — use TanStack Query
- No `useEffect` for derived state — use `useMemo` or compute inline
- `useCallback` and `useMemo` only when measured performance requires it

## Mobile-Specific
- Navigation: Expo Router file-based routing with typed routes
- Deep linking: configured in `app.json` with custom scheme `nelo://`
- OTA updates: EAS Update with staged rollouts (10% → 50% → 100%)
- Minimum OS: iOS 16, Android 12 (API 31)
- The app uses Hermes engine on both platforms

## Web-Specific
- Server Components by default, `"use client"` only when needed
- Use `next/image` for all images, never raw `<img>`
- Metadata exported from `layout.tsx` and `page.tsx` files
- API routes in `app/api/` use tRPC adapters
- Vercel Fluid Compute enabled for all serverless functions

## Security Rules
- NEVER put API keys, secrets, or tokens in client-side code
- All payment-related code must be reviewed by a human (no AI-only changes)
- Authentication logic changes require manual review
- Use Clerk's middleware for route protection, never roll custom auth
- Sanitize all user input before rendering (XSS prevention)
- All database queries use Drizzle's parameterized queries (SQL injection prevention)

## Testing Conventions
- Test files live next to source files: `Component.test.tsx`
- Use RNTL `screen` queries: `getByText`, `getByRole`, `getByTestId`
- Mock API calls with MSW, not manual fetch mocks
- E2E tests cover critical paths: auth, checkout, onboarding
- Minimum coverage: 80% for `packages/`, 60% for `apps/`

## Things Claude Should Know
- The app processes real money. Be extra careful with anything touching
  the payment flow, balance calculations, or transaction history.
- We use Drizzle ORM, not Prisma. Do not suggest Prisma patterns.
- Our tRPC routers return typed errors with error codes. Do not use
  generic `throw new Error()`.
- The mobile app supports offline mode. Any data mutation must handle
  the offline queue in `packages/utils/offlineQueue.ts`.
- When creating new screens, always add analytics tracking using the
  `useAnalytics` hook from `packages/utils/analytics`.
- The admin dashboard has different auth — it uses Clerk organization
  roles, not individual user auth.
```

This is roughly 100 lines of context. It takes 30 minutes to write well, and it saves hours every single day. Every time Claude Code reads your codebase, it starts with this file. Every decision it makes is filtered through these conventions. The difference between a project with a good `CLAUDE.md` and a project without one is dramatic -- it is the difference between getting code that matches your patterns and getting generic code that you have to manually rewrite.

### 2.4 Nested CLAUDE.md Files

For monorepos, you can place additional `CLAUDE.md` files in subdirectories. Claude Code reads the root file plus any `CLAUDE.md` in the directory it is currently working in:

```markdown
<!-- apps/mobile/CLAUDE.md -->
# Mobile App Context

## Navigation Structure
Tab navigator with 5 tabs: Home, Search, Portfolio, Activity, Profile.
Each tab has a stack navigator for drill-down screens.
Modal screens are presented over tabs using Expo Router groups.

## Screen Template
Every new screen should follow this pattern:
1. Create screen file in the appropriate tab directory
2. Add `ErrorBoundary` wrapper
3. Add `useAnalytics().trackScreen('ScreenName')` in useEffect
4. Use `SafeAreaView` from `react-native-safe-area-context`
5. Use `useHeaderHeight()` for scroll content inset

## Platform-Specific Code
- Use `Platform.select()` for small differences
- Use `.ios.tsx` / `.android.tsx` file extensions for large differences
- Always test on both platforms before marking a task complete

## Common Pitfalls
- Do NOT use `position: absolute` for bottom-fixed elements on iOS --
  it breaks with the keyboard. Use `KeyboardAvoidingView`.
- The `StatusBar` component must be `translucent` on Android for
  proper safe area handling.
- Expo Router's `useLocalSearchParams` returns strings -- always parse
  numeric IDs with `Number()`.
```

```markdown
<!-- apps/web/CLAUDE.md -->
# Web App Context

## Route Structure
App Router with route groups:
- `(marketing)` — public pages (landing, pricing, blog)
- `(dashboard)` — authenticated app pages
- `(auth)` — sign-in, sign-up, forgot-password
- `api/` — tRPC and webhook handlers

## Data Fetching Patterns
- Server Components: fetch data directly with tRPC server caller
- Client Components: use TanStack Query with tRPC hooks
- Never mix: do not call tRPC from a Server Component via the
  client-side hook

## Caching Strategy
- Static pages: ISR with 60-second revalidation
- Dynamic pages: no-store with streaming Suspense boundaries
- API routes: Cache-Control headers set per-endpoint

## Performance Budget
- LCP < 2.5s on 4G connection
- Bundle size < 250KB first-load JS
- No layout shift (CLS = 0) on any page
```

### 2.5 Skills — Reusable Workflows

Skills are markdown files in `.claude/skills/` that encode your team's workflows. When Claude Code needs to perform a specific task, it can reference these skills for step-by-step instructions.

Here is a set of production-grade skills for a frontend monorepo:

```markdown
<!-- .claude/skills/create-screen.md -->
# Create New Screen

When creating a new screen for the mobile app:

## Steps
1. Determine which tab/stack the screen belongs to
2. Create the route file in `apps/mobile/app/(tabs)/[tab]/[screen].tsx`
3. Use the screen template:

```tsx
import { SafeAreaView } from 'react-native-safe-area-context';
import { Stack } from 'expo-router';
import { ErrorBoundary } from '@repo/ui';
import { useAnalytics } from '@repo/utils/analytics';

export default function ScreenNameScreen() {
  const analytics = useAnalytics();

  useEffect(() => {
    analytics.trackScreen('ScreenName');
  }, []);

  return (
    <ErrorBoundary>
      <Stack.Screen options={{ title: 'Screen Title' }} />
      <SafeAreaView style={{ flex: 1 }}>
        {/* Screen content */}
      </SafeAreaView>
    </ErrorBoundary>
  );
}
```

4. If the screen fetches data, use TanStack Query hooks
5. Add the screen to the navigation type definitions in `types/navigation.ts`
6. Create a basic test in `__tests__/ScreenName.test.tsx`
7. If the screen has a form, use React Hook Form + Zod validation
```

```markdown
<!-- .claude/skills/create-api-endpoint.md -->
# Create tRPC Endpoint

When creating a new API endpoint:

## Steps
1. Determine which router the endpoint belongs to (or create a new one)
2. Add the procedure to the router in `packages/api/src/routers/`
3. Every procedure MUST have:
   - Zod input schema
   - Zod output schema
   - Proper auth check (protectedProcedure for authenticated routes)
   - Typed error handling with error codes
4. Add the router to the app router in `packages/api/src/root.ts` if new
5. Run `pnpm typecheck` to verify types propagate to consumers
6. Write a test for the endpoint in `packages/api/src/__tests__/`

## Template:
```typescript
export const featureRouter = createTRPCRouter({
  getItem: protectedProcedure
    .input(z.object({ id: z.string().uuid() }))
    .output(itemSchema)
    .query(async ({ ctx, input }) => {
      const item = await ctx.db.query.items.findFirst({
        where: eq(items.id, input.id),
      });
      if (!item) {
        throw new TRPCError({
          code: 'NOT_FOUND',
          message: `Item ${input.id} not found`,
        });
      }
      return item;
    }),
});
```
```

```markdown
<!-- .claude/skills/refactor-to-zustand.md -->
# Refactor Redux/Context to Zustand

When migrating state management to Zustand:

## Steps
1. Identify all state, actions, and selectors in the existing store
2. Create the Zustand store in `stores/use[Domain]Store.ts`
3. Map Redux actions to Zustand actions (methods on the store)
4. Map Redux selectors to Zustand selectors (exported functions)
5. Use `useShallow` for multi-value selections to prevent re-renders
6. Replace `useSelector` / `useContext` calls in components
7. Remove the old Redux slice / Context provider
8. Run `pnpm test` to verify all tests still pass
9. Update any Storybook stories that mock the old store

## Zustand Store Template:
```typescript
import { create } from 'zustand';
import { useShallow } from 'zustand/react/shallow';

interface DomainState {
  items: Item[];
  isLoading: boolean;
  error: string | null;
  // Actions
  fetchItems: () => Promise<void>;
  addItem: (item: Item) => void;
  reset: () => void;
}

export const useDomainStore = create<DomainState>((set, get) => ({
  items: [],
  isLoading: false,
  error: null,

  fetchItems: async () => {
    set({ isLoading: true, error: null });
    try {
      const items = await api.getItems();
      set({ items, isLoading: false });
    } catch (error) {
      set({ error: error.message, isLoading: false });
    }
  },

  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),

  reset: () => set({ items: [], isLoading: false, error: null }),
}));

// Selectors
export const useItemCount = () =>
  useDomainStore((state) => state.items.length);
```
```

```markdown
<!-- .claude/skills/write-tests.md -->
# Write Tests for a Component

When writing tests for a React Native component:

## Rules
- Use RNTL (React Native Testing Library), not Enzyme
- Import `screen` and `render` from `@testing-library/react-native`
- Query by text, role, or testID -- NEVER by component type
- Mock API calls with MSW handlers, not manual fetch mocks
- Test user-visible behavior, not implementation details
- Each test should have a clear arrange-act-assert structure
- Use `waitFor` for async assertions, not arbitrary timeouts
- Group related tests with `describe` blocks
- Use `userEvent` from `@testing-library/react-native` for interactions

## Test Template:
```typescript
import { render, screen, waitFor } from '@testing-library/react-native';
import userEvent from '@testing-library/user-event';
import { server } from '@/test/msw-server';
import { http, HttpResponse } from 'msw';
import { ComponentName } from './ComponentName';
import { QueryWrapper } from '@/test/QueryWrapper';

describe('ComponentName', () => {
  it('renders the initial state correctly', () => {
    render(<ComponentName />, { wrapper: QueryWrapper });
    expect(screen.getByText('Expected Text')).toBeOnTheScreen();
  });

  it('handles user interaction', async () => {
    const user = userEvent.setup();
    render(<ComponentName />, { wrapper: QueryWrapper });

    await user.press(screen.getByRole('button', { name: 'Submit' }));

    await waitFor(() => {
      expect(screen.getByText('Success')).toBeOnTheScreen();
    });
  });

  it('displays error state when API fails', async () => {
    server.use(
      http.get('/api/data', () => {
        return HttpResponse.json(
          { error: 'Server error' },
          { status: 500 }
        );
      })
    );

    render(<ComponentName />, { wrapper: QueryWrapper });

    await waitFor(() => {
      expect(screen.getByText('Something went wrong')).toBeOnTheScreen();
    });
  });
});
```
```

### 2.6 Hooks — Automated Guardrails

Hooks run automatically when Claude Code takes specific actions. They are your safety net -- ensuring that every file Claude writes is formatted, every change passes type checking, and every modification follows your rules.

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "echo 'Files being modified — will auto-format after write'"
      },
      {
        "matcher": "Bash",
        "command": "echo 'Command execution — checking for dangerous operations'",
        "description": "Warn before shell commands"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm biome check --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true",
        "description": "Auto-format any file Claude writes or edits"
      },
      {
        "matcher": "Write|Edit",
        "command": "pnpm tsc --noEmit --pretty false 2>&1 | head -20",
        "description": "Quick type check after file changes"
      }
    ],
    "Stop": [
      {
        "command": "pnpm typecheck && pnpm lint",
        "description": "Full type check and lint when Claude finishes a task"
      }
    ]
  }
}
```

The `PostToolUse` hooks are especially powerful. Every time Claude Code writes or edits a file, Biome automatically formats it and TypeScript checks it for type errors. This means Claude Code gets immediate feedback -- if it writes code with a type error, it sees the error and can fix it in the same conversation.

The `Stop` hook runs a comprehensive check when Claude finishes a task. If the type check or lint fails, Claude Code sees the failures and can ask whether to fix them.

#### Advanced Hook Patterns

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "if echo \"$CLAUDE_FILE_PATH\" | grep -q 'payments\\|auth\\|security'; then echo '⚠️ SECURITY-SENSITIVE FILE MODIFIED: $CLAUDE_FILE_PATH — Requires human review'; fi",
        "description": "Flag security-sensitive file modifications"
      },
      {
        "matcher": "Write",
        "command": "if echo \"$CLAUDE_FILE_PATH\" | grep -qE '\\.test\\.(tsx?|jsx?)$'; then cd $(dirname \"$CLAUDE_FILE_PATH\") && pnpm vitest run $(basename \"$CLAUDE_FILE_PATH\") --reporter=verbose 2>&1 | tail -20; fi",
        "description": "Auto-run test files after writing them"
      }
    ]
  }
}
```

The first hook flags any modification to files in payment, auth, or security directories. This is a critical guardrail -- AI should not be making unchecked changes to security-sensitive code.

The second hook automatically runs a test file after Claude Code writes it. This creates a rapid feedback loop: Claude writes a test, the test runs, Claude sees if it passes or fails, and can iterate.

### 2.7 MCP Servers — External Tool Access

MCP (Model Context Protocol) servers extend Claude Code's capabilities by connecting it to external tools and data sources. For frontend development, several MCP servers are transformative:

```json
// .claude/mcp.json
{
  "mcpServers": {
    "figma": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/figma-mcp-server"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "${FIGMA_ACCESS_TOKEN}"
      }
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/github-mcp-server"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "sentry": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@sentry/mcp-server"],
      "env": {
        "SENTRY_AUTH_TOKEN": "${SENTRY_AUTH_TOKEN}",
        "SENTRY_ORG": "nelo",
        "SENTRY_PROJECT": "mobile-app"
      }
    },
    "postgres": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/postgres-mcp-server"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

**Figma MCP** is the most impactful for frontend development. It lets Claude Code read Figma designs directly -- component properties, layout structure, colors, typography, spacing. Combined with a good `CLAUDE.md` that specifies your design system tokens, Claude Code can generate components that match your Figma designs with remarkable fidelity.

**Sentry MCP** lets Claude Code read error reports, stack traces, and crash data directly. Instead of copying and pasting a stack trace into the conversation, you say "look at the top crash in Sentry" and Claude Code reads it, traces it through your codebase, and proposes a fix.

**GitHub MCP** lets Claude Code read issues, PRs, and code reviews. "Implement the feature described in issue #234" becomes a single-prompt workflow.

**Postgres MCP** lets Claude Code understand your data model by querying the actual database. When it generates a component that displays user data, it can check the actual schema to make sure the field names and types are correct.

### 2.8 Configuring for a React Native + Next.js Monorepo

The full configuration for a production monorepo:

```
project-root/
├── CLAUDE.md                          # Root project context
├── .claude/
│   ├── settings.json                  # Hooks configuration
│   ├── mcp.json                       # MCP server connections
│   └── skills/
│       ├── create-screen.md           # New screen workflow
│       ├── create-component.md        # New component workflow
│       ├── create-api-endpoint.md     # New tRPC endpoint workflow
│       ├── write-tests.md             # Test writing guidelines
│       ├── refactor-to-zustand.md     # Redux → Zustand migration
│       ├── debug-crash.md             # Crash debugging workflow
│       ├── performance-audit.md       # Performance review workflow
│       └── pr-review.md              # PR review checklist
├── apps/
│   ├── mobile/
│   │   └── CLAUDE.md                  # Mobile-specific context
│   └── web/
│       └── CLAUDE.md                  # Web-specific context
└── packages/
    └── ui/
        └── CLAUDE.md                  # Design system context
```

This configuration takes about an hour to set up thoroughly. That hour pays for itself within the first day of use.

---

## 3. CLAUDE CODE WORKFLOWS

Theory is useless without practice. Here are the workflows that a frontend architect uses daily with Claude Code, each with real prompts and expected results.

### 3.1 Workflow: Build a Screen From a Figma Design

```
PROMPT:
"Implement the Profile Settings screen from this Figma design:
https://figma.com/file/abc123/designs?node-id=456

It should:
- Use React Hook Form for form state
- Validate with Zod (email format, name min 2 chars, phone optional)
- Call the user.updateProfile tRPC mutation on submit
- Show a success toast on save
- Handle offline mode using the offline queue
- Match our existing Settings screens pattern (see apps/mobile/app/(tabs)/profile/settings.tsx)
- Include proper accessibility labels for all form fields"
```

What Claude Code does:
1. Reads the Figma design via MCP (layout, colors, spacing, typography)
2. Reads `CLAUDE.md` for conventions and tech stack
3. Reads the existing settings screen for pattern reference
4. Reads the tRPC `user.updateProfile` mutation for the input schema
5. Generates the screen component with React Hook Form + Zod
6. Generates the validation schema matching the tRPC input
7. Adds offline queue integration
8. Adds accessibility labels
9. Runs Biome format (via hook)
10. Runs type check (via hook)
11. Generates a test file
12. Generates a Storybook story

**What you review:** The UX decisions -- does the form feel right? Are the error messages user-friendly? Is the keyboard behavior correct? Does the design match pixel-for-pixel? These are judgment calls that Claude cannot make.

### 3.2 Workflow: Write Tests for a Component

```
PROMPT:
"Write comprehensive tests for the TransactionCard component in
packages/ui/src/components/TransactionCard.tsx

Cover:
- Rendering with different transaction types (credit, debit, pending)
- Formatting of currency amounts
- Tap interaction (should navigate to transaction detail)
- Accessibility labels
- Edge cases: very long merchant names, zero amounts, future dates
- Loading skeleton state"
```

What Claude Code generates:

```typescript
// packages/ui/src/components/__tests__/TransactionCard.test.tsx
import { render, screen } from '@testing-library/react-native';
import userEvent from '@testing-library/user-event';
import { TransactionCard } from '../TransactionCard';

const mockTransaction = {
  id: 'txn_123',
  type: 'debit' as const,
  amount: -4599, // cents
  currency: 'USD',
  merchant: 'Whole Foods Market',
  date: new Date('2026-04-05T14:30:00Z'),
  status: 'completed' as const,
};

const mockOnPress = jest.fn();

describe('TransactionCard', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('rendering', () => {
    it('displays merchant name and formatted amount for debit', () => {
      render(
        <TransactionCard
          transaction={mockTransaction}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByText('Whole Foods Market')).toBeOnTheScreen();
      expect(screen.getByText('-$45.99')).toBeOnTheScreen();
    });

    it('displays positive amount with plus sign for credit', () => {
      render(
        <TransactionCard
          transaction={{ ...mockTransaction, type: 'credit', amount: 10000 }}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByText('+$100.00')).toBeOnTheScreen();
    });

    it('shows pending indicator for pending transactions', () => {
      render(
        <TransactionCard
          transaction={{ ...mockTransaction, status: 'pending' }}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByText('Pending')).toBeOnTheScreen();
    });

    it('formats date relative to today', () => {
      const today = new Date();
      render(
        <TransactionCard
          transaction={{ ...mockTransaction, date: today }}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByText('Today')).toBeOnTheScreen();
    });
  });

  describe('interaction', () => {
    it('calls onPress with transaction ID when tapped', async () => {
      const user = userEvent.setup();
      render(
        <TransactionCard
          transaction={mockTransaction}
          onPress={mockOnPress}
        />
      );

      await user.press(screen.getByRole('button'));
      expect(mockOnPress).toHaveBeenCalledWith('txn_123');
    });
  });

  describe('accessibility', () => {
    it('has accessible label describing the transaction', () => {
      render(
        <TransactionCard
          transaction={mockTransaction}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByRole('button')).toHaveAccessibleName(
        'Whole Foods Market, debit $45.99, April 5'
      );
    });
  });

  describe('edge cases', () => {
    it('truncates very long merchant names', () => {
      const longName = 'A'.repeat(100);
      render(
        <TransactionCard
          transaction={{ ...mockTransaction, merchant: longName }}
          onPress={mockOnPress}
        />
      );

      const merchantText = screen.getByTestId('merchant-name');
      expect(merchantText.props.numberOfLines).toBe(1);
    });

    it('displays zero amount correctly', () => {
      render(
        <TransactionCard
          transaction={{ ...mockTransaction, amount: 0 }}
          onPress={mockOnPress}
        />
      );

      expect(screen.getByText('$0.00')).toBeOnTheScreen();
    });
  });

  describe('loading state', () => {
    it('renders skeleton when loading', () => {
      render(<TransactionCard.Skeleton />);

      expect(screen.getByTestId('transaction-skeleton')).toBeOnTheScreen();
    });
  });
});
```

**What you review:** Are the assertions meaningful? Does the test actually verify the behavior users care about, or is it testing implementation details? Are the edge cases the right edge cases for your business domain? AI-generated tests tend to test the obvious paths well but miss the subtle business logic edge cases.

### 3.3 Workflow: Refactor a State Store

```
PROMPT:
"Refactor the Redux auth store in apps/mobile/src/store/authSlice.ts to Zustand.

Requirements:
- Preserve all existing functionality
- Keep the same public API (action names, selector shapes) to minimize
  component changes
- Add TypeScript strict types for the store
- Use the persist middleware for token storage
- Handle the token refresh flow that currently uses a Redux thunk
- Update all components that import from the old store
- Update all tests
- Remove the old Redux slice and provider when done

Follow the refactor-to-zustand skill."
```

Claude Code will:
1. Read the `refactor-to-zustand` skill for step-by-step instructions
2. Read the entire Redux auth slice to understand state, actions, and selectors
3. Search all files that import from the auth slice
4. Create the new Zustand store with matching API
5. Update each component file to use the new store
6. Update each test file
7. Run tests after each file change
8. Remove the Redux slice and any redux-toolkit imports
9. Clean up the Redux store configuration if this was the last slice
10. Run full test suite to verify

This is the kind of refactoring that takes a human engineer a full day. Claude Code does it in 10-15 minutes. And because it runs tests after each change, you get incremental verification that nothing broke.

### 3.4 Workflow: Review a PR for Performance Issues

```
PROMPT:
"Review PR #456 for performance issues. Focus on:
- Unnecessary re-renders (missing memoization, unstable references)
- Bundle size impact (new dependencies, large imports)
- Network waterfall issues (sequential fetches that could be parallel)
- Memory leaks (subscriptions without cleanup, timers without clear)
- React Native thread blocking (heavy computation on JS thread)

Read the PR diff from GitHub and give me a detailed review."
```

Claude Code reads the PR via GitHub MCP and produces a structured review:

```markdown
## Performance Review: PR #456 — Add Portfolio Analytics Screen

### Critical Issues

1. **Unstable callback in FlatList renderItem** (PortfolioChart.tsx:45)
   The `renderItem` callback is created inline, causing FlatList to
   re-render every item on every parent render. Wrap in `useCallback`
   or extract to a named component.

2. **Sequential API calls** (usePortfolioData.ts:12-24)
   `fetchHoldings()` and `fetchPerformance()` are awaited sequentially.
   They are independent — use `Promise.all()` to parallelize.
   Estimated savings: 200-400ms on initial load.

### Moderate Issues

3. **Full lodash import** (utils/calculations.ts:1)
   `import _ from 'lodash'` imports the entire 72KB library.
   Change to `import groupBy from 'lodash/groupBy'` for tree-shaking.
   Bundle impact: ~68KB reduction.

4. **Missing cleanup in useEffect** (PortfolioChart.tsx:67)
   The ResizeObserver is created but never disconnected. This will
   leak memory if the component unmounts during an observation.

### Minor Issues

5. **Inline style object** (AnalyticsCard.tsx:23)
   `style={{ marginTop: 12 }}` creates a new object every render.
   Use a Tamagui token or extract to a constant.

### Looks Good
- TanStack Query usage is correct with proper stale times
- The Zustand store updates are properly batched
- Skeleton loading states prevent layout shift
```

### 3.5 Workflow: Debug a Production Crash

```
PROMPT:
"We're seeing this crash in Sentry affecting 3% of users on iOS.
Look at Sentry issue MOBILE-1234 via the Sentry MCP, trace it
through the codebase, and tell me what is happening and how to fix it."
```

Claude Code:
1. Reads the Sentry issue via MCP (stack trace, device info, breadcrumbs, user context)
2. Identifies the crashing line in the codebase
3. Reads the surrounding code and traces the data flow
4. Checks the breadcrumbs for the sequence of events leading to the crash
5. Identifies the root cause
6. Proposes and optionally implements a fix
7. Writes a regression test

```
CLAUDE CODE RESPONSE:

## Root Cause Analysis: MOBILE-1234

**Crash:** TypeError: Cannot read property 'balance' of undefined
**Location:** apps/mobile/src/screens/Portfolio/PortfolioSummary.tsx:87
**Affected:** 3.2% of iOS users, 0% Android

### What's Happening
The crash occurs in `PortfolioSummary` when `usePortfolioQuery` returns
data where `portfolio.accounts[0]` is undefined. This happens for new
users who have completed onboarding but have not yet linked a bank
account. The accounts array is empty.

### Stack Trace Path
1. User opens Portfolio tab → PortfolioScreen mounts
2. usePortfolioQuery fetches /api/portfolio
3. API returns { accounts: [], totalBalance: 0 }
4. PortfolioSummary.tsx:87 accesses `data.accounts[0].balance`
5. Crash: accounts[0] is undefined

### Why iOS Only
Android users see a loading spinner longer due to a separate rendering
bug (the `isLoaded` flag resolves slower on Android), which gives the
accounts data time to populate from a secondary fetch. The iOS crash
is actually the correct behavior being exposed — the code assumes
accounts are always present.

### Fix
```typescript
// Before (line 87)
const primaryBalance = data.accounts[0].balance;

// After
const primaryBalance = data.accounts[0]?.balance ?? 0;
```

But the real fix is to handle the empty-accounts state explicitly:

```typescript
if (data.accounts.length === 0) {
  return <LinkBankAccountCTA />;
}
```

### Regression Test Written
Added test case for empty accounts array in PortfolioSummary.test.tsx
```

This kind of debugging -- reading error reports, tracing through code, understanding platform-specific behavior, proposing both the quick fix and the proper fix -- is where Claude Code provides enormous value. A human engineer might spend 30-60 minutes on this investigation. Claude Code does it in 2-3 minutes.

---

## 4. CURSOR AND WINDSURF

Claude Code is the most powerful tool for autonomous, multi-step tasks. But it runs in a terminal, separate from your editor. For the kind of AI assistance that happens *while you are typing code* -- inline completions, quick edits, conversational exploration of a file you are looking at -- IDE-integrated tools are better.

### 4.1 Cursor

Cursor is a fork of VS Code with deep AI integration. It is the best IDE-integrated AI tool for frontend development in 2026.

**Tab completion.** Cursor's tab completion is context-aware. It reads the file you are editing, the files you have open, and your recent edits to predict what you want to type next. For frontend development, this means:

- Start typing a component prop interface and Cursor completes it based on how the component is used elsewhere in the codebase
- Write one handler function and Cursor suggests the pattern for the next three
- Start a test case and Cursor completes the assertion based on what the component actually renders

**Inline editing (Cmd+K).** Highlight a block of code, press Cmd+K, and describe what you want changed. Cursor edits the code inline, showing you a diff:

```
Highlight a function → Cmd+K → "add error handling for the case where
the API returns a 429 rate limit response, with exponential backoff"

Cursor shows you the diff inline. Accept or reject.
```

**Multi-file editing (Composer).** Cursor's Composer mode can make coordinated changes across multiple files. This is useful for cross-cutting changes:

```
"Add a lastUpdated timestamp to the UserProfile type, update the
tRPC router to return it, update the Zustand store to store it,
and update the ProfileScreen to display it"
```

Composer edits all four files and shows you the diffs.

**Chat with codebase context.** Cursor's chat panel lets you ask questions about your codebase with full context. It indexes your project and can find relevant code that you might not think to look for:

```
"How does the offline queue work? Walk me through the flow from
when a user makes a transaction while offline to when it syncs."
```

### 4.2 Windsurf

Windsurf (formerly Codeium) is another AI-powered IDE, similar to Cursor but with a different approach to multi-file editing. Its "Cascade" feature is an agentic flow within the IDE -- you describe a task, and Windsurf creates a step-by-step plan, executes it across files, and shows you the results.

The key difference from Cursor: Windsurf's Cascade is more explicitly agentic. It shows you a plan before executing, lets you approve or modify steps, and maintains context across a longer conversation. It sits between Cursor's inline editing and Claude Code's fully autonomous execution.

### 4.3 When Cursor vs. Claude Code

```
USE CURSOR WHEN:                         USE CLAUDE CODE WHEN:
=========================================+==========================================
You are in the middle of writing code     | You need to create something from scratch
  and want inline suggestions             |   (new screen, new feature, new module)
                                          |
You need a quick edit to a function       | You need a multi-step refactoring that
  you are looking at right now            |   touches many files
                                          |
You want to explore what a piece of       | You need to debug a crash by tracing
  code does conversationally              |   through multiple files
                                          |
You need tab completion while typing      | You need to generate tests for an
  repetitive patterns                     |   entire module
                                          |
You want inline diff review of a          | You need to review a PR with
  specific change                         |   codebase-wide context
                                          |
You are pair programming and want         | You want to delegate a complete task
  real-time assistance                    |   and review the result
```

The practical workflow: **use Cursor as your editor, Claude Code as your architect.** Cursor handles the moment-to-moment coding assistance. Claude Code handles the planning, scaffolding, refactoring, and review.

Many engineers run both simultaneously -- Cursor open in their IDE, Claude Code running in a terminal beside it. The IDE handles the "flow state" editing. The terminal handles the "thinking" work.

---

## 5. GITHUB COPILOT

GitHub Copilot is the most widely adopted AI coding tool, integrated into every major editor. It is also the most misunderstood -- because it is excellent at some things and actively harmful at others.

### 5.1 Where Copilot Excels

**Boilerplate completion.** Copilot is exceptional at completing patterns you have written before. Write the first test case, and Copilot suggests the next five in the same pattern. Write the first form field, and Copilot completes the remaining fields. Write the first API handler, and Copilot generates handlers for the other CRUD operations.

**Comment-driven generation.** Write a comment describing a function, and Copilot generates the implementation:

```typescript
// Format a price in cents to a display string with currency symbol
// Examples: 1099 → "$10.99", 500 → "$5.00", 0 → "$0.00"
function formatPrice(cents: number, currency: string = 'USD'): string {
  // Copilot generates a reasonable implementation here
}
```

**Known patterns.** Copilot is trained on millions of repositories. For common patterns -- React hooks, Express middleware, Zod schemas, Tailwind classes -- it generates correct code quickly. If you are writing code that thousands of other engineers have written before, Copilot knows the pattern.

**Import statements.** Copilot is surprisingly good at suggesting the correct import statement when you start typing a function or component name. It scans your project to determine which module to import from.

### 5.2 Where Copilot Hurts

**Novel architecture.** When you are implementing something that does not follow a common pattern -- a custom state machine, a novel animation sequence, a bespoke offline sync algorithm -- Copilot suggests code that *looks* correct but is wrong. It pattern-matches to the nearest common pattern, which may have entirely different semantics than what you need.

**Security-sensitive code.** Copilot does not understand security. It will suggest code that is vulnerable to XSS, SQL injection, or authentication bypass because those vulnerable patterns exist in its training data. Never accept Copilot suggestions in authentication, payment, or authorization code without careful review.

**Large context reasoning.** Copilot works at the file level. It does not understand how your component fits into the broader architecture. It does not know that the function it is completing is called from five different contexts with different invariants. For changes that require understanding of the broader system, Copilot's suggestions are often locally correct but globally wrong.

**Outdated patterns.** Copilot's training data includes old patterns. It will suggest `useEffect` for data fetching when you should use TanStack Query. It will suggest `getServerSideProps` when you should use App Router server components. It will suggest class components. Always verify that Copilot's suggestions use your project's current patterns.

### 5.3 Copilot for PRs

GitHub Copilot for Pull Requests can generate PR descriptions, suggest review comments, and summarize changes. It is useful for:

- **Auto-generating PR descriptions** from commit messages and diffs
- **Suggesting test cases** you might have missed
- **Flagging potential issues** in the diff (unused variables, missing error handling)

It is not useful for:
- Understanding whether the change is architecturally sound
- Evaluating UX quality
- Catching business logic errors

### 5.4 Copilot Configuration for Frontend

```json
// .github/copilot-instructions.md
# Copilot Instructions

## Project Context
This is a React Native + Next.js monorepo using TypeScript.

## Preferences
- Use named exports, not default exports
- Use Zustand for state management, not Redux
- Use TanStack Query for data fetching, not useEffect
- Use Biome for linting, not ESLint
- Use Tamagui for styling in shared components
- Use Tailwind CSS for web-only components
- Always add TypeScript types — never use `any`

## Do Not Suggest
- Class components
- getServerSideProps or getStaticProps (use App Router patterns)
- Redux or MobX patterns
- Enzyme testing patterns (use RNTL)
- Moment.js (use date-fns)
- Lodash full imports (use individual imports)
```

This configuration file tells Copilot about your project's conventions, reducing the frequency of outdated or wrong suggestions.

---

## 6. AI CODE REVIEW

Code review is one of the highest-leverage activities a frontend architect performs. It is also one of the most time-consuming. AI code review does not replace human review -- it augments it by catching the mechanical issues so the human reviewer can focus on design and logic.

### 6.1 What AI Catches Well

```
HIGH AI REVIEW ACCURACY                    LOW AI REVIEW ACCURACY
==========================================+==========================================
Bug patterns                               | Business logic correctness
  - Null/undefined access                  |   "Should this amount be negative?"
  - Off-by-one errors                      |   "Is this the right formula?"
  - Missing await on async functions       |   "Should this user see this data?"
  - Unclosed resources                     |
                                           | UX quality
Performance issues                         |   "Is this interaction intuitive?"
  - Missing memoization                    |   "Does this flow make sense?"
  - Unnecessary re-renders                 |   "Is this error message helpful?"
  - Bundle size regressions                |
  - Sequential async where parallel works  | Architecture appropriateness
                                           |   "Is this the right abstraction?"
Accessibility violations                   |   "Will this scale to 10x users?"
  - Missing ARIA labels                    |   "Does this belong in this module?"
  - Missing keyboard navigation            |
  - Insufficient color contrast            | Security nuances
  - Missing alt text                       |   "Is this auth check sufficient?"
                                           |   "Can this be exploited via timing?"
Style violations                           |
  - Inconsistent naming                    | Product fit
  - Import order issues                    |   "Does this match the spec?"
  - Missing types                          |   "Is this what the user actually needs?"
```

### 6.2 Setting Up AI Review in CI

**Claude in GitHub (via Claude Code GitHub Action):**

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Claude Code Review
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          model: claude-sonnet-4-20250514
          prompt: |
            Review this PR for:
            1. Bugs and potential runtime errors
            2. Performance issues (re-renders, bundle size, memory leaks)
            3. Accessibility violations
            4. TypeScript type safety
            5. Consistency with project conventions in CLAUDE.md

            For each issue, provide:
            - File and line number
            - Severity (critical, moderate, minor)
            - Specific suggestion for fixing

            Do NOT comment on style issues (Biome handles that).
            Do NOT comment on test coverage (we track that separately).
            Focus on issues that would cause bugs or degrade quality.
```

**CodeRabbit:**

CodeRabbit provides automated AI code review with deep understanding of codebases. It comments directly on PRs with specific, actionable feedback:

```yaml
# .coderabbit.yaml
language: en-US
reviews:
  profile: assertive
  path_instructions:
    - path: "apps/mobile/**"
      instructions: |
        Review with React Native and Expo best practices in mind.
        Check for platform-specific issues (iOS vs Android).
        Verify accessibility with RNTL patterns.
    - path: "packages/ui/**"
      instructions: |
        This is the shared design system. Changes here affect all apps.
        Check for cross-platform compatibility (web + mobile).
        Verify Tamagui usage follows our token system.
    - path: "**/*.test.*"
      instructions: |
        Verify tests test behavior, not implementation.
        Check for meaningful assertions (not just "renders without crashing").
        Ensure proper async handling with waitFor.
  auto_review:
    enabled: true
    drafts: false
```

### 6.3 The Human + AI Review Workflow

The optimal workflow is not "AI reviews, then human reviews." It is "AI reviews first, flags mechanical issues, human reviews what AI flagged plus the design/logic layer."

```
PULL REQUEST OPENED
        │
        ▼
┌──────────────────┐
│   CI Runs         │  Lint, type check, tests, bundle size check
└──────┬───────────┘
        │
        ▼
┌──────────────────┐
│   AI Review       │  CodeRabbit / Claude reviews the diff
│                   │  Flags bugs, perf issues, a11y, types
└──────┬───────────┘
        │
        ▼
┌──────────────────┐
│   Author Fixes    │  Author addresses AI feedback
│   AI Feedback     │  (Some may be false positives — dismiss those)
└──────┬───────────┘
        │
        ▼
┌──────────────────┐
│   Human Review    │  Architect reviews:
│                   │  - Architecture decisions
│                   │  - Business logic correctness
│                   │  - UX quality
│                   │  - Security implications
│                   │  - Remaining AI flags they disagree with
└──────┬───────────┘
        │
        ▼
┌──────────────────┐
│   Merge           │
└──────────────────┘
```

This workflow reduces human review time by 30-50% because the human reviewer does not need to catch the mechanical issues. They can focus entirely on the things that require human judgment.

### 6.4 Dealing with False Positives

AI code review will produce false positives -- issues that are technically correct but wrong for your specific context. A few common ones in frontend:

- "This useCallback is unnecessary" -- but it is necessary because the callback is passed to a PureComponent in a FlatList
- "This try/catch is too broad" -- but it is intentionally broad because this is an error boundary handler
- "This function is too long" -- but it is a state machine transition function that is clearer as one linear flow

When you get a false positive, do not just dismiss it. Add context to your `CLAUDE.md` or PR review configuration so the AI learns. Over time, your AI review becomes more accurate because it has more context about your specific patterns and decisions.

---

## 7. AI-GENERATED TESTS

Tests are one of the highest-value outputs of AI code generation. Writing tests is time-consuming, the patterns are repetitive, and the quality of AI-generated tests is often very good -- with one critical caveat.

### 7.1 The Quality Bar

AI-generated tests need human review for meaningful assertions. Here is the common failure mode:

```typescript
// AI-generated test — LOOKS good but IS NOT
it('renders the payment form', () => {
  render(<PaymentForm />);
  expect(screen.getByTestId('payment-form')).toBeOnTheScreen();
});
```

This test proves that the component renders without crashing. It does not prove that the payment form works correctly. A human reviewer needs to ask: "What assertions actually matter here?"

```typescript
// Human-reviewed version — actually tests behavior
it('submits payment with valid card details', async () => {
  const user = userEvent.setup();
  const onSuccess = jest.fn();

  render(
    <PaymentForm onSuccess={onSuccess} />,
    { wrapper: QueryWrapper }
  );

  await user.type(
    screen.getByLabelText('Card number'),
    '4242424242424242'
  );
  await user.type(screen.getByLabelText('Expiry'), '12/27');
  await user.type(screen.getByLabelText('CVC'), '123');
  await user.press(screen.getByRole('button', { name: 'Pay $49.99' }));

  await waitFor(() => {
    expect(onSuccess).toHaveBeenCalledWith(
      expect.objectContaining({ status: 'succeeded' })
    );
  });
});
```

The rule: **AI generates the test structure, you verify the assertions.**

### 7.2 Generating Jest/RNTL Tests From Component Code

```
PROMPT:
"Generate tests for the CheckoutScreen component at
apps/mobile/app/(tabs)/cart/checkout.tsx

Test the following user flows:
1. Complete checkout with valid address and payment method
2. Checkout with invalid address (show validation errors)
3. Checkout when cart is empty (redirect to home)
4. Checkout with network failure (show retry option)
5. Apply a promo code successfully
6. Apply an invalid promo code (show error message)
7. Remove item from cart during checkout (update totals)

Use the write-tests skill. Mock the API with MSW handlers."
```

Claude Code generates a comprehensive test file, using MSW for API mocking, testing real user flows rather than implementation details, and covering both happy paths and error cases.

### 7.3 Generating Maestro E2E Flows

Maestro is the leading mobile E2E testing framework. Its YAML-based test files are straightforward enough that AI generates them very well:

```
PROMPT:
"Generate Maestro E2E flows for the user onboarding journey:
1. User opens app for the first time
2. Sees welcome screen, taps 'Get Started'
3. Enters name and email
4. Verifies email with OTP code
5. Sets up a PIN
6. Confirms PIN
7. Sees the home screen with empty state

Follow our existing Maestro flows in e2e/flows/ for patterns.
Handle loading states with waitForAnimation.
Use testIDs from the actual components."
```

```yaml
# e2e/flows/onboarding.yaml
appId: com.nelo.app
---
- launchApp:
    clearState: true

# Welcome screen
- assertVisible: "Welcome to Nelo"
- tapOn: "Get Started"

# Name and email
- assertVisible: "Create your account"
- tapOn:
    id: "input-name"
- inputText: "Test User"
- tapOn:
    id: "input-email"
- inputText: "test@example.com"
- tapOn: "Continue"

# Email verification
- assertVisible: "Verify your email"
- waitForAnimationToEnd
- tapOn:
    id: "otp-input-0"
- inputText: "123456"
- waitForAnimationToEnd

# PIN setup
- assertVisible: "Set up your PIN"
- tapOn:
    id: "pin-input"
- inputText: "1234"

# PIN confirmation
- assertVisible: "Confirm your PIN"
- tapOn:
    id: "pin-input"
- inputText: "1234"

# Home screen
- waitForAnimationToEnd
- assertVisible: "Welcome, Test User"
- assertVisible: "Link your bank account"
```

### 7.4 Generating Playwright Tests

For web applications, AI excels at generating Playwright tests from page descriptions:

```
PROMPT:
"Generate Playwright tests for the Dashboard page at apps/web/app/(dashboard)/page.tsx

Test:
1. Dashboard loads with portfolio summary cards
2. Clicking a portfolio card navigates to detail page
3. Date range picker filters chart data
4. Export button downloads CSV
5. Mobile responsive layout (cards stack vertically)
6. Unauthenticated user redirects to sign-in

Use our Playwright config in playwright.config.ts. Use page objects if we
have them, otherwise create them."
```

```typescript
// e2e/tests/dashboard.spec.ts
import { test, expect } from '@playwright/test';
import { DashboardPage } from '../pages/DashboardPage';
import { SignInPage } from '../pages/SignInPage';

test.describe('Dashboard', () => {
  let dashboard: DashboardPage;

  test.beforeEach(async ({ page }) => {
    dashboard = new DashboardPage(page);
    await dashboard.goto();
    await dashboard.waitForLoad();
  });

  test('displays portfolio summary cards', async () => {
    await expect(dashboard.totalBalanceCard).toBeVisible();
    await expect(dashboard.totalBalanceCard).toContainText('$');
    await expect(dashboard.performanceCard).toBeVisible();
    await expect(dashboard.holdingsCard).toBeVisible();
  });

  test('navigates to portfolio detail on card click', async ({ page }) => {
    await dashboard.totalBalanceCard.click();
    await expect(page).toHaveURL(/\/portfolio\/.+/);
  });

  test('filters chart data by date range', async () => {
    await dashboard.dateRangePicker.click();
    await dashboard.selectDateRange('Last 30 days');
    await expect(dashboard.chart).toBeVisible();
    // Chart should update — verify the date labels changed
    await expect(dashboard.chartDateLabel).toContainText('30 days');
  });

  test('exports data as CSV', async ({ page }) => {
    const downloadPromise = page.waitForEvent('download');
    await dashboard.exportButton.click();
    const download = await downloadPromise;
    expect(download.suggestedFilename()).toMatch(/portfolio.*\.csv/);
  });

  test('stacks cards vertically on mobile', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 812 });
    const cards = dashboard.summaryCards;
    const boxes = await Promise.all(
      (await cards.all()).map((card) => card.boundingBox())
    );
    // Verify vertical stacking: each card's top is below the previous
    for (let i = 1; i < boxes.length; i++) {
      expect(boxes[i]!.y).toBeGreaterThan(boxes[i - 1]!.y);
    }
  });
});

test.describe('Dashboard — unauthenticated', () => {
  test('redirects to sign-in', async ({ page }) => {
    // Clear auth cookies
    await page.context().clearCookies();
    await page.goto('/dashboard');
    await expect(page).toHaveURL(/\/sign-in/);
  });
});
```

---

## 8. AI COMPONENT GENERATION

The three main paths for AI component generation: Vercel v0 for shadcn/ui components, Figma-to-code for design-driven development, and Claude Code for custom components from specs.

### 8.1 Vercel v0

v0 by Vercel generates React components from natural language descriptions. It produces shadcn/ui + Tailwind CSS components that are production-ready and copy-pasteable.

**When to use v0:**
- You need a UI component that follows standard web patterns
- You are prototyping and want a visual starting point
- You need shadcn/ui components styled for a specific use case
- You want to explore different UI approaches for a feature

**When NOT to use v0:**
- You need React Native components (v0 generates web-only code)
- You need components that integrate deeply with your state management
- You need cross-platform components
- You need components with complex business logic

**Effective v0 prompts:**

```
// Bad prompt — too vague
"Create a dashboard"

// Good prompt — specific and constrained
"Create a financial portfolio dashboard card that shows:
- Total balance in large text ($XX,XXX.XX format)
- 24h change as percentage with green/red color
- A sparkline chart showing 7-day performance
- An icon indicating the asset type (stock, crypto, bond)

Use shadcn/ui Card component, Tailwind CSS, and Recharts for the
sparkline. Dark mode support. Responsive — full width on mobile,
1/3 width on desktop."
```

The key to effective v0 prompts: specify the data shape, the visual hierarchy, the interaction model, and the constraints (responsive behavior, dark mode, component library).

### 8.2 Figma-to-Code with AI

The Figma MCP server enables a workflow that was impossible before: Claude Code reads a Figma design and generates component code directly from the design properties.

```
PROMPT:
"Implement the UserProfileCard component from Figma:
https://figma.com/file/abc123?node-id=789

Use Tamagui for styling (cross-platform).
Match the spacing, typography, and colors exactly.
Map Figma colors to our design token system in packages/ui/tokens.ts.
The component should accept a User type from packages/types.
Add an onPress prop for the entire card.
Include avatar, name, username, bio (max 2 lines), and follower count."
```

Claude Code via Figma MCP:
1. Reads the Figma node properties (layout, auto-layout, fills, typography)
2. Maps Figma auto-layout to Tamagui flex properties
3. Maps Figma colors to the closest design tokens
4. Maps Figma typography to the closest text variants
5. Generates a component that matches the design structure

The result is not pixel-perfect -- it never is, and it should not be. It is 80-90% of the way there, and the remaining 10-20% is fine-tuning that a human does in minutes instead of building from scratch in hours.

### 8.3 Generating React Native Components From Design Specs

When you do not have Figma access or prefer to describe the component verbally:

```
PROMPT:
"Create a TransactionListItem component for our mobile app:

Visual spec:
- Horizontal layout: icon (40x40) | text column | amount column
- Icon: category icon with colored circular background
- Text column: merchant name (bold, 1 line) + date/time (gray, smaller)
- Amount column: right-aligned, bold, red for debits, green for credits
- Bottom border separator (1px, gray-100)
- Entire row is pressable with subtle scale animation on press
- Height: 72px with 16px horizontal padding

Props:
- transaction: Transaction type from our types package
- onPress: (id: string) => void

Behavior:
- Haptic feedback on press (soft impact)
- Swipe-to-delete with confirmation
- Long press shows action menu (copy amount, share, dispute)

Follow the component patterns in our ui package.
Use Tamagui for styling.
Use react-native-reanimated for the press animation."
```

The more specific your prompt about visual specs, the better the output. Think of it like a design spec document -- dimensions, colors, typography, spacing, interaction behavior.

### 8.4 Prompt Engineering That Produces Good Components

```
COMPONENT PROMPT CHECKLIST:
===========================

1. STRUCTURE: What is the layout? (flex direction, alignment, wrapping)
2. DATA: What are the props? (types, optional/required, defaults)
3. VISUAL: Colors, typography, spacing (use your token names)
4. INTERACTION: What happens on press/hover/focus/swipe?
5. STATES: Loading, empty, error, disabled, focused
6. ANIMATION: What moves, how fast, what easing?
7. RESPONSIVE: How does it change at different sizes?
8. ACCESSIBILITY: Labels, roles, keyboard behavior
9. PLATFORM: Any iOS/Android/web differences?
10. PATTERN: Reference an existing component for style consistency

The more of these 10 you specify, the better the generated code.
Skip any and AI will guess — sometimes correctly, sometimes not.
```

---

## 9. THE DESIGN-TO-CODE PIPELINE

The modern frontend workflow collapses the gap between design and implementation using AI. Here is the full pipeline:

```
DESIGN-TO-CODE PIPELINE
========================

Designer creates              Developer triggers
in Figma                      Claude Code
    │                             │
    ▼                             ▼
┌───────────┐  Figma MCP   ┌─────────────┐
│           │──────────────▶│             │
│  Figma    │  reads design │ Claude Code │
│  Design   │               │             │
│           │◀──────────────│  generates  │
└───────────┘  designer     │  code       │
                reviews     └──────┬──────┘
                                   │
                         ┌─────────┼─────────┐
                         ▼         ▼         ▼
                    ┌─────────┐ ┌──────┐ ┌──────┐
                    │Component│ │ Test │ │Story │
                    │  .tsx   │ │.test │ │.story│
                    └────┬────┘ └──┬───┘ └──┬───┘
                         │        │        │
                         ▼        ▼        ▼
                    ┌──────────────────────────┐
                    │    Developer Reviews      │
                    │  - Visual fidelity check  │
                    │  - UX judgment calls      │
                    │  - Business logic verify  │
                    │  - Accessibility audit    │
                    └────────────┬─────────────┘
                                │
                         ┌──────┴──────┐
                         ▼             ▼
                    ┌─────────┐  ┌──────────┐
                    │Storybook│  │  CI/CD   │
                    │validates│  │  runs    │
                    │ visually│  │  tests   │
                    └─────────┘  └──────────┘
                                      │
                                      ▼
                              ┌──────────────┐
                              │  Deployed to  │
                              │  Preview URL  │
                              └──────────────┘
```

### 9.1 The Before and After

**Before AI pipeline (traditional):**

```
Day 1: Designer finishes design in Figma
Day 1: Designer hands off to developer with a link
Day 2: Developer reads design, asks clarifying questions
Day 2: Developer starts building component structure
Day 2: Developer spends 3 hours on layout and styling
Day 3: Developer wires up state and API connections
Day 3: Developer adds loading/error states
Day 3: Developer writes tests
Day 4: Design review — "the spacing is wrong, the font size is
        different on the header, the tap target is too small"
Day 4: Developer fixes design feedback
Day 5: Code review — reviewer catches performance issue and
        missing error handling
Day 5: Developer addresses code review
Day 5: Merged and deployed
                                        TOTAL: ~5 days
```

**After AI pipeline:**

```
Day 1, 9:00 AM: Designer finishes design in Figma
Day 1, 9:30 AM: Developer prompts Claude Code with Figma link
Day 1, 9:45 AM: Claude Code generates component + tests + story
Day 1, 10:30 AM: Developer reviews, adjusts UX details
Day 1, 11:00 AM: Storybook visual review with designer (looks good)
Day 1, 11:30 AM: AI code review catches a missing memoization
Day 1, 12:00 PM: Human code review (architecture looks good)
Day 1, 1:00 PM: Merged and deployed
                                        TOTAL: ~4 hours
```

### 9.2 Making It Work in Practice

The pipeline only works well when:

1. **Your Figma designs use a consistent design system** that maps to your code tokens. If the Figma uses arbitrary colors and spacing, the AI has nothing to map to.

2. **Your CLAUDE.md documents the design token mapping.** Claude Code needs to know that Figma's "Primary/500" maps to your code's `$color.primary` token.

3. **Your component library has established patterns.** Claude Code generates code that matches existing patterns. If your components have no consistent structure, the generated code will also be inconsistent.

4. **You invest time in skills.** A "create-component" skill that encodes your exact component structure, testing patterns, and storybook format means every generated component starts from the right template.

The pipeline does NOT work when:
- Designs are ad-hoc with no system (AI has no patterns to follow)
- The developer accepts AI output without review (bugs ship)
- The team skips the Storybook review step (visual bugs ship)
- Security-sensitive components go through the pipeline without extra scrutiny

---

## 10. MULTI-AGENT DEVELOPMENT

One of the most powerful patterns in 2026 is using multiple AI agents working in parallel. This is not science fiction -- Claude Code supports subagent execution, and the workflow is practical today.

### 10.1 The Plan-Dispatch-Review Pattern

```
MULTI-AGENT WORKFLOW
====================

    ┌──────────────────────┐
    │   ARCHITECT (you)    │
    │                      │
    │   1. Define the task │
    │   2. Break into parts│
    │   3. Set constraints │
    └──────────┬───────────┘
               │
        ┌──────┴──────┐
        │  PLANNER    │  (Claude Code main session)
        │  agent      │
        │             │  - Reads codebase
        │             │  - Creates implementation plan
        │             │  - Identifies independent tasks
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐
│AGENT 1 │ │AGENT 2 │ │AGENT 3 │  (subagents — parallel)
│        │ │        │ │        │
│Creates │ │Creates │ │Creates │
│screen  │ │API     │ │tests   │
│components│ │endpoints│ │for all │
│        │ │        │ │modules │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
    └──────────┼──────────┘
               │
        ┌──────┴──────┐
        │  REVIEWER   │  (Claude Code review pass)
        │  agent      │
        │             │  - Checks consistency
        │             │  - Verifies types compile
        │             │  - Runs full test suite
        │             │  - Reports issues
        └──────┬──────┘
               │
               ▼
    ┌──────────────────────┐
    │   ARCHITECT (you)    │
    │                      │
    │   Final review       │
    │   Merge decision     │
    └──────────────────────┘
```

### 10.2 Practical Example: Building a Feature Module

```
PROMPT TO PLANNER AGENT:
"We need to build the Notifications feature for the mobile app.

Requirements:
- Notification list screen with pull-to-refresh
- Notification detail screen
- Push notification handling (foreground + background)
- Notification preferences screen (toggle by type)
- tRPC endpoints: list, markRead, markAllRead, updatePreferences
- Zustand store for notification state + unread count badge
- Tests for all components and endpoints
- Maestro E2E flow for the main notification journey

Break this into independent tasks that can be worked on in parallel.
Identify any dependencies between tasks."
```

The planner analyzes the requirements and produces:

```
PLAN:
=====

Independent tasks (can run in parallel):
1. tRPC endpoints (no UI dependency)
2. Zustand store (depends on types, but not on UI or API implementation)
3. Notification preferences screen (standalone form)
4. Push notification handler (standalone service)

Sequential tasks (depend on the above):
5. Notification list screen (needs store + API)
6. Notification detail screen (needs store + API)
7. Integration tests (needs all components)
8. Maestro E2E flow (needs all screens)

Dispatching agents 1-4 in parallel...
```

Claude Code dispatches four subagents, each working on their independent task simultaneously. When all four complete, the planner dispatches tasks 5 and 6 (which depend on the store and API), then 7 and 8.

### 10.3 When Multi-Agent Works vs. Does Not

```
GOOD FIT FOR MULTI-AGENT:
- Building a feature with clear, independent modules
- Writing tests for multiple components simultaneously
- Refactoring multiple independent stores
- Generating documentation for multiple packages
- Creating multiple screens that do not share state

BAD FIT FOR MULTI-AGENT:
- Deeply interconnected changes where one file affects many others
- Refactoring a single complex module (needs sequential reasoning)
- Debugging (requires following a single thread of investigation)
- Changes to shared types (all agents need the same type definitions)
- Changes to navigation structure (affects all screens)
```

The key principle: **multi-agent works when the tasks are independent.** If agent 2 needs to know what agent 1 did, they should not run in parallel.

---

## 11. PROMPT ENGINEERING FOR CODE

The difference between a prompt that produces usable code and a prompt that produces garbage is not length -- it is specificity. Here are the patterns that work and the anti-patterns that waste your time.

### 11.1 The Specificity Spectrum

```
USELESS PROMPT:
"Make a login screen"

VAGUE PROMPT:
"Create a login screen with email and password"

DECENT PROMPT:
"Create a login screen with email and password fields, a submit
button, and error handling. Use React Hook Form and Zod validation."

EXCELLENT PROMPT:
"Create the login screen for our Expo Router mobile app at
apps/mobile/app/(auth)/sign-in.tsx

Requirements:
- Email field with keyboard type 'email-address' and autocomplete
- Password field with secure text entry and show/hide toggle
- 'Sign In' button that calls auth.signIn tRPC mutation
- 'Forgot Password' link that navigates to /(auth)/forgot-password
- 'Create Account' link that navigates to /(auth)/sign-up
- Form validation with Zod: email must be valid, password min 8 chars
- Error display below each field for validation errors
- General error banner for API errors (wrong credentials, rate limit)
- Loading state on submit button (disabled + spinner)
- Keyboard avoiding behavior for iOS
- Biometric login option if device supports it (use expo-local-authentication)

Follow the pattern in our existing /(auth)/sign-up.tsx screen.
Use the useAuthStore for session management after successful login.
Track 'sign_in_screen_view' and 'sign_in_success' analytics events."
```

The excellent prompt specifies the exact file path, references existing patterns, names the specific libraries and APIs, describes the visual structure, enumerates all interactive states, and connects to the broader app architecture.

### 11.2 Prompt Patterns That Work

**Reference existing code:**
```
"Follow the pattern in src/screens/ProfileScreen.tsx"
"Use the same form structure as the CheckoutForm component"
"Match the API error handling pattern in src/utils/apiClient.ts"
```

Referencing existing code is the single most effective technique. Claude Code reads the referenced file and replicates the patterns. This ensures consistency without you having to describe every convention.

**Describe behavior, not just features:**
```
// Bad: feature-oriented
"Add a search feature"

// Good: behavior-oriented
"Add a search bar that:
- Appears at the top of the screen
- Debounces input by 300ms before making API calls
- Shows a loading indicator while fetching
- Displays results in a FlatList below the search bar
- Shows 'No results' empty state
- Preserves the search query when navigating to a result and back
- Clears on long-press of the X button"
```

**Specify error handling explicitly:**
```
"Handle these error cases:
- Network offline: show offline banner with retry button
- API 401: redirect to login screen, clear auth state
- API 429: show 'Too many requests, try again in X seconds'
- API 500: show generic error with 'Contact support' option
- Validation error: show inline field errors below each field"
```

AI tends to generate happy-path code. If you do not specify error handling, you will get code that works perfectly when everything goes right and crashes when anything goes wrong.

**Set explicit constraints:**
```
"Do NOT use useEffect for data fetching — use TanStack Query"
"Do NOT use any — all types must be explicit"
"Do NOT install new dependencies — use what we already have"
"Do NOT use default exports"
```

Negative constraints are extremely useful. They prevent AI from defaulting to common patterns that violate your project's conventions.

### 11.3 Anti-Patterns

**Too much at once:**
```
// Bad: asking for an entire feature in one prompt
"Build the entire e-commerce checkout flow with cart, shipping address,
payment method selection, order review, confirmation, and order tracking"

// Good: break it into sequential prompts
"Build the CartScreen first. It shows line items from the useCartStore,
a subtotal, and a 'Proceed to Checkout' button."

// Then:
"Now build the ShippingAddressScreen. It takes the user to a form
for entering their shipping address. The form saves to the useCheckoutStore."
```

**No context about existing patterns:**
```
// Bad: Claude Code guesses your patterns
"Create a data fetching hook"

// Good: Claude Code follows your patterns
"Create a data fetching hook following the pattern in
src/hooks/useTransactions.ts — use TanStack Query with our
standard query key structure and error mapping"
```

**Assuming AI knows your business domain:**
```
// Bad: domain-specific term without explanation
"Calculate the portfolio NAV using the waterfall method"

// Good: explain the business logic
"Calculate the portfolio NAV (Net Asset Value):
- Sum all holding values (quantity * current price)
- Subtract any pending fees
- The 'waterfall method' means: first allocate returns to preferred
  investors up to their hurdle rate, then split remaining between
  GP (20%) and LPs (80%)"
```

---

## 12. AI-ASSISTED DEBUGGING

Debugging is where AI provides some of its most dramatic time savings. The pattern is simple: give AI the error context, your codebase, and let it trace the problem.

### 12.1 The Debugging Conversation Pattern

Every AI-assisted debugging session follows this structure:

```
1. PRESENT THE ERROR (with full context)
   - Stack trace or error message
   - Where it happened (screen, action, device)
   - Reproduction steps if known
   - How many users are affected

2. AI INVESTIGATES
   - Reads the crashing code
   - Traces the call chain
   - Checks recent changes
   - Identifies the root cause

3. AI PROPOSES FIX
   - Shows the minimal fix
   - Explains why it works
   - Shows the proper fix if different from minimal
   - Suggests regression test

4. HUMAN VALIDATES
   - Does the root cause make sense?
   - Is the fix correct?
   - Are there other places with the same bug?
   - Does the regression test cover the case?
```

### 12.2 Debugging Production Crashes

```
PROMPT:
"We have a Sentry issue with 450 occurrences in the last 24 hours:

TypeError: Cannot read properties of null (reading 'map')
  at TransactionList (TransactionList.tsx:34)
  at PortfolioScreen (PortfolioScreen.tsx:78)
  at Route(/(tabs)/portfolio)

Device breakdown: 80% iOS, 20% Android
App version: 2.4.1
User segment: all users, but more frequent for new users

Investigate and propose a fix."
```

Claude Code response:

```
## Investigation

1. Reading TransactionList.tsx:34 — the line is:
   `{transactions.map(tx => <TransactionItem key={tx.id} .../>)}`

2. The `transactions` variable comes from:
   `const { transactions } = usePortfolioStore()`

3. The store initializes transactions as `null`:
   `transactions: null as Transaction[] | null`

4. The query that populates transactions is in usePortfolioQuery:
   ```typescript
   const { data } = useQuery({
     queryKey: ['portfolio', userId],
     queryFn: () => api.portfolio.get()
   });
   // On success, sets store: usePortfolioStore.setState({ transactions: data.transactions })
   ```

5. Race condition: TransactionList renders before the query completes.
   The store has `null`, and `.map()` is called on null.

## Why More Frequent for New Users
New users have no cached data, so the query takes longer. Returning users
may have stale data that loads instantly from TanStack Query cache.

## Why 80% iOS
iOS renders the initial frame faster than Android, hitting the race
condition more frequently. Android's slower initial render gives the
query more time to complete.

## Fix

Option A (Quick fix):
```typescript
// TransactionList.tsx:34
{transactions?.map(tx => <TransactionItem key={tx.id} .../>) ?? <EmptyState />}
```

Option B (Proper fix — initialize with empty array, not null):
```typescript
// usePortfolioStore.ts
transactions: [] as Transaction[], // was: null as Transaction[] | null
isTransactionsLoaded: false,       // use explicit flag for "has data been fetched"
```

This way, `.map()` never fails, and you check `isTransactionsLoaded`
for distinguishing "no data yet" from "data loaded but empty."

## Regression Test
```typescript
it('renders empty state when transactions list is empty', () => {
  usePortfolioStore.setState({ transactions: [], isTransactionsLoaded: true });
  render(<TransactionList />);
  expect(screen.getByText('No transactions yet')).toBeOnTheScreen();
});

it('renders loading state before transactions load', () => {
  usePortfolioStore.setState({ transactions: [], isTransactionsLoaded: false });
  render(<TransactionList />);
  expect(screen.getByTestId('transaction-skeleton')).toBeOnTheScreen();
});
```
```

### 12.3 Debugging Performance Issues

```
PROMPT:
"The Portfolio screen is janky when scrolling. Here is the Flipper
performance profile showing frame drops.

The FlatList has ~200 items. I see JS frame times of 40-80ms
(should be <16ms for 60fps). The flamechart shows most time
spent in TransactionItem re-renders.

What is causing the re-renders and how do we fix it?"
```

Claude Code reads the component code and identifies:

```
## Root Cause Analysis

1. TransactionItem receives `onPress` as an inline arrow function:
   ```tsx
   renderItem={({ item }) => (
     <TransactionItem
       transaction={item}
       onPress={() => handlePress(item.id)}  // NEW FUNCTION EVERY RENDER
     />
   )}
   ```
   This creates a new function reference on every render, causing
   every TransactionItem to re-render on every scroll frame.

2. TransactionItem is not memoized. Even with stable props, it
   would still re-render when the parent re-renders.

3. The store selector `usePortfolioStore(state => state)` selects
   the entire store, causing re-render on ANY store change, not
   just transaction changes.

## Fix

1. Memoize the callback:
   ```tsx
   const handleItemPress = useCallback((id: string) => {
     router.push(`/transaction/${id}`);
   }, []);

   renderItem={({ item }) => (
     <TransactionItem
       transaction={item}
       onPress={handleItemPress}
     />
   )}
   ```

2. Memoize TransactionItem:
   ```tsx
   export const TransactionItem = memo(function TransactionItem({
     transaction,
     onPress,
   }: TransactionItemProps) {
     // ...
   });
   ```

3. Fix the store selector:
   ```tsx
   // Before
   const { transactions } = usePortfolioStore(state => state);

   // After
   const transactions = usePortfolioStore(state => state.transactions);
   ```

Expected improvement: JS frame time from 40-80ms to <16ms.
The FlatList will only re-render items that actually changed.
```

### 12.4 Debugging Build Failures

```
PROMPT:
"Our CI build is failing with this error. I have no idea what is
going on. Diagnose and fix:

Error: Module not found: Can't resolve '@repo/ui/tokens'
  in '/apps/web/src/components/ThemeProvider.tsx'

This was working yesterday. The only change was updating
packages/ui/package.json to add a new export."
```

Claude Code:

```
## Diagnosis

Reading the recent change to packages/ui/package.json...

The new export was added without the correct `exports` map entry:

```json
// packages/ui/package.json — current
{
  "exports": {
    ".": "./src/index.ts",
    "./components/*": "./src/components/*.tsx"
    // Missing: "./tokens" entry
  }
}
```

The `@repo/ui/tokens` import worked before because you did not
have an `exports` field. Without `exports`, Node resolves any
subpath. Once you add `exports`, ONLY the declared paths are
resolvable. Everything else 404s.

## Fix

Add the tokens export:
```json
{
  "exports": {
    ".": "./src/index.ts",
    "./components/*": "./src/components/*.tsx",
    "./tokens": "./src/tokens.ts"
  }
}
```

Or, if you have many subpath exports, use a wildcard:
```json
{
  "exports": {
    ".": "./src/index.ts",
    "./*": "./src/*.ts"
  }
}
```
```

This is a classic "took me 2 hours to figure out, takes AI 30 seconds" debugging scenario. The AI recognizes the pattern instantly because it has seen thousands of package.json export issues.

---

## 13. AI-ASSISTED DOCUMENTATION

Documentation is the most unloved part of software engineering. AI makes it tolerable.

### 13.1 Generating JSDoc Comments

```
PROMPT:
"Add JSDoc comments to all exported functions and types in
packages/utils/src/currency.ts

Include:
- Description of what the function does
- @param descriptions with types
- @returns description
- @example with realistic usage
- @throws for any error cases"
```

Before:

```typescript
export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(amount / 100);
}
```

After:

```typescript
/**
 * Formats a monetary amount from cents to a localized currency string.
 *
 * @param amount - The amount in cents (e.g., 1099 for $10.99).
 *   Must be an integer. Negative values are formatted with a minus sign.
 * @param currency - ISO 4217 currency code (e.g., 'USD', 'EUR', 'GBP').
 * @returns Formatted currency string (e.g., "$10.99", "-$5.00", "EUR 15.50").
 *
 * @example
 * ```typescript
 * formatCurrency(1099, 'USD');  // "$10.99"
 * formatCurrency(-500, 'USD');  // "-$5.00"
 * formatCurrency(0, 'EUR');     // "EUR 0.00"
 * ```
 *
 * @throws {RangeError} If the currency code is not a valid ISO 4217 code.
 */
export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency,
  }).format(amount / 100);
}
```

AI-generated JSDoc is consistently good because it can infer behavior from the implementation. This is one area where AI output rarely needs significant editing.

### 13.2 Generating Architecture Decision Records (ADRs)

```
PROMPT:
"Generate an ADR for our decision to migrate from Redux to Zustand.
Read the old Redux store in apps/mobile/src/store/ and the new
Zustand stores in apps/mobile/src/stores/ to understand the scope
of the migration.

Follow the ADR format:
- Title
- Status (accepted)
- Context (why we considered this)
- Decision (what we decided)
- Consequences (positive, negative, risks)"
```

Claude Code generates a well-structured ADR based on the actual code differences it reads. It can quantify things like "reduced store boilerplate by X lines" and "eliminated Y Redux middleware dependencies" because it reads both codebases.

### 13.3 When AI Documentation Needs Heavy Editing

AI documentation is weakest when:
- **Explaining "why" behind architectural decisions** -- AI can see what the code does, but not why you chose that approach over alternatives
- **Documenting known workarounds** -- "We use this pattern because of a bug in React Navigation 7.2" is institutional knowledge that is not in the code
- **Writing onboarding guides** -- What a new engineer needs to know is not just what the code does, but what order to learn things and what common mistakes to avoid
- **Describing trade-offs** -- AI can document the current approach but struggles to articulate the alternatives you considered and rejected

For these cases, use AI to generate the skeleton and fill in the judgment calls yourself.

---

## 14. THE AI-AUGMENTED ARCHITECT

The frontend architect role is changing. Here is how.

### 14.1 What You Spend Less Time On

```
BEFORE AI (typical architect week):
====================================
Architecture decisions + system design     10%
Implementation (hands-on coding)           35%
Code review                                20%
Debugging and firefighting                 15%
Documentation and communication            10%
Mentoring and team support                 10%

AFTER AI (typical architect week):
====================================
Architecture decisions + system design     25%  ↑↑
Implementation (hands-on coding)           10%  ↓↓ (AI handles boilerplate)
Code review (AI-assisted)                  15%  ↓  (AI catches mechanical issues)
Debugging and firefighting                  5%  ↓↓ (AI-assisted root cause)
Documentation and communication             5%  ↓  (AI generates drafts)
AI workflow design + prompt engineering    10%  NEW
Mentoring and team support                 15%  ↑  (more time available)
Strategic planning + cross-team alignment  15%  ↑  (more time available)
```

The shift is dramatic: you move from 35% implementation to 10%, and that freed time goes into the highest-leverage activities -- architecture decisions, mentoring, and strategic planning.

### 14.2 New Skills for the AI-Augmented Architect

**Prompt Engineering for Teams.** You are not just writing prompts for yourself. You are designing the prompts, skills, and workflows that your entire team uses. A well-crafted CLAUDE.md and skill set makes every engineer on your team faster.

**AI Output Quality Judgment.** You need to quickly evaluate whether AI-generated code is correct, performant, secure, and consistent with your architecture. This is a different skill from writing code -- it is closer to code review than coding. The best architects develop a rapid scan pattern: check the types, check the error handling, check the imports, check the dependencies, spot-check the logic.

**Workflow Design.** You design the workflows that combine AI tools into effective pipelines. Which tasks should use Claude Code? Which should use Cursor? What should the CI AI review check for? How should skills be structured? This is systems design applied to your development process.

**Guardrail Design.** You decide what AI should NOT do. What files should be excluded from AI edits? What changes require human review? What hooks should run to catch mistakes? The architect who does not set boundaries ends up with AI-generated code that works in demo but fails in production.

### 14.3 The Judgment Paradox

Here is the paradox of AI-augmented development: **the less code you write by hand, the more your code judgment matters.**

When you wrote every line yourself, bugs were caught by the act of writing. You noticed the edge case while typing the condition. You remembered the platform quirk while implementing the handler. You thought of the race condition while coding the async flow.

When AI writes the code, those bugs still exist. But you did not have the "writing moment" to catch them. Your only line of defense is review -- and review requires sharper judgment than writing, because you are scanning for problems in someone else's logic rather than building the logic yourself.

The best AI-augmented architects are not the ones who prompt the best. They are the ones who **review** the best. They spot the null pointer exception in a 200-line generated component in 30 seconds. They notice the missing error boundary that the AI "forgot." They catch the subtle business logic error that is technically correct but semantically wrong.

This judgment does not come from AI training. It comes from years of writing code, shipping bugs, debugging production issues, and building the intuition for what can go wrong. AI accelerates your output. It does not accelerate your judgment. That still takes time, experience, and deliberate practice.

---

## 15. RISKS AND BOUNDARIES

This is the most important section of this chapter. Everything above teaches you to go fast. This section teaches you when to slow down.

### 15.1 When NOT to Trust AI

```
HIGH RISK — Always human-review, never AI-only:
================================================
- Authentication and authorization logic
- Payment processing and financial calculations
- Encryption, hashing, and security token handling
- User data privacy and PII handling
- Rate limiting and abuse prevention
- Database migrations on production data
- Cross-origin and CORS configuration
- Session management and token refresh flows
- Permission checks and access control
- Terms of service and legal compliance logic

MODERATE RISK — AI draft + careful human review:
================================================
- API input validation (AI is good here but verify edge cases)
- Error boundaries and crash handling
- Offline data synchronization
- Cache invalidation logic
- WebSocket connection management
- Push notification handling
- Deep linking and URL routing
- Analytics event tracking

LOWER RISK — AI is generally trustworthy:
================================================
- UI component layout and styling
- Form rendering (not validation logic)
- Storybook stories
- Unit tests for pure functions
- Documentation and JSDoc
- Boilerplate scaffolding
- CSS/Tailwind classes
- Import organization
```

### 15.2 AI Hallucinations in Code

AI hallucinations in code are particularly dangerous because they look plausible. A hallucinated fact in text is often obviously wrong. A hallucinated pattern in code is often syntactically valid, type-safe, and logically coherent -- but semantically wrong.

Common hallucination patterns in frontend code:

**Non-existent APIs:**
```typescript
// AI generates this — the API does not exist
import { useStableCallback } from 'react';

// What it should be:
import { useCallback } from 'react';
```

**Deprecated patterns:**
```typescript
// AI generates getServerSideProps — deprecated in App Router
export async function getServerSideProps() { ... }

// What it should be:
// Server Component that fetches data directly
async function Page() {
  const data = await fetchData();
  return <Component data={data} />;
}
```

**Plausible but wrong logic:**
```typescript
// AI generates a currency conversion that looks correct
function convertCurrency(amount: number, rate: number): number {
  return Math.round(amount * rate * 100) / 100;
}

// The bug: rounding should happen AFTER conversion, but the order of
// operations matters for financial calculations. This can produce
// off-by-one-cent errors that accumulate over thousands of transactions.
// The correct approach depends on your business rules for rounding.
```

**Mixing up similar libraries:**
```typescript
// AI confuses React Navigation and Expo Router
import { useNavigation } from '@react-navigation/native';
navigation.navigate('Profile', { userId });

// In an Expo Router project, it should be:
import { useRouter } from 'expo-router';
router.push(`/profile/${userId}`);
```

### 15.3 The "Works But Is Wrong" Problem

The most dangerous AI output is code that passes all tests, compiles without errors, and appears to work correctly in development -- but is wrong in a way that only manifests in production under specific conditions.

Examples:

**Race condition that only appears under load:**
```typescript
// AI generates a "correct" optimistic update
const handleLike = async () => {
  setLiked(true);                   // Optimistic update
  setLikeCount(prev => prev + 1);   // Increment count
  await api.like(postId);           // API call
};

// The bug: if the user rapidly taps like/unlike, the count
// drifts because each tap increments independently without
// tracking whether the API call succeeded or the previous
// tap's API call completed. This needs a queue or mutex.
```

**Memory leak that only appears in long sessions:**
```typescript
// AI generates a WebSocket connection
useEffect(() => {
  const ws = new WebSocket(url);
  ws.onmessage = (event) => {
    setMessages(prev => [...prev, JSON.parse(event.data)]);
  };
  return () => ws.close();
}, [url]);

// The bug: the `setMessages` closure captures `prev` from the
// render cycle, but if messages arrive faster than React batches
// renders, messages can be lost. Also, the `messages` array grows
// unboundedly. After hours of use, memory consumption spirals.
```

**Platform-specific behavior AI does not know about:**
```typescript
// AI generates a scroll handler
const handleScroll = (event) => {
  const scrollY = event.nativeEvent.contentOffset.y;
  if (scrollY < -50) {
    triggerRefresh();
  }
};

// The bug: on Android, contentOffset.y can be negative during
// overscroll. On iOS, it is negative during bounce. The behavior
// is different, and the threshold needs platform-specific tuning.
// AI does not know this because it is not documented in most
// tutorials — it is tribal knowledge from shipping mobile apps.
```

### 15.4 The Junior Engineer PR Analogy

The best mental model for AI-generated code: **review it like a junior engineer's PR.**

A good junior engineer:
- Gets the structure right most of the time
- Follows patterns they have seen
- Misses edge cases
- Does not know the historical context for weird decisions
- Produces code that works in the happy path
- Sometimes introduces subtle bugs that only experienced eyes catch
- Writes tests that test the obvious cases but miss the tricky ones
- Needs guidance on security, performance, and architecture

This is exactly the profile of AI-generated code. Your review approach should be the same:

1. **Check the structure.** Does it follow your patterns? Are the files in the right place? Are the imports correct?
2. **Check the types.** Are all types explicit? Are there any `any` escapes? Do the types match the actual data shapes?
3. **Check error handling.** What happens when things fail? Are all error cases covered? Are error messages user-friendly?
4. **Check edge cases.** What about empty lists? Null values? Very long strings? Network failures? Concurrent operations?
5. **Check security.** Is user input sanitized? Are auth checks in place? Is sensitive data exposed?
6. **Check performance.** Are there unnecessary re-renders? Is memoization used correctly? Are there memory leak risks?
7. **Run the code.** Does it actually work? Not just compile -- does it behave correctly with real data?

### 15.5 The Trust Calibration

Over time, you develop a calibrated sense of when to trust AI output:

```
TRUST CALIBRATION MATRIX
=========================

                    SIMPLE TASK         COMPLEX TASK
                    (boilerplate,       (architecture,
                    known patterns)     novel logic)
                ┌───────────────────┬────────────────────┐
WELL-CONFIGURED │                   │                    │
CLAUDE.md +     │  HIGH TRUST       │  MODERATE TRUST    │
skills + hooks  │  Light review     │  Careful review    │
                │  ~90% accuracy    │  ~70% accuracy     │
                ├───────────────────┼────────────────────┤
NO CONFIG       │                   │                    │
(bare AI tool)  │  MODERATE TRUST   │  LOW TRUST         │
                │  Standard review  │  Heavy review or   │
                │  ~75% accuracy    │  rewrite  ~50%     │
                └───────────────────┴────────────────────┘
```

The configuration investment (CLAUDE.md, skills, hooks) directly increases the trust level. A well-configured AI tool produces consistently better output because it knows your patterns, constraints, and conventions.

### 15.6 Establishing Team Boundaries

Every team using AI tools needs explicit rules. Here is a starting template:

```markdown
# AI Usage Policy for Frontend Team

## Required
- All AI-generated code must pass CI (lint, type check, tests)
- All AI-generated code must be reviewed by a human before merge
- CLAUDE.md must be kept up-to-date with current conventions
- Security-sensitive files must have human-only review flag

## Prohibited
- AI-only changes to payment, auth, or security code
- Merging AI-generated code without human review
- Using AI for code that handles PII without privacy review
- Accepting AI suggestions for cryptographic operations

## Guidelines
- Use AI for first drafts, not final drafts
- When AI generates tests, verify the assertions are meaningful
- When AI suggests a pattern you have not seen before, research it
- When AI and your instinct disagree, trust your instinct and investigate
- Document when you override AI suggestions (helps calibrate future trust)

## Escalation
- If AI generates code you are unsure about, flag for senior review
- If AI suggests an architectural change, discuss with the team first
- If AI-generated code passes tests but feels wrong, do not ship it
```

---

## 16. PUTTING IT ALL TOGETHER

Here is the complete toolkit for the AI-augmented frontend architect in 2026:

```
THE AI-AUGMENTED FRONTEND TOOLKIT
===================================

TOOL                USE FOR                         FREQUENCY
────────────────────────────────────────────────────────────────
Claude Code         Multi-step tasks, refactoring,  Multiple times/day
                    debugging, code generation,
                    PR review, test generation

Cursor/Windsurf     In-flow editing, tab complete,  Constantly (IDE)
                    inline changes, quick questions

GitHub Copilot      Boilerplate completion,          Constantly (background)
                    import suggestions, repetitive
                    patterns

CodeRabbit/         Automated PR review, CI          Every PR (automated)
Claude GH Action    integration

Vercel v0           UI component prototyping,        Weekly (prototyping)
                    shadcn/ui generation

Figma MCP           Design-to-code translation       Multiple times/week

Sentry MCP          Production crash debugging        As needed

GitHub MCP          Issue-to-implementation,          Multiple times/day
                    PR context
```

### The Daily Workflow

```
MORNING:
1. Open Cursor (IDE with AI tab completion active)
2. Start Claude Code in terminal beside IDE
3. Check Sentry for new crashes → "Claude, investigate MOBILE-1234"
4. Review AI-generated PR comments on last night's PRs
5. Triage today's tasks

BUILDING:
6. For new screens: prompt Claude Code with Figma link
7. For refactoring: prompt Claude Code with the scope
8. For quick edits: use Cursor's inline editing
9. For boilerplate: let Copilot tab-complete
10. Always review AI output before committing

REVIEWING:
11. AI review catches mechanical issues automatically
12. You review for architecture, business logic, UX quality
13. Provide feedback to junior engineers on AI usage

ENDING:
14. Claude Code generates docs for the day's changes
15. Update CLAUDE.md if conventions changed
16. Update skills if you found a repeatable workflow
```

### The Meta-Skill

The most important skill is not using any single AI tool well. It is knowing which tool to reach for in each moment. That instinct develops with practice, and it is what separates the architect who uses AI to ship 5x faster from the engineer who uses AI to ship 5x more bugs.

The AI is a power tool. Power tools in the hands of a skilled craftsperson produce extraordinary results. Power tools in the hands of someone who does not understand the material produce dangerous results. Know your material -- your codebase, your architecture, your users, your constraints -- and the AI becomes the most powerful tool in your workshop.

---

## CHAPTER SUMMARY

```
KEY TAKEAWAYS
=============

1. AI replaces the tedious 80%, not the creative 20%. Your judgment,
   architecture skills, and UX instinct are more valuable than ever.

2. Claude Code with CLAUDE.md + skills + hooks is the highest-leverage
   configuration. Invest one hour in setup, save hours every day.

3. Use the right tool: Claude Code for autonomous tasks, Cursor for
   in-flow editing, Copilot for boilerplate completion.

4. AI code review augments human review — it catches bugs, performance
   issues, and accessibility violations. Humans catch business logic,
   architecture, and UX issues.

5. AI-generated tests need human review for meaningful assertions.
   The structure is usually right; the assertions need verification.

6. The design-to-code pipeline (Figma MCP → Claude Code → Storybook →
   tests) reduces implementation time from days to hours.

7. Multi-agent development works for independent, parallel tasks.
   Sequential, interconnected changes still need single-threaded work.

8. Prompt specificity directly correlates with output quality. Reference
   existing files, specify constraints, describe behavior not features.

9. Never trust AI with security-critical code, payment logic, or
   authentication flows without careful human review.

10. Review AI output like a junior engineer's PR: trust the structure,
    verify the logic, always check edge cases and error handling.
```

```
QUICK REFERENCE: AI TOOL SELECTION
===================================

"I need to build a new screen from a design"
  → Claude Code + Figma MCP

"I need to fix this crash"
  → Claude Code + Sentry MCP

"I need to refactor this module"
  → Claude Code (autonomous, multi-file)

"I need to finish typing this function"
  → Cursor tab completion

"I need to edit this block of code"
  → Cursor inline edit (Cmd+K)

"I need to review this PR"
  → CodeRabbit (automated) + your human review

"I need tests for this component"
  → Claude Code with write-tests skill

"I need a shadcn component quickly"
  → Vercel v0

"I need to understand this code"
  → Cursor chat or Claude Code

"I need to make the same change in 10 files"
  → Claude Code (or Cursor Composer)
```

> **Next Chapter:** [Ch 50: The Future of Frontend Architecture] — where the platform, the tools, and the role are headed
