<!--
  CHAPTER: 32
  TITLE: PWAs, Web APIs & Cross-Platform Web
  PART: VI — Vercel & the Web
  PREREQS: Chapters 27, 28
  KEY_TOPICS: Progressive Web Apps, service workers, Web App Manifest, installability, offline web, Intersection Observer, Web Workers, Geolocation API, Notifications API, Web Share API, Web Bluetooth, WebRTC, Web Animations API, Expo for Web
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 32: PWAs, Web APIs & Cross-Platform Web

> **Part VI — Vercel & the Web** | Prerequisites: Chapters 27, 28 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- A Progressive Web App is just a web app with three things: HTTPS, a Web App Manifest, and a service worker. That is it. No framework required, no app store required, no special build step. Get those three right and you unlock installability, offline support, and an app-like experience.
- Service workers are the backbone of PWAs. They intercept network requests, cache responses, and enable offline functionality. Use Serwist (the maintained Workbox successor) to avoid writing raw service worker code — it handles caching strategies, precaching, and runtime caching with battle-tested patterns.
- The web platform in 2026 has APIs most frontend developers have never touched: Intersection Observer for lazy loading, Web Workers for CPU-intensive tasks off the main thread, Web Share for native share sheets, Screen Wake Lock for keeping the display on, and dozens more. Knowing what the platform gives you for free prevents you from reaching for npm packages you do not need.
- Expo for Web (via react-native-web) lets you render React Native components in the browser. It is real, it works, and Expo Router supports web routing. But it is not a replacement for Next.js — it is a strategy for sharing mobile components on web when you already have a React Native app.
- The PWA vs. native decision is not binary. Use progressive enhancement: start with a great web experience, add PWA capabilities where they matter, and go native only when you need hardware APIs or app store distribution that the web cannot provide.

</details>

Let me say something controversial: most mobile apps should have been websites.

I mean it. That restaurant app you downloaded, used once to place an order, and then it sat in your phone taking up 87MB until you deleted it six months later? That should have been a website. The event app that sent you to the App Store, made you wait for a 45MB download, asked for your email, asked for notification permission, asked for location permission, and then showed you a schedule that could have been a single HTML page? Website.

The app store model is brilliant for certain categories of software — games, social media, productivity tools you use daily, anything that needs deep hardware integration. But for a huge swath of use cases, the web is a better distribution mechanism. No install friction. No update delays. No 30% platform tax. Universal links that just work.

Progressive Web Apps are what happens when you take the web's distribution model and add the capabilities that used to be native-only: offline support, home screen installation, push notifications, background sync. The result is something that loads like a website, installs like an app, and updates like neither — instantly, silently, without asking the user to do anything.

This chapter covers PWAs from the ground up, then goes wide into the Web APIs that every frontend architect should know exist — because the platform gives you far more than most developers realize.

### In This Chapter
- PWA Fundamentals — What makes a PWA, installability, the app-like experience
- Service Workers — Registration, lifecycle, caching strategies, Serwist
- Web App Manifest — Configuration for installability and appearance
- Offline Web — Cache API, IndexedDB, offline fallbacks, background sync
- Observation & Threading APIs — Intersection Observer, Resize Observer, Mutation Observer, Web Workers
- Media & Device APIs — Geolocation, Notifications, Web Share, Clipboard, Wake Lock
- Advanced Web APIs — WebRTC, Web Bluetooth, Payment Request, Credential Management
- Expo for Web — React Native components in the browser
- PWA vs. Native — Decision framework

### Related Chapters
- [Ch 27: Developer Experience & Tooling] — Build tooling that supports PWA development
- [Ch 28: Beast Mode] — Advanced web performance techniques
- [Ch 12: Offline & Real-Time] — Offline patterns in React Native
- [Ch 25: Next.js App Router] — Server-side rendering and the web platform

---

## 32.1 PWA Fundamentals — What Makes a PWA

### The Three Requirements

A Progressive Web App is not a framework. It is not a library. It is not a special build mode. A PWA is any web application that meets three criteria:

1. **Served over HTTPS** — The browser will not register a service worker on an insecure origin (localhost gets a pass for development)
2. **Has a Web App Manifest** — A JSON file that tells the browser how to display the app when installed
3. **Has a service worker** — A JavaScript file that runs in the background and can intercept network requests

That is the entire list. If your web app has those three things, it is a PWA. Chrome will show the install prompt. Safari will let users add it to their home screen. The app will appear in the user's app drawer, task switcher, and (on most platforms) the system search.

```
The Three Pillars of a PWA:

┌─────────────────────────────────────────────────────────┐
│                    YOUR WEB APP                          │
│                                                          │
│   ┌──────────┐   ┌──────────────┐   ┌──────────────┐   │
│   │  HTTPS   │ + │  Manifest    │ + │  Service      │   │
│   │          │   │  (JSON)      │   │  Worker (JS)  │   │
│   └──────────┘   └──────────────┘   └──────────────┘   │
│                                                          │
│   Security       Identity &          Offline &           │
│   foundation     appearance          caching             │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   INSTALLABLE PWA   │
              │   • Home screen     │
              │   • App switcher    │
              │   • Offline support │
              │   • Push notifs     │
              └─────────────────────┘
```

### Installability Criteria

Each browser has slightly different requirements for when it will show the install prompt. Here is what Chrome requires as of 2026:

| Requirement | Details |
|-------------|---------|
| HTTPS | Must be served over a secure connection |
| Web App Manifest | Must include `name` or `short_name`, `icons` (at least 192x192 and 512x512), `start_url`, `display` (standalone, fullscreen, or minimal-ui) |
| Service Worker | Must have a registered service worker with a `fetch` event handler |
| Not already installed | The app must not already be installed on the device |
| Engagement heuristic | User must have interacted with the domain (Chrome removed the strict "30 seconds" rule but still requires some engagement) |

Safari on iOS has different (and historically more limited) behavior:

| Feature | Chrome / Android | Safari / iOS |
|---------|-----------------|--------------|
| Install prompt | Automatic `beforeinstallprompt` event | Manual — user must tap "Add to Home Screen" in share sheet |
| Service worker support | Full | Full (since iOS 16.4 for push notifications) |
| Push notifications | Full | Supported since iOS 16.4 (PWA must be added to home screen first) |
| Background sync | Supported | Not supported |
| Persistent storage | Navigator.storage API | Limited — Safari may evict after 7 days of non-use |
| Badging API | Supported | Supported (since iOS 16.4) |

### The Add to Home Screen Prompt

On Android with Chrome, you can programmatically control when the install prompt appears. The browser fires a `beforeinstallprompt` event that you can intercept and defer:

```typescript
// pwa-install.ts — Controlling the install prompt

let deferredPrompt: BeforeInstallPromptEvent | null = null;

// Listen for the browser's install prompt
window.addEventListener('beforeinstallprompt', (event) => {
  // Prevent the default mini-infobar from appearing
  event.preventDefault();
  
  // Save the event so we can trigger it later
  deferredPrompt = event;
  
  // Show your own custom install UI
  showInstallButton();
});

// When the user clicks your custom install button
async function handleInstallClick() {
  if (!deferredPrompt) return;
  
  // Show the browser's install prompt
  deferredPrompt.prompt();
  
  // Wait for the user's response
  const result = await deferredPrompt.userChoice;
  
  if (result.outcome === 'accepted') {
    analytics.track('pwa_installed');
    console.log('User accepted the install prompt');
  } else {
    analytics.track('pwa_install_dismissed');
    console.log('User dismissed the install prompt');
  }
  
  // The prompt can only be used once
  deferredPrompt = null;
  hideInstallButton();
}

// Detect if the app is already installed
window.addEventListener('appinstalled', () => {
  // The app was installed — clean up
  deferredPrompt = null;
  hideInstallButton();
  analytics.track('pwa_installed_confirmed');
});

// Check if running as installed PWA
function isInstalledPWA(): boolean {
  // Check display-mode media query
  if (window.matchMedia('(display-mode: standalone)').matches) {
    return true;
  }
  // iOS Safari
  if ((navigator as any).standalone === true) {
    return true;
  }
  return false;
}
```

### The App-Like Experience

A well-built PWA is indistinguishable from a native app to most users. Here is what "app-like" actually means in practice:

**No browser chrome.** When opened from the home screen, a standalone PWA hides the browser's URL bar, tabs, and navigation buttons. The user sees only your app.

**Smooth transitions.** Use the View Transitions API (now well-supported) for page transitions that feel native:

```typescript
// View Transitions for app-like navigation
function navigateTo(url: string) {
  if (!document.startViewTransition) {
    // Fallback for browsers without View Transitions
    window.location.href = url;
    return;
  }

  document.startViewTransition(async () => {
    // Update the DOM — in a framework like Next.js,
    // this happens via router.push()
    await updateContent(url);
  });
}
```

