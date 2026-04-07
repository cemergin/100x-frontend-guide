<!--
  CHAPTER: 10
  TITLE: State Management at Scale
  PART: III — State, Data & Communication
  PREREQS: Chapter 1, 3
  KEY_TOPICS: Zustand, Jotai, Legend State, TanStack Query, Redux, React Hook Form, Zod, MMKV, finite states, type states, XState Store, anti-patterns, three-category model, useSyncExternalStore, URL state, nuqs, data normalization
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 9: State Management at Scale

> **Part III — State, Data & Communication** | Prerequisites: Chapters 1, 3 | Difficulty: Intermediate to Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Split state into three categories: server state (TanStack Query), client state (Zustand), and form state (React Hook Form + Zod); mixing them in one store is the root cause of most "state management is hard" complaints
- TanStack Query owns everything that comes from your API -- loading states, caching, background refetch, optimistic updates; stop putting API data in Redux or Context
- Zustand is the default for client state: tiny API, no boilerplate, works outside React, and selectors prevent unnecessary re-renders
- React Hook Form + Zod gives you performant forms with runtime validation and compile-time type safety; never use controlled inputs for complex forms
- Finite state machines (XState Store) eliminate impossible states; use them for complex flows like checkout, onboarding, or multi-step wizards

</details>

Let's start with the controversial truth that will save you months of pain: **you probably have a state management problem, but it's not the one you think it is.**

Most teams that say "we need better state management" actually mean "our state is a tangled mess of `useState` hooks, prop drilling, contexts that re-render the entire app, and Redux stores that mix server data with UI state." They reach for a new library — and the new library works great for about three months, until it's a tangled mess again.

The problem was never the library. The problem was that they put everything in the same bucket: API responses sitting next to theme preferences sitting next to form input values, all managed by the same tool with the same update patterns and the same lifecycle. That's like storing your clothes, food, and cleaning supplies all in the same cabinet because "they're all things I own."

This chapter introduces the **three-category model** — the single most impactful idea in modern frontend state architecture — and then shows you the best tool for each category, when to pick alternatives, and the anti-patterns that will ruin any approach if you let them.

### In This Chapter
- The Three Categories of State (server, client, form)
- React Anti-Patterns That Created This Mess
- Server State: TanStack Query (query factories, dependent queries, parallel queries, prefetching, infinite queries)
- Client State: Zustand (slices, middleware, devtools, computed state)
- Atomic State: Jotai (atom composition, async atoms, atom families, persistence)
- Offline-First State: Legend State (sync with conflict resolution)
- Form State: React Hook Form + Zod (multi-step, dynamic fields, file uploads)
- Finite States and Type States (XState Store checkout flow)
- External Store Patterns (useSyncExternalStore)
- URL State Management (nuqs, shareable/bookmarkable state)
- Data Normalization (complete social media feed example)
- Performance Deep Dive (measuring re-renders, Context vs selectors, optimization)
- Redux Toolkit Comparison (side-by-side with Zustand + TanStack Query)
- Undo/Redo with Event Sourcing
- The Decision Framework
- Migration Paths (from Redux, from Context)

### Related Chapters
- [Ch 10: Data Fetching & Server Communication] — deep dive on TanStack Query and API patterns
- [Ch 11: Caching Strategies] — how state management intersects with caching
- [Ch 12: Offline-First & Real-Time] — Legend State sync patterns
- [Ch 13: Performance Optimization] — how state choices affect render performance

---

## 1. THE THREE CATEGORIES OF STATE

This is the foundation. Get this right and everything else follows. Get it wrong and no library will save you.

Every piece of state in your application falls into one of three categories:

```
┌─────────────────────────────────────────────────────┐
│  SERVER STATE (async, cached, shared)                │
│  "Data that lives on your server"                    │
│                                                      │
│  Characteristics:                                    │
│  - Fetched asynchronously from an API                │
│  - Has loading, error, and success states            │
│  - Can be stale (server value changed since fetch)   │
│  - Multiple components might need the same data      │
│  - Should be cached and background-refreshed         │
│                                                      │
│  Tool: TanStack Query (React Query)                  │
│  Examples: user profile, product list, notifications │
├─────────────────────────────────────────────────────┤
│  CLIENT STATE (sync, local, transient)               │
│  "Data that lives only in the browser/app"           │
│                                                      │
│  Characteristics:                                    │
│  - Set synchronously by user actions or app logic    │
│  - No loading state (it's always available)          │
│  - Source of truth is the client, not a server       │
│  - May need to persist between sessions (or not)     │
│                                                      │
│  Tool: Zustand / Jotai / Legend State                │
│  Examples: theme, auth token, selected filters,      │
│            sidebar open, cart (before checkout)       │
├─────────────────────────────────────────────────────┤
│  FORM STATE (ephemeral, validated)                   │
│  "Data the user is currently entering"               │
│                                                      │
│  Characteristics:                                    │
│  - Extremely short-lived (exists during form entry)  │
│  - Needs validation (instant feedback)               │
│  - Should NOT live in global state                   │
│  - Submitted to server, then discarded               │
│                                                      │
│  Tool: React Hook Form + Zod                         │
│  Examples: login form, search input, checkout flow   │
└─────────────────────────────────────────────────────┘
```

### Why This Separation Matters

**Different lifecycles.** Server state is fetched, cached, refreshed, and invalidated on a time-based or event-based schedule. Client state is set once and read many times. Form state is born when a form mounts and dies when it submits or unmounts. Managing these with the same tool creates impossible lifecycle conflicts.

**Different update patterns.** Server state updates come from the network (background refetch reveals a new value). Client state updates come from user actions (toggle the sidebar). Form state updates come from keystrokes. Each pattern has different performance characteristics and different optimal data structures.

**Different performance implications.** When server state updates (a new product is added to the catalog), you want to update the product list without re-rendering the sidebar and the navigation. When the theme changes, you want to re-render the theme-dependent components without re-fetching data. When a form input changes, you want to re-render only the input field, not the entire page.

**Different error handling.** Server state errors are network errors, 500s, 404s — you show a retry button or a fallback UI. Client state doesn't really "error" — you validate transitions. Form state errors are validation errors — you show inline messages next to fields. Cramming all three into one error handling pattern makes every error handling path worse.

> **The Redux Trap:** Redux became the de facto state manager in React because it was there first and it could do everything. And that was the problem — it *did* everything. Server state, client state, form state, all lived in the same store. A Saga that fetched user data lived next to a reducer that toggled a modal. When the store updated, all connected components re-evaluated their selectors. Teams built enormous Redux stores with hundreds of actions and reducers, and the abstraction that was supposed to simplify state management became the largest source of complexity in the application.

### The Three-Category Litmus Test

When you encounter a new piece of state, ask three questions:

1. **Does it come from an API?** → Server state. TanStack Query.
2. **Is the user currently typing/selecting it in a form?** → Form state. React Hook Form.
3. **Everything else** → Client state. Zustand.

That's it. I have seen this test correctly classify state in hundreds of real applications. The edge cases are rare and usually involve URL state (covered in Section 11) or derived state (which shouldn't be state at all — covered in Section 2).

Here's a concrete example. A product page has:

| State | Category | Tool |
|-------|----------|------|
| Product details | Server (from API) | TanStack Query |
| Related products | Server (from API) | TanStack Query |
| User reviews | Server (from API) | TanStack Query |
| Selected variant (size/color) | Client (UI selection) | Zustand or URL state |
| Cart items | Client (persisted locally) | Zustand + MMKV |
| "Add to cart" animation playing | Client (ephemeral UI) | useState (local) |
| Review form text | Form (user input) | React Hook Form |
| Star rating being selected | Form (user input) | React Hook Form |

No category confusion. No tool trying to do what another tool does better.

---

## 2. REACT ANTI-PATTERNS THAT CREATED THIS MESS

Before we get to solutions, let's name the specific anti-patterns that make state management painful. You'll recognize most of these from codebases you've worked in. I've seen every single one of these in production at companies with hundreds of engineers. They're not mistakes made by junior developers — they're mistakes made by smart people using the wrong mental model.

### 2.1 The Derived State Anti-Pattern

```tsx
// ❌ BAD: Deriving state with useEffect
const [items, setItems] = useState<Item[]>([]);
const [filteredItems, setFilteredItems] = useState<Item[]>([]);
const [selectedCategory, setSelectedCategory] = useState('all');

useEffect(() => {
  setFilteredItems(
    selectedCategory === 'all'
      ? items
      : items.filter(item => item.category === selectedCategory)
  );
}, [items, selectedCategory]);

// ✅ GOOD: Derive during render
const [items, setItems] = useState<Item[]>([]);
const [selectedCategory, setSelectedCategory] = useState('all');

const filteredItems = selectedCategory === 'all'
  ? items
  : items.filter(item => item.category === selectedCategory);
```

The `useEffect` version creates an extra render cycle. State updates → render → effect fires → `setFilteredItems` → *another* render. The direct derivation computes the value during render — one render, done.

**Rule:** If state B can be computed from state A, don't store B. Compute it.

I've seen this pattern cost a team two weeks of debugging. They had a dashboard with filters, and the filtered results were stored in separate state. Users reported "flickering" — the UI would briefly show unfiltered results before the effect ran and applied the filter. The fix was deleting the `filteredItems` state entirely and computing it inline. Two weeks of debugging, thirty seconds of fixing.

**But what about expensive computations?** Use `useMemo`:

```tsx
const filteredItems = useMemo(
  () => selectedCategory === 'all'
    ? items
    : items.filter(item => item.category === selectedCategory),
  [items, selectedCategory]
);
```

`useMemo` caches the computation result and only recomputes when dependencies change. It still doesn't create an extra render cycle — it just skips the computation if inputs haven't changed.

### 2.2 The Redundant State Anti-Pattern

```tsx
// ❌ BAD: Storing redundant state
const [hotels, setHotels] = useState<Hotel[]>([]);
const [selectedHotel, setSelectedHotel] = useState<Hotel | null>(null);

// When hotels update, selectedHotel might be stale:
// it's a copy, not a reference

// ✅ GOOD: Store the ID, derive the object
const [hotels, setHotels] = useState<Hotel[]>([]);
const [selectedHotelId, setSelectedHotelId] = useState<string | null>(null);

const selectedHotel = hotels.find(h => h.id === selectedHotelId) ?? null;
```

Store the minimum — an ID, an index, a key — and derive the full object. This eliminates an entire class of bugs where two copies of the same data drift apart.

I once debugged a booking app where users reported that the price shown on the hotel card didn't match the price on the detail screen. The team was storing the full hotel object in both the list and the selection state. When a background refetch updated prices in the list, the selected hotel kept the old price. The fix: store `selectedHotelId`, derive the object. Instant consistency.

### 2.3 The Cascading Effects Anti-Pattern

```tsx
// ❌ BAD: useEffect dominoes
const [city, setCity] = useState('NYC');
const [flights, setFlights] = useState([]);
const [hotels, setHotels] = useState([]);
const [activities, setActivities] = useState([]);

useEffect(() => { fetchFlights(city).then(setFlights); }, [city]);
useEffect(() => { fetchHotels(city).then(setHotels); }, [flights]);
useEffect(() => { fetchActivities(city).then(setActivities); }, [hotels]);
// Three renders, waterfall fetches, impossible to trace data flow
```

Each `useEffect` triggers on the *result* of the previous one, creating a cascade of renders. The data flow is invisible — you can't look at any single effect and understand the full sequence. This is "effect spaghetti."

**Fix:** Use a reducer to centralize the logic, or use TanStack Query with `enabled` flags to express the dependency chain declaratively:

```tsx
// ✅ GOOD: TanStack Query with declarative dependencies
const flights = useQuery({
  queryKey: ['flights', city],
  queryFn: () => fetchFlights(city),
});

const hotels = useQuery({
  queryKey: ['hotels', city],
  queryFn: () => fetchHotels(city),
  enabled: !!flights.data, // Only fetch when flights are loaded
});

const activities = useQuery({
  queryKey: ['activities', city],
  queryFn: () => fetchActivities(city),
  enabled: !!hotels.data, // Only fetch when hotels are loaded
});
```

