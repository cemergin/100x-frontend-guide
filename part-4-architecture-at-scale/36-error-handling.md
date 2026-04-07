<!--
  CHAPTER: 36
  TITLE: Error Handling Architecture — Graceful Failures Everywhere
  PART: IV — Architecture at Scale
  PREREQS: Chapters 10, 20
  KEY_TOPICS: error boundaries, API error standardization, user-facing error UX, retry patterns, graceful degradation, error reporting, toast notifications, error codes, network errors, timeout handling, dead letter patterns
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 36: Error Handling Architecture — Graceful Failures Everywhere

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 10, 20 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Errors are not exceptional -- they are a certainty. Your network WILL fail, your API WILL 500, your code WILL throw. Design for it from day one, not as a hotfix after production breaks.
- Build a layered error handling architecture: component-level catches for granular recovery, screen-level boundaries for section isolation, app-level boundaries as the last resort -- never show a white screen.
- Standardize your API error format: every error response should include `{ code, message, details, requestId }` so the client can make consistent decisions about display, retry, and reporting.
- Error boundaries are your React safety net, but they only catch render errors -- you still need try/catch for async operations, event handlers, and effects.
- Retry with exponential backoff and jitter; never retry 401s; always retry 503s; use idempotency keys for mutations so retries do not create duplicates.
- Graceful degradation means showing cached data, hiding broken features, and suggesting alternatives -- never a white screen, never a raw stack trace, never an unhelpful "Something went wrong."
- Report everything to Sentry/Crashlytics with context: user ID, route, device info, breadcrumbs. The error report is useless without the context to reproduce it.
- Dead letter patterns handle mutations that cannot be retried: log them, alert on them, and provide manual intervention for the user.

</details>

Here is a story I hear at least twice a year. A team ships a feature. It works perfectly in development. It works in staging. It ships to production. Two days later, the CEO forwards an email from an angry customer: "Your app shows a white screen when I try to check out." The team investigates. A third-party API returned an unexpected response format. One field was `null` instead of an empty string. A `.length` call threw. React's render cycle caught the error, but there was no error boundary. The entire app unmounted. White screen.

The fix was a one-line null check. The damage was a lost customer, a frantic weekend, and a team that now has trust issues with their own code.

**This chapter exists so that story never happens to you.**

Error handling is not a feature. It is not a nice-to-have. It is not something you add in "the polish phase." It is the foundation that determines whether your application is production-grade or a prototype pretending to be one. The difference between a 1x developer and a 100x architect is not that the 100x architect writes code that never fails -- it is that the 100x architect writes code that fails *gracefully*, *visibly*, and *recoverably*.

### In This Chapter
- The Error Handling Philosophy (errors are expected, not exceptional)
- Error Taxonomy (network, HTTP, JavaScript, React, native -- each needs different handling)
- Error Boundaries (granular placement, fallback UIs, recovery patterns)
- API Error Standardization (consistent format, error codes, field-level mapping)
- Network Error Handling (offline detection, retry, backoff, circuit breakers)
- User-Facing Error UX (toasts, inline errors, full-screen states, decision trees)
- Global Error Handlers (unhandled rejections, native crash handlers, Sentry integration)
- Graceful Degradation (cached data, feature hiding, progressive failure)
- Dead Letter Patterns (failed mutations, alerting, manual intervention)
- The Complete Error Architecture (layered handling from component to reporting)

### Related Chapters
- [Ch 10: Data Fetching & Server Communication] -- API error handling, retry, and the typed client
- [Ch 35: Forms at Scale] -- form-specific error patterns (server errors, field-level mapping)
- [Ch 20: Codebase Management] -- where error handling modules live in your architecture
- [Ch 12: Offline-First & Real-Time] -- offline error handling and mutation queues

---

## 1. THE ERROR HANDLING PHILOSOPHY

Let me be blunt: **if your error handling strategy is "try/catch and console.error," you do not have an error handling strategy.** You have a prayer.

Production applications face a constant barrage of failures:

- The user's phone loses signal in an elevator
- Your CDN returns a stale response
- A database migration breaks an API endpoint for 45 seconds
- A third-party auth provider returns a 503
- The user's browser has an extension that mutates the DOM and breaks your React tree
- The user submits a form and the server validates differently than the client
- A background sync fires when the app is in a stale state
- The user's device runs out of memory

Each of these failures requires a different response. Some need a retry. Some need a fallback. Some need user communication. Some need silent recovery. Some need all four. The architecture you build needs to handle all of them without bespoke code for each case.

### 1.1 The Three Principles

**Principle 1: Errors are expected, not exceptional.** Design your happy path and your error path with equal care. If you have a loading state, you need an error state. If you have a submit handler, you need a failure handler. No exceptions.

**Principle 2: The user should always know what happened and what to do next.** "Something went wrong" is not an error message. "We couldn't save your changes because our servers are temporarily unavailable. Your changes have been saved locally and will sync when the connection is restored." -- that is an error message.

**Principle 3: The developer should always be able to reproduce the failure.** Every error report needs context: what the user was doing, what the request looked like, what the response was, what the app state was. An error report that says "TypeError: Cannot read properties of undefined" with no context is useless.

### 1.2 The Cost of Bad Error Handling

```
┌─────────────────────────────────────────────────────────────┐
│              THE COST OF BAD ERROR HANDLING                   │
│                                                              │
│  WHITE SCREEN (no error boundary)                           │
│  ├── User sees nothing                                      │
│  ├── Cannot recover without refresh                         │
│  ├── 100% loss of user trust                                │
│  └── Cost: users leave, never come back                     │
│                                                              │
│  SILENT FAILURE (catch + console.error)                     │
│  ├── User thinks action succeeded                           │
│  ├── Data is lost without user knowing                      │
│  ├── Corruption compounds over time                         │
│  └── Cost: data integrity, support tickets                  │
│                                                              │
│  GENERIC ERROR ("Something went wrong")                     │
│  ├── User knows something failed                            │
│  ├── User does not know what to do                          │
│  ├── User tries random things (refresh, re-submit)          │
│  └── Cost: duplicate submissions, frustrated users          │
│                                                              │
│  RAW ERROR (stack trace shown to user)                      │
│  ├── User sees technical gibberish                          │
│  ├── Security risk (exposes internals)                      │
│  ├── User loses confidence in the product                   │
│  └── Cost: trust, security                                  │
│                                                              │
│  GRACEFUL ERROR (proper handling)                           │
│  ├── User knows what happened                               │
│  ├── User knows what to do next                             │
│  ├── App remains usable                                     │
│  ├── Developer gets full context to fix it                  │
│  └── Cost: engineering time upfront (but saves 10x later)   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. ERROR TAXONOMY

Not all errors are created equal. Each category requires different detection, different handling, and different communication.

### 2.1 Network Errors

These happen when the request never reaches the server or the response never reaches the client.

```typescript
// types/errors.ts

export class NetworkError extends Error {
  public readonly isOffline: boolean;
  public readonly isTimeout: boolean;

  constructor(
    message: string,
    options: { isOffline?: boolean; isTimeout?: boolean } = {},
  ) {
    super(message);
    this.name = 'NetworkError';
    this.isOffline = options.isOffline ?? false;
    this.isTimeout = options.isTimeout ?? false;
  }
}

// Detection
function classifyNetworkError(error: unknown): NetworkError {
  if (error instanceof TypeError && error.message === 'Failed to fetch') {
    // Could be offline, CORS, DNS failure, or connection refused
    const isOffline = typeof navigator !== 'undefined' && !navigator.onLine;
    return new NetworkError(
      isOffline ? 'You appear to be offline' : 'Unable to reach the server',
      { isOffline },
    );
  }

  if (error instanceof DOMException && error.name === 'AbortError') {
    return new NetworkError('Request timed out', { isTimeout: true });
  }

  return new NetworkError('Network request failed');
}
```

**Handling strategy:**
- **Offline:** Show an offline banner, queue mutations for later replay, serve cached data
- **Timeout:** Retry with exponential backoff (the server might be slow, not dead)
- **DNS/Connection refused:** The server is down -- show a maintenance message, do not retry immediately

### 2.2 HTTP Errors

The request reached the server and came back with an error status.

```typescript
// types/errors.ts

