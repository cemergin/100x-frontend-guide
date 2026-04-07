<!--
  CHAPTER: 20
  TITLE: Codebase Management, DX & Editor Mastery
  PART: IV — Architecture at Scale
  PREREQS: Chapters 0d, 15
  KEY_TOPICS: VS Code, editor productivity, extensions, keybindings, debugging, ESLint, Biome, Prettier, EditorConfig, linting strategy, formatting, git hooks, Husky, lint-staged, code quality automation, codebase health, dependency updates, Renovate, technical debt management
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 20: Codebase Management, DX & Editor Mastery

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 0d, 15 | Difficulty: Beginner to Intermediate

Here's a truth that takes most engineers too long to learn: **your codebase is a product, and your teammates are its users.** The features you ship to customers are built on top of the developer experience you ship to each other. A fast, well-organized, lint-clean, auto-formatted codebase with one-command setup is a force multiplier. A messy one with inconsistent formatting, no linting, manual setup steps, and mystery configurations is a tax on every feature, every review, and every onboarding.

The best frontend architects I've worked with obsess over this. They don't just build features -- they build the environment in which features get built. They configure their editors like instruments. They automate every repetitive quality check. They treat a broken lint rule with the same urgency as a broken test. And the result is teams that ship faster, with fewer bugs, with less friction, and with engineers who actually enjoy opening the codebase every morning.

This chapter is about becoming that person. We're going to cover everything from VS Code keyboard shortcuts to ESLint flat config to git hook pipelines to dependency management strategy. Some of this will feel basic. Do it anyway. The compound interest of a well-maintained codebase is extraordinary.

### In This Chapter
- The Codebase Is Your Product -- why DX determines team velocity
- VS Code Mastery -- shortcuts, settings, extensions, debugging, profiles
- Linting Strategy -- ESLint flat config, essential rules, performance
- Formatting -- Prettier vs Biome, EditorConfig, pipeline order
- Git Hooks and Automation -- Husky, lint-staged, commitlint
- Maintaining Codebases -- dependency updates, dead code, tech debt
- Git Workflow -- branches, commits, PRs, code review
- The 10-Minute Setup -- copy-paste automation for new projects

### Related Chapters
- [Ch 0d: Development Environment Setup] -- initial tooling installation
- [Ch 15: Monorepo Architecture] -- workspace structure these tools operate within
- [Ch 16: Testing Strategy] -- test automation that complements linting
- [Ch 21: CI/CD Pipeline] -- where these checks run in the cloud

---

## 1. THE CODEBASE IS YOUR PRODUCT

Let me make this concrete. You're onboarding a new engineer. They clone the repo. What happens next?

**Scenario A (bad codebase):**
1. Clone. Run `npm install`. It fails -- wrong Node version.
2. Install nvm, switch to the right version. Nobody documented which one. Try 18. Nope. Try 20. Works.
3. Run `npm start`. Missing environment variables. Ask in Slack. Wait 2 hours for someone in a different timezone to respond.
4. Got the env vars. App starts but crashes immediately. A dependency needs a local Postgres database. Not documented.
5. Get Postgres running. App starts. Open a file. No linting. No formatting. Tabs vs spaces is a religious war in the git history. Some files use single quotes, some use double quotes, some use both in the same line.
6. Make a change. Open a PR. CI fails on a lint rule that doesn't run locally. Fix it blindly. CI passes. Merge. Two days have passed.

**Scenario B (good codebase):**
1. Clone. Run `make setup` (or `pnpm install`). Everything installs, including git hooks.
2. A `.nvmrc` file auto-switches to the right Node version. A `.env.example` file documents every variable with comments. `docker compose up` starts all dependencies.
3. App starts on first try. Open a file. ESLint and Prettier are configured. Errors show inline via Error Lens. Save the file and it auto-formats.
4. Make a change. Commit. Pre-commit hooks run format, lint, and type-check on staged files in 3 seconds. Everything passes.
5. Open a PR. The PR template prompts for context. CI passes on the first run because everything that runs in CI also runs locally.
6. Thirty minutes from clone to merged PR.

**Scenario B is not magic. It's configuration.** Every single thing in that list is achievable with the tools in this chapter, in a single afternoon of setup.

### The Cost of Bad DX

Bad DX doesn't show up on any dashboard. There's no Datadog alert for "engineer spent 20 minutes fighting ESLint config." But the costs are real:

- **Context switching cost:** Every time an engineer has to stop thinking about the problem and start thinking about the tooling, you lose 10-15 minutes of productive focus.
- **Onboarding cost:** If it takes a week to get productive in your codebase instead of a day, that's a week of salary burned per new hire.
- **Review cost:** If PRs have formatting noise (whitespace changes, import reordering), reviewers waste time on irrelevant diffs and miss real issues.
- **Bug cost:** If linting catches 5% of bugs before they ship (a conservative estimate -- the real number is higher for things like exhaustive deps and rules of hooks), every skipped lint check is a bug you'll find in production instead.
- **Morale cost:** Engineers who fight their tools every day burn out faster and are more likely to leave. Engineers who work in a clean, well-maintained codebase feel pride in their work.

### The 100x Principle

A 100x engineer doesn't just write code 100 times faster. They create environments where everyone around them ships faster. The DX investments in this chapter are the highest-leverage work you can do as a frontend architect. A day spent setting up ESLint, Prettier, Husky, and lint-staged pays for itself within the first week and continues paying dividends for years.

---

## 2. VS CODE MASTERY

VS Code is the dominant editor for frontend development. According to the Stack Overflow Developer Survey, over 73% of developers use it. More importantly, for React Native and Expo development, it has the best ecosystem of extensions and debugging support.

But most developers use maybe 10% of what VS Code can do. They use it like a text editor with syntax highlighting. That's like buying a Ferrari and only driving it in first gear. Let's fix that.

### 2.1 Essential Keyboard Shortcuts

These are the shortcuts that separate the fast engineers from everyone else. Learn these until they're muscle memory.

