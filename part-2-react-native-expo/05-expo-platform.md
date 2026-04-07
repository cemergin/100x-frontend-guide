<!--
  CHAPTER: 5
  TITLE: Expo: The Modern React Native Platform
  PART: II — React Native & Expo
  PREREQS: Chapter 1
  KEY_TOPICS: Expo SDK 54, CNG, prebuild, config plugins, Expo Modules API, managed vs bare, React Compiler, expo-dev-client
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 5: Expo — The Modern React Native Platform

---

## The Thirty-Second Version

If you're starting a React Native project in 2026 and you're not using Expo, you need a very good reason. Not a "my tech lead heard bad things in 2019" reason. A real, technical, "we have 200k lines of Objective-C we can't touch" reason. Expo went from being the training wheels of React Native to being the chassis, the engine, and the transmission. This chapter explains how that happened, what Continuous Native Generation actually means for your architecture, and how to wield Expo's full power without hitting the walls that used to exist.

---

## 5.1 Why Expo Won

### The Old Narrative

Let's start with what people used to say about Expo, because if you've been in the React Native ecosystem for more than a couple years, you've heard some version of this:

> "Expo is great for prototypes, but for production apps you need to eject."

> "You can't use custom native modules with Expo."

> "Expo apps are bloated because they include the entire SDK."

> "Real engineers don't use Expo."

Every single one of these statements was, at some point, partially true. And every single one of them is now wrong.

Here's what the Expo of 2019 looked like:

- A monolithic runtime that bundled every native module into your binary
- A managed workflow that gave you zero access to native code
- An "eject" process that was one-way and terrifying
- No way to use arbitrary native libraries
- Build infrastructure that was slow and opaque

The criticism was legitimate. If you needed a native module that Expo didn't ship, you were stuck. If you ejected, you lost all of Expo's tooling benefits. It was a binary choice: convenience or control. And serious apps almost always needed control.

### What Changed

Three innovations flipped the entire calculus:

1. **Continuous Native Generation (CNG)** — The native `ios/` and `android/` directories are generated artifacts, not source code you maintain
2. **Expo Modules API** — A modern, type-safe way to write native modules that works seamlessly with Expo's build system
3. **Config Plugins** — Declarative modifications to native project configuration without maintaining native code

These three changes didn't just improve Expo. They made the concept of "ejecting" obsolete. There's nothing to eject from anymore because you were never locked in.

### The Numbers Tell the Story

Let's look at where the ecosystem is now:

| Metric | 2020 | 2023 | 2026 |
|--------|------|------|------|
| New RN projects using Expo | ~25% | ~55% | ~85% |
| React Native's official recommendation | "One of several options" | "Recommended framework" | "The recommended way" |
| Companies using Expo in production | Hundreds | Thousands | Tens of thousands |
| EAS Build monthly builds | N/A | ~500K | ~4M |
| Expo SDK modules available | ~35 | ~50 | ~70+ |

The React Native team at Meta officially recommends Expo as the framework for new React Native projects. Not as a wrapper. Not as training wheels. As the framework. This happened because Expo solved problems that the bare React Native workflow never did:

- **Reproducible builds** — Your CI produces the same binary regardless of what's installed on the build machine
- **Upgradeable native code** — When React Native releases a new version, you don't manually migrate Xcode project files
- **Declarative native configuration** — You describe what you want in TypeScript, not in XML and Plist files
- **Unified tooling** — One CLI for development, building, updating, and submitting

### Who's Using Expo in Production

This isn't a hobby tool. Here's a sample of companies shipping with Expo:

- **Shopify** — Shop app, one of the most downloaded commerce apps globally
- **Discord** — Migrated significant portions of their mobile stack to React Native with Expo
- **Coinbase** — Fintech with extreme security and compliance requirements
- **Pinterest** — Billions of monthly impressions
- **Microsoft** — Multiple apps across their portfolio
- **Kraken** — Cryptocurrency exchange handling billions in daily volume
- **Flexport** — Enterprise logistics
- **Cameo** — Video marketplace

These aren't startups who picked Expo because it was easy. These are companies with dedicated native engineering teams who picked Expo because it was better.

### The Cultural Shift

There's a subtler change that matters here. The React Native community went through a phase where using Expo was seen as a signal that you weren't a "real" native developer. That attitude has completely reversed. Now, maintaining raw `ios/` and `android/` directories when you don't have to is seen as unnecessary complexity — the same way nobody hand-writes webpack configs anymore when Next.js handles it for you.

The 100x architect understands this: **the goal is not to demonstrate mastery of complexity. The goal is to ship reliable software with minimal accidental complexity.** Expo eliminates an enormous category of accidental complexity from React Native development.

---

## 5.2 Continuous Native Generation (CNG)

### The Core Idea

CNG is the single most important concept in modern Expo development. If you understand nothing else in this chapter, understand this.

Here's the premise: **your native `ios/` and `android/` directories are not source code. They are build artifacts.**

In traditional React Native development, you have a project that looks like this:

```
my-app/
  android/
    app/
      build.gradle
      src/
        main/
          AndroidManifest.xml
          java/
            com/myapp/
              MainActivity.java
              MainApplication.java
    build.gradle
    gradle.properties
    settings.gradle
  ios/
    MyApp/
      AppDelegate.mm
      Info.plist
    MyApp.xcodeproj/
      project.pbxproj    <-- This file is a war crime
    Podfile
    Podfile.lock
  src/
    App.tsx
  package.json
```

You see those `android/` and `ios/` directories? In a traditional React Native project, those are yours to maintain. When you install a native library, you modify the Podfile. When you need a new permission, you edit Info.plist. When React Native upgrades, you painstakingly merge changes into your Xcode project file — a binary-esque file format that was never designed for human editing or version control.

This is where an extraordinary amount of React Native pain lived. The `project.pbxproj` file alone has been the source of more merge conflicts, more corrupted projects, and more wasted engineering hours than any other file in the mobile development world.

CNG says: **stop doing that.** Instead, describe your native configuration in `app.config.ts`, and let Expo generate the native projects at build time.

```
my-expo-app/
  src/
    App.tsx
  app.config.ts       <-- This is your native configuration
  package.json
```

That's it. No `ios/` directory. No `android/` directory. They get generated when you need them.

### How Prebuild Works

The `npx expo prebuild` command is where CNG happens. Here's the flow:

```
app.config.ts
     │
     ▼
┌─────────────┐
│  expo        │
│  prebuild    │ ◄── reads installed packages
│              │ ◄── runs config plugins
└─────────────┘
     │
     ▼
┌─────────────┐     ┌─────────────┐
│   ios/       │     │  android/    │
│   directory  │     │  directory   │
│  (generated) │     │  (generated) │
└─────────────┘     └─────────────┘
     │                     │
     ▼                     ▼
  xcodebuild           gradle
     │                     │
     ▼                     ▼
   .ipa                 .apk/.aab
```

Let's trace through what happens when you run `npx expo prebuild`:

**Step 1: Read the app config**

Expo reads your `app.config.ts` (or `app.json`) and resolves all dynamic values — environment variables, conditional logic, computed properties.

**Step 2: Apply the template**

Expo starts with a base native project template. This template matches the React Native version you're using and includes all the boilerplate that every RN app needs.

**Step 3: Apply intrinsic config**

Properties from your app config get mapped to their native equivalents:

```typescript
// app.config.ts
export default {
  name: "My App",
  slug: "my-app",
  version: "1.0.0",
  ios: {
    bundleIdentifier: "com.mycompany.myapp",
    buildNumber: "42",
    supportsTablet: true,
  },
  android: {
    package: "com.mycompany.myapp",
    versionCode: 42,
    adaptiveIcon: {
      foregroundImage: "./assets/adaptive-icon.png",
      backgroundColor: "#FFFFFF",
    },
  },
};
```

This gets translated to:

- `Info.plist` entries for `CFBundleIdentifier`, `CFBundleVersion`
- `build.gradle` entries for `applicationId`, `versionCode`
- Android manifest entries
- Xcode project settings

**Step 4: Run config plugins**

This is the extensibility layer. Every installed Expo package (and many community packages) ships with a config plugin that modifies the native project. For example, `expo-camera` includes a plugin that:

- Adds the `NSCameraUsageDescription` key to `Info.plist`
- Adds the `CAMERA` permission to `AndroidManifest.xml`
- Links the native camera module

**Step 5: Run autolinking**

Expo scans your `node_modules` for packages that have native code and automatically configures the native projects to include them. No manual `pod install` or Gradle configuration needed.

**Step 6: Output native projects**

The generated `ios/` and `android/` directories are written to disk. They're fully functional native projects that you can open in Xcode or Android Studio.

### The Prebuild Command in Practice

