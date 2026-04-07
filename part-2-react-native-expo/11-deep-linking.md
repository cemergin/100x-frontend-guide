<!--
  CHAPTER: 11
  TITLE: Deep Linking & Universal Links
  PART: II — React Native & Expo
  PREREQS: Chapters 6, 8
  KEY_TOPICS: Universal Links, App Links, deferred deep links, Expo Router deep linking, URL schemes, QR codes, attribution, branch.io, app.config.ts linking config, AASA file, Digital Asset Links
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 11: Deep Linking & Universal Links

> **Part II — React Native & Expo** | Prerequisites: Chapters 6, 8 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Deep linking = a URL that opens a specific screen in your app, not just the home screen. Three flavors: URL schemes (`myapp://`), Universal Links (iOS, `https://`), and App Links (Android, `https://`)
- Expo Router makes deep linking almost free — every file-based route IS a URL already; configure your scheme in `app.config.ts` and you're 80% there
- Universal Links (iOS) require an Apple App Site Association (AASA) file hosted on your domain + correct entitlements; App Links (Android) require a Digital Asset Links (`assetlinks.json`) file + intent filters with `autoVerify`
- Deferred deep links handle the "user doesn't have the app installed yet" case — they go to the store, install, then navigate to the right screen on first open
- Test deep links obsessively on real devices; simulators lie, cached AASA files haunt you, and Android verification silently fails more often than you'd think

</details>

Deep linking is one of those features that sounds simple ("just open a URL in the app") and turns into a multi-platform configuration nightmare the moment you actually try to ship it. I've watched teams burn entire sprints debugging why their Universal Links work in development but break in production. I've seen Android App Links that verify on one device but not another. I've debugged deferred deep links where the attribution data vanishes into thin air because someone forgot about the clipboard API deprecation on iOS 16.

Here's the thing: deep linking isn't hard conceptually. A URL maps to a screen. That's it. What's hard is the platform-specific configuration, the edge cases, the testing matrix, and the fact that Apple and Google have different (and sometimes contradictory) requirements. This chapter gives you the complete picture — from basic URL schemes to production-grade Universal Links with deferred deep linking and attribution tracking.

### In This Chapter
- What Deep Linking Actually Is (and the Three Types)
- Expo Router Deep Linking: The Happy Path
- Universal Links (iOS): AASA, Entitlements, and Debugging
- Android App Links: Digital Asset Links and Intent Filters
- Deferred Deep Links: The "Install First" Problem
- QR Codes: Generation and Scanning
- Deep Linking from Notifications
- Testing Deep Links (Without Losing Your Mind)
- Attribution & Install Tracking

### Related Chapters
- [Ch 7: Navigation Architecture] — Expo Router and React Navigation fundamentals
- [Ch 6: EAS Mastery] — Build configuration that affects linking entitlements
- [Ch 8: Styling & Animation] — transition animations when navigating from deep links
- [Ch 14: Push Notifications] — notification-triggered deep links

---

## 1. WHAT DEEP LINKING ACTUALLY IS

At its simplest, a deep link is a URL that opens a specific screen in your mobile app instead of just launching the home screen.

Without deep linking:
- User taps a link → App opens → Home screen → User manually navigates to the content

With deep linking:
- User taps a link → App opens → Specific screen with the right content

That's the entire value proposition. But the implementation has three distinct flavors, and choosing the right one matters.

### 1.1 URL Schemes (Custom Schemes)

```
myapp://products/123
myapp://settings/profile
myapp://auth/reset?token=abc123
```

**How they work:** You register a custom URL scheme (`myapp://`) with the operating system. When any app or browser encounters that scheme, it hands it to your app.

**Pros:**
- Simple to implement
- Work on both iOS and Android
- No server-side configuration needed
- Good for development and testing

**Cons:**
- Not unique — any app can register the same scheme (scheme hijacking)
- No fallback — if the app isn't installed, the URL just fails silently
- Can't be used in web browsers (they don't know what `myapp://` means)
- Apple and Google both discourage them for production use
- No ownership verification — there's no proof that `myapp://` belongs to you

**When to use them:** Development, testing, app-to-app communication within your own app suite, or as a fallback for platforms where Universal/App Links aren't available.

```tsx
// app.config.ts — registering a URL scheme
export default {
  expo: {
    scheme: 'myapp',
    // This registers myapp:// with both iOS and Android
  },
};
```

### 1.2 Universal Links (iOS)

```
https://myapp.com/products/123
https://myapp.com/settings/profile
```

**How they work:** You host a special file (Apple App Site Association / AASA) on your domain that tells iOS "these URL patterns should open in my app." When iOS encounters a matching URL, it opens your app instead of Safari — but only if the app is installed.

**Pros:**
- Secure — only you can host files on your domain, so no other app can claim your links
- Graceful fallback — if the app isn't installed, the URL just opens in Safari (your website)
- Standard HTTPS URLs — work everywhere (emails, messages, social media, web pages)
- Apple's recommended approach for deep linking

