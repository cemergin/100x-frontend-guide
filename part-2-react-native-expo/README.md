# Part II — React Native & Expo

> Building mobile apps that feel native, deploy cleanly, and don't crash.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| 6 | [Expo: The Modern React Native Platform](./05-expo-platform.md) | 2,260 | Beginner to Intermediate | Expo SDK 54, CNG, prebuild, config plugins, Expo Modules API, managed vs bare, React Compiler, expo-dev-client |
| 7 | [EAS: Build, Submit, Update — The Complete Deployment Pipeline](./06-eas-mastery.md) | 2,877 | Intermediate to Advanced | EAS Build, EAS Submit, EAS Update, OTA, channels, branches, fingerprinting, eas.json, build profiles, credentials, code signing |
| 8 | [Navigation Architecture](./07-navigation.md) | 1,373 | Intermediate | Expo Router v4, React Navigation 7, deep linking, type safety, lazy loading, prefetching, auth routing |
| 9 | [Styling & Animation](./08-styling-animation.md) | 3,180 | Intermediate | NativeWind, Tamagui, Unistyles, Reanimated v3, Gesture Handler, expo-image, design tokens |
| 10 | [App Lifecycle, Background Tasks & Push Notifications](./10-app-lifecycle.md) | 4,203 | Intermediate to Advanced | AppState, splash screen, background fetch, background processing, push notifications, FCM, APNs, Expo Notifications, local notifications |
| 11 | [Deep Linking & Universal Links](./11-deep-linking.md) | 2,619 | Intermediate | Universal Links, App Links, deferred deep links, Expo Router deep linking, URL schemes, QR codes, attribution, AASA file, Digital Asset Links |
| 12 | [Storage, Persistence & File System](./12-storage-persistence.md) | 2,889 | Intermediate | MMKV, SQLite, expo-sqlite, drizzle-orm, WatermelonDB, Realm, expo-file-system, document picker, downloads, encrypted storage, migration patterns |
| 13 | [Device APIs & Native Features](./13-device-apis.md) | 5,205 | Intermediate | Camera, location, maps, contacts, calendar, haptics, clipboard, share sheet, barcode scanning, WebView, sensors, NFC, Bluetooth, media playback |
| 14 | [Permissions, Accessibility & Internationalization](./14-permissions-a11y-i18n.md) | 3,153 | Intermediate | Permissions, VoiceOver, TalkBack, semantic roles, focus management, a11y testing, react-i18next, expo-localization, RTL, pluralization |
| 15 | [App Icons, Splash Screens, Assets & Config Deep Dive](./15-app-assets-config.md) | 1,794 | Beginner to Intermediate | Adaptive icons, dynamic app icons, splash screen config, asset generation, app.config.ts deep dive, eas.json deep dive, expo plugins, environment-specific config |
| 40 | [Micro-Interactions & Motion Design Cookbook](./40-micro-interactions.md) | 4,171 | Intermediate to Advanced | Shared element transitions, skeleton loaders, pull-to-refresh, swipe actions, bottom sheets, toast animations, spring physics, gesture-driven interfaces, Reanimated recipes, Moti |

## Reading Order

1. **Chapter 6** (Expo Platform) first -- it establishes the project foundation everything else builds on.
2. **Chapter 7** (EAS) next -- deployment pipeline knowledge informs how you structure your app.
3. **Chapter 8** (Navigation) and **Chapter 9** (Styling & Animation) can be read in either order.
4. **Chapter 10** (App Lifecycle) after Chapters 6-7 -- covers AppState, background tasks, and push notifications.
5. **Chapter 11** (Deep Linking) after Chapters 6 and 8 -- builds on navigation knowledge.
6. **Chapter 12** (Storage) and **Chapter 13** (Device APIs) can be read in either order after Chapter 6.
7. **Chapter 14** (Permissions, A11y, i18n) after Chapter 6 -- cross-cutting concerns for every feature.
8. **Chapter 15** (Assets & Config) after Chapter 6 -- polishing your app for store submission.
9. **Chapter 40** (Micro-Interactions) last -- builds on Chapter 9's animation foundations.

## Prerequisites

- Chapter 1 (React Native Architecture & Internals) is required for Chapters 6 and 9.
- Chapter 6 (Expo Platform) is required for Chapters 7, 8, 10-15.
- Chapter 8 (Navigation) is helpful for Chapter 11 (Deep Linking).
- Chapter 9 (Styling & Animation) is required for Chapter 40 (Micro-Interactions).

## What You'll Be Able to Do After This Part

- Set up and configure Expo projects using Continuous Native Generation and config plugins.
- Build, sign, submit, and OTA-update mobile apps through EAS with repeatable pipelines.
- Design a navigation architecture with type-safe routes, deep linking, and lazy loading.
- Build responsive, performant UIs with 60fps animations running on the native thread.
- Implement push notifications, background tasks, and app lifecycle management.
- Integrate device APIs (camera, location, contacts, sensors) with proper permission handling.
- Build accessible, internationalized apps that work for everyone.
- Create polished micro-interactions and motion design that make your app feel premium.
