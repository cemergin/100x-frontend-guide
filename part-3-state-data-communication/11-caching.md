<!--
  CHAPTER: 12
  TITLE: Caching Strategies — Mobile & Web
  PART: III — State, Data & Communication
  PREREQS: Chapters 9-10
  KEY_TOPICS: MMKV, AsyncStorage, TanStack Query cache, HTTP caching, Cache-Control, ETag, stale-while-revalidate, CDN architecture, Vercel Edge Config, Redis, Upstash, image caching, expo-image, offline-first, cache-first patterns
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 11: Caching Strategies — Mobile & Web

> **Part III — State, Data & Communication** | Prerequisites: Chapters 9-10 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Caching is not a bolt-on optimization; it is a first-class architectural decision that determines perceived speed, offline capability, and server costs
- Your app has five cache layers: in-memory (TanStack Query), local storage (MMKV), HTTP cache, CDN edge, and server-side (Redis/Upstash); each has different speed, lifetime, and invalidation characteristics
- MMKV is 30x faster than AsyncStorage for local persistence; use it for everything except large blobs
- HTTP caching headers (`Cache-Control`, `ETag`, `stale-while-revalidate`) are the highest-leverage optimization most teams ignore; set them correctly and the browser/CDN does the work for you
- Cache invalidation is the hard part; use time-based expiry for most data, event-driven invalidation for critical data, and never cache user-specific data on shared CDN edges

</details>

Every millisecond your user stares at a loading spinner is a millisecond they're reconsidering whether your app is worth their time. And here's the uncomfortable truth: **most of those loading spinners are unnecessary.** The data was already fetched thirty seconds ago. The image was already downloaded. The configuration hasn't changed in a week. But because nobody thought about caching — really thought about it, as a first-class architectural concern — the app hits the network again, waits for the round trip, parses the response, and finally renders what it already had.

Caching isn't a performance optimization you bolt on later. It's a fundamental architectural decision that affects your app's perceived speed, offline capability, data consistency model, and even your server costs. The difference between an app that feels instant and one that feels sluggish usually isn't faster servers or better algorithms — it's smarter caching.

This chapter is going to take you through every caching layer in a modern React Native and web application, from the device's local storage to the CDN edge. We'll cover what to cache, where to cache it, when to invalidate, and the tradeoffs you make at every level. By the end, you'll have a mental model for designing cache architectures that make your apps feel like they're reading from local memory — because, most of the time, they are.

### In This Chapter
- The Caching Mental Model — layers, lifetimes, and invalidation
- Local Storage: MMKV vs AsyncStorage (and why the 30x speed difference matters)
- TanStack Query Cache — the most important cache in your app
- HTTP Caching: Cache-Control, ETag, and stale-while-revalidate
- CDN Architecture — edge caching for global performance
- Vercel Edge Config — sub-millisecond reads at the edge
- Redis & Upstash — server-side caching for API responses
- Image Caching with expo-image
- Offline-First and Cache-First Patterns
- Cache Invalidation Strategies (the hard part)
- Building a Unified Cache Architecture

### Related Chapters
- [Ch 9: State Management at Scale] — how caching intersects with the three-category model
- [Ch 10: Data Fetching & Server Communication] — TanStack Query fetching that feeds these caches
- [Ch 12: Offline-First & Real-Time] — extending cache-first patterns to full offline support
- [Ch 13: Performance Optimization] — measuring the impact of caching decisions

---

## 1. THE CACHING MENTAL MODEL

Before we dive into tools, you need a mental model. Caching is not one thing — it's a series of layers, each with different characteristics, lifetimes, and invalidation strategies.

```
┌───────────────────────────────────────────────────────────┐
│                     YOUR USER'S DEVICE                     │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  In-Memory Cache (React state, TanStack Query)       │ │
│  │  Speed: ~0ms | Lifetime: Session | Size: ~50-200MB   │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  Local Storage (MMKV / AsyncStorage / SQLite)        │ │
│  │  Speed: 0.5-15ms | Lifetime: Persistent | Size: GBs  │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  HTTP Cache (browser cache, NSURLCache)              │ │
│  │  Speed: 1-5ms | Lifetime: Header-driven | Size: GBs  │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  Image Cache (expo-image disk cache)                 │ │
│  │  Speed: 1-10ms | Lifetime: LRU eviction | Size: GBs  │ │
│  └──────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
                          │
                          │  Network Request
                          ▼
┌───────────────────────────────────────────────────────────┐
│                      THE EDGE                              │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  CDN Cache (Vercel, Cloudflare, Fastly)              │ │
│  │  Speed: 5-30ms | Lifetime: Header-driven             │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  Edge Config (Vercel Edge Config)                    │ │
│  │  Speed: <1ms | Lifetime: Write-triggered             │ │
│  └──────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
                          │
                          │  Cache Miss
                          ▼
┌───────────────────────────────────────────────────────────┐
│                      THE ORIGIN                            │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Application Cache (Redis / Upstash)                 │ │
│  │  Speed: 1-5ms | Lifetime: TTL-based                  │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  Database Query Cache                                │ │
│  │  Speed: 5-50ms | Lifetime: Query-dependent           │ │
│  ├──────────────────────────────────────────────────────┤ │
│  │  Database                                            │ │
│  │  Speed: 5-100ms | Source of Truth                    │ │
│  └──────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

Every cache layer answers the same four questions:

1. **What do we store?** Raw bytes? Parsed objects? Rendered components?
2. **How long is it valid?** Time-based (TTL)? Event-based? Version-based?
3. **How do we invalidate?** Passive expiry? Active purge? Revalidation?
4. **What happens on a miss?** Fall through to the next layer? Show stale data? Show a loading state?

The art of caching is getting these answers right for each layer and each type of data. Let's go layer by layer.

---

## 2. LOCAL STORAGE: MMKV VS ASYNCSTORAGE

This is where most React Native developers first encounter caching, and it's where most of them make their first mistake: using AsyncStorage for everything.

### AsyncStorage: The Default That Shouldn't Be

AsyncStorage was React Native's original key-value storage solution. It's simple, it's well-documented, and it's horrifyingly slow for anything beyond trivial use cases.

```typescript
// AsyncStorage - the API is fine, the performance isn't
import AsyncStorage from '@react-native-async-storage/async-storage';

// Every operation is asynchronous (goes through the Bridge/JSI to native SQLite)
await AsyncStorage.setItem('user_preferences', JSON.stringify(preferences));
const prefs = JSON.parse(await AsyncStorage.getItem('user_preferences') || '{}');
```

Here's what's actually happening when you call `AsyncStorage.setItem()`:

1. Your JavaScript value is serialized to a JSON string
2. The string is passed through JSI to the native layer
3. The native layer opens (or reuses) a SQLite connection
4. A SQL INSERT or UPDATE statement is executed
5. SQLite flushes to disk (by default, synchronously for data integrity)
6. The native layer returns the result through JSI
7. Your Promise resolves

For a single read/write, this takes **5-15 milliseconds** on a modern device. On a lower-end Android device, it can be **30-50ms**. That doesn't sound bad until you realize that restoring your app's state on cold start might require 10-20 reads. That's 100-300ms just reading cached data — on a fast device.

### MMKV: The 30x Speed Difference

MMKV (Memory-Mapped Key-Value) is a key-value storage framework originally developed by WeChat for handling the data storage needs of an app with over a billion users. It was open-sourced by Tencent and adapted for React Native by Marc Rousavy (the same person behind react-native-vision-camera and react-native-worklets).

```typescript
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// Synchronous! No await, no promises, no callbacks
storage.set('user_preferences', JSON.stringify(preferences));
const prefs = JSON.parse(storage.getString('user_preferences') || '{}');