**File Navigation:**
| Shortcut | Action | Why It Matters |
|----------|--------|----------------|
| `Cmd+P` | Quick Open (file search) | Never use the sidebar to find files. Type 2-3 characters of the filename. Fuzzy matching handles the rest. This should be your primary way of opening files. |
| `Cmd+Shift+P` | Command Palette | Access every VS Code command. Can't remember a shortcut? Search for the command here. |
| `Ctrl+Tab` | Switch between open files | Like Alt+Tab for your editor. Navigate recent files without touching the mouse. |
| `Cmd+\` | Split editor | Side-by-side editing. Put the component on the left, its test on the right. |
| `Cmd+B` | Toggle sidebar | Hide the sidebar when you need more code space. You're using `Cmd+P` to navigate anyway. |
| `Cmd+J` | Toggle terminal panel | Quick access to the integrated terminal without leaving the editor. |

**Code Editing:**
| Shortcut | Action | Why It Matters |
|----------|--------|----------------|
| `Cmd+D` | Select next occurrence | Multi-cursor editing. Select a variable name, hit `Cmd+D` to select the next occurrence, repeat. Now type to rename all of them simultaneously. Faster than find-and-replace for local edits. |
| `Cmd+Shift+L` | Select all occurrences | Like `Cmd+D` but selects every occurrence at once. |
| `Option+Up/Down` | Move line up/down | Rearrange lines without cut/paste. |
| `Option+Shift+Up/Down` | Copy line up/down | Duplicate the current line. Incredibly useful for creating similar code. |
| `Cmd+Shift+K` | Delete line | Delete the entire current line without selecting it first. |
| `Cmd+/` | Toggle line comment | Comment/uncomment the current line or selection. |
| `Cmd+.` | Quick Fix | **This is the big one.** When you see a yellow lightbulb or an error underline, hit `Cmd+.` to see auto-fixes. Add missing imports, fix ESLint errors, generate switch cases, extract to variable -- VS Code and your extensions offer dozens of automatic fixes. |
| `F2` | Rename symbol | Rename a variable/function/component across the entire project. Infinitely better than find-and-replace because it understands scope. |
| `Cmd+Shift+F` | Global search | Search across all files in the workspace. Supports regex. Use this to find usages, patterns, and dead code. |
| `Cmd+Shift+H` | Global search and replace | Find and replace across the entire project. Use with caution but extremely powerful with regex. |

**Navigation Within Code:**
| Shortcut | Action | Why It Matters |
|----------|--------|----------------|
| `Cmd+G` | Go to line | Jump directly to a line number. Essential when reading error stack traces. |
| `Cmd+Shift+O` | Go to symbol in file | Jump to any function, class, or interface in the current file. |
| `Cmd+T` | Go to symbol in workspace | Jump to any symbol across the entire project. |
| `F12` | Go to definition | Jump to where a function/variable/type is defined. |
| `Option+F12` | Peek definition | See the definition inline without leaving the current file. |
| `Shift+F12` | Find all references | See everywhere a symbol is used. Essential for understanding impact before refactoring. |
| `Cmd+Shift+\` | Jump to matching bracket | Navigate between opening and closing brackets. |
| `Ctrl+-` / `Ctrl+Shift+-` | Go back / Go forward | Navigate through your cursor history. Jump to a definition with F12, then come back with `Ctrl+-`. Use this constantly. |

**The Meta-Shortcut:** If you can only remember one thing from this section, remember `Cmd+Shift+P`. The Command Palette lets you search for any action, and it shows the keyboard shortcut next to each command. Use it to discover shortcuts organically.

### 2.2 Settings.json Optimization

VS Code's default settings are designed for the broadest possible audience. They're not optimized for React Native development. Here's the `settings.json` I recommend:

```jsonc
// User settings.json (Cmd+Shift+P → "Preferences: Open User Settings (JSON)")
{
  // -- Editor Behavior --
  "editor.formatOnSave": true,                    // Auto-format every time you save
  "editor.defaultFormatter": "esbenp.prettier-vscode", // Use Prettier as default formatter
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",            // Auto-fix ESLint issues on save
    "source.organizeImports": "never"              // Let ESLint handle import order, not TS
  },
  "editor.linkedEditing": true,                    // Rename matching HTML/JSX tags together
  "editor.bracketPairColorization.enabled": true,  // Color-code matching brackets
  "editor.guides.bracketPairs": "active",          // Highlight the active bracket pair
  "editor.suggestSelection": "first",              // Always pre-select the first suggestion
  "editor.acceptSuggestionOnCommitCharacter": false, // Don't auto-accept on . or (
  "editor.tabSize": 2,                             // 2 spaces — the standard for JS/TS
  "editor.insertSpaces": true,                     // Spaces, not tabs

  // -- Editor Appearance --
  "editor.minimap.enabled": false,                 // Turn off minimap. It wastes space.
  "editor.renderWhitespace": "boundary",           // Show whitespace at word boundaries
  "editor.fontFamily": "'Monaspace Neon', 'Fira Code', Menlo, monospace",
  "editor.fontLigatures": true,                    // => becomes ⇒, !== becomes ≢, etc.
  "editor.fontSize": 13,
  "editor.lineHeight": 1.6,                        // Slightly more breathing room
  "editor.cursorBlinking": "smooth",
  "editor.cursorSmoothCaretAnimation": "on",
  "editor.smoothScrolling": true,
  "editor.stickyScroll.enabled": true,             // Pin parent scopes at top of editor

  // -- Files --
  "files.autoSave": "onFocusChange",              // Auto-save when switching files/apps
  "files.trimTrailingWhitespace": true,            // Remove trailing whitespace on save
  "files.insertFinalNewline": true,                // Always end files with a newline
  "files.trimFinalNewlines": true,                 // Remove extra blank lines at end

  // -- Search --
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/build": true,
    "**/.expo": true,
    "**/android/build": true,
    "**/ios/Pods": true,
    "**/.next": true
  },

  // -- TypeScript --
  "typescript.preferences.importModuleSpecifier": "non-relative", // Use path aliases
  "typescript.updateImportsOnFileMove.enabled": "always",
  "typescript.suggest.autoImports": true,
  "typescript.inlayHints.parameterNames.enabled": "literals",     // Show param names for literals
  "typescript.inlayHints.functionLikeReturnTypes.enabled": true,

  // -- Terminal --
  "terminal.integrated.defaultProfile.osx": "zsh",
  "terminal.integrated.fontSize": 12,
  "terminal.integrated.scrollback": 10000,

  // -- Git --
  "git.autofetch": true,                           // Auto-fetch to show remote changes
  "git.confirmSync": false,                        // Don't prompt before push/pull
  "git.enableSmartCommit": true,                   // Commit all changes if nothing is staged

  // -- Explorer --
  "explorer.confirmDelete": false,                 // Stop asking "are you sure?" every time
  "explorer.confirmDragAndDrop": false,
  "explorer.fileNesting.enabled": true,            // Nest related files together
  "explorer.fileNesting.patterns": {
    "*.ts": "${capture}.test.ts, ${capture}.spec.ts, ${capture}.d.ts",
    "*.tsx": "${capture}.test.tsx, ${capture}.spec.tsx, ${capture}.stories.tsx",
    "package.json": "package-lock.json, pnpm-lock.yaml, yarn.lock, .npmrc, .nvmrc",
    "tsconfig.json": "tsconfig.*.json",
    ".eslintrc*": ".eslintignore, eslint.config.*",
    ".prettierrc*": ".prettierignore"
  },

  // -- Workbench --
  "workbench.startupEditor": "none",               // Don't show welcome tab
  "workbench.editor.enablePreview": false,          // Always open files in full tabs
  "workbench.tree.indent": 16                       // Wider indentation in file tree
}
```

**A few callouts:**

**Minimap off.** I know this is controversial. The minimap is a visual indicator of where you are in a file, but it takes up ~10% of your screen width and provides information you can get from `Cmd+Shift+O` (go to symbol) or the breadcrumb bar. On a 13" laptop, that screen real estate matters.

**Format on save.** This is non-negotiable. If you're manually formatting code, you're wasting keystrokes on something a computer can do perfectly. Configure it, forget about it, never think about formatting again.

**Auto-save on focus change.** You switch to the browser to test something, your file auto-saves. You switch to the terminal, your file auto-saves. Combined with format-on-save, this means your files are always saved and always formatted. No more "oh I forgot to save."

**File nesting.** This collapses related files in the explorer: `Button.tsx`, `Button.test.tsx`, and `Button.stories.tsx` nest under a single `Button.tsx` entry. Reduces visual noise dramatically in component-heavy projects.

### 2.3 Essential Extensions

Here's the extension stack I recommend for React Native + Expo + Next.js development. I'm deliberately keeping this list short. Every extension adds startup time and cognitive overhead. Only install what you'll use daily.

**Core (install these immediately):**

| Extension | Why |
|-----------|-----|
| **ESLint** (`dbaeumer.vscode-eslint`) | Inline linting errors and auto-fix on save. The foundation of code quality in the editor. |
| **Prettier** (`esbenp.prettier-vscode`) | Auto-formatting. Works with format-on-save. |
| **Error Lens** (`usernameheo.errorlens`) | Shows errors and warnings inline, right next to the code, instead of requiring you to hover. This is a game-changer -- you see issues immediately without squinting at underlines. |
| **Pretty TypeScript Errors** (`yoavbls.pretty-ts-errors`) | Rewrites TypeScript's notoriously unreadable error messages into human-friendly format. When you see a 50-line generic type error, this extension turns it into something you can actually understand. |
| **GitLens** (`eamodio.gitlens`) | Inline git blame, file history, commit graph. See who changed every line and when. Essential for understanding why code is the way it is. |
| **GitHub Copilot** (`github.copilot`) | AI autocomplete. Controversial to some, but the productivity gains are real. Use it as a typing accelerator, not a thinking replacement. |

**React Native Specific:**

| Extension | Why |
|-----------|-----|
| **React Native Tools** (`msjsdiag.vscode-react-native`) | Debugging support for React Native apps. Attach to running apps, set breakpoints, inspect variables. |
| **Expo Tools** (`expo.vscode-expo-tools`) | Autocomplete for `app.json`/`app.config.ts`, EAS config, Expo Router file-based routing hints. |
| **ES7+ React/Redux/React-Native Snippets** (`dsznajder.es7-react-js-snippets`) | Type `rnfc` and get a full React Native functional component. Type `us` and get a `useState` hook. Saves thousands of keystrokes per week. |
| **Tailwind CSS IntelliSense** (`bradlc.vscode-tailwindcss`) | If you're using NativeWind (Tailwind for React Native), this gives you autocomplete, linting, and hover previews for class names. |

**Utility:**

| Extension | Why |
|-----------|-----|
| **Import Cost** (`wix.vscode-import-cost`) | Shows the bundle size of every import inline. Useful for catching accidentally heavy imports (`import _ from 'lodash'` vs `import groupBy from 'lodash/groupBy'`). |
| **Todo Tree** (`gruntfuggly.todo-tree`) | Finds all `TODO`, `FIXME`, `HACK` comments in your codebase and shows them in a tree view. Good for tracking technical debt. |
| **Code Spell Checker** (`streetsidesoftware.code-spell-checker`) | Catches typos in variable names, comments, and strings. A surprising number of bugs come from misspelled property names. |

**Extensions I specifically do NOT recommend:**

- **Bracket Pair Colorizer** -- VS Code has this built-in now. The extension is deprecated and slower.
- **Auto Rename Tag** -- VS Code has `linkedEditing` built-in (we enabled it in settings above).
- **Path Intellisense** -- TypeScript's built-in path resolution with `tsconfig.json` paths is better.
- **Any "beautify" extension** -- Prettier handles all formatting. Multiple formatters create conflicts.

### 2.4 Workspace Settings vs User Settings

VS Code has two levels of settings:

- **User Settings** (`~/Library/Application Support/Code/User/settings.json`): Apply to every project you open. This is where you put personal preferences -- font, theme, keybindings, minimap preference.

- **Workspace Settings** (`.vscode/settings.json` in the project root): Apply only to this project. This is where you put team-shared configuration -- formatter choice, lint-on-save, file associations, search exclusions.

**The rule:** Personal preferences go in User Settings. Team standards go in Workspace Settings (committed to git).

Here's what a workspace `.vscode/settings.json` should look like for a React Native + Next.js monorepo:

```jsonc
// .vscode/settings.json (committed to git)
{
  // Formatter
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },

  // TypeScript
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,

  // File associations
  "files.associations": {
    "*.css": "tailwindcss"
  },

  // Search exclusions (project-specific)
  "search.exclude": {
    "**/node_modules": true,
    "**/.expo": true,
    "**/android/build": true,
    "**/ios/Pods": true,
    "**/dist": true,
    "**/.next": true,
    "**/coverage": true
  },

  // Recommended extensions
  // (Users see a prompt to install these when they open the project)
  // This actually goes in .vscode/extensions.json, shown below
}
```

And the companion `.vscode/extensions.json`:

```jsonc
// .vscode/extensions.json (committed to git)
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "usernameheo.errorlens",
    "yoavbls.pretty-ts-errors",
    "bradlc.vscode-tailwindcss",
    "expo.vscode-expo-tools"
  ],
  "unwantedRecommendations": [
    "hookyqr.beautify",
    "coenraads.bracket-pair-colorizer-2"
  ]
}
```

When a teammate opens the project, VS Code will prompt them to install the recommended extensions. This is how you standardize tooling without sending a Slack message that says "hey everyone install these 6 extensions."

### 2.5 Debugging in VS Code

Debugging with `console.log` is fine for simple issues. But for anything involving state management, navigation flows, or async timing, the VS Code debugger saves enormous amounts of time. You can set breakpoints, inspect variables, step through code, and watch expressions -- all without leaving the editor.

**Debugging React Native (Expo):**

Create a `.vscode/launch.json` file:

```jsonc
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Expo App",
      "type": "reactnative",
      "request": "attach",
      "platform": "exponent",
      "cwd": "${workspaceFolder}/apps/mobile",
      "sourceMaps": true,
      "sourceMapPathOverrides": {
        "webpack:///./~/*": "${workspaceFolder}/node_modules/*",
        "webpack:///./*": "${workspaceFolder}/*"
      }
    },
    {
      "name": "Debug Android",
      "type": "reactnative",
      "request": "launch",
      "platform": "android",
      "cwd": "${workspaceFolder}/apps/mobile"
    },
    {
      "name": "Debug iOS",
      "type": "reactnative",
      "request": "launch",
      "platform": "ios",
      "cwd": "${workspaceFolder}/apps/mobile"
    }
  ]
}
```

**How to use it:**
1. Start your Expo app with `npx expo start`
2. Open the Debug panel (`Cmd+Shift+D`)
3. Select "Debug Expo App" from the dropdown
4. Press F5 (or click the green play button)
5. Set breakpoints by clicking the gutter next to line numbers
6. When the app hits a breakpoint, you'll see the call stack, local variables, and you can step through code line by line

**Debugging Jest Tests:**

Add this configuration to your `launch.json`:

```jsonc
{
  "name": "Debug Current Test File",
  "type": "node",
  "request": "launch",
  "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/jest",
  "args": [
    "${relativeFile}",
    "--config",
    "${workspaceFolder}/apps/mobile/jest.config.ts",
    "--no-cache",
    "--no-coverage",
    "--runInBand"
  ],
  "console": "integratedTerminal",
  "cwd": "${workspaceFolder}",
  "env": {
    "NODE_ENV": "test"
  }
}
```

Now you can open a test file, set a breakpoint inside a test case, and press F5. The test runs and pauses at your breakpoint. You can inspect variables, evaluate expressions, and understand exactly why a test is failing.

**Debugging Next.js (Server-Side):**

```jsonc
{
  "name": "Debug Next.js",
  "type": "node",
  "request": "launch",
  "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/next",
  "args": ["dev"],
  "cwd": "${workspaceFolder}/apps/web",
  "env": {
    "NODE_OPTIONS": "--inspect"
  },
  "console": "integratedTerminal",
  "serverReadyAction": {
    "pattern": "started server on .+, url: (https?://.+)",
    "uriFormat": "%s",
    "action": "debugWithChrome"
  }
}
```

This starts Next.js in debug mode and automatically opens Chrome when the server is ready, with both server-side and client-side debugging connected.

### 2.6 Multi-Root Workspaces for Monorepos

If your monorepo has `apps/mobile`, `apps/web`, and `packages/shared`, you might want VS Code to treat each as a separate root -- with its own settings, its own search scope, and its own extension context.

Create a `.code-workspace` file:

```jsonc
// myproject.code-workspace
{
  "folders": [
    {
      "name": "Mobile App",
      "path": "apps/mobile"
    },
    {
      "name": "Web App",
      "path": "apps/web"
    },
    {
      "name": "Shared Packages",
      "path": "packages"
    },
    {
      "name": "Root (Config)",
      "path": "."
    }
  ],
  "settings": {
    "typescript.tsdk": "node_modules/typescript/lib",
    "search.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/.expo": true
    }
  }
}
```

Open this with `File → Open Workspace from File`. Each folder appears as a separate root in the explorer, with its own search context and file tree. This is especially useful when you want to search "only in the mobile app" or "only in shared packages."

### 2.7 VS Code Profiles

Profiles let you have completely different VS Code configurations for different kinds of work. Different extensions, different settings, different themes.

**Why this matters:** You probably don't need React Native Tools when you're writing a Next.js page. You probably don't need Tailwind IntelliSense when you're writing a Markdown document. Extensions you don't need slow down your editor and clutter your command palette.

**Setting up profiles:**

1. `Cmd+Shift+P` → "Profiles: Create Profile"
2. Name it (e.g., "React Native", "Web", "Writing")
3. Choose what to include: settings, keyboard shortcuts, extensions, UI state

**My profiles:**

- **React Native:** All the RN extensions, Expo Tools, mobile-focused debugger configs, dark theme
- **Web:** Next.js extensions, Tailwind IntelliSense, browser debugging configs
- **Writing:** Minimal extensions, Zen mode enabled, larger font, light theme, Markdown extensions only

Switch between profiles with `Cmd+Shift+P` → "Profiles: Switch Profile." You can also assign different profiles to different workspace folders, so opening your mobile project automatically activates the React Native profile.

### 2.8 Snippets and Snippet Generators

Custom snippets are one of VS Code's most underused features. Instead of typing the same boilerplate patterns hundreds of times, define them once and trigger them with a few keystrokes.

**Creating a custom snippet:**

`Cmd+Shift+P` → "Snippets: Configure User Snippets" → "typescriptreact.json"

```jsonc
{
  "React Native Functional Component": {
    "prefix": "rnfc",
    "body": [
      "import { View, ${2:Text} } from 'react-native';",
      "",
      "type ${1:ComponentName}Props = {",
      "  $3",
      "};",
      "",
      "export function ${1:ComponentName}({ $4 }: ${1:ComponentName}Props) {",
      "  return (",
      "    <View>",
      "      <${2:Text}>${1:ComponentName}</${2:Text}>",
      "    </View>",
      "  );",
      "}",
      ""
    ],
    "description": "React Native Functional Component with props type"
  },

  "useState Hook": {
    "prefix": "ust",
    "body": [
      "const [${1:state}, set${1/(.*)/${1:/capitalize}/}] = useState<${2:type}>(${3:initialValue});"
    ],
    "description": "useState hook with type"
  },

  "useEffect Hook": {
    "prefix": "uef",
    "body": [
      "useEffect(() => {",
      "  ${1:// effect}",
      "",
      "  return () => {",
      "    ${2:// cleanup}",
      "  };",
      "}, [${3:deps}]);"
    ],
    "description": "useEffect hook with cleanup"
  },

  "TanStack Query Hook": {
    "prefix": "uquery",
    "body": [
      "const { data, isPending, error } = useQuery({",
      "  queryKey: ['${1:key}'],",
      "  queryFn: async () => {",
      "    ${2:// fetch logic}",
      "  },",
      "});"
    ],
    "description": "TanStack Query useQuery hook"
  },

  "Zustand Store": {
    "prefix": "zstore",
    "body": [
      "import { create } from 'zustand';",
      "",
      "type ${1:StoreName}State = {",
      "  ${2:value}: ${3:string};",
      "  set${2/(.*)/${1:/capitalize}/}: (${2:value}: ${3:string}) => void;",
      "};",
      "",
      "export const use${1:StoreName}Store = create<${1:StoreName}State>((set) => ({",
      "  ${2:value}: ${4:initialValue},",
      "  set${2/(.*)/${1:/capitalize}/}: (${2:value}) => set({ ${2:value} }),",
      "}));"
    ],
    "description": "Zustand store with typed state"
  }
}
```

**Pro tip:** The snippet syntax supports transforms like `${1/(.*)/${1:/capitalize}/}`, which capitalizes the first letter. So if you type `count` for the state variable name in `ust`, the setter automatically becomes `setCount`. Snippet transforms are documented in the VS Code docs and can save you surprising amounts of time.

**Workspace snippets:** You can also create project-specific snippets in `.vscode/*.code-snippets` files. These get committed to git, so the whole team has access:

```jsonc
// .vscode/project.code-snippets
{
  "API Route Handler": {
    "prefix": "apiroute",
    "scope": "typescript",
    "body": [
      "import { NextRequest, NextResponse } from 'next/server';",
      "",
      "export async function ${1|GET,POST,PUT,DELETE|}(request: NextRequest) {",
      "  try {",
      "    ${2:// handler logic}",
      "",
      "    return NextResponse.json({ $3 });",
      "  } catch (error) {",
      "    console.error('${4:Route} error:', error);",
      "    return NextResponse.json(",
      "      { error: 'Internal server error' },",
      "      { status: 500 }",
      "    );",
      "  }",
      "}"
    ],
    "description": "Next.js App Router API route handler"
  }
}
```

### 2.9 Terminal Integration

VS Code's integrated terminal is surprisingly powerful. Stop switching between VS Code and a separate terminal app.

**Split terminals:** `Cmd+\` in the terminal panel splits it. Run your Expo dev server in one pane and your test watcher in another. Both visible at once.

**Named terminals:** Right-click on a terminal tab to rename it. "Expo", "Tests", "Git" -- labeled terminals are much easier to navigate than "zsh", "zsh", "zsh".

**Tasks:** VS Code tasks automate common terminal commands. Define them in `.vscode/tasks.json`:

```jsonc
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Start Expo",
      "type": "shell",
      "command": "npx expo start --clear",
      "options": {
        "cwd": "${workspaceFolder}/apps/mobile"
      },
      "isBackground": true,
      "problemMatcher": [],
      "group": "none",
      "presentation": {
        "reveal": "always",
        "panel": "dedicated",
        "echo": true
      }
    },
    {
      "label": "Start Next.js",
      "type": "shell",
      "command": "pnpm dev",
      "options": {
        "cwd": "${workspaceFolder}/apps/web"
      },
      "isBackground": true,
      "problemMatcher": [],
      "presentation": {
        "reveal": "always",
        "panel": "dedicated"
      }
    },
    {
      "label": "Run Tests (Watch)",
      "type": "shell",
      "command": "pnpm test --watch",
      "options": {
        "cwd": "${workspaceFolder}"
      },
      "isBackground": true,
      "problemMatcher": [],
      "presentation": {
        "reveal": "always",
        "panel": "dedicated"
      }
    },
    {
      "label": "Type Check",
      "type": "shell",
      "command": "pnpm tsc --noEmit",
      "options": {
        "cwd": "${workspaceFolder}"
      },
      "problemMatcher": "$tsc",
      "group": "build"
    },
    {
      "label": "Start All (Mobile + Web + Tests)",
      "dependsOn": ["Start Expo", "Start Next.js", "Run Tests (Watch)"],
      "dependsOrder": "parallel",
      "problemMatcher": []
    }
  ]
}
```

Run tasks with `Cmd+Shift+P` → "Tasks: Run Task". The "Start All" task launches Expo, Next.js, and the test watcher simultaneously in separate terminal panels. One command to start your entire development environment.

---

## 3. LINTING STRATEGY

Linting is the single most effective automated quality check you can run on a codebase. A well-configured linter catches bugs before they become PRs, enforces architectural decisions, and eliminates entire categories of code review feedback. It's not just about code style -- it's about code correctness.

### 3.1 ESLint: The Foundation

ESLint has been the standard JavaScript/TypeScript linter for a decade. As of 2025, the ecosystem has consolidated around the **flat config format** (`eslint.config.js`), which replaces the older `.eslintrc.*` files. If you're starting a new project, use flat config exclusively. If you're maintaining an old project, plan a migration.

**Why flat config is better:**
- Single file instead of scattered `.eslintrc` files in nested directories
- Explicit, composable configuration using JavaScript (not JSON/YAML guessing games)
- Better performance through explicit file targeting
- Cleaner plugin resolution (no more `extends` confusion)

### 3.2 ESLint Flat Config for a React Native + Next.js Monorepo

Here's a complete, production-ready `eslint.config.js` for a monorepo with React Native (Expo) and Next.js apps:

```js
// eslint.config.js (root of monorepo)
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';
import importPlugin from 'eslint-plugin-import';
import boundaries from 'eslint-plugin-boundaries';

export default tseslint.config(
  // Global ignores
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/build/**',
      '**/.expo/**',
      '**/.next/**',
      '**/android/**',
      '**/ios/**',
      '**/coverage/**',
      '**/*.generated.*',
    ],
  },

  // Base config for all JS/TS files
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,

  // TypeScript parser settings
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  // React settings (shared between RN and Next.js)
  {
    files: ['**/*.tsx', '**/*.jsx'],
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooksPlugin,
    },
    settings: {
      react: {
        version: 'detect',
      },
    },
    rules: {
      // React rules
      'react/jsx-uses-react': 'off',            // Not needed with new JSX transform
      'react/react-in-jsx-scope': 'off',         // Not needed with new JSX transform
      'react/prop-types': 'off',                 // TypeScript handles this
      'react/display-name': 'warn',
      'react/jsx-no-leaked-render': 'error',     // Prevent {count && <View />} (renders 0)
      'react/jsx-key': ['error', {               // Require keys in lists
        checkFragmentShorthand: true,
        warnOnDuplicates: true,
      }],

      // Rules of Hooks — NON-NEGOTIABLE
      'react-hooks/rules-of-hooks': 'error',     // Hooks must be called in the same order
      'react-hooks/exhaustive-deps': 'warn',     // Warn about missing dependencies
    },
  },

  // TypeScript-specific rules
  {
    files: ['**/*.ts', '**/*.tsx'],
    rules: {
      // -- Correctness --
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',                  // Allow _unused params
        varsIgnorePattern: '^_',
        destructuredArrayIgnorePattern: '^_',
      }],
      '@typescript-eslint/no-explicit-any': 'warn',  // Push toward proper types
      '@typescript-eslint/no-floating-promises': 'error', // Unhandled promises are bugs
      '@typescript-eslint/no-misused-promises': 'error',  // Prevents async in wrong places
      '@typescript-eslint/await-thenable': 'error',       // Don't await non-promises

      // -- Consistency --
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',                   // import type { Foo } from './foo'
        fixStyle: 'separate-type-imports',
      }],
      '@typescript-eslint/consistent-type-definitions': ['error', 'type'], // type > interface

      // -- Import hygiene --
      '@typescript-eslint/no-require-imports': 'error',   // ES modules only
    },
  },

  // Import ordering
  {
    plugins: {
      import: importPlugin,
    },
    rules: {
      'import/order': ['error', {
        groups: [
          'builtin',           // node:fs, node:path
          'external',          // react, react-native
          'internal',          // @/lib, @repo/shared
          ['parent', 'sibling'], // ../foo, ./bar
          'index',             // ./
          'type',              // type imports last
        ],
        pathGroups: [
          { pattern: 'react', group: 'external', position: 'before' },
          { pattern: 'react-native', group: 'external', position: 'before' },
          { pattern: 'expo-**', group: 'external', position: 'before' },
          { pattern: '@repo/**', group: 'internal', position: 'before' },
          { pattern: '@/**', group: 'internal', position: 'before' },
        ],
        pathGroupsExcludedImportTypes: ['type'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      }],
      'import/no-duplicates': 'error',            // Merge imports from same module
    },
  },

  // Architectural boundaries (optional but powerful)
  {
    plugins: {
      boundaries,
    },
    settings: {
      'boundaries/elements': [
        { type: 'app-mobile', pattern: 'apps/mobile/**' },
        { type: 'app-web', pattern: 'apps/web/**' },
        { type: 'package-ui', pattern: 'packages/ui/**' },
        { type: 'package-shared', pattern: 'packages/shared/**' },
      ],
    },
    rules: {
      'boundaries/element-types': ['error', {
        default: 'disallow',
        rules: [
          // Mobile app can import from ui and shared packages
          { from: 'app-mobile', allow: ['package-ui', 'package-shared'] },
          // Web app can import from ui and shared packages
          { from: 'app-web', allow: ['package-ui', 'package-shared'] },
          // UI package can import from shared
          { from: 'package-ui', allow: ['package-shared'] },
          // Shared package cannot import from any app or UI
          { from: 'package-shared', allow: [] },
          // Apps CANNOT import from each other
        ],
      }],
    },
  },

  // Next.js specific overrides
  {
    files: ['apps/web/**/*.ts', 'apps/web/**/*.tsx'],
    rules: {
      // Next.js uses default exports for pages and layouts
      'import/no-default-export': 'off',
    },
  },

  // Test file overrides
  {
    files: ['**/*.test.*', '**/*.spec.*', '**/__tests__/**'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'off',        // Mocks often need any
      '@typescript-eslint/no-floating-promises': 'off',   // Test assertions handle this
      '@typescript-eslint/no-unsafe-assignment': 'off',
    },
  },

  // Config files (JS, not TypeScript-checked)
  {
    files: ['*.config.js', '*.config.ts', '*.config.mjs'],
    ...tseslint.configs.disableTypeChecked,
  },
);
```

### 3.3 Understanding the Essential Rules

Let me explain why each category of rules matters, because "just add recommended" isn't good enough -- you need to understand what you're enforcing and why.

**Rules of Hooks (`react-hooks/rules-of-hooks: 'error'`):**

This is the most important React lint rule. Hooks must be called in the same order on every render. If you call a hook inside a condition or a loop, React's hook system breaks silently -- your state gets out of sync, your effects run on wrong dependencies, your app produces bugs that are nightmarish to debug.

```tsx
// BAD -- hook called conditionally
function UserProfile({ userId }: { userId: string | null }) {
  if (!userId) return null;
  const { data } = useQuery({ queryKey: ['user', userId] }); // ERROR: called after early return
  return <Text>{data?.name}</Text>;
}