Now the dependency chain is visible in the code. Each query declares what it depends on. No cascading effects, no invisible render chains.

### 2.4 The useState Explosion

```tsx
// ❌ BAD: Seven related states managed independently
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [error, setError] = useState<Error | null>(null);
const [data, setData] = useState<User | null>(null);
const [isSubmitting, setIsSubmitting] = useState(false);
const [isRetrying, setIsRetrying] = useState(false);

// These seven booleans have 128 possible combinations.
// Only ~4 are valid. Nothing enforces which ones.
// Bug: isLoading && isSuccess can be true simultaneously.
```

This is what finite states solve. More on this in Section 8.

### 2.5 The Context Sledgehammer

```tsx
// ❌ BAD: One giant context for everything
const AppContext = createContext<{
  user: User | null;
  theme: 'light' | 'dark';
  notifications: Notification[];
  cart: CartItem[];
  sidebarOpen: boolean;
  locale: string;
  featureFlags: Record<string, boolean>;
}>({ /* defaults */ });

function AppProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(appReducer, initialState);
  return (
    <AppContext.Provider value={state}>
      {children}
    </AppContext.Provider>
  );
}

// Problem: when `notifications` changes (every 30 seconds),
// EVERY component reading ANY value from AppContext re-renders.
// The theme toggle component re-renders when a notification arrives.
// The cart badge re-renders when the sidebar opens.
```

I call this the "Context Sledgehammer" because it smashes every component equally, regardless of what they actually need. I've seen this pattern cause 200+ unnecessary re-renders per notification update in a production app. The team thought React was slow. React was fine — they were just re-rendering the entire tree every 30 seconds.

The fix isn't "split into multiple contexts" (though that helps). The fix is to use a tool that has selectors. More in Section 4.

### 2.6 The Sync Two Sources of Truth Anti-Pattern

```tsx
// ❌ BAD: Copying server data into client state
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
const [localUser, setLocalUser] = useState<User | null>(null);

useEffect(() => {
  if (user) setLocalUser(user);
}, [user]);

// Now you have two copies of the user. Which one is authoritative?
// When the query refetches in the background, localUser is stale.
// When you mutate localUser, the query cache is stale.
// You've recreated the exact problem the three-category model solves.
```

**Rule:** Server data lives in TanStack Query's cache. Period. If you need to transform it, use the `select` option. If you need to modify it optimistically, use `onMutate`. Never copy it into separate state.

```tsx
// ✅ GOOD: Transform with select, don't copy
const { data: displayName } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  select: (user) => `${user.firstName} ${user.lastName}`,
});
```

---

## 3. SERVER STATE: TANSTACK QUERY

TanStack Query (formerly React Query) is the standard for server state in 2026. It's not just a fetching library — it's a **server state synchronization engine** with caching, background refetching, optimistic updates, pagination, infinite queries, and garbage collection built in.

### 3.1 Why Not Just fetch + useState?

```tsx
// The "simple" approach (❌ this is what you outgrow)
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => { if (!cancelled) { setUser(data); setLoading(false); }})
      .catch(err => { if (!cancelled) { setError(err); setLoading(false); }});
    return () => { cancelled = true; };
  }, [userId]);

  return { user, loading, error };
}
```

This seems fine for one hook. Now multiply by 30 screens, each with 2-3 data dependencies. You'll need to handle:
- Cache invalidation (when should you refetch?)
- Deduplication (two components requesting the same user)
- Background refetch (user came back from background, is data stale?)
- Optimistic updates (show the change immediately, revert on error)
- Pagination and infinite scroll
- Retry logic with backoff
- Request cancellation on unmount
- Prefetching for navigation

That's what TanStack Query does out of the box.

### 3.2 Core Concepts

```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// QUERIES — read data, with automatic caching and background refresh
function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],          // Cache key — unique per user
    queryFn: () => api.getUser(userId),  // The actual fetch function
    staleTime: 5 * 60 * 1000,           // Data is "fresh" for 5 min
    gcTime: 30 * 60 * 1000,             // Keep in cache for 30 min after last use
  });
}

// MUTATIONS — write data, with optimistic updates
function useUpdateProfile() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: ProfileUpdate) => api.updateProfile(data),
    // Optimistic update: show the change immediately
    onMutate: async (newData) => {
      await queryClient.cancelQueries({ queryKey: ['user'] });
      const previous = queryClient.getQueryData(['user']);
      queryClient.setQueryData(['user'], (old: User) => ({ ...old, ...newData }));
      return { previous };
    },
    // Rollback on error
    onError: (_err, _newData, context) => {
      queryClient.setQueryData(['user'], context?.previous);
    },
    // Always refetch after mutation to get server truth
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['user'] });
    },
  });
}
```

### 3.3 Query Keys: Your Cache Addressing Scheme

Query keys are how TanStack Query identifies cached data. Get them right and cache invalidation is trivial. Get them wrong and you'll have stale data bugs everywhere.

```tsx
// Hierarchical keys enable granular invalidation
['users']                     // All users
['users', 'list']             // The users list
['users', 'list', { page: 1, sort: 'name' }]  // Specific list page
['users', 'detail', userId]   // Specific user

// Invalidate all user-related queries:
queryClient.invalidateQueries({ queryKey: ['users'] });

// Invalidate only the list (not detail views):
queryClient.invalidateQueries({ queryKey: ['users', 'list'] });

// Invalidate one specific user:
queryClient.invalidateQueries({ queryKey: ['users', 'detail', userId] });
```

### 3.4 Query Factories: The Pattern That Scales

As your app grows, you'll have dozens of query keys scattered across hooks. The query factory pattern centralizes them into a single, type-safe object:

```tsx
// src/queries/users.ts
export const userQueries = {
  all: () => ['users'] as const,
  lists: () => [...userQueries.all(), 'list'] as const,
  list: (filters: UserFilters) =>
    [...userQueries.lists(), filters] as const,
  details: () => [...userQueries.all(), 'detail'] as const,
  detail: (userId: string) =>
    [...userQueries.details(), userId] as const,
  profile: (userId: string) =>
    [...userQueries.detail(userId), 'profile'] as const,
};

// src/queries/products.ts
export const productQueries = {
  all: () => ['products'] as const,
  lists: () => [...productQueries.all(), 'list'] as const,
  list: (filters: ProductFilters) =>
    [...productQueries.lists(), filters] as const,
  details: () => [...productQueries.all(), 'detail'] as const,
  detail: (productId: string) =>
    [...productQueries.details(), productId] as const,
  reviews: (productId: string) =>
    [...productQueries.detail(productId), 'reviews'] as const,
};

// Usage in hooks — clean, consistent, type-safe
function useUserList(filters: UserFilters) {
  return useQuery({
    queryKey: userQueries.list(filters),
    queryFn: () => api.getUsers(filters),
  });
}

function useUserDetail(userId: string) {
  return useQuery({
    queryKey: userQueries.detail(userId),
    queryFn: () => api.getUser(userId),
  });
}

// Invalidation is clean:
queryClient.invalidateQueries({ queryKey: userQueries.all() });       // All user data
queryClient.invalidateQueries({ queryKey: userQueries.lists() });     // All user lists
queryClient.invalidateQueries({ queryKey: userQueries.detail('u1') }); // One user + sub-queries
```

Why this matters: without factories, your query keys are magic strings scattered across your codebase. Change a key format and you break caching silently. With factories, you change it in one place and TypeScript catches any mismatches.

### 3.5 Dependent Queries

When one query depends on the result of another, use the `enabled` option:

```tsx
// Fetch the user, then fetch their organization
function useUserOrganization(userId: string) {
  const userQuery = useQuery({
    queryKey: userQueries.detail(userId),
    queryFn: () => api.getUser(userId),
  });

  const orgQuery = useQuery({
    queryKey: ['organization', userQuery.data?.organizationId],
    queryFn: () => api.getOrganization(userQuery.data!.organizationId),
    enabled: !!userQuery.data?.organizationId,  // Only runs when user data is available
  });

  return {
    user: userQuery.data,
    organization: orgQuery.data,
    isLoading: userQuery.isLoading || orgQuery.isLoading,
    error: userQuery.error || orgQuery.error,
  };
}
```

The `enabled: false` state means the query won't fire at all — no request, no loading state, no wasted bandwidth. When `enabled` flips to `true`, TanStack Query fires the request automatically.

### 3.6 Parallel Queries

When you need multiple independent queries, just call them. TanStack Query fires them in parallel automatically:

```tsx
function useProductPage(productId: string) {
  // These three queries fire simultaneously — no waterfall
  const product = useQuery({
    queryKey: productQueries.detail(productId),
    queryFn: () => api.getProduct(productId),
  });

  const reviews = useQuery({
    queryKey: productQueries.reviews(productId),
    queryFn: () => api.getProductReviews(productId),
  });

  const recommendations = useQuery({
    queryKey: ['recommendations', productId],
    queryFn: () => api.getRecommendations(productId),
  });

  return { product, reviews, recommendations };
}
```

For dynamic parallel queries (e.g., fetching details for a list of IDs), use `useQueries`:

```tsx
function useMultipleUsers(userIds: string[]) {
  return useQueries({
    queries: userIds.map((id) => ({
      queryKey: userQueries.detail(id),
      queryFn: () => api.getUser(id),
      staleTime: 5 * 60 * 1000,
    })),
  });
}
```

### 3.7 Prefetching for Navigation

One of TanStack Query's most powerful features: prefetch data before the user navigates, so the next screen loads instantly.

```tsx
// Prefetch on hover/focus (web)
function ProductCard({ product }: { product: ProductSummary }) {
  const queryClient = useQueryClient();

  const prefetchProduct = () => {
    queryClient.prefetchQuery({
      queryKey: productQueries.detail(product.id),
      queryFn: () => api.getProduct(product.id),
      staleTime: 5 * 60 * 1000, // Don't refetch if we already have fresh data
    });
  };

  return (
    <Link
      href={`/products/${product.id}`}
      onMouseEnter={prefetchProduct}  // Prefetch on hover
      onFocus={prefetchProduct}       // Prefetch on keyboard focus
    >
      <ProductCardContent product={product} />
    </Link>
  );
}

// Prefetch on screen focus (React Native)
function ProductListScreen() {
  const queryClient = useQueryClient();
  const { data: products } = useQuery({
    queryKey: productQueries.lists(),
    queryFn: () => api.getProducts(),
  });

  // When user taps a product, prefetch its detail before navigation
  const handleProductPress = (productId: string) => {
    // Fire prefetch immediately — don't await it
    queryClient.prefetchQuery({
      queryKey: productQueries.detail(productId),
      queryFn: () => api.getProduct(productId),
    });
    // Navigate immediately — the detail screen will show cached data
    navigation.navigate('ProductDetail', { productId });
  };

  return (
    <FlatList
      data={products}
      renderItem={({ item }) => (
        <Pressable onPress={() => handleProductPress(item.id)}>
          <ProductCard product={item} />
        </Pressable>
      )}
    />
  );
}
```

The result: the user taps a product, navigation starts, and the detail screen already has data in the cache. No loading spinner. This is the single easiest way to make your app feel faster.

### 3.8 Infinite Queries for Feeds

Social feeds, product catalogs, message histories — anything with "load more" or infinite scroll:

```tsx
function useFeed() {
  return useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) => api.getFeed({ cursor: pageParam, limit: 20 }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    getPreviousPageParam: (firstPage) => firstPage.previousCursor ?? undefined,
  });
}

// Usage in a component
function FeedScreen() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useFeed();

  // Flatten pages into a single list
  const posts = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <FlatList
      data={posts}
      renderItem={({ item }) => <FeedPost post={item} />}
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  );
}
```

**Key detail:** `data.pages` is an array of page responses. Each page contains the items and cursor info. The `flatMap` creates a flat list for rendering. When the user scrolls to the bottom, `fetchNextPage` loads the next page and appends it to `data.pages`.

### 3.9 staleTime vs gcTime

This trips up everyone. They're different clocks:

- **staleTime** (default: 0): How long data is "fresh." While fresh, TanStack Query returns cached data without refetching. After staleTime, data is "stale" — TQ will still return it instantly but also trigger a background refetch.
- **gcTime** (default: 5 min): How long data stays in cache after the last component using it unmounts. After gcTime, the cache entry is garbage collected.

