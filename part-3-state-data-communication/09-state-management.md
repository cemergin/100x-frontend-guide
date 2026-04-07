<!--
  CHAPTER: 10
  TITLE: State Management at Scale
  PART: III — State, Data & Communication
  PREREQS: Chapter 1, 3
  KEY_TOPICS: Zustand, Jotai, Legend State, TanStack Query, Redux, React Hook Form, Zod, MMKV, finite states, type states, XState Store, anti-patterns, three-category model
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
- Server State: TanStack Query
- Client State: Zustand (and when Jotai or Legend State)
- Form State: React Hook Form + Zod
- Finite States and Type States
- External Store Patterns (XState Store, useSyncExternalStore)
- Data Normalization
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

> **The Redux Trap:** Redux became the de facto state manager in React because it was there first and it could do everything. And that was the problem — it *did* everything. Server state, client state, form state, all lived in the same store. A Saga that fetched user data lived next to a reducer that toggled a modal. When the store updated, all connected components re-evaluated their selectors. Teams built enormous Redux stores with hundreds of actions and reducers, and the abstraction that was supposed to simplify state management became the largest source of complexity in the application.

---

## 2. REACT ANTI-PATTERNS THAT CREATED THIS MESS

Before we get to solutions, let's name the specific anti-patterns that make state management painful. You'll recognize most of these from codebases you've worked in.

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

**Fix:** Use a reducer to centralize the logic, or use TanStack Query with `enabled` flags to express the dependency chain declaratively.

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

This is what finite states solve. More on this in Section 6.

---

## 3. SERVER STATE: TANSTACK QUERY

TanStack Query (formerly React Query) is the standard for server state in 2026. It's not just a fetching library — it's a **server state synchronization engine** with caching, background refetching, optimistic updates, pagination, infinite queries, and garbage collection built in.

### Why Not Just fetch + useState?

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

### Core Concepts

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

### Query Keys: Your Cache Addressing Scheme

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

### staleTime vs gcTime

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

---

## 4. CLIENT STATE: ZUSTAND

Zustand is the default client state manager in 2026. At ~1KB, with zero boilerplate, and React Native support out of the box, it's what you reach for when you need global UI state.

### Why Zustand Won

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

### Zustand with MMKV Persistence

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

### When to Use Jotai Instead

Jotai (~2.5KB) uses an **atomic** model — small, independent pieces of state that can depend on each other. It shines when you have many interdependent values:

```tsx
import { atom, useAtom } from 'jotai';

// Base atoms
const filtersAtom = atom({ category: 'all', priceRange: [0, 100] as [number, number] });
const sortAtom = atom<'price' | 'rating' | 'newest'>('newest');

// Derived atom — recomputes when dependencies change
const filteredProductsAtom = atom(async (get) => {
  const filters = get(filtersAtom);
  const sort = get(sortAtom);
  return api.getProducts({ ...filters, sort });
});

// Only the component reading filteredProducts re-renders when filters change
```

**Pick Jotai when:** You have complex filter UIs where multiple pieces of state depend on each other, or when you want React-style atomic composition rather than a centralized store.

**Pick Zustand when:** You want a simple, centralized store for global app state. Most apps. This is the default.

### When to Use Legend State

Legend State is purpose-built for **offline-first** React Native apps. It integrates MMKV persistence and server sync directly into the state layer:

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

**Pick Legend State when:** You're building an app for unreliable connectivity (field work, travel, rural areas) and need offline-first with automatic sync.

---

## 5. FORM STATE: REACT HOOK FORM + ZOD

Form state is the third category, and it's the one most teams get wrong by putting it in global state.

Forms are ephemeral. They exist while the user is filling them out and disappear when submitted. Putting form values in Redux or Zustand means:
- Global state updates on every keystroke
- Re-renders propagate beyond the form
- Form cleanup requires explicit actions
- Form state outlives the form (stale data on re-mount)

React Hook Form avoids all of this by storing form state in refs, not React state. Keystrokes don't trigger re-renders. Only validation and submission do.

