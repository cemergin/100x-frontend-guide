<!--
  CHAPTER: 10
  TITLE: Data Fetching & Server Communication
  PART: III — State, Data & Communication
  PREREQS: Chapter 9
  KEY_TOPICS: TanStack Query, tRPC, REST, GraphQL, WebSockets, SSE, API layer, retry, backoff, pagination, infinite scroll, optimistic updates, offline
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 10: Data Fetching & Server Communication

> **Part III — State, Data & Communication** | Prerequisites: Chapter 9 | Difficulty: Intermediate to Advanced

Chapter 9 taught you that server state is a separate category that deserves its own tool. This chapter is the deep dive into everything that tool touches: how your frontend talks to your backend, how it handles the messy reality of networks, and how you build an API layer that stays maintainable at scale.

Here is the uncomfortable truth most tutorials skip: **the fetch call is the easiest part.** The hard parts are token refresh races, retrying without hammering a struggling server, showing optimistic UI that rolls back gracefully, keeping a mobile app functional on a subway, and doing all of this without turning your codebase into a spaghetti of interceptors and retry loops.

This chapter covers all of it. We will build a typed API layer from scratch, master TanStack Query's advanced patterns, wire up tRPC for end-to-end type safety, choose between REST/GraphQL/tRPC with clear reasoning, implement real-time communication, and handle the offline world. By the end, you will have a complete mental model for how data moves through a modern frontend application.

### In This Chapter
- API Layer Architecture (typed clients, interceptors, error standardization)
- TanStack Query Advanced Patterns (query factories, dependent queries, prefetching, infinite queries)
- Optimistic Updates with Rollback
- tRPC End-to-End (Next.js + React Native in a monorepo)
- REST vs GraphQL vs tRPC Decision Matrix
- Real-Time Communication (WebSockets, SSE, Socket.io)
- Pagination Patterns (cursor-based vs offset-based)
- Retry and Backoff Strategies
- Offline Patterns (mutation queues, persistence, replay)

### Related Chapters
- [Ch 9: State Management at Scale] -- the three-category model and server state foundations
- [Ch 11: Caching Strategies] -- how caching intersects with data fetching
- [Ch 12: Offline-First & Real-Time] -- Legend State sync and deeper offline patterns
- [Ch 13: Performance Optimization] -- network performance, bundle size, render cost

---

## 1. API LAYER ARCHITECTURE

Every frontend codebase that grows past a few screens needs an API layer -- a single place where all HTTP communication is configured, typed, and instrumented. Without it, you end up with `fetch` calls scattered across components, each one handling auth headers slightly differently, each one parsing errors in its own way, and each one a liability when the backend team changes the error format.

### 1.1 The Typed Fetch Wrapper

Let's build one from scratch. No libraries. Just `fetch` with types, error handling, and the extension points you will actually need.

```typescript
// src/api/client.ts

// --- Error Types ---

interface ApiErrorResponse {
  code: string;
  message: string;
  details?: Record<string, unknown>;
}

export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    public details?: Record<string, unknown>,
  ) {
    super(`API Error [${status}] ${code}`);
    this.name = 'ApiError';
  }
}

export class NetworkError extends Error {
  constructor(message: string = 'Network request failed') {
    super(message);
    this.name = 'NetworkError';
  }
}

// --- Config ---

type Environment = 'development' | 'staging' | 'production';

const BASE_URLS: Record<Environment, string> = {
  development: 'http://localhost:3001/api',
  staging: 'https://staging-api.yourapp.com',
  production: 'https://api.yourapp.com',
};

function getBaseUrl(): string {
  const env = (process.env.NODE_ENV ?? 'development') as Environment;
  // Allow override for testing or custom environments
  return process.env.NEXT_PUBLIC_API_URL ?? BASE_URLS[env] ?? BASE_URLS.development;
}

// --- Token Management ---

type TokenGetter = () => Promise<string | null>;
type TokenRefresher = () => Promise<string | null>;

let getToken: TokenGetter = async () => null;
let refreshToken: TokenRefresher = async () => null;

export function configureAuth(getter: TokenGetter, refresher: TokenRefresher) {
  getToken = getter;
  refreshToken = refresher;
}

// --- The Client ---

interface RequestConfig extends Omit<RequestInit, 'body'> {
  body?: unknown;
  params?: Record<string, string | number | boolean | undefined>;
  skipAuth?: boolean;
  timeout?: number;
}

async function request<T>(
  method: string,
  path: string,
  config: RequestConfig = {},
): Promise<T> {
  const { body, params, skipAuth = false, timeout = 30_000, ...init } = config;

  // Build URL with query params
  const url = new URL(path, getBaseUrl());
  if (params) {
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined) {
        url.searchParams.set(key, String(value));
      }
    });
  }

  // Build headers
  const headers = new Headers(init.headers);
  if (body && !(body instanceof FormData)) {
    headers.set('Content-Type', 'application/json');
  }

  if (!skipAuth) {
    const token = await getToken();
    if (token) {
      headers.set('Authorization', `Bearer ${token}`);
    }
  }

  // Timeout via AbortController
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url.toString(), {
      method,
      headers,
      body: body instanceof FormData ? body : body ? JSON.stringify(body) : undefined,
      signal: controller.signal,
      ...init,
    });

    clearTimeout(timeoutId);

    // Handle 401 with token refresh (one retry)
    if (response.status === 401 && !skipAuth) {
      const newToken = await refreshToken();
      if (newToken) {
        headers.set('Authorization', `Bearer ${newToken}`);
        const retryResponse = await fetch(url.toString(), {
          method,
          headers,
          body: body instanceof FormData ? body : body ? JSON.stringify(body) : undefined,
          ...init,
        });
        if (retryResponse.ok) {
          return retryResponse.status === 204 ? (undefined as T) : await retryResponse.json();
        }
        // If retry also fails, fall through to error handling below
        return handleErrorResponse<T>(retryResponse);
      }
    }

    if (!response.ok) {
      return handleErrorResponse<T>(response);
    }

    // 204 No Content
    if (response.status === 204) {
      return undefined as T;
    }

    return await response.json();
  } catch (error) {
    clearTimeout(timeoutId);
    if (error instanceof ApiError) throw error;
    if (error instanceof DOMException && error.name === 'AbortError') {
      throw new NetworkError('Request timed out');
    }
    throw new NetworkError(
      error instanceof Error ? error.message : 'Unknown network error',
    );
  }
}

async function handleErrorResponse<T>(response: Response): Promise<T> {
  let errorBody: ApiErrorResponse;
  try {
    errorBody = await response.json();
  } catch {
    errorBody = { code: 'UNKNOWN_ERROR', message: response.statusText };
  }
  throw new ApiError(response.status, errorBody.code, errorBody.details);
}

// --- Public API ---

export const api = {
  get: <T>(path: string, config?: RequestConfig) =>
    request<T>('GET', path, config),

  post: <T>(path: string, body?: unknown, config?: RequestConfig) =>
    request<T>('POST', path, { ...config, body }),

  put: <T>(path: string, body?: unknown, config?: RequestConfig) =>
    request<T>('PUT', path, { ...config, body }),

  patch: <T>(path: string, body?: unknown, config?: RequestConfig) =>
    request<T>('PATCH', path, { ...config, body }),

  delete: <T>(path: string, config?: RequestConfig) =>
    request<T>('DELETE', path, config),
};
```

**Why this structure matters:**

