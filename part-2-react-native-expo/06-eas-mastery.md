<!--
  CHAPTER: 6
  TITLE: EAS: Build, Submit, Update — The Complete Deployment Pipeline
  PART: II — React Native & Expo
  PREREQS: Chapter 5
  KEY_TOPICS: EAS Build, EAS Submit, EAS Update, OTA, channels, branches, fingerprinting, eas.json, build profiles, credentials, EAS Workflows, App Store, Google Play, code signing
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 6: EAS: Build, Submit, Update — The Complete Deployment Pipeline

> "Shipping is your company's heartbeat. If it takes two weeks to push a hotfix, you don't have a deployment pipeline — you have a prayer chain."

---

## Introduction

Here is the hard truth nobody tells you when you start building mobile apps: **writing the code is maybe 40% of the job**. The other 60% is getting that code signed, compiled, submitted, reviewed, approved, and into the hands of real users — and then doing it again next Tuesday when your PM discovers a critical typo on the onboarding screen.

If you have ever manually uploaded an `.ipa` to App Store Connect through Transporter, waited 45 minutes for processing, realized you forgot to bump the build number, then started the whole dance over again — you know the pain. If you have spent a weekend debugging why your Android keystore password "stopped working" because someone on the team regenerated it — you know the rage.

Expo Application Services (EAS) exists to make all of that pain go away. Not to abstract it away so you never understand it — that is a different philosophy, and it fails the moment something breaks. EAS makes it go away by giving you a structured, repeatable, debuggable pipeline that handles the gnarly parts (code signing, cloud compilation, store submission, over-the-air updates) while keeping you in control.

This chapter is a deep dive. We are going to cover EAS Build, EAS Submit, and EAS Update — the three pillars — and then go further into workflows, pricing, code signing internals, and the store-specific deployment strategies that separate teams that ship confidently from teams that ship and pray.

Let's get into it.

---

## 6.1 The Three Pillars of EAS

EAS is not one tool. It is three interconnected services that together form a complete deployment pipeline for React Native apps. Think of them as stages in a relay race:

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  EAS Build   │────▶│  EAS Submit  │────▶│  EAS Update  │
│              │     │              │     │              │
│ Cloud compile│     │ Store upload │     │ OTA patches  │
│ + signing    │     │ + metadata   │     │ JS bundle    │
└─────────────┘     └──────────────┘     └──────────────┘
      ▲                                         │
      │         ┌──────────────────┐            │
      └─────────│  When native     │◀───────────┘
                │  code changes    │   (fingerprint mismatch)
                └──────────────────┘
```

### EAS Build: The Cloud Compiler

EAS Build takes your project source code, runs `npx expo prebuild` to generate native projects, installs dependencies, compiles the native binaries, signs them with the appropriate credentials, and produces artifacts (`.ipa` for iOS, `.aab`/`.apk` for Android). All of this happens on Expo's cloud infrastructure — you do not need a Mac to build iOS apps.

**When you use it:** Every time you change native code, add a native module, update native configuration, or need a fresh binary for the stores.

### EAS Submit: The Store Automator

EAS Submit takes the artifacts from EAS Build (or your local machine) and uploads them to the App Store (via App Store Connect) or Google Play (via the Google Play Developer API). It handles the authentication, upload protocols, and can even manage your store metadata.

**When you use it:** Every time you have a binary that needs to reach testers (TestFlight, internal testing tracks) or go through store review.

### EAS Update: The Over-the-Air Patcher

EAS Update ships JavaScript bundle and asset changes directly to users' devices without going through the app stores. When a user opens your app, it checks for updates, downloads the new bundle, and applies it — either immediately or on next launch.

**When you use it:** Bug fixes, copy changes, UI tweaks, feature flag adjustments — anything that does not touch native code.

### How They Fit Together

Here is the mental model that matters:

1. You **build** a binary and submit it to the stores. This binary contains a specific "runtime version" — think of it as a compatibility contract.
2. Users download that binary from the App Store / Google Play.
3. You ship **updates** (JS bundles) over the air. These updates target a specific runtime version and channel.
4. When you change native code, the runtime fingerprint changes. Now your OTA updates would be incompatible with the old binary. Time to **build** again.

This cycle — build, submit, update, update, update, build again — is the heartbeat of a well-run mobile team. Teams that understand this cycle ship daily. Teams that don't understand it ship monthly and break things when they try to go faster.

---

## 6.2 EAS Build Deep Dive

### 6.2.1 How Cloud Builds Work: The Six-Step Sequence

When you run `eas build`, here is what actually happens behind the scenes:

```
Step 1: UPLOAD
  Your project source is compressed and uploaded to EAS servers.
  .easignore controls what gets excluded (similar to .gitignore).
  
Step 2: PREBUILD
  `npx expo prebuild` generates the native iOS/Android projects.
  This is where your app.json/app.config.js config gets translated
  into native code (Info.plist, AndroidManifest.xml, etc.).

Step 3: INSTALL DEPENDENCIES
  npm/yarn/pnpm install for JS deps.
  pod install for iOS (CocoaPods).
  Gradle dependency resolution for Android.

Step 4: COMPILE
  Xcode build for iOS (via xcodebuild).
  Gradle build for Android.
  This is where most build time goes.

Step 5: SIGN
  iOS: Apply provisioning profile + distribution certificate.
  Android: Sign with keystore (upload key or app signing key).

Step 6: ARTIFACT
  Produce the final .ipa (iOS) or .aab/.apk (Android).
  Upload to EAS servers for download or submission.