export class ApiError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,
    public readonly details?: Record<string, unknown>,
    public readonly requestId?: string,
  ) {
    super(`API Error [${status}]: ${code}`);
    this.name = 'ApiError';
  }

  get isClientError() { return this.status >= 400 && this.status < 500; }
  get isServerError() { return this.status >= 500; }
  get isRetryable() {
    // 429: rate limited, 502-504: gateway errors (usually transient)
    return [429, 502, 503, 504].includes(this.status);
  }
  get isAuthError() { return [401, 403].includes(this.status); }
  get isValidationError() { return this.status === 400 || this.status === 422; }
  get isNotFound() { return this.status === 404; }
}
```

**Handling decision tree:**

```
HTTP Error received
├── 400/422 (Validation Error)
│   └── Show field-level errors inline (see Chapter 35)
├── 401 (Unauthorized)
│   └── Refresh token → retry once → redirect to login
├── 403 (Forbidden)
│   └── Show "access denied" message → suggest contacting admin
├── 404 (Not Found)
│   └── Show "not found" page → suggest navigation options
├── 409 (Conflict)
│   └── Show conflict resolution UI → let user choose
├── 429 (Rate Limited)
│   └── Retry after Retry-After header → show "please wait"
├── 500 (Internal Server Error)
│   └── Show generic error → report to Sentry → do not retry
├── 502/503/504 (Gateway/Service Unavailable)
│   └── Retry with backoff → show "temporarily unavailable"
└── Unknown status
    └── Log and report → show generic error
```

### 2.3 JavaScript Errors

Runtime errors in your code: `TypeError`, `ReferenceError`, `RangeError`, etc.

```typescript
// These are bugs. They should not happen in production.
// But they will. The question is what happens next.

// TypeError: accessing property of undefined
const user = undefined;
user.name; // TypeError: Cannot read properties of undefined

// RangeError: array bounds
const arr = [1, 2, 3];
arr.length = -1; // RangeError

// SyntaxError (in eval or dynamic code)
JSON.parse('not json'); // SyntaxError
```

**Handling strategy:** These are bugs. Report them to Sentry immediately with full context. Show an error boundary fallback. Never silently swallow them.

### 2.4 React Render Errors

Errors that occur during rendering, in lifecycle methods, or in constructors of class components. These are the ones that cause white screens without error boundaries.

```tsx
// This will crash the entire app without an error boundary
function UserProfile({ user }: { user: User }) {
  // If user.address is null, this crashes during render
  return <p>{user.address.street}</p>;
}
```

**Handling strategy:** Error boundaries (Section 3).

### 2.5 Native Errors (React Native)

In React Native, you also deal with native-layer errors:

```typescript
// Native crash (SIGABRT, SIGSEGV)
// These are usually from native modules or OOM conditions
// You cannot catch these in JavaScript -- use Crashlytics

// JavaScript exception in native callback
// These can be caught with ErrorUtils.setGlobalHandler

// Out of memory
// Nothing you can do except prevent it: virtualize lists,
// release image caches, monitor memory usage
```

---

## 3. ERROR BOUNDARIES

Error boundaries are React's mechanism for catching errors during rendering. Without them, a single error in any component unmounts the entire React tree. White screen.

### 3.1 The Base Error Boundary

React error boundaries must be class components (as of React 19, there is no hook equivalent for `componentDidCatch`):

```tsx
// components/ErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from 'react';

interface ErrorBoundaryProps {
  children: ReactNode;
  fallback?: ReactNode | ((error: Error, reset: () => void) => ReactNode);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  level?: 'app' | 'screen' | 'section' | 'component';
}

interface ErrorBoundaryState {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Report to error tracking service
    this.props.onError?.(error, errorInfo);

    // Default reporting
    reportError(error, {
      componentStack: errorInfo.componentStack,
      level: this.props.level ?? 'component',
    });
  }

  reset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      const { fallback } = this.props;
      const { error } = this.state;

      if (typeof fallback === 'function') {
        return fallback(error!, this.reset);
      }

      if (fallback) {
        return fallback;
      }

      // Default fallback
      return <DefaultErrorFallback error={error!} onReset={this.reset} />;
    }

    return this.props.children;
  }
}

function DefaultErrorFallback({
  error,
  onReset,
}: {
  error: Error;
  onReset: () => void;
}) {
  return (
    <div role="alert" className="error-fallback">
      <h2>Something went wrong</h2>
      <p>We have been notified and are looking into it.</p>
      <button onClick={onReset}>Try Again</button>
      {process.env.NODE_ENV === 'development' && (
        <pre className="error-details">{error.message}</pre>
      )}
    </div>
  );
}
```

### 3.2 Granular Error Boundary Placement

The key insight: **place error boundaries at multiple levels**. If a sidebar widget crashes, only the widget should show an error fallback -- not the entire page.

```tsx
// app/layout.tsx -- App-level boundary (last resort)
export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html>
      <body>
        <ErrorBoundary
          level="app"
          fallback={(error, reset) => (
            <AppCrashScreen error={error} onReset={reset} />
          )}
        >
          <Providers>
            {children}
          </Providers>
        </ErrorBoundary>
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx -- Screen-level boundary
export default function DashboardLayout({ children }: { children: ReactNode }) {
  return (
    <div className="dashboard-layout">
      <Sidebar />
      <ErrorBoundary
        level="screen"
        fallback={(error, reset) => (
          <ScreenErrorState error={error} onRetry={reset} />
        )}
      >
        <main>{children}</main>
      </ErrorBoundary>
    </div>
  );
}

// components/Dashboard.tsx -- Section-level boundaries
function Dashboard() {
  return (
    <div className="dashboard-grid">
      <ErrorBoundary
        level="section"
        fallback={<WidgetErrorCard title="Revenue" />}
      >
        <RevenueChart />
      </ErrorBoundary>

      <ErrorBoundary
        level="section"
        fallback={<WidgetErrorCard title="Recent Orders" />}
      >
        <RecentOrders />
      </ErrorBoundary>

      <ErrorBoundary
        level="section"
        fallback={<WidgetErrorCard title="User Activity" />}
      >
        <UserActivityFeed />
      </ErrorBoundary>
    </div>
  );
}

function WidgetErrorCard({ title }: { title: string }) {
  return (
    <div className="widget-card widget-error" role="alert">
      <h3>{title}</h3>
      <p>This section could not be loaded.</p>
      <button onClick={() => window.location.reload()}>
        Reload page
      </button>
    </div>
  );
}
```

### 3.3 Error Boundary Placement Strategy

```
┌─────────────────────────────────────────────────────────────┐
│           ERROR BOUNDARY PLACEMENT STRATEGY                  │
│                                                              │
│  LEVEL 1: APP ROOT                                          │
│  ├── Catches: catastrophic failures                         │
│  ├── Fallback: full-screen crash page with reload button    │
│  ├── Reports: always (this is a critical error)             │
│  └── Example: provider initialization failure               │
│                                                              │
│  LEVEL 2: ROUTE/SCREEN                                      │
│  ├── Catches: page-level render failures                    │
│  ├── Fallback: error page with navigation options           │
│  ├── Reports: always                                        │
│  └── Example: data fetching component throws during render  │
│                                                              │
│  LEVEL 3: SECTION                                           │
│  ├── Catches: independent section failures                  │
│  ├── Fallback: placeholder card with retry button           │
│  ├── Reports: yes, but lower priority                       │
│  └── Example: chart library throws on bad data              │
│                                                              │
│  LEVEL 4: COMPONENT                                         │
│  ├── Catches: individual component failures                 │
│  ├── Fallback: hidden or minimal placeholder                │
│  ├── Reports: yes, but batched                              │
│  └── Example: third-party embed fails to load               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 What Error Boundaries Do NOT Catch

This is critical. Error boundaries catch errors during:
- Rendering (return statement)
- Lifecycle methods
- Constructors of child components

Error boundaries do **NOT** catch errors in:
- Event handlers (onClick, onSubmit, etc.)
- Async code (setTimeout, promises, async/await)
- Server-side rendering
- Errors in the error boundary itself

For everything error boundaries do not catch, you need explicit try/catch:

```tsx
function SubmitButton() {
  const [error, setError] = useState<Error | null>(null);

  const handleClick = async () => {
    try {
      setError(null);
      await api.submit();
    } catch (err) {
      // Error boundaries won't catch this -- it's an event handler
      setError(err instanceof Error ? err : new Error('Submit failed'));
      reportError(err);
    }
  };

  return (
    <div>
      <button onClick={handleClick}>Submit</button>
      {error && (
        <p role="alert" className="text-red-600">
          {error.message}
        </p>
      )}
    </div>
  );
}
```

### 3.5 The Query Error Boundary Pattern

TanStack Query has first-class error boundary support. This is the cleanest pattern for data fetching errors:

```tsx
// components/UserProfile.tsx
import { useSuspenseQuery } from '@tanstack/react-query';

// This component uses Suspense for loading and ErrorBoundary for errors
function UserProfileContent({ userId }: { userId: string }) {
  const { data: user } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
  });

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Parent wraps with both boundaries
function UserProfilePage({ userId }: { userId: string }) {
  return (
    <ErrorBoundary
      fallback={(error, reset) => (
        <UserProfileError error={error} onRetry={reset} />
      )}
    >
      <Suspense fallback={<UserProfileSkeleton />}>
        <UserProfileContent userId={userId} />
      </Suspense>
    </ErrorBoundary>
  );
}

function UserProfileError({
  error,
  onRetry,
}: {
  error: Error;
  onRetry: () => void;
}) {
  if (error instanceof ApiError && error.isNotFound) {
    return (
      <div className="not-found">
        <h2>User not found</h2>
        <p>This user may have been deleted or the link may be incorrect.</p>
        <Link href="/users">Browse all users</Link>
      </div>
    );
  }

  return (
    <div className="error-state">
      <h2>Could not load user profile</h2>
      <p>Please try again or contact support if the problem persists.</p>
      <button onClick={onRetry}>Retry</button>
    </div>
  );
}
```

