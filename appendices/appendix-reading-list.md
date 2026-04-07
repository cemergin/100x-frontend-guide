<!--
  TYPE: appendix
  TITLE: Reading List
  APPENDIX: C
  UPDATED: 2026-04-07
-->

# Appendix C: Reading List

> 50+ essential articles, talks, papers, and tools — curated for the 100x frontend architect.

Each entry includes a one-line summary and a link. Organized by topic, ordered by impact within each section.

---

## Architecture & Patterns

1. **"Scaling React Native at Shopify"** — Shopify Engineering
   How Shopify rebuilt Shop in React Native, including monorepo structure, performance budgets, and team scaling.
   [shopify.engineering/react-native-future-mobile-shopify](https://shopify.engineering/react-native-future-mobile-shopify)

2. **"How Discord Stores Trillions of Messages"** — Discord Engineering
   Real-world data architecture at scale — migration from Cassandra to ScyllaDB and the lessons learned.
   [discord.com/blog/how-discord-stores-trillions-of-messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)

3. **"The Composable Architecture"** — Point-Free
   Functional architecture for building apps with state management, side effects, and testing baked in.
   [github.com/pointfreeco/swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture)

4. **"Feature-Sliced Design"** — FSD Documentation
   Architectural methodology for organizing frontend code by feature slices rather than technical layers.
   [feature-sliced.design](https://feature-sliced.design/)

5. **"Patterns.dev"** — Lydia Hallie & Addy Osmani
   Comprehensive reference of rendering patterns, design patterns, and performance patterns for modern web apps.
   [patterns.dev](https://www.patterns.dev/)

---

## React Native & Expo

6. **"The New Architecture"** — React Native Documentation
   Official guide to JSI, Fabric, TurboModules, and Codegen — the foundational architecture shift.
   [reactnative.dev/docs/the-new-architecture/landing-page](https://reactnative.dev/docs/the-new-architecture/landing-page)

7. **"Expo CNG (Continuous Native Generation)"** — Expo Documentation
   How Expo generates native projects on demand, enabling managed workflow with full native access.
   [docs.expo.dev/workflow/continuous-native-generation](https://docs.expo.dev/workflow/continuous-native-generation/)

8. **"EAS Update: How It Works"** — Expo Documentation
   Deep dive into OTA updates — branches, channels, fingerprinting, and rollback strategies.
   [docs.expo.dev/eas-update/how-it-works](https://docs.expo.dev/eas-update/how-it-works/)

9. **"Performance in React Native"** — Callstack Blog
   Practical guide to identifying and fixing performance bottlenecks in React Native apps, from the team behind Reanimated.
   [callstack.com/blog/performance-in-react-native](https://www.callstack.com/blog/react-native-performance-guide)

10. **"Building a 60fps React Native App"** — William Candillon (YouTube)
    Video series on building smooth animations with Reanimated and Gesture Handler.
    [youtube.com/@wcandillon](https://www.youtube.com/@wcandillon)

11. **"React Native at Microsoft"** — Microsoft DevBlogs
    How Microsoft ships React Native in Office, Teams, Xbox, and Windows — scale, architecture, and native integration.
    [devblogs.microsoft.com/react-native](https://devblogs.microsoft.com/react-native/)

12. **"Hermes: A New JavaScript Engine Optimized for React Native"** — Meta Engineering
    The design decisions behind Hermes: AOT compilation, bytecode format, and mobile GC optimization.
    [hermesengine.dev](https://hermesengine.dev/)

13. **"Expo Config Plugins"** — Expo Documentation
    How to write config plugins that modify native project files during prebuild, enabling CNG customization.
    [docs.expo.dev/config-plugins/introduction](https://docs.expo.dev/config-plugins/introduction/)

14. **"FlashList: Fast & Performant React Native List"** — Shopify
    Shopify's drop-in FlatList replacement with cell recycling for up to 10x better list performance.
    [shopify.github.io/flash-list](https://shopify.github.io/flash-list/)

---

## TypeScript

15. **"The TypeScript Handbook"** — TypeScript Documentation
    The canonical resource for TypeScript. Start here, return here.
    [typescriptlang.org/docs/handbook](https://www.typescriptlang.org/docs/handbook/intro.html)

16. **"Type Challenges"** — Anthony Fu
    Collection of TypeScript type-system puzzles from easy to extreme. The best way to internalize advanced types.
    [github.com/type-challenges/type-challenges](https://github.com/type-challenges/type-challenges)

17. **"Effective TypeScript: 83 Specific Ways to Improve Your TypeScript"** — Dan Vanderkam
    The most practical TypeScript book. Every chapter is a concrete, actionable improvement.
    [effectivetypescript.com](https://effectivetypescript.com/)

18. **"Total TypeScript"** — Matt Pocock
    Video courses and tips covering TypeScript from basics to extreme generics. The best modern TS educator.
    [totaltypescript.com](https://www.totaltypescript.com/)

19. **"Zod: TypeScript-First Schema Validation"** — Colin McDonnell
    Runtime validation with static type inference. The bridge between runtime and compile-time safety.
    [zod.dev](https://zod.dev/)

---

## Performance

20. **"web.dev Performance"** — Google Chrome Team
    The authoritative resource on Core Web Vitals, loading performance, and rendering optimization.
    [web.dev/performance](https://web.dev/performance/)

21. **"Rendering on the Web"** — Jason Miller & Addy Osmani
    Comprehensive comparison of SSR, SSG, CSR, ISR, streaming, and hybrid rendering strategies.
    [web.dev/rendering-on-the-web](https://web.dev/articles/rendering-on-the-web)

22. **"How Browsers Work"** — Tali Garsiel & Paul Irish
    Deep dive into browser internals: parsing, render tree construction, layout, and painting.
    [web.dev/howbrowserswork](https://web.dev/articles/howbrowserswork)

23. **"Optimizing Interaction to Next Paint"** — web.dev
    Guide to diagnosing and fixing INP issues — the newest Core Web Vital.
    [web.dev/inp](https://web.dev/articles/optimize-inp)

24. **"The Cost of JavaScript"** — Addy Osmani
    Why JavaScript is the most expensive resource on the web and how to reduce its impact.
    [v8.dev/blog/cost-of-javascript-2019](https://v8.dev/blog/cost-of-javascript-2019)

25. **"Reassure: Performance Testing for React Native"** — Callstack
    Library for catching performance regressions in CI by measuring render counts and durations.
    [github.com/callstack/reassure](https://github.com/callstack/reassure)

---

## State Management

26. **"TanStack Query Documentation"** — Tanner Linsley
    The definitive guide to server state management: queries, mutations, caching, pagination, optimistic updates.
    [tanstack.com/query/latest](https://tanstack.com/query/latest)

27. **"Practical React Query"** — TkDodo (Dominik Dorfmeister)
    Blog series covering real-world TanStack Query patterns. The best supplementary guide to the docs.
    [tkdodo.eu/blog/practical-react-query](https://tkdodo.eu/blog/practical-react-query)

28. **"Zustand Documentation"** — Daishi Kato
    Tiny, fast, scalable state management. The docs are remarkably clear for such a small API surface.
    [zustand-demo.pmnd.rs](https://zustand-demo.pmnd.rs/)

29. **"Jotai Documentation"** — Daishi Kato
    Atomic state management for React. Excellent for derived state and fine-grained reactivity.
    [jotai.org](https://jotai.org/)

30. **"XState Documentation"** — David Khourshid
    State machines and statecharts for JavaScript. Prevents impossible states through formal modeling.
    [stately.ai/docs/xstate](https://stately.ai/docs/xstate)

---

## Testing

31. **"Testing Library Documentation"** — Kent C. Dodds
    The guiding philosophy: test the way users interact, not implementation details.
    [testing-library.com](https://testing-library.com/)

32. **"Maestro: The Simplest E2E Testing Framework"** — mobile.dev
    YAML-based mobile E2E testing. Write flows in minutes, not hours.
    [maestro.mobile.dev](https://maestro.mobile.dev/)

33. **"MSW (Mock Service Worker) Documentation"** — Artem Zakharchenko
    Network-level API mocking for browser and Node.js. Test against realistic API behavior.
    [mswjs.io](https://mswjs.io/)

34. **"Playwright Documentation"** — Microsoft
    Cross-browser E2E testing with auto-wait, network interception, and trace viewer.
    [playwright.dev](https://playwright.dev/)

35. **"Testing React Native Apps"** — Callstack Blog
    Patterns for unit testing, component testing, and E2E testing in React Native.
    [callstack.com/blog/testing-react-native-apps](https://www.callstack.com/blog/guide-to-react-native-testing)

36. **"But Really, What Is a JavaScript Test?"** — Kent C. Dodds
    Foundational article on what testing actually is, from assertion basics to framework design.
    [kentcdodds.com/blog/but-really-what-is-a-javascript-test](https://kentcdodds.com/blog/but-really-what-is-a-javascript-test)

---

## Vercel & Next.js

37. **"Next.js Documentation"** — Vercel
    The official docs. App Router, Server Components, Server Actions, caching, and deployment.
    [nextjs.org/docs](https://nextjs.org/docs)

38. **"Vercel Documentation"** — Vercel
    Platform docs covering deployments, functions, edge network, analytics, and integrations.
    [vercel.com/docs](https://vercel.com/docs)

39. **"How Vercel Builds Vercel"** — Vercel Engineering
    Internal architecture, dogfooding, and the engineering culture behind the platform.
    [vercel.com/blog/how-vercel-builds-vercel](https://vercel.com/blog)

40. **"Partial Prerendering"** — Vercel Blog
    How PPR combines static and dynamic rendering in a single request for optimal performance.
    [vercel.com/blog/partial-prerendering](https://vercel.com/blog/partial-prerendering-with-next-js)

41. **"Understanding React Server Components"** — Vercel Blog
    The mental model shift: server-first components, the module graph boundary, and when to use `"use client"`.
    [vercel.com/blog/understanding-react-server-components](https://vercel.com/blog/understanding-react-server-components)

42. **"Fluid Compute"** — Vercel Blog
    How Vercel's serverless model eliminates cold starts by keeping functions warm and streaming responses.
    [vercel.com/blog/fluid-compute](https://vercel.com/blog/introducing-fluid-compute)

43. **"Vercel AI SDK Documentation"** — Vercel
    Build AI-powered features with a unified API across LLM providers. Streaming, tool calling, structured output.
    [sdk.vercel.ai/docs](https://sdk.vercel.ai/docs)

---

## Monitoring & Operations

44. **"Sentry React Native Documentation"** — Sentry
    Crash reporting, performance monitoring, and session replay for React Native apps.
    [docs.sentry.io/platforms/react-native](https://docs.sentry.io/platforms/react-native/)

45. **"The Practical Guide to Alerting"** — Rob Ewaschuk (Google SRE)
    Philosophy of good alerting: symptom-based, not cause-based. Reduce noise, increase signal.
    [docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit)

46. **"Google SRE Book"** — Google
    Free online book covering monitoring, alerting, incident response, and reliability engineering.
    [sre.google/sre-book/table-of-contents](https://sre.google/sre-book/table-of-contents/)

47. **"Firebase Crashlytics Documentation"** — Google
    Free crash reporting with real-time alerts, breadcrumbs, and integration with Google Play vitals.
    [firebase.google.com/docs/crashlytics](https://firebase.google.com/docs/crashlytics)

---

## Security

48. **"OWASP Mobile Top 10"** — OWASP Foundation
    The ten most critical mobile application security risks. Essential checklist for every mobile release.
    [owasp.org/www-project-mobile-top-10](https://owasp.org/www-project-mobile-top-10/)

49. **"OAuth 2.0 for Mobile & Native Apps (RFC 8252)"** — IETF
    The standard for implementing OAuth in mobile apps. Mandates PKCE, forbids embedded WebViews.
    [datatracker.ietf.org/doc/html/rfc8252](https://datatracker.ietf.org/doc/html/rfc8252)

50. **"Securing React Native Applications"** — Callstack Blog
    Practical security guide covering certificate pinning, secure storage, code obfuscation, and jailbreak detection.
    [callstack.com/blog/securing-react-native-applications](https://www.callstack.com/blog/securing-react-native-applications)

51. **"OWASP Web Application Security Testing Guide"** — OWASP Foundation
    Comprehensive testing methodology for web application security. The reference for penetration testing.
    [owasp.org/www-project-web-security-testing-guide](https://owasp.org/www-project-web-security-testing-guide/)

---

## Design Systems

52. **"Design Systems"** — Alla Kholmatova (Smashing Magazine)
    The foundational book on building and maintaining design systems. Covers language, patterns, and process.
    [smashingmagazine.com/design-systems-book](https://www.smashingmagazine.com/design-systems-book/)

53. **"Radix UI Primitives"** — WorkOS
    Unstyled, accessible component primitives. The foundation for custom design systems.
    [radix-ui.com](https://www.radix-ui.com/)

54. **"Building a Design System in React Native"** — Shopify Engineering
    How Shopify's Polaris design system extends to React Native with shared tokens and platform-specific components.
    [shopify.engineering/building-polaris-for-react-native](https://shopify.engineering/)

55. **"Storybook Documentation"** — Storybook
    Component development environment. Build, test, and document UI components in isolation.
    [storybook.js.org](https://storybook.js.org/)

---

## Payments & Monetization

56. **"Stripe Documentation"** — Stripe
    The most comprehensive payment API docs in the industry. PaymentIntents, webhooks, subscriptions.
    [stripe.com/docs](https://stripe.com/docs)

57. **"RevenueCat Documentation"** — RevenueCat
    In-app purchases and subscriptions made simple. Server-side receipt validation and entitlement management.
    [revenuecat.com/docs](https://www.revenuecat.com/docs/)

58. **"Implementing In-App Purchases in Expo"** — Expo Documentation
    Guide to integrating `expo-in-app-purchases` and RevenueCat with Expo managed workflow.
    [docs.expo.dev/guides/in-app-purchases](https://docs.expo.dev/guides/in-app-purchases/)

---

## Monorepos & Tooling

59. **"Turborepo Documentation"** — Vercel
    Task-based build system with content-addressable caching and remote cache sharing.
    [turbo.build/repo/docs](https://turbo.build/repo/docs)

60. **"pnpm Documentation"** — pnpm
    Fast, disk-efficient package manager. Strict dependency resolution prevents phantom dependencies.
    [pnpm.io](https://pnpm.io/)

61. **"Changesets Documentation"** — Changesets
    Version management and changelog generation for monorepos. Integrates with GitHub Actions.
    [github.com/changesets/changesets](https://github.com/changesets/changesets)

62. **"Monorepos in JavaScript & TypeScript"** — Nrwl Blog
    Comprehensive overview of monorepo strategies, tooling comparison, and organizational patterns.
    [nx.dev/concepts/more-concepts/why-monorepos](https://nx.dev/concepts/more-concepts/why-monorepos)

---

## Talks & Videos

63. **"React Native: The New Architecture"** — Nicola Corti (React Native EU)
    Deep technical walkthrough of JSI, Fabric, and TurboModules from a Meta engineer.
    [youtube.com/watch?v=EGKBS6bHBRQ](https://www.youtube.com/results?search_query=react+native+new+architecture+nicola+corti)

64. **"Goodbye, useEffect"** — David Khourshid (React Summit)
    Why most useEffect calls are wrong, and how to think about effects, events, and state machines instead.
    [youtube.com/watch?v=HPoC-k7Rxwo](https://www.youtube.com/watch?v=HPoC-k7Rxwo)

65. **"React Server Components from Scratch"** — Dan Abramov
    Build React Server Components from first principles to truly understand the mental model.
    [github.com/reactwg/server-components/discussions/5](https://github.com/reactwg/server-components/discussions/5)

66. **"The Weird Things About JavaScript"** — JavaScript Conferences
    Entertaining and educational exploration of JavaScript's quirks — closures, coercion, prototypes, and the event loop.
    [javascript.info](https://javascript.info/)

---

*Total: 66 resources. Last updated: 2026-04-07.*

> **See also:** [Appendix A: Glossary](./appendix-glossary.md) | [Appendix B: Cheat Sheet](./appendix-cheatsheet.md)