// Type-specific methods (no JSON serialization needed for primitives)
storage.set('onboarding_complete', true);
storage.set('launch_count', 42);
storage.set('last_sync', Date.now());

const isComplete = storage.getBoolean('onboarding_complete'); // true
const count = storage.getNumber('launch_count'); // 42
```

Why is MMKV so much faster? The answer is in the name: **memory mapping.**

```
┌─────────────────────────────────────────────┐
│  AsyncStorage (SQLite-based)                 │
│                                              │
│  App Memory → JSI → SQLite API → File I/O   │
│                                              │
│  Multiple copies of data in memory           │
│  SQL parsing overhead                        │
│  Transaction overhead                        │
│  Full ACID compliance cost                   │
│  ~5-15ms per operation                       │
├─────────────────────────────────────────────┤
│  MMKV (Memory-mapped)                        │
│                                              │
│  App Memory → mmap'd File (same address!)    │
│                                              │
│  Zero-copy reads (data IS in memory)         │
│  No SQL parsing                              │
│  Protobuf encoding (compact + fast)          │
│  CRC checksum for integrity                  │
│  ~0.03-0.5ms per operation                   │
└─────────────────────────────────────────────┘
```

MMKV uses `mmap()` to map a file directly into the process's virtual address space. When you read a value, you're reading from memory that the operating system keeps in sync with the file on disk. There's no SQL parsing, no transaction overhead, no multiple copies. The data is encoded using Protocol Buffers (protobuf), which is more compact and faster to parse than JSON.

**The benchmark numbers are staggering:**

| Operation | AsyncStorage | MMKV | Speedup |
|-----------|-------------|------|---------|
| Write 1000 items | ~4,500ms | ~140ms | 32x |
| Read 1000 items | ~3,800ms | ~65ms | 58x |
| Delete 1000 items | ~3,200ms | ~52ms | 61x |
| Contains check | ~4ms | ~0.03ms | 133x |

*Benchmarks from react-native-mmkv documentation, measured on iPhone 12.*

### Setting Up MMKV Properly

```typescript
// storage/mmkv.ts
import { MMKV } from 'react-native-mmkv';

// Default instance for general app data
export const storage = new MMKV();

// Encrypted instance for sensitive data
export const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'your-encryption-key', // In production, derive from Keychain/Keystore
});

// Separate instance for cache (can be cleared without losing user data)
export const cacheStorage = new MMKV({
  id: 'cache-storage',
});
```

### MMKV as a Zustand Persistence Layer

This is where MMKV really shines — as the persistence backend for your state management:

```typescript
// store/auth.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { storage } from '../storage/mmkv';

// Create a Zustand-compatible storage adapter for MMKV
const mmkvStorage = createJSONStorage(() => ({
  getItem: (name: string) => {
    const value = storage.getString(name);
    return value ?? null;
  },
  setItem: (name: string, value: string) => {
    storage.set(name, value);
  },
  removeItem: (name: string) => {
    storage.delete(name);
  },
}));

interface AuthStore {
  token: string | null;
  user: User | null;
  setAuth: (token: string, user: User) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      setAuth: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    {
      name: 'auth-storage',
      storage: mmkvStorage,
    }
  )
);
```

Because MMKV is synchronous, your Zustand store rehydrates instantly on app start — no loading state, no flash of unauthenticated content, no race conditions between "is the store ready?" and "render the app."

### When to Still Use AsyncStorage

There are exactly two scenarios:

1. **You're using Expo Go** (not a development build) and can't use native modules. AsyncStorage works in Expo Go; MMKV requires a development build.
2. **You need to store very large blobs** (>10MB) where MMKV's memory-mapping could cause memory pressure on low-end devices. In this case, consider SQLite instead.

For everything else, use MMKV. It's one of those rare upgrades where the migration cost is minimal and the benefit is enormous.

### MMKV with Expo

If you're using Expo (and you should be — see Chapter 5), MMKV works seamlessly with development builds:

```bash
npx expo install react-native-mmkv
npx expo prebuild  # or just run your dev build
```

No additional native configuration needed. The Expo config plugin handles everything.

---

## 3. TANSTACK QUERY CACHE

If MMKV is your persistence layer, TanStack Query is your runtime caching layer — and it's arguably the most important cache in your entire application.

We covered TanStack Query's fetching patterns in Chapter 10. Here we focus specifically on its caching behavior, because understanding the cache is the difference between "React Query works fine" and "React Query makes our app feel impossibly fast."

### The Two-Tier Cache Model

TanStack Query maintains two distinct caches:

```
┌─────────────────────────────────────────────────┐
│  QUERY CACHE (in-memory)                         │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  Active Queries (components mounted)         │ │
│  │  - Have observers (React components)         │ │
│  │  - Re-render components on updates           │ │
│  │  - Kept fresh via staleTime / refetch        │ │
│  ├─────────────────────────────────────────────┤ │
│  │  Inactive Queries (no mounted components)    │ │
│  │  - No observers, but data still in memory    │ │
│  │  - Kept for gcTime (default: 5 minutes)      │ │
│  │  - Instant cache hit if component remounts   │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Two critical timers control this:**

- **`staleTime`**: How long data is considered "fresh." While fresh, no refetch happens even if the component remounts or the window refocuses. Default is `0` (always stale).
- **`gcTime`** (garbage collection time, formerly `cacheTime`): How long inactive data stays in memory. Once a query has no observers (no mounted components using it) for this duration, it's garbage collected. Default is `5 minutes`.

This is where most teams mess up. They leave `staleTime` at `0`, which means every time a component mounts, it triggers a background refetch — even if the data was fetched 100ms ago.

### Configuring staleTime for Real Apps

```typescript
// lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Default: data is considered fresh for 30 seconds
      // This prevents redundant refetches on navigation
      staleTime: 30 * 1000,

      // Keep inactive data in memory for 10 minutes
      gcTime: 10 * 60 * 1000,

      // Don't retry on 4xx errors (they won't magically succeed)
      retry: (failureCount, error) => {
        if (error instanceof ApiError && error.status >= 400 && error.status < 500) {
          return false;
        }
        return failureCount < 3;
      },
    },
  },
});
```

But staleTime should vary by data type. Static reference data can be stale for hours. Real-time notifications should be stale immediately:

```typescript
// Query-specific staleTime examples

// User profile: doesn't change often, 5 minute staleTime
const { data: profile } = useQuery({
  queryKey: ['user', 'profile'],
  queryFn: fetchProfile,
  staleTime: 5 * 60 * 1000,
});

// Product catalog: changes rarely, cache aggressively
const { data: products } = useQuery({
  queryKey: ['products', categoryId],
  queryFn: () => fetchProducts(categoryId),
  staleTime: 30 * 60 * 1000, // 30 minutes
});

// Notifications: should always refetch
const { data: notifications } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  staleTime: 0, // Always stale, always refetch
  refetchInterval: 30 * 1000, // Poll every 30 seconds
});

// Feature flags: cache for the entire session
const { data: flags } = useQuery({
  queryKey: ['feature-flags'],
  queryFn: fetchFeatureFlags,
  staleTime: Infinity, // Never stale during this session
});
```