1. **Single point of configuration.** Every request goes through `request()`. Change the auth header format once, it changes everywhere.
2. **Typed responses.** `api.get<User>('/users/me')` returns `Promise<User>`. Your IDE autocompletes the response shape.
3. **Error standardization.** Every failure becomes either an `ApiError` (server responded with an error) or a `NetworkError` (couldn't reach the server). Consumers don't need to parse HTTP status codes -- they catch typed errors.
4. **Token refresh built in.** The 401 retry happens transparently. Components never know it happened.

### 1.2 Usage in Domain Modules

Don't call `api.get` directly from components. Build domain-specific modules:

```typescript
// src/api/users.ts
import { api } from './client';

export interface User {
  id: string;
  email: string;
  name: string;
  avatarUrl: string | null;
  role: 'admin' | 'member' | 'viewer';
  createdAt: string;
}

export interface UpdateUserPayload {
  name?: string;
  avatarUrl?: string | null;
}

export const usersApi = {
  me: () => api.get<User>('/users/me'),

  getById: (id: string) => api.get<User>(`/users/${id}`),

  update: (id: string, payload: UpdateUserPayload) =>
    api.patch<User>(`/users/${id}`, payload),

  list: (params?: { role?: User['role']; page?: number; limit?: number }) =>
    api.get<{ users: User[]; total: number }>('/users', { params }),

  uploadAvatar: (id: string, file: File) => {
    const form = new FormData();
    form.append('avatar', file);
    return api.post<{ avatarUrl: string }>(`/users/${id}/avatar`, form);
  },
};
```

```typescript
// src/api/posts.ts
import { api } from './client';

export interface Post {
  id: string;
  title: string;
  body: string;
  authorId: string;
  status: 'draft' | 'published' | 'archived';
  createdAt: string;
  updatedAt: string;
}

export interface CreatePostPayload {
  title: string;
  body: string;
  status?: Post['status'];
}

export const postsApi = {
  list: (params?: { authorId?: string; status?: Post['status']; cursor?: string; limit?: number }) =>
    api.get<{ posts: Post[]; nextCursor: string | null }>('/posts', { params }),

  getById: (id: string) => api.get<Post>(`/posts/${id}`),

  create: (payload: CreatePostPayload) =>
    api.post<Post>('/posts', payload),

  update: (id: string, payload: Partial<CreatePostPayload>) =>
    api.patch<Post>(`/posts/${id}`, payload),

  delete: (id: string) => api.delete<void>(`/posts/${id}`),
};
```

This gives you a clean separation: the API client handles HTTP concerns (auth, errors, timeouts), domain modules handle shapes and endpoints, and components/hooks handle when to fetch and what to do with the data.

### 1.3 The Axios Alternative

Some teams prefer Axios for its interceptor system. Here's the equivalent setup:

```typescript
// src/api/axios-client.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';

const client = axios.create({
  baseURL: getBaseUrl(),
  timeout: 30_000,
  headers: { 'Content-Type': 'application/json' },
});

// --- Request Interceptor: Attach Token ---
client.interceptors.request.use(async (config) => {
  const token = await getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// --- Response Interceptor: Token Refresh ---
let isRefreshing = false;
let failedQueue: Array<{
  resolve: (token: string) => void;
  reject: (error: unknown) => void;
}> = [];

function processQueue(error: unknown, token: string | null) {
  failedQueue.forEach(({ resolve, reject }) => {
    if (error) reject(error);
    else resolve(token!);
  });
  failedQueue = [];
}

client.interceptors.response.use(
  (response) => response,
  async (error: AxiosError<ApiErrorResponse>) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & { _retry?: boolean };

    // Token refresh on 401
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue this request until refresh completes
        return new Promise((resolve, reject) => {
          failedQueue.push({
            resolve: (token: string) => {
              originalRequest.headers.Authorization = `Bearer ${token}`;
              resolve(client(originalRequest));
            },
            reject,
          });
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await refreshToken();
        if (newToken) {
          processQueue(null, newToken);
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
          return client(originalRequest);
        }
        processQueue(new Error('Refresh failed'), null);
        // Redirect to login or dispatch logout
        throw new ApiError(401, 'SESSION_EXPIRED');
      } catch (refreshError) {
        processQueue(refreshError, null);
        throw refreshError;
      } finally {
        isRefreshing = false;
      }
    }

    // Standardize errors
    if (error.response) {
      const data = error.response.data;
      throw new ApiError(
        error.response.status,
        data?.code ?? 'UNKNOWN_ERROR',
        data?.details,
      );
    }
    throw new NetworkError(error.message);
  },
);

export { client as axiosClient };
```

**The key difference:** The Axios interceptor has a request queue for concurrent 401s. If three requests fail with 401 simultaneously, only one refresh call is made, and the other two wait for it to complete. The plain `fetch` version above doesn't handle this -- which is fine for most apps, but becomes a bug in apps that fire many parallel requests.

### 1.4 fetch vs Axios: The Real Trade-offs

| Concern | `fetch` (native) | Axios |
|---------|-------------------|-------|
| Bundle size | 0 KB (built-in) | ~14 KB minified |
| Interceptors | Manual (wrap function) | Built-in chain |
| Request cancellation | AbortController | AbortController (or CancelToken, deprecated) |
| Automatic JSON parsing | Manual (`response.json()`) | Automatic |
| Error on 4xx/5xx | No (must check `response.ok`) | Yes (throws) |
| Request/response transforms | Manual | Built-in |
| Upload progress | No | Yes |
| React Native | Works | Works |
| Node.js/SSR | Native since Node 18 | Works everywhere |

**My recommendation:** Use `fetch` for new projects. The bundle size argument matters in React Native. The interceptor gap is closed by the wrapper pattern above. If you need upload progress tracking or you're maintaining a codebase that already uses Axios, keep Axios -- it's fine. Don't migrate just because Twitter told you to.

### 1.5 Error Handling Strategy

Every API error in your app should be catchable and actionable. Here's a utility layer that makes error handling consistent:

```typescript
// src/api/errors.ts

export function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}

export function isNetworkError(error: unknown): error is NetworkError {
  return error instanceof NetworkError;
}

export function isNotFound(error: unknown): boolean {
  return isApiError(error) && error.status === 404;
}

export function isUnauthorized(error: unknown): boolean {
  return isApiError(error) && error.status === 401;
}

export function isForbidden(error: unknown): boolean {
  return isApiError(error) && error.status === 403;
}

export function isValidationError(error: unknown): boolean {
  return isApiError(error) && error.status === 422;
}

export function getValidationErrors(error: unknown): Record<string, string[]> {
  if (isApiError(error) && error.status === 422 && error.details) {
    return error.details as Record<string, string[]>;
  }
  return {};
}

export function getUserFacingMessage(error: unknown): string {
  if (isNetworkError(error)) {
    return 'Unable to connect. Check your internet connection and try again.';
  }
  if (isApiError(error)) {
    switch (error.status) {
      case 401: return 'Your session has expired. Please sign in again.';
      case 403: return 'You don\'t have permission to do that.';
      case 404: return 'The requested resource was not found.';
      case 422: return 'Please check your input and try again.';
      case 429: return 'Too many requests. Please wait a moment.';
      case 500:
      case 502:
      case 503: return 'Something went wrong on our end. Please try again later.';
      default: return error.message;
    }
  }
  return 'An unexpected error occurred.';
}
```

This keeps your components clean. Instead of checking HTTP status codes in a catch block, you call `getUserFacingMessage(error)` and show the result. When the backend team changes an error message, you update one function.

---

## 2. TANSTACK QUERY ADVANCED PATTERNS

Chapter 9 introduced TanStack Query as the server state tool. Here we go deep -- the patterns that separate "I've used React Query" from "I've built production apps with it."

### 2.1 Query Key Factories

Query keys are the foundation of TanStack Query's caching and invalidation system. When you scatter string keys across your codebase, you inevitably get typos, stale invalidation, and cache misses. Query key factories fix this.

```typescript
// src/queries/keys.ts

export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: { role?: string; page?: number }) =>
    [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
  me: () => [...userKeys.all, 'me'] as const,
};

export const postKeys = {
  all: ['posts'] as const,
  lists: () => [...postKeys.all, 'list'] as const,
  list: (filters: { authorId?: string; status?: string; cursor?: string }) =>
    [...postKeys.lists(), filters] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail: (id: string) => [...postKeys.details(), id] as const,
  comments: (postId: string) => [...postKeys.detail(postId), 'comments'] as const,
};

export const notificationKeys = {
  all: ['notifications'] as const,
  unreadCount: () => [...notificationKeys.all, 'unread-count'] as const,
  list: (cursor?: string) => [...notificationKeys.all, 'list', cursor] as const,
};
```

**Why this hierarchy matters:** TanStack Query can invalidate by prefix. When you call `queryClient.invalidateQueries({ queryKey: userKeys.all })`, it invalidates every user query -- lists, details, the current user, all of it. When you call `queryClient.invalidateQueries({ queryKey: userKeys.lists() })`, it invalidates all user lists but leaves individual user details cached. This granularity is critical for performance.

```typescript
// After creating a new user:
queryClient.invalidateQueries({ queryKey: userKeys.lists() });
// All user lists refetch. User detail caches stay warm.

// After updating a specific user:
queryClient.invalidateQueries({ queryKey: userKeys.detail(userId) });
// Only that user's detail refetches.

// Nuclear option after a role change that affects everything:
queryClient.invalidateQueries({ queryKey: userKeys.all });
// Every user-related query refetches.
```

### 2.2 Query Options Factories

TanStack Query v5 introduced `queryOptions()`, a helper that lets you define query configuration in one place and reuse it everywhere. This is the natural extension of key factories.

```typescript
// src/queries/users.ts
import { queryOptions, infiniteQueryOptions } from '@tanstack/react-query';
import { usersApi, type User } from '../api/users';
import { userKeys } from './keys';

export function userListOptions(filters: { role?: string; page?: number } = {}) {
  return queryOptions({
    queryKey: userKeys.list(filters),
    queryFn: () => usersApi.list(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
}

export function userDetailOptions(id: string) {
  return queryOptions({
    queryKey: userKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    staleTime: 10 * 60 * 1000, // 10 minutes
    enabled: !!id, // Don't fetch with empty ID
  });
}

export function currentUserOptions() {
  return queryOptions({
    queryKey: userKeys.me(),
    queryFn: () => usersApi.me(),
    staleTime: 30 * 60 * 1000, // 30 minutes -- current user doesn't change often
    gcTime: Infinity, // Never garbage collect the current user
  });
}
```

Now hooks, prefetching, and direct cache reads all use the same source of truth:

```typescript
// In a component
const { data: user } = useQuery(userDetailOptions(userId));

// In a route loader (prefetching)
queryClient.prefetchQuery(userDetailOptions(userId));

// Reading from cache directly
const cachedUser = queryClient.getQueryData(userDetailOptions(userId).queryKey);

// In a test
const options = userDetailOptions('user-123');
expect(options.queryKey).toEqual(['users', 'detail', 'user-123']);
```

### 2.3 Dependent Queries

Sometimes query B depends on the result of query A. The classic example: fetch the current user, then fetch their organization.

```typescript
function useCurrentUserOrganization() {
  // Step 1: Fetch current user
  const { data: user } = useQuery(currentUserOptions());

  // Step 2: Fetch their org -- only runs when user.orgId is available
  const orgQuery = useQuery({
    queryKey: ['organizations', user?.orgId],
    queryFn: () => orgsApi.getById(user!.orgId),
    enabled: !!user?.orgId, // This is the key -- query won't run until user is loaded
  });

  return {
    user,
    organization: orgQuery.data,
    isLoading: !user || orgQuery.isLoading,
  };
}
```

The `enabled` flag is the mechanism. When it's `false`, the query sits in a `pending` state without firing. When the user loads and `user.orgId` becomes truthy, TanStack Query automatically triggers the organization fetch.

**Common mistake:** Don't use `useEffect` to trigger the second query. That creates an extra render cycle and fights the declarative model.

```typescript
// ❌ BAD: Imperative dependent fetching
function useCurrentUserOrganization() {
  const { data: user } = useQuery(currentUserOptions());
  const [org, setOrg] = useState<Org | null>(null);

  useEffect(() => {
    if (user?.orgId) {
      orgsApi.getById(user.orgId).then(setOrg);
    }
  }, [user?.orgId]);

  return { user, organization: org };
  // Lost: caching, deduplication, background refetch, loading states, error handling
}
```

### 2.4 Parallel Queries

When you need multiple independent pieces of data, fire them all at once:

```typescript
function useDashboardData(userId: string) {
  const userQuery = useQuery(userDetailOptions(userId));
  const postsQuery = useQuery(postListOptions({ authorId: userId }));
  const statsQuery = useQuery({
    queryKey: ['stats', userId],
    queryFn: () => statsApi.getForUser(userId),
  });

  return {
    user: userQuery.data,
    posts: postsQuery.data,
    stats: statsQuery.data,
    isLoading: userQuery.isLoading || postsQuery.isLoading || statsQuery.isLoading,
    errors: [userQuery.error, postsQuery.error, statsQuery.error].filter(Boolean),
  };
}
```

All three queries fire simultaneously. If the user data is already cached, it returns instantly while posts and stats fetch in parallel. No waterfall.

For dynamic lists of parallel queries, use `useQueries`:

```typescript
function useMultipleUsers(userIds: string[]) {
  const queries = useQueries({
    queries: userIds.map((id) => userDetailOptions(id)),
  });

  return {
    users: queries.map((q) => q.data).filter(Boolean),
    isLoading: queries.some((q) => q.isLoading),
    errors: queries.map((q) => q.error).filter(Boolean),
  };
}
```

### 2.5 Prefetching for Navigation

One of the biggest wins with TanStack Query is prefetching data before the user navigates. The page loads instantly because the data is already in cache.

```typescript
// Prefetch on hover (web)
function PostListItem({ post }: { post: Post }) {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    // Start fetching the post detail when user hovers the link
    queryClient.prefetchQuery(postDetailOptions(post.id));
  };

  return (
    <Link
      href={`/posts/${post.id}`}
      onMouseEnter={handleMouseEnter}
    >
      <h3>{post.title}</h3>
      <p>{post.body.slice(0, 100)}...</p>
    </Link>
  );
}
```

```typescript
// Prefetch in route loaders (React Router / Expo Router)
// React Router v7 example:
export function loader({ params }: LoaderFunctionArgs) {
  const { postId } = params;
  // Don't await -- just start the fetch. The component will useQuery
  // and either get the cached data or wait for this fetch to complete.
  queryClient.prefetchQuery(postDetailOptions(postId!));
  return null;
}

// Expo Router with layout-level prefetching
function FeedLayout() {
  const queryClient = useQueryClient();
  const segments = useSegments();

  useEffect(() => {
    // When user enters the feed section, prefetch the first page
    queryClient.prefetchInfiniteQuery(feedInfiniteOptions());
  }, []);

  return <Slot />;
}
```

**The timing trick:** On mobile, prefetch on screen focus rather than hover (there is no hover). Use React Navigation's `useFocusEffect` or Expo Router's layout effects.

```typescript
// React Native: Prefetch when tab becomes active
import { useFocusEffect } from '@react-navigation/native';

function ProfileTab() {
  const queryClient = useQueryClient();

  useFocusEffect(
    useCallback(() => {
      queryClient.prefetchQuery(currentUserOptions());
      queryClient.prefetchQuery(userActivityOptions());
    }, [queryClient]),
  );

  return <ProfileScreen />;
}
```

### 2.6 Infinite Queries for Feeds

The most common mobile pattern: a scrollable list that loads more items as you scroll. TanStack Query's `useInfiniteQuery` handles this elegantly.

```typescript
// src/queries/posts.ts
import { infiniteQueryOptions } from '@tanstack/react-query';
import { postsApi } from '../api/posts';
import { postKeys } from './keys';

export function feedInfiniteOptions(authorId?: string) {
  return infiniteQueryOptions({
    queryKey: postKeys.list({ authorId, status: 'published' }),
    queryFn: ({ pageParam }) =>
      postsApi.list({
        authorId,
        status: 'published',
        cursor: pageParam,
        limit: 20,
      }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 2 * 60 * 1000,
  });
}
```

```tsx
// src/screens/FeedScreen.tsx
import { useInfiniteQuery } from '@tanstack/react-query';
import { FlatList, ActivityIndicator, RefreshControl } from 'react-native';
import { feedInfiniteOptions } from '../queries/posts';

function FeedScreen() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    isRefetching,
    refetch,
  } = useInfiniteQuery(feedInfiniteOptions());

  // Flatten all pages into a single array
  const posts = data?.pages.flatMap((page) => page.posts) ?? [];

  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <PostCard post={item} />}
      // Pull to refresh
      refreshControl={
        <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
      }
      // Infinite scroll
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      }}
      onEndReachedThreshold={0.5}
      // Loading indicators
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator style={{ padding: 16 }} /> : null
      }
      ListEmptyComponent={
        isLoading ? <FeedSkeleton /> : <EmptyFeed />
      }
    />
  );
}
```

**The `onEndReachedThreshold` matters.** A value of `0.5` means "start fetching when the user is halfway through the last visible screen of content." Too low (like `0.1`) and users see a loading spinner. Too high (like `1.0`) and you fetch pages the user might never scroll to.

### 2.7 Paginated Queries (Page-Based)

For traditional pagination (page 1, 2, 3...) where you show page numbers:

```typescript
function usePaginatedUsers(page: number, role?: string) {
  return useQuery({
    queryKey: userKeys.list({ role, page }),
    queryFn: () => usersApi.list({ role, page, limit: 20 }),
    placeholderData: keepPreviousData, // Keep showing old page while new one loads
  });
}
```

```tsx
function UserListPage() {
  const [page, setPage] = useState(1);
  const [role, setRole] = useState<string | undefined>();

  const { data, isLoading, isPlaceholderData } = usePaginatedUsers(page, role);
  const queryClient = useQueryClient();

  // Prefetch next page
  useEffect(() => {
    if (data && data.total > page * 20) {
      queryClient.prefetchQuery({
        queryKey: userKeys.list({ role, page: page + 1 }),
        queryFn: () => usersApi.list({ role, page: page + 1, limit: 20 }),
      });
    }
  }, [data, page, role, queryClient]);

  const totalPages = data ? Math.ceil(data.total / 20) : 0;

  return (
    <div>
      <UserTable users={data?.users ?? []} isStale={isPlaceholderData} />
      <Pagination
        current={page}
        total={totalPages}
        onChange={setPage}
        disabled={isPlaceholderData}
      />
    </div>
  );
}
```

The `keepPreviousData` (v5) / `placeholderData: keepPreviousData` option is what makes paginated UI feel good. Without it, switching pages shows a loading spinner while the new page fetches. With it, the old page stays visible (optionally dimmed) while the new one loads in the background.

### 2.8 Mutations: The Complete Pattern

Mutations are how you tell the server to change something. Here's the full pattern with loading states, error handling, and cache invalidation:

```typescript
// src/queries/mutations/useCreatePost.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postsApi, type CreatePostPayload, type Post } from '../../api/posts';
import { postKeys } from '../keys';

export function useCreatePost() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (payload: CreatePostPayload) => postsApi.create(payload),

    onSuccess: (newPost) => {
      // Option 1: Invalidate the list (simplest, triggers refetch)
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });

      // Option 2: Also seed the detail cache with the returned data
      queryClient.setQueryData(postKeys.detail(newPost.id), newPost);
    },

    onError: (error) => {
      // Error handling is usually done in the component, but you can
      // add logging or analytics here
      console.error('Failed to create post:', error);
    },
  });
}

// Usage in component
function CreatePostForm() {
  const createPost = useCreatePost();

  const handleSubmit = (data: CreatePostPayload) => {
    createPost.mutate(data, {
      onSuccess: (post) => {
        router.push(`/posts/${post.id}`);
      },
      onError: (error) => {
        toast.error(getUserFacingMessage(error));
      },
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
      <button type="submit" disabled={createPost.isPending}>
        {createPost.isPending ? 'Creating...' : 'Create Post'}
      </button>
    </form>
  );
}
```

---

## 3. OPTIMISTIC UPDATES WITH ROLLBACK

Optimistic updates make your app feel instant. Instead of waiting for the server to confirm a change, you update the UI immediately and roll back if the server rejects it. The user sees zero latency for most operations.

But they add real complexity. Here's when they're worth it and how to do them properly.

### 3.1 When to Use Optimistic Updates

| Scenario | Optimistic? | Why |
|----------|-------------|-----|
| Like/unlike a post | Yes | High frequency, low stakes, server rarely rejects |
| Toggle a boolean setting | Yes | Instant feedback matters, low conflict risk |
| Reorder items in a list | Yes | Drag-and-drop requires immediate visual feedback |
| Delete an item | Maybe | Depends on stakes. A todo? Yes. An invoice? No. |
| Create a new item | Rarely | You don't have the server-generated ID yet |
| Financial transaction | No | Stakes too high, must wait for server confirmation |
| Multi-user collaborative edit | No | Conflict resolution needs server arbitration |

**Rule of thumb:** Use optimistic updates when (a) the operation almost always succeeds, (b) the cost of a brief inconsistency is low, and (c) the latency difference is perceptible to users.

### 3.2 The Full Pattern

```typescript
// src/queries/mutations/useToggleLike.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { postsApi, type Post } from '../../api/posts';
import { postKeys } from '../keys';

interface ToggleLikeContext {
  previousPost: Post | undefined;
}

export function useToggleLike(postId: string) {
  const queryClient = useQueryClient();

  return useMutation<Post, Error, void, ToggleLikeContext>({
    mutationFn: () => postsApi.toggleLike(postId),

    // --- Step 1: Optimistic Update ---
    onMutate: async () => {
      // Cancel any in-flight refetches so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: postKeys.detail(postId) });

      // Snapshot the current value (for rollback)
      const previousPost = queryClient.getQueryData<Post>(postKeys.detail(postId));

      // Optimistically update the cache
      queryClient.setQueryData<Post>(postKeys.detail(postId), (old) => {
        if (!old) return old;
        return {
          ...old,
          isLiked: !old.isLiked,
          likeCount: old.isLiked ? old.likeCount - 1 : old.likeCount + 1,
        };
      });

      // Return the snapshot as context for rollback
      return { previousPost };
    },

    // --- Step 2: Rollback on Error ---
    onError: (_error, _variables, context) => {
      // Restore the previous value
      if (context?.previousPost) {
        queryClient.setQueryData(postKeys.detail(postId), context.previousPost);
      }
    },

    // --- Step 3: Reconcile on Settle ---
    onSettled: () => {
      // Whether success or error, refetch to ensure cache matches server
      queryClient.invalidateQueries({ queryKey: postKeys.detail(postId) });
    },
  });
}
```

Let's break down the three callbacks:

1. **`onMutate`** fires before the mutation function. This is where you cancel related queries, snapshot the current cache, and write the optimistic value. The return value becomes the `context` available in the other callbacks.

2. **`onError`** fires if the mutation fails. You use the context (the snapshot) to restore the cache to its previous state. The user briefly sees their action, then sees it revert.

3. **`onSettled`** fires regardless of success or failure. You invalidate the query to trigger a background refetch, which ensures the cache eventually reflects the true server state.

### 3.3 Optimistic Updates in Lists

Updating a single item is straightforward. Lists are trickier because you're modifying an array inside cached data:

```typescript
// Optimistic delete from a list
export function useDeletePost() {
  const queryClient = useQueryClient();

  return useMutation<void, Error, string, { previousLists: Map<string, unknown> }>({
    mutationFn: (postId: string) => postsApi.delete(postId),

    onMutate: async (postId) => {
      // Cancel all list queries
      await queryClient.cancelQueries({ queryKey: postKeys.lists() });

      // Snapshot ALL list caches (there might be multiple filtered lists)
      const previousLists = new Map<string, unknown>();
      const listQueries = queryClient.getQueriesData({ queryKey: postKeys.lists() });
      listQueries.forEach(([key, data]) => {
        previousLists.set(JSON.stringify(key), data);
      });

      // Optimistically remove the post from all list caches
      queryClient.setQueriesData<{ posts: Post[]; total: number }>(
        { queryKey: postKeys.lists() },
        (old) => {
          if (!old) return old;
          return {
            posts: old.posts.filter((p) => p.id !== postId),
            total: old.total - 1,
          };
        },
      );

      // Also remove the detail cache
      queryClient.removeQueries({ queryKey: postKeys.detail(postId) });

      return { previousLists };
    },

    onError: (_error, _postId, context) => {
      // Restore all list caches
      context?.previousLists.forEach((data, keyString) => {
        const key = JSON.parse(keyString);
        queryClient.setQueryData(key, data);
      });
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    },
  });
}
```

### 3.4 Optimistic Reorder

Drag-and-drop reordering is the canonical case for optimistic updates. The user expects items to stay where they dropped them.

```typescript
export function useReorderItems(listId: string) {
  const queryClient = useQueryClient();

  return useMutation<
    void,
    Error,
    { itemId: string; newIndex: number },
    { previousItems: Item[] | undefined }
  >({
    mutationFn: ({ itemId, newIndex }) =>
      listsApi.reorder(listId, itemId, newIndex),

    onMutate: async ({ itemId, newIndex }) => {
      await queryClient.cancelQueries({ queryKey: listKeys.items(listId) });

      const previousItems = queryClient.getQueryData<Item[]>(
        listKeys.items(listId),
      );

      queryClient.setQueryData<Item[]>(listKeys.items(listId), (old) => {
        if (!old) return old;
        const items = [...old];
        const currentIndex = items.findIndex((i) => i.id === itemId);
        if (currentIndex === -1) return old;
        const [item] = items.splice(currentIndex, 1);
        items.splice(newIndex, 0, item);
        return items;
      });

      return { previousItems };
    },

    onError: (_error, _variables, context) => {
      if (context?.previousItems) {
        queryClient.setQueryData(listKeys.items(listId), context.previousItems);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: listKeys.items(listId) });
    },
  });
}
```

### 3.5 When Optimistic Updates Go Wrong

The biggest risk is **state divergence**: your UI shows one thing, the server has another, and the reconciliation in `onSettled` causes a jarring UI jump.

Common causes:
- **Concurrent mutations:** User likes a post, then unlikes it before the first mutation settles. The `onSettled` invalidation from the first mutation overwrites the second optimistic update.
- **Stale snapshots:** The snapshot in `onMutate` captured old data. Rollback restores a version that's already out of date.
- **Partial failures in batch operations:** Three items reordered, server accepts two but rejects one.

**Mitigation:** For high-frequency operations (rapid likes/unlikes, fast drag-and-drop), debounce mutations or use the mutation's `mutationKey` to deduplicate:

```typescript
useMutation({
  mutationKey: ['toggle-like', postId], // Deduplicates by key
  mutationFn: () => postsApi.toggleLike(postId),
  // ...
});
```

---

## 4. tRPC END-TO-END

tRPC gives you end-to-end type safety between your backend and frontend without code generation, without schema files, and without a build step. If you own both ends of the wire (your team writes the API and the client), tRPC eliminates an entire class of bugs.

### 4.1 How tRPC Works

The core idea: your backend defines procedures (queries and mutations) as TypeScript functions. tRPC infers the input and output types from those functions and makes them available to the client. Change a field name on the server, your client gets a type error immediately. No `openapi-generator`, no `graphql-codegen`, no "regenerate the client" step.

```
Server (TypeScript) ──type inference──> Client (TypeScript)
     │                                         │
     │  router.user.getById                    │  trpc.user.getById.useQuery('123')
     │  input: z.string()                      │  input: string ✓ (inferred)
     │  output: { id, name, email }            │  output: { id, name, email } ✓ (inferred)
     │                                         │
     └──── No codegen, no schema file ─────────┘
```

### 4.2 Server Setup (Next.js App Router)

```typescript
// packages/api/src/trpc.ts (in a monorepo shared package)
import { initTRPC, TRPCError } from '@trpc/server';
import superjson from 'superjson';
import { z } from 'zod';
import type { Session } from './auth';

// Context available to all procedures
interface Context {
  session: Session | null;
  db: typeof db; // Your database client (Drizzle, Prisma, etc.)
}

const t = initTRPC.context<Context>().create({
  transformer: superjson, // Handles Dates, Maps, Sets, etc.
  errorFormatter({ shape, error }) {
    return {
      ...shape,
      data: {
        ...shape.data,
        zodError:
          error.cause instanceof z.ZodError ? error.cause.flatten() : null,
      },
    };
  },
});

export const router = t.router;
export const publicProcedure = t.procedure;

// Middleware: require authentication
const enforceAuth = t.middleware(({ ctx, next }) => {
  if (!ctx.session?.user) {
    throw new TRPCError({ code: 'UNAUTHORIZED' });
  }
  return next({
    ctx: { ...ctx, session: ctx.session as Session & { user: NonNullable<Session['user']> } },
  });
});

export const protectedProcedure = t.procedure.use(enforceAuth);
```

```typescript
// packages/api/src/routers/user.ts
import { z } from 'zod';
import { router, protectedProcedure, publicProcedure } from '../trpc';

export const userRouter = router({
  me: protectedProcedure.query(async ({ ctx }) => {
    const user = await ctx.db.query.users.findFirst({
      where: eq(users.id, ctx.session.user.id),
    });
    if (!user) throw new TRPCError({ code: 'NOT_FOUND' });
    return user;
  }),

  getById: publicProcedure
    .input(z.string().uuid())
    .query(async ({ ctx, input }) => {
      const user = await ctx.db.query.users.findFirst({
        where: eq(users.id, input),
      });
      if (!user) throw new TRPCError({ code: 'NOT_FOUND' });
      return user;
    }),

  update: protectedProcedure
    .input(z.object({
      name: z.string().min(1).max(100).optional(),
      bio: z.string().max(500).optional(),
      avatarUrl: z.string().url().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const updated = await ctx.db
        .update(users)
        .set(input)
        .where(eq(users.id, ctx.session.user.id))
        .returning();
      return updated[0];
    }),

  list: protectedProcedure
    .input(z.object({
      cursor: z.string().uuid().optional(),
      limit: z.number().min(1).max(100).default(20),
      role: z.enum(['admin', 'member', 'viewer']).optional(),
    }))
    .query(async ({ ctx, input }) => {
      const { cursor, limit, role } = input;
      const items = await ctx.db.query.users.findMany({
        where: and(
          role ? eq(users.role, role) : undefined,
          cursor ? gt(users.id, cursor) : undefined,
        ),
        limit: limit + 1, // Fetch one extra to determine if there's a next page
        orderBy: asc(users.id),
      });

      let nextCursor: string | undefined;
      if (items.length > limit) {
        const nextItem = items.pop()!;
        nextCursor = nextItem.id;
      }

      return { items, nextCursor };
    }),
});
```

```typescript
// packages/api/src/routers/post.ts
import { z } from 'zod';
import { router, protectedProcedure, publicProcedure } from '../trpc';

export const postRouter = router({
  feed: publicProcedure
    .input(z.object({
      cursor: z.string().uuid().optional(),
      limit: z.number().min(1).max(50).default(20),
    }))
    .query(async ({ ctx, input }) => {
      const { cursor, limit } = input;
      const items = await ctx.db.query.posts.findMany({
        where: and(
          eq(posts.status, 'published'),
          cursor ? lt(posts.createdAt, cursor) : undefined,
        ),
        limit: limit + 1,
        orderBy: desc(posts.createdAt),
        with: { author: true },
      });

      let nextCursor: string | undefined;
      if (items.length > limit) {
        nextCursor = items.pop()!.createdAt;
      }

      return { items, nextCursor };
    }),

  create: protectedProcedure
    .input(z.object({
      title: z.string().min(1).max(200),
      body: z.string().min(1),
      status: z.enum(['draft', 'published']).default('draft'),
    }))
    .mutation(async ({ ctx, input }) => {
      const [post] = await ctx.db
        .insert(posts)
        .values({ ...input, authorId: ctx.session.user.id })
        .returning();
      return post;
    }),

  toggleLike: protectedProcedure
    .input(z.string().uuid())
    .mutation(async ({ ctx, input: postId }) => {
      const existing = await ctx.db.query.likes.findFirst({
        where: and(eq(likes.postId, postId), eq(likes.userId, ctx.session.user.id)),
      });

      if (existing) {
        await ctx.db.delete(likes).where(eq(likes.id, existing.id));
        return { liked: false };
      } else {
        await ctx.db.insert(likes).values({ postId, userId: ctx.session.user.id });
        return { liked: true };
      }
    }),
});
```

```typescript
// packages/api/src/root.ts
import { router } from './trpc';
import { userRouter } from './routers/user';
import { postRouter } from './routers/post';

export const appRouter = router({
  user: userRouter,
  post: postRouter,
});

// THE KEY EXPORT: this type is what the client imports
export type AppRouter = typeof appRouter;
```

### 4.3 Next.js Integration

```typescript
// apps/web/src/app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch';
import { appRouter } from '@acme/api';
import { createContext } from '@acme/api/context';

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext: () => createContext(req),
  });

export { handler as GET, handler as POST };
```

```typescript
// apps/web/src/lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@acme/api';

export const trpc = createTRPCReact<AppRouter>();
```

```typescript
// apps/web/src/lib/trpc-provider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { useState } from 'react';
import superjson from 'superjson';
import { trpc } from './trpc';

function getBaseUrl() {
  if (typeof window !== 'undefined') return ''; // browser should use relative path
  if (process.env.VERCEL_URL) return `https://${process.env.VERCEL_URL}`;
  return 'http://localhost:3000';
}