```
Time: 0    1min   5min   10min   30min
      │     │      │      │       │
      └─ Fetch
           └─ staleTime=5min: data is fresh, no refetch
                  └─ Data is stale. Next mount triggers background refetch.
                         └─ gcTime=30min: data still in cache
                                   └─ gcTime expires: cache entry deleted
```

**For mobile apps:** Set `staleTime` generously (5-15 min) to minimize unnecessary network requests on cellular. Set `gcTime` even longer (30-60 min) because navigating back to a screen should show cached data instantly.

**A common mistake I see in production:** leaving `staleTime` at 0 (the default) and then wondering why the app makes hundreds of API requests when the user navigates between tabs. Every tab mount triggers a refetch because data is instantly stale. Set a reasonable `staleTime` and those unnecessary refetches disappear.

---

## 4. CLIENT STATE: ZUSTAND

Zustand is the default client state manager in 2026. At ~1KB, with zero boilerplate, and React Native support out of the box, it's what you reach for when you need global UI state.

### 4.1 Why Zustand Won

```tsx
// That's it. That's the store.
import { create } from 'zustand';

interface AppStore {
  theme: 'light' | 'dark';
  sidebarOpen: boolean;
  toggleTheme: () => void;
  toggleSidebar: () => void;
}

export const useAppStore = create<AppStore>((set) => ({
  theme: 'light',
  sidebarOpen: false,
  toggleTheme: () => set((s) => ({ theme: s.theme === 'light' ? 'dark' : 'light' })),
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));

// Usage — only re-renders when the selected value changes
function ThemeButton() {
  const theme = useAppStore((s) => s.theme);
  const toggleTheme = useAppStore((s) => s.toggleTheme);
  return <Pressable onPress={toggleTheme}><Text>{theme}</Text></Pressable>;
}
```

Compare this to Redux:
- No Provider wrapper needed
- No actions/action creators/action types
- No reducers
- No dispatch
- No connect/mapStateToProps
- Selectors are built-in (the function you pass to `useAppStore`)
- TypeScript support is first-class without extra types

### 4.2 Zustand with MMKV Persistence

For React Native, persisting state between sessions is critical — auth tokens, onboarding completion, user preferences. Zustand's persist middleware + MMKV (30x faster than AsyncStorage) is the standard pattern:

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

const mmkvStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};

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
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);
```

### 4.3 The Slices Pattern for Large Stores

When your store grows past 10-15 properties, a single flat object becomes hard to maintain. The slices pattern lets you split a store into logical sections while keeping it as a single Zustand store:

```tsx
// src/store/slices/authSlice.ts
import { StateCreator } from 'zustand';

export interface AuthSlice {
  token: string | null;
  user: User | null;
  setAuth: (token: string, user: User) => void;
  logout: () => void;
}

export const createAuthSlice: StateCreator<
  AppStore,       // The full store type
  [],
  [],
  AuthSlice       // This slice's type
> = (set) => ({
  token: null,
  user: null,
  setAuth: (token, user) => set({ token, user }),
  logout: () => set({ token: null, user: null }),
});

// src/store/slices/uiSlice.ts
export interface UISlice {
  theme: 'light' | 'dark';
  sidebarOpen: boolean;
  activeModal: string | null;
  toggleTheme: () => void;
  toggleSidebar: () => void;
  openModal: (id: string) => void;
  closeModal: () => void;
}

export const createUISlice: StateCreator<
  AppStore,
  [],
  [],
  UISlice
> = (set) => ({
  theme: 'light',
  sidebarOpen: false,
  activeModal: null,
  toggleTheme: () => set((s) => ({ theme: s.theme === 'light' ? 'dark' : 'light' })),
  toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
  openModal: (id) => set({ activeModal: id }),
  closeModal: () => set({ activeModal: null }),
});

// src/store/slices/cartSlice.ts
export interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (itemId: string) => void;
  updateQuantity: (itemId: string, quantity: number) => void;
  clearCart: () => void;
  totalItems: () => number;
  totalPrice: () => number;
}

export const createCartSlice: StateCreator<
  AppStore,
  [],
  [],
  CartSlice
> = (set, get) => ({
  items: [],
  addItem: (item) => set((s) => {
    const existing = s.items.find((i) => i.id === item.id);
    if (existing) {
      return {
        items: s.items.map((i) =>
          i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
        ),
      };
    }
    return { items: [...s.items, { ...item, quantity: 1 }] };
  }),
  removeItem: (itemId) => set((s) => ({
    items: s.items.filter((i) => i.id !== itemId),
  })),
  updateQuantity: (itemId, quantity) => set((s) => ({
    items: s.items.map((i) =>
      i.id === itemId ? { ...i, quantity } : i
    ),
  })),
  clearCart: () => set({ items: [] }),
  // Derived values via get() — not stored, computed on access
  totalItems: () => get().items.reduce((sum, i) => sum + i.quantity, 0),
  totalPrice: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),
});

// src/store/index.ts — combine all slices
import { create } from 'zustand';
import { createAuthSlice, AuthSlice } from './slices/authSlice';
import { createUISlice, UISlice } from './slices/uiSlice';
import { createCartSlice, CartSlice } from './slices/cartSlice';

export type AppStore = AuthSlice & UISlice & CartSlice;

export const useAppStore = create<AppStore>()((...args) => ({
  ...createAuthSlice(...args),
  ...createUISlice(...args),
  ...createCartSlice(...args),
}));
```

Slices keep files small, concerns separated, and TypeScript happy. Each slice can access the full store through `get()`, so cross-slice logic (e.g., "clear cart on logout") is straightforward:

```tsx
// In authSlice, access cart via get()
logout: () => {
  set({ token: null, user: null });
  // Cross-slice action: also clear the cart
  set({ items: [] });
},
```

### 4.4 Computed / Derived State in Zustand

Zustand doesn't have built-in computed properties like MobX or Vue. You have three options:

**Option 1: Compute in the selector (recommended for most cases)**

```tsx
// The selector computes the derived value
const totalItems = useAppStore((s) =>
  s.items.reduce((sum, i) => sum + i.quantity, 0)
);
```

Simple and works great. But if you use this in multiple components, you're duplicating the computation logic.

**Option 2: Methods on the store using `get()`**

```tsx
// In the store definition
totalItems: () => get().items.reduce((sum, i) => sum + i.quantity, 0),

// Usage
const store = useAppStore();
const total = store.totalItems(); // Note: this is a function call, not a property
```

This centralizes the logic but has a gotcha: calling `store.totalItems()` returns a value, not a subscription. The component won't re-render when items change unless you also select something that does change.

**Option 3: Derived selectors with `useShallow` or custom equality**

```tsx
import { useShallow } from 'zustand/react/shallow';

// Select multiple values with shallow comparison
const { items, theme } = useAppStore(
  useShallow((s) => ({ items: s.items, theme: s.theme }))
);
```

### 4.5 Middleware: Immer for Immutable Updates

Deeply nested state updates in plain Zustand require spread operators everywhere:

```tsx
// Without immer — spread city everywhere
updateAddress: (userId, newCity) => set((s) => ({
  users: {
    ...s.users,
    [userId]: {
      ...s.users[userId],
      address: {
        ...s.users[userId].address,
        city: newCity,
      },
    },
  },
})),
```

With the immer middleware, you mutate directly and immer produces the immutable update:

```tsx
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

interface SettingsStore {
  preferences: {
    notifications: {
      email: boolean;
      push: boolean;
      sms: boolean;
    };
    privacy: {
      profileVisible: boolean;
      showActivity: boolean;
    };
  };
  updateNotificationPref: (key: keyof SettingsStore['preferences']['notifications'], value: boolean) => void;
}

export const useSettingsStore = create<SettingsStore>()(
  immer((set) => ({
    preferences: {
      notifications: { email: true, push: true, sms: false },
      privacy: { profileVisible: true, showActivity: true },
    },
    updateNotificationPref: (key, value) =>
      set((state) => {
        // Direct mutation — immer handles immutability
        state.preferences.notifications[key] = value;
      }),
  }))
);
```

**When to use immer:** When your state has more than two levels of nesting. For flat state, the spread operator is fine and you avoid the immer dependency.

### 4.6 Middleware: Devtools

Zustand integrates with Redux DevTools, giving you time-travel debugging for free:

```tsx
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

export const useAppStore = create<AppStore>()(
  devtools(
    (set) => ({
      // ... your store
    }),
    {
      name: 'AppStore',        // Name shown in DevTools
      enabled: __DEV__,         // Only in development
    }
  )
);

// Name your actions for better DevTools output
set({ theme: 'dark' }, false, 'toggleTheme');
//                       ^       ^ action name shown in DevTools
//                       replace flag (false = merge)
```

**Combining middleware:** Zustand middleware composes. You can stack persist + immer + devtools:

```tsx
export const useAppStore = create<AppStore>()(
  devtools(
    persist(
      immer(
        (set) => ({
          // your store
        })
      ),
      { name: 'app-storage', storage: createJSONStorage(() => mmkvStorage) }
    ),
    { name: 'AppStore', enabled: __DEV__ }
  )
);
```

### 4.7 Zustand vs Context: Performance Comparison

Let's make the performance difference concrete. Consider a store with a `notifications` count that updates every 30 seconds:

```tsx
// CONTEXT VERSION
const AppContext = createContext<{ notifications: number; theme: string }>({
  notifications: 0,
  theme: 'light',
});

function NotificationBadge() {
  const { notifications } = useContext(AppContext); // Re-renders on ANY context change
  return <Badge count={notifications} />;
}

function ThemeLabel() {
  const { theme } = useContext(AppContext); // Also re-renders when notifications change!
  return <Text>{theme}</Text>;
}

// When notifications updates: BOTH components re-render
// Render count after 10 notification updates:
//   NotificationBadge: 10 renders ✓
//   ThemeLabel: 10 renders ✗ (should be 0)
```

```tsx
// ZUSTAND VERSION
const useAppStore = create<{ notifications: number; theme: string }>((set) => ({
  notifications: 0,
  theme: 'light',
}));

function NotificationBadge() {
  const notifications = useAppStore((s) => s.notifications); // Only re-renders when notifications change
  return <Badge count={notifications} />;
}

function ThemeLabel() {
  const theme = useAppStore((s) => s.theme); // Only re-renders when theme changes
  return <Text>{theme}</Text>;
}

// When notifications updates: only NotificationBadge re-renders
// Render count after 10 notification updates:
//   NotificationBadge: 10 renders ✓
//   ThemeLabel: 0 renders ✓
```

That's the difference. Context re-renders every consumer. Zustand selectors re-render only consumers whose selected value actually changed. In a real app with 50+ components reading from shared state, this is the difference between smooth 60fps scrolling and janky, stuttering UI.

### 4.8 Using Zustand Outside React

One of Zustand's underappreciated features: you can read and write state outside of React components. This is invaluable for API interceptors, analytics, navigation guards, and background tasks.

```tsx
// In an API interceptor — no hooks, no components
import { useAuthStore } from './store';

api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
    }
    return Promise.reject(error);
  }
);

// Subscribe to changes outside React
const unsubscribe = useAuthStore.subscribe(
  (state) => state.token,
  (token) => {
    if (!token) {
      navigation.reset({ routes: [{ name: 'Login' }] });
    }
  }
);
```

---

## 5. ATOMIC STATE: JOTAI

Jotai (~2.5KB) uses an **atomic** model — small, independent pieces of state that can depend on each other. Where Zustand is a single centralized store, Jotai is a collection of atoms that compose like React components.

### 5.1 When to Pick Jotai Over Zustand

**Pick Jotai when:** You have complex filter UIs where multiple pieces of state depend on each other, or when you want React-style bottom-up composition rather than a centralized store. Jotai shines in UIs with lots of independent, interconnected state atoms.

**Pick Zustand when:** You want a simple, centralized store for global app state. Most apps. This is the default.

The mental model difference:
- **Zustand**: "I have a store. Components select slices of it."
- **Jotai**: "I have atoms. Components use the atoms they need. Atoms can depend on other atoms."

### 5.2 Atom Composition

Jotai atoms compose naturally — derived atoms depend on other atoms, creating a dependency graph:

```tsx
import { atom, useAtom, useAtomValue } from 'jotai';