```css
/* CSS for view transitions */
::view-transition-old(root) {
  animation: fade-out 200ms ease-out;
}

::view-transition-new(root) {
  animation: fade-in 200ms ease-in;
}

@keyframes fade-out {
  from { opacity: 1; }
  to { opacity: 0; }
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

**Pull-to-refresh control.** By default, Chrome shows its own pull-to-refresh UI in standalone mode. You can override this:

```css
/* Disable default pull-to-refresh in standalone PWA */
body {
  overscroll-behavior-y: contain;
}
```

**Status bar theming.** The `theme_color` in your manifest and a meta tag control the status bar color:

```html
<meta name="theme-color" content="#1a1a2e" media="(prefers-color-scheme: dark)">
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
```

**Safe area handling.** On devices with notches or rounded corners:

```css
/* Handle safe areas in PWA standalone mode */
body {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

/* For the viewport meta tag */
/* <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover"> */
```

---

## 32.2 Service Workers — The Brain of Your PWA

### What a Service Worker Actually Is

A service worker is a JavaScript file that runs in a separate thread from your main page. It sits between your web app and the network, acting as a programmable proxy. Every network request your page makes passes through the service worker first, giving you the ability to intercept, modify, cache, or fabricate responses.

```
Without service worker:
  Browser ──── request ───▶ Network ──── response ───▶ Browser

With service worker:
  Browser ──── request ───▶ Service Worker ──── request ───▶ Network
                                  │                             │
                                  │        ┌─────────┐          │
                                  │◀───────│  Cache   │◀────────│
                                  │        └─────────┘          │
                                  │                             │
  Browser ◀── response ───── Service Worker ◀── response ──────┘
```

Key characteristics of service workers:

- **No DOM access.** They cannot touch the page directly. Communication happens via `postMessage`.
- **Fully async.** No synchronous APIs (no `localStorage`, no synchronous XHR).
- **HTTPS only.** They can intercept all requests for your origin — that power requires a secure context.
- **Event-driven.** They wake up when events fire and go back to sleep when idle. No persistent state in memory.
- **Separate lifecycle.** They install and activate independently of your page. A new service worker can be waiting while an old one is still active.

### Registration

Register your service worker from your main app code:

```typescript
// register-sw.ts — Service worker registration

export async function registerServiceWorker() {
  if (!('serviceWorker' in navigator)) {
    console.log('Service workers not supported');
    return;
  }

  try {
    const registration = await navigator.serviceWorker.register('/sw.js', {
      scope: '/',
      // 'module' type allows using ES module imports in the service worker
      type: 'module',
    });

    console.log('SW registered with scope:', registration.scope);

    // Check for updates periodically
    setInterval(() => {
      registration.update();
    }, 60 * 60 * 1000); // Check every hour

    // Listen for new service worker
    registration.addEventListener('updatefound', () => {
      const newWorker = registration.installing;
      if (!newWorker) return;

      newWorker.addEventListener('statechange', () => {
        if (
          newWorker.state === 'activated' &&
          navigator.serviceWorker.controller
        ) {
          // New service worker activated — notify user
          showUpdateAvailableUI();
        }
      });
    });
  } catch (error) {
    console.error('SW registration failed:', error);
  }
}
```

### The Service Worker Lifecycle

This is where most people get confused, so pay attention. The service worker lifecycle has three phases:

```
┌────────────────────────────────────────────────────────────┐
│                SERVICE WORKER LIFECYCLE                      │
│                                                              │
│  1. INSTALL                                                  │
│     ├── Browser downloads sw.js                             │
│     ├── Parses and executes it                              │
│     ├── 'install' event fires                               │
│     ├── You cache static assets here (precaching)           │
│     └── waitUntil() keeps install phase open                │
│                                                              │
│  2. WAIT (if previous SW is still active)                   │
│     ├── New SW is "installed" but "waiting"                 │
│     ├── Old SW still controls existing pages                │
│     └── skipWaiting() bypasses this phase                   │
│                                                              │
│  3. ACTIVATE                                                 │
│     ├── 'activate' event fires                              │
│     ├── Old caches cleaned up here                          │
│     ├── clients.claim() takes control of open pages         │
│     └── SW now intercepts fetch events                      │
│                                                              │
│  4. IDLE / FETCH / TERMINATED                               │
│     ├── SW sleeps until an event fires                      │
│     ├── 'fetch' event for every network request             │
│     ├── 'push' event for push notifications                 │
│     ├── 'sync' event for background sync                    │
│     └── Browser may terminate idle SW to save memory        │
└────────────────────────────────────────────────────────────┘
```

Here is a raw service worker that demonstrates all three phases:

```typescript
// sw.ts — Service worker with lifecycle handlers

const CACHE_NAME = 'app-cache-v1';
const PRECACHE_URLS = [
  '/',
  '/offline.html',
  '/styles/main.css',
  '/scripts/app.js',
  '/images/logo.svg',
];

// Phase 1: INSTALL — Cache static assets
self.addEventListener('install', (event: ExtendableEvent) => {
  console.log('[SW] Installing...');
  
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      console.log('[SW] Precaching static assets');
      return cache.addAll(PRECACHE_URLS);
    })
  );
  
  // Skip waiting — activate immediately
  // (only do this if you handle the implications)
  // self.skipWaiting();
});

// Phase 3: ACTIVATE — Clean up old caches
self.addEventListener('activate', (event: ExtendableEvent) => {
  console.log('[SW] Activating...');
  
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => {
            console.log(`[SW] Deleting old cache: ${name}`);
            return caches.delete(name);
          })
      );
    })
  );
  
  // Take control of all open pages immediately
  // Without this, pages opened before activation
  // won't be controlled until refresh
  self.clients.claim();
});

// Phase 4: FETCH — Intercept network requests
self.addEventListener('fetch', (event: FetchEvent) => {
  // Only handle GET requests
  if (event.request.method !== 'GET') return;
  
  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      if (cachedResponse) {
        return cachedResponse;
      }
      return fetch(event.request);
    })
  );
});
```

### Caching Strategies

This is the meat of service worker development. There are five caching strategies, and choosing the right one for each type of resource is what separates a PWA that feels snappy from one that shows stale data or slow loading spinners.

#### Strategy 1: Cache First (Cache Falling Back to Network)

Best for: Static assets that rarely change — images, fonts, CSS, JS bundles with content hashes.

```typescript
// Cache First strategy
self.addEventListener('fetch', (event: FetchEvent) => {
  if (isStaticAsset(event.request.url)) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        if (cached) {
          return cached; // Instant! No network hit.
        }
        // Not in cache — fetch, cache, and return
        return fetch(event.request).then((response) => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, clone);
          });
          return response;
        });
      })
    );
  }
});

function isStaticAsset(url: string): boolean {
  return /\.(js|css|png|jpg|jpeg|svg|woff2|woff|ttf)(\?.*)?$/.test(url);
}
```

```
Cache First flow:
  Request ──▶ Cache hit? ──YES──▶ Return cached response (fast!)
                  │
                  NO
                  │
                  ▼
              Fetch from network
                  │
                  ├──▶ Cache response
                  └──▶ Return response
```

#### Strategy 2: Network First (Network Falling Back to Cache)

Best for: API responses, dynamic content, anything where freshness matters more than speed.

```typescript
// Network First strategy
self.addEventListener('fetch', (event: FetchEvent) => {
  if (isApiRequest(event.request.url)) {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          // Got a response — cache it for offline use
          const clone = response.clone();
          caches.open(API_CACHE).then((cache) => {
            cache.put(event.request, clone);
          });
          return response;
        })
        .catch(() => {
          // Network failed — try the cache
          return caches.match(event.request).then((cached) => {
            if (cached) return cached;
            // Nothing in cache either — return offline page
            return caches.match('/offline.html');
          });
        })
    );
  }
});
```

```
Network First flow:
  Request ──▶ Fetch from network
                  │
              Success? ──YES──▶ Cache response ──▶ Return response
                  │
                  NO (timeout or error)
                  │
                  ▼
              Cache hit? ──YES──▶ Return cached (stale but better than nothing)
                  │
                  NO
                  │
                  ▼
              Return offline fallback page
```

#### Strategy 3: Stale-While-Revalidate

Best for: Content that updates occasionally but where showing slightly stale data is acceptable — user profile data, product listings, blog posts.

```typescript
// Stale-While-Revalidate strategy
self.addEventListener('fetch', (event: FetchEvent) => {
  if (isContentRequest(event.request.url)) {
    event.respondWith(
      caches.open(CONTENT_CACHE).then((cache) => {
        return cache.match(event.request).then((cached) => {
          // Start the network fetch regardless
          const networkFetch = fetch(event.request).then((response) => {
            // Update the cache with the fresh response
            cache.put(event.request, response.clone());
            return response;
          });

          // Return cached immediately if available,
          // otherwise wait for network
          return cached || networkFetch;
        });
      })
    );
  }
});
```

```
Stale-While-Revalidate flow:
  Request ──┬──▶ Return cached immediately (stale but fast!)
            │
            └──▶ Fetch from network in background
                     │
                     ▼
                 Update cache (next request gets fresh data)
```

#### Strategy 4: Cache Only

Best for: Assets that you precached during install and that will never change (versioned assets).

```typescript
// Cache Only
event.respondWith(caches.match(event.request));
```

#### Strategy 5: Network Only

Best for: Requests that must always be fresh — analytics pings, authentication endpoints, payment processing.

```typescript
// Network Only
event.respondWith(fetch(event.request));
```

### Strategy Decision Matrix

| Resource Type | Strategy | Why |
|---------------|----------|-----|
| App shell HTML | Cache First with background update | Instant load, update silently |
| Hashed JS/CSS bundles | Cache First | Content hash = immutable |
| Fonts | Cache First | Rarely change, expensive to fetch |
| Images | Cache First | Large, slow to fetch |
| API responses (read) | Stale-While-Revalidate | Show data fast, update in background |
| API responses (write) | Network Only + Background Sync | Must hit server |
| User-specific data | Network First | Freshness matters |
| Analytics | Network Only | Do not cache |
| Auth endpoints | Network Only | Must be real-time |
| Third-party resources | Network First | You do not control their caching |

### Serwist — The Practical Way to Build Service Workers

Nobody writes raw service worker code in production anymore. Workbox was Google's answer to this, and Serwist is the actively maintained community fork that picked up where Workbox left off.

Serwist gives you:
- Precaching with revision management
- Runtime caching with named strategies
- Background sync for offline form submissions
- Expiration policies for cached resources
- Navigation preload for faster page loads

```bash
# Install Serwist
pnpm add @serwist/next  # For Next.js
# or
pnpm add serwist        # For vanilla/other frameworks
```

#### Serwist with Next.js

```typescript
// next.config.ts
import withSerwist from '@serwist/next';

const nextConfig = {
  // Your existing Next.js config
};

export default withSerwist({
  swSrc: 'src/sw.ts',
  swDest: 'public/sw.js',
  cacheOnNavigation: true,
  reloadOnOnline: true,
  disable: process.env.NODE_ENV === 'development',
})(nextConfig);
```

```typescript
// src/sw.ts — Serwist service worker for Next.js
import { defaultCache } from '@serwist/next/worker';
import { Serwist } from 'serwist';