### Persisting the Query Cache to MMKV

Here's where things get powerful. TanStack Query has a persistence plugin that lets you save the entire query cache to disk and restore it on app start. Combined with MMKV, this means your app can show cached data *instantly* — before any network request completes.

```typescript
// lib/query-persister.ts
import { createSyncStoragePersister } from '@tanstack/query-sync-storage-persister';
import { MMKV } from 'react-native-mmkv';

const queryStorage = new MMKV({ id: 'tanstack-query-cache' });

export const queryPersister = createSyncStoragePersister({
  storage: {
    getItem: (key: string) => queryStorage.getString(key) ?? null,
    setItem: (key: string, value: string) => queryStorage.set(key, value),
    removeItem: (key: string) => queryStorage.delete(key),
  },
  // Only persist data that's less than 24 hours old
  // This prevents stale cache from accumulating indefinitely
  serialize: (data) => JSON.stringify(data),
  deserialize: (data) => JSON.parse(data),
});
```

```typescript
// App.tsx
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { queryClient } from './lib/query-client';
import { queryPersister } from './lib/query-persister';

export default function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: queryPersister,
        maxAge: 24 * 60 * 60 * 1000, // Don't restore data older than 24 hours
        buster: APP_VERSION, // Invalidate cache on app update
        dehydrateOptions: {
          shouldDehydrateQuery: (query) => {
            // Don't persist error states or non-success queries
            if (query.state.status !== 'success') return false;

            // Don't persist queries marked as non-cacheable
            const meta = query.meta as { persist?: boolean } | undefined;
            if (meta?.persist === false) return false;

            return true;
          },
        },
      }}
    >
      <YourApp />
    </PersistQueryClientProvider>
  );
}
```

**The user experience impact is dramatic.** On cold start:

1. MMKV loads the persisted query cache synchronously (~1-5ms)
2. TanStack Query hydrates from the cache — all screens show data immediately
3. Background refetches fire for stale data
4. UI updates silently when fresh data arrives

The user sees content instantly. No loading spinners. No skeleton screens. The app feels like it was never closed.

### Cache Prefetching

Don't wait for the user to navigate — prefetch data for likely destinations:

```typescript
// Prefetch on hover (web) or on screen focus (mobile)
function ProductListItem({ product }: { product: Product }) {
  const queryClient = useQueryClient();

  const prefetchDetails = useCallback(() => {
    queryClient.prefetchQuery({
      queryKey: ['product', product.id],
      queryFn: () => fetchProductDetails(product.id),
      staleTime: 5 * 60 * 1000,
    });
  }, [product.id, queryClient]);

  return (
    <Pressable
      onPress={() => navigate('ProductDetail', { id: product.id })}
      // On web: prefetch on hover
      // On mobile: prefetch when item becomes visible
      onHoverIn={prefetchDetails}
      onLayout={prefetchDetails}
    >
      <ProductCard product={product} />
    </Pressable>
  );
}
```

### Selective Cache Invalidation

Cache invalidation is the hard part. TanStack Query gives you surgical tools:

```typescript
const queryClient = useQueryClient();

// Invalidate a specific query
queryClient.invalidateQueries({ queryKey: ['product', productId] });

// Invalidate all products (any query key starting with 'products')
queryClient.invalidateQueries({ queryKey: ['products'] });

// Invalidate everything for a user (fuzzy matching)
queryClient.invalidateQueries({ queryKey: ['user'] });

// Optimistic update with rollback
const updateProduct = useMutation({
  mutationFn: (update: ProductUpdate) => api.updateProduct(update),
  onMutate: async (update) => {
    // Cancel any outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['product', update.id] });

    // Snapshot the previous value
    const previous = queryClient.getQueryData(['product', update.id]);

    // Optimistically update the cache
    queryClient.setQueryData(['product', update.id], (old: Product) => ({
      ...old,
      ...update,
    }));

    return { previous };
  },
  onError: (err, update, context) => {
    // Rollback on error
    queryClient.setQueryData(['product', update.id], context?.previous);
  },
  onSettled: () => {
    // Refetch to ensure consistency
    queryClient.invalidateQueries({ queryKey: ['product', update.id] });
  },
});
```

---

## 4. HTTP CACHING: CACHE-CONTROL, ETAG, AND STALE-WHILE-REVALIDATE

HTTP caching is the oldest and most battle-tested caching layer in the stack. It's also the most misunderstood by frontend developers. Most React Native and React developers have a vague sense that "the browser caches things" but couldn't tell you the difference between `max-age` and `stale-while-revalidate`, or when an ETag saves bandwidth versus when it doesn't.

### Cache-Control: The Director

The `Cache-Control` header is how your server tells caches (browser, CDN, proxy) what to do:

```
Cache-Control: public, max-age=3600, stale-while-revalidate=86400
```

Let's break this down:

```
┌─────────────────────────────────────────────────────────────┐
│  Cache-Control Directives                                    │
│                                                              │
│  STORAGE:                                                    │
│  public    → Any cache can store this (CDN, browser)        │
│  private   → Only the end user's cache (not CDN)            │
│  no-store  → Don't cache at all                             │
│  no-cache  → Cache it, but revalidate every time            │
│                                                              │
│  FRESHNESS:                                                  │
│  max-age=N        → Fresh for N seconds                     │
│  s-maxage=N       → CDN-specific max-age (overrides)        │
│  stale-while-revalidate=N → Serve stale for N seconds       │
│                              while revalidating in bg        │
│  stale-if-error=N → Serve stale if origin returns error     │
│                                                              │
│  VALIDATION:                                                 │
│  must-revalidate  → Once stale, MUST revalidate (no stale)  │
│  immutable        → Never revalidate (content never changes)│
└─────────────────────────────────────────────────────────────┘
```

### The stale-while-revalidate Pattern

This is the HTTP caching directive that changed everything. It's the HTTP-level equivalent of TanStack Query's "show stale data while refetching in the background":

```
Cache-Control: public, max-age=60, stale-while-revalidate=3600
```

Here's what this means in practice:

```
Time 0:        Request → Cache MISS → Fetch from origin → Cache response
                (User waits for origin response)

Time 30s:      Request → Cache HIT (fresh) → Return immediately
                (Data is only 30s old, within max-age=60)

Time 90s:      Request → Cache HIT (stale) → Return stale immediately
                                              → Background revalidation to origin
                (Data is 90s old, past max-age but within stale-while-revalidate)
                (User gets instant response with slightly stale data)

Time 3700s:    Request → Cache MISS → Fetch from origin
                (Data is past stale-while-revalidate window)
                (User must wait for origin response)
```

**For your API endpoints, this is the most common pattern you should reach for:**

```typescript
// Next.js API route with stale-while-revalidate
export async function GET(request: Request) {
  const products = await fetchProducts();

  return Response.json(products, {
    headers: {
      // Fresh for 1 minute, serve stale for up to 1 hour
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=3600',
    },
  });
}
```

### ETag: Content-Based Validation