```bash
# Generate both platforms
npx expo prebuild

# Generate only iOS
npx expo prebuild --platform ios

# Clean generate (delete existing native dirs first)
npx expo prebuild --clean

# Skip dependency installation
npx expo prebuild --no-install
```

The `--clean` flag is important. If you've generated native projects before and want a fresh start (say, after upgrading Expo SDK), `--clean` wipes the existing directories and regenerates from scratch. This is the "I want a clean slate" button that traditional React Native projects never had.

### Should You Gitignore the Native Directories?

This is one of the most common questions, and the answer reveals whether you've fully bought into CNG.

**The Expo recommendation: Yes, add `ios/` and `android/` to `.gitignore`.**

```gitignore
# .gitignore
/ios
/android
```

This feels scary the first time. "But what if I need to debug native code?" You still can — just run `npx expo prebuild` and the directories appear. Open Xcode. Debug. When you're done, your changes should be codified as a config plugin, not as manual edits to generated files.

Think of it like this: you wouldn't commit your `node_modules` directory. You wouldn't commit your webpack output. The native directories are the same category — derived artifacts that can be reproduced from source.

**When to NOT gitignore them:**

- You have a brownfield app with custom native code that can't be expressed as config plugins
- Your CI system can't run `prebuild` (rare, but some enterprise setups have constraints)
- You have native engineers who actively work in Xcode/Android Studio and need those directories in version control

### The Upgrade Story

Here's where CNG really shines. Let's compare upgrading React Native with and without CNG.

**Without CNG (traditional bare workflow):**

1. Read the upgrade guide
2. Run the upgrade helper tool
3. Manually apply dozens of changes to `ios/` and `android/` files
4. Resolve merge conflicts in `project.pbxproj` (pain)
5. Fix broken CocoaPods (more pain)
6. Fix Gradle version mismatches (even more pain)
7. Pray that nothing silently broke
8. Spend 1-5 days on this process

**With CNG:**

1. Update `expo` and React Native versions in `package.json`
2. Run `npx expo prebuild --clean`
3. Done

That's not a simplification. That's literally the process. Because the native projects are generated from templates that Expo maintains, upgrading means generating from the new template. All the Xcode project file migrations, all the Gradle changes, all the CocoaPods updates — Expo handles it.

I've seen teams that budgeted two-week sprints for React Native upgrades. With CNG, it's a morning task.

### CNG and Reproducible Builds

CNG gives you something that traditional React Native projects struggle with: reproducible builds.

In a traditional project, your native build depends on:
- What version of Xcode is installed on the build machine
- What version of CocoaPods is installed
- What's in the global Gradle cache
- Whether someone ran `pod install` with the right flags
- The phase of the moon

With CNG, your build depends on:
- Your `package.json` (locked by `package-lock.json` or `yarn.lock`)
- Your `app.config.ts`
- The Expo SDK version

That's it. Two developers on different machines, with different global tool versions, will get the same native project from `npx expo prebuild`. This is the kind of determinism that makes CI/CD reliable and debugging possible.

### A Mental Model for CNG

If you've used infrastructure-as-code tools like Terraform or Pulumi, CNG will feel familiar. Instead of manually provisioning servers and hoping you documented every change, you declare your desired state and let the tool figure out how to get there.

| IaC Concept | CNG Equivalent |
|-------------|----------------|
| `main.tf` | `app.config.ts` |
| `terraform plan` | `npx expo prebuild` |
| `terraform apply` | `npx expo run:ios` / `eas build` |
| Terraform providers | Config plugins |
| State file | Generated `ios/` and `android/` dirs |
| `terraform destroy` | Delete `ios/` and `android/` dirs |

The analogy isn't perfect, but it captures the key insight: **you declare what you want, not how to get it.**

---

## 5.3 Config Plugins

### The Extension Mechanism

Config plugins are how you customize native code without maintaining it. They're functions that modify the native project configuration during prebuild. If CNG is the engine, config plugins are the tuning parameters.

Here's the simplest possible config plugin:

```typescript
// my-plugin.ts
import { ConfigPlugin } from "expo/config-plugins";

const withMyPlugin: ConfigPlugin = (config) => {
  // Modify the config object
  return config;
};

export default withMyPlugin;
```

A config plugin receives the Expo config, modifies it, and returns it. That's the entire API surface. The power comes from what you can modify.

### The Plugin API

Expo provides a set of "mod" functions that give you access to specific native configuration files:

#### iOS Mods

| Mod | What it modifies | Common use cases |
|-----|-----------------|------------------|
| `withInfoPlist` | `Info.plist` | Permissions, URL schemes, background modes |
| `withEntitlementsPlist` | `*.entitlements` | App groups, push notifications, sign-in with Apple |
| `withXcodeProject` | `project.pbxproj` | Build settings, frameworks, resources |
| `withPodfile` | `Podfile` | CocoaPods dependencies, build settings |
| `withPodfileProperties` | `Podfile.properties.json` | Flipper, new architecture flags |
| `withAppDelegate` | `AppDelegate.mm` | Startup code, deep link handlers |

#### Android Mods

| Mod | What it modifies | Common use cases |
|-----|-----------------|------------------|
| `withAndroidManifest` | `AndroidManifest.xml` | Permissions, intent filters, metadata |
| `withMainActivity` | `MainActivity.java/kt` | Activity configuration |
| `withMainApplication` | `MainApplication.java/kt` | App startup code |
| `withProjectBuildGradle` | Root `build.gradle` | Repositories, classpath dependencies |
| `withAppBuildGradle` | App `build.gradle` | Dependencies, build types, signing configs |
| `withStringsXml` | `strings.xml` | String resources |
| `withColors` | `colors.xml` | Color resources |

### Example: Adding Background Location Permission

Let's build a real config plugin. Say your app needs background location access. Here's how you do it without ever touching native code:

```typescript
// plugins/withBackgroundLocation.ts
import {
  ConfigPlugin,
  withInfoPlist,
  withAndroidManifest,
} from "expo/config-plugins";

const withBackgroundLocation: ConfigPlugin<{
  locationAlwaysUsageDescription?: string;
}> = (config, { locationAlwaysUsageDescription } = {}) => {
  // iOS: Add background location permission and background mode
  config = withInfoPlist(config, (config) => {
    // Add the permission description
    config.modResults.NSLocationAlwaysAndWhenInUseUsageDescription =
      locationAlwaysUsageDescription ||
      "This app needs access to your location in the background to provide navigation updates.";

    config.modResults.NSLocationAlwaysUsageDescription =
      locationAlwaysUsageDescription ||
      "This app needs access to your location in the background to provide navigation updates.";

    // Enable the background mode
    if (!config.modResults.UIBackgroundModes) {
      config.modResults.UIBackgroundModes = [];
    }
    if (!config.modResults.UIBackgroundModes.includes("location")) {
      config.modResults.UIBackgroundModes.push("location");
    }

    return config;
  });

  // Android: Add fine location and background location permissions
  config = withAndroidManifest(config, (config) => {
    const mainApplication = config.modResults.manifest;

    // Ensure the permissions array exists
    if (!mainApplication["uses-permission"]) {
      mainApplication["uses-permission"] = [];
    }

    const permissions = mainApplication["uses-permission"];

    const addPermission = (name: string) => {
      if (!permissions.some((p) => p.$?.["android:name"] === name)) {
        permissions.push({
          $: { "android:name": name },
        });
      }
    };

    addPermission("android.permission.ACCESS_FINE_LOCATION");
    addPermission("android.permission.ACCESS_COARSE_LOCATION");
    addPermission("android.permission.ACCESS_BACKGROUND_LOCATION");
    addPermission("android.permission.FOREGROUND_SERVICE");
    addPermission("android.permission.FOREGROUND_SERVICE_LOCATION");

    return config;
  });

  return config;
};

export default withBackgroundLocation;
```

Now use it in your app config:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from "expo/config";

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: "My Navigation App",
  slug: "my-nav-app",
  plugins: [
    [
      "./plugins/withBackgroundLocation",
      {
        locationAlwaysUsageDescription:
          "We use your location in the background to provide turn-by-turn navigation.",
      },
    ],
  ],
});
```

When you run `npx expo prebuild`, this plugin will:
1. Add the location permission strings to `Info.plist`
2. Add the `location` background mode to `UIBackgroundModes`
3. Add the necessary Android permissions to `AndroidManifest.xml`

No Xcode. No Android Studio. No manual XML editing. Declarative, version-controlled, and reproducible.

### Example: Adding a Custom URL Scheme

```typescript
// plugins/withCustomScheme.ts
import { ConfigPlugin, withInfoPlist, withAndroidManifest } from "expo/config-plugins";

