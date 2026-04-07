<!--
  CHAPTER: 19
  TITLE: CI/CD for Mobile & Web
  PART: IV — Architecture at Scale
  PREREQS: Chapters 5, 6, 15
  KEY_TOPICS: GitHub Actions, EAS Build CI, preview deployments, performance budgets, semantic versioning, Changesets, mobile+web monorepo CI, Fastlane, self-hosted runners
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 18: CI/CD for Mobile & Web

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 5, 6, 15 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Build only what changed (Turborepo `--filter` + GitHub Actions path triggers); a README typo should not trigger an iOS build
- Use EAS Build for mobile CI (cloud compilation, signing, no local Mac needed) and Vercel for web CI (preview deployments on every PR, production on merge to main)
- Enforce performance budgets in CI: block PRs that exceed bundle size limits, Lighthouse score thresholds, or build time ceilings
- Automate version bumps with Changesets; let CI compute `versionCode`/`buildNumber` from git history so humans never manually manage version numbers
- Cancel superseded CI runs, cache aggressively (node_modules, Turborepo, Gradle, CocoaPods), and fail fast on lint/type errors before running expensive builds

</details>

Here's the dirty secret about most mobile+web monorepos: **their CI/CD pipeline is the most expensive, most fragile, least understood part of the entire system.** Teams that build beautiful React Native apps with immaculate state management and pixel-perfect designs will have a 45-minute CI pipeline held together with duct tape, retry loops, and a Slack message that says "CI is flaky again, just re-run it."

I've seen a Series B startup burning $4,000/month on GitHub Actions minutes because every PR triggered iOS, Android, and web builds — even when the change was a README typo. I've seen a team of 30 engineers lose an average of 40 minutes per day waiting for CI. That's 20 person-hours per day. Over a year, that's roughly $600,000 in wasted engineering salary. On CI.

The difference between a team that ships confidently twice a day and a team that dreads merging to main is almost always the pipeline. Not the code. Not the tests. The pipeline.

This chapter is about building a CI/CD pipeline that is fast, correct, and cheap — in that order. One that builds only what changed, tests only what's affected, deploys automatically when checks pass, cancels superseded runs, and fails fast when something is wrong. One pipeline, from one monorepo, deploying to iOS, Android, and web.

### In This Chapter
- The CI/CD Philosophy — principles that prevent $4,000/month bills
- GitHub Actions Patterns — matrix builds, caching, concurrency, reusable workflows
- Mobile CI with EAS — triggering cloud builds from CI, managing profiles and secrets
- Web CI with Vercel — preview deployments, production on merge, environment variables
- Testing in CI — unit, integration, E2E across mobile and web
- Performance Budgets — enforcing bundle size, Lighthouse scores, build time limits
- Self-Hosted CI for Mobile — Fastlane, macOS runners, the cost/complexity trade-off
- Semantic Versioning for Mobile — Changesets, versionCode, buildNumber, automating bumps
- The Complete Pipeline — lint to deploy, one monorepo, three platforms
- Deployment Gates — manual approval, staged rollouts, canary releases

### Related Chapters
- [Ch 5: Expo Platform] — EAS Build and EAS Update fundamentals
- [Ch 6: Monorepo Architecture] — Turborepo workspace structure this pipeline builds on
- [Ch 15: Testing Strategy] — the tests this pipeline runs
- [Ch 19: OTA Updates] — how EAS Update fits into the deployment story
- [Ch 20: App Store Deployment] — what happens after CI builds the binary

---

## 1. THE CI/CD PHILOSOPHY

Before we write a single line of YAML, we need to align on principles. Every decision in this chapter flows from five rules:

### Rule 1: Build Only What Changed

If someone changes a file in `packages/ui`, you don't need to rebuild the iOS app, the Android app, *and* the web app. You need to rebuild whatever depends on `packages/ui`. Turborepo gives you this with `--filter`. GitHub Actions gives you path-based triggers. Use both.

```
┌─────────────────────────────────────────────────┐
│               MONOREPO CHANGE                    │
│                                                  │
│   Changed: packages/ui/src/Button.tsx            │
│                                                  │
│   Affected:                                      │
│   ├── apps/mobile (imports @repo/ui)     → BUILD │
│   ├── apps/web (imports @repo/ui)        → BUILD │
│   └── packages/api-client               → SKIP  │
│                                                  │
│   Not affected: don't build, don't test, save $  │
└─────────────────────────────────────────────────┘
```

**Shopify does this.** Their monorepo CI uses a combination of Bazel target analysis and custom change detection to skip unaffected builds. The result: a PR that touches only the admin dashboard doesn't trigger builds for Shop, Inbox, or POS. They estimated this saves thousands of CI minutes per week.

### Rule 2: Test Only What's Affected

Same principle, applied to tests. If you changed `packages/api-client`, run the tests for `api-client` and everything that depends on it. Don't run the E2E tests for the marketing website.

Turborepo handles this beautifully:

```bash
# Run tests only for packages affected by changes since main
npx turbo test --filter='...[origin/main]'
```

That `[origin/main]` syntax means "packages that changed since the main branch." The `...` prefix means "and everything that depends on them." This single command is the foundation of efficient CI.

### Rule 3: Deploy Automatically When Checks Pass

If lint passes, types check, tests pass, and the build succeeds — deploy. Don't wait for a human to click a button. The whole point of CI is that the machine verifies what a human would verify, but faster and more consistently.

