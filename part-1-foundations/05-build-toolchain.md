<!--
  CHAPTER: 5
  TITLE: The Build Toolchain — From package.json to Running App
  PART: I — Foundations
  PREREQS: Chapters 0, 0d
  KEY_TOPICS: bundlers, transpilers, Babel, SWC, esbuild, Metro, Webpack, Vite, Turbopack, Rspack, Bun, package.json anatomy, module systems, ESM vs CJS, tree shaking, code splitting, minification, source maps, hot module replacement
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 5: The Build Toolchain — From package.json to Running App

> **Part I — Foundations** | Prerequisites: Chapters 0, 0d | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Your TypeScript/JSX code cannot run anywhere directly; it goes through transpilation (TS/JSX to JS), bundling (resolving imports into one or more files), tree shaking, minification, and source map generation before it reaches a device
- SWC and esbuild have largely replaced Babel for transpilation; Metro handles React Native bundling, Vite dominates web, and Turbopack is the Next.js default
- Tree shaking eliminates dead code at build time but only works with ES modules; a single side-effectful barrel file can pull your entire dependency tree into the bundle
- Source maps are what let you debug production crashes with real file names and line numbers; ship them to your error tracking service, not to your users
- Hot Module Replacement swaps changed modules at runtime without losing component state; understanding its boundaries prevents the "why didn't my change show up" confusion

</details>

Here is the dirty secret of frontend engineering in 2026: **most developers have no idea what happens between the code they write and the code that runs.** They write TypeScript. They write JSX. They import from node_modules. They run `npm run build` and something comes out the other end. When it works, they don't think about it. When it breaks, they paste the error into a search engine and pray.

This chapter ends that. We are going to open the black box — every step of it — from the `package.json` that defines your project to the final bytes that land on a phone or browser. You will understand what transpilation actually does, why there are six competing bundlers, how tree shaking decides what code lives and what code dies, why your source maps matter for crash reports at 3am, and how Hot Module Replacement makes your changes appear instantly without blowing away your application state.

If you have ever stared at a webpack config and felt like you were reading an alien language, this chapter is for you. If you have ever wondered why your bundle is 2MB when your source code is 200KB, this chapter is for you. If you have ever had a production crash where the stack trace pointed to `a.js:1:48293` and wished you understood source maps, this chapter is *especially* for you.

### In This Chapter
- The question nobody answers: what happens between TypeScript and the browser?
- package.json anatomy: every field that matters
- Module systems: CommonJS vs ESM, and why the ecosystem is finally converging
- The transpilation step: Babel, SWC, esbuild, TSC
- The bundling step: Metro, Webpack, Vite, Turbopack, Rspack, Bun
- Tree shaking: how bundlers eliminate dead code
- Code splitting: loading only what you need
- Minification and compression: making bytes smaller
- Source maps: mapping compiled code back to your source
- Hot Module Replacement: instant feedback loops
- The full pipeline: tracing a file from .tsx to running app
- Bun as a runtime: the all-in-one challenger

### Related Chapters
- [Ch 1: React Native Internals] — Hermes bytecode compilation, the last step of the mobile pipeline
- [Ch 4: TypeScript at Scale] — tsconfig, project references, the type system that feeds the transpiler
- [Ch 13: Performance Optimization] — bundle size budgets, lazy loading strategies
- [Ch 17: CI/CD] — how build toolchains integrate into continuous integration

---

## 1. THE QUESTION NOBODY ANSWERS

You open your editor. You write this:

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { useUser } from '@/hooks/useUser';

export function ProfileScreen() {
  const { user, isLoading } = useUser();

  if (isLoading) return <View style={styles.center}><Text>Loading...</Text></View>;

  return (
    <View style={styles.container}>
      <Text style={styles.name}>{user?.name ?? 'Anonymous'}</Text>
      <Text style={styles.email}>{user?.email}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  name: { fontSize: 24, fontWeight: 'bold' },
  email: { fontSize: 16, color: '#666' },
});
```

This file cannot run anywhere. Not in a browser. Not on a phone. Not in Node.js. It contains:

- **TypeScript** — browsers and phones run JavaScript, not TypeScript
- **JSX** — `<View>` and `<Text>` are not valid JavaScript syntax
- **Path aliases** — `@/hooks/useUser` is not a real file path that any runtime understands
- **ES module imports** — some runtimes don't support `import` (older Node.js, some build targets)
- **Optional chaining** — `user?.name` needs to be compiled for older targets
- **Nullish coalescing** — `??` is the same story

Between this file and a running application, at least five things need to happen:

```
Your .tsx file
     │
     ▼
┌─────────────┐
│  TYPE CHECK  │  tsc --noEmit: "Does this code have type errors?"
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  TRANSPILE   │  SWC/Babel: "Convert TSX/TS to plain JavaScript"
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   BUNDLE     │  Webpack/Vite/Metro: "Resolve all imports, combine into output files"
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  OPTIMIZE    │  Tree shake, split, minify, compress
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   SERVE      │  Deploy to CDN, compile to Hermes bytecode, etc.
└─────────────┘
```

Each of these steps has an entire ecosystem of tools. Each tool makes different tradeoffs. And the choices you make here affect bundle size, build speed, developer experience, and production performance in ways that are invisible until they bite you.

Let's start at the beginning: the file that defines your entire project.

---

## 2. PACKAGE.JSON ANATOMY

Every JavaScript/TypeScript project starts with `package.json`. Most developers treat it as "the file where dependencies live." It is actually a rich configuration document that tells the entire ecosystem — npm, bundlers, runtimes, TypeScript — what your project is, how to use it, and what it exports.

Here is a production-quality `package.json` for a shared library in a monorepo:

```json
{
  "name": "@acme/ui-components",
  "version": "2.4.1",
  "description": "Shared UI component library for Acme mobile and web apps",
  "license": "MIT",
  "type": "module",

  "main": "./dist/cjs/index.js",
  "module": "./dist/esm/index.js",
  "types": "./dist/types/index.d.ts",

  "exports": {
    ".": {
      "import": {
        "types": "./dist/types/index.d.ts",
        "default": "./dist/esm/index.js"
      },
      "require": {
        "types": "./dist/types/index.d.ts",
        "default": "./dist/cjs/index.cjs"
      }
    },
    "./Button": {
      "import": {
        "types": "./dist/types/Button.d.ts",
        "default": "./dist/esm/Button.js"
      },
      "require": {
        "types": "./dist/types/Button.d.ts",
        "default": "./dist/cjs/Button.cjs"
      }
    },
    "./package.json": "./package.json"
  },

  "files": ["dist", "README.md"],
  "sideEffects": false,

  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch",
    "typecheck": "tsc --noEmit",
    "lint": "eslint src/",
    "test": "vitest run",
    "test:watch": "vitest"
  },

  "dependencies": {
    "react": "^19.0.0",
    "react-native": "^0.78.0"
  },

  "devDependencies": {
    "@types/react": "^19.0.0",
    "tsup": "^8.4.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0",
    "eslint": "^9.0.0"
  },

  "peerDependencies": {
    "react": ">=18.0.0",
    "react-native": ">=0.72.0"
  },

  "engines": {
    "node": ">=20.0.0"
  }
}
```

Let's break down every field that matters.

### 2.1 Identity Fields

**`name`** — The package name. Scoped packages (`@acme/ui-components`) are namespaced to an organization. This prevents name collisions on npm and makes it clear who owns what in a monorepo.

**`version`** — Follows semver (semantic versioning): `MAJOR.MINOR.PATCH`. Breaking changes bump major, new features bump minor, bug fixes bump patch. In a monorepo with internal packages, version matters less (you're consuming from source), but for published packages, it's your contract with consumers.

**`description`** and **`license`** — Metadata. npm uses description for search. License tells consumers what they can do with your code.

### 2.2 Entry Points: main, module, types

These three fields are the "old way" of telling consumers where your code lives:

**`main`** — The CommonJS entry point. When someone does `require('@acme/ui-components')`, Node.js looks here. Points to a `.js` file with `module.exports`.

**`module`** — The ES module entry point. Bundlers (Webpack, Rollup, Vite) look here first. Points to a `.js` file with `export` statements. **This is not an official Node.js field** — it's a convention that bundlers adopted. Node.js ignores it.

**`types`** — The TypeScript type definition entry point. When TypeScript resolves `import { Button } from '@acme/ui-components'`, it looks here for the `.d.ts` file that describes the types.

```
Consumer writes:                     Resolution order:
import { Button } from '@acme/ui'   1. Check "exports" (modern)
                                     2. Check "module" (bundlers)
                                     3. Check "main" (Node.js CJS)
                                     
TypeScript type resolution:          1. Check "exports" → "types" condition
                                     2. Check "types" / "typings" field
                                     3. Look for index.d.ts next to "main"
```

### 2.3 The "exports" Field — The Most Important Field in 2026

The `exports` field (also called "conditional exports" or "export maps") is a package.json feature introduced in Node.js 12.7 that has become **the single most important field** for library authors. It replaces `main`, `module`, and `types` with a unified, conditional system.

Why is it so important? Because it solves four problems at once:

**1. Conditional resolution.** Different consumers get different files:

```json
{
  "exports": {
    ".": {
      "import": "./dist/esm/index.js",    // ES module consumers
      "require": "./dist/cjs/index.cjs",   // CommonJS consumers
      "react-native": "./src/index.ts",    // React Native (Metro) gets source
      "browser": "./dist/browser/index.js", // Browser-specific build
      "default": "./dist/esm/index.js"     // Fallback
    }
  }
}
```

Metro sees the `react-native` condition and gets the TypeScript source directly (it handles transpilation itself). Vite sees the `import` condition and gets the pre-built ESM. Webpack sees `import` or `require` depending on the import syntax. Everyone gets the right thing.

**2. Subpath exports.** You can expose specific sub-modules without exposing your entire file structure:

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./Button": "./dist/components/Button.js",
    "./hooks": "./dist/hooks/index.js",
    "./utils": "./dist/utils/index.js"
  }
}
```

Now `import { Button } from '@acme/ui/Button'` works, but `import { internal } from '@acme/ui/dist/internal/secret'` does not. The `exports` field acts as an **access control list** for your package.

**3. Type resolution.** The `types` condition inside exports tells TypeScript exactly where to find type definitions for each subpath:

```json
{
  "exports": {
    "./Button": {
      "import": {
        "types": "./dist/types/Button.d.ts",
        "default": "./dist/esm/Button.js"
      }
    }
  }
}
```

**Important:** The `types` condition must come **first** within each condition block. TypeScript stops at the first match, and if `default` comes before `types`, TypeScript will try to derive types from the JavaScript file (and fail or infer `any`).

**4. Encapsulation.** Without `exports`, consumers can deep-import anything: `import { something } from '@acme/ui/dist/internal/helpers/utils'`. With `exports`, only the paths you explicitly expose are accessible. Everything else throws a `ERR_PACKAGE_PATH_NOT_EXPORTED` error. This protects you from consumers depending on your internal file structure.

> **The migration pain:** If you add `exports` to a package that didn't have it, you might break consumers who were deep-importing internal files. This is why many library authors add `exports` gradually, starting with the main entry point and adding subpaths as needed. But for new packages in 2026, there is no reason not to use `exports` from day one.

### 2.4 "type": "module"

This field tells Node.js how to interpret `.js` files in your package:

- `"type": "module"` — `.js` files are ES modules (use `import`/`export`)
- `"type": "commonjs"` (or omitted) — `.js` files are CommonJS (use `require`/`module.exports`)

Regardless of this setting:
- `.mjs` files are **always** ES modules
- `.cjs` files are **always** CommonJS

**The 2026 reality:** Most new projects set `"type": "module"`. The ecosystem has converged enough that ESM-first is the default. But you'll still encounter CJS-only packages in production codebases, which is why dual publishing (providing both ESM and CJS builds) remains necessary for libraries.

### 2.5 Dependencies

Three kinds of dependencies, each with a different purpose:

**`dependencies`** — Packages your code needs at runtime. Installed when someone `npm install`s your package. If your component imports `lodash` at runtime, `lodash` goes here.

**`devDependencies`** — Packages you need during development and build, but not at runtime. TypeScript, ESLint, Vitest, your bundler — all devDependencies. They are **not** installed when someone consumes your package as a dependency.

**`peerDependencies`** — Packages your code needs, but that should be provided by the consumer. The classic example: a React component library lists `react` as a peer dependency because it needs React to work, but it should use the *same* React instance as the consuming app, not bundle its own copy.

```
Your app
├── react@19.0.0           ← installed once
├── @acme/ui-components    ← peerDeps: react >= 18
│   └── (uses the app's react@19.0.0, not its own copy)
└── @acme/data-hooks       ← peerDeps: react >= 18
    └── (also uses the app's react@19.0.0)
```

**Without peerDependencies:** If both `@acme/ui-components` and `@acme/data-hooks` listed `react` as a regular dependency, npm might install two copies of React. Two React instances in one app means hooks break, context doesn't work, and you get the infamous "Invalid Hook Call" error. peerDependencies prevent this.

**The npm 7+ change:** Since npm 7, peer dependencies are installed automatically by default. Before npm 7, they were just warnings. This means your `peerDependencies` version ranges matter more than ever — a too-strict range blocks installation; a too-loose range might allow incompatible versions.

### 2.6 The "scripts" Field

Scripts are the command-line interface to your project. The convention has settled on a few standard names:

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "clean": "rm -rf .next dist node_modules/.cache"
  }
}
```

**`npm run <script>`** or **`yarn <script>`** or **`pnpm <script>`** or **`bun <script>`** — they all execute the command. The key insight: scripts run with `node_modules/.bin` prepended to the `PATH`, so you can reference any installed binary directly (`vitest` instead of `./node_modules/.bin/vitest`).

**Pre/post hooks:** npm supports `pre<script>` and `post<script>` hooks. `prebuild` runs before `build`, `postbuild` runs after. Useful for cleaning dist directories before builds or running type checks after.

### 2.7 "sideEffects"

This field tells bundlers whether your package's modules have side effects. A module with side effects is one that does something just by being imported — registers a global, modifies a prototype, writes to the DOM.

```json
{
  "sideEffects": false
}
```

`"sideEffects": false` means: "Every module in this package is pure. If you import something from this package and don't use it, the entire module can be safely eliminated." This is critical for tree shaking (Section 6).

You can also be more specific:

```json
{
  "sideEffects": ["./src/global-styles.css", "./src/polyfills.js"]
}
```

This says: "Most modules are pure, but these specific files have side effects — don't eliminate them even if nothing is explicitly imported from them."

### 2.8 "engines" and "workspaces"

**`engines`** — Declares which Node.js (and npm/yarn/pnpm) versions your project supports:

```json
{
  "engines": {
    "node": ">=20.0.0",
    "pnpm": ">=9.0.0"
  }
}
```

This is advisory by default. To make it enforced, add an `.npmrc` with `engine-strict=true`. In a monorepo, `engines` in the root `package.json` sets the baseline for the entire project.

**`workspaces`** — Defines a monorepo structure. The root `package.json` declares which directories contain sub-packages:

```json
{
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

With this, `npm install` (or `yarn install`, or `pnpm install`) hoists shared dependencies to the root, creates symlinks between workspace packages, and lets you import `@acme/ui-components` from `@acme/mobile-app` without publishing to npm. Each package manager handles workspaces slightly differently — pnpm uses a strict, non-hoisted approach by default, while npm and yarn hoist aggressively.

### 2.9 "files"

Controls what gets published to npm. Without `files`, npm publishes everything (minus what's in `.npmignore` and `.gitignore`). With `files`, only the listed paths are included:

```json
{
  "files": ["dist", "README.md", "LICENSE"]
}
```

**Always set this** for published packages. Without it, you'll accidentally publish your `src/`, test files, `.env` files, and other things that don't belong in the package.

---

## 3. MODULE SYSTEMS — COMMONJS VS ESM

This is one of those topics where, if you don't understand the history, the current state makes no sense. So let's start from the beginning.

### 3.1 CommonJS (CJS): The Original Module System

When Node.js launched in 2009, JavaScript had no module system. Browsers loaded scripts with `<script>` tags, and everything was global. Node.js needed a way to structure server-side code into files, so Ryan Dahl adopted the CommonJS specification:

```js
// math.js — exporting
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;
module.exports = { add, multiply };

// app.js — importing
const { add, multiply } = require('./math');
console.log(add(2, 3)); // 5
```

**How `require()` works under the hood:**

1. **Resolve** the file path (check `./math`, `./math.js`, `./math/index.js`, then `node_modules/`)
2. **Load** the file contents from disk
3. **Wrap** the contents in a function: `(function(exports, require, module, __filename, __dirname) { ... })`
4. **Execute** the wrapped function
5. **Cache** the `module.exports` value — subsequent `require()` calls return the cached value, they don't re-execute the file
6. **Return** `module.exports`

**The key characteristic of CJS: `require()` is synchronous and dynamic.** It executes at runtime, and you can put it anywhere — inside an `if` statement, inside a loop, inside a function. This makes CJS extremely flexible but fundamentally impossible to statically analyze.

```js
// This is valid CJS — and it's why tree shaking can't work with CJS
const moduleName = process.env.USE_FANCY ? 'fancy-module' : 'basic-module';
const lib = require(moduleName);  // which module? depends on runtime.

if (someCondition) {
  require('./side-effect-module');  // imported or not? depends on runtime.
}
```

### 3.2 ES Modules (ESM): The Standard

ES2015 (ES6) introduced a native module system to JavaScript itself. This is the one you know:

```js
// math.js — exporting
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b;
export default function subtract(a, b) { return a - b; }

// app.js — importing
import { add, multiply } from './math.js';
import subtract from './math.js';
```

**How ESM differs from CJS:**

1. **Static structure.** `import` and `export` must be at the top level — not inside `if` statements, not inside functions. The module graph is known before any code executes.
2. **Asynchronous loading.** ESM modules are loaded asynchronously (this matters for browsers, which fetch modules over the network).
3. **Live bindings.** ESM exports are live references, not copies. If the exporting module changes a value, the importing module sees the change. CJS exports are copies (snapshot at the time of `require()`).
4. **Strict mode by default.** ESM files always run in strict mode.

**Why the static structure matters enormously:** Because the import graph is known at build time (before any code runs), bundlers can:
- Know exactly which exports are used and which are not → **tree shaking**
- Build the complete dependency graph before execution → **deterministic builds**
- Analyze circular dependencies at build time → **early error detection**

```
CJS: require() is dynamic → bundler can't know what's imported until runtime → no tree shaking
ESM: import is static   → bundler knows the full graph at build time    → tree shaking works
```

### 3.3 The "type": "module" Transition

Node.js added ESM support gradually, starting with experimental flags in Node 12 and becoming stable in Node 16+. The transition mechanism is the `"type"` field in `package.json`:

```
package.json "type" field    .js files parsed as    .mjs files    .cjs files
─────────────────────────    ──────────────────     ──────────    ──────────
"commonjs" (or omitted)      CommonJS               ESM           CommonJS
"module"                     ESM                    ESM           CommonJS
```

**The interop problem:** CJS can `require()` CJS, and ESM can `import` ESM. But cross-format is painful:

```js
// ESM can import CJS (with caveats)
import cjsModule from 'cjs-package';          // gets the default export (module.exports)
// Named imports from CJS may or may not work, depending on the package and Node version

// CJS cannot require() ESM synchronously
const esmModule = require('esm-package');       // ERROR!
// CJS must use dynamic import() for ESM
const esmModule = await import('esm-package');  // works, but now you need top-level await or async
```

This interop asymmetry is why the CJS-to-ESM migration has been so painful. A popular package switching to ESM-only can break all its CJS consumers. Libraries like `chalk`, `got`, and `node-fetch` went ESM-only and caused widespread breakage. The ecosystem learned: **if you're a library, publish both formats** (dual publishing).

### 3.4 Dual Package Publishing

The recommended approach for libraries in 2026 is to publish both CJS and ESM:

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.ts",
        "default": "./dist/index.js"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  }
}
```

Tools like **tsup**, **unbuild**, and **pkgroll** make dual publishing straightforward — they take your TypeScript source and produce both `.js` (ESM) and `.cjs` (CJS) outputs.

```bash
# tsup handles dual publishing with one command
npx tsup src/index.ts --format cjs,esm --dts

# Output:
# dist/
#   index.js      (ESM)
#   index.cjs     (CJS)
#   index.d.ts    (TypeScript declarations for ESM)
#   index.d.cts   (TypeScript declarations for CJS)
```

### 3.5 How Node.js Resolves Modules

When you write `import { something } from 'some-package'`, Node.js follows a resolution algorithm:

```
1. Is it a built-in? (fs, path, http, etc.) → Use built-in
2. Does it start with ./ or ../ or / ? → Resolve as relative file path
3. Otherwise → Search node_modules directories, walking up the file tree:
   ./node_modules/some-package/
   ../node_modules/some-package/
   ../../node_modules/some-package/
   (repeat until root)

For each candidate directory:
   a. Check package.json "exports" field (if it exists, use it exclusively)
   b. Check package.json "main" field
   c. Check index.js

For file resolution (relative imports):
   a. Exact path: ./utils.js
   b. Add extensions: ./utils → ./utils.js, ./utils.mjs, ./utils.cjs
   c. Directory: ./utils → ./utils/index.js
```

**Bundlers extend this.** Webpack, Vite, and Metro all add their own resolution logic on top of Node's algorithm — path aliases (`@/` prefix), platform-specific extensions (`.ios.js`, `.android.js`, `.web.js`), and custom conditions.

### 3.6 The State of the Ecosystem in 2026

The ecosystem has largely converged on ESM. Here's where things stand:

- **New packages** are almost universally ESM-first (many are ESM-only)
- **Established packages** mostly offer dual ESM + CJS builds
- **Legacy packages** are CJS-only but still work because bundlers handle the interop
- **Node.js** has full ESM support and recently added `require()` for ESM modules behind a flag (Node 22+), easing the migration further
- **Bundlers** handle CJS/ESM interop transparently for app developers — you rarely think about it when building an app
- **Library authors** still need to think about it carefully for their published output

**The practical advice:** For applications, set `"type": "module"` and write ESM. Your bundler handles everything else. For libraries, dual publish with `exports` until the last CJS consumers in your ecosystem have migrated.

---

## 4. THE TRANSPILATION STEP

You write TypeScript with JSX. Runtimes execute JavaScript. Something needs to convert one to the other. That something is a **transpiler** (sometimes called a "compiler," though purists will argue the distinction).

Transpilation does several things:
- **Strip TypeScript types** — `const x: string = "hello"` becomes `const x = "hello"`
- **Convert JSX** — `<Button onClick={fn}>Click</Button>` becomes `React.createElement(Button, { onClick: fn }, "Click")` or the new JSX transform's `_jsx(Button, { onClick: fn, children: "Click" })`
- **Downlevel syntax** — Convert modern JavaScript features to older syntax for target environments that don't support them
- **Apply code transforms** — Custom plugins that modify the AST (Abstract Syntax Tree)

### 4.1 Babel — The OG Transpiler

Babel was the tool that made modern JavaScript possible before browsers caught up. When ES2015 shipped with arrow functions, classes, destructuring, and template literals, Babel let developers use those features immediately by compiling them to ES5 that every browser understood.

**How Babel works — the three phases:**

```
Source Code (string)
       │
       ▼