**Cons:**
- Requires server-side configuration (AASA file)
- Requires proper app entitlements
- iOS caches the AASA file aggressively — changes can take hours to propagate
- Debugging is painful (Apple's tooling is mediocre)
- HTTPS only — no HTTP, no exceptions

**When to use them:** Production apps. Always. This should be your primary deep linking mechanism on iOS.

### 1.3 App Links (Android)

```
https://myapp.com/products/123
https://myapp.com/settings/profile
```

**How they work:** Similar to Universal Links but for Android. You host a Digital Asset Links file (`assetlinks.json`) on your domain that tells Android "these URL patterns should open in my app." When Android encounters a matching URL, it opens your app.

**Pros:**
- Secure — domain verification proves ownership
- Standard HTTPS URLs
- Google's recommended approach
- Can be auto-verified (no user disambiguation dialog)

**Cons:**
- Requires server-side configuration
- Auto-verification can silently fail
- Different OEMs handle App Links differently (Samsung, Xiaomi, etc.)
- User can manually disable app link handling in settings

**When to use them:** Production apps on Android. Same as Universal Links — this is the default.

### 1.4 The Deep Linking Decision Matrix

```
┌──────────────────────┬─────────────────┬─────────────────┬──────────────────┐
│                      │ URL Scheme      │ Universal Links │ App Links        │
│                      │ myapp://        │ (iOS, https://) │ (Android,https://)│
├──────────────────────┼─────────────────┼─────────────────┼──────────────────┤
│ Platform             │ iOS + Android   │ iOS only        │ Android only     │
│ Requires domain      │ No              │ Yes             │ Yes              │
│ Requires server file │ No              │ AASA file       │ assetlinks.json  │
│ Fallback if no app   │ Error/nothing   │ Opens website   │ Opens website    │
│ Secure (verified)    │ No              │ Yes             │ Yes              │
│ Works in browsers    │ No              │ Yes             │ Yes              │
│ Recommended for prod │ No              │ Yes             │ Yes              │
│ Setup difficulty     │ Easy            │ Medium          │ Medium           │
│ Debugging difficulty │ Easy            │ Hard            │ Medium           │
└──────────────────────┴─────────────────┴─────────────────┴──────────────────┘
```

**My recommendation:** Use all three. URL schemes for development and fallback. Universal Links for iOS production. App Links for Android production. The URL patterns should be identical across all three so your routing code doesn't branch based on how the app was opened.

---

## 2. EXPO ROUTER DEEP LINKING: THE HAPPY PATH

Here's the beautiful thing about Expo Router: **every route is already a URL**. If you have a file at `app/products/[id].tsx`, the URL `/products/123` will navigate to that screen. Deep linking is built into the architecture.

### 2.1 Basic Configuration

```ts
// app.config.ts
export default {
  expo: {
    name: 'MyApp',
    slug: 'myapp',
    scheme: 'myapp', // Registers myapp:// URL scheme
    
    // For Expo Router
    plugins: [
      'expo-router',
    ],
    
    experiments: {
      typedRoutes: true, // Type-safe route names
    },
  },
};
```

With just this configuration, you can already deep link using URL schemes:

```bash
# iOS Simulator
npx uri-scheme open "myapp://products/123" --ios

# Android Emulator
npx uri-scheme open "myapp://products/123" --android

# Or using adb directly
adb shell am start -a android.intent.action.VIEW -d "myapp://products/123"

# Or using xcrun on iOS
xcrun simctl openurl booted "myapp://products/123"
```

### 2.2 How Expo Router Resolves Deep Links

When a deep link comes in, Expo Router maps the URL path to your file structure:

```
URL: myapp://products/123?color=red

Resolves to:
  app/products/[id].tsx

With params:
  { id: '123', color: 'red' }
```

The mapping is automatic and follows these rules:

```
URL Path              → File                         → Params
/                     → app/index.tsx                → {}
/about                → app/about.tsx                → {}
/products/123         → app/products/[id].tsx        → { id: '123' }
/users/42/posts/7     → app/users/[userId]/posts/    → { userId: '42',
                        [postId].tsx                    postId: '7' }
/search?q=react       → app/search.tsx               → { q: 'react' }
```

### 2.3 Handling Deep Links in Your Components

```tsx
// app/products/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { useEffect } from 'react';
import { View, Text } from 'react-native';

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  
  useEffect(() => {
    // This runs whether the user navigated here manually
    // OR via a deep link — same code path
    console.log('Viewing product:', id);
  }, [id]);

  return (
    <View>
      <Text>Product {id}</Text>
    </View>
  );
}
```

**This is the key insight:** with Expo Router, you don't need separate "deep link handling" code. Your screen components receive params the same way regardless of whether the user navigated from within the app or via a deep link. The framework handles the routing.

### 2.4 Handling Deep Links at the Root Level

Sometimes you need to intercept deep links before they reach individual screens — for example, to check authentication or log analytics:

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import * as Linking from 'expo-linking';

export default function RootLayout() {
  useEffect(() => {
    // Handle deep links when the app is already open (warm start)
    const subscription = Linking.addEventListener('url', (event) => {
      console.log('Deep link received (warm):', event.url);
      trackDeepLink(event.url);
    });

    // Handle the deep link that opened the app (cold start)
    Linking.getInitialURL().then((url) => {
      if (url) {
        console.log('Deep link received (cold):', url);
        trackDeepLink(url);
      }
    });

    return () => subscription.remove();
  }, []);

  return <Stack />;
}

function trackDeepLink(url: string) {
  const parsed = Linking.parse(url);
  analytics.track('deep_link_opened', {
    path: parsed.path,
    queryParams: parsed.queryParams,
    scheme: parsed.scheme,
  });
}
```

### 2.5 Deep Link with Authentication Guards

A common scenario: a deep link targets an authenticated screen, but the user isn't logged in.

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack, useRouter, useSegments, useRootNavigation } from 'expo-router';
import { useAuth } from '../hooks/useAuth';

export default function RootLayout() {
  const { isAuthenticated, isLoading } = useAuth();
  const segments = useSegments();
  const router = useRouter();

  useEffect(() => {
    if (isLoading) return;

    const inAuthGroup = segments[0] === '(auth)';

    if (!isAuthenticated && !inAuthGroup) {
      // User is not signed in and is trying to access a protected route
      // Store the intended destination so we can redirect after login
      const intendedRoute = '/' + segments.join('/');
      router.replace({
        pathname: '/(auth)/login',
        params: { redirect: intendedRoute },
      });
    } else if (isAuthenticated && inAuthGroup) {
      // User is signed in but on the auth screen
      router.replace('/');
    }
  }, [isAuthenticated, segments, isLoading]);

  if (isLoading) {
    return <LoadingScreen />;
  }

  return <Stack />;
}
```

```tsx
// app/(auth)/login.tsx
import { useLocalSearchParams, useRouter } from 'expo-router';

export default function LoginScreen() {
  const { redirect } = useLocalSearchParams<{ redirect?: string }>();
  const router = useRouter();

  const handleLogin = async () => {
    await performLogin();
    
    if (redirect) {
      // Navigate to the originally intended deep link destination
      router.replace(redirect as any);
    } else {
      router.replace('/');
    }
  };

  return (
    // Login UI...
  );
}
```

### 2.6 Testing Deep Links with `npx uri-scheme`

Expo provides a handy CLI tool for testing:

```bash
# List registered schemes for your app
npx uri-scheme list --ios
npx uri-scheme list --android

# Open a URL in the iOS Simulator
npx uri-scheme open "myapp://products/123" --ios

# Open a URL in the Android Emulator
npx uri-scheme open "myapp://products/123" --android

# Open a Universal Link (iOS)
npx uri-scheme open "https://myapp.com/products/123" --ios

# Open an App Link (Android)
npx uri-scheme open "https://myapp.com/products/123" --android
```

**Pro tip:** Create a `deep-link-test.sh` script that tests all your critical deep link paths:

```bash
#!/bin/bash
# deep-link-test.sh — Test all critical deep link paths

SCHEME="myapp"
DOMAIN="myapp.com"
PLATFORM=${1:-ios} # default to iOS

LINKS=(
  "$SCHEME://products/123"
  "$SCHEME://settings/profile"
  "$SCHEME://orders/456"
  "$SCHEME://search?q=react+native"
  "https://$DOMAIN/products/123"
  "https://$DOMAIN/settings/profile"
  "https://$DOMAIN/share/abc"
)

echo "Testing deep links on $PLATFORM..."
for link in "${LINKS[@]}"; do
  echo ""
  echo "Testing: $link"
  npx uri-scheme open "$link" --$PLATFORM
  echo "Press enter to test next link..."
  read
done

echo "All deep links tested!"
```

---

## 3. UNIVERSAL LINKS (iOS)

Universal Links are Apple's way of saying "this HTTPS URL should open in your app, not Safari." They're the gold standard for iOS deep linking, but the setup has a few moving parts.

### 3.1 How Universal Links Work Under the Hood

Here's the flow:

```
1. User installs your app
   └─→ iOS reads your app's entitlements
   └─→ Finds your associated domains: applinks:myapp.com
   └─→ Fetches https://myapp.com/.well-known/apple-app-site-association
   └─→ Caches the AASA file locally

2. User taps an https://myapp.com/products/123 link
   └─→ iOS checks its cached AASA file
   └─→ Finds that /products/* should open in your app
   └─→ Opens your app with the URL
   └─→ Your app receives the URL and navigates to the right screen

3. If the app is NOT installed:
   └─→ iOS just opens the URL in Safari (normal web behavior)
```

**Critical detail:** iOS fetches the AASA file at **install time** (and periodically refreshes it), NOT when the user taps a link. This means:
- Changes to your AASA file don't take effect until iOS re-fetches it
- The cache can persist for hours or even days
- You can't force a refresh — reinstalling the app is the most reliable way

### 3.2 The Apple App Site Association (AASA) File

This is a JSON file that lives at `https://yourdomain.com/.well-known/apple-app-site-association`. No file extension. No redirect. Must be served over HTTPS with a valid certificate.

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": [
          "TEAM_ID.com.yourcompany.yourapp"
        ],
        "components": [
          {
            "/": "/products/*",
            "comment": "Product pages"
          },
          {
            "/": "/users/*/profile",
            "comment": "User profiles"
          },
          {
            "/": "/share/*",
            "comment": "Share links"
          },
          {
            "/": "/orders/*",
            "comment": "Order details"
          },
          {
            "/": "/settings",
            "comment": "Settings page"
          },
          {
            "/": "/auth/reset",
            "?": { "token": "?*" },
            "comment": "Password reset with token"
          },
          {
            "/": "/api/*",
            "exclude": true,
            "comment": "Exclude API routes — these should stay in the browser"
          },
          {
            "/": "/admin/*",
            "exclude": true,
            "comment": "Exclude admin panel"
          }
        ]
      }
    ]
  }
}
```

**AASA v2 (iOS 15+) vs v1:**

The `components` array syntax is the v2 format (iOS 15+). Older versions use `paths`:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.yourcompany.yourapp",
        "paths": [
          "/products/*",
          "/users/*/profile",
          "/share/*",
          "NOT /api/*",
          "NOT /admin/*"
        ]
      }
    ]
  }
}
```

**My recommendation:** Include both `components` (v2) and `paths` (v1) in the same file for maximum compatibility:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.yourcompany.yourapp",
        "paths": [
          "/products/*",
          "/users/*/profile",
          "/share/*",
          "NOT /api/*"
        ],
        "appIDs": [
          "TEAM_ID.com.yourcompany.yourapp"
        ],
        "components": [
          { "/": "/products/*" },
          { "/": "/users/*/profile" },
          { "/": "/share/*" },
          { "/": "/api/*", "exclude": true }
        ]
      }
    ]
  }
}
```

### 3.3 Finding Your Team ID and App ID

Your `appID` has the format `TEAM_ID.BUNDLE_ID`:

```bash
# Find your Team ID:
# 1. Apple Developer Portal → Membership → Team ID
# 2. Or from Xcode: Signing & Capabilities → Team

# Find your Bundle ID:
# In app.config.ts: expo.ios.bundleIdentifier
# Or in Xcode: General → Bundle Identifier

# Example:
# Team ID: A1B2C3D4E5
# Bundle ID: com.mycompany.myapp
# App ID: A1B2C3D4E5.com.mycompany.myapp
```

### 3.4 Hosting the AASA File

The AASA file MUST be:
- Served at `https://yourdomain.com/.well-known/apple-app-site-association`
- Served over HTTPS (not HTTP)
- Served with `Content-Type: application/json`
- Not behind a redirect (no 301/302)
- Not behind authentication
- Accessible from Apple's CDN (Apple fetches it from their servers, not from the user's device)

**With Vercel/Next.js:**

```
public/.well-known/apple-app-site-association
```

```ts
// next.config.js — ensure proper content type
const nextConfig = {
  async headers() {
    return [
      {
        source: '/.well-known/apple-app-site-association',
        headers: [
          {
            key: 'Content-Type',
            value: 'application/json',
          },
        ],
      },
    ];
  },
};
```