The exception is production mobile releases (we'll cover deployment gates in Section 10), but for web preview deployments and mobile preview builds, automation should be the default.

### Rule 4: Cancel Superseded Runs

When a developer pushes a fix to their PR, the previous CI run is no longer relevant. Kill it. GitHub Actions' `concurrency` feature does this:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

This single block saves more CI minutes than any caching strategy. I've seen repos where 30-40% of CI runs were superseded before they finished. Without cancellation, those runs consumed full minutes and blocked runners for other PRs.

### Rule 5: Fail Fast

If linting fails in 20 seconds, don't wait for the 15-minute iOS build to finish before telling the developer. Structure your pipeline so cheap checks run first and expensive checks run only after cheap ones pass.

```
┌─────────────────────────────────────────────────────────┐
│  FAIL FAST PIPELINE                                      │
│                                                          │
│  Stage 1 (< 1 min):  lint + typecheck + format           │
│       ↓ passes                                           │
│  Stage 2 (2-5 min):  unit tests + integration tests      │
│       ↓ passes                                           │
│  Stage 3 (5-20 min): builds (iOS, Android, Web)          │
│       ↓ passes                                           │
│  Stage 4 (5-15 min): E2E tests                           │
│       ↓ passes                                           │
│  Stage 5:            deploy                              │
│                                                          │
│  Each stage only runs if the previous one passes.        │
│  Average feedback time: 45 seconds (lint failure)        │
│  vs. 25 minutes (everything passes)                      │
└─────────────────────────────────────────────────────────┘
```

### The Cost Reality

Let's do the math that most teams don't do until the bill arrives.

GitHub Actions pricing (as of 2026):
- Linux runners: $0.008/minute
- macOS runners: $0.08/minute (10x Linux!)
- Large macOS runners (M1): $0.16/minute

An iOS build on EAS takes ~15 minutes. On a self-hosted macOS runner, ~20 minutes. On a GitHub-hosted macOS runner, ~25 minutes.

| Scenario | Minutes/PR | PRs/day | Monthly Cost |
|----------|-----------|---------|-------------|
| Full build every PR (GitHub macOS) | 45 min | 20 | ~$2,160 |
| Filtered builds + caching | 15 min avg | 20 | ~$720 |
| Filtered + EAS offload | 8 min GH + EAS | 20 | ~$384 + EAS plan |
| Optimized (all principles) | 5 min avg GH | 20 | ~$240 + EAS plan |

The difference between naive and optimized is **9x in cost** and **9x in developer wait time.** Both matter. The first hits your bank account. The second hits your velocity.

---

## 2. GITHUB ACTIONS PATTERNS

GitHub Actions is the CI system for most React Native + web monorepos. Not because it's the best CI system ever built — it has real limitations — but because it's integrated with GitHub, it's flexible enough, and the ecosystem of reusable actions is enormous. Let's cover the patterns that matter.

### 2.1 Matrix Builds

When you need to build for multiple platforms, a matrix strategy lets you define the variations once and let GitHub Actions fan them out:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Stage 1: Cheap checks (fail fast)
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo lint typecheck --filter='...[origin/main]'

  # Stage 2: Tests
  test:
    needs: lint-and-typecheck
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for --filter
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo test --filter='...[origin/main]'

  # Stage 3: Platform builds
  build:
    needs: test
    strategy:
      fail-fast: true
      matrix:
        include:
          - platform: web
            runs-on: ubuntu-latest
          - platform: ios
            runs-on: macos-14  # M1 runner
          - platform: android
            runs-on: ubuntu-latest
    runs-on: ${{ matrix.runs-on }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo build --filter=@repo/${{ matrix.platform }}
```

**The `fail-fast: true` is important.** If the web build fails, it cancels the iOS and Android builds immediately. No point burning macOS runner minutes when you know the pipeline is already broken.

But wait — there's a problem with this naive approach. **The iOS build is running on a macOS runner even when no mobile code changed.** Let's fix that.

### 2.2 Conditional Platform Builds

```yaml
  # Determine what changed
  changes:
    runs-on: ubuntu-latest
    outputs:
      mobile: ${{ steps.filter.outputs.mobile }}
      web: ${{ steps.filter.outputs.web }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            mobile:
              - 'apps/mobile/**'
              - 'packages/ui/**'
              - 'packages/api-client/**'
              - 'packages/shared/**'
            web:
              - 'apps/web/**'
              - 'packages/ui/**'
              - 'packages/api-client/**'
              - 'packages/shared/**'
            shared:
              - 'packages/**'
              - '.github/**'
              - 'package.json'
              - 'turbo.json'

  build-ios:
    needs: [test, changes]
    if: needs.changes.outputs.mobile == 'true' || needs.changes.outputs.shared == 'true'
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      # ... iOS build steps

  build-android:
    needs: [test, changes]
    if: needs.changes.outputs.mobile == 'true' || needs.changes.outputs.shared == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      # ... Android build steps

  build-web:
    needs: [test, changes]
    if: needs.changes.outputs.web == 'true' || needs.changes.outputs.shared == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo build --filter=@repo/web
```

Now a change to `apps/web/src/pages/about.tsx` only triggers the web build. The iOS and Android jobs are skipped entirely. On a team that merges 20 PRs/day where 60% are web-only, this saves roughly **$1,000/month in macOS runner costs alone.**

### 2.3 Caching Everything

Caching is the single biggest lever for CI speed after filtering. Here's what to cache and how:

#### Node Modules

The `actions/setup-node` cache handles `yarn.lock`-based caching automatically:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: 'yarn'
```

But for monorepos, you might want more granular caching:

```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: |
      node_modules
      apps/*/node_modules
      packages/*/node_modules
    key: node-modules-${{ runner.os }}-${{ hashFiles('yarn.lock') }}
    restore-keys: |
      node-modules-${{ runner.os }}-
```

#### Turborepo Remote Cache

This is the single most impactful CI optimization for monorepos. Turborepo can cache task outputs (build artifacts, test results, lint results) and share them across CI runs.

```yaml
# In your workflow
env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

steps:
  - run: yarn turbo build test lint --filter='...[origin/main]'
    # Turborepo will:
    # 1. Check the remote cache for each task
    # 2. If cached, restore the output instantly
    # 3. If not cached, run the task and upload the output
```

The math is compelling. Say `packages/ui` hasn't changed. Without Turborepo cache, every PR rebuilds it (~30 seconds). With cache, it restores in ~2 seconds. Across 50 packages and 20 PRs/day, that's the difference between 25 minutes and 1 minute of redundant work per PR.

**Set up Turborepo remote caching with Vercel:**

```bash
# One-time setup
npx turbo login
npx turbo link
```

Then add `TURBO_TOKEN` to your GitHub Actions secrets. Done.

#### CocoaPods Cache (iOS)

CocoaPods is notoriously slow. Caching the Pods directory saves 2-5 minutes per iOS build:

```yaml
- name: Cache CocoaPods
  uses: actions/cache@v4
  with:
    path: apps/mobile/ios/Pods
    key: pods-${{ runner.os }}-${{ hashFiles('apps/mobile/ios/Podfile.lock') }}
    restore-keys: |
      pods-${{ runner.os }}-

- name: Install CocoaPods
  working-directory: apps/mobile/ios
  run: |
    if [ ! -d "Pods" ]; then
      pod install
    fi
```

#### Gradle Cache (Android)

```yaml
- name: Cache Gradle
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ runner.os }}-${{ hashFiles('apps/mobile/android/**/*.gradle*', 'apps/mobile/android/gradle-wrapper.properties') }}
    restore-keys: |
      gradle-${{ runner.os }}-
```

#### The Cache Summary

| Cache Target | Typical Save | Key Strategy |
|-------------|-------------|-------------|
| node_modules | 30-90s | Hash of lockfile |
| Turborepo remote cache | 1-10min | Content hash of inputs |
| CocoaPods | 2-5min | Hash of Podfile.lock |
| Gradle | 1-3min | Hash of gradle files |
| Expo prebuild | 3-8min | Hash of app.json + plugins |
| EAS fingerprint | Entire build | Runtime version hash |

### 2.4 Concurrency Groups

We covered the basic pattern earlier. Here's the advanced version that handles multiple workflows:

```yaml
# For PR checks — cancel previous runs on same PR
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

# For deployments — don't cancel, queue instead
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

The distinction matters. For PR checks, you always want the latest push to win. For production deployments, you want them to queue — canceling a half-finished deployment is worse than waiting.

### 2.5 Reusable Workflows

When your pipeline grows, you'll want to share workflow logic across repos or across workflow files. GitHub Actions supports this with reusable workflows:

```yaml
# .github/workflows/reusable-node-setup.yml
name: Node Setup
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
      working-directory:
        type: string
        default: '.'

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
        working-directory: ${{ inputs.working-directory }}
```

```yaml
# .github/workflows/ci.yml
jobs:
  lint:
    uses: ./.github/workflows/reusable-node-setup.yml
    with:
      node-version: '20'
```

**Vercel uses this pattern extensively.** Their open-source repos share workflow definitions across hundreds of repositories through a central `.github` repository. It means a caching improvement or security fix propagates everywhere at once.

### 2.6 Composite Actions for Repeated Steps

When you find yourself copying the same 5 steps across jobs, extract them into a composite action:

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Install deps and set up caching'

inputs:
  turbo-token:
    description: 'Turborepo remote cache token'
    required: false

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'

    - name: Install dependencies
      shell: bash
      run: yarn install --frozen-lockfile

    - name: Configure Turborepo
      if: inputs.turbo-token != ''
      shell: bash
      run: echo "TURBO_TOKEN=${{ inputs.turbo-token }}" >> $GITHUB_ENV
```

Then use it anywhere:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-project
    with:
      turbo-token: ${{ secrets.TURBO_TOKEN }}
  - run: yarn turbo lint
```

---

## 3. MOBILE CI WITH EAS

EAS (Expo Application Services) Build is the cloud build service for Expo and React Native apps. It solves the hardest problem in mobile CI: **you need macOS to build for iOS, and macOS CI runners are expensive.** EAS offloads the build to Expo's cloud, runs it on their infrastructure, and gives you back a signed binary.

### 3.1 EAS Build Profiles

Your `eas.json` defines build profiles — configurations that tell EAS how to build your app for different scenarios:

```json
{
  "cli": {
    "version": ">= 12.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      },
      "env": {
        "APP_ENV": "development"
      },
      "channel": "development"
    },
    "preview": {
      "distribution": "internal",
      "ios": {
        "resourceClass": "m-medium"
      },
      "android": {
        "buildType": "apk"
      },
      "env": {
        "APP_ENV": "preview"
      },
      "channel": "preview",
      "autoIncrement": true
    },
    "production": {
      "ios": {
        "resourceClass": "m-medium"
      },
      "env": {
        "APP_ENV": "production"
      },
      "channel": "production",
      "autoIncrement": true
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "team@yourcompany.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./google-service-account.json",
        "track": "internal"
      }
    }
  }
}
```

**The profile strategy:**

| Profile | Trigger | Distribution | Purpose |
|---------|---------|-------------|---------|
| `development` | Manual | Internal (dev client) | Local development with custom native code |
| `preview` | PR merge to `develop` | Internal (TestFlight / APK) | QA testing before production |
| `production` | Tag or manual | Store (App Store / Play Store) | Production release |

### 3.2 Triggering EAS Build from GitHub Actions

Here's the workflow that triggers an EAS build when a PR is opened or updated, but only if mobile code changed:

```yaml
# .github/workflows/eas-preview.yml
name: EAS Preview Build

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'apps/mobile/**'
      - 'packages/ui/**'
      - 'packages/shared/**'
      - 'packages/api-client/**'

concurrency:
  group: eas-preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [ios, android]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build
        working-directory: apps/mobile
        run: |
          eas build \
            --profile preview \
            --platform ${{ matrix.platform }} \
            --non-interactive \
            --no-wait \
            --message "PR #${{ github.event.pull_request.number }}: ${{ github.event.pull_request.title }}"

      - name: Comment build link on PR
        if: matrix.platform == 'ios'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '📱 EAS builds triggered! Check progress at https://expo.dev/accounts/${{ vars.EXPO_ACCOUNT }}/projects/${{ vars.EXPO_PROJECT }}/builds'
            })
```

**Key decisions explained:**

- **`--no-wait`**: The GitHub Actions job finishes immediately after triggering the EAS build. The actual build runs on EAS infrastructure. This means your CI minutes stay low (the GH job runs for ~1 minute), and the 15-minute iOS build happens on EAS, not on your dime.
- **`--non-interactive`**: Required in CI. Without it, EAS will prompt for input.
- **Path filters**: Only triggers when mobile-related code changes.
- **Concurrency group per PR**: A new push cancels the previous build trigger (but note: the EAS build itself isn't canceled — you'd need the EAS API for that).

### 3.3 Production Builds on Merge

```yaml
# .github/workflows/eas-production.yml
name: EAS Production Build

on:
  push:
    branches: [main]
    paths:
      - 'apps/mobile/**'
      - 'packages/ui/**'
      - 'packages/shared/**'
      - 'packages/api-client/**'