### 3.6 Resettable Error Boundaries with Route Changes

Error boundaries persist their error state. If the user navigates to a different page and back, the error boundary still shows the fallback. You need to reset it on route changes:

```tsx
// components/RouteErrorBoundary.tsx
import { usePathname } from 'next/navigation';
import { useEffect, useRef } from 'react';

export function RouteErrorBoundary({ children }: { children: ReactNode }) {
  const pathname = usePathname();
  const boundaryRef = useRef<ErrorBoundary>(null);

  // Reset error boundary when the route changes
  useEffect(() => {
    boundaryRef.current?.reset();
  }, [pathname]);

  return (
    <ErrorBoundary ref={boundaryRef} level="screen">
      {children}
    </ErrorBoundary>
  );
}
```

In Next.js App Router, you also get the built-in `error.tsx` convention:

```tsx
// app/dashboard/error.tsx
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Report to Sentry
    reportError(error);
  }, [error]);

  return (
    <div className="error-page">
      <h2>Dashboard Error</h2>
      <p>Something went wrong loading the dashboard.</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

---

## 4. API ERROR STANDARDIZATION

Every API your frontend calls should return errors in the same format. Without this, every component that handles errors needs bespoke parsing logic. That is a maintenance nightmare.

### 4.1 The Standard Error Response

```typescript
// shared/types/api-error.ts

/**
 * Every API error response MUST conform to this shape.
 * This is a contract between frontend and backend.
 */
export interface ApiErrorResponse {
  /** Machine-readable error code (e.g., 'VALIDATION_ERROR', 'NOT_FOUND') */
  code: string;

  /** Human-readable message (can be shown to users for known codes) */
  message: string;

  /** Additional context (field errors, rate limit info, etc.) */
  details?: {
    /** Field-level validation errors */
    fieldErrors?: Record<string, string>;
    /** When the user can retry (for rate limiting) */
    retryAfter?: number;
    /** For conflict errors: the current server version */
    serverVersion?: number;
    /** Any additional context */
    [key: string]: unknown;
  };

  /** Unique request ID for tracing in logs */
  requestId: string;
}
```

### 4.2 Error Codes Enum

```typescript
// shared/constants/error-codes.ts

export const ErrorCode = {
  // Authentication
  UNAUTHORIZED: 'UNAUTHORIZED',
  TOKEN_EXPIRED: 'TOKEN_EXPIRED',
  INVALID_CREDENTIALS: 'INVALID_CREDENTIALS',
  ACCOUNT_LOCKED: 'ACCOUNT_LOCKED',

  // Authorization
  FORBIDDEN: 'FORBIDDEN',
  INSUFFICIENT_PERMISSIONS: 'INSUFFICIENT_PERMISSIONS',

  // Validation
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_INPUT: 'INVALID_INPUT',

  // Resources
  NOT_FOUND: 'NOT_FOUND',
  ALREADY_EXISTS: 'ALREADY_EXISTS',
  VERSION_CONFLICT: 'VERSION_CONFLICT',

  // Rate limiting
  RATE_LIMITED: 'RATE_LIMITED',

  // Server
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',
  MAINTENANCE: 'MAINTENANCE',

  // Business logic
  INSUFFICIENT_BALANCE: 'INSUFFICIENT_BALANCE',
  SUBSCRIPTION_EXPIRED: 'SUBSCRIPTION_EXPIRED',
  FEATURE_DISABLED: 'FEATURE_DISABLED',
} as const;

export type ErrorCode = (typeof ErrorCode)[keyof typeof ErrorCode];
```

### 4.3 Mapping Error Codes to User-Facing Messages

```typescript
// utils/error-messages.ts

const ERROR_MESSAGES: Record<string, string> = {
  [ErrorCode.UNAUTHORIZED]: 'Please log in to continue.',
  [ErrorCode.TOKEN_EXPIRED]: 'Your session has expired. Please log in again.',
  [ErrorCode.INVALID_CREDENTIALS]: 'Invalid email or password.',
  [ErrorCode.ACCOUNT_LOCKED]:
    'Your account has been locked. Please contact support.',
  [ErrorCode.FORBIDDEN]:
    'You do not have permission to perform this action.',
  [ErrorCode.INSUFFICIENT_PERMISSIONS]:
    'You need additional permissions. Contact your admin.',
  [ErrorCode.VALIDATION_ERROR]: 'Please check your input and try again.',
  [ErrorCode.NOT_FOUND]: 'The requested resource was not found.',
  [ErrorCode.ALREADY_EXISTS]: 'This resource already exists.',
  [ErrorCode.VERSION_CONFLICT]:
    'This resource was modified by someone else. Please refresh and try again.',
  [ErrorCode.RATE_LIMITED]:
    'Too many requests. Please wait a moment and try again.',
  [ErrorCode.INTERNAL_ERROR]:
    'Something went wrong on our end. We have been notified.',
  [ErrorCode.SERVICE_UNAVAILABLE]:
    'Our service is temporarily unavailable. Please try again in a few minutes.',
  [ErrorCode.MAINTENANCE]:
    'We are currently undergoing maintenance. Please check back shortly.',
  [ErrorCode.INSUFFICIENT_BALANCE]: 'Insufficient balance for this transaction.',
  [ErrorCode.SUBSCRIPTION_EXPIRED]:
    'Your subscription has expired. Please renew to continue.',
  [ErrorCode.FEATURE_DISABLED]:
    'This feature is not available on your current plan.',
};

const FALLBACK_MESSAGE = 'An unexpected error occurred. Please try again.';

export function getUserFacingMessage(error: unknown): string {
  if (error instanceof ApiError) {
    return ERROR_MESSAGES[error.code] ?? error.message ?? FALLBACK_MESSAGE;
  }

  if (error instanceof NetworkError) {
    if (error.isOffline) return 'You appear to be offline. Please check your connection.';
    if (error.isTimeout) return 'The request took too long. Please try again.';
    return 'Unable to connect to the server. Please try again.';
  }

  return FALLBACK_MESSAGE;
}
```

### 4.4 API Client Error Parsing

In Chapter 10, we built a typed API client. Here is how error parsing fits in:

```typescript
// api/client.ts

async function request<T>(method: string, path: string, config: RequestConfig = {}): Promise<T> {
  const { timeout = 30_000, ...init } = config;
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(url, {
      method,
      signal: controller.signal,
      ...init,
    });

    clearTimeout(timeoutId);

    if (!response.ok) {
      // Parse the standardized error response
      let errorBody: ApiErrorResponse;
      try {
        errorBody = await response.json();
      } catch {
        // Response is not JSON -- create a synthetic error
        errorBody = {
          code: 'UNKNOWN',
          message: response.statusText,
          requestId: response.headers.get('x-request-id') ?? 'unknown',
        };
      }

      throw new ApiError(
        response.status,
        errorBody.code,
        errorBody.details,
        errorBody.requestId,
      );
    }

    return response.json();
  } catch (error) {
    clearTimeout(timeoutId);

    // Already an ApiError -- rethrow
    if (error instanceof ApiError) throw error;

    // Network-level error -- classify it
    throw classifyNetworkError(error);
  }
}
```

### 4.5 Server-Side: Consistent Error Responses

Your API must always return the standard format, even for framework-level errors:

```typescript
// middleware/error-handler.ts (Express example)
import { ZodError } from 'zod';
import { v4 as uuid } from 'uuid';

export function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  const requestId = req.headers['x-request-id'] as string ?? uuid();

  // Zod validation errors
  if (err instanceof ZodError) {
    const fieldErrors: Record<string, string> = {};
    for (const issue of err.issues) {
      const field = issue.path.join('.');
      if (!fieldErrors[field]) {
        fieldErrors[field] = issue.message;
      }
    }

    return res.status(400).json({
      code: 'VALIDATION_ERROR',
      message: 'Validation failed',
      details: { fieldErrors },
      requestId,
    });
  }

  // Known business errors
  if (err instanceof BusinessError) {
    return res.status(err.statusCode).json({
      code: err.code,
      message: err.message,
      details: err.details,
      requestId,
    });
  }

  // Unknown errors -- never leak internals
  console.error(`[${requestId}] Unhandled error:`, err);

  return res.status(500).json({
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred',
    requestId,
  });
}
```

---

## 5. NETWORK ERROR HANDLING

Network errors are the most common class of errors in mobile and web applications. Your users are on subways, in elevators, on slow hotel Wi-Fi, and behind corporate proxies that randomly drop connections.

### 5.1 Offline Detection

```typescript
// hooks/useNetworkStatus.ts
import { useState, useEffect, useCallback } from 'react';
// For React Native, use @react-native-community/netinfo instead

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(
    typeof navigator !== 'undefined' ? navigator.onLine : true,
  );

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return { isOnline };
}