// GOOD -- hook always called, data conditionally used
function UserProfile({ userId }: { userId: string | null }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    enabled: !!userId, // query doesn't run when userId is null
  });
  if (!userId) return null;
  return <Text>{data?.name}</Text>;
}
```

The linter catches the bad pattern immediately. Without it, you might not notice the bug until a specific state transition causes a crash in production.

**No Floating Promises (`@typescript-eslint/no-floating-promises: 'error'`):**

An unhandled promise rejection is a silent bug. The operation fails, no error is shown, no error boundary catches it, and the user sees stale data or a frozen UI.

```tsx
// BAD -- promise result ignored
async function saveUser(data: UserData) {
  api.updateUser(data); // ERROR: floating promise. If this fails, nobody knows.
}

// GOOD -- promise handled
async function saveUser(data: UserData) {
  await api.updateUser(data); // Now errors propagate properly
}

// ALSO GOOD -- explicitly ignoring (when you truly don't care)
function fireAndForget() {
  void analytics.track('button_clicked'); // void tells both TS and the linter: "I know"
}
```

**Consistent Type Imports (`@typescript-eslint/consistent-type-imports: 'error'`):**

This rule enforces `import type { Foo }` for type-only imports. Why does this matter? Because regular imports are included in the JavaScript bundle. Type-only imports are erased at compile time. When you `import type`, you guarantee that the import adds zero bytes to your bundle.

```tsx
// BAD -- value import for something used only as a type
import { UserData } from './types'; // This import exists in the JS bundle