const withCustomScheme: ConfigPlugin<{ scheme: string }> = (config, { scheme }) => {
  // iOS
  config = withInfoPlist(config, (config) => {
    const existingSchemes = config.modResults.CFBundleURLTypes || [];
    const hasScheme = existingSchemes.some(
      (type: any) => type.CFBundleURLSchemes?.includes(scheme)
    );

    if (!hasScheme) {
      existingSchemes.push({
        CFBundleURLSchemes: [scheme],
      });
    }

    config.modResults.CFBundleURLTypes = existingSchemes;
    return config;
  });

  // Android
  config = withAndroidManifest(config, (config) => {
    const mainActivity =
      config.modResults.manifest.application?.[0]?.activity?.find(
        (a: any) => a.$?.["android:name"] === ".MainActivity"
      );

    if (mainActivity) {
      if (!mainActivity["intent-filter"]) {
        mainActivity["intent-filter"] = [];
      }

      mainActivity["intent-filter"].push({
        action: [{ $: { "android:name": "android.intent.action.VIEW" } }],
        category: [
          { $: { "android:name": "android.intent.category.DEFAULT" } },
          { $: { "android:name": "android.intent.category.BROWSABLE" } },
        ],
        data: [{ $: { "android:scheme": scheme } }],
      });
    }

    return config;
  });

  return config;
};

export default withCustomScheme;
```

### Example: Adding a Gradle Dependency

```typescript
// plugins/withGradleDependency.ts
import { ConfigPlugin, withAppBuildGradle } from "expo/config-plugins";

