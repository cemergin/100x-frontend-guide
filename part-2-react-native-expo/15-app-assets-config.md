<!--
  CHAPTER: 15
  TITLE: App Icons, Splash Screens, Assets & Config Deep Dive
  PART: II — React Native & Expo
  PREREQS: Chapter 6
  KEY_TOPICS: adaptive icons, dynamic app icons, splash screen config, asset generation, app store screenshots, app.config.ts deep dive, eas.json deep dive, expo plugins, environment-specific config
  DIFFICULTY: Beginner → Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 15: App Icons, Splash Screens, Assets & Config Deep Dive

> **Part II — React Native & Expo** | Prerequisites: Chapter 6 | Difficulty: Beginner to Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Start with a single 1024x1024 PNG for your app icon. Expo generates every size for both platforms automatically. For Android adaptive icons, provide a separate foreground (1024x1024 with transparent background) and optionally a background color or image. For iOS 18+ dark mode icons, provide a dark variant.
- Use expo-splash-screen to keep the splash visible until your app is ready (data loaded, fonts loaded, auth checked). The splash screen is not decorative — it is a UX bridge between launch and interactive state.
- app.config.ts is the single source of truth for your entire app. Learn every property that matters. Do not copy-paste a config from a tutorial — understand what each field does and why it exists.
- eas.json controls how your app is built, submitted, and updated. Master build profiles (development, preview, production), auto-increment, channels, and credential management.
- Use APP_VARIANT to generate different app names, bundle IDs, and icons for dev/staging/production from a single codebase. This lets you install all three variants side-by-side on the same device.
- Config plugins are how you modify native project configuration without maintaining native code. Understand how they work before you write one — most of the time, an existing plugin already does what you need.

</details>

Nobody talks about this stuff at conferences. Nobody writes blog posts titled "How We Nailed Our Splash Screen Configuration." It is not glamorous. But I will tell you what is glamorous: shipping an app where the icon looks crisp on every device, the splash screen does not flash white before your app loads, the store listing has professional screenshots, and your CI pipeline builds three different app variants from one codebase without anyone touching a config file.

This is the infrastructure chapter. The one that saves you from the 2 AM discovery that your Android icon has a white square around it because you did not understand adaptive icons. The one that prevents the support ticket from your QA team saying "the staging app overwrote the production app on my phone." The one that means you never manually edit an Xcode project file again.

Let us get into it.

### In This Chapter
- App Icons — All platforms, adaptive icons, dark mode, dynamic icons
- Splash Screens — Configuration, animated splash, keeping splash visible
- Asset Management — Bundled vs CDN, preloading, fonts, image optimization
- App Store Assets — Screenshots, preview videos, promotional art
- app.config.ts Deep Dive — Every property that matters
- eas.json Deep Dive — Build profiles, submit, credentials
- Environment-Specific Config — APP_VARIANT for dev/staging/production
- Config Plugins — How they work, common plugins, writing your own

### Related Chapters
- [Ch 5: Expo Platform] — Continuous Native Generation and the Expo ecosystem
- [Ch 6: EAS Mastery] — Build, submit, and update pipeline
- [Ch 16: Design Systems] — Design tokens and component libraries
- [Ch 18: CI/CD] — Automated build and deployment pipelines

---

## 15.1 App Icons — Getting Every Pixel Right

### The Icon Landscape

Your app icon is the first thing users see. It appears on the home screen, in the app store, in the task switcher, in notifications, in Settings, in Spotlight search. It is rendered at dozens of different sizes across two platforms. Getting it wrong — a blurry icon, a white border, a dark-mode icon that disappears into a dark background — is an instant signal that the app is not professional.

Here is every icon size you need to worry about:

#### iOS Icon Sizes

| Size | Where It Appears |
|------|-----------------|
| 1024x1024 | App Store listing |
| 180x180 (60pt @3x) | Home screen (iPhone) |
| 120x120 (60pt @2x) | Home screen (older iPhone) |
| 167x167 (83.5pt @2x) | Home screen (iPad Pro) |
| 152x152 (76pt @2x) | Home screen (iPad) |
| 120x120 (40pt @3x) | Spotlight search (iPhone) |
| 80x80 (40pt @2x) | Spotlight search (iPad) |
| 87x87 (29pt @3x) | Settings (iPhone) |
| 58x58 (29pt @2x) | Settings (iPad) |
| 76x76 | Notifications |
| 40x40 | Notifications (small) |

#### Android Icon Sizes

| Size | Density | Where It Appears |
|------|---------|-----------------|
| 512x512 | Play Store | Store listing |
| 192x192 | xxxhdpi | Home screen (high-end) |
| 144x144 | xxhdpi | Home screen |
| 96x96 | xhdpi | Home screen |
| 72x72 | hdpi | Home screen (older devices) |
| 48x48 | mdpi | Home screen (low-end) |

**The good news:** You do not need to create all of these manually. Expo generates every required size from a single source image.

### Expo's Icon Generation

Provide a single 1024x1024 PNG in your Expo config, and Expo handles the rest:

```typescript
// app.config.ts
export default {
  expo: {
    icon: './assets/icon.png', // 1024x1024 PNG, no transparency for iOS
  },
};
```

**Requirements for the source icon:**
- **Size:** 1024x1024 pixels exactly
- **Format:** PNG
- **iOS:** No transparency (Apple rejects icons with alpha channels). Use a solid background.
- **Shape:** Square. iOS rounds the corners automatically. Android applies its own mask (circle, squircle, etc. depending on the launcher).
- **Safe zone:** Keep your logo within the center 80% of the image. The outer 20% may be clipped by platform icon masks.

```
┌─────────────────────────────────┐
│                                 │
│   ┌───────────────────────┐    │
│   │                       │    │
│   │                       │    │
│   │      YOUR LOGO        │    │
│   │    (keep within       │    │
│   │     this zone)        │    │
│   │                       │    │
│   │                       │    │
│   └───────────────────────┘    │
│                                 │
│  ← 10% margin on each side →  │
└─────────────────────────────────┘
         1024 x 1024
```

### Android Adaptive Icons

Android 8.0 introduced adaptive icons — icons that consist of a **foreground** layer and a **background** layer. The system composites them together and applies a mask (circle, rounded square, squircle, etc.) based on the device manufacturer's preference.

```
Adaptive Icon Composition:

┌─────────────┐   ┌─────────────┐
│             │   │             │
│ FOREGROUND  │ + │ BACKGROUND  │
│ (your logo, │   │ (solid color│
│  transparent│   │  or image)  │
│  background)│   │             │
└─────────────┘   └─────────────┘
        │                │
        ▼                ▼
┌─────────────────────────┐
│    COMPOSITED ICON      │
│  ┌─────────────────┐    │
│  │  /‾‾‾‾‾‾‾‾‾\   │    │
│  │ |  ★ LOGO ★  |  │    │  ← Mask applied by OS
│  │  \_________/   │    │     (circle, squircle, etc.)
│  └─────────────────┘    │
└─────────────────────────┘
```

```typescript
// app.config.ts — Adaptive icon configuration
export default {
  expo: {
    icon: './assets/icon.png', // Fallback for iOS and non-adaptive Android

    android: {
      adaptiveIcon: {
        foregroundImage: './assets/adaptive-icon-foreground.png',
        // 1024x1024, transparent background
        // Your logo should be within the center 66% (the "safe zone")
        // The outer 33% is used for parallax animation effects

        backgroundColor: '#1a1a2e',
        // OR use a background image:
        // backgroundImage: './assets/adaptive-icon-background.png',

        monochromeImage: './assets/adaptive-icon-monochrome.png',
        // Optional: for Android 13+ themed icons
        // Single-color silhouette of your icon
      },
    },
  },
};
```

**Adaptive icon safe zones are more aggressive than you think.** The foreground image is 108dp x 108dp, but only the inner 72dp x 72dp (66%) is guaranteed to be visible. The outer area is used for the parallax motion effect when the user tilts their device or long-presses the icon. If your logo extends into the outer zone, it will get clipped on some devices.