// Base atoms — the source of truth
const categoryAtom = atom<string>('all');
const priceRangeAtom = atom<[number, number]>([0, 500]);
const sortByAtom = atom<'price' | 'rating' | 'newest'>('newest');
const searchQueryAtom = atom<string>('');

// Derived atom — computes filters object from base atoms
const activeFiltersAtom = atom((get) => ({
  category: get(categoryAtom),
  priceRange: get(priceRangeAtom),
  sortBy: get(sortByAtom),
  search: get(searchQueryAtom),
}));

// Derived atom — counts active filters for a badge
const activeFilterCountAtom = atom((get) => {
  let count = 0;
  if (get(categoryAtom) !== 'all') count++;
  if (get(priceRangeAtom)[0] > 0 || get(priceRangeAtom)[1] < 500) count++;
  if (get(searchQueryAtom) !== '') count++;
  return count;
});

// Write-only atom — resets all filters
const resetFiltersAtom = atom(null, (_get, set) => {
  set(categoryAtom, 'all');
  set(priceRangeAtom, [0, 500]);
  set(sortByAtom, 'newest');
  set(searchQueryAtom, '');
});

// Usage — each component only subscribes to what it reads
function FilterBadge() {
  const count = useAtomValue(activeFilterCountAtom);
  return count > 0 ? <Badge count={count} /> : null;
}

function CategoryFilter() {
  const [category, setCategory] = useAtom(categoryAtom);
  // Only re-renders when category changes, not when price or sort changes
  return <Picker value={category} onChange={setCategory} />;
}

function ResetButton() {
  const [, reset] = useAtom(resetFiltersAtom);
  return <Button onPress={reset} title="Clear Filters" />;
}
```

Each component subscribes to exactly the atoms it uses. Changing the category doesn't re-render the price range component. This granularity is Jotai's core advantage.

### 5.3 Async Atoms

Jotai atoms can be asynchronous. An async read atom behaves like a mini data-fetching hook:

```tsx
import { atom, useAtomValue } from 'jotai';
import { Suspense } from 'react';

const userIdAtom = atom<string>('user-1');

// Async derived atom — fetches when userId changes
const userAtom = atom(async (get) => {
  const userId = get(userIdAtom);
  const response = await fetch(`/api/users/${userId}`);
  return response.json() as Promise<User>;
});

// Async atom that depends on another async atom
const userPostsAtom = atom(async (get) => {
  const user = await get(userAtom); // Awaits the user atom
  const response = await fetch(`/api/users/${user.id}/posts`);
  return response.json() as Promise<Post[]>;
});

// Usage — wrap in Suspense
function UserProfile() {
  return (
    <Suspense fallback={<Spinner />}>
      <UserContent />
    </Suspense>
  );
}

function UserContent() {
  const user = useAtomValue(userAtom);       // Suspends until resolved
  const posts = useAtomValue(userPostsAtom); // Suspends until resolved
  return (
    <View>
      <Text>{user.name}</Text>
      <Text>{posts.length} posts</Text>
    </View>
  );
}
```

**Important caveat:** Async atoms are great for simple cases, but they don't give you TanStack Query's caching, background refetch, or optimistic updates. For server state, TanStack Query is still the right tool. Use async atoms for client-side computations that happen to be async (e.g., reading from IndexedDB, computing a heavy filter).

### 5.4 Atom Families

When you need a family of similar atoms (e.g., one atom per item in a list), use `atomFamily`:

```tsx
import { atom } from 'jotai';
import { atomFamily } from 'jotai/utils';

// Creates a unique atom for each todoId
const todoAtomFamily = atomFamily((todoId: string) =>
  atom<Todo>({
    id: todoId,
    text: '',
    completed: false,
  })
);

// Atom that tracks all todo IDs
const todoIdsAtom = atom<string[]>([]);

// Usage — each todo item gets its own atom
function TodoItem({ todoId }: { todoId: string }) {
  const [todo, setTodo] = useAtom(todoAtomFamily(todoId));
  // Only THIS todo item re-renders when its data changes
  // Other todo items are completely unaffected
  return (
    <View>
      <Checkbox
        value={todo.completed}
        onValueChange={(completed) => setTodo({ ...todo, completed })}
      />
      <Text>{todo.text}</Text>
    </View>
  );
}

function TodoList() {
  const [todoIds] = useAtom(todoIdsAtom);
  return (
    <View>
      {todoIds.map((id) => (
        <TodoItem key={id} todoId={id} />
      ))}
    </View>
  );
}
```

Atom families give you per-item state isolation, which is critical for large lists where re-rendering the entire list on one item's change would be expensive.

### 5.5 atomWithStorage: Persistent Atoms

```tsx
import { atomWithStorage, createJSONStorage } from 'jotai/utils';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

const mmkvStorage = createJSONStorage<any>(() => ({
  getItem: (key: string) => storage.getString(key) ?? null,
  setItem: (key: string, value: string) => storage.set(key, value),
  removeItem: (key: string) => storage.delete(key),
}));

// This atom automatically persists to MMKV
const themeAtom = atomWithStorage<'light' | 'dark'>('theme', 'light', mmkvStorage);
const onboardingCompleteAtom = atomWithStorage('onboarding-complete', false, mmkvStorage);
const recentSearchesAtom = atomWithStorage<string[]>('recent-searches', [], mmkvStorage);

// Usage is identical to a regular atom
function ThemeToggle() {
  const [theme, setTheme] = useAtom(themeAtom);
  // Persists automatically. Kill the app, reopen — theme is preserved.
  return (
    <Switch
      value={theme === 'dark'}
      onValueChange={() => setTheme(theme === 'light' ? 'dark' : 'light')}
    />
  );
}
```

---

## 6. OFFLINE-FIRST STATE: LEGEND STATE

Legend State is purpose-built for **offline-first** React Native apps. It integrates MMKV persistence and server sync directly into the state layer. Where Zustand + TanStack Query requires you to build offline support yourself, Legend State makes it the default behavior.

### 6.1 Basic Setup

```tsx
import { observable } from '@legendapp/state';
import { synced } from '@legendapp/state/sync';
import { ObservablePersistMMKV } from '@legendapp/state/persist-plugins/mmkv';

const tasks$ = observable(
  synced({
    initial: [] as Task[],
    persist: { name: 'tasks', plugin: ObservablePersistMMKV },
    sync: {
      get: () => fetch('/api/tasks').then(r => r.json()),
      set: ({ value }) => fetch('/api/tasks', {
        method: 'POST',
        body: JSON.stringify(value),
      }),
    },
  })
);
// Changes made offline are queued and synced when connectivity returns
```

### 6.2 Complete Offline Sync with Conflict Resolution

Here's a real-world example: a field service app where technicians create and update work orders offline. Multiple technicians might update the same work order. When they reconnect, conflicts need resolution.

```tsx
import { observable, when } from '@legendapp/state';
import { synced } from '@legendapp/state/sync';
import { ObservablePersistMMKV } from '@legendapp/state/persist-plugins/mmkv';
import NetInfo from '@react-native-community/netinfo';

// --- Types ---
interface WorkOrder {
  id: string;
  title: string;
  status: 'pending' | 'in-progress' | 'completed';
  notes: string;
  assigneeId: string;
  updatedAt: number; // Timestamp for conflict resolution
  version: number;   // Optimistic locking version
}

interface SyncMetadata {
  lastSyncedAt: number;
  pendingChanges: PendingChange[];
}

interface PendingChange {
  id: string;
  workOrderId: string;
  type: 'create' | 'update' | 'delete';
  payload: Partial<WorkOrder>;
  timestamp: number;
}

// --- Conflict Resolution ---
type ConflictStrategy = 'last-write-wins' | 'server-wins' | 'client-wins' | 'manual';

function resolveConflict(
  clientVersion: WorkOrder,
  serverVersion: WorkOrder,
  strategy: ConflictStrategy
): WorkOrder {
  switch (strategy) {
    case 'last-write-wins':
      // Whoever wrote last wins
      return clientVersion.updatedAt > serverVersion.updatedAt
        ? clientVersion
        : serverVersion;

    case 'server-wins':
      return serverVersion;

    case 'client-wins':
      return clientVersion;

    case 'manual':
      // Field-level merge: take the newest value for each field
      return {
        ...serverVersion,
        ...Object.fromEntries(
          Object.entries(clientVersion).filter(([key, value]) => {
            if (key === 'updatedAt' || key === 'version') return false;
            return value !== serverVersion[key as keyof WorkOrder];
          })
        ),
        updatedAt: Math.max(clientVersion.updatedAt, serverVersion.updatedAt),
        version: Math.max(clientVersion.version, serverVersion.version) + 1,
      };
  }
}

// --- Observable Store with Sync ---
const workOrders$ = observable(
  synced({
    initial: {} as Record<string, WorkOrder>,
    persist: {
      name: 'work-orders',
      plugin: ObservablePersistMMKV,
    },
    sync: {
      get: async () => {
        const response = await fetch('/api/work-orders');
        const orders: WorkOrder[] = await response.json();
        // Convert array to record keyed by ID
        return Object.fromEntries(orders.map((o) => [o.id, o]));
      },
      set: async ({ value, changes }) => {
        // Send only changed items
        const changedOrders = changes.map((change) => ({
          ...change,
          payload: value[change.path[0] as string],
        }));

        const response = await fetch('/api/work-orders/batch', {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ orders: changedOrders }),
        });

        const result = await response.json();

        // Handle conflicts
        if (result.conflicts?.length > 0) {
          for (const conflict of result.conflicts) {
            const clientVersion = value[conflict.workOrderId];
            const serverVersion = conflict.serverVersion;
            const resolved = resolveConflict(
              clientVersion,
              serverVersion,
              'last-write-wins'
            );
            // Update local state with resolved version
            workOrders$[conflict.workOrderId].set(resolved);
          }
        }

        return result;
      },
    },
    retry: {
      infinite: true,     // Keep retrying until connectivity returns
      backoff: 'exponential',
      maxDelay: 30000,    // Max 30 seconds between retries
    },
  })
);

// --- Sync status tracking ---
const syncStatus$ = observable<{
  isOnline: boolean;
  isSyncing: boolean;
  lastSyncedAt: number | null;
  pendingChangesCount: number;
}>({
  isOnline: true,
  isSyncing: false,
  lastSyncedAt: null,
  pendingChangesCount: 0,
});

// Monitor connectivity
NetInfo.addEventListener((state) => {
  syncStatus$.isOnline.set(!!state.isConnected);
});

// --- Usage in components ---
function WorkOrderCard({ orderId }: { orderId: string }) {
  // Legend State reactive component — auto re-renders on change
  const order = workOrders$[orderId].get();
  const isOnline = syncStatus$.isOnline.get();

  const updateStatus = (newStatus: WorkOrder['status']) => {
    workOrders$[orderId].set({
      ...order,
      status: newStatus,
      updatedAt: Date.now(),
      version: order.version + 1,
    });
    // If offline, change is persisted to MMKV and synced when online
    // If online, sync happens immediately
  };

  return (
    <View>
      <Text>{order.title}</Text>
      <Text>Status: {order.status}</Text>
      {!isOnline && <Text style={styles.offlineBadge}>Offline — changes will sync</Text>}
      <Button title="Mark Complete" onPress={() => updateStatus('completed')} />
    </View>
  );
}
```

**Pick Legend State when:** You're building an app for unreliable connectivity (field work, travel, rural areas) and need offline-first with automatic sync. If your users will frequently make changes offline, Legend State's built-in conflict resolution and sync queue will save you weeks of custom implementation.

---

## 7. FORM STATE: REACT HOOK FORM + ZOD

Form state is the third category, and it's the one most teams get wrong by putting it in global state.

Forms are ephemeral. They exist while the user is filling them out and disappear when submitted. Putting form values in Redux or Zustand means:
- Global state updates on every keystroke
- Re-renders propagate beyond the form
- Form cleanup requires explicit actions
- Form state outlives the form (stale data on re-mount)

React Hook Form avoids all of this by storing form state in refs, not React state. Keystrokes don't trigger re-renders. Only validation and submission do.

### 7.1 Basic Form with Zod Validation

```tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const profileSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  bio: z.string().max(500, 'Bio must be under 500 characters').optional(),
});