const withGradleDependency: ConfigPlugin<{
  dependency: string;
  configuration?: string;
}> = (config, { dependency, configuration = "implementation" }) => {
  return withAppBuildGradle(config, (config) => {
    if (config.modResults.language === "groovy") {
      const depLine = `    ${configuration} '${dependency}'`;

      // Avoid duplicates
      if (!config.modResults.contents.includes(depLine)) {
        config.modResults.contents = config.modResults.contents.replace(
          /dependencies\s*\{/,
          `dependencies {\n${depLine}`
        );
      }
    }

    return config;
  });
};

export default withGradleDependency;
```

### Third-Party Config Plugins

You don't have to write every plugin yourself. The ecosystem has matured to the point where most popular native libraries ship their own config plugins.

```typescript
// app.config.ts
export default {
  plugins: [
    // Expo's own plugins — many are auto-configured
    "expo-router",
    "expo-camera",
    "expo-location",

    // Third-party plugins
    ["@react-native-firebase/app", { /* options */ }],
    ["react-native-maps", { googleMapsApiKey: process.env.GOOGLE_MAPS_KEY }],
    ["@sentry/react-native/expo", { organization: "my-org", project: "my-app" }],
    ["expo-build-properties", {
      ios: { deploymentTarget: "15.1" },
      android: { compileSdkVersion: 34, targetSdkVersion: 34, minSdkVersion: 24 },
    }],

    // Your own custom plugins
    ["./plugins/withBackgroundLocation", { /* options */ }],
  ],
};
```

### Plugin Execution Order

Plugins run in the order they're listed in the `plugins` array. This matters when plugins modify the same files. If plugin A adds a permission and plugin B reads the permission list, B needs to come after A.

In practice, ordering issues are rare because most plugins modify different parts of the native configuration. But when you hit a weird build error after adding a new plugin, the execution order is worth checking.

### Debugging Config Plugins

When a config plugin isn't doing what you expect, here's the debugging workflow:

```bash
# 1. Generate native projects
npx expo prebuild --clean

# 2. Inspect the generated files
cat ios/MyApp/Info.plist
cat android/app/src/main/AndroidManifest.xml

# 3. Look for your changes
# If they're not there, your plugin has a bug

# 4. Run prebuild with verbose logging
npx expo prebuild --clean 2>&1 | grep -i "plugin\|error\|warn"
```

You can also use the `expo-config-plugin-tests` package to write unit tests for your plugins:

```typescript
// __tests__/withBackgroundLocation.test.ts
import { withBackgroundLocation } from "../plugins/withBackgroundLocation";
import { ExpoConfig } from "expo/config";

describe("withBackgroundLocation", () => {
  it("adds background location mode to Info.plist", () => {
    const config: ExpoConfig = {
      name: "test",
      slug: "test",
    };

    const result = withBackgroundLocation(config, {
      locationAlwaysUsageDescription: "Test description",
    });

    // Inspect the result...
  });
});
```

### The Config Plugin Decision Tree

When should you write a config plugin vs. doing something else?

```
Do you need to modify native project files?
├── No → You don't need a config plugin
└── Yes
    ├── Is there an existing Expo SDK module for this?
    │   └── Yes → Use the SDK module (it has a built-in plugin)
    ├── Is there a community plugin?
    │   └── Yes → Use it (check maintenance status first)
    └── No
        ├── Is the change to a configuration file (plist, manifest, gradle)?
        │   └── Yes → Write a config plugin ← THIS IS THE SWEET SPOT
        └── Is the change to source code (Swift, Kotlin, ObjC)?
            ├── Simple (add a line to AppDelegate) → Config plugin with dangerous mod
            └── Complex (new native module) → Expo Modules API (Section 5.5)
```

---

## 5.4 App Config (`app.config.ts`)

### The Brain of Your Expo App

The app config is where everything about your app's identity, behavior, and native configuration lives. It's the single source of truth that drives CNG, and getting it right is one of the most impactful things you can do for your project's long-term health.

### `app.json` vs. `app.config.ts`

Expo supports two formats:

| Feature | `app.json` | `app.config.ts` |
|---------|-----------|-----------------|
| Static values | Yes | Yes |
| Environment variables | No | Yes |
| Conditional logic | No | Yes |
| TypeScript | No | Yes |
| Imports | No | Yes |
| Comments | No | Yes |
| Computed values | No | Yes |
| IDE autocomplete | Limited | Full |

There's no reason to use `app.json` for any non-trivial project. Use `app.config.ts`. You get TypeScript, you get dynamic values, you get the ability to import helper functions. The only reason `app.json` still exists is backward compatibility and the simplicity of the "getting started" experience.

One important note: if you have both `app.json` and `app.config.ts`, the config file wins. The JSON is loaded first and passed to the config function as the `config` parameter in the `ConfigContext`. So you can use `app.json` as a base and override in TypeScript:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from "expo/config";

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,  // Spread the base from app.json
  name: process.env.APP_ENV === "production" ? "My App" : "My App (Dev)",
});
```

### A Production-Ready `app.config.ts`

Here's a comprehensive app config that I'd consider production-grade. I'll annotate every section:

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from "expo/config";

// ─── Environment Setup ────────────────────────────────────────────
// Expo supports .env files natively. Values prefixed with EXPO_PUBLIC_
// are available in client-side code. Others are only available at build time.

const IS_PRODUCTION = process.env.APP_ENV === "production";
const IS_STAGING = process.env.APP_ENV === "staging";
const IS_DEVELOPMENT = !IS_PRODUCTION && !IS_STAGING;

// Use different app identifiers per environment so you can install
// all three variants on the same device simultaneously.
const APP_IDENTIFIER = IS_PRODUCTION
  ? "com.mycompany.myapp"
  : IS_STAGING
    ? "com.mycompany.myapp.staging"
    : "com.mycompany.myapp.dev";

const APP_NAME = IS_PRODUCTION
  ? "My App"
  : IS_STAGING
    ? "My App (Staging)"
    : "My App (Dev)";

export default ({ config }: ConfigContext): ExpoConfig => ({
  // ─── Identity ─────────────────────────────────────────────────────
  // These define your app's identity across both platforms.
  name: APP_NAME,
  slug: "my-app",
  owner: "my-expo-account",         // Your Expo account or organization
  version: "2.4.1",                 // User-facing version string
  orientation: "portrait",
  scheme: "myapp",                   // Deep link scheme: myapp://

  // ─── Assets ───────────────────────────────────────────────────────
  icon: "./assets/icon.png",         // 1024x1024 PNG, no transparency
  splash: {
    image: "./assets/splash.png",    // Splash screen image
    resizeMode: "contain",
    backgroundColor: "#1A1A2E",
  },
  userInterfaceStyle: "automatic",   // Respects system dark/light mode

  // ─── Expo-Specific ────────────────────────────────────────────────
  // These control Expo's services (EAS, updates, etc.)
  runtimeVersion: {
    policy: "appVersion",            // OTA updates are scoped to app version
  },
  updates: {
    url: "https://u.expo.dev/your-project-id",
    fallbackToCacheTimeout: 3000,    // Wait 3s for update check before loading cached bundle
    checkAutomatically: "ON_LOAD",
    enabled: !IS_DEVELOPMENT,        // Disable OTA updates in dev
  },

  // ─── iOS Configuration ────────────────────────────────────────────
  ios: {
    bundleIdentifier: APP_IDENTIFIER,
    buildNumber: "142",              // Increment for each App Store submission
    supportsTablet: true,
    config: {
      usesNonExemptEncryption: false, // Required: does your app use custom crypto?
    },
    infoPlist: {
      // Add any Info.plist entries directly
      ITSAppUsesNonExemptEncryption: false,
      LSApplicationQueriesSchemes: ["mailto", "tel", "sms"],
    },
    entitlements: {
      "aps-environment": IS_PRODUCTION ? "production" : "development",
      "com.apple.developer.associated-domains": [
        `applinks:${IS_PRODUCTION ? "myapp.com" : "staging.myapp.com"}`,
      ],
    },
    associatedDomains: [
      `applinks:${IS_PRODUCTION ? "myapp.com" : "staging.myapp.com"}`,
    ],
    privacyManifests: {
      NSPrivacyAccessedAPITypes: [
        {
          NSPrivacyAccessedAPIType:
            "NSPrivacyAccessedAPICategoryUserDefaults",
          NSPrivacyAccessedAPITypeReasons: ["CA92.1"],
        },
      ],
    },
  },

  // ─── Android Configuration ────────────────────────────────────────
  android: {
    package: APP_IDENTIFIER,
    versionCode: 142,                // Integer, must increment for Play Store
    adaptiveIcon: {
      foregroundImage: "./assets/adaptive-icon.png",
      backgroundColor: "#1A1A2E",
    },
    permissions: [
      "CAMERA",
      "READ_EXTERNAL_STORAGE",
      "WRITE_EXTERNAL_STORAGE",
      "ACCESS_FINE_LOCATION",
      "ACCESS_COARSE_LOCATION",
      "RECEIVE_BOOT_COMPLETED",
      "VIBRATE",
    ],
    intentFilters: [
      {
        action: "VIEW",
        autoVerify: true,
        data: [
          {
            scheme: "https",
            host: IS_PRODUCTION ? "myapp.com" : "staging.myapp.com",
            pathPrefix: "/app",
          },
        ],
        category: ["BROWSABLE", "DEFAULT"],
      },
    ],
    googleServicesFile: IS_PRODUCTION
      ? "./config/google-services-prod.json"
      : "./config/google-services-dev.json",
  },

  // ─── Web Configuration ────────────────────────────────────────────
  web: {
    bundler: "metro",
    output: "server",                // SSR support with expo-router
    favicon: "./assets/favicon.png",
  },

  // ─── Plugins ──────────────────────────────────────────────────────
  plugins: [
    "expo-router",
    "expo-secure-store",
    ["expo-camera", {
      cameraPermission: "Allow $(PRODUCT_NAME) to access your camera to scan documents.",
      microphonePermission: "Allow $(PRODUCT_NAME) to access your microphone for video recording.",
    }],
    ["expo-location", {
      locationAlwaysAndWhenInUsePermission:
        "Allow $(PRODUCT_NAME) to use your location for navigation.",
      locationAlwaysPermission:
        "Allow $(PRODUCT_NAME) to use your location in the background.",
      locationWhenInUsePermission:
        "Allow $(PRODUCT_NAME) to use your location.",
      isIosBackgroundLocationEnabled: true,
      isAndroidBackgroundLocationEnabled: true,
      isAndroidForegroundServiceEnabled: true,
    }],
    ["expo-notifications", {
      icon: "./assets/notification-icon.png",
      color: "#1A1A2E",
    }],
    ["expo-image-picker", {
      photosPermission: "Allow $(PRODUCT_NAME) to access your photos.",
    }],
    ["expo-build-properties", {
      ios: {
        deploymentTarget: "15.1",
        useFrameworks: "static",
      },
      android: {
        compileSdkVersion: 35,
        targetSdkVersion: 35,
        minSdkVersion: 24,
        kotlinVersion: "1.9.24",
      },
    }],
    ["@sentry/react-native/expo", {
      organization: process.env.SENTRY_ORG,
      project: process.env.SENTRY_PROJECT,
    }],
    [
      "expo-font",
      {
        fonts: [
          "./assets/fonts/Inter-Regular.ttf",
          "./assets/fonts/Inter-Medium.ttf",
          "./assets/fonts/Inter-SemiBold.ttf",
          "./assets/fonts/Inter-Bold.ttf",
        ],
      },
    ],
  ],

  // ─── Experiments ──────────────────────────────────────────────────
  experiments: {
    typedRoutes: true,               // Type-safe routing with expo-router
  },

  // ─── Extra ────────────────────────────────────────────────────────
  // Values here are available at runtime via Constants.expoConfig.extra
  extra: {
    eas: {
      projectId: "your-eas-project-id-here",
    },
    apiUrl: IS_PRODUCTION
      ? "https://api.myapp.com"
      : IS_STAGING
        ? "https://api-staging.myapp.com"
        : "http://localhost:3000",
    sentryDsn: process.env.SENTRY_DSN,
    environment: process.env.APP_ENV || "development",
  },
});
```

### Environment Variables

Expo has first-class support for environment variables. Here's how it works:

**Build-time variables** — Available in `app.config.ts` and during the build process:

```bash
# .env.local (gitignored)
SENTRY_DSN=https://abc@sentry.io/123
SENTRY_ORG=my-org
SENTRY_PROJECT=my-app
GOOGLE_MAPS_KEY=AIzaSy...
```

**Client-side variables** — Available in your JavaScript code at runtime. Must be prefixed with `EXPO_PUBLIC_`:

```bash
# .env
EXPO_PUBLIC_API_URL=https://api.myapp.com
EXPO_PUBLIC_POSTHOG_KEY=phc_abc123
EXPO_PUBLIC_ENVIRONMENT=production
```

Access them in your code:

```typescript
// Works anywhere in your client code
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
const posthogKey = process.env.EXPO_PUBLIC_POSTHOG_KEY;
```

**Environment file precedence:**

```
.env                    # Loaded in all cases
.env.local              # Loaded in all cases, gitignored
.env.[mode]             # Loaded for specific mode (e.g., .env.production)
.env.[mode].local       # Loaded for specific mode, gitignored
```

The mode is set via the `--mode` flag or `APP_ENV` environment variable:

```bash
# Uses .env.production
APP_ENV=production npx expo start

# Or with EAS Build
# eas.json profiles set the environment
```

### Accessing Config at Runtime

Your app config values are available at runtime through the `expo-constants` package:

```typescript
import Constants from "expo-constants";

// Access the extra field
const apiUrl = Constants.expoConfig?.extra?.apiUrl;
const environment = Constants.expoConfig?.extra?.environment;

// Access standard fields
const appVersion = Constants.expoConfig?.version;
const appName = Constants.expoConfig?.name;
```

### The `extra` vs `EXPO_PUBLIC_` Decision

Both mechanisms get values into your runtime code. Which should you use?

| Approach | When to use |
|----------|-------------|
| `EXPO_PUBLIC_*` env vars | Simple key-value pairs, feature flags, API URLs |
| `extra` in app config | Computed values, values derived from build-time logic |
| Config plugins | Values that need to modify native configuration |

My recommendation: use `EXPO_PUBLIC_*` for most things. It's the simplest, most standard approach. Use `extra` when you need to compute values in your config function. Use config plugins when you need native-side configuration.

---

## 5.5 Expo Modules API

### The End of the Old Bridge

Before Expo Modules API (and the New Architecture's TurboModules), writing a native module for React Native was a painful exercise in boilerplate. You had to:

1. Write Objective-C++ or Java code
2. Implement a specific protocol/interface
3. Manually serialize types between JS and native
4. Handle threading yourself
5. Write platform-specific module registration code
6. Pray that the bridge serialization didn't silently drop your data

The Expo Modules API replaces all of this with a modern, type-safe, Swift-and-Kotlin-first approach. It's built on top of the New Architecture (JSI) when available, and it falls back gracefully when it's not.

### Why It Matters for Architects

Even if you never write a native module yourself, the Expo Modules API matters to you for two reasons:

1. **It's what Expo's own SDK uses** — Every `expo-*` package is built with it. Understanding the architecture helps you debug issues.
2. **It dramatically lowers the cost of going native** — When your team needs a native capability that doesn't exist in the ecosystem, the cost of building it yourself dropped from "dedicate a native engineer for two weeks" to "a senior RN developer can do it in a day."

### Anatomy of an Expo Module

An Expo module has this structure:

```
my-native-module/
  src/
    index.ts                    # TypeScript API
    MyModule.types.ts           # Type definitions
  ios/
    MyModule.swift              # iOS implementation
    MyModule.podspec            # CocoaPods spec (auto-generated by create-expo-module)
  android/
    src/main/java/expo/modules/mymodule/
      MyModule.kt               # Android implementation
    build.gradle                # Gradle config
  expo-module.config.json       # Module metadata
```

### A Real Example: Device Haptics Module

Let's build a simple module that provides haptic feedback. This shows the complete flow from native to JavaScript.

**Step 1: Scaffold the module**

```bash
npx create-expo-module my-haptics
```

**Step 2: Define the TypeScript API**

```typescript
// src/index.ts
import MyHapticsModule from "./MyHapticsModule";

export type HapticStyle = "light" | "medium" | "heavy" | "success" | "warning" | "error";

export function trigger(style: HapticStyle = "medium"): void {
  return MyHapticsModule.trigger(style);
}

export function isSupported(): boolean {
  return MyHapticsModule.isSupported();
}
```

**Step 3: iOS Implementation (Swift)**

```swift
// ios/MyHapticsModule.swift
import ExpoModulesCore
import UIKit

public class MyHapticsModule: Module {
  public func definition() -> ModuleDefinition {
    Name("MyHaptics")

    Function("isSupported") {
      return UIDevice.current.value(forKey: "_feedbackSupportLevel") as? Int ?? 0 > 0
    }

    Function("trigger") { (style: String) in
      switch style {
      case "light":
        let generator = UIImpactFeedbackGenerator(style: .light)
        generator.impactOccurred()
      case "medium":
        let generator = UIImpactFeedbackGenerator(style: .medium)
        generator.impactOccurred()
      case "heavy":
        let generator = UIImpactFeedbackGenerator(style: .heavy)
        generator.impactOccurred()
      case "success":
        let generator = UINotificationFeedbackGenerator()
        generator.notificationOccurred(.success)
      case "warning":
        let generator = UINotificationFeedbackGenerator()
        generator.notificationOccurred(.warning)
      case "error":
        let generator = UINotificationFeedbackGenerator()
        generator.notificationOccurred(.error)
      default:
        let generator = UIImpactFeedbackGenerator(style: .medium)
        generator.impactOccurred()
      }
    }
  }
}
```

**Step 4: Android Implementation (Kotlin)**

```kotlin
// android/src/main/java/expo/modules/myhaptics/MyHapticsModule.kt
package expo.modules.myhaptics

import android.os.Build
import android.os.VibrationEffect
import android.os.Vibrator
import android.os.VibratorManager
import android.content.Context
import expo.modules.kotlin.modules.Module
import expo.modules.kotlin.modules.ModuleDefinition

class MyHapticsModule : Module() {
  override fun definition() = ModuleDefinition {
    Name("MyHaptics")

    Function("isSupported") {
      val vibrator = getVibrator()
      vibrator?.hasVibrator() ?: false
    }

    Function("trigger") { style: String ->
      val vibrator = getVibrator() ?: return@Function

      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        val effect = when (style) {
          "light" -> VibrationEffect.createPredefined(VibrationEffect.EFFECT_TICK)
          "medium" -> VibrationEffect.createPredefined(VibrationEffect.EFFECT_CLICK)
          "heavy" -> VibrationEffect.createPredefined(VibrationEffect.EFFECT_HEAVY_CLICK)
          "success", "warning", "error" ->
            VibrationEffect.createPredefined(VibrationEffect.EFFECT_DOUBLE_CLICK)
          else -> VibrationEffect.createPredefined(VibrationEffect.EFFECT_CLICK)
        }
        vibrator.vibrate(effect)
      } else {
        @Suppress("DEPRECATION")
        vibrator.vibrate(50)
      }
    }
  }

  private fun getVibrator(): Vibrator? {
    val context = appContext.reactContext ?: return null
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
      val manager = context.getSystemService(Context.VIBRATOR_MANAGER_SERVICE) as? VibratorManager
      manager?.defaultVibrator
    } else {
      @Suppress("DEPRECATION")
      context.getSystemService(Context.VIBRATOR_SERVICE) as? Vibrator
    }
  }
}
```

**Step 5: Module config**

```json
// expo-module.config.json
{
  "platforms": ["ios", "android"],
  "ios": {
    "modules": ["MyHapticsModule"]
  },
  "android": {
    "modules": ["expo.modules.myhaptics.MyHapticsModule"]
  }
}
```

**Step 6: Use it in your app**

```typescript
import { trigger, isSupported } from "my-haptics";