const serwist = new Serwist({
  // Precache entries are injected at build time
  precacheEntries: self.__SW_MANIFEST,
  precacheOptions: {
    cleanupOutdatedCaches: true,
    concurrency: 10,
  },
  skipWaiting: true,
  clientsClaim: true,
  navigationPreload: true,
  runtimeCaching: defaultCache,
  fallbacks: {
    entries: [
      {
        url: '/offline',
        matcher({ request }) {
          return request.destination === 'document';
        },
      },
    ],
  },
});

serwist.addEventListeners();
```

#### Custom Serwist Runtime Caching

```typescript
// src/sw.ts — Custom caching strategies with Serwist
import {
  CacheFirst,
  NetworkFirst,
  StaleWhileRevalidate,
  ExpirationPlugin,
  CacheableResponsePlugin,
} from 'serwist';

const serwist = new Serwist({
  precacheEntries: self.__SW_MANIFEST,
  skipWaiting: true,
  clientsClaim: true,
  runtimeCaching: [
    // Cache Google Fonts stylesheets
    {
      matcher: ({ url }) => url.origin === 'https://fonts.googleapis.com',
      handler: new StaleWhileRevalidate({
        cacheName: 'google-fonts-stylesheets',
      }),
    },
    // Cache Google Fonts webfont files
    {
      matcher: ({ url }) => url.origin === 'https://fonts.gstatic.com',
      handler: new CacheFirst({
        cacheName: 'google-fonts-webfonts',
        plugins: [
          new CacheableResponsePlugin({ statuses: [0, 200] }),
          new ExpirationPlugin({
            maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
            maxEntries: 30,
          }),
        ],
      }),
    },
    // Cache images
    {
      matcher: ({ request }) => request.destination === 'image',
      handler: new CacheFirst({
        cacheName: 'images',
        plugins: [
          new CacheableResponsePlugin({ statuses: [0, 200] }),
          new ExpirationPlugin({
            maxEntries: 100,
            maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
          }),
        ],
      }),
    },
    // Cache API responses
    {
      matcher: ({ url }) => url.pathname.startsWith('/api/'),
      handler: new NetworkFirst({
        cacheName: 'api-responses',
        networkTimeoutSeconds: 5,
        plugins: [
          new CacheableResponsePlugin({ statuses: [0, 200] }),
          new ExpirationPlugin({
            maxEntries: 50,
            maxAgeSeconds: 60 * 60, // 1 hour
          }),
        ],
      }),
    },
    // Default handler for navigation requests
    {
      matcher: ({ request }) => request.mode === 'navigate',
      handler: new NetworkFirst({
        cacheName: 'pages',
        networkTimeoutSeconds: 3,
      }),
    },
  ],
});

serwist.addEventListeners();
```

### Service Worker Update Flow

One of the trickiest parts of service worker development is managing updates. Here is the pattern I recommend:

```typescript
// hooks/useServiceWorkerUpdate.ts
import { useEffect, useState, useCallback } from 'react';

export function useServiceWorkerUpdate() {
  const [updateAvailable, setUpdateAvailable] = useState(false);
  const [registration, setRegistration] =
    useState<ServiceWorkerRegistration | null>(null);

  useEffect(() => {
    if (!('serviceWorker' in navigator)) return;

    navigator.serviceWorker.ready.then((reg) => {
      setRegistration(reg);

      // Check for waiting service worker
      if (reg.waiting) {
        setUpdateAvailable(true);
      }

      // Listen for new service worker installing
      reg.addEventListener('updatefound', () => {
        const newWorker = reg.installing;
        if (!newWorker) return;

        newWorker.addEventListener('statechange', () => {
          if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
            // New worker installed but waiting — update available
            setUpdateAvailable(true);
          }
        });
      });
    });

    // Handle controller change (new SW took over)
    let refreshing = false;
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (!refreshing) {
        refreshing = true;
        window.location.reload();
      }
    });
  }, []);

  const applyUpdate = useCallback(() => {
    if (!registration?.waiting) return;
    // Tell the waiting SW to skip waiting and activate
    registration.waiting.postMessage({ type: 'SKIP_WAITING' });
  }, [registration]);

  return { updateAvailable, applyUpdate };
}
```

```tsx
// components/UpdateBanner.tsx
import { useServiceWorkerUpdate } from '../hooks/useServiceWorkerUpdate';

export function UpdateBanner() {
  const { updateAvailable, applyUpdate } = useServiceWorkerUpdate();

  if (!updateAvailable) return null;

  return (
    <div className="fixed bottom-4 left-4 right-4 bg-blue-600 text-white
                    p-4 rounded-lg shadow-lg flex items-center justify-between
                    z-50 mx-auto max-w-md">
      <p className="text-sm font-medium">
        A new version is available
      </p>
      <button
        onClick={applyUpdate}
        className="ml-4 px-4 py-2 bg-white text-blue-600
                   rounded font-medium text-sm hover:bg-blue-50"
      >
        Update
      </button>
    </div>
  );
}
```

---

## 32.3 Web App Manifest — Your App's Identity Card

The Web App Manifest is a JSON file that tells the browser everything it needs to know about how to present your app when installed. Think of it as the `app.json` of the web world.

### The Complete Manifest

```json
{
  "name": "Acme Project Manager",
  "short_name": "Acme PM",
  "description": "Manage projects, track time, and collaborate with your team.",
  "start_url": "/dashboard?source=pwa",
  "scope": "/",
  "display": "standalone",
  "orientation": "any",
  "theme_color": "#1a1a2e",
  "background_color": "#1a1a2e",
  "dir": "ltr",
  "lang": "en-US",
  "categories": ["productivity", "business"],
  "icons": [
    {
      "src": "/icons/icon-48x48.png",
      "sizes": "48x48",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-192x192-maskable.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512x512-maskable.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/desktop-dashboard.png",
      "sizes": "1920x1080",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Project dashboard on desktop"
    },
    {
      "src": "/screenshots/mobile-tasks.png",
      "sizes": "1080x1920",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Task list on mobile"
    }
  ],
  "shortcuts": [
    {
      "name": "New Project",
      "short_name": "New",
      "description": "Create a new project",
      "url": "/projects/new",
      "icons": [
        {
          "src": "/icons/shortcut-new.png",
          "sizes": "96x96"
        }
      ]
    },
    {
      "name": "My Tasks",
      "short_name": "Tasks",
      "description": "View your assigned tasks",
      "url": "/tasks/mine",
      "icons": [
        {
          "src": "/icons/shortcut-tasks.png",
          "sizes": "96x96"
        }
      ]
    }
  ],
  "related_applications": [],
  "prefer_related_applications": false
}
```

### Manifest Property Reference

| Property | What It Does | Why It Matters |
|----------|-------------|----------------|
| `name` | Full name shown in install dialog and app launcher | This is your app's display name — make it recognizable |
| `short_name` | Shorter name for home screen (12 chars max recommended) | Home screen icons have limited label space |
| `start_url` | URL loaded when app is launched from home screen | Add a query param (`?source=pwa`) to track PWA launches in analytics |
| `scope` | URL scope the service worker controls | Everything outside scope opens in the browser, not your PWA shell |
| `display` | How the app window appears | `standalone` = no browser chrome. `fullscreen` = no status bar either. `minimal-ui` = small nav bar. `browser` = normal browser tab. |
| `theme_color` | Color of the title bar / status bar | Creates visual continuity between your app and the OS |
| `background_color` | Splash screen background color | Shown briefly while app loads — match it to your app background |
| `icons` | App icons at various sizes | Need at least 192x192 and 512x512. Provide `maskable` variants for Android adaptive icons. |
| `screenshots` | Screenshots shown in the "richer install UI" | Chrome on Android shows these in the install sheet. Use `form_factor` to target desktop vs mobile. |
| `shortcuts` | Quick actions from long-pressing the app icon | Like iOS 3D Touch shortcuts — direct links to key features |
| `orientation` | Preferred screen orientation | `any` for most apps. `portrait` for mobile-first. `landscape` for media apps. |
| `categories` | App categories for potential store listing | Informational — browsers may use for classification |

### Display Modes Compared

```
┌─────────────────────────────────────────────────────┐
│  BROWSER (display: "browser")                       │
│  ┌─────────────────────────────────────────────┐    │
│  │ ◀ ▶ ↻  https://app.com          ⭐ ≡       │    │
│  ├─────────────────────────────────────────────┤    │
│  │                                             │    │
│  │              Your App                       │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│  Full browser UI — URL bar, tabs, bookmarks         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  MINIMAL-UI (display: "minimal-ui")                 │
│  ┌─────────────────────────────────────────────┐    │
│  │ ◀    app.com                            ≡   │    │
│  ├─────────────────────────────────────────────┤    │
│  │                                             │    │
│  │              Your App                       │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│  Minimal nav — back button, origin, menu            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  STANDALONE (display: "standalone")                  │
│  ┌─────────────────────────────────────────────┐    │
│  │  9:41                          📶 🔋        │    │
│  ├─────────────────────────────────────────────┤    │
│  │                                             │    │
│  │              Your App                       │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│  No browser UI — just status bar + your content     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  FULLSCREEN (display: "fullscreen")                  │
│  ┌─────────────────────────────────────────────┐    │
│  │                                             │    │
│  │              Your App                       │    │
│  │         (no status bar either)              │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│  Nothing but your app — good for games/media        │
└─────────────────────────────────────────────────────┘
```

### Linking the Manifest

```html
<!-- In your HTML <head> -->
<link rel="manifest" href="/manifest.json">

<!-- Apple-specific PWA meta tags (Safari still needs these) -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Acme PM">