type ProfileForm = z.infer<typeof profileSchema>;

function EditProfile() {
  const { control, handleSubmit, formState: { errors, isSubmitting } } = useForm<ProfileForm>({
    resolver: zodResolver(profileSchema),
    defaultValues: { name: '', email: '', bio: '' },
  });

  const onSubmit = async (data: ProfileForm) => {
    await api.updateProfile(data);
  };

  return (
    <View>
      <Controller
        control={control}
        name="name"
        render={({ field: { onChange, value } }) => (
          <TextInput value={value} onChangeText={onChange} />
        )}
      />
      {errors.name && <Text style={styles.error}>{errors.name.message}</Text>}
      {/* ... more fields ... */}
      <Button title="Save" onPress={handleSubmit(onSubmit)} disabled={isSubmitting} />
    </View>
  );
}
```

**Why Zod:** Zod schemas give you runtime validation (for form input) AND TypeScript types (for type safety) from a single source of truth. The `z.infer<typeof schema>` pattern means you never manually write form types that can drift from your validation rules.

### 7.2 Multi-Step Form

Multi-step forms (wizards) are one of the hardest UI patterns to get right. Here's how to do it with React Hook Form — each step validates independently, but all data is collected in a single form instance:

```tsx
import { useForm, FormProvider, useFormContext } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Schema for the complete form
const checkoutSchema = z.object({
  // Step 1: Shipping
  firstName: z.string().min(1, 'Required'),
  lastName: z.string().min(1, 'Required'),
  address: z.string().min(5, 'Address too short'),
  city: z.string().min(1, 'Required'),
  zipCode: z.string().regex(/^\d{5}(-\d{4})?$/, 'Invalid zip code'),
  // Step 2: Payment
  cardNumber: z.string().regex(/^\d{16}$/, 'Must be 16 digits'),
  expiry: z.string().regex(/^\d{2}\/\d{2}$/, 'Format: MM/YY'),
  cvv: z.string().regex(/^\d{3,4}$/, 'Must be 3-4 digits'),
  // Step 3: Review (no additional fields)
});

type CheckoutFormData = z.infer<typeof checkoutSchema>;

// Fields for each step — used for per-step validation
const stepFields: Record<number, (keyof CheckoutFormData)[]> = {
  0: ['firstName', 'lastName', 'address', 'city', 'zipCode'],
  1: ['cardNumber', 'expiry', 'cvv'],
  2: [], // Review step — no fields to validate
};

function CheckoutWizard() {
  const [step, setStep] = useState(0);

  const methods = useForm<CheckoutFormData>({
    resolver: zodResolver(checkoutSchema),
    defaultValues: {
      firstName: '', lastName: '', address: '', city: '', zipCode: '',
      cardNumber: '', expiry: '', cvv: '',
    },
    mode: 'onBlur', // Validate on blur for better UX
  });

  const goToNext = async () => {
    // Validate only the current step's fields
    const fieldsToValidate = stepFields[step];
    const isValid = await methods.trigger(fieldsToValidate);
    if (isValid) setStep((s) => s + 1);
  };

  const goToPrevious = () => setStep((s) => s - 1);

  const onSubmit = async (data: CheckoutFormData) => {
    await api.createOrder(data);
  };

  return (
    <FormProvider {...methods}>
      <View>
        <StepIndicator currentStep={step} totalSteps={3} />

        {step === 0 && <ShippingStep />}
        {step === 1 && <PaymentStep />}
        {step === 2 && <ReviewStep />}

        <View style={styles.buttons}>
          {step > 0 && (
            <Button title="Back" onPress={goToPrevious} />
          )}
          {step < 2 ? (
            <Button title="Next" onPress={goToNext} />
          ) : (
            <Button
              title="Place Order"
              onPress={methods.handleSubmit(onSubmit)}
              disabled={methods.formState.isSubmitting}
            />
          )}
        </View>
      </View>
    </FormProvider>
  );
}

// Each step is a separate component that uses FormProvider context
function ShippingStep() {
  const { control, formState: { errors } } = useFormContext<CheckoutFormData>();
  return (
    <View>
      <Controller
        control={control}
        name="firstName"
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput
            placeholder="First Name"
            value={value}
            onChangeText={onChange}
            onBlur={onBlur}
          />
        )}
      />
      {errors.firstName && <Text style={styles.error}>{errors.firstName.message}</Text>}
      {/* ... more fields ... */}
    </View>
  );
}
```

The key insight: `methods.trigger(fieldsToValidate)` validates only the specified fields, so you can validate step-by-step without submitting the form. The full form data is preserved across steps because React Hook Form stores everything in a single form instance.

### 7.3 Dynamic Fields with useFieldArray

For forms where users add/remove items — order line items, team members, address lists:

```tsx
import { useForm, useFieldArray, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const teamSchema = z.object({
  teamName: z.string().min(1, 'Team name required'),
  members: z.array(z.object({
    name: z.string().min(1, 'Name required'),
    email: z.string().email('Invalid email'),
    role: z.enum(['admin', 'member', 'viewer']),
  })).min(1, 'At least one member required')
    .max(20, 'Maximum 20 members'),
});

type TeamForm = z.infer<typeof teamSchema>;

function TeamSetupForm() {
  const { control, handleSubmit, formState: { errors } } = useForm<TeamForm>({
    resolver: zodResolver(teamSchema),
    defaultValues: {
      teamName: '',
      members: [{ name: '', email: '', role: 'member' }],
    },
  });

  const { fields, append, remove, move } = useFieldArray({
    control,
    name: 'members',
  });

  return (
    <View>
      <Controller
        control={control}
        name="teamName"
        render={({ field: { onChange, value } }) => (
          <TextInput placeholder="Team Name" value={value} onChangeText={onChange} />
        )}
      />

      {fields.map((field, index) => (
        <View key={field.id} style={styles.memberRow}>
          <Controller
            control={control}
            name={`members.${index}.name`}
            render={({ field: { onChange, value } }) => (
              <TextInput placeholder="Name" value={value} onChangeText={onChange} />
            )}
          />
          <Controller
            control={control}
            name={`members.${index}.email`}
            render={({ field: { onChange, value } }) => (
              <TextInput placeholder="Email" value={value} onChangeText={onChange} />
            )}
          />
          <Controller
            control={control}
            name={`members.${index}.role`}
            render={({ field: { onChange, value } }) => (
              <Picker selectedValue={value} onValueChange={onChange}>
                <Picker.Item label="Admin" value="admin" />
                <Picker.Item label="Member" value="member" />
                <Picker.Item label="Viewer" value="viewer" />
              </Picker>
            )}
          />
          {fields.length > 1 && (
            <Pressable onPress={() => remove(index)}>
              <Text>Remove</Text>
            </Pressable>
          )}
          {errors.members?.[index]?.email && (
            <Text style={styles.error}>
              {errors.members[index].email?.message}
            </Text>
          )}
        </View>
      ))}

      <Button
        title="Add Member"
        onPress={() => append({ name: '', email: '', role: 'member' })}
        disabled={fields.length >= 20}
      />
      {errors.members?.root && (
        <Text style={styles.error}>{errors.members.root.message}</Text>
      )}

      <Button title="Create Team" onPress={handleSubmit(onSubmit)} />
    </View>
  );
}
```

`useFieldArray` gives you `append`, `remove`, `move`, `insert`, `prepend`, `swap`, and `update` — all the operations you need for dynamic lists, without manual array index management.

### 7.4 File Upload Integration

Forms with file uploads need special handling because files aren't serializable JSON:

```tsx
const postSchema = z.object({
  title: z.string().min(1, 'Title required'),
  body: z.string().min(10, 'Body must be at least 10 characters'),
  image: z
    .custom<File>()
    .refine((file) => file?.size <= 5 * 1024 * 1024, 'Max file size is 5MB')
    .refine(
      (file) => ['image/jpeg', 'image/png', 'image/webp'].includes(file?.type),
      'Only JPEG, PNG, and WebP images are accepted'
    )
    .optional(),
});

type PostForm = z.infer<typeof postSchema>;

function CreatePostForm() {
  const { control, handleSubmit, setValue, watch } = useForm<PostForm>({
    resolver: zodResolver(postSchema),
  });

  const selectedImage = watch('image');

  const pickImage = async () => {
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      quality: 0.8,
    });

    if (!result.canceled) {
      const asset = result.assets[0];
      // Create a File-like object for validation
      const file = {
        uri: asset.uri,
        type: asset.mimeType ?? 'image/jpeg',
        name: asset.fileName ?? 'photo.jpg',
        size: asset.fileSize ?? 0,
      };
      setValue('image', file as any, { shouldValidate: true });
    }
  };

  const onSubmit = async (data: PostForm) => {
    const formData = new FormData();
    formData.append('title', data.title);
    formData.append('body', data.body);
    if (data.image) {
      formData.append('image', data.image);
    }
    await api.createPost(formData);
  };

  return (
    <View>
      {/* title and body controllers ... */}
      <Pressable onPress={pickImage} style={styles.imagePicker}>
        {selectedImage ? (
          <Image source={{ uri: (selectedImage as any).uri }} style={styles.preview} />
        ) : (
          <Text>Tap to add an image</Text>
        )}
      </Pressable>
      <Button title="Publish" onPress={handleSubmit(onSubmit)} />
    </View>
  );
}
```

---

## 8. FINITE STATES AND TYPE STATES

Remember the `useState` explosion from Section 2.4? Seven boolean flags with 128 possible combinations, only 4 of which are valid? Here's the fix.

### 8.1 Finite States: Replace Booleans with Enums

```tsx
// ❌ BAD: Boolean soup
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [data, setData] = useState<User | null>(null);
const [error, setError] = useState<Error | null>(null);

// ✅ GOOD: Finite states
type Status = 'idle' | 'loading' | 'error' | 'success';
const [status, setStatus] = useState<Status>('idle');
const [data, setData] = useState<User | null>(null);
const [error, setError] = useState<Error | null>(null);

// Even better: combine into a single state
if (status === 'loading') return <Spinner />;
if (status === 'error') return <ErrorView error={error!} />;
if (status === 'success') return <UserProfile user={data!} />;
return <EmptyState />;
```

### 8.2 Type States: Discriminated Unions

Take it further with TypeScript discriminated unions — make invalid states **unrepresentable:**

```tsx
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: T };

// TypeScript enforces that:
// - When status is 'error', error is always present
// - When status is 'success', data is always present
// - You can NEVER have data and error simultaneously
// - There's no combination of fields that represents an invalid state

function UserProfile() {
  const [state, setState] = useState<RequestState<User>>({ status: 'idle' });

  switch (state.status) {
    case 'idle': return <Text>Ready to load</Text>;
    case 'loading': return <Spinner />;
    case 'error': return <Text>Error: {state.error.message}</Text>;
    case 'success': return <Text>Hello, {state.data.name}</Text>;
    // TypeScript enforces exhaustiveness — forget a case and it's a compile error
  }
}
```

This pattern makes impossible states impossible to construct. You literally cannot write the bug where `isLoading` and `isSuccess` are both true.

### 8.3 XState Store: A Checkout Flow State Machine

For complex flows with multiple steps, guards, and side effects, a state machine formalizes the logic. XState Store provides a lightweight state machine without the full XState actor model:

```tsx
import { createStore } from '@xstate/store';

// --- Types ---
interface CheckoutContext {
  cart: CartItem[];
  shippingAddress: Address | null;
  paymentMethod: PaymentMethod | null;
  orderId: string | null;
  error: string | null;
  promoCode: string | null;
  promoDiscount: number;
}