// React Native version
import NetInfo from '@react-native-community/netinfo';

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(true);
  const [connectionType, setConnectionType] = useState<string>('unknown');

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsOnline(state.isConnected ?? true);
      setConnectionType(state.type);
    });

    return unsubscribe;
  }, []);

  return { isOnline, connectionType };
}
```

### 5.2 Offline Banner Component

```tsx
// components/OfflineBanner.tsx
export function OfflineBanner() {
  const { isOnline } = useNetworkStatus();
  const [wasOffline, setWasOffline] = useState(false);
  const [showRestored, setShowRestored] = useState(false);

  useEffect(() => {
    if (!isOnline) {
      setWasOffline(true);
    } else if (wasOffline) {
      setShowRestored(true);
      const timer = setTimeout(() => {
        setShowRestored(false);
        setWasOffline(false);
      }, 3000);
      return () => clearTimeout(timer);
    }
  }, [isOnline, wasOffline]);

  if (!isOnline) {
    return (
      <div
        role="alert"
        className="fixed top-0 left-0 right-0 bg-red-600 text-white text-center py-2 z-50"
      >
        You are offline. Changes will be saved when your connection is restored.
      </div>
    );
  }

  if (showRestored) {
    return (
      <div
        role="status"
        className="fixed top-0 left-0 right-0 bg-green-600 text-white text-center py-2 z-50"
      >
        Connection restored. Syncing your changes...
      </div>
    );
  }

  return null;
}
```

### 5.3 Retry with Exponential Backoff and Jitter

```typescript
// utils/retry.ts

interface RetryOptions {
  maxRetries: number;
  baseDelayMs: number;
  maxDelayMs: number;
  shouldRetry?: (error: unknown, attempt: number) => boolean;
  onRetry?: (error: unknown, attempt: number, delayMs: number) => void;
}

const DEFAULT_OPTIONS: RetryOptions = {
  maxRetries: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30000,
  shouldRetry: (error) => {
    // Never retry client errors (except 429)
    if (error instanceof ApiError) {
      return error.isRetryable;
    }
    // Always retry network errors (except offline -- no point)
    if (error instanceof NetworkError) {
      return !error.isOffline;
    }
    // Don't retry unknown errors
    return false;
  },
};

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {},
): Promise<T> {
  const opts = { ...DEFAULT_OPTIONS, ...options };

  let lastError: unknown;

  for (let attempt = 0; attempt <= opts.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Check if we should retry
      if (attempt >= opts.maxRetries || !opts.shouldRetry?.(error, attempt)) {
        throw error;
      }

      // Calculate delay with exponential backoff + jitter
      const exponentialDelay = opts.baseDelayMs * Math.pow(2, attempt);
      const jitter = Math.random() * opts.baseDelayMs; // Add randomness
      const delay = Math.min(exponentialDelay + jitter, opts.maxDelayMs);

      // Handle 429 Retry-After header
      if (error instanceof ApiError && error.status === 429 && error.details?.retryAfter) {
        const retryAfterMs = (error.details.retryAfter as number) * 1000;
        opts.onRetry?.(error, attempt, retryAfterMs);
        await sleep(retryAfterMs);
      } else {
        opts.onRetry?.(error, attempt, delay);
        await sleep(delay);
      }
    }
  }

  throw lastError;
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

### 5.4 Circuit Breaker Pattern

When an endpoint is repeatedly failing, stop hammering it. A circuit breaker prevents cascading failures:

```typescript
// utils/circuit-breaker.ts

type CircuitState = 'closed' | 'open' | 'half-open';

interface CircuitBreakerOptions {
  failureThreshold: number;  // Failures before opening
  resetTimeMs: number;       // Time before trying again
  halfOpenRequests: number;  // Requests allowed in half-open state
}

export class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failureCount = 0;
  private lastFailureTime = 0;
  private halfOpenSuccesses = 0;

  constructor(
    private name: string,
    private options: CircuitBreakerOptions = {
      failureThreshold: 5,
      resetTimeMs: 30000,
      halfOpenRequests: 2,
    },
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // Check if circuit should transition from open to half-open
    if (this.state === 'open') {
      const timeSinceLastFailure = Date.now() - this.lastFailureTime;
      if (timeSinceLastFailure >= this.options.resetTimeMs) {
        this.state = 'half-open';
        this.halfOpenSuccesses = 0;
      } else {
        throw new CircuitBreakerError(
          `Circuit breaker "${this.name}" is open. Try again in ${
            Math.ceil((this.options.resetTimeMs - timeSinceLastFailure) / 1000)
          } seconds.`,
        );
      }
    }

    try {
      const result = await fn();

      // Success
      if (this.state === 'half-open') {
        this.halfOpenSuccesses++;
        if (this.halfOpenSuccesses >= this.options.halfOpenRequests) {
          // Enough successes in half-open -- close the circuit
          this.state = 'closed';
          this.failureCount = 0;
        }
      } else {
        this.failureCount = 0; // Reset on success
      }

      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.options.failureThreshold) {
        this.state = 'open';
        console.warn(
          `Circuit breaker "${this.name}" opened after ${this.failureCount} failures`,
        );
      }

      throw error;
    }
  }

  getState(): CircuitState {
    return this.state;
  }

  reset(): void {
    this.state = 'closed';
    this.failureCount = 0;
    this.halfOpenSuccesses = 0;
  }
}

export class CircuitBreakerError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'CircuitBreakerError';
  }
}
```

Usage:

```typescript
// api/services.ts
const paymentCircuit = new CircuitBreaker('payment-api', {
  failureThreshold: 3,
  resetTimeMs: 60000, // 1 minute
  halfOpenRequests: 1,
});

export async function processPayment(data: PaymentData) {
  try {
    return await paymentCircuit.execute(() => api.post('/payments', data));
  } catch (error) {
    if (error instanceof CircuitBreakerError) {
      // Payment service is down -- show a specific message
      throw new ApiError(503, 'SERVICE_UNAVAILABLE', {
        message: 'Payment processing is temporarily unavailable. Please try again shortly.',
      });
    }
    throw error;
  }
}
```

### 5.5 Mutation Queue for Offline

When the user is offline and performs a mutation, queue it for replay when the connection is restored:

```typescript
// utils/mutation-queue.ts

interface QueuedMutation {
  id: string;
  endpoint: string;
  method: 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  body: unknown;
  timestamp: number;
  retryCount: number;
  idempotencyKey: string;
}

const QUEUE_STORAGE_KEY = 'offline-mutation-queue';

class MutationQueue {
  private queue: QueuedMutation[] = [];
  private processing = false;

  constructor() {
    this.loadFromStorage();
    this.setupConnectivityListener();
  }

  private loadFromStorage() {
    try {
      const stored = localStorage.getItem(QUEUE_STORAGE_KEY);
      this.queue = stored ? JSON.parse(stored) : [];
    } catch {
      this.queue = [];
    }
  }

  private saveToStorage() {
    try {
      localStorage.setItem(QUEUE_STORAGE_KEY, JSON.stringify(this.queue));
    } catch {
      // Storage full -- oldest items will be lost
    }
  }

  private setupConnectivityListener() {
    if (typeof window !== 'undefined') {
      window.addEventListener('online', () => this.processQueue());
    }
  }

  enqueue(mutation: Omit<QueuedMutation, 'id' | 'timestamp' | 'retryCount'>): string {
    const id = crypto.randomUUID();
    this.queue.push({
      ...mutation,
      id,
      timestamp: Date.now(),
      retryCount: 0,
    });
    this.saveToStorage();
    return id;
  }

  async processQueue(): Promise<void> {
    if (this.processing || this.queue.length === 0) return;

    this.processing = true;

    const toProcess = [...this.queue];

    for (const mutation of toProcess) {
      try {
        await fetch(mutation.endpoint, {
          method: mutation.method,
          headers: {
            'Content-Type': 'application/json',
            'Idempotency-Key': mutation.idempotencyKey,
          },
          body: JSON.stringify(mutation.body),
        });

        // Success -- remove from queue
        this.queue = this.queue.filter((m) => m.id !== mutation.id);
        this.saveToStorage();
      } catch (error) {
        mutation.retryCount++;

        if (mutation.retryCount >= 5) {
          // Move to dead letter queue
          this.moveToDeadLetter(mutation);
          this.queue = this.queue.filter((m) => m.id !== mutation.id);
        }

        // Stop processing if we're offline again
        if (error instanceof NetworkError && error.isOffline) {
          break;
        }
      }
    }

    this.processing = false;
    this.saveToStorage();
  }

  private moveToDeadLetter(mutation: QueuedMutation) {
    const deadLetters = JSON.parse(
      localStorage.getItem('dead-letter-queue') ?? '[]',
    );
    deadLetters.push({
      ...mutation,
      movedAt: Date.now(),
      reason: 'max_retries_exceeded',
    });
    localStorage.setItem('dead-letter-queue', JSON.stringify(deadLetters));

    // Notify the error reporting service
    reportError(new Error(`Mutation moved to dead letter queue: ${mutation.endpoint}`), {
      mutation,
      level: 'warning',
    });
  }

  getPendingCount(): number {
    return this.queue.length;
  }

  getDeadLetterCount(): number {
    const deadLetters = JSON.parse(
      localStorage.getItem('dead-letter-queue') ?? '[]',
    );
    return deadLetters.length;
  }
}

export const mutationQueue = new MutationQueue();
```

