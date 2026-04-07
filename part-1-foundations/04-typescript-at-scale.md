<!--
  CHAPTER: 4
  TITLE: TypeScript at Scale
  PART: I — Foundations
  PREREQS: None
  KEY_TOPICS: project references, barrel files, diagnostics, monorepo tsconfig, discriminated unions, branded types, Zod, ESLint boundaries, dependency-cruiser
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 4: TypeScript at Scale

> **Part I — Foundations** | Prerequisites: None | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- TypeScript type checking consumes 80%+ of compile time; barrel files, deep generics, and distributive conditional types are the usual culprits for slow builds
- Use project references and composite builds to split type checking into independent, cacheable units across a monorepo
- Kill barrel files (`index.ts` re-exports) -- they force the compiler to load entire dependency trees; use direct imports instead
- Run `tsc --diagnostics` and `tsc --generateTrace` to find exactly what is slow; fix the bottleneck, not a guess
- ESLint boundaries and dependency-cruiser enforce module boundaries at lint time, preventing the dependency tangles that make TypeScript slow in the first place

</details>

Here's a truth that nobody tells junior engineers: **TypeScript at 500 lines and TypeScript at 500,000 lines are completely different experiences.** At 500 lines, TypeScript is a helpful assistant that catches typos. At 500,000 lines, TypeScript is a build system, an architecture enforcement tool, and — if you're not careful — a 45-second tax on every keystroke in your editor.

I've seen TypeScript codebases where the type checker takes 90 seconds to run. Where `tsc --watch` consumes 4GB of RAM. Where a single barrel file import pulls in 200 modules. Where the IDE autocomplete freezes for 10 seconds because a Zod schema creates a recursive type that blows up the type inference engine.

I've also seen TypeScript codebases at the same scale where the type checker runs in under 5 seconds, the IDE is snappy, the types catch real bugs, and the dependency graph is clean enough that you can reason about any module in isolation.

The difference is not talent. It's knowledge — knowing how the TypeScript compiler actually works, what makes it slow, and how to structure types for large codebases. This chapter gives you that knowledge.

### In This Chapter
- Project References and Composite Builds
- The Barrel File Anti-Pattern (and What to Do Instead)
- TypeScript Diagnostics: Finding What's Slow
- Monorepo tsconfig Architecture
- Advanced Type Patterns: Discriminated Unions, Branded Types
- Zod Schema Inference and Runtime Validation
- ESLint Architectural Linting and Boundary Enforcement
- dependency-cruiser: Enforcing Module Boundaries

### Related Chapters
- [Ch 5: Expo Platform] — TypeScript configuration for Expo projects
- [Ch 6: App Architecture] — applying these patterns to real application structure
- [Ch 11: Testing Strategy] — typing test utilities and mocks
- [Ch 13: Performance Optimization] — TypeScript's impact on build performance

---

## 1. HOW THE TYPESCRIPT COMPILER ACTUALLY WORKS

Before we can optimize TypeScript, we need to understand what it does:

### 1.1 The Compilation Pipeline

```
Source Files (.ts, .tsx)
        ↓
    Scanner (lexing)
        ↓
    Parser (AST construction)
        ↓
    Binder (symbol table construction)
        ↓
    Type Checker (type inference, type checking)
        ↓
    Emitter (generate .js, .d.ts, .js.map)
```

**The Type Checker is where 80%+ of the time goes.** Scanning, parsing, and emitting are fast. Type checking is where TypeScript resolves every type reference, evaluates every generic instantiation, checks every assignment, and infers every omitted type annotation.

### 1.2 What Makes Type Checking Slow

The TypeScript type checker is essentially an interpreter for a Turing-complete type language. Certain patterns cause exponential or combinatorial type evaluation:

**1. Deep generic instantiation:**
```typescript
// Each level doubles the type computation
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Apply to a 5-level-deep object with 50 properties each
type Config = DeepPartial<DeepNestedConfig>; // Expensive
```

**2. Distributive conditional types over unions:**
```typescript
// For a union of 100 members, this evaluates the conditional 100 times
type ExtractStrings<T> = T extends string ? T : never;
type Result = ExtractStrings<LargeUnion>; // 100 evaluations
```

**3. Complex mapped types:**
```typescript
// Mapped type over a type with many keys
type ReadonlyDeep<T> = {
  readonly [P in keyof T]: T[P] extends object ? ReadonlyDeep<T[P]> : T[P];
};
```

**4. Recursive type aliases:**
```typescript
// Each recursion level multiplies the work
type JSON = string | number | boolean | null | JSON[] | { [key: string]: JSON };
```

**5. Excessive function overloads:** TypeScript tries each overload signature in order. With 20+ overloads (common in libraries like Express or jQuery), type resolution for each call site is expensive.

### 1.3 Module Resolution

When TypeScript encounters an `import` statement, it must find the corresponding module. This involves:
1. Checking the module map (already resolved modules)
2. Trying various file extensions (.ts, .tsx, .d.ts, .js)
3. Checking `paths` mappings in tsconfig
4. Checking `node_modules` (potentially traversing multiple directories)
5. Reading `package.json` `exports` maps

In a monorepo with deep `node_modules` nesting, module resolution alone can account for significant time. This is why `moduleResolution: "bundler"` (introduced in TypeScript 5.0) is important — it matches how modern bundlers resolve modules, avoiding unnecessary filesystem lookups.