<!-- Apple touch icons (Safari does not use the manifest for icons) -->
<link rel="apple-touch-icon" sizes="180x180" href="/icons/apple-touch-icon.png">
```

### Next.js Manifest Generation

In Next.js, you can generate the manifest dynamically:

```typescript
// app/manifest.ts — Dynamic manifest generation in Next.js
import type { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Acme Project Manager',
    short_name: 'Acme PM',
    description: 'Manage projects and collaborate with your team.',
    start_url: '/dashboard?source=pwa',
    display: 'standalone',
    background_color: '#1a1a2e',
    theme_color: '#1a1a2e',
    icons: [
      {
        src: '/icons/icon-192x192.png',
        sizes: '192x192',
        type: 'image/png',
      },
      {
        src: '/icons/icon-512x512.png',
        sizes: '512x512',
        type: 'image/png',
      },
      {
        src: '/icons/icon-512x512-maskable.png',
        sizes: '512x512',
        type: 'image/png',
        purpose: 'maskable',
      },
    ],
  };
}
```

---

## 32.4 Offline Web — Making Your App Work Without a Connection

### The Cache API

The Cache API is a key-value store where the key is a `Request` and the value is a `Response`. It is the foundation of offline support in service workers.

```typescript
// Cache API basics

// Open or create a cache
const cache = await caches.open('my-cache-v1');

// Add a response to the cache
await cache.put(
  new Request('/api/user'),
  new Response(JSON.stringify({ name: 'Alice' }), {
    headers: { 'Content-Type': 'application/json' },
  })
);

// Match a request against the cache
const response = await caches.match(new Request('/api/user'));
if (response) {
  const data = await response.json();
  console.log(data.name); // 'Alice'
}

// Add multiple URLs (fetches and caches them)
await cache.addAll(['/page1.html', '/page2.html', '/styles.css']);

// Delete a cache
await caches.delete('my-cache-v1');

// List all cache names
const names = await caches.keys();
console.log(names); // ['my-cache-v1', 'my-cache-v2']
```

### IndexedDB for Structured Data

The Cache API is great for caching HTTP responses, but for structured application data — user records, form drafts, offline queues — you need IndexedDB. It is a transactional, indexed, object-oriented database built into every browser.

The raw IndexedDB API is notoriously painful to use. Use `idb` by Jake Archibald for a Promise-based wrapper:

```bash
pnpm add idb
```

```typescript
// lib/offline-db.ts — IndexedDB with idb
import { openDB, type DBSchema, type IDBPDatabase } from 'idb';

// Define your database schema
interface AppDB extends DBSchema {
  'offline-queue': {
    key: string;
    value: {
      id: string;
      url: string;
      method: string;
      body: string;
      timestamp: number;
      retryCount: number;
    };
    indexes: {
      'by-timestamp': number;
    };
  };
  'cached-data': {
    key: string;
    value: {
      id: string;
      data: unknown;
      lastUpdated: number;
      expiresAt: number;
    };
    indexes: {
      'by-expiry': number;
    };
  };
  'draft-forms': {
    key: string;
    value: {
      id: string;
      formId: string;
      data: Record<string, unknown>;
      savedAt: number;
    };
    indexes: {
      'by-form': string;
    };
  };
}

let dbInstance: IDBPDatabase<AppDB> | null = null;

export async function getDB(): Promise<IDBPDatabase<AppDB>> {
  if (dbInstance) return dbInstance;

  dbInstance = await openDB<AppDB>('app-database', 1, {
    upgrade(db) {
      // Offline queue for pending API requests
      const offlineStore = db.createObjectStore('offline-queue', {
        keyPath: 'id',
      });
      offlineStore.createIndex('by-timestamp', 'timestamp');

      // Cached API data
      const cacheStore = db.createObjectStore('cached-data', {
        keyPath: 'id',
      });
      cacheStore.createIndex('by-expiry', 'expiresAt');

      // Draft form data
      const draftStore = db.createObjectStore('draft-forms', {
        keyPath: 'id',
      });
      draftStore.createIndex('by-form', 'formId');
    },
  });

  return dbInstance;
}

// Queue a request for when we're back online
export async function queueOfflineRequest(
  url: string,
  method: string,
  body: unknown
): Promise<void> {
  const db = await getDB();
  await db.put('offline-queue', {
    id: crypto.randomUUID(),
    url,
    method,
    body: JSON.stringify(body),
    timestamp: Date.now(),
    retryCount: 0,
  });
}

// Process the offline queue
export async function processOfflineQueue(): Promise<void> {
  const db = await getDB();
  const pending = await db.getAllFromIndex(
    'offline-queue',
    'by-timestamp'
  );

  for (const request of pending) {
    try {
      const response = await fetch(request.url, {
        method: request.method,
        headers: { 'Content-Type': 'application/json' },
        body: request.body,
      });

      if (response.ok) {
        await db.delete('offline-queue', request.id);
      } else if (request.retryCount >= 3) {
        // Give up after 3 retries
        await db.delete('offline-queue', request.id);
        console.error(`Failed to sync: ${request.url}`, response.status);
      } else {
        // Increment retry count
        await db.put('offline-queue', {
          ...request,
          retryCount: request.retryCount + 1,
        });
      }
    } catch (error) {
      // Still offline — stop processing
      break;
    }
  }
}

// Save form draft
export async function saveDraft(
  formId: string,
  data: Record<string, unknown>
): Promise<void> {
  const db = await getDB();
  await db.put('draft-forms', {
    id: `${formId}-draft`,
    formId,
    data,
    savedAt: Date.now(),
  });
}

// Load form draft
export async function loadDraft(
  formId: string
): Promise<Record<string, unknown> | null> {
  const db = await getDB();
  const draft = await db.get('draft-forms', `${formId}-draft`);
  return draft?.data ?? null;
}
```

### Offline Fallback Page

Every PWA should have an offline fallback page. When the user navigates to a page that is not cached and the network is down, they should see something better than Chrome's dinosaur:

```typescript
// In your service worker — serve offline page for uncached navigations
self.addEventListener('fetch', (event: FetchEvent) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match('/offline') || caches.match('/offline.html');
      })
    );
  }
});
```

```tsx
// app/offline/page.tsx — Next.js offline fallback page
export default function OfflinePage() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen
                    bg-gray-50 dark:bg-gray-900 p-6">
      <div className="text-6xl mb-6">📡</div>
      <h1 className="text-2xl font-bold text-gray-900 dark:text-white mb-2">
        You are offline
      </h1>
      <p className="text-gray-600 dark:text-gray-400 text-center max-w-md mb-8">
        It looks like you have lost your internet connection. Some features
        may be unavailable until you reconnect.
      </p>
      <div className="space-y-3 w-full max-w-xs">
        <button
          onClick={() => window.location.reload()}
          className="w-full px-4 py-3 bg-blue-600 text-white rounded-lg
                     font-medium hover:bg-blue-700 transition-colors"
        >
          Try again
        </button>
        <button
          onClick={() => window.history.back()}
          className="w-full px-4 py-3 bg-gray-200 dark:bg-gray-700
                     text-gray-900 dark:text-white rounded-lg
                     font-medium hover:bg-gray-300 transition-colors"
        >
          Go back
        </button>
      </div>
    </div>
  );
}
```

### Background Sync API

Background Sync lets you defer actions until the user has a stable connection. When the user submits a form offline, you queue it. When connectivity returns, the browser fires a `sync` event in your service worker — even if the user has closed the tab.

```typescript
// In your main app — register a sync event
async function submitFormWithSync(data: FormData) {
  // Save the data to IndexedDB first
  await queueOfflineRequest('/api/submit', 'POST', Object.fromEntries(data));

  // Request a background sync
  const registration = await navigator.serviceWorker.ready;
  
  if ('sync' in registration) {
    await registration.sync.register('sync-forms');
  } else {
    // Background Sync not supported — try immediately
    await processOfflineQueue();
  }
}
```

```typescript
// In your service worker — handle the sync event
self.addEventListener('sync', (event: SyncEvent) => {
  if (event.tag === 'sync-forms') {
    event.waitUntil(processOfflineQueue());
  }
});
```

### Online/Offline Detection in React

```typescript
// hooks/useOnlineStatus.ts
import { useSyncExternalStore } from 'react';

function subscribe(callback: () => void) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

function getServerSnapshot() {
  return true; // Assume online for SSR
}

export function useOnlineStatus(): boolean {
  return useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
}
```

```tsx
// Usage
function SyncStatus() {
  const isOnline = useOnlineStatus();

  return (
    <div className={`flex items-center gap-2 text-sm ${
      isOnline ? 'text-green-600' : 'text-amber-600'
    }`}>
      <div className={`w-2 h-2 rounded-full ${
        isOnline ? 'bg-green-500' : 'bg-amber-500 animate-pulse'
      }`} />
      {isOnline ? 'Synced' : 'Working offline'}
    </div>
  );
}
```

---

## 32.5 Observation & Threading APIs — The APIs You Should Already Be Using

These are the web platform APIs that separate developers who reach for npm packages from developers who reach for the platform. Every one of these is built into the browser, has zero bundle cost, and is better than most libraries that try to replicate them.

### Intersection Observer

The Intersection Observer API tells you when an element enters or exits the viewport (or any ancestor element). It is the foundation of:

- Lazy loading images
- Infinite scroll
- "Seen" tracking for analytics
- Sticky headers with state changes
- Reveal animations on scroll

Before Intersection Observer, you had to listen to scroll events, call `getBoundingClientRect()` on every element, and do the math yourself — on the main thread, synchronously, 60 times per second. It was a performance disaster.

```typescript
// hooks/useIntersectionObserver.ts
import { useEffect, useRef, useState, type RefObject } from 'react';

interface UseIntersectionObserverOptions {
  threshold?: number | number[];
  rootMargin?: string;
  root?: Element | null;
  freezeOnceVisible?: boolean;
}

