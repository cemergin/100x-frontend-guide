# Part V — Deployment & Operations

> Shipping, monitoring, securing, and keeping your app alive at scale.

## Chapters

| Ch | Title | Lines | Difficulty | Key Topics |
|----|-------|------:|-----------|------------|
| 22 | [Mobile Monitoring & Observability](./20-monitoring.md) | 3,667 | Intermediate to Advanced | Sentry, Crashlytics, crash-free rate, ANR, source maps, custom traces, RUM, dashboards, alerts, performance monitoring, error boundaries |
| 23 | [Firebase Console Mastery](./21-firebase.md) | 3,173 | Intermediate | Crashlytics, Analytics, Performance Monitoring, Remote Config, Cloud Messaging, App Distribution, @react-native-firebase |
| 24 | [Security & Data Protection](./22-security.md) | 2,475 | Intermediate | Secure storage, API keys, certificate pinning, biometrics, OWASP Mobile Top 10, OAuth2 PKCE, token management, code obfuscation, ProGuard, jailbreak detection |
| 25 | [Mobile Metrics That Matter](./23-metrics.md) | 1,862 | Intermediate | TTI, FPS, memory, Play Console vitals, App Store Connect, Datadog RUM, dashboards, alerts |
| 26 | [Payments & Money Handling — Stripe, IAP & Revenue](./24-payments.md) | 3,206 | Intermediate to Advanced | Stripe, React Native Stripe SDK, Payment Intents, Subscriptions, Apple Pay, Google Pay, In-App Purchases, RevenueCat, webhooks, PCI compliance |
| 33 | [Authentication & Authorization — From Login Screen to RBAC](./33-authentication.md) | 4,679 | Intermediate to Advanced | OAuth2, PKCE, JWT, session management, refresh tokens, Clerk, Stack Auth, Supabase Auth, Firebase Auth, Auth.js, passkeys, biometric auth, RBAC, MFA, magic links |
| 37 | [Feature Flags, A/B Testing & Experimentation](./37-feature-flags.md) | 2,971 | Intermediate | LaunchDarkly, Statsig, Unleash, Vercel feature flags, Firebase Remote Config, A/B testing, gradual rollouts, kill switches, experiment-driven development |
| 39 | [Email, Transactional Comms & In-App Messaging](./39-email-comms.md) | 3,420 | Intermediate | Resend, React Email, SendGrid, email templates, transactional email, SMTP, email deliverability, in-app messaging, Knock, Novu, notification center |
| 43 | [Analytics, Attribution & Data-Driven Development](./43-analytics-attribution.md) | 3,816 | Intermediate | Firebase Analytics, Mixpanel, Amplitude, PostHog, Segment, event taxonomy, funnels, cohorts, retention, attribution, App Tracking Transparency, GDPR consent |
| 45 | [Release Management — Versioning, Changelogs & Ship Cadence](./45-release-management.md) | 2,278 | Intermediate | Semantic versioning, Changesets, release branches, staged rollouts, changelogs, release notes, app store versioning, OTA vs binary releases, hotfix process, monorepo releases |

## Reading Order

1. **Ch 22** (Monitoring) is the foundation -- read it first.
2. **Ch 23** (Firebase) extends monitoring with Firebase-specific services and pairs naturally with Ch 22.
3. **Ch 25** (Metrics) deepens the measurement concepts from Ch 22 with platform-specific metrics.
4. **Ch 24** (Security) is independent and can be read at any point.
5. **Ch 26** (Payments) is self-contained -- read it when you need to integrate payments.
6. **Ch 33** (Authentication) after Chapters 6, 22, 27 -- covers the full auth stack from login to RBAC.
7. **Ch 37** (Feature Flags) after Chapters 19, 22 -- gradual rollouts and experimentation.
8. **Ch 39** (Email & Comms) after Chapters 27, 34 -- transactional email and in-app messaging.
9. **Ch 43** (Analytics) after Chapters 21, 25 -- event tracking, funnels, and attribution.
10. **Ch 45** (Release Management) after Chapters 7, 19 -- versioning and ship cadence for mobile and web.

## Prerequisites

- **Ch 22**: Chapters 1, 5
- **Ch 23**: Chapter 22
- **Ch 24**: Chapters 5, 6
- **Ch 25**: Chapter 22
- **Ch 26**: Chapters 5, 6, 24
- **Ch 33**: Chapters 6, 22, 27
- **Ch 37**: Chapters 19, 22
- **Ch 39**: Chapters 27, 34
- **Ch 43**: Chapters 23, 25
- **Ch 45**: Chapters 7, 19

## What You'll Be Able to Do After This Part

- Set up dual monitoring with Sentry and Crashlytics, including source map uploads, error boundaries, and alert thresholds.
- Instrument Firebase services (Crashlytics, Analytics, Remote Config, FCM, App Distribution) in a production React Native app.
- Implement defense-in-depth mobile security: secure storage, OAuth2 PKCE, certificate pinning, biometrics, and OWASP Top 10 mitigations.
- Define and track the mobile metrics that actually matter (crash-free rate, cold start, FPS, ANR rate, memory) with real dashboards.
- Integrate Stripe payments on both mobile and web, with proper webhook handling, idempotency, and PCI compliance.
- Build a complete authentication system with OAuth2 PKCE, passkeys, RBAC, and multi-provider support.
- Run feature flag-driven development with gradual rollouts, A/B testing, and kill switches.
- Set up transactional email with React Email and Resend, plus in-app notification centers.
- Implement privacy-first analytics with proper event taxonomy, attribution, and consent management.
- Manage releases with semantic versioning, changelogs, staged rollouts, and hotfix processes.