---

## 6. USER-FACING ERROR UX

The most important thing about error handling is what the user sees. Every error needs a response that tells the user: (1) what happened, (2) whether it is their fault or yours, and (3) what they should do next.

### 6.1 Toast Notifications

Toasts are for transient, non-blocking errors that do not require user action:

```typescript
// utils/toast.ts
// Using sonner (lightweight, composable toast library for React)

import { toast } from 'sonner';

export const showToast = {
  success(message: string) {
    toast.success(message);
  },

  error(message: string, options?: { action?: { label: string; onClick: () => void } }) {
    toast.error(message, {
      duration: 6000, // Errors stay longer
      action: options?.action,
    });
  },

  networkError() {
    toast.error('Unable to connect to the server. Please check your connection.', {
      duration: Infinity, // Don't auto-dismiss network errors
      id: 'network-error', // Prevent duplicate toasts
      action: {
        label: 'Retry',
        onClick: () => window.location.reload(),
      },
    });
  },

  saveFailed(onRetry: () => void) {
    toast.error('Failed to save your changes.', {
      action: {
        label: 'Retry',
        onClick: onRetry,
      },
    });
  },

  saveSuccess() {
    toast.success('Changes saved successfully.');
  },
};
```

For React Native, use a similar pattern with a toast library like `react-native-toast-message` or `burnt`:

```typescript
// utils/toast.native.ts
import Toast from 'react-native-toast-message';

export const showToast = {
  success(message: string) {
    Toast.show({
      type: 'success',
      text1: message,
      visibilityTime: 3000,
    });
  },

  error(message: string) {
    Toast.show({
      type: 'error',
      text1: 'Error',
      text2: message,
      visibilityTime: 6000,
    });
  },
};
```

### 6.2 Inline Errors

For form validation and field-level errors, always show the error next to the relevant input. Never rely solely on a toast for validation errors.

```tsx
// components/InlineError.tsx
export function InlineError({
  message,
  id,
}: {
  message: string;
  id: string;
}) {
  return (
    <p
      id={id}
      role="alert"
      className="text-sm text-red-600 mt-1"
    >
      {message}
    </p>
  );
}

// Usage
<input
  {...register('email')}
  aria-invalid={!!errors.email}
  aria-describedby={errors.email ? 'email-error' : undefined}
/>
{errors.email && <InlineError id="email-error" message={errors.email.message!} />}
```

### 6.3 Full-Screen Error States

For page-level errors (data fetch failed, 404, 500), use a full-screen error state:

```tsx
// components/error-states/FullScreenError.tsx
export function FullScreenError({
  title,
  description,
  action,
  secondaryAction,
  illustration,
}: {
  title: string;
  description: string;
  action?: { label: string; onClick: () => void };
  secondaryAction?: { label: string; onClick: () => void };
  illustration?: ReactNode;
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-[60vh] text-center px-4">
      {illustration && (
        <div className="mb-8">{illustration}</div>
      )}
      <h1 className="text-2xl font-bold mb-2">{title}</h1>
      <p className="text-gray-600 mb-8 max-w-md">{description}</p>
      <div className="flex gap-4">
        {action && (
          <button
            onClick={action.onClick}
            className="btn-primary"
          >
            {action.label}
          </button>
        )}
        {secondaryAction && (
          <button
            onClick={secondaryAction.onClick}
            className="btn-secondary"
          >
            {secondaryAction.label}
          </button>
        )}
      </div>
    </div>
  );
}

// Pre-built error screens
export function NotFoundScreen() {
  const router = useRouter();
  return (
    <FullScreenError
      title="Page not found"
      description="The page you are looking for does not exist or has been moved."
      action={{ label: 'Go Home', onClick: () => router.push('/') }}
      secondaryAction={{ label: 'Go Back', onClick: () => router.back() }}
    />
  );
}

export function ServerErrorScreen({ onRetry }: { onRetry: () => void }) {
  return (
    <FullScreenError
      title="Something went wrong"
      description="We encountered an unexpected error. Our team has been notified and is working on a fix."
      action={{ label: 'Try Again', onClick: onRetry }}
      secondaryAction={{
        label: 'Contact Support',
        onClick: () => window.open('mailto:support@yourapp.com'),
      }}
    />
  );
}

export function MaintenanceScreen() {
  return (
    <FullScreenError
      title="Under maintenance"
      description="We are performing scheduled maintenance. We will be back shortly."
      action={{
        label: 'Check Status',
        onClick: () => window.open('https://status.yourapp.com'),
      }}
    />
  );
}
```

### 6.4 Empty State vs Error State

These are different. An empty state means "we successfully fetched data and there is nothing." An error state means "we tried to fetch data and failed." They require different UIs:

```tsx
// components/DataList.tsx
function DataList() {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: ['items'],
    queryFn: api.getItems,
  });

  if (isLoading) return <ListSkeleton />;

  if (isError) {
    // ERROR state: something went wrong
    return (
      <div role="alert" className="error-state">
        <AlertCircleIcon />
        <h3>Could not load items</h3>
        <p>{getUserFacingMessage(error)}</p>
        <button onClick={() => refetch()}>Retry</button>
      </div>
    );
  }

  if (data.length === 0) {
    // EMPTY state: success, but no data
    return (
      <div className="empty-state">
        <InboxIcon />
        <h3>No items yet</h3>
        <p>Create your first item to get started.</p>
        <button onClick={openCreateDialog}>Create Item</button>
      </div>
    );
  }

  return (
    <ul>
      {data.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

### 6.5 The Error UX Decision Tree

```
Error occurs
├── Is it a validation error?
│   └── YES → Show inline next to the field (never toast-only)
├── Is it a transient error (network, timeout)?
│   ├── Is the user in the middle of an action?
│   │   └── YES → Show toast with retry button
│   └── Is it a background fetch?
│       └── YES → Show stale data with a subtle "update failed" indicator
├── Is it a 404?
│   └── YES → Full-screen "not found" with navigation options
├── Is it a 401/403?
│   ├── 401 → Attempt token refresh → redirect to login if refresh fails
│   └── 403 → Show "access denied" with contact admin suggestion
├── Is it a 500?
│   └── YES → Full-screen error with retry + contact support
├── Is it a render error (caught by boundary)?
│   ├── Section-level → Show section placeholder with retry
│   └── Screen-level → Show screen error with navigation options
└── Is it a complete app crash?
    └── YES → Show crash screen with reload button
```

---

## 7. GLOBAL ERROR HANDLERS

Some errors escape your try/catch blocks and error boundaries. You need global handlers to catch them.

### 7.1 Unhandled Promise Rejections (Web)

```typescript
// utils/global-error-handlers.ts

export function setupGlobalErrorHandlers() {
  // Unhandled promise rejections
  if (typeof window !== 'undefined') {
    window.addEventListener('unhandledrejection', (event) => {
      event.preventDefault(); // Prevent console error

      const error = event.reason;

      reportError(error, {
        type: 'unhandled_promise_rejection',
        level: 'error',
      });

      // Show a generic toast -- the user should know something went wrong
      showToast.error('An unexpected error occurred. Please try again.');
    });

    // Uncaught errors (runtime errors that escape everything)
    window.addEventListener('error', (event) => {
      reportError(event.error ?? new Error(event.message), {
        type: 'uncaught_error',
        level: 'fatal',
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
      });
    });
  }
}
```

### 7.2 React Native Global Handler

```typescript
// utils/global-error-handlers.native.ts
import { ErrorUtils } from 'react-native';