┌─────────────┐
│    PARSE     │  Source → Abstract Syntax Tree (AST)
│  @babel/     │  Turns your code into a tree data structure
│   parser     │  representing every expression, statement, identifier
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  TRANSFORM   │  AST → Modified AST
│  plugins &   │  Each plugin visits AST nodes and modifies them
│  presets     │  (e.g., "replace ArrowFunctionExpression with FunctionExpression")
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  GENERATE    │  Modified AST → Output Code (string) + Source Map
│  @babel/     │  Turns the tree back into code
│  generator   │  
└─────────────┘
```

**Presets are collections of plugins.** The three you'll see everywhere:

**`@babel/preset-env`** — Compiles modern JavaScript (ES2015+) to whatever target you specify. It reads your `browserslist` config (or `targets` option) and only applies the transforms needed:

```json
// babel.config.json
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.25%, not dead",
      "modules": false,
      "useBuiltIns": "usage",
      "corejs": 3
    }]
  ]
}
```

`"modules": false` is critical — it tells Babel not to convert ES module `import`/`export` to CommonJS `require`/`module.exports`. You want to preserve ES modules so your bundler can tree-shake them.

**`@babel/preset-typescript`** — Strips TypeScript syntax. This is a fast, dumb transform: it removes type annotations without checking them. It does not do type checking. This is why most projects separate type checking (`tsc --noEmit`) from transpilation (Babel or SWC).

**`@babel/preset-react`** — Converts JSX to JavaScript function calls. With the "automatic" runtime (React 17+), it imports `jsx` from `react/jsx-runtime` instead of requiring you to `import React from 'react'` in every file.

```json
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic"
    }]
  ]
}
```

**Plugins** are individual transforms. Some examples:
- `@babel/plugin-proposal-decorators` — Enables decorator syntax
- `babel-plugin-module-resolver` — Rewrites import paths (for path aliases like `@/`)
- `react-native-reanimated/plugin` — Transforms Reanimated worklet functions

**Why Babel is being replaced:**

Babel is written in JavaScript. It parses your code into an AST (in JavaScript), transforms the AST (in JavaScript), and generates code from the AST (in JavaScript). Every step is CPU-bound work running in a single-threaded JavaScript process.

For a small project, this is fine. For a large monorepo with thousands of files, Babel transpilation can take 30-60 seconds on a cold build. In development, you feel this as slow HMR — every file change requires a full parse-transform-generate cycle.

The newer transpilers (SWC, esbuild) do the same work in compiled languages (Rust, Go) and are **20-70x faster.**

### 4.2 SWC — The Rust-Powered Successor

SWC (Speedy Web Compiler) is a Rust-based transpiler created by Donny (강동윤). It is designed as a drop-in replacement for Babel, implementing the same transform model but compiled to native machine code.

**Why SWC is fast:**

1. **Native code.** Rust compiles to machine code that runs at CPU speed, not interpreted JavaScript.
2. **Parallelism.** Rust's ownership model enables safe multi-threaded parsing and transformation. SWC can process multiple files simultaneously.
3. **Efficient memory.** Rust has no garbage collector — memory is allocated and freed deterministically, avoiding GC pauses that plague large Babel builds.
4. **Optimized parser.** SWC's parser is specifically optimized for throughput, not generality.

**Where SWC is used in 2026:**
- **Next.js** — Uses SWC as its default transpiler (replaced Babel in Next.js 12+)
- **Turbopack** — Built on SWC internally
- **Parcel** — Uses SWC for JavaScript/TypeScript transforms
- **Deno** — Uses SWC for TypeScript stripping
- **rspack** — Uses SWC's transforms through its Rust integration

**SWC configuration** (`.swcrc`):

```json
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "tsx": true,
      "decorators": true
    },
    "transform": {
      "react": {
        "runtime": "automatic",
        "importSource": "react"
      }
    },
    "target": "es2022",
    "minify": {
      "compress": true,
      "mangle": true
    }
  },
  "module": {
    "type": "es6"
  }
}
```

**The plugin gap:** SWC supports plugins written in Wasm, but the ecosystem is much smaller than Babel's. If you need a specific Babel plugin that doesn't have an SWC equivalent, you're stuck with Babel for that transform. In practice, the most common plugins (React, TypeScript, module resolution, decorators) all have SWC equivalents, so this gap affects fewer teams every year.

**Benchmarks (real-world, not synthetic):**

```
Project: 3,200 TypeScript files, React Native app
─────────────────────────────────────────────────
Transpiler        Cold Build    HMR (single file)
─────────────     ──────────    ─────────────────
Babel             34.2s         420ms
SWC               1.8s          18ms
─────────────────────────────────────────────────
Speedup: ~19x cold, ~23x HMR
```

These numbers vary by project, but the magnitude is consistent: SWC is 15-70x faster depending on the workload.

### 4.3 esbuild — The Go-Powered Speedster

esbuild is a JavaScript bundler and transpiler written in Go by Evan Wallace (co-founder of Figma). It was designed from the ground up for speed, and it achieves that speed through aggressive parallelism and minimal abstraction.

**What esbuild does:**
- Transpiles TypeScript and JSX to JavaScript
- Bundles modules together (it's also a bundler — more in Section 5)
- Minifies output
- Generates source maps

**What esbuild doesn't do (or does minimally):**
- **No type checking.** Like SWC, it strips types without checking them.
- **Limited plugin API.** esbuild's plugin system is intentionally minimal. Complex transforms that Babel plugins handle (like Reanimated worklets or Relay's GraphQL transform) are not possible in esbuild.
- **No support for some TypeScript features.** `const enum`, `namespace`, and some decorator proposals are not supported.

**Where esbuild shines:**

esbuild is the transpiler behind **Vite's development server.** When you run `vite dev`, Vite uses esbuild to transpile individual files on-demand as the browser requests them. Because esbuild can transpile a single file in under a millisecond, Vite's dev server feels instantaneous.

```bash
# esbuild as a standalone transpiler
npx esbuild src/app.tsx --bundle --outfile=dist/app.js --format=esm --target=es2022

# esbuild as a standalone minifier
npx esbuild dist/app.js --minify --outfile=dist/app.min.js
```

**The esbuild + Vite relationship:**
```
Development:  Vite uses esbuild for transpilation (fast transforms)
Production:   Vite uses Rollup for bundling (better tree shaking, code splitting)
```

This dual-engine approach is Vite's key insight: use the fastest tool for development (where speed matters most) and the most optimized tool for production (where output quality matters most).

### 4.4 TSC — The TypeScript Compiler

TypeScript's own compiler (`tsc`) can both type-check and transpile. But in modern projects, these two jobs are almost always separated.

**Why separate type checking from transpilation?**

```
tsc (does everything):
  Parse TS → Build type graph → Check types → Emit JS
  Speed: slow (must understand entire program's type relationships)

SWC/Babel (transpile only):
  Parse TS → Strip types → Emit JS
  Speed: fast (no type analysis needed)

tsc --noEmit (type check only):
  Parse TS → Build type graph → Check types → Done (no JS output)
  Speed: slow, but runs in parallel with your bundler
```

**The practical pattern in every modern project:**

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "typecheck": "tsc --noEmit"
  }
}
```

Your bundler (Next.js/Turbopack, Vite, Metro) uses SWC or esbuild for fast transpilation during development and builds. `tsc --noEmit` runs separately — in your editor (via the TypeScript language server), in CI (as a pre-commit or build step), or as a parallel process.

**Why this split matters for speed:**

Type checking is inherently slow because TypeScript must understand the relationships between *every* type in your program. Changing one file can cascade into re-checking hundreds of files. If your transpiler had to wait for type checking, every HMR update would be slow.

By decoupling them, your dev server stays fast (SWC transpiles without type checking) while your editor and CI still catch type errors (tsc runs in the background).

**TSC configuration for monorepos (the relevant bits):**

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true
  }
}
```

Two flags deserve explanation:

**`isolatedModules: true`** — Ensures every file can be transpiled in isolation, without understanding other files. This is required when using SWC, esbuild, or Babel for transpilation, because they process files one at a time. It flags patterns that require cross-file type information (like `const enum` re-exports).

**`verbatimModuleSyntax: true`** — Requires you to use `import type` for type-only imports. This makes it unambiguous which imports are values (keep them in the output) and which are types (remove them during transpilation). Without this, the transpiler has to guess — and sometimes guesses wrong.

```ts
// With verbatimModuleSyntax: true
import { UserService } from './services';       // value import — kept in output
import type { User } from './types';             // type import — removed during transpilation
import { type Config, createApp } from './app';  // mixed — Config removed, createApp kept
```

### 4.5 Transpiler Comparison Summary

```
┌──────────┬───────────┬────────────┬─────────────┬──────────────┐
│          │   Babel   │    SWC     │   esbuild   │     TSC      │
├──────────┼───────────┼────────────┼─────────────┼──────────────┤
│ Language │ JavaScript│   Rust     │     Go      │  TypeScript  │
│ Speed    │ Baseline  │  20-70x   │  10-100x   │   ~1x        │
│ TS types │ Strip     │   Strip    │   Strip     │ Check+Strip  │
│ JSX      │ ✓         │   ✓        │   ✓         │   ✓          │
│ Plugins  │ Massive   │  Growing   │   Limited   │   None       │
│ Minify   │ No*       │   ✓        │   ✓         │   No         │
│ Bundle   │ No        │   No**     │   ✓         │   No         │
│ Used by  │ RN/Metro  │ Next.js    │  Vite dev   │ Type check   │
│          │ (legacy)  │ Turbopack  │  tsup       │ only (2026)  │
│          │           │ Rspack     │  Vitest     │              │
└──────────┴───────────┴────────────┴─────────────┴──────────────┘

* Babel has no built-in minifier; projects use Terser alongside it
** SWC has experimental bundling (swcpack) but it's not production-ready
```

---

## 5. THE BUNDLING STEP

Transpilation converts individual files from TypeScript/JSX to JavaScript. Bundling takes all those individual files — your code, your dependencies, their dependencies — and combines them into output files that can actually be loaded by a browser or runtime.

### 5.1 What Bundlers Actually Do

A bundler's job is more complex than "concatenating files." Here's the full process:

```
1. ENTRY POINT(S)
   Start from one or more entry files (e.g., src/index.tsx)
        │
        ▼
2. RESOLUTION
   For every import statement, resolve the actual file path:
   - 'react' → node_modules/react/index.js
   - './Button' → src/components/Button.tsx
   - '@/hooks/useUser' → src/hooks/useUser.ts
        │
        ▼
3. DEPENDENCY GRAPH
   Build a directed graph of every file and its dependencies:
   index.tsx → [App.tsx, react, react-dom]
   App.tsx → [Button.tsx, useUser.ts, react]
   Button.tsx → [react, styles.css]
   useUser.ts → [react, @tanstack/react-query, api.ts]
   ...
        │
        ▼
4. TRANSFORMATION
   Apply transforms to each file:
   - TypeScript → JavaScript (SWC/Babel/esbuild)
   - CSS → CSS Modules / extracted stylesheets
   - Images → data URLs or asset references
   - JSON → JavaScript objects
        │
        ▼
5. TREE SHAKING
   Analyze exports: which are imported by other modules? Remove unused exports.
        │
        ▼
6. CODE SPLITTING
   At dynamic import() boundaries, split the graph into chunks:
   - Main chunk (always loaded)
   - Route chunks (loaded on navigation)
   - Vendor chunks (shared dependencies)
        │
        ▼
7. OPTIMIZATION
   Minify each chunk, generate source maps, compute content hashes for filenames.
        │
        ▼
8. OUTPUT
   Write final files to disk:
   dist/
     index-a1b2c3.js      (main bundle)
     profile-d4e5f6.js     (route chunk)
     vendor-789abc.js      (shared deps)
     index-a1b2c3.js.map   (source map)
     index.html             (with script tags referencing the above)