export function TRPCProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 5 * 60 * 1000,
            retry: 2,
          },
        },
      }),
  );

  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: `${getBaseUrl()}/api/trpc`,
          transformer: superjson,
          headers() {
            return {
              // Auth headers if needed (or handle via cookies)
            };
          },
        }),
      ],
    }),
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

### 4.4 React Native Client (Same Monorepo)

This is the monorepo advantage. The React Native app imports the same `AppRouter` type and gets the same type safety without any code generation step.

```typescript
// apps/mobile/src/lib/trpc.ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '@acme/api';

export const trpc = createTRPCReact<AppRouter>();
```

```typescript
// apps/mobile/src/lib/trpc-provider.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { httpBatchLink } from '@trpc/client';
import { useState } from 'react';
import superjson from 'superjson';
import { trpc } from './trpc';
import { getAuthToken } from '../auth/storage'; // MMKV-based token store

const API_URL = __DEV__
  ? 'http://localhost:3000/api/trpc'
  : 'https://api.yourapp.com/trpc';

export function TRPCProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 5 * 60 * 1000,
            retry: (failureCount, error) => {
              // Don't retry on 4xx errors
              if (error instanceof Error && 'data' in error) {
                const trpcError = error as any;
                if (trpcError.data?.httpStatus < 500) return false;
              }
              return failureCount < 3;
            },
            networkMode: 'offlineFirst', // Important for mobile
          },
        },
      }),
  );

  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: API_URL,
          transformer: superjson,
          async headers() {
            const token = await getAuthToken();
            return token ? { Authorization: `Bearer ${token}` } : {};
          },
        }),
      ],
    }),
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

### 4.5 Using tRPC in Components

```tsx
// The developer experience is where tRPC shines