export function useIntersectionObserver<T extends Element>(
  options: UseIntersectionObserverOptions = {}
): [RefObject<T | null>, IntersectionObserverEntry | null] {
  const {
    threshold = 0,
    rootMargin = '0px',
    root = null,
    freezeOnceVisible = false,
  } = options;

  const elementRef = useRef<T | null>(null);
  const [entry, setEntry] = useState<IntersectionObserverEntry | null>(null);

  const frozen = entry?.isIntersecting && freezeOnceVisible;

  useEffect(() => {
    const element = elementRef.current;
    if (!element || frozen) return;

    const observer = new IntersectionObserver(
      ([observerEntry]) => {
        setEntry(observerEntry);
      },
      { threshold, rootMargin, root }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, [threshold, rootMargin, root, frozen]);

  return [elementRef, entry];
}
```

#### Lazy Loading Images

```tsx
// components/LazyImage.tsx
import { useIntersectionObserver } from '../hooks/useIntersectionObserver';

interface LazyImageProps {
  src: string;
  alt: string;
  width: number;
  height: number;
  className?: string;
}

export function LazyImage({ src, alt, width, height, className }: LazyImageProps) {
  const [ref, entry] = useIntersectionObserver<HTMLDivElement>({
    rootMargin: '200px', // Start loading 200px before it enters viewport
    freezeOnceVisible: true,
  });

  const isVisible = entry?.isIntersecting ?? false;

  return (
    <div
      ref={ref}
      className={className}
      style={{ width, height, backgroundColor: '#f0f0f0' }}
    >
      {isVisible && (
        <img
          src={src}
          alt={alt}
          width={width}
          height={height}
          loading="lazy"
          className="w-full h-full object-cover"
        />
      )}
    </div>
  );
}
```

#### Infinite Scroll

```tsx
// hooks/useInfiniteScroll.ts
import { useCallback, useRef } from 'react';

export function useInfiniteScroll(
  loadMore: () => void,
  hasMore: boolean,
  isLoading: boolean
) {
  const observer = useRef<IntersectionObserver | null>(null);

  const sentinelRef = useCallback(
    (node: HTMLElement | null) => {
      if (isLoading) return;
      if (observer.current) observer.current.disconnect();

      observer.current = new IntersectionObserver(
        (entries) => {
          if (entries[0].isIntersecting && hasMore) {
            loadMore();
          }
        },
        { rootMargin: '400px' }
      );

      if (node) observer.current.observe(node);
    },
    [isLoading, hasMore, loadMore]
  );

  return sentinelRef;
}

// Usage in a list component
function ProductList() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage } =
    useInfiniteQuery({
      queryKey: ['products'],
      queryFn: ({ pageParam }) => fetchProducts(pageParam),
      getNextPageParam: (lastPage) => lastPage.nextCursor,
      initialPageParam: 0,
    });

  const sentinelRef = useInfiniteScroll(
    fetchNextPage,
    !!hasNextPage,
    isFetchingNextPage
  );

  return (
    <div>
      {data?.pages.flatMap((page) =>
        page.items.map((product) => (
          <ProductCard key={product.id} product={product} />
        ))
      )}
      {/* Invisible sentinel element at the bottom */}
      <div ref={sentinelRef} className="h-1" />
      {isFetchingNextPage && <LoadingSpinner />}
    </div>
  );
}
```

#### Viewport Tracking for Analytics

```typescript
// hooks/useViewportTracking.ts
import { useEffect, useRef } from 'react';

export function useViewportTracking(
  elementId: string,
  threshold = 0.5,
  minDuration = 1000
) {
  const hasTracked = useRef(false);
  const enteredAt = useRef<number | null>(null);

  useEffect(() => {
    if (hasTracked.current) return;

    const element = document.getElementById(elementId);
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          enteredAt.current = Date.now();
        } else if (enteredAt.current) {
          const duration = Date.now() - enteredAt.current;
          if (duration >= minDuration && !hasTracked.current) {
            hasTracked.current = true;
            analytics.track('element_viewed', {
              elementId,
              duration,
              intersectionRatio: entry.intersectionRatio,
            });
          }
          enteredAt.current = null;
        }
      },
      { threshold }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, [elementId, threshold, minDuration]);
}
```

### Resize Observer

Resize Observer tells you when an element's size changes. Not the window — the element. This is crucial for responsive components that need to adapt to their container size, not the viewport.

```typescript
// hooks/useResizeObserver.ts
import { useEffect, useRef, useState, type RefObject } from 'react';

interface Size {
  width: number;
  height: number;
}

export function useResizeObserver<T extends Element>(): [
  RefObject<T | null>,
  Size | null
] {
  const ref = useRef<T | null>(null);
  const [size, setSize] = useState<Size | null>(null);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new ResizeObserver((entries) => {
      const entry = entries[0];
      if (!entry) return;

      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });

    observer.observe(element);
    return () => observer.disconnect();
  }, []);

  return [ref, size];
}
```

```tsx
// Responsive chart that adapts to container width
function ResponsiveChart({ data }: { data: DataPoint[] }) {
  const [containerRef, size] = useResizeObserver<HTMLDivElement>();

  return (
    <div ref={containerRef} className="w-full">
      {size && (
        <Chart
          data={data}
          width={size.width}
          height={Math.min(size.width * 0.6, 400)}
        />
      )}
    </div>
  );
}
```

### Mutation Observer

Mutation Observer watches for changes to the DOM tree. You rarely need it in React (because React owns the DOM), but it is essential when integrating with third-party widgets, monitoring external DOM changes, or building developer tools.

```typescript
// Watch for DOM changes from a third-party widget
function watchThirdPartyWidget(container: HTMLElement) {
  const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
      if (mutation.type === 'childList') {
        // New nodes added — check if the widget has loaded
        const widget = container.querySelector('.third-party-widget');
        if (widget) {
          applyCustomStyles(widget);
          observer.disconnect();
        }
      }
      if (mutation.type === 'attributes') {
        // An attribute changed — maybe a loading state
        console.log(`${mutation.attributeName} changed on`, mutation.target);
      }
    }
  });

  observer.observe(container, {
    childList: true,     // Watch for added/removed children
    subtree: true,       // Watch all descendants, not just direct children
    attributes: true,    // Watch for attribute changes
    attributeFilter: ['class', 'data-loaded'], // Only these attributes
  });

  return () => observer.disconnect();
}
```

### Web Workers — Heavy Lifting Off the Main Thread

The main thread in a browser does everything: runs your JavaScript, handles user input, paints the screen, runs animations. When you have CPU-intensive work — parsing a large CSV, running a search algorithm, encrypting data, processing images — doing it on the main thread blocks everything else. The UI freezes. Animations stutter. The user thinks your app crashed.

Web Workers give you additional threads that run in the background. They cannot access the DOM, but they can do heavy computation without blocking the UI.

```typescript
// workers/csv-parser.worker.ts
// This file runs in its own thread

self.addEventListener('message', async (event) => {
  const { csv, options } = event.data;

  try {
    const rows = csv.split('\n');
    const headers = rows[0].split(',').map((h: string) => h.trim());
    const results: Record<string, string>[] = [];

    for (let i = 1; i < rows.length; i++) {
      if (!rows[i].trim()) continue;

      const values = rows[i].split(',');
      const row: Record<string, string> = {};

      headers.forEach((header: string, index: number) => {
        row[header] = values[index]?.trim() ?? '';
      });

      // Apply filters if provided
      if (options?.filter) {
        const passes = Object.entries(options.filter).every(
          ([key, value]) => row[key] === value
        );
        if (!passes) continue;
      }

      results.push(row);

      // Report progress every 10,000 rows
      if (i % 10000 === 0) {
        self.postMessage({
          type: 'progress',
          processed: i,
          total: rows.length,
        });
      }
    }

    self.postMessage({
      type: 'complete',
      data: results,
      totalRows: results.length,
    });
  } catch (error) {
    self.postMessage({
      type: 'error',
      message: error instanceof Error ? error.message : 'Parse failed',
    });
  }
});
```

```typescript
// hooks/useCsvParser.ts
import { useCallback, useRef, useState } from 'react';

interface ParseProgress {
  processed: number;
  total: number;
}

export function useCsvParser() {
  const [isProcessing, setIsProcessing] = useState(false);
  const [progress, setProgress] = useState<ParseProgress | null>(null);
  const workerRef = useRef<Worker | null>(null);

  const parse = useCallback(
    (csv: string, filter?: Record<string, string>) => {
      return new Promise<Record<string, string>[]>((resolve, reject) => {
        // Create a new worker
        const worker = new Worker(
          new URL('../workers/csv-parser.worker.ts', import.meta.url),
          { type: 'module' }
        );
        workerRef.current = worker;
        setIsProcessing(true);

        worker.addEventListener('message', (event) => {
          switch (event.data.type) {
            case 'progress':
              setProgress({
                processed: event.data.processed,
                total: event.data.total,
              });
              break;
            case 'complete':
              setIsProcessing(false);
              setProgress(null);
              worker.terminate();
              resolve(event.data.data);
              break;
            case 'error':
              setIsProcessing(false);
              setProgress(null);
              worker.terminate();
              reject(new Error(event.data.message));
              break;
          }
        });

        worker.postMessage({ csv, options: { filter } });
      });
    },
    []
  );

  const cancel = useCallback(() => {
    workerRef.current?.terminate();
    setIsProcessing(false);
    setProgress(null);
  }, []);

  return { parse, cancel, isProcessing, progress };
}
```

### SharedArrayBuffer and Atomics

For scenarios where multiple workers need to share memory (image processing pipelines, real-time audio processing, WASM-based computation), `SharedArrayBuffer` gives you shared memory between threads:

```typescript
// Shared memory between main thread and workers
// NOTE: Requires COOP/COEP headers:
//   Cross-Origin-Opener-Policy: same-origin
//   Cross-Origin-Embedder-Policy: require-corp

// Main thread
const sharedBuffer = new SharedArrayBuffer(1024 * 1024); // 1MB shared memory
const sharedArray = new Int32Array(sharedBuffer);

// Send the buffer to a worker (zero-copy — same memory)
worker.postMessage({ buffer: sharedBuffer });

// Atomics for thread-safe operations
Atomics.store(sharedArray, 0, 42);        // Thread-safe write
const value = Atomics.load(sharedArray, 0); // Thread-safe read
Atomics.add(sharedArray, 1, 10);            // Thread-safe increment
Atomics.wait(sharedArray, 2, 0);            // Block until value changes
Atomics.notify(sharedArray, 2);             // Wake up waiting threads
```

> **When you need this:** You almost certainly do not. SharedArrayBuffer is for high-performance computing scenarios — real-time audio processing, complex physics simulations, or when you are compiling C/C++ to WASM and need shared memory. For 99% of web apps, regular `postMessage` between workers is sufficient.

---

## 32.6 Media & Device APIs — Accessing Hardware from the Browser

The web platform has grown APIs for accessing device hardware that most frontend developers do not know exist. Here is what is available, when to use it, and what the browser support looks like.

### Geolocation API

The oldest and most well-supported device API. Every browser supports it.

```typescript
// lib/geolocation.ts