```
Adaptive Icon Foreground Safe Zone:

┌─────────────────────────────────────┐
│                                     │
│     PARALLAX / CLIPPING ZONE        │
│     (may not be visible)            │
│                                     │
│   ┌─────────────────────────────┐   │
│   │                             │   │
│   │      SAFE ZONE (66%)        │   │
│   │      Put your logo here     │   │
│   │                             │   │
│   └─────────────────────────────┘   │
│                                     │
│                                     │
└─────────────────────────────────────┘
```

### iOS 18+ Dark Mode Icons

Starting with iOS 18, users can set their home screen to dark mode, and apps can provide dark variants of their icons. Expo supports this:

```typescript
// app.config.ts — Dark mode icon for iOS 18+
export default {
  expo: {
    icon: './assets/icon-light.png',

    ios: {
      icon: {
        light: './assets/icon-light.png',
        dark: './assets/icon-dark.png',
        tinted: './assets/icon-tinted.png',
        // 'tinted' is a monochrome version that iOS tints
        // with the user's chosen color
      },
    },
  },
};
```

**Design tips for dark mode icons:**
- Do not just invert your light icon. Redesign it for dark backgrounds.
- Test on actual iOS devices — the simulator does not always represent icon rendering accurately.
- Keep the icon recognizable. Your users should know it is your app regardless of light/dark mode.
- The tinted variant should be a single-color silhouette that works when overlaid with any color.

### Dynamic App Icons (Changing at Runtime)

Some apps let users choose their app icon — a personalization feature that users love. Think of how apps like Telegram or GitHub offer alternative icons.

```bash
# Install the Expo module
npx expo install expo-dynamic-app-icon
```

```typescript
// app.config.ts — Define alternative icons
export default {
  expo: {
    plugins: [
      [
        'expo-dynamic-app-icon',
        {
          default: {
            image: './assets/icons/default.png',
            prerendered: true,
          },
          dark: {
            image: './assets/icons/dark.png',
            prerendered: true,
          },
          pride: {
            image: './assets/icons/pride.png',
            prerendered: true,
          },
          minimal: {
            image: './assets/icons/minimal.png',
            prerendered: true,
          },
        },
      ],
    ],
  },
};
```

```typescript
// screens/IconPicker.tsx — Let users choose their icon
import * as DynamicAppIcon from 'expo-dynamic-app-icon';
import { Alert, Platform } from 'react-native';

const ICONS = [
  { id: 'default', label: 'Default', preview: require('./assets/icons/default.png') },
  { id: 'dark', label: 'Dark', preview: require('./assets/icons/dark.png') },
  { id: 'pride', label: 'Pride', preview: require('./assets/icons/pride.png') },
  { id: 'minimal', label: 'Minimal', preview: require('./assets/icons/minimal.png') },
];

export function IconPickerScreen() {
  const currentIcon = DynamicAppIcon.getAppIcon();

  const changeIcon = async (iconName: string) => {
    try {
      const result = await DynamicAppIcon.setAppIcon(iconName);
      if (result) {
        // iOS shows a system alert when the icon changes.
        // There is no way to suppress it. This is by design.
        // On Android, it may take a moment for the launcher to update.
        if (Platform.OS === 'android') {
          Alert.alert('Icon Updated', 'Your home screen icon will update shortly.');
        }
      }
    } catch (error) {
      console.error('Failed to change app icon:', error);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Choose Your Icon</Text>
      <View style={styles.grid}>
        {ICONS.map((icon) => (
          <Pressable
            key={icon.id}
            onPress={() => changeIcon(icon.id)}
            style={[
              styles.iconOption,
              currentIcon === icon.id && styles.iconOptionSelected,
            ]}
          >
            <Image source={icon.preview} style={styles.iconPreview} />
            <Text style={styles.iconLabel}>{icon.label}</Text>
            {currentIcon === icon.id && (
              <View style={styles.checkmark}>
                <Text>✓</Text>
              </View>
            )}
          </Pressable>
        ))}
      </View>
    </View>
  );
}
```

**Caveats with dynamic app icons:**
- On iOS, the system shows an alert dialog every time the icon changes. You cannot suppress this. It is an Apple design decision.
- On Android, some launchers are slow to update. The user might need to restart their launcher or wait a few seconds.
- Each alternative icon must be included in the app binary at build time. You cannot download icons from the internet and set them.
- Keep the number of alternatives reasonable. Each icon adds to your binary size.

---

## 15.2 Splash Screens — The Bridge Between Launch and Ready

### Why Splash Screens Matter

The splash screen is not decoration. It is a UX mechanism that bridges the gap between the moment the user taps your icon and the moment your app is ready to use. Without a splash screen, users see a white (or black) screen for 1-3 seconds while your JavaScript bundle loads, your fonts load, your authentication state is checked, and your initial data is fetched. That blank screen feels broken.

A well-configured splash screen:
1. Appears instantly when the app launches (it is a native screen, not a React component)
2. Shows your brand (logo + background color)
3. Stays visible until your app is actually ready
4. Transitions smoothly to your first screen

### expo-splash-screen Configuration

```bash
npx expo install expo-splash-screen
```

```typescript
// app.config.ts — Splash screen configuration
export default {
  expo: {
    splash: {
      image: './assets/splash-icon.png',
      // The image that appears on the splash screen.
      // This is NOT a full-screen image — it is your logo/icon
      // centered on the background color.
      // Recommended: 200x200 to 400x400 for the logo.

      backgroundColor: '#1a1a2e',
      // The color that fills the rest of the screen.
      // Match this to your app's primary background.

      resizeMode: 'contain',
      // 'contain' — scales image to fit within the screen, maintaining aspect ratio
      // 'cover' — scales image to fill the screen, may crop

      // iOS-specific
      dark: {
        image: './assets/splash-icon-dark.png',
        backgroundColor: '#000000',
      },
    },

    // For full-bleed splash screens (edge to edge image):
    // Use the new expo-splash-screen plugin approach instead:
    plugins: [
      [
        'expo-splash-screen',
        {
          image: './assets/splash-icon.png',
          imageWidth: 200,
          backgroundColor: '#1a1a2e',
          dark: {
            image: './assets/splash-icon-dark.png',
            backgroundColor: '#000000',
          },
        },
      ],
    ],
  },
};
```

### Keeping the Splash Screen Visible Until Ready

This is the most important pattern in this section. By default, the splash screen hides as soon as the first React component renders. But your app is not ready when the first component renders — you still need to load fonts, check auth, fetch initial data. Here is the correct pattern:

```typescript
// App.tsx — Keep splash visible until truly ready
import * as SplashScreen from 'expo-splash-screen';
import * as Font from 'expo-font';
import { useEffect, useState, useCallback } from 'react';
import { View } from 'react-native';

// Prevent the splash screen from auto-hiding
SplashScreen.preventAutoHideAsync();

export default function App() {
  const [appIsReady, setAppIsReady] = useState(false);

  useEffect(() => {
    async function prepare() {
      try {
        // Load fonts
        await Font.loadAsync({
          'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
          'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
        });

        // Check authentication state
        await checkAuthState();

        // Pre-fetch critical data
        await prefetchHomeScreenData();

        // Any other initialization
        await initializeAnalytics();
      } catch (error) {
        // If something fails, we still want to show the app
        // (with an error state, not a splash screen forever)
        console.warn('Initialization error:', error);
      } finally {
        setAppIsReady(true);
      }
    }

    prepare();
  }, []);

  const onLayoutRootView = useCallback(async () => {
    if (appIsReady) {
      // Hide the splash screen AFTER the root view has laid out.
      // This ensures there is no flash of white between splash and app.
      await SplashScreen.hideAsync();
    }
  }, [appIsReady]);

  if (!appIsReady) {
    return null; // Splash screen is still visible
  }

  return (
    <View style={{ flex: 1 }} onLayout={onLayoutRootView}>
      <RootNavigator />
    </View>
  );
}
```

### Animated Splash Screens