While `Cache-Control` is time-based ("this data is valid for N seconds"), ETags are content-based ("this data matches this fingerprint"):

```
// Server response
HTTP/1.1 200 OK
ETag: "abc123def456"
Cache-Control: no-cache

// Subsequent client request
GET /api/products
If-None-Match: "abc123def456"

// Server response (data hasn't changed)
HTTP/1.1 304 Not Modified
// No body! Saves bandwidth.

// Server response (data changed)
HTTP/1.1 200 OK
ETag: "xyz789"
// Full response body with new data
```

ETags save bandwidth (304 responses have no body), but they don't save the round trip. The client still has to ask the server "is my cached version still valid?" — which means network latency is still in play.

**When to use ETags vs Cache-Control:**

| Scenario | Use |
|----------|-----|
| Static assets (JS, CSS, images) | `Cache-Control: immutable, max-age=31536000` + content hash in filename |
| API responses that change unpredictably | ETag + `Cache-Control: no-cache` |
| API responses with known update frequency | `Cache-Control: max-age=N, stale-while-revalidate=M` |
| Personalized data (user-specific) | `Cache-Control: private, max-age=N` + ETag |
| Sensitive data (financial, medical) | `Cache-Control: no-store` |

### HTTP Caching in React Native

Here's something that trips up React Native developers: **React Native's `fetch()` respects HTTP cache headers on iOS but not consistently on Android.**

On iOS, `NSURLCache` handles HTTP caching automatically. On Android, the behavior depends on the networking library (OkHttp by default) and its configuration.

To get consistent caching behavior across platforms:

```typescript
// Use React Native's cache mode for platform-aware behavior
const response = await fetch('https://api.example.com/products', {
  headers: {
    'If-None-Match': cachedETag, // Manual ETag handling
  },
  cache: 'default', // or 'no-cache', 'force-cache', etc.
});

// Better approach: let TanStack Query handle the caching layer
// and let HTTP caching handle the network layer
// The two layers complement each other perfectly
```

In practice, most React Native apps should rely on TanStack Query for application-level caching and treat HTTP caching as a bonus optimization that reduces bandwidth usage. Don't build your app's caching strategy around HTTP headers alone — they're not reliable enough across platforms.

---

## 5. CDN ARCHITECTURE

A Content Delivery Network (CDN) is a geographically distributed network of servers that cache and serve content from locations close to your users. If your API server is in `us-east-1` and your user is in Tokyo, a CDN edge node in Tokyo can serve cached responses in 10-30ms instead of 200-300ms.

### How CDN Caching Works

```
User in Tokyo                User in London
     │                            │
     ▼                            ▼
┌─────────┐                 ┌─────────┐
│ Tokyo    │                │ London   │
│ Edge     │                │ Edge     │
│ Node     │                │ Node     │
└────┬─────┘                └────┬─────┘
     │ Cache MISS                │ Cache HIT
     ▼                           │ (Return cached)
┌─────────┐                      │
│ Origin   │◄─── Cache fill ─────┘
│ Server   │     (first request from London edge)
│ us-east-1│
└──────────┘
```

### CDN Caching for API Responses

Most developers think of CDNs for static assets (JS bundles, images), but caching API responses at the edge is where the real wins are for mobile apps:

```typescript
// Next.js API route with CDN-aware caching
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const category = searchParams.get('category');

  const products = await db.products.findMany({
    where: { category },
  });

  return Response.json(products, {
    headers: {
      // s-maxage: CDN-specific (browser uses max-age)
      // This caches at the edge for 5 minutes
      // but the browser only caches for 1 minute
      'Cache-Control': 'public, max-age=60, s-maxage=300, stale-while-revalidate=3600',

      // Vary header tells CDN to cache different versions
      // based on these request headers
      'Vary': 'Accept-Language',
    },
  });
}
```

### Cache Invalidation at the Edge

The hardest problem with CDN caching is invalidation. When your data changes, how do you tell every edge node worldwide to drop its cached version?

**Option 1: Short TTLs**
```
Cache-Control: s-maxage=60, stale-while-revalidate=3600
```
Data is stale within a minute. Simple, but you accept up to 60 seconds of staleness.

**Option 2: On-Demand Revalidation (Vercel/Next.js)**
```typescript
// Trigger revalidation from a webhook or mutation
import { revalidatePath, revalidateTag } from 'next/cache';

// When a product is updated via webhook
export async function POST(request: Request) {
  const { productId } = await request.json();

  // Purge all cached responses tagged with this product
  revalidateTag(`product-${productId}`);

  // Or purge a specific path
  revalidatePath(`/products/${productId}`);

  return Response.json({ revalidated: true });
}
```

**Option 3: Cache Tags (Cloudflare, Fastly)**
```typescript
// Tag responses so you can purge them selectively
return new Response(JSON.stringify(data), {
  headers: {
    'Cache-Tag': `product-${productId}, category-${categoryId}`,
    'Surrogate-Key': `product-${productId} category-${categoryId}`, // Fastly
  },
});
```

### Vercel's Edge Caching Model

If you're deploying on Vercel (and for web frontends, you probably should be), here's how their caching works:

```
┌──────────────────────────────────────────────────────┐
│  Vercel Edge Network                                  │
│                                                       │
│  Request → Edge Cache Check                          │
│              │                                        │
│              ├── HIT (fresh) → Return cached          │
│              │                                        │
│              ├── HIT (stale, SWR window) →            │
│              │   Return stale + background revalidate │
│              │                                        │
│              └── MISS → Run Serverless Function       │
│                          → Cache response at edge     │
│                          → Return to client           │
│                                                       │
│  ISR (Incremental Static Regeneration):               │
│  - Static pages cached at edge                        │
│  - Revalidated on-demand or by time interval          │
│  - First visitor after revalidation sees stale        │
│  - Background regeneration for next visitor           │
└──────────────────────────────────────────────────────┘
```

---

## 6. VERCEL EDGE CONFIG

Edge Config is a global, low-latency data store designed specifically for reads that need to be sub-millisecond at the edge. It's not a general-purpose cache — it's for configuration data that your edge functions and middleware need to access without hitting an origin server.

### What Goes in Edge Config

```
┌─────────────────────────────────────────────────┐
│  Good candidates for Edge Config:                │
│  - Feature flags                                 │
│  - A/B test assignments                          │
│  - IP/country-based redirect rules               │
│  - Rate limiting configuration                   │
│  - Maintenance mode flags                        │
│  - API key rotation (which key is active)        │
│  - Regional configuration                        │
│                                                   │
│  Bad candidates for Edge Config:                  │
│  - User data (use a database)                    │
│  - Frequently changing data (>100 writes/day)    │
│  - Large datasets (Edge Config has size limits)  │
│  - Data that needs complex queries               │
└─────────────────────────────────────────────────┘
```

### Using Edge Config

```typescript
// middleware.ts (runs at the edge, before your app)
import { NextRequest, NextResponse } from 'next/server';
import { get } from '@vercel/edge-config';

export async function middleware(request: NextRequest) {
  // Sub-millisecond read at the edge — data is colocated
  // with the function, not fetched over the network
  const maintenanceMode = await get<boolean>('maintenance_mode');

  if (maintenanceMode) {
    return NextResponse.rewrite(new URL('/maintenance', request.url));
  }

  // Feature flag check at the edge
  const featureFlags = await get<Record<string, boolean>>('feature_flags');

  if (featureFlags?.['new-checkout'] && request.nextUrl.pathname === '/checkout') {
    return NextResponse.rewrite(new URL('/checkout-v2', request.url));
  }

  return NextResponse.next();
}
```

