<!--
  CHAPTER: 45
  TITLE: Release Management — Versioning, Changelogs & Ship Cadence
  PART: V — Deployment & Operations
  PREREQS: Chapters 7, 19
  KEY_TOPICS: semantic versioning, Changesets, release branches, staged rollouts, changelogs, release notes, app store versioning, OTA vs binary releases, release cadence, hotfix process, release trains, monorepo releases
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 45: Release Management — Versioning, Changelogs & Ship Cadence

> "Ship small, ship often. Every day you don't ship is a day your users don't benefit and your risk compounds." — Tobias Lutke, CEO of Shopify
>
> "Release management is not about the act of shipping. It's about shipping with confidence." — Every engineer who's ever rolled back a release at 2 AM

---

<details>
<summary><strong>TL;DR</strong></summary>

- Mobile apps have two version numbers: the user-facing version name ("2.4.1") and the store-facing build number (a monotonically increasing integer). Confusing them will get your store submission rejected.
- Use Changesets (`@changesets/cli`) in your monorepo to automate version bumps, changelog generation, and coordinated releases across packages. Every PR that changes user-facing behavior includes a changeset file. On merge, a bot opens a "Version Packages" PR. Merge that to cut a release.
- Adopt a regular release cadence (weekly or bi-weekly) and stick to it. Release trains create predictability for engineering, product, QA, and marketing. Ship small, ship often.
- Use staged rollouts for every mobile release: iOS phased release (1% to 100% over 7 days) and Android staged rollout (any percentage you choose). Monitor crash-free rate at each stage. Halt at the first sign of trouble.
- For hotfixes, use the OTA vs binary decision tree: JS-only changes go via EAS Update (minutes), native changes require a binary build and store review (hours/days). The `@expo/fingerprint` tool automates this decision in CI.
- Roll back instantly on web (Vercel), publish a previous OTA bundle for mobile JS issues, or halt a staged rollout for binary issues. Always have a rollback plan before you ship.

</details>

Every team ships code. Few teams ship code *well*. The difference between a team that deploys with swagger and a team that deploys with dread comes down to release management — the unglamorous, deeply important discipline of getting code from `main` to users' devices in a way that's predictable, reversible, and observable.

I've worked with teams that had brilliant engineers, beautiful codebases, and absolutely chaotic release processes. One team I consulted with had a shared Google Doc called "Release Steps" that was 47 bullet points long, half of them outdated, and required three people to be online simultaneously to execute. They released once a month because nobody wanted to be the person to kick off the process. By the time each release went out, it contained six weeks of changes (because the previous release was always delayed), which made each release riskier, which made the team more cautious, which made releases less frequent. A death spiral.

Another team I worked with shipped to production twice a day. Their mobile app had a weekly release train. Web deploys happened on every merge to main. Hotfixes went out in under an hour. Their crash-free rate was consistently above 99.8%. They weren't more talented than the first team. They just had better release management.

This chapter gives you everything you need to be the second team.

### In This Chapter
- Versioning for Mobile — the two numbers every mobile engineer must understand
- Changesets — automated versioning and changelog generation for monorepos
- Release Cadence — choosing and maintaining a release schedule
- Staged Rollouts — rolling out to 1% before you roll out to 100%
- Hotfix Process — when production is on fire and you need to ship NOW
- OTA vs Binary Releases — the decision that determines minutes vs days
- Release Notes — writing notes humans actually read
- Monorepo Release Coordination — versioning when mobile and web share packages
- Rollback Strategies — undoing a bad release on every platform
- The Complete Release Checklist — the pre-flight, flight, and post-flight of every release

### Related Chapters
- [Ch 7: Expo Platform] — EAS Build and EAS Update fundamentals
- [Ch 15: Monorepo Architecture] — the workspace structure releases are built from
- [Ch 18: CI/CD for Mobile & Web] — the pipeline that powers automated releases
- [Ch 19: OTA Updates] — deep dive on EAS Update
- [Ch 20: Monitoring & Observability] — what you watch after you ship

---

## 1. VERSIONING FOR MOBILE

If you come from web development, versioning feels simple. You deploy a new version to your server, and every user gets it immediately. There's one version in production at any given time. Done.

Mobile is different. When you submit a build to the App Store or Play Store, you're not replacing the old version. You're uploading a new artifact alongside it. Users have to choose to update (or have auto-update enabled). At any given moment, your user base is running dozens of different versions of your app. And each store has strict, unforgiving rules about version numbers.

Get these rules wrong and your submission gets rejected. Get them really wrong and you have to re-upload, wait in the review queue again, and explain to your PM why the release is delayed by two days.

### 1.1 The Two Numbers

Every mobile app has two version identifiers. Not one. Two. And they serve completely different purposes.

```
┌──────────────────────────────────────────────────────────────────┐
│  THE TWO VERSION NUMBERS                                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  VERSION NAME (User-Facing)                                  │ │
│  │                                                              │ │
│  │  "2.4.1"                                                     │ │
│  │                                                              │ │
│  │  - What users see in the App Store / Play Store              │ │
│  │  - Follows semantic versioning: MAJOR.MINOR.PATCH            │ │
│  │  - Human-meaningful: "2.4.1" tells you something             │ │
│  │  - iOS: CFBundleShortVersionString                           │ │
│  │  - Android: versionName                                      │ │
│  │  - Expo: "version" in app.json / app.config.ts               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  BUILD NUMBER (Store-Facing)                                  │ │
│  │                                                              │ │
│  │  147                                                          │ │
│  │                                                              │ │
│  │  - What the store uses to order builds                       │ │
│  │  - Monotonically increasing integer                          │ │
│  │  - NOT shown to users                                        │ │
│  │  - iOS: CFBundleVersion                                      │ │
│  │  - Android: versionCode                                      │ │
│  │  - Expo: ios.buildNumber / android.versionCode               │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

**Version Name** is the semantic version. It follows the `MAJOR.MINOR.PATCH` convention that you already know:
- **MAJOR** (2.x.x) — Breaking changes. Redesigned UI, removed features, anything that fundamentally changes the user experience. In practice, most consumer apps bump major for big launches or rebrands.
- **MINOR** (x.4.x) — New features, significant improvements. The stuff product managers write blog posts about.
- **PATCH** (x.x.1) — Bug fixes, performance improvements, minor tweaks. The stuff nobody notices unless it was broken before.

**Build Number** is a monotonically increasing integer. It's the store's way of knowing which binary is newer. If you upload build 147 today and try to upload build 146 tomorrow, the store will reject it. Period.

### 1.2 Platform-Specific Rules

Here's where it gets tricky, because iOS and Android have subtly different rules:

**iOS (App Store Connect):**

```
CFBundleShortVersionString = "2.4.1"    // Version name
CFBundleVersion = "147"                  // Build number (string, not int!)

Rules:
- CFBundleVersion must increase for each TestFlight / App Store upload
- CFBundleVersion is per-version: you CAN reset it when you bump
  CFBundleShortVersionString
  (e.g., version "2.4.1" build "5" → version "2.5.0" build "1")
- CFBundleVersion can contain up to 3 dot-separated integers: "1.2.3"
  But almost everyone uses a single integer. Keep it simple.
- Both are strings in Info.plist (even though build number is conceptually numeric)
```

**Android (Google Play Console):**

```
versionName = "2.4.1"     // Version name (arbitrary string, for display only)
versionCode = 147          // Build number (32-bit integer, max: 2,147,483,647)

Rules:
- versionCode must STRICTLY INCREASE across ALL uploads. Ever. Forever.
  You cannot reset it. If your last upload was versionCode 147,
  your next upload must be >= 148.
- versionCode is global across your app — not per-version, not per-track.
- versionName is purely for display. Google Play doesn't validate it at all.
  You could set it to "banana" and Play would accept it.
  (Don't do that.)
- If you use APK splits or App Bundles with per-ABI outputs, each split
  needs a unique versionCode. Teams often multiply by 10 and add an ABI offset.
```

### 1.3 Semantic Versioning in app.config.ts

In an Expo project, you configure both numbers in your app config:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: 'MyApp',
  slug: 'my-app',

  // Version name — user-facing, follows semver
  version: '2.4.1',

  ios: {
    bundleIdentifier: 'com.mycompany.myapp',
    // Build number — per TestFlight/App Store upload
    buildNumber: '147',
  },

  android: {
    package: 'com.mycompany.myapp',
    // Build number — must increase forever, never resets
    versionCode: 147,
  },
});
```

### 1.4 Auto-Incrementing Build Numbers with EAS

Manually managing build numbers is a recipe for mistakes. Someone forgets to bump it, the store rejects the upload, and now your release is delayed. EAS Build solves this with automatic incrementing:

```json
// eas.json
{
  "build": {
    "production": {
      "autoIncrement": true,
      "ios": {
        "buildNumber": "auto"
      },
      "android": {
        "versionCode": "auto"
      }
    },
    "preview": {
      "autoIncrement": true
    }
  }
}
```

When `autoIncrement` is `true`, EAS queries the App Store Connect and Google Play Console APIs to find the latest build number and increments it. You never think about it. You never get a store rejection because of it. This is one of those things where the automation isn't a nice-to-have — it's the only sane approach.

**But there's a nuance.** If you're running multiple builds in parallel (say, building for production and preview simultaneously), auto-increment can race. The solution:

```typescript
// app.config.ts — for teams running parallel builds
import { ExpoConfig, ConfigContext } from 'expo/config';

const IS_CI = process.env.CI === 'true';
const BUILD_NUMBER = process.env.BUILD_NUMBER; // Set by CI

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  version: '2.4.1',

  ios: {
    bundleIdentifier: 'com.mycompany.myapp',
    // In CI, use the explicitly passed build number
    // Locally, let EAS auto-increment
    buildNumber: IS_CI && BUILD_NUMBER ? BUILD_NUMBER : undefined,
  },

  android: {
    package: 'com.mycompany.myapp',
    versionCode: IS_CI && BUILD_NUMBER ? parseInt(BUILD_NUMBER, 10) : undefined,
  },
});
```

### 1.5 Computing Build Numbers from Git

A popular pattern is to derive the build number from something deterministic, like the number of commits on `main`:

```bash
# Count commits on main — gives you a monotonically increasing integer
git rev-list --count main
# Output: 1847

# Or use the CI run number (GitHub Actions)
echo $GITHUB_RUN_NUMBER
# Output: 523
```

```yaml
# .github/workflows/build.yml
jobs:
  build-mobile:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for rev-list --count

      - name: Compute build number
        id: version
        run: |
          BUILD_NUM=$(git rev-list --count HEAD)
          echo "build_number=$BUILD_NUM" >> $GITHUB_OUTPUT
          echo "Build number: $BUILD_NUM"

      - name: Build with EAS
        run: |
          eas build --platform all --non-interactive
        env:
          BUILD_NUMBER: ${{ steps.version.outputs.build_number }}
```

I recommend the `git rev-list --count` approach over `GITHUB_RUN_NUMBER` because it's deterministic — any engineer on any machine can compute the same number. `GITHUB_RUN_NUMBER` only exists in GitHub Actions and resets if you recreate the workflow.

### 1.6 Version Number Anti-Patterns

Let me save you from mistakes I've seen teams make:

**Anti-pattern 1: Manual build number management.** "Everyone, the current build number is 147. If you're building, use 148." This falls apart the moment two people build simultaneously. Use EAS auto-increment or git-derived numbers.

**Anti-pattern 2: Using dates as build numbers.** "20260407" as a build number seems clever until you need to ship two builds in one day and realize 20260407 is already taken. Also, it wastes your 32-bit integer space on Android.

**Anti-pattern 3: Forgetting that Android versionCode never resets.** Teams migrating from one build system to another sometimes start their new versionCode at 1. The Play Store immediately rejects it because the previous build was 847. Always query the current maximum before setting up a new build system.

**Anti-pattern 4: Putting the build number in the version name.** "2.4.1-147" as the user-facing version is noise. Users don't care about build 147. Keep the version name clean for users and the build number for the store.

---

## 2. CHANGESETS — AUTOMATED VERSIONING FOR MONOREPOS

If you're working in a monorepo (and if you've read Chapter 15, you should be), you need a system for tracking which packages changed, what kind of change it was, and when to cut a release. Doing this manually is a fool's errand. I've watched a team lead spend 30 minutes before every release going through merged PRs trying to figure out what changed and whether it was a major, minor, or patch bump. That's 30 minutes of a senior engineer's time, every week, doing something a tool should do.