```

Now let's look at each major bundler and understand where it fits.

### 5.2 Metro — React Native's Bundler

Metro is the JavaScript bundler for React Native. It's built by Meta specifically for mobile development, and it works differently from web bundlers in ways that matter.

**How Metro works — three phases:**

```
┌─────────────┐     ┌─────────────────┐     ┌───────────────┐
│  RESOLUTION  │ ──▶ │  TRANSFORMATION  │ ──▶ │ SERIALIZATION  │
│              │     │                  │     │                │
│ Resolve each │     │ Transpile each   │     │ Combine into   │
│ import to a  │     │ file individually│     │ a single JS    │
│ file path    │     │ (Babel or SWC)   │     │ bundle         │
└─────────────┘     └─────────────────┘     └───────────────┘
```

**Key differences from web bundlers:**

**1. Per-file transformation.** Metro transforms each file independently. There's no "module concatenation" or "scope hoisting" like Webpack does. Each file is wrapped in a function and gets a numeric module ID:

```js
// Metro output (simplified)
__d(function(global, _$$_REQUIRE, _$$_IMPORT_DEFAULT, _$$_IMPORT_ALL, module, exports, _dependencyMap) {
  var _react = _$$_REQUIRE(_dependencyMap[0]);   // require('react')
  var _reactNative = _$$_REQUIRE(_dependencyMap[1]); // require('react-native')

  function ProfileScreen() {
    // ... your component code, transpiled
  }
  exports.default = ProfileScreen;
}, 42, [0, 1]); // module ID 42, depends on modules 0 and 1
```

**2. Limited tree shaking.** Metro has experimental tree shaking support since RN 0.73+ (enabled via `experimentalImportSupport` in metro.config.js), but it's not as mature as web bundler tree shaking. For production apps, still avoid barrel files and use direct imports. Barrel files (`index.ts` that re-exports everything) are especially problematic in React Native — they pull in every module they re-export.

**3. Lazy bundling in development.** In dev mode, Metro doesn't bundle your entire app upfront. It starts the server, and when the app requests the bundle, Metro resolves and transforms only the files needed for the entry point. As you navigate to new screens, Metro transforms those files on-demand. This is why `npx expo start` is fast even for large apps.

**4. Platform-specific resolution.** Metro resolves platform extensions automatically:

```
import { Button } from './Button';

Metro resolution order:
  1. ./Button.ios.tsx     (on iOS)
  2. ./Button.native.tsx  (on any native platform)
  3. ./Button.tsx          (universal)

  1. ./Button.android.tsx  (on Android)
  2. ./Button.native.tsx   (on any native platform)
  3. ./Button.tsx           (universal)
```

**Metro configuration** (`metro.config.js`):

```js
const { getDefaultConfig } = require('expo/metro-config');

const config = getDefaultConfig(__dirname);

// Add custom file extensions
config.resolver.sourceExts = ['js', 'jsx', 'ts', 'tsx', 'json', 'cjs', 'mjs'];

// Add asset extensions
config.resolver.assetExts = [...config.resolver.assetExts, 'db', 'sqlite'];

// Custom module resolution
config.resolver.extraNodeModules = {
  '@': path.resolve(__dirname, 'src'),
};

// Enable package exports resolution
config.resolver.unstable_enablePackageExports = true;

module.exports = config;
```

**Metro + Hermes: the full mobile pipeline.** Metro outputs a JavaScript bundle. For production builds, this bundle is then compiled to Hermes bytecode (`.hbc`). We'll trace this full pipeline in Section 11.

### 5.3 Webpack — The Veteran

Webpack is the bundler that won the bundler wars of 2015-2020. It's feature-rich, battle-tested, and still runs a large percentage of production web applications. Understanding Webpack is important because (a) you'll encounter it in enterprise codebases, and (b) the concepts it popularized — loaders, plugins, code splitting, HMR — are the vocabulary the entire ecosystem uses.

**Core concepts:**

**Entry** — The starting point(s) of your application:

```js
// webpack.config.js
module.exports = {
  entry: {
    main: './src/index.tsx',
    admin: './src/admin/index.tsx', // multiple entry points → multiple bundles
  },
};
```

**Loaders** — Transforms applied to files as they're imported. Webpack only understands JavaScript natively; loaders teach it to handle TypeScript, CSS, images, fonts, and anything else:

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'swc-loader',  // or 'babel-loader' or 'ts-loader'
        exclude: /node_modules/,
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader', 'postcss-loader'],
        // Loaders execute right-to-left:
        // postcss-loader: process Tailwind/autoprefixer
        // css-loader: resolve @import and url()
        // style-loader: inject CSS into DOM via <style> tags
      },
      {
        test: /\.(png|jpg|gif|svg)$/,
        type: 'asset',  // Webpack 5 asset modules
      },
    ],
  },
};
```

**Plugins** — Hook into Webpack's build lifecycle for broader transformations:

```js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  plugins: [
    new HtmlWebpackPlugin({ template: './src/index.html' }),
    new MiniCssExtractPlugin(),
    process.env.ANALYZE && new BundleAnalyzerPlugin(),
  ].filter(Boolean),
};
```

**Output** — Where and how the bundles are written:

```js
module.exports = {
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].[contenthash].js',  // content hash for cache busting
    chunkFilename: '[name].[contenthash].chunk.js',
    publicPath: '/',
    clean: true,  // clean dist/ before each build
  },
};
```

**Why Webpack is slow:**

Webpack is written in JavaScript. It builds the module graph in JavaScript. It runs loaders in JavaScript. On a large project with thousands of modules, the CPU-bound work of parsing, transforming, and optimizing all happens in a single-threaded JavaScript process. Webpack tried to address this with caching (persistent cache in Webpack 5) and parallelism (thread-loader), but the fundamental limitation remains: it's JavaScript doing CPU-intensive work.

**When you'll still encounter Webpack:**
- Existing Next.js projects before the Turbopack migration
- Create React App (CRA) projects — though CRA is effectively abandoned
- Enterprise codebases with complex custom configurations
- Projects using Webpack-specific features like Module Federation

### 5.4 Vite — Development Speed, Production Quality

Vite (French for "fast") was created by Evan You (also the creator of Vue.js) and represents a fundamental rethinking of the development server.

**Vite's key insight: don't bundle during development.**

Traditional bundlers (Webpack) process your entire application into bundles before the dev server can show anything. Vite takes a different approach:

```
Webpack dev server:
  Start → Bundle ENTIRE app → Serve → Ready (10-30 seconds)
  File change → Re-bundle affected modules → HMR update (1-5 seconds)

Vite dev server:
  Start → Ready IMMEDIATELY (sub-second)
  Browser requests /src/App.tsx → Transform that ONE file → Serve it
  File change → Transform that ONE file → HMR update (< 50ms)
```

**How this works:** Modern browsers support ES modules natively. Vite serves your source files as individual ES modules, using `<script type="module">` tags. When the browser encounters an `import`, it sends a request to the Vite dev server, which transforms that single file (using esbuild) and returns it. The browser's native module system handles the rest.

```html
<!-- Vite injects this into your index.html -->
<script type="module" src="/src/main.tsx"></script>

<!-- Browser sees the import in main.tsx and requests: -->
<!-- GET /src/App.tsx -->
<!-- GET /node_modules/.vite/deps/react.js (pre-bundled by esbuild) -->
<!-- GET /src/components/Button.tsx -->
<!-- ... each import becomes an HTTP request -->
```

**Dependency pre-bundling:** There's a catch — node_modules packages often have hundreds of internal files. If Vite served every file individually, a single `import React from 'react'` would cause hundreds of cascading HTTP requests (React has many internal modules). So Vite uses esbuild to pre-bundle dependencies into single files on first start:

```
First run:
  Vite scans your imports → Finds react, react-dom, @tanstack/react-query, etc.
  → esbuild bundles each dependency into a single file
  → Cached in node_modules/.vite/

Subsequent runs:
  Dependencies are already cached → Instant start

Your source files:
  Never pre-bundled → Served individually, transformed on-demand
```

**Production builds:** For production, Vite switches from esbuild to **Rollup**. Why? Rollup produces better optimized output — it has superior tree shaking, more flexible code splitting, and a mature plugin ecosystem for production optimizations. The tradeoff is speed (Rollup is slower than esbuild), but for production builds, output quality matters more than build speed.

**Vite configuration** (`vite.config.ts`):

```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc'; // Uses SWC for React transforms
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [
    react(),
    tsconfigPaths(), // Resolve TypeScript path aliases
  ],
  build: {
    target: 'es2022',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          query: ['@tanstack/react-query'],
        },
      },
    },
  },
  server: {
    port: 3000,
    hmr: {
      overlay: true,  // Show errors as an overlay in the browser
    },
  },
});
```

**When to use Vite:** Vite is the default choice for new web projects in 2026 that aren't using Next.js. It's used by SvelteKit, Nuxt, Remix (optionally), Astro, and countless standalone React apps. If you're building a web app and you don't need Next.js's server features, Vite is where you start.

### 5.5 Turbopack — Next.js's Rust-Powered Bundler

Turbopack is a Rust-based bundler built by Vercel specifically for Next.js. It was announced in October 2022 and became the default development bundler for Next.js 15+.

**The core innovation: incremental computation.**

Turbopack is built on top of **Turbo Engine**, an incremental computation framework. The idea is borrowed from build systems like Bazel and Buck: every operation (parsing a file, resolving an import, transforming JSX) is a **function** with defined inputs. The engine caches the output of every function. When an input changes, only the functions whose inputs actually changed are re-executed.

```
Traditional bundler (Webpack):
  File changes → Re-run affected loaders → Re-build affected chunks
  (must re-process every file in the affected chunk)

Turbopack:
  File changes → Check which functions have invalidated inputs
  → Re-run ONLY those functions → Patch the output
  (granular, function-level caching)
```

**Why this matters for HMR:** When you change a single file, Turbopack doesn't need to re-process the entire module graph or even the entire chunk. It re-processes exactly the changed file and any dependent computations, then patches the running application. This is why Turbopack's HMR scales: the update time is proportional to the size of the change, not the size of the application.

**Turbopack uses SWC internally** for JavaScript/TypeScript transformation. Since both are written in Rust, there's no serialization overhead between the bundler and the transpiler — the AST stays in Rust memory throughout the pipeline.

**Using Turbopack:**

```bash
# Turbopack is enabled by default in Next.js 15+
next dev             # uses Turbopack for development
next build           # uses Turbopack for production (Next.js 15.3+)

# Explicitly opt in or out
next dev --turbopack     # explicitly enable
next dev --no-turbopack  # fall back to Webpack
```

**Turbopack configuration** (through `next.config.ts`):

```ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  // Turbopack-specific configuration
  turbopack: {
    rules: {
      '*.svg': {
        loaders: ['@svgr/webpack'],
        as: '*.js',
      },
    },
    resolveAlias: {
      // Custom module resolution
      'underscore': 'lodash',
    },
    resolveExtensions: ['.tsx', '.ts', '.jsx', '.js', '.mjs', '.json'],
  },
};

export default nextConfig;
```

**Current state (2026):** Turbopack handles both development and production builds for Next.js. It supports the vast majority of Webpack loaders through a compatibility layer, though some edge cases still require the Webpack fallback.

### 5.6 Rspack — Webpack-Compatible, Rust-Fast

Rspack (pronounced "are-es-pack") is a Webpack-compatible bundler written in Rust by ByteDance. It's designed as a **drop-in replacement for Webpack** — same configuration format, same loader and plugin APIs — but with native Rust performance.

**Why Rspack exists:** Many companies have large Webpack codebases with complex, customized configurations. Rewriting those configurations for Vite or Turbopack would be a massive investment. Rspack lets them keep their existing configs and get 5-10x faster builds by simply swapping the bundler.

```js
// rspack.config.js — intentionally similar to webpack.config.js
const { HtmlRspackPlugin } = require('@rspack/core');

module.exports = {
  entry: './src/index.tsx',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: {
          loader: 'builtin:swc-loader',  // Uses SWC, built into Rspack
          options: {
            jsc: {
              parser: { syntax: 'typescript', tsx: true },
              transform: { react: { runtime: 'automatic' } },
            },
          },
        },
      },
    ],
  },
  plugins: [new HtmlRspackPlugin({ template: './index.html' })],
  optimization: {
    minimize: true,
  },
};
```