// GOOD -- type import is erased at compile time
import type { UserData } from './types'; // Zero bundle impact
```

**Leaked Render Prevention (`react/jsx-no-leaked-render: 'error'`):**

One of React Native's most common visual bugs:

```tsx
// BAD -- renders "0" on screen when count is 0
function Badge({ count }: { count: number }) {
  return <View>{count && <Text>{count}</Text>}</View>;
  // When count is 0, this renders 0 (a number), not null
}

// GOOD -- explicit boolean check
function Badge({ count }: { count: number }) {
  return <View>{count > 0 ? <Text>{count}</Text> : null}</View>;
}
```

This is so common and so sneaky that it deserves its own lint rule. A bare `0` rendering in a React Native `<View>` can cause crashes on Android, not just a visual bug.

### 3.4 Architectural Boundary Enforcement with eslint-plugin-boundaries

This is one of the most underused tools in the ESLint ecosystem, and it's incredibly powerful for monorepos. `eslint-plugin-boundaries` lets you define rules about which packages can import from which other packages.

**Why this matters:** Without enforcement, boundaries erode. Someone imports a mobile-only hook into the web app "just this once." Someone imports from the web app into a shared package, creating a circular dependency. Over time, your clean architecture becomes a tangled mess where everything depends on everything.

With the boundaries config shown above:
- `packages/shared` cannot import from any app -- it stays truly shared
- `packages/ui` can only import from `packages/shared` -- no app-specific code leaking in
- Apps can import from packages but not from each other -- no cross-app coupling
- Any violation is a lint error, caught immediately, not discovered during a painful refactor months later

**When to invest in custom rules:** Most teams should never write custom ESLint rules. The ecosystem provides rules for 99% of use cases. Write a custom rule only when:
1. You have a recurring code review comment that could be automated (e.g., "don't use `console.log` in production code")
2. You have an architectural pattern that must be enforced (e.g., "all API calls must go through the `api/` layer")
3. The existing rules genuinely don't cover your case

If you find yourself reaching for a custom rule, first check if `eslint-plugin-boundaries`, `eslint-plugin-import`, or `@typescript-eslint` already covers it.

### 3.5 ESLint Performance

ESLint can be slow. On a large monorepo, a full lint pass might take 30+ seconds. This matters because slow linting means developers disable it (format-on-save becomes annoying) or CI takes too long.

**Diagnosing slow rules:**

```bash
# Run ESLint with timing enabled
TIMING=1 npx eslint .

# Output shows time per rule:
# Rule                               | Time (ms) | Relative
# @typescript-eslint/no-floating-promises |    4532  |    32.1%
# import/no-cycle                     |    3211  |    22.7%
# @typescript-eslint/no-misused-promises |    2145  |    15.2%
```

The usual suspects for slow ESLint:
1. **Type-checked rules** (`@typescript-eslint/no-floating-promises`, etc.) -- these invoke the TypeScript compiler, which is inherently slower. They're worth the cost for the bugs they catch, but be aware.
2. **`import/no-cycle`** -- this rule traverses the entire dependency graph. On large projects, it can be the single slowest rule. Consider running it only in CI, not on every save.
3. **`import/no-unresolved`** -- can be slow with complex path aliases. TypeScript already catches unresolved imports, so this rule is often redundant.

**Caching:**

```bash
# ESLint caches results and only re-lints changed files
npx eslint --cache --cache-location node_modules/.cache/eslint .
```

Add `--cache` to every ESLint invocation in your scripts. The first run is slow; subsequent runs only lint changed files and finish in seconds.

**Lint only changed files (in CI):**

```bash
# Get list of changed files from git
CHANGED_FILES=$(git diff --name-only --diff-filter=ACMRT HEAD~1 HEAD -- '*.ts' '*.tsx')