concurrency:
  group: eas-production
  cancel-in-progress: false  # Never cancel a production build

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [ios, android]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build and Submit
        working-directory: apps/mobile
        run: |
          eas build \
            --profile production \
            --platform ${{ matrix.platform }} \
            --non-interactive \
            --auto-submit
```

The `--auto-submit` flag is powerful: after the build succeeds, EAS automatically submits it to the App Store (TestFlight) and Google Play (internal track). Your pipeline goes from "merge to main" to "binary in TestFlight" with zero human intervention.

### 3.4 Managing Secrets

EAS needs several secrets to build and sign your app. Here's where each one lives:

```
┌──────────────────────────────────────────────────────────────┐
│  SECRET MANAGEMENT FOR MOBILE CI                              │
│                                                               │
│  GitHub Actions Secrets:                                      │
│  ├── EXPO_TOKEN          → Authenticates eas-cli              │
│  └── (that's it for EAS — EAS manages the rest)              │
│                                                               │
│  EAS Secrets (managed via eas secret:create):                │
│  ├── iOS signing         → Managed by EAS automatically      │
│  │                         (credentials service)              │
│  ├── Android keystore    → Uploaded once, stored in EAS      │
│  ├── SENTRY_AUTH_TOKEN   → For source map uploads            │
│  ├── API_URL             → Per-environment API endpoints     │
│  └── GOOGLE_SERVICES     → Firebase config (base64 encoded)  │
│                                                               │
│  NEVER put in repo:                                           │
│  ├── .p12 / .mobileprovision files                           │
│  ├── Android keystore files                                   │
│  └── App Store Connect API keys                              │
└──────────────────────────────────────────────────────────────┘
```

```bash
# Set EAS secrets from the command line
eas secret:create --scope project --name SENTRY_AUTH_TOKEN --value "sntrys_..."
eas secret:create --scope project --name API_URL --value "https://api.yourapp.com"

# Or use environment-specific secrets in eas.json
# The env block in each profile maps to EAS secrets
```

**One thing that trips people up:** EAS manages iOS signing credentials automatically through its credentials service. You don't need to deal with provisioning profiles, certificates, or p12 files. When you run `eas build` for the first time, it generates and stores everything. This alone saves hours of DevOps pain compared to managing signing manually.

### 3.5 EAS Fingerprint and Smart Rebuilds

EAS has a feature called fingerprint-based builds that avoids redundant native builds. The fingerprint is a hash of everything that affects the native build: native dependencies, app.json config, Expo SDK version, native code, etc.

If the fingerprint hasn't changed since the last build, you don't need a new native build — you can use EAS Update (OTA) instead. This is a massive time and cost saver.

```yaml
# .github/workflows/smart-deploy.yml
name: Smart Deploy

on:
  push:
    branches: [main]

jobs:
  check-fingerprint:
    runs-on: ubuntu-latest
    outputs:
      native-changed: ${{ steps.fingerprint.outputs.changed }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile

      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Check fingerprint
        id: fingerprint
        working-directory: apps/mobile
        run: |
          CURRENT=$(npx expo-updates fingerprint:generate --platform ios | jq -r '.hash')
          PREVIOUS=$(eas build:list --profile production --platform ios --limit 1 --json | jq -r '.[0].runtimeVersion')
          if [ "$CURRENT" != "$PREVIOUS" ]; then
            echo "changed=true" >> $GITHUB_OUTPUT
          else
            echo "changed=false" >> $GITHUB_OUTPUT
          fi

  native-build:
    needs: check-fingerprint
    if: needs.check-fingerprint.outputs.native-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Build native
        working-directory: apps/mobile
        run: eas build --profile production --platform all --non-interactive

  ota-update:
    needs: check-fingerprint
    if: needs.check-fingerprint.outputs.native-changed == 'false'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - name: Publish OTA update
        working-directory: apps/mobile
        run: |
          eas update \
            --branch production \
            --message "$(git log -1 --pretty=%B)" \
            --non-interactive
```

**The impact is dramatic.** A native build takes 15-20 minutes and costs an EAS build credit. An OTA update takes 1-2 minutes and is essentially free. If 80% of your changes are JS-only (new features, bug fixes, UI tweaks), you skip 80% of your native builds.

---

## 4. WEB CI WITH VERCEL

For the web side of your monorepo, Vercel's GitHub integration handles most of the CI/CD story automatically. But there are patterns and configurations that make a big difference.

### 4.1 Automatic Preview Deployments

When you connect your GitHub repo to Vercel, every PR automatically gets a preview deployment. This is one of Vercel's killer features — reviewers can see the actual running application, not just code diffs.

The default behavior is:
- PR opened/updated -> Preview deployment
- Merge to main -> Production deployment

But in a monorepo, you need to tell Vercel which directory contains the web app:

```json
// vercel.json (in the repo root)
{
  "buildCommand": "cd ../.. && npx turbo build --filter=@repo/web",
  "installCommand": "cd ../.. && yarn install --frozen-lockfile",
  "framework": "nextjs",
  "outputDirectory": "apps/web/.next"
}
```

Or configure it in the Vercel dashboard under Project Settings -> General -> Root Directory: `apps/web`.

### 4.2 Ignoring Builds for Unrelated Changes

Vercel rebuilds on every push by default. In a monorepo, that means a change to `apps/mobile` triggers a web rebuild. Fix this with Vercel's Ignored Build Step:

```bash
#!/bin/bash
# vercel-ignore-build.sh
# Place in repo root or configure in Vercel dashboard

echo "VERCEL_GIT_COMMIT_REF: $VERCEL_GIT_COMMIT_REF"

# Always build on main
if [ "$VERCEL_GIT_COMMIT_REF" = "main" ]; then
  echo "Building: main branch"
  exit 1  # 1 = proceed with build
fi

# Check if web-related files changed
git diff HEAD~1 --name-only | grep -qE '^(apps/web/|packages/ui/|packages/shared/|packages/api-client/)'

if [ $? -eq 0 ]; then
  echo "Building: web-related files changed"
  exit 1  # 1 = proceed with build
else
  echo "Skipping: no web-related changes"
  exit 0  # 0 = skip build
fi
```

Configure in Vercel dashboard: Project Settings -> Git -> Ignored Build Step -> Run Script: `bash vercel-ignore-build.sh`

Or use Turborepo's built-in approach:

```bash
npx turbo-ignore @repo/web
```

This checks if anything in `@repo/web`'s dependency graph changed. If not, it exits 0 (skip). Simple and correct.

### 4.3 Environment Variables Per Deployment

Vercel lets you set environment variables per environment (Production, Preview, Development):

```
┌────────────────────────────────────────────────────┐
│  ENVIRONMENT VARIABLES BY DEPLOYMENT                │
│                                                     │
│  Production (main):                                 │
│  ├── NEXT_PUBLIC_API_URL = https://api.myapp.com   │
│  ├── DATABASE_URL = postgres://prod:...             │
│  └── NEXT_PUBLIC_ANALYTICS_ID = G-PROD123          │
│                                                     │
│  Preview (PRs):                                     │
│  ├── NEXT_PUBLIC_API_URL = https://staging.myapp.com│
│  ├── DATABASE_URL = postgres://staging:...          │
│  └── NEXT_PUBLIC_ANALYTICS_ID = G-STAGING456       │
│                                                     │
│  Development (vercel dev):                          │
│  ├── NEXT_PUBLIC_API_URL = http://localhost:3001    │
│  ├── DATABASE_URL = postgres://local:...            │
│  └── NEXT_PUBLIC_ANALYTICS_ID = (empty)             │
└────────────────────────────────────────────────────┘
```

**Pro tip:** Use `vercel env pull` to sync these to your local `.env.local`:

```bash
vercel env pull .env.local --environment=development
```

This ensures local development uses the same variable names and structure as CI, preventing the "works on my machine" problem for environment-dependent features.

### 4.4 Combining Vercel with GitHub Actions

Sometimes you need more than Vercel's built-in CI. Maybe you want to run tests before Vercel builds, or you want to post Lighthouse scores on the PR. Here's how to combine them:

```yaml
# .github/workflows/web-checks.yml
name: Web Checks

on:
  pull_request:
    paths:
      - 'apps/web/**'
      - 'packages/ui/**'
      - 'packages/shared/**'

jobs:
  web-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo test --filter=@repo/web

  lighthouse:
    runs-on: ubuntu-latest
    needs: web-tests
    steps:
      - uses: actions/checkout@v4
      - name: Wait for Vercel preview
        uses: patrickedqvist/wait-for-vercel-preview@v1.3.2
        id: vercel
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          max_timeout: 600

      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v12
        with:
          urls: ${{ steps.vercel.outputs.url }}
          configPath: './lighthouserc.json'
          uploadArtifacts: true

      - name: Comment results
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('.lighthouseci/lhr-*.json', 'utf8'));
            const perf = Math.round(results.categories.performance.score * 100);
            const a11y = Math.round(results.categories.accessibility.score * 100);
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Lighthouse Results\n| Metric | Score |\n|--------|-------|\n| Performance | ${perf} |\n| Accessibility | ${a11y} |`
            });
```

This runs your tests in GitHub Actions (where you have full control) and then uses Vercel's preview deployment for Lighthouse testing. Best of both worlds.

---

## 5. TESTING IN CI

Chapter 15 covered testing strategy in depth. Here we'll focus specifically on how to run those tests efficiently in CI.

### 5.1 Jest Unit Tests

```yaml
test-unit:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'

    - run: yarn install --frozen-lockfile

    # Run tests only for changed packages
    - name: Run unit tests
      run: |
        yarn turbo test \
          --filter='...[origin/main]' \
          -- --ci --coverage --maxWorkers=2

    # Upload coverage for PR comment
    - name: Upload coverage
      if: github.event_name == 'pull_request'
      uses: actions/upload-artifact@v4
      with:
        name: coverage
        path: '**/coverage/lcov.info'
```

**The `--maxWorkers=2` flag matters.** GitHub-hosted runners have 2 vCPUs. Setting `maxWorkers` higher than the CPU count causes context-switching overhead and actually slows tests down. I've seen teams cut their test time by 30% just by setting this correctly.

### 5.2 React Native Testing Library Tests

Same as unit tests, but with a few React Native-specific considerations:

```yaml
test-mobile:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'

    - run: yarn install --frozen-lockfile

    - name: Run mobile tests
      run: |
        yarn turbo test \
          --filter=@repo/mobile \
          -- --ci --maxWorkers=2
      env:
        # Some RN tests need this
        TZ: UTC
```

### 5.3 E2E Tests with Maestro (Mobile)

Maestro is the E2E testing tool that has taken over mobile testing. It's simpler than Detox, more reliable than Appium, and actually works in CI:

```yaml
# .github/workflows/e2e-mobile.yml
name: Mobile E2E Tests

on:
  pull_request:
    paths:
      - 'apps/mobile/**'
      - 'packages/ui/**'

jobs:
  e2e-ios:
    runs-on: macos-14  # M1 runner for iOS simulator
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      # Install Maestro
      - name: Install Maestro
        run: |
          curl -Ls "https://get.maestro.mobile.dev" | bash
          echo "$HOME/.maestro/bin" >> $GITHUB_PATH

      # Build the app for simulator
      - name: Build iOS (simulator)
        working-directory: apps/mobile
        run: |
          npx expo prebuild --platform ios --clean
          xcodebuild \
            -workspace ios/*.xcworkspace \
            -scheme YourApp \
            -configuration Debug \
            -sdk iphonesimulator \
            -destination 'platform=iOS Simulator,name=iPhone 15' \
            -derivedDataPath build \
            build

      # Boot simulator
      - name: Boot Simulator
        run: |
          xcrun simctl boot "iPhone 15" || true
          xcrun simctl install "iPhone 15" apps/mobile/build/Build/Products/Debug-iphonesimulator/YourApp.app

      # Run Maestro tests
      - name: Run E2E tests
        run: |
          xcrun simctl launch "iPhone 15" com.yourcompany.yourapp
          maestro test apps/mobile/maestro/ --format junit --output maestro-results.xml

      # Upload results
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: maestro-results
          path: maestro-results.xml

      # Post results on PR
      - name: Publish test results
        if: always() && github.event_name == 'pull_request'
        uses: dorny/test-reporter@v1
        with:
          name: Maestro E2E Results
          path: maestro-results.xml
          reporter: java-junit
```

**Or use Maestro Cloud**, which is significantly simpler:

```yaml
e2e-maestro-cloud:
  runs-on: ubuntu-latest
  needs: build
  steps:
    - uses: actions/checkout@v4

    - name: Download build artifact
      uses: actions/download-artifact@v4
      with:
        name: preview-build

    - name: Run Maestro Cloud
      uses: mobile-dev-inc/action-maestro-cloud@v1
      with:
        api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
        app-file: app-preview.apk
        workspace: apps/mobile/maestro/
```

Maestro Cloud runs your tests on their infrastructure, against your APK or IPA. No simulator management, no macOS runner costs. The trade-off is another service to pay for — but for most teams, it's worth it.

### 5.4 E2E Tests with Playwright (Web)

```yaml
e2e-web:
  runs-on: ubuntu-latest
  needs: [test-unit, build-web]
  steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'

    - run: yarn install --frozen-lockfile

    - name: Install Playwright browsers
      run: npx playwright install --with-deps chromium

    - name: Build web app
      run: yarn turbo build --filter=@repo/web

    - name: Run Playwright tests
      run: |
        yarn turbo test:e2e --filter=@repo/web
      env:
        BASE_URL: http://localhost:3000
        CI: true

    - name: Upload Playwright report
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: playwright-report
        path: apps/web/playwright-report/
```

### 5.5 Parallel Test Execution

For large test suites, sharding across multiple runners cuts wall-clock time dramatically:

```yaml
test-unit:
  runs-on: ubuntu-latest
  strategy:
    matrix:
      shard: [1, 2, 3, 4]
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'
    - run: yarn install --frozen-lockfile
    - run: |
        yarn jest \
          --ci \
          --shard=${{ matrix.shard }}/4 \
          --coverage

# For Playwright
test-e2e:
  runs-on: ubuntu-latest
  strategy:
    matrix:
      shard: [1, 2, 3, 4]
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'
    - run: yarn install --frozen-lockfile
    - run: npx playwright install --with-deps chromium
    - run: |
        npx playwright test \
          --shard=${{ matrix.shard }}/4
```

**The math:** A 12-minute test suite split across 4 shards runs in ~3 minutes. You use the same total compute, but the developer gets feedback 4x faster. The cost is identical (4 runners x 3 min = 12 runner-minutes), but developer time is worth far more than CI minutes.

### 5.6 The Fail-Fast Testing Strategy

Structure your tests to give the fastest possible feedback:

```yaml
jobs:
  # Level 1: Fastest checks (< 30s)
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo lint typecheck --filter='...[origin/main]'

  # Level 2: Unit tests (1-3 min)
  unit-tests:
    needs: static-analysis
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: |
          yarn turbo test --filter='...[origin/main]' \
            -- --ci --shard=${{ matrix.shard }}/2

  # Level 3: Integration tests (3-5 min)
  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: yarn turbo test:integration --filter='...[origin/main]'

  # Level 4: E2E (5-15 min) - only if everything else passes
  e2e:
    needs: integration-tests
    # Only run E2E on PRs targeting main, not on every push
    if: github.event_name == 'pull_request' && github.base_ref == 'main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - run: npx playwright install --with-deps chromium
      - run: yarn turbo test:e2e --filter=@repo/web
```

---

## 6. PERFORMANCE BUDGETS

A performance budget is a contract: "this build artifact will not exceed these limits." It's not a suggestion. It's a CI check that blocks the merge. Without budgets, performance regresses one small PR at a time — each individual change looks harmless, but over six months the bundle has grown 40% and nobody noticed because nobody was measuring.

### 6.1 Bundle Size Budgets

#### Web (Next.js)

Next.js has built-in bundle analysis. Add a size check to CI:

```js
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    // Enable build output analysis
  },
};

module.exports = nextConfig;
```

Use `@next/bundle-analyzer` for detailed reporting and a custom check for enforcement:

```json
// apps/web/package.json
{
  "scripts": {
    "build": "next build",
    "analyze": "ANALYZE=true next build",
    "check:bundle-size": "node scripts/check-bundle-size.js"
  }
}
```

```js
// apps/web/scripts/check-bundle-size.js
const fs = require('fs');
const path = require('path');

// Budget: max sizes in bytes
const BUDGETS = {
  'First Load JS (shared)': 90 * 1024,   // 90 KB
  '/_app': 120 * 1024,                     // 120 KB
  '/': 150 * 1024,                          // 150 KB
  '/dashboard': 200 * 1024,                 // 200 KB
};

// Read the Next.js build manifest
const buildDir = path.join(__dirname, '..', '.next');
const buildManifest = JSON.parse(
  fs.readFileSync(path.join(buildDir, 'build-manifest.json'), 'utf8')
);

// Check each route's bundle size
let failed = false;
const results = [];

for (const [route, maxSize] of Object.entries(BUDGETS)) {
  const pages = buildManifest.pages[route];
  if (!pages) continue;

  let totalSize = 0;
  for (const file of pages) {
    const filePath = path.join(buildDir, file);
    if (fs.existsSync(filePath)) {
      totalSize += fs.statSync(filePath).size;
    }
  }

  const overBudget = totalSize > maxSize;
  if (overBudget) failed = true;

  results.push({
    route,
    size: totalSize,
    budget: maxSize,
    status: overBudget ? 'FAIL' : 'PASS',
  });
}

console.table(results.map(r => ({
  Route: r.route,
  Size: `${(r.size / 1024).toFixed(1)} KB`,
  Budget: `${(r.budget / 1024).toFixed(1)} KB`,
  Status: r.status,
})));

if (failed) {
  console.error('\nBundle size budget exceeded! See table above.');
  process.exit(1);
}

console.log('\nAll bundle size budgets passed.');
```

#### React Native

For React Native, measure the JS bundle size:

```json
// apps/mobile/package.json
{
  "scripts": {
    "bundle:ios": "npx react-native bundle --platform ios --dev false --entry-file index.js --bundle-output /tmp/ios.bundle",
    "bundle:android": "npx react-native bundle --platform android --dev false --entry-file index.js --bundle-output /tmp/android.bundle",
    "check:bundle-size": "node scripts/check-rn-bundle.js"
  }
}
```

```js
// apps/mobile/scripts/check-rn-bundle.js
const fs = require('fs');

const MAX_BUNDLE_SIZE = 2.5 * 1024 * 1024; // 2.5 MB

const platforms = [
  { name: 'iOS', path: '/tmp/ios.bundle' },
  { name: 'Android', path: '/tmp/android.bundle' },
];

let failed = false;

for (const platform of platforms) {
  if (!fs.existsSync(platform.path)) {
    console.log(`${platform.name}: Bundle not found, skipping`);
    continue;
  }

  const size = fs.statSync(platform.path).size;
  const sizeMB = (size / (1024 * 1024)).toFixed(2);
  const budgetMB = (MAX_BUNDLE_SIZE / (1024 * 1024)).toFixed(2);
  const overBudget = size > MAX_BUNDLE_SIZE;

  if (overBudget) failed = true;

  console.log(
    `${platform.name}: ${sizeMB} MB / ${budgetMB} MB ${overBudget ? '!! OVER BUDGET' : '(ok)'}`
  );
}

if (failed) {
  console.error('\nReact Native bundle size budget exceeded!');
  process.exit(1);
}
```

### 6.2 Lighthouse Score Thresholds

```json
// lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "categories:accessibility": ["error", { "minScore": 0.95 }],
        "categories:best-practices": ["warn", { "minScore": 0.9 }],
        "categories:seo": ["warn", { "minScore": 0.9 }],
        "first-contentful-paint": ["error", { "maxNumericValue": 1800 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "interactive": ["error", { "maxNumericValue": 3800 }],
        "total-blocking-time": ["error", { "maxNumericValue": 300 }]
      }
    },
    "collect": {
      "numberOfRuns": 3,
      "settings": {
        "preset": "desktop"
      }
    }
  }
}
```

**Important: Run Lighthouse 3 times and take the median.** CI environments are noisy — a single run can vary by 10-15 points depending on what else is happening on the runner. Three runs with median gives you a stable, reproducible score.

```yaml
# In your workflow
- name: Run Lighthouse
  uses: treosh/lighthouse-ci-action@v12
  with:
    urls: |
      ${{ steps.vercel.outputs.url }}
      ${{ steps.vercel.outputs.url }}/dashboard
      ${{ steps.vercel.outputs.url }}/pricing
    configPath: './lighthouserc.json'
    uploadArtifacts: true
    temporaryPublicStorage: true
```

### 6.3 Build Time Budgets

This one is less common but equally important. If your CI pipeline takes more than X minutes, something is wrong — either a cache broke, a dependency bloated, or someone added a heavy build step:

```yaml
# In your workflow, set a timeout on the entire job
build-web:
  runs-on: ubuntu-latest
  timeout-minutes: 10  # If this takes > 10 min, fail loudly
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: 20
        cache: 'yarn'
    - run: yarn install --frozen-lockfile

    - name: Build with timing
      run: |
        START=$(date +%s)
        yarn turbo build --filter=@repo/web
        END=$(date +%s)
        DURATION=$((END - START))
        echo "Build took ${DURATION}s"
        
        # Budget: 180 seconds (3 minutes)
        if [ $DURATION -gt 180 ]; then
          echo "::error::Build time budget exceeded: ${DURATION}s > 180s"
          exit 1
        fi
```

### 6.4 The Budget Dashboard

Post all budgets as a PR comment so every reviewer sees the impact:

```yaml
- name: Post budget report
  if: github.event_name == 'pull_request'
  uses: actions/github-script@v7
  with:
    script: |
      const budgets = {
        webBundle: '87.2 KB / 90 KB',
        rnBundle: '2.1 MB / 2.5 MB',
        lighthouse: '94 / 90 (min)',
        buildTime: '142s / 180s (max)',
      };
      
      const body = `### Performance Budget Report
      
      | Metric | Value | Budget | Status |
      |--------|-------|--------|--------|
      | Web Bundle (shared) | 87.2 KB | 90 KB | :white_check_mark: |
      | RN Bundle (iOS) | 2.1 MB | 2.5 MB | :white_check_mark: |
      | Lighthouse Perf | 94 | >= 90 | :white_check_mark: |
      | Build Time | 142s | <= 180s | :white_check_mark: |
      
      All budgets passed.`;
      
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body
      });
```

**Airbnb pioneered this approach.** They post bundle size diffs on every PR, showing exactly how much each change adds to or removes from the bundle. Engineers see the cost of their code before it merges. Over time, this created a culture where bundle size is a first-class concern, not an afterthought.

---

## 7. SELF-HOSTED CI FOR MOBILE

EAS Build is the right choice for most teams. But there are legitimate reasons to run your own mobile CI:

- **Regulatory requirements** that prohibit sending source code to third-party build services
- **Custom native modules** that require specific build toolchains not available on EAS
- **Cost optimization** at scale (50+ builds/day makes self-hosted cheaper)
- **Build time optimization** — self-hosted M1/M2 machines with SSDs can be significantly faster than cloud runners

### 7.1 Fastlane for iOS

Fastlane is the standard tool for automating iOS (and Android) build, signing, and distribution. Here's a complete setup:

```ruby
# apps/mobile/ios/fastlane/Fastfile

default_platform(:ios)

platform :ios do
  before_all do
    setup_ci if ENV['CI']
  end

  desc "Build the app for TestFlight"
  lane :beta do
    # Sync signing from a private git repo
    match(
      type: "appstore",
      app_identifier: "com.yourcompany.yourapp",
      readonly: true,
      git_url: "git@github.com:yourcompany/certificates.git",
    )

    # Increment build number
    increment_build_number(
      build_number: ENV['BUILD_NUMBER'] || latest_testflight_build_number + 1,
      xcodeproj: "YourApp.xcodeproj"
    )

    # Build
    build_app(
      workspace: "YourApp.xcworkspace",
      scheme: "YourApp",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.yourcompany.yourapp" => "match AppStore com.yourcompany.yourapp"
        }
      },
      output_directory: "./build",
      output_name: "YourApp.ipa"
    )

    # Upload to TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      api_key_path: "fastlane/api_key.json"
    )

    # Upload source maps to Sentry
    sentry_upload_dsym(
      auth_token: ENV['SENTRY_AUTH_TOKEN'],
      org_slug: 'your-org',
      project_slug: 'your-project',
    )
  end

  desc "Build for simulator (CI testing)"
  lane :build_simulator do
    build_app(
      workspace: "YourApp.xcworkspace",
      scheme: "YourApp",
      configuration: "Debug",
      destination: "generic/platform=iOS Simulator",
      derived_data_path: "./build",
      skip_archive: true,
    )
  end
end
```

```ruby
# apps/mobile/ios/fastlane/Matchfile
git_url("git@github.com:yourcompany/certificates.git")
storage_mode("git")
type("appstore")
app_identifier(["com.yourcompany.yourapp"])
```

### 7.2 Fastlane for Android

```ruby
# apps/mobile/android/fastlane/Fastfile

default_platform(:android)

platform :android do
  desc "Build and upload to Play Store internal track"
  lane :beta do
    # Clean
    gradle(
      task: "clean",
      project_dir: "."
    )

    # Build AAB
    gradle(
      task: "bundle",
      build_type: "Release",
      project_dir: ".",
      properties: {
        "android.injected.signing.store.file" => ENV['KEYSTORE_PATH'],
        "android.injected.signing.store.password" => ENV['KEYSTORE_PASSWORD'],
        "android.injected.signing.key.alias" => ENV['KEY_ALIAS'],
        "android.injected.signing.key.password" => ENV['KEY_PASSWORD'],
      }
    )

    # Upload to Play Store
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      json_key: ENV['GOOGLE_PLAY_JSON_KEY'],
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end

  desc "Build APK for testing"
  lane :build_debug do
    gradle(
      task: "assemble",
      build_type: "Debug",
      project_dir: "."
    )
  end
end
```

### 7.3 Self-Hosted macOS Runners

For iOS builds, you need macOS. GitHub's macOS runners work but are expensive and slow. Self-hosted runners on Mac hardware are faster and cheaper at scale:

```yaml
# .github/workflows/ios-self-hosted.yml
name: iOS Build (Self-Hosted)

on:
  push:
    branches: [main]
    paths:
      - 'apps/mobile/**'

jobs:
  build-ios:
    runs-on: [self-hosted, macOS, ARM64]  # Your M1/M2 Mac
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.app

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: yarn install --frozen-lockfile

      - name: Install CocoaPods
        working-directory: apps/mobile/ios
        run: |
          bundle install
          bundle exec pod install

      - name: Build with Fastlane
        working-directory: apps/mobile/ios
        run: bundle exec fastlane beta
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_PRIVATE_KEY: ${{ secrets.MATCH_GIT_PRIVATE_KEY }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
```

**Setting up the runner:**

```bash
# On your Mac mini / Mac Studio
# 1. Download the runner from GitHub (Settings -> Actions -> Runners -> New self-hosted runner)
# 2. Configure it
./config.sh \
  --url https://github.com/yourcompany/yourrepo \
  --token YOUR_RUNNER_TOKEN \
  --labels macOS,ARM64 \
  --name "mac-builder-1"

# 3. Install as a service
./svc.sh install
./svc.sh start

# 4. Keep it updated
# Set up a cron job to update Xcode, CocoaPods, etc.
```

### 7.4 The Cost/Complexity Trade-Off

| Factor | EAS Build | Self-Hosted | GitHub macOS Runner |
|--------|----------|------------|-------------------|
| Setup time | 5 minutes | 2-4 hours | 0 (just YAML) |
| Maintenance | None (managed) | Weekly updates | None |
| iOS signing | Automatic | Manual (Match) | Manual (Match) |
| Build time | 12-20 min | 8-15 min | 20-30 min |
| Cost (100 builds/mo) | ~$100 (Production plan) | ~$50 (power bill) + hardware | ~$400 |
| Cost (500 builds/mo) | ~$300 | ~$50 | ~$2,000 |
| Scalability | Unlimited (queued) | Limited by hardware | Limited by billing |
| Security | Expo manages infra | You manage everything | GitHub manages infra |

**My recommendation:**

- **< 100 builds/month**: EAS Build. The managed experience is worth the cost.
- **100-500 builds/month**: EAS Build, unless you have specific requirements that force self-hosted.
- **500+ builds/month**: Consider self-hosted for iOS (the savings are real), keep using EAS or GitHub Actions for Android (Linux builds are cheap).
- **Regulated industries**: Self-hosted is often required. Accept the maintenance burden.

---

## 8. SEMANTIC VERSIONING FOR MOBILE

Mobile versioning is more complex than web versioning because you're dealing with two version systems simultaneously: the user-facing version (what users see in the App Store) and the build number (what the store uses to order builds).

### 8.1 The Two Version Numbers

```
┌────────────────────────────────────────────────────────────┐
│  MOBILE VERSION NUMBERS                                     │
│                                                             │
│  iOS:                                                       │
│  ├── CFBundleShortVersionString = "2.4.1"                  │
│  │   (User-facing version, shown in App Store)             │
│  └── CFBundleVersion = "147"                                │
│      (Build number, must increment for each TestFlight      │
│       upload, not shown to users)                           │
│                                                             │
│  Android:                                                   │
│  ├── versionName = "2.4.1"                                 │
│  │   (User-facing version, shown in Play Store)            │
│  └── versionCode = 147                                      │
│      (Build number, must be strictly increasing integer,    │
│       used for update ordering)                             │
│                                                             │
│  Expo (app.json / app.config.ts):                          │
│  ├── version = "2.4.1"         → maps to both versionName │
│  │                               and CFBundleShortVersion  │
│  ├── ios.buildNumber = "147"   → CFBundleVersion           │
│  └── android.versionCode = 147 → versionCode              │
└────────────────────────────────────────────────────────────┘
```

**The rules that will save you from store rejections:**

1. `versionCode` / `buildNumber` must be **strictly increasing** for each store upload. You cannot re-upload with the same or lower number.
2. `version` / `versionName` follows semver and is what users see. You can have multiple builds (build numbers) for the same version.
3. On iOS, `buildNumber` is per-version — you can reset it when you bump the version. On Android, `versionCode` is global and must always go up.

### 8.2 Changesets for Version Management

Changesets is a tool from the Atlassian team that automates version bumping and changelog generation. It works beautifully in monorepos because it tracks which packages changed and what kind of change it was (major, minor, patch).

```bash
# Install
yarn add -D @changesets/cli @changesets/changelog-github

# Initialize
npx changeset init
```

```json
// .changeset/config.json
{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": [
    "@changesets/changelog-github",
    { "repo": "yourcompany/yourrepo" }
  ],
  "commit": false,
  "fixed": [],
  "linked": [
    ["@repo/mobile", "@repo/web"]
  ],
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

**The workflow:**

1. Developer makes changes and creates a changeset:

```bash
npx changeset
# Prompts:
# Which packages changed? @repo/mobile
# What kind of change? patch
# Summary: Fix crash on profile screen when user has no avatar
```

This creates a file like `.changeset/brave-dogs-dance.md`:

```markdown
---
'@repo/mobile': patch
---

Fix crash on profile screen when user has no avatar
```

2. When merged to main, a GitHub Action consumes the changesets and creates a "Version Packages" PR:

```yaml
# .github/workflows/version.yml
name: Version

on:
  push:
    branches: [main]

concurrency:
  group: version-${{ github.ref }}

jobs:
  version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Create Release PR or Publish
        uses: changesets/action@v1
        with:
          title: 'chore: version packages'
          commit: 'chore: version packages'
          version: yarn changeset version
          publish: yarn release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

3. When you merge the "Version Packages" PR, it bumps versions and triggers the release.

### 8.3 Automating Mobile Version Bumps

Here's where it gets mobile-specific. Changesets bumps `package.json` versions. But you also need to update `app.json` (or `app.config.ts`) for Expo, and potentially native files.

```ts
// apps/mobile/app.config.ts
import { ExpoConfig } from 'expo/config';
import { version } from './package.json';

const IS_PRODUCTION = process.env.APP_ENV === 'production';

const config: ExpoConfig = {
  name: IS_PRODUCTION ? 'YourApp' : 'YourApp (Dev)',
  slug: 'your-app',
  version, // Pulled from package.json — Changesets manages this!
  ios: {
    bundleIdentifier: 'com.yourcompany.yourapp',
    buildNumber: process.env.BUILD_NUMBER || '1',
  },
  android: {
    package: 'com.yourcompany.yourapp',
    versionCode: parseInt(process.env.BUILD_NUMBER || '1', 10),
  },
  // ...
};

export default config;
```

**The key insight:** `version` comes from `package.json` (managed by Changesets), and `buildNumber`/`versionCode` comes from the build system (auto-incremented by EAS or your CI).

With EAS, set `"appVersionSource": "remote"` in `eas.json` to let EAS auto-increment build numbers:

```json
{
  "cli": {
    "appVersionSource": "remote"
  },
  "build": {
    "production": {
      "autoIncrement": true
    }
  }
}
```

EAS tracks the latest build number server-side and increments it for each build. You never have to think about build numbers again.

### 8.4 The Complete Version Flow

```
┌────────────────────────────────────────────────────────┐
│  VERSION FLOW                                           │
│                                                         │
│  1. Developer creates changeset on feature branch       │
│     npx changeset → .changeset/brave-dogs-dance.md     │
│                                                         │
│  2. Feature branch merged to main                       │
│     Changesets Action creates "Version Packages" PR     │
│     - Bumps package.json: 2.4.0 → 2.4.1               │
│     - Generates CHANGELOG.md entry                      │
│     - Commits changes                                   │
│                                                         │
│  3. "Version Packages" PR merged                        │
│     - app.config.ts reads new version from package.json │
│     - CI triggers EAS production build                  │
│     - EAS auto-increments buildNumber: 146 → 147       │
│     - Build submitted to stores                         │
│                                                         │
│  Result:                                                │
│  - App Store: v2.4.1 (build 147)                       │
│  - Play Store: v2.4.1 (versionCode 147)                │
│  - CHANGELOG.md has the entry                           │
│  - Git tag: @repo/mobile@2.4.1                         │
└────────────────────────────────────────────────────────┘
```

---

## 9. THE COMPLETE PIPELINE

Let's put it all together. Here's a production-grade CI/CD pipeline for a monorepo with a React Native (Expo) mobile app and a Next.js web app. One workflow file, handling the full lifecycle from lint to deploy.

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

# ─────────────────────────────────────────────────────────────
# STAGE 0: Determine what changed
# ─────────────────────────────────────────────────────────────
jobs:
  changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    outputs:
      mobile: ${{ steps.filter.outputs.mobile }}
      web: ${{ steps.filter.outputs.web }}
      any-code: ${{ steps.filter.outputs.any-code }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            mobile:
              - 'apps/mobile/**'
              - 'packages/ui/**'
              - 'packages/shared/**'
              - 'packages/api-client/**'
            web:
              - 'apps/web/**'
              - 'packages/ui/**'
              - 'packages/shared/**'
              - 'packages/api-client/**'
            any-code:
              - '**/*.ts'
              - '**/*.tsx'
              - '**/*.js'
              - '**/*.jsx'
              - '**/*.json'
              - '!**/*.md'

  # ─────────────────────────────────────────────────────────────
  # STAGE 1: Static Analysis (< 1 minute)
  # ─────────────────────────────────────────────────────────────
  static-analysis:
    name: Lint & Typecheck
    needs: changes
    if: needs.changes.outputs.any-code == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Lint
        run: yarn turbo lint --filter='...[origin/main]'

      - name: Typecheck
        run: yarn turbo typecheck --filter='...[origin/main]'

      - name: Format check
        run: yarn prettier --check "**/*.{ts,tsx,js,jsx,json,css,md}"

  # ─────────────────────────────────────────────────────────────
  # STAGE 2: Unit & Integration Tests (2-5 minutes)
  # ─────────────────────────────────────────────────────────────
  unit-tests:
    name: Unit Tests (shard ${{ matrix.shard }})
    needs: static-analysis
    runs-on: ubuntu-latest
    strategy:
      fail-fast: true
      matrix:
        shard: [1, 2]
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Run tests
        run: |
          yarn turbo test \
            --filter='...[origin/main]' \
            -- --ci --coverage --maxWorkers=2 --shard=${{ matrix.shard }}/2

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.shard }}
          path: '**/coverage/lcov.info'

  # ─────────────────────────────────────────────────────────────
  # STAGE 3: Builds
  # ─────────────────────────────────────────────────────────────

  # Web Build (handled by Vercel automatically on PR)
  # This job only runs checks — Vercel deploys independently
  build-web:
    name: Build Web
    needs: [unit-tests, changes]
    if: needs.changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Build
        run: yarn turbo build --filter=@repo/web

      - name: Check bundle size
        working-directory: apps/web
        run: node scripts/check-bundle-size.js

  # Mobile Build (EAS)
  build-mobile:
    name: EAS Build (${{ matrix.platform }})
    needs: [unit-tests, changes]
    if: needs.changes.outputs.mobile == 'true'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [ios, android]
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Determine profile
        id: profile
        run: |
          if [ "${{ github.event_name }}" = "push" ] && [ "${{ github.ref }}" = "refs/heads/main" ]; then
            echo "profile=production" >> $GITHUB_OUTPUT
            echo "submit=--auto-submit" >> $GITHUB_OUTPUT
          else
            echo "profile=preview" >> $GITHUB_OUTPUT
            echo "submit=" >> $GITHUB_OUTPUT
          fi

      - name: Build
        working-directory: apps/mobile
        run: |
          eas build \
            --profile ${{ steps.profile.outputs.profile }} \
            --platform ${{ matrix.platform }} \
            --non-interactive \
            --no-wait \
            ${{ steps.profile.outputs.submit }} \
            --message "${{ github.event.head_commit.message || github.event.pull_request.title }}"

  # Mobile bundle size check (doesn't need EAS)
  check-mobile-bundle:
    name: Check RN Bundle Size
    needs: [unit-tests, changes]
    if: needs.changes.outputs.mobile == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Bundle iOS
        working-directory: apps/mobile
        run: npx react-native bundle --platform ios --dev false --entry-file index.js --bundle-output /tmp/ios.bundle

      - name: Bundle Android
        working-directory: apps/mobile
        run: npx react-native bundle --platform android --dev false --entry-file index.js --bundle-output /tmp/android.bundle

      - name: Check sizes
        working-directory: apps/mobile
        run: node scripts/check-rn-bundle.js

  # ─────────────────────────────────────────────────────────────
  # STAGE 4: E2E Tests
  # ─────────────────────────────────────────────────────────────
  e2e-web:
    name: E2E Tests (Web)
    needs: [build-web, changes]
    if: >
      needs.changes.outputs.web == 'true' &&
      github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'

      - run: yarn install --frozen-lockfile

      - name: Install Playwright
        run: npx playwright install --with-deps chromium

      - name: Wait for Vercel preview
        uses: patrickedqvist/wait-for-vercel-preview@v1.3.2
        id: vercel
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          max_timeout: 600

      - name: Run Playwright
        run: npx playwright test --project=chromium
        env:
          BASE_URL: ${{ steps.vercel.outputs.url }}

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: apps/web/playwright-report/

  # Lighthouse (runs against Vercel preview)
  lighthouse:
    name: Lighthouse Audit
    needs: [build-web, changes]
    if: >
      needs.changes.outputs.web == 'true' &&
      github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Wait for Vercel preview
        uses: patrickedqvist/wait-for-vercel-preview@v1.3.2
        id: vercel
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          max_timeout: 600

      - name: Run Lighthouse
        uses: treosh/lighthouse-ci-action@v12
        with:
          urls: ${{ steps.vercel.outputs.url }}
          configPath: './lighthouserc.json'
          uploadArtifacts: true

  # ─────────────────────────────────────────────────────────────
  # STAGE 5: Summary
  # ─────────────────────────────────────────────────────────────
  pipeline-status:
    name: Pipeline Status
    needs: [
      static-analysis,
      unit-tests,
      build-web,
      build-mobile,
      check-mobile-bundle,
      e2e-web,
      lighthouse
    ]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Check pipeline status
        run: |
          echo "Static Analysis: ${{ needs.static-analysis.result }}"
          echo "Unit Tests: ${{ needs.unit-tests.result }}"
          echo "Web Build: ${{ needs.build-web.result }}"
          echo "Mobile Build: ${{ needs.build-mobile.result }}"
          echo "Mobile Bundle Check: ${{ needs.check-mobile-bundle.result }}"
          echo "E2E Web: ${{ needs.e2e-web.result }}"
          echo "Lighthouse: ${{ needs.lighthouse.result }}"
          
          # Fail if any required job failed (ignore skipped jobs)
          if [[ "${{ needs.static-analysis.result }}" == "failure" ]] || \
             [[ "${{ needs.unit-tests.result }}" == "failure" ]] || \
             [[ "${{ needs.build-web.result }}" == "failure" ]] || \
             [[ "${{ needs.build-mobile.result }}" == "failure" ]] || \
             [[ "${{ needs.check-mobile-bundle.result }}" == "failure" ]]; then
            echo "Pipeline failed!"
            exit 1
          fi
          
          echo "Pipeline passed!"