**With Express:**

```ts
app.get('/.well-known/apple-app-site-association', (req, res) => {
  res.set('Content-Type', 'application/json');
  res.sendFile(path.join(__dirname, 'apple-app-site-association.json'));
});
```

**With Cloudflare Pages:**

```
public/.well-known/apple-app-site-association
```

Add a `_headers` file:

```
/.well-known/apple-app-site-association
  Content-Type: application/json
```

### 3.5 Expo Configuration for Universal Links

```ts
// app.config.ts
export default {
  expo: {
    name: 'MyApp',
    scheme: 'myapp',
    
    ios: {
      bundleIdentifier: 'com.mycompany.myapp',
      associatedDomains: [
        'applinks:myapp.com',
        'applinks:*.myapp.com', // Wildcard subdomain support
      ],
    },
    
    plugins: [
      'expo-router',
      // If you need more control over associated domains:
      [
        'expo-apple-authentication', // or any plugin that modifies entitlements
        {
          // Additional entitlements if needed
        },
      ],
    ],
  },
};
```

**What this does behind the scenes:**
1. Adds the `com.apple.developer.associated-domains` entitlement to your iOS app
2. Lists `applinks:myapp.com` as an associated domain
3. When the app is installed, iOS fetches `https://myapp.com/.well-known/apple-app-site-association`

### 3.6 Expo Config Plugin for Custom Entitlements

If you need more control over the entitlements (for example, adding webcredentials or activitycontinuation):

```ts
// plugins/withAssociatedDomains.ts
import { ConfigPlugin, withEntitlementsPlist } from 'expo/config-plugins';

const withAssociatedDomains: ConfigPlugin<string[]> = (config, domains) => {
  return withEntitlementsPlist(config, (mod) => {
    mod.modResults['com.apple.developer.associated-domains'] = domains;
    return mod;
  });
};

export default withAssociatedDomains;
```

```ts
// app.config.ts
export default {
  expo: {
    plugins: [
      [
        './plugins/withAssociatedDomains',
        [
          'applinks:myapp.com',
          'applinks:staging.myapp.com',
          'webcredentials:myapp.com', // For password autofill
          'activitycontinuation:myapp.com', // For Handoff
        ],
      ],
    ],
  },
};
```

### 3.7 Debugging Universal Links

This is where things get painful. Here's your debugging toolkit:

**1. Apple's AASA Validator:**

```
https://app-site-association.cdn-apple.com/a/v1/yourdomain.com
```

This is the URL that Apple's CDN uses to fetch and cache your AASA file. Visit this URL to see what Apple sees. If it's outdated, Apple hasn't re-fetched your file yet.

**2. Apple's developer tool:**

```
https://search.developer.apple.com/appsearch-validation-tool/
```

Enter your domain to validate the AASA file format and content.

**3. Console.app on macOS:**

```
1. Open Console.app on your Mac
2. Connect your iPhone or select Simulator
3. Filter for "swcd" (the process that handles associated domains)
4. Trigger a Universal Link
5. Look for errors like:
   - "Failed to fetch AASA file"
   - "No matching paths"
   - "App ID mismatch"
```

**4. `sysdiagnose` on device:**

For deep debugging, you can capture a sysdiagnose that includes associated domain logs:

```bash
# On device: Settings → Privacy → Analytics → Analytics Data
# Look for entries starting with "swcutil"
```

**5. Direct AASA fetch test:**

```bash
# Verify your AASA file is accessible
curl -v https://myapp.com/.well-known/apple-app-site-association

# Check it returns valid JSON
curl -s https://myapp.com/.well-known/apple-app-site-association | jq .

# Verify Apple's CDN copy
curl -s https://app-site-association.cdn-apple.com/a/v1/myapp.com | jq .
```

**6. Developer Mode (iOS 16+):**

In iOS 16+, you can enable developer mode to bypass some caching:

```
Settings → Developer → Associated Domains Development
→ Enable
```

This makes iOS re-fetch the AASA file on every app launch instead of caching it. **Only use this during development.**

### 3.8 Common Universal Link Gotchas

**Gotcha 1: Safari won't trigger Universal Links from the same domain.**

If the user is on `https://myapp.com/page1` and taps a link to `https://myapp.com/products/123`, Safari will NOT open the app. It stays in Safari. This is by design — Apple assumes if you're already on the website, you want to stay there.

**Workaround:** Use a different domain for deep links (`https://links.myapp.com/products/123`) or use JavaScript to detect the app and redirect:

```html
<script>
  // Detect if we should redirect to the app
  const userAgent = navigator.userAgent;
  const isIOS = /iPad|iPhone|iPod/.test(userAgent);
  
  if (isIOS) {
    // Try the Universal Link first (will be caught by iOS if app is installed)
    // Then fall back to App Store after a delay
    window.location = 'https://links.myapp.com/products/123';
    setTimeout(() => {
      window.location = 'https://apps.apple.com/app/id123456789';
    }, 500);
  }
</script>
```

**Gotcha 2: The AASA file is cached by Apple's CDN.**

Apple fetches your AASA through their own CDN (`app-site-association.cdn-apple.com`). Even if you update your file, Apple's CDN might serve the old version for hours. There's no way to force a purge.

**Gotcha 3: Universal Links require a user gesture.**

You can't programmatically trigger a Universal Link. The user must tap something — a link, a button, a notification. `window.location = 'https://...'` from JavaScript won't trigger the app to open.

**Gotcha 4: Long press shows the "Open in App" banner.**

If a user long-presses a Universal Link, they see a context menu with "Open in [App Name]." If they choose "Open in Safari" instead, iOS *remembers this preference* and stops opening that domain in the app. The user has to long-press again and choose "Open in [App Name]" to re-enable it.

**Gotcha 5: Redirect chains break Universal Links.**

If your link goes through redirects (`short.link/abc` → `myapp.com/products/123`), the Universal Link might not trigger because iOS checks the *initial* URL, not the final destination.

---

## 4. ANDROID APP LINKS

Android App Links are Google's equivalent of Universal Links. Same concept, different implementation.

### 4.1 How App Links Work on Android

```
1. User installs your app
   └─→ Android reads your app's intent filters
   └─→ Finds autoVerify="true"
   └─→ Fetches https://myapp.com/.well-known/assetlinks.json
   └─→ Verifies the certificate fingerprint matches your app

2. User taps an https://myapp.com/products/123 link
   └─→ Android checks verified domains
   └─→ Opens your app (no disambiguation dialog)

3. If verification failed or app not installed:
   └─→ Opens in the browser (normal behavior)
```

### 4.2 The Digital Asset Links File

This file lives at `https://yourdomain.com/.well-known/assetlinks.json`:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": [
        "14:6D:E9:83:C5:73:06:50:D8:EE:B9:95:2F:34:FC:64:16:A0:83:42:E6:1D:BE:A8:8A:04:96:B2:3F:CF:44:E5"
      ]
    }
  }
]
```

### 4.3 Getting Your SHA-256 Certificate Fingerprint

This is the part that trips up most teams. You need the fingerprint of the signing certificate used to build your APK/AAB.

**For development (debug keystore):**

```bash
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android

# Look for:
# SHA256: 14:6D:E9:83:C5:73:06:50:...
```

**For production (EAS Build):**

```bash
# If using EAS Build's managed signing:
eas credentials --platform android

# Select your project → Look for SHA-256 fingerprint
# Or download the keystore and use keytool:
eas credentials --platform android
# Choose "Download keystore"

keytool -list -v -keystore downloaded-keystore.jks -alias key-alias
```

**For Google Play App Signing:**

If you use Google Play App Signing (you should), Google re-signs your app with their own key. You need BOTH fingerprints in your `assetlinks.json`:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": [
        "YOUR_UPLOAD_KEY_FINGERPRINT",
        "GOOGLE_PLAY_SIGNING_KEY_FINGERPRINT"
      ]
    }
  }
]
```

Find Google's signing key fingerprint in the Play Console:
```
Play Console → Your App → Setup → App Signing → App signing key certificate
→ SHA-256 certificate fingerprint
```

### 4.4 Expo Configuration for App Links

```ts
// app.config.ts
export default {
  expo: {
    android: {
      package: 'com.mycompany.myapp',
      intentFilters: [
        {
          action: 'VIEW',
          autoVerify: true, // This is critical — enables auto-verification
          data: [
            {
              scheme: 'https',
              host: 'myapp.com',
              pathPrefix: '/products',
            },
            {
              scheme: 'https',
              host: 'myapp.com',
              pathPrefix: '/users',
            },
            {
              scheme: 'https',
              host: 'myapp.com',
              pathPrefix: '/share',
            },
            {
              scheme: 'https',
              host: 'myapp.com',
              pathPrefix: '/orders',
            },
          ],
          category: ['BROWSABLE', 'DEFAULT'],
        },
      ],
    },
  },
};
```

### 4.5 Intent Filter Patterns

Android intent filters support several matching patterns:

```ts
// Exact path
{ scheme: 'https', host: 'myapp.com', path: '/about' }
// Matches: https://myapp.com/about
// Doesn't match: https://myapp.com/about/us

// Path prefix
{ scheme: 'https', host: 'myapp.com', pathPrefix: '/products' }
// Matches: https://myapp.com/products, https://myapp.com/products/123
// Matches: https://myapp.com/products/anything/here

// Path pattern (with wildcards)
{ scheme: 'https', host: 'myapp.com', pathPattern: '/users/.*/profile' }
// Matches: https://myapp.com/users/123/profile
// Note: .* in pathPattern matches any characters

// Host wildcard
{ scheme: 'https', host: '*.myapp.com', pathPrefix: '/' }
// Matches any subdomain of myapp.com
```

### 4.6 Verifying App Links

```bash
# Check if your assetlinks.json is accessible
curl -v https://myapp.com/.well-known/assetlinks.json

# Use Google's verification tool
# Visit: https://developers.google.com/digital-asset-links/tools/generator

# Check verification status on device (requires adb)
adb shell pm get-app-links com.mycompany.myapp

# Expected output:
# com.mycompany.myapp:
#   ID: ...
#   Signatures: [...]
#   Domains:
#     myapp.com:
#       Status: verified  ← This is what you want
```

**Force re-verification:**

```bash
# Clear existing verification state
adb shell pm set-app-links --package com.mycompany.myapp 0 all

# Trigger re-verification
adb shell pm verify-app-links --re-verify com.mycompany.myapp

# Check status again
adb shell pm get-app-links com.mycompany.myapp
```

### 4.7 Common Android App Link Gotchas

**Gotcha 1: `autoVerify` must be `true`.**

Without `autoVerify: true`, Android will show a disambiguation dialog ("Open with: Browser or YourApp?"). Users hate this. Always set `autoVerify: true`.

**Gotcha 2: ALL domains in intent filters must verify.**

If you have intent filters for `myapp.com` AND `staging.myapp.com`, BOTH domains must have valid `assetlinks.json` files. If even one fails verification, ALL App Links for your app may fail on some Android versions.

**Gotcha 3: Google Play App Signing changes your certificate.**

If you're using Google Play App Signing (the default for new apps), the certificate fingerprint you need in `assetlinks.json` is Google's signing key, NOT your upload key. Include both to be safe.

**Gotcha 4: Some OEMs break App Links.**

Samsung, Xiaomi, and other manufacturers sometimes have custom link handling that interferes with standard App Links. Always test on real devices from major manufacturers.

**Gotcha 5: Users can disable App Links.**

Users can go to Settings → Apps → Your App → Open by default → and disable link handling. There's nothing you can do about this except provide a good experience that makes users want to keep it enabled.

---

## 5. DEFERRED DEEP LINKS

This is where deep linking gets really interesting (and really complicated). Deferred deep links solve the "user doesn't have the app installed" problem.

### 5.1 The Problem

Normal deep link flow when app IS installed:
```
User taps link → App opens → Right screen ✓
```

Normal deep link flow when app is NOT installed:
```
User taps link → Browser opens → User sees your website → User goes to App Store →
User installs app → User opens app → HOME SCREEN ✗
```

The user loses context. They clicked a link to a specific product, installed the app because of that link, and then... they're staring at the home screen with no idea where the product is.

### 5.2 How Deferred Deep Links Work

```
User taps link → Browser opens → Link service captures URL + device fingerprint →
User goes to App Store → User installs app → User opens app →
App checks link service → "Hey, this device clicked /products/123" →
App navigates to /products/123 ✓
```

The "deferred" part means the deep link is deferred until after installation. The link service remembers what the user clicked, and the app retrieves that information on first launch.

### 5.3 Implementation Options

**Option 1: Branch.io (Most Popular)**

Branch is the industry standard for deferred deep links. It handles the complexity of device fingerprinting, attribution, and cross-platform linking.

```bash
npx expo install react-native-branch
```

```ts
// app.config.ts
export default {
  expo: {
    plugins: [
      [
        'react-native-branch',
        {
          apiKey: 'key_live_xxxx', // Your Branch key
          iosAppDomain: 'myapp.app.link', // Branch-provided domain
        },
      ],
    ],
  },
};
```

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import branch from 'react-native-branch';
import { useRouter } from 'expo-router';

export default function RootLayout() {
  const router = useRouter();

  useEffect(() => {
    // Subscribe to Branch deep links
    const unsubscribe = branch.subscribe(({ error, params }) => {
      if (error) {
        console.error('Branch error:', error);
        return;
      }

      if (!params) return;

      // Check if this is a deferred deep link (first open after install)
      if (params['+is_first_session']) {
        console.log('First session — deferred deep link');
      }

      // Navigate based on the deep link data
      if (params.screen === 'product' && params.productId) {
        router.push(`/products/${params.productId}`);
      } else if (params.screen === 'profile' && params.userId) {
        router.push(`/users/${params.userId}/profile`);
      } else if (params['$canonical_url']) {
        // Parse the canonical URL and navigate
        const url = new URL(params['$canonical_url']);
        router.push(url.pathname);
      }
    });

    return () => unsubscribe();
  }, []);

  return <Stack />;
}
```

Creating a Branch link:

```tsx
import branch from 'react-native-branch';

async function createShareLink(productId: string, productName: string) {
  const branchUniversalObject = await branch.createBranchUniversalObject(
    `product/${productId}`,
    {
      title: productName,
      contentDescription: `Check out ${productName}`,
      contentImageUrl: `https://myapp.com/images/products/${productId}.jpg`,
      contentMetadata: {
        customMetadata: {
          screen: 'product',
          productId: productId,
        },
      },
    }
  );

  const linkProperties = {
    feature: 'share',
    channel: 'in-app',
    campaign: 'product-sharing',
  };

  const controlParams = {
    $desktop_url: `https://myapp.com/products/${productId}`,
    $ios_url: `https://myapp.com/products/${productId}`,
    $android_url: `https://myapp.com/products/${productId}`,
  };

  const { url } = await branchUniversalObject.generateShortUrl(
    linkProperties,
    controlParams
  );

  return url; // Returns something like https://myapp.app.link/abc123
}
```

**Option 2: Expo's Built-in Approach (No Third-Party Service)**

If you don't want to use a third-party service, you can build basic deferred deep linking yourself using your own backend:

```tsx
// Backend: POST /api/deferred-links
// Stores: { linkId, targetUrl, createdAt, fingerprint }

// When user clicks a link but doesn't have the app:
// 1. Your website captures the link ID and stores it in a cookie/localStorage
// 2. Website redirects to App Store
// 3. On first app launch, you check your backend

// app/hooks/useDeferredDeepLink.ts
import { useEffect } from 'react';
import { useRouter } from 'expo-router';
import * as Application from 'expo-application';
import AsyncStorage from '@react-native-async-storage/async-storage';

const DEFERRED_LINK_CHECKED_KEY = 'deferred_link_checked';

export function useDeferredDeepLink() {
  const router = useRouter();

  useEffect(() => {
    checkDeferredDeepLink();
  }, []);

  async function checkDeferredDeepLink() {
    // Only check on first launch
    const alreadyChecked = await AsyncStorage.getItem(DEFERRED_LINK_CHECKED_KEY);
    if (alreadyChecked) return;

    try {
      // Get device identifiers for fingerprinting
      const installationId = await Application.getInstallationTimeAsync();
      
      const response = await fetch('https://api.myapp.com/api/deferred-links/match', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          platform: Platform.OS,
          installTime: installationId,
          // Add any other fingerprinting data
        }),
      });

      const data = await response.json();

      if (data.targetUrl) {
        const url = new URL(data.targetUrl);
        router.push(url.pathname);
      }
    } catch (error) {
      console.error('Deferred deep link check failed:', error);
    } finally {
      await AsyncStorage.setItem(DEFERRED_LINK_CHECKED_KEY, 'true');
    }
  }
}
```

**Option 3: Adjust.com**

Adjust is another popular attribution and deep linking platform:

```bash
npx expo install react-native-adjust
```

```ts
// app.config.ts
export default {
  expo: {
    plugins: [
      [
        'react-native-adjust',
        {
          appToken: 'YOUR_APP_TOKEN',
          environment: 'production', // or 'sandbox'
        },
      ],
    ],
  },
};
```

```tsx
// app/_layout.tsx
import { Adjust, AdjustConfig, AdjustEvent } from 'react-native-adjust';
import { useEffect } from 'react';

