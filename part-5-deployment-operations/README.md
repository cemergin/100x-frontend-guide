# Part V — Deployment & Operations

> Shipping, monitoring, securing, and keeping your app alive at scale.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|-------|-----------|------------|
| [22](./20-monitoring.md) | Mobile Monitoring & Observability | 3656 | Intermediate to Advanced | Sentry, Crashlytics, crash-free rate, ANR, source maps, custom traces, RUM, dashboards, alerts, performance monitoring, error boundaries |
| [23](./21-firebase.md) | Firebase Console Mastery | 3162 | Intermediate | Crashlytics, Analytics, Performance Monitoring, Remote Config, Cloud Messaging, App Distribution, @react-native-firebase |
| [24](./22-security.md) | Security & Data Protection | 2464 | Intermediate | secure storage, API keys, certificate pinning, biometrics, OWASP Mobile Top 10, OAuth2 PKCE, token management, code obfuscation, ProGuard, jailbreak detection |
| [25](./23-metrics.md) | Mobile Metrics That Matter | 1399 | Intermediate | TTI, cold start, FPS, jank measurement, memory usage, network latency, app size, Play Console vitals, ANR rate, Datadog RUM |
| [26](./24-payments.md) | Payments & Money Handling — Stripe, IAP & Revenue | 3195 | Intermediate to Advanced | Stripe, React Native Stripe SDK, Payment Intents, Subscriptions, Apple Pay, Google Pay, In-App Purchases, RevenueCat, webhooks, PCI compliance |

## Reading Order

1. **Ch 22** (Monitoring) is the foundation — read it first.
2. **Ch 23** (Firebase) extends monitoring with Firebase-specific services and pairs naturally with Ch 22.
3. **Ch 25** (Metrics) deepens the measurement concepts from Ch 22 with platform-specific metrics.
4. **Ch 24** (Security) is independent and can be read at any point.
5. **Ch 26** (Payments) is the most self-contained chapter — read it when you need to integrate payments.

## Prerequisites

- **Ch 22**: Chapters 1, 6
- **Ch 23**: Chapter 22
- **Ch 24**: Chapters 6, 7
- **Ch 25**: Chapter 22
- **Ch 26**: Chapters 6, 7, 24

## What You'll Be Able to Do After This Part

- Set up dual monitoring with Sentry and Crashlytics, including source map uploads, error boundaries, and alert thresholds.
- Instrument Firebase services (Crashlytics, Analytics, Remote Config, FCM, App Distribution) in a production React Native app.
- Implement defense-in-depth mobile security: secure storage, OAuth2 PKCE, certificate pinning, biometrics, and OWASP Top 10 mitigations.
- Define and track the mobile metrics that actually matter (crash-free rate, cold start, FPS, ANR rate, memory) with real dashboards and alert rules.
- Integrate Stripe payments on both mobile (Payment Sheet, Apple Pay, Google Pay) and web (Stripe Elements), with proper webhook handling, idempotency, and PCI compliance.
- Manage subscriptions and in-app purchases through RevenueCat.
- Run staged rollouts, hotfix processes, and post-mortems with operational discipline.