# Lint only those files
echo "$CHANGED_FILES" | xargs npx eslint --cache
```

This reduces a 30-second full lint to a 2-second targeted lint. We'll automate this with lint-staged in section 5.

---

## 4. FORMATTING

Formatting is the lowest-value thing an engineer can spend time on and the highest-value thing a tool can automate. Tabs vs spaces, semicolons vs no semicolons, trailing commas vs no trailing commas -- none of these decisions matter. What matters is that everyone on the team uses the same format, enforced automatically, so PRs never contain formatting noise and code reviews focus on logic.

### 4.1 Prettier: The Standard

Prettier is an opinionated code formatter. It deliberately gives you very few options because the point is to stop arguing about formatting. You configure it once, and it handles everything: JavaScript, TypeScript, JSX, CSS, JSON, Markdown, YAML, HTML, and GraphQL.

```jsonc
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindFunctions": ["cn", "cva"]
}
```

**Why these choices:**
- **`semi: true`** -- Semicolons prevent ASI (Automatic Semicolon Insertion) edge cases. It's one less thing to think about.
- **`singleQuote: true`** -- Fewer keystrokes than double quotes. The React ecosystem has mostly settled on single quotes.
- **`trailingComma: "all"`** -- Trailing commas make diffs cleaner (adding a new item only changes one line) and prevent missing-comma bugs.
- **`printWidth: 100`** -- The default 80 is too narrow for modern TypeScript with generics. 100 is a good balance between readability and avoiding excessive line wrapping.
- **`prettier-plugin-tailwindcss`** -- Automatically sorts Tailwind class names into a canonical order. Eliminates "which order should these classes be in?" from code review.

**Prettier ignore file:**

```gitignore
# .prettierignore
node_modules
dist
build
.expo
.next
android
ios
coverage
pnpm-lock.yaml
*.generated.*
```

### 4.2 Biome: The Fast Alternative

Biome (formerly Rome) is a Rust-based tool that combines linting and formatting in a single binary. It's 10-100x faster than ESLint + Prettier because it doesn't run in Node.js.

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100,
    "lineEnding": "lf"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "correctness": {
        "useExhaustiveDependencies": "warn",
        "useHookAtTopLevel": "error",
        "noUnusedImports": "error",
        "noUnusedVariables": "error"
      },
      "style": {
        "useConst": "error",
        "noNonNullAssertion": "warn",
        "useImportType": "error"
      },
      "suspicious": {
        "noExplicitAny": "warn",
        "noConsoleLog": "warn"
      },
      "complexity": {
        "noForEach": "warn"
      }
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always",
      "trailingCommas": "all",
      "arrowParentheses": "always"
    }
  },
  "files": {
    "ignore": [
      "node_modules",
      "dist",
      "build",
      ".expo",
      ".next",
      "android",
      "ios",
      "coverage"
    ]
  }
}
```

**When to choose Biome over Prettier + ESLint:**

| Factor | Prettier + ESLint | Biome |
|--------|-------------------|-------|
| Speed | Slower (Node.js) | 10-100x faster (Rust) |
| Ecosystem | Massive plugin ecosystem | Growing but limited |
| ESLint plugins | Full access (react-hooks, boundaries, etc.) | Built-in equivalents for many rules |
| Prettier plugins | Tailwind, imports, etc. | Fewer available |
| Community | Industry standard, everyone knows it | Smaller community, newer |
| Monorepo support | Well-established | Good and improving |

**My recommendation for 2026:** Use **ESLint + Prettier** for projects that need `eslint-plugin-react-hooks`, `eslint-plugin-boundaries`, or other specialized plugins. Use **Biome** for greenfield projects or projects where speed is the primary concern and you don't need niche plugins. You can also use Biome for formatting only (replacing Prettier) while keeping ESLint for linting.

### 4.3 EditorConfig: Cross-Editor Consistency

Not everyone on your team uses VS Code. Some people use Neovim, WebStorm, or Cursor. EditorConfig is a simple format that most editors support natively, ensuring basic formatting consistency across all editors.

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[*.{yaml,yml}]
indent_size = 2

[Makefile]
indent_style = tab
```

EditorConfig handles the basics (indent style, line endings, trailing whitespace) so that even before Prettier runs, files are in a reasonable state. It's a safety net -- if someone's Prettier extension isn't working, EditorConfig prevents the worst formatting inconsistencies.

### 4.4 The Pipeline Order: Format, Lint, Type-Check

When you run quality checks, the order matters:

1. **Format first** (Prettier/Biome) -- Reformats code to a consistent style. This must run before linting because many lint rules (like `max-len`) are affected by formatting.

2. **Lint second** (ESLint) -- Checks for code quality issues on already-formatted code. Lint rules that have auto-fix will clean up issues that the formatter doesn't handle (like import ordering, unused variables).

3. **Type-check third** (TypeScript `tsc --noEmit`) -- Checks type correctness. This runs last because it's the slowest and because formatting/linting changes might fix type issues (like removing unused imports that cause "imported but not used" errors).

This pipeline is the order for both your git hooks (section 5) and your CI pipeline (chapter 21).

```
Save file → Prettier formats → ESLint auto-fixes → TypeScript checks
              (instant)           (instant)          (background)
```

In VS Code, with the settings we configured earlier, steps 1 and 2 happen automatically on save. Step 3 happens in the background via the TypeScript language server, with errors shown inline.

---

## 5. GIT HOOKS AND AUTOMATION

Here's the problem: you've set up ESLint and Prettier, configured VS Code to format on save, and written a 200-line `eslint.config.js`. Then someone on the team commits from the command line. Or uses a different editor. Or has a broken extension. And suddenly your clean codebase has unformatted code in it.

The solution: **git hooks.** Run quality checks automatically at commit time, before the code ever reaches the repository. If the checks fail, the commit is rejected. No exceptions, no "I'll fix it in the next commit," no formatting noise in PRs.

### 5.1 Husky: Git Hooks Made Easy

Husky is a tool that makes it easy to set up git hooks in a JavaScript project. Without Husky, you'd have to manually create shell scripts in `.git/hooks/`, which is fragile and not committed to version control.

**Installation:**

```bash
pnpm add -D husky
npx husky init
```

This creates a `.husky/` directory and a `prepare` script in `package.json` that installs hooks automatically when someone runs `pnpm install`.

**The pre-commit hook:**

```bash
#!/bin/sh
# .husky/pre-commit
pnpm lint-staged
```

That's it. One line. The real logic lives in lint-staged.

### 5.2 lint-staged: Run Checks Only on Staged Files

lint-staged is the key to fast pre-commit hooks. Instead of running ESLint on your entire codebase (which could take 30+ seconds), it runs only on the files you're about to commit. A typical commit touches 3-10 files, so the check finishes in 2-5 seconds.

```jsonc
// .lintstagedrc
{
  "*.{ts,tsx}": [
    "prettier --write",
    "eslint --fix --cache"
  ],
  "*.{js,jsx,mjs,cjs}": [
    "prettier --write",
    "eslint --fix --cache"
  ],
  "*.{json,md,yaml,yml}": [
    "prettier --write"
  ],
  "*.css": [
    "prettier --write"
  ]
}
```

**What happens when you commit:**
1. Git identifies staged files (the files you `git add`-ed)
2. lint-staged filters them by the glob patterns
3. For each matching file: Prettier formats it, then ESLint fixes what it can
4. If Prettier or ESLint changed any files, those changes are automatically added to the commit
5. If ESLint has unfixable errors, the commit is rejected

**The result:** Every commit in your repository is formatted and lint-clean. No exceptions.

**Adding type-checking to the pipeline:**

Type-checking on staged files is tricky because TypeScript needs to see the full project to check types correctly -- you can't type-check a single file in isolation. The solution is to run the full type-check but only on pre-commit:

```jsonc
// .lintstagedrc (with type-checking)
{
  "*.{ts,tsx}": [
    "prettier --write",
    "eslint --fix --cache",
    "bash -c 'pnpm tsc --noEmit'"
  ],
  "*.{js,jsx,mjs,cjs}": [
    "prettier --write",
    "eslint --fix --cache"
  ],
  "*.{json,md,yaml,yml}": [
    "prettier --write"
  ]
}
```

**Note:** The `bash -c 'pnpm tsc --noEmit'` trick runs `tsc` once for the entire project, regardless of how many TS files are staged. Without the `bash -c` wrapper, lint-staged would try to pass individual file paths to `tsc`, which doesn't work the way you'd expect.

**Caveat:** Full type-checking on pre-commit can take 5-15 seconds on large projects. If this is too slow, move type-checking to CI only and keep pre-commit fast with just format + lint. The trade-off is catching type errors later, but commits stay fast.

### 5.3 commitlint: Conventional Commit Messages

Conventional commits are a lightweight convention on top of commit messages. They look like this:

```
feat: add user profile screen
fix: prevent crash when user data is null
chore: update expo SDK to 52
docs: add API documentation for auth module
refactor: extract payment logic into service layer
test: add integration tests for checkout flow
```

**Why bother?** Because conventional commits enable:
- **Automated changelogs** -- tools can generate release notes from commit messages
- **Semantic versioning** -- `feat:` bumps minor version, `fix:` bumps patch
- **Searchable history** -- `git log --grep="^fix:"` shows only bug fixes
- **PR context** -- a well-structured commit message tells reviewers what to focus on

**Setup:**

```bash
pnpm add -D @commitlint/cli @commitlint/config-conventional
```

```js
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat',     // New feature
      'fix',      // Bug fix
      'chore',    // Maintenance (deps, configs)
      'docs',     // Documentation
      'refactor', // Code change that neither fixes a bug nor adds a feature
      'test',     // Adding or updating tests
      'style',    // Formatting, whitespace (no code change)
      'perf',     // Performance improvement
      'ci',       // CI/CD changes
      'revert',   // Revert a previous commit
    ]],
    'subject-case': [2, 'always', 'lower-case'],  // lowercase subjects
    'header-max-length': [2, 'always', 72],        // keep subjects short
    'body-max-line-length': [2, 'always', 100],
  },
};
```

**Git hook for commit messages:**

```bash
#!/bin/sh
# .husky/commit-msg
npx --no -- commitlint --edit "$1"
```

Now if someone tries to commit with a message like "fixed stuff" or "WIP", commitlint rejects it and explains the expected format. This feels strict at first, but after a week it becomes second nature, and your git history becomes genuinely useful.

### 5.4 The Complete Pre-Commit Pipeline

Here's what happens when an engineer commits, end to end:

```
git commit -m "feat: add user profile avatar upload"
  │
  ├─ .husky/commit-msg
  │   └─ commitlint validates message format ✓
  │
  ├─ .husky/pre-commit
  │   └─ lint-staged runs on staged files:
  │       ├─ Prettier formats *.ts, *.tsx, *.json ✓
  │       ├─ ESLint --fix auto-corrects *.ts, *.tsx ✓
  │       ├─ ESLint reports unfixable errors (if any) ✗ ← commit blocked
  │       └─ (Optional) tsc --noEmit type-checks ✓
  │
  └─ Commit created ✓