export default function RootLayout() {
  useEffect(() => {
    const adjustConfig = new AdjustConfig(
      'YOUR_APP_TOKEN',
      AdjustConfig.EnvironmentProduction
    );

    adjustConfig.setDeferredDeeplinkCallback((uri) => {
      console.log('Deferred deep link:', uri);
      // Navigate to the deep link destination
      const url = new URL(uri);
      router.push(url.pathname);
      return true; // Return true to allow Adjust to open the link
    });

    Adjust.initSdk(adjustConfig);
  }, []);

  return <Stack />;
}
```

### 5.4 Deferred Deep Link Gotchas

**The clipboard trick is dead.** Before iOS 16, some services (like Branch) would copy data to the clipboard when the user clicked a link, then read it on first app launch. Apple killed this in iOS 16 with clipboard access prompts. Don't rely on it.

**Device fingerprinting is probabilistic.** Without a unique identifier that persists across install, deferred deep links rely on matching device characteristics (IP address, screen size, OS version, etc.). This isn't 100% accurate. Expect 80-90% match rates on iOS and slightly better on Android.

**The time window matters.** Most services only attempt to match deferred deep links within a window (24-48 hours from click to install). After that, the link data expires.

**Testing deferred deep links is painful.** You have to: uninstall the app → clear all caches → click the link in a browser → go to the store → install → open. Every time. Build a debug mode that lets you simulate this flow.

---

## 6. QR CODES

QR codes are just another way to deliver a URL. But they're increasingly important for real-world → app transitions.

### 6.1 Generating QR Codes

```bash
npx expo install react-native-qrcode-svg
# or
npx expo install react-native-qrcode-skia  # if using Skia
```

```tsx
// components/QRCodeGenerator.tsx
import React from 'react';
import { View, StyleSheet } from 'react-native';
import QRCode from 'react-native-qrcode-svg';

interface QRCodeGeneratorProps {
  url: string;
  size?: number;
  logo?: any; // require('./logo.png')
}

export function QRCodeGenerator({ url, size = 200, logo }: QRCodeGeneratorProps) {
  return (
    <View style={styles.container}>
      <QRCode
        value={url}
        size={size}
        logo={logo}
        logoSize={size * 0.2}
        logoBackgroundColor="white"
        logoBorderRadius={10}
        backgroundColor="white"
        color="black"
        // Error correction level — higher = more resilient to damage/logos
        ecl="M" // L, M, Q, H
      />
    </View>
  );
}

// Usage:
<QRCodeGenerator
  url="https://myapp.com/products/123"
  size={250}
  logo={require('../assets/logo.png')}
/>
```

### 6.2 Generating QR Codes on the Backend

For dynamic QR codes (tracking, analytics), generate them server-side:

```ts
// Backend: /api/qr-code
import QRCode from 'qrcode';

export async function generateQRCode(targetUrl: string, campaignId: string) {
  // Create a tracking URL
  const trackingUrl = `https://myapp.com/qr/${campaignId}?target=${encodeURIComponent(targetUrl)}`;

  // Generate QR code as data URL
  const dataUrl = await QRCode.toDataURL(trackingUrl, {
    width: 400,
    margin: 2,
    color: {
      dark: '#000000',
      light: '#FFFFFF',
    },
    errorCorrectionLevel: 'M',
  });

  return dataUrl;
}
```

### 6.3 Scanning QR Codes

Expo provides QR code scanning through the camera:

```bash
npx expo install expo-camera
```

```tsx
// components/QRScanner.tsx
import React, { useState, useEffect } from 'react';
import { View, Text, StyleSheet, Alert } from 'react-native';
import { CameraView, useCameraPermissions, BarcodeScanningResult } from 'expo-camera';
import { useRouter } from 'expo-router';
import * as Linking from 'expo-linking';

export function QRScanner() {
  const [permission, requestPermission] = useCameraPermissions();
  const [scanned, setScanned] = useState(false);
  const router = useRouter();

  if (!permission) {
    return <View />;
  }

  if (!permission.granted) {
    return (
      <View style={styles.container}>
        <Text style={styles.message}>Camera permission is required to scan QR codes.</Text>
        <Button title="Grant Permission" onPress={requestPermission} />
      </View>
    );
  }

  const handleBarCodeScanned = (result: BarcodeScanningResult) => {
    if (scanned) return;
    setScanned(true);

    const { data } = result;
    console.log('Scanned QR code:', data);

    // Check if the URL is one of ours
    if (data.startsWith('https://myapp.com/') || data.startsWith('myapp://')) {
      // Parse and navigate within the app
      const parsed = Linking.parse(data);
      if (parsed.path) {
        router.push(`/${parsed.path}`);
      }
    } else {
      // External URL — ask user if they want to open it
      Alert.alert(
        'External Link',
        `Open ${data}?`,
        [
          { text: 'Cancel', onPress: () => setScanned(false) },
          { text: 'Open', onPress: () => Linking.openURL(data) },
        ]
      );
    }
  };

  return (
    <View style={styles.container}>
      <CameraView
        style={styles.camera}
        barcodeScannerSettings={{
          barcodeTypes: ['qr'],
        }}
        onBarcodeScanned={handleBarCodeScanned}
      />
      {scanned && (
        <Button title="Scan Again" onPress={() => setScanned(false)} />
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
  },
  camera: {
    flex: 1,
  },
  message: {
    textAlign: 'center',
    padding: 20,
    fontSize: 16,
  },
});
```

### 6.4 QR Code Best Practices

**Size matters.** A QR code needs to be large enough to scan from the expected distance. Rule of thumb: the QR code size should be at least 1/10th of the scanning distance. For a poster scanned from 3 feet away, the QR code should be at least 3.6 inches.

**Error correction.** Use at least level M (15% error correction) if you're putting a logo in the center. Use L (7%) for maximum data density. Use H (30%) for QR codes that might get physically damaged (printed on products, outdoor signage).

**Always use a tracking URL.** Don't put the final destination directly in the QR code. Use a redirect URL so you can change the destination later and track scans:

```
QR Code → https://myapp.com/qr/campaign-123
                ↓ (server-side redirect)
         https://myapp.com/products/456?utm_source=qr&utm_campaign=campaign-123
```

**Test with multiple QR readers.** Different phones and QR reader apps handle QR codes differently. Test with at least: iPhone camera, Android camera, and one third-party QR app.

---

## 7. DEEP LINKING FROM NOTIFICATIONS

When a user taps a push notification, you want to navigate them to the relevant screen. This is a deep link triggered by a notification instead of a URL.

### 7.1 Notification Payload with Deep Link Data

```json
{
  "to": "ExponentPushToken[xxxxx]",
  "title": "New message from Alice",
  "body": "Hey, check out this product!",
  "data": {
    "screen": "chat",
    "chatId": "abc123",
    "deepLink": "/chat/abc123"
  }
}
```

### 7.2 Handling Notification Navigation

```tsx
// app/_layout.tsx
import { useEffect, useRef } from 'react';
import { Stack, useRouter } from 'expo-router';
import * as Notifications from 'expo-notifications';

export default function RootLayout() {
  const router = useRouter();
  const notificationResponseListener = useRef<Notifications.Subscription>();

  useEffect(() => {
    // Handle notification taps when app is already running (foreground/background)
    notificationResponseListener.current =
      Notifications.addNotificationResponseReceivedListener((response) => {
        const data = response.notification.request.content.data;
        handleNotificationNavigation(data);
      });

    // Handle notification that launched the app (cold start)
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        const data = response.notification.request.content.data;
        handleNotificationNavigation(data);
      }
    });

    return () => {
      if (notificationResponseListener.current) {
        Notifications.removeNotificationSubscription(
          notificationResponseListener.current
        );
      }
    };
  }, []);

  function handleNotificationNavigation(data: Record<string, any>) {
    if (!data) return;

    // Option 1: Use a direct deep link path from the notification data
    if (data.deepLink) {
      router.push(data.deepLink);
      return;
    }

    // Option 2: Map notification types to screens
    switch (data.screen) {
      case 'chat':
        router.push(`/chat/${data.chatId}`);
        break;
      case 'order':
        router.push(`/orders/${data.orderId}`);
        break;
      case 'product':
        router.push(`/products/${data.productId}`);
        break;
      default:
        // Unknown notification type — just go home
        console.warn('Unknown notification screen:', data.screen);
        break;
    }
  }

  return <Stack />;
}
```

### 7.3 The Navigation Timing Problem

There's a subtle bug that catches almost everyone: when the app is cold-started from a notification tap, the navigation state might not be ready when you try to navigate.

```tsx
// BAD — navigation might not be ready
useEffect(() => {
  Notifications.getLastNotificationResponseAsync().then((response) => {
    if (response) {
      router.push('/chat/123'); // 💥 Navigation state not ready!
    }
  });
}, []);

// GOOD — wait for navigation to be ready
import { useRootNavigationState } from 'expo-router';

export default function RootLayout() {
  const router = useRouter();
  const rootNavigationState = useRootNavigationState();
  const pendingNotificationRef = useRef<any>(null);

  // Store pending notification data
  useEffect(() => {
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        pendingNotificationRef.current = response.notification.request.content.data;
      }
    });
  }, []);

  // Navigate once the navigation state is ready
  useEffect(() => {
    if (!rootNavigationState?.key) return; // Navigation not ready
    if (!pendingNotificationRef.current) return; // No pending notification

    handleNotificationNavigation(pendingNotificationRef.current);
    pendingNotificationRef.current = null; // Clear the pending notification
  }, [rootNavigationState?.key]);

  return <Stack />;
}
```

### 7.4 Notification Deep Link with Auth Check

```tsx
// hooks/useNotificationDeepLink.ts
import { useEffect, useRef } from 'react';
import { useRouter } from 'expo-router';
import * as Notifications from 'expo-notifications';
import { useAuth } from './useAuth';