function ProfileScreen() {
  // Fully typed -- hover over `data` and your IDE shows the exact return type
  // from the server procedure. No generated types file to import.
  const { data: user, isLoading } = trpc.user.me.useQuery();

  const updateUser = trpc.user.update.useMutation({
    onSuccess: () => {
      // Invalidation works exactly like TanStack Query
      // because tRPC *is* TanStack Query under the hood
      utils.user.me.invalidate();
    },
  });

  const utils = trpc.useUtils();

  if (isLoading) return <ProfileSkeleton />;
  if (!user) return <NotFound />;

  return (
    <View>
      <Text style={styles.name}>{user.name}</Text>
      <Text style={styles.email}>{user.email}</Text>
      <Button
        onPress={() => updateUser.mutate({ name: 'New Name' })}
        loading={updateUser.isPending}
      >
        Update Name
      </Button>
    </View>
  );
}
```

```tsx
// Infinite query with tRPC
function FeedScreen() {
  const feed = trpc.post.feed.useInfiniteQuery(
    { limit: 20 },
    {
      getNextPageParam: (lastPage) => lastPage.nextCursor,
    },
  );

  const posts = feed.data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <PostCard post={item} />}
      onEndReached={() => feed.hasNextPage && feed.fetchNextPage()}
      onEndReachedThreshold={0.5}
    />
  );
}
```

### 4.6 The Monorepo Structure

```
apps/
  web/                  # Next.js
    src/
      lib/trpc.ts       # createTRPCReact<AppRouter>()
  mobile/               # Expo / React Native
    src/
      lib/trpc.ts       # createTRPCReact<AppRouter>() (same pattern)
packages/
  api/                  # Shared backend
    src/
      trpc.ts           # initTRPC, procedures, middleware
      root.ts           # appRouter, export type AppRouter
      routers/
        user.ts
        post.ts
        notification.ts
  db/                   # Shared database schema
    src/
      schema.ts         # Drizzle schema
      index.ts          # db client export
```

The `packages/api` package is the single source of truth. Both `apps/web` and `apps/mobile` import `AppRouter` from it. When you add a field to a user router response, both apps see it instantly. When you remove a field, both apps get type errors. This is the dream that GraphQL codegen promises but delivers with a build step and a `.graphql` schema file. tRPC delivers it with pure TypeScript inference.

---

## 5. REST VS GRAPHQL VS tRPC

This is one of those topics where people have strong opinions and weak reasoning. Let's fix that with a clear decision framework.

### 5.1 The Decision Matrix

| Factor | REST | GraphQL | tRPC |
|--------|------|---------|------|
| **Best when** | Public APIs, third-party consumers | Complex data graphs, multiple clients with different needs | You own both ends, TypeScript everywhere |
| **Type safety** | Manual (OpenAPI + codegen) | Semi-auto (codegen from schema) | Automatic (TypeScript inference) |
| **Over-fetching** | Common (fixed response shapes) | Solved (client specifies fields) | N/A (you control both ends) |
| **Under-fetching** | Common (multiple requests needed) | Solved (nested queries) | Solved (compose procedures) |
| **Caching** | HTTP caching (CDN, browser) works naturally | Harder (POST requests, normalized cache) | HTTP caching for queries (GET) |
| **Tooling maturity** | Excellent (Postman, curl, browser) | Good (Apollo DevTools, GraphiQL) | Good (tRPC panel, TanStack Query DevTools) |
| **Learning curve** | Low | Medium-High | Low (if you know TypeScript) |
| **Bundle size (client)** | Minimal | 30-60 KB (Apollo) or ~10 KB (urql) | ~10 KB |
| **Real-time** | WebSockets / SSE (separate) | Subscriptions (built-in) | Subscriptions (built-in) |
| **File uploads** | Native (multipart/form-data) | Painful (spec extensions) | Manual (separate endpoint) |
| **Team boundary** | Great (contract-first) | Great (schema-first) | Poor (shared TypeScript required) |
| **Mobile network** | Good (HTTP caching) | Medium (no HTTP caching) | Good (uses HTTP GET for queries) |

### 5.2 When REST Wins

**Public APIs.** If anyone outside your team will consume your API, REST is the standard. Every language has an HTTP client. `curl` works. Browsers can hit your endpoints directly. OpenAPI specs generate clients in any language.

**HTTP caching.** REST GETs are cacheable by CDNs, browsers, and intermediary proxies. A `GET /posts/123` with proper `Cache-Control` headers means your CDN serves the response without hitting your server. GraphQL sends everything as POST, which CDNs won't cache by default.

**Simplicity.** For CRUD apps with straightforward data models, REST is the simplest choice. One endpoint per resource, standard HTTP methods, standard status codes. No query language to learn.

### 5.3 When GraphQL Wins

**Multiple clients with different data needs.** If your web app shows a post with comments, likes, and author details, but your mobile app only shows the post title and like count, GraphQL lets each client request exactly what it needs. REST would require either separate endpoints or accepting over-fetching.

**Complex data graphs.** If your domain has deeply nested relationships (e.g., organization -> teams -> members -> projects -> tasks -> comments), GraphQL's nested queries avoid the waterfall of REST calls.

**Rapid frontend iteration.** Frontend teams can add new fields to their queries without backend changes, as long as the schema exposes those fields. This decouples deployment schedules.

```graphql
# The web app requests everything
query PostDetail($id: ID!) {
  post(id: $id) {
    id
    title
    body
    author { id name avatarUrl }
    comments(first: 10) {
      edges {
        node { id body author { name } }
      }
    }
    likeCount
    isLikedByMe
  }
}

# The mobile app requests a subset
query PostCard($id: ID!) {
  post(id: $id) {
    id
    title
    likeCount
    isLikedByMe
  }
}
```

### 5.4 When tRPC Wins

**Monorepo with TypeScript on both ends.** This is tRPC's sweet spot. You get type safety without codegen, instant feedback when APIs change, and the simplest possible mental model: "it's just a function call."

**Speed of development.** There is no schema to maintain, no types to generate, no build step between changing the API and using it. You change the server, the client type-checks immediately. For small-to-medium teams iterating fast, this is a massive productivity multiplier.

**React Native + Next.js apps.** The monorepo structure in Section 4.6 is the canonical example. One API package, two clients, zero codegen.

### 5.5 When tRPC Loses

**Team boundaries.** If your backend team uses Go or Python, tRPC is off the table. It requires TypeScript on both ends.

**Public APIs.** tRPC is not designed for third-party consumption. It's an internal protocol.

**Very large organizations.** When you have 50 frontend engineers and 30 backend engineers working on different deployment schedules, a schema-first approach (GraphQL or OpenAPI) provides a better contract boundary than shared TypeScript types.

### 5.6 The Hybrid Approach

You don't have to pick one. Many production apps use:
- **tRPC** for internal app-to-API communication
- **REST** for public-facing APIs and webhooks
- **GraphQL** for a specific data-heavy feature (like a dashboard with complex filtering)

```typescript
// Your tRPC router can coexist with REST endpoints
// Next.js App Router:

// /api/trpc/[trpc]/route.ts  -- tRPC for internal use
// /api/v1/webhooks/route.ts  -- REST for Stripe webhooks
// /api/v1/public/route.ts    -- REST for public API
```

---

## 6. REAL-TIME COMMUNICATION

Some data should not be polled. Chat messages, collaborative cursors, live notifications, stock prices -- these need to arrive the instant they happen. Let's cover the two main approaches and when to use each.

### 6.1 WebSockets vs Server-Sent Events

| Feature | WebSockets | Server-Sent Events (SSE) |
|---------|------------|--------------------------|
| Direction | Bidirectional | Server to client only |
| Protocol | `ws://` / `wss://` | Regular HTTP (`text/event-stream`) |
| Reconnection | Manual | Automatic (built into EventSource) |
| Binary data | Yes | No (text only) |
| HTTP/2 multiplexing | No (separate TCP) | Yes |
| Browser support | Universal | Universal (except IE, who cares) |
| Through proxies/CDNs | Sometimes problematic | Works everywhere HTTP works |
| Max connections | ~6 per domain (browser limit) | Same HTTP limit, but multiplexed with HTTP/2 |
| Complexity | Higher | Lower |

**Use WebSockets when:**
- You need bidirectional communication (chat, collaborative editing, gaming)
- You're sending binary data (audio/video streams)
- You need very low latency in both directions

**Use SSE when:**
- Server pushes data to client (notifications, live feeds, progress updates)
- Client sends data via normal HTTP requests (most cases)
- You want automatic reconnection without code
- You want to work through all proxies and CDNs without issues

### 6.2 Server-Sent Events Implementation

SSE is dramatically underused. For most "real-time" features (notifications, feed updates, progress bars), SSE is simpler, more reliable, and easier to scale than WebSockets.

```typescript
// Server: Next.js API route
// app/api/notifications/stream/route.ts

export async function GET(request: Request) {
  const encoder = new TextEncoder();
  const userId = await getUserIdFromRequest(request);

  const stream = new ReadableStream({
    async start(controller) {
      // Send a comment to keep connection alive
      const keepAlive = setInterval(() => {
        controller.enqueue(encoder.encode(': keepalive\n\n'));
      }, 30_000);

      // Subscribe to notifications for this user
      const unsubscribe = notificationService.subscribe(userId, (notification) => {
        const data = JSON.stringify(notification);
        controller.enqueue(encoder.encode(`event: notification\ndata: ${data}\n\n`));
      });

      // Clean up on disconnect
      request.signal.addEventListener('abort', () => {
        clearInterval(keepAlive);
        unsubscribe();
        controller.close();
      });
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      Connection: 'keep-alive',
    },
  });
}
```

```typescript
// Client: React hook for SSE
// src/hooks/useNotificationStream.ts

import { useEffect, useRef, useCallback } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { notificationKeys } from '../queries/keys';

export function useNotificationStream() {
  const queryClient = useQueryClient();
  const eventSourceRef = useRef<EventSource | null>(null);

  useEffect(() => {
    const eventSource = new EventSource('/api/notifications/stream');
    eventSourceRef.current = eventSource;

    eventSource.addEventListener('notification', (event) => {
      const notification = JSON.parse(event.data);

      // Update TanStack Query cache with the new notification
      queryClient.setQueryData<Notification[]>(
        notificationKeys.all,
        (old = []) => [notification, ...old],
      );

      // Update unread count
      queryClient.setQueryData<number>(
        notificationKeys.unreadCount(),
        (old = 0) => old + 1,
      );
    });

    eventSource.onerror = () => {
      // EventSource automatically reconnects -- this fires on each reconnect attempt
      console.warn('Notification stream disconnected, reconnecting...');
    };

    return () => {
      eventSource.close();
    };
  }, [queryClient]);
}
```

```typescript
// React Native: EventSource polyfill
// React Native doesn't have native EventSource.
// Use a polyfill like 'react-native-sse' or 'eventsource'

// apps/mobile/src/hooks/useNotificationStream.ts
import EventSource from 'react-native-sse';

export function useNotificationStream(token: string) {
  const queryClient = useQueryClient();

  useEffect(() => {
    const es = new EventSource(`${API_URL}/notifications/stream`, {
      headers: { Authorization: `Bearer ${token}` },
    });

    es.addEventListener('notification', (event: any) => {
      const notification = JSON.parse(event.data);
      queryClient.setQueryData<Notification[]>(
        notificationKeys.all,
        (old = []) => [notification, ...old],
      );
    });

    return () => es.close();
  }, [token, queryClient]);
}
```

### 6.3 WebSocket Implementation

For bidirectional communication (chat is the canonical example):

```typescript
// src/lib/websocket.ts

type MessageHandler = (data: unknown) => void;
type ConnectionHandler = () => void;

interface WebSocketClientOptions {
  url: string;
  getToken: () => Promise<string | null>;
  onConnect?: ConnectionHandler;
  onDisconnect?: ConnectionHandler;
  reconnectInterval?: number;
  maxReconnectAttempts?: number;
}

export class WebSocketClient {
  private ws: WebSocket | null = null;
  private handlers = new Map<string, Set<MessageHandler>>();
  private reconnectAttempts = 0;
  private reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private intentionallyClosed = false;

  constructor(private options: WebSocketClientOptions) {}

  async connect() {
    this.intentionallyClosed = false;
    const token = await this.options.getToken();
    const url = new URL(this.options.url);
    if (token) url.searchParams.set('token', token);

    this.ws = new WebSocket(url.toString());

    this.ws.onopen = () => {
      this.reconnectAttempts = 0;
      this.options.onConnect?.();
    };

    this.ws.onmessage = (event) => {
      try {
        const message = JSON.parse(event.data);
        const { type, payload } = message;
        const handlers = this.handlers.get(type);
        handlers?.forEach((handler) => handler(payload));
      } catch (error) {
        console.error('WebSocket message parse error:', error);
      }
    };

    this.ws.onclose = () => {
      this.options.onDisconnect?.();
      if (!this.intentionallyClosed) {
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }

  private scheduleReconnect() {
    const maxAttempts = this.options.maxReconnectAttempts ?? 10;
    if (this.reconnectAttempts >= maxAttempts) {
      console.error('WebSocket max reconnect attempts reached');
      return;
    }

    const baseInterval = this.options.reconnectInterval ?? 1000;
    // Exponential backoff with jitter
    const delay = Math.min(
      baseInterval * Math.pow(2, this.reconnectAttempts) + Math.random() * 1000,
      30_000,
    );

    this.reconnectTimer = setTimeout(() => {
      this.reconnectAttempts++;
      this.connect();
    }, delay);
  }

  on(type: string, handler: MessageHandler) {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler);

    // Return unsubscribe function
    return () => {
      this.handlers.get(type)?.delete(handler);
    };
  }

  send(type: string, payload: unknown) {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({ type, payload }));
    } else {
      console.warn('WebSocket not connected, message dropped:', type);
    }
  }

  disconnect() {
    this.intentionallyClosed = true;
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }
    this.ws?.close();
  }
}
```

```tsx
// src/hooks/useChat.ts

import { useEffect, useRef, useCallback, useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { WebSocketClient } from '../lib/websocket';

interface ChatMessage {
  id: string;
  channelId: string;
  authorId: string;
  body: string;
  createdAt: string;
}

export function useChat(channelId: string) {
  const queryClient = useQueryClient();
  const clientRef = useRef<WebSocketClient | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const client = new WebSocketClient({
      url: `${WS_URL}/chat`,
      getToken: () => getAuthToken(),
      onConnect: () => setIsConnected(true),
      onDisconnect: () => setIsConnected(false),
    });

    clientRef.current = client;
    client.connect();

    // Join channel
    client.on('connected', () => {
      client.send('join', { channelId });
    });

    // Listen for new messages
    client.on('message', (message: unknown) => {
      const msg = message as ChatMessage;
      if (msg.channelId === channelId) {
        queryClient.setQueryData<ChatMessage[]>(
          ['chat', channelId, 'messages'],
          (old = []) => [...old, msg],
        );
      }
    });

    // Listen for typing indicators
    client.on('typing', (data: unknown) => {
      const { userId, channelId: typingChannel } = data as {
        userId: string;
        channelId: string;
      };
      if (typingChannel === channelId) {
        queryClient.setQueryData<string[]>(
          ['chat', channelId, 'typing'],
          (old = []) => [...new Set([...old, userId])],
        );
        // Clear typing indicator after 3 seconds
        setTimeout(() => {
          queryClient.setQueryData<string[]>(
            ['chat', channelId, 'typing'],
            (old = []) => old.filter((id) => id !== userId),
          );
        }, 3000);
      }
    });

    return () => {
      client.send('leave', { channelId });
      client.disconnect();
    };
  }, [channelId, queryClient]);

  const sendMessage = useCallback(
    (body: string) => {
      clientRef.current?.send('message', { channelId, body });
    },
    [channelId],
  );

  const sendTyping = useCallback(() => {
    clientRef.current?.send('typing', { channelId });
  }, [channelId]);

  return { sendMessage, sendTyping, isConnected };
}
```

### 6.4 Socket.io vs Native WebSocket

| Factor | Native WebSocket | Socket.io |
|--------|-----------------|-----------|
| Bundle size | 0 KB (built-in) | ~45 KB (client) |
| Auto-reconnection | Manual | Built-in |
| Rooms/namespaces | Manual | Built-in |
| Fallback (long-polling) | No | Yes |
| Binary support | Yes | Yes |
| Acknowledgments | Manual | Built-in |
| Broadcasting | Manual (server) | Built-in (server) |
| React Native | Works | Works (with polyfill) |

**Use native WebSocket when:** You want minimal bundle size, have simple needs, or are connecting to a third-party WebSocket server that doesn't use Socket.io.

**Use Socket.io when:** You need rooms, acknowledgments, and auto-reconnection out of the box, and the server already uses Socket.io. Don't use Socket.io for one end -- it's a protocol on top of WebSocket, so both client and server must use it.

### 6.5 Integrating Real-Time with TanStack Query

The pattern you've seen throughout this section is the right one: real-time events update the TanStack Query cache directly. This gives you the best of both worlds -- declarative data fetching with imperative real-time updates.

```typescript
// Pattern: Real-time as cache invalidation
// Instead of updating the cache directly (which can get complex),
// you can simply invalidate the query and let TanStack Query refetch.

// This is simpler but adds latency:
client.on('post:updated', (data: { postId: string }) => {
  queryClient.invalidateQueries({ queryKey: postKeys.detail(data.postId) });
});

// Pattern: Real-time as cache update
// Direct cache update is instant but requires you to maintain cache structure:
client.on('post:updated', (data: { post: Post }) => {
  queryClient.setQueryData(postKeys.detail(data.post.id), data.post);
});

// Pattern: Hybrid -- update cache AND invalidate to reconcile
client.on('post:updated', (data: { post: Post }) => {
  // Instant update
  queryClient.setQueryData(postKeys.detail(data.post.id), data.post);
  // Background validation (catches fields the event didn't include)
  queryClient.invalidateQueries({
    queryKey: postKeys.detail(data.post.id),
    refetchType: 'none', // Don't refetch immediately, just mark stale
  });
});
```

---

## 7. PAGINATION PATTERNS

Pagination seems simple until you actually build it. There are two fundamentally different approaches, and choosing the wrong one creates real problems on mobile.

### 7.1 Offset-Based Pagination

The traditional approach: `GET /posts?page=2&limit=20` or `GET /posts?offset=20&limit=20`.

```
Page 1: offset=0,  limit=20  → items 1-20
Page 2: offset=20, limit=20  → items 21-40
Page 3: offset=40, limit=20  → items 41-60
```