```

**Total time: 2-5 seconds** (without type-checking), **5-15 seconds** (with type-checking).

**Why this matters:** Without this pipeline, a developer pushes code, CI runs for 5-10 minutes, and discovers a lint error. The developer has already context-switched to something else. They have to switch back, fix the error, push again, and wait another 5-10 minutes. With the pre-commit pipeline, the error is caught in 3 seconds, fixed immediately, and CI passes on the first try. Over a week, this saves hours of wasted CI time and context-switching.

### 5.5 Bypassing Hooks (When Necessary)

Sometimes you genuinely need to bypass hooks -- a hotfix at 2 AM, a commit that only changes a generated file, a work-in-progress save. Git supports this:

```bash
git commit --no-verify -m "hotfix: emergency production fix"
```

The `--no-verify` flag skips all hooks. Use it sparingly and only when you understand why. If you find yourself using it regularly, your hooks are too strict or too slow.

---

## 6. MAINTAINING CODEBASES

Setting up linting and formatting is the easy part. Keeping a codebase healthy over months and years -- that's the real challenge. Dependencies go stale, dead code accumulates, tech debt compounds, and the codebase gradually becomes harder to work in. Here's how to fight entropy.

### 6.1 Dependency Updates with Renovate

Outdated dependencies are security vulnerabilities, performance regressions, and accumulated migration debt. The longer you wait to update, the harder each update becomes.

**Renovate** is a bot that automatically creates PRs to update your dependencies. It's more configurable than Dependabot and handles monorepos better.

```jsonc
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended",
    ":automergeMinor",
    ":automergePatch",
    "group:allNonMajor"
  ],
  "schedule": ["before 8am on Monday"],
  "timezone": "America/New_York",
  "labels": ["dependencies"],
  "packageRules": [
    {
      "description": "Auto-merge patch updates for production deps",
      "matchUpdateTypes": ["patch"],
      "matchDepTypes": ["dependencies"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "description": "Auto-merge minor updates for dev deps",
      "matchUpdateTypes": ["minor", "patch"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true,
      "automergeType": "branch"
    },
    {
      "description": "Group Expo SDK updates together",
      "matchPackagePatterns": ["^expo", "^@expo/"],
      "groupName": "Expo SDK",
      "automerge": false
    },
    {
      "description": "Group React Native updates together",
      "matchPackagePatterns": ["^react-native", "^@react-native/"],
      "groupName": "React Native",
      "automerge": false
    },
    {
      "description": "Require manual review for major updates",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["dependencies", "breaking-change"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

**The strategy:**
- **Patch updates (1.0.x):** Auto-merge. These are bug fixes. If your tests pass, ship it.
- **Minor updates (1.x.0) for dev deps:** Auto-merge. Your users don't see dev dependencies.
- **Minor updates for prod deps:** Review briefly, then merge. Check the changelog for behavior changes.
- **Major updates:** Review carefully. Read the migration guide. Test thoroughly. These can have breaking changes.
- **Expo and React Native:** Always group and review manually. These are your platform -- treat updates with respect.

**Renovate vs Dependabot:** Dependabot (GitHub's built-in tool) is simpler to set up but less configurable. It can't group related updates, its auto-merge is less flexible, and it doesn't handle monorepo workspaces as well. For serious projects, Renovate is worth the setup.

### 6.2 Dead Code Detection with knip

Dead code is invisible tech debt. Unused exports, unreferenced files, unnecessary dependencies -- they clutter the codebase, confuse new engineers, and bloat the bundle.

**knip** is a tool that finds dead code in JavaScript/TypeScript projects:

```bash
pnpm add -D knip
```

```jsonc
// knip.json
{
  "workspaces": {
    "apps/mobile": {
      "entry": ["app/**/*.tsx", "app/_layout.tsx"],
      "project": ["**/*.{ts,tsx}"],
      "ignore": ["**/*.test.*", "**/*.spec.*"]
    },
    "apps/web": {
      "entry": ["app/**/*.tsx", "app/layout.tsx"],
      "project": ["**/*.{ts,tsx}"]
    },
    "packages/shared": {
      "entry": ["src/index.ts"],
      "project": ["src/**/*.{ts,tsx}"]
    },
    "packages/ui": {
      "entry": ["src/index.ts"],
      "project": ["src/**/*.{ts,tsx}"]
    }
  }
}
```

```bash
# Run knip to find dead code
npx knip

# Example output:
# Unused files (2)
#   packages/shared/src/utils/legacy-auth.ts
#   packages/ui/src/components/OldButton.tsx
#
# Unused exports (5)
#   packages/shared/src/api/client.ts: formatLegacyResponse
#   packages/ui/src/components/Card.tsx: CardVariant
#   ...
#
# Unused dependencies (3)
#   packages/shared: moment (use date-fns instead?)
#   apps/mobile: react-native-gesture-handler (not imported anywhere)
#   ...
```

Run knip monthly (or add it to CI). Every unused file and dependency you remove makes the codebase simpler, the bundle smaller, and onboarding faster.

### 6.3 Bundle Analysis

Your app's bundle size directly affects cold start time (React Native) and page load time (Next.js). Monitor it.

**For React Native:**

```bash
# Analyze your Metro bundle
npx react-native-bundle-visualizer

# Or use the --stats flag with Metro
npx expo export --platform ios --dump-sourcemap
```

**For Next.js:**

```bash
# Next.js has built-in bundle analysis
# Add to next.config.js:
# const withBundleAnalyzer = require('@next/bundle-analyzer')({ enabled: process.env.ANALYZE === 'true' });
ANALYZE=true pnpm build
```

**The Import Cost extension** we installed earlier shows bundle size inline for every import statement. When you see `import { format } from 'date-fns'` showing `5.2K (gzipped)`, you know that's fine. When you see `import moment from 'moment'` showing `72K (gzipped)`, you know it's time to switch.

**`why-did-you-render`** is a React profiling tool that warns when components re-render unnecessarily. Add it to your dev build:

```tsx
// app/_layout.tsx (development only)
if (__DEV__) {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true,
    trackHooks: true,
  });
}
```

### 6.4 Technical Debt Management

Every codebase accumulates technical debt. The question isn't whether to have debt -- it's whether you manage it intentionally or let it manage you.

**The Boy Scout Rule:** Leave the code cleaner than you found it. Every PR that touches a file is an opportunity to fix a small thing -- rename a confusing variable, add a missing type, remove a dead import. These micro-improvements compound over time.

**Debt Tickets:** Maintain a "tech debt" tag in your issue tracker. When you encounter debt that's too large to fix in the current PR, create a ticket. This makes debt visible and prioritizable, rather than invisible and growing.

**Focused Tech Debt Sprints:** Every 4-6 sprints, dedicate a sprint (or half a sprint) to tech debt reduction. Pick the highest-impact items from the debt backlog. These sprints are morale boosters -- engineers love cleaning up code they've been complaining about.

**A practical prioritization framework for tech debt:**

| Category | Impact | When to Fix |
|----------|--------|-------------|
| Security vulnerabilities | Critical | Immediately |
| Broken tests / flaky tests | High | This sprint |
| Deprecated dependencies with migration guides | High | Next sprint |
| Dead code / unused dependencies | Medium | Tech debt sprint |
| Missing types (`any` usage) | Medium | Boy scout rule |
| Inconsistent patterns | Low | When touching the file |
| Old code style | Low | Never (unless it causes bugs) |

### 6.5 Code Ownership with CODEOWNERS

GitHub's CODEOWNERS file automatically assigns reviewers based on which files a PR changes. This is essential for teams larger than 5-6 people.

```gitignore
# .github/CODEOWNERS

# Default: the frontend platform team reviews everything
* @org/frontend-platform

# Mobile app: mobile team
/apps/mobile/ @org/mobile-team

# Web app: web team
/apps/web/ @org/web-team

# Shared packages: platform team (changes here affect everyone)
/packages/shared/ @org/frontend-platform
/packages/ui/ @org/frontend-platform @org/design-system

# Infrastructure and config: platform team
/.github/ @org/frontend-platform
/eslint.config.js @org/frontend-platform
/tsconfig*.json @org/frontend-platform

# CI/CD: devops team
/.github/workflows/ @org/devops
```

**Why this matters:** Without CODEOWNERS, PRs get assigned to whoever happens to be online, which often means the wrong person reviews. Changes to shared packages might get approved by someone who doesn't understand the downstream impact. CODEOWNERS ensures the right experts see the right changes.

### 6.6 Documentation as Code

Documentation that lives outside the codebase dies. Documentation inside the codebase survives.

**README per package:** Every package in your monorepo should have a README that explains:
- What the package does (one paragraph)
- How to use it (code example)
- How to develop it (setup instructions)
- Key decisions (why it's built this way)

```markdown
# @repo/ui

Shared UI component library for mobile and web apps. Built with React Native
components that work cross-platform via NativeWind.

## Usage

\`\`\`tsx
import { Button, Card, Input } from '@repo/ui';
\`\`\`

## Development

\`\`\`bash
pnpm --filter @repo/ui dev    # Start Storybook
pnpm --filter @repo/ui test   # Run tests
\`\`\`

## Key Decisions

- Components use NativeWind (Tailwind) for styling
- All components are uncontrolled by default with optional controlled mode
- Exported components must work on both iOS, Android, and web
```

**Architecture Decision Records (ADRs):** For significant architectural decisions, write an ADR. These are short documents that explain:
- What was decided
- Why (context, constraints, alternatives considered)
- Consequences (trade-offs accepted)

Store them in a `docs/decisions/` folder and number them sequentially:

```
docs/decisions/
  001-use-expo-router.md
  002-zustand-over-redux.md
  003-monorepo-with-turborepo.md
  004-nativewind-for-styling.md
```

ADRs prevent the same debates from happening every 6 months when someone new joins and asks "why didn't we use Redux?" The answer is in ADR 002.

### 6.7 Deprecation Patterns

Codebases evolve. APIs change. Components get replaced. The question is whether you manage these transitions gracefully or let old and new code coexist forever in a confusing state.

**Deprecating a component:**

```tsx
/**
 * @deprecated Use `<Button>` from `@repo/ui` instead.
 * This component will be removed in v3.0.
 * Migration guide: docs/migrations/old-button-to-button.md
 */
export function OldButton(props: OldButtonProps) {
  if (__DEV__) {
    console.warn(
      'OldButton is deprecated. Use <Button> from @repo/ui instead. ' +
      'See docs/migrations/old-button-to-button.md'
    );
  }
  return <TouchableOpacity {...props} />;
}
```

**Deprecating an API function:**

```tsx
/**
 * @deprecated Use `fetchUser` from `@repo/shared/api` instead.
 * Removal target: 2026-Q3
 */
export function getUserData(id: string): Promise<User> {
  // Still works, but logs a warning in development
  if (__DEV__) {
    console.warn('getUserData is deprecated. Use fetchUser from @repo/shared/api.');
  }
  return fetchUser(id);
}
```

**The deprecation lifecycle:**
1. **Announce:** Add `@deprecated` JSDoc tag. Log warnings in development. Create a migration guide.
2. **Track:** Use knip or grep to find all usages. Create tickets to migrate each one.
3. **Migrate:** Replace usages over 2-3 sprints. The PR diff should be straightforward since the replacement is documented.
4. **Remove:** Once all usages are gone, delete the deprecated code. If you're not sure, leave it for one more release cycle with an error-level warning.

**Enforcing deprecation with ESLint:**

You can write a simple ESLint config to flag specific imports:

```js
// In your eslint.config.js
{
  rules: {
    'no-restricted-imports': ['error', {
      patterns: [
        {
          group: ['*/legacy/*'],
          message: 'Legacy modules are deprecated. See docs/migrations/ for alternatives.',
        },
      ],
      paths: [
        {
          name: 'moment',
          message: 'Use date-fns instead. See ADR-007.',
        },
      ],
    }],
  },
}
```

---

## 7. GIT WORKFLOW

Your git workflow is the process through which code moves from an engineer's brain to production. A good workflow is lightweight, consistent, and catches problems early. A bad workflow is either too bureaucratic (15-step review process for a typo fix) or too loose (everyone pushes to main with no review).

### 7.1 Branch Naming Conventions

Consistent branch names make `git branch` output meaningful, enable CI/CD automation (different pipelines for feature vs fix branches), and help teammates understand what you're working on.

```
feature/user-profile-avatar      # New functionality
fix/crash-on-empty-user-data     # Bug fix
chore/update-expo-sdk-52         # Maintenance, dependencies
docs/api-authentication-guide    # Documentation
refactor/extract-payment-service # Code restructuring
test/checkout-integration-tests  # Adding tests
hotfix/production-auth-failure   # Emergency production fix
```

**Rules:**
- Lowercase, hyphen-separated
- Prefix matches conventional commit type
- Short but descriptive (avoid `feature/stuff` or `fix/bug`)
- Include ticket number if your team uses one: `feature/PROJ-123-user-profile`

### 7.2 Conventional Commits in Practice

We set up commitlint in section 5.3. Here's how to write good conventional commits:

```
feat: add biometric authentication for login

Add Face ID / fingerprint support using expo-local-authentication.
Users can enable biometrics in Settings > Security.

Closes PROJ-456
```

**Structure:**
```
<type>(<optional scope>): <description>

<optional body>

<optional footer>
```

**Good commit messages:**
```
feat(auth): add biometric login with Face ID support
fix(profile): prevent crash when user avatar URL is null
chore(deps): update expo SDK from 51 to 52
refactor(api): extract HTTP client into shared package
perf(list): virtualize transaction list for 10x scroll performance
test(checkout): add integration tests for payment flow
docs(readme): add development setup instructions for new engineers
```

**Bad commit messages:**
```
fix bug                          # What bug? Where?
update stuff                     # What stuff?
WIP                              # Work in progress is not a commit message
asdf                             # Please
Fixes things                     # Capital letter, no type prefix, vague
```

### 7.3 PR Templates

A PR template ensures every pull request includes the context reviewers need. Without a template, you get PRs titled "Update" with no description, and reviewers have to read every line of code to understand what's happening.

Create `.github/pull_request_template.md`:

```markdown
## What

<!-- What does this PR do? One paragraph. -->

## Why

<!-- Why is this change needed? Link to issue/ticket if applicable. -->

## How

<!-- How was this implemented? Key technical decisions. -->

## Testing

<!-- How was this tested? -->
- [ ] Unit tests added/updated
- [ ] Manual testing on iOS
- [ ] Manual testing on Android
- [ ] Manual testing on web (if applicable)

## Screenshots/Videos

<!-- If this is a UI change, include before/after screenshots or a video. -->

## Checklist

- [ ] Types are correct (no new `any` usage)
- [ ] No console.log statements left
- [ ] Accessibility: new UI elements have proper labels
- [ ] PR title follows conventional commit format
```

**The "What, Why, How" structure** is deliberate:
- **What:** Lets reviewers know the scope before they look at code
- **Why:** Provides the context needed to evaluate whether the approach makes sense
- **How:** Highlights key decisions so reviewers focus on the important parts

### 7.4 Code Review Best Practices

Code review is one of the highest-leverage engineering activities. A good review catches bugs, spreads knowledge, and maintains code quality. A bad review is either a rubber stamp ("LGTM") or a nitpick festival that demoralizes the author.

**For reviewers:**

1. **Review the "what" and "why" before the "how."** Read the PR description first. Understand the goal. Then evaluate whether the code achieves that goal correctly. Don't start with line-by-line comments on code you don't understand the purpose of.

2. **Distinguish between blocking and non-blocking feedback.** Use a convention:
   - `blocking:` This must be changed before merge (bugs, security issues, architectural problems)
   - `nit:` This could be better but isn't a merge blocker (naming, style, minor improvements)
   - `question:` I want to understand this, not necessarily change it
   - `suggestion:` Consider this alternative (non-blocking)

3. **Review the test coverage.** If the PR adds new functionality, are there tests? If it fixes a bug, is there a test that would have caught the bug?

4. **Check for missing error handling.** This is the most common source of bugs in code review. What happens when the API returns an error? What happens when the input is null? What happens when the network is offline?

5. **Timebox your review.** If you can't finish a meaningful review in 30 minutes, the PR is too large. Ask the author to split it. PRs over 400 lines of code have dramatically lower review quality.

**For authors:**

1. **Keep PRs small.** Aim for under 200 lines of meaningful changes (not counting generated code, lock files, or snapshot updates). Small PRs get faster reviews, better feedback, and merge sooner.

2. **Self-review before requesting review.** Read your own diff as if you're the reviewer. You'll catch 30% of issues before anyone else sees them.

3. **Respond to every comment.** Even if it's just "Good point, fixed" or "Intentional because X." Unresolved comments are ambiguous -- the reviewer doesn't know if you saw them.

4. **Don't take feedback personally.** Code review is about the code, not about you. A "this approach has a memory leak" comment is helpful, not an attack.

### 7.5 Squash Merge vs Merge Commit

Two strategies for merging PRs, with different trade-offs:

**Squash merge:** Combines all commits in a PR into a single commit on the target branch.
- **Pro:** Clean, linear history. One commit per feature/fix. Easy to revert (revert one commit).
- **Con:** Loses granular commit history from the PR.
- **When to use:** Most PRs. Especially small-to-medium features, bug fixes, and maintenance.

**Merge commit:** Creates a merge commit that preserves all individual commits from the PR branch.
- **Pro:** Preserves full commit history. Good for large changes where individual commits tell a story.
- **Con:** History includes WIP commits, fixup commits, and merge noise.
- **When to use:** Large architectural changes, multi-day features where individual commits have meaning.

**My recommendation:** Default to squash merge. Use merge commits only when the PR author has carefully structured their commit history and each commit is meaningful on its own. In practice, this means squash merge 90% of the time.

**Configure this in GitHub:** Settings → General → Pull Requests → check "Allow squash merging" and set "Default commit message" to "Pull request title and description." Uncheck "Allow merge commits" if you want to enforce squash.

---

## 8. THE 10-MINUTE SETUP

Here's the copy-paste setup that configures everything we've discussed for a new project. Run these commands and you'll have ESLint + Prettier + Husky + lint-staged + commitlint + EditorConfig configured in minutes. This is the "never think about tooling again" setup.

### 8.1 The Script

```bash
#!/bin/bash
# setup-dx.sh — One-command DX setup for React Native + Next.js projects
# Usage: chmod +x setup-dx.sh && ./setup-dx.sh

set -e

echo "=== Setting up DX tooling ==="

# ---- 1. Install dependencies ----
echo "Installing dev dependencies..."
pnpm add -D \
  eslint \
  @eslint/js \
  typescript-eslint \
  eslint-plugin-react \
  eslint-plugin-react-hooks \
  eslint-plugin-import \
  prettier \
  eslint-config-prettier \
  prettier-plugin-tailwindcss \
  husky \
  lint-staged \
  @commitlint/cli \
  @commitlint/config-conventional

# ---- 2. ESLint config ----
echo "Creating eslint.config.js..."
cat > eslint.config.js << 'ESLINT_EOF'
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';
import importPlugin from 'eslint-plugin-import';

export default tseslint.config(
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**',
      '**/build/**',
      '**/.expo/**',
      '**/.next/**',
      '**/android/**',
      '**/ios/**',
      '**/coverage/**',
    ],
  },

  js.configs.recommended,
  ...tseslint.configs.recommended,

  {
    files: ['**/*.tsx', '**/*.jsx'],
    plugins: {
      react: reactPlugin,
      'react-hooks': reactHooksPlugin,
    },
    settings: {
      react: { version: 'detect' },
    },
    rules: {
      'react/jsx-uses-react': 'off',
      'react/react-in-jsx-scope': 'off',
      'react/prop-types': 'off',
      'react/jsx-no-leaked-render': 'error',
      'react/jsx-key': ['error', { checkFragmentShorthand: true }],
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },

  {
    files: ['**/*.ts', '**/*.tsx'],
    rules: {
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/consistent-type-imports': ['error', {
        prefer: 'type-imports',
        fixStyle: 'separate-type-imports',
      }],
    },
  },

  {
    plugins: { import: importPlugin },
    rules: {
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', ['parent', 'sibling'], 'index', 'type'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      }],
      'import/no-duplicates': 'error',
    },
  },

  {
    files: ['**/*.test.*', '**/*.spec.*'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'off',
    },
  },
);
ESLINT_EOF

# ---- 3. Prettier config ----
echo "Creating .prettierrc..."
cat > .prettierrc << 'PRETTIER_EOF'
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "endOfLine": "lf",
  "plugins": ["prettier-plugin-tailwindcss"]
}
PRETTIER_EOF

cat > .prettierignore << 'IGNORE_EOF'
node_modules
dist
build
.expo
.next
android
ios
coverage
pnpm-lock.yaml
*.generated.*
IGNORE_EOF

# ---- 4. EditorConfig ----
echo "Creating .editorconfig..."
cat > .editorconfig << 'EDITOR_EOF'
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
EDITOR_EOF

# ---- 5. Husky ----
echo "Setting up Husky..."
npx husky init

cat > .husky/pre-commit << 'HUSKY_PRE_EOF'
pnpm lint-staged
HUSKY_PRE_EOF

cat > .husky/commit-msg << 'HUSKY_MSG_EOF'
npx --no -- commitlint --edit "$1"
HUSKY_MSG_EOF

# ---- 6. lint-staged ----
echo "Creating .lintstagedrc..."
cat > .lintstagedrc << 'STAGED_EOF'
{
  "*.{ts,tsx}": [
    "prettier --write",
    "eslint --fix --cache"
  ],
  "*.{js,jsx,mjs,cjs}": [
    "prettier --write",
    "eslint --fix --cache"
  ],
  "*.{json,md,yaml,yml,css}": [
    "prettier --write"
  ]
}
STAGED_EOF

# ---- 7. commitlint ----
echo "Creating commitlint.config.js..."
cat > commitlint.config.js << 'COMMIT_EOF'
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'header-max-length': [2, 'always', 72],
    'subject-case': [2, 'always', 'lower-case'],
  },
};
COMMIT_EOF

# ---- 8. VS Code workspace settings ----
echo "Creating VS Code settings..."
mkdir -p .vscode

cat > .vscode/settings.json << 'VSCODE_EOF'
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "search.exclude": {
    "**/node_modules": true,
    "**/.expo": true,
    "**/android/build": true,
    "**/ios/Pods": true,
    "**/dist": true,
    "**/.next": true,
    "**/coverage": true
  }
}
VSCODE_EOF

cat > .vscode/extensions.json << 'EXT_EOF'
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "usernameheo.errorlens",
    "yoavbls.pretty-ts-errors",
    "bradlc.vscode-tailwindcss",
    "expo.vscode-expo-tools"
  ]
}
EXT_EOF

# ---- 9. Package.json scripts ----
echo "Adding scripts to package.json..."
npx json -I -f package.json -e '
  this.scripts = this.scripts || {};
  this.scripts["lint"] = "eslint .";
  this.scripts["lint:fix"] = "eslint --fix .";
  this.scripts["format"] = "prettier --write .";
  this.scripts["format:check"] = "prettier --check .";
  this.scripts["typecheck"] = "tsc --noEmit";
  this.scripts["check"] = "pnpm format:check && pnpm lint && pnpm typecheck";
  this.scripts["prepare"] = "husky";
' 2>/dev/null || echo "Note: Add lint/format/typecheck scripts to package.json manually if json CLI is not available."

echo ""
echo "=== DX setup complete! ==="
echo ""
echo "What was configured:"
echo "  ✓ ESLint (flat config, TypeScript, React, import ordering)"
echo "  ✓ Prettier (with Tailwind plugin)"
echo "  ✓ EditorConfig"
echo "  ✓ Husky (pre-commit + commit-msg hooks)"
echo "  ✓ lint-staged (format + lint on staged files)"
echo "  ✓ commitlint (conventional commit enforcement)"
echo "  ✓ VS Code workspace settings + recommended extensions"
echo ""
echo "Next steps:"
echo "  1. Run 'pnpm install' to activate Husky hooks"
echo "  2. Open VS Code and install recommended extensions when prompted"
echo "  3. Make a commit to test: git commit -m 'chore: set up DX tooling'"
```

### 8.2 What This Gives You

After running this script, your project has:

1. **Auto-formatting on save** -- Prettier formats every file when you hit Cmd+S
2. **Inline linting** -- ESLint errors appear in VS Code as you type
3. **Pre-commit quality gate** -- Every commit is automatically formatted and linted
4. **Commit message enforcement** -- Conventional commits are required
5. **Consistent editor config** -- Any editor respects basic formatting rules
6. **Recommended extensions** -- New team members get prompted to install the right tools

**Total setup time:** Under 10 minutes, including installation.

**Ongoing maintenance time:** Near zero. The configuration is stable. You'll update ESLint and Prettier versions with Renovate. The rules rarely need changing.

### 8.3 The Biome Alternative

If you prefer Biome over ESLint + Prettier, the setup is even simpler:

```bash
# Install Biome
pnpm add -D @biomejs/biome

# Initialize config
npx @biomejs/biome init

# Configure (edit biome.json as shown in section 4.2)
```

Then update your lint-staged config:

```jsonc
// .lintstagedrc (Biome version)
{
  "*.{ts,tsx,js,jsx}": [
    "biome check --write"
  ],
  "*.{json,md,yaml,yml,css}": [
    "biome format --write"
  ]
}
```

And your VS Code settings:

```jsonc
// .vscode/settings.json (Biome version)
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit"
  }
}
```

Biome replaces both ESLint and Prettier with a single tool. The configuration is simpler, the execution is faster, and there's only one tool to maintain. The trade-off is a smaller plugin ecosystem.

---

## PUTTING IT ALL TOGETHER

Let me paint the picture of what your development workflow looks like after implementing everything in this chapter.

**Morning, 9:00 AM.** You open VS Code. Your React Native profile loads automatically with the right extensions. You run the "Start All" task -- Expo, Next.js, and the test watcher start in parallel in named terminal panels.

**9:05 AM.** You open a component with `Cmd+P`, type three characters, and you're there. Error Lens shows two warnings inline: an unused variable and a missing dependency in a `useEffect`. You fix both with `Cmd+.` (quick fix). Save. Prettier formats. ESLint auto-fixes the import order.

**10:30 AM.** You've finished a feature. You commit: `git commit -m "feat(profile): add avatar upload with image compression"`. The pre-commit hook runs in 3 seconds -- format, lint, pass. Commitlint validates the message format. Done.

**10:32 AM.** You open a PR using the template. Fill in What/Why/How/Testing. CI runs: format check, lint, type-check, tests. Everything passes on the first try because everything that runs in CI already ran locally.

**10:45 AM.** A teammate reviews your PR. No formatting comments. No "missing import" comments. No "unused variable" comments. The linter already caught all of those. The review focuses on the actual logic: "Is this the right compression algorithm? Should we handle the case where the image is too large?"

**11:00 AM.** You squash merge. The commit message is clean and conventional. The feature ships.

**Meanwhile, in the background:** Renovate opens a PR to update three patch dependencies. CI passes. They auto-merge. You never think about it.

**Once a month:** You run knip and find two unused exports and one unused dependency. You remove them in a quick PR. The codebase stays lean.

**This is what a well-maintained codebase feels like.** Not glamorous. Not exciting. Just smooth, fast, friction-free development where engineers spend their time thinking about user problems instead of fighting tools.

The best codebase is the one you never think about. It just works. Every save formats. Every commit validates. Every dependency updates. Every PR gets the right reviewer. Every decision is documented.

Set it up once. Reap the benefits forever.

---

## QUICK REFERENCE

### Essential Keyboard Shortcuts Cheat Sheet

```
File Navigation:
  Cmd+P           → Open file (fuzzy search)
  Cmd+Shift+P     → Command palette
  Cmd+B           → Toggle sidebar
  Cmd+J           → Toggle terminal
  Cmd+\           → Split editor
  Ctrl+Tab        → Switch open files

Editing:
  Cmd+D           → Select next occurrence (multi-cursor)
  Cmd+Shift+L     → Select all occurrences
  Cmd+.           → Quick fix (THE most important shortcut)
  F2              → Rename symbol
  Cmd+/           → Toggle comment
  Cmd+Shift+K     → Delete line
  Option+Up/Down  → Move line

Navigation:
  F12             → Go to definition
  Shift+F12       → Find all references
  Ctrl+-          → Go back
  Cmd+Shift+F     → Global search
  Cmd+G           → Go to line
  Cmd+Shift+O     → Go to symbol in file
```

### Configuration Files Summary

| File | Purpose |
|------|---------|
| `eslint.config.js` | Linting rules (flat config) |
| `.prettierrc` | Code formatting rules |
| `.editorconfig` | Cross-editor basics |
| `.husky/pre-commit` | Pre-commit hook (runs lint-staged) |
| `.husky/commit-msg` | Commit message validation |
| `.lintstagedrc` | Defines what runs on staged files |
| `commitlint.config.js` | Commit message format rules |
| `.vscode/settings.json` | Workspace editor settings |
| `.vscode/extensions.json` | Recommended extensions |
| `.vscode/launch.json` | Debugger configurations |
| `.vscode/tasks.json` | Automated tasks |
| `renovate.json` | Dependency update automation |
| `knip.json` | Dead code detection config |
| `.github/CODEOWNERS` | Auto-assign PR reviewers |
| `.github/pull_request_template.md` | PR description template |

### The Pipeline

```
On Save:      Prettier → ESLint auto-fix → TypeScript (background)
On Commit:    lint-staged (Prettier → ESLint) → commitlint
On PR:        CI (format check → lint → typecheck → test)
Weekly:       Renovate PRs for dependency updates
Monthly:      knip for dead code detection
Quarterly:    Tech debt sprint
```