```

### Understanding the Pipeline Flow

```
┌───────────────────────────────────────────────────────────────┐
│                    COMPLETE CI/CD PIPELINE                      │
│                                                                 │
│  PR opened / Push to branch:                                    │
│                                                                 │
│  ┌──────────┐                                                   │
│  │ Detect   │──→ What changed? mobile / web / both / neither    │
│  │ Changes  │                                                   │
│  └──────────┘                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────┐                                              │
│  │ Static        │  lint + typecheck + format  (< 1 min)        │
│  │ Analysis      │                                              │
│  └───────────────┘                                              │
│       │                                                         │
│       ▼                                                         │
│  ┌───────────────┐                                              │
│  │ Unit Tests    │  Jest sharded x2  (2-4 min)                  │
│  │ (sharded)     │                                              │
│  └───────────────┘                                              │
│       │                                                         │
│       ├───────────────────┬───────────────────┐                 │
│       ▼                   ▼                   ▼                 │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │ Build    │    │ EAS Build    │    │ RN Bundle    │          │
│  │ Web      │    │ iOS+Android  │    │ Size Check   │          │
│  └──────────┘    └──────────────┘    └──────────────┘          │
│       │                                                         │
│       ├───────────────────┐                                     │
│       ▼                   ▼                                     │
│  ┌──────────┐    ┌──────────────┐                              │
│  │ E2E      │    │ Lighthouse   │                              │
│  │ (PW)     │    │ Audit        │                              │
│  └──────────┘    └──────────────┘                              │
│       │                   │                                     │
│       └───────────────────┘                                     │
│               │                                                 │
│               ▼                                                 │
│  ┌──────────────────┐                                          │
│  │ Pipeline Status   │  Summary + required check for merge      │
│  └──────────────────┘                                          │
│                                                                 │
│  Merge to main:                                                 │
│  ├── Vercel deploys web to production automatically             │
│  ├── EAS builds production mobile binaries                      │
│  └── EAS auto-submits to App Store / Play Store                │
└───────────────────────────────────────────────────────────────┘
```

**Total CI time for a typical PR:**

| Scenario | Wall-clock Time | Cost |
|----------|---------------|------|
| Web-only change | ~6 min | ~$0.05 |
| Mobile-only change | ~5 min (GH) + 15 min (EAS) | ~$0.04 + EAS credit |
| Shared package change | ~8 min (GH) + 15 min (EAS) | ~$0.07 + EAS credit |
| README change | Skipped | $0.00 |

That's a pipeline that respects both your time and your budget.

---

## 10. DEPLOYMENT GATES

Not everything should deploy automatically. Production mobile releases need more guardrails than preview builds. Here's a layered approach.

### 10.1 Manual Approval for Production

GitHub Actions supports manual approval through environments:

```yaml
# .github/workflows/release-mobile.yml
name: Release Mobile