**Key features:**
- **Webpack loader compatibility.** Most Webpack loaders work unchanged with Rspack.
- **Module Federation.** Rspack supports Webpack's Module Federation for micro-frontend architectures, which Vite does not natively support.
- **Built-in SWC.** Rspack uses SWC for transpilation without requiring a separate loader, which is faster than Webpack's `swc-loader` because there's no JavaScript-to-Rust serialization boundary.
- **Built-in CSS processing.** Rspack handles CSS natively (including CSS Modules) without needing `css-loader` and `style-loader`.

**Rspack vs Vite — when to pick which:**

```
Choose Rspack when:
  - You have an existing Webpack config you don't want to rewrite
  - You need Module Federation
  - You need maximum Webpack loader/plugin compatibility
  - You're migrating incrementally from Webpack

Choose Vite when:
  - Starting a new project
  - You want the native ESM dev server experience
  - You don't need Webpack-specific features
  - You're using a framework that defaults to Vite (SvelteKit, Nuxt, Astro)
```

### 5.7 Bun — The All-in-One

Bun is not just a bundler. It's a JavaScript runtime (replacing Node.js), a package manager (replacing npm/yarn/pnpm), a test runner (replacing Jest/Vitest), and a bundler (replacing Webpack/Vite). Built by Jarred Sumner, written in Zig and C++, using JavaScriptCore (not V8).

**Bun as a bundler:**

```bash
# Bundle a web application
bun build src/index.tsx --outdir ./dist --target browser --minify --sourcemap=external

# Bundle a Node.js application (for deployment)
bun build src/server.ts --outdir ./dist --target node --minify

# Bundle with code splitting
bun build src/index.tsx --outdir ./dist --splitting --format esm
```

**Bun.build() API (programmatic bundling):**

```ts
const result = await Bun.build({
  entrypoints: ['./src/index.tsx'],
  outdir: './dist',
  target: 'browser',
  format: 'esm',
  splitting: true,
  sourcemap: 'external',
  minify: {
    whitespace: true,
    identifiers: true,
    syntax: true,
  },
  define: {
    'process.env.NODE_ENV': JSON.stringify('production'),
  },
  plugins: [
    // Bun supports plugins similar to esbuild
  ],
});

if (!result.success) {
  console.error('Build failed:', result.logs);
}
```

**Bun as a package manager:**

```bash
# Install dependencies (5-25x faster than npm)
bun install

# Add a package
bun add @tanstack/react-query

# Add a dev dependency
bun add -d vitest

# Run a script from package.json
bun run build

# Execute a TypeScript file directly (no build step)
bun run src/script.ts
```

Bun's package manager is fast because:
- It uses a global cache (packages are downloaded once, hardlinked to projects)
- It resolves dependencies in parallel
- The installer is written in Zig, not JavaScript
- It uses a binary lockfile (`bun.lockb`) that's faster to read/write than text-based lockfiles

**Where Bun fits in 2026:** Bun has matured significantly and is production-ready for many use cases. However, the ecosystem is large, and some npm packages have subtle Node.js-specific behaviors that Bun handles differently. We'll cover Bun as a runtime in detail in Section 12.

### 5.8 Bundler Comparison Summary

```
┌────────────┬──────────┬──────────────┬───────────────┬──────────────────┐
│            │ Language │  Dev Server  │  Prod Build   │  Primary Use     │
├────────────┼──────────┼──────────────┼───────────────┼──────────────────┤
│ Metro      │ JS       │ Lazy bundle  │ Full bundle   │ React Native     │
│ Webpack    │ JS       │ Full bundle  │ Full bundle   │ Legacy web apps  │
│ Vite       │ JS       │ Native ESM   │ Rollup        │ New web apps     │
│ Turbopack  │ Rust     │ Incremental  │ Incremental   │ Next.js          │
│ Rspack     │ Rust     │ Full bundle  │ Full bundle   │ Webpack migration│
│ Bun        │ Zig/C++  │ N/A*         │ Full bundle   │ All-in-one       │
└────────────┴──────────┴──────────────┴───────────────┴──────────────────┘

* Bun has a dev server but it's less mature than Vite's; most Bun users
  pair it with Vite or Next.js for development
```

---

## 6. TREE SHAKING — DEAD CODE ELIMINATION

Tree shaking is the process of eliminating code that is never used. The name comes from the metaphor of shaking a tree — the dead leaves fall off. In bundler terms, the "tree" is your module graph, and the "dead leaves" are exports that nobody imports.

### 6.1 How Tree Shaking Works

Tree shaking requires **ES module static imports.** Because `import` and `export` are static (known at build time), the bundler can build a complete graph of which exports are imported by which modules.

```ts
// math.ts
export function add(a: number, b: number) { return a + b; }
export function subtract(a: number, b: number) { return a - b; }
export function multiply(a: number, b: number) { return a * b; }
export function divide(a: number, b: number) { return a / b; }

// app.ts
import { add } from './math';
console.log(add(2, 3));
```

The bundler sees that `app.ts` only imports `add` from `math.ts`. The functions `subtract`, `multiply`, and `divide` are never imported by anything. The bundler marks them as "unused" and removes them from the output.

```
Before tree shaking:      After tree shaking:
─────────────────         ─────────────────
add()        ← used       add()       ← kept
subtract()   ← unused     (removed)
multiply()   ← unused     (removed)
divide()     ← unused     (removed)
```

### 6.2 Why CommonJS Defeats Tree Shaking

Tree shaking cannot work with CommonJS because `require()` is dynamic:

```js
// With CJS, the bundler can't know what's used at build time
const math = require('./math');
// Does the app use math.add? math.multiply? All of them? None?
// The bundler can't tell until runtime.

// Even worse:
const fn = condition ? 'add' : 'subtract';
math[fn](2, 3);  // which export is accessed? depends on runtime.
```

Because the bundler cannot statically determine which properties of `module.exports` are accessed, it must include the *entire* module. This is why converting CommonJS dependencies to ESM equivalents can dramatically reduce bundle size.

### 6.3 Side Effects and the "sideEffects" Field

A **side effect** is any operation that affects something outside the module's exports — writing to the DOM, modifying a global variable, registering an event listener, making an HTTP request.

```ts
// This module has side effects — it modifies the global scope
Array.prototype.customMethod = function() { /* ... */ };
export const helper = () => {};

// This module is pure — no side effects
export const add = (a: number, b: number) => a + b;
export const multiply = (a: number, b: number) => a * b;
```

**Why side effects matter for tree shaking:** If a module has side effects, the bundler cannot safely remove it, even if none of its exports are imported. The side effect (modifying `Array.prototype`) must still execute.

The `"sideEffects"` field in `package.json` tells the bundler which modules are safe to eliminate:

```json
{
  "sideEffects": false
}
```

This says: "Every module in this package is pure. If you don't use any of its exports, you can remove the entire module." This is one of the single most impactful optimizations you can make for a library.

**Example impact:**

```
lodash (CJS, no sideEffects):
  import { debounce } from 'lodash';
  Bundle includes: entire lodash library (~70KB minified)

lodash-es (ESM, sideEffects: false):
  import { debounce } from 'lodash-es';
  Bundle includes: only debounce + its internal deps (~2KB minified)
```

### 6.4 Barrel Files — The Tree Shaking Killer

A barrel file is an `index.ts` that re-exports from multiple other files:

```ts
// src/components/index.ts (barrel file)
export { Button } from './Button';
export { Input } from './Input';
export { Modal } from './Modal';
export { DatePicker } from './DatePicker';
export { Chart } from './Chart';        // imports chart.js (200KB)
export { DataGrid } from './DataGrid';  // imports ag-grid (500KB)
```

When you import from the barrel:

```ts
import { Button } from '@/components';
```

**What you expect:** Only `Button` code is included.

**What often happens:** The bundler imports the barrel, which imports all six components, which imports all their dependencies. Even with tree shaking, if any of those modules have side effects (or the bundler can't prove they don't), you pull in everything.

**The problem is especially severe with Metro** (React Native), which does not tree-shake at all. Importing from a barrel file in a React Native project pulls in every single re-exported module and all their dependencies.

**The fix:** Import directly from the source file:

```ts
// Instead of:
import { Button } from '@/components';

// Do this:
import { Button } from '@/components/Button';
```

Or structure your `exports` in `package.json` to expose direct subpaths:

```json
{
  "exports": {
    "./Button": "./dist/Button.js",
    "./Input": "./dist/Input.js"
  }
}
```

> **Real-world impact:** A team at a major fintech company discovered their React Native app was 4MB larger than expected because a single barrel file in their shared component library was pulling in a charting library and a date picker library on every screen — even screens that didn't use charts or date pickers. Removing the barrel file and using direct imports reduced their bundle by 3.2MB.

---

## 7. CODE SPLITTING — LOADING ONLY WHAT YOU NEED

Code splitting is the strategy of breaking your application into smaller chunks that are loaded on demand rather than all at once. The goal: reduce the initial bundle size so the app loads faster, then load additional code as the user navigates.

### 7.1 Dynamic import() — The Split Point

The `import()` function (dynamic import) is the mechanism that creates split points. Unlike static `import` declarations (which are resolved at build time), `import()` returns a Promise and is evaluated at runtime:

```ts
// Static import — included in the main bundle
import { heavyFunction } from './heavy-module';

// Dynamic import — creates a separate chunk loaded on demand
const { heavyFunction } = await import('./heavy-module');
```

When a bundler encounters `import()`, it creates a new chunk for the imported module and its dependencies. The chunk is loaded at runtime when the `import()` call executes.

### 7.2 Route-Based Splitting with React.lazy()

The most common code splitting pattern is splitting by route. Each route gets its own chunk:

```tsx
import { lazy, Suspense } from 'react';

// Each lazy() call creates a split point
const HomeScreen = lazy(() => import('./screens/HomeScreen'));
const ProfileScreen = lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = lazy(() => import('./screens/SettingsScreen'));
const AnalyticsScreen = lazy(() => import('./screens/AnalyticsScreen'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Router>
        <Route path="/" element={<HomeScreen />} />
        <Route path="/profile" element={<ProfileScreen />} />
        <Route path="/settings" element={<SettingsScreen />} />
        <Route path="/analytics" element={<AnalyticsScreen />} />
      </Router>
    </Suspense>
  );
}
```

**The bundle structure looks like this:**

```
dist/
  main-a1b2c3.js           ← Router, Suspense, shared deps (always loaded)
  home-d4e5f6.js            ← HomeScreen + its deps (loaded on /)
  profile-789abc.js         ← ProfileScreen + its deps (loaded on /profile)
  settings-def012.js        ← SettingsScreen + its deps (loaded on /settings)
  analytics-345678.js       ← AnalyticsScreen + its deps (loaded on /analytics)
  vendor-9abcde.js          ← react, react-dom (shared, loaded once)
```

The user lands on `/` and downloads `main.js` + `home.js` + `vendor.js`. When they navigate to `/profile`, only `profile.js` is downloaded. The analytics screen (which might pull in a heavy charting library) is never loaded unless the user actually visits it.

### 7.3 Code Splitting for React Native

Code splitting in React Native works differently than on the web because there's no network to "fetch" additional chunks over — the entire bundle ships with the app binary (or via OTA update).

**However, code splitting still matters for React Native:**

1. **Lazy loading with Metro.** Metro supports `import()` for inline `require()` at runtime. The code is still in the bundle, but parsing and execution are deferred:

```tsx
// React Native lazy screen loading
const AnalyticsScreen = lazy(() => import('./screens/AnalyticsScreen'));
```

Metro includes `AnalyticsScreen` in the bundle, but the module is not parsed and executed until the user navigates to that screen. On low-end devices, deferred parsing can noticeably improve startup time.

2. **Re.Pack for true split bundles.** Re.Pack (a Webpack-based alternative to Metro for React Native) supports true code splitting where chunks are loaded at runtime. This is useful for super-apps with dynamically loaded mini-apps.

3. **RAM bundles (indexed).** Metro can produce "RAM bundles" where modules are stored in an indexed format and loaded individually on demand from disk, rather than parsing the entire bundle at startup.

### 7.4 Strategic Splitting