Enter Changesets.

### 2.1 What Changesets Is

[Changesets](https://github.com/changesets/changesets) is a tool originally created at Atlassian for managing versioning and changelogs in multi-package repositories. It's now the de facto standard for monorepo versioning in the JavaScript ecosystem. Turborepo recommends it. pnpm recommends it. If you're running a monorepo, you should be using it.

The core idea is simple:

1. When a developer makes a change that affects a package's public API or behavior, they add a **changeset file** to their PR — a small markdown file that describes what changed and whether it's a major, minor, or patch change.
2. On merge to main, a **Changesets GitHub Action** (or bot) collects all pending changeset files and opens a "Version Packages" PR that bumps versions and updates changelogs.
3. When you merge the "Version Packages" PR, that's your release. Versions are bumped, changelogs are generated, and packages are published.

No more guessing. No more "wait, did we bump the version?" No more release meetings to decide what version number to use. The process is mechanical, deterministic, and automated.

### 2.2 Setting Up Changesets

```bash
# Install Changesets CLI
pnpm add -D -w @changesets/cli

# Initialize — creates a .changeset directory at the root
pnpm changeset init
```

This creates a `.changeset` directory with a `config.json` file:

```json
// .changeset/config.json
{
  "$schema": "https://unpkg.com/@changesets/config@3.1.1/schema.json",
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

Let me walk you through each option because the defaults are not optimal for most teams:

```json
// .changeset/config.json — production configuration
{
  "$schema": "https://unpkg.com/@changesets/config@3.1.1/schema.json",

  // Use the GitHub changelog format — it links PRs and authors
  "changelog": [
    "@changesets/changelog-github",
    { "repo": "your-org/your-repo" }
  ],

  // Don't auto-commit version bumps — let the bot open a PR instead
  "commit": false,

  // "fixed" groups — packages that should ALWAYS have the same version.
  // If any package in a fixed group bumps, they ALL bump.
  // Great for your mobile app + its config packages.
  "fixed": [
    ["@repo/mobile", "@repo/mobile-config"]
  ],

  // "linked" groups — packages whose versions are linked.
  // If one gets a major bump, they all get a major bump.
  // But they can have different patch/minor versions.
  "linked": [
    ["@repo/ui", "@repo/ui-native", "@repo/ui-web"]
  ],

  // "restricted" for private packages, "public" for npm-published packages
  "access": "restricted",

  // The branch Changesets considers "the base"
  "baseBranch": "main",

  // When a dependency gets a patch bump, bump dependents too
  "updateInternalDependencies": "patch",

  // Packages to exclude from Changesets management
  "ignore": [
    "@repo/eslint-config",
    "@repo/typescript-config"
  ]
}
```

Install the GitHub changelog plugin:

```bash
pnpm add -D -w @changesets/changelog-github
```

### 2.3 The Developer Workflow

Here's what the day-to-day flow looks like for every developer on your team:

**Step 1: Make your changes.**

```bash
# You've just added a new Button variant to the UI package
git checkout -b feat/outline-button
# ... make changes ...
git add .
```

**Step 2: Add a changeset.**

```bash
pnpm changeset
```

This launches an interactive CLI:

```
🦋  Which packages would you like to include? 
◯ @repo/mobile
◉ @repo/ui
◯ @repo/web
◯ @repo/api-client

🦋  Which packages should have a major bump? 
◯ @repo/ui

🦋  Which packages should have a minor bump? 
◉ @repo/ui

🦋  Please enter a summary for this change (this will be in the changelog):
🦋  Added outline variant to Button component

🦋  === Summary of changesets ===
🦋  minor: @repo/ui

🦋  Is this your desired changeset? (Y/n) Y
🦋  Changeset added! - you can now commit it
```

This creates a file like `.changeset/brave-tigers-dance.md`:

```markdown
---
'@repo/ui': minor
---

Added outline variant to Button component
```

That's it. The changeset file is a tiny markdown file with YAML frontmatter. It specifies which packages changed and what type of bump each needs. The body is the changelog entry.

**Step 3: Commit the changeset with your code.**

```bash
git add .changeset/brave-tigers-dance.md
git commit -m "feat(ui): add outline variant to Button"
git push origin feat/outline-button
```

**Step 4: Open a PR.** The changeset file goes with the code change. Reviewers can see exactly what version bump this PR will cause and what the changelog entry will say.

### 2.4 The Changeset Bot

Install the [Changesets GitHub bot](https://github.com/apps/changeset-bot) on your repository. It does two things:

1. **On every PR**, it comments whether the PR includes a changeset. If it doesn't, it reminds the author to add one (or confirms that no changeset is needed for changes that don't affect package versions — like README updates or CI config changes).

2. **When changeset files accumulate on main**, the bot (or a GitHub Action — I recommend the action) opens a "Version Packages" PR that:
   - Removes all pending changeset files
   - Bumps package versions according to the changesets
   - Updates CHANGELOG.md files for each affected package
   - Shows a clear diff of all version changes

### 2.5 The Release GitHub Action

Here's the complete CI workflow that powers the automated release process:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false  # Never cancel a release in progress

jobs:
  release:
    name: Version & Release
    runs-on: ubuntu-latest
    permissions:
      contents: write        # To create tags and releases
      pull-requests: write   # To create the "Version Packages" PR
      id-token: write        # For npm provenance (optional)

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Create Release PR or Publish
        id: changesets
        uses: changesets/action@v1
        with:
          # Command to run when there ARE pending changesets
          # (creates/updates the "Version Packages" PR)
          version: pnpm changeset version

          # Command to run when the "Version Packages" PR is merged
          # (publishes packages, creates git tags, creates GitHub releases)
          publish: pnpm run release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}  # Only if publishing to npm

      # After packages are versioned, trigger mobile builds if needed
      - name: Trigger Mobile Build
        if: steps.changesets.outputs.published == 'true'
        run: |
          # Check if the mobile app version changed
          MOBILE_CHANGED=$(echo '${{ steps.changesets.outputs.publishedPackages }}' \
            | jq -r '.[] | select(.name == "@repo/mobile") | .name')

          if [ -n "$MOBILE_CHANGED" ]; then
            echo "Mobile app version bumped — triggering EAS Build"
            cd apps/mobile
            eas build --platform all --profile production --non-interactive
          fi
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
```

And the `release` script in your root `package.json`:

```json
// package.json (root)
{
  "scripts": {
    "release": "changeset publish",
    "version-packages": "changeset version"
  }
}
```

### 2.6 Automated Changelog Generation

When you merge the "Version Packages" PR, Changesets generates changelogs like this:

```markdown
<!-- packages/ui/CHANGELOG.md -->

# @repo/ui

## 2.5.0

### Minor Changes

- [#234](https://github.com/your-org/your-repo/pull/234)
  [`a1b2c3d`](https://github.com/your-org/your-repo/commit/a1b2c3d)
  Thanks [@developer-name](https://github.com/developer-name)! —
  Added outline variant to Button component

- [#228](https://github.com/your-org/your-repo/pull/228)
  [`d4e5f6a`](https://github.com/your-org/your-repo/commit/d4e5f6a)
  Thanks [@other-dev](https://github.com/other-dev)! —
  Added size="xl" option to all form components

### Patch Changes

- [#231](https://github.com/your-org/your-repo/pull/231)
  [`b7c8d9e`](https://github.com/your-org/your-repo/commit/b7c8d9e)
  Thanks [@developer-name](https://github.com/developer-name)! —
  Fixed focus ring not appearing on Button in Safari

## 2.4.2

### Patch Changes
...
```

Every entry links to the PR, the commit, and the author. It's generated from the changeset files, so it's always accurate. No more "what changed in this release?" meetings. No more scrolling through git log trying to reconstruct a changelog. The changelog writes itself.

### 2.7 Enforcing Changesets in CI

You want to require changesets on PRs that change package code but not on PRs that only touch CI config, docs, or tooling. Here's a GitHub Action that does that:

```yaml
# .github/workflows/changeset-check.yml
name: Changeset Check

on:
  pull_request:
    branches: [main]

jobs:
  changeset-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      - name: Check for changeset
        run: |
          # Get list of changed files
          CHANGED_FILES=$(git diff --name-only origin/main...HEAD)

          # Check if any source files changed (not just config/docs)
          SOURCE_CHANGED=$(echo "$CHANGED_FILES" | grep -E '^(apps|packages)/.*\.(ts|tsx|js|jsx)$' || true)

          if [ -z "$SOURCE_CHANGED" ]; then
            echo "No source files changed — changeset not required"
            exit 0
          fi

          # Source files changed — check for changeset
          CHANGESET_FILES=$(echo "$CHANGED_FILES" | grep -E '^\.changeset/.*\.md$' || true)

          if [ -z "$CHANGESET_FILES" ]; then
            echo "::error::Source files changed but no changeset found."
            echo "::error::Run 'pnpm changeset' to add one, or add an empty changeset"
            echo "::error::with 'pnpm changeset --empty' if this change doesn't need a version bump."
            exit 1
          fi

          echo "Changeset found — all good"
```

For changes that genuinely don't need a version bump (refactors, internal tooling), developers can add an empty changeset:

```bash
pnpm changeset --empty
```

This creates a changeset file with no version bumps, signaling "I considered whether this needs a changeset and it doesn't."

---

## 3. RELEASE CADENCE

How often should you release? This question generates surprisingly heated debates. Some teams swear by continuous deployment. Others insist on monthly releases. The right answer depends on your team, your app, and your users — but I have strong opinions about the direction you should lean.

### 3.1 The Case for Shipping Often

Every release you ship contains some amount of risk. The risk is proportional to the number of changes in the release. Here's the math:

```
┌──────────────────────────────────────────────────────────────────┐
│  RELEASE FREQUENCY vs. RISK                                       │
│                                                                   │
│  Risk per release                                                 │
│  ▲                                                                │
│  │                                                                │
│  │  ●                                                             │
│  │     ●  Monthly releases:                                       │
│  │        ● 100+ changes per release                              │
│  │           ● High risk, hard to debug                           │
│  │              ●                                                 │
│  │                 ●  Bi-weekly releases:                         │
│  │                    ● 30-50 changes per release                 │
│  │                       ●                                        │
│  │                          ●  Weekly releases:                   │
│  │                             ● 10-20 changes per release       │
│  │                                ●  ●  ●  ●  ●  Low risk       │
│  ├──────────────────────────────────────────────────────────── ▶ │
│  1/mo  2/mo     1/wk     2/wk        Daily    Ship frequency     │
│                                                                   │
│  KEY INSIGHT:                                                     │
│  - 1 release with 100 changes ≠ 10 releases with 10 changes     │
│  - The big release is exponentially riskier because changes       │
│    interact in unpredictable ways                                 │
│  - Small releases are easier to debug, easier to roll back,      │
│    easier to bisect                                               │
└──────────────────────────────────────────────────────────────────┘
```

The 2023 DORA report (State of DevOps) found that elite-performing teams deploy on demand, multiple times per day. High performers deploy between once per week and once per month. Low performers deploy between once per month and once every six months. This isn't correlation — it's causation in both directions. Frequent releases make teams better, and better teams release more frequently.

### 3.2 Web vs Mobile: Different Cadences

For **web** apps deployed on Vercel (or any modern hosting), the answer is simple: **ship on every merge to main.** Every PR that passes CI and gets merged triggers a production deployment. No release trains, no batching, no waiting. If someone merges a fix at 3 PM, it's live at 3:01 PM. This is the gold standard, and modern web infrastructure makes it trivially achievable.

For **mobile** apps, it's more nuanced because:
1. App Store review takes time (usually 24-48 hours, sometimes longer)
2. Users have to update (or wait for auto-update)
3. You can't instantly roll back a binary release
4. OTA updates (EAS Update) handle JS changes but not native changes

Most mature mobile teams land on one of these cadences:

**Weekly Release Train (Recommended for most teams)**

```
┌──────────────────────────────────────────────────────────────────┐
│  WEEKLY RELEASE TRAIN                                             │
│                                                                   │
│  Monday      Code freeze for this week's release                 │
│              Branch cut: release/2.5.0                            │
│              QA begins on release branch                          │
│                                                                   │
│  Tuesday     QA continues, bug fixes cherry-picked to release    │
│              branch. New feature work continues on main.          │
│                                                                   │
│  Wednesday   Release candidate built and submitted to stores     │
│              App Store review begins                              │
│                                                                   │
│  Thursday    App Store review completes (usually)                │
│              Staged rollout begins: 10%                           │
│                                                                   │
│  Friday      Monitor crash-free rate at 10%                      │
│              If healthy: bump to 25%                              │
│              If unhealthy: halt rollout, hotfix                   │
│                                                                   │
│  Weekend     Staged rollout continues automatically               │
│              On-call engineer monitors                            │
│                                                                   │
│  Next Mon    Full rollout (100%) if all metrics healthy           │
│              Next release cycle begins                            │
└──────────────────────────────────────────────────────────────────┘
```

**Bi-Weekly Release Train (For smaller teams or regulated industries)**

Same structure, stretched across two weeks. QA has more time, the rollout is more gradual, and there's less pressure. The trade-off is that features take longer to reach users and releases contain more changes (more risk per release).

**Continuous + Weekly Hybrid (For teams with OTA)**

If you're using EAS Update (and you should be), you can get the best of both worlds:

- **JS-only changes** ship via OTA continuously — even multiple times per day
- **Native changes** follow the weekly release train
- The vast majority of your changes are JS-only, so most of your work ships continuously

This is the approach I recommend for teams that have OTA set up. It gives you web-like deployment speed for most changes while maintaining the discipline of a release train for native changes.

### 3.3 Cutting a Release Branch

When it's time to cut a release, here's the process:

```bash
# 1. Create the release branch from main
git checkout main
git pull origin main
git checkout -b release/2.5.0

# 2. Bump version (if not using Changesets for this)
# With Changesets, this is already done in the "Version Packages" PR
# Without Changesets:
npm version minor --no-git-tag-version
# Update app.config.ts version field

# 3. Push the release branch
git push origin release/2.5.0

# 4. CI triggers a build from the release branch
# EAS Build picks up the new version from app.config.ts
```

**Important:** After cutting the release branch, main continues to receive new feature PRs. The release branch only receives cherry-picked bug fixes. This is how you avoid the "code freeze means everyone stops working" problem.

```
┌──────────────────────────────────────────────────────────────────┐
│  BRANCHING MODEL                                                  │
│                                                                   │
│  main ──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──→         │
│              │              ↑ cherry-pick                         │
│              └── release/2.5.0 ──●──●──●──→ tag v2.5.0          │
│                                                                   │
│  Features merge to main continuously.                             │
│  Only bug fixes are cherry-picked to the release branch.         │
│  When the release ships, tag it and delete the branch.           │
└──────────────────────────────────────────────────────────────────┘
```

### 3.4 The Changesets Approach to Release Branches

If you're using Changesets (and you should be), the release branch workflow integrates naturally:

```bash
# Option A: Cut release from the "Version Packages" PR
# 1. Merge the "Version Packages" PR (which bumps versions and updates changelogs)
# 2. Create a release branch from that commit
# 3. Build and submit from the release branch

# Option B: Use Changesets snapshot releases for pre-release
pnpm changeset version --snapshot preview
# This creates versions like "2.5.0-preview-20260407"
# Useful for TestFlight / internal testing builds
```

Most teams use Option A. The "Version Packages" PR acts as the release trigger. When the team lead merges it, that's the signal that this batch of changes is ready to ship.

---

## 4. STAGED ROLLOUTS

Shipping to 100% of your users on day one is gambling. You're betting that your QA process caught every bug, that no edge case exists in the wild that you didn't test for, and that nothing in the OS/device/network landscape has changed since you last tested. That's a lot of bets.

Staged rollouts let you hedge. You ship to a small percentage of users first, monitor for crashes and issues, and gradually increase the percentage. If something goes wrong, you've affected 1% of users instead of 100%.

### 4.1 iOS Phased Release

Apple calls this "Phased Release" and it follows a fixed schedule:

```
┌──────────────────────────────────────────────────────────────────┐
│  iOS PHASED RELEASE SCHEDULE                                      │
│                                                                   │
│  Day 1:    1%  of users get the update                           │
│  Day 2:    2%                                                     │
│  Day 3:    5%                                                     │
│  Day 4:   10%                                                     │
│  Day 5:   20%                                                     │
│  Day 6:   50%                                                     │
│  Day 7:  100%                                                     │
│                                                                   │
│  Total: 7 days from start to full rollout                        │
│                                                                   │
│  IMPORTANT CAVEATS:                                               │
│  - Users who manually check for updates bypass phased release    │
│  - Users who have auto-update enabled are included in the rollout│
│  - You can PAUSE a phased release at any time                    │
│  - You can RESUME a paused release (continues from where it was) │
│  - You can RELEASE TO ALL immediately at any time                │
│  - You CANNOT reduce the percentage (only pause or release all)  │
│  - The 7-day schedule is fixed — you can't customize percentages │
│  - Phased release only applies to AUTO-updates, not manual ones  │
└──────────────────────────────────────────────────────────────────┘
```

To enable phased release, you select it in App Store Connect when submitting your build. There's no code change needed — it's a store-level feature.

**The limitation:** You can't choose custom percentages on iOS. It's Apple's fixed 7-day schedule or nothing. This is less flexible than Android, but the 7-day ramp is actually well-designed for most apps.

### 4.2 Android Staged Rollout

Google Play gives you much more control:

```
┌──────────────────────────────────────────────────────────────────┐
│  ANDROID STAGED ROLLOUT                                           │
│                                                                   │
│  You choose the percentage. Any value from 0.1% to 100%.        │
│                                                                   │
│  Recommended ramp schedule:                                       │
│                                                                   │
│  Stage 1:    1%   — Canary. Watch for crash spikes.              │
│              Wait 24 hours. Check crash-free rate.               │
│                                                                   │
│  Stage 2:    5%   — Still small. Broader device coverage.        │
│              Wait 24 hours. Check crash-free + ANR rate.         │
│                                                                   │
│  Stage 3:   20%   — Significant. You'll see edge cases now.     │
│              Wait 24-48 hours. Check all metrics.                │
│                                                                   │
│  Stage 4:   50%   — Half your users. If it's fine here, it's    │
│              probably fine everywhere.                            │
│              Wait 24 hours.                                       │
│                                                                   │
│  Stage 5:  100%   — Full rollout.                                │
│                                                                   │
│  At ANY stage you can:                                            │
│  - HALT the rollout (users who got it keep it)                   │
│  - INCREASE the percentage                                       │
│  - Jump to 100%                                                   │
│  - You CANNOT decrease the percentage (only halt)                │
└──────────────────────────────────────────────────────────────────┘
```

### 4.3 Automating Staged Rollout with Fastlane

You can automate the staged rollout progression with Fastlane or the store APIs:

```ruby
# fastlane/Fastfile

platform :android do
  lane :rollout_increase do |options|
    percentage = options[:percentage] || 10

    supply(
      track: 'production',
      rollout: (percentage.to_f / 100).to_s,
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )

    # Notify the team
    slack(
      message: "Android rollout increased to #{percentage}%",
      channel: "#releases",
    )
  end

  lane :rollout_halt do
    supply(
      track: 'production',
      rollout: '0',  # Halts the rollout
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )

    slack(
      message: ":rotating_light: Android rollout HALTED",
      channel: "#releases",
      default_payloads: [:git_author],
    )
  end
end
```

Or with the EAS CLI and the app store APIs directly:

```bash
# Increase Android rollout to 20%
eas submit --platform android --latest --rollout-percentage 20

# Check current rollout status
# (You'll need the Google Play Developer API for this)
```

### 4.4 Monitoring at Each Stage

A staged rollout without monitoring is just a slower way to ship a bug to everyone. At each stage, you should be watching:

```typescript
// What to monitor at each rollout stage

interface RolloutStageChecklist {
  // Must-have metrics
  crashFreeRate: number;        // Target: > 99.5%
  anrRate: number;              // Target: < 0.5% (Android)
  errorRate: number;            // Target: no new error spikes

  // Should-have metrics
  appStartTime: number;         // Target: no regression from previous version
  apiErrorRate: number;         // Target: no increase
  memoryUsage: number;          // Target: no significant increase
  batteryDrain: number;         // Target: no significant increase

  // Nice-to-have metrics
  userRetention: number;        // Target: no drop
  featureAdoption: number;      // Target: new feature usage as expected
  appStoreRating: number;       // Target: no rating drop
}
```

**The halt decision:** If your crash-free rate drops below 99.3% at any stage, halt the rollout immediately. Don't wait. Don't "give it a few more hours to see if it stabilizes." Halt, investigate, and fix. The cost of halting a healthy release is zero (you just resume later). The cost of not halting a broken release is potentially thousands of affected users.

### 4.5 The Staged Rollout CI Workflow

Here's a GitHub Action that automates the staged rollout progression with monitoring gates:

```yaml
# .github/workflows/staged-rollout.yml
name: Staged Rollout

on:
  workflow_dispatch:
    inputs:
      platform:
        description: 'Platform (android/ios)'
        required: true
        type: choice
        options:
          - android
          - ios
      action:
        description: 'Action'
        required: true
        type: choice
        options:
          - increase
          - halt
          - release-to-all
      percentage:
        description: 'Rollout percentage (for increase action)'
        required: false
        type: number

jobs:
  rollout:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check metrics before increasing
        if: inputs.action == 'increase'
        run: |
          # Query Sentry for crash-free rate
          CRASH_FREE=$(curl -s \
            -H "Authorization: Bearer ${{ secrets.SENTRY_AUTH_TOKEN }}" \
            "https://sentry.io/api/0/projects/$ORG/$PROJECT/stats/" \
            | jq '.crashFreeRate')

          echo "Current crash-free rate: $CRASH_FREE%"

          # Fail if crash-free rate is below threshold
          if (( $(echo "$CRASH_FREE < 99.3" | bc -l) )); then
            echo "::error::Crash-free rate ($CRASH_FREE%) is below 99.3%"
            echo "::error::Cannot increase rollout. Investigate crashes first."
            exit 1
          fi

      - name: Execute rollout action
        run: |
          case "${{ inputs.action }}" in
            increase)
              echo "Increasing ${{ inputs.platform }} rollout to ${{ inputs.percentage }}%"
              # Execute via Fastlane or store API
              ;;
            halt)
              echo "HALTING ${{ inputs.platform }} rollout"
              # Execute via Fastlane or store API
              ;;
            release-to-all)
              echo "Releasing to 100% on ${{ inputs.platform }}"
              # Execute via Fastlane or store API
              ;;
          esac

      - name: Notify team
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_WEBHOOK }}
          webhook-type: incoming-webhook
          payload: |
            {
              "text": "Rollout ${{ inputs.action }}: ${{ inputs.platform }} → ${{ inputs.percentage || '100' }}%"
            }
```

---

## 5. HOTFIX PROCESS

It's 2 AM. Your phone buzzes. Sentry is firing alerts. The crash-free rate just dropped from 99.7% to 97.1%. A critical flow in your app is broken. Users are complaining on Twitter. Your PM is awake and asking for an ETA.

This is the moment your release management process is tested. Not when everything is going well. Not when you have time to plan. Right now, when the building is on fire and you need to ship a fix as fast as humanly possible.

The first 60 seconds matter. Here's the playbook:

### 5.1 The Hotfix Decision Tree

```
┌──────────────────────────────────────────────────────────────────┐
│  HOTFIX DECISION TREE                                             │
│                                                                   │
│  Production crash detected                                        │
│  │                                                                │
│  ├─ Is it a JS-only change that fixes it?                        │
│  │  (UI bug, logic error, wrong API URL, missing null check)     │
│  │  │                                                             │
│  │  ├─ YES → OTA UPDATE (EAS Update)                             │
│  │  │        Time to fix: 10-30 minutes                          │
│  │  │        Deploy: immediate (no store review)                 │
│  │  │        Risk: low (can roll back instantly)                  │
│  │  │                                                             │
│  │  └─ NO → Does it require a native code change?               │
│  │     (new native module, native config, asset change)          │
│  │     │                                                          │
│  │     ├─ YES → BINARY RELEASE (EAS Build + Store Submit)        │
│  │     │        Time to fix: 1-4 hours (build) + 24-48h (review)│
│  │     │        Deploy: after store approval                     │
│  │     │        Risk: medium (can't roll back binary easily)     │
│  │     │                                                          │
│  │     └─ Can you WORK AROUND it with a JS change?              │
│  │        (feature flag, disable broken feature, fallback)       │
│  │        │                                                       │
│  │        ├─ YES → OTA UPDATE with workaround                   │
│  │        │        Then ship proper fix in next regular release  │
│  │        │                                                       │
│  │        └─ NO → BINARY RELEASE (no choice)                    │
│  │                Consider: halt staged rollout if in progress   │
│  │                Consider: expedited review (Apple allows this) │
│  │                                                                │
│  └─ Is the issue affecting all users or a subset?               │
│     │                                                             │
│     ├─ ALL USERS → Immediate action required                    │
│     │                                                             │
│     └─ SUBSET → Can you use a feature flag to disable for all   │
│        while you fix? (Remote config / LaunchDarkly / Statsig)  │
│        │                                                          │
│        ├─ YES → Disable via feature flag (seconds)              │
│        │        Then ship proper fix                             │
│        │                                                          │
│        └─ NO → Fix via OTA or binary as above                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 OTA Hotfix (Minutes)

When the fix is JS-only, EAS Update is your best friend. Here's the step-by-step:

```bash
# 1. Identify the issue (look at Sentry, reproduce if possible)

# 2. Create a hotfix branch from the release tag
git checkout -b hotfix/crash-on-checkout v2.5.0
# Or from the release branch if it still exists:
git checkout -b hotfix/crash-on-checkout release/2.5.0

# 3. Make the fix
# ... edit the code ...

# 4. Test locally (as much as you can at 2 AM)
pnpm test -- --related
# Quick smoke test on a device

# 5. Publish the OTA update
eas update --branch production --message "Fix: crash on checkout screen"

# That's it. Users on the production channel will get the update
# the next time they open the app (or within minutes if the app
# is configured for eager updates).
```

The key insight: EAS Update deploys a new JS bundle to a **branch** (also called a channel). All users on that branch get the update. There's no store review. No waiting. The fix is live in minutes.

```typescript
// Configuring eager updates in your app
// This ensures users get the hotfix as soon as possible

import * as Updates from 'expo-updates';
import { useEffect } from 'react';

function useEagerUpdates() {
  useEffect(() => {
    async function checkForUpdate() {
      if (__DEV__) return;

      try {
        const update = await Updates.checkForUpdateAsync();
        if (update.isAvailable) {
          await Updates.fetchUpdateAsync();
          // Optionally restart immediately for critical fixes
          // await Updates.reloadAsync();

          // Or show a gentle prompt
          Alert.alert(
            'Update Available',
            'A new version is ready. Restart to apply?',
            [
              { text: 'Later', style: 'cancel' },
              { text: 'Restart', onPress: () => Updates.reloadAsync() },
            ]
          );
        }
      } catch (e) {
        // Update check failed — not critical, don't crash the app
        console.warn('Update check failed:', e);
      }
    }

    // Check for updates on app foreground
    checkForUpdate();
  }, []);
}
```

### 5.3 Binary Hotfix (Hours/Days)

When you need a native change, the path is longer:

```bash
# 1. Create hotfix branch (same as above)
git checkout -b hotfix/native-crash v2.5.0

# 2. Make the fix

# 3. Bump the patch version
# In app.config.ts: version "2.5.0" → "2.5.1"
# Build number will auto-increment via EAS

# 4. Build the binary
eas build --platform all --profile production --non-interactive

# 5. Submit to stores
eas submit --platform all --latest

# 6. Wait for review...
# iOS: Request expedited review if critical
# https://developer.apple.com/contact/app-store/?topic=expedite

# 7. Once approved, start staged rollout
# Start at higher percentage than usual for hotfixes (20-50%)
# because the urgency outweighs the caution
```

**Apple Expedited Review:** For critical issues (crashes, security vulnerabilities, legal compliance), Apple offers expedited review. Go to https://developer.apple.com/contact/app-store/?topic=expedite and explain the issue. In my experience, expedited reviews are approved about 80% of the time for legitimate crash fixes and typically complete within 2-6 hours instead of 24-48 hours. Don't abuse this — Apple tracks how often you request it, and frivolous requests reduce your credibility.

### 5.4 The Hotfix Runbook

Every team should have a written hotfix runbook. Here's a template:

```
┌──────────────────────────────────────────────────────────────────┐
│  HOTFIX RUNBOOK                                                   │
│                                                                   │
│  STEP 1: ASSESS (5 minutes)                                      │
│  □ What is the impact? (crash rate, users affected, revenue)     │
│  □ Is it all users or a subset? (platform, version, region)      │
│  □ Is there an existing workaround? (feature flag, server-side)  │
│  □ Who owns this code? (check git blame, notify them)            │
│                                                                   │
│  STEP 2: DECIDE (2 minutes)                                      │
│  □ OTA or binary? (use the decision tree above)                  │
│  □ Should we halt the staged rollout? (if one is in progress)    │
│  □ Should we disable the feature via feature flag? (if possible) │
│  □ Who is fixing it? (the code owner, or whoever is available)   │
│                                                                   │
│  STEP 3: FIX (15-60 minutes)                                     │
│  □ Branch from the release tag, not from main                    │
│  □ Minimal fix only — don't bundle other changes                 │
│  □ Write a test that reproduces the issue                        │
│  □ Get a review (even at 2 AM — ping someone)                   │
│                                                                   │
│  STEP 4: SHIP (5 min for OTA, hours for binary)                 │
│  □ Run CI checks (even in a hurry, don't skip tests)            │
│  □ Deploy: eas update (OTA) or eas build + eas submit (binary)  │
│  □ Announce in #releases channel with: what broke, what fixed,   │
│    how many users affected, ETA for full resolution              │
│                                                                   │
│  STEP 5: FOLLOW UP (next business day)                           │
│  □ Cherry-pick the fix to main (if branched from release tag)    │
│  □ Write a postmortem (even short ones have value)               │
│  □ Add monitoring for the failure mode (alert, not just log)     │
│  □ Discuss in retro: could we have caught this before release?   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. OTA VS BINARY RELEASES

This is one of the most consequential decisions in mobile release management: when can you ship an over-the-air (OTA) JavaScript update, and when do you need to build and submit a new binary to the stores? Get it right, and you have web-like deployment speed for most of your changes. Get it wrong, and you ship a broken OTA update that crashes the app because the JS bundle references a native module that doesn't exist in the installed binary.

### 6.1 The Fundamental Rule

The rule is simple in principle:

- **JS-only changes → OTA.** If your change only touches JavaScript/TypeScript code, React components, styles, images bundled in JS, or any other asset that the Metro bundler processes, it can go OTA.
- **Native changes → Binary.** If your change touches anything in `ios/` or `android/`, adds/removes/updates a native module, changes `app.json`/`app.config.ts` in ways that affect the native build (permissions, schemes, plugins), or updates a dependency that includes native code, you need a new binary.

The problem is that in a real codebase, it's not always obvious which category a change falls into. Does updating `react-native-reanimated` from 3.8.0 to 3.8.1 require a binary? (It depends — patch versions *usually* don't change native code, but sometimes they do.) Does adding a new `expo-*` package require a binary? (Almost always yes.)

### 6.2 @expo/fingerprint: The Automated Decision

This is where `@expo/fingerprint` comes in. It's a tool that generates a hash (fingerprint) of everything that affects the native build. If the fingerprint changes, you need a binary. If it doesn't, you can ship OTA.

```bash
# Install
npx expo install @expo/fingerprint

# Generate a fingerprint of the current project
npx @expo/fingerprint --platform ios
# Output: a hash like "abc123def456..."

# Compare with the fingerprint of the currently deployed binary
# If different → binary build needed
# If same → OTA is safe
```

Here's how to use it in CI to automatically decide between OTA and binary:

```yaml
# .github/workflows/smart-deploy.yml
name: Smart Deploy

on:
  push:
    branches: [main]

jobs:
  detect-deploy-type:
    name: Detect Deploy Type
    runs-on: ubuntu-latest
    outputs:
      needs-binary-ios: ${{ steps.fingerprint.outputs.ios-changed }}
      needs-binary-android: ${{ steps.fingerprint.outputs.android-changed }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      - name: Check fingerprint
        id: fingerprint
        run: |
          # Get the fingerprint of the current code
          IOS_FINGERPRINT=$(npx @expo/fingerprint --platform ios)
          ANDROID_FINGERPRINT=$(npx @expo/fingerprint --platform android)

          # Get the fingerprint of the last deployed binary
          # (stored as a build artifact or in a known location)
          LAST_IOS=$(cat .fingerprints/ios.txt 2>/dev/null || echo "none")
          LAST_ANDROID=$(cat .fingerprints/android.txt 2>/dev/null || echo "none")

          echo "Current iOS fingerprint:  $IOS_FINGERPRINT"
          echo "Last iOS fingerprint:     $LAST_IOS"
          echo "Current Android fingerprint: $ANDROID_FINGERPRINT"
          echo "Last Android fingerprint:    $LAST_ANDROID"

          if [ "$IOS_FINGERPRINT" != "$LAST_IOS" ]; then
            echo "ios-changed=true" >> $GITHUB_OUTPUT
            echo "iOS native code changed — binary build needed"
          else
            echo "ios-changed=false" >> $GITHUB_OUTPUT
            echo "iOS native code unchanged — OTA is safe"
          fi

          if [ "$ANDROID_FINGERPRINT" != "$LAST_ANDROID" ]; then
            echo "android-changed=true" >> $GITHUB_OUTPUT
            echo "Android native code changed — binary build needed"
          else
            echo "android-changed=false" >> $GITHUB_OUTPUT
            echo "Android native code unchanged — OTA is safe"
          fi

  deploy-ota:
    name: Deploy OTA Update
    needs: detect-deploy-type
    if: >
      needs.detect-deploy-type.outputs.needs-binary-ios == 'false' &&
      needs.detect-deploy-type.outputs.needs-binary-android == 'false'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      - name: Publish OTA update
        run: |
          cd apps/mobile
          eas update --branch production \
            --message "$(git log -1 --pretty=%B)" \
            --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

  deploy-binary:
    name: Build & Submit Binary
    needs: detect-deploy-type
    if: >
      needs.detect-deploy-type.outputs.needs-binary-ios == 'true' ||
      needs.detect-deploy-type.outputs.needs-binary-android == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      - name: Build binary
        run: |
          cd apps/mobile

          PLATFORMS=""
          if [ "${{ needs.detect-deploy-type.outputs.needs-binary-ios }}" == "true" ]; then
            PLATFORMS="ios"
          fi
          if [ "${{ needs.detect-deploy-type.outputs.needs-binary-android }}" == "true" ]; then
            PLATFORMS="${PLATFORMS:+$PLATFORMS,}android"
          fi

          eas build --platform $PLATFORMS \
            --profile production \
            --non-interactive \
            --auto-submit
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

      - name: Update stored fingerprints
        run: |
          mkdir -p .fingerprints
          npx @expo/fingerprint --platform ios > .fingerprints/ios.txt
          npx @expo/fingerprint --platform android > .fingerprints/android.txt
          git add .fingerprints/
          git commit -m "chore: update native fingerprints [skip ci]"
          git push
```

### 6.3 What Counts as a "Native Change"?

This is the practical reference. Bookmark it.

```
┌──────────────────────────────────────────────────────────────────┐
│  CHANGES THAT REQUIRE A BINARY BUILD                              │
│                                                                   │
│  ✗ Adding/removing/updating a package with native code           │
│    (react-native-*, expo-*, @react-native-*)                     │
│  ✗ Changing app.json/app.config.ts fields that affect native:    │
│    - permissions (ios.infoPlist, android.permissions)             │
│    - schemes (scheme, ios.associatedDomains)                     │
│    - plugins (expo plugins array)                                │
│    - splash screen configuration                                  │
│    - icon changes                                                 │
│    - bundleIdentifier / package name changes                     │
│  ✗ Modifying files in ios/ or android/ directories               │
│  ✗ Changing the Expo SDK version                                  │
│  ✗ Changing the React Native version                              │
│  ✗ Adding a new Expo config plugin                                │
│  ✗ Changing EAS build profile settings                            │
│                                                                   │
│  CHANGES THAT CAN GO OTA                                          │
│                                                                   │
│  ✓ Any .ts/.tsx/.js/.jsx file changes                            │
│  ✓ Style changes (StyleSheet, NativeWind, Tailwind classes)      │
│  ✓ New React components or screens                                │
│  ✓ Navigation changes (adding/removing screens in JS)            │
│  ✓ API endpoint changes                                           │
│  ✓ Business logic changes                                         │
│  ✓ Image/font assets bundled by Metro                             │
│  ✓ Updating JS-only packages (lodash, date-fns, zod, etc.)      │
│  ✓ Feature flag changes                                           │
│  ✓ Localization string changes                                    │
│  ✓ Environment variable changes (JS-side only)                   │
└──────────────────────────────────────────────────────────────────┘
```

### 6.4 The Runtime Version Strategy

EAS Update uses the concept of a **runtime version** to ensure OTA updates are only delivered to compatible binaries. If the runtime version of the OTA update doesn't match the runtime version of the installed binary, the update is silently ignored. This prevents the nightmare scenario of shipping a JS bundle that references a native module the binary doesn't have.

```typescript
// app.config.ts — runtime version strategies

export default {
  // Strategy 1: Manual (not recommended)
  // You manually set runtimeVersion and bump it when native changes occur
  runtimeVersion: '1.0.0',

  // Strategy 2: "appVersion" policy (simple, good starting point)
  // Uses the "version" field as the runtime version
  // OTA updates only go to binaries with matching version
  runtimeVersion: {
    policy: 'appVersion',
  },

  // Strategy 3: "fingerprint" policy (recommended)
  // Uses @expo/fingerprint to compute runtime version automatically
  // If native code changes, fingerprint changes, OTA won't deliver to old binary
  runtimeVersion: {
    policy: 'fingerprint',
  },

  // Strategy 4: "nativeVersion" policy
  // Uses ios.buildNumber and android.versionCode
  runtimeVersion: {
    policy: 'nativeVersion',
  },
};
```

I strongly recommend the `fingerprint` policy. It's the only strategy that's truly automatic. The `appVersion` policy requires you to remember to bump the version when native code changes. The `fingerprint` policy does it for you. Humans forget. Hashes don't.

---

## 7. RELEASE NOTES

Nobody reads release notes. Except when they do. And when they do, bad release notes create support tickets, confused users, and missed opportunities to communicate value.

There are two audiences for release notes, and they need very different things.

### 7.1 User-Facing Release Notes (App Store "What's New")

These are the notes that appear in the App Store and Play Store update screen. Most users glance at them for half a second before tapping "Update." But some users (power users, enterprise customers, accessibility users) read them carefully.

**Rules for good user-facing release notes:**

```
┌──────────────────────────────────────────────────────────────────┐
│  GOOD RELEASE NOTES                                               │
│                                                                   │
│  What's New in 2.5.0:                                            │
│                                                                   │
│  • You can now export your data as CSV from the Reports screen   │
│  • The checkout flow is faster — we reduced load time by 40%     │
│  • Fixed an issue where notifications weren't appearing on       │
│    Android 14 devices                                             │
│  • Improved accessibility: all buttons now work with TalkBack    │
│    and VoiceOver                                                  │
│                                                                   │
│  ─────────────────────────────────────────────────────────────── │
│                                                                   │
│  BAD RELEASE NOTES                                                │
│                                                                   │
│  • Bug fixes and performance improvements                        │
│    (What bugs? What performance? This says nothing.)             │
│                                                                   │
│  • Updated React Native to 0.76                                  │
│    (Users don't know what React Native is.)                      │
│                                                                   │
│  • Fixed JIRA-4521                                                │
│    (Users don't have access to your Jira.)                       │
│                                                                   │
│  • Refactored the state management layer                         │
│    (Users don't care about your architecture.)                   │
└──────────────────────────────────────────────────────────────────┘
```

**The 4 rules:**
1. **Write for users, not engineers.** No jargon. No ticket numbers. No internal references.
2. **Lead with value.** New features first, then improvements, then bug fixes.
3. **Be specific about bug fixes.** "Fixed a crash when opening the camera on Samsung devices" is better than "Bug fixes."
4. **Keep it scannable.** Bullet points. Short sentences. No paragraphs.

### 7.2 Internal Release Notes (Engineering)

These are for your team — engineering, QA, product, customer support. They need detail. They need ticket numbers. They need to know exactly what changed and why.

This is where Changesets shines. The auto-generated changelog is your internal release note:

```markdown
## @repo/mobile v2.5.0

### Minor Changes
- [#234](https://github.com/org/repo/pull/234) — Added CSV export to Reports
- [#228](https://github.com/org/repo/pull/228) — Redesigned checkout flow

### Patch Changes
- [#231](https://github.com/org/repo/pull/231) — Fixed notification delivery on Android 14
- [#229](https://github.com/org/repo/pull/229) — Fixed TalkBack focus order on Settings screen
- Updated @repo/ui to 2.4.1 (focus ring fix)
- Updated @repo/api-client to 1.8.0 (new reports endpoint)
```

### 7.3 Automating Release Notes from Changesets

You can automatically generate user-facing release notes from your Changesets changelog by filtering and reformatting:

```typescript
// scripts/generate-release-notes.ts
import { readFileSync } from 'fs';

interface ChangelogEntry {
  type: 'major' | 'minor' | 'patch';
  description: string;
  pr?: number;
}

function generateUserFacingNotes(changelogPath: string): string {
  const changelog = readFileSync(changelogPath, 'utf-8');

  // Parse the latest version section from the Changesets changelog
  const latestSection = changelog.split('\n## ')[1];
  if (!latestSection) return 'Bug fixes and performance improvements.';

  const entries: ChangelogEntry[] = [];

  const lines = latestSection.split('\n');
  let currentType: 'major' | 'minor' | 'patch' = 'patch';

  for (const line of lines) {
    if (line.includes('Major Changes')) currentType = 'major';
    else if (line.includes('Minor Changes')) currentType = 'minor';
    else if (line.includes('Patch Changes')) currentType = 'patch';
    else if (line.startsWith('- ')) {
      // Extract the description (after the PR link and commit hash)
      const description = line
        .replace(/- \[#\d+\].*? — /, '')  // Remove PR link
        .replace(/- .*?! — /, '')          // Remove Thanks line
        .trim();

      if (description && !description.startsWith('Updated @repo/')) {
        entries.push({ type: currentType, description });
      }
    }
  }

  // Sort: features first, then improvements, then fixes
  const features = entries.filter(e => e.type === 'minor' || e.type === 'major');
  const fixes = entries.filter(e => e.type === 'patch');

  const notes: string[] = [];
  if (features.length > 0) {
    notes.push(...features.map(e => `• ${e.description}`));
  }
  if (fixes.length > 0) {
    notes.push(...fixes.map(e => `• ${e.description}`));
  }

  return notes.join('\n') || 'Bug fixes and performance improvements.';
}

// Usage
const notes = generateUserFacingNotes('apps/mobile/CHANGELOG.md');
console.log(notes);
```

### 7.4 GitHub Releases

When you merge the "Version Packages" PR, create a GitHub Release for traceability:

```yaml
# In your release workflow, after the Changesets action runs:
- name: Create GitHub Release
  if: steps.changesets.outputs.published == 'true'
  uses: actions/github-script@v7
  with:
    script: |
      const packages = JSON.parse('${{ steps.changesets.outputs.publishedPackages }}');
      const mobilePackage = packages.find(p => p.name === '@repo/mobile');

      if (mobilePackage) {
        const tag = `v${mobilePackage.version}`;

        // Read the changelog for this version
        const fs = require('fs');
        const changelog = fs.readFileSync('apps/mobile/CHANGELOG.md', 'utf-8');
        const latestSection = changelog.split('\n## ').slice(1, 2).join('');

        await github.rest.repos.createRelease({
          owner: context.repo.owner,
          repo: context.repo.repo,
          tag_name: tag,
          name: `Mobile ${tag}`,
          body: latestSection,
          draft: false,
          prerelease: false,
        });
      }
```

---

## 8. MONOREPO RELEASE COORDINATION

In a monorepo where mobile and web share packages (`@repo/ui`, `@repo/api-client`, `@repo/shared-utils`), versioning gets interesting. When `@repo/ui` bumps from 2.4.0 to 2.5.0, do both the mobile app and the web app need to release? What if only mobile uses the new Button variant?

### 8.1 Independent vs Synchronized Versioning

There are two schools of thought:

**Independent Versioning:** Each package and app has its own version. `@repo/mobile` can be at 3.1.0 while `@repo/web` is at 5.2.0 and `@repo/ui` is at 2.5.0. They version independently based on their own change history.

**Synchronized Versioning:** All packages in the monorepo share a single version number. When anything changes, everything bumps. `@repo/mobile`, `@repo/web`, and `@repo/ui` are all at 2.5.0.

Here's my recommendation:

```
┌──────────────────────────────────────────────────────────────────┐
│  VERSIONING STRATEGY RECOMMENDATION                               │
│                                                                   │
│  SHARED PACKAGES (@repo/ui, @repo/api-client, @repo/utils):     │
│  → Independent versioning                                        │
│  → Each package has its own semver                               │
│  → Changesets handles dependency bumps automatically             │
│                                                                   │
│  APPS (@repo/mobile, @repo/web):                                 │
│  → Independent versioning                                        │
│  → Mobile follows store release cadence (weekly)                 │
│  → Web deploys on every merge to main (continuous)               │
│  → They don't need to be in sync                                 │
│                                                                   │
│  EXCEPTION — "fixed" groups for tightly coupled packages:        │
│  → If @repo/mobile-config MUST match @repo/mobile's version,    │
│    use Changesets "fixed" groups to keep them in sync            │
│                                                                   │
│  WHY NOT SYNCHRONIZED?                                            │
│  → Web deploys continuously, mobile deploys weekly               │
│  → Forcing them to share a version creates artificial coupling   │
│  → A web-only change shouldn't bump the mobile version           │
│  → Different platforms have different version constraints         │
│    (Android versionCode can't reset, iOS build numbers can)      │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 The Changesets Approach to Monorepo Releases

Changesets was built for this exact problem. Here's how it handles the dependency chain:

```
Scenario: Developer updates @repo/ui (adds outline Button)

1. Developer adds changeset:
   ---
   '@repo/ui': minor
   ---
   Added outline variant to Button

2. Changesets calculates the dependency graph:
   @repo/ui is used by:
   ├── @repo/mobile (depends on @repo/ui)
   └── @repo/web (depends on @repo/ui)

3. With "updateInternalDependencies": "patch":
   @repo/ui: 2.4.0 → 2.5.0 (minor, as specified)
   @repo/mobile: 3.1.0 → 3.1.1 (patch, because a dependency changed)
   @repo/web: 5.2.0 → 5.2.1 (patch, because a dependency changed)

4. The "Version Packages" PR shows all three bumps.
   The CHANGELOG for each package notes the dependency update.
```

This is elegant. The developer only had to think about `@repo/ui`. Changesets figured out that `@repo/mobile` and `@repo/web` need to bump too, and it did so as a patch (not a minor) because the *app* didn't add a feature — one of its *dependencies* did.

### 8.3 Linked vs Fixed Groups

Changesets offers two mechanisms for coordinating package versions:

```json
// .changeset/config.json
{
  // "fixed" — packages always share the EXACT same version
  // If any package bumps, they ALL bump to the same version
  "fixed": [
    // These packages are always at the same version
    ["@repo/mobile", "@repo/mobile-config", "@repo/mobile-codegen"]
  ],

  // "linked" — packages share the same BUMP TYPE
  // If one gets a major bump, they all get a major bump
  // But they can have different version numbers
  "linked": [
    // If @repo/ui gets a major bump, ui-native and ui-web
    // also get a major bump (but can be at different versions)
    ["@repo/ui", "@repo/ui-native", "@repo/ui-web"]
  ]
}
```

**When to use "fixed":** When packages are truly the same thing split across files. Like a mobile app and its code generation config — they *must* be at the same version because they're deployed together.

**When to use "linked":** When packages are related but separate. Like cross-platform UI packages — a breaking change in one usually means a breaking change in the others, but they might be at different patch versions because of independent bug fixes.

### 8.4 The Release Flow in a Monorepo

Here's how a typical release week looks in a monorepo with Changesets:

```
Monday:
  9:00 AM   Merge the accumulated "Version Packages" PR
            → @repo/ui bumps to 2.5.0
            → @repo/api-client bumps to 1.8.0
            → @repo/mobile bumps to 3.2.0
            → @repo/web bumps to 5.3.0

  9:05 AM   CI detects mobile version bump → triggers EAS Build
            CI detects web version bump → Vercel auto-deploys

  9:10 AM   Web is live (5.3.0 deployed to production via Vercel)
            Mobile binary is building (EAS cloud build)

  10:00 AM  Mobile build completes → auto-submitted to App Store & Play Store

Tuesday:
  App Store review completes
  Play Store review completes
  Start staged rollout: iOS phased release, Android 5%

Wednesday:
  Monitor metrics
  Android bumped to 20%

Thursday:
  Android bumped to 50%
  iOS auto-progresses through phased schedule

Friday:
  Android bumped to 100%
  iOS continues phased rollout (auto)

Next Monday:
  iOS phased rollout complete (100%)
  Next release cycle begins
```

### 8.5 When Web and Mobile Releases Conflict

Sometimes a shared package update needs to ship on web immediately but isn't ready for mobile yet (maybe the mobile UI using the new component needs more QA). Here's how to handle it:

**Option 1: Feature flags.** The shared package update ships, but the mobile app only uses the new functionality behind a feature flag. Web enables the flag immediately. Mobile enables it in the next release.

**Option 2: Selective changelog.** The changeset for the shared package update triggers a web deploy immediately (since web deploys continuously) and a mobile version bump. But the mobile release train picks it up on its normal schedule — no rush.

**Option 3: Multiple release branches.** For truly different timelines, you can have `release/web/5.3.0` and `release/mobile/3.2.0` branches. But this adds complexity. Feature flags are almost always the better choice.

---

## 9. ROLLBACK STRATEGIES

Things will go wrong. Not if, when. The mark of a mature team isn't that they never ship bugs — it's that they can recover quickly when they do. Every platform has different rollback mechanisms, and you need to know all of them before you need them. Discovering your rollback strategy during an incident is too late.

### 9.1 Web Rollback (Seconds)

Vercel makes web rollback trivial:

```bash
# Instant rollback to a previous deployment
vercel rollback [deployment-url]

# Or from the Vercel dashboard:
# Deployments → Find the last known good deployment → "..." → "Promote to Production"
```

This takes effect in seconds. Vercel's edge network points to the previous deployment. No rebuild. No waiting. The old deployment is still there, fully intact.

**The Vercel rollback model:**
- Every deployment is immutable and lives forever (until you delete it)
- "Rolling back" is really "promoting a previous deployment to production"
- There's no downtime — traffic switches instantly
- You can roll forward again just as easily

For non-Vercel web deployments (self-hosted, AWS, etc.), maintain at least the last 3 deployments as ready-to-promote targets. Use blue-green or canary deployment patterns.

### 9.2 Mobile OTA Rollback (Minutes)

If you shipped a bad OTA update via EAS Update, you can "roll back" by publishing the previous bundle:

```bash
# Option 1: Republish the previous good update
# Find the previous update ID
eas update:list --branch production

# Roll back by republishing the previous update
eas update:rollback --branch production

# Option 2: Publish a new update from the last known good commit
git checkout v2.5.0  # The last known good release tag
eas update --branch production --message "Rollback: reverting bad OTA update"
```

When a user opens the app after the rollback, they'll download the previous (good) bundle and the bad update is effectively erased. The speed depends on how your app checks for updates:

- **On app start** (default): Users get the fix next time they open the app
- **Eager checking** (recommended for production): Users get the fix within minutes

### 9.3 Mobile Binary Rollback (Complex)

This is the hardest rollback scenario. A bad binary is in the store, users are downloading it, and you can't un-publish it.

**Your options:**

```
┌──────────────────────────────────────────────────────────────────┐
│  BINARY ROLLBACK OPTIONS                                          │
│                                                                   │
│  1. HALT STAGED ROLLOUT (if still in progress)                   │
│     - Stops new users from getting the bad update                │
│     - Users who already have it keep it                          │
│     - Best case: you caught it early in the rollout              │
│     - Android: halt via Play Console or Fastlane                 │
│     - iOS: pause phased release in App Store Connect             │
│                                                                   │
│  2. SHIP A HOTFIX BINARY (the standard approach)                 │
│     - Build a new binary with the fix                            │
│     - Submit for expedited review (Apple) / review (Google)      │
│     - iOS expedited review: 2-6 hours typically                  │
│     - Google Play review: usually < 24 hours                     │
│     - Start a new staged rollout for the fix                     │
│                                                                   │
│  3. PATCH VIA OTA (if possible)                                  │
│     - If the fix can be done in JS, ship an OTA update           │
│     - This patches the bad binary without a store review         │
│     - The next app launch picks up the OTA fix                   │
│     - Best case: minutes to resolution                           │
│                                                                   │
│  4. SERVER-SIDE MITIGATION (immediate)                           │
│     - Feature flag to disable the broken feature                 │
│     - Force update prompt to push users to a good version        │
│     - API-level workarounds                                       │
│     - This is a band-aid, not a fix — but it stops the bleeding │
│                                                                   │
│  5. PULL THE UPDATE (nuclear option)                             │
│     - Remove the update from the store entirely                  │
│     - Users who don't have it can't get it                       │
│     - Users who have it keep it (you can't uninstall)            │
│     - The previous version becomes the latest again              │
│     - Android: straightforward in Play Console                   │
│     - iOS: more complex, may require Apple support               │
│     - Use as last resort only                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 9.4 Database Migration Rollbacks

If your release includes database schema changes (via Prisma, Drizzle, or raw migrations), rollback becomes more complex because data changes may not be reversible.

**Rules for safe database migrations:**

1. **Always write backward-compatible migrations.** A new column should have a default value. A renamed column should keep the old column too (and backfill) until all app versions that reference the old name are gone.

2. **Never delete columns in the same release that stops using them.** The sequence is:
   - Release 1: Stop reading from the old column, start reading from the new column
   - Release 2 (after all old binary versions are gone): Delete the old column

3. **Separate deploy from migrate.** Don't run migrations automatically on deploy. Run them as a separate step with a manual trigger or approval gate.

```typescript
// Prisma migration strategy for safe rollbacks
// migrations/20260407_add_display_name.ts

// FORWARD migration — always backward-compatible
export async function up(db: Database) {
  // Add new column with default value — old code still works
  await db.schema.alterTable('users', (table) => {
    table.string('display_name').defaultTo('');
  });

  // Backfill from existing data
  await db.raw(`
    UPDATE users
    SET display_name = COALESCE(first_name || ' ' || last_name, email)
    WHERE display_name = ''
  `);
}

// ROLLBACK migration — optional but recommended
export async function down(db: Database) {
  await db.schema.alterTable('users', (table) => {
    table.dropColumn('display_name');
  });
}
```

### 9.5 The Rollback Matrix

Here's a quick-reference matrix for every rollback scenario:

```
┌────────────────┬──────────────┬──────────────┬─────────────────┐
│ Platform       │ Speed        │ Complexity   │ Method          │
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Web (Vercel)   │ Seconds      │ Trivial      │ Promote previous│
│                │              │              │ deployment      │
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Mobile OTA     │ Minutes      │ Low          │ Publish previous│
│ (EAS Update)   │              │              │ bundle          │
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Mobile Binary  │ Hours-Days   │ High         │ Halt rollout +  │
│ (staged)       │              │              │ hotfix binary   │
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Mobile Binary  │ Hours-Days   │ Very High    │ Hotfix binary + │
│ (full rollout) │              │              │ expedited review│
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Database       │ Minutes-Hours│ Very High    │ Reverse         │
│ (migrations)   │              │              │ migration       │
│                │              │              │ (if possible)   │
├────────────────┼──────────────┼──────────────┼─────────────────┤
│ Feature Flag   │ Seconds      │ Trivial      │ Toggle flag off │
│ (server-side)  │              │              │ in dashboard    │
└────────────────┴──────────────┴──────────────┴─────────────────┘
```

The takeaway: feature flags and OTA are your fastest rollback mechanisms. Design your system so that most issues can be mitigated by one of these two methods. Reserve binary hotfixes and database rollbacks for when you truly have no other option.

---

## 10. THE COMPLETE RELEASE CHECKLIST

After everything we've covered — versioning, Changesets, staged rollouts, hotfixes, OTA, rollbacks — let me distill it into a single, actionable checklist. Print this out. Put it in your team's wiki. Reference it every release until it becomes muscle memory.

### 10.1 Pre-Release Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│  PRE-RELEASE CHECKLIST                                            │
│                                                                   │
│  CODE & QUALITY                                                   │
│  □ All planned PRs are merged to main                            │
│  □ "Version Packages" PR has been reviewed and is ready to merge │
│  □ All CI checks pass on the version bump commit                 │
│  □ No open P0/P1 bugs targeting this release                     │
│  □ All changeset files accurately describe changes               │
│  □ CHANGELOG.md entries reviewed for accuracy                    │
│                                                                   │
│  TESTING                                                          │
│  □ Unit tests pass (100%)                                        │
│  □ Integration tests pass                                        │
│  □ E2E tests pass on target platforms (Maestro for mobile,       │
│    Playwright for web)                                            │
│  □ Manual QA on release candidate build (if required by team)    │
│  □ Tested on minimum supported OS versions                       │
│    (iOS 16+, Android API 24+, or whatever your minimums are)    │
│  □ Tested on both small and large screen devices                 │
│  □ Tested with accessibility tools (VoiceOver, TalkBack)         │
│  □ Regression test on critical flows:                            │
│    - Sign up / Sign in                                            │
│    - Core feature (whatever your app's main thing is)            │
│    - Payments (if applicable)                                     │
│    - Push notifications                                           │
│                                                                   │
│  PERFORMANCE                                                      │
│  □ Bundle size hasn't regressed (check CI budget gates)          │
│  □ App start time hasn't regressed                               │
│  □ No new performance warnings in profiler                       │
│  □ Web: Lighthouse scores meet thresholds                        │
│                                                                   │
│  ASSETS & METADATA                                                │
│  □ App Store screenshots updated (if UI changed significantly)   │
│  □ Play Store screenshots updated (if UI changed significantly)  │
│  □ User-facing release notes written                             │
│  □ "What's New" text prepared for both stores                    │
│  □ App Store keywords updated (if relevant)                      │
│                                                                   │
│  INFRASTRUCTURE                                                   │
│  □ API backward compatibility confirmed (old app versions still  │
│    work with current API)                                        │
│  □ Database migrations tested (if any)                           │
│  □ Feature flags configured for new features                     │
│  □ Remote config values set (if any)                             │
│  □ Monitoring dashboards ready for the new version               │
│  □ Sentry release created for new version                        │
│  □ Source maps / dSYMs will be uploaded during build              │
└──────────────────────────────────────────────────────────────────┘
```

### 10.2 Release Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│  RELEASE CHECKLIST                                                │
│                                                                   │
│  VERSION BUMP                                                     │
│  □ Merge the "Version Packages" PR                               │
│  □ Verify version numbers in package.json / app.config.ts        │
│  □ Verify CHANGELOG.md was generated correctly                   │
│  □ Git tag created (automated by Changesets action)              │
│                                                                   │
│  WEB DEPLOY                                                       │
│  □ Vercel production deployment triggered (automatic on merge)   │
│  □ Deployment successful (check Vercel dashboard)                │
│  □ Smoke test production URL                                     │
│  □ Sentry release finalized for web                              │
│                                                                   │
│  MOBILE BUILD                                                     │
│  □ EAS Build triggered (automatic or manual)                     │
│  □ iOS build successful                                          │
│  □ Android build successful                                      │
│  □ Source maps uploaded to Sentry                                │
│  □ dSYMs uploaded to Sentry / Crashlytics                        │
│                                                                   │
│  STORE SUBMISSION                                                 │
│  □ iOS submitted to App Store Connect                            │
│    □ "What's New" text added                                     │
│    □ Phased release enabled                                      │
│    □ Submitted for review                                        │
│  □ Android submitted to Google Play Console                      │
│    □ Release notes added (per-language if localized)             │
│    □ Staged rollout percentage set (start at 1-5%)               │
│    □ Submitted for review                                        │
│                                                                   │
│  STORE REVIEW                                                     │
│  □ iOS review passed                                             │
│  □ Android review passed                                         │
│  □ If rejected: read rejection reason, fix, resubmit             │
│                                                                   │
│  STAGED ROLLOUT BEGINS                                            │
│  □ iOS phased release started (automatic after approval)         │
│  □ Android staged rollout started at initial percentage          │
│  □ Team notified in #releases channel                            │
│  □ On-call engineer knows a release is rolling out               │
└──────────────────────────────────────────────────────────────────┘
```

### 10.3 Post-Release Checklist

```
┌──────────────────────────────────────────────────────────────────┐
│  POST-RELEASE CHECKLIST                                           │
│                                                                   │
│  MONITORING (First 48 Hours)                                      │
│  □ Crash-free rate monitored hourly for first 24h                │
│    Target: > 99.5% (same or better than previous version)        │
│  □ ANR rate monitored (Android)                                  │
│    Target: < 0.5%                                                │
│  □ Error rate in Sentry — no new error spikes                    │
│  □ API error rate — no increase correlated with new version      │
│  □ App start time — no regression                                │
│  □ User-reported issues in support channels monitored            │
│  □ App store reviews monitored for new complaints                │
│                                                                   │
│  STAGED ROLLOUT PROGRESSION                                      │
│  □ Day 1 (1-5%): Metrics healthy? → Increase to 20%             │
│  □ Day 2 (20%): Metrics healthy? → Increase to 50%              │
│  □ Day 3 (50%): Metrics healthy? → Increase to 100%             │
│  □ If metrics unhealthy at ANY stage: HALT. Don't increase.     │
│    Investigate, fix, and either:                                 │
│    - Ship OTA fix and resume rollout                             │
│    - Ship new binary and restart rollout                         │
│                                                                   │
│  COMMUNICATION                                                    │
│  □ Release announcement posted in #releases channel              │
│  □ Product/marketing notified (if user-facing features shipped)  │
│  □ Customer support briefed on new features / known issues       │
│  □ Documentation updated (if API or behavior changed)            │
│                                                                   │
│  CLEANUP                                                          │
│  □ Release branch deleted (if using release branches)            │
│  □ Hotfix branches merged back to main and deleted               │
│  □ GitHub Release created with changelog                         │
│  □ Sentry release finalized with commit range                    │
│  □ Stale feature flags cleaned up (features fully rolled out)    │
│                                                                   │
│  RETROSPECTIVE (Async or in Weekly Meeting)                      │
│  □ What went well with this release?                             │
│  □ What was painful?                                             │
│  □ Did anything slip through testing?                            │
│  □ Are there automation improvements we should make?             │
│  □ Update this checklist with lessons learned                    │
└──────────────────────────────────────────────────────────────────┘
```

### 10.4 The Complete CI/CD + Release Workflow

Here's how it all fits together, from commit to user:

```
┌──────────────────────────────────────────────────────────────────┐
│  THE COMPLETE RELEASE FLOW                                        │
│                                                                   │
│  1. DEVELOP                                                       │
│     Developer creates PR with code + changeset file              │
│     ↓                                                             │
│  2. REVIEW                                                        │
│     PR reviewed: code, tests, changeset                          │
│     Changeset bot confirms changeset is present                  │
│     CI passes: lint, typecheck, test, build                      │
│     ↓                                                             │
│  3. MERGE                                                         │
│     PR merged to main                                            │
│     Web: auto-deploys to Vercel (continuous)                     │
│     Mobile: changeset accumulates                                │
│     ↓                                                             │
│  4. VERSION                                                       │
│     Changesets bot opens "Version Packages" PR                   │
│     Shows: all version bumps + generated changelogs              │
│     Team lead reviews and merges when ready to release           │
│     ↓                                                             │
│  5. BUILD                                                         │
│     CI detects version bump → checks fingerprint                 │
│     Fingerprint unchanged → OTA update published                 │
│     Fingerprint changed → EAS Build triggered                    │
│     ↓                                                             │
│  6. SUBMIT                                                        │
│     Binary auto-submitted to App Store + Play Store              │
│     "What's New" text from generated release notes               │
│     Source maps + dSYMs uploaded to Sentry                       │
│     ↓                                                             │
│  7. REVIEW                                                        │
│     Wait for store review (24-48 hours typically)                │
│     ↓                                                             │
│  8. ROLLOUT                                                       │
│     iOS: phased release (7-day ramp)                             │
│     Android: staged rollout (manual percentage bumps)            │
│     Monitor crash-free rate, ANR rate, errors at each stage      │
│     ↓                                                             │
│  9. FULL RELEASE                                                  │
│     100% of users have the update                                │
│     ↓                                                             │
│  10. MONITOR                                                      │
│      48-hour post-release monitoring window                      │
│      Crash-free rate, performance metrics, user feedback         │
│      ↓                                                             │
│  11. NEXT CYCLE                                                   │
│      Release branch cleaned up                                   │
│      Next set of changesets accumulating                         │
│      Repeat                                                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## PUTTING IT ALL TOGETHER

Release management isn't sexy. Nobody's going to put "designed a staged rollout process" on their conference talk abstract. But it's the difference between a team that ships with confidence and a team that ships with crossed fingers.

The core principles are:

1. **Automate everything that can be automated.** Version bumps, changelog generation, build numbers, store submissions, fingerprint detection — all of it. Humans are bad at repetitive tasks. Computers are great at them.

2. **Ship small, ship often.** A weekly release with 15 changes is dramatically less risky than a monthly release with 60 changes. The math is not linear — risk compounds.

3. **Always have a rollback plan.** Before you ship, know exactly how you'd un-ship. Feature flags, OTA rollback, staged rollout halt, Vercel instant rollback — have the plan ready before you need it.

4. **Monitor after you ship.** A release isn't done when it's in the store. It's done when you've confirmed it's healthy in production. The 48-hour monitoring window is not optional.

5. **Write it down.** The checklist, the runbook, the decision tree — put them where your team can find them. The worst time to figure out your hotfix process is during a hotfix.

Here's what the tooling stack looks like when everything is wired up:

| Concern | Tool | Chapter Reference |
|---------|------|-------------------|
| Version management | Changesets | This chapter |
| Build numbers | EAS auto-increment | This chapter |
| Mobile builds | EAS Build | Ch 7, 18 |
| OTA updates | EAS Update | Ch 19 |
| Web deploys | Vercel | Ch 18 |
| Fingerprint detection | @expo/fingerprint | This chapter |
| Error monitoring | Sentry + Crashlytics | Ch 20 |
| Feature flags | Statsig / LaunchDarkly | Ch 21 |
| Store submission | EAS Submit / Fastlane | Ch 18 |
| CI orchestration | GitHub Actions | Ch 18 |
| Monorepo management | Turborepo | Ch 15 |

This is not a lot of tools. Most of them you already have if you've followed the earlier chapters. Release management is less about adding new tools and more about connecting the tools you have into a coherent process.

The team that treats releases as a boring, well-oiled machine — that's the team that ships features while everyone else is debugging their release process.

---

## CHAPTER SUMMARY

Release management for a mobile + web monorepo requires understanding platform-specific versioning (version name vs build number), adopting automated tooling (Changesets for version bumps and changelogs, EAS for build number auto-increment), maintaining a regular cadence (weekly release trains for mobile, continuous for web), using staged rollouts as a safety net (never ship to 100% on day one), and having well-practiced playbooks for hotfixes and rollbacks.

The key decisions:
- **OTA vs Binary**: Use `@expo/fingerprint` to automate the decision. JS-only → OTA (minutes). Native changes → binary (hours/days).
- **Release cadence**: Weekly for mobile, continuous for web. Adjust based on team size and risk tolerance.
- **Rollback**: Feature flags (seconds), OTA rollback (minutes), staged rollout halt (immediate), binary hotfix (hours/days). Know which tool to reach for before you need it.

---

### Up Next

**Chapter 46: Infrastructure as Code** — You've mastered the release process. Now let's talk about the infrastructure that supports it — how to define, version, and deploy your backend infrastructure with the same discipline you apply to your frontend code.

### Further Reading
- [Changesets Documentation](https://github.com/changesets/changesets)
- [EAS Build Documentation](https://docs.expo.dev/build/introduction/)
- [EAS Update Documentation](https://docs.expo.dev/eas-update/introduction/)
- [@expo/fingerprint Documentation](https://docs.expo.dev/build-reference/app-versions/)
- [Apple Phased Release](https://developer.apple.com/help/app-store-connect/update-your-app/release-a-version-update-in-phases)
- [Google Play Staged Rollouts](https://support.google.com/googleplay/android-developer/answer/6346149)
- [DORA State of DevOps Report](https://dora.dev/research/)
- [Semantic Versioning Specification](https://semver.org/)
- [Vercel Instant Rollback](https://vercel.com/docs/deployments/instant-rollback)