on:
  workflow_dispatch:
    inputs:
      platform:
        description: 'Platform to release'
        required: true
        type: choice
        options:
          - all
          - ios
          - android
      skip-e2e:
        description: 'Skip E2E tests (emergency hotfix only)'
        required: false
        type: boolean
        default: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build
        working-directory: apps/mobile
        run: |
          PLATFORM=${{ github.event.inputs.platform }}
          if [ "$PLATFORM" = "all" ]; then
            PLATFORM="all"
          fi
          eas build --profile production --platform $PLATFORM --non-interactive

  e2e:
    needs: build
    if: github.event.inputs.skip-e2e != 'true'
    runs-on: ubuntu-latest
    steps:
      # ... run E2E tests against the build
      - run: echo "Running E2E tests..."

  # Requires manual approval in GitHub
  approve-release:
    needs: [build, e2e]
    runs-on: ubuntu-latest
    environment: production  # This triggers the approval gate
    steps:
      - run: echo "Release approved"

  submit:
    needs: approve-release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'yarn'
      - run: yarn install --frozen-lockfile
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Submit to stores
        working-directory: apps/mobile
        run: |
          eas submit --profile production --platform ${{ github.event.inputs.platform }} --non-interactive
```

Configure the approval gate in GitHub: Settings -> Environments -> production -> Required reviewers -> Add your team leads.

### 10.2 Staged Rollouts

Both the App Store and Google Play support staged rollouts. For Android, you can automate this:

```ruby
# In your Fastlane config
lane :staged_rollout do |options|
  percentage = options[:percentage] || 10

  upload_to_play_store(
    track: "production",
    rollout: percentage.to_s,  # "0.1" for 10%, "0.5" for 50%
    aab: "app/build/outputs/bundle/release/app-release.aab",
    json_key: ENV['GOOGLE_PLAY_JSON_KEY'],
    skip_upload_metadata: true,
    skip_upload_images: true,
    skip_upload_screenshots: true,
  )