// --- The Store (State Machine) ---
const checkoutStore = createStore({
  context: {
    cart: [],
    shippingAddress: null,
    paymentMethod: null,
    orderId: null,
    error: null,
    promoCode: null,
    promoDiscount: 0,
  } satisfies CheckoutContext,

  on: {
    SET_CART: (context, event: { items: CartItem[] }) => ({
      ...context,
      cart: event.items,
      error: null,
    }),

    SET_SHIPPING: (context, event: { address: Address }) => {
      // Guard: cart must not be empty
      if (context.cart.length === 0) {
        return { ...context, error: 'Cart is empty' };
      }
      return {
        ...context,
        shippingAddress: event.address,
        error: null,
      };
    },

    SET_PAYMENT: (context, event: { payment: PaymentMethod }) => {
      // Guard: shipping must be set first
      if (!context.shippingAddress) {
        return { ...context, error: 'Set shipping address first' };
      }
      return {
        ...context,
        paymentMethod: event.payment,
        error: null,
      };
    },

    APPLY_PROMO: (context, event: { code: string; discount: number }) => ({
      ...context,
      promoCode: event.code,
      promoDiscount: event.discount,
      error: null,
    }),

    REMOVE_PROMO: (context) => ({
      ...context,
      promoCode: null,
      promoDiscount: 0,
    }),

    PLACE_ORDER_SUCCESS: (context, event: { orderId: string }) => ({
      ...context,
      orderId: event.orderId,
      error: null,
    }),

    PLACE_ORDER_FAILURE: (context, event: { error: string }) => ({
      ...context,
      error: event.error,
    }),

    RESET: () => ({
      cart: [],
      shippingAddress: null,
      paymentMethod: null,
      orderId: null,
      error: null,
      promoCode: null,
      promoDiscount: 0,
    }),
  },
});

// --- Derived values ---
function getCheckoutStep(context: CheckoutContext): 'cart' | 'shipping' | 'payment' | 'review' | 'confirmation' {
  if (context.orderId) return 'confirmation';
  if (context.paymentMethod) return 'review';
  if (context.shippingAddress) return 'payment';
  if (context.cart.length > 0) return 'shipping';
  return 'cart';
}

function getOrderTotal(context: CheckoutContext): number {
  const subtotal = context.cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
  return Math.max(0, subtotal - context.promoDiscount);
}

// --- Usage in React ---
function CheckoutFlow() {
  const snapshot = useSyncExternalStore(
    checkoutStore.subscribe,
    () => checkoutStore.getSnapshot().context
  );

  const step = getCheckoutStep(snapshot);
  const total = getOrderTotal(snapshot);

  const placeOrder = async () => {
    try {
      const result = await api.createOrder({
        items: snapshot.cart,
        shippingAddress: snapshot.shippingAddress!,
        paymentMethod: snapshot.paymentMethod!,
        promoCode: snapshot.promoCode,
      });
      checkoutStore.send({ type: 'PLACE_ORDER_SUCCESS', orderId: result.orderId });
    } catch (err) {
      checkoutStore.send({
        type: 'PLACE_ORDER_FAILURE',
        error: err instanceof Error ? err.message : 'Order failed',
      });
    }
  };

  return (
    <View>
      <StepIndicator step={step} />
      {snapshot.error && <ErrorBanner message={snapshot.error} />}

      {step === 'cart' && (
        <CartStep
          items={snapshot.cart}
          onUpdateCart={(items) => checkoutStore.send({ type: 'SET_CART', items })}
        />
      )}
      {step === 'shipping' && (
        <ShippingStep
          onSubmit={(address) => checkoutStore.send({ type: 'SET_SHIPPING', address })}
        />
      )}
      {step === 'payment' && (
        <PaymentStep
          onSubmit={(payment) => checkoutStore.send({ type: 'SET_PAYMENT', payment })}
        />
      )}
      {step === 'review' && (
        <ReviewStep
          context={snapshot}
          total={total}
          onPlaceOrder={placeOrder}
        />
      )}
      {step === 'confirmation' && (
        <ConfirmationStep
          orderId={snapshot.orderId!}
          onNewOrder={() => checkoutStore.send({ type: 'RESET' })}
        />
      )}
    </View>
  );
}
```

The state machine enforces order: you can't set payment before shipping, you can't submit an empty cart. The guards are explicit and centralized. If you need to add a new step (e.g., gift wrapping), you add one event handler — not scattered `if` statements across five components.

---

## 9. EXTERNAL STORE PATTERNS: useSyncExternalStore

React 18 introduced `useSyncExternalStore` for safely subscribing to external data sources. This is the primitive that Zustand and other state managers use under the hood. You should reach for it when you need to sync React with something that isn't React state.

### 9.1 Real Example: Syncing with Browser/Device APIs

```tsx
import { useSyncExternalStore } from 'react';

// --- Online status ---
function subscribeOnlineStatus(callback: () => void) {
  window.addEventListener('online', callback);
  window.addEventListener('offline', callback);
  return () => {
    window.removeEventListener('online', callback);
    window.removeEventListener('offline', callback);
  };
}

function getOnlineStatus() {
  return navigator.onLine;
}

function getServerOnlineStatus() {
  return true; // SSR fallback
}

function useOnlineStatus() {
  return useSyncExternalStore(
    subscribeOnlineStatus,
    getOnlineStatus,
    getServerOnlineStatus  // Used during SSR
  );
}

// --- Window dimensions ---
function useWindowSize() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }),
    () => ({ width: 0, height: 0 }) // SSR fallback
  );
}
```

### 9.2 Syncing with a Third-Party SDK

Many SDKs have their own internal state that your React app needs to reflect:

```tsx
// Syncing with a WebSocket connection manager
class WebSocketManager {
  private listeners = new Set<() => void>();
  private _status: 'connecting' | 'connected' | 'disconnected' = 'disconnected';
  private _messages: Message[] = [];

  get status() { return this._status; }
  get messages() { return this._messages; }

  connect(url: string) {
    this._status = 'connecting';
    this.notify();

    this.ws = new WebSocket(url);
    this.ws.onopen = () => {
      this._status = 'connected';
      this.notify();
    };
    this.ws.onmessage = (event) => {
      this._messages = [...this._messages, JSON.parse(event.data)];
      this.notify();
    };
    this.ws.onclose = () => {
      this._status = 'disconnected';
      this.notify();
    };
  }

  subscribe(listener: () => void) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  private notify() {
    this.listeners.forEach((l) => l());
  }

  getSnapshot() {
    return { status: this._status, messages: this._messages };
  }
}

const wsManager = new WebSocketManager();

// React hook that subscribes to the WebSocket manager
function useWebSocket() {
  const snapshot = useSyncExternalStore(
    (cb) => wsManager.subscribe(cb),
    () => wsManager.getSnapshot()
  );
  return snapshot;
}

// Usage
function ChatScreen() {
  const { status, messages } = useWebSocket();

  if (status === 'connecting') return <Spinner />;
  if (status === 'disconnected') return <ReconnectButton />;

  return (
    <FlatList
      data={messages}
      renderItem={({ item }) => <MessageBubble message={item} />}
    />
  );
}
```

The pattern: the external system (WebSocket, SDK, browser API) manages its own state. `useSyncExternalStore` provides a safe bridge to React's rendering model, ensuring updates are consistent even during concurrent rendering.

---

## 10. URL STATE MANAGEMENT

There's a fourth category of state that doesn't fit neatly into the three-category model: **URL state**. This is state that should be reflected in the URL for sharing, bookmarking, and browser back/forward navigation.

### 10.1 What Belongs in the URL

- Search queries and filters
- Pagination (current page, page size)
- Sort order
- Selected tab or view mode
- Modal open state (for shareable deep links)
- Any state where the user would expect "copy URL" to preserve their current view

### 10.2 Using nuqs for URL State

`nuqs` (or `next-usequerystate` for Next.js) provides type-safe URL search param management:

```tsx
import { useQueryState, parseAsString, parseAsInteger, parseAsStringEnum } from 'nuqs';

// Basic usage — syncs state with ?search=...
function ProductSearch() {
  const [search, setSearch] = useQueryState('search', parseAsString.withDefault(''));
  const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1));
  const [sort, setSort] = useQueryState(
    'sort',
    parseAsStringEnum(['price', 'rating', 'newest']).withDefault('newest')
  );
  const [category, setCategory] = useQueryState('category', parseAsString.withDefault('all'));

  // URL reflects state: /products?search=shoes&page=2&sort=price&category=footwear
  // User can bookmark or share this URL and get the exact same view

  const { data: products } = useQuery({
    queryKey: productQueries.list({ search, page, sort, category }),
    queryFn: () => api.getProducts({ search, page, sort, category }),
  });

  return (
    <View>
      <TextInput
        value={search}
        onChangeText={(text) => {
          setSearch(text);
          setPage(1); // Reset page when search changes
        }}
        placeholder="Search products..."
      />
      <SortPicker value={sort} onChange={setSort} />
      <CategoryFilter value={category} onChange={setCategory} />
      <ProductGrid products={products?.items ?? []} />
      <Pagination
        currentPage={page}
        totalPages={products?.totalPages ?? 1}
        onPageChange={setPage}
      />
    </View>
  );
}
```

### 10.3 URL State + TanStack Query Integration

The pattern becomes powerful when URL state drives query keys:

```tsx
import { useQueryStates, parseAsString, parseAsInteger, parseAsArrayOf } from 'nuqs';

function useProductFilters() {
  // Batch multiple query params
  const [filters, setFilters] = useQueryStates({
    search: parseAsString.withDefault(''),
    page: parseAsInteger.withDefault(1),
    sort: parseAsString.withDefault('newest'),
    categories: parseAsArrayOf(parseAsString, ',').withDefault([]),
    minPrice: parseAsInteger.withDefault(0),
    maxPrice: parseAsInteger.withDefault(10000),
  });

  return { filters, setFilters };
}

function ProductListScreen() {
  const { filters, setFilters } = useProductFilters();

  // URL state flows directly into query key — cache is per-URL
  const { data, isLoading } = useQuery({
    queryKey: productQueries.list(filters),
    queryFn: () => api.getProducts(filters),
    placeholderData: keepPreviousData, // Show old data while new data loads
  });

  const updateFilter = (key: string, value: any) => {
    setFilters({
      ...filters,
      [key]: value,
      page: key === 'page' ? value : 1, // Reset page on filter change
    });
  };

  // ...
}
```

Benefits: the browser back button undoes filter changes. Users can share filter combinations via URL. Each unique set of filters has its own cache entry in TanStack Query. Pagination works with browser history.

### 10.4 When URL State vs Client State

| Use URL State When | Use Client State (Zustand) When |
|---|---|
| User expects to share/bookmark the view | State is private to the session |
| Browser back should undo the change | Back button shouldn't affect it |
| State represents "what am I looking at" | State represents "how am I looking at it" |
| Search filters, pagination, sort | Theme, sidebar toggle, tooltips |
| Selected tab in a shareable view | Which accordion is expanded |

---

## 11. DATA NORMALIZATION

When your server returns nested data, storing it as-is creates problems. Here's a complete before/after with a social media feed — the kind of data structure that breaks without normalization.

### 11.1 The Problem: Nested Data

```tsx
// Server returns a feed like this:
const feedResponse = {
  posts: [
    {
      id: 'post-1',
      text: 'Just shipped the new feature!',
      createdAt: '2026-04-07T10:00:00Z',
      author: { id: 'user-1', name: 'Alice', avatar: 'alice.jpg' },
      likes: [
        { id: 'user-2', name: 'Bob', avatar: 'bob.jpg' },
        { id: 'user-3', name: 'Charlie', avatar: 'charlie.jpg' },
      ],
      comments: [
        {
          id: 'comment-1',
          text: 'Amazing work!',
          author: { id: 'user-2', name: 'Bob', avatar: 'bob.jpg' },
          replies: [
            {
              id: 'comment-2',
              text: 'Thanks Bob!',
              author: { id: 'user-1', name: 'Alice', avatar: 'alice.jpg' },
            },
          ],
        },
        {
          id: 'comment-3',
          text: 'When does it go live?',
          author: { id: 'user-3', name: 'Charlie', avatar: 'charlie.jpg' },
          replies: [],
        },
      ],
    },
    {
      id: 'post-2',
      text: 'Coffee break thoughts...',
      createdAt: '2026-04-07T09:30:00Z',
      author: { id: 'user-2', name: 'Bob', avatar: 'bob.jpg' },
      likes: [
        { id: 'user-1', name: 'Alice', avatar: 'alice.jpg' },
      ],
      comments: [],
    },
  ],
};