---

## 2. PROJECT REFERENCES AND COMPOSITE BUILDS

Project references are TypeScript's answer to monorepo scale. They let you split a large project into smaller, independently type-checked pieces.

### 2.1 The Problem

Without project references, `tsc` type-checks your entire codebase as one unit. In a monorepo with 10 packages, every change triggers a full type check of all 10 packages — even if you only changed one file in one package.

```
monorepo/
  packages/
    ui/         (200 files)
    api-client/ (50 files)
    utils/      (100 files)
    app/        (500 files, imports ui, api-client, utils)

Without project references:
  Change a file in utils/ → tsc checks all 850 files
  Time: 30 seconds

With project references:
  Change a file in utils/ → tsc checks 100 files (utils)
  Then incrementally updates dependents from .d.ts
  Time: 3 seconds
```

### 2.2 Setting Up Project References

**Step 1: Make each package a composite project**

```jsonc
// packages/utils/tsconfig.json
{
  "compilerOptions": {
    "composite": true,         // Required for project references
    "declaration": true,       // Generate .d.ts files
    "declarationMap": true,    // Source maps for .d.ts (IDE navigation)
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "target": "es2022"
  },
  "include": ["src/**/*"]
}
```

**Step 2: Reference dependencies**

```jsonc
// packages/app/tsconfig.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../utils" },
    { "path": "../ui" },
    { "path": "../api-client" }
  ],
  "include": ["src/**/*"]
}
```

**Step 3: Root tsconfig for building everything**

```jsonc
// tsconfig.json (root)
{
  "files": [],
  "references": [
    { "path": "packages/utils" },
    { "path": "packages/ui" },
    { "path": "packages/api-client" },
    { "path": "packages/app" }
  ]
}
```

**Step 4: Build with `--build`**

```bash
# Build all projects in dependency order
tsc --build

# Build with incremental caching
tsc --build --incremental

# Force clean rebuild
tsc --build --clean

# Watch mode
tsc --build --watch
```

### 2.3 How It Works Internally

When you run `tsc --build`:
1. TypeScript reads the root tsconfig and discovers all referenced projects
2. It builds a dependency graph from the `references` field
3. It processes projects in topological order (dependencies first)
4. For each project, it checks if the `.d.ts` outputs are up to date (using `.tsbuildinfo`)
5. If up to date, it skips the project entirely
6. If not, it type-checks only that project and regenerates `.d.ts` files
7. Downstream projects see the new `.d.ts` files and are re-checked if needed

**The key insight:** With project references, TypeScript type-checks each package against the `.d.ts` files of its dependencies, not the source files. This means a change in `utils/src/helpers.ts` only requires re-type-checking `utils/`. If the change doesn't alter the `.d.ts` signature, downstream packages aren't re-checked at all.

### 2.4 Common Pitfalls

**Circular references:**
```jsonc
// packages/a/tsconfig.json
{ "references": [{ "path": "../b" }] }

// packages/b/tsconfig.json
{ "references": [{ "path": "../a" }] }

// ERROR: Project references may not form a circular graph
```

Solution: Extract shared types into a third package that both reference.

**Missing `composite` flag:**
```
error TS6306: Referenced project must have setting "composite": true
```

Every project that appears in a `references` array must have `composite: true`.

**IDE not recognizing project references:**
Some editors need to be pointed at the root tsconfig. In VS Code:

```jsonc
// .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

### 2.5 Project References with Turborepo/Nx

Modern monorepo tools (Turborepo, Nx) integrate with project references:

```jsonc
// turbo.json
{
  "tasks": {
    "typecheck": {
      "dependsOn": ["^typecheck"],  // Run deps first
      "inputs": ["src/**/*.ts", "src/**/*.tsx", "tsconfig.json"],
      "outputs": ["dist/**/*.d.ts", ".tsbuildinfo"]
    }
  }
}
```

```bash
# Turborepo runs typecheck in dependency order, caching results
turbo typecheck
```

This gives you the benefits of both project references (incremental TypeScript builds) and Turborepo (remote caching, parallel execution, dependency-aware task scheduling).

---

## 3. THE BARREL FILE ANTI-PATTERN

Barrel files are `index.ts` files that re-export everything from a directory:

```typescript
// src/components/index.ts (barrel file)
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
export { Table } from './Table';
export { Chart } from './Chart';
export { DatePicker } from './DatePicker';
// ... 50 more exports
```

The intent is convenience — `import { Button } from '@/components'` instead of `import { Button } from '@/components/Button'`.

### 3.1 Why Barrel Files Are Toxic at Scale

**Problem 1: Module graph explosion**

When you import from a barrel file, TypeScript (and your bundler) must process *every* module that the barrel re-exports, even if you only use one:

```typescript
// You want one component
import { Button } from '@/components';

