<!--
  TYPE: appendix
  TITLE: Reading List
  APPENDIX: C
  UPDATED: 2026-04-07
-->

# Appendix C: Reading List

> 100+ essential articles, talks, papers, and tools — curated for the 100x frontend architect.

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

## Authentication & Authorization

67. **"Clerk Documentation"** — Clerk
    Drop-in authentication with pre-built UI components, session management, and multi-factor auth for Next.js and React.
    [clerk.com/docs](https://clerk.com/docs)

68. **"Auth.js v5 Guide"** — Auth.js
    The successor to NextAuth.js — framework-agnostic authentication with support for OAuth, credentials, and magic links.
    [authjs.dev](https://authjs.dev/)

69. **"The Complete Guide to OAuth 2.0 and PKCE"** — Auth0
    End-to-end walkthrough of the OAuth 2.0 authorization code flow with PKCE, the standard for SPAs and native apps.
    [auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)

70. **"Web Authentication API (WebAuthn)"** — MDN Web Docs
    Reference for the WebAuthn API — passwordless authentication using biometrics, security keys, and passkeys.
    [developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)

---

## Database & ORM

71. **"Drizzle ORM Documentation"** — Drizzle Team
    TypeScript-first ORM with SQL-like query syntax, zero dependencies, and excellent edge runtime support.
    [orm.drizzle.team](https://orm.drizzle.team/)

72. **"Neon Serverless Postgres Documentation"** — Neon
    Serverless Postgres with branching, autoscaling to zero, and a WebSocket-based driver for edge functions.
    [neon.tech/docs](https://neon.tech/docs)

73. **"Prisma vs Drizzle: A Practical Comparison"** — Various Engineering Blogs
    Head-to-head comparison of the two leading TypeScript ORMs — API design, performance, migration workflows, and edge compatibility.
    [orm.drizzle.team/docs/prisma](https://orm.drizzle.team/docs/prisma)

---

## GraphQL

74. **"Apollo Client Documentation"** — Apollo GraphQL
    Full-featured GraphQL client for React with normalized caching, optimistic UI, and local state management.
    [apollographql.com/docs/react](https://www.apollographql.com/docs/react/)

75. **"Production Ready GraphQL"** — Marc-André Giroux (Book)
    The definitive guide to building GraphQL APIs at scale — schema design, pagination, error handling, and performance.
    [productionreadygraphql.com](https://productionreadygraphql.com/)

76. **"GraphQL at Scale: How Netflix Uses Federation"** — Netflix Tech Blog
    How Netflix uses GraphQL Federation to compose a unified API from hundreds of microservices.
    [netflixtechblog.com/how-netflix-scales-its-api-with-graphql-federation](https://netflixtechblog.com/how-netflix-scales-its-api-with-graphql-federation-part-1-ae3557c187e2)

---

## Search

77. **"Algolia Documentation"** — Algolia
    Hosted search API with typo tolerance, faceting, and instant results. The standard for product and content search.
    [algolia.com/doc](https://www.algolia.com/doc/)

78. **"Postgres Full-Text Search is Good Enough"** — Rachid Belaid
    Practical guide to implementing full-text search with PostgreSQL — tsvector, tsquery, ranking, and indexing without external services.
    [rachbelaid.com/postgres-full-text-search-is-good-enough](http://rachbelaid.com/postgres-full-text-search-is-good-enough/)

---

## Feature Flags

79. **"Statsig Documentation"** — Statsig
    Feature flags, A/B testing, and product analytics with statistical rigor. Includes server and client SDKs.
    [docs.statsig.com](https://docs.statsig.com/)

80. **"Feature Flags Best Practices"** — LaunchDarkly
    Comprehensive guide to feature flag lifecycle management — naming conventions, flag debt, progressive rollouts, and kill switches.
    [launchdarkly.com/blog/feature-flag-best-practices](https://launchdarkly.com/blog/feature-flag-best-practices/)

---

## Real-time

81. **"Socket.io Documentation"** — Socket.io
    Event-driven library for real-time bidirectional communication with automatic reconnection and fallback transports.
    [socket.io/docs](https://socket.io/docs/v4/)

82. **"Collaborative Editing with CRDTs"** — Martin Kleppmann
    Academic yet accessible explanation of Conflict-free Replicated Data Types for building real-time collaborative editors.
    [martin.kleppmann.com/papers/crdt-book-chapter.pdf](https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html)

83. **"Liveblocks Documentation"** — Liveblocks
    Real-time collaboration infrastructure — presence, storage, comments, and notifications with React hooks.
    [liveblocks.io/docs](https://liveblocks.io/docs)

---

## Analytics

84. **"PostHog Documentation"** — PostHog
    Open-source product analytics with event tracking, session replay, feature flags, and A/B testing in a single platform.
    [posthog.com/docs](https://posthog.com/docs)

85. **"Building an Analytics Pipeline"** — Various Engineering Blogs
    Patterns for building event collection, processing, and warehousing pipelines — from client instrumentation to data lake.
    [posthog.com/blog/analytics-pipeline](https://posthog.com/blog)

---

## Email

86. **"Resend Documentation"** — Resend
    Modern email API built for developers with React component support, deliverability tools, and webhook events.
    [resend.com/docs](https://resend.com/docs)

87. **"React Email"** — React Email
    Build and preview email templates using React components — bridging the gap between modern DX and HTML email constraints.
    [react.email](https://react.email/)

88. **"Email Deliverability Guide"** — Postmark
    Everything you need to know about email deliverability — SPF, DKIM, DMARC, reputation management, and bounce handling.
    [postmarkapp.com/guides/email-deliverability](https://postmarkapp.com/guides/email-deliverability)

---

## SSR & Cookies

89. **"Understanding Cookies in Next.js"** — Next.js Documentation
    How to read, set, and manage cookies in Server Components, Route Handlers, Server Actions, and Middleware.
    [nextjs.org/docs/app/api-reference/functions/cookies](https://nextjs.org/docs/app/api-reference/functions/cookies)

90. **"Authentication in Server Components"** — Next.js Patterns
    Patterns for session management, token validation, and role-based access in the App Router's server-first model.
    [nextjs.org/docs/app/building-your-application/authentication](https://nextjs.org/docs/app/building-your-application/authentication)

91. **"HTTP Cookies"** — MDN Web Docs
    Complete reference for cookie attributes — SameSite, Secure, HttpOnly, Path, Domain, and partitioned cookies.
    [developer.mozilla.org/en-US/docs/Web/HTTP/Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

---

## UX & Design

92. **"Human Interface Guidelines: Designing for iPhone"** — Apple
    Apple's official design guidance covering layout, navigation, typography, color, and platform conventions for mobile.
    [developer.apple.com/design/human-interface-guidelines](https://developer.apple.com/design/human-interface-guidelines/)

93. **"Material Design 3 Guidelines"** — Google
    Google's latest design system with adaptive layouts, dynamic color, and component specifications for cross-platform apps.
    [m3.material.io](https://m3.material.io/)

94. **"Don't Make Me Think, Revisited"** — Steve Krug (Book)
    The classic usability book — intuitive navigation, visual hierarchy, and the art of removing friction from every interaction.
    [sensible.com/dont-make-me-think](https://sensible.com/dont-make-me-think/)

95. **"Refactoring UI"** — Adam Wathan & Steve Schoger (Book)
    Tactical design advice for developers — spacing, typography, color, depth, and layout decisions that make UI look professional.
    [refactoringui.com](https://www.refactoringui.com/)

---

## Performance (Extended)

96. **"Core Web Vitals"** — web.dev (Google Chrome Team)
    Comprehensive guides to LCP, INP, and CLS — what they measure, why they matter, and how to optimize each metric.
    [web.dev/explore/learn-core-web-vitals](https://web.dev/explore/learn-core-web-vitals)

97. **"Performance Budgets 101"** — web.dev
    How to define, enforce, and monitor performance budgets — preventing regressions before they ship.
    [web.dev/articles/performance-budgets-101](https://web.dev/articles/performance-budgets-101)

---

## Bonus: Foundational References

98. **"React Documentation"** — React Team
    The official React docs with interactive examples. Covers components, hooks, Server Components, and escape hatches.
    [react.dev](https://react.dev/)

99. **"MDN Web Docs"** — Mozilla
    The single most important web reference. HTML, CSS, JavaScript, Web APIs — comprehensive, accurate, and always current.
    [developer.mozilla.org](https://developer.mozilla.org/)

100. **"The Pragmatic Programmer"** — David Thomas & Andrew Hunt (Book)
     Timeless software engineering wisdom — DRY, orthogonality, tracer bullets, and the broken window theory applied to code.
     [pragprog.com/titles/tpp20](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/)

101. **"System Design Interview"** — Alex Xu (Book)
     Step-by-step framework for designing scalable systems — load balancers, caches, databases, and message queues explained clearly.
     [bytebytego.com](https://bytebytego.com/)

102. **"A Comprehensive Guide to Error Handling in React"** — Kent C. Dodds
     Error boundaries, try/catch in Server Components, and user-friendly error recovery patterns.
     [kentcdodds.com/blog/use-react-error-boundary](https://kentcdodds.com/blog/use-react-error-boundary-to-handle-errors-in-react)

103. **"CSS for JavaScript Developers"** — Josh W. Comeau
     Deep understanding of CSS layout, stacking contexts, and animations — taught from a JavaScript developer's mental model.
     [css-for-js.dev](https://css-for-js.dev/)

---

*Total: 103 resources. Last updated: 2026-04-07.*

> **See also:** [Appendix A: Glossary](./appendix-glossary.md) | [Appendix B: Cheat Sheet](./appendix-cheatsheet.md)