// Problems:
// 1. Alice appears 3 times (as author, as liker, as commenter).
//    Update her avatar? You need to find every occurrence.
// 2. Bob appears 3 times. Same problem.
// 3. Deep nesting makes updates painful (spread spread spread).
// 4. If Bob changes his name, some copies show "Bob", others show "Robert".
```

### 11.2 The Solution: Normalize

```tsx
// --- Types ---
interface NormalizedState {
  users: Record<string, NormalizedUser>;
  posts: Record<string, NormalizedPost>;
  comments: Record<string, NormalizedComment>;
  feedOrder: string[]; // Ordered list of post IDs for the feed
}

interface NormalizedUser {
  id: string;
  name: string;
  avatar: string;
}

interface NormalizedPost {
  id: string;
  text: string;
  createdAt: string;
  authorId: string;
  likeUserIds: string[];
  commentIds: string[];
}

interface NormalizedComment {
  id: string;
  text: string;
  authorId: string;
  replyIds: string[];
  postId: string;
}

// --- Normalizer ---
function normalizeFeed(response: FeedResponse): NormalizedState {
  const state: NormalizedState = {
    users: {},
    posts: {},
    comments: {},
    feedOrder: [],
  };

  for (const post of response.posts) {
    // Normalize the author
    state.users[post.author.id] = post.author;

    // Normalize likes (just user references)
    for (const liker of post.likes) {
      state.users[liker.id] = liker;
    }

    // Normalize comments recursively
    const commentIds: string[] = [];
    for (const comment of post.comments) {
      state.users[comment.author.id] = comment.author;
      const replyIds: string[] = [];

      for (const reply of comment.replies) {
        state.users[reply.author.id] = reply.author;
        state.comments[reply.id] = {
          id: reply.id,
          text: reply.text,
          authorId: reply.author.id,
          replyIds: [],
          postId: post.id,
        };
        replyIds.push(reply.id);
      }

      state.comments[comment.id] = {
        id: comment.id,
        text: comment.text,
        authorId: comment.author.id,
        replyIds,
        postId: post.id,
      };
      commentIds.push(comment.id);
    }

    // Normalize the post
    state.posts[post.id] = {
      id: post.id,
      text: post.text,
      createdAt: post.createdAt,
      authorId: post.author.id,
      likeUserIds: post.likes.map((l) => l.id),
      commentIds,
    };
    state.feedOrder.push(post.id);
  }

  return state;
}
```

### 11.3 After Normalization

```tsx
// The same data, normalized:
const normalized: NormalizedState = {
  users: {
    'user-1': { id: 'user-1', name: 'Alice', avatar: 'alice.jpg' },
    'user-2': { id: 'user-2', name: 'Bob', avatar: 'bob.jpg' },
    'user-3': { id: 'user-3', name: 'Charlie', avatar: 'charlie.jpg' },
  },
  posts: {
    'post-1': {
      id: 'post-1',
      text: 'Just shipped the new feature!',
      createdAt: '2026-04-07T10:00:00Z',
      authorId: 'user-1',
      likeUserIds: ['user-2', 'user-3'],
      commentIds: ['comment-1', 'comment-3'],
    },
    'post-2': {
      id: 'post-2',
      text: 'Coffee break thoughts...',
      createdAt: '2026-04-07T09:30:00Z',
      authorId: 'user-2',
      likeUserIds: ['user-1'],
      commentIds: [],
    },
  },
  comments: {
    'comment-1': {
      id: 'comment-1',
      text: 'Amazing work!',
      authorId: 'user-2',
      replyIds: ['comment-2'],
      postId: 'post-1',
    },
    'comment-2': {
      id: 'comment-2',
      text: 'Thanks Bob!',
      authorId: 'user-1',
      replyIds: [],
      postId: 'post-1',
    },
    'comment-3': {
      id: 'comment-3',
      text: 'When does it go live?',
      authorId: 'user-3',
      replyIds: [],
      postId: 'post-1',
    },
  },
  feedOrder: ['post-1', 'post-2'],
};

// Now:
// 1. Alice exists in exactly ONE place. Update her avatar once, done.
// 2. Adding a like is: post.likeUserIds.push(userId) — no nested spread.
// 3. Adding a comment is: create the comment, push its ID to the post.
// 4. Rendering: look up by ID. O(1) access, no searching.
```

### 11.4 When to Normalize (and When Not To)

**Normalize when:**
- Same entity appears in multiple places (user profiles in posts, comments, likes)
- You need to update an entity and have the change reflected everywhere
- You're building a social app, CMS, or anything with interconnected entities
- Performance matters for large lists (O(1) lookups vs O(n) searches)

**Don't normalize when:**
- Data is read-only and appears in only one context
- The dataset is small (under 100 items)
- TanStack Query's cache handles your use case (each entity has its own query key)
- The normalization complexity outweighs the consistency benefits

**A rule of thumb:** If the same entity ID appears more than three times in your cached data, normalize.

---

## 12. PERFORMANCE DEEP DIVE

State management is the number one cause of performance problems in React applications, and almost always for the same reason: unnecessary re-renders. Not slow renders — *unnecessary* renders.

### 12.1 Measuring Re-Renders

Before optimizing, measure. You can't fix what you can't see.

**React DevTools Profiler:** The built-in React DevTools profiler shows which components rendered and why. Enable "Record why each component rendered" in settings. This is your primary tool.

**Manual render counting (development only):**

```tsx
// Quick and dirty render counter for debugging
function useRenderCount(componentName: string) {
  const renderCount = useRef(0);
  renderCount.current++;

  useEffect(() => {
    if (__DEV__) {
      console.log(`[Render] ${componentName}: ${renderCount.current}`);
    }
  });
}

function ExpensiveList() {
  useRenderCount('ExpensiveList');
  // ...
}
```

**React Scan:** A third-party tool that visually highlights re-rendering components. Drop `<script src="https://unpkg.com/react-scan/dist/auto.global.js">` in your HTML and every re-render shows a visual flash. Extremely effective for spotting unnecessary re-renders.

### 12.2 Why Context Causes Re-Renders

React Context was designed for dependency injection (passing values deep into the tree), not for high-frequency state updates. Here's why:

```tsx
// When the Provider's value changes, React re-renders EVERY consumer
<AppContext.Provider value={state}>
  {children}
</AppContext.Provider>
```

When `state` changes, React walks the tree and re-renders every component that calls `useContext(AppContext)` — even if the specific value that component reads didn't change. There are no selectors. There's no way to say "I only care about `state.theme`."

**Common "fix" that doesn't work:** Splitting the context.

```tsx
// People try this:
const ThemeContext = createContext<string>('light');
const NotificationsContext = createContext<number>(0);
const CartContext = createContext<CartItem[]>([]);

// This helps with cross-category re-renders, but WITHIN a category,
// if CartContext has 5 values and one changes, all cart consumers re-render.
```

**The real fix:** Use a state manager with selectors (Zustand, Jotai). The selector function tells the library exactly what to watch. If the selected value hasn't changed, no re-render.

### 12.3 Selector Optimization Deep Dive

```tsx
// ❌ BAD: Selecting the entire store
function CartBadge() {
  const store = useAppStore(); // Re-renders on ANY store change
  return <Badge count={store.items.length} />;
}

// ❌ ALSO BAD: Creating new objects in selectors
function CartBadge() {
  // This creates a NEW object every render → always re-renders
  const { count, total } = useAppStore((s) => ({
    count: s.items.length,
    total: s.items.reduce((sum, i) => sum + i.price, 0),
  }));
  return <Badge count={count} />;
}

// ✅ GOOD: Primitive selectors (reference equality is trivial)
function CartBadge() {
  const count = useAppStore((s) => s.items.length); // Primitive → stable
  return <Badge count={count} />;
}

// ✅ GOOD: Use useShallow for object selectors
import { useShallow } from 'zustand/react/shallow';

function CartSummary() {
  const { count, total } = useAppStore(
    useShallow((s) => ({
      count: s.items.length,
      total: s.items.reduce((sum, i) => sum + i.price, 0),
    }))
  );
  return <Text>{count} items, ${total}</Text>;
}
```

**Why `useShallow` works:** Zustand uses referential equality (`===`) to decide if a re-render is needed. `{ count: 5 } !== { count: 5 }` because they're different objects, even though the values are identical. `useShallow` does a shallow comparison of the object's properties instead: if `prev.count === next.count && prev.total === next.total`, no re-render.

**The golden rule of selectors:** Select the *narrowest* slice of state you need. If you need a single number, select that number directly — not the object that contains it.

### 12.4 Avoiding Re-Render Cascades with Stable References

```tsx
// ❌ BAD: Creating new callback on every render
function ParentComponent() {
  const items = useAppStore((s) => s.items);

  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <ItemCard
          item={item}
          // New function every render → ItemCard always re-renders
          onPress={() => handlePress(item.id)}
        />
      )}
    />
  );
}

// ✅ GOOD: Stable callback with useCallback
function ParentComponent() {
  const items = useAppStore((s) => s.items);
  const handlePress = useCallback((id: string) => {
    navigation.navigate('Detail', { id });
  }, []);

  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <ItemCard item={item} onPress={handlePress} />
      )}
    />
  );
}

// ✅ EVEN BETTER: Actions from the store are already stable references
function ParentComponent() {
  const items = useAppStore((s) => s.items);
  const removeItem = useAppStore((s) => s.removeItem); // Stable — same function always

  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <ItemCard item={item} onRemove={removeItem} />
      )}
    />
  );
}
```

Zustand store actions are defined once during store creation. Their reference never changes. This means you can pass them directly as props without `useCallback`, and `React.memo`'d children won't re-render.

---

## 13. REDUX TOOLKIT COMPARISON

Some teams are already on Redux Toolkit, or their company mandates it. That's fine — RTK is a solid tool. But understanding the differences helps you make informed decisions.

### 13.1 The Same Feature: Shopping Cart

**Redux Toolkit version:**

```tsx
// cartSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CartState {
  items: CartItem[];
}

const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] } as CartState,
  reducers: {
    addItem: (state, action: PayloadAction<CartItem>) => {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        existing.quantity += 1;
      } else {
        state.items.push({ ...action.payload, quantity: 1 });
      }
    },
    removeItem: (state, action: PayloadAction<string>) => {
      state.items = state.items.filter((i) => i.id !== action.payload);
    },
    updateQuantity: (state, action: PayloadAction<{ id: string; quantity: number }>) => {
      const item = state.items.find((i) => i.id === action.payload.id);
      if (item) item.quantity = action.payload.quantity;
    },
    clearCart: (state) => {
      state.items = [];
    },
  },
});

export const { addItem, removeItem, updateQuantity, clearCart } = cartSlice.actions;
export default cartSlice.reducer;

// store.ts
import { configureStore } from '@reduxjs/toolkit';
import cartReducer from './cartSlice';