interface LocationResult {
  latitude: number;
  longitude: number;
  accuracy: number;
  timestamp: number;
}

export function getCurrentPosition(): Promise<LocationResult> {
  return new Promise((resolve, reject) => {
    if (!('geolocation' in navigator)) {
      reject(new Error('Geolocation not supported'));
      return;
    }

    navigator.geolocation.getCurrentPosition(
      (position) => {
        resolve({
          latitude: position.coords.latitude,
          longitude: position.coords.longitude,
          accuracy: position.coords.accuracy,
          timestamp: position.timestamp,
        });
      },
      (error) => {
        switch (error.code) {
          case error.PERMISSION_DENIED:
            reject(new Error('Location permission denied'));
            break;
          case error.POSITION_UNAVAILABLE:
            reject(new Error('Location unavailable'));
            break;
          case error.TIMEOUT:
            reject(new Error('Location request timed out'));
            break;
          default:
            reject(new Error('Unknown geolocation error'));
        }
      },
      {
        enableHighAccuracy: true,
        timeout: 10000,
        maximumAge: 300000, // Accept cached position up to 5 minutes old
      }
    );
  });
}

// Watch position continuously
export function watchPosition(
  onUpdate: (location: LocationResult) => void,
  onError?: (error: Error) => void
): () => void {
  const watchId = navigator.geolocation.watchPosition(
    (position) => {
      onUpdate({
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
        accuracy: position.coords.accuracy,
        timestamp: position.timestamp,
      });
    },
    (error) => {
      onError?.(new Error(error.message));
    },
    {
      enableHighAccuracy: true,
      timeout: 15000,
      maximumAge: 0,
    }
  );

  return () => navigator.geolocation.clearWatch(watchId);
}
```

```typescript
// React hook
import { useEffect, useState } from 'react';
import { getCurrentPosition, type LocationResult } from '../lib/geolocation';

export function useGeolocation() {
  const [location, setLocation] = useState<LocationResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const requestLocation = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const result = await getCurrentPosition();
      setLocation(result);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to get location');
    } finally {
      setIsLoading(false);
    }
  };

  return { location, error, isLoading, requestLocation };
}
```

### Web Notifications API

Push notifications on the web. Supported everywhere, including iOS Safari (since 16.4, but only for installed PWAs).

```typescript
// lib/notifications.ts

export async function requestNotificationPermission(): Promise<NotificationPermission> {
  if (!('Notification' in window)) {
    console.warn('Notifications not supported');
    return 'denied';
  }

  if (Notification.permission === 'granted') {
    return 'granted';
  }

  if (Notification.permission === 'denied') {
    // User previously denied — cannot re-request
    return 'denied';
  }

  // Request permission
  return Notification.requestPermission();
}

export function showNotification(
  title: string,
  options?: NotificationOptions
): Notification | null {
  if (Notification.permission !== 'granted') return null;

  return new Notification(title, {
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    ...options,
  });
}

// For PWA push notifications via service worker
export async function subscribeToPush(): Promise<PushSubscription | null> {
  const registration = await navigator.serviceWorker.ready;

  // Check if already subscribed
  const existing = await registration.pushManager.getSubscription();
  if (existing) return existing;

  // Subscribe
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true, // Must be true — silent push is not allowed
    applicationServerKey: urlBase64ToUint8Array(
      process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
    ),
  });

  // Send subscription to your server
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  });

  return subscription;
}

function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const rawData = window.atob(base64);
  return Uint8Array.from(rawData, (char) => char.charCodeAt(0));
}
```

```typescript
// In your service worker — handle push events
self.addEventListener('push', (event: PushEvent) => {
  const data = event.data?.json() ?? {
    title: 'New notification',
    body: 'You have a new update',
  };

  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192x192.png',
      badge: '/icons/badge-72x72.png',
      data: { url: data.url },
      actions: data.actions || [],
    })
  );
});

// Handle notification click
self.addEventListener('notificationclick', (event: NotificationEvent) => {
  event.notification.close();

  const url = event.notification.data?.url || '/';

  event.waitUntil(
    self.clients.matchAll({ type: 'window' }).then((clients) => {
      // Focus existing tab if open
      for (const client of clients) {
        if (client.url === url && 'focus' in client) {
          return client.focus();
        }
      }
      // Otherwise open new tab
      return self.clients.openWindow(url);
    })
  );
});
```

### Web Share API

The Web Share API gives you the native share sheet — the same one you see when you tap "Share" in a native app. It is significantly better than building your own share dialog with social media buttons, because it includes every app the user has installed.

```typescript
// lib/share.ts

interface ShareData {
  title: string;
  text?: string;
  url: string;
  files?: File[];
}

export async function shareContent(data: ShareData): Promise<boolean> {
  // Check if Web Share is supported
  if (!navigator.canShare) {
    // Fallback — copy to clipboard
    await navigator.clipboard.writeText(data.url);
    return false; // Indicate that we used fallback
  }

  // Check if we can share this specific data (including files)
  if (!navigator.canShare(data)) {
    // Data not shareable — try without files
    const { files, ...withoutFiles } = data;
    if (navigator.canShare(withoutFiles)) {
      await navigator.share(withoutFiles);
      return true;
    }
    return false;
  }

  try {
    await navigator.share(data);
    return true;
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') {
      // User cancelled — not an error
      return false;
    }
    throw error;
  }
}
```

```tsx
// components/ShareButton.tsx
function ShareButton({ title, url }: { title: string; url: string }) {
  const [copied, setCopied] = useState(false);

  const handleShare = async () => {
    const shared = await shareContent({ title, url });
    if (!shared) {
      // Fallback was used (clipboard copy)
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    }
  };

  return (
    <button onClick={handleShare} className="flex items-center gap-2">
      <ShareIcon className="w-5 h-5" />
      {copied ? 'Link copied!' : 'Share'}
    </button>
  );
}
```

### Clipboard API

Modern, async clipboard access that replaces the old `document.execCommand('copy')` hack:

```typescript
// Copy text
await navigator.clipboard.writeText('Hello, clipboard!');

// Copy rich content (HTML, images)
const blob = new Blob(['<b>Bold text</b>'], { type: 'text/html' });
await navigator.clipboard.write([
  new ClipboardItem({
    'text/html': blob,
    'text/plain': new Blob(['Bold text'], { type: 'text/plain' }),
  }),
]);

// Read text from clipboard
const text = await navigator.clipboard.readText();

// Read rich content
const items = await navigator.clipboard.read();
for (const item of items) {
  for (const type of item.types) {
    const blob = await item.getType(type);
    console.log(type, await blob.text());
  }
}
```

### Screen Wake Lock API

Prevents the screen from turning off. Essential for recipe apps, presentation modes, workout trackers, or any app where the user is looking at the screen but not touching it.

```typescript
// hooks/useWakeLock.ts
import { useCallback, useEffect, useRef, useState } from 'react';

export function useWakeLock() {
  const [isActive, setIsActive] = useState(false);
  const wakeLockRef = useRef<WakeLockSentinel | null>(null);

  const request = useCallback(async () => {
    if (!('wakeLock' in navigator)) {
      console.warn('Wake Lock not supported');
      return;
    }

    try {
      wakeLockRef.current = await navigator.wakeLock.request('screen');
      setIsActive(true);

      wakeLockRef.current.addEventListener('release', () => {
        setIsActive(false);
      });
    } catch (error) {
      console.error('Wake Lock request failed:', error);
    }
  }, []);

  const release = useCallback(async () => {
    await wakeLockRef.current?.release();
    wakeLockRef.current = null;
    setIsActive(false);
  }, []);

  // Re-acquire wake lock when page becomes visible again
  useEffect(() => {
    const handleVisibilityChange = () => {
      if (document.visibilityState === 'visible' && isActive && !wakeLockRef.current) {
        request();
      }
    };

    document.addEventListener('visibilitychange', handleVisibilityChange);
    return () => document.removeEventListener('visibilitychange', handleVisibilityChange);
  }, [isActive, request]);

  return { isActive, request, release };
}
```

### Vibration API

Simple haptic feedback. Android only (Safari does not support it).

```typescript
// Simple vibration
navigator.vibrate(200); // Vibrate for 200ms

// Pattern: vibrate, pause, vibrate, pause, vibrate
navigator.vibrate([100, 50, 100, 50, 200]);

// Stop vibration
navigator.vibrate(0);
```

### Media & Device API Support Matrix

| API | Chrome | Safari | Firefox | Use Case |
|-----|--------|--------|---------|----------|
| Geolocation | Full | Full | Full | Maps, location-based features |
| Notifications | Full | PWA only (iOS 16.4+) | Full | Alerts, messaging |
| Web Share | Full | Full | Full (Android) | Native share sheet |
| Clipboard | Full | Full | Full | Copy/paste, rich content |
| Screen Wake Lock | Full | Full (16.4+) | Full | Presentations, recipes, workouts |
| Vibration | Full | No | Full | Haptic feedback |
| getUserMedia | Full | Full | Full | Camera, microphone access |
| Media Capture | Full | Full | Full | Photo/video capture |

---

## 32.7 Advanced Web APIs — When You Need More

These APIs are more specialized. You will not use them in every project, but when you need them, they are incredibly powerful — and knowing they exist can save you from building a native app when the web would suffice.

### WebRTC — Real-Time Video and Audio

WebRTC enables peer-to-peer audio, video, and data communication directly in the browser. No plugins. No downloads. It powers Google Meet, Discord's web client, and hundreds of telehealth platforms.

```typescript
// lib/webrtc.ts — Simplified WebRTC connection

class PeerConnection {
  private pc: RTCPeerConnection;
  private localStream: MediaStream | null = null;