For a more polished experience, you can animate the transition from splash to app. The approach: render your own splash screen component on top of the app, animate it out, then remove it.

```typescript
// components/AnimatedSplash.tsx
import { useEffect, useRef, useState } from 'react';
import { Animated, Dimensions, StyleSheet, View } from 'react-native';
import * as SplashScreen from 'expo-splash-screen';

const { width, height } = Dimensions.get('window');

interface AnimatedSplashProps {
  children: React.ReactNode;
  isReady: boolean;
}

export function AnimatedSplash({ children, isReady }: AnimatedSplashProps) {
  const [splashHidden, setSplashHidden] = useState(false);
  const fadeAnim = useRef(new Animated.Value(1)).current;
  const scaleAnim = useRef(new Animated.Value(1)).current;

  useEffect(() => {
    if (isReady) {
      // Hide the native splash screen
      SplashScreen.hideAsync();

      // Animate our custom splash overlay out
      Animated.parallel([
        Animated.timing(fadeAnim, {
          toValue: 0,
          duration: 500,
          useNativeDriver: true,
        }),
        Animated.timing(scaleAnim, {
          toValue: 1.5,
          duration: 500,
          useNativeDriver: true,
        }),
      ]).start(() => {
        setSplashHidden(true);
      });
    }
  }, [isReady, fadeAnim, scaleAnim]);

  return (
    <View style={styles.container}>
      {children}
      {!splashHidden && (
        <Animated.View
          style={[
            styles.splashOverlay,
            {
              opacity: fadeAnim,
              transform: [{ scale: scaleAnim }],
            },
          ]}
        >
          <Animated.Image
            source={require('../assets/splash-icon.png')}
            style={styles.splashImage}
            resizeMode="contain"
          />
        </Animated.View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  splashOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: '#1a1a2e',
    alignItems: 'center',
    justifyContent: 'center',
  },
  splashImage: {
    width: 200,
    height: 200,
  },
});
```

```typescript
// App.tsx — Usage
export default function App() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    prepare().then(() => setIsReady(true));
  }, []);

  return (
    <AnimatedSplash isReady={isReady}>
      <RootNavigator />
    </AnimatedSplash>
  );
}
```

---

## 15.3 Asset Management — Bundled, Remote, and Everything Between

### Bundled Assets vs. CDN Assets

Your app has two kinds of assets:

**Bundled assets** are included in the app binary. They are available offline, load instantly, and increase your binary size. Use them for:
- App icon, splash screen
- Core UI images (onboarding illustrations, empty states, placeholder images)
- Fonts
- Lottie animations for core UI
- Sound effects

**CDN assets** are downloaded at runtime from a server. They reduce binary size but require a network connection and have loading latency. Use them for:
- User-uploaded images
- Product images
- Marketing content that changes frequently
- Large media files (videos, high-res images)

```typescript
// Bundled asset — available immediately, increases binary size
const logo = require('./assets/logo.png');

// CDN asset — needs network, but doesn't affect binary size
const productImage = { uri: 'https://cdn.example.com/products/123.jpg' };
```

### expo-asset for Preloading

If you have bundled assets that need to be loaded before your app displays (like images for an onboarding flow), preload them during the splash screen phase:

```typescript
// lib/preload.ts
import { Asset } from 'expo-asset';
import * as Font from 'expo-font';

const PRELOAD_IMAGES = [
  require('../assets/onboarding-1.png'),
  require('../assets/onboarding-2.png'),
  require('../assets/onboarding-3.png'),
  require('../assets/empty-state.png'),
];

const PRELOAD_FONTS = {
  'Inter-Regular': require('../assets/fonts/Inter-Regular.ttf'),
  'Inter-Medium': require('../assets/fonts/Inter-Medium.ttf'),
  'Inter-SemiBold': require('../assets/fonts/Inter-SemiBold.ttf'),
  'Inter-Bold': require('../assets/fonts/Inter-Bold.ttf'),
};

export async function preloadAssets(): Promise<void> {
  const imagePromises = PRELOAD_IMAGES.map((image) => {
    return Asset.fromModule(image).downloadAsync();
  });

  const fontPromise = Font.loadAsync(PRELOAD_FONTS);

  await Promise.all([...imagePromises, fontPromise]);
}
```

### Font Loading

Fonts are one of the most common sources of visual jank in mobile apps. If you render text before the font is loaded, the user sees a flash of the system font (or nothing) before the custom font appears.

```bash
npx expo install expo-font
```

```typescript
// Load fonts during app initialization
import * as Font from 'expo-font';

// Method 1: In your preparation function
await Font.loadAsync({
  'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
  'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
});

// Method 2: Using the useFonts hook (simpler for small apps)
import { useFonts } from 'expo-font';

export default function App() {
  const [fontsLoaded] = useFonts({
    'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
    'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
  });

  if (!fontsLoaded) return null;

  return <RootNavigator />;
}
```

**Variable fonts** save significant binary size by including all weights in a single file:

```typescript
// Instead of loading 6 separate font files (120KB each = 720KB)...
await Font.loadAsync({
  'Inter-Light': require('./assets/fonts/Inter-Light.ttf'),
  'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
  'Inter-Medium': require('./assets/fonts/Inter-Medium.ttf'),
  'Inter-SemiBold': require('./assets/fonts/Inter-SemiBold.ttf'),
  'Inter-Bold': require('./assets/fonts/Inter-Bold.ttf'),
  'Inter-ExtraBold': require('./assets/fonts/Inter-ExtraBold.ttf'),
});

// ...load one variable font file (~300KB)
await Font.loadAsync({
  'Inter': require('./assets/fonts/Inter-Variable.ttf'),
});

// Usage with fontWeight (React Native supports this with variable fonts)
<Text style={{ fontFamily: 'Inter', fontWeight: '600' }}>
  Semi-bold text
</Text>
```

### Image Optimization Pipeline

Images are usually the largest assets in a mobile app. An optimization pipeline can reduce your binary size dramatically:

```typescript
// scripts/optimize-images.ts
// Run this in CI before building

import sharp from 'sharp';
import { glob } from 'glob';
import path from 'path';

async function optimizeImages() {
  const images = await glob('assets/**/*.{png,jpg,jpeg}');

  for (const imagePath of images) {
    const ext = path.extname(imagePath).toLowerCase();
    const image = sharp(imagePath);
    const metadata = await image.metadata();

    if (ext === '.png') {
      await image
        .png({
          quality: 80,
          compressionLevel: 9,
          palette: true, // Use palette-based encoding when possible
        })
        .toFile(imagePath.replace(ext, '.optimized.png'));
    } else if (ext === '.jpg' || ext === '.jpeg') {
      await image
        .jpeg({
          quality: 80,
          mozjpeg: true, // Use mozjpeg for better compression
        })
        .toFile(imagePath.replace(ext, '.optimized.jpg'));
    }

    const original = metadata.size || 0;
    const optimized = (await sharp(imagePath.replace(ext, `.optimized${ext}`)).metadata()).size || 0;
    const savings = ((1 - optimized / original) * 100).toFixed(1);

    console.log(`${imagePath}: ${original} → ${optimized} (${savings}% smaller)`);
  }
}

optimizeImages();
```

**Image format decision guide:**

| Format | Best For | Compression | Transparency |
|--------|----------|-------------|-------------|
| PNG | Icons, logos, UI elements with transparency | Lossless | Yes |
| JPEG | Photos, complex images | Lossy | No |
| WebP | Most images (smaller than PNG/JPEG) | Both | Yes |
| SVG | Scalable icons, simple illustrations | N/A | Yes |
| Lottie (JSON) | Animated illustrations | N/A | Yes |

---

## 15.4 App Store Assets — The Screenshots and Art That Sell Your App

### Screenshot Sizes

Both app stores require screenshots at specific sizes. Miss a size and your listing looks broken. Here is every size you need:

#### iOS Screenshot Sizes (Required)