```

Let's look at the timing breakdown for a typical medium-complexity app:

| Step | iOS (Large Worker) | Android (Large Worker) | iOS (Medium Worker) | Android (Medium Worker) |
|------|-------------------|----------------------|--------------------|-----------------------|
| Upload | 10-30s | 10-30s | 10-30s | 10-30s |
| Prebuild | 15-30s | 10-20s | 15-30s | 10-20s |
| Install deps | 60-120s | 30-60s | 90-180s | 45-90s |
| Compile | 300-600s | 180-420s | 600-1200s | 360-720s |
| Sign | 5-10s | 5-10s | 5-10s | 5-10s |
| Artifact | 10-30s | 10-20s | 10-30s | 10-20s |
| **Total** | **7-14 min** | **4-9 min** | **12-25 min** | **8-15 min** |

The compile step dominates. This is why resource class selection matters, and why caching is critical.

### 6.2.2 Build Profiles: dev, preview, production

Build profiles are defined in `eas.json` and represent different build configurations for different purposes. Here is a production-ready `eas.json` that I recommend as a starting point:

```json
{
  "cli": {
    "version": ">= 13.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "base": {
      "node": "20.18.0",
      "env": {
        "EXPO_NO_TELEMETRY": "1"
      },
      "cache": {
        "key": "v1",
        "paths": [
          "~/.ccache"
        ]
      }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "resourceClass": "medium",
      "ios": {
        "simulator": false,
        "buildConfiguration": "Debug"
      },
      "android": {
        "buildType": "apk",
        "gradleCommand": ":app:assembleDebug"
      },
      "channel": "development",
      "env": {
        "APP_ENV": "development",
        "API_URL": "https://api-dev.yourapp.com"
      }
    },
    "development:simulator": {
      "extends": "development",
      "ios": {
        "simulator": true
      },
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "extends": "base",
      "distribution": "internal",
      "resourceClass": "medium",
      "ios": {
        "buildConfiguration": "Release"
      },
      "android": {
        "buildType": "apk"
      },
      "channel": "preview",
      "env": {
        "APP_ENV": "staging",
        "API_URL": "https://api-staging.yourapp.com"
      }
    },
    "preview:device": {
      "extends": "preview",
      "ios": {
        "enterpriseProvisioning": "adhoc"
      }
    },
    "production": {
      "extends": "base",
      "resourceClass": "large",
      "autoIncrement": true,
      "ios": {
        "buildConfiguration": "Release",
        "image": "macos-ventura-14.2-xcode-15.1"
      },
      "android": {
        "buildType": "app-bundle"
      },
      "channel": "production",
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.yourapp.com"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
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

Let's break down what each profile is for:

**`development`** — This is your daily driver. It builds a development client (a custom version of Expo Go that includes your native modules). The `distribution: "internal"` means it is distributed via EAS's internal distribution (QR code / link), not through the stores. You use `apk` for Android because it can be sideloaded directly — no Play Store needed.

**`development:simulator`** — Same as development, but targets iOS Simulator. You cannot install a regular `.ipa` on a simulator; it needs a simulator-specific build. This profile is for engineers who want to test on the simulator with the dev client.

**`preview`** — This is for QA and stakeholders. It is a Release build (optimized, no dev menu) distributed internally. Think of it as "what the app will feel like in production, but not going through the stores." PMs and designers install this to test features before they ship.

**`production`** — The real deal. Uses `large` resource class for faster builds (time is money when you are blocking a release). Produces an `.aab` (Android App Bundle) because that is what Google Play requires. Auto-increments version numbers. This is what goes to the stores.

### 6.2.3 The `extends` Pattern

Notice the `"extends": "base"` pattern. This is how you avoid duplicating configuration across profiles. The `base` profile is never built directly — it is a template that other profiles inherit from.

```
base (node version, cache config, env)
  ├── development (dev client, internal dist, debug)
  │     └── development:simulator (simulator-specific)
  ├── preview (release build, internal dist)
  │     └── preview:device (adhoc provisioning)
  └── production (store dist, large resource class)
```

This hierarchy is important. When you need to change the Node version, you change it once in `base` and every profile picks it up. When you need to add a global environment variable, same deal.

### 6.2.4 Resource Classes: When to Pay for Large

EAS offers two resource classes for builds:

| | Medium | Large |
|---|---|---|
| **iOS** | M1 (3 CPU, 12 GB RAM) | M1 (4+ CPU, 14 GB RAM) |
| **Android** | 4 CPU, 16 GB RAM | 8 CPU, 32 GB RAM |
| **Cost** | 1x build credit per minute | 2x build credits per minute |
| **Typical iOS build** | 15-25 minutes | 7-14 minutes |
| **Typical Android build** | 8-15 minutes | 4-9 minutes |

**My rule of thumb:**

- Use `medium` for development and preview builds. These are not blocking releases, and the time difference is not worth 2x the cost.
- Use `large` for production builds. When you are cutting a release, every minute matters. Your whole team might be waiting. A 7-minute build vs a 20-minute build is the difference between shipping before lunch and shipping after a long meeting where everyone lost context.
- If you are on a tight budget, use `medium` for everything and go get coffee during builds.

### 6.2.5 Build Hooks and Custom Build Steps

Sometimes the standard build pipeline is not enough. Maybe you need to run a script before the build starts, or modify the native project after prebuild but before compilation. EAS supports custom build steps via `eas-build-pre-install.sh`, `eas-build-post-install.sh`, and `eas-build-on-success.sh` lifecycle hooks.

But the more powerful approach is **EAS Build custom functions** in `eas.json`:

```json
{
  "build": {
    "production": {
      "ios": {
        "image": "macos-ventura-14.2-xcode-15.1"
      },
      "config": "eas-build-config.yml"
    }
  }
}
```

And then in `eas-build-config.yml`:

```yaml
build:
  name: Custom production build
  steps:
    - eas/checkout
    - run:
        name: Validate environment
        command: |
          echo "Building for $APP_ENV"
          if [ -z "$SENTRY_AUTH_TOKEN" ]; then
            echo "ERROR: SENTRY_AUTH_TOKEN not set"
            exit 1
          fi
    - eas/install_node_modules
    - run:
        name: Run type check
        command: npx tsc --noEmit
    - run:
        name: Run unit tests
        command: npx jest --ci --passWithNoTests
    - eas/prebuild
    - run:
        name: Inject build metadata
        command: |
          echo "BUILD_COMMIT=$(git rev-parse HEAD)" >> "$EXPO_PUBLIC_ENV_FILE"
          echo "BUILD_TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$EXPO_PUBLIC_ENV_FILE"
    - eas/run_gradle  # Android
    # OR
    - eas/run_fastlane  # iOS
```

This is incredibly powerful. You can:

- Run tests before compilation (fail fast, save build minutes)
- Validate that required secrets are present
- Inject build metadata into the app
- Run custom native code modifications
- Upload source maps to Sentry/Bugsnag
- Notify Slack on success/failure

**Real-world example from a fintech team I worked with:** They had a compliance requirement that every production build must pass static analysis. They added a custom step that runs `eslint` with their security-focused rules. If any security rule fails, the build aborts. This caught a hardcoded API key that a junior engineer accidentally committed. The build failed, the key never made it into a production binary, and nobody had to have an awkward incident review meeting.

### 6.2.6 Credential Management

This is where most teams lose their minds. iOS code signing is genuinely complex, and Android keystores are deceptively simple until you lose one.

**iOS Credentials: The Cast of Characters**

```
Apple Developer Account
  └── Distribution Certificate (p12)
        ├── private key (NEVER lose this)
        └── certificate (signed by Apple)
  
  └── Provisioning Profile
        ├── App ID (com.yourcompany.yourapp)
        ├── Certificate reference
        ├── Device UDIDs (for ad hoc)
        └── Entitlements (push notifications, etc.)
```

When you run `eas build` for the first time, EAS will ask you how you want to handle credentials:

1. **Let EAS manage them** (recommended for most teams)
2. **Provide your own** (for teams with existing CI/CD or strict security requirements)

If you let EAS manage them, here is what happens:

- EAS generates a distribution certificate and stores the private key encrypted on their servers.
- EAS creates the appropriate provisioning profile (ad hoc for internal distribution, App Store for production).
- EAS handles certificate rotation, profile updates, and device registration.
- You can view and download your credentials with `eas credentials`.

This is a massive quality-of-life improvement. Before EAS, the iOS credential management workflow looked like this:

1. Open Keychain Access
2. Generate a Certificate Signing Request
3. Upload it to the Apple Developer Portal
4. Download the certificate
5. Install it in your keychain
6. Create a provisioning profile in the portal
7. Download and install the profile
8. Pray that Xcode recognizes everything
9. Export the certificate as .p12
10. Copy it to your CI server
11. Import it on the CI server's keychain
12. Set the correct code signing settings in Xcode
13. Realize step 8 failed and start over

EAS reduces this to: `eas build --profile production --platform ios`. First time, it asks a couple of questions. After that, it just works.

**Android Credentials: Simpler but Scarier**

```
Android Keystore (.jks or .keystore)
  └── Key Alias
        ├── private key
        └── certificate

Google Play App Signing
  └── Upload Key (you provide to Google)
  └── App Signing Key (Google manages)
```

Android has fewer moving parts, but the stakes are higher in one specific way: **if you lose your Android keystore and have not enrolled in Google Play App Signing, your app is dead.** You cannot update it. You have to publish a new app with a new package name.

EAS manages your Android keystore the same way it manages iOS credentials. It generates a keystore, stores it encrypted, and uses it for every build. You can download it with `eas credentials` if you need it for other CI systems.

**My strong recommendation:** Always enroll in Google Play App Signing. This means Google holds the actual signing key, and you only hold an "upload key." If you lose your upload key, you can reset it through Google Play Console. Your app lives on. EAS sets this up correctly by default.

**Credential Commands You Should Know:**

```bash
# View all credentials for your project
eas credentials

# Download iOS distribution certificate
eas credentials --platform ios

# Download Android keystore
eas credentials --platform android

# Set up credentials interactively
eas credentials --platform ios --profile production

# Clear credentials (dangerous - know what you are doing)
eas credentials --platform ios --clear
```

### 6.2.7 Build Cache Optimization

EAS caches certain things between builds automatically (CocoaPods, Gradle caches). But you can significantly speed up builds with smart custom caching.

In your `eas.json`:

```json
{
  "build": {
    "base": {
      "cache": {
        "key": "v1-{{ hashFiles('package-lock.json') }}",
        "paths": [
          "node_modules",
          "~/.cocoapods",
          "ios/Pods"
        ]
      }
    }
  }
}
```

**Cache key strategy:** Use a version prefix (`v1-`) so you can bust the cache by changing it to `v2-`. Use `hashFiles` to automatically invalidate when dependencies change.

**What to cache:**

| Path | Why | Impact |
|------|-----|--------|
| `node_modules` | Skip npm install if lockfile unchanged | 30-60s saved |
| `~/.cocoapods` | CocoaPods specs repo | 20-40s saved |
| `ios/Pods` | Compiled pods | 30-90s saved |
| `~/.gradle/caches` | Gradle dependency cache | 20-60s saved |

**What NOT to cache:**

- The entire `ios/` or `android/` directories. Prebuild should regenerate these fresh to avoid stale native code.
- `~/.ccache` unless you have configured ccache properly for your native code.
- Anything that changes frequently. Cache hits matter more than cache size.

**When to bust the cache:**

- After upgrading Expo SDK (change `v1-` to `v2-`)
- After adding a new native module
- When builds start failing for mysterious reasons (the cache is always the first suspect)

### 6.2.8 Common Build Failures and How to Debug Them

Let me save you hours of pain with this troubleshooting guide. These are the failures I see most often, ranked by frequency:

**1. "No signing certificate found" (iOS)**

```
error: No signing certificate "iOS Distribution" found
```

**What happened:** Your distribution certificate expired, was revoked, or EAS doesn't have it.

**Fix:**
```bash
eas credentials --platform ios
# Select "Remove current distribution certificate"
# Then rebuild - EAS will create a new one
```

**2. "Provisioning profile doesn't match" (iOS)**

```
error: Provisioning profile "XXX" doesn't include signing certificate "YYY"
```

**What happened:** The provisioning profile references a certificate that does not match the one being used for signing. This usually happens after certificate rotation.

**Fix:**
```bash
eas credentials --platform ios
# Remove the provisioning profile
# Rebuild - EAS will create a new profile with the current certificate
```

**3. CocoaPods version mismatch (iOS)**

```
[!] The version of CocoaPods used to generate the lockfile (1.14.3) is
higher than the version of the current executable (1.13.0)
```

**Fix:** Pin the CocoaPods version in your Gemfile or specify the image version in `eas.json`:

```json
{
  "build": {
    "production": {
      "ios": {
        "image": "macos-ventura-14.2-xcode-15.1",
        "cocoapods": "1.15.2"
      }
    }
  }
}
```

**4. Gradle out of memory (Android)**

```
java.lang.OutOfMemoryError: Java heap space
```

**Fix:** Increase Gradle memory in `android/gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g -XX:+HeapDumpOnOutOfMemoryError
```

Or better, switch to `large` resource class for that build profile.

**5. "INSTALL_FAILED_UPDATE_INCOMPATIBLE" (Android)**

```
INSTALL_FAILED_UPDATE_INCOMPATIBLE: Existing package com.yourapp
 signatures do not match newer version
```

**What happened:** You're trying to install an APK signed with a different keystore over an existing installation.

**Fix:** Uninstall the existing app on the device first. This is not an EAS bug — it is an Android security feature.

**6. Node version mismatch**

```
error: The engine "node" is incompatible with this module.
Expected version ">=18.0.0". Got "16.20.0"
```

**Fix:** Pin your Node version in `eas.json`:

```json
{
  "build": {
    "base": {
      "node": "20.18.0"
    }
  }
}
```

**7. Native module not linked**

```
error: Cannot find native module 'react-native-some-library'
```

**What happened:** The native module was not properly linked during prebuild. This often happens with libraries that require manual configuration.

**Fix:** Check if the library has an Expo config plugin. If it does, add it to `app.json`:

```json
{
  "expo": {
    "plugins": [
      "react-native-some-library"
    ]
  }
}
```

If it does not have a config plugin, you may need to write one (see Chapter 5) or eject to a custom dev client setup.

**General debugging strategy:**

```bash
# View full build logs
eas build:list
# Click the build URL to see logs in the browser

# Or view logs in terminal
eas build:view --platform ios

# Re-run with verbose logging
eas build --profile production --platform ios --clear-cache

# Check what prebuild generates locally
npx expo prebuild --clean
# Inspect ios/ and android/ directories
```

**Pro tip:** Before debugging a cloud build failure, try building locally first. Run `npx expo prebuild --clean` and then build with Xcode or Android Studio. If it fails locally too, you have isolated the problem to your code/config rather than the cloud environment.

---

## 6.3 EAS Submit Deep Dive

### 6.3.1 Automating App Store Connect Submission

Getting your iOS app from "compiled binary" to "live on the App Store" involves multiple stages:

```
Binary Upload → Processing → TestFlight Review → TestFlight → 
  App Review → Approved → Release (manual or phased)
```

EAS Submit handles the first step — uploading the binary to App Store Connect. Here is how to set it up:

**Step 1: Configure the Apple API Key**

Apple's App Store Connect API uses a key-based authentication system. You need an API key with the "App Manager" role.

1. Go to [App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api)
2. Generate a new key with "App Manager" role
3. Download the `.p8` file (you can only download it once!)
4. Note the Key ID and Issuer ID

```bash
# Store the API key with EAS
eas credentials --platform ios
# Choose "App Store Connect API Key"
# Enter your Key ID, Issuer ID, and path to .p8 file
```

Or configure it in `eas.json`:

```json
{
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345",
        "ascApiKeyPath": "./keys/AuthKey_XXXXXXXXXX.p8",
        "ascApiKeyIssuerId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "ascApiKeyId": "XXXXXXXXXX"
      }
    }
  }
}
```

**Important:** Do not commit the `.p8` file to git. Add it to `.gitignore` and store it in your secrets manager. Better yet, use EAS Secrets:

```bash
eas secret:create --scope project --name ASC_API_KEY_P8 \
  --type file --value ./keys/AuthKey_XXXXXXXXXX.p8
```

**Step 2: Submit**

```bash
# Submit the latest production build
eas submit --platform ios --profile production

# Submit a specific build
eas submit --platform ios --id BUILD_ID

# Submit a local binary
eas submit --platform ios --path ./path/to/your-app.ipa

# Build and submit in one command
eas build --profile production --platform ios --auto-submit
```

The `--auto-submit` flag is gold. It chains the build and submit steps so that as soon as the build completes, submission begins automatically. No manual intervention required.

**Step 3: TestFlight**

After EAS uploads the binary, Apple processes it (usually 5-30 minutes). Once processing completes, the build appears in TestFlight. If you have external testers, the build goes through TestFlight review (usually 24-48 hours, often faster).

**The flow for internal testers:** Upload → Processing (5-30 min) → Available in TestFlight. No review required for internal testers.

**The flow for external testers:** Upload → Processing → TestFlight Review (24-48h) → Available to external testers.

### 6.3.2 Automating Google Play Submission

Google Play uses a track-based system:

```
Internal Testing → Closed Testing → Open Testing → Production
      ↑                                              ↓
    (fastest)                                  (staged rollout)
```

**Step 1: Create a Google Service Account**

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a service account with no special roles
3. Download the JSON key file
4. Go to Google Play Console → Setup → API Access
5. Link your Google Cloud project
6. Grant the service account "Release Manager" permission

```json
{
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./keys/google-service-account.json",
        "track": "internal",
        "releaseStatus": "completed"
      }
    }
  }
}
```

**Step 2: Submit**

```bash
# Submit to internal testing track
eas submit --platform android --profile production

# Build and submit in one command
eas build --profile production --platform android --auto-submit
```

**Track options:**

| Track | Purpose | Review Required | Time to Available |
|-------|---------|----------------|-------------------|
| `internal` | Team testing (up to 100 testers) | No | Minutes |
| `alpha` | Closed testing | No | Minutes |
| `beta` | Open testing | Sometimes | Minutes to hours |
| `production` | Public release | Yes (first time, then usually auto-approved) | Hours to days |

**Release status options:**

- `completed` — Immediately available to users on the track
- `draft` — Uploaded but not released; must be manually released from Play Console
- `halted` — Paused release (useful for staged rollouts)

**My recommendation for most teams:** Always submit to `internal` first. Test there. Then promote from `internal` to `production` through the Play Console UI. This gives you a manual checkpoint before going live.

### 6.3.3 Metadata Management with EAS Metadata

EAS Metadata lets you manage your app store listings (descriptions, screenshots, keywords, privacy information) as code. This is a game-changer for teams that:

- Want store metadata in version control
- Need to coordinate metadata changes with releases
- Manage multiple localizations
- Want PR reviews on store listing changes

```bash
# Pull current metadata from the stores
eas metadata:pull

# Push metadata to the stores
eas metadata:push
```

This creates a `store.config.json` file:

```json
{
  "configVersion": 0,
  "apple": {
    "info": {
      "en-US": {
        "title": "Your App Name",
        "subtitle": "The Best App Ever Made",
        "description": "A longer description of your app...",
        "keywords": "keyword1,keyword2,keyword3",
        "marketingUrl": "https://yourapp.com",
        "supportUrl": "https://yourapp.com/support",
        "privacyPolicyUrl": "https://yourapp.com/privacy",
        "releaseNotes": "Bug fixes and performance improvements."
      }
    },
    "categories": ["PRODUCTIVITY", "BUSINESS"],
    "copyright": "2026 Your Company",
    "advisory": {
      "alcoholTobaccoOrDrugUseOrReferences": "NONE",
      "gamblingSimulated": "NONE",
      "matureOrSuggestiveThemes": "NONE",
      "profanityOrCrudeHumor": "NONE",
      "sexualContentOrNudity": "NONE",
      "violenceCartoonOrFantasy": "NONE",
      "violenceRealistic": "NONE",
      "violenceRealisticProlonged": "NONE",
      "horrorFearThemes": "NONE",
      "gamblingAndContests": "NONE",
      "medicalOrTreatmentInformation": "NONE",
      "unrestricted_web_access": false
    }
  },
  "google": {
    "info": {
      "en-US": {
        "title": "Your App Name",
        "shortDescription": "Short description for Google Play",
        "fullDescription": "Full description...",
        "video": "https://youtube.com/watch?v=..."
      }
    }
  }
}
```

**Why this matters:** Store metadata changes used to be a manual, error-prone process. Someone logs into App Store Connect, types in the new description, forgets to update the French translation, submits for review, and gets rejected because the German privacy policy URL is broken. With EAS Metadata, it is a code change that goes through PR review like everything else.

### 6.3.4 Common Rejection Reasons and How to Avoid Them

I have been through enough app reviews to write a book on just this topic. Here are the rejections I see most often:

**Apple App Store Rejections:**

| Rejection | Guideline | How to Avoid |
|-----------|-----------|--------------|
| Incomplete metadata | 2.1 | Fill in ALL fields. No placeholder text. Privacy policy URL must work. |
| Crashes on launch | 2.1 | Test on physical devices before submitting. Not just simulators. |
| Broken login | 2.1 | Provide demo credentials in the App Review Information section. Always. |
| In-app purchase issues | 3.1.1 | If you sell digital goods, you MUST use Apple's IAP. No exceptions. |
| Privacy nutrition label mismatch | 5.1.2 | Your privacy label must accurately reflect all data collection. If you use analytics, declare it. |
| Missing purpose string | 5.1.1 | Every permission must have a human-readable description in Info.plist. |
| Hidden features | 2.3.1 | Do not hide functionality behind secret gestures or codes. Reviewers will find out. |
| Guideline 4.3 (Spam) | 4.3 | Do not submit apps that are too similar to existing apps. Applies to white-label apps. |

**Google Play Rejections:**

| Rejection | Policy | How to Avoid |
|-----------|--------|--------------|
| Data safety section inaccurate | Data Safety | Fill in the data safety form honestly. Cross-reference with your SDKs. |
| Missing privacy policy | Privacy | Must have a working privacy policy URL in the Play listing AND in the app. |
| Deceptive behavior | Deceptive Behavior | Do not describe features in the listing that do not exist in the app. |
| Target API level too low | Target API Level | Must target the current year's required API level (currently API 34+). |
| Background location | Location | If you use background location, you must fill out a declaration form and it takes weeks. |
| Foreground service misuse | Foreground Service | Only use foreground services for their declared type. Random background tasks are not allowed. |

**The meta-advice:** Test your app on a physical device, with a fresh install, as if you have never seen it before. Go through every flow. Every button. Every edge case. That is what reviewers do.

### 6.3.5 Versioning Strategy

Mobile versioning is more complex than web versioning because you have multiple numbers to manage:

```
Version String (user-facing):   1.2.3
  ├── Major: Breaking changes, new features
  ├── Minor: New features, improvements  
  └── Patch: Bug fixes

Build Number (store-facing):
  iOS buildNumber:    42
  Android versionCode: 42
    └── Must be monotonically increasing
    └── Cannot reuse a number, ever
```

**The problem:** Every binary submitted to the stores must have a unique build number. If you submit build 42 to TestFlight, you cannot submit another build 42 — even if the version string changed from 1.2.3 to 1.3.0.

**EAS's solution: Remote version source**

In your `eas.json`:

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

With `appVersionSource: "remote"`, EAS tracks the version numbers on their servers. With `autoIncrement: true`, every production build automatically gets the next build number. You never have to think about it.

**My recommended versioning workflow:**

1. Set `version` in `app.json` manually when you want to change the user-facing version (e.g., 1.2.0 to 1.3.0).
2. Let EAS auto-increment the build number.
3. Use the `expo-constants` package to read the version in your app: `Constants.expoConfig.version`.
4. Tag your git repository with the version: `git tag v1.3.0-42` (version-buildNumber).

```bash
# Manually set the remote version
eas build:version:set --platform ios --build-number 100
eas build:version:set --platform android --version-code 100

# View current remote version
eas build:version:get --platform ios
eas build:version:get --platform android
```

---

## 6.4 EAS Update (OTA) Deep Dive

This is where EAS becomes truly transformative. Over-the-air updates let you ship bug fixes and new features without going through the app stores. No review wait. No user action required. You push, they get it.

### 6.4.1 How OTA Updates Work

When you publish an update with EAS Update, here is what happens:

```
Developer side:
  1. You run `eas update`
  2. Expo bundles your JS code and assets
  3. The bundle is uploaded to EAS servers
  4. It's associated with a branch and runtime version

User side:
  1. User opens the app
  2. The expo-updates library checks for new updates
  3. If an update is available (matching channel + runtime version):
     a. Download the new JS bundle + assets
     b. Store them locally
  4. On next app launch (or immediately, depending on config):
     a. Load the new bundle instead of the embedded one
```

**What gets shipped OTA:**
- JavaScript code (your React components, business logic, navigation)
- Static assets (images, fonts, JSON files imported through `require()`)
- Expo config changes that only affect JS-side behavior

**What does NOT get shipped OTA:**
- Native code (Objective-C, Swift, Java, Kotlin)
- Native module configurations
- Permission changes (new permissions in Info.plist or AndroidManifest.xml)
- Native library additions or upgrades
- App icons, splash screens (these are baked into the binary)

The mental model: **if the change would require `npx expo prebuild` to regenerate native files, it cannot be shipped OTA.**

### 6.4.2 Publishing Updates

```bash
# Publish an update to the production channel
eas update --branch production --message "Fix checkout button alignment"

# Publish to a specific channel
eas update --channel preview --message "New onboarding flow"

# Publish for a specific platform
eas update --branch production --platform ios --message "iOS-only hotfix"

# Publish with an auto-generated message from git
eas update --branch production --auto
```

**What happens under the hood:**

```bash
# EAS Update essentially does this:
npx expo export --platform ios --platform android
# Produces: dist/bundles/ios-xxxxx.js, dist/bundles/android-xxxxx.js
# Plus: dist/assets/... (images, fonts, etc.)
# Then uploads everything to EAS servers
```

### 6.4.3 Channels and Branches: The Deployment Topology

This is the most confusing part of EAS Update for newcomers, so let me be very precise.

**Branch:** A sequence of updates. Think of it like a git branch — it is a timeline of versions. When you publish an update, it goes to a branch.

**Channel:** A pointer that maps to a branch. Your app binaries point to a channel, not a branch. Channels are configured in `eas.json` via the `channel` property in each build profile.

```
Channel: "production"  ──points to──▶  Branch: "production"
Channel: "preview"     ──points to──▶  Branch: "preview"
Channel: "staging"     ──points to──▶  Branch: "staging"
```

**Why the indirection?** Because channels let you do things like:

```
Channel: "production"  ──points to──▶  Branch: "production-v2"
                        (was pointing to "production-v1")
```

You can atomically switch which branch a channel points to. This is how you do rollbacks, A/B testing, and gradual rollouts.

```bash
# Point the production channel to a different branch
eas channel:edit production --branch production-rollback

# Create a new channel
eas channel:create staging

# List all channels
eas channel:list

# View which branch a channel points to
eas channel:view production
```

**The practical topology for most teams:**

```
Build Profile     Channel        Branch           Purpose
─────────────     ──────────     ──────────       ───────
development       development    development      Dev testing
preview           preview        preview          QA / stakeholders  
production        production     production       Live users
```

**Advanced topology for teams that need staging:**

```
Build Profile     Channel        Branch           Purpose
─────────────     ──────────     ──────────       ───────
development       development    development      Dev testing
preview           preview        preview          QA / stakeholders
staging           staging        staging          Pre-production
production        production     production       Live users
```

### 6.4.4 Runtime Versioning and @expo/fingerprint

Here is the million-dollar question: **how does the system know when an OTA update is compatible with a given binary?**

The answer is **runtime versioning**. Every binary has a runtime version, and every update targets a runtime version. If they match, the update is compatible. If they do not match, the update is ignored.

There are three ways to set the runtime version:

**Option 1: Manual (fragile, not recommended)**

```json
{
  "expo": {
    "runtimeVersion": "1.0.0"
  }
}
```

You manually bump this whenever native code changes. The problem: you will forget. And when you forget, you will ship a JS update that references a native module that does not exist in the binary. The app will crash.

**Option 2: Using `exposdk` policy**

```json
{
  "expo": {
    "runtimeVersion": {
      "policy": "sdkVersion"
    }
  }
}
```

The runtime version is derived from the Expo SDK version. Updates are compatible as long as the SDK version matches. This is simple but coarse-grained — you might have native changes between builds that both use the same SDK version.

**Option 3: Fingerprinting (recommended)**

```json
{
  "expo": {
    "runtimeVersion": {
      "policy": "fingerprint"
    }
  }
}
```

This is the magic option. The `@expo/fingerprint` package computes a hash of all native dependencies, configuration, and code. If anything native changes — a new library, an updated config plugin, a permissions change — the fingerprint changes, and OTA updates to old binaries stop.

**How fingerprinting works:**

```bash
# View the current fingerprint
npx @expo/fingerprint ./

# Output:
# {
#   "hash": "abc123def456...",
#   "sources": [
#     { "type": "dir", "path": "ios", "hash": "..." },
#     { "type": "dir", "path": "android", "hash": "..." },
#     { "type": "file", "path": "app.json", "hash": "..." },
#     { "type": "file", "path": "package.json", "hash": "..." }
#   ]
# }
```

The fingerprint includes:
- All native source files (ios/, android/)
- Native dependency versions (from Podfile.lock, build.gradle)
- Expo config plugins and their configurations
- SDK version
- Native module versions

The fingerprint does NOT include:
- JavaScript source code
- Static assets (images, fonts)
- Environment variables (unless they affect native configuration)

**Decision tree for OTA vs binary update:**

```
Did you change any of these?
  ├── Native code (ios/, android/)          → Binary update required
  ├── Added/removed a native module         → Binary update required  
  ├── Changed app.json native config        → Binary update required
  │   (permissions, bundle ID, scheme, etc.)
  ├── Updated Expo SDK                      → Binary update required
  ├── Changed config plugins                → Binary update required
  │
  └── Only JS/TS code and assets?           → OTA update is fine!
      ├── Bug fixes                         → OTA
      ├── New screens/components            → OTA
      ├── Updated API endpoints             → OTA
      ├── Changed styling                   → OTA
      └── New images/fonts (JS-imported)    → OTA
```

### 6.4.5 Rollback Strategies

Things go wrong. An update might introduce a regression. You need the ability to roll back quickly.

**Strategy 1: Publish a revert update**

```bash
# Revert to the previous git commit and publish
git revert HEAD
eas update --branch production --message "Revert: undo broken checkout"
```

This is the simplest approach. You create a new update that is effectively the old code.

**Strategy 2: Republish a known-good commit**

```bash
# Check out the last known good state
git checkout v1.2.3

# Publish from that state
eas update --branch production --message "Rollback to v1.2.3"

# Go back to main
git checkout main
```

**Strategy 3: Channel reassignment (instant)**

```bash
# You had: production channel → production branch
# Redirect to a rollback branch
eas channel:edit production --branch production-v1.2.3

# Later, when the fix is ready:
eas channel:edit production --branch production
```

This is the fastest rollback because it does not require publishing anything. It just changes the pointer.

**Strategy 4: Republish a previous update group**

```bash
# List recent update groups
eas update:list --branch production

# Republish a specific update group
eas update:republish --group UPDATE_GROUP_ID --branch production
```

**My recommendation:** For critical rollbacks, use channel reassignment (Strategy 3) because it is instant. For planned reverts, use Strategy 1 (publish a revert). Always have a "known-good" branch that you can point channels to in an emergency.

### 6.4.6 When OTA is NOT Appropriate

I want to be very explicit about this because getting it wrong crashes apps:

**Never ship OTA when:**

1. **You added a new native module.** The JS will try to import it, the native side will not have it, and you get a red screen.

2. **You changed permissions.** Adding camera permission requires a new binary. The permission string in Info.plist is native configuration.

3. **You updated the Expo SDK.** SDK upgrades almost always change native code.

4. **You changed the app icon or splash screen.** These are native assets baked into the binary.

5. **You modified a config plugin.** Config plugins generate native code during prebuild.

6. **You need to pass app store review.** Some apps have compliance requirements that every version goes through review. OTA bypasses review.

7. **Apple's guidelines say so.** Apple's App Store Review Guidelines (3.3.2) allow OTA updates for JavaScript and assets, but not for changing the primary purpose of the app, adding a storefront or code distribution mechanism, or bypassing their review process in spirit.

**The safe approach:** Use fingerprinting (Option 3 above). If the fingerprint changes, you need a new binary. If it does not change, OTA is safe. This removes the guesswork entirely.

### 6.4.7 Update Groups and Deployment Patterns

When you publish an update, EAS creates an "update group" — a set of platform-specific updates that were published together.

```bash
# Publish for both platforms (creates one update group)
eas update --branch production --message "Fix alignment bug"

# This creates:
# Update Group: abc123
#   ├── iOS update (JS bundle for iOS)
#   └── Android update (JS bundle for Android)
```

**Deployment patterns:**

**Pattern 1: Ship to preview first, then production**

```bash
# Deploy to preview for QA
eas update --branch preview --message "New feature: dark mode"

# QA tests on preview builds...
# ✅ All good

# Deploy to production
eas update --branch production --message "New feature: dark mode"
```

**Pattern 2: Gradual rollout with multiple branches**

```bash
# Create a canary branch
eas update --branch production-canary --message "Experimental: new checkout"

# Point 5% of production traffic to canary
# (This requires custom logic in your app to choose branches)

# If metrics look good, publish to main production branch
eas update --branch production --message "New checkout flow"
```

**Pattern 3: Feature-flag gated updates**

```bash
# Ship the code to everyone, gated behind a feature flag
eas update --branch production --message "Add dark mode (flagged off)"

# Enable the flag server-side for 10% of users
# Monitor metrics
# Gradually increase to 100%
```

Pattern 3 is my recommendation for most teams. Feature flags give you finer control than branch-based rollouts, and they work with your existing feature flag infrastructure (LaunchDarkly, Statsig, Unleash, etc.).

### 6.4.8 Configuring Update Behavior in Your App

The `expo-updates` library controls how your app checks for and applies updates. Configure it in `app.json`:

```json
{
  "expo": {
    "updates": {
      "enabled": true,
      "checkAutomatically": "ON_LOAD",
      "fallbackToCacheTimeout": 3000,
      "url": "https://u.expo.dev/YOUR_PROJECT_ID"
    }
  }
}
```

**`checkAutomatically` options:**

- `"ON_LOAD"` — Check for updates every time the app is opened. If an update is available, it is downloaded in the background and applied on the next launch.
- `"ON_ERROR_RECOVERY"` — Only check for updates if the app encounters an error loading the current bundle. This is a safety net, not a delivery mechanism.
- `"NEVER"` — Manual control only. You must call the Updates API yourself.

**For most apps, I recommend `"ON_LOAD"` with custom logic:**

```typescript
// app/_layout.tsx or similar entry point
import * as Updates from 'expo-updates';
import { useEffect } from 'react';
import { AppState, Platform } from 'react-native';

export function useOTAUpdates() {
  useEffect(() => {
    if (__DEV__) return; // No updates in development

    async function checkForUpdate() {
      try {
        const update = await Updates.checkForUpdateAsync();
        
        if (update.isAvailable) {
          // Download the update
          await Updates.fetchUpdateAsync();
          
          // Strategy: Apply on next launch (less disruptive)
          // The update will be loaded next time the app starts
          
          // OR Strategy: Apply immediately (for critical fixes)
          // await Updates.reloadAsync();
        }
      } catch (error) {
        // Don't crash the app because of an update check failure
        console.warn('Error checking for OTA update:', error);
      }
    }

    // Check on initial load
    checkForUpdate();

    // Also check when app comes to foreground
    const subscription = AppState.addEventListener(
      'change',
      (nextAppState) => {
        if (nextAppState === 'active') {
          checkForUpdate();
        }
      }
    );

    return () => subscription.remove();
  }, []);
}
```

**Critical update pattern (force update):**

```typescript
import * as Updates from 'expo-updates';
import { Alert } from 'react-native';

async function checkForCriticalUpdate() {
  try {
    const update = await Updates.checkForUpdateAsync();
    
    if (update.isAvailable) {
      const manifest = update.manifest;
      const metadata = manifest?.extra?.expoClient?.extra;
      
      // Check if the update is marked as critical
      if (metadata?.critical === true) {
        await Updates.fetchUpdateAsync();
        
        Alert.alert(
          'Update Required',
          'A critical update has been installed. The app will restart.',
          [
            {
              text: 'Restart Now',
              onPress: () => Updates.reloadAsync(),
            },
          ],
          { cancelable: false }
        );
      } else {
        // Non-critical: download silently, apply on next launch
        await Updates.fetchUpdateAsync();
      }
    }
  } catch (error) {
    console.warn('Update check failed:', error);
  }
}
```

---

## 6.5 App Store and Google Play Deployment Best Practices

### 6.5.1 App Store Connect: The Complete Workflow

**TestFlight Distribution**

TestFlight is Apple's official beta testing platform. It supports two groups:

1. **Internal Testers:** Members of your App Store Connect team (up to 100). No review required. Builds are available within minutes of upload.

2. **External Testers:** Anyone with an email address (up to 10,000). Requires TestFlight review (a lighter review than full App Review). Usually takes 24-48 hours for the first build, then faster for subsequent builds.

```bash
# Build and auto-submit to TestFlight
eas build --profile production --platform ios --auto-submit

# Or manually submit after building
eas submit --platform ios --latest
```

**Setting up TestFlight groups:**

This part is manual (done in App Store Connect):

1. Go to your app → TestFlight
2. Create internal test groups and add team members
3. Create external test groups and add email addresses or share a public link
4. When a new build appears, it is automatically distributed to internal testers
5. External testers get it after TestFlight review passes

**App Review: What Reviewers Look At**

Apple's reviewers will:

- Launch your app on a physical device
- Go through the main user flows
- Test with the demo account you provide (always provide one!)
- Check that your privacy labels match your actual data collection
- Verify that all links in the app work
- Check that your IAP configuration is correct
- Test on both iPhone and iPad if you support both

**Preparation checklist before submitting to App Review:**

```
Pre-Submission Checklist:
  ✅ App launches without crashing on physical device
  ✅ All user flows work end-to-end
  ✅ Demo account credentials provided in App Review Information
  ✅ Privacy policy URL works and is up to date
  ✅ Privacy nutrition labels accurately reflect data collection
  ✅ All NSUsageDescription strings are human-readable and accurate
  ✅ No placeholder text or lorem ipsum anywhere
  ✅ All external links work
  ✅ Screenshots match the actual app
  ✅ In-app purchases (if any) work correctly
  ✅ App functions without network (or gracefully handles no network)
  ✅ No references to other platforms ("Download on Google Play")
  ✅ Age rating questionnaire filled out accurately
```

**Privacy Nutrition Labels**

Since iOS 14, Apple requires you to declare what data your app collects. This appears on your App Store listing as "App Privacy" cards. Getting this wrong is a common rejection reason.

Here is how to think about it:

| If your app uses... | You must declare... |
|---------------------|---------------------|
| Analytics (Firebase, Amplitude, Mixpanel) | Analytics data, device ID |
| Crash reporting (Sentry, Crashlytics) | Crash data, diagnostics |
| Push notifications (with token) | Device ID |
| User authentication | Email, name (whatever you collect) |
| Advertising (AdMob, Facebook SDK) | Advertising data, device ID, tracking |
| Location services | Precise or coarse location |
| Photos/Camera | Photos or videos |

**Staged Rollout**

Once approved, you have two options:

1. **Immediate release:** Available to all users at once.
2. **Phased release:** Rolls out over 7 days (1%, 2%, 5%, 10%, 20%, 50%, 100%). You can pause or accelerate at any time.

My recommendation: **Always use phased release for production apps with more than 10,000 users.** If something goes wrong, you can pause the rollout and fix it before most users are affected.

```
Phased Release Timeline:
  Day 1:  1% of users
  Day 2:  2% of users
  Day 3:  5% of users
  Day 4:  10% of users
  Day 5:  20% of users
  Day 6:  50% of users
  Day 7:  100% of users
```

### 6.5.2 Google Play Console: The Complete Workflow

**Testing Tracks**

Google Play has a more flexible testing system than Apple:

1. **Internal testing:** Up to 100 testers. No review. Available in minutes. This is your daily build for QA.

2. **Closed testing (Alpha):** Invite-only via email or Google Groups. No review initially. Good for external beta testers or partner companies.

3. **Open testing (Beta):** Anyone can join from your Play listing. Subject to review. Good for public betas.

4. **Production:** The real deal. Subject to review. Supports staged rollouts.

**Staged Rollouts on Google Play**

Google Play's staged rollout is more flexible than Apple's:

```bash
# In eas.json, you can specify a rollout percentage
{
  "submit": {
    "production": {
      "android": {
        "track": "production",
        "rollout": 0.1
      }
    }
  }
}
```

This starts the rollout at 10%. You can then increase it from the Google Play Console:

```
Manual staged rollout:
  10% → monitor for 24h → 25% → monitor → 50% → monitor → 100%
```

**Pre-Launch Reports**

Google Play automatically tests your app on a set of real devices before release. Check the pre-launch report for:

- Crashes detected
- Security vulnerabilities
- Accessibility issues
- Performance problems

These reports are free and incredibly useful. Check them before promoting from internal to production.

**Android Vitals**

Once your app is live, Android Vitals tracks:

| Metric | Bad Threshold | What It Means |
|--------|--------------|---------------|
| ANR rate | > 0.47% | App Not Responding — your main thread is blocked |
| Crash rate | > 1.09% | Unhandled exceptions |
| Excessive wakeups | > 10 per hour | Your app is waking the device too often |
| Stuck background wake locks | > 0.30% | You're holding wake locks too long |
| Excessive background Wi-Fi scans | > 4 per hour | Stop scanning so much |

**If your vitals exceed the "bad" thresholds, Google Play will:**
1. Show a warning on your listing ("This app may not work well on your device")
2. Reduce your ranking in search results
3. In extreme cases, suppress your app

**Data Safety Section**

Google Play's equivalent of Apple's privacy labels. You must fill this out accurately.

```
Required declarations:
  ├── Does your app collect or share user data?
  ├── What types of data? (location, personal info, financial, etc.)
  ├── For what purpose? (app functionality, analytics, advertising)
  ├── Is collection optional or required?
  ├── Is data encrypted in transit?
  ├── Can users request data deletion?
  └── Is your app compliant with the Children's Online Privacy Protection Act?
```

**Play Integrity API**

The Play Integrity API (formerly SafetyNet) helps you verify that:
- The app is running on a genuine Android device
- The app binary has not been tampered with
- The user has a genuine Google Play account

For apps that handle payments, sensitive data, or competitive gaming, integrating Play Integrity is a good practice. It is not required for store submission, but it protects against modded APKs and emulator fraud.

### 6.5.3 Review Times and Expedited Reviews

**Apple App Review:**

| Situation | Typical Wait |
|-----------|-------------|
| First submission (new app) | 1-3 days |
| Subsequent updates | 12-48 hours |
| After a rejection (resubmission) | 12-48 hours |
| Expedited review (must request) | 1-24 hours |
| Holidays (Thanksgiving, Christmas) | 3-7+ days |

**When to request an expedited review:**
- Critical bug fix that affects a large number of users
- Security vulnerability
- Time-sensitive event (app for a conference happening tomorrow)

Do not request expedited review because your PM "really wants this shipped by Friday." Apple will reject the request and you will lose credibility for future requests.

```
To request expedited review:
  1. Go to https://developer.apple.com/contact/app-store/
  2. Select "Request an expedited app review"
  3. Provide a clear, honest explanation of why it's urgent
  4. Submit and wait (usually 1-24 hours)
```

**Google Play Review:**

| Situation | Typical Wait |
|-----------|-------------|
| Internal testing track | No review (minutes) |
| First production release | 1-7 days (can be longer) |
| Subsequent production updates | Hours to 1-3 days |
| After a policy violation | 3-7+ days (extra scrutiny) |
| New developer account | 3-14 days (extra scrutiny) |

Google does not offer expedited review, but their internal track bypass makes this less of an issue. Ship to internal, test there, then promote to production.

### 6.5.4 Common Pitfalls That Get You Rejected

**The "Minimum Functionality" Trap (Apple Guideline 4.2)**

Apple rejects apps they consider "not useful enough." If your app is basically a WebView wrapper around a website, it will get rejected. If your app has only one screen with minimal functionality, it might get rejected. Your app needs to provide enough value to justify its existence as a native app.

**The "Sign in with Apple" Requirement (Apple Guideline 4.8)**

If your app offers any third-party sign-in (Google, Facebook, etc.), you MUST also offer Sign in with Apple. No exceptions. This catches people off guard all the time.

**The "Guideline 4.3 Spam" Nightmare**

If you build white-label apps (the same app customized for different clients), Apple may reject them under Guideline 4.3. They consider each variant "spam" if the apps are too similar. The solution: use a single app with multi-tenancy, or make each variant significantly different in functionality.

**The Google Play "Deceptive Behavior" Trap**

Do not claim functionality in your store listing that requires a paid subscription unless you clearly state that. Do not use misleading screenshots. Do not use keywords in your description that do not relate to your app. Google's automated systems are getting better at catching this.

**Missing Physical Device Testing**

Both stores test on physical devices. "It works on the simulator" is not sufficient. Specific things that break on physical devices but not simulators:
- Push notifications (simulators do not support them)
- Camera/microphone access
- Background location
- Bluetooth
- NFC
- Performance under memory pressure

---

## 6.6 EAS Workflows — CI/CD with EAS

EAS Workflows is Expo's CI/CD system built specifically for React Native apps. It integrates with GitHub and lets you define build/test/deploy pipelines as YAML.

### 6.6.1 Basic Workflow Configuration

Create `.eas/workflows/build-and-submit.yml`:

```yaml
name: Build and Submit on Release

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  test:
    name: Run Tests
    runs_on:
      machine: medium
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Setup Node
        uses: eas/use_node@v1
        with:
          node_version: '20.18.0'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Run unit tests
        run: npx jest --ci --coverage

      - name: Run linter
        run: npx eslint . --max-warnings 0

  build_ios:
    name: Build iOS
    needs: test
    runs_on:
      machine: large
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Build iOS
        uses: eas/build@v1
        with:
          platform: ios
          profile: production

  build_android:
    name: Build Android
    needs: test
    runs_on:
      machine: large
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Build Android
        uses: eas/build@v1
        with:
          platform: android
          profile: production

  submit_ios:
    name: Submit iOS
    needs: build_ios
    runs_on:
      machine: medium
    steps:
      - name: Submit to App Store
        uses: eas/submit@v1
        with:
          platform: ios
          profile: production

  submit_android:
    name: Submit Android
    needs: build_android
    runs_on:
      machine: medium
    steps:
      - name: Submit to Google Play
        uses: eas/submit@v1
        with:
          platform: android
          profile: production
```

This workflow:
1. Runs tests, type checking, and linting on every push to `main`
2. If tests pass, builds iOS and Android in parallel
3. If builds succeed, submits to both stores

### 6.6.2 PR Preview Workflow

Create `.eas/workflows/pr-preview.yml`:

```yaml
name: PR Preview Build

on:
  pull_request:
    branches:
      - main

jobs:
  fingerprint_check:
    name: Check Native Changes
    runs_on:
      machine: medium
    outputs:
      native_changed: ${{ steps.check.outputs.changed }}
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Setup Node
        uses: eas/use_node@v1

      - name: Install dependencies
        run: npm ci

      - name: Check fingerprint
        id: check
        run: |
          CURRENT=$(npx @expo/fingerprint .)
          # Compare with main branch fingerprint
          git fetch origin main
          git checkout origin/main -- .expo/fingerprint.json 2>/dev/null || true
          PREVIOUS=$(cat .expo/fingerprint.json 2>/dev/null || echo "{}")
          if [ "$CURRENT" != "$PREVIOUS" ]; then
            echo "changed=true" >> $GITHUB_OUTPUT
            echo "Native fingerprint changed - binary build required"
          else
            echo "changed=false" >> $GITHUB_OUTPUT
            echo "No native changes - OTA update sufficient"
          fi

  build_preview:
    name: Build Preview
    needs: fingerprint_check
    if: needs.fingerprint_check.outputs.native_changed == 'true'
    runs_on:
      machine: medium
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Build Preview
        uses: eas/build@v1
        with:
          platform: all
          profile: preview

  update_preview:
    name: Publish OTA Preview
    needs: fingerprint_check
    if: needs.fingerprint_check.outputs.native_changed == 'false'
    runs_on:
      machine: medium
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Setup Node
        uses: eas/use_node@v1

      - name: Install dependencies
        run: npm ci

      - name: Publish update
        uses: eas/update@v1
        with:
          branch: pr-${{ github.event.pull_request.number }}
          message: "PR #${{ github.event.pull_request.number }}: ${{ github.event.pull_request.title }}"
```

This smart workflow:
1. Checks if native code changed by comparing fingerprints
2. If native code changed: triggers a full binary build (preview profile)
3. If only JS changed: publishes an OTA update (much faster and cheaper)

This is the kind of optimization that saves teams thousands of dollars per month in build credits.

### 6.6.3 Hotfix Workflow

Create `.eas/workflows/hotfix.yml`:

```yaml
name: Hotfix OTA Update

on:
  workflow_dispatch:
    inputs:
      message:
        description: 'Update message'
        required: true
      critical:
        description: 'Is this a critical update?'
        required: false
        default: 'false'

jobs:
  deploy_hotfix:
    name: Deploy Hotfix
    runs_on:
      machine: medium
    steps:
      - name: Checkout
        uses: eas/checkout@v1

      - name: Setup Node
        uses: eas/use_node@v1

      - name: Install dependencies
        run: npm ci

      - name: Run smoke tests
        run: npx jest --ci --testPathPattern='smoke'

      - name: Publish to production
        uses: eas/update@v1
        with:
          branch: production
          message: "HOTFIX: ${{ github.event.inputs.message }}"

      - name: Notify Slack
        if: always()
        run: |
          STATUS=${{ job.status }}
          curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 Hotfix $STATUS: ${{ github.event.inputs.message }}\"}" \
            ${{ secrets.SLACK_WEBHOOK_URL }}
```

This workflow is manually triggered (workflow_dispatch) for emergency hotfixes. It runs smoke tests and then publishes an OTA update directly to production.

### 6.6.4 Using GitHub Actions Instead of EAS Workflows

If you prefer GitHub Actions (or need to run your pipeline on GitHub's infrastructure), you can still use EAS commands in your GHA workflow:

```yaml
# .github/workflows/eas-build.yml
name: EAS Build
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Setup EAS
        uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build iOS
        run: eas build --platform ios --profile production --non-interactive

      - name: Build Android
        run: eas build --platform android --profile production --non-interactive

      - name: Submit iOS
        run: eas submit --platform ios --latest --non-interactive

      - name: Submit Android
        run: eas submit --platform android --latest --non-interactive
```

**Key difference:** With GitHub Actions, the builds still happen on EAS infrastructure. GitHub Actions is just triggering the builds. With EAS Workflows, the entire pipeline runs on EAS infrastructure. EAS Workflows has the advantage of tighter integration (build artifacts flow between steps automatically) but GitHub Actions gives you more flexibility for non-EAS tasks (running your own Docker containers, integrating with other services, etc.).

**My recommendation:**
- Use EAS Workflows if your pipeline is primarily build/submit/update
- Use GitHub Actions if you have complex CI requirements (e2e tests, custom Docker, multiple services)
- You can mix both: use GitHub Actions for testing and EAS Workflows for building/deploying

---

## 6.7 Pricing and Optimization

Understanding EAS pricing is critical for teams at scale. A careless setup can blow through your budget quickly.

### 6.7.1 Pricing Tiers (as of 2026)

| | Free | Production | Enterprise |
|---|---|---|---|
| **Build credits/month** | 30 | Varies by plan | Custom |
| **Update bandwidth** | 1 GB | 20+ GB | Custom |
| **Build concurrency** | 1 | 2-4 | Custom |
| **Priority queue** | No | Yes | Yes |
| **Build worker size** | Medium only | Medium + Large | Custom |
| **Support** | Community | Email | Dedicated |
| **Price** | $0 | From $99/month | Contact sales |

**Build credit math:**

One build credit equals one minute of build time on a medium worker. Large workers consume 2 credits per minute.

| Build Type | Medium Worker | Large Worker |
|---|---|---|
| iOS production | ~20 min = 20 credits | ~10 min = 20 credits |
| Android production | ~12 min = 12 credits | ~6 min = 12 credits |
| iOS + Android | ~32 credits | ~32 credits |

Wait — look at that. On large workers, you spend the same credits but get your builds twice as fast. This is why I always recommend large workers for production builds. You pay the same, you just get results faster.

**The calculation changes for development builds:**

| Build Type | Medium Worker | Large Worker |
|---|---|---|
| iOS dev client | ~15 min = 15 credits | ~8 min = 16 credits |
| Android dev client | ~10 min = 10 credits | ~5 min = 10 credits |

For dev builds, medium workers are cheaper in credit terms because the compile step is shorter (debug builds are faster than release builds).

### 6.7.2 Optimization Strategies

**Strategy 1: Minimize unnecessary builds**

The single biggest cost optimization is not building when you do not need to. Use fingerprinting to determine whether a PR requires a new binary build or if an OTA update is sufficient:

```
PR changes JS only?        → OTA update (0 build credits)
PR changes native code?    → Binary build (20-32 credits)
```

For a team that ships 20 PRs a week, and only 3 of them touch native code, this alone saves:

```
Without optimization: 20 PRs x 32 credits = 640 credits/week
With optimization:    3 PRs x 32 credits + 17 OTA = 96 credits/week
Savings: 85%
```

**Strategy 2: Build platform-specific only when needed**

Do not build both iOS and Android if only one platform is affected. If a PR only changes an iOS-specific config plugin, only build iOS.

**Strategy 3: Use the right build profile**

Do not use `production` profile for QA testing. Use `preview`. The `preview` profile can use medium workers and skip some production-only optimizations.

**Strategy 4: Cache aggressively**

Proper cache configuration (as described in section 6.2.7) can save 2-5 minutes per build. Over hundreds of builds per month, that adds up to significant credit savings.

**Strategy 5: Avoid building on every commit**

Set up your CI to only trigger builds on:
- Merges to `main` (not every push to feature branches)
- Manual triggers for feature branch builds
- Tags for production releases

```yaml
# Only build on main or tags
on:
  push:
    branches: [main]
    tags: ['v*']
  # Allow manual triggers for feature branches
  workflow_dispatch:
```

**Strategy 6: Update bandwidth optimization**

EAS Update charges for bandwidth. To minimize bandwidth:

1. **Keep your JS bundle small.** Tree-shake unused code. Lazy-load screens.
2. **Optimize assets.** Compress images before importing them. Use WebP instead of PNG where possible.
3. **Use asset fingerprinting.** Unchanged assets are not re-downloaded.

```bash
# Check your update size before publishing
npx expo export --platform ios --platform android
du -sh dist/
# If dist/ is > 10MB, you might want to optimize
```

### 6.7.3 Free Tier Strategies for Startups

If you are on the free tier (30 credits/month), here is how to make it work:

1. **Build dev clients locally.** Use `npx expo run:ios` and `npx expo run:android` on your local machine for development builds. Only use EAS Build for production.

2. **Build one platform at a time.** Do not build both platforms unless you need both.

3. **Use OTA updates aggressively.** With fingerprinting, most of your updates can go OTA for free.

4. **Combine builds.** If you have multiple fixes, batch them into a single build instead of building for each one.

5. **Use the Android emulator and iOS Simulator.** You can build dev clients for simulators locally in about 5 minutes.

```
Monthly budget on free tier:
  30 credits ÷ 32 credits per dual-platform build = ~1 production build/month
  
  OR
  
  30 credits ÷ 20 credits per iOS build = ~1.5 iOS builds/month
  
  Plus unlimited OTA updates (within 1GB bandwidth)
```

This is tight but workable for a solo developer or very early stage startup. Once you are shipping regularly, the $99/month Production plan pays for itself in time savings.

---

## 6.8 Code Signing Explained

Code signing is one of those topics that most developers avoid until it breaks. Then they spend three days in a fog of certificates, profiles, and keychain errors. Let me demystify it.

### 6.8.1 Why Code Signing Exists

Code signing serves two purposes:

1. **Identity:** It proves that the app was built by you (or your organization). Users and the OS can verify that the app has not been tampered with since you signed it.

2. **Authorization:** It proves that you have the right to distribute the app. Apple and Google use code signing to enforce their distribution policies.

Without code signing, anyone could modify your app binary, inject malware, and redistribute it. The App Store and Google Play would have no way to verify that the app is legitimate.

### 6.8.2 iOS Code Signing: The Full Picture

iOS code signing is famously complex. Let me walk through every component.

**Certificates**

```
Apple Root Certificate Authority
  └── Apple Worldwide Developer Relations CA
        └── Your Distribution Certificate
              ├── Public key (in the certificate)
              └── Private key (in your keychain / EAS)
```

There are two types of certificates you care about:

1. **Development Certificate:** Used to sign apps for development (running on your test devices).
2. **Distribution Certificate:** Used to sign apps for distribution (App Store, TestFlight, Ad Hoc).

Each Apple Developer account can have a limited number of active distribution certificates:

| Account Type | Distribution Certificates |
|---|---|
| Individual | 3 |
| Organization | 3 |
| Enterprise | 3 |

**Important:** The certificate contains only the public key. The private key is generated on the machine that created the certificate request. If you lose the private key, you must revoke the certificate and create a new one.

This is why EAS credential management is so valuable. EAS securely stores both the certificate and the private key. You never have to worry about which team member's MacBook has the only copy of the private key.

**Provisioning Profiles**

A provisioning profile is a document that ties together:

1. An App ID (your bundle identifier)
2. A certificate (which proves your identity)
3. A list of device UDIDs (for development and ad hoc distribution)
4. Entitlements (what the app is allowed to do)

```
Provisioning Profile Types:
  
  Development:
    - App ID: com.yourcompany.yourapp
    - Certificate: Your Development Certificate
    - Devices: [iPhone-A, iPhone-B, iPad-C]
    - Entitlements: [push-notifications, background-modes]
    - Distribution: None (development only)

  Ad Hoc:
    - App ID: com.yourcompany.yourapp
    - Certificate: Your Distribution Certificate
    - Devices: [iPhone-A, iPhone-B, iPad-C, ...]
    - Entitlements: [push-notifications, background-modes]
    - Distribution: Internal (limited to listed devices)

  App Store:
    - App ID: com.yourcompany.yourapp
    - Certificate: Your Distribution Certificate
    - Devices: None (not device-limited)
    - Entitlements: [push-notifications, background-modes]
    - Distribution: App Store / TestFlight
```

**EAS and provisioning profiles:**

When you build with EAS, it automatically:
1. Creates or uses an existing distribution certificate
2. Creates the appropriate provisioning profile type based on your build profile:
   - `distribution: "internal"` → Ad Hoc profile
   - `distribution: "store"` (or production builds) → App Store profile
3. Registers devices for ad hoc profiles (via `eas device:create`)
4. Embeds the profile into the signed binary

**Device Registration for Ad Hoc Builds**

For internal distribution (preview/development builds), EAS uses ad hoc provisioning profiles. These profiles must include the UDID of every device that will run the app.

```bash
# Register a new device
eas device:create

# This generates a URL. The person with the device opens the URL,
# which installs a configuration profile that reports the UDID back to EAS.

# After registering new devices, rebuild to get an updated provisioning profile
eas build --profile preview --platform ios
```

**The 100-Device Limit**

Apple limits ad hoc provisioning profiles to 100 devices per year (per device type). This means you can register up to 100 iPhones, 100 iPads, etc. Device registrations reset annually when you renew your developer membership.

For larger teams, consider Apple's **Enterprise Distribution** program or use TestFlight (which has no device limit).

**Entitlements**

Entitlements are capabilities that your app requests from iOS:

```xml
<!-- Entitlements.plist (generated by EAS/Xcode) -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
  <key>aps-environment</key>
  <string>production</string>
  
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:yourapp.com</string>
    <string>webcredentials:yourapp.com</string>
  </array>
  
  <key>com.apple.developer.applesignin</key>
  <array>
    <string>Default</string>
  </array>
</dict>
</plist>
```

Common entitlements:
- `aps-environment`: Push notifications (development or production)
- `com.apple.developer.associated-domains`: Universal links, web credentials
- `com.apple.developer.applesignin`: Sign in with Apple
- `com.apple.developer.in-app-payments`: Apple Pay
- `com.apple.developer.healthkit`: HealthKit access
- `com.apple.developer.icloud-container-identifiers`: iCloud

In Expo, you configure entitlements through `app.json` and config plugins:

```json
{
  "expo": {
    "ios": {
      "entitlements": {
        "com.apple.developer.associated-domains": [
          "applinks:yourapp.com"
        ]
      },
      "infoPlist": {
        "UIBackgroundModes": ["remote-notification", "fetch"]
      }
    }
  }
}
```

### 6.8.3 Android Code Signing: The Full Picture

Android code signing is simpler in structure but has its own gotchas.

**Keystores**

An Android keystore is a file (`.jks` or `.keystore`) that contains one or more key pairs (private key + certificate).

```
Keystore File (.jks)
  └── Key Alias: "upload-key"
        ├── Private key
        ├── Certificate
        └── Password
  └── Keystore password (protects the file itself)
```

You need:
1. **Keystore password:** To open the keystore file
2. **Key alias:** To select which key to use (a keystore can contain multiple keys)
3. **Key password:** To use the specific key

**Google Play App Signing**

Since 2021, Google Play manages the "app signing key" — the key that signs the APK/AAB that users download. You provide an "upload key" — the key that signs the artifact you upload to Google Play.

```
Your machine / EAS:
  Signs with Upload Key → Uploads to Google Play

Google Play:
  Verifies Upload Key → Re-signs with App Signing Key → Delivers to users

Why this matters:
  If you lose your Upload Key → Google can reset it for you
  If you lost your App Signing Key → Your app would be dead
  But Google manages the App Signing Key → You can't lose it
```

**This is a massive safety improvement.** Before Google Play App Signing, losing your keystore meant your app was dead. You could never update it again. Now, the worst case is a temporary inconvenience while Google resets your upload key.

**How EAS handles Android signing:**

When you run `eas build` for Android the first time:

1. EAS generates a keystore with an upload key
2. Stores it encrypted on EAS servers
3. Uses it to sign every subsequent build
4. When you first submit to Google Play, EAS provides the upload key and Google generates the app signing key

```bash
# View your Android credentials
eas credentials --platform android

# Download your keystore (for backup or other CI systems)
eas credentials --platform android
# Select "Download keystore"

# The keystore is a .jks file. Guard it carefully.
```

**Debug vs Release Signing**

Development builds are signed with a debug keystore that Android Studio auto-generates:

```
Debug keystore: ~/.android/debug.keystore
  Key alias: androiddebugkey
  Keystore password: android
  Key password: android
```

This is universal — every Android developer has the same debug keystore. It is only for development. Release builds must be signed with your project's keystore.

**Key Rotation (Rare but Important)**

If you suspect your upload key is compromised:

1. Go to Google Play Console → Setup → App Signing
2. Request an upload key reset
3. Google will generate a new upload key for you
4. Download it and configure it in EAS

```bash
# After Google resets your upload key:
eas credentials --platform android
# Select "Upload new keystore"
# Upload the new keystore from Google Play Console
```

### 6.8.4 Code Signing Best Practices

**For both platforms:**

1. **Let EAS manage credentials** unless you have a specific reason not to. EAS handles certificate rotation, profile management, and secure storage. Doing it yourself is a maintenance burden.

2. **Back up your credentials.** Even with EAS managing them, download backups:
   ```bash
   eas credentials --platform ios   # Download certificate + profile
   eas credentials --platform android  # Download keystore
   ```
   Store backups in a secure location (1Password, AWS Secrets Manager, etc.).

3. **Never commit credentials to git.** Add these to `.gitignore`:
   ```
   *.p12
   *.p8
   *.mobileprovision
   *.jks
   *.keystore
   google-service-account.json
   ```

4. **Use EAS Secrets for CI/CD.** Instead of storing credentials in your repository or CI environment variables:
   ```bash
   eas secret:create --scope project --name GOOGLE_SERVICE_ACCOUNT \
     --type file --value ./google-service-account.json
   ```

5. **Rotate credentials proactively.** iOS distribution certificates expire after one year. Do not wait until they expire — plan certificate rotation during a low-traffic period.

**iOS-specific:**

6. **Enroll in the Apple Developer Enterprise Program** only if you need internal distribution to 100+ devices. It is $299/year and has stricter compliance requirements.

7. **Use App Store distribution for TestFlight.** TestFlight does not have the 100-device limit that ad hoc distribution has.

8. **Check your certificate expiration dates:**
   ```bash
   eas credentials --platform ios
   # Look for "Expiration Date" in the output
   ```

**Android-specific:**

9. **Always enroll in Google Play App Signing.** There is no good reason not to. It is free and protects you from keystroke loss.

10. **Use a strong keystore password.** The debug keystore uses "android" as the password. Your production keystore should use a strong, unique password stored in a secrets manager.

---

## 6.9 Putting It All Together: The Complete Release Workflow

Let me paint the picture of how a mature team uses all of this together. This is the workflow I recommend for teams shipping weekly:

### 6.9.1 The Weekly Release Cycle

```
Monday:
  ├── Engineers merge PRs to main
  ├── Each PR triggers: lint → test → fingerprint check
  │     ├── JS-only change? → OTA update to preview branch
  │     └── Native change? → Preview build triggered
  └── QA tests on preview builds throughout the day

Tuesday-Wednesday:
  ├── More PRs merge
  ├── QA continues testing on preview builds/updates
  └── Bug fixes ship as OTA updates to preview

Thursday (Release Cut):
  ├── Release manager creates a release branch: release/1.3.0
  ├── Final QA on release branch
  ├── eas build --profile production --platform all --auto-submit
  │     ├── iOS: Submitted to App Store Connect → TestFlight
  │     └── Android: Submitted to Google Play → Internal testing
  └── PM and QA do final smoke test on TestFlight / internal track

Friday:
  ├── iOS: Submit for App Review
  │     └── eas metadata:push (update release notes)
  ├── Android: Promote internal → production (staged at 10%)
  └── Release manager monitors for issues

Following Week:
  ├── iOS: Approved → Phased release starts
  ├── Android: Increase rollout 10% → 25% → 50% → 100%
  ├── Monitor crash rates and user reports
  └── Hotfixes ship as OTA updates (no new binary needed)
```

### 6.9.2 The Hotfix Flow

```
1. Bug reported in production
2. Engineer creates fix on hotfix branch
3. Fix is merged to main
4. Fingerprint check: JS-only change? 
   ├── Yes → eas update --branch production --message "Fix: crash on checkout"
   │         (Live to users in minutes, no store review)
   └── No → eas build --profile production --auto-submit
             (Full build + store submission cycle)
```

The key insight: **most hotfixes are JS-only.** The crash was in your checkout logic, not in a native module. OTA gets the fix to users in minutes instead of days.

### 6.9.3 Complete eas.json for This Workflow

Here is the `eas.json` that supports this entire workflow:

```json
{
  "cli": {
    "version": ">= 13.0.0",
    "appVersionSource": "remote",
    "promptToConfigurePushNotifications": false
  },
  "build": {
    "base": {
      "node": "20.18.0",
      "cache": {
        "key": "v1",
        "cacheDefaultPaths": true,
        "customPaths": ["~/.ccache"]
      },
      "env": {
        "EXPO_NO_TELEMETRY": "1",
        "SENTRY_ORG": "your-org",
        "SENTRY_PROJECT": "your-app"
      }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "resourceClass": "medium",
      "ios": {
        "simulator": false
      },
      "android": {
        "buildType": "apk",
        "gradleCommand": ":app:assembleDebug"
      },
      "channel": "development",
      "env": {
        "APP_ENV": "development",
        "API_URL": "https://api-dev.yourapp.com"
      }
    },
    "development:simulator": {
      "extends": "development",
      "ios": {
        "simulator": true
      }
    },
    "preview": {
      "extends": "base",
      "distribution": "internal",
      "resourceClass": "medium",
      "ios": {
        "buildConfiguration": "Release"
      },
      "android": {
        "buildType": "apk"
      },
      "channel": "preview",
      "env": {
        "APP_ENV": "staging",
        "API_URL": "https://api-staging.yourapp.com"
      }
    },
    "production": {
      "extends": "base",
      "resourceClass": "large",
      "autoIncrement": true,
      "ios": {
        "buildConfiguration": "Release",
        "image": "macos-ventura-14.2-xcode-15.1"
      },
      "android": {
        "buildType": "app-bundle"
      },
      "channel": "production",
      "env": {
        "APP_ENV": "production",
        "API_URL": "https://api.yourapp.com"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./keys/google-service-account.json",
        "track": "internal",
        "releaseStatus": "completed"
      }
    }
  }
}
```

### 6.9.4 Commands Cheat Sheet

```bash
# ─── BUILDING ───────────────────────────────────────

# Build dev client for testing
eas build --profile development --platform ios
eas build --profile development --platform android

# Build preview for QA
eas build --profile preview --platform all

# Build production
eas build --profile production --platform all

# Build + auto submit
eas build --profile production --platform all --auto-submit

# View build status
eas build:list

# View specific build logs
eas build:view BUILD_ID

# Cancel a running build
eas build:cancel BUILD_ID


# ─── SUBMITTING ─────────────────────────────────────

# Submit latest build
eas submit --platform ios --latest
eas submit --platform android --latest

# Submit specific build
eas submit --platform ios --id BUILD_ID

# Submit local file
eas submit --platform ios --path ./app.ipa


# ─── UPDATING (OTA) ────────────────────────────────

# Publish update
eas update --branch production --message "Description"

# Publish with auto message from git
eas update --branch production --auto

# Publish for specific platform
eas update --branch production --platform ios

# List recent updates
eas update:list --branch production

# View update details
eas update:view UPDATE_GROUP_ID

# Republish a previous update
eas update:republish --group UPDATE_GROUP_ID


# ─── CHANNELS ──────────────────────────────────────

# List channels
eas channel:list

# View channel details
eas channel:view production

# Point channel to different branch (rollback!)
eas channel:edit production --branch production-rollback

# Create new channel
eas channel:create staging


# ─── CREDENTIALS ───────────────────────────────────

# Manage iOS credentials
eas credentials --platform ios

# Manage Android credentials
eas credentials --platform android

# Create a secret
eas secret:create --scope project --name MY_SECRET --value "secret-value"

# List secrets
eas secret:list


# ─── VERSIONING ────────────────────────────────────

# View remote version
eas build:version:get --platform ios
eas build:version:get --platform android

# Set remote version
eas build:version:set --platform ios --build-number 100
eas build:version:set --platform android --version-code 100


# ─── DEVICES (for internal distribution) ───────────

# Register a new device
eas device:create

# List registered devices
eas device:list

# Delete a device
eas device:delete DEVICE_UDID


# ─── METADATA ──────────────────────────────────────

# Pull metadata from stores
eas metadata:pull

# Push metadata to stores
eas metadata:push
```

---

## 6.10 Decision Trees and Mental Models

### When to Build vs Update

```
┌─────────────────────────────────────────┐
│         Did native code change?          │
│     (run: npx @expo/fingerprint .)       │
└─────────────────┬───────────────────────┘
                  │
          ┌───────┴───────┐
          │               │
         YES              NO
          │               │
    ┌─────▼──────┐  ┌────▼─────┐
    │ EAS Build  │  │EAS Update│
    │ + Submit   │  │  (OTA)   │
    │            │  │          │
    │ Days to    │  │ Minutes  │
    │ reach users│  │ to reach │
    │            │  │ users    │
    └────────────┘  └──────────┘
```

### Choosing a Resource Class

```
┌──────────────────────────────────────┐
│    What are you building?            │
└───────────────┬──────────────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
   Dev       Preview    Production
    │           │           │
  Medium     Medium      Large
    │           │           │
  ~$0.30     ~$0.30     ~$0.60/min
  per min    per min    (but 2x faster)
```

### Choosing a Distribution Method

```
┌──────────────────────────────────────────────┐
│          Who needs this build?                │
└─────────────────┬────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
  Engineers      QA/PM        Users
    │             │             │
  Dev Client   Preview       Production
    │             │             │
  Internal     Internal      Store
  distribution distribution  distribution
    │             │             │
  EAS build    EAS build     EAS build
  --profile    --profile     --profile
  development  preview       production
                              --auto-submit
```

---

## 6.11 Real-World War Stories

### The Keystore That Vanished

A startup I advised had their Android keystore on the CTO's laptop. The CTO left the company. The laptop was reformatted. Nobody had a backup. The app — which had 50,000 users — could never be updated again. They had to publish a new app under a new package name and ask all their users to migrate.

**Lesson:** Use EAS credential management or back up your keystore in at least two secure locations. Enroll in Google Play App Signing. This failure mode is 100% preventable.

### The OTA That Crashed Production

A team shipped an OTA update that called a new native module they had just added. The module was in their development branch and the fingerprint had changed — but they had hardcoded the runtime version instead of using fingerprinting. The OTA was delivered to users running the old binary (without the new native module). The app crashed on launch for every user.

**Lesson:** Use `"policy": "fingerprint"` for runtime versioning. Do not hardcode it. Do not use `sdkVersion` policy. Fingerprinting exists to prevent exactly this scenario.

### The Christmas Eve Rejection

A gaming company submitted their holiday-themed update on December 22. Apple rejected it on December 23 for a minor metadata issue (a broken support URL). They fixed it and resubmitted, but Apple's review team was on reduced capacity for the holidays. The update was not approved until January 3. They missed the entire holiday season — their biggest revenue period.

**Lesson:** Submit your holiday updates at least 2-3 weeks before major holidays. Apple publishes their holiday schedule in advance. Plan around it. Also, if the update is JS-only changes and the binary is already approved, use OTA updates to ship time-sensitive content.

### The 100-Device Limit Surprise

A team with 50 engineers and 30 QA testers tried to distribute preview builds via EAS internal distribution (ad hoc). They hit the 100-device limit. Half the QA team could not install the preview builds. They panicked and almost bought an Enterprise Developer account ($299/year, with compliance requirements).

**Lesson:** For teams with more than ~30 testers, use TestFlight for iOS distribution instead of ad hoc. TestFlight has no device limit. The tradeoff is that TestFlight requires a build review (12-48 hours for external testers), but internal testers (App Store Connect team members, up to 100 accounts) get builds instantly with no review.

---

## 6.12 Chapter Summary

EAS is the deployment backbone of modern Expo apps. Here is what you should take away:

1. **EAS Build** compiles your native code in the cloud. Use build profiles to separate development, preview, and production concerns. Let EAS manage your credentials.

2. **EAS Submit** automates store uploads. Use `--auto-submit` to chain builds and submissions. Use EAS Metadata to manage store listings as code.

3. **EAS Update** ships JS changes over the air in minutes. Use fingerprinting for runtime versioning. Use channels and branches for deployment topology. Know when OTA is and is not appropriate.

4. **Code signing** is complex but manageable. Understand the components (certificates, profiles, keystores) even if EAS manages them for you. Always have backups. Always enroll in Google Play App Signing.

5. **CI/CD** ties it all together. Use EAS Workflows or GitHub Actions to automate the entire pipeline. The smart play is using fingerprinting to decide between binary builds and OTA updates.

6. **Store deployment** requires planning. Know the review timelines, common rejection reasons, and staged rollout strategies for both Apple and Google.

The teams that master this pipeline ship confidently, fix bugs in minutes, and never lose a weekend to a broken certificate. The teams that don't, well, they are still uploading `.ipa` files through Transporter and praying.

Don't be the second team.

---

> **Next up — Chapter 7:** We dive into navigation architecture with Expo Router, file-based routing, deep linking, and building navigation patterns that scale from 5 screens to 500.