export function setupGlobalErrorHandlers() {
  // JavaScript errors in React Native
  const defaultHandler = ErrorUtils.getGlobalHandler();

  ErrorUtils.setGlobalHandler((error, isFatal) => {
    reportError(error, {
      type: 'react_native_global',
      isFatal,
      level: isFatal ? 'fatal' : 'error',
    });

    if (isFatal) {
      // Show a crash dialog
      Alert.alert(
        'Unexpected Error',
        'The app encountered an unexpected error and needs to restart.',
        [
          {
            text: 'Restart',
            onPress: () => {
              // On React Native, you might use expo-updates to reload
              // or RNRestart to restart the app
              Updates.reloadAsync();
            },
          },
        ],
      );
    }

    // Call the default handler too (for Crashlytics, etc.)
    defaultHandler(error, isFatal);
  });

  // Unhandled promise rejections
  // In React Native, you need a polyfill or the built-in tracking:
  if (__DEV__) {
    // In dev, let them bubble to the yellow box
  } else {
    const tracking = require('promise/setimmediate/rejection-tracking');
    tracking.enable({
      allRejections: true,
      onUnhandled: (id: number, error: Error) => {
        reportError(error, {
          type: 'unhandled_promise_rejection',
          level: 'error',
        });
      },
    });
  }
}
```

### 7.3 Error Reporting (Sentry Integration)

```typescript
// utils/error-reporting.ts
import * as Sentry from '@sentry/react'; // or @sentry/react-native

export function initErrorReporting() {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NODE_ENV,
    release: process.env.NEXT_PUBLIC_APP_VERSION,
    tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
    integrations: [
      Sentry.browserTracingIntegration(),
      Sentry.replayIntegration({
        // Record sessions that had errors
        maskAllText: true,
        blockAllMedia: true,
      }),
    ],
    replaysSessionSampleRate: 0,      // Don't record normal sessions
    replaysOnErrorSampleRate: 1.0,     // Record 100% of sessions with errors

    beforeSend(event, hint) {
      // Filter out known non-actionable errors
      const error = hint.originalException;

      if (error instanceof ApiError) {
        // Don't report 401s (expected when session expires)
        if (error.isAuthError) return null;

        // Don't report 404s (usually just broken links)
        if (error.isNotFound) return null;

        // Add API context
        event.tags = {
          ...event.tags,
          'api.status': String(error.status),
          'api.code': error.code,
        };
        event.extra = {
          ...event.extra,
          requestId: error.requestId,
          details: error.details,
        };
      }

      if (error instanceof NetworkError) {
        // Don't report offline errors (not actionable)
        if (error.isOffline) return null;

        event.tags = {
          ...event.tags,
          'network.timeout': String(error.isTimeout),
        };
      }

      return event;
    },
  });
}

// The reporting function used throughout the app
export function reportError(
  error: unknown,
  context: Record<string, unknown> = {},
) {
  if (process.env.NODE_ENV === 'development') {
    console.error('[Error Report]', error, context);
    return;
  }

  Sentry.withScope((scope) => {
    // Add context
    Object.entries(context).forEach(([key, value]) => {
      scope.setExtra(key, value);
    });

    // Set level based on context
    if (context.level === 'fatal') {
      scope.setLevel('fatal');
    } else if (context.level === 'warning') {
      scope.setLevel('warning');
    }

    // Capture
    if (error instanceof Error) {
      Sentry.captureException(error);
    } else {
      Sentry.captureMessage(String(error));
    }
  });
}

// Sentry error boundary wrapper (provides automatic reporting)
export const SentryErrorBoundary = Sentry.withErrorBoundary;
```

### 7.4 Adding User Context

```typescript
// When the user logs in, set their context for all future error reports
export function setErrorReportingUser(user: {
  id: string;
  email: string;
  plan: string;
}) {
  Sentry.setUser({
    id: user.id,
    email: user.email,
    plan: user.plan,
  });
}

// When they log out, clear it
export function clearErrorReportingUser() {
  Sentry.setUser(null);
}

// Add breadcrumbs for user actions (so you can see what they did before the error)
export function addBreadcrumb(
  category: string,
  message: string,
  data?: Record<string, unknown>,
) {
  Sentry.addBreadcrumb({
    category,
    message,
    data,
    level: 'info',
    timestamp: Date.now() / 1000,
  });
}

// Usage in your app:
// Navigation
addBreadcrumb('navigation', 'Navigated to /dashboard');

// User action
addBreadcrumb('user', 'Clicked "Add to Cart"', { productId: '123' });

// API call
addBreadcrumb('api', 'POST /api/orders', { itemCount: 3 });
```

---

## 8. GRACEFUL DEGRADATION

Graceful degradation is the art of failing partially instead of completely. When a feature fails, the rest of the app should keep working.

### 8.1 Showing Cached Data on Error

```tsx
// components/ProductList.tsx
function ProductList() {
  const { data, error, isError, isFetching, dataUpdatedAt } = useQuery({
    queryKey: ['products'],
    queryFn: api.getProducts,
    staleTime: 5 * 60 * 1000, // 5 minutes
    // Keep showing the previous data even if the refetch fails
    placeholderData: keepPreviousData,
  });

  return (
    <div>
      {/* Show a stale data warning if the last fetch failed */}
      {isError && data && (
        <div role="alert" className="stale-data-banner">
          <p>
            Showing data from {formatTimeAgo(dataUpdatedAt)}.
            {' '}
            <button onClick={() => refetch()}>Refresh</button>
          </p>
        </div>
      )}

      {/* Still show the data even though the refresh failed */}
      {data ? (
        <ul>
          {data.map((product) => (
            <li key={product.id}>{product.name}</li>
          ))}
        </ul>
      ) : isError ? (
        <FullScreenError
          title="Could not load products"
          description={getUserFacingMessage(error)}
          action={{ label: 'Retry', onClick: refetch }}
        />
      ) : null}

      {/* Subtle loading indicator for background refresh */}
      {isFetching && data && (
        <div className="absolute top-2 right-2">
          <Spinner size="sm" />
        </div>
      )}
    </div>
  );
}
```

### 8.2 Feature Flags for Broken Features

When a feature is broken in production, you need the ability to hide it without deploying:

```typescript
// utils/feature-degradation.ts

interface FeatureHealth {
  name: string;
  healthy: boolean;
  lastError?: string;
  errorCount: number;
  disabledUntil?: number;
}

class FeatureHealthTracker {
  private features: Map<string, FeatureHealth> = new Map();
  private readonly ERROR_THRESHOLD = 3;
  private readonly COOLDOWN_MS = 5 * 60 * 1000; // 5 minutes

  recordError(featureName: string, error: Error) {
    const current = this.features.get(featureName) ?? {
      name: featureName,
      healthy: true,
      errorCount: 0,
    };

    current.errorCount++;
    current.lastError = error.message;

    if (current.errorCount >= this.ERROR_THRESHOLD) {
      current.healthy = false;
      current.disabledUntil = Date.now() + this.COOLDOWN_MS;
      console.warn(`Feature "${featureName}" disabled due to repeated errors`);
    }

    this.features.set(featureName, current);
  }

  recordSuccess(featureName: string) {
    const current = this.features.get(featureName);
    if (current) {
      current.errorCount = 0;
      current.healthy = true;
      current.disabledUntil = undefined;
    }
  }

  isHealthy(featureName: string): boolean {
    const feature = this.features.get(featureName);
    if (!feature) return true;

    // Check if cooldown has expired
    if (feature.disabledUntil && Date.now() >= feature.disabledUntil) {
      feature.healthy = true;
      feature.errorCount = 0;
      feature.disabledUntil = undefined;
      return true;
    }

    return feature.healthy;
  }
}

export const featureHealth = new FeatureHealthTracker();
```

```tsx
// components/DegradableFeature.tsx
export function DegradableFeature({
  name,
  children,
  fallback = null,
}: {
  name: string;
  children: ReactNode;
  fallback?: ReactNode;
}) {
  const [isHealthy, setIsHealthy] = useState(featureHealth.isHealthy(name));

  if (!isHealthy) {
    return <>{fallback}</>;
  }

  return (
    <ErrorBoundary
      level="component"
      onError={(error) => {
        featureHealth.recordError(name, error);
        setIsHealthy(false);
      }}
      fallback={fallback}
    >
      {children}
    </ErrorBoundary>
  );
}