| Device | Size (pixels) | Required? |
|--------|--------------|-----------|
| iPhone 6.9" (16 Pro Max) | 1320 x 2868 | Required (covers all iPhones) |
| iPhone 6.7" (15 Plus/Pro Max) | 1290 x 2796 | Required |
| iPhone 6.5" (14 Plus, 11 Pro Max) | 1284 x 2778 or 1242 x 2688 | Required |
| iPhone 5.5" (8 Plus, SE 3rd gen) | 1242 x 2208 | Optional (Apple can scale down) |
| iPad Pro 13" | 2064 x 2752 | Required if app supports iPad |
| iPad Pro 11" | 1668 x 2388 | Required if app supports iPad |

**Pro tip:** As of 2026, Apple lets you provide just the 6.9" and 6.7" screenshots and automatically scales them for smaller devices. But if you want pixel-perfect control at every size, provide all of them.

#### Google Play Screenshot Requirements

| Type | Size | Required? |
|------|------|-----------|
| Phone screenshots | Min 320px, max 3840px on any side. 16:9 or 9:16 aspect ratio. | Yes (2-8 screenshots) |
| Tablet screenshots | Same as phone, but 7"+ tablets | Required if targeting tablets |
| Feature graphic | 1024 x 500 | Yes |
| TV banner | 1280 x 720 | Only if targeting Android TV |

### Generating Screenshots with Maestro

Manual screenshots are a nightmare. Every time you change the UI, you need to retake dozens of screenshots across multiple device sizes. Automate this with Maestro:

```yaml
# .maestro/screenshots/onboarding-flow.yaml
appId: com.acme.app
---
- clearState
- launchApp

# Screenshot 1: Welcome screen
- waitForAnimationToEnd
- takeScreenshot: screenshots/01-welcome

# Screenshot 2: Feature highlights
- tapOn: "Get Started"
- waitForAnimationToEnd
- takeScreenshot: screenshots/02-features

# Screenshot 3: Main dashboard
- tapOn: "Continue"
- waitForAnimationToEnd
- takeScreenshot: screenshots/03-dashboard

# Screenshot 4: Project view
- tapOn: "Sample Project"
- waitForAnimationToEnd
- takeScreenshot: screenshots/04-project

# Screenshot 5: Settings
- tapOn:
    id: "tab-settings"
- waitForAnimationToEnd
- takeScreenshot: screenshots/05-settings
```

```bash
# Run on all required device sizes
maestro test .maestro/screenshots/onboarding-flow.yaml \
  --device "iPhone 16 Pro Max" \
  --output screenshots/iphone-6.9

maestro test .maestro/screenshots/onboarding-flow.yaml \
  --device "iPhone 15 Plus" \
  --output screenshots/iphone-6.7

maestro test .maestro/screenshots/onboarding-flow.yaml \
  --device "iPad Pro 13-inch" \
  --output screenshots/ipad-13
```

### Generating Screenshots with Fastlane

Fastlane's `snapshot` tool uses UI tests to capture screenshots:

```ruby
# fastlane/Snapfile
devices([
  "iPhone 16 Pro Max",
  "iPhone 15 Plus",
  "iPad Pro (13-inch) (M4)",
])

languages(["en-US", "de-DE", "ja"])

scheme("MyApp")
output_directory("./screenshots")
clear_previous_screenshots(true)
```

### App Preview Videos

Short videos (15-30 seconds) that autoplay in the App Store listing. They dramatically increase conversion rates.