export function useNotificationDeepLink() {
  const router = useRouter();
  const { isAuthenticated, isLoading } = useAuth();
  const pendingNavigation = useRef<string | null>(null);

  // Listen for notification taps
  useEffect(() => {
    const subscription = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const deepLink = response.notification.request.content.data?.deepLink;
        if (deepLink) {
          navigateOrDefer(deepLink);
        }
      }
    );

    // Check for launch notification
    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        const deepLink = response.notification.request.content.data?.deepLink;
        if (deepLink) {
          navigateOrDefer(deepLink);
        }
      }
    });

    return () => subscription.remove();
  }, []);

  // Process deferred navigation once auth loads
  useEffect(() => {
    if (isLoading) return;
    if (!pendingNavigation.current) return;

    if (isAuthenticated) {
      router.push(pendingNavigation.current);
    } else {
      // Redirect to login with the intended destination
      router.push({
        pathname: '/(auth)/login',
        params: { redirect: pendingNavigation.current },
      });
    }
    pendingNavigation.current = null;
  }, [isLoading, isAuthenticated]);

  function navigateOrDefer(deepLink: string) {
    if (isLoading) {
      // Auth not ready — defer
      pendingNavigation.current = deepLink;
    } else if (isAuthenticated) {
      router.push(deepLink);
    } else {
      router.push({
        pathname: '/(auth)/login',
        params: { redirect: deepLink },
      });
    }
  }
}
```

---

## 8. TESTING DEEP LINKS

Testing deep links is uniquely annoying because it requires platform-specific tools, real devices for Universal/App Links, and careful attention to caching behavior.

### 8.1 Testing URL Schemes

URL schemes are the easiest to test because they don't require any server-side setup:

```bash
# iOS Simulator
xcrun simctl openurl booted "myapp://products/123"
xcrun simctl openurl booted "myapp://settings/profile"
xcrun simctl openurl booted "myapp://search?q=react%20native"

# Android Emulator
adb shell am start -a android.intent.action.VIEW \
  -d "myapp://products/123" \
  com.mycompany.myapp

# Using Expo's CLI
npx uri-scheme open "myapp://products/123" --ios
npx uri-scheme open "myapp://products/123" --android
```

### 8.2 Testing Universal Links (iOS)

Universal Links are harder to test because they require a valid AASA file on a real domain.

**On Simulator (limited):**

```bash
# Universal Links on Simulator require:
# 1. A valid AASA file hosted on a real domain
# 2. The app to have correct entitlements

xcrun simctl openurl booted "https://myapp.com/products/123"
```

**On Real Device (recommended):**

1. Install the app via TestFlight or development build
2. Send yourself a link via iMessage, Notes, or email (NOT from Safari on the same domain)
3. Tap the link
4. The app should open directly

**Debug mode (iOS 16+):**

```
Settings → Developer → Associated Domains Development → Enable
```

This tells iOS to re-fetch the AASA file on every app launch. Essential for development.

**Validating your AASA file:**

```bash
# 1. Direct fetch
curl -s https://myapp.com/.well-known/apple-app-site-association | jq .

# 2. Apple's CDN cache
curl -s https://app-site-association.cdn-apple.com/a/v1/myapp.com | jq .

# 3. Apple's validation tool
# https://search.developer.apple.com/appsearch-validation-tool/
```

### 8.3 Testing App Links (Android)

**On Emulator:**

```bash
# Test with adb
adb shell am start -a android.intent.action.VIEW \
  -c android.intent.category.BROWSABLE \
  -d "https://myapp.com/products/123"

# Check verification status
adb shell pm get-app-links com.mycompany.myapp

# Force re-verification
adb shell pm verify-app-links --re-verify com.mycompany.myapp
```

**On Real Device:**

1. Install the app
2. Send yourself a link via any messaging app
3. Tap the link — it should open directly in the app without a disambiguation dialog
4. If you see "Open with: Chrome or MyApp?" → verification failed

**Validating your assetlinks.json:**

```bash
# Direct fetch
curl -s https://myapp.com/.well-known/assetlinks.json | jq .

# Google's Statement List Generator and Tester
# https://developers.google.com/digital-asset-links/tools/generator
```

### 8.4 Automated Deep Link Testing

Build automated tests for your deep link routing:

```tsx
// __tests__/deeplinks.test.ts
import { renderRouter, screen } from 'expo-router/testing-library';

describe('Deep Link Routing', () => {
  it('routes /products/[id] to ProductScreen', async () => {
    renderRouter(
      {
        'products/[id]': () => <Text testID="product-screen">Product</Text>,
      },
      {
        initialUrl: '/products/123',
      }
    );

    expect(screen.getByTestId('product-screen')).toBeTruthy();
  });

  it('routes /users/[userId]/profile to ProfileScreen', async () => {
    renderRouter(
      {
        'users/[userId]/profile': () => <Text testID="profile-screen">Profile</Text>,
      },
      {
        initialUrl: '/users/42/profile',
      }
    );

    expect(screen.getByTestId('profile-screen')).toBeTruthy();
  });

  it('routes unknown paths to not-found', async () => {
    renderRouter(
      {
        '+not-found': () => <Text testID="not-found">404</Text>,
      },
      {
        initialUrl: '/this/does/not/exist',
      }
    );

    expect(screen.getByTestId('not-found')).toBeTruthy();
  });

  it('preserves query params from deep links', async () => {
    function SearchScreen() {
      const { q } = useLocalSearchParams<{ q: string }>();
      return <Text testID="search-query">{q}</Text>;
    }

    renderRouter(
      {
        search: SearchScreen,
      },
      {
        initialUrl: '/search?q=react+native',
      }
    );

    expect(screen.getByTestId('search-query').props.children).toBe('react native');
  });
});
```

### 8.5 The Deep Link Testing Checklist

Run through this checklist before every release:

```
□ URL Schemes
  □ myapp://                     → Home screen
  □ myapp://products/123         → Product detail
  □ myapp://settings             → Settings
  □ myapp://invalid-route        → 404 / graceful fallback
  □ myapp://protected-screen     → Login redirect if not authenticated

□ Universal Links (iOS)
  □ AASA file accessible at https://domain/.well-known/apple-app-site-association
  □ AASA file on Apple's CDN matches expected content
  □ https://domain/products/123  → Opens in app (from iMessage)
  □ https://domain/products/123  → Opens in Safari (app not installed)
  □ https://domain/api/anything  → Stays in Safari (excluded path)
  □ Tested on real iOS device, not just Simulator

□ App Links (Android)
  □ assetlinks.json accessible at https://domain/.well-known/assetlinks.json
  □ Certificate fingerprint matches (both upload key and Play signing key)
  □ `adb shell pm get-app-links` shows "verified"
  □ https://domain/products/123  → Opens in app (no disambiguation dialog)
  □ Tested on real Android device (Samsung recommended as largest market share)

□ Deferred Deep Links
  □ Click link → App Store → Install → Open → Correct screen
  □ Attribution data arrives correctly (campaign, source, medium)
  □ Timeout case: click link → wait 48+ hours → install → should go to home

□ Notification Deep Links
  □ Tap notification (app in foreground) → Correct screen
  □ Tap notification (app in background) → Correct screen
  □ Tap notification (app killed) → Correct screen
  □ Tap notification when not authenticated → Login → Redirect

□ Edge Cases
  □ Malformed URLs → Graceful error handling
  □ Expired deep links → Appropriate fallback
  □ Deep link during app update → Handled gracefully
```

---

## 9. ATTRIBUTION & INSTALL TRACKING

Deep links aren't just for navigation — they're a critical part of measuring marketing effectiveness. When someone installs your app from a campaign, you need to know which campaign, which channel, and which creative drove the install.

### 9.1 UTM Parameters

UTM parameters are the standard way to track marketing campaign attribution:

```
https://myapp.com/products/123?utm_source=instagram&utm_medium=social&utm_campaign=summer_sale&utm_content=carousel_ad&utm_term=running_shoes
```

```tsx
// hooks/useUTMTracking.ts
import { useEffect } from 'react';
import { useLocalSearchParams } from 'expo-router';

interface UTMParams {
  utm_source?: string;
  utm_medium?: string;
  utm_campaign?: string;
  utm_content?: string;
  utm_term?: string;
}

export function useUTMTracking() {
  const params = useLocalSearchParams<UTMParams>();

  useEffect(() => {
    const utmParams: UTMParams = {};
    
    if (params.utm_source) utmParams.utm_source = params.utm_source;
    if (params.utm_medium) utmParams.utm_medium = params.utm_medium;
    if (params.utm_campaign) utmParams.utm_campaign = params.utm_campaign;
    if (params.utm_content) utmParams.utm_content = params.utm_content;
    if (params.utm_term) utmParams.utm_term = params.utm_term;

    if (Object.keys(utmParams).length > 0) {
      // Track the attribution
      analytics.track('deep_link_attribution', utmParams);

      // Store for later use (e.g., when user signs up)
      analytics.setUserProperties({
        first_touch_source: utmParams.utm_source,
        first_touch_medium: utmParams.utm_medium,
        first_touch_campaign: utmParams.utm_campaign,
      });
    }
  }, [params]);
}
```

### 9.2 Attribution Architecture

Here's how attribution typically flows through a production app:

```
Marketing Campaign
    │
    ▼