### Edge Config for React Native Apps

While Edge Config is a Vercel/web concept, your React Native app benefits from it indirectly:

```typescript
// Your API endpoint reads Edge Config to make decisions
// that affect what data your React Native app receives

// api/config/route.ts
import { get } from '@vercel/edge-config';

export async function GET() {
  const [featureFlags, appConfig] = await Promise.all([
    get('feature_flags'),
    get('mobile_app_config'),
  ]);

  return Response.json({
    featureFlags,
    appConfig,
  }, {
    headers: {
      'Cache-Control': 'public, max-age=300, stale-while-revalidate=3600',
    },
  });
}

// In your React Native app
const { data: config } = useQuery({
  queryKey: ['app-config'],
  queryFn: fetchAppConfig,
  staleTime: 5 * 60 * 1000,
});
```

---

## 7. REDIS & UPSTASH: SERVER-SIDE CACHING

When your API endpoints need to cache expensive computations, database query results, or third-party API responses, Redis is the standard answer. And Upstash makes it serverless-friendly.

### Why Redis for API Caching

```
Without Redis:
  Client → API Route → Database Query (50ms) → Response
  Client → API Route → Database Query (50ms) → Response
  Client → API Route → Database Query (50ms) → Response
  (Every request hits the database)

With Redis:
  Client → API Route → Redis GET (1ms) → Response (HIT)
  Client → API Route → Redis MISS → Database (50ms) → Redis SET → Response
  Client → API Route → Redis GET (1ms) → Response (HIT)
  (Most requests served from Redis)
```

### Upstash Redis Setup

Upstash provides serverless Redis that works well with edge and serverless deployments (pay-per-request, HTTP-based protocol, no persistent connections needed):

```typescript
// lib/redis.ts
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Cache wrapper with TTL
export async function cached<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttlSeconds: number = 300
): Promise<T> {
  // Try cache first
  const cached = await redis.get<T>(key);
  if (cached !== null) {
    return cached;
  }

  // Cache miss — fetch from source
  const data = await fetcher();

  // Store in cache with TTL
  await redis.set(key, data, { ex: ttlSeconds });

  return data;
}
```

### Caching Patterns with Upstash

```typescript
// api/products/[id]/route.ts
import { cached, redis } from '@/lib/redis';

export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const product = await cached(
    `product:${params.id}`,
    () => db.products.findUnique({ where: { id: params.id } }),
    600 // Cache for 10 minutes
  );

  if (!product) {
    return Response.json({ error: 'Not found' }, { status: 404 });
  }

  return Response.json(product);
}

// When a product is updated, invalidate the cache
export async function PATCH(
  request: Request,
  { params }: { params: { id: string } }
) {
  const update = await request.json();
  const product = await db.products.update({
    where: { id: params.id },
    data: update,
  });

  // Invalidate the cache
  await redis.del(`product:${params.id}`);

  // Also invalidate any list caches that might contain this product
  const listKeys = await redis.keys('products:list:*');
  if (listKeys.length > 0) {
    await redis.del(...listKeys);
  }

  return Response.json(product);
}
```

### Cache-Aside Pattern (Read-Through)

The most common pattern for Redis caching:

```typescript
// lib/cache-patterns.ts
import { redis } from './redis';

// Cache-aside with stale-while-revalidate semantics
export async function cacheAside<T>(
  key: string,
  fetcher: () => Promise<T>,
  options: {
    freshTtl: number;    // How long data is considered fresh
    staleTtl: number;    // How long stale data can be served while revalidating
  }
): Promise<T> {
  const cached = await redis.get<{
    data: T;
    fetchedAt: number;
  }>(key);

  const now = Date.now();

  // Fresh cache hit
  if (cached && (now - cached.fetchedAt) < options.freshTtl * 1000) {
    return cached.data;
  }

  // Stale cache hit — return stale data but revalidate in background
  if (cached && (now - cached.fetchedAt) < options.staleTtl * 1000) {
    // Background revalidation (don't await)
    fetcher().then(async (fresh) => {
      await redis.set(key, {
        data: fresh,
        fetchedAt: Date.now(),
      }, { ex: options.staleTtl });
    }).catch(console.error);

    return cached.data;
  }

  // Cache miss or expired — must fetch
  const data = await fetcher();
  await redis.set(key, {
    data,
    fetchedAt: Date.now(),
  }, { ex: options.staleTtl });

  return data;
}
```

### Upstash Rate Limiting (Bonus Cache Use Case)

Rate limiting is essentially a cache pattern — you're caching request counts:

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { redis } from '@/lib/redis';

const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(100, '1 m'), // 100 requests per minute
  analytics: true,
});

// In your API route or middleware
export async function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success, limit, remaining, reset } = await ratelimit.limit(ip);

  if (!success) {
    return Response.json(
      { error: 'Too many requests' },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
        },
      }
    );
  }

  return NextResponse.next();
}
```

---

## 8. IMAGE CACHING WITH EXPO-IMAGE

Images are usually the largest assets your app downloads, and they're the most visible when caching fails. A product image that loads instantly from cache vs. one that shows a loading placeholder for 500ms is the difference between an app that feels polished and one that feels broken.

### Why expo-image Instead of React Native's Image

React Native's built-in `<Image>` component has mediocre caching behavior:

- On iOS, it uses `NSURLCache` which has a relatively small default size and can be evicted unpredictably
- On Android, it uses Fresco with a complex and hard-to-configure cache hierarchy
- Neither platform gives you good control over cache policies
- No built-in support for modern image formats (AVIF, WebP) with automatic negotiation

`expo-image` (powered by SDWebImage on iOS and Glide on Android) gives you:

```typescript
import { Image } from 'expo-image';

// Basic usage with automatic caching
<Image
  source={{ uri: 'https://cdn.example.com/product-123.jpg' }}
  style={{ width: 200, height: 200 }}
  // Cache policy: how aggressively to cache
  cachePolicy="memory-disk" // Default: check memory first, then disk
  // Placeholder: what to show while loading
  placeholder={blurhash}
  // Transition: how to animate from placeholder to loaded image
  transition={200}
  // Content fit: like CSS object-fit
  contentFit="cover"
/>
```

### Cache Policies

```typescript
// Memory + Disk (default): fastest, highest storage use
<Image cachePolicy="memory-disk" source={{ uri: imageUrl }} />

// Disk only: saves memory, still fast
<Image cachePolicy="disk" source={{ uri: imageUrl }} />

// Memory only: doesn't persist between sessions
<Image cachePolicy="memory" source={{ uri: imageUrl }} />

// None: always fetch from network
<Image cachePolicy="none" source={{ uri: imageUrl }} />
```

### Blurhash Placeholders

The visual experience of image loading matters enormously. A blank space or a generic placeholder is jarring. Blurhash generates a compact representation (20-30 characters) that can be rendered as a beautiful blurry placeholder:

```typescript
// Server-side: generate blurhash when uploading images
import { encode } from 'blurhash';
import sharp from 'sharp';