function SaveButton() {
  const handlePress = async () => {
    await saveData();
    if (isSupported()) {
      trigger("success");
    }
  };

  return <Button onPress={handlePress} title="Save" />;
}
```

### What the Module Definition API Supports

The `ModuleDefinition` DSL is rich. Here's what you can do:

```swift
public func definition() -> ModuleDefinition {
  // Name the module
  Name("MyModule")

  // Synchronous functions (called on JS thread)
  Function("syncMethod") { (arg: String) -> String in
    return arg.uppercased()
  }

  // Async functions (called on a background thread)
  AsyncFunction("asyncMethod") { (url: String) -> Data in
    let data = try await URLSession.shared.data(from: URL(string: url)!)
    return data.0
  }

  // Constants exported to JS
  Constants {
    return ["PI": Double.pi, "platform": "ios"]
  }

  // Events that can be sent to JS
  Events("onProgress", "onComplete")

  // Views (native UI components)
  View(MyNativeView.self) {
    Prop("title") { (view, title: String) in
      view.titleLabel.text = title
    }

    Events("onTap")
  }

  // Lifecycle callbacks
  OnCreate {
    // Module was created
  }

  OnDestroy {
    // Module is being destroyed
  }

  OnAppEntersForeground {
    // App came to foreground
  }

  OnAppEntersBackground {
    // App went to background
  }
}
```

### Expo Modules vs. TurboModules

If you're familiar with the React Native New Architecture, you might wonder how Expo Modules relate to TurboModules. Here's the comparison:

| Feature | Expo Modules API | TurboModules (Raw) |
|---------|-----------------|-------------------|
| Language | Swift / Kotlin | ObjC++ / Java |
| Type safety | Built-in | Manual codegen |
| Boilerplate | Minimal | Significant |
| Works without New Architecture | Yes (graceful fallback) | No |
| View support | Built-in `View` DSL | Fabric components (complex) |
| Event support | Built-in `Events` | Manual implementation |
| Learning curve | Moderate | Steep |
| Integration with Expo ecosystem | Native | Requires adapters |
| Performance | Excellent (JSI when available) | Excellent (JSI native) |

My recommendation: **use Expo Modules API unless you have a specific reason not to.** The specific reasons would be: you're contributing to React Native core, you need to squeeze out the absolute last microsecond of performance, or you're working in a codebase that's already heavily invested in raw TurboModules.

---

## 5.6 Expo SDK 54+ Features

### The Current Landscape

Expo SDK 54 (the latest stable as of this writing) represents the maturation of many features that were experimental in earlier versions. Let's walk through the highlights that matter for production apps.

### React Compiler Integration

This is the headline feature. The React Compiler (formerly React Forget) automatically memoizes your components. You no longer need to manually wrap things in `useMemo`, `useCallback`, and `React.memo` — the compiler does it for you at build time.

```typescript
// Before: Manual memoization hell
const MemoizedList = React.memo(function UserList({ users, onSelect }) {
  const sortedUsers = useMemo(
    () => users.sort((a, b) => a.name.localeCompare(b.name)),
    [users]
  );

  const handleSelect = useCallback(
    (id: string) => {
      onSelect(id);
    },
    [onSelect]
  );

  return (
    <FlatList
      data={sortedUsers}
      renderItem={({ item }) => (
        <UserRow user={item} onSelect={handleSelect} />
      )}
    />
  );
});

// After: The React Compiler handles memoization automatically
function UserList({ users, onSelect }) {
  const sortedUsers = users.sort((a, b) => a.name.localeCompare(b.name));

  const handleSelect = (id: string) => {
    onSelect(id);
  };

  return (
    <FlatList
      data={sortedUsers}
      renderItem={({ item }) => (
        <UserRow user={item} onSelect={handleSelect} />
      )}
    />
  );
}
```

The code is cleaner, there are fewer opportunities for bugs (stale closures from wrong dependency arrays), and performance is equal or better because the compiler is more precise about what actually needs memoization.

**How to enable it:**

```typescript
// app.json or app.config.ts
{
  "experiments": {
    "reactCompiler": true
  }
}
```

Or in `metro.config.js` for more control:

```javascript
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");

const config = getDefaultConfig(__dirname);

config.transformer = {
  ...config.transformer,
  experimentalImportSupport: false,
  inlineRequires: true,
};

module.exports = config;
```

The numbers are compelling: **83% of EAS builds now have the React Compiler enabled.** It's not experimental anymore. It's the default path. If you're not using it, you're leaving performance on the table and writing more code than you need to.

**Important caveats:**

- The compiler assumes your components follow the Rules of React. If you have components that mutate state directly or have side effects in render, the compiler might change behavior.
- You can opt out specific components with the `"use no memo"` directive.
- Check the compiler output with `npx react-compiler-healthcheck` before enabling globally.

### `expo-image`

`expo-image` is the definitive replacement for React Native's built-in `Image` component. It's not a minor improvement — it's a fundamental upgrade.

```typescript
import { Image } from "expo-image";