Good code splitting is about knowing *where* to split:

```
Split here (high impact):
  ✓ Routes — each route is a natural split point
  ✓ Modals — heavy dialogs loaded on demand
  ✓ Below-the-fold content — content not visible on initial load
  ✓ Admin/debug panels — rarely accessed features
  ✓ Heavy libraries — charting, rich text editors, PDF viewers

Don't split here (over-splitting):
  ✗ Every component — creates too many tiny chunks, adds overhead
  ✗ Shared utilities — they'd be duplicated across chunks
  ✗ Frequently accessed routes — the loading delay hurts UX
```

**Prefetching:** On the web, you can prefetch chunks that the user is likely to need next:

```tsx
// Prefetch on hover (mouse intent)
<Link
  to="/settings"
  onMouseEnter={() => import('./screens/SettingsScreen')}
>
  Settings
</Link>

// Or use a router that does this automatically
// Next.js prefetches <Link> targets when they enter the viewport
```

---

## 8. MINIFICATION AND COMPRESSION

After bundling, your JavaScript is correct but bloated. Variable names are descriptive, comments explain the code, whitespace makes it readable. None of that is needed at runtime. Minification and compression strip it all away.

### 8.1 What Minification Does

Minification applies several transformations to reduce code size without changing behavior:

**1. Whitespace removal:**
```js
// Before:
function calculateTotal(items) {
  let total = 0;
  for (const item of items) {
    total += item.price * item.quantity;
  }
  return total;
}

// After:
function calculateTotal(items){let total=0;for(const item of items)total+=item.price*item.quantity;return total}
```

**2. Identifier mangling (renaming):**
```js
// Before:
function calculateTotal(items) {
  let total = 0;
  for (const item of items) {
    total += item.price * item.quantity;
  }
  return total;
}

// After:
function a(b){let c=0;for(const d of b)c+=d.price*d.quantity;return c}
```

Note: `price` and `quantity` are not mangled because they're property accesses — the minifier can't safely rename them (they might come from external data).

**3. Dead code elimination:**
```js
// Before:
if (process.env.NODE_ENV === 'development') {
  console.log('Debug info:', state);
}

// After (production build, process.env.NODE_ENV replaced with "production"):
// (entire block removed — "production" === "development" is always false)
```

**4. Syntax optimizations:**
```js
// Before:
const isActive = status === 'active' ? true : false;
if (isActive === true) { doSomething(); }

// After:
"active"===status&&doSomething();
```

### 8.2 Minification Tools

**Terser** — The standard JavaScript minifier. Fork of UglifyJS with ES6+ support. Written in JavaScript. Produces the best output quality but is the slowest option:

```bash
npx terser dist/app.js --compress --mangle --output dist/app.min.js
```

**esbuild minify** — The fastest minifier. Written in Go. Produces slightly larger output than Terser (1-3% larger) but is 10-100x faster:

```bash
npx esbuild dist/app.js --minify --outfile=dist/app.min.js
```

**SWC minify** — Rust-based, similar speed to esbuild. Used by Next.js by default:

```js
// next.config.ts
export default {
  swcMinify: true,  // Default since Next.js 13
};
```

**When to pick which:**

```
Terser:   Best compression. Use when bundle size is critical and build time isn't.
esbuild:  Best speed. Use during development or when build time matters more.
SWC:      Good balance. Use when it's already your transpiler (Next.js).
```

### 8.3 Compression: Gzip and Brotli

Minification reduces the size of the JavaScript text. Compression reduces the size of the bytes sent over the network.

```
Original:          845 KB
After minification: 287 KB  (66% reduction)
After gzip:         78 KB   (91% total reduction)
After brotli:       64 KB   (92% total reduction)
```

**Gzip** — The established standard. Supported by every browser and CDN. Compression level 1-9 (higher = smaller but slower to compress). Most servers use level 6 as a default.

**Brotli** — Google's newer compression algorithm. 15-25% smaller than gzip for JavaScript. Supported by all modern browsers. Requires HTTPS.

**How compression works in practice:**

```
Build pipeline:                      Serving:
  .tsx → transpile → bundle →          Browser: "Accept-Encoding: br, gzip"
  tree-shake → split → minify →        Server/CDN: serves .br file
  pre-compress (.gz, .br)              Browser: decompresses transparently
```

Most CDNs (Vercel, Cloudflare, AWS CloudFront) handle compression automatically. Vercel compresses all static assets with Brotli and falls back to gzip for older clients. You don't need to configure this — it just works.

For self-hosted setups, pre-compress your files at build time:

```js
// vite.config.ts with compression plugin
import viteCompression from 'vite-plugin-compression';

export default defineConfig({
  plugins: [
    viteCompression({ algorithm: 'gzip' }),
    viteCompression({ algorithm: 'brotliCompress' }),
  ],
});
```

### 8.4 The Full Optimization Chain

Here's the complete chain from source to served bytes, with approximate sizes for a typical React application:

```
Step                          Example Size    Reduction
────────────────────────────  ────────────    ─────────
TypeScript source             1,200 KB        (baseline)
After transpilation           1,100 KB        8% (types removed)
After bundling                  950 KB        21% (dedup, scope hoist)
After tree shaking              620 KB        48% (dead code gone)
After code splitting            380 KB*       68% (initial chunk only)
After minification              145 KB        88%
After Brotli compression         42 KB        96%

* Initial chunk — other chunks loaded on demand
```

---

## 9. SOURCE MAPS

Source maps are files that map compiled, minified, bundled code back to the original source files. Without them, a crash in production gives you this:

```
TypeError: Cannot read properties of undefined (reading 'name')
    at a.js:1:48293
    at c.js:1:12847
    at Object.d (c.js:1:12902)
```

With source maps, the same crash gives you this:

```
TypeError: Cannot read properties of undefined (reading 'name')
    at ProfileScreen (src/screens/ProfileScreen.tsx:24:18)
    at useUser (src/hooks/useUser.ts:31:5)
    at QueryClientProvider (node_modules/@tanstack/react-query/src/QueryClientProvider.tsx:42:12)
```

The difference between debugging for 5 minutes and debugging for 5 hours.

### 9.1 How Source Maps Work

A source map is a JSON file that contains a mapping between positions in the generated code and positions in the original source. The format is defined by the Source Map V3 specification:

```json
{
  "version": 3,
  "file": "app.min.js",
  "sources": [
    "src/screens/ProfileScreen.tsx",
    "src/hooks/useUser.ts",
    "src/components/Button.tsx"
  ],
  "sourcesContent": [
    "import React from 'react';\n...",
    "import { useQuery } from '...';\n...",
    "export function Button({ ... }) {\n..."
  ],
  "names": ["ProfileScreen", "useUser", "isLoading", "user"],
  "mappings": "AAAA,SAAS,OAAO,CAAC,KAAK,EAAE..."
}
```

**The `mappings` field** is the core. It's a Base64 VLQ-encoded string that maps every position in the generated code to a position in the original source. The encoding is compact — a mapping for a 1MB bundle might only be 200KB.

**`sourcesContent`** embeds the original source code in the source map. This means crash reporting tools can show the exact source code without needing access to your source repository.

### 9.2 Source Map Types

**External source maps** (`.map` files):
```
dist/
  app.min.js            ← served to users
  app.min.js.map        ← NOT served to users (kept on server or uploaded to Sentry)
```

The JavaScript file contains a comment pointing to the source map:
```js
//# sourceMappingURL=app.min.js.map
```

**Inline source maps** (embedded in the JS file):
```js
//# sourceMappingURL=data:application/json;base64,eyJ2ZXJzaW9uIjozLC...
```

The source map is Base64-encoded and embedded directly in the JavaScript file. This is convenient for development (one file instead of two) but **never use this in production** — it makes your bundle much larger.

**Hidden source maps** (no `sourceMappingURL` comment):
The `.map` file is generated but the JavaScript file has no reference to it. The source map is uploaded to your crash reporting tool (Sentry, Crashlytics) but is never exposed to the browser. This is the **recommended approach for production.**

### 9.3 Source Maps and Crash Reporting

This is where source maps earn their keep. Sentry, Bugsnag, Crashlytics, and other crash reporting tools use source maps to translate minified stack traces into readable ones.

**For web (Next.js / Vercel):**

```ts
// next.config.ts
import { withSentryConfig } from '@sentry/nextjs';

const nextConfig: NextConfig = {
  // Generate source maps but don't expose them to the browser
  productionBrowserSourceMaps: false,
};

export default withSentryConfig(nextConfig, {
  // Upload source maps to Sentry during build
  org: 'acme',
  project: 'web-app',
  authToken: process.env.SENTRY_AUTH_TOKEN,
  silent: true,
  hideSourceMaps: true, // remove sourceMappingURL comments
});
```

**For React Native (Expo + Sentry):**

```ts
// app.config.ts
export default {
  plugins: [
    [
      '@sentry/react-native/expo',
      {
        organization: 'acme',
        project: 'mobile-app',
      },
    ],
  ],
};
```

Sentry's Expo plugin automatically:
1. Generates source maps during EAS Build
2. Uploads them to Sentry with the correct release/dist identifiers
3. Associates them with the Hermes bytecode bundle
4. Strips the `sourceMappingURL` comment from the production bundle

**Hermes complicates source maps.** React Native with Hermes has two compilation steps: JS → Hermes bytecode. The source map must map from Hermes bytecode positions all the way back to your original TypeScript source. This requires composing two source maps (TSX→JS and JS→HBC) into one. Metro and the Hermes compiler handle this composition, but misconfiguration can result in incorrect line numbers in crash reports.

### 9.4 Security: Don't Expose Source Maps in Production

Source maps contain your original source code — variable names, comments, file structure, business logic. Exposing them in production means anyone can view your source code by opening the browser's devtools Sources panel.

**The checklist:**
- Do not serve `.map` files from your CDN or static file server
- Remove `sourceMappingURL` comments from production bundles (or use hidden source maps)
- Upload source maps to your crash reporting tool's servers, not your own
- Verify by opening your production site's devtools Sources — you should see minified code, not your original source

Vercel and most modern deployment platforms handle this correctly by default — they generate source maps for Sentry but don't serve them to browsers.

---

## 10. HOT MODULE REPLACEMENT (HMR)

Hot Module Replacement is the mechanism that lets you edit code and see the result instantly in your running application without losing state. You change a component's style, and the color updates in the browser — your form inputs still have text in them, your modal is still open, your scroll position is preserved.

HMR is what makes the modern development experience feel like magic. And understanding how it works explains why it sometimes breaks.

### 10.1 How HMR Works (General Model)

```
1. You save a file
        │
        ▼
2. File watcher detects the change
   (chokidar, fs.watch, or native OS events)
        │
        ▼
3. Bundler/dev server re-transforms ONLY the changed file
   (SWC/esbuild transpiles one file — milliseconds)
        │
        ▼
4. Dev server determines what's affected
   (walks the module graph backwards from the changed file)
        │
        ▼
5. Server sends an update to the client
   (WebSocket message with the new module code)
        │
        ▼
6. Client runtime replaces the old module with the new one
   (without reloading the page)
        │
        ▼
7. If the module is a React component, React re-renders it
   (Fast Refresh preserves state for function components)
```

### 10.2 Vite's Native ESM HMR

Vite's HMR is fast because of its native ESM dev server architecture. When a file changes:

1. Vite re-transforms only that file (using esbuild — microseconds)
2. Vite checks the module graph: which modules import this one?
3. If the changed module or any ancestor accepts HMR updates, Vite sends only the changed module to the browser
4. The browser re-evaluates the module and React's Fast Refresh re-renders the affected components

**Why Vite's HMR scales:** The update time is proportional to the size of the changed file, not the size of the application. Whether your app has 100 files or 10,000 files, updating a single component takes the same amount of time (typically under 50ms).

Vite's HMR API (for library authors):

```ts
// Custom HMR handling in a module
if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // Handle the update
    // Called when THIS module is replaced
  });

  import.meta.hot.dispose(() => {
    // Cleanup before the old module is replaced
    // Remove event listeners, clear timers, etc.
  });
}
```