export const store = configureStore({
  reducer: { cart: cartReducer },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// hooks.ts
import { useSelector, useDispatch } from 'react-redux';

export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
export const useAppDispatch: () => AppDispatch = useDispatch;

// CartComponent.tsx
function CartScreen() {
  const items = useAppSelector((state) => state.cart.items);
  const dispatch = useAppDispatch();

  return (
    <View>
      {items.map((item) => (
        <CartItem
          key={item.id}
          item={item}
          onRemove={() => dispatch(removeItem(item.id))}
          onUpdateQuantity={(qty) =>
            dispatch(updateQuantity({ id: item.id, quantity: qty }))
          }
        />
      ))}
      <Button title="Clear Cart" onPress={() => dispatch(clearCart())} />
    </View>
  );
}
```

**Zustand version (same feature):**

```tsx
// cartStore.ts
import { create } from 'zustand';

interface CartStore {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (itemId: string) => void;
  updateQuantity: (itemId: string, quantity: number) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>((set) => ({
  items: [],
  addItem: (item) => set((s) => {
    const existing = s.items.find((i) => i.id === item.id);
    if (existing) {
      return { items: s.items.map((i) => i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i) };
    }
    return { items: [...s.items, { ...item, quantity: 1 }] };
  }),
  removeItem: (itemId) => set((s) => ({
    items: s.items.filter((i) => i.id !== itemId),
  })),
  updateQuantity: (itemId, quantity) => set((s) => ({
    items: s.items.map((i) => i.id === itemId ? { ...i, quantity } : i),
  })),
  clearCart: () => set({ items: [] }),
}));

// CartComponent.tsx
function CartScreen() {
  const items = useCartStore((s) => s.items);
  const removeItem = useCartStore((s) => s.removeItem);
  const updateQuantity = useCartStore((s) => s.updateQuantity);
  const clearCart = useCartStore((s) => s.clearCart);

  return (
    <View>
      {items.map((item) => (
        <CartItem
          key={item.id}
          item={item}
          onRemove={() => removeItem(item.id)}
          onUpdateQuantity={(qty) => updateQuantity(item.id, qty)}
        />
      ))}
      <Button title="Clear Cart" onPress={clearCart} />
    </View>
  );
}
```

### 13.2 The Comparison

| Aspect | Redux Toolkit | Zustand |
|--------|--------------|---------|
| **Files for a cart feature** | 4 (slice, store, hooks, component) | 2 (store, component) |
| **Boilerplate** | createSlice, configureStore, Provider, typed hooks | create, done |
| **Bundle size** | ~11KB (RTK) + ~5KB (react-redux) | ~1KB |
| **Provider required** | Yes (wrap app in `<Provider>`) | No |
| **Access outside React** | `store.getState()`, `store.dispatch()` | `useStore.getState()` |
| **DevTools** | Built-in | Via middleware |
| **Immer** | Built-in (createSlice uses immer) | Via middleware |
| **Middleware ecosystem** | Vast (sagas, thunks, listeners) | Smaller but sufficient |
| **Server state** | RTK Query (built-in) | TanStack Query (separate) |
| **Learning curve** | Steeper (more concepts) | Minimal |

### 13.3 When to Choose Redux Toolkit

- Your team already knows Redux and has a large Redux codebase
- You want server state management (RTK Query) bundled with client state management in one package
- You need the Redux middleware ecosystem (sagas, listener middleware)
- Your company has a mandate for Redux (large enterprises often do)
- You value the Redux DevTools time-travel debugging as a primary workflow tool

### 13.4 When to Choose Zustand + TanStack Query

- Starting a new project with no legacy constraints
- You want minimal boilerplate and a tiny bundle
- You want the best-in-class tool for each category (TanStack Query for server state is more feature-rich than RTK Query)
- You're building a React Native app (Zustand's smaller size matters on mobile)
- Your team is small and values simplicity

---

## 14. UNDO/REDO WITH EVENT SOURCING

One of the most elegant patterns in state management: instead of storing the current state, store the *events* that produced it.

```tsx
type Event =
  | { type: 'ADD_ITEM'; item: Item }
  | { type: 'REMOVE_ITEM'; itemId: string }
  | { type: 'UPDATE_QUANTITY'; itemId: string; quantity: number };

function cartReducer(state: CartState, event: Event): CartState {
  switch (event.type) {
    case 'ADD_ITEM':
      return { ...state, items: [...state.items, event.item] };
    case 'REMOVE_ITEM':
      return { ...state, items: state.items.filter(i => i.id !== event.itemId) };
    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: state.items.map(i =>
          i.id === event.itemId ? { ...i, quantity: event.quantity } : i
        ),
      };
  }
}

// Undo: replay all events except the last one
function undo(events: Event[]): Event[] {
  return events.slice(0, -1);
}

// Redo: re-add the removed event
function redo(events: Event[], undoneEvents: Event[]): Event[] {
  const lastUndone = undoneEvents[undoneEvents.length - 1];
  return lastUndone ? [...events, lastUndone] : events;
}

// Current state = reduce all events from initial state
const currentState = events.reduce(cartReducer, initialState);
```

### Undo/Redo as a Zustand Store

```tsx
import { create } from 'zustand';

interface UndoableStore<TState, TEvent> {
  state: TState;
  events: TEvent[];
  undoneEvents: TEvent[];
  dispatch: (event: TEvent) => void;
  undo: () => void;
  redo: () => void;
  canUndo: boolean;
  canRedo: boolean;
}

function createUndoableStore<TState, TEvent>(
  reducer: (state: TState, event: TEvent) => TState,
  initialState: TState
) {
  return create<UndoableStore<TState, TEvent>>((set, get) => ({
    state: initialState,
    events: [],
    undoneEvents: [],
    canUndo: false,
    canRedo: false,

    dispatch: (event) => {
      const { events } = get();
      const newEvents = [...events, event];
      const newState = newEvents.reduce(reducer, initialState);
      set({
        state: newState,
        events: newEvents,
        undoneEvents: [], // Clear redo stack on new action
        canUndo: true,
        canRedo: false,
      });
    },

    undo: () => {
      const { events, undoneEvents } = get();
      if (events.length === 0) return;
      const lastEvent = events[events.length - 1];
      const newEvents = events.slice(0, -1);
      const newState = newEvents.reduce(reducer, initialState);
      set({
        state: newState,
        events: newEvents,
        undoneEvents: [...undoneEvents, lastEvent],
        canUndo: newEvents.length > 0,
        canRedo: true,
      });
    },

    redo: () => {
      const { events, undoneEvents } = get();
      if (undoneEvents.length === 0) return;
      const lastUndone = undoneEvents[undoneEvents.length - 1];
      const newEvents = [...events, lastUndone];
      const newState = newEvents.reduce(reducer, initialState);
      set({
        state: newState,
        events: newEvents,
        undoneEvents: undoneEvents.slice(0, -1),
        canUndo: true,
        canRedo: undoneEvents.length > 1,
      });
    },
  }));
}

// Usage
const useDocumentStore = createUndoableStore(documentReducer, initialDocument);
```

This pattern gives you undo, redo, audit trails, and debugging for free. Every state change is an event. Replay the events to reproduce any state. Perfect for editors, drawing apps, form builders, and any UI where users expect undo.

---

## 15. THE DECISION FRAMEWORK

Use this when starting a new project or deciding on a library:

| Scenario | Server State | Client State | Form State |
|----------|-------------|-------------|------------|
| **New Expo project** | TanStack Query | Zustand | React Hook Form + Zod |
| **Complex filter UI** | TanStack Query | Jotai | React Hook Form + Zod |
| **Offline-first app** | Legend State sync | Legend State | React Hook Form + Zod |
| **Legacy Redux codebase** | TanStack Query (new) + keep Redux (old) | Keep Redux | React Hook Form + Zod |
| **Enterprise, large team** | RTK Query | Redux Toolkit | React Hook Form + Zod |
| **Simple 2-3 screens** | fetch + useState | React Context | Controlled inputs |
| **URL-heavy filters/search** | TanStack Query + nuqs | nuqs (URL state) | React Hook Form + Zod |

### Migration from Redux

If you're on Redux and want to migrate, do it incrementally:

1. **Day 1:** Install TanStack Query. Use it for all *new* API calls. Don't touch existing Redux sagas/thunks.
2. **Week 2:** Install Zustand. Use it for all *new* client state. Don't touch existing Redux slices.
3. **Ongoing:** When you touch an existing screen for a feature change, migrate its Redux code to TQ/Zustand at the same time.
4. **Eventually:** Redux only remains for deeply integrated global state. Some teams keep it for years — that's fine.

**Never do a "big bang" migration.** You'll break things, miss edge cases, and demoralize your team. Incremental migration with the grain of your feature work is the only sane path.

I once worked with a team that tried a big-bang Redux-to-MobX migration. Three months in, they had a half-Redux, half-MobX codebase with adapters between them. The adapters had bugs. They ended up reverting the entire migration and starting over incrementally. The incremental approach took six months but shipped zero regressions.

### Migration from React Context

React Context is fine for 2-3 values that change infrequently (theme, locale). It's terrible for anything that changes often because it triggers re-renders in *every* component that reads the context, even if the specific value they need didn't change.

```tsx
// ❌ Context re-render problem
const AppContext = createContext<{
  user: User;
  theme: string;
  notifications: number;
}>({...});

// When notifications changes, EVERY component reading AppContext re-renders
// — even the ones that only care about theme.

// ✅ Fix: Use Zustand selectors
const useNotifications = () => useAppStore(s => s.notifications);
// Only components that read notifications re-render when it changes.
```

If you have more than one Context, or if your Context values change more than once per minute, migrate to Zustand. It's a 15-minute refactor for a single context:

```tsx
// Before: Context
const ThemeContext = createContext<{ theme: string; setTheme: (t: string) => void }>({
  theme: 'light',
  setTheme: () => {},
});

function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// After: Zustand (delete the Provider, delete the Context file)
const useThemeStore = create<{ theme: string; setTheme: (t: string) => void }>((set) => ({
  theme: 'light',
  setTheme: (theme) => set({ theme }),
}));

// Component changes:
// Before: const { theme } = useContext(ThemeContext);
// After:  const theme = useThemeStore((s) => s.theme);
```

No Provider wrapping. No Context creation. No re-render problems. Fifteen minutes.

---

## 16. PUTTING IT ALL TOGETHER

Here's the state architecture for a well-structured React Native app:

```
┌──────────────────────────────────────────────────────────────┐
│                    YOUR REACT NATIVE APP                      │
│                                                               │
│  ┌─────────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  TanStack Query  │  │   Zustand    │  │  React Hook     │ │
│  │  (Server State)  │  │(Client State)│  │  Form + Zod     │ │
│  │                  │  │              │  │  (Form State)    │ │
│  │  - User profile  │  │  - Theme     │  │  - Login         │ │
│  │  - Product list  │  │  - Auth      │  │  - Checkout      │ │
│  │  - Notifications │  │  - Sidebar   │  │  - Search        │ │
│  │  - Feed data     │  │  - Filters   │  │  - Settings      │ │
│  │  - Reviews       │  │  - Cart      │  │  - Profile edit  │ │
│  └────────┬─────────┘  └──────┬───────┘  └──────┬──────────┘ │
│           │                    │                  │            │
│  ┌────────▼────────────────────▼──────────────────▼──────────┐│
│  │                         MMKV                              ││
│  │              (Persistent storage, 30x faster              ││
│  │               than AsyncStorage)                          ││
│  └───────────────────────────────────────────────────────────┘│
│                                                               │
│  ┌──────────────────┐  ┌──────────────────┐                  │
│  │   URL State       │  │  XState Store    │                  │
│  │   (nuqs)          │  │  (Complex flows) │                  │
│  │                   │  │                  │                  │
│  │  - Search params  │  │  - Checkout flow │                  │
│  │  - Filters        │  │  - Onboarding    │                  │
│  │  - Pagination     │  │  - Multi-step    │                  │
│  │  - Sort order     │  │    wizards       │                  │
│  └──────────────────┘  └──────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

**Three core tools, three categories, zero overlap.** Plus two specialized tools for edge cases (URL state and state machines). Each tool does what it's best at. No tool tries to do everything. The complexity of your state management scales linearly with the complexity of your features — not exponentially with the number of things your state manager tries to handle.

### Quick Reference: "Where Does This State Go?"

| State | Category | Tool | Persistent? |
|-------|----------|------|-------------|
| API responses | Server | TanStack Query | Cache (gcTime) |
| User authentication | Client | Zustand + MMKV | Yes |
| Theme preference | Client | Zustand + MMKV | Yes |
| Shopping cart | Client | Zustand + MMKV | Yes |
| Sidebar open/closed | Client | Zustand | No |
| Selected filter | URL or Client | nuqs or Zustand | Via URL |
| Current page number | URL | nuqs | Via URL |
| Form field values | Form | React Hook Form | No |
| Form validation errors | Form | React Hook Form + Zod | No |
| Checkout flow step | Client/Machine | XState Store | Optional |
| Optimistic update | Server | TanStack Query (onMutate) | No |
| Offline queue | Client | Legend State | Yes |

This is the state architecture that scales from a weekend project to a million-user app. The tools might change (Jotai instead of Zustand, Legend State for offline), but the three-category separation doesn't. Get this right, and state management stops being a problem and becomes an asset.

---

*Next: [Chapter 10 — Data Fetching & Server Communication](./10-data-fetching.md) — deep patterns for TanStack Query, tRPC, and real-time communication.*