Deep Link (with UTM params or Branch/Adjust metadata)
    │
    ├── App Installed → Deferred Deep Link Service → First Launch Attribution
    │
    └── App Already Installed → Direct Deep Link → Session Attribution
    │
    ▼
Analytics Platform (Amplitude, Mixpanel, Segment, etc.)
    │
    ├── User Properties: first_touch_source, first_touch_campaign, etc.
    │
    └── Events: deep_link_opened, campaign_attribution, etc.
    │
    ▼
Marketing Dashboard
    │
    ├── CAC per channel (Customer Acquisition Cost)
    ├── ROAS per campaign (Return on Ad Spend)
    └── Conversion funnel by source
```

### 9.3 Branch.io Attribution

Branch provides rich attribution data automatically:

```tsx
// When using Branch, you get attribution data in the subscribe callback
branch.subscribe(({ error, params }) => {
  if (error) return;
  if (!params) return;

  // Branch provides standardized attribution fields
  const attribution = {
    // Channel (where the link was shared)
    channel: params['~channel'],         // e.g., 'facebook', 'email', 'sms'
    
    // Feature (what type of link)
    feature: params['~feature'],         // e.g., 'share', 'referral', 'marketing'
    
    // Campaign
    campaign: params['~campaign'],       // e.g., 'summer_sale_2026'
    
    // Stage
    stage: params['~stage'],             // e.g., 'new_user', 'existing_user'
    
    // Tags
    tags: params['~tags'],               // e.g., ['promo', 'discount']
    
    // Is this the first session (install attribution)?
    isFirstSession: params['+is_first_session'],
    
    // Click timestamp
    clickTimestamp: params['+click_timestamp'],
    
    // Match type (how confident is the attribution)
    matchType: params['+match_guaranteed'] ? 'exact' : 'probabilistic',
    
    // Custom data you attached to the link
    productId: params['productId'],
    referrerId: params['referrerId'],
  };

  analytics.track('attribution', attribution);
});
```

### 9.4 Self-Built Attribution Tracking

If you're not using Branch or Adjust, here's a minimal attribution system:

```tsx
// lib/attribution.ts
import * as Linking from 'expo-linking';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Platform } from 'react-native';
import * as Application from 'expo-application';

const ATTRIBUTION_KEY = 'install_attribution';

interface AttributionData {
  source?: string;
  medium?: string;
  campaign?: string;
  content?: string;
  term?: string;
  deepLinkPath?: string;
  timestamp: string;
  platform: string;
  appVersion: string;
  isFirstLaunch: boolean;
}

export async function captureAttribution(url: string): Promise<AttributionData> {
  const parsed = Linking.parse(url);
  const isFirstLaunch = !(await AsyncStorage.getItem('has_launched'));

  const attribution: AttributionData = {
    source: parsed.queryParams?.utm_source as string,
    medium: parsed.queryParams?.utm_medium as string,
    campaign: parsed.queryParams?.utm_campaign as string,
    content: parsed.queryParams?.utm_content as string,
    term: parsed.queryParams?.utm_term as string,
    deepLinkPath: parsed.path ?? undefined,
    timestamp: new Date().toISOString(),
    platform: Platform.OS,
    appVersion: Application.nativeApplicationVersion ?? 'unknown',
    isFirstLaunch,
  };

  // Store attribution data
  if (isFirstLaunch) {
    await AsyncStorage.setItem(ATTRIBUTION_KEY, JSON.stringify(attribution));
    await AsyncStorage.setItem('has_launched', 'true');
  }

  return attribution;
}

export async function getInstallAttribution(): Promise<AttributionData | null> {
  const data = await AsyncStorage.getItem(ATTRIBUTION_KEY);
  return data ? JSON.parse(data) : null;
}
```

### 9.5 Server-Side Attribution Endpoint

```ts
// Backend: POST /api/attribution
interface AttributionEvent {
  userId?: string;
  deviceId: string;
  platform: 'ios' | 'android';
  appVersion: string;
  source?: string;
  medium?: string;
  campaign?: string;
  content?: string;
  term?: string;
  deepLinkPath?: string;
  isInstall: boolean;
  timestamp: string;
}

app.post('/api/attribution', async (req, res) => {
  const event: AttributionEvent = req.body;

  // Store in your analytics database
  await db.attributionEvents.create({
    data: {
      ...event,
      createdAt: new Date(),
    },
  });

  // If this is an install attribution, update user record
  if (event.isInstall && event.userId) {
    await db.users.update({
      where: { id: event.userId },
      data: {
        installSource: event.source,
        installMedium: event.medium,
        installCampaign: event.campaign,
      },
    });
  }

  res.json({ success: true });
});
```

### 9.6 Attribution Best Practices

**1. First-touch vs last-touch attribution.** Most mobile apps should track first-touch (what caused the install) because that's what marketing teams care about for CAC calculations. Store it permanently on the user record.

**2. Don't rely on a single attribution method.** Use UTM parameters as a fallback for when deferred deep links fail. Use device fingerprinting as a fallback for when UTM parameters aren't available. Belt and suspenders.

**3. Attribution windows.** Set clear attribution windows:
- Click-through attribution: 7-30 days (user clicked the ad)
- View-through attribution: 1-7 days (user saw but didn't click the ad)
- Deferred deep link matching: 24-48 hours

**4. Privacy compliance.** Attribution tracking is subject to privacy regulations:
- iOS: App Tracking Transparency (ATT) — ask permission before tracking
- Android: Google's Privacy Sandbox — moving toward limited attribution
- GDPR: User consent required in EU
- CCPA: Disclosure required in California

```tsx
// Request ATT permission before attribution tracking on iOS
import { requestTrackingPermissionsAsync } from 'expo-tracking-transparency';

async function setupAttribution() {
  if (Platform.OS === 'ios') {
    const { status } = await requestTrackingPermissionsAsync();
    if (status !== 'granted') {
      // User denied tracking — use limited attribution only
      console.log('Tracking not permitted, using limited attribution');
      return;
    }
  }

  // Full attribution tracking
  initializeFullAttribution();
}
```

**5. Send attribution data with every significant event.** Don't just track it at install time. Attach attribution data to key events (sign_up, first_purchase, subscription_start) so you can calculate ROAS per campaign:

```tsx
analytics.track('subscription_started', {
  plan: 'premium',
  price: 9.99,
  // Attribution context
  install_source: attributionData.source,
  install_campaign: attributionData.campaign,
  install_medium: attributionData.medium,
});
```

---

## 10. PUTTING IT ALL TOGETHER: A PRODUCTION DEEP LINKING SETUP

Here's a complete, production-ready deep linking configuration that ties everything together:

### 10.1 The app.config.ts

```ts
// app.config.ts
import { ExpoConfig } from 'expo/config';

const config: ExpoConfig = {
  name: 'MyApp',
  slug: 'myapp',
  scheme: 'myapp',
  
  ios: {
    bundleIdentifier: 'com.mycompany.myapp',
    associatedDomains: [
      'applinks:myapp.com',
      'applinks:links.myapp.com',
      'webcredentials:myapp.com',
    ],
  },

  android: {
    package: 'com.mycompany.myapp',
    intentFilters: [
      {
        action: 'VIEW',
        autoVerify: true,
        data: [
          { scheme: 'https', host: 'myapp.com', pathPrefix: '/products' },
          { scheme: 'https', host: 'myapp.com', pathPrefix: '/users' },
          { scheme: 'https', host: 'myapp.com', pathPrefix: '/share' },
          { scheme: 'https', host: 'myapp.com', pathPrefix: '/orders' },
          { scheme: 'https', host: 'myapp.com', pathPrefix: '/chat' },
          { scheme: 'https', host: 'links.myapp.com', pathPrefix: '/' },
        ],
        category: ['BROWSABLE', 'DEFAULT'],
      },
    ],
  },

  plugins: [
    'expo-router',
    [
      'react-native-branch',
      {
        apiKey: process.env.BRANCH_KEY,
        iosAppDomain: 'myapp.app.link',
      },
    ],
  ],

  experiments: {
    typedRoutes: true,
  },
};

export default config;
```

### 10.2 The Root Layout (All Deep Link Handling)

```tsx
// app/_layout.tsx
import { useEffect, useRef } from 'react';
import { Stack, useRouter, useSegments, useRootNavigationState } from 'expo-router';
import * as Linking from 'expo-linking';
import * as Notifications from 'expo-notifications';
import branch from 'react-native-branch';
import { useAuth } from '../hooks/useAuth';
import { captureAttribution } from '../lib/attribution';
import { analytics } from '../lib/analytics';