// TypeScript processes ALL of these to resolve the barrel:
// Button.ts, Input.ts, Modal.ts, Table.ts, Chart.ts, DatePicker.ts, ...
// Plus all of THEIR imports: React, chart libraries, date libraries, ...
```

In a real codebase, a single barrel import can transitively pull in hundreds of modules. I've measured barrel files that caused TypeScript to load 300+ modules for a single import.

**Problem 2: Circular dependencies**

Barrel files create implicit connections between unrelated modules. `Button` and `Chart` have nothing to do with each other, but they're both in the same barrel. If `Chart` imports something from `utils`, and `utils` imports `Button` from `@/components`, you've got a circular dependency through the barrel.

**Problem 3: Tree-shaking failure**

Bundlers have gotten better at tree-shaking barrel files, but it's still imperfect. Side effects in any module in the barrel can prevent the bundler from eliminating unused modules:

```typescript
// DatePicker.ts
import 'dayjs/locale/en'; // Side effect import — bundler can't tree-shake this module

// You imported Button, but DatePicker's side effect gets included too
```

**Problem 4: IDE performance**

The TypeScript language server in your IDE evaluates barrel files for autocomplete. A barrel with 100 exports means the language server evaluates 100 modules when you type `import { } from '@/components'`. This is why autocomplete freezes in large codebases.

### 3.2 The Fix: Direct Imports

```typescript
// Instead of barrel:
import { Button } from '@/components';

// Use direct imports:
import { Button } from '@/components/Button';

// Or with package.json exports (for packages):
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "exports": {
    "./button": "./src/Button.tsx",
    "./input": "./src/Input.tsx",
    "./modal": "./src/Modal.tsx"
  }
}

// Usage:
import { Button } from '@myapp/ui/button';
```

### 3.3 Enforcing No Barrel Imports

```javascript
// ESLint rule to ban barrel imports
// .eslintrc.js
module.exports = {
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['@/components', '@/components/index'],
          message: 'Import directly from the component file: @/components/Button',
        },
        {
          group: ['@/utils', '@/utils/index'],
          message: 'Import directly from the util file: @/utils/formatDate',
        },
      ],
    }],
  },
};
```

### 3.4 When Barrel Files Are OK

Barrel files are acceptable when:
1. The package is small (under 10 exports)
2. Every consumer uses most of the exports
3. The package has no heavy dependencies
4. You're building a library and the barrel IS the public API

For example, a `@myapp/types` package that exports only TypeScript types (zero runtime cost) is fine as a barrel.

---

## 4. TYPESCRIPT DIAGNOSTICS: FINDING WHAT'S SLOW

TypeScript ships with built-in diagnostic tools that most developers never use.

### 4.1 `--extendedDiagnostics`

```bash
tsc --extendedDiagnostics
```

Output:
```
Files:                         1,247
Lines of Library:              38,241
Lines of Definitions:          92,847
Lines of TypeScript:           187,432
Lines of JavaScript:           0
Lines of JSON:                 0
Lines of Other:                0
Identifiers:                   435,891
Symbols:                       312,456
Types:                         198,234
Instantiations:                1,247,891    ← Watch this number
Memory used:                   892,134K     ← And this one
Assignability cache size:      234,567
Identity cache size:           12,345
Subtype cache size:            45,678
Strict subtype cache size:     23,456
I/O Read time:                 0.42s
Parse time:                    1.23s
ResolveModule time:            0.89s
ResolveTypeReference time:     0.12s
Program time:                  2.66s
Bind time:                     0.78s
Check time:                    18.34s       ← The bottleneck
Emit time:                     0.45s
Total time:                    22.23s
```

**What to look for:**
- **Instantiations:** If this number is over 1M, you likely have expensive generic types. Over 5M is a red flag.
- **Check time:** Should be the largest chunk. If it's disproportionately large compared to file count, you have expensive types.
- **Memory used:** Over 2GB suggests the type checker is doing too much work.
- **ResolveModule time:** High values indicate deep `node_modules` or misconfigured `paths`.

### 4.2 `--generateTrace`

```bash
tsc --generateTrace ./trace-output
```

This generates Chrome DevTools trace files that you can analyze in `chrome://tracing`:

1. Run the command above
2. Open `chrome://tracing` in Chrome
3. Load the `trace.json` file from the output directory
4. Look for the largest blocks in the flame chart

The trace shows exactly which types take the longest to check. You'll often find one or two types that dominate the entire check time.

### 4.3 `@typescript/analyze-trace`

```bash
npx @typescript/analyze-trace ./trace-output
```

This tool analyzes the trace and gives you a human-readable report:

```
Hot spots:
  1. Check type: DeepPartial<AppConfig>    2,345ms  (12.8%)
  2. Check type: RouterParams<Routes>      1,892ms  (10.3%)
  3. Resolve module: @/components/index    1,234ms  (6.7%)
  4. Check type: ZodInfer<UserSchema>        987ms  (5.4%)
```

Now you know exactly where to focus your optimization efforts.

### 4.4 Practical Optimization Examples

**Before: Expensive recursive type**
```typescript
// 2,345ms to check
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object 
    ? T[P] extends Array<infer U> 
      ? Array<DeepPartial<U>>
      : DeepPartial<T[P]>
    : T[P];
};

type Config = DeepPartial<FullAppConfig>; // FullAppConfig has 200 properties, 5 levels deep
```

**After: Limit recursion depth**
```typescript
type DeepPartial<T, Depth extends number[] = []> = 
  Depth['length'] extends 3 
    ? Partial<T>  // Stop recursing after 3 levels
    : {
        [P in keyof T]?: T[P] extends object 
          ? DeepPartial<T[P], [...Depth, 0]>
          : T[P];
      };
```