  constructor(
    private signalingChannel: SignalingChannel,
    private onRemoteStream: (stream: MediaStream) => void
  ) {
    this.pc = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        {
          urls: 'turn:turn.example.com:3478',
          username: 'user',
          credential: 'pass',
        },
      ],
    });

    // When we receive a remote track, pass it to the callback
    this.pc.addEventListener('track', (event) => {
      this.onRemoteStream(event.streams[0]);
    });

    // Send ICE candidates to the remote peer
    this.pc.addEventListener('icecandidate', (event) => {
      if (event.candidate) {
        this.signalingChannel.send({
          type: 'ice-candidate',
          candidate: event.candidate,
        });
      }
    });

    // Handle incoming signaling messages
    this.signalingChannel.onMessage(async (message) => {
      switch (message.type) {
        case 'offer':
          await this.pc.setRemoteDescription(message.offer);
          const answer = await this.pc.createAnswer();
          await this.pc.setLocalDescription(answer);
          this.signalingChannel.send({ type: 'answer', answer });
          break;
        case 'answer':
          await this.pc.setRemoteDescription(message.answer);
          break;
        case 'ice-candidate':
          await this.pc.addIceCandidate(message.candidate);
          break;
      }
    });
  }

  async startCall(): Promise<void> {
    // Get local media
    this.localStream = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: {
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true,
      },
    });

    // Add tracks to the connection
    this.localStream.getTracks().forEach((track) => {
      this.pc.addTrack(track, this.localStream!);
    });

    // Create and send offer
    const offer = await this.pc.createOffer();
    await this.pc.setLocalDescription(offer);
    this.signalingChannel.send({ type: 'offer', offer });
  }

  toggleMute(): boolean {
    const audioTrack = this.localStream?.getAudioTracks()[0];
    if (audioTrack) {
      audioTrack.enabled = !audioTrack.enabled;
      return audioTrack.enabled;
    }
    return false;
  }

  toggleVideo(): boolean {
    const videoTrack = this.localStream?.getVideoTracks()[0];
    if (videoTrack) {
      videoTrack.enabled = !videoTrack.enabled;
      return videoTrack.enabled;
    }
    return false;
  }

  disconnect(): void {
    this.localStream?.getTracks().forEach((track) => track.stop());
    this.pc.close();
  }
}
```

### Web Bluetooth

Connect to Bluetooth Low Energy (BLE) devices directly from the browser. Used for IoT dashboards, fitness device integrations, and hardware configuration tools.

```typescript
// lib/bluetooth.ts — Connect to a BLE heart rate monitor

export async function connectHeartRateMonitor(
  onHeartRate: (bpm: number) => void
): Promise<() => void> {
  // Request a Bluetooth device (shows browser picker)
  const device = await navigator.bluetooth.requestDevice({
    filters: [{ services: ['heart_rate'] }],
    optionalServices: ['battery_service'],
  });

  const server = await device.gatt!.connect();
  const service = await server.getPrimaryService('heart_rate');
  const characteristic = await service.getCharacteristic(
    'heart_rate_measurement'
  );

  // Start notifications
  await characteristic.startNotifications();

  const handleValue = (event: Event) => {
    const value = (event.target as BluetoothRemoteGATTCharacteristic).value!;
    // Heart rate is in the second byte (or first + second for > 255 bpm)
    const flags = value.getUint8(0);
    const is16Bit = flags & 0x01;
    const heartRate = is16Bit ? value.getUint16(1, true) : value.getUint8(1);
    onHeartRate(heartRate);
  };

  characteristic.addEventListener('characteristicvaluechanged', handleValue);

  // Return cleanup function
  return () => {
    characteristic.removeEventListener('characteristicvaluechanged', handleValue);
    device.gatt?.disconnect();
  };
}
```

> **Browser support:** Chrome and Edge on desktop and Android. Not supported in Safari or Firefox. This is a Chrome-only API in practice. Plan accordingly.

### Payment Request API

The Payment Request API provides a browser-native checkout UI. It autofills payment information, supports Apple Pay and Google Pay, and reduces checkout friction dramatically.

```typescript
// lib/payment.ts

interface PaymentItem {
  label: string;
  amount: { currency: string; value: string };
}

export async function requestPayment(
  items: PaymentItem[],
  total: PaymentItem
): Promise<PaymentResponse | null> {
  if (!('PaymentRequest' in window)) {
    return null; // Fall back to your custom checkout form
  }

  const methods: PaymentMethodData[] = [
    {
      supportedMethods: 'https://google.com/pay',
      data: {
        environment: 'PRODUCTION',
        apiVersion: 2,
        apiVersionMinor: 0,
        merchantInfo: { merchantName: 'Acme Inc' },
        allowedPaymentMethods: [
          {
            type: 'CARD',
            parameters: {
              allowedAuthMethods: ['PAN_ONLY', 'CRYPTOGRAM_3DS'],
              allowedCardNetworks: ['VISA', 'MASTERCARD', 'AMEX'],
            },
            tokenizationSpecification: {
              type: 'PAYMENT_GATEWAY',
              parameters: {
                gateway: 'stripe',
                'stripe:publishableKey': 'pk_live_...',
              },
            },
          },
        ],
      },
    },
    {
      supportedMethods: 'https://apple.com/apple-pay',
      data: {
        version: 3,
        merchantIdentifier: 'merchant.com.acme',
        merchantCapabilities: ['supports3DS'],
        supportedNetworks: ['visa', 'masterCard', 'amex'],
        countryCode: 'US',
      },
    },
  ];

  const details: PaymentDetailsInit = {
    displayItems: items,
    total,
  };

  try {
    const request = new PaymentRequest(methods, details);

    // Check if the user has a payment method available
    const canMakePayment = await request.canMakePayment();
    if (!canMakePayment) return null;

    const response = await request.show();

    // Process payment with your server
    const result = await processPaymentOnServer(response);

    if (result.success) {
      await response.complete('success');
    } else {
      await response.complete('fail');
    }

    return response;
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') {
      return null; // User cancelled
    }
    throw error;
  }
}
```

### Web NFC

Read and write NFC tags from the browser. Chrome on Android only.

```typescript
// Read NFC tags
async function readNFC() {
  if (!('NDEFReader' in window)) {
    throw new Error('NFC not supported');
  }

  const reader = new NDEFReader();
  await reader.scan();

  reader.addEventListener('reading', (event: any) => {
    console.log('NFC serial number:', event.serialNumber);
    for (const record of event.message.records) {
      console.log('Record type:', record.recordType);
      console.log('Data:', new TextDecoder().decode(record.data));
    }
  });
}

// Write to NFC tags
async function writeNFC(text: string) {
  const writer = new NDEFReader();
  await writer.write({
    records: [{ recordType: 'text', data: text }],
  });
}
```

### Credential Management API

Helps users sign in faster by working with the browser's credential manager:

```typescript
// Store credentials after successful sign-in
async function storeCredential(username: string, password: string) {
  if (!('credentials' in navigator)) return;

  const credential = new PasswordCredential({
    id: username,
    password,
    name: username,
  });

  await navigator.credentials.store(credential);
}

// Auto-fill credentials on page load
async function getStoredCredential(): Promise<PasswordCredential | null> {
  if (!('credentials' in navigator)) return null;

  const credential = await navigator.credentials.get({
    password: true,
    mediation: 'optional', // 'silent' for no UI, 'required' for always show picker
  });

  return credential as PasswordCredential | null;
}

// WebAuthn / Passkeys — the future of authentication
async function registerPasskey(username: string) {
  const challenge = await fetch('/api/auth/challenge').then((r) => r.json());

  const credential = await navigator.credentials.create({
    publicKey: {
      challenge: Uint8Array.from(challenge.challenge, (c: string) =>
        c.charCodeAt(0)
      ),
      rp: { name: 'Acme Inc', id: 'acme.com' },
      user: {
        id: Uint8Array.from(username, (c) => c.charCodeAt(0)),
        name: username,
        displayName: username,
      },
      pubKeyCredParams: [
        { type: 'public-key', alg: -7 },   // ES256
        { type: 'public-key', alg: -257 },  // RS256
      ],
      authenticatorSelection: {
        authenticatorAttachment: 'platform', // Use device biometrics
        residentKey: 'required',
        userVerification: 'required',
      },
    },
  });

  // Send credential to server for storage
  await fetch('/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(credential),
  });
}
```

### Advanced API Support Matrix

| API | Chrome | Safari | Firefox | Primary Use Case |
|-----|--------|--------|---------|-----------------|
| WebRTC | Full | Full | Full | Video/audio calls |
| Web Bluetooth | Full | No | No | IoT, fitness devices |
| Payment Request | Full | Apple Pay | Limited | Native checkout |
| Web NFC | Android only | No | No | Tag reading/writing |
| Web Serial | Full | No | No | Hardware devices |
| Credential Mgmt | Full | Passkeys | Partial | Auto sign-in, passkeys |
| File System Access | Full | No | No | Local file editing |
| Web USB | Full | No | No | USB device access |

---

## 32.8 Expo for Web — React Native Components in the Browser

### What It Is

Expo for Web uses `react-native-web` to render React Native components as HTML elements in the browser. A `<View>` becomes a `<div>`. A `<Text>` becomes a `<span>`. A `<TouchableOpacity>` becomes a clickable `<div>` with the right ARIA attributes.

This is not a gimmick. It is a production-grade strategy used by companies like Twitter (now X), which built its entire web app with react-native-web.

```
React Native Component  ──▶  react-native-web  ──▶  HTML Element
     <View>              ──▶                    ──▶  <div>
     <Text>              ──▶                    ──▶  <span>
     <Image>             ──▶                    ──▶  <img>
     <ScrollView>        ──▶                    ──▶  <div> (with overflow)
     <TextInput>         ──▶                    ──▶  <input>
     <Pressable>         ──▶                    ──▶  <div> (with handlers)
```

### Expo Router Web Support

Expo Router supports web routing out of the box. The same file-based routes that work on mobile also work on web:

```
app/
  _layout.tsx       → Root layout (both platforms)
  index.tsx         → / on web, initial screen on mobile
  about.tsx         → /about on web
  (tabs)/
    _layout.tsx     → Tab navigator on mobile, tab bar on web
    home.tsx        → /home on web
    profile.tsx     → /profile on web
  settings/
    index.tsx       → /settings on web
    [id].tsx        → /settings/123 on web