async function generateBlurhash(imagePath: string): Promise<string> {
  const { data, info } = await sharp(imagePath)
    .raw()
    .ensureAlpha()
    .resize(32, 32, { fit: 'inside' })
    .toBuffer({ resolveWithObject: true });

  return encode(
    new Uint8ClampedArray(data),
    info.width,
    info.height,
    4, // x components
    3  // y components
  );
}

// Store the blurhash alongside the image URL in your database
// { imageUrl: 'https://...', blurhash: 'LEHV6nWB2yk8pyo0adR*.7kCMdnj' }
```

```typescript
// Client-side: use blurhash as placeholder
function ProductImage({ product }: { product: Product }) {
  return (
    <Image
      source={{ uri: product.imageUrl }}
      placeholder={{ blurhash: product.blurhash }}
      transition={300}
      contentFit="cover"
      style={{ width: '100%', aspectRatio: 1 }}
      recyclingKey={product.id} // Helps with FlashList recycling
    />
  );
}
```

### Image Prefetching

For critical images (hero images, above-the-fold content), prefetch before the component renders:

```typescript
import { Image } from 'expo-image';

// Prefetch a single image
await Image.prefetch('https://cdn.example.com/hero.jpg');

// Prefetch multiple images
await Image.prefetch([
  'https://cdn.example.com/product-1.jpg',
  'https://cdn.example.com/product-2.jpg',
  'https://cdn.example.com/product-3.jpg',
]);

// Prefetch on screen focus (e.g., before navigating)
function useImagePrefetch(urls: string[]) {
  const isFocused = useIsFocused();

  useEffect(() => {
    if (isFocused) {
      Image.prefetch(urls);
    }
  }, [isFocused, urls]);
}
```

### Cache Size Management

```typescript
// Clear the image cache (useful for "clear cache" settings)
await Image.clearDiskCache();
await Image.clearMemoryCache();
```

### CDN Image Optimization

Don't just cache images — optimize them at the CDN level:

```typescript
// Use Vercel Image Optimization (or Cloudinary, Imgix, etc.)
function OptimizedImage({ src, width, height }: ImageProps) {
  // For React Native, use a CDN transformation URL
  const mobileUrl = `https://cdn.example.com/img/${width}x${height}/${src}`;

  return (
    <Image
      source={{ uri: mobileUrl }}
      style={{ width, height }}
      contentFit="cover"
    />
  );
}
```

---

## 9. OFFLINE-FIRST AND CACHE-FIRST PATTERNS

Cache-first is a spectrum. At one end, you show cached data while refreshing in the background (stale-while-revalidate). At the other end, your app works fully offline and syncs when connectivity returns. Where you land on this spectrum depends on your use case.

### The Cache-First Spectrum

```
    ← Less Offline Capability                    More Offline Capability →

    ┌───────────┬──────────────┬───────────────┬──────────────┐
    │ Network   │ Stale-While  │ Offline       │ Offline      │
    │ First     │ Revalidate   │ Readable      │ Read/Write   │
    │           │              │               │              │
    │ Always    │ Show cache,  │ Full app      │ Full app     │
    │ fetch,    │ refetch in   │ works offline │ works offline│
    │ show      │ background   │ for reads,    │ including    │
    │ loading   │              │ queue writes  │ mutations    │
    │ spinner   │              │               │              │
    │           │              │               │              │
    │ Simple    │ Good UX,     │ Great UX,     │ Complex,     │
    │ but poor  │ good         │ moderate      │ needs        │
    │ UX        │ balance      │ complexity    │ conflict     │
    │           │              │               │ resolution   │
    └───────────┴──────────────┴───────────────┴──────────────┘
```

### Implementing Cache-First with TanStack Query + MMKV

For most apps, the "stale-while-revalidate" level is the right balance:

```typescript
// hooks/useCachedQuery.ts
import { useQuery, UseQueryOptions } from '@tanstack/react-query';

/**
 * A query hook that always shows cached data immediately,
 * then silently updates in the background.
 */
export function useCachedQuery<T>(
  options: UseQueryOptions<T> & { queryKey: readonly unknown[]; queryFn: () => Promise<T> }
) {
  return useQuery({
    ...options,
    // Show stale data immediately (no loading state for cache hits)
    staleTime: 0,
    // Keep data in cache for 24 hours
    gcTime: 24 * 60 * 60 * 1000,
    // Don't show loading state if we have cached data
    placeholderData: (previousData) => previousData,
    // Refetch strategies
    refetchOnMount: true,
    refetchOnWindowFocus: true,
    refetchOnReconnect: true,
  });
}
```

### Detecting Connectivity

For offline-aware patterns, you need to know when the device is online:

```typescript
// hooks/useNetworkState.ts
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';
import { onlineManager } from '@tanstack/react-query';

// Tell TanStack Query about connectivity changes
// This pauses/resumes queries and mutations automatically
NetInfo.addEventListener((state) => {
  onlineManager.setOnline(
    state.isConnected != null &&
    state.isConnected &&
    Boolean(state.isInternetReachable)
  );
});

export function useNetworkState() {
  const [networkState, setNetworkState] = useState<NetInfoState | null>(null);

  useEffect(() => {
    return NetInfo.addEventListener(setNetworkState);
  }, []);

  return {
    isConnected: networkState?.isConnected ?? true,
    isInternetReachable: networkState?.isInternetReachable ?? true,
    type: networkState?.type,
  };
}
```

### Offline Mutation Queue

When the user performs mutations while offline, queue them and replay when connectivity returns:

```typescript
// lib/offline-mutations.ts
import { storage } from './mmkv';
import NetInfo from '@react-native-community/netinfo';

// Persist failed mutations to MMKV
const MUTATION_QUEUE_KEY = 'offline-mutation-queue';

interface QueuedMutation {
  id: string;
  endpoint: string;
  method: 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  body: unknown;
  timestamp: number;
  retryCount: number;
}

export function getQueuedMutations(): QueuedMutation[] {
  const raw = storage.getString(MUTATION_QUEUE_KEY);
  return raw ? JSON.parse(raw) : [];
}

export function queueMutation(mutation: Omit<QueuedMutation, 'id' | 'timestamp' | 'retryCount'>) {
  const queue = getQueuedMutations();
  queue.push({
    ...mutation,
    id: `${Date.now()}_${Math.random().toString(36).slice(2)}`,
    timestamp: Date.now(),
    retryCount: 0,
  });
  storage.set(MUTATION_QUEUE_KEY, JSON.stringify(queue));
}

export function removeMutation(id: string) {
  const queue = getQueuedMutations().filter(m => m.id !== id);
  storage.set(MUTATION_QUEUE_KEY, JSON.stringify(queue));
}

// Replay queued mutations when connectivity returns
export function setupOfflineMutationReplay() {
  NetInfo.addEventListener(async (state) => {
    if (!state.isConnected || !state.isInternetReachable) return;

    const queue = getQueuedMutations();
    if (queue.length === 0) return;

    console.log(`Replaying ${queue.length} queued mutations`);

    for (const mutation of queue) {
      try {
        await fetch(mutation.endpoint, {
          method: mutation.method,
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(mutation.body),
        });
        removeMutation(mutation.id);
      } catch (error) {
        console.error(`Failed to replay mutation ${mutation.id}:`, error);
        // Don't remove — will retry on next connectivity change
      }
    }
  });
}
```

### Optimistic UI with Offline Support

The user doesn't care that they're offline. They expect the app to work:

```typescript
// hooks/useOfflineMutation.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { onlineManager } from '@tanstack/react-query';
import { queueMutation } from '../lib/offline-mutations';