**Pros:**
- Simple to implement
- Can jump to any page (`page=50`)
- Easy to show total page count
- Works with SQL `OFFSET/LIMIT`

**Cons:**
- **Item shifts.** If a new item is inserted while the user is on page 2, page 3 will include a duplicate from page 2. If an item is deleted, page 3 will skip an item.
- **Performance at scale.** `OFFSET 10000` in SQL means the database scans and discards 10,000 rows before returning 20. This gets slower as offset grows.
- **Terrible for infinite scroll.** When you append page 3 to pages 1+2 in a mobile list, the item shift problem means duplicates appear in the list.

### 7.2 Cursor-Based Pagination

The modern approach: `GET /posts?cursor=abc123&limit=20`. The cursor is an opaque token (usually an encoded ID or timestamp) that points to the last item the client received.

```
Page 1: cursor=null       → items 1-20, nextCursor="item20-id"
Page 2: cursor="item20-id" → items 21-40, nextCursor="item40-id"
Page 3: cursor="item40-id" → items 41-60, nextCursor="item60-id"
```

**Pros:**
- **Stable results.** Inserts and deletes don't cause duplicates or skips, because the cursor anchors to a specific item, not a position.
- **Performance at scale.** `WHERE id > cursor LIMIT 20` uses an index seek. Same speed whether it's page 1 or page 1000.
- **Perfect for infinite scroll.** Append-only semantics match the mobile UX.

**Cons:**
- Can't jump to arbitrary pages (no "page 50" button)
- Harder to show total count (requires separate count query)
- Cursor design requires thought (what if the cursor item is deleted?)

### 7.3 Why Cursor Is Better for Mobile

Mobile apps use infinite scroll almost exclusively. Users scroll through a feed and the app loads more at the bottom. This is an append-only operation, which is exactly what cursor-based pagination provides.

Offset-based pagination in a feed means: if someone posts a new item while you're scrolling, the item you just saw at position 20 shifts to position 21, and when you load the next page starting at offset 20, you see it again. On a phone with a FlatList, this manifests as a duplicate item suddenly appearing, or a weird jump in scroll position.

### 7.4 Full Implementation: Cursor-Based with useInfiniteQuery

Backend:

```typescript
// Server: Cursor-based pagination
export const postRouter = router({
  feed: publicProcedure
    .input(z.object({
      cursor: z.string().optional(),
      limit: z.number().min(1).max(50).default(20),
    }))
    .query(async ({ ctx, input }) => {
      const { cursor, limit } = input;

      const items = await ctx.db.query.posts.findMany({
        where: and(
          eq(posts.status, 'published'),
          cursor
            ? lt(posts.createdAt, new Date(cursor))
            : undefined,
        ),
        limit: limit + 1, // Fetch one extra to check for next page
        orderBy: desc(posts.createdAt),
        with: {
          author: { columns: { id: true, name: true, avatarUrl: true } },
        },
      });

      let nextCursor: string | undefined;
      if (items.length > limit) {
        const lastItem = items.pop()!; // Remove the extra item
        nextCursor = lastItem.createdAt.toISOString();
      }

      return {
        items,
        nextCursor,
      };
    }),
});
```

Client (TanStack Query):

```tsx
function useFeed() {
  return useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: async ({ pageParam }) => {
      const response = await postsApi.list({
        cursor: pageParam,
        limit: 20,
        status: 'published',
      });
      return response;
    },
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 2 * 60 * 1000,
  });
}
```

Client (tRPC):

```tsx
function useFeed() {
  return trpc.post.feed.useInfiniteQuery(
    { limit: 20 },
    {
      getNextPageParam: (lastPage) => lastPage.nextCursor,
      staleTime: 2 * 60 * 1000,
    },
  );
}
```

The FlatList integration:

```tsx
function FeedScreen() {
  const { data, fetchNextPage, hasNextPage, isFetchingNextPage, refetch, isRefetching } =
    useFeed();

  const posts = useMemo(
    () => data?.pages.flatMap((page) => page.items) ?? [],
    [data],
  );

  const handleEndReached = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <PostCard post={item} />}
      onEndReached={handleEndReached}
      onEndReachedThreshold={0.5}
      refreshControl={
        <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
      }
      ListFooterComponent={
        isFetchingNextPage ? (
          <View style={styles.footer}>
            <ActivityIndicator />
          </View>
        ) : null
      }
      // Performance optimizations for long lists
      removeClippedSubviews
      maxToRenderPerBatch={10}
      windowSize={5}
      getItemLayout={undefined} // Only use if items have fixed height
    />
  );
}
```

### 7.5 Bidirectional Pagination

Some feeds need to load in both directions -- older items at the bottom and newer items at the top (like a chat). TanStack Query supports this:

```typescript
function useChatMessages(channelId: string) {
  return useInfiniteQuery({
    queryKey: ['chat', channelId, 'messages'],
    queryFn: ({ pageParam }) =>
      chatApi.getMessages({
        channelId,
        cursor: pageParam?.cursor,
        direction: pageParam?.direction ?? 'older',
        limit: 30,
      }),
    initialPageParam: undefined as
      | { cursor: string; direction: 'older' | 'newer' }
      | undefined,
    getNextPageParam: (lastPage) =>
      lastPage.olderCursor
        ? { cursor: lastPage.olderCursor, direction: 'older' as const }
        : undefined,
    getPreviousPageParam: (firstPage) =>
      firstPage.newerCursor
        ? { cursor: firstPage.newerCursor, direction: 'newer' as const }
        : undefined,
  });
}
```

---

## 8. RETRY AND BACKOFF

Networks fail. Servers overload. Mobile connections drop. Your retry strategy is the difference between a resilient app and one that either crashes or DDoSes its own backend.

### 8.1 The Fundamentals

**Exponential backoff:** Each retry waits longer than the last. First retry after 1 second, second after 2 seconds, third after 4 seconds. This gives the server time to recover.

**Jitter:** Randomize the backoff delay. Without jitter, if 1000 clients all get a 503 at the same time, they all retry at 1s, then 2s, then 4s -- creating synchronized thundering herds at predictable intervals. With jitter, they spread out.

**Retry budgets:** Limit the total number of retries OR the total time spent retrying. Without limits, a broken endpoint generates infinite retry traffic.

### 8.2 The Math

```
delay = min(baseDelay * 2^attempt + random(0, jitter), maxDelay)
```

```typescript
// src/lib/retry.ts

interface RetryOptions {
  maxAttempts: number;
  baseDelay: number;      // milliseconds
  maxDelay: number;        // milliseconds
  jitterFactor: number;    // 0 to 1
  shouldRetry?: (error: unknown, attempt: number) => boolean;
}

const defaultOptions: RetryOptions = {
  maxAttempts: 3,
  baseDelay: 1000,
  maxDelay: 30_000,
  jitterFactor: 0.5,
};

export function calculateDelay(attempt: number, options: RetryOptions): number {
  const exponential = options.baseDelay * Math.pow(2, attempt);
  const jitter = exponential * options.jitterFactor * Math.random();
  return Math.min(exponential + jitter, options.maxDelay);
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {},
): Promise<T> {
  const opts = { ...defaultOptions, ...options };
  let lastError: unknown;

  for (let attempt = 0; attempt < opts.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Check if we should retry this specific error
      if (opts.shouldRetry && !opts.shouldRetry(error, attempt)) {
        throw error;
      }

      // Don't retry on client errors (4xx) -- those won't change
      if (error instanceof ApiError && error.status >= 400 && error.status < 500) {
        throw error;
      }

      // Don't wait after the last attempt
      if (attempt < opts.maxAttempts - 1) {
        const delay = calculateDelay(attempt, opts);
        await new Promise((resolve) => setTimeout(resolve, delay));
      }
    }
  }

  throw lastError;
}
```

Usage:

```typescript
// Retry with defaults (3 attempts, exponential backoff)
const user = await withRetry(() => usersApi.getById('123'));

// Custom retry for a flaky endpoint
const report = await withRetry(() => reportsApi.generate(), {
  maxAttempts: 5,
  baseDelay: 2000,
  maxDelay: 60_000,
  shouldRetry: (error) => {
    // Only retry on 503 (service unavailable)
    return error instanceof ApiError && error.status === 503;
  },
});
```

### 8.3 TanStack Query's Built-in Retry

TanStack Query has retry built in. You usually don't need the manual retry utility above for queries -- but understanding the config is important:

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 3,                          // Number of retries (default: 3)
      retryDelay: (attemptIndex) =>      // Custom backoff
        Math.min(1000 * 2 ** attemptIndex, 30_000),
    },
    mutations: {
      retry: 0,                          // Mutations don't retry by default
    },
  },
});
```

You can also configure retry per query:

```typescript
// Don't retry 404s
useQuery({
  queryKey: ['user', userId],
  queryFn: () => usersApi.getById(userId),
  retry: (failureCount, error) => {
    if (error instanceof ApiError && error.status === 404) return false;
    return failureCount < 3;
  },
});

// Retry mutations for idempotent operations
useMutation({
  mutationFn: (data) => postsApi.update(postId, data),
  retry: 2, // Safe because PUT is idempotent
});
```

### 8.4 Idempotency and Retry Safety

Not all operations are safe to retry. A `GET` request is always safe. A `PUT` (replace) is safe. A `DELETE` is safe (deleting twice is a no-op or 404). But a `POST` that creates a resource? Retrying it might create duplicates.

**Solution: Idempotency keys.**

```typescript
// Client sends a unique key with the request
import { v4 as uuid } from 'uuid';

export function useCreateOrder() {
  return useMutation({
    mutationFn: async (orderData: CreateOrderPayload) => {
      const idempotencyKey = uuid(); // Generate once per user action
      return api.post<Order>('/orders', orderData, {
        headers: { 'Idempotency-Key': idempotencyKey },
      });
    },
    retry: 2, // Now safe to retry -- server deduplicates by idempotency key
  });
}
```

The server stores the idempotency key and returns the same response for duplicate requests. This makes `POST` retryable.

### 8.5 Network Detection in React Native

Mobile apps need to know when the network is available. TanStack Query can pause queries when offline, but you need to tell it about network state.

```typescript
// apps/mobile/src/lib/network.ts
import NetInfo from '@react-native-community/netinfo';
import { onlineManager } from '@tanstack/react-query';

// Tell TanStack Query about network state
export function setupNetworkListener() {
  onlineManager.setEventListener((setOnline) => {
    return NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected);
    });
  });
}

// Call this in your app entry point
// App.tsx
useEffect(() => {
  setupNetworkListener();
}, []);
```

Now TanStack Query automatically pauses queries when offline and resumes them when the connection returns. Mutations queue up and fire when back online (with the `networkMode: 'offlineFirst'` setting).

```typescript
// You can also check network state manually
import NetInfo from '@react-native-community/netinfo';

async function isOnline(): Promise<boolean> {
  const state = await NetInfo.fetch();
  return !!state.isConnected;
}

// Use in UI
function OfflineBanner() {
  const [online, setOnline] = useState(true);

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected);
    });
    return unsubscribe;
  }, []);

  if (online) return null;

  return (
    <View style={styles.banner}>
      <Text style={styles.bannerText}>
        You're offline. Changes will sync when you reconnect.
      </Text>
    </View>
  );
}
```

### 8.6 Retry Visualization

Here is what exponential backoff with jitter looks like over 5 attempts:

```
Attempt 0: Immediate
Attempt 1: ~1.0-1.5s  (base=1000, jitter=0-500)
Attempt 2: ~2.0-3.0s  (base=2000, jitter=0-1000)
Attempt 3: ~4.0-6.0s  (base=4000, jitter=0-2000)
Attempt 4: ~8.0-12.0s (base=8000, jitter=0-4000)

Total time before giving up: ~15-22 seconds

Without jitter (thundering herd):
1000 clients all retry at exactly 1s, 2s, 4s, 8s
Server gets 1000 requests every retry interval