end

# Usage:
# fastlane staged_rollout percentage:10   # 10% of users
# fastlane staged_rollout percentage:50   # 50% of users
# fastlane staged_rollout percentage:100  # Full rollout
```

With EAS Submit, you configure the track in `eas.json`:

```json
{
  "submit": {
    "production": {
      "android": {
        "track": "production",
        "rollout": 0.1
      }
    },
    "full-rollout": {
      "android": {
        "track": "production",
        "rollout": 1.0
      }
    }
  }
}
```

**A typical rollout schedule:**

| Day | Percentage | Action |
|-----|-----------|--------|
| Day 0 | 1% | Release to a canary group |
| Day 1 | 10% | Monitor crash rates, ANRs |
| Day 2 | 25% | Check analytics for anomalies |
| Day 3 | 50% | If metrics are stable, expand |
| Day 5 | 100% | Full rollout |

If crash rate exceeds threshold at any stage, halt the rollout and investigate.

### 10.3 Canary Releases with Vercel

Vercel's skew protection and rolling releases let you do canary-style deployments for web:

```
┌────────────────────────────────────────────────────────┐
│  VERCEL CANARY DEPLOYMENT                               │
│                                                         │
│  Current production: v2.4.0 (serving 100% of traffic)  │
│                                                         │
│  1. Deploy v2.4.1 → Vercel creates new deployment      │
│  2. Vercel routes 1% of traffic to v2.4.1              │
│  3. Monitor error rates, Core Web Vitals                │
│  4. Gradually increase: 5% → 25% → 50% → 100%         │
│  5. If metrics degrade, Vercel rolls back automatically │
│                                                         │
│  No custom infra. No load balancer config.              │
│  Just enable it in Vercel dashboard.                    │
└────────────────────────────────────────────────────────┘
```

Enable in Vercel dashboard: Project Settings -> Deployment Protection -> Rolling Releases.

You can also promote manually via the Vercel CLI:

```bash
# List recent deployments
vercel ls