// Usage
function Dashboard() {
  return (
    <div>
      {/* If the AI recommendations feature keeps crashing, hide it */}
      <DegradableFeature
        name="ai-recommendations"
        fallback={
          <div className="widget-placeholder">
            <p>Recommendations temporarily unavailable</p>
          </div>
        }
      >
        <AiRecommendations />
      </DegradableFeature>

      {/* Critical features still render normally */}
      <RecentOrders />
    </div>
  );
}
```

### 8.3 Degradation Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│              DEGRADATION HIERARCHY                           │
│                                                              │
│  LEVEL 1: Full functionality (everything works)             │
│  └── Show all features, real-time data, full interactivity  │
│                                                              │
│  LEVEL 2: Stale data (network issues)                       │
│  └── Show cached data with "last updated" indicator         │
│  └── Queue mutations for later                              │
│  └── Disable real-time features                             │
│                                                              │
│  LEVEL 3: Partial features (some services down)             │
│  └── Hide broken features                                   │
│  └── Show "temporarily unavailable" placeholders            │
│  └── Suggest alternative workflows                          │
│                                                              │
│  LEVEL 4: Read-only mode (write services down)              │
│  └── Disable all mutations                                  │
│  └── Show "read-only" banner                                │
│  └── Queue mutations for later replay                       │
│                                                              │
│  LEVEL 5: Maintenance mode (planned downtime)               │
│  └── Show maintenance page                                  │
│  └── Display expected resolution time                       │
│  └── Link to status page                                    │
│                                                              │
│  LEVEL 6: Complete failure (app crash)                      │
│  └── Show crash screen with reload button                   │
│  └── Report to error tracking                               │
│  └── NEVER show a white screen                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. DEAD LETTER PATTERNS

Some mutations fail and cannot be retried. The payment gateway returned an ambiguous error. The server accepted the request but the response was lost. The user submitted a form but the app crashed before confirming. These are "dead letters" -- messages that could not be delivered.

### 9.1 The Dead Letter Queue

```typescript
// utils/dead-letter-queue.ts

interface DeadLetter {
  id: string;
  type: string;
  payload: unknown;
  error: {
    message: string;
    code?: string;
    status?: number;
  };
  timestamp: number;
  attempts: number;
  lastAttempt: number;
  resolved: boolean;
  resolution?: 'retried_successfully' | 'manually_resolved' | 'abandoned';
}

class DeadLetterQueue {
  private readonly STORAGE_KEY = 'dead-letter-queue';

  getAll(): DeadLetter[] {
    try {
      return JSON.parse(localStorage.getItem(this.STORAGE_KEY) ?? '[]');
    } catch {
      return [];
    }
  }

  add(letter: Omit<DeadLetter, 'id' | 'timestamp' | 'resolved'>): void {
    const letters = this.getAll();
    letters.push({
      ...letter,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      resolved: false,
    });
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(letters));

    // Alert the user
    showToast.error(
      'A previous action could not be completed. Check your pending actions.',
      {
        action: {
          label: 'View',
          onClick: () => router.push('/account/pending-actions'),
        },
      },
    );

    // Report to error tracking
    reportError(new Error(`Dead letter: ${letter.type}`), {
      level: 'warning',
      payload: letter.payload,
      error: letter.error,
    });
  }

  async retry(id: string, retryFn: (payload: unknown) => Promise<void>): Promise<boolean> {
    const letters = this.getAll();
    const letter = letters.find((l) => l.id === id);

    if (!letter) return false;

    try {
      await retryFn(letter.payload);
      letter.resolved = true;
      letter.resolution = 'retried_successfully';
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(letters));
      return true;
    } catch (error) {
      letter.attempts++;
      letter.lastAttempt = Date.now();
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(letters));
      return false;
    }
  }

  resolve(id: string, resolution: DeadLetter['resolution']): void {
    const letters = this.getAll();
    const letter = letters.find((l) => l.id === id);
    if (letter) {
      letter.resolved = true;
      letter.resolution = resolution;
      localStorage.setItem(this.STORAGE_KEY, JSON.stringify(letters));
    }
  }

  getUnresolved(): DeadLetter[] {
    return this.getAll().filter((l) => !l.resolved);
  }

  cleanup(olderThanMs: number = 7 * 24 * 60 * 60 * 1000): void {
    const letters = this.getAll();
    const cutoff = Date.now() - olderThanMs;
    const filtered = letters.filter(
      (l) => !l.resolved || l.timestamp > cutoff,
    );
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(filtered));
  }
}

export const deadLetterQueue = new DeadLetterQueue();
```

### 9.2 Pending Actions UI

```tsx
// app/account/pending-actions/page.tsx
export default function PendingActionsPage() {
  const [deadLetters, setDeadLetters] = useState(deadLetterQueue.getUnresolved());

  if (deadLetters.length === 0) {
    return (
      <div className="empty-state">
        <CheckCircleIcon />
        <h2>No pending actions</h2>
        <p>All your actions have been completed successfully.</p>
      </div>
    );
  }

  return (
    <div>
      <h1>Pending Actions</h1>
      <p className="text-gray-600 mb-6">
        These actions could not be completed. You can retry them or dismiss them.
      </p>

      {deadLetters.map((letter) => (
        <div key={letter.id} className="pending-action-card">
          <div>
            <h3>{getActionTitle(letter.type)}</h3>
            <p className="text-sm text-gray-500">
              {formatTimeAgo(letter.timestamp)} -- {letter.attempts} attempts
            </p>
            <p className="text-sm text-red-600">{letter.error.message}</p>
          </div>

          <div className="flex gap-2">
            <button
              onClick={async () => {
                const success = await deadLetterQueue.retry(
                  letter.id,
                  getRetryFn(letter.type),
                );
                if (success) {
                  showToast.success('Action completed successfully!');
                  setDeadLetters(deadLetterQueue.getUnresolved());
                } else {
                  showToast.error('Retry failed. Please try again later.');
                }
              }}
            >
              Retry
            </button>
            <button
              onClick={() => {
                deadLetterQueue.resolve(letter.id, 'manually_resolved');
                setDeadLetters(deadLetterQueue.getUnresolved());
              }}
              className="text-gray-500"
            >
              Dismiss
            </button>
            <a href="mailto:support@yourapp.com" className="text-blue-600">
              Contact Support
            </a>
          </div>
        </div>
      ))}
    </div>
  );
}

function getActionTitle(type: string): string {
  const titles: Record<string, string> = {
    'order.create': 'Place Order',
    'payment.process': 'Process Payment',
    'profile.update': 'Update Profile',
    'document.save': 'Save Document',
  };
  return titles[type] ?? 'Unknown Action';
}
```

---

## 10. THE COMPLETE ERROR ARCHITECTURE

Let's put it all together. Here is the full layered error handling architecture from the innermost component to the outermost reporting layer.

### 10.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                THE ERROR HANDLING LAYERS                     │
│                                                              │
│  LAYER 1: COMPONENT (innermost)                             │
│  ├── try/catch in event handlers                            │
│  ├── Error state with inline display                        │
│  └── Retry logic for individual operations                  │
│                                                              │
│  LAYER 2: SECTION (ErrorBoundary level="section")           │
│  ├── Catches render errors in widgets/cards                 │
│  ├── Shows placeholder fallback                             │
│  └── Rest of the page keeps working                         │
│                                                              │
│  LAYER 3: SCREEN (ErrorBoundary level="screen")             │
│  ├── Catches page-level render errors                       │
│  ├── Shows full-page error state with navigation            │
│  └── App shell (nav, sidebar) stays functional              │
│                                                              │
│  LAYER 4: APP (ErrorBoundary level="app")                   │
│  ├── Last-resort catch for catastrophic failures            │
│  ├── Shows crash screen with reload button                  │
│  └── Reports as fatal to error tracking                     │
│                                                              │
│  LAYER 5: GLOBAL HANDLERS                                   │
│  ├── window.onerror / ErrorUtils.setGlobalHandler           │
│  ├── unhandledrejection                                     │
│  └── Catches anything that escaped all boundaries           │
│                                                              │
│  LAYER 6: ERROR REPORTING (Sentry/Crashlytics)              │
│  ├── Receives reports from all layers                       │
│  ├── Deduplicates, groups, alerts                           │
│  ├── Includes user context, breadcrumbs, session replay     │
│  └── Triggers PagerDuty/Slack alerts for critical errors    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Wiring It All Together

```tsx
// app/providers.tsx -- The provider stack with error handling at every level

import { setupGlobalErrorHandlers } from '@/utils/global-error-handlers';
import { initErrorReporting } from '@/utils/error-reporting';

// Initialize once on app start
if (typeof window !== 'undefined') {
  initErrorReporting();
  setupGlobalErrorHandlers();
}

export function Providers({ children }: { children: ReactNode }) {
  return (
    // Layer 4: App-level error boundary
    <ErrorBoundary
      level="app"
      fallback={(error, reset) => <AppCrashScreen error={error} onReset={reset} />}
    >
      <QueryClientProvider client={queryClient}>
        <AuthProvider>
          <ThemeProvider>
            {/* Offline detection */}
            <OfflineBanner />

            {/* Toast container for transient errors */}
            <Toaster position="top-center" richColors />

            {/* Layer 3: Screen-level error boundary via layout */}
            {children}
          </ThemeProvider>
        </AuthProvider>
      </QueryClientProvider>
    </ErrorBoundary>
  );
}
```

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html>
      <body>
        <Providers>
          {children}
        </Providers>
      </body>
    </html>
  );
}
```

```tsx
// app/(authenticated)/layout.tsx -- Authenticated layout with screen-level boundary
export default function AuthenticatedLayout({ children }: { children: ReactNode }) {
  return (
    <div className="app-shell">
      {/* Sidebar is outside the error boundary -- always visible */}
      <Sidebar />

      <main className="main-content">
        {/* Layer 3: Screen-level boundary */}
        <RouteErrorBoundary>
          {children}
        </RouteErrorBoundary>
      </main>
    </div>
  );
}
```