### 10.3 Metro's Fast Refresh (React Native)

Fast Refresh is React's HMR implementation for function components. It was built by Dan Abramov and is the default in React Native via Metro.

**How Fast Refresh works:**

1. Metro detects file change
2. Metro re-transforms the changed file (Babel or SWC)
3. Metro sends the updated module to the running app via WebSocket
4. The Metro client runtime replaces the module
5. React re-renders the affected component tree, **preserving state for function components**

**The state preservation rule:** Fast Refresh preserves `useState` and `useRef` values when you edit a function component, **as long as the component's signature doesn't change.** If you add, remove, or reorder hooks, Fast Refresh performs a full remount (losing state) because React's hook rules require a stable hook call order.

```tsx
// Edit this component — change the color, add text — state is preserved:
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <View>
      <Text style={{ color: 'blue' }}>{count}</Text>  {/* change 'blue' to 'red' → state preserved */}
      <Button onPress={() => setCount(c => c + 1)} title="Increment" />
    </View>
  );
}

// Add a new hook — state is RESET (full remount):
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');  // new hook → full remount
  // ...
}
```

**What triggers a full reload (not just Fast Refresh):**
- Editing a file that's not a React component (utility functions, config files)
- Syntax errors (the error overlay appears; fixing it restores the app)
- Changing a file that's imported before React initializes

### 10.4 Turbopack's Incremental HMR

Turbopack's HMR leverages its incremental computation engine. When a file changes:

1. The file watcher triggers invalidation of the file's "read" function
2. The Turbo Engine propagates invalidation to all dependent functions
3. Only the functions with invalidated inputs re-execute
4. The minimal delta is sent to the browser

Because Turbopack's computation graph is more granular than traditional bundlers (it tracks individual functions and expressions, not just files), its HMR updates are smaller and faster. On large Next.js applications, Turbopack's HMR is consistently under 50ms regardless of codebase size.

### 10.5 Why HMR Sometimes Breaks

HMR is not perfect. It can fail in several ways:

**1. Module state:** If a module maintains state outside of React (a plain variable, a Map, a singleton), HMR replaces the module code but the stale state persists:

```ts
// This module has state that HMR can't track
let connectionPool = new Map();  // persists across HMR updates
export function getConnection(id: string) {
  if (!connectionPool.has(id)) {
    connectionPool.set(id, createConnection(id));
  }
  return connectionPool.get(id);
}
```

After HMR, `createConnection` might be updated, but `connectionPool` still has old connections. The fix: use `import.meta.hot.dispose()` to clean up, or structure code so state lives in React.

**2. Side effects on import:** If a module performs side effects when imported (registers a global, starts a listener), HMR re-executes those side effects:

```ts
// This runs every time the module is HMR'd
window.addEventListener('resize', handleResize);  // adds a NEW listener each time
```

After several HMR updates, you have dozens of resize listeners. The fix: guard side effects or clean them up in `dispose()`.

**3. Circular dependencies:** HMR follows the module graph. Circular dependencies can confuse the update propagation, leading to stale references or incomplete updates.

**4. Non-React files in the update chain:** Fast Refresh only preserves state for React function components. If a changed file exports a plain function that's imported by a component, the component will remount (lose state) because Fast Refresh can't determine that the component itself is safe to preserve.

**The pragmatic approach:** When HMR feels broken — stale data, weird visual glitches, state that doesn't match what you expect — do a full reload (Cmd+R in the browser, Cmd+R in the React Native simulator). Then investigate whether your module has side effects or non-React state that confuses HMR.

---

## 11. THE FULL PIPELINE — END TO END

Now let's trace two real files through the complete pipeline: one for React Native (mobile) and one for Next.js (web). This is where everything in this chapter comes together.

### 11.1 React Native Pipeline: .tsx to App Binary

You have a file `ProfileScreen.tsx` in a React Native app using Expo.

```
Source: ProfileScreen.tsx (TypeScript + JSX)
Target: Hermes bytecode in an iOS/Android app binary
```

**Step 1: Type Checking (tsc --noEmit)**

TypeScript checks your code for type errors. This runs in your editor (via the TypeScript language server) and in CI (via `tsc --noEmit`). It does not produce any output files.

```bash
npx tsc --noEmit
# "ProfileScreen.tsx(24,18): error TS2339: Property 'name' does not exist on type 'undefined'."
```

**Step 2: Transpilation (Babel/SWC via Metro)**

Metro's transformer (Babel by default, SWC with `@expo/metro-config` in newer Expo versions) converts your TypeScript + JSX to plain JavaScript:

```
Input:                                    Output:
─────                                     ──────
import type { User } from '../types';     (removed — type-only import)
const name: string = user?.name ?? '';    const name = user?.name ?? '';
<Text style={styles.name}>{name}</Text>   _jsx(Text, {style: styles.name,
                                                      children: name})
```

**Step 3: Bundling (Metro)**

Metro resolves every import, assigns numeric module IDs, and concatenates everything into a single JavaScript bundle:

```
Resolution:
  ProfileScreen.tsx
    → react (module ID 0)
    → react-native (module ID 1)
    → ./useUser (module ID 42)
      → @tanstack/react-query (module ID 87)
      → ./api (module ID 93)

Output: single .js bundle with all modules wrapped in __d() calls
```

**Step 4: Hermes Compilation (AOT Bytecode)**

During `eas build`, the JavaScript bundle is compiled to Hermes bytecode:

```
metro-bundle.js  ──▶  Hermes Compiler  ──▶  metro-bundle.hbc
(JavaScript text)     (hermesc CLI)          (Hermes bytecode, ~30% smaller)
```

This step is Hermes-specific and unique to React Native. The bytecode is what Hermes executes at runtime — no JavaScript parsing needed on the device.

**Step 5: App Binary Packaging**

The `.hbc` file is embedded in the app binary:
- **iOS:** Added to the Xcode project as a resource, included in the `.ipa` archive
- **Android:** Added to `assets/` in the APK/AAB

**Step 6: Source Map Composition**