export function useCreateTodo() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newTodo: CreateTodoInput) => {
      if (!onlineManager.isOnline()) {
        // Queue for later and return optimistic result
        queueMutation({
          endpoint: '/api/todos',
          method: 'POST',
          body: newTodo,
        });
        return {
          ...newTodo,
          id: `temp-${Date.now()}`,
          createdAt: new Date().toISOString(),
          _offline: true,
        };
      }
      return api.createTodo(newTodo);
    },
    onMutate: async (newTodo) => {
      await queryClient.cancelQueries({ queryKey: ['todos'] });
      const previous = queryClient.getQueryData(['todos']);

      // Optimistically add the todo
      queryClient.setQueryData(['todos'], (old: Todo[] = []) => [
        {
          ...newTodo,
          id: `temp-${Date.now()}`,
          createdAt: new Date().toISOString(),
          _offline: !onlineManager.isOnline(),
        },
        ...old,
      ]);

      return { previous };
    },
    onError: (err, newTodo, context) => {
      queryClient.setQueryData(['todos'], context?.previous);
    },
    onSettled: () => {
      if (onlineManager.isOnline()) {
        queryClient.invalidateQueries({ queryKey: ['todos'] });
      }
    },
  });
}
```

---

## 10. CACHE INVALIDATION STRATEGIES

Phil Karlton famously said there are only two hard things in computer science: cache invalidation and naming things. He was right. Cache invalidation is where elegant caching architectures turn into debugging nightmares — if you don't have a strategy.

### The Four Invalidation Patterns

```
┌─────────────────────────────────────────────────────────┐
│  1. TIME-BASED (TTL)                                     │
│     "This data expires after N seconds"                  │
│                                                          │
│     Pros: Simple, predictable, no coordination needed    │
│     Cons: Data can be stale up to TTL, wasteful for      │
│           data that changes rarely                       │
│     Use for: API responses, CDN cache, Redis             │
│                                                          │
│  2. EVENT-BASED                                          │
│     "Invalidate when this thing happens"                 │
│                                                          │
│     Pros: Data is fresh immediately after changes        │
│     Cons: Requires coordination (webhooks, events)       │
│     Use for: User mutations, admin changes, webhooks     │
│                                                          │
│  3. VERSION-BASED                                        │
│     "Each version has a unique identifier"               │
│                                                          │
│     Pros: Never serves wrong version, great for assets   │
│     Cons: Requires version tracking infrastructure       │
│     Use for: Static assets, API schemas, configs         │
│                                                          │
│  4. HYBRID                                               │
│     "TTL for the common case, event for the critical"    │
│                                                          │
│     Pros: Best of both worlds                            │
│     Cons: More complex to implement                      │
│     Use for: Most real-world applications                │
└─────────────────────────────────────────────────────────┘
```

### The Hybrid Strategy in Practice

```typescript
// The hybrid approach: TTL as a safety net, events for freshness
// This is what most production apps should use

// 1. Set reasonable TTLs as a safety net
const CACHE_TTLS = {
  userProfile: 5 * 60,       // 5 minutes
  productList: 10 * 60,      // 10 minutes
  productDetail: 10 * 60,    // 10 minutes
  featureFlags: 30 * 60,     // 30 minutes
  staticContent: 24 * 60 * 60, // 24 hours
} as const;

// 2. Event-based invalidation for mutations
export function useUpdateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: api.updateProduct,
    onSuccess: (updatedProduct) => {
      // Immediately update the cache (no waiting for refetch)
      queryClient.setQueryData(
        ['product', updatedProduct.id],
        updatedProduct
      );
      // Invalidate lists that might contain this product
      queryClient.invalidateQueries({ queryKey: ['products'] });

      // Also invalidate server-side caches
      fetch('/api/revalidate', {
        method: 'POST',
        body: JSON.stringify({
          tags: [`product-${updatedProduct.id}`],
        }),
      });
    },
  });
}

// 3. Version-based for static assets
// In your build process, hash file contents into URLs:
// bundle.abc123.js — change the file, change the hash, new URL = new cache entry
// Set Cache-Control: immutable, max-age=31536000
```

### Cache Warming

Don't wait for users to trigger cache fills. Warm critical caches proactively:

```typescript
// scripts/warm-cache.ts
// Run this after deploying or after a data migration

import { redis } from '../lib/redis';

async function warmProductCache() {
  const products = await db.products.findMany({
    where: { active: true },
    orderBy: { viewCount: 'desc' },
    take: 1000, // Warm the top 1000 most viewed products
  });

  const pipeline = redis.pipeline();

  for (const product of products) {
    pipeline.set(
      `product:${product.id}`,
      JSON.stringify(product),
      { ex: 3600 }
    );
  }

  await pipeline.exec();
  console.log(`Warmed cache for ${products.length} products`);
}
```

---

## 11. BUILDING A UNIFIED CACHE ARCHITECTURE

Now let's put it all together. Here's how all the caching layers work in concert for a mobile e-commerce app:

```
┌──────────────────────────────────────────────────────────────┐
│  USER OPENS APP                                               │
│                                                               │
│  1. MMKV loads persisted TanStack Query cache (1-5ms)        │
│  2. All screens render with cached data immediately           │
│  3. Background refetches fire for stale data                  │
│                                                               │
│  USER BROWSES PRODUCTS                                        │
│                                                               │
│  1. Product list query:                                       │
│     a. TanStack Query checks in-memory cache                 │
│     b. If stale: fetch from API                              │
│     c. API route checks Redis cache (1ms)                    │
│     d. If Redis miss: query database (50ms), store in Redis  │
│     e. Response passes through CDN (cached for next user)    │
│     f. TanStack Query caches response in memory              │
│     g. MMKV persists query cache (background)                │
│                                                               │
│  2. Product images:                                           │
│     a. expo-image checks memory cache                        │
│     b. If miss: check disk cache                             │
│     c. If miss: fetch from CDN (optimized size/format)       │
│     d. Store in memory + disk cache                          │
│                                                               │
│  USER GOES OFFLINE                                            │
│                                                               │
│  1. NetInfo detects connectivity change                       │
│  2. TanStack Query pauses refetches                          │
│  3. All cached data still available for reading               │
│  4. Mutations queued to MMKV                                 │
│  5. Images served from expo-image disk cache                 │
│                                                               │
│  USER COMES BACK ONLINE                                       │
│                                                               │
│  1. NetInfo detects connectivity restored                    │
│  2. Queued mutations replayed in order                       │
│  3. TanStack Query resumes background refetches              │
│  4. Fresh data silently replaces stale cache                 │
└──────────────────────────────────────────────────────────────┘
```

### The Configuration Blueprint

```typescript
// cache-config.ts — your caching decisions in one place