With jitter:
1000 clients retry spread across the entire window
Server load is smooth, recovery is faster
```

---

## 9. OFFLINE PATTERNS

Offline support is the most overengineered feature in most apps -- until it's the most underengineered feature in an app that actually needs it. Here's how to think about it clearly and implement it without losing your mind.

### 9.1 The Spectrum of Offline Support

Not every app needs full offline support. Pick your level:

| Level | Description | Example | Complexity |
|-------|-------------|---------|------------|
| **0: None** | App is unusable without network | Video call app | Lowest |
| **1: Graceful degradation** | Show cached data, disable mutations | News reader | Low |
| **2: Read-heavy offline** | Full read access, mutations queue | Email client | Medium |
| **3: Full offline-first** | Full read/write, conflict resolution | Note-taking app, field service | High |

Most apps should aim for Level 1 or 2. Level 3 requires conflict resolution, which is a Hard Problem.

### 9.2 TanStack Query Persistence

TanStack Query can persist its cache to disk, giving you Level 1 offline support with almost no work:

```typescript
// apps/mobile/src/lib/query-persistence.ts
import { QueryClient } from '@tanstack/react-query';
import {
  PersistQueryClientProvider,
  type PersistedClient,
} from '@tanstack/react-query-persist-client';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV({ id: 'query-cache' });

// MMKV-based persister (much faster than AsyncStorage)
const mmkvPersister = {
  persistClient: (client: PersistedClient) => {
    storage.set('tanstack-query-cache', JSON.stringify(client));
  },
  restoreClient: (): PersistedClient | undefined => {
    const data = storage.getString('tanstack-query-cache');
    return data ? JSON.parse(data) : undefined;
  },
  removeClient: () => {
    storage.delete('tanstack-query-cache');
  },
};

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // 24 hours -- keep cached data for a day
      staleTime: 1000 * 60 * 5,     // 5 minutes
      networkMode: 'offlineFirst',   // Use cache first, then revalidate
    },
  },
});

// In your app root:
export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: mmkvPersister,
        maxAge: 1000 * 60 * 60 * 24, // Don't restore data older than 24 hours
        dehydrateOptions: {
          shouldDehydrateQuery: (query) => {
            // Only persist successful queries
            return query.state.status === 'success';
          },
        },
      }}
    >
      {children}
    </PersistQueryClientProvider>
  );
}
```

With this setup:
1. When the app opens, the cache is restored from MMKV instantly (MMKV is synchronous).
2. Queries return the cached data immediately, then background-refetch if online.
3. If offline, cached data is shown without errors.
4. After 24 hours, stale cache is discarded.

### 9.3 Offline Mutation Queue

Level 2 offline support means the user can perform actions while offline, and those actions replay when they come back online. This is harder than it sounds.

```typescript
// src/lib/offline-queue.ts
import { MMKV } from 'react-native-mmkv';
import NetInfo from '@react-native-community/netinfo';

interface QueuedMutation {
  id: string;
  type: string;
  payload: unknown;
  createdAt: number;
  retryCount: number;
}

const storage = new MMKV({ id: 'mutation-queue' });

class OfflineMutationQueue {
  private queue: QueuedMutation[] = [];
  private processing = false;
  private handlers = new Map<string, (payload: unknown) => Promise<void>>();

  constructor() {
    // Restore queue from disk
    const saved = storage.getString('queue');
    if (saved) {
      this.queue = JSON.parse(saved);
    }

    // Process queue when network comes back
    NetInfo.addEventListener((state) => {
      if (state.isConnected) {
        this.processQueue();
      }
    });
  }

  registerHandler(type: string, handler: (payload: unknown) => Promise<void>) {
    this.handlers.set(type, handler);
  }

  async enqueue(type: string, payload: unknown): Promise<string> {
    const mutation: QueuedMutation = {
      id: crypto.randomUUID(),
      type,
      payload,
      createdAt: Date.now(),
      retryCount: 0,
    };

    this.queue.push(mutation);
    this.persist();

    // Try to process immediately if online
    const state = await NetInfo.fetch();
    if (state.isConnected) {
      this.processQueue();
    }

    return mutation.id;
  }

  async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const mutation = this.queue[0];
      const handler = this.handlers.get(mutation.type);

      if (!handler) {
        console.error(`No handler for mutation type: ${mutation.type}`);
        this.queue.shift();
        this.persist();
        continue;
      }

      try {
        await handler(mutation.payload);
        this.queue.shift(); // Remove successful mutation
        this.persist();
      } catch (error) {
        mutation.retryCount++;

        if (mutation.retryCount >= 5) {
          console.error('Mutation failed after 5 retries, discarding:', mutation);
          this.queue.shift();
          this.persist();
          // TODO: Notify user of failed mutation
          continue;
        }

        // Check if we're offline
        const state = await NetInfo.fetch();
        if (!state.isConnected) {
          break; // Stop processing, wait for network
        }

        // Wait before retrying (exponential backoff)
        await new Promise((r) =>
          setTimeout(r, Math.min(1000 * Math.pow(2, mutation.retryCount), 30_000)),
        );
      }
    }

    this.processing = false;
  }

  private persist() {
    storage.set('queue', JSON.stringify(this.queue));
  }

  getQueueLength(): number {
    return this.queue.length;
  }

  getQueue(): readonly QueuedMutation[] {
    return this.queue;
  }
}

export const offlineQueue = new OfflineMutationQueue();
```

Register handlers at app startup:

```typescript
// src/lib/offline-handlers.ts
import { offlineQueue } from './offline-queue';
import { postsApi } from '../api/posts';
import { commentsApi } from '../api/comments';

export function registerOfflineHandlers() {
  offlineQueue.registerHandler('create-post', async (payload) => {
    await postsApi.create(payload as CreatePostPayload);
  });

  offlineQueue.registerHandler('create-comment', async (payload) => {
    const { postId, body } = payload as { postId: string; body: string };
    await commentsApi.create(postId, { body });
  });

  offlineQueue.registerHandler('toggle-like', async (payload) => {
    const { postId } = payload as { postId: string };
    await postsApi.toggleLike(postId);
  });
}
```

Use in components:

```tsx
function CreateCommentForm({ postId }: { postId: string }) {
  const [body, setBody] = useState('');
  const queryClient = useQueryClient();

  const handleSubmit = async () => {
    if (!body.trim()) return;

    // Optimistic update: add the comment to the cache immediately
    const optimisticComment: Comment = {
      id: `temp-${Date.now()}`,
      postId,
      body: body.trim(),
      authorId: currentUser.id,
      createdAt: new Date().toISOString(),
      isPending: true, // Flag for UI to show as pending
    };

    queryClient.setQueryData<Comment[]>(
      ['posts', postId, 'comments'],
      (old = []) => [...old, optimisticComment],
    );

    // Queue the mutation (works offline)
    await offlineQueue.enqueue('create-comment', { postId, body: body.trim() });

    setBody('');

    // If online, invalidate to get the real data
    const state = await NetInfo.fetch();
    if (state.isConnected) {
      queryClient.invalidateQueries({ queryKey: ['posts', postId, 'comments'] });
    }
  };

  return (
    <View>
      <TextInput value={body} onChangeText={setBody} placeholder="Write a comment..." />
      <Button onPress={handleSubmit}>Send</Button>
    </View>
  );
}
```

### 9.4 TanStack Query's Built-in Mutation Persistence

TanStack Query v5 has built-in support for persisting mutations through the `MutationCache` and the persistence plugin. This is simpler than building your own queue for most cases:

```typescript
import { MutationCache, QueryClient } from '@tanstack/react-query';

const mutationCache = new MutationCache({
  onSuccess: () => {
    // You can add global success handling here
  },
  onError: (error) => {
    // Global error handling for mutations
    console.error('Mutation failed:', error);
  },
});

const queryClient = new QueryClient({
  mutationCache,
  defaultOptions: {
    mutations: {
      networkMode: 'offlineFirst',
      retry: 3,
      retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30_000),
    },
  },
});

// With the persister from section 9.2, mutations are automatically
// persisted and replayed when the app restarts and comes back online.
```

The key setting is `networkMode: 'offlineFirst'`. With this:
- Mutations fire immediately if online
- If offline, mutations are paused and queued
- When the device comes back online, queued mutations fire automatically
- If the app is killed while offline, the persistence plugin restores and replays them

### 9.5 Legend State Sync

Legend State takes a fundamentally different approach to offline: instead of queuing mutations, it synchronizes state. You modify local state, and Legend State handles syncing those changes to the server.

```typescript
// src/state/posts.ts
import { observable } from '@legendapp/state';
import { synced } from '@legendapp/state/sync';
import { syncedCrud } from '@legendapp/state/sync-plugins/crud';

export const posts$ = observable(
  syncedCrud({
    // How to read
    list: async () => {
      const response = await postsApi.list({ status: 'published', limit: 100 });
      return response.posts;
    },

    // How to create
    create: async (post) => {
      return postsApi.create(post);
    },

    // How to update
    update: async (post) => {
      return postsApi.update(post.id, post);
    },

    // How to delete
    delete: async (post) => {
      await postsApi.delete(post.id);
    },

    // Persistence config
    persist: {
      name: 'posts',
      plugin: observablePersistMMKV, // Use MMKV for React Native
    },

    // Sync when online
    retry: {
      infinite: true,
      backoff: 'exponential',
      maxDelay: 30_000,
    },

    // Conflict resolution
    fieldUpdatedAt: 'updatedAt', // Use this field to determine which version wins
  }),
);
```

```tsx
// Usage in component
import { observer } from '@legendapp/state/react';
import { posts$ } from '../state/posts';

const FeedScreen = observer(function FeedScreen() {
  const posts = posts$.get();

  // Modify local state -- Legend State syncs to server
  const handleLike = (postId: string) => {
    posts$[postId].isLiked.set((prev) => !prev);
    // That's it. Legend State handles the sync.
  };

  return (
    <FlatList
      data={Object.values(posts)}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <PostCard post={item} onLike={() => handleLike(item.id)} />
      )}
    />
  );
});
```

Legend State's approach is excellent for apps where the data model is simple and bidirectional sync makes sense (todo lists, notes, settings). For complex apps with business logic in mutations (e.g., "creating an order triggers an email"), the explicit mutation queue approach is better because you need server-side logic to run.

### 9.6 Choosing Your Offline Strategy

```
┌─────────────────────────────────────────────────────────┐
│ Do you need offline support at all?                     │
│                                                         │
│ NO  → Skip this section. Show "You're offline" banner.  │
│ YES ↓                                                   │
├─────────────────────────────────────────────────────────┤
│ Read-only offline sufficient?                           │
│                                                         │
│ YES → TanStack Query + MMKV persister (Section 9.2)    │
│ NO  ↓                                                   │
├─────────────────────────────────────────────────────────┤
│ Mutations are simple CRUD?                              │
│                                                         │
│ YES → Legend State sync (Section 9.5)                   │
│       or TanStack Query mutation persistence (9.4)      │
│ NO  ↓                                                   │
├─────────────────────────────────────────────────────────┤
│ Mutations have server-side business logic?              │
│                                                         │
│ YES → Custom offline queue (Section 9.3)                │
│       Queue the mutation intent, replay when online.    │
│       Server handles business logic on replay.          │
│ NO  ↓                                                   │
├─────────────────────────────────────────────────────────┤
│ Need conflict resolution for multi-device sync?         │
│                                                         │
│ YES → Consider a sync engine (PowerSync, ElectricSQL,   │
│       Legend State with conflict resolution).            │
│       This is Hard. Budget for it.                      │
└─────────────────────────────────────────────────────────┘
```

---

## 10. PUTTING IT ALL TOGETHER

Let's see how all these patterns compose in a real app architecture. Here's the data layer for a social app with a feed, profiles, real-time notifications, and offline support.

### 10.1 The Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Components                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ FeedScreen│  │ Profile  │  │  Chat    │  │ Settings │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │         │
├───────┴──────────────┴──────────────┴──────────────┴────────┤
│                    Custom Hooks Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ useFeed  │  │useProfile │  │ useChat  │  │useSettings│   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │         │
├───────┴──────────────┴──────────┬───┴──────────────┴────────┤
│            TanStack Query       │    WebSocket Client       │
│  ┌──────────────────────────┐   │  ┌────────────────────┐   │
│  │ Queries · Mutations      │   │  │ Chat messages      │   │
│  │ Infinite · Prefetch      │   │  │ Typing indicators  │   │
│  │ Optimistic · Persisted   │   │  │ Presence           │   │
│  └────────────┬─────────────┘   │  └──────────┬─────────┘   │
│               │                 │              │             │
├───────────────┴─────────────────┴──────────────┴────────────┤
│                      API Layer                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Typed fetch client · Token refresh · Error handling  │   │
│  │ OR tRPC client with httpBatchLink                    │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             │                               │
├─────────────────────────────┴───────────────────────────────┤
│                    Persistence Layer                        │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────┐   │
│  │ MMKV         │  │ Query Cache     │  │ Mutation     │   │
│  │ (tokens,     │  │ Persister       │  │ Queue        │   │
│  │  settings)   │  │ (offline read)  │  │ (offline     │   │
│  │              │  │                 │  │  write)      │   │
│  └──────────────┘  └─────────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 The Query Configuration

```typescript
// src/lib/query-client.ts