```

```typescript
// app.config.ts — Enable web in Expo
export default {
  expo: {
    name: 'MyApp',
    slug: 'my-app',
    platforms: ['ios', 'android', 'web'],
    web: {
      bundler: 'metro',
      output: 'single',      // SPA mode
      // output: 'static',   // Static site generation
      // output: 'server',   // Server-side rendering
      favicon: './assets/favicon.png',
    },
  },
};
```

### Platform-Specific Code

When you need different behavior on web vs mobile:

```typescript
// components/MediaPlayer.tsx
import { Platform } from 'react-native';

export function MediaPlayer({ url }: { url: string }) {
  if (Platform.OS === 'web') {
    // Use HTML5 video on web
    return (
      <video
        src={url}
        controls
        style={{ width: '100%', maxHeight: 400 }}
      />
    );
  }

  // Use expo-av on mobile
  const { Video } = require('expo-av');
  return (
    <Video
      source={{ uri: url }}
      useNativeControls
      style={{ width: '100%', height: 300 }}
    />
  );
}
```

```typescript
// Platform-specific file extensions
// Button.tsx         → shared (default)
// Button.web.tsx     → web only
// Button.native.tsx  → iOS and Android
// Button.ios.tsx     → iOS only
// Button.android.tsx → Android only

// The bundler automatically picks the right file
import { Button } from './Button'; // Resolves to the correct platform file
```

### When to Use Expo for Web vs. Next.js

This is the question everyone asks, and the answer is clearer than you might think:

| Factor | Expo for Web | Next.js |
|--------|-------------|---------|
| **You already have a React Native app** | Use Expo Web to share components | Build separately |
| **SEO matters** | Limited (SPA by default, SSG available) | Excellent (SSR, SSG, ISR) |
| **You need server-side rendering** | Experimental support | First-class |
| **Team knows React Native** | Natural fit | Need to learn web-specific patterns |
| **Team knows web/React** | Learning curve for RN primitives | Natural fit |
| **Performance-critical web app** | Good, but not optimized for web-specific patterns | Excellent with server components, streaming |
| **Content-heavy site** | Not ideal | Built for this |
| **Internal tool / dashboard** | Great for code sharing | Also great |
| **Complex web interactions** | Limited by RN abstraction | Full access to web APIs |
| **Shared component library** | The whole point | Need react-native-web separately |

**My recommendation:** If you have a React Native app and want to share 60-80% of your UI code on web, use Expo for Web. If you are building a web-first application — especially one that needs SEO, server-side rendering, or complex web-specific interactions — use Next.js. If you need both, use a monorepo with shared packages (see Chapter 22).

### Expo Web Limitations

Be honest with yourself about what does not work well:

1. **No native navigation animations.** React Navigation's native stack transitions do not exist on web. You get CSS-based transitions or nothing.
2. **Bundle size.** react-native-web adds overhead compared to plain React DOM. For web-first apps, this is unnecessary weight.
3. **Web-specific APIs.** You cannot use `<canvas>`, `<video>` with custom controls, or complex CSS features (like `grid` or `container queries`) through React Native primitives. You have to drop down to raw HTML.
4. **Third-party library support.** Many React Native libraries do not support web. Check before you build your architecture around code sharing.
5. **SEO.** While Expo supports static output, it does not have the sophisticated server-rendering pipeline of Next.js.
6. **Developer ecosystem.** Web tooling (React DevTools, Chrome DevTools, CSS debugging) works differently when your components are React Native primitives rendered via react-native-web.

### The Monorepo Approach — Best of Both Worlds

The most successful cross-platform teams I have worked with use a monorepo that shares business logic and design tokens, but uses platform-native UI frameworks:

```
monorepo/
  packages/
    shared/              ← Business logic, API clients, types
      src/
        api/
        hooks/
        types/
        utils/
    ui/                  ← Shared design tokens and primitives
      src/
        tokens/          ← Colors, spacing, typography
        primitives/      ← Components with .native.tsx and .web.tsx variants
    mobile/              ← Expo / React Native app
      app/
    web/                 ← Next.js app
      app/
```

This gives you code sharing where it matters (business logic, API layer, types) without forcing React Native's abstractions onto the web.

---

## 32.9 PWA vs. Native — The Decision Framework

### When to Build a PWA

Build a PWA when:

- **Distribution matters more than features.** You want anyone with a URL to access your app — no app store, no download, no approval process.
- **The app is content-focused.** News readers, blogs, documentation, dashboards, admin tools.
- **Your users visit occasionally.** Restaurant menus, event schedules, conference apps, seasonal tools.
- **You need fast iteration.** Deploy instantly. No waiting for app review. A/B test without store restrictions.
- **Budget is limited.** One codebase for all platforms. One team. One deployment pipeline.
- **Offline is nice-to-have, not critical.** PWAs support offline, but the experience is not as robust as native apps with full local storage and background processing.

### When to Go Native

Build a native app (or React Native) when:

- **You need app store distribution.** Some users only install apps from the store. Some businesses require it.
- **Hardware access is deep.** Bluetooth (beyond BLE), NFC (beyond Android Chrome), ARKit/ARCore, HealthKit/Health Connect, background location, Siri/Assistant integration.
- **Performance is extreme.** Games, real-time video processing, complex animations, heavy computation.
- **Push notifications are critical on iOS.** PWA push on iOS requires the app to be added to the home screen, and user adoption of that flow is lower than native push permission.
- **Background processing is essential.** Long-running tasks, music playback, fitness tracking, geofencing.
- **You want monetization through the stores.** In-app purchases, subscriptions via StoreKit/Play Billing.

### The Decision Matrix

| Requirement | PWA | Native (RN/Expo) | Notes |
|-------------|-----|-------------------|-------|
| No install friction | Great | Poor | PWA wins on distribution |
| App store presence | No | Yes | Some audiences expect it |
| SEO / discoverability | Excellent | None | Web content is indexable |
| Offline support | Good | Excellent | Native has more robust local storage |
| Push notifications | Good (limited iOS) | Excellent | Native is more reliable |
| Camera / microphone | Good | Excellent | Web has getUserMedia, native has more control |
| Bluetooth | BLE only, Chrome only | Full | Native wins clearly |
| Background tasks | Very limited | Full | Service workers are restricted |
| Performance ceiling | High (but lower than native) | Very high | Native has direct hardware access |
| Update speed | Instant | OTA: minutes, Store: days | PWA wins |
| Development cost | Lower | Higher | One web codebase vs platform-specific |
| Monetization | Web payments | In-app purchases | Different revenue models |

### The Progressive Enhancement Strategy

Here is the approach I recommend for most projects:

```
Level 0: RESPONSIVE WEBSITE
  A great mobile web experience.
  Fast, accessible, works everywhere.
  This is your foundation.

Level 1: ENHANCED PWA
  Add a manifest and service worker.
  Enable installation and offline support.
  Add push notifications where supported.
  Maybe 70% of your users never need more than this.

Level 2: NATIVE APP (where needed)
  Build a React Native / Expo app for the 30% who need:
  - App store distribution
  - Deep hardware access
  - Background processing
  - Native-quality animations

  Share business logic and API layer with the web app.
  Use a monorepo to keep them in sync.
```

This is not a compromise. It is a strategy. You start with the widest reach (the web), add progressive enhancement (PWA), and go native only where the web falls short. Each level builds on the last. Each level reaches a different audience.

The 100x architect does not ask "should we build a web app or a native app?" They ask "what is the minimum platform that delivers the experience our users need?" and then they build that, with a clear upgrade path to more.

---

## 32.10 Putting It All Together — A PWA Checklist

Before you ship a PWA, run through this checklist:

### Required
- [ ] Served over HTTPS
- [ ] Web App Manifest with name, icons (192x192, 512x512), start_url, display
- [ ] Service worker registered with fetch handler
- [ ] Offline fallback page
- [ ] Responsive design that works on all screen sizes
- [ ] `<meta name="viewport">` with `width=device-width`
- [ ] `<meta name="theme-color">` set

### Recommended
- [ ] Maskable icons for Android adaptive icon shapes
- [ ] Screenshots in manifest for rich install UI
- [ ] Shortcuts for quick actions from home screen
- [ ] Cache-first for static assets, stale-while-revalidate for content
- [ ] IndexedDB for structured offline data
- [ ] Service worker update flow with user notification
- [ ] Analytics tracking for PWA installs and usage
- [ ] `apple-mobile-web-app-capable` meta tags for Safari
- [ ] Apple touch icon (180x180)
- [ ] `overscroll-behavior-y: contain` to prevent pull-to-refresh conflicts

### Performance
- [ ] Core Web Vitals passing (LCP < 2.5s, INP < 200ms, CLS < 0.1)
- [ ] Service worker precaching for app shell
- [ ] Font preloading
- [ ] Image optimization (WebP/AVIF, responsive images)
- [ ] Code splitting and lazy loading for routes

### Testing
- [ ] Lighthouse PWA audit passing
- [ ] Tested on Chrome, Safari, Firefox, Edge
- [ ] Tested offline behavior
- [ ] Tested install flow on Android and iOS
- [ ] Tested service worker update flow
- [ ] Tested on slow 3G connection

---

## Summary

The web platform is far more capable than most frontend developers give it credit for. A well-built PWA can deliver an app-like experience to billions of users without requiring an app store download. Service workers give you fine-grained control over caching and offline behavior. The Web API surface — from Intersection Observer to WebRTC to the Payment Request API — covers an enormous range of capabilities.

The key insight is this: **you do not need to choose between web and native.** You need to choose the right platform for each feature. Start with the web. Add PWA capabilities where they add value. Go native where the web cannot follow. Use a monorepo to share what can be shared.

That is the 100x architect's approach to cross-platform: not "one codebase to rule them all," but "the right platform for each use case, with maximum code reuse where it matters."

---

*Next up: Chapter 33 covers web performance in depth — Core Web Vitals, rendering strategies, and the performance budgets that keep your web app fast.*