```tsx
import { useForm } from 'react-hook-form';
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

---

## 6. FINITE STATES AND TYPE STATES

Remember the `useState` explosion from Section 2.4? Seven boolean flags with 128 possible combinations, only 4 of which are valid? Here's the fix.

### Finite States: Replace Booleans with Enums

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

### Type States: Discriminated Unions

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

---

## 7. THE DECISION FRAMEWORK

Use this when starting a new project or deciding on a library:

| Scenario | Server State | Client State | Form State |
|----------|-------------|-------------|------------|
| **New Expo project** | TanStack Query | Zustand | React Hook Form + Zod |
| **Complex filter UI** | TanStack Query | Jotai | React Hook Form + Zod |
| **Offline-first app** | Legend State sync | Legend State | React Hook Form + Zod |
| **Legacy Redux codebase** | TanStack Query (new) + keep Redux (old) | Keep Redux | React Hook Form + Zod |
| **Enterprise, large team** | RTK Query | Redux Toolkit | React Hook Form + Zod |
| **Simple 2-3 screens** | fetch + useState | React Context | Controlled inputs |

### Migration from Redux

If you're on Redux and want to migrate, do it incrementally:

1. **Day 1:** Install TanStack Query. Use it for all *new* API calls. Don't touch existing Redux sagas/thunks.
2. **Week 2:** Install Zustand. Use it for all *new* client state. Don't touch existing Redux slices.
3. **Ongoing:** When you touch an existing screen for a feature change, migrate its Redux code to TQ/Zustand at the same time.
4. **Eventually:** Redux only remains for deeply integrated global state. Some teams keep it for years — that's fine.

**Never do a "big bang" migration.** You'll break things, miss edge cases, and demoralize your team. Incremental migration with the grain of your feature work is the only sane path.

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

If you have more than one Context, or if your Context values change more than once per minute, migrate to Zustand. It's a 15-minute refactor for a single context.

---

## 8. DATA NORMALIZATION

When your server returns nested data, storing it as-is creates problems:

```tsx
// Server returns:
{
  "id": "post-1",
  "title": "Hello World",
  "author": { "id": "user-1", "name": "Alice" },
  "comments": [
    { "id": "comment-1", "text": "Great post!", "author": { "id": "user-2", "name": "Bob" } },
    { "id": "comment-2", "text": "Thanks!", "author": { "id": "user-1", "name": "Alice" } }
  ]
}
// Alice appears twice. Update her name in one place, the other is stale.
```

**Normalize:** Store entities by ID, reference by ID:

```tsx
// Normalized:
{
  users: {
    'user-1': { id: 'user-1', name: 'Alice' },
    'user-2': { id: 'user-2', name: 'Bob' },
  },
  posts: {
    'post-1': { id: 'post-1', title: 'Hello World', authorId: 'user-1', commentIds: ['comment-1', 'comment-2'] },
  },
  comments: {
    'comment-1': { id: 'comment-1', text: 'Great post!', authorId: 'user-2' },
    'comment-2': { id: 'comment-2', text: 'Thanks!', authorId: 'user-1' },
  }
}
```

Now Alice's name lives in exactly one place. Update it once, every reference is current.

**When to normalize:** When the same entity appears in multiple places, or when you need to update an entity and have the change reflected everywhere. Common for social apps, comment systems, e-commerce (products appear in lists, detail views, carts, wishlists).

**When NOT to normalize:** When data is read-only and doesn't appear in multiple contexts. TanStack Query's cache handles simple cases fine without manual normalization.

---

## 9. UNDO/REDO WITH EVENT SOURCING

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

This pattern gives you undo, redo, audit trails, and debugging for free. Every state change is an event. Replay the events to reproduce any state. Perfect for editors, drawing apps, form builders, and any UI where users expect undo.

---

## 10. PUTTING IT ALL TOGETHER

Here's the state architecture for a well-structured React Native app:

```
┌──────────────────────────────────────────────────────────┐
│                    YOUR REACT NATIVE APP                  │
│                                                           │
│  ┌─────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  TanStack Query  │  │   Zustand    │  │  React Hook │ │
│  │  (Server State)  │  │(Client State)│  │  Form + Zod │ │
│  │                  │  │              │  │ (Form State) │ │
│  │  - User profile  │  │  - Theme     │  │  - Login     │ │
│  │  - Product list  │  │  - Auth      │  │  - Checkout  │ │
│  │  - Notifications │  │  - Sidebar   │  │  - Search    │ │
│  │  - Feed data     │  │  - Filters   │  │  - Settings  │ │
│  └────────┬─────────┘  └──────┬───────┘  └──────┬──────┘ │
│           │                    │                  │        │
│  ┌────────▼────────────────────▼──────────────────▼──────┐│
│  │                      MMKV                             ││
│  │           (Persistent storage, 30x faster             ││
│  │            than AsyncStorage)                          ││
│  └───────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

**Three tools, three categories, zero overlap.** Each tool does what it's best at. No tool tries to do everything. The complexity of your state management scales linearly with the complexity of your features — not exponentially with the number of things your state manager tries to handle.

This is the state architecture that scales from a weekend project to a million-user app. The tools might change (Jotai instead of Zustand, Legend State for offline), but the three-category separation doesn't. Get this right, and state management stops being a problem and becomes an asset.

---

*Next: [Chapter 10 — Data Fetching & Server Communication](./10-data-fetching.md) — deep patterns for TanStack Query, tRPC, and real-time communication.*