export const CACHE_CONFIG = {
  // TanStack Query defaults
  query: {
    defaultStaleTime: 30 * 1000,      // 30 seconds
    defaultGcTime: 10 * 60 * 1000,    // 10 minutes
    persistMaxAge: 24 * 60 * 60 * 1000, // 24 hours
  },

  // Per-data-type stale times
  staleTime: {
    userProfile: 5 * 60 * 1000,       // 5 minutes
    productList: 2 * 60 * 1000,       // 2 minutes
    productDetail: 5 * 60 * 1000,     // 5 minutes
    cart: 0,                           // Always refetch
    notifications: 0,                  // Always refetch
    featureFlags: Infinity,            // Never stale in session
    categories: 30 * 60 * 1000,       // 30 minutes
  },

  // Redis TTLs (seconds)
  redis: {
    productList: 300,                  // 5 minutes
    productDetail: 600,                // 10 minutes
    userProfile: 300,                  // 5 minutes
    searchResults: 60,                 // 1 minute
    featureFlags: 1800,                // 30 minutes
  },

  // CDN Cache-Control headers
  cdn: {
    staticAssets: 'public, max-age=31536000, immutable',
    apiPublic: 'public, s-maxage=60, stale-while-revalidate=3600',
    apiPrivate: 'private, max-age=60',
    noCache: 'no-store',
  },

  // Image caching
  images: {
    cachePolicy: 'memory-disk' as const,
    prefetchCount: 10,                 // Prefetch first 10 images in lists
  },
} as const;
```

### Monitoring Your Cache

Caching without monitoring is flying blind. Track these metrics:

```typescript
// lib/cache-metrics.ts

// Track TanStack Query cache performance
export function setupCacheMonitoring(queryClient: QueryClient) {
  const queryCache = queryClient.getQueryCache();

  queryCache.subscribe((event) => {
    if (event.type === 'updated') {
      const query = event.query;
      const fetchStatus = query.state.fetchStatus;

      // Track cache hit rates
      if (fetchStatus === 'idle' && query.state.status === 'success') {
        analytics.track('cache_hit', {
          queryKey: JSON.stringify(query.queryKey),
        });
      }

      if (fetchStatus === 'fetching') {
        analytics.track('cache_miss', {
          queryKey: JSON.stringify(query.queryKey),
          hadStaleData: query.state.data !== undefined,
        });
      }
    }
  });
}

// Track Redis cache performance (server-side)
export async function cachedWithMetrics<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number
): Promise<T> {
  const start = performance.now();
  const cached = await redis.get<T>(key);

  if (cached !== null) {
    metrics.increment('redis.cache.hit');
    metrics.timing('redis.cache.latency', performance.now() - start);
    return cached;
  }

  metrics.increment('redis.cache.miss');
  const data = await fetcher();
  await redis.set(key, data, { ex: ttl });
  metrics.timing('redis.cache.fetch_latency', performance.now() - start);

  return data;
}
```

---

## 12. COMMON CACHING MISTAKES

Let me save you some pain. These are the mistakes I see over and over:

### Mistake 1: Caching Everything with the Same TTL

```typescript
// BAD: One TTL to rule them all
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5 * 60 * 1000 }, // 5 minutes for everything
  },
});

// GOOD: Different data has different freshness requirements
// User's cart: always fresh (staleTime: 0)
// Product catalog: 5 minutes
// Static FAQ content: 1 hour
// Feature flags: session-long
```

### Mistake 2: Not Handling Cache Versioning Across App Updates

```typescript
// BAD: User updates the app, old cached data causes crashes
// because the data shape changed

// GOOD: Version your cache
<PersistQueryClientProvider
  persistOptions={{
    persister: queryPersister,
    buster: APP_VERSION, // Cache is invalidated on app update
  }}
/>
```

### Mistake 3: Caching Error States

```typescript
// BAD: TanStack Query caches the error, user sees error even
// after the server recovers

// GOOD: Only persist successful queries
shouldDehydrateQuery: (query) => {
  return query.state.status === 'success';
}
```

### Mistake 4: Ignoring Memory Pressure on Mobile

```typescript
// BAD: Cache everything in memory indefinitely
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: Infinity },
  },
});

// GOOD: Be mindful of mobile memory constraints
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 5 * 60 * 1000, // 5 minutes for most queries
    },
  },
});

// For large datasets (lists with images, etc.),
// be even more aggressive
const { data } = useQuery({
  queryKey: ['large-dataset'],
  queryFn: fetchLargeDataset,
  gcTime: 60 * 1000, // Only 1 minute in memory
});
```

### Mistake 5: Not Warming Caches After Deployment

```typescript
// BAD: Deploy new version → all CDN and Redis caches are cold
// → first users get slow responses

// GOOD: Warm caches as part of your deployment pipeline
// In your CI/CD:
// 1. Deploy new version
// 2. Run cache warming script
// 3. Route traffic to new version
```

---

## 13. THE DECISION FRAMEWORK

When designing your caching strategy, walk through this checklist for each type of data:

```
┌─────────────────────────────────────────────────────────────┐
│  FOR EACH DATA TYPE, ASK:                                    │
│                                                              │
│  1. How often does this data change?                         │
│     Rarely (hours/days) → Long TTL, cache aggressively      │
│     Sometimes (minutes) → Moderate TTL, SWR pattern          │
│     Frequently (seconds) → Short TTL or no cache             │
│                                                              │
│  2. How bad is it if users see stale data?                   │
│     Not bad (product descriptions) → Cache aggressively      │
│     Somewhat bad (prices) → Short TTL + event invalidation   │
│     Very bad (account balance) → No cache or very short TTL  │
│                                                              │
│  3. How expensive is it to fetch?                            │
│     Cheap (simple DB query) → Cache is optional              │
│     Moderate (joins, aggregations) → Cache in Redis          │
│     Expensive (3rd party API, ML inference) → Cache long     │
│                                                              │
│  4. Is this data personalized?                               │
│     No (same for all users) → CDN + Redis + long TTL         │
│     Yes (user-specific) → Client cache + short Redis TTL     │
│                                                              │
│  5. Does this data need to work offline?                     │
│     No → In-memory TanStack Query cache is sufficient        │
│     Yes → Persist to MMKV + offline mutation queue           │
└─────────────────────────────────────────────────────────────┘
```

---

## CHAPTER SUMMARY

Caching is not a single technique — it's a multi-layered architecture where each layer serves a different purpose:

- **MMKV** gives you synchronous, 30x-faster-than-AsyncStorage persistence on the device
- **TanStack Query** manages your runtime cache with stale-while-revalidate semantics and intelligent background refetching
- **HTTP caching** (Cache-Control, ETag, stale-while-revalidate) reduces bandwidth and latency at the network level
- **CDN caching** serves responses from edge nodes close to your users
- **Vercel Edge Config** provides sub-millisecond configuration reads at the edge
- **Redis/Upstash** caches expensive server-side operations
- **expo-image** handles image caching with memory and disk tiers

The key insight is that these layers are not alternatives — they're complementary. A well-architected app uses all of them, with each layer acting as a fallback for the one above it. The result is an app that feels instant, works offline, and costs less to operate because most requests never reach your origin server.

In the next chapter, we'll extend these cache-first patterns into full offline-first architectures with conflict resolution, CRDTs, and real-time sync — the frontier of mobile app architecture.

---

*Next: [Chapter 12: Offline-First & Real-Time Patterns]*
*Previous: [Chapter 10: Data Fetching & Server Communication]*