```tsx
// app/(authenticated)/dashboard/page.tsx -- Dashboard with section-level boundaries
export default function DashboardPage() {
  return (
    <div className="dashboard-grid">
      {/* Layer 2: Section-level boundaries for each widget */}
      <DegradableFeature name="revenue-chart" fallback={<WidgetPlaceholder title="Revenue" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <RevenueChart />
        </Suspense>
      </DegradableFeature>

      <DegradableFeature name="recent-orders" fallback={<WidgetPlaceholder title="Orders" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <RecentOrders />
        </Suspense>
      </DegradableFeature>

      <DegradableFeature name="analytics" fallback={<WidgetPlaceholder title="Analytics" />}>
        <Suspense fallback={<WidgetSkeleton />}>
          <AnalyticsWidget />
        </Suspense>
      </DegradableFeature>
    </div>
  );
}
```

```tsx
// components/RecentOrders.tsx -- Component-level error handling
function RecentOrders() {
  const { data: orders } = useSuspenseQuery({
    queryKey: ['recent-orders'],
    queryFn: api.getRecentOrders,
  });

  const handleCancelOrder = async (orderId: string) => {
    // Layer 1: Component-level try/catch for event handlers
    try {
      await api.cancelOrder(orderId);
      showToast.success('Order cancelled');
    } catch (error) {
      if (error instanceof ApiError) {
        if (error.code === 'ORDER_ALREADY_SHIPPED') {
          showToast.error('This order has already shipped and cannot be cancelled.');
          return;
        }
      }

      // For retryable errors, offer retry
      showToast.error('Could not cancel order. Please try again.', {
        action: {
          label: 'Retry',
          onClick: () => handleCancelOrder(orderId),
        },
      });

      reportError(error, {
        action: 'cancel_order',
        orderId,
      });
    }
  };

  return (
    <div className="widget-card">
      <h3>Recent Orders</h3>
      <ul>
        {orders.map((order) => (
          <li key={order.id}>
            {order.name} - ${order.total}
            {order.status === 'pending' && (
              <button onClick={() => handleCancelOrder(order.id)}>
                Cancel
              </button>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 10.3 The Error Handling Utility Belt

Put all your error utilities in one place:

```typescript
// utils/errors/index.ts

// Re-export everything
export { ApiError, NetworkError, classifyNetworkError } from './types';
export { ErrorCode } from './codes';
export { getUserFacingMessage } from './messages';
export { reportError, addBreadcrumb, setErrorReportingUser } from './reporting';
export { withRetry } from './retry';
export { CircuitBreaker, CircuitBreakerError } from './circuit-breaker';
export { mutationQueue } from './mutation-queue';
export { deadLetterQueue } from './dead-letter-queue';
export { featureHealth } from './feature-health';
export { showToast } from './toast';

// Convenience: wrap an async function with standard error handling
export async function withErrorHandling<T>(
  fn: () => Promise<T>,
  options: {
    onError?: (error: unknown) => void;
    showToast?: boolean;
    reportToSentry?: boolean;
    context?: Record<string, unknown>;
  } = {},
): Promise<T | null> {
  const {
    onError,
    showToast: shouldToast = true,
    reportToSentry = true,
    context = {},
  } = options;

  try {
    return await fn();
  } catch (error) {
    if (shouldToast) {
      showToast.error(getUserFacingMessage(error));
    }

    if (reportToSentry) {
      reportError(error, context);
    }

    onError?.(error);
    return null;
  }
}

// Usage:
const result = await withErrorHandling(
  () => api.updateProfile(data),
  {
    context: { action: 'update_profile', userId },
    onError: (error) => {
      if (error instanceof ApiError && error.isValidationError) {
        setFieldErrors(error.details?.fieldErrors);
      }
    },
  },
);
```

### 10.4 TanStack Query Error Configuration

Configure TanStack Query to work with your error handling architecture:

```typescript
// lib/query-client.ts
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // Don't retry client errors (they won't magically succeed)
        if (error instanceof ApiError && error.isClientError) return false;
        // Don't retry if offline
        if (error instanceof NetworkError && error.isOffline) return false;
        // Retry up to 3 times for server/network errors
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => {
        // Exponential backoff with jitter
        return Math.min(1000 * 2 ** attemptIndex + Math.random() * 1000, 30000);
      },
      staleTime: 60 * 1000, // 1 minute
    },
    mutations: {
      retry: false, // Don't auto-retry mutations (they might not be idempotent)
    },
  },

  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only report errors for queries that have been successfully fetched before
      // (to avoid flooding Sentry with errors on first load)
      if (query.state.data !== undefined) {
        reportError(error, {
          type: 'query_error',
          queryKey: query.queryKey,
        });
      }
    },
  }),

  mutationCache: new MutationCache({
    onError: (error, variables, context, mutation) => {
      reportError(error, {
        type: 'mutation_error',
        mutationKey: mutation.options.mutationKey,
      });

      // Global mutation error handling
      if (error instanceof ApiError) {
        if (error.isAuthError) {
          // Redirect to login
          router.push('/login');
          return;
        }
      }
    },
  }),
});
```

### 10.5 The App Crash Screen

The absolute last resort. Every app needs one.

```tsx
// components/AppCrashScreen.tsx
export function AppCrashScreen({
  error,
  onReset,
}: {
  error: Error;
  onReset: () => void;
}) {
  const [reportSent, setReportSent] = useState(false);

  useEffect(() => {
    // This is a critical error -- always report immediately
    reportError(error, {
      level: 'fatal',
      type: 'app_crash',
    });
    setReportSent(true);
  }, [error]);

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 px-4">
      <div className="max-w-md text-center">
        <div className="text-6xl mb-4">:(</div>
        <h1 className="text-2xl font-bold mb-2">
          Something went seriously wrong
        </h1>
        <p className="text-gray-600 mb-6">
          The app encountered a critical error.
          {reportSent
            ? ' Our team has been automatically notified.'
            : ' Please try reloading the page.'}
        </p>

        <div className="flex flex-col gap-3">
          <button
            onClick={onReset}
            className="btn-primary"
          >
            Try to Recover
          </button>
          <button
            onClick={() => window.location.reload()}
            className="btn-secondary"
          >
            Reload Page
          </button>
          <a
            href="mailto:support@yourapp.com"
            className="text-sm text-blue-600 hover:underline"
          >
            Contact Support
          </a>
        </div>

        {process.env.NODE_ENV === 'development' && (
          <details className="mt-8 text-left">
            <summary className="cursor-pointer text-sm text-gray-500">
              Error Details (dev only)
            </summary>
            <pre className="mt-2 p-4 bg-gray-900 text-red-400 text-xs overflow-auto rounded">
              {error.message}
              {'\n\n'}
              {error.stack}
            </pre>
          </details>
        )}
      </div>
    </div>
  );
}
```

---

## WRAPPING UP

Error handling is not glamorous. Nobody is going to tweet about your error boundary placement strategy. No one will write a blog post about how your circuit breaker pattern saved the day. But the absence of good error handling is immediately visible -- in white screens, in lost data, in support tickets, in users who leave and never come back.

Here is the checklist for a production-grade error handling architecture:

```
┌─────────────────────────────────────────────────────────────┐
│           ERROR HANDLING READINESS CHECKLIST                 │
│                                                              │
│  ✓ Error boundaries at app, screen, and section levels      │
│  ✓ Standardized API error format (code, message, details)   │
│  ✓ Error codes enum shared between frontend and backend     │
│  ✓ User-facing message mapping for all error codes          │
│  ✓ Network error detection (offline, timeout, DNS)          │
│  ✓ Retry with exponential backoff for transient errors      │
│  ✓ Circuit breaker for repeatedly failing endpoints         │
│  ✓ Offline mutation queue with replay on reconnect          │
│  ✓ Toast notifications for transient errors                 │
│  ✓ Inline errors for validation failures                    │
│  ✓ Full-screen error states for page-level failures         │
│  ✓ Global error handlers for unhandled exceptions           │
│  ✓ Sentry/Crashlytics with user context and breadcrumbs     │
│  ✓ Dead letter queue for unrecoverable mutations            │
│  ✓ Graceful degradation (cached data, feature hiding)       │
│  ✓ Never a white screen                                     │
│  ✓ Never a raw error message shown to users                 │
│  ✓ Never a silent failure that loses data                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

If you can check every box, your application is ready for the real world -- where networks fail, servers crash, code throws, and users still expect a seamless experience. That is what separates a prototype from a product.

---

> **Next:** [Chapter 37](/part-4-architecture-at-scale/37-next-chapter.md)
>
> **Previous:** [Chapter 35: Forms at Scale](/part-3-state-data-communication/35-forms-at-scale.md)