# Promote a specific deployment to production
vercel promote <deployment-url>
```

### 10.4 Rollback Strategy

Things will go wrong. Have a plan:

**Web (Vercel):**
```bash
# Instant rollback to previous deployment
vercel rollback

# Or promote a specific known-good deployment
vercel promote <deployment-url>
```

Vercel rollbacks are instant because all previous deployments are immutable and still exist. You're not "redeploying" — you're "re-pointing."

**Mobile (OTA with EAS Update):**
```bash
# Roll back by re-publishing the previous update
eas update:republish --group <previous-update-group-id> --branch production
```

**Mobile (Native build):**
For native binary rollbacks, you're at the mercy of the store review process. This is why staged rollouts are critical for mobile — catching issues at 1% is infinitely better than catching them at 100%.

---

## 11. PUTTING IT ALL INTO PRACTICE

We've covered a lot of ground. Let me distill it into the decisions you need to make and the order you should make them.

### The Decision Checklist

```
┌──────────────────────────────────────────────────────────────┐
│  CI/CD DECISION CHECKLIST                                     │
│                                                               │
│  1. Build system                                              │
│     □ Turborepo for monorepo task orchestration               │
│     □ turbo.json configured with correct task dependencies    │
│     □ Remote caching enabled (Vercel or self-hosted)          │
│                                                               │
│  2. Mobile builds                                             │
│     □ EAS Build for most teams                                │
│     □ Fastlane + self-hosted only if required                 │
│     □ eas.json with development/preview/production profiles   │
│     □ Signing credentials managed (EAS auto or Match)         │
│                                                               │
│  3. Web builds                                                │
│     □ Vercel GitHub integration connected                     │
│     □ Ignored Build Step configured for monorepo              │
│     □ Environment variables per environment                   │
│                                                               │
│  4. Testing                                                   │
│     □ Unit tests run in CI with --filter and sharding         │
│     □ E2E web with Playwright against Vercel preview          │
│     □ E2E mobile with Maestro (local or Cloud)                │
│     □ Tests gated: cheap first, expensive after               │
│                                                               │
│  5. Performance                                               │
│     □ Bundle size budgets (web + RN)                          │
│     □ Lighthouse thresholds                                   │
│     □ Build time limits                                       │
│     □ Results posted as PR comments                           │
│                                                               │
│  6. Versioning                                                │
│     □ Changesets for semver management                        │
│     □ app.config.ts reads version from package.json           │
│     □ Build numbers auto-incremented by EAS                   │
│                                                               │
│  7. Deployment                                                │
│     □ Preview: automatic on PR                                │
│     □ Production web: automatic on merge to main              │
│     □ Production mobile: manual trigger or approval gate      │
│     □ Staged rollout plan documented                          │
│     □ Rollback plan documented and tested                     │
│                                                               │
│  8. Cost control                                              │
│     □ Path filters on all workflows                           │
│     □ Concurrency groups with cancel-in-progress              │
│     □ macOS runners only when absolutely necessary            │
│     □ Turborepo remote cache hit rate > 80%                   │
│     □ Monthly CI cost tracked and budgeted                    │
└──────────────────────────────────────────────────────────────┘
```

### The Maturity Model

Not every team needs the full pipeline on day one. Here's how to grow into it:

**Level 1: The Basics (Week 1)**
- GitHub Actions with lint + typecheck + test on PR
- Vercel auto-deploy for web
- Manual `eas build` for mobile
- No caching beyond yarn.lock

**Level 2: Efficient (Month 1)**
- Path-based filters so only affected platforms build
- Turborepo remote caching
- CocoaPods and Gradle caching
- EAS Build triggered from CI
- Concurrency groups

**Level 3: Production-Grade (Month 3)**
- Full fail-fast pipeline (static -> unit -> build -> E2E)
- Performance budgets with PR comments
- Changesets for versioning
- Staged rollouts for mobile
- Lighthouse gates for web

**Level 4: Enterprise (Month 6+)**
- Self-hosted runners for iOS (if needed)
- Maestro Cloud E2E
- Fingerprint-based smart deploys (OTA vs native)
- Deployment approval gates
- Rollback automation
- Cost dashboards and alerting

### Common Pitfalls

**Pitfall 1: "Let's just build everything on every PR."** This is the $4,000/month mistake. Start with path filters from day one. It takes 10 minutes to set up and saves thousands.

**Pitfall 2: "Our CI is flaky, let's add retries."** Retries mask the real problem. A flaky test is a broken test. A flaky build step is a broken build step. Fix the root cause. The only acceptable retry is for network operations (npm install, pod install) where transient failures are genuinely external.

**Pitfall 3: "We don't need E2E in CI, we test manually."** You test manually until you don't. The first time you ship a broken login flow because someone "forgot to check" is the last time you'll say this.

**Pitfall 4: "Let's put all our secrets in GitHub Actions."** Minimize secrets in GitHub. Use EAS credentials service for signing. Use Vercel environment variables for web config. The fewer secrets in GitHub, the smaller your attack surface.

**Pitfall 5: "Our pipeline is 40 minutes but it works."** A 40-minute pipeline is a tax on every PR. Engineers context-switch, lose flow state, and sometimes skip running CI locally because "I'll just let CI catch it." Fast CI isn't a luxury — it's a multiplier on your entire team's productivity.

---

## CHAPTER SUMMARY

CI/CD for a mobile+web monorepo is a solved problem, but it's a solved problem with many moving parts. The core principles are simple:

1. **Build only what changed** — path filters and Turborepo's `--filter`
2. **Test only what's affected** — Turborepo's `[origin/main]` syntax
3. **Deploy automatically** — Vercel for web, EAS for mobile
4. **Cancel superseded runs** — concurrency groups
5. **Fail fast** — cheap checks first, expensive checks last

The tooling stack that makes this work:
- **GitHub Actions** for orchestration
- **Turborepo** for monorepo task management and remote caching
- **EAS Build** for cloud-based mobile builds
- **Vercel** for web preview and production deployments
- **Changesets** for version management
- **Lighthouse CI** for web performance gates
- **Maestro** for mobile E2E
- **Playwright** for web E2E

The result is a pipeline that gives developers feedback in under a minute for common errors (lint, type), under 5 minutes for correctness (tests), and under 20 minutes for the full build-test-deploy cycle — while costing a fraction of the naive approach.

Your CI/CD pipeline is infrastructure. Treat it with the same care you'd give your application architecture. Version it. Review changes to it. Monitor its performance. Because every minute your pipeline wastes is a minute your team doesn't spend shipping.

---

### Up Next

**Chapter 19: OTA Updates with EAS Update** — You've built the pipeline that creates native binaries. Now let's talk about how to ship JavaScript updates without going through the store review process — the pattern that lets you fix bugs in production in minutes, not days.

### Further Reading
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [EAS Build Documentation](https://docs.expo.dev/build/introduction/)
- [Vercel CI/CD Documentation](https://vercel.com/docs/deployments/git)
- [Turborepo Remote Caching](https://turbo.build/repo/docs/core-concepts/remote-caching)
- [Changesets Documentation](https://github.com/changesets/changesets)
- [Fastlane Documentation](https://docs.fastlane.tools/)
- [Maestro Documentation](https://maestro.mobile.dev/)