**iOS App Previews:**
- Up to 3 videos per localization
- 15-30 seconds each
- Must use one of the supported resolutions (match your screenshot device sizes)
- No hands or device frames in the video (Apple's current rule)

**Google Play Promo Videos:**
- YouTube link
- Can be any length, but 30-120 seconds is recommended
- Landscape format

### App Store Promotional Art

```
iOS App Store:
  ┌──────────────────────────────────────┐
  │  Promotional Art (not required but   │
  │  used for featured placements)       │
  │  1024 x 1024 — App icon             │
  └──────────────────────────────────────┘

Google Play:
  ┌──────────────────────────────────────┐
  │  Feature Graphic (REQUIRED)          │
  │  1024 x 500 pixels                   │
  │  Shown at the top of your listing    │
  │  and in promotional placements       │
  └──────────────────────────────────────┘
  
  ┌──────────────────────────────────────┐
  │  High-res Icon                       │
  │  512 x 512 pixels                    │
  │  Used throughout the Play Store      │
  └──────────────────────────────────────┘
```

---

## 15.5 app.config.ts Deep Dive — Every Property That Matters

The `app.config.ts` (or `app.config.js`, or `app.json`) is the single source of truth for your Expo application. It controls everything from your app's name to its native permissions to its build configuration. Most developers only touch the properties they need for their current task and never read the full documentation. This section fixes that.

### Why app.config.ts Over app.json

Use `app.config.ts` (TypeScript) instead of `app.json` because:

1. **Dynamic values.** You can read environment variables, compute values, and conditionally include configuration.
2. **Type safety.** You get autocomplete and type checking.
3. **Comments.** JSON does not support comments. TypeScript does.
4. **Logic.** You can use `if` statements, functions, and imports.

```typescript
// app.config.ts — The basic structure
import { ExpoConfig, ConfigContext } from 'expo/config';

const IS_DEV = process.env.APP_VARIANT === 'development';
const IS_STAGING = process.env.APP_VARIANT === 'staging';
const IS_PROD = !IS_DEV && !IS_STAGING;

const getAppName = () => {
  if (IS_DEV) return 'Acme (Dev)';
  if (IS_STAGING) return 'Acme (Staging)';
  return 'Acme';
};

const getBundleId = () => {
  if (IS_DEV) return 'com.acme.app.dev';
  if (IS_STAGING) return 'com.acme.app.staging';
  return 'com.acme.app';
};

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: getAppName(),
  slug: 'acme-app',
  // ... rest of config
});
```

### The Complete Property Reference

#### Core Properties

| Property | What | Why | Trade-offs |
|----------|------|-----|-----------|
| `name` | Display name of your app on the home screen and in the stores | Users see this everywhere. Keep it short (iOS truncates at ~12 characters on home screen). | Changing after launch confuses existing users. |
| `slug` | URL-friendly identifier used by Expo services | Used for Expo project URLs, EAS project identification. Must be unique within your Expo account. | Cannot be changed after first publish. Choose carefully. |
| `version` | User-facing version string (e.g., "1.2.3") | Shown in app stores. Users see this in Settings and store listings. Follow semver. | Must be manually updated. Use CI to automate. |
| `orientation` | Supported orientations: `portrait`, `landscape`, `default` | `default` allows both. Most apps use `portrait` unless they are media or game apps. | Locking to portrait simplifies layout testing but limits tablet usability. |
| `icon` | Path to the 1024x1024 app icon PNG | The single source from which all platform-specific icon sizes are generated. | Use a separate `android.adaptiveIcon` for Android. |
| `scheme` | Custom URL scheme (e.g., `acme`) enabling `acme://` deep links | Required for deep linking, OAuth callbacks, and inter-app communication. | Each variant needs a unique scheme to avoid conflicts. |
| `userInterfaceStyle` | Theme mode: `light`, `dark`, `automatic` | `automatic` follows the device setting. Use this unless you have a strong reason not to. | If your app does not support dark mode yet, explicitly set `light`. |
| `platforms` | Array of supported platforms: `['ios', 'android', 'web']` | Controls which platforms Expo generates builds for. | Adding `web` means testing an entire additional platform. |

#### iOS Configuration

| Property | What | Why | Trade-offs |
|----------|------|-----|-----------|
| `ios.bundleIdentifier` | Unique identifier (e.g., `com.acme.app`) | Apple uses this to identify your app. Must be unique across the entire App Store. | Cannot be changed after first submission. Ever. Choose very carefully. |
| `ios.buildNumber` | Internal build number (e.g., "42") | Apple requires this to be incremented for each upload. Not shown to users. | Use `autoIncrement` in eas.json to handle this automatically. |
| `ios.supportsTablet` | Whether the app supports iPad | If `true`, the app appears as an iPad app in the store. If `false`, it runs in iPhone compatibility mode on iPad. | iPad support means testing on iPad screen sizes. But it significantly expands your audience. |
| `ios.requireFullScreen` | Disables iPad multitasking (Split View, Slide Over) | Some apps (games, video) do not work in split screen. | Setting this to `true` means your app cannot be used in Split View. Most apps should set this to `false`. |
| `ios.infoPlist` | Raw Info.plist overrides | Direct access to iOS configuration that Expo does not have dedicated properties for. | Use sparingly. Config plugins are usually a better option. |
| `ios.entitlements` | App entitlements (capabilities) | Required for Apple Sign In, Push Notifications, iCloud, App Groups, etc. | Each entitlement must be enabled in your Apple Developer account too. |
| `ios.associatedDomains` | Universal Links domains | Enables links like `https://acme.com/invite/123` to open your app directly. | Requires an `apple-app-site-association` file on your domain. |
| `ios.privacyManifests` | iOS Privacy Manifest (required since spring 2024) | Apple requires you to declare what data your app collects and why. | Must be accurate. Apple reviews this during app review. |
| `ios.config.usesNonExemptEncryption` | Whether the app uses encryption beyond HTTPS | If `false`, you skip the encryption export compliance questionnaire in App Store Connect. Most apps should set this to `false`. | Setting incorrectly can cause legal issues. |

```typescript
// app.config.ts — iOS configuration example
ios: {
  bundleIdentifier: getBundleId(),
  buildNumber: '1',
  supportsTablet: true,
  requireFullScreen: false,
  config: {
    usesNonExemptEncryption: false,
  },
  infoPlist: {
    NSCameraUsageDescription: 'We need camera access to scan documents.',
    NSPhotoLibraryUsageDescription: 'We need photo access to upload profile pictures.',
    NSLocationWhenInUseUsageDescription: 'We use your location to show nearby stores.',
    CFBundleAllowMixedLocalizations: true,
    ITSAppUsesNonExemptEncryption: false,
  },
  entitlements: {
    'aps-environment': 'production',
    'com.apple.developer.associated-domains': [
      'applinks:acme.com',
      'applinks:staging.acme.com',
    ],
  },
  associatedDomains: [
    'applinks:acme.com',
  ],
  privacyManifests: {
    NSPrivacyAccessedAPITypes: [
      {
        NSPrivacyAccessedAPIType: 'NSPrivacyAccessedAPICategoryUserDefaults',
        NSPrivacyAccessedAPITypeReasons: ['CA92.1'],
      },
    ],
  },
},
```

#### Android Configuration

| Property | What | Why | Trade-offs |
|----------|------|-----|-----------|
| `android.package` | Unique package name (e.g., `com.acme.app`) | Google Play uses this to identify your app. Must be unique across the Play Store. | Cannot be changed after first upload. Ever. |
| `android.versionCode` | Integer build number (e.g., 42) | Google Play requires this to increase with every upload. | Use `autoIncrement` in eas.json. |
| `android.adaptiveIcon` | Adaptive icon configuration (foreground, background, monochrome) | Required for modern Android. Without it, your icon gets a white square background on many launchers. | Need to design a separate foreground layer with proper safe zones. |
| `android.permissions` | Android permissions the app requests | Expo includes sensible defaults. Override to add or remove permissions. | Requesting unnecessary permissions hurts store conversion (users see the list before installing). |
| `android.blockedPermissions` | Permissions to explicitly exclude | Some libraries add permissions you do not need. Block them here. | May break features that depend on those permissions. |
| `android.intentFilters` | Deep link and URL handling configuration | Equivalent to iOS's `associatedDomains`. Configures which URLs open your app. | Requires Digital Asset Links verification on your domain. |
| `android.softwareKeyboardLayoutMode` | Keyboard behavior: `resize` or `pan` | `resize` shrinks the view. `pan` scrolls the view. `resize` is usually better. | Some layouts look bad when resized. Test with the keyboard open. |
| `android.allowBackup` | Whether the app data is included in device backups | `true` means users can restore app data when switching devices. | May expose sensitive data in backups. Set to `false` for financial apps. |
| `android.googleServicesFile` | Path to `google-services.json` | Required for Firebase, Google Sign-In, Google Maps, etc. | Must match the package name in the file. Different for each variant. |

```typescript
// app.config.ts — Android configuration example
android: {
  package: getBundleId(),
  versionCode: 1,
  adaptiveIcon: {
    foregroundImage: './assets/adaptive-icon-foreground.png',
    backgroundColor: '#1a1a2e',
    monochromeImage: './assets/adaptive-icon-monochrome.png',
  },
  permissions: [
    'CAMERA',
    'READ_MEDIA_IMAGES',
    'ACCESS_FINE_LOCATION',
    'POST_NOTIFICATIONS',
    'RECEIVE_BOOT_COMPLETED', // For scheduled notifications
  ],
  blockedPermissions: [
    'RECORD_AUDIO', // We don't need this, but a dependency requests it
  ],
  intentFilters: [
    {
      action: 'VIEW',
      autoVerify: true,
      data: [
        {
          scheme: 'https',
          host: 'acme.com',
          pathPrefix: '/invite',
        },
      ],
      category: ['BROWSABLE', 'DEFAULT'],
    },
  ],
  softwareKeyboardLayoutMode: 'resize',
  googleServicesFile: IS_DEV
    ? './google-services-dev.json'
    : IS_STAGING
      ? './google-services-staging.json'
      : './google-services.json',
},
```

#### Other Important Properties

| Property | What | Why | Trade-offs |
|----------|------|-----|-----------|
| `plugins` | Array of config plugins that modify native configuration | The primary way to configure native modules in CNG. See section 15.8. | Order matters. Some plugins conflict with each other. |
| `experiments` | Experimental Expo features | Enable new features before they are stable. | May break. May be removed. Do not depend on them in production without understanding the risk. |
| `extra` | Custom data accessible at runtime via `Constants.expoConfig.extra` | Pass environment-specific values (API URLs, feature flags) into your JS code. | Do not put secrets here. This data is embedded in the JS bundle and is not secure. |
| `updates` | EAS Update configuration (URL, check frequency, fallback) | Controls how OTA updates are fetched and applied. See Chapter 6. | Wrong config here can cause update loops or broken apps. |
| `owner` | Expo account that owns the project | Required for EAS services. Set this explicitly in team environments. | If not set, Expo uses the logged-in user, which breaks CI/CD. |
| `runtimeVersion` | Version compatibility marker for EAS Updates | Determines which OTA updates are compatible with which binary. | Use `{ policy: 'fingerprint' }` for automatic management. |
| `hooks` | Build lifecycle hooks (postPublish, postExport) | Run custom code after publishing or exporting. | Limited to specific lifecycle events. Config plugins are more powerful. |

```typescript
// app.config.ts — Other important properties
export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: getAppName(),
  slug: 'acme-app',
  version: '1.4.2',
  orientation: 'portrait',
  icon: './assets/icon.png',
  userInterfaceStyle: 'automatic',
  scheme: IS_DEV ? 'acme-dev' : IS_STAGING ? 'acme-staging' : 'acme',
  owner: 'acme-team',

  runtimeVersion: {
    policy: 'fingerprint', // Automatically track native changes
  },

  updates: {
    url: 'https://u.expo.dev/your-project-id',
    fallbackToCacheTimeout: 3000,
    checkAutomatically: 'ON_LOAD',
    // 'ON_LOAD' — check every app launch
    // 'ON_ERROR_RECOVERY' — only check after a crash
    // 'NEVER' — manual updates only
  },

  extra: {
    eas: {
      projectId: 'your-expo-project-id',
    },
    apiUrl: IS_DEV
      ? 'http://localhost:3000'
      : IS_STAGING
        ? 'https://staging-api.acme.com'
        : 'https://api.acme.com',
    enableDebugMode: IS_DEV,
    sentryDsn: 'https://examplePublicKey@sentry.io/0',
  },

  // iOS and Android configs from above...
  ios: { /* ... */ },
  android: { /* ... */ },

  plugins: [
    'expo-router',
    'expo-font',
    'expo-splash-screen',
    [
      'expo-notifications',
      {
        icon: './assets/notification-icon.png',
        color: '#1a1a2e',
      },
    ],
    [
      'expo-camera',
      {
        cameraPermission: 'Allow Acme to access your camera for document scanning.',
      },
    ],
    [
      'expo-location',
      {
        locationAlwaysAndWhenInUsePermission:
          'Allow Acme to use your location to find nearby stores.',
        isAndroidBackgroundLocationEnabled: false,
      },
    ],
    [
      '@sentry/react-native/expo',
      {
        organization: 'acme',
        project: 'acme-mobile',
      },
    ],
  ],
});
```

---

## 15.6 eas.json Deep Dive — Controlling Your Build Pipeline

The `eas.json` file controls how EAS Build compiles your app, how EAS Submit uploads it, and how build profiles relate to update channels. This is the file that makes the difference between a team that ships confidently and a team that prays after every build.

### The Complete eas.json Structure

```json
{
  "cli": {
    "version": ">= 14.0.0",
    "appVersionSource": "remote",
    "requireCommit": true
  },
  "build": {
    "base": {
      "node": "22.12.0",
      "env": {
        "SENTRY_AUTH_TOKEN": "...",
        "APP_ENV": "production"
      }
    },
    "development": {
      "extends": "base",
      "developmentClient": true,
      "distribution": "internal",
      "channel": "development",
      "env": {
        "APP_VARIANT": "development",
        "APP_ENV": "development"
      },
      "ios": {
        "simulator": false,
        "resourceClass": "m-medium"
      },
      "android": {
        "buildType": "apk",
        "resourceClass": "medium"
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
      "channel": "staging",
      "autoIncrement": true,
      "env": {
        "APP_VARIANT": "staging",
        "APP_ENV": "staging"
      },
      "ios": {
        "resourceClass": "m-medium"
      },
      "android": {
        "buildType": "apk",
        "resourceClass": "medium"
      }
    },
    "production": {
      "extends": "base",
      "distribution": "store",
      "channel": "production",
      "autoIncrement": true,
      "env": {
        "APP_VARIANT": "production",
        "APP_ENV": "production"
      },
      "ios": {
        "resourceClass": "m-large",
        "image": "macos-sequoia-15.4-xcode-16.3"
      },
      "android": {
        "buildType": "app-bundle",
        "resourceClass": "large"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "appleId": "team@acme.com",
        "ascAppId": "1234567890",
        "appleTeamId": "ABCDE12345"
      },
      "android": {
        "serviceAccountKeyPath": "./google-play-key.json",
        "track": "internal",
        "releaseStatus": "draft"
      }
    }
  }
}
```

### Build Property Reference

| Property | What | Why | Trade-offs |
|----------|------|-----|-----------|
| `extends` | Inherit from another profile | DRY configuration. Base profile has shared settings, specific profiles override. | Deeply nested extends can be confusing. Keep it to one level. |
| `developmentClient` | Build with expo-dev-client | Creates a debug build with the React Native dev menu, hot reloading, and ability to connect to local Metro bundler. | Not for distribution to non-developers. Larger binary. |
| `distribution` | `"internal"`, `"store"`, `"simulator"` | `internal` — ad-hoc/device-registered distribution. `store` — signed for App Store / Play Store. `simulator` — iOS simulator only. | `internal` requires device registration for iOS. |
| `channel` | The EAS Update channel this build listens to | Determines which OTA updates this build receives. `development` builds get dev updates, `production` builds get production updates. | Sending a broken update to the wrong channel is a production incident. |
| `autoIncrement` | Automatically increment buildNumber/versionCode | Eliminates the "forgot to bump the build number" problem. EAS tracks the latest number remotely. | Requires `appVersionSource: "remote"` in the `cli` section. |
| `resourceClass` | Build machine size: `"default"`, `"medium"`, `"large"` | Larger machines build faster. `large` is roughly 2x faster than `default`. | Larger machines cost more on paid Expo plans. |
| `image` | Specific build machine image | Pin to a specific Xcode version, macOS version, or Android SDK version. | Pinning prevents automatic updates. You must manually update when new Xcode versions are required. |
| `env` | Environment variables available during the build | Set API keys, feature flags, and variant identifiers that your `app.config.ts` reads. | Do not put secrets that end up in the JS bundle. Use EAS Secrets for truly sensitive values. |
| `cache` | Build caching configuration | Cache CocoaPods, Gradle dependencies, and node_modules between builds. | Cached builds are faster but may use stale dependencies. Disable cache when debugging build issues. |
| `node` | Node.js version for the build | Pin the Node version to match your development environment. | Older Node versions may not support your dependencies. |

#### iOS-Specific Build Properties

| Property | What | Why |
|----------|------|-----|
| `simulator` | Build for iOS Simulator | Simulator builds cannot be installed on physical devices. Use for local development. |
| `enterpriseProvisioning` | `"universal"` or `"adhoc"` | For Apple Enterprise accounts. `universal` works on any device (no registration). |
| `autoCredentials` | Let EAS manage provisioning profiles automatically | Set to `true` (default). EAS creates and manages provisioning profiles. Only set to `false` if you must use existing credentials. |
| `credentialsSource` | `"remote"` (EAS manages) or `"local"` (you provide) | `remote` is almost always correct. Use `local` only for enterprise or very specific credential requirements. |

#### Android-Specific Build Properties

| Property | What | Why |
|----------|------|-----|
| `buildType` | `"apk"` or `"app-bundle"` | `apk` for internal testing and direct device install. `app-bundle` for Play Store (required for new apps). |
| `gradleCommand` | Custom Gradle command | Override the default build command. Useful for custom build flavors. |
| `artifactPath` | Custom path to the output artifact | If your Gradle config outputs to a non-standard location. |

### Submit Property Reference

| Property | What | Why |
|----------|------|-----|
| `ios.appleId` | Apple ID email for App Store Connect | The account that manages your app in App Store Connect. |
| `ios.ascAppId` | App Store Connect app ID | The numeric ID of your app in App Store Connect (not the bundle ID). |
| `ios.appleTeamId` | Apple Developer Team ID | Required when the account belongs to multiple teams. |
| `android.serviceAccountKeyPath` | Path to Google Play service account JSON | Required for automated Play Store uploads. Create this in the Google Play Console under API access. |
| `android.track` | Play Store track: `"internal"`, `"alpha"`, `"beta"`, `"production"` | Start with `internal` for QA testing, then promote to `production`. |
| `android.releaseStatus` | `"draft"`, `"completed"`, `"halted"`, `"inProgress"` | `draft` requires manual release in the console. `completed` releases immediately. |
| `android.rollout` | Percentage for staged rollout (0.0 to 1.0) | Start with 0.1 (10% of users), monitor, then increase. Only valid with `inProgress` status. |

---

## 15.7 Environment-Specific Config — One Codebase, Three Apps

### The Problem

You need three versions of your app:
1. **Development** — connects to local or dev API, has debug tools, uses a different icon so you know which app is which
2. **Staging** — connects to staging API, used for QA testing, has a distinct icon and name
3. **Production** — the real app, connects to production API, submitted to the stores

Without environment-specific config, you either:
- Constantly edit your config file when switching environments (error-prone, guaranteed to cause a production incident eventually)
- Use a single bundle ID and overwrite the previous version when installing a different environment (cannot test side-by-side)

### The Solution: APP_VARIANT

Set an `APP_VARIANT` environment variable that your `app.config.ts` reads to generate the correct configuration:

```typescript
// app.config.ts — Environment-aware configuration
import { ExpoConfig, ConfigContext } from 'expo/config';

const APP_VARIANT = process.env.APP_VARIANT || 'production';

// Variant-specific overrides
const VARIANT_CONFIG = {
  development: {
    name: 'Acme (Dev)',
    bundleId: 'com.acme.app.dev',
    scheme: 'acme-dev',
    icon: './assets/icons/icon-dev.png',
    adaptiveIconForeground: './assets/icons/adaptive-dev-foreground.png',
    apiUrl: 'http://localhost:3000',
    googleServicesFile: './config/google-services-dev.json',
    googleServicesFileIos: './config/GoogleService-Info-dev.plist',
  },
  staging: {
    name: 'Acme (Staging)',
    bundleId: 'com.acme.app.staging',
    scheme: 'acme-staging',
    icon: './assets/icons/icon-staging.png',
    adaptiveIconForeground: './assets/icons/adaptive-staging-foreground.png',
    apiUrl: 'https://staging-api.acme.com',
    googleServicesFile: './config/google-services-staging.json',
    googleServicesFileIos: './config/GoogleService-Info-staging.plist',
  },
  production: {
    name: 'Acme',
    bundleId: 'com.acme.app',
    scheme: 'acme',
    icon: './assets/icons/icon.png',
    adaptiveIconForeground: './assets/icons/adaptive-foreground.png',
    apiUrl: 'https://api.acme.com',
    googleServicesFile: './config/google-services.json',
    googleServicesFileIos: './config/GoogleService-Info.plist',
  },
} as const;

type VariantKey = keyof typeof VARIANT_CONFIG;
const variant = VARIANT_CONFIG[APP_VARIANT as VariantKey] || VARIANT_CONFIG.production;

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: variant.name,
  slug: 'acme-app',
  version: '1.4.2',
  icon: variant.icon,
  scheme: variant.scheme,

  ios: {
    bundleIdentifier: variant.bundleId,
    googleServicesFile: variant.googleServicesFileIos,
    // ... other iOS config
  },

  android: {
    package: variant.bundleId,
    googleServicesFile: variant.googleServicesFile,
    adaptiveIcon: {
      foregroundImage: variant.adaptiveIconForeground,
      backgroundColor: '#1a1a2e',
    },
    // ... other Android config
  },

  extra: {
    eas: { projectId: 'your-project-id' },
    apiUrl: variant.apiUrl,
    appVariant: APP_VARIANT,
  },

  // ... plugins, updates, etc.
});
```

### Variant-Specific Icons

Create distinct icons for each variant so you can identify them at a glance on your device:

```
assets/
  icons/
    icon.png                    ← Production (clean, final design)
    icon-staging.png            ← Staging (same design with "S" badge)
    icon-dev.png                ← Dev (same design with "D" badge or orange tint)
    adaptive-foreground.png     ← Production adaptive
    adaptive-staging-foreground.png ← Staging adaptive
    adaptive-dev-foreground.png ← Dev adaptive
```

A common pattern is to take your production icon and add a colored banner in the corner:

```
Production:        Staging:           Dev:
┌──────────┐      ┌──────────┐      ┌──────────┐
│          │      │       ┌──│      │       ┌──│
│   LOGO   │      │  LOGO │S ││      │  LOGO │D ││
│          │      │       └──│      │       └──│
│          │      │  (yellow)│      │  (orange)│
└──────────┘      └──────────┘      └──────────┘
```

### Side-by-Side Installation

Because each variant has a unique bundle ID, you can install all three on the same device simultaneously:

```bash
# Build and install development variant
APP_VARIANT=development eas build --profile development --platform ios

# Build and install staging variant
APP_VARIANT=staging eas build --profile preview --platform ios

# The production app is already installed from the App Store

# All three appear as separate apps on the home screen:
# "Acme (Dev)"  "Acme (Staging)"  "Acme"
```

### Accessing Config at Runtime

```typescript
// lib/config.ts — Runtime access to config values
import Constants from 'expo-constants';

export const config = {
  apiUrl: Constants.expoConfig?.extra?.apiUrl as string,
  appVariant: Constants.expoConfig?.extra?.appVariant as string,
  isProduction: Constants.expoConfig?.extra?.appVariant === 'production',
  isDevelopment: Constants.expoConfig?.extra?.appVariant === 'development',
  isStaging: Constants.expoConfig?.extra?.appVariant === 'staging',
  sentryDsn: Constants.expoConfig?.extra?.sentryDsn as string,
  easProjectId: Constants.expoConfig?.extra?.eas?.projectId as string,
};

// Usage
import { config } from '../lib/config';

const response = await fetch(`${config.apiUrl}/users/me`);

if (config.isDevelopment) {
  // Show debug tools
}
```

### Environment Variables vs. APP_VARIANT

Understand the difference:

| Mechanism | When It Runs | What It Affects | Example |
|-----------|-------------|-----------------|---------|
| `APP_VARIANT` in `app.config.ts` | At prebuild time | Native configuration (bundle ID, app name, icon) | Different app icons per environment |
| `process.env` in `app.config.ts` | At prebuild time | Config values, plugins, native config | Different Google Services files per environment |
| `extra` in app.config.ts | At runtime | Values accessible via `Constants.expoConfig.extra` | API URL, feature flags |
| EAS Secrets | At build time only | Not in the JS bundle | API keys for build-time services (Sentry auth token) |
| `.env` files (with expo-env) | At bundling time | Values inlined into JS bundle | Public API keys, analytics IDs |

---

## 15.8 Config Plugins — Modifying Native Code Without Maintaining It

### How Config Plugins Work

Config plugins are the mechanism by which Expo's Continuous Native Generation (CNG) modifies native project files. When you run `npx expo prebuild`, Expo:

1. Generates the `ios/` and `android/` directories from scratch
2. Applies your `app.config.ts` settings
3. Runs every config plugin in the `plugins` array, in order
4. Each plugin can modify native files (Info.plist, AndroidManifest.xml, Podfile, build.gradle, Entitlements, etc.)

```
┌─────────────────┐
│  app.config.ts   │
│                  │
│  plugins: [      │
│    pluginA,      │──── Plugin A modifies Info.plist
│    pluginB,      │──── Plugin B adds a Gradle dependency
│    pluginC,      │──── Plugin C modifies AndroidManifest.xml
│  ]               │
└─────────────────┘
         │
         ▼
┌─────────────────┐     ┌──────────────────┐
│  expo prebuild   │────▶│  Generated ios/   │
│                  │     │  Generated android│
│  Base template   │     │  + all plugin     │
│  + plugins       │     │    modifications  │
└─────────────────┘     └──────────────────┘
```

### Common Config Plugins

| Plugin | What It Does | When You Need It |
|--------|-------------|-----------------|
| `expo-notifications` | Configures push notification entitlements, Android notification icons and colors | Any app with push notifications |
| `expo-location` | Sets location permission descriptions, enables background location | Location-based features |
| `expo-camera` | Sets camera permission descriptions | Camera access |
| `expo-media-library` | Sets photo library permission descriptions | Photo/video access |
| `expo-sensors` | Configures motion sensor permissions | Pedometer, accelerometer |
| `expo-build-properties` | Sets native build properties (Kotlin version, compileSdkVersion, deployment target) | Fixing native build issues, targeting specific platform versions |
| `@sentry/react-native/expo` | Configures Sentry source maps, native crash reporting | Error monitoring |
| `expo-apple-authentication` | Adds Sign in with Apple entitlement | Apple Sign In |
| `expo-dev-client` | Configures development client build | Development builds |
| `expo-router` | Configures deep linking scheme and associated domains | File-based routing |

### Using Plugins in app.config.ts

```typescript
// app.config.ts — Plugin configuration
plugins: [
  // Plugin with no options — just a string
  'expo-router',

  // Plugin with options — array of [plugin, options]
  [
    'expo-notifications',
    {
      icon: './assets/notification-icon.png',
      color: '#1a1a2e',
      sounds: ['./assets/sounds/notification.wav'],
    },
  ],

  // expo-build-properties — fixing native build issues
  [
    'expo-build-properties',
    {
      ios: {
        deploymentTarget: '15.1',
        useFrameworks: 'static',
        flipper: false,
      },
      android: {
        compileSdkVersion: 35,
        targetSdkVersion: 35,
        buildToolsVersion: '35.0.0',
        kotlinVersion: '1.9.25',
        enableProguardInReleaseBuilds: true,
        enableShrinkResourcesInReleaseBuilds: true,
      },
    },
  ],

  // Custom plugin defined inline (for simple modifications)
  [
    'expo-build-properties',
    {
      android: {
        enableProguardInReleaseBuilds: true,
      },
    },
  ],
],
```

### Writing a Simple Custom Plugin

Sometimes you need to modify native configuration in a way that no existing plugin handles. Writing a basic config plugin is simpler than you might think:

```typescript
// plugins/withAndroidSplashFullscreen.ts
import {
  ConfigPlugin,
  withAndroidStyles,
  AndroidConfig,
} from 'expo/config-plugins';

/**
 * Makes the Android splash screen truly fullscreen
 * by hiding the navigation bar and status bar.
 */
const withAndroidSplashFullscreen: ConfigPlugin = (config) => {
  return withAndroidStyles(config, (config) => {
    const styles = config.modResults;

    // Find or create the AppTheme style
    const appTheme = styles.resources.style?.find(
      (style: any) => style.$.name === 'AppTheme'
    );

    if (appTheme) {
      // Add fullscreen flags
      const items = appTheme.item || [];

      // Check if item already exists before adding
      if (!items.find((item: any) => item.$.name === 'android:windowFullscreen')) {
        items.push({
          $: { name: 'android:windowFullscreen' },
          _: 'true',
        });
      }

      appTheme.item = items;
    }

    return config;
  });
};

export default withAndroidSplashFullscreen;
```

```typescript
// plugins/withInfoPlistAddition.ts
import { ConfigPlugin, withInfoPlist } from 'expo/config-plugins';

/**
 * Add custom Info.plist entries.
 * Useful when you need to set keys that Expo doesn't have
 * a dedicated config property for.
 */
const withInfoPlistAddition: ConfigPlugin<Record<string, any>> = (
  config,
  plistEntries
) => {
  return withInfoPlist(config, (config) => {
    for (const [key, value] of Object.entries(plistEntries)) {
      config.modResults[key] = value;
    }
    return config;
  });
};

export default withInfoPlistAddition;
```

```typescript
// plugins/withAndroidManifestAddition.ts
import {
  ConfigPlugin,
  withAndroidManifest,
  AndroidConfig,
} from 'expo/config-plugins';

/**
 * Add a custom <queries> entry to AndroidManifest.xml
 * Required for deep linking to specific apps on Android 11+.
 */
const withAndroidQueries: ConfigPlugin<string[]> = (config, packages) => {
  return withAndroidManifest(config, (config) => {
    const manifest = config.modResults.manifest;

    if (!manifest.queries) {
      manifest.queries = [];
    }

    // Add package queries for each specified package
    const packageEntries = packages.map((pkg) => ({
      package: [{ $: { 'android:name': pkg } }],
    }));

    manifest.queries.push(...packageEntries);

    return config;
  });
};

export default withAndroidQueries;
```

Using your custom plugins:

```typescript
// app.config.ts
import withAndroidSplashFullscreen from './plugins/withAndroidSplashFullscreen';
import withInfoPlistAddition from './plugins/withInfoPlistAddition';
import withAndroidQueries from './plugins/withAndroidManifestAddition';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  // ...
  plugins: [
    // Built-in plugins first
    'expo-router',
    'expo-font',

    // Your custom plugins
    withAndroidSplashFullscreen,
    [
      withInfoPlistAddition,
      {
        LSApplicationQueriesSchemes: ['instagram', 'twitter', 'fb'],
      },
    ],
    [
      withAndroidQueries,
      ['com.instagram.android', 'com.twitter.android'],
    ],
  ],
});
```

### Plugin Development Tips

1. **Start by reading existing plugins.** The Expo source code for built-in plugins is the best reference. Look at how `expo-notifications` or `expo-camera` implement their config plugins.

2. **Use the right mod.** Each native file has a corresponding "mod" function:
   - `withInfoPlist` — iOS Info.plist
   - `withEntitlementsPlist` — iOS Entitlements
   - `withPodfile` — iOS Podfile
   - `withAppDelegate` — iOS AppDelegate
   - `withAndroidManifest` — Android AndroidManifest.xml
   - `withAndroidStyles` — Android styles.xml
   - `withAndroidColors` — Android colors.xml
   - `withMainActivity` — Android MainActivity
   - `withMainApplication` — Android MainApplication
   - `withProjectBuildGradle` — Android project-level build.gradle
   - `withAppBuildGradle` — Android app-level build.gradle

3. **Test with `npx expo prebuild --clean`.** This regenerates native directories from scratch so you can verify your plugin's output.

4. **Plugins run in order.** If plugin B depends on changes made by plugin A, put A first in the array.

5. **Do not modify files outside the mod system.** If you directly read/write files in `ios/` or `android/`, your changes may be overwritten by other plugins or by `expo prebuild`.

---

## 15.9 The Complete Config Checklist

Before you consider your app's configuration done, run through this:

### Icons
- [ ] 1024x1024 PNG for primary icon (no transparency for iOS)
- [ ] Android adaptive icon foreground (1024x1024, transparent background, logo in center 66%)
- [ ] Android adaptive icon background color (or image)
- [ ] Android 13+ monochrome icon (optional but recommended)
- [ ] iOS dark mode icon variant (iOS 18+, optional but recommended)
- [ ] Dynamic app icons configured (if applicable)
- [ ] Each APP_VARIANT has a visually distinct icon

### Splash Screen
- [ ] Splash image configured with appropriate background color
- [ ] Dark mode splash variant configured
- [ ] `SplashScreen.preventAutoHideAsync()` called at app root
- [ ] Splash hidden only after fonts, auth, and critical data are ready
- [ ] No white flash between splash and first screen

### Assets
- [ ] Fonts preloaded before first render
- [ ] Critical images preloaded during splash
- [ ] Images optimized (PNG/JPEG compression, appropriate format)
- [ ] Bundle size measured and under budget

### app.config.ts
- [ ] Using `.ts` not `.json` (dynamic values, type safety)
- [ ] `bundleIdentifier` and `package` set correctly per variant
- [ ] `version` follows semver, updated for releases
- [ ] `scheme` is unique per variant (for deep linking)
- [ ] Permission descriptions are user-friendly and specific
- [ ] `runtimeVersion` uses `fingerprint` policy
- [ ] `extra` contains environment-specific runtime values (API URL, feature flags)
- [ ] No secrets in `extra` (use EAS Secrets instead)
- [ ] `owner` set for team projects

### eas.json
- [ ] Build profiles for development, preview, and production
- [ ] `autoIncrement` enabled for preview and production
- [ ] `channel` set correctly for each profile (maps to EAS Update channels)
- [ ] `resourceClass` appropriate for build speed needs
- [ ] Submit profiles configured for App Store and Google Play
- [ ] `appVersionSource: "remote"` set in cli section

### Environment Config
- [ ] APP_VARIANT controls name, bundle ID, icon, and API URL
- [ ] All three variants can be installed side-by-side on one device
- [ ] Google Services files are per-variant
- [ ] Deep link scheme is per-variant

---

## Summary

Configuration is not glamorous, but it is the foundation everything else sits on. A misconfigured app icon makes your app look unprofessional. A missing splash screen configuration causes a white flash that makes your app feel broken. A single bundle ID for all environments means your QA team cannot test staging without losing the production app.

The 100x architect treats configuration as infrastructure. They set it up once, set it up right, and then never think about it again — because the `app.config.ts` reads the environment, generates the right values, and produces three distinct app variants from a single codebase. The `eas.json` builds them correctly, signs them correctly, and submits them correctly.

Do not skip this chapter's checklist. Every item on it exists because someone, somewhere, shipped a broken build because they did not check it. Learn from their pain.

---

*Next up: Chapter 16 covers Design Systems — building a component library that scales across your entire organization.*