// Basic usage
<Image
  source="https://example.com/photo.jpg"
  style={{ width: 300, height: 200 }}
  contentFit="cover"
  transition={200}
/>

// With blurhash placeholder (instant perceived load)
<Image
  source="https://example.com/photo.jpg"
  placeholder={{ blurhash: "LKO2:N%2Tw=w]~RBVZRi};RPxuwH" }}
  contentFit="cover"
  transition={300}
  style={{ width: "100%", aspectRatio: 16 / 9 }}
/>

// SVG support (no extra library needed)
<Image
  source={require("./assets/logo.svg")}
  style={{ width: 100, height: 100 }}
/>

// Animated images (GIF, APNG, WebP animation)
<Image
  source="https://example.com/animation.gif"
  style={{ width: 200, height: 200 }}
  autoplay={true}
/>
```

Why it matters:

| Feature | RN `Image` | `expo-image` |
|---------|-----------|--------------|
| Caching | Basic, unreliable | Aggressive disk + memory cache |
| Formats | JPEG, PNG, GIF | JPEG, PNG, GIF, WebP, AVIF, SVG, HEIC, animated |
| Placeholders | None | Blurhash, thumbhash, low-res, solid color |
| Transitions | None | Customizable fade/cross-dissolve |
| Memory | Frequent OOM on large lists | Efficient recycling |
| Content fit | `resizeMode` (limited) | `contentFit` + `contentPosition` (CSS-like) |
| Performance | JS-side decoding | Native decoding, zero bridge overhead |

If you're still using the built-in `Image` component, switch. Today. It's a drop-in replacement for most cases.

### Expo Router v4

Expo Router brings file-based routing to React Native, inspired by Next.js. Version 4 (shipping with SDK 54) adds several important features:

```
app/
  _layout.tsx           # Root layout
  index.tsx             # Home screen (/)
  (tabs)/
    _layout.tsx         # Tab navigator layout
    home.tsx            # /home tab
    search.tsx          # /search tab
    profile.tsx         # /profile tab
  settings/
    _layout.tsx         # Settings stack layout
    index.tsx           # /settings
    notifications.tsx   # /settings/notifications
    privacy.tsx         # /settings/privacy
  [user]/
    index.tsx           # /[user] - dynamic route
    posts/
      [id].tsx          # /[user]/posts/[id] - nested dynamic
  +not-found.tsx        # 404 handler
  +html.tsx             # Custom HTML wrapper (web)
```

Key features in v4:

**Typed routes** — Full TypeScript support for route parameters:

```typescript
// With experiments.typedRoutes enabled in app.config.ts
import { router } from "expo-router";

// Type error: '/setttings' is not a valid route
router.push("/setttings");

// Correct — TypeScript knows this route exists
router.push("/settings/notifications");

// Dynamic routes are typed too
router.push({
  pathname: "/[user]/posts/[id]",
  params: { user: "john", id: "123" },
});
```

**API Routes** — Server-side endpoints, just like Next.js:

```typescript
// app/api/users+api.ts
import { ExpoRequest, ExpoResponse } from "expo-router/server";

export async function GET(request: ExpoRequest) {
  const users = await db.users.findMany();
  return ExpoResponse.json(users);
}

export async function POST(request: ExpoRequest) {
  const body = await request.json();
  const user = await db.users.create({ data: body });
  return ExpoResponse.json(user, { status: 201 });
}
```

**Universal links** — Deep links and web URLs from the same route definition:

```typescript
// app.config.ts
{
  scheme: "myapp",
  web: {
    output: "server",
  },
  experiments: {
    typedRoutes: true,
  },
}
```

A route at `app/products/[id].tsx` automatically handles:
- Web: `https://myapp.com/products/123`
- Deep link: `myapp://products/123`
- Universal link: `https://myapp.com/products/123` (on mobile, opens the app)

### `expo-camera`

The camera module got a significant rewrite in recent SDKs. The new API is simpler and more powerful:

```typescript
import { CameraView, useCameraPermissions } from "expo-camera";
import { useState, useRef } from "react";

function CameraScreen() {
  const [permission, requestPermission] = useCameraPermissions();
  const [facing, setFacing] = useState<"front" | "back">("back");
  const cameraRef = useRef<CameraView>(null);

  if (!permission) return <View />;

  if (!permission.granted) {
    return (
      <View style={styles.container}>
        <Text>We need camera access to scan documents</Text>
        <Button onPress={requestPermission} title="Grant Permission" />
      </View>
    );
  }

  const takePicture = async () => {
    if (cameraRef.current) {
      const photo = await cameraRef.current.takePictureAsync({
        quality: 0.8,
        exif: true,
      });
      console.log(photo.uri);
    }
  };

  return (
    <View style={styles.container}>
      <CameraView
        ref={cameraRef}
        style={styles.camera}
        facing={facing}
        barcodeScannerSettings={{
          barcodeTypes: ["qr", "ean13"],
        }}
        onBarcodeScanned={(result) => {
          console.log("Scanned:", result.data);
        }}
      >
        <View style={styles.controls}>
          <Button
            title="Flip"
            onPress={() => setFacing(f => f === "back" ? "front" : "back")}
          />
          <Button title="Capture" onPress={takePicture} />
        </View>
      </CameraView>
    </View>
  );
}
```

The integrated barcode scanner is particularly valuable — you no longer need a separate library for QR code scanning.

### `expo-notifications`

Push notifications with Expo have always been one of its strongest selling points. The current API handles the full lifecycle:

```typescript
import * as Notifications from "expo-notifications";
import * as Device from "expo-device";
import Constants from "expo-constants";

// Configure notification handling
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

async function registerForPushNotifications() {
  if (!Device.isDevice) {
    console.warn("Push notifications only work on physical devices");
    return null;
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== "granted") {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== "granted") {
    return null;
  }

  // Get the Expo push token
  const token = await Notifications.getExpoPushTokenAsync({
    projectId: Constants.expoConfig?.extra?.eas?.projectId,
  });

  // Android requires a notification channel
  if (Platform.OS === "android") {
    await Notifications.setNotificationChannelAsync("default", {
      name: "Default",
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: "#FF231F7C",
    });
  }

  return token.data;
}

// Listen for notifications
function useNotificationListeners() {
  useEffect(() => {
    // When a notification is received while app is foregrounded
    const foregroundSub = Notifications.addNotificationReceivedListener(
      (notification) => {
        console.log("Foreground notification:", notification);
      }
    );

    // When user taps on a notification
    const responseSub = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const data = response.notification.request.content.data;
        // Navigate based on notification data
        if (data.screen) {
          router.push(data.screen as string);
        }
      }
    );

    return () => {
      foregroundSub.remove();
      responseSub.remove();
    };
  }, []);
}
```

### `expo-dev-client`

This one is easy to overlook but it's transformative for the development experience. `expo-dev-client` gives you a custom development build of your app — think of it as your own personal Expo Go, but with your specific native modules included.

```bash
# Install
npx expo install expo-dev-client

# Create a development build
npx expo run:ios
# or
eas build --profile development --platform ios
```

Why it matters:

- **Expo Go** is great for getting started, but it only includes Expo SDK modules. If you install `react-native-maps` or any other library with native code, Expo Go can't run it.
- **Development builds** include all your native dependencies. They connect to your local Metro bundler just like Expo Go, but they support everything your production app supports.
- **Faster iteration** — You build the native binary once (or rarely), then iterate on JavaScript instantly via hot reload. Native code changes are infrequent; JS changes are constant.

The typical workflow:

```
Day 1: Create development build (5-10 min)
Day 2-30: Iterate on JS code with hot reload (instant)
Day 31: Add a new native library, rebuild dev client (5-10 min)
Day 32-60: Continue iterating on JS (instant)
```

Compare this to the bare React Native workflow where every native dependency change requires rebuilding, which can take 5-15 minutes each time.

### Other Notable SDK 54 Features

| Feature | What it does | Why it matters |
|---------|-------------|----------------|
| `expo-sqlite` | Embedded SQLite database | Local-first apps, offline storage, replaces AsyncStorage for complex data |
| `expo-file-system/next` | Modern file system API | Streaming, random access, better performance than the legacy API |
| `expo-video` | Native video player | Replaces `expo-av` for video, better performance and API |
| `expo-web-browser` | In-app browser | OAuth flows, opening links without leaving your app |
| `expo-haptics` | Haptic feedback | Native-feeling interactions |
| `expo-blur` | Blur views | iOS-native blur effects, performant |
| `expo-linear-gradient` | Gradient views | No more `react-native-linear-gradient` dependency |
| React Native 0.77+ | Latest RN | New Architecture stable, improved performance |

---

## 5.7 Managed vs. Bare: A Distinction That's Almost Gone

### The Historical Split

Historically, Expo had two workflows:

- **Managed workflow** — You use Expo's tools exclusively. No `ios/` or `android/` directories. Expo handles all native configuration.
- **Bare workflow** — You "eject" from Expo and maintain native code yourself. You can still use Expo libraries, but you're responsible for native project maintenance.

This was a real, meaningful distinction in 2020. In 2026, it's almost irrelevant. Here's why.

### What Changed

CNG collapsed the distinction. Consider:

- **Managed apps can customize native code** — through config plugins. You don't need to maintain `ios/` and `android/` directories to add a permission, modify a build setting, or even add a native dependency.
- **Managed apps can use any native library** — through Expo Modules API and `expo-dev-client`. The "Expo doesn't support library X" problem is gone.
- **Bare apps still use Expo tooling** — even if you maintain native directories, you still use `expo-cli`, `expo-router`, `expo-updates`, and EAS. The tooling works in both modes.

The spectrum now looks like this:

```
Full CNG (recommended)          CNG + native overrides           Full bare
──────────────────────────────────────────────────────────────────────────
No ios/android dirs             ios/android exist but are         ios/android are
Config plugins only             partially generated               fully manual
Gitignore native dirs           Some manual native code           Everything in git
                                Config plugins + manual changes   
                                                                  
95% of apps ─────────────────►  4% of apps ────────────────────►  1% of apps
```

### When You're "Bare" Now

The only time you truly need a bare workflow is when:

1. **Brownfield integration** — You're adding React Native to an existing native iOS or Android app. CNG assumes it's building the entire native project; it can't generate just a piece of one.

2. **Custom native build systems** — Your company uses a custom build pipeline that doesn't work with EAS or standard Xcode/Gradle builds.

3. **Extremely specialized native code** — You have native engineers who need to work in Xcode/Android Studio daily, writing significant amounts of Swift/Kotlin that can't be expressed as Expo modules.

4. **Legacy codebases** — You started with bare React Native years ago and the migration cost to CNG isn't justified (though I'd argue it almost always is).

### The Migration Path

If you're on a bare workflow and want to move to CNG, here's the process:

```bash
# 1. Ensure your app.config.ts captures all your native configuration
# 2. Convert manual native changes to config plugins
# 3. Run prebuild and compare
npx expo prebuild --clean

# 4. Diff the generated native dirs against your existing ones
diff -r ios/ ios-generated/
diff -r android/ android-generated/

# 5. For each difference, either:
#    a. Add the configuration to app.config.ts
#    b. Write a config plugin
#    c. Accept that this specific change needs manual maintenance

# 6. Once the generated dirs match your requirements:
#    a. Delete ios/ and android/
#    b. Add them to .gitignore
#    c. Run prebuild to regenerate
```

The key insight: **you don't have to migrate all at once.** You can keep your native directories in git while gradually moving configuration into `app.config.ts` and config plugins. When the generated output matches what you need, you flip the switch.

### My Recommendation

Unless you're in one of the four situations listed above, use CNG. Full stop. The benefits are too significant to ignore:

- No more merge conflicts in native files
- Reproducible builds
- Trivial React Native upgrades
- Declarative configuration
- Config plugins as the extension mechanism

If someone on your team says "but we might need to customize native code someday," the answer is: "Yes, and when that day comes, we'll write a config plugin or an Expo module. We don't need to maintain an entire native project to prepare for that possibility."

---

## 5.8 When NOT to Use Expo

I've spent most of this chapter singing Expo's praises, so let me be honest about its limitations. Being a 100x architect means knowing when a tool is wrong for the job.

### Brownfield Apps

If you're adding React Native to an existing native app — say, building a new feature in RN within a large Swift or Kotlin codebase — Expo's CNG model doesn't fit. CNG generates the entire native project from scratch. It can't generate a React Native module that plugs into your existing native app.

For brownfield scenarios, you're looking at:
- Raw React Native integration (using the community's integration guide)
- `react-native-host` for more structured integration
- Custom native modules that bridge your existing native code to RN

This is genuinely a case where bare React Native, without Expo, might be the right choice. Though even here, you can use many `expo-*` packages as standalone libraries.

### Highly Custom Native Build Pipelines

Some large companies have custom build systems. They don't use Xcode directly; they have internal tools that compile iOS apps. They don't use Gradle; they have proprietary Android build toolchains.

Expo and EAS assume standard build tools. If your build pipeline is non-standard, you'll fight the tooling constantly. In this case, bare React Native with your custom build integration is the way to go.

### Teams with Heavy Native Investment

If your team has five iOS engineers and five Android engineers who live in Xcode and Android Studio all day, and they're building complex custom UI components in SwiftUI and Jetpack Compose, forcing them through Expo's abstraction layer might be counterproductive.

This isn't about Expo being incapable — it's about team dynamics. Engineers who are deeply skilled in native development may find config plugins and the Expo Modules API to be an unnecessary indirection layer that slows them down.

That said, I've seen this objection evaporate when native engineers actually try the Expo Modules API. Writing Swift instead of Objective-C++ for native modules is a significant quality-of-life improvement.

### Real-Time Audio/Video Processing

If your app's core feature is real-time audio processing, video editing with custom filters, or AR experiences, you'll likely need direct access to native APIs at a level that's easier to achieve outside Expo's module system. Libraries like `react-native-vision-camera` work with Expo, but if you're building on top of them with custom frame processors, you might want more direct native control.

### The Decision Matrix

| Factor | Favor Expo | Favor Bare RN |
|--------|-----------|---------------|
| New greenfield app | Strong | - |
| Team is primarily JS/TS | Strong | - |
| Need rapid iteration | Strong | - |
| Standard mobile app features | Strong | - |
| Adding RN to existing native app | - | Strong |
| Custom build pipeline | - | Strong |
| Team is primarily native engineers | - | Moderate |
| Extreme native customization | - | Moderate |
| Standard native customization | Moderate (config plugins) | - |
| CI/CD simplicity | Strong (EAS) | - |
| OTA updates needed | Strong (EAS Update) | - |
| Multi-platform (iOS + Android + Web) | Strong | - |

### The Honest Truth

For about 95% of React Native projects, Expo is the right choice. The 5% where it isn't are real, but they're edge cases. If you're reading this book and building a new mobile app with a JavaScript-focused team, use Expo. The question isn't "should we use Expo?" It's "do we have a specific, technical reason NOT to use Expo?"

---

## 5.9 Setting Up a New Expo Project: The Right Way

Let me give you the concrete steps to start a project correctly. Not the tutorial version — the "I'm shipping this to the App Store" version.

### Step 1: Create the Project

```bash
# Use the latest template with expo-router
npx create-expo-app@latest my-app --template tabs

# Or for a blank canvas
npx create-expo-app@latest my-app --template blank-typescript
```

### Step 2: Set Up Your Config

Replace `app.json` with `app.config.ts`. Use the production-ready template from section 5.4 as your starting point.

### Step 3: Configure Environment Files

```bash
# .env — shared, committed
EXPO_PUBLIC_API_URL=http://localhost:3000

# .env.local — local overrides, gitignored
SENTRY_DSN=your-dsn-here

# .env.production — production values, committed
EXPO_PUBLIC_API_URL=https://api.myapp.com

# .env.staging — staging values, committed  
EXPO_PUBLIC_API_URL=https://api-staging.myapp.com
```

### Step 4: Set Up EAS

```bash
npx eas-cli login
npx eas init
```

Configure `eas.json`:

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
      "env": {
        "APP_ENV": "development"
      }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging"
      },
      "channel": "staging"
    },
    "production": {
      "autoIncrement": true,
      "env": {
        "APP_ENV": "production"
      },
      "channel": "production"
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "your@apple.id",
        "ascAppId": "your-app-store-connect-id",
        "appleTeamId": "YOUR_TEAM_ID"
      },
      "android": {
        "serviceAccountKeyPath": "./config/play-store-key.json",
        "track": "internal"
      }
    }
  }
}
```

### Step 5: Gitignore the Right Things

```gitignore
# Native directories (CNG - they're generated)
/ios
/android

# Dependencies
node_modules/

# Expo
.expo/
dist/
web-build/

# Environment
.env.local
.env.*.local

# Build artifacts
*.jks
*.p8
*.p12
*.key
*.mobileprovision
*.orig.*

# macOS
.DS_Store

# IDE
.vscode/
.idea/

# Secrets (NEVER commit these)
config/google-services-*.json
config/play-store-key.json
```

### Step 6: Create Your First Dev Build

```bash
# Local build (requires Xcode / Android Studio)
npx expo run:ios
npx expo run:android