Two source maps are generated and composed:
1. TSX → JS source map (from Metro's transformer)
2. JS → HBC source map (from Hermes compiler)

These are composed into a single TSX → HBC source map and uploaded to Sentry.

```
The complete pipeline:

ProfileScreen.tsx
       │
       ├──▶ tsc --noEmit (type check, no output)
       │
       ▼
Metro Transformer (Babel/SWC)
       │  strips types, converts JSX, resolves aliases
       ▼
Metro Resolver + Serializer
       │  resolves imports, assigns module IDs, concatenates
       ▼
Single JavaScript Bundle (index.bundle.js)
       │
       ├──▶ Source Map: .tsx → .js positions
       │
       ▼
Hermes Compiler (hermesc)
       │  compiles JS to bytecode (AOT)
       ▼
Hermes Bytecode Bundle (index.bundle.hbc)
       │
       ├──▶ Source Map: .js → .hbc positions
       │    (composed with .tsx → .js map → final .tsx → .hbc map)
       │    (uploaded to Sentry)
       │
       ▼
App Binary (.ipa / .apk)
       │
       ▼
User's Device
  Hermes loads .hbc → executes bytecode → React renders → native views appear
```

### 11.2 Next.js Pipeline: .tsx to Deployed Web App

You have a file `app/profile/page.tsx` in a Next.js app deployed to Vercel.

```
Source: page.tsx (TypeScript + JSX, React Server Component)
Target: Optimized JavaScript chunks served from Vercel's Edge Network
```

**Step 1: Type Checking (tsc --noEmit)**

Same as React Native — runs in the editor and CI. No output.

**Step 2: Transpilation (SWC via Turbopack)**

Turbopack uses SWC internally to transform your TypeScript + JSX:

```
Input:                                    Output:
─────                                     ──────
import type { User } from '@/types';      (removed)
export default async function Page() {    export default async function Page() {
  const user = await getUser();             const user = await getUser();
  return <h1>{user.name}</h1>;              return _jsx("h1", {children: user.name});
}                                         }
```

**Step 3: Bundling (Turbopack)**

Turbopack builds the dependency graph, resolves imports, and creates the module graph. Unlike Metro, Turbopack separates server code from client code:

```
Server Component Graph:                Client Component Graph:
  page.tsx (server)                      InteractiveProfile.tsx (client)
    → getUser (server)                     → react (shared)
    → InteractiveProfile (client ref)      → useState, useEffect
    → database (server-only)               → Button.tsx
```

Server components are bundled separately — they run on the server and never ship to the browser. Only client components are included in the browser bundles.

**Step 4: Tree Shaking**

Turbopack analyzes the ESM import graph and removes unused exports. Server-only code (database queries, API secrets) is completely eliminated from client bundles.

**Step 5: Code Splitting**

Turbopack automatically splits the output:
- **One chunk per route** — each `page.tsx` gets its own chunk
- **Shared chunks** — dependencies used by multiple routes are extracted into shared chunks
- **Framework chunk** — React, Next.js runtime code in a separate, cacheable chunk

**Step 6: Minification (SWC)**

SWC minifies each chunk — removing whitespace, mangling identifiers, eliminating dead code.

**Step 7: Compression and Deployment**

Vercel compresses the output with Brotli and distributes it across its Edge Network.

```
The complete pipeline:

app/profile/page.tsx
       │
       ├──▶ tsc --noEmit (type check, no output)
       │
       ▼
Turbopack (SWC transform)
       │  strips types, converts JSX
       │  separates server and client components
       ▼
Turbopack (bundle + tree shake)
       │  resolves imports, builds module graph
       │  eliminates dead code, removes server-only code from client
       ▼
Turbopack (code split)
       │  route-based chunks, shared chunks, vendor chunk
       ▼
SWC Minify
       │  mangle, compress, dead code elimination
       ▼
Output:
  .next/static/chunks/
    app/profile/page-a1b2c3.js    (route chunk, ~15KB)
    commons-d4e5f6.js              (shared deps, ~45KB)
    framework-789abc.js            (React, Next.js, ~80KB)
    [each with .js.map source maps]
       │
       ├──▶ Source maps uploaded to Sentry
       │
       ▼
Vercel Edge Network
  Brotli compressed, CDN cached, served globally
  
  profile-a1b2c3.js: 15KB → 4KB (Brotli)
  commons-d4e5f6.js: 45KB → 12KB (Brotli)
  framework-789abc.js: 80KB → 22KB (Brotli)
       │
       ▼
User's Browser
  Downloads chunks → Parses JS → React hydrates → Interactive page
```

### 11.3 Comparing the Two Pipelines

```
                        React Native (Expo)         Next.js (Vercel)
────────────────────    ───────────────────         ─────────────────
Type check              tsc --noEmit                tsc --noEmit
Transpiler              Babel or SWC (via Metro)    SWC (via Turbopack)
Bundler                 Metro                       Turbopack
Tree shaking            No                          Yes
Code splitting          Limited (lazy loading)      Automatic (routes)
Minification            Hermes bytecode compile     SWC minify
Compression             N/A (local binary)          Brotli/gzip
Source maps             Composed (JS→HBC)           Standard (TS→JS)
Final format            Hermes bytecode (.hbc)      Minified JavaScript
Delivery                App binary (App Store)      CDN (Edge Network)
```

---

## 12. BUN AS A RUNTIME

Bun deserves its own section because it challenges the assumption that you need separate tools for each step of the toolchain. Bun is a **JavaScript runtime** (like Node.js), a **package manager** (like npm/pnpm), a **bundler** (like esbuild/Webpack), a **test runner** (like Jest/Vitest), and a **TypeScript executor** (like ts-node/tsx) — all in one binary.

### 12.1 Bun vs Node.js — What's Different

**Runtime engine:** Node.js uses V8 (Google's JS engine). Bun uses JavaScriptCore (Apple's JS engine, the one that powers Safari). Both are fast; they have different performance characteristics depending on the workload.

**TypeScript execution:** Node.js (as of v22+) has experimental TypeScript stripping, but it's limited. Bun runs TypeScript natively — no compilation step needed:

```bash
# Node.js: need a loader or pre-compilation
node --experimental-strip-types src/server.ts   # limited, no JSX
npx tsx src/server.ts                            # uses esbuild under the hood

# Bun: just works
bun src/server.ts         # TypeScript, JSX, all supported natively
bun src/script.tsx        # even TSX works
```

**Package installation:**

```bash
# npm: ~30 seconds for a large project
npm install

# pnpm: ~10 seconds
pnpm install

# Bun: ~3 seconds
bun install
```

Bun achieves this speed through a global binary cache, parallel downloads, and a Zig-based resolver. The `bun.lockb` lockfile is binary (not human-readable YAML/JSON like other lockfiles), which is faster to read and write but harder to diff in code review. Bun also supports a text-based `bun.lock` format for better git diffs.

### 12.2 Running Scripts

```bash
# All equivalent:
npm run dev          # reads scripts.dev from package.json, spawns a shell
yarn dev             # same, slightly different shell behavior
pnpm dev             # same
bun run dev          # same, but bun's process spawning is faster
bun dev              # shorthand — bun allows omitting 'run'
```

Bun's `bun run` is faster than `npm run` because it doesn't spawn a shell to execute the script — it runs the command directly.

### 12.3 Bun as a Server Runtime

```ts
// server.ts — runs directly with `bun server.ts`
const server = Bun.serve({
  port: 3000,
  fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === '/api/users') {
      return Response.json([
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' },
      ]);
    }
    return new Response('Not Found', { status: 404 });
  },
});

console.log(`Server running at http://localhost:${server.port}`);
```

Bun's HTTP server uses a native implementation (not Node.js's `http` module), which is significantly faster for simple request/response patterns. However, **Express and most Node.js HTTP frameworks work with Bun** because Bun implements Node.js's `http` and `net` APIs.

### 12.4 When to Use Bun

**Use Bun for:**
- **Development speed.** `bun install` + `bun run dev` is noticeably faster than npm/Node alternatives. If your team is frustrated by slow installs and slow script startup, Bun is the quickest improvement.
- **Script execution.** `bun run scripts/migrate.ts` is faster than `npx tsx scripts/migrate.ts`. For CLI tools and scripts, Bun's startup time advantage is tangible.
- **Test runner.** `bun test` is a fast, Jest-compatible test runner. For projects that don't need Vitest's Vite integration, it's a solid alternative.

**Stick with Node.js for:**
- **Production servers.** Node.js has decades of battle-testing. Its behavior in edge cases (memory pressure, high connection counts, graceful shutdown) is well-documented. Bun is catching up, but Node.js is the safer choice for production-critical services.
- **Ecosystem edge cases.** Some npm packages use Node.js-specific internals that Bun doesn't implement (native addons with node-gyp, some fs edge cases, specific stream behaviors). If your dependency tree is large and diverse, Node.js compatibility is more reliable.
- **Container deployments.** Node.js Docker images are smaller, better-documented, and more commonly used in enterprise Kubernetes setups. Bun's Docker story is improving but less mature.

**The pragmatic approach in 2026:**

```
Development:    Use Bun for installs, scripts, and test running
                bun install → bun run dev → bun test

CI/CD:          Use either — Bun works well in CI for speed
                bun install → bun run build → bun run test

Production:     Use Node.js for application servers
                Bun for build tooling, Node.js for runtime

React Native:   Bun doesn't affect RN directly (Metro handles bundling)
                Use Bun as a faster package manager: bun install
```

### 12.5 Bun + Frameworks

**Bun with Next.js:**

```bash
# Use Bun for package management, Next.js handles everything else
bun install
bun run dev    # runs next dev --turbopack (Turbopack, not Bun's bundler)
bun run build  # runs next build (Turbopack)
```

You get faster installs and script startup, but the actual bundling is still Turbopack. This is the recommended setup — let each tool do what it does best.

**Bun with Expo:**

```bash
# Use Bun as the package manager
bun install
bunx expo start   # Metro handles bundling
bunx eas build    # EAS handles the native build
```

Same principle — Bun accelerates the package management and script running, while Metro and EAS handle the React Native-specific work.

---

## 13. PUTTING IT ALL TOGETHER — DECISION FRAMEWORK

After all of this, the question is: what should you actually use? Here's the decision framework:

### For a New React Native (Expo) Project

```
Package manager:    pnpm (or Bun for speed)
Type checker:       tsc --noEmit (in CI and editor)
Transpiler:         Babel (Metro default) or SWC (@expo/metro-config)
Bundler:            Metro (the only real option for RN)
Minifier:           Hermes compiler (bytecode is inherently compact)
Source maps:        Metro + Hermes composition → Sentry
HMR:                Metro Fast Refresh

Your metro.config.js and app.config.ts are the only build configs you touch.
Expo handles the rest.
```

### For a New Next.js (Vercel) Project

```
Package manager:    pnpm (or Bun for speed)
Type checker:       tsc --noEmit (in CI and editor)
Transpiler:         SWC (via Turbopack, default)
Bundler:            Turbopack (default in Next.js 15+)
Tree shaking:       Automatic (ESM)
Code splitting:     Automatic (route-based)
Minifier:           SWC minify (default)
Source maps:        Turbopack → Sentry
HMR:                Turbopack incremental HMR

Your next.config.ts is the only build config you touch.
Vercel handles compression and CDN.
```

### For a Standalone Web App (No Framework)

```
Package manager:    pnpm (or Bun for speed)
Type checker:       tsc --noEmit
Transpiler:         SWC (via @vitejs/plugin-react-swc)
Bundler:            Vite (dev: native ESM, prod: Rollup)
Tree shaking:       Automatic (Rollup)
Code splitting:     Manual (dynamic import) + Rollup
Minifier:           esbuild (Vite default) or Terser (for max compression)
Source maps:        Vite → Sentry
HMR:                Vite native ESM HMR

Your vite.config.ts is the only build config you touch.
```

### For a Library (Published to npm)

```
Package manager:    pnpm
Type checker:       tsc --noEmit (strict)
Build tool:         tsup (wraps esbuild, handles dual CJS/ESM output)
Output:             dist/index.js (ESM) + dist/index.cjs (CJS) + dist/index.d.ts (types)
package.json:       "exports" field with "import" and "require" conditions
                    "sideEffects": false
                    "files": ["dist"]
Source maps:        Included in dist for consumer debugging
```

### For a Webpack Migration

```
Before:             Webpack + Babel + Terser
Option A:           Rspack (drop-in replacement, 5-10x faster)
Option B:           Vite (different config format, but better DX)
Option C:           Next.js migration (if you want the framework benefits)

Rspack is the lowest-risk option. Same config, faster builds.
```

---

## 14. COMMON PROBLEMS AND SOLUTIONS

### Problem: "Build is slow"

```
Diagnosis:
  1. Run your build with timing: TIMING=1 webpack or vite build --debug
  2. Identify the bottleneck: transpilation? bundling? minification?

Solutions by bottleneck:
  Transpilation slow:    Switch from Babel to SWC
  Bundling slow:         Switch from Webpack to Rspack or Turbopack
  Minification slow:     Switch from Terser to SWC minify or esbuild minify
  All slow:              Your project might just be too big for JS-based tools.
                         Switch to a Rust/Go-based toolchain.
```

### Problem: "Bundle is too large"

```
Diagnosis:
  1. Run a bundle analyzer:
     - Webpack: webpack-bundle-analyzer
     - Vite: rollup-plugin-visualizer
     - Next.js: @next/bundle-analyzer
     - React Native: npx react-native-bundle-visualizer

  2. Look for:
     - Duplicate packages (two versions of lodash, three copies of moment)
     - Barrel file imports pulling in unused code
     - Missing "sideEffects": false in library package.json
     - Full library imports instead of specific imports

Solutions:
  Replace barrel imports:     import { Button } from './Button' (not from './index')
  Replace large libraries:    date-fns instead of moment, just the lodash functions you need
  Add sideEffects: false:     to your own libraries' package.json
  Enable tree shaking:        ensure you're using ESM imports, not require()
  Code split aggressively:    lazy() for routes, dynamic import() for heavy features
```

### Problem: "HMR isn't working / state keeps resetting"

```
Diagnosis:
  Check the dev server console for messages like:
  - "[HMR] Full reload needed" — something in the update chain can't be hot-replaced
  - "Cannot apply update" — the module doesn't accept HMR

Common causes:
  1. Editing a non-component file (config, utility) that's imported at the top level
  2. Module has side effects (event listeners, global state)
  3. Hook order changed (added/removed a useState)
  4. Circular dependencies confusing the update graph

Solutions:
  1. Accept that some edits cause full reloads — it's normal
  2. Structure side effects to be cleanable (use dispose() or useEffect cleanup)
  3. Don't add/remove hooks while testing with HMR — refresh manually
  4. Break circular dependencies
```

### Problem: "Source maps show wrong line numbers"

```
Diagnosis:
  1. Check that source maps are being generated:
     - Look for .map files in your build output
     - Check your bundler config for sourcemap: true
  2. Check that source maps are being uploaded correctly to Sentry
  3. For React Native: Hermes source map composition can fail silently

Solutions:
  Sentry not matching:    Ensure release/dist versions match between the app and uploaded maps
  Hermes wrong lines:     Update @sentry/react-native — source map composition has improved
  No source maps at all:  Check that the build isn't stripping them (productionBrowserSourceMaps)
```

### Problem: "Can't import ESM package from CJS"

```
Error: require() of ES Module /path/to/module not supported

Solutions:
  1. Switch your project to ESM: add "type": "module" to package.json
  2. Use dynamic import: const mod = await import('esm-package')
  3. Use a bundler — it handles CJS/ESM interop transparently
  4. Pin to the last CJS version of the package (temporary fix)
```

---

## KEY TAKEAWAYS

1. **The build toolchain is not optional knowledge.** Understanding it is the difference between guessing at performance problems and diagnosing them.

2. **The ecosystem has converged on ESM.** Use `"type": "module"`, use `import`/`export`, configure `"exports"` in package.json. Fight for ESM in your codebase.

3. **Transpilation and type checking are separate jobs.** Use SWC or esbuild for fast transpilation. Use `tsc --noEmit` for type checking. Don't make one tool do both.

4. **Your bundler choice depends on your framework.** Next.js uses Turbopack. Expo uses Metro. Standalone web apps use Vite. Webpack migrations use Rspack. Don't fight the defaults.

5. **Tree shaking only works with ESM.** If your bundle is larger than expected, check for CommonJS imports, barrel files, and missing `"sideEffects": false`.

6. **Source maps are not optional.** Without them, your crash reports are useless. Upload them to Sentry. Don't expose them to production users.

7. **HMR is a development convenience, not a guarantee.** When it breaks, refresh. When it keeps breaking, fix the underlying module structure.

8. **The full pipeline is long: TS → type check → transpile → bundle → tree shake → split → minify → compress → serve.** Every step matters. Every step has tools. The best tools in 2026 are written in Rust and Go, not JavaScript.

---

## FURTHER READING

- [SWC documentation](https://swc.rs/docs/getting-started) — Configuration and plugin development
- [Vite documentation](https://vitejs.dev/guide/) — Architecture and configuration
- [Turbopack documentation](https://nextjs.org/docs/architecture/turbopack) — Next.js integration
- [Node.js modules documentation](https://nodejs.org/api/packages.html) — Package exports, conditional exports
- [Metro documentation](https://metrobundler.dev/) — React Native bundling
- [Bun documentation](https://bun.sh/docs) — Runtime, bundler, package manager
- [Source Map V3 specification](https://tc39.es/source-map/) — How source maps work
- [Rspack documentation](https://rspack.dev/) — Webpack-compatible Rust bundler

---

*Next chapter: [Chapter 6: EAS — Build, Submit, Update](../part-2-react-native-expo/06-eas-mastery.md) — where we take this build pipeline and deploy it to real devices through Expo Application Services.*