**Before: Over-inferred Zod schema**
```typescript
// 987ms to infer
const UserSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  preferences: z.object({
    theme: z.enum(['light', 'dark']),
    notifications: z.object({
      email: z.boolean(),
      push: z.boolean(),
      sms: z.boolean(),
      // ... 20 more fields
    }),
    // ... 10 more nested objects
  }),
});

type User = z.infer<typeof UserSchema>;
```

**After: Explicit type, Zod for validation only**
```typescript
// Define the type explicitly
interface User {
  name: string;
  email: string;
  preferences: UserPreferences;
}

interface UserPreferences {
  theme: 'light' | 'dark';
  notifications: NotificationSettings;
}

// Use Zod for runtime validation, not type inference
const UserSchema: z.ZodType<User> = z.object({
  name: z.string(),
  email: z.string().email(),
  preferences: z.object({
    theme: z.enum(['light', 'dark']),
    notifications: NotificationSettingsSchema,
  }),
});
```

This gives you the same runtime validation but the type checker doesn't need to infer the type from the schema — you've provided it explicitly.

---

## 5. MONOREPO TSCONFIG ARCHITECTURE

A well-structured tsconfig hierarchy is the foundation of TypeScript performance in a monorepo.

### 5.1 The Base Config Pattern

```jsonc
// tsconfig.base.json (root — shared settings)
{
  "compilerOptions": {
    // Strictness
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true,
    
    // Module system
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    
    // Interop
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    
    // Output
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "skipLibCheck": true,
    
    // Target
    "target": "es2022",
    "lib": ["es2023"]
  }
}
```

### 5.2 Package-Level Configs

```jsonc
// packages/ui/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "jsx": "react-jsx",
    "lib": ["es2023", "dom", "dom.iterable"]
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts", "**/*.test.tsx", "**/*.spec.ts"]
}
```

```jsonc
// packages/api-client/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src"
    // No jsx, no dom — this is a pure Node/universal package
  },
  "references": [
    { "path": "../utils" }
  ],
  "include": ["src/**/*"]
}
```

```jsonc
// apps/mobile/tsconfig.json (Expo app)
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "references": [
    { "path": "../../packages/ui" },
    { "path": "../../packages/api-client" },
    { "path": "../../packages/utils" }
  ],
  "include": ["src/**/*", "app/**/*", "expo-env.d.ts"]
}
```

### 5.3 Key Compiler Options Explained

**`skipLibCheck: true`** — Skips type checking of `.d.ts` files (including those in `node_modules`). This is almost always what you want. The alternative is spending 10+ seconds checking types you don't control and can't fix.

**`isolatedModules: true`** — Ensures each file can be transpiled independently (no `const enum`, no namespace merging across files). Required for esbuild, SWC, Babel — basically every modern transpiler.

**`verbatimModuleSyntax: true`** — Enforces that `import type` is used for type-only imports. This helps bundlers tree-shake and prevents accidental runtime imports of type-only modules.

```typescript
// With verbatimModuleSyntax: true
import type { User } from './types';     // OK: type-only import
import { formatUser } from './utils';    // OK: value import
import { User } from './types';          // ERROR: must use 'import type'
```

**`noUncheckedIndexedAccess: true`** — Makes index access return `T | undefined` instead of `T`:

```typescript
const arr = [1, 2, 3];
const val = arr[5];
// Without noUncheckedIndexedAccess: val is number
// With noUncheckedIndexedAccess: val is number | undefined ← correct!

const obj: Record<string, string> = { a: 'hello' };
const val2 = obj['missing'];
// Without: val2 is string
// With: val2 is string | undefined ← correct!
```

**`moduleResolution: "bundler"`** — Uses module resolution rules that match modern bundlers (Vite, esbuild, webpack, Metro). Supports `package.json` `exports`, doesn't require file extensions in imports.