# Or cloud build via EAS (no local native toolchain needed)
eas build --profile development --platform ios
eas build --profile development --platform android
```

### Step 7: Start Developing

```bash
npx expo start
```

Scan the QR code with your development build. You're now iterating with hot reload on a fully native app.

---

## 5.10 Common Pitfalls and How to Avoid Them

### Pitfall 1: Editing Generated Native Files Directly

**Symptom:** You edit a file in `ios/` or `android/`, it works, then the next `prebuild --clean` wipes your changes.

**Fix:** Every native modification should go through `app.config.ts` or a config plugin. If you find yourself in Xcode changing a build setting, stop and write a plugin instead.

### Pitfall 2: Not Understanding the Prebuild Cache

**Symptom:** You changed your app config, but the native project doesn't reflect the changes.

**Fix:** Use `--clean` flag: `npx expo prebuild --clean`. Without it, prebuild tries to incrementally update, which can miss things.

### Pitfall 3: Bloated Bundles from Unused Expo Modules

**Symptom:** Your app binary is larger than expected.

**Fix:** Expo uses tree-shaking and autolinking. Only modules you `import` are included in the JS bundle. Only modules with native code in your `node_modules` are linked into the native binary. If you install `expo-camera` but never import it, the JS bundle won't include it — but the native code will still be linked. Remove unused packages from `package.json`.

### Pitfall 4: Environment Variable Confusion

**Symptom:** Your environment variables are undefined at runtime.

**Fix:** Remember the rules:
- Client-side variables MUST be prefixed with `EXPO_PUBLIC_`
- Changes to env vars require restarting Metro
- EAS Build uses env vars from `eas.json` profiles, not your local `.env` files

### Pitfall 5: Config Plugin Ordering Issues

**Symptom:** A config plugin doesn't seem to work, but it works when you move it in the plugins array.

**Fix:** Plugins execute in order. If plugin B depends on changes made by plugin A, A must come first. Debug by running `npx expo prebuild --clean` and inspecting the generated native files.

### Pitfall 6: Version Mismatches

**Symptom:** Cryptic build errors after updating one package.

**Fix:** Always use `npx expo install` instead of `npm install` for Expo-related packages. It ensures version compatibility:

```bash
# Right way — installs the version compatible with your SDK
npx expo install expo-camera expo-location expo-notifications

# Wrong way — might install an incompatible version
npm install expo-camera
```

### Pitfall 7: Forgetting `expo-dev-client` for Custom Native Modules

**Symptom:** You install a library with native code and it crashes in Expo Go.

**Fix:** Expo Go only includes Expo SDK native modules. For anything else, you need a development build:

```bash
npx expo install expo-dev-client
eas build --profile development --platform ios
```

---

## 5.11 Architecture Patterns with Expo

### Monorepo Setup

For larger projects, a monorepo with Expo works beautifully:

```
my-monorepo/
  apps/
    mobile/              # Expo app
      app.config.ts
      package.json
    web/                 # Next.js app (if separate from Expo web)
      package.json
  packages/
    ui/                  # Shared UI components
      package.json
    api-client/          # Shared API client
      package.json
    utils/               # Shared utilities
      package.json
    config/              # Shared configuration
      package.json
  package.json           # Workspace root
  turbo.json             # Turborepo config (if using)
```

Expo has first-class monorepo support. You just need to configure Metro to resolve packages from the workspace root:

```javascript
// apps/mobile/metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const path = require("path");

const projectRoot = __dirname;
const monorepoRoot = path.resolve(projectRoot, "../..");

const config = getDefaultConfig(projectRoot);

// Watch all files in the monorepo
config.watchFolders = [monorepoRoot];

// Resolve packages from the monorepo root
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, "node_modules"),
  path.resolve(monorepoRoot, "node_modules"),
];

module.exports = config;
```

### Feature-Based Architecture

Within an Expo app, I recommend organizing by feature, not by type:

```
app/                         # Expo Router routes
  (tabs)/
    _layout.tsx
    home.tsx
    search.tsx
    profile.tsx
  (auth)/
    _layout.tsx
    sign-in.tsx
    sign-up.tsx
    forgot-password.tsx
features/
  auth/
    api/
      useSignIn.ts
      useSignUp.ts
    components/
      SignInForm.tsx
      SocialButtons.tsx
    hooks/
      useSession.ts
    stores/
      authStore.ts
  products/
    api/
      useProducts.ts
      useProductDetail.ts
    components/
      ProductCard.tsx
      ProductList.tsx
    hooks/
      useCart.ts
  notifications/
    api/
      useNotifications.ts
    components/
      NotificationBell.tsx
    hooks/
      usePushToken.ts
    services/
      notificationService.ts
shared/
  components/
    Button.tsx
    Input.tsx
    Avatar.tsx
  hooks/
    useDebounce.ts
    useKeyboard.ts
  utils/
    formatting.ts
    validation.ts
  constants/
    colors.ts
    spacing.ts
```

The `app/` directory is for routing only — thin screens that compose features. The `features/` directory is where the business logic lives. This separation means your routing layer and your feature logic evolve independently.

### Testing Strategy

```
__tests__/
  e2e/                    # Detox or Maestro tests
    flows/
      onboarding.test.ts
      checkout.test.ts
  integration/             # Testing features together
    auth-flow.test.tsx
features/
  auth/
    __tests__/
      useSignIn.test.ts    # Unit tests colocated with feature
      SignInForm.test.tsx
```

For Expo specifically:
- **Unit tests** — Jest with `jest-expo` preset
- **Component tests** — React Native Testing Library
- **E2E tests** — Maestro (recommended) or Detox
- **Visual regression** — Storybook with `@storybook/react-native`

```json
// package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:e2e": "maestro test .maestro/",
    "test:e2e:ios": "maestro test .maestro/ --platform ios",
    "test:e2e:android": "maestro test .maestro/ --platform android"
  },
  "jest": {
    "preset": "jest-expo",
    "transformIgnorePatterns": [
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|native-base|react-native-svg)"
    ]
  }
}
```

---

## 5.12 The Expo Ecosystem at a Glance

Here's a quick reference of the major pieces and how they fit together:

| Component | What it does | Chapter coverage |
|-----------|-------------|-----------------|
| **Expo CLI** | Development server, prebuild, run commands | This chapter |
| **Expo SDK** | First-party native modules (camera, location, etc.) | This chapter |
| **Expo Router** | File-based navigation | Ch 7 |
| **EAS Build** | Cloud native builds | Ch 6 |
| **EAS Submit** | App Store / Play Store submission | Ch 6 |
| **EAS Update** | Over-the-air JavaScript updates | Ch 6 |
| **EAS Metadata** | App Store metadata management | Ch 6 |
| **Expo Modules API** | Building custom native modules | This chapter |
| **Config Plugins** | Declarative native configuration | This chapter |
| **expo-dev-client** | Custom development builds | This chapter |
| **expo-updates** | OTA update runtime | Ch 6 |

---

## Looking Ahead: Chapter 6 — EAS (Expo Application Services)

You now understand Expo as a development platform: CNG, config plugins, the app config, the Modules API, and the SDK. But building and running locally is only half the story. Getting your app to users — building production binaries, submitting to app stores, pushing over-the-air updates, managing environments — that's where EAS comes in.

Chapter 6 covers:
- **EAS Build** — Cloud builds without maintaining native toolchains locally
- **EAS Submit** — Automated App Store and Play Store submissions
- **EAS Update** — Over-the-air updates that let you ship bug fixes without app store review
- **EAS Metadata** — Managing app store listings as code
- **Build profiles** — Development, staging, preview, and production build configurations
- **The update/build versioning dance** — How OTA updates relate to native builds and when you need a new binary vs. a JS update

The combination of Expo (this chapter) and EAS (next chapter) gives you a complete mobile development and delivery platform. The pieces are designed to work together, and understanding both is what separates someone who can build a React Native app from someone who can ship and maintain one at scale.

---

## Chapter Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Expo's evolution** | From "training wheels" to "the platform." The default for serious RN projects. |
| **CNG** | Native directories are generated artifacts, not source code. This is the core innovation. |
| **Config plugins** | Declarative native customization. Write TypeScript, not XML. |
| **App config** | Use `app.config.ts`, not `app.json`. Dynamic, typed, environment-aware. |
| **Expo Modules API** | Modern native modules in Swift/Kotlin. Low barrier, high capability. |
| **SDK 54** | React Compiler (83% adoption), expo-image, expo-router v4, stable New Architecture. |
| **Managed vs. bare** | The distinction is nearly gone. CNG + config plugins give you native customization without bare-workflow maintenance. |
| **When NOT to use Expo** | Brownfield apps, custom build systems, teams with heavy native investment. ~5% of projects. |

The mental model to take away: **Expo is to React Native what Next.js is to React.** It's the framework layer that provides the opinions, tooling, and infrastructure so you can focus on your product. And just like you wouldn't start a new React web app with a raw webpack config in 2026, you shouldn't start a new React Native app without Expo unless you have a very specific reason.

Now let's go ship this thing. On to EAS.