import { QueryClient, MutationCache } from '@tanstack/react-query';
import { ApiError, NetworkError, getUserFacingMessage } from '../api/errors';
import { toast } from '../ui/toast'; // Your toast implementation

export const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error, _variables, _context, mutation) => {
      // Only show toast for mutations that don't handle their own errors
      if (mutation.options.onError) return;
      toast.error(getUserFacingMessage(error));
    },
  }),
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,       // 5 minutes
      gcTime: 1000 * 60 * 60 * 24,    // 24 hours (for offline persistence)
      retry: (failureCount, error) => {
        // Don't retry on 4xx errors
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) =>
        Math.min(1000 * 2 ** attemptIndex, 30_000),
      networkMode: 'offlineFirst',
      refetchOnWindowFocus: true,      // Refetch when tab/app becomes active
      refetchOnReconnect: true,        // Refetch when network comes back
    },
    mutations: {
      networkMode: 'offlineFirst',
      retry: (failureCount, error) => {
        if (error instanceof ApiError && error.status < 500) return false;
        return failureCount < 2;
      },
    },
  },
});
```

### 10.3 Complete Custom Hook Example

```typescript
// src/hooks/useFeed.ts

import { useInfiniteQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useCallback, useMemo } from 'react';
import { postsApi, type Post } from '../api/posts';
import { postKeys } from '../queries/keys';

export function useFeed() {
  const queryClient = useQueryClient();

  // Infinite query for the feed
  const feedQuery = useInfiniteQuery({
    queryKey: postKeys.list({ status: 'published' }),
    queryFn: ({ pageParam }) =>
      postsApi.list({ status: 'published', cursor: pageParam, limit: 20 }),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    staleTime: 2 * 60 * 1000,
  });

  // Flatten pages into a single list
  const posts = useMemo(
    () => feedQuery.data?.pages.flatMap((page) => page.posts) ?? [],
    [feedQuery.data],
  );

  // Like mutation with optimistic update
  const likeMutation = useMutation({
    mutationFn: (postId: string) => postsApi.toggleLike(postId),
    onMutate: async (postId) => {
      await queryClient.cancelQueries({ queryKey: postKeys.list({ status: 'published' }) });

      const previousData = queryClient.getQueryData(
        postKeys.list({ status: 'published' }),
      );

      // Update the post in all pages
      queryClient.setQueryData(
        postKeys.list({ status: 'published' }),
        (old: any) => {
          if (!old) return old;
          return {
            ...old,
            pages: old.pages.map((page: any) => ({
              ...page,
              posts: page.posts.map((post: Post) =>
                post.id === postId
                  ? {
                      ...post,
                      isLiked: !post.isLiked,
                      likeCount: post.isLiked ? post.likeCount - 1 : post.likeCount + 1,
                    }
                  : post,
              ),
            })),
          };
        },
      );

      return { previousData };
    },
    onError: (_error, _postId, context) => {
      if (context?.previousData) {
        queryClient.setQueryData(
          postKeys.list({ status: 'published' }),
          context.previousData,
        );
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.list({ status: 'published' }) });
    },
  });

  const loadMore = useCallback(() => {
    if (feedQuery.hasNextPage && !feedQuery.isFetchingNextPage) {
      feedQuery.fetchNextPage();
    }
  }, [feedQuery]);

  const toggleLike = useCallback(
    (postId: string) => {
      likeMutation.mutate(postId);
    },
    [likeMutation],
  );

  return {
    posts,
    isLoading: feedQuery.isLoading,
    isRefreshing: feedQuery.isRefetching && !feedQuery.isFetchingNextPage,
    isFetchingMore: feedQuery.isFetchingNextPage,
    hasMore: feedQuery.hasNextPage ?? false,
    error: feedQuery.error,
    refetch: feedQuery.refetch,
    loadMore,
    toggleLike,
  };
}
```

### 10.4 The Component

```tsx
// src/screens/FeedScreen.tsx

import { FlatList, RefreshControl, View, Text, StyleSheet } from 'react-native';
import { useFeed } from '../hooks/useFeed';
import { useNotificationStream } from '../hooks/useNotificationStream';
import { PostCard } from '../components/PostCard';
import { FeedSkeleton } from '../components/FeedSkeleton';
import { ErrorView } from '../components/ErrorView';
import { getUserFacingMessage } from '../api/errors';

export function FeedScreen() {
  const feed = useFeed();

  // Real-time notification updates via SSE
  useNotificationStream();

  if (feed.isLoading) return <FeedSkeleton />;
  if (feed.error && feed.posts.length === 0) {
    return (
      <ErrorView
        message={getUserFacingMessage(feed.error)}
        onRetry={feed.refetch}
      />
    );
  }

  return (
    <FlatList
      data={feed.posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => (
        <PostCard
          post={item}
          onLike={() => feed.toggleLike(item.id)}
        />
      )}
      refreshControl={
        <RefreshControl
          refreshing={feed.isRefreshing}
          onRefresh={feed.refetch}
        />
      }
      onEndReached={feed.loadMore}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        feed.isFetchingMore ? <LoadingFooter /> : null
      }
      ListEmptyComponent={
        <View style={styles.empty}>
          <Text style={styles.emptyText}>No posts yet. Be the first!</Text>
        </View>
      }
      // Performance
      removeClippedSubviews
      maxToRenderPerBatch={10}
      windowSize={5}
    />
  );
}

function LoadingFooter() {
  return (
    <View style={styles.footer}>
      <ActivityIndicator size="small" />
    </View>
  );
}

const styles = StyleSheet.create({
  empty: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 40 },
  emptyText: { fontSize: 16, color: '#666', textAlign: 'center' },
  footer: { padding: 16, alignItems: 'center' },
});
```

---

## 11. COMMON MISTAKES AND HOW TO AVOID THEM

### 11.1 Fetching in useEffect

```tsx
// ❌ BAD: Manual fetching in useEffect
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    usersApi.getById(userId)
      .then((data) => { if (!cancelled) setUser(data); })
      .catch((err) => { if (!cancelled) setError(err); })
      .finally(() => { if (!cancelled) setLoading(false); });
    return () => { cancelled = true; };
  }, [userId]);

  // No caching, no deduplication, no background refetch, no retry,
  // no loading state management, and you wrote all this boilerplate.
}

// ✅ GOOD: TanStack Query
function UserProfile({ userId }: { userId: string }) {
  const { data: user, isLoading, error } = useQuery(userDetailOptions(userId));
  // Cached, deduplicated, retried, background-refreshed, typed, 1 line.
}
```

### 11.2 Not Deduplicating Requests

If three components on the same screen all call `useQuery({ queryKey: ['user', userId] })`, TanStack Query makes ONE network request. If three components each call `fetch('/api/users/' + userId)` in `useEffect`, you get THREE network requests.

### 11.3 Forgetting to Cancel Queries on Unmount

TanStack Query handles this automatically. If the component unmounts before the query resolves, the result is cached but no state update is attempted on the unmounted component.

With manual `useEffect` fetching, you need the `cancelled` flag (shown above), or you get the infamous "Can't perform a React state update on an unmounted component" warning.

### 11.4 Over-Invalidating

```typescript
// ❌ BAD: Nuclear invalidation after every mutation
onSuccess: () => {
  queryClient.invalidateQueries(); // Invalidates EVERYTHING
}

// ✅ GOOD: Targeted invalidation
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: postKeys.lists() });
}
```

### 11.5 Stale Closures in Mutation Callbacks

```tsx
// ❌ BAD: Stale closure over `items`
function ItemList() {
  const [items, setItems] = useState<Item[]>([]);

  const deleteMutation = useMutation({
    mutationFn: (id: string) => itemsApi.delete(id),
    onSuccess: (_, deletedId) => {
      // This captures `items` at the time the mutation was CREATED,
      // not when onSuccess fires. If items changed in between, you lose updates.
      setItems(items.filter((i) => i.id !== deletedId));
    },
  });
}

// ✅ GOOD: Functional update avoids stale closure
onSuccess: (_, deletedId) => {
  setItems((prev) => prev.filter((i) => i.id !== deletedId));
}

// ✅ BETTER: Let TanStack Query manage the cache
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: itemKeys.lists() });
}
```

### 11.6 Not Setting staleTime

The default `staleTime` in TanStack Query is `0`, meaning data is considered stale immediately. Every time a component mounts, a refetch happens. For many queries, this is wasteful.

```typescript
// Set globally for a reasonable default
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
    },
  },
});

// Override per-query for data that changes frequently
useQuery({
  queryKey: ['stock-price', ticker],
  queryFn: () => stockApi.getPrice(ticker),
  staleTime: 10 * 1000, // 10 seconds
  refetchInterval: 10 * 1000, // Poll every 10 seconds
});
```

---

## 12. DECISION FRAMEWORK

When you're starting a new project or refactoring an existing one, use this framework to make your data fetching decisions:

### 12.1 API Protocol

```
Do you control the backend?
├── NO → REST (universal compatibility)
├── YES → Is the backend TypeScript?
│         ├── NO → REST (with OpenAPI codegen) or GraphQL
│         └── YES → Is it a monorepo?
│                   ├── YES → tRPC (the sweet spot)
│                   └── NO → tRPC (with package publishing) or REST
```

### 12.2 Data Fetching Library

```
TanStack Query. That's the answer.

Seriously. Unless you're using a framework that provides its own solution
(Relay for GraphQL, SWR if you're already invested), use TanStack Query.
It works with REST, GraphQL, tRPC, or any async function.
```

### 12.3 Real-Time

```
Do you need bidirectional communication?
├── YES → WebSocket (or Socket.io if you need rooms/namespaces)
└── NO  → Server-Sent Events (simpler, more reliable, HTTP-native)
```

### 12.4 Offline

```
What's your target platform?
├── Web only → Probably don't need offline. Maybe service worker caching.
├── Mobile   → At minimum: TanStack Query persistence for cached reads.
│              If users create content offline: mutation queue.
│              If multi-device sync: sync engine (PowerSync, Legend State).
└── Both     → Build the offline layer in shared code.
               Use MMKV for React Native, localStorage/IndexedDB for web.
```

---

## Summary

This chapter covered the full stack of data fetching and server communication:

1. **API Layer:** Build a typed client with error standardization and token refresh. Keep HTTP concerns in one place.

2. **TanStack Query:** Use query key factories, query options factories, dependent queries with `enabled`, parallel queries, prefetching for instant navigation, and infinite queries for feeds.

3. **Optimistic Updates:** Apply the `onMutate` / `onError` / `onSettled` pattern. Use them for high-frequency, low-stakes operations. Skip them for critical operations.

4. **tRPC:** If you own both ends and use TypeScript, tRPC eliminates codegen entirely. The monorepo setup shares types between web and mobile.

5. **Protocol Choice:** REST for public APIs and simplicity. GraphQL for complex data graphs and multiple clients. tRPC for TypeScript monorepos.

6. **Real-Time:** SSE for server-to-client push (simpler, more reliable). WebSocket for bidirectional communication (chat, collaboration).

7. **Pagination:** Cursor-based for mobile (stable, performant). Offset-based when you need page numbers.

8. **Retry:** Exponential backoff with jitter. Never retry client errors. Use idempotency keys for non-idempotent operations.

9. **Offline:** Level 1 (cached reads) is almost free with TanStack Query persistence. Level 2 (mutation queues) requires explicit design. Level 3 (conflict resolution) is hard -- use a sync engine.

The patterns in this chapter compose. A cursor-paginated infinite query with optimistic likes, offline persistence, and real-time updates via SSE is not a Frankenstein monster -- it's the standard architecture for a modern mobile feed. Each piece has a clear responsibility, and TanStack Query is the orchestration layer that ties them together.

Next up in Chapter 11, we'll dive into caching strategies -- how the data fetching patterns here interact with HTTP caching, CDN caching, and application-level caching to minimize network requests and maximize performance.