### 5.4 Path Aliases

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@components/*": ["./src/components/*"],
      "@hooks/*": ["./src/hooks/*"],
      "@utils/*": ["./src/utils/*"]
    }
  }
}
```

**Important:** `paths` only affects TypeScript's type resolution. Your bundler needs matching configuration:

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

```javascript
// For Expo, use babel-plugin-module-resolver or expo's built-in alias support
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    plugins: [
      ['module-resolver', {
        root: ['./src'],
        alias: {
          '@': './src',
        },
      }],
    ],
  };
};
```

---

## 6. DISCRIMINATED UNIONS: THE MOST POWERFUL TYPE PATTERN

Discriminated unions (also called tagged unions) are the single most useful type pattern for application development. They model states that are mutually exclusive — and they make invalid states unrepresentable.

### 6.1 The Pattern

```typescript
// A discriminated union uses a common "tag" field to distinguish variants
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

// TypeScript narrows the type based on the tag
function renderState(state: AsyncState<User>) {
  switch (state.status) {
    case 'idle':
      return <EmptyState />;
    case 'loading':
      return <Spinner />;
    case 'success':
      return <UserCard user={state.data} />;  // TypeScript knows data exists
    case 'error':
      return <ErrorBanner error={state.error} />; // TypeScript knows error exists
  }
}
```

### 6.2 Real-World Examples

**API Response:**
```typescript
type ApiResponse<T> =
  | { ok: true; data: T; status: number }
  | { ok: false; error: string; status: number; retryable: boolean };

async function fetchUser(id: string): Promise<ApiResponse<User>> {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) {
      return { 
        ok: false, 
        error: `HTTP ${res.status}`, 
        status: res.status,
        retryable: res.status >= 500,
      };
    }
    return { ok: true, data: await res.json(), status: res.status };
  } catch (e) {
    return { ok: false, error: String(e), status: 0, retryable: true };
  }
}

// Usage — TypeScript ensures you handle both cases
const result = await fetchUser('123');
if (result.ok) {
  console.log(result.data.name); // TypeScript knows data exists
} else {
  if (result.retryable) {       // TypeScript knows retryable exists
    retry();
  }
}
```

**Navigation Events:**
```typescript
type NavigationEvent =
  | { type: 'push'; screen: string; params?: Record<string, unknown> }
  | { type: 'pop'; count: number }
  | { type: 'replace'; screen: string; params?: Record<string, unknown> }
  | { type: 'reset'; routes: Array<{ screen: string; params?: Record<string, unknown> }> };

function handleNavigation(event: NavigationEvent) {
  switch (event.type) {
    case 'push':
      navigator.push(event.screen, event.params);
      break;
    case 'pop':
      navigator.pop(event.count);
      break;
    case 'replace':
      navigator.replace(event.screen, event.params);
      break;
    case 'reset':
      navigator.reset(event.routes);
      break;
  }
}
```

**Form Field Validation:**
```typescript
type ValidationResult =
  | { valid: true }
  | { valid: false; errors: string[] };

function validateEmail(email: string): ValidationResult {
  if (!email.includes('@')) {
    return { valid: false, errors: ['Invalid email format'] };
  }
  return { valid: true };
}
```

### 6.3 Exhaustive Checking

The most powerful feature of discriminated unions is exhaustive checking — TypeScript tells you if you forgot a case:

```typescript
// Helper: This function should never be called if all cases are handled
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}

type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'rectangle'; width: number; height: number }
  | { kind: 'triangle'; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2;
    case 'rectangle':
      return shape.width * shape.height;
    case 'triangle':
      return (shape.base * shape.height) / 2;
    default:
      return assertNever(shape); // If a new Shape variant is added, this errors
  }
}

// If someone adds { kind: 'pentagon'; ... } to the Shape union,
// assertNever(shape) will show a type error because shape is not 'never' —
// it's the unhandled 'pentagon' variant. 
```

This is a compile-time safety net that catches missing cases when your union types evolve.

---

## 7. BRANDED TYPES: MAKING INVALID STATES UNREPRESENTABLE

Branded types add a phantom type tag to primitive types, making them incompatible with each other even though the runtime values are identical.

### 7.1 The Problem

```typescript
function transferMoney(fromAccountId: string, toAccountId: string, amount: number) {
  // How easy is it to mix up fromAccountId and toAccountId?
  // TypeScript won't catch: transferMoney(toId, fromId, amount)
}

function processOrder(userId: string, orderId: string, productId: string) {
  // Three string arguments — any permutation compiles
}
```

### 7.2 The Solution: Branded Types

```typescript
// Brand definition
type Brand<T, B extends string> = T & { readonly __brand: B };

// Branded string types
type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;
type ProductId = Brand<string, 'ProductId'>;
type AccountId = Brand<string, 'AccountId'>;

// Constructor functions (runtime: just returns the value; compile time: adds the brand)
function UserId(id: string): UserId { return id as UserId; }
function OrderId(id: string): OrderId { return id as OrderId; }
function ProductId(id: string): ProductId { return id as ProductId; }
function AccountId(id: string): AccountId { return id as AccountId; }

// Now this is type-safe
function processOrder(userId: UserId, orderId: OrderId, productId: ProductId) {
  // ...
}

const uid = UserId('user-123');
const oid = OrderId('order-456');
const pid = ProductId('prod-789');

processOrder(uid, oid, pid);  // OK
processOrder(oid, uid, pid);  // ERROR: OrderId is not assignable to UserId
processOrder('user-123', oid, pid); // ERROR: string is not assignable to UserId
```

### 7.3 Branded Types for Validation

```typescript
type Email = Brand<string, 'Email'>;
type NonEmptyString = Brand<string, 'NonEmptyString'>;
type PositiveNumber = Brand<number, 'PositiveNumber'>;
type Url = Brand<string, 'Url'>;

// Validation constructors
function Email(input: string): Email {
  if (!input.includes('@') || !input.includes('.')) {
    throw new Error(`Invalid email: ${input}`);
  }
  return input as Email;
}

function PositiveNumber(input: number): PositiveNumber {
  if (input <= 0) {
    throw new Error(`Expected positive number, got: ${input}`);
  }
  return input as PositiveNumber;
}

// Functions that accept branded types are guaranteed valid input
function sendEmail(to: Email, subject: NonEmptyString, body: string) {
  // `to` is guaranteed to be a valid email
  // `subject` is guaranteed to be non-empty
  // No runtime validation needed here — it happened at the boundary
}
```

### 7.4 Combining Branded Types with Zod

```typescript
import { z } from 'zod';

// Define schemas that produce branded types
const EmailSchema = z.string().email().transform((val): Email => val as Email);
const UserIdSchema = z.string().uuid().transform((val): UserId => val as UserId);
const PositiveNumberSchema = z.number().positive().transform((val): PositiveNumber => val as PositiveNumber);

const CreateUserSchema = z.object({
  email: EmailSchema,
  name: z.string().min(1),
  age: PositiveNumberSchema,
});

type CreateUserInput = z.output<typeof CreateUserSchema>;
// { email: Email; name: string; age: PositiveNumber }

// At the API boundary
function handleCreateUser(rawInput: unknown) {
  const input = CreateUserSchema.parse(rawInput);
  // input.email is Email (branded)
  // input.age is PositiveNumber (branded)
  createUser(input); // Fully type-safe with branded types
}
```

---

## 8. ZOD: RUNTIME VALIDATION AND TYPE INFERENCE

Zod is the standard for runtime type validation in TypeScript. It bridges the gap between compile-time types and runtime data.

### 8.1 Why You Need Runtime Validation

TypeScript types are erased at runtime. When you receive data from an API, a form, local storage, or a URL parameter, the TypeScript compiler has no idea what the actual data looks like. It trusts whatever type you assert:

```typescript
// This compiles but will crash at runtime if the API returns unexpected data
const user = await fetch('/api/user').then(r => r.json()) as User;
console.log(user.name.toUpperCase()); // Runtime error if name is undefined
```

Zod validates at runtime and infers TypeScript types from the schema:

```typescript
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'viewer']),
  createdAt: z.string().datetime(),
});

type User = z.infer<typeof UserSchema>;
// { id: string; name: string; email: string; role: 'admin' | 'user' | 'viewer'; createdAt: string }

// Safe parsing — returns success/error, never throws
const result = UserSchema.safeParse(apiResponse);
if (result.success) {
  const user = result.data; // Correctly typed as User
} else {
  console.error('Validation failed:', result.error.issues);
}
```

### 8.2 Schema Composition

```typescript
// Base schemas
const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  state: z.string().length(2),
  zip: z.string().regex(/^\d{5}(-\d{4})?$/),
});

const ContactSchema = z.object({
  phone: z.string().optional(),
  email: z.string().email(),
});

// Composition
const CustomerSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
}).merge(ContactSchema).extend({
  addresses: z.array(AddressSchema).min(1),
  preferredAddressIndex: z.number().int().nonnegative(),
});

// Picking and omitting
const CustomerSummary = CustomerSchema.pick({ id: true, name: true, email: true });
const CustomerUpdate = CustomerSchema.omit({ id: true }).partial();

// Discriminated unions
const PaymentSchema = z.discriminatedUnion('method', [
  z.object({ method: z.literal('card'), cardNumber: z.string(), expiry: z.string() }),
  z.object({ method: z.literal('paypal'), paypalEmail: z.string().email() }),
  z.object({ method: z.literal('bank'), accountNumber: z.string(), routingNumber: z.string() }),
]);
```

### 8.3 Transform and Preprocess

```typescript
// Transform: convert the validated value
const DateSchema = z.string().datetime().transform(str => new Date(str));
type DateOutput = z.output<typeof DateSchema>; // Date (not string)

// Preprocess: modify input before validation
const NumberFromString = z.preprocess(
  (val) => typeof val === 'string' ? parseInt(val, 10) : val,
  z.number().int().positive()
);

NumberFromString.parse('42');  // 42
NumberFromString.parse(42);    // 42
NumberFromString.parse('abc'); // ZodError
```

### 8.4 Zod Performance Considerations

Zod schemas are powerful but can be expensive to instantiate and infer:

```typescript
// DON'T create schemas inside components
function UserForm() {
  // This creates a new schema object every render
  const schema = z.object({ name: z.string() }); // BAD
}

// DO define schemas at module level
const UserFormSchema = z.object({ name: z.string() });

function UserForm() {
  // Use the pre-created schema
  const result = UserFormSchema.safeParse(data);
}
```

**Type inference performance:** For very large schemas (50+ fields, deep nesting), `z.infer` can slow down the type checker. In these cases, define the type explicitly and use `z.ZodType<T>` to validate the schema conforms to it:

```typescript
// For large schemas, define the type explicitly
interface LargeConfig {
  database: DatabaseConfig;
  cache: CacheConfig;
  auth: AuthConfig;
  // ... many more fields
}

// Then validate the schema matches
const LargeConfigSchema: z.ZodType<LargeConfig> = z.object({
  database: DatabaseConfigSchema,
  cache: CacheConfigSchema,
  auth: AuthConfigSchema,
  // ...
});
```

### 8.5 Zod Alternatives

| Library | Bundle Size | Speed | TypeScript DX |
|---------|-------------|-------|---------------|
| Zod | ~14kB | Good | Excellent |
| Valibot | ~1kB | Excellent | Good |
| ArkType | ~5kB | Excellent | Excellent (runtime type syntax) |
| TypeBox | ~8kB | Excellent | Good (JSON Schema based) |

If bundle size is critical (React Native, especially), consider Valibot — it's tree-shakeable and much smaller.

---

## 9. ESLINT ARCHITECTURAL LINTING

ESLint can enforce architectural boundaries, not just code style.

### 9.1 eslint-plugin-boundaries

This plugin enforces which modules can import from which other modules:

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['boundaries'],
  settings: {
    'boundaries/elements': [
      { type: 'feature', pattern: 'src/features/*' },
      { type: 'shared', pattern: 'src/shared/*' },
      { type: 'ui', pattern: 'src/ui/*' },
      { type: 'app', pattern: 'src/app/*' },
    ],
    'boundaries/ignore': ['**/*.test.ts', '**/*.spec.ts'],
  },
  rules: {
    'boundaries/element-types': ['error', {
      default: 'disallow',
      rules: [
        // Features can import from shared and ui
        { from: 'feature', allow: ['shared', 'ui'] },
        // Shared can only import from shared
        { from: 'shared', allow: ['shared'] },
        // UI can import from shared
        { from: 'ui', allow: ['shared'] },
        // App can import from everything
        { from: 'app', allow: ['feature', 'shared', 'ui'] },
        // NOBODY can import from features except app
        // This prevents cross-feature coupling
      ],
    }],
  },
};
```

### 9.2 eslint-plugin-import

```javascript
module.exports = {
  plugins: ['import'],
  rules: {
    // Prevent circular dependencies
    'import/no-cycle': ['error', { maxDepth: 3 }],
    
    // Enforce import order
    'import/order': ['error', {
      groups: [
        'builtin',
        'external',
        'internal',
        ['parent', 'sibling'],
        'index',
        'type',
      ],
      'newlines-between': 'always',
      alphabetize: { order: 'asc' },
    }],
    
    // Prevent importing from parent directories in certain paths
    'import/no-relative-parent-imports': 'off', // Use with boundaries instead
    
    // Ban specific imports
    'no-restricted-imports': ['error', {
      paths: [
        {
          name: 'lodash',
          message: 'Import specific lodash functions: lodash/debounce',
        },
        {
          name: 'moment',
          message: 'Use date-fns or dayjs instead of moment.',
        },
      ],
    }],
  },
};
```

### 9.3 Custom ESLint Rules for Architecture

```javascript
// eslint-local-rules/no-direct-db-access.js
module.exports = {
  create(context) {
    return {
      ImportDeclaration(node) {
        const importPath = node.source.value;
        const filePath = context.getFilename();
        
        // Only allow database imports in repository files
        if (
          importPath.includes('@/database') &&
          !filePath.includes('/repositories/')
        ) {
          context.report({
            node,
            message: 'Database access is only allowed in repository files. Use a repository instead.',
          });
        }
      },
    };
  },
};
```

---

## 10. DEPENDENCY-CRUISER: ENFORCING MODULE BOUNDARIES

dependency-cruiser analyzes your module dependency graph and enforces rules about what can depend on what.

### 10.1 Setup

```bash
npm install --save-dev dependency-cruiser
npx depcruise --init
```

### 10.2 Rule Configuration

```javascript
// .dependency-cruiser.cjs
module.exports = {
  forbidden: [
    {
      name: 'no-circular',
      severity: 'error',
      comment: 'Circular dependencies cause initialization issues and make code harder to reason about',
      from: {},
      to: { circular: true },
    },
    {
      name: 'no-orphans',
      severity: 'warn',
      comment: 'Orphaned modules are not imported by anything — they may be dead code',
      from: { orphan: true, pathNot: ['\\.d\\.ts$', '\\.test\\.ts$', '\\.spec\\.ts$', '\\.config\\.'] },
      to: {},
    },
    {
      name: 'features-cannot-import-features',
      severity: 'error',
      comment: 'Features should not depend on each other directly. Use shared modules.',
      from: { path: '^src/features/([^/]+)/' },
      to: { 
        path: '^src/features/([^/]+)/',
        pathNot: '^src/features/$1/'  // Can import from own feature
      },
    },
    {
      name: 'shared-cannot-import-features',
      severity: 'error',
      comment: 'Shared modules must not depend on features',
      from: { path: '^src/shared/' },
      to: { path: '^src/features/' },
    },
    {
      name: 'no-dev-deps-in-src',
      severity: 'error',
      comment: 'Production code should not import devDependencies',
      from: { path: '^src/' },
      to: { dependencyTypes: ['npm-dev'] },
    },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    tsPreCompilationDeps: true,
    tsConfig: { fileName: './tsconfig.json' },
    enhancedResolveOptions: {
      exportsFields: ['exports'],
      conditionNames: ['import', 'require', 'node', 'default'],
    },
  },
};
```

### 10.3 Visualization

```bash
# Generate a dependency graph as SVG
npx depcruise src --include-only "^src" --output-type dot | dot -T svg > dependency-graph.svg

# Generate a report
npx depcruise src --include-only "^src" --output-type err

# Focus on a specific module
npx depcruise src/features/auth --include-only "^src" --output-type dot | dot -T svg > auth-deps.svg
```

### 10.4 CI Integration

```yaml
# .github/workflows/architecture.yml
name: Architecture Check
on: [pull_request]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - name: Check dependency rules
        run: npx depcruise src --config --output-type err
      - name: Check for new circular dependencies
        run: npx depcruise src --config --output-type err --focus "circular"
```

---

## 11. ADVANCED TYPE PATTERNS

### 11.1 Template Literal Types for API Routes

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';

type ApiRoute = `/api/${string}`;

type RouteHandler<
  Method extends HttpMethod,
  Route extends ApiRoute,
  Body = never,
  Response = unknown,
> = {
  method: Method;
  route: Route;
  handler: (req: { body: Body; params: ExtractParams<Route> }) => Promise<Response>;
};

// Extract path parameters from route string
type ExtractParams<T extends string> = 
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param]: string } & ExtractParams<`/${Rest}`>
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

// Usage
type UserRoute = ExtractParams<'/api/users/:userId/posts/:postId'>;
// { userId: string; postId: string }
```

### 11.2 Builder Pattern with Types

```typescript
class QueryBuilder<
  TTable extends string = never,
  TSelect extends string = '*',
  TWhere extends Record<string, unknown> = {},
> {
  private table?: string;
  private selectFields?: string[];
  private whereClause?: Record<string, unknown>;
  
  from<T extends string>(table: T): QueryBuilder<T, TSelect, TWhere> {
    this.table = table;
    return this as any;
  }
  
  select<S extends string>(...fields: S[]): QueryBuilder<TTable, S, TWhere> {
    this.selectFields = fields;
    return this as any;
  }
  
  where<W extends Record<string, unknown>>(clause: W): QueryBuilder<TTable, TSelect, W> {
    this.whereClause = clause;
    return this as any;
  }
  
  build(): { table: TTable; select: TSelect; where: TWhere } {
    return {
      table: this.table as any,
      select: (this.selectFields ?? '*') as any,
      where: (this.whereClause ?? {}) as any,
    };
  }
}

// Usage
const query = new QueryBuilder()
  .from('users')
  .select('id', 'name', 'email')
  .where({ active: true })
  .build();

// Type: { table: 'users'; select: 'id' | 'name' | 'email'; where: { active: boolean } }
```

### 11.3 Const Assertions and Satisfies

```typescript
// `as const` makes values deeply readonly and literal-typed
const ROUTES = {
  home: '/',
  profile: '/profile',
  settings: '/settings',
  post: '/post/:id',
} as const;

type Route = typeof ROUTES[keyof typeof ROUTES];
// '/' | '/profile' | '/settings' | '/post/:id'

// `satisfies` checks the type without widening
const THEME = {
  colors: {
    primary: '#007AFF',
    secondary: '#5856D6',
    background: '#FFFFFF',
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
  },
} as const satisfies {
  colors: Record<string, `#${string}`>;
  spacing: Record<string, number>;
};

// THEME.colors.primary is type '#007AFF' (literal), not string
// AND it's validated against the satisfies constraint
```

### 11.4 Type-Safe Event Emitter

```typescript
type EventMap = {
  'user:login': { userId: string; timestamp: number };
  'user:logout': { userId: string };
  'order:created': { orderId: string; items: string[] };
  'order:shipped': { orderId: string; trackingNumber: string };
};

class TypedEventEmitter<Events extends Record<string, unknown>> {
  private handlers = new Map<string, Set<Function>>();
  
  on<K extends keyof Events & string>(
    event: K,
    handler: (payload: Events[K]) => void
  ): () => void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, new Set());
    }
    this.handlers.get(event)!.add(handler);
    
    // Return unsubscribe function
    return () => this.handlers.get(event)?.delete(handler);
  }
  
  emit<K extends keyof Events & string>(event: K, payload: Events[K]): void {
    this.handlers.get(event)?.forEach(handler => handler(payload));
  }
}

const events = new TypedEventEmitter<EventMap>();

events.on('user:login', (payload) => {
  // payload is { userId: string; timestamp: number }
  console.log(`User ${payload.userId} logged in at ${payload.timestamp}`);
});

events.emit('user:login', { userId: '123', timestamp: Date.now() }); // OK
events.emit('user:login', { userId: '123' }); // ERROR: missing timestamp
events.emit('user:logout', { userId: '123', timestamp: 0 }); // ERROR: no timestamp in logout
```

---

## 12. CHAPTER SUMMARY

TypeScript at scale requires a different approach than TypeScript in a small project:

1. **Project references** split your codebase into independently type-checked units. Combined with Turborepo/Nx, they give you incremental, cached, parallel type checking.

2. **Barrel files are an anti-pattern** at scale. They cause module graph explosion, circular dependencies, and IDE slowdowns. Use direct imports and `package.json` exports.

3. **Diagnostics are essential.** Use `--extendedDiagnostics` to find bottlenecks, `--generateTrace` to identify expensive types, and `@typescript/analyze-trace` to get actionable insights.

4. **Monorepo tsconfig** should use a base config for shared settings, composite projects for each package, and `moduleResolution: "bundler"` for modern module resolution.

5. **Discriminated unions** are the most powerful type pattern for application code. They make invalid states unrepresentable and enable exhaustive checking.

6. **Branded types** prevent stringly-typed APIs — you can't accidentally swap a UserId for an OrderId.

7. **Zod** bridges the compile-time/runtime gap. For large schemas, define types explicitly and use `z.ZodType<T>` to avoid type inference performance issues.

8. **ESLint boundaries and dependency-cruiser** enforce architectural constraints at lint time and in CI. They prevent the slow architectural decay that makes large codebases unmaintainable.

The difference between a TypeScript codebase that scales and one that doesn't is not about using more types — it's about using the right types in the right places, structuring the project so the compiler can do its job efficiently, and enforcing boundaries so the dependency graph stays clean.

---

> **Next:** [Chapter 5: Expo Platform] covers the Expo ecosystem — the build system, development workflow, and how Expo builds on the React Native architecture from Chapter 1.