export default function RootLayout() {
  const router = useRouter();
  const segments = useSegments();
  const rootNavigationState = useRootNavigationState();
  const { isAuthenticated, isLoading } = useAuth();
  const pendingDeepLink = useRef<string | null>(null);

  // === 1. Branch (deferred deep links + attribution) ===
  useEffect(() => {
    const unsubscribe = branch.subscribe(({ error, params }) => {
      if (error || !params) return;

      // Track attribution
      analytics.track('branch_deep_link', {
        channel: params['~channel'],
        campaign: params['~campaign'],
        isFirstSession: params['+is_first_session'],
      });

      // Navigate
      const path = params['$canonical_url']
        ? new URL(params['$canonical_url']).pathname
        : params.deepLink;

      if (path) {
        navigateOrDefer(path);
      }
    });

    return () => unsubscribe();
  }, []);

  // === 2. Standard deep links (URL schemes + Universal/App Links) ===
  useEffect(() => {
    // Cold start
    Linking.getInitialURL().then((url) => {
      if (url) {
        captureAttribution(url);
        const parsed = Linking.parse(url);
        if (parsed.path) {
          navigateOrDefer(`/${parsed.path}`);
        }
      }
    });

    // Warm start
    const subscription = Linking.addEventListener('url', (event) => {
      captureAttribution(event.url);
      const parsed = Linking.parse(event.url);
      if (parsed.path) {
        navigateOrDefer(`/${parsed.path}`);
      }
    });

    return () => subscription.remove();
  }, []);

  // === 3. Notification deep links ===
  useEffect(() => {
    const subscription = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const deepLink = response.notification.request.content.data?.deepLink;
        if (deepLink) navigateOrDefer(deepLink);
      }
    );

    Notifications.getLastNotificationResponseAsync().then((response) => {
      if (response) {
        const deepLink = response.notification.request.content.data?.deepLink;
        if (deepLink) navigateOrDefer(deepLink);
      }
    });

    return () => subscription.remove();
  }, []);

  // === 4. Process pending deep link when navigation is ready ===
  useEffect(() => {
    if (!rootNavigationState?.key) return;
    if (!pendingDeepLink.current) return;
    if (isLoading) return;

    const link = pendingDeepLink.current;
    pendingDeepLink.current = null;

    if (isAuthenticated || isPublicRoute(link)) {
      router.push(link);
    } else {
      router.push({
        pathname: '/(auth)/login',
        params: { redirect: link },
      });
    }
  }, [rootNavigationState?.key, isLoading, isAuthenticated]);

  function navigateOrDefer(path: string) {
    if (!rootNavigationState?.key || isLoading) {
      pendingDeepLink.current = path;
      return;
    }

    if (isAuthenticated || isPublicRoute(path)) {
      router.push(path);
    } else {
      router.push({
        pathname: '/(auth)/login',
        params: { redirect: path },
      });
    }
  }

  function isPublicRoute(path: string): boolean {
    const publicRoutes = ['/about', '/terms', '/privacy', '/share/'];
    return publicRoutes.some((route) => path.startsWith(route));
  }

  return <Stack />;
}
```

### 10.3 Server-Side Configuration

```json
// public/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.mycompany.myapp",
        "paths": [
          "/products/*",
          "/users/*",
          "/share/*",
          "/orders/*",
          "/chat/*",
          "NOT /api/*",
          "NOT /admin/*",
          "NOT /_next/*"
        ],
        "appIDs": ["TEAM_ID.com.mycompany.myapp"],
        "components": [
          { "/": "/products/*" },
          { "/": "/users/*" },
          { "/": "/share/*" },
          { "/": "/orders/*" },
          { "/": "/chat/*" },
          { "/": "/api/*", "exclude": true },
          { "/": "/admin/*", "exclude": true },
          { "/": "/_next/*", "exclude": true }
        ]
      }
    ]
  }
}
```

```json
// public/.well-known/assetlinks.json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": [
        "UPLOAD_KEY_FINGERPRINT",
        "PLAY_STORE_SIGNING_KEY_FINGERPRINT"
      ]
    }
  }
]
```

---

## 11. DEEP LINKING ARCHITECTURE PATTERNS

### 11.1 The Deep Link Router Pattern

For complex apps, centralize all deep link resolution in a single router:

```tsx
// lib/deepLinkRouter.ts
type DeepLinkHandler = {
  pattern: RegExp;
  handler: (match: RegExpMatchArray) => string; // Returns the in-app route
};

const deepLinkHandlers: DeepLinkHandler[] = [
  {
    pattern: /^\/products\/(\w+)$/,
    handler: (match) => `/products/${match[1]}`,
  },
  {
    pattern: /^\/users\/(\w+)\/profile$/,
    handler: (match) => `/users/${match[1]}/profile`,
  },
  {
    pattern: /^\/share\/(\w+)$/,
    handler: async (match) => {
      // Share links might need server-side resolution
      const response = await fetch(`https://api.myapp.com/share/${match[1]}`);
      const data = await response.json();
      return data.targetRoute; // e.g., "/products/456"
    },
  },
  {
    pattern: /^\/invite\/(\w+)$/,
    handler: (match) => {
      // Store the invite code for later processing
      storeInviteCode(match[1]);
      return '/onboarding';
    },
  },
];

export function resolveDeepLink(path: string): string | null {
  for (const { pattern, handler } of deepLinkHandlers) {
    const match = path.match(pattern);
    if (match) {
      return handler(match);
    }
  }
  
  // No handler found — try to navigate directly
  // (Expo Router will handle 404 if the route doesn't exist)
  return path;
}
```

### 11.2 The Short Link Pattern

Instead of putting full paths in QR codes or sharing long URLs, use short links that resolve server-side:

```
https://myapp.com/s/abc123
    → Server looks up abc123
    → Returns { targetPath: '/products/456', metadata: {...} }
    → App navigates to /products/456
```

```ts
// Backend
app.get('/s/:code', async (req, res) => {
  const link = await db.shortLinks.findUnique({
    where: { code: req.params.code },
  });

  if (!link) {
    return res.status(404).json({ error: 'Link not found' });
  }

  // Track the click
  await db.linkClicks.create({
    data: {
      linkId: link.id,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      timestamp: new Date(),
    },
  });

  // For web browsers, redirect to the target
  // For the app (detected via user-agent or query param), return JSON
  if (req.headers.accept?.includes('application/json')) {
    return res.json({
      targetPath: link.targetPath,
      metadata: link.metadata,
    });
  }

  // Web redirect (also works as Universal Link fallback)
  res.redirect(302, `https://myapp.com${link.targetPath}`);
});
```

### 11.3 Analytics-Enriched Deep Links

```tsx
// lib/enrichedDeepLink.ts
import { analytics } from './analytics';
import * as Linking from 'expo-linking';

export function createEnrichedLink(
  path: string,
  context: {
    source: string;
    campaign?: string;
    content?: string;
  }
): string {
  const params = new URLSearchParams({
    utm_source: context.source,
    ...(context.campaign && { utm_campaign: context.campaign }),
    ...(context.content && { utm_content: context.content }),
    utm_medium: 'app_share', // Always 'app_share' for in-app sharing
  });

  return `https://myapp.com${path}?${params.toString()}`;
}

// Usage in a share button:
function ShareButton({ productId, productName }: Props) {
  const handleShare = async () => {
    const url = createEnrichedLink(`/products/${productId}`, {
      source: 'in_app',
      campaign: 'product_share',
      content: productId,
    });

    await Share.share({
      message: `Check out ${productName}! ${url}`,
      url, // iOS only — puts the URL in the share sheet separately
    });

    analytics.track('share_initiated', { productId, url });
  };

  return <Button onPress={handleShare} title="Share" />;
}
```

---

## SUMMARY

Deep linking is infrastructure. Like authentication or push notifications, it touches every part of your app and needs to be set up correctly from the start. Retrofitting deep links into an existing app is possible but painful — you'll be modifying navigation logic, adding server-side config, updating app store settings, and testing across platforms.

Here's the minimum viable deep linking setup for a production app:

1. **URL scheme** (`myapp://`) in `app.config.ts` for development and fallback
2. **Universal Links** (iOS) with AASA file on your domain + associated domains entitlement
3. **App Links** (Android) with `assetlinks.json` on your domain + intent filters with `autoVerify`
4. **Deferred deep links** via Branch.io (or similar) for install attribution
5. **Notification deep links** with proper navigation timing
6. **UTM parameter tracking** on all shared links
7. **Automated tests** for critical deep link paths
8. **A testing checklist** that runs before every release

Get these right and your app becomes a first-class citizen of the URL-based web. Every screen is addressable. Every marketing campaign is trackable. Every notification lands the user in exactly the right place.

Get them wrong and... well, you'll spend a lot of time in Console.app wondering why your AASA file isn't being fetched. Ask me how I know.

---

**Next:** [Chapter 12: Storage, Persistence & File System](/part-2-react-native-expo/12-storage-persistence) — how to store data on the device, from key-value pairs to SQLite databases to encrypted vaults.
