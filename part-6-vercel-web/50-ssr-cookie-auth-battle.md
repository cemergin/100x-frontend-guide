<!--
  CHAPTER: 50
  TITLE: The SSR Battle — Cookies, Auth, State & Hydration Hell
  PART: VI — Vercel & the Web
  PREREQS: Chapters 28, 33
  KEY_TOPICS: SSR auth, cookies, httpOnly, sameSite, secure cookies, cookie management, server-side auth, client-side auth, hydration mismatch, state hydration, SSR + CSR state sync, context in SSR, monitoring in SSR, analytics in SSR, session management, CSRF, cookie parsing, middleware auth, Server Component auth, Client Component auth, Next.js cookies API, Vercel Fluid Compute, edge cookies
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 50: The SSR Battle — Cookies, Auth, State & Hydration Hell

> **Part VI — Vercel & the Web** | Prerequisites: Chapters 28, 33 | Difficulty: Advanced

> "The server sees the cookie. The client sees the cookie. But they never see the same cookie at the same time, and that is the root of all SSR suffering."

---

<details>
<summary><strong>TL;DR</strong></summary>

- SSR auth is hard because the server and client live in fundamentally different worlds: the server has cookies but no localStorage, the client has both but cannot read httpOnly cookies, and they render at different times with potentially different auth state
- Use httpOnly + Secure + SameSite=Lax cookies for session tokens; use a separate non-httpOnly cookie or server-rendered props to communicate "logged in" state to Client Components
- Middleware is your first line of defense: validate sessions, redirect unauthenticated users, and set request headers before Server Components run
- Cache auth checks with `React.cache()` so you validate the session once per request, not once per Server Component
- Hydration mismatches are the #1 SSR bug category; they happen when the server renders one thing and the client expects another. The fix is always: make the server and client agree on the same initial state
- State management in SSR requires per-request isolation: Zustand stores, React Context, and TanStack Query clients must be scoped to a single request or you leak state between users
- CSRF is your problem again once you use cookies for auth; SameSite=Lax handles most cases, but Server Actions and API routes need explicit protection for cross-origin scenarios

</details>

I am going to be honest with you: this chapter exists because I have watched dozens of teams ship SSR applications and hit the same walls. Not because they were bad engineers. Because the model is genuinely hard. You are running the same React code in two completely different environments, at two different times, with two different sets of available APIs, and you are expected to produce identical output. That is a tall order.

This is not the chapter where I explain the theory of Server Components or walk you through `app/` directory structure. Chapter 28 covers that. This is the chapter about the sharp edges you hit after you think you understand SSR. The cookie that works locally but breaks in production. The hydration warning that only shows up for logged-in users. The Zustand store that leaks one user's data to another. The analytics event that fires twice or not at all.

If you have shipped an SSR app and nodded along to any of that: welcome. This chapter is for you.

### In This Chapter
- Why SSR Auth Is a Nightmare -- the fundamental model conflict between server and client
- Cookies 101 for Frontend Engineers -- everything SPAs let you forget
- The Auth Flow in SSR -- end-to-end, from sign-in to hydrated dashboard
- Middleware Auth -- the first line of defense, with complete implementation
- Server Component Auth -- reading cookies, validating sessions, caching checks
- Client Component Auth -- the four patterns for auth state on the client
- Hydration Mismatch Hell -- causes, debugging, and the patterns that prevent it
- State Management in SSR -- Context, Zustand, TanStack Query across the boundary
- Monitoring and Analytics in SSR -- tracking events across server and client
- CSRF Protection -- because cookies make CSRF your problem again
- Cookie Management Patterns -- production patterns for every cookie type
- The Complete SSR Auth Setup -- end-to-end working implementation
- Common SSR Auth Bugs and Fixes -- the bugs you will encounter, with solutions

### Related Chapters
- [Ch 28: Next.js App Router] -- Server Components, Client Components, the rendering model
- [Ch 33: Authentication] -- auth providers, OAuth2, JWT, token management
- [Ch 22: Security] -- secure storage, OWASP, transport security
- [Ch 11: Caching] -- caching strategies that interact with auth
- [Ch 20: Monitoring] -- observability foundations

---

## 1. WHY SSR AUTH IS A NIGHTMARE

### 1.1 The Two-World Problem

Here is the fundamental tension. Read this carefully because every bug in this chapter traces back to it.

**The server** renders your React components to HTML. During this render, it has access to:
- Incoming HTTP headers (including cookies)
- Environment variables and secrets
- Databases and APIs (directly, no fetch needed)
- The full Node.js runtime

It does NOT have access to:
- `window`, `document`, `navigator`
- `localStorage`, `sessionStorage`
- The DOM
- Browser-only APIs (Web Crypto browser API, Notification API, etc.)

**The client** hydrates the HTML and takes over interactivity. It has access to:
- `window`, `document`, `navigator`
- `localStorage`, `sessionStorage`
- The DOM
- `document.cookie` (but only non-httpOnly cookies)
- Browser APIs

It does NOT have access to:
- Server-side environment variables (unless prefixed with `NEXT_PUBLIC_`)
- Databases directly
- httpOnly cookies (the browser sends them, but JavaScript cannot read them)

Now here is the critical part: **the server renders first, then the client hydrates**. They do not run simultaneously. There is a gap. And in that gap, the state of the world can differ.

```
Timeline of an SSR page load:

1. Browser sends request (with cookies)
2. Server receives request, reads cookies
3. Server renders React components to HTML
4. Server sends HTML to browser
5. Browser displays HTML (user sees content!)
6. Browser downloads JavaScript bundle
7. Browser executes JavaScript
8. React hydrates -- attaches event handlers, makes components interactive
9. Client-side React takes over

The gap between steps 3 and 8 is where the pain lives.
```

The server knew the user was authenticated (it read the cookie in step 2). It rendered the dashboard with the user's name, their data, their permissions. But when the client hydrates in step 8, Client Components run their `useState` initializers, their `useEffect` hooks fire, and they might reach for auth state in a way that does not have access to the same information.

If the client renders something different from what the server rendered, React throws a hydration mismatch error. At best, you get a console warning. At worst, React discards the server-rendered HTML and re-renders from scratch, which defeats the entire purpose of SSR.

### 1.2 Why Auth Libraries Struggle

Most authentication libraries were built for one world or the other:

**SPA-era auth libraries** (Auth0 SPA SDK, Firebase Client SDK, old Supabase client):
- Store tokens in memory or localStorage
- Use `fetch` to call auth endpoints from the client
- Provide React hooks (`useUser()`, `useAuth()`) that work via Context + useEffect
- Assume the component renders in a browser
- When used in SSR: return `null` or `undefined` on the server, then the real value on the client = hydration mismatch

**Server-era auth libraries** (Passport.js, express-session, old NextAuth):
- Read cookies or session headers from the request
- Validate sessions on the server
- Assume they are running in a request handler, not a React component
- When used in SSR: work on the server but have no story for the client

Modern libraries like Clerk, Auth.js v5, and Supabase SSR bridge both worlds. They provide:
- Middleware that reads cookies and validates sessions
- Server Component helpers that access the validated session
- Client Component providers that receive the session from the server

But even with these libraries, you need to understand what is happening under the hood. Because when something breaks (and it will), the debugging requires understanding the two-world problem.

### 1.3 The Cookie Paradox

Here is the specific paradox that trips up every team:

1. You want your session token in an httpOnly cookie (for security -- JavaScript cannot steal it via XSS)
2. httpOnly means client-side JavaScript cannot read the cookie
3. But Client Components need to know if the user is logged in (to show the right UI, to make authenticated API calls)
4. So how does the client know the auth state?

This is not a bug. This is the design working correctly. httpOnly cookies are meant to be invisible to JavaScript. The browser sends them automatically with every request, but `document.cookie` cannot see them.

The solutions to this paradox are the core of section 6. But first, you need to understand cookies.

---

## 2. COOKIES 101 FOR FRONTEND ENGINEERS

If you spent the last few years building SPAs with JWT tokens in localStorage and `Authorization: Bearer` headers, you probably skipped cookies entirely. SSR brings them back with a vengeance. Here is everything you need to know.

### 2.1 What Cookies Actually Are

A cookie is a key-value pair that the server sets via a response header and the browser stores and automatically sends back on every subsequent request to that domain.

```
// Server response header:
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000

// On every subsequent request to the same domain, the browser sends:
Cookie: session_id=abc123
```

That is the entire model. The server says "store this," and the browser obediently sends it back on every request. No JavaScript involved. No explicit `fetch` configuration needed. The browser does it automatically.

This automatic behavior is both the superpower and the danger of cookies:
- **Superpower**: The server always has the session token. No client-side code needed to attach auth headers.
- **Danger**: The browser sends cookies to your domain even if the request was initiated by an attacker's website (this is CSRF, and we will cover it in section 10).

### 2.2 Cookie Attributes Deep Dive

Every cookie has attributes that control its behavior. Getting these wrong is the source of a shocking number of production bugs.

#### httpOnly

```
Set-Cookie: session=abc123; HttpOnly
```

When `HttpOnly` is set, the cookie is invisible to JavaScript. `document.cookie` will not include it. You cannot read it, write it, or delete it from client-side code. The browser still sends it with requests, but your JavaScript cannot touch it.

**When to use**: Always for session tokens, refresh tokens, and any security-sensitive value. If an XSS attack compromises your page, the attacker's injected JavaScript cannot steal httpOnly cookies.

**The trade-off**: Your Client Components cannot check "am I logged in?" by reading the cookie directly. This is the fundamental tension of SSR auth.

```typescript
// This will NOT include httpOnly cookies:
const cookies = document.cookie; // "theme=dark; locale=en"
// The session cookie is there (the browser sends it), but JS can't see it

// On the server (Next.js), you CAN read it:
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const session = cookieStore.get('session'); // Works! Server can read httpOnly cookies
}
```

#### Secure

```
Set-Cookie: session=abc123; Secure
```

The cookie is only sent over HTTPS connections. On HTTP, the browser will not include it in requests.

**When to use**: Always in production. The only exception is local development on `http://localhost` (browsers typically exempt localhost from this restriction).

**Common bug**: Setting `Secure` during local development with a non-localhost custom domain. The cookie silently disappears because the connection is HTTP.

#### SameSite

This attribute controls whether the browser sends the cookie with cross-site requests. It is your primary defense against CSRF attacks.

```
Set-Cookie: session=abc123; SameSite=Strict
Set-Cookie: session=abc123; SameSite=Lax
Set-Cookie: session=abc123; SameSite=None; Secure
```

**Strict**: The cookie is NEVER sent with cross-site requests. If a user clicks a link to your site from an email or another website, the cookie will not be included in that navigation. The user will appear logged out on the first page load, then logged in when they navigate within your site.

**Lax** (recommended for most cases): The cookie IS sent with top-level navigations (clicking a link, typing URL) but NOT with cross-site sub-requests (images, iframes, AJAX from other sites). This is the sweet spot: users clicking links from email land on your site logged in, but cross-site form submissions and AJAX calls do not carry the cookie.

**None**: The cookie is sent with ALL requests, including cross-site. Requires `Secure` flag. Use this only when you have a legitimate cross-site need (embedded iframes, cross-domain API calls with credentials).

```
SameSite behavior matrix:

+------------------------+--------+------+------+
| Scenario               | Strict | Lax  | None |
+------------------------+--------+------+------+
| User clicks link to    | NO     | YES  | YES  |
| your site from email   |        |      |      |
+------------------------+--------+------+------+
| User types your URL    | YES    | YES  | YES  |
| in address bar         |        |      |      |
+------------------------+--------+------+------+
| Form POST from another | NO     | NO   | YES  |
| site to your site      |        |      |      |
+------------------------+--------+------+------+
| fetch() from another   | NO     | NO   | YES  |
| site to your API       |        |      |      |
+------------------------+--------+------+------+
| Image/iframe on        | NO     | NO   | YES  |
| another site from      |        |      |      |
| your domain            |        |      |      |
+------------------------+--------+------+------+
| Navigation within      | YES    | YES  | YES  |
| your site              |        |      |      |
+------------------------+--------+------+------+
```

**The SameSite=Strict gotcha**: If you use Strict, users clicking a link to your site from Google, from an email, from Slack -- they will arrive unauthenticated. The session cookie will not be sent. The next navigation within your site will include the cookie and they will appear logged in. This creates a confusing flash-of-logged-out-state. Use Lax to avoid this.

#### Domain

```
Set-Cookie: session=abc123; Domain=example.com
```

Controls which domains the cookie is sent to:

- **No Domain attribute**: Cookie is sent only to the exact domain that set it (`app.example.com` only, not `api.example.com`).
- **Domain=example.com**: Cookie is sent to `example.com` AND all subdomains (`app.example.com`, `api.example.com`, etc.).

**Common pattern**: If your API is at `api.example.com` and your app is at `app.example.com`, set `Domain=example.com` so the session cookie is sent to both.

**Common bug**: Setting `Domain=app.example.com` and then wondering why `api.example.com` does not receive the cookie.

#### Path

```
Set-Cookie: refresh_token=xyz789; Path=/api/auth/refresh
```

The cookie is only sent for requests to paths that match or are children of the specified path. Default is `/` (sent with all paths).

**Use case**: Your refresh token should only be sent to the refresh endpoint. No reason to send it with every page request.

```typescript
// Refresh token: only sent to the refresh endpoint
'Set-Cookie': `refresh_token=${token}; Path=/api/auth/refresh; HttpOnly; Secure; SameSite=Strict; Max-Age=7776000`

// Session token: sent with all requests
'Set-Cookie': `session=${token}; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=2592000`
```

#### Expires vs Max-Age

```
// Expires: specific date/time
Set-Cookie: session=abc123; Expires=Thu, 07 May 2026 00:00:00 GMT

// Max-Age: seconds from now
Set-Cookie: session=abc123; Max-Age=2592000  // 30 days
```

If neither is set, the cookie is a **session cookie** and is deleted when the browser closes (though modern browsers with "restore tabs" often keep session cookies alive).

**Use Max-Age** instead of Expires. It is relative and you do not need to compute future dates. `Max-Age=0` deletes the cookie immediately.

```typescript
// Common durations:
const COOKIE_MAX_AGE = {
  session: 30 * 24 * 60 * 60,       // 30 days
  refresh: 90 * 24 * 60 * 60,       // 90 days
  rememberMe: 365 * 24 * 60 * 60,   // 1 year
  shortLived: 60 * 60,               // 1 hour
  deleteNow: 0,                       // Delete immediately
} as const;
```

### 2.3 Cookie Size Limits

- **4,096 bytes** per cookie (name + value + attributes)
- **Approximately 50 cookies** per domain (varies by browser)
- **Total cookie storage** of roughly 4KB * 50 = ~200KB per domain

If your JWT is 2KB (common with many claims), you are already at half the per-cookie limit. If you add a refresh token, a CSRF token, user preferences, consent flags -- you can hit limits.

**Practical implications**:
- Keep JWTs lean. Do not stuff the entire user profile into the token.
- Use database sessions instead of JWT if your session data is large.
- If you must store a large JWT, consider splitting it across multiple cookies (libraries like `next-auth` do this).

### 2.4 Third-Party Cookies Are Dying

Third-party cookies (cookies set by a domain different from the one in the address bar) are being phased out. Chrome, Firefox, and Safari have all implemented restrictions.

**What this means for you**:
- If your auth system relies on an auth provider's domain setting cookies (e.g., `auth.provider.com` sets a cookie while the user is on `your-app.com`), that is a third-party cookie and it may be blocked.
- The fix: proxy the auth provider through your own domain, or use the auth provider's embedded/redirect flow that sets cookies from your domain.
- Clerk, Auth.js, and Supabase SSR all handle this correctly by setting cookies from your domain.

### 2.5 Cookies in Next.js

Next.js provides the `cookies()` function from `next/headers` for reading and writing cookies. But where you can use it depends on context.

#### Reading Cookies

```typescript
// Server Components -- YES
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const session = cookieStore.get('session');
  const allCookies = cookieStore.getAll();
  const hasConsent = cookieStore.has('cookie_consent');
  // ...
}

// Middleware -- YES
import { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const session = request.cookies.get('session');
  const allCookies = request.cookies.getAll();
  // ...
}

// Route Handlers -- YES
import { cookies } from 'next/headers';

export async function GET() {
  const cookieStore = await cookies();
  const session = cookieStore.get('session');
  // ...
}

// Server Actions -- YES
'use server';
import { cookies } from 'next/headers';

export async function getUser() {
  const cookieStore = await cookies();
  const session = cookieStore.get('session');
  // ...
}

// Client Components -- NO (cannot import next/headers)
// Use document.cookie for non-httpOnly cookies only
```

#### Setting Cookies

This is where it gets tricky. You **cannot** set cookies in Server Components. The HTTP response headers are already being sent by the time a Server Component renders. You can only set cookies in places that control the response:

```typescript
// Server Actions -- YES
'use server';
import { cookies } from 'next/headers';

export async function signIn(formData: FormData) {
  const session = await authenticate(formData);
  const cookieStore = await cookies();
  cookieStore.set('session', session.token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 30, // 30 days
    path: '/',
  });
}

// Route Handlers -- YES
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const body = await request.json();
  const session = await authenticate(body);
  const cookieStore = await cookies();
  cookieStore.set('session', session.token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 30,
    path: '/',
  });
  return Response.json({ success: true });
}

// Middleware -- YES (via NextResponse)
import { NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  const response = NextResponse.next();
  response.cookies.set('visited', 'true', {
    httpOnly: false,
    maxAge: 60 * 60 * 24 * 365,
  });
  return response;
}

// Server Components -- NO
// By the time a Server Component renders, it's too late to set response headers
```

#### Deleting Cookies

```typescript
// Server Action
'use server';
import { cookies } from 'next/headers';

export async function signOut() {
  const cookieStore = await cookies();
  cookieStore.delete('session');
  cookieStore.delete('refresh_token');
  cookieStore.delete('user_flags');
  // Or set with maxAge: 0
  cookieStore.set('session', '', { maxAge: 0 });
}
```

---

## 3. THE AUTH FLOW IN SSR

Let me walk through the complete lifecycle, from sign-in to hydrated authenticated page, step by step.

### 3.1 Sign-In Flow

```
Step 1: User fills in credentials and submits form
Step 2: Server Action receives credentials
Step 3: Server validates credentials (check password hash, call auth provider, etc.)
Step 4: Server creates a session:
        - JWT: sign a token with user claims
        - Database session: create a row in sessions table, get session ID
Step 5: Server sets httpOnly cookie with the session token/ID
Step 6: Server redirects to the dashboard (or wherever the user was trying to go)
```

Here is the implementation:

```typescript
// app/auth/actions.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { z } from 'zod';
import { hashVerify } from '@/lib/crypto';
import { db } from '@/lib/db';
import { createSession } from '@/lib/session';

const signInSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export async function signIn(
  prevState: { error: string | null },
  formData: FormData,
) {
  // 1. Validate input
  const parsed = signInSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!parsed.success) {
    return { error: 'Invalid email or password format.' };
  }

  // 2. Find user and verify password
  const user = await db.user.findUnique({
    where: { email: parsed.data.email },
  });

  if (!user || !await hashVerify(parsed.data.password, user.passwordHash)) {
    // Deliberate: same error for "user not found" and "wrong password"
    // Prevents enumeration attacks
    return { error: 'Invalid email or password.' };
  }

  // 3. Create session
  const session = await createSession(user.id);

  // 4. Set cookie
  const cookieStore = await cookies();
  cookieStore.set('session', session.token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 30, // 30 days
    path: '/',
  });

  // 5. Set a non-httpOnly flag cookie so the client knows the user is authenticated
  // This cookie has NO sensitive data -- just a boolean flag
  cookieStore.set('auth_state', 'authenticated', {
    httpOnly: false, // Client JS CAN read this
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 30,
    path: '/',
  });

  // 6. Redirect
  redirect('/dashboard');
}
```

```tsx
// app/auth/sign-in/page.tsx
'use client';

import { useActionState } from 'react';
import { signIn } from '../actions';

export default function SignInPage() {
  const [state, formAction, isPending] = useActionState(signIn, { error: null });

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          name="email"
          type="email"
          required
          autoComplete="email"
        />
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          name="password"
          type="password"
          required
          autoComplete="current-password"
        />
      </div>

      {state.error && (
        <div role="alert" className="text-red-600">
          {state.error}
        </div>
      )}

      <button type="submit" disabled={isPending}>
        {isPending ? 'Signing in...' : 'Sign in'}
      </button>
    </form>
  );
}
```

### 3.2 Subsequent Request Flow

Once the session cookie is set, every subsequent request looks like this:

```
Step 1: Browser sends GET /dashboard
        Automatically includes: Cookie: session=<token>; auth_state=authenticated
Step 2: Middleware intercepts the request
        - Reads the session cookie
        - Validates the session (JWT verify or DB lookup)
        - If invalid: redirect to /auth/sign-in
        - If valid: allow the request to proceed, optionally set request headers
Step 3: Layout Server Component runs
        - Reads the session cookie
        - Fetches user data
        - Passes user data to AuthProvider (Client Component) as props
Step 4: Page Server Component runs
        - Also has access to the validated session
        - Fetches page-specific data for this user
        - Renders HTML with the user's data
Step 5: HTML sent to browser
        - User sees their dashboard immediately (with their name, data, everything)
Step 6: JavaScript loads, React hydrates
        - AuthProvider receives the server-rendered user data as initial props
        - No loading state, no flash of unauthenticated content
        - Client Components can now call useAuth() to get user data
Step 7: User interacts with the page
        - Client-side navigation uses the AuthProvider context
        - Server Actions automatically include cookies
```

### 3.3 The Gap: How Does the Client Know?

Here is the key question: after hydration, how does a Client Component know the user is authenticated?

The server rendered the page with auth data. The HTML includes the user's name, their avatar, their dashboard. But when Client Components hydrate and start running their JavaScript, they need programmatic access to auth state for things like:

- Conditionally showing UI elements
- Making authenticated fetch requests (the browser sends cookies automatically, but what if you need the user ID?)
- Checking permissions before enabling buttons
- Displaying the user's name in a dropdown

The cookie that proves the user is authenticated is httpOnly -- Client JavaScript cannot read it. So how does the client get auth data?

Four options, each with trade-offs. We will explore them all in section 6. The short answer: **pass auth data from the Server Component to the Client Component as props**. This is the simplest, most reliable approach.

---

## 4. MIDDLEWARE AUTH — THE FIRST LINE OF DEFENSE

Middleware runs before your page renders. It is the bouncer at the door. If the user is not authenticated, they never reach your Server Components at all.

### 4.1 Basic Middleware Auth

```typescript
// middleware.ts (at the root of your project)
import { NextRequest, NextResponse } from 'next/server';
import { verifySession } from '@/lib/session';

// Paths that do not require authentication
const publicPaths = new Set([
  '/auth/sign-in',
  '/auth/sign-up',
  '/auth/forgot-password',
  '/auth/reset-password',
  '/',
  '/pricing',
  '/about',
  '/blog',
]);

function isPublicPath(pathname: string): boolean {
  if (publicPaths.has(pathname)) return true;
  // Allow all /blog/* paths
  if (pathname.startsWith('/blog/')) return true;
  // Allow API auth routes
  if (pathname.startsWith('/api/auth/')) return true;
  // Allow static assets
  if (pathname.startsWith('/_next/')) return true;
  if (pathname.includes('.')) return true; // Files with extensions (favicon.ico, etc.)
  return false;
}

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Skip auth for public paths
  if (isPublicPath(pathname)) {
    return NextResponse.next();
  }

  // Read the session cookie
  const sessionCookie = request.cookies.get('session');

  if (!sessionCookie?.value) {
    // No session cookie -- redirect to sign-in
    const signInUrl = new URL('/auth/sign-in', request.url);
    signInUrl.searchParams.set('redirect', pathname);
    return NextResponse.redirect(signInUrl);
  }

  // Validate the session
  const session = await verifySession(sessionCookie.value);

  if (!session) {
    // Invalid or expired session -- clear cookie and redirect
    const signInUrl = new URL('/auth/sign-in', request.url);
    signInUrl.searchParams.set('redirect', pathname);
    const response = NextResponse.redirect(signInUrl);
    response.cookies.delete('session');
    response.cookies.delete('auth_state');
    return response;
  }

  // Session is valid -- pass user info to Server Components via headers
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-user-id', session.userId);
  requestHeaders.set('x-user-role', session.role);

  return NextResponse.next({
    request: {
      headers: requestHeaders,
    },
  });
}

export const config = {
  matcher: [
    /*
     * Match all request paths except:
     * - _next/static (static files)
     * - _next/image (image optimization)
     * - favicon.ico (favicon)
     */
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### 4.2 JWT Validation in Middleware

For JWT-based sessions, middleware is fast because you can validate the token without a database call:

```typescript
// lib/session.ts
import { SignJWT, jwtVerify, type JWTPayload } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

export interface SessionPayload extends JWTPayload {
  userId: string;
  role: 'user' | 'admin' | 'editor';
  email: string;
}

export async function createSessionToken(payload: Omit<SessionPayload, 'iat' | 'exp'>): Promise<string> {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('30d')
    .sign(JWT_SECRET);
}

export async function verifySession(token: string): Promise<SessionPayload | null> {
  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);
    return payload as SessionPayload;
  } catch {
    // Token is invalid, expired, or tampered with
    return null;
  }
}
```

**Why `jose` and not `jsonwebtoken`?** The `jose` library works in all JavaScript runtimes: Node.js, Edge Runtime, Workers. The `jsonwebtoken` library depends on Node.js crypto and does NOT work in Edge middleware. If you are running middleware on Vercel's Edge Runtime (the default), you must use a runtime-compatible library.

### 4.3 Database Session Validation in Middleware

If you use database-backed sessions, middleware needs a database call. This is slower but more secure because you can revoke sessions instantly.

```typescript
// lib/session.ts (database session variant)
import { db } from '@/lib/db';

export async function verifySession(token: string): Promise<SessionPayload | null> {
  const session = await db.session.findUnique({
    where: { token },
    include: { user: { select: { id: true, role: true, email: true } } },
  });

  if (!session) return null;
  if (session.expiresAt < new Date()) {
    // Session expired -- clean it up
    await db.session.delete({ where: { token } });
    return null;
  }

  return {
    userId: session.user.id,
    role: session.user.role,
    email: session.user.email,
  };
}
```

**Trade-off**: DB-backed sessions add latency to every request (the middleware runs on every matched path). Mitigations:
- Use a fast datastore for sessions (Redis, Upstash, DynamoDB)
- Use JWT with short expiry (15 minutes) + refresh token in a separate cookie for the best of both worlds
- Cache the session validation result in middleware headers so Server Components do not re-validate

### 4.4 Rate Limiting in Middleware

While we are in middleware, add rate limiting. It is the natural place for it.

```typescript
// lib/rate-limit.ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// 60 requests per minute per IP
export const rateLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(60, '1 m'),
  analytics: true,
});

// Stricter limit for auth endpoints: 5 attempts per minute per IP
export const authRateLimiter = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(5, '1 m'),
  analytics: true,
});
```

```typescript
// middleware.ts (with rate limiting added)
import { NextRequest, NextResponse } from 'next/server';
import { verifySession } from '@/lib/session';
import { rateLimiter, authRateLimiter } from '@/lib/rate-limit';

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const ip = request.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? '127.0.0.1';

  // Rate limit auth endpoints more strictly
  if (pathname.startsWith('/api/auth/') || pathname.startsWith('/auth/')) {
    const { success, limit, remaining, reset } = await authRateLimiter.limit(ip);
    if (!success) {
      return new NextResponse('Too many requests', {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString(),
          'Retry-After': Math.ceil((reset - Date.now()) / 1000).toString(),
        },
      });
    }
  }

  // General rate limit for all other paths
  if (!pathname.startsWith('/auth/')) {
    const { success } = await rateLimiter.limit(ip);
    if (!success) {
      return new NextResponse('Too many requests', { status: 429 });
    }
  }

  // ... rest of auth middleware from 4.1
  if (isPublicPath(pathname)) {
    return NextResponse.next();
  }

  const sessionCookie = request.cookies.get('session');
  if (!sessionCookie?.value) {
    const signInUrl = new URL('/auth/sign-in', request.url);
    signInUrl.searchParams.set('redirect', pathname);
    return NextResponse.redirect(signInUrl);
  }

  const session = await verifySession(sessionCookie.value);
  if (!session) {
    const signInUrl = new URL('/auth/sign-in', request.url);
    signInUrl.searchParams.set('redirect', pathname);
    const response = NextResponse.redirect(signInUrl);
    response.cookies.delete('session');
    response.cookies.delete('auth_state');
    return response;
  }

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-user-id', session.userId);
  requestHeaders.set('x-user-role', session.role);

  return NextResponse.next({ request: { headers: requestHeaders } });
}
```

### 4.5 The Clerk/Auth.js Middleware Pattern

If you use Clerk or Auth.js, their middleware wraps yours:

```typescript
// middleware.ts with Clerk
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isPublicRoute = createRouteMatcher([
  '/',
  '/auth/sign-in(.*)',
  '/auth/sign-up(.*)',
  '/api/webhooks(.*)',
  '/blog(.*)',
]);

export default clerkMiddleware(async (auth, request) => {
  if (!isPublicRoute(request)) {
    await auth.protect(); // Redirects to sign-in if not authenticated
  }
});

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

```typescript
// middleware.ts with Auth.js (NextAuth v5)
import { auth } from '@/auth';

export default auth((req) => {
  const isLoggedIn = !!req.auth;
  const isPublicPath = req.nextUrl.pathname === '/' ||
    req.nextUrl.pathname.startsWith('/auth/');

  if (!isLoggedIn && !isPublicPath) {
    return Response.redirect(new URL('/auth/sign-in', req.nextUrl));
  }
});

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

### 4.6 Edge Middleware vs Node.js Middleware

By default, Next.js middleware runs on the Edge Runtime. This means:
- It is fast (cold starts under 25ms)
- It runs close to the user (global edge locations)
- It has limited APIs (no `fs`, no `child_process`, limited `crypto`)
- It has limited library compatibility (no native Node.js modules)

If you need full Node.js APIs in middleware (for example, to use a database driver that requires TCP connections), you can opt into the Node.js runtime:

```typescript
// middleware.ts
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
  // Uncomment to use Node.js runtime instead of Edge:
  // runtime: 'nodejs',
};
```

**Recommendation**: Keep middleware on Edge Runtime. Use JWT validation (which works on Edge) for the fast path. If you need database lookups, use a REST/HTTP-compatible database client (like Prisma with Accelerate, or Neon's HTTP driver, or Upstash Redis).

---

## 5. SERVER COMPONENT AUTH

Server Components are where SSR auth shines. They can read cookies directly, validate sessions, fetch user data, and render HTML with full auth context -- no loading states, no flashing, no client-side fetch.

### 5.1 Reading Cookies in Server Components

```typescript
// app/dashboard/page.tsx
import { cookies } from 'next/headers';
import { verifySession } from '@/lib/session';
import { db } from '@/lib/db';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get('session')?.value;

  if (!sessionToken) {
    redirect('/auth/sign-in?redirect=/dashboard');
  }

  const session = await verifySession(sessionToken);

  if (!session) {
    redirect('/auth/sign-in?redirect=/dashboard');
  }

  // Now we have the authenticated user
  const user = await db.user.findUnique({
    where: { id: session.userId },
    include: { dashboardMetrics: true },
  });

  return (
    <div>
      <h1>Welcome back, {user?.name}</h1>
      <DashboardMetrics data={user?.dashboardMetrics} />
    </div>
  );
}
```

### 5.2 The `auth()` Helper Pattern

You do not want to repeat cookie reading and session validation in every Server Component. Create a cached helper:

```typescript
// lib/auth.ts
import { cookies } from 'next/headers';
import { cache } from 'react';
import { verifySession, type SessionPayload } from '@/lib/session';
import { db } from '@/lib/db';

// React.cache() ensures this runs ONCE per request, even if called from
// multiple Server Components in the same render tree
export const getSession = cache(async (): Promise<SessionPayload | null> => {
  const cookieStore = await cookies();
  const token = cookieStore.get('session')?.value;
  if (!token) return null;
  return verifySession(token);
});

export const getUser = cache(async () => {
  const session = await getSession();
  if (!session) return null;

  return db.user.findUnique({
    where: { id: session.userId },
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      avatarUrl: true,
    },
  });
});

// Convenience: throw redirect if not authenticated
export async function requireAuth() {
  const session = await getSession();
  if (!session) {
    const { redirect } = await import('next/navigation');
    redirect('/auth/sign-in');
  }
  return session;
}
```

### 5.3 Why `React.cache()` Is Critical

Without `React.cache()`, if you have a layout that calls `getUser()` and a page that also calls `getUser()`, the session is validated twice and the user is fetched from the database twice in the same request. With `React.cache()`, the function runs once and the result is memoized for the duration of the request.

```tsx
// app/dashboard/layout.tsx
import { getUser } from '@/lib/auth';
import { AuthProvider } from '@/components/auth-provider';

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser(); // First call: runs the function

  return (
    <AuthProvider user={user}>
      <nav>
        <span>Logged in as {user?.name}</span>
      </nav>
      {children}
    </AuthProvider>
  );
}

// app/dashboard/page.tsx
import { getUser } from '@/lib/auth';

export default async function DashboardPage() {
  const user = await getUser(); // Second call: returns cached result (no DB query)

  return (
    <div>
      <h1>Dashboard for {user?.name}</h1>
      {/* ... */}
    </div>
  );
}
```

**Important**: `React.cache()` scopes to a single request. There is no risk of leaking data between users. Each request gets its own cache.

### 5.4 Passing Auth Data to Client Components

This is the recommended pattern for bridging the server/client gap. The Server Component does the auth work and passes the result to Client Components as serializable props.

```tsx
// app/dashboard/layout.tsx (Server Component)
import { getUser } from '@/lib/auth';
import { AuthProvider } from '@/components/auth-provider';
import { redirect } from 'next/navigation';

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser();

  if (!user) {
    redirect('/auth/sign-in');
  }

  // Serialize only what the client needs
  // Do NOT pass sensitive data (password hash, internal IDs, etc.)
  const clientUser = {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    avatarUrl: user.avatarUrl,
  };

  return (
    <AuthProvider user={clientUser}>
      {children}
    </AuthProvider>
  );
}
```

```tsx
// components/auth-provider.tsx
'use client';

import { createContext, useContext, type ReactNode } from 'react';

export interface AuthUser {
  id: string;
  name: string;
  email: string;
  role: string;
  avatarUrl: string | null;
}

interface AuthContextValue {
  user: AuthUser | null;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextValue>({
  user: null,
  isAuthenticated: false,
});

export function AuthProvider({
  user,
  children,
}: {
  user: AuthUser | null;
  children: ReactNode;
}) {
  return (
    <AuthContext.Provider value={{ user, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  return useContext(AuthContext);
}
```

Now any Client Component inside the dashboard layout can use `useAuth()`:

```tsx
// components/user-menu.tsx
'use client';

import { useAuth } from '@/components/auth-provider';
import { signOut } from '@/app/auth/actions';

export function UserMenu() {
  const { user, isAuthenticated } = useAuth();

  if (!isAuthenticated || !user) return null;

  return (
    <div>
      <img src={user.avatarUrl ?? '/default-avatar.png'} alt={user.name} />
      <span>{user.name}</span>
      <button onClick={() => signOut()}>Sign out</button>
    </div>
  );
}
```

No loading state. No useEffect to fetch auth. No hydration mismatch. The server rendered the user's name, the client hydrates with the same data from the AuthProvider props.

---

## 6. CLIENT COMPONENT AUTH — THE HARD PART

The server can read cookies and access auth state directly. Client Components cannot read httpOnly cookies. Here are the four patterns for getting auth state to the client, ranked from best to worst.

### 6.1 Option 1: Server-Rendered Props (Recommended)

This is what we built in section 5.4. The Server Component fetches auth data, passes it as props to a Client Component AuthProvider, and all child Client Components use the context.

```
Server Component (reads cookie, validates session, fetches user)
  └─ AuthProvider (Client Component, receives user as prop)
       └─ useAuth() works in all children
```

**Pros**:
- No extra network requests
- No loading state
- No hydration mismatch (server and client see the same data)
- The most performant option

**Cons**:
- Auth data is only as fresh as the page load. If the user's role changes while they are on the page, the client will not know until the next navigation.
- Requires the Client Component to be a child of a Server Component that does the auth work

**When to use**: Almost always. This should be your default.

### 6.2 Option 2: Fetch `/api/me` on Mount

If you cannot use the Server Component pattern (maybe you are building a library, or you have Client Components that are not children of an auth-aware Server Component), you can fetch auth state from an API endpoint.

```typescript
// app/api/me/route.ts
import { getUser } from '@/lib/auth';
import { NextResponse } from 'next/server';

export async function GET() {
  const user = await getUser();
  if (!user) {
    return NextResponse.json({ user: null }, { status: 401 });
  }
  return NextResponse.json({
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      avatarUrl: user.avatarUrl,
    },
  });
}
```

```tsx
// hooks/use-auth-fetch.ts
'use client';

import { useEffect, useState } from 'react';
import type { AuthUser } from '@/components/auth-provider';

export function useAuthFetch() {
  const [user, setUser] = useState<AuthUser | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch('/api/me')
      .then((res) => res.json())
      .then((data) => {
        setUser(data.user);
        setIsLoading(false);
      })
      .catch(() => {
        setUser(null);
        setIsLoading(false);
      });
  }, []);

  return { user, isLoading, isAuthenticated: !!user };
}
```

**Pros**:
- Works anywhere, no dependency on Server Component parents
- Auth state is fresh at the time of the fetch

**Cons**:
- Extra network request on every page load
- Loading state while fetching (flash of unauthenticated content)
- Hydration mismatch risk: server renders one thing, client renders "loading..." before the fetch completes
- The browser sends cookies automatically, so the API route can read the httpOnly cookie

**When to use**: When you genuinely cannot use the Server Component pattern. Rare in Next.js App Router projects.

### 6.3 Option 3: Non-httpOnly Flag Cookie

Set a second cookie alongside the httpOnly session cookie. This flag cookie is NOT httpOnly, so client JavaScript can read it. It contains no sensitive data -- just a signal that the user is authenticated.

```typescript
// When setting the session cookie (in your sign-in action):
cookieStore.set('session', sessionToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 30,
  path: '/',
});

cookieStore.set('auth_state', 'authenticated', {
  httpOnly: false, // CLIENT CAN READ THIS
  secure: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 30,
  path: '/',
});
```

```typescript
// Client-side: check if user appears to be logged in
function isClientAuthenticated(): boolean {
  return document.cookie.includes('auth_state=authenticated');
}
```

**Pros**:
- Client can check auth status synchronously, no fetch needed
- Can help prevent hydration mismatches (both server and client see the cookie)

**Cons**:
- The flag cookie can be out of sync with the session cookie (session expires but flag does not, or vice versa)
- Provides no user data (name, role, etc.) -- just a boolean
- An attacker who steals this cookie learns that the user has a session (though they cannot use it)
- You must keep the two cookies in sync (set together, delete together)

**When to use**: As a supplement to Option 1, for edge cases where Client Components need a quick synchronous check before the full auth context is available.

### 6.4 Option 4: Server Action Auth Check

Use `useActionState` with a Server Action that returns the current auth state:

```tsx
// actions/get-auth.ts
'use server';

import { getUser } from '@/lib/auth';

export async function getAuthState() {
  const user = await getUser();
  return {
    user: user ? {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
    } : null,
  };
}
```

```tsx
// components/auth-action-provider.tsx
'use client';

import { useEffect, useActionState } from 'react';
import { getAuthState } from '@/actions/get-auth';

export function useAuthAction() {
  const [state, dispatch] = useActionState(getAuthState, { user: null });

  useEffect(() => {
    dispatch();
  }, [dispatch]);

  return {
    user: state.user,
    isAuthenticated: !!state.user,
    refresh: dispatch,
  };
}
```

**Pros**:
- Uses Server Actions, which automatically include cookies
- The `refresh` function re-checks auth state when needed
- Naturally works with React's concurrent features

**Cons**:
- Same loading state issue as Option 2
- Server Action invocation on mount

**When to use**: When you want a simple way to re-check auth state after a mutation (e.g., after the user changes their profile).

### 6.5 The Recommended Combination

In practice, use Option 1 as your primary strategy and add Option 3 as a lightweight supplement:

```tsx
// app/layout.tsx (root layout)
import { getUser } from '@/lib/auth';
import { AuthProvider } from '@/components/auth-provider';

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser();

  const clientUser = user ? {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    avatarUrl: user.avatarUrl,
  } : null;

  return (
    <html lang="en">
      <body>
        <AuthProvider user={clientUser}>
          {children}
        </AuthProvider>
      </body>
    </html>
  );
}
```

This gives you:
- Server-rendered auth data on every page (no loading states)
- Client-side `useAuth()` that works immediately after hydration
- No extra network requests
- No hydration mismatches

---

## 7. HYDRATION MISMATCH HELL

Hydration mismatch is the #1 SSR bug category. It happens when the HTML rendered by the server does not match what React produces on the client during hydration. React expects to find exactly the same DOM structure, and when it does not, bad things happen.

### 7.1 What Hydration Mismatch Actually Is

```
Server renders:  <div><p>Welcome, Alice</p></div>
Client hydrates: <div><p>Welcome, </p></div>

React says: "Warning: Text content did not match. Server: 'Welcome, Alice' Client: 'Welcome, '"
```

At best, React patches the DOM to match the client output (discarding the server's HTML). At worst, it throws an error and the entire subtree re-renders from scratch. Either way, you lose the benefits of SSR.

### 7.2 Auth-Related Mismatches

The most common source of hydration mismatches in SSR apps is auth state. Here is a classic example:

```tsx
// BAD: This causes hydration mismatch
'use client';

import { useEffect, useState } from 'react';

export function Navbar() {
  const [user, setUser] = useState(null); // Client starts with null

  useEffect(() => {
    // Fetch user from API after mount
    fetch('/api/me').then(r => r.json()).then(data => setUser(data.user));
  }, []);

  return (
    <nav>
      <a href="/">Home</a>
      {user ? (
        <span>Welcome, {user.name}</span> // Server might have rendered this
      ) : (
        <a href="/sign-in">Sign In</a> // Client renders this during hydration
      )}
    </nav>
  );
}
```

What happens:
1. Server: does not run `useEffect`, but might render different HTML based on some other mechanism
2. Actually, if this is a standalone Client Component with no server-passed props, the server renders with `user = null` (the initial state), showing "Sign In"
3. Client hydrates with `user = null`, showing "Sign In" -- no mismatch here
4. After hydration, `useEffect` runs, fetches user, updates to "Welcome, Alice"
5. User sees a flash from "Sign In" to "Welcome, Alice"

That is not technically a hydration mismatch, but it is a bad UX (flash of unauthenticated content). The real mismatch happens when you try to be clever:

```tsx
// BAD: Actual hydration mismatch
'use client';

export function Navbar() {
  // Read non-httpOnly cookie directly
  const isLoggedIn = typeof window !== 'undefined'
    ? document.cookie.includes('auth_state=authenticated')
    : false;

  return (
    <nav>
      {isLoggedIn ? <span>Dashboard</span> : <a href="/sign-in">Sign In</a>}
    </nav>
  );
}
```

What happens:
1. Server: `typeof window !== 'undefined'` is `false`, so `isLoggedIn` is `false`, renders "Sign In"
2. Client: `typeof window !== 'undefined'` is `true`, reads cookie, `isLoggedIn` is `true`, renders "Dashboard"
3. MISMATCH: Server rendered "Sign In", client expects "Dashboard"

### 7.3 Date/Time Mismatches

The server and client are in different timezones. If you render dates during SSR, the output will differ:

```tsx
// BAD: Date mismatch
export function Timestamp() {
  return <span>{new Date().toLocaleString()}</span>;
  // Server (UTC): "4/7/2026, 2:00:00 AM"
  // Client (US Pacific): "4/6/2026, 7:00:00 PM"
  // MISMATCH
}
```

**Fix**: Use `useEffect` to render client-local dates after hydration, or always format in UTC on the server:

```tsx
'use client';

import { useState, useEffect } from 'react';

export function Timestamp({ iso }: { iso: string }) {
  const [formatted, setFormatted] = useState(iso); // Server-safe: use ISO string

  useEffect(() => {
    // After hydration, show local time
    setFormatted(new Date(iso).toLocaleString());
  }, [iso]);

  return <time dateTime={iso}>{formatted}</time>;
}
```

### 7.4 localStorage-Dependent UI

The server has no localStorage. Any component that reads localStorage during render will mismatch:

```tsx
// BAD: localStorage in render
'use client';

export function ThemeToggle() {
  const theme = typeof window !== 'undefined'
    ? localStorage.getItem('theme') ?? 'light'
    : 'light';

  return <button>{theme === 'dark' ? '🌙' : '☀️'}</button>;
  // If user's localStorage has theme=dark:
  // Server renders ☀️ (light, the fallback)
  // Client renders 🌙 (dark, from localStorage)
  // MISMATCH
}
```

**Fix**: Same pattern -- start with the server-safe default, update after hydration:

```tsx
'use client';

import { useState, useEffect } from 'react';

export function ThemeToggle() {
  const [theme, setTheme] = useState('light'); // Server-safe default

  useEffect(() => {
    const stored = localStorage.getItem('theme');
    if (stored) setTheme(stored);
  }, []);

  return (
    <button onClick={() => {
      const next = theme === 'dark' ? 'light' : 'dark';
      setTheme(next);
      localStorage.setItem('theme', next);
    }}>
      {theme === 'dark' ? 'Dark' : 'Light'} mode
    </button>
  );
}
```

**Better fix**: Use cookies for theme preference so the server can render the correct theme from the start:

```typescript
// middleware.ts
export function middleware(request: NextRequest) {
  const theme = request.cookies.get('theme')?.value ?? 'light';
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-theme', theme);
  return NextResponse.next({ request: { headers: requestHeaders } });
}

// app/layout.tsx (Server Component)
import { cookies } from 'next/headers';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const cookieStore = await cookies();
  const theme = cookieStore.get('theme')?.value ?? 'light';

  return (
    <html lang="en" data-theme={theme}>
      <body>{children}</body>
    </html>
  );
}
```

### 7.5 Random Values

```tsx
// BAD: Random values differ between server and client
export function RandomGreeting() {
  const greetings = ['Hello', 'Hi', 'Hey', 'Howdy'];
  const greeting = greetings[Math.floor(Math.random() * greetings.length)];
  return <h1>{greeting}!</h1>;
  // Server picks "Hey", client picks "Hello" -- MISMATCH
}
```

**Fix**: Use `useId()` for consistent IDs, or compute random values only on the client:

```tsx
'use client';

import { useState, useEffect } from 'react';

export function RandomGreeting() {
  const [greeting, setGreeting] = useState('Hello'); // Deterministic default

  useEffect(() => {
    const greetings = ['Hello', 'Hi', 'Hey', 'Howdy'];
    setGreeting(greetings[Math.floor(Math.random() * greetings.length)]);
  }, []);

  return <h1>{greeting}!</h1>;
}
```

### 7.6 suppressHydrationWarning

React provides an escape hatch: `suppressHydrationWarning`. It tells React to not warn about mismatches for that specific element.

```tsx
<time dateTime={iso} suppressHydrationWarning>
  {new Date(iso).toLocaleString()}
</time>
```

**Use sparingly**. This does not fix the mismatch -- it just silences the warning. React will still use the client's output, which means the server-rendered HTML flashes to a different value. It is acceptable for truly cosmetic differences (like date formatting) but not for structural mismatches.

### 7.7 Debugging Hydration Mismatches

The error messages are notoriously unhelpful. Here is how to actually find the cause:

**Step 1: Look at the error message**

React 19 has improved hydration error messages significantly. They now show you the specific HTML that mismatched:

```
Hydration failed because the server rendered HTML didn't match the client.
As a result this tree will be regenerated on the client.

In <div>:
  Server:  <p>Welcome, Alice</p>
  Client:  <p>Welcome, </p>
```

**Step 2: Binary search with `suppressHydrationWarning`**

If the error message is not specific enough, add `suppressHydrationWarning` to parent elements, narrowing down until you find the exact element causing the mismatch.

**Step 3: Check the usual suspects**

- `typeof window !== 'undefined'` checks that change render output
- `document.cookie` reads
- `localStorage.getItem()` reads
- `new Date()` without UTC
- `Math.random()`
- `navigator.userAgent` checks
- Media queries (`window.matchMedia`) -- the server does not know screen size
- Browser extension injections that modify the DOM

**Step 4: View source and compare**

Open the page, right-click, "View Page Source" to see the server-rendered HTML. Then inspect the DOM in DevTools to see what React produced on the client. Find the difference.

**Step 5: Use React DevTools**

The Profiler in React DevTools can show you which components re-rendered after hydration, which often points to the mismatch source.

### 7.8 The Universal Anti-Mismatch Pattern

When you have client-only data that the server cannot know, use this pattern:

```tsx
'use client';

import { useState, useEffect } from 'react';

export function ClientOnly({
  children,
  fallback = null,
}: {
  children: React.ReactNode;
  fallback?: React.ReactNode;
}) {
  const [isMounted, setIsMounted] = useState(false);

  useEffect(() => {
    setIsMounted(true);
  }, []);

  if (!isMounted) return fallback;
  return children;
}

// Usage:
<ClientOnly fallback={<div className="h-8 w-32 animate-pulse bg-gray-200 rounded" />}>
  <UserMenu />
</ClientOnly>
```

The server renders the fallback. The client renders the fallback during hydration (matching the server). Then `useEffect` fires, `isMounted` becomes true, and the real content appears. No mismatch.

**Use this judiciously**. If you wrap everything in `<ClientOnly>`, you have defeated the purpose of SSR. The goal is to minimize the amount of UI that requires this pattern.

---

## 8. STATE MANAGEMENT IN SSR

State management libraries were designed for the client. Bringing them into SSR introduces a class of bugs that do not exist in client-only applications.

### 8.1 React Context in SSR

React Context ONLY works in Client Components. Server Components cannot use `useContext` because they have no state and no lifecycle. But you can wrap a Server Component's output in a Client Component provider:

```tsx
// This is the pattern:
// Server Component → fetches data → passes to Client Provider → children use context

// app/layout.tsx (Server Component)
import { getUser } from '@/lib/auth';
import { getTheme } from '@/lib/theme';
import { Providers } from '@/components/providers';

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const [user, theme] = await Promise.all([getUser(), getTheme()]);

  return (
    <html lang="en">
      <body>
        <Providers user={user} theme={theme}>
          {children}
        </Providers>
      </body>
    </html>
  );
}

// components/providers.tsx (Client Component)
'use client';

import { AuthProvider } from './auth-provider';
import { ThemeProvider } from './theme-provider';
import type { AuthUser } from './auth-provider';

export function Providers({
  user,
  theme,
  children,
}: {
  user: AuthUser | null;
  theme: string;
  children: React.ReactNode;
}) {
  return (
    <ThemeProvider initialTheme={theme}>
      <AuthProvider user={user}>
        {children}
      </AuthProvider>
    </ThemeProvider>
  );
}
```

### 8.2 Zustand in SSR — The Singleton Trap

Zustand stores are typically created as singletons (module-level constants). In the browser, this is fine -- there is one user, one store. On the server, this is catastrophic.

The server handles multiple requests concurrently. If your Zustand store is a singleton, Request A writes user data to the store, and Request B reads Request A's user data. You have just leaked one user's data to another.

```typescript
// BAD: Singleton store leaks state between requests
import { create } from 'zustand';

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));
// On the server, this is shared across ALL concurrent requests
```

**The fix: create stores per-request using React Context**.

```typescript
// lib/stores/auth-store.ts
import { createStore } from 'zustand';

export interface AuthState {
  user: AuthUser | null;
  isAuthenticated: boolean;
}

export interface AuthActions {
  setUser: (user: AuthUser | null) => void;
  clearUser: () => void;
}

export type AuthStore = AuthState & AuthActions;

export function createAuthStore(initialUser: AuthUser | null = null) {
  return createStore<AuthStore>((set) => ({
    user: initialUser,
    isAuthenticated: !!initialUser,
    setUser: (user) => set({ user, isAuthenticated: !!user }),
    clearUser: () => set({ user: null, isAuthenticated: false }),
  }));
}
```

```tsx
// components/auth-store-provider.tsx
'use client';

import { createContext, useContext, useRef, type ReactNode } from 'react';
import { useStore } from 'zustand';
import { createAuthStore, type AuthStore } from '@/lib/stores/auth-store';
import type { AuthUser } from '@/components/auth-provider';

type AuthStoreApi = ReturnType<typeof createAuthStore>;

const AuthStoreContext = createContext<AuthStoreApi | null>(null);

export function AuthStoreProvider({
  user,
  children,
}: {
  user: AuthUser | null;
  children: ReactNode;
}) {
  const storeRef = useRef<AuthStoreApi | null>(null);

  if (!storeRef.current) {
    storeRef.current = createAuthStore(user);
  }

  return (
    <AuthStoreContext.Provider value={storeRef.current}>
      {children}
    </AuthStoreContext.Provider>
  );
}

export function useAuthStore<T>(selector: (state: AuthStore) => T): T {
  const store = useContext(AuthStoreContext);
  if (!store) {
    throw new Error('useAuthStore must be used within AuthStoreProvider');
  }
  return useStore(store, selector);
}
```

```tsx
// Usage in Server Component
import { getUser } from '@/lib/auth';
import { AuthStoreProvider } from '@/components/auth-store-provider';

export default async function Layout({ children }: { children: React.ReactNode }) {
  const user = await getUser();
  return (
    <AuthStoreProvider user={user}>
      {children}
    </AuthStoreProvider>
  );
}

// Usage in Client Component
'use client';
import { useAuthStore } from '@/components/auth-store-provider';

export function UserGreeting() {
  const userName = useAuthStore((s) => s.user?.name);
  return <span>Hello, {userName}</span>;
}
```

Each request creates a new store instance via the provider. No shared state. No leaks.

### 8.3 TanStack Query in SSR

TanStack Query (React Query) has a well-designed SSR story. The pattern is: prefetch on the server, dehydrate the cache to HTML, hydrate on the client.

```typescript
// lib/query-client.ts
import { QueryClient } from '@tanstack/react-query';

// Create a new QueryClient PER REQUEST on the server
// On the client, create once and reuse
function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // With SSR, we want to set a stale time so that
        // queries are not re-fetched immediately on the client
        staleTime: 60 * 1000, // 1 minute
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

export function getQueryClient() {
  if (typeof window === 'undefined') {
    // Server: always make a new query client
    return makeQueryClient();
  }
  // Browser: make a new client once and reuse it
  if (!browserQueryClient) {
    browserQueryClient = makeQueryClient();
  }
  return browserQueryClient;
}
```

```tsx
// components/query-provider.tsx
'use client';

import { QueryClientProvider } from '@tanstack/react-query';
import { getQueryClient } from '@/lib/query-client';
import type { ReactNode } from 'react';

export function QueryProvider({ children }: { children: ReactNode }) {
  const queryClient = getQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}
```

```tsx
// app/dashboard/page.tsx (Server Component with prefetching)
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { getQueryClient } from '@/lib/query-client';
import { getSession } from '@/lib/auth';
import { DashboardContent } from './dashboard-content';

export default async function DashboardPage() {
  const session = await getSession();
  const queryClient = getQueryClient();

  // Prefetch on the server -- the data is fetched once and embedded in the HTML
  await queryClient.prefetchQuery({
    queryKey: ['dashboard', session?.userId],
    queryFn: async () => {
      const res = await fetch(`${process.env.API_URL}/dashboard/${session?.userId}`, {
        headers: { Authorization: `Bearer ${session?.token}` },
      });
      return res.json();
    },
  });

  await queryClient.prefetchQuery({
    queryKey: ['notifications', session?.userId],
    queryFn: async () => {
      const res = await fetch(`${process.env.API_URL}/notifications/${session?.userId}`, {
        headers: { Authorization: `Bearer ${session?.token}` },
      });
      return res.json();
    },
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <DashboardContent />
    </HydrationBoundary>
  );
}
```

```tsx
// app/dashboard/dashboard-content.tsx (Client Component)
'use client';

import { useQuery } from '@tanstack/react-query';
import { useAuth } from '@/components/auth-provider';

export function DashboardContent() {
  const { user } = useAuth();

  // This query was prefetched on the server
  // On the client, it uses the dehydrated data immediately (no loading state)
  // After staleTime, it refetches in the background
  const { data: dashboard } = useQuery({
    queryKey: ['dashboard', user?.id],
    queryFn: async () => {
      const res = await fetch(`/api/dashboard/${user?.id}`);
      return res.json();
    },
  });

  const { data: notifications } = useQuery({
    queryKey: ['notifications', user?.id],
    queryFn: async () => {
      const res = await fetch(`/api/notifications/${user?.id}`);
      return res.json();
    },
  });

  return (
    <div>
      <h1>Welcome back, {user?.name}</h1>
      <div className="grid grid-cols-2 gap-4">
        <MetricsPanel data={dashboard?.metrics} />
        <NotificationList items={notifications?.items} />
      </div>
    </div>
  );
}
```

The key insight: `HydrationBoundary` serializes the prefetched data into the HTML. When TanStack Query initializes on the client, it finds the dehydrated data and uses it immediately. No loading state, no refetch, no flash. After `staleTime` expires, TanStack Query silently refetches in the background to keep data fresh.

### 8.4 The "Initial Data" Pattern

A simpler alternative to the full dehydration flow: pass server-fetched data as `initialData` to `useQuery`:

```tsx
// Server Component
import { db } from '@/lib/db';
import { getSession } from '@/lib/auth';
import { MetricsPanel } from './metrics-panel';

export default async function DashboardPage() {
  const session = await getSession();
  const metrics = await db.metrics.findMany({
    where: { userId: session?.userId },
    orderBy: { date: 'desc' },
    take: 30,
  });

  return <MetricsPanel initialData={metrics} userId={session?.userId ?? ''} />;
}
```

```tsx
// Client Component
'use client';

import { useQuery } from '@tanstack/react-query';

export function MetricsPanel({
  initialData,
  userId,
}: {
  initialData: Metric[];
  userId: string;
}) {
  const { data: metrics } = useQuery({
    queryKey: ['metrics', userId],
    queryFn: () => fetch(`/api/metrics/${userId}`).then(r => r.json()),
    initialData,
    staleTime: 60 * 1000,
  });

  return (
    <div>
      {metrics.map((m) => (
        <MetricCard key={m.id} metric={m} />
      ))}
    </div>
  );
}
```

**Trade-off**: Simpler than dehydration, but `initialData` does not track when it was fetched. TanStack Query treats it as if it was fetched at time zero, so it might be considered stale immediately unless you set `staleTime`.

### 8.5 URL State: The SSR-Safe State Mechanism

URL search parameters are the only state that both the server and client can read without any synchronization:

```tsx
// app/products/page.tsx (Server Component)
export default async function ProductsPage({
  searchParams,
}: {
  searchParams: Promise<{ category?: string; sort?: string; page?: string }>;
}) {
  const params = await searchParams;
  const category = params.category ?? 'all';
  const sort = params.sort ?? 'newest';
  const page = parseInt(params.page ?? '1', 10);

  const products = await db.product.findMany({
    where: category !== 'all' ? { category } : undefined,
    orderBy: sort === 'newest' ? { createdAt: 'desc' } : { price: 'asc' },
    skip: (page - 1) * 20,
    take: 20,
  });

  return (
    <div>
      <FilterBar category={category} sort={sort} />
      <ProductGrid products={products} />
      <Pagination page={page} />
    </div>
  );
}
```

```tsx
// components/filter-bar.tsx (Client Component)
'use client';

import { useRouter, useSearchParams, usePathname } from 'next/navigation';

export function FilterBar({ category, sort }: { category: string; sort: string }) {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  function updateFilter(key: string, value: string) {
    const params = new URLSearchParams(searchParams.toString());
    params.set(key, value);
    params.set('page', '1'); // Reset to page 1 on filter change
    router.push(`${pathname}?${params.toString()}`);
  }

  return (
    <div className="flex gap-4">
      <select
        value={category}
        onChange={(e) => updateFilter('category', e.target.value)}
      >
        <option value="all">All Categories</option>
        <option value="electronics">Electronics</option>
        <option value="clothing">Clothing</option>
      </select>
      <select
        value={sort}
        onChange={(e) => updateFilter('sort', e.target.value)}
      >
        <option value="newest">Newest</option>
        <option value="price-asc">Price: Low to High</option>
      </select>
    </div>
  );
}
```

URL state is:
- Shareable (copy the URL, get the same view)
- Bookmarkable
- SSR-compatible (server reads searchParams, client reads useSearchParams)
- No hydration mismatch risk (both sides read the same URL)
- Survives refresh

For filter state, sort state, pagination, tab selection, modal open/close -- prefer URL state.

---

## 9. MONITORING AND ANALYTICS IN SSR

When your code runs on both the server and the client, your monitoring and analytics strategy must account for both. You cannot use the same tools in both environments.

### 9.1 Server-Side Analytics

Browser analytics SDKs (Google Analytics, Segment, Mixpanel) do NOT work in Server Components. They rely on `window`, `document`, and browser APIs that do not exist on the server.

For server-side tracking, use server-side SDKs or HTTP APIs:

```typescript
// lib/analytics-server.ts
// Server-side analytics using HTTP API (works in Server Components and Server Actions)

interface ServerEvent {
  event: string;
  userId?: string;
  properties?: Record<string, unknown>;
  timestamp?: string;
}

export async function trackServerEvent(event: ServerEvent) {
  // Example: Segment HTTP API
  await fetch('https://api.segment.io/v1/track', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${Buffer.from(process.env.SEGMENT_WRITE_KEY + ':').toString('base64')}`,
    },
    body: JSON.stringify({
      userId: event.userId ?? 'anonymous',
      event: event.event,
      properties: event.properties,
      timestamp: event.timestamp ?? new Date().toISOString(),
    }),
  });
}

// Usage in Server Component:
export default async function DashboardPage() {
  const session = await getSession();

  // Track page view server-side
  await trackServerEvent({
    event: 'page_viewed',
    userId: session?.userId,
    properties: { page: '/dashboard', source: 'ssr' },
  });

  // ... render page
}

// Usage in Server Action:
export async function updateProfile(formData: FormData) {
  'use server';
  const session = await getSession();

  // ... update profile

  await trackServerEvent({
    event: 'profile_updated',
    userId: session?.userId,
    properties: { fields: ['name', 'bio'] },
  });
}
```

### 9.2 Client-Side Analytics

Standard browser SDKs work in Client Components:

```tsx
// components/analytics-provider.tsx
'use client';

import { useEffect } from 'react';
import { usePathname, useSearchParams } from 'next/navigation';

export function AnalyticsProvider({ children }: { children: React.ReactNode }) {
  const pathname = usePathname();
  const searchParams = useSearchParams();

  useEffect(() => {
    // Track page views on client-side navigation
    const url = `${pathname}${searchParams.toString() ? `?${searchParams.toString()}` : ''}`;
    trackClientPageView(url);
  }, [pathname, searchParams]);

  return children;
}

function trackClientPageView(url: string) {
  // Google Analytics 4
  if (typeof window !== 'undefined' && (window as any).gtag) {
    (window as any).gtag('event', 'page_view', {
      page_location: url,
    });
  }

  // Or Segment
  if (typeof window !== 'undefined' && (window as any).analytics) {
    (window as any).analytics.page({ url });
  }
}
```

### 9.3 The Dual-Tracking Pattern

In production, you want both:

```
Server-side events:           Client-side events:
- page_viewed (SSR)           - page_viewed (client navigation)
- action_performed            - button_clicked
- api_called                  - form_submitted
- error_occurred              - scroll_depth
                              - web_vitals
                              - element_viewed (intersection observer)
```

The server tracks what happened (data changes, API calls, errors). The client tracks what the user did (clicks, scrolls, form interactions).

**Deduplication**: If you track `page_viewed` on both server and client, you get double-counting. Options:
1. Only track `page_viewed` on the client (misses initial SSR page view if JS fails)
2. Only track `page_viewed` on the server (misses client-side navigations)
3. Track both with a `source` property and deduplicate in your analytics pipeline
4. Track server-side for the initial load, client-side for subsequent navigations (recommended)

```tsx
// app/layout.tsx (Server Component)
import { headers } from 'next/headers';
import { trackServerEvent } from '@/lib/analytics-server';
import { getSession } from '@/lib/auth';

export default async function Layout({ children }: { children: React.ReactNode }) {
  const session = await getSession();
  const headersList = await headers();
  const pathname = headersList.get('x-pathname') ?? '/';

  // Track initial SSR page view
  await trackServerEvent({
    event: 'page_viewed',
    userId: session?.userId,
    properties: { page: pathname, source: 'ssr', isInitialLoad: true },
  });

  return (
    <html lang="en">
      <body>
        <AnalyticsProvider>
          {children}
        </AnalyticsProvider>
      </body>
    </html>
  );
}
```

### 9.4 Error Monitoring: Sentry Across the Boundary

Sentry requires different initialization for server and client:

```typescript
// instrumentation.ts (Next.js instrumentation file)
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    // Server-side Sentry
    const Sentry = await import('@sentry/nextjs');
    Sentry.init({
      dsn: process.env.SENTRY_DSN,
      tracesSampleRate: 0.1,
      environment: process.env.NODE_ENV,
    });
  }

  if (process.env.NEXT_RUNTIME === 'edge') {
    // Edge runtime Sentry
    const Sentry = await import('@sentry/nextjs');
    Sentry.init({
      dsn: process.env.SENTRY_DSN,
      tracesSampleRate: 0.05,
      environment: process.env.NODE_ENV,
    });
  }
}
```

```tsx
// app/global-error.tsx (Client Component -- catches errors in the root layout)
'use client';

import * as Sentry from '@sentry/nextjs';
import { useEffect } from 'react';

export default function GlobalError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <div>
          <h2>Something went wrong</h2>
          <button onClick={() => reset()}>Try again</button>
        </div>
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/error.tsx (Client Component -- catches errors in the dashboard route)
'use client';

import * as Sentry from '@sentry/nextjs';
import { useEffect } from 'react';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    Sentry.captureException(error, {
      tags: { section: 'dashboard' },
    });
  }, [error]);

  return (
    <div>
      <h2>Dashboard error</h2>
      <p>We could not load your dashboard. Please try again.</p>
      <button onClick={() => reset()}>Retry</button>
    </div>
  );
}
```

For Server Components, errors are caught by the nearest `error.tsx` boundary. Sentry's Next.js SDK automatically instruments Server Components and Server Actions, but you should add manual context:

```typescript
// In a Server Action:
'use server';

import * as Sentry from '@sentry/nextjs';

export async function submitOrder(formData: FormData) {
  const session = await getSession();

  try {
    // ... process order
  } catch (error) {
    Sentry.captureException(error, {
      user: { id: session?.userId },
      tags: { action: 'submitOrder' },
      extra: { formData: Object.fromEntries(formData) },
    });
    return { error: 'Order failed. Please try again.' };
  }
}
```

### 9.5 Performance Monitoring

Performance is split across server and client:

**Server metrics** (available on the server only):
- TTFB (Time to First Byte)
- Server Component render time
- Database query duration
- Server Action execution time

**Client metrics** (available on the client only):
- LCP (Largest Contentful Paint)
- FID/INP (First Input Delay / Interaction to Next Paint)
- CLS (Cumulative Layout Shift)
- Hydration time

```tsx
// app/layout.tsx -- add Web Vitals reporting
import { SpeedInsights } from '@vercel/speed-insights/next';
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <SpeedInsights />
        <Analytics />
      </body>
    </html>
  );
}
```

For custom server timing:

```typescript
// lib/server-timing.ts
export function withServerTiming<T>(
  name: string,
  fn: () => Promise<T>,
): Promise<T> {
  const start = performance.now();
  return fn().finally(() => {
    const duration = performance.now() - start;
    console.log(`[server-timing] ${name}: ${duration.toFixed(2)}ms`);
    // In production, send to your metrics backend
  });
}

// Usage:
const metrics = await withServerTiming('db:metrics', () =>
  db.metrics.findMany({ where: { userId: session.userId } })
);
```

### 9.6 OpenTelemetry for Full-Stack Tracing

OpenTelemetry lets you trace a request from the browser through middleware, Server Components, database queries, and back:

```typescript
// instrumentation.ts
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const { NodeSDK } = await import('@opentelemetry/sdk-node');
    const { OTLPTraceExporter } = await import('@opentelemetry/exporter-trace-otlp-http');
    const { getNodeAutoInstrumentations } = await import('@opentelemetry/auto-instrumentations-node');

    const sdk = new NodeSDK({
      traceExporter: new OTLPTraceExporter({
        url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
      }),
      instrumentations: [
        getNodeAutoInstrumentations({
          '@opentelemetry/instrumentation-http': { enabled: true },
          '@opentelemetry/instrumentation-fetch': { enabled: true },
        }),
      ],
    });

    sdk.start();
  }
}
```

With OpenTelemetry configured, you get distributed traces that show:
```
Browser Request GET /dashboard (250ms total)
├── Middleware: auth validation (15ms)
├── Layout Server Component (180ms)
│   ├── getSession() (5ms) [cached]
│   └── getUser() (25ms)
│       └── Prisma: SELECT users (20ms)
├── Page Server Component (150ms)
│   ├── getSession() (0ms) [React.cache hit]
│   └── getDashboardMetrics() (120ms)
│       └── Prisma: SELECT metrics (110ms)
└── Client Hydration (45ms)
```

---

## 10. CSRF PROTECTION

When you use cookies for authentication, CSRF (Cross-Site Request Forgery) becomes a real threat. In the SPA world with `Authorization: Bearer` headers, CSRF was not an issue because the attacker's site could not set that header. With cookies, the browser sends them automatically -- including to requests initiated by malicious sites.

### 10.1 What CSRF Is

```
1. User is logged in to your-app.com (session cookie is set)
2. User visits evil-site.com (in a different tab)
3. evil-site.com has: <form action="https://your-app.com/api/transfer" method="POST">
                        <input name="amount" value="10000">
                        <input name="to" value="attacker-account">
                      </form>
                      <script>document.forms[0].submit()</script>
4. Browser submits the form to your-app.com WITH the session cookie
5. Your server sees a valid session and processes the transfer
```

The server cannot distinguish a legitimate request from a CSRF request because both carry valid cookies.

### 10.2 Defenses

#### SameSite Cookies (Primary Defense)

`SameSite=Lax` blocks the attack above because form submissions from cross-origin sites do not include the cookie. This handles the vast majority of CSRF scenarios.

```typescript
// This is why we set SameSite=Lax on session cookies:
cookieStore.set('session', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax', // Blocks cross-site form submissions
  maxAge: 60 * 60 * 24 * 30,
  path: '/',
});
```

#### Server Action CSRF Protection (Built-in)

Next.js Server Actions have built-in CSRF protection. They use the `Origin` header to verify that the request came from the same origin. If the `Origin` does not match, the request is rejected.

```typescript
// Server Actions are automatically protected:
'use server';

export async function transferFunds(formData: FormData) {
  // Next.js has already verified the Origin header by this point
  // If this was a cross-origin request, it never reached here
}
```

#### API Route CSRF Protection (Manual)

API routes (Route Handlers) do NOT have automatic CSRF protection. You need to implement it yourself:

```typescript
// lib/csrf.ts
import { headers } from 'next/headers';

export async function verifyCsrf(): boolean {
  const headersList = await headers();
  const origin = headersList.get('origin');
  const host = headersList.get('host');

  if (!origin) {
    // No Origin header -- this is a same-origin request or a simple GET
    // For state-changing operations, require Origin
    return false;
  }

  const originUrl = new URL(origin);
  const expectedHost = host?.split(':')[0];

  return originUrl.hostname === expectedHost;
}
```

```typescript
// app/api/transfer/route.ts
import { verifyCsrf } from '@/lib/csrf';
import { getSession } from '@/lib/auth';

export async function POST(request: Request) {
  // Verify CSRF
  if (!await verifyCsrf()) {
    return new Response('CSRF validation failed', { status: 403 });
  }

  const session = await getSession();
  if (!session) {
    return new Response('Unauthorized', { status: 401 });
  }

  // ... process the request
}
```

#### Double-Submit Cookie Pattern

For extra security, implement the double-submit cookie pattern. Set a random CSRF token as a cookie AND require it in the request body or header. An attacker cannot read the cookie value (due to same-origin policy), so they cannot include it in the request body.

```typescript
// lib/csrf.ts
import { cookies } from 'next/headers';
import { randomBytes } from 'crypto';

export async function generateCsrfToken(): Promise<string> {
  const token = randomBytes(32).toString('hex');
  const cookieStore = await cookies();
  cookieStore.set('csrf_token', token, {
    httpOnly: false, // Client needs to read this to include in requests
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
    maxAge: 60 * 60, // 1 hour
    path: '/',
  });
  return token;
}

export async function verifyCsrfToken(tokenFromRequest: string): Promise<boolean> {
  const cookieStore = await cookies();
  const tokenFromCookie = cookieStore.get('csrf_token')?.value;
  if (!tokenFromCookie || !tokenFromRequest) return false;
  return tokenFromCookie === tokenFromRequest;
}
```

```tsx
// components/csrf-form.tsx
'use client';

import { useEffect, useState } from 'react';

export function CsrfForm({
  action,
  children,
}: {
  action: string;
  children: React.ReactNode;
}) {
  const [csrfToken, setCsrfToken] = useState('');

  useEffect(() => {
    // Read the CSRF token from the non-httpOnly cookie
    const match = document.cookie.match(/csrf_token=([^;]+)/);
    if (match) setCsrfToken(match[1]);
  }, []);

  return (
    <form action={action} method="POST">
      <input type="hidden" name="csrf_token" value={csrfToken} />
      {children}
    </form>
  );
}
```

---

## 11. COOKIE MANAGEMENT PATTERNS

In a production application, you will have multiple cookies serving different purposes. Here is the full taxonomy.

### 11.1 Cookie Types

```typescript
// lib/cookies.ts

import { cookies } from 'next/headers';
import type { ResponseCookie } from 'next/dist/compiled/@edge-runtime/cookies';

// ============================================================
// SESSION COOKIE
// The core authentication cookie. httpOnly, secure, lax.
// ============================================================
export const SESSION_COOKIE: ResponseCookie = {
  name: 'session',
  value: '', // Set at runtime
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 30, // 30 days
  path: '/',
};

// ============================================================
// REFRESH TOKEN COOKIE
// Longer-lived, restricted path, stricter sameSite.
// Only sent to the refresh endpoint to minimize exposure.
// ============================================================
export const REFRESH_COOKIE: ResponseCookie = {
  name: 'refresh_token',
  value: '',
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  maxAge: 60 * 60 * 24 * 90, // 90 days
  path: '/api/auth/refresh',
};

// ============================================================
// AUTH STATE FLAG COOKIE
// Non-httpOnly so client JS can check "am I logged in?"
// Contains NO sensitive data.
// ============================================================
export const AUTH_STATE_COOKIE: ResponseCookie = {
  name: 'auth_state',
  value: '',
  httpOnly: false,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 30, // 30 days (matches session)
  path: '/',
};

// ============================================================
// PREFERENCES COOKIE
// Non-httpOnly so client can read theme/language.
// ============================================================
export const PREFERENCES_COOKIE: ResponseCookie = {
  name: 'preferences',
  value: '',
  httpOnly: false,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 365, // 1 year
  path: '/',
};

// ============================================================
// CONSENT COOKIE
// Non-httpOnly so client checks before loading analytics.
// ============================================================
export const CONSENT_COOKIE: ResponseCookie = {
  name: 'cookie_consent',
  value: '',
  httpOnly: false,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 365, // 1 year
  path: '/',
};
```

### 11.2 Cookie Utility Functions

```typescript
// lib/cookie-utils.ts

import { cookies } from 'next/headers';
import type { ResponseCookie } from 'next/dist/compiled/@edge-runtime/cookies';

/**
 * Set a cookie with full type safety.
 * Only works in Server Actions, Route Handlers, and middleware.
 */
export async function setCookie(
  template: ResponseCookie,
  value: string,
) {
  const cookieStore = await cookies();
  cookieStore.set({
    ...template,
    value,
  });
}

/**
 * Get a cookie value. Returns undefined if not found.
 * Works in Server Components, Server Actions, Route Handlers, and middleware.
 */
export async function getCookie(name: string): Promise<string | undefined> {
  const cookieStore = await cookies();
  return cookieStore.get(name)?.value;
}

/**
 * Delete a cookie. Must use the same path and domain as when it was set.
 * Only works in Server Actions, Route Handlers, and middleware.
 */
export async function deleteCookie(template: ResponseCookie) {
  const cookieStore = await cookies();
  cookieStore.set({
    ...template,
    value: '',
    maxAge: 0,
  });
}

/**
 * Parse a cookie value that contains JSON.
 */
export function parseCookieJson<T>(value: string | undefined): T | null {
  if (!value) return null;
  try {
    return JSON.parse(decodeURIComponent(value)) as T;
  } catch {
    return null;
  }
}

/**
 * Serialize a value to a cookie-safe JSON string.
 */
export function serializeCookieJson(value: unknown): string {
  return encodeURIComponent(JSON.stringify(value));
}
```

### 11.3 Client-Side Cookie Utilities

For non-httpOnly cookies that the client needs to read and write:

```typescript
// lib/client-cookies.ts
// Only import this in Client Components

export function getClientCookie(name: string): string | undefined {
  if (typeof document === 'undefined') return undefined;
  const match = document.cookie.match(new RegExp(`(^| )${name}=([^;]+)`));
  return match ? decodeURIComponent(match[2]) : undefined;
}

export function setClientCookie(
  name: string,
  value: string,
  options: {
    maxAge?: number;
    path?: string;
    secure?: boolean;
    sameSite?: 'strict' | 'lax' | 'none';
  } = {},
) {
  const parts = [
    `${name}=${encodeURIComponent(value)}`,
    `path=${options.path ?? '/'}`,
  ];
  if (options.maxAge !== undefined) parts.push(`max-age=${options.maxAge}`);
  if (options.secure) parts.push('secure');
  if (options.sameSite) parts.push(`samesite=${options.sameSite}`);
  document.cookie = parts.join('; ');
}

export function deleteClientCookie(name: string, path = '/') {
  document.cookie = `${name}=; path=${path}; max-age=0`;
}
```

### 11.4 Cookie Rotation

For long-lived sessions, rotate the session token periodically to limit the window of exposure if a token is compromised:

```typescript
// lib/session.ts
export async function rotateSessionIfNeeded(token: string): Promise<string | null> {
  const session = await verifySession(token);
  if (!session) return null;

  // If the token was issued more than 24 hours ago, rotate it
  const issuedAt = session.iat ? session.iat * 1000 : 0;
  const twentyFourHoursAgo = Date.now() - 24 * 60 * 60 * 1000;

  if (issuedAt < twentyFourHoursAgo) {
    // Issue a new token with the same claims
    return createSessionToken({
      userId: session.userId,
      role: session.role,
      email: session.email,
    });
  }

  return null; // No rotation needed
}
```

```typescript
// middleware.ts (add rotation)
export async function middleware(request: NextRequest) {
  // ... auth validation from section 4 ...

  const sessionCookie = request.cookies.get('session');
  if (!sessionCookie?.value) {
    // ... redirect to sign-in
  }

  const session = await verifySession(sessionCookie.value);
  if (!session) {
    // ... redirect to sign-in
  }

  // Attempt session rotation
  const newToken = await rotateSessionIfNeeded(sessionCookie.value);

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-user-id', session.userId);
  requestHeaders.set('x-user-role', session.role);

  const response = NextResponse.next({ request: { headers: requestHeaders } });

  // If the token was rotated, update the cookie
  if (newToken) {
    response.cookies.set('session', newToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 30,
      path: '/',
    });
  }

  return response;
}
```

### 11.5 Clearing All Auth Cookies on Logout

When the user signs out, clear ALL auth-related cookies. Missing one causes ghost sessions:

```typescript
// app/auth/actions.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';
import { db } from '@/lib/db';
import {
  SESSION_COOKIE,
  REFRESH_COOKIE,
  AUTH_STATE_COOKIE,
} from '@/lib/cookies';
import { deleteCookie } from '@/lib/cookie-utils';

export async function signOut() {
  const cookieStore = await cookies();
  const sessionToken = cookieStore.get('session')?.value;

  // Invalidate the session in the database (if using DB sessions)
  if (sessionToken) {
    await db.session.deleteMany({ where: { token: sessionToken } });
  }

  // Clear ALL auth cookies
  await deleteCookie(SESSION_COOKIE);
  await deleteCookie(REFRESH_COOKIE);
  await deleteCookie(AUTH_STATE_COOKIE);

  redirect('/auth/sign-in');
}
```

### 11.6 Cross-Subdomain Cookies

If you have `app.example.com` and `api.example.com`, both need access to the session cookie:

```typescript
// Set cookie with domain for subdomain sharing
cookieStore.set('session', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 30,
  path: '/',
  domain: '.example.com', // Leading dot is optional but explicit
  // This cookie is now sent to:
  // example.com, app.example.com, api.example.com, admin.example.com, etc.
});
```

**Warning**: This means ANY subdomain can read the cookie. If you have user-generated content on `user-content.example.com` or if you host untrusted code on a subdomain, they get the session cookie too. In that case, use a separate domain for untrusted content.

---

## 12. THE COMPLETE SSR AUTH SETUP

Here is every piece wired together. This is a complete, working SSR auth implementation. Copy it, adapt it, ship it.

### 12.1 Session Library

```typescript
// lib/session.ts
import { SignJWT, jwtVerify, type JWTPayload } from 'jose';
import { cache } from 'react';
import { cookies } from 'next/headers';

// ============================================================
// Configuration
// ============================================================
const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);
const SESSION_DURATION = '30d';
const SESSION_MAX_AGE = 60 * 60 * 24 * 30; // 30 days in seconds

// ============================================================
// Types
// ============================================================
export interface SessionPayload extends JWTPayload {
  userId: string;
  role: 'user' | 'admin' | 'editor';
  email: string;
}

// ============================================================
// Create a new session token
// ============================================================
export async function createSessionToken(
  payload: Pick<SessionPayload, 'userId' | 'role' | 'email'>,
): Promise<string> {
  return new SignJWT(payload)
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime(SESSION_DURATION)
    .sign(JWT_SECRET);
}

// ============================================================
// Verify a session token
// ============================================================
export async function verifySessionToken(
  token: string,
): Promise<SessionPayload | null> {
  try {
    const { payload } = await jwtVerify(token, JWT_SECRET);
    return payload as SessionPayload;
  } catch {
    return null;
  }
}

// ============================================================
// Get the current session (cached per request)
// ============================================================
export const getSession = cache(async (): Promise<SessionPayload | null> => {
  const cookieStore = await cookies();
  const token = cookieStore.get('session')?.value;
  if (!token) return null;
  return verifySessionToken(token);
});

// ============================================================
// Set session cookie (for Server Actions and Route Handlers)
// ============================================================
export async function setSessionCookie(token: string) {
  const cookieStore = await cookies();

  cookieStore.set('session', token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: SESSION_MAX_AGE,
    path: '/',
  });

  cookieStore.set('auth_state', 'authenticated', {
    httpOnly: false,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: SESSION_MAX_AGE,
    path: '/',
  });
}

// ============================================================
// Clear session cookies (for Server Actions and Route Handlers)
// ============================================================
export async function clearSessionCookies() {
  const cookieStore = await cookies();
  cookieStore.set('session', '', { maxAge: 0, path: '/' });
  cookieStore.set('auth_state', '', { maxAge: 0, path: '/' });
  cookieStore.set('refresh_token', '', { maxAge: 0, path: '/api/auth/refresh' });
}
```

### 12.2 User Data Access

```typescript
// lib/auth.ts
import { cache } from 'react';
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/session';
import { db } from '@/lib/db';

// ============================================================
// Types
// ============================================================
export interface AuthUser {
  id: string;
  name: string;
  email: string;
  role: 'user' | 'admin' | 'editor';
  avatarUrl: string | null;
}

// ============================================================
// Get the current user (cached per request)
// ============================================================
export const getUser = cache(async (): Promise<AuthUser | null> => {
  const session = await getSession();
  if (!session) return null;

  const user = await db.user.findUnique({
    where: { id: session.userId },
    select: {
      id: true,
      name: true,
      email: true,
      role: true,
      avatarUrl: true,
    },
  });

  return user;
});

// ============================================================
// Require auth -- redirects if not authenticated
// ============================================================
export async function requireAuth(): Promise<AuthUser> {
  const user = await getUser();
  if (!user) redirect('/auth/sign-in');
  return user;
}

// ============================================================
// Require specific role
// ============================================================
export async function requireRole(
  ...roles: AuthUser['role'][]
): Promise<AuthUser> {
  const user = await requireAuth();
  if (!roles.includes(user.role)) {
    redirect('/unauthorized');
  }
  return user;
}
```

### 12.3 Middleware

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifySessionToken } from '@/lib/session';

const PUBLIC_PATHS = new Set([
  '/',
  '/auth/sign-in',
  '/auth/sign-up',
  '/auth/forgot-password',
  '/pricing',
  '/blog',
  '/about',
]);

function isPublicPath(pathname: string): boolean {
  if (PUBLIC_PATHS.has(pathname)) return true;
  if (pathname.startsWith('/blog/')) return true;
  if (pathname.startsWith('/api/auth/')) return true;
  if (pathname.startsWith('/_next/')) return true;
  if (/\.(.+)$/.test(pathname)) return true;
  return false;
}

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  if (isPublicPath(pathname)) {
    return NextResponse.next();
  }

  const sessionToken = request.cookies.get('session')?.value;

  if (!sessionToken) {
    return redirectToSignIn(request, pathname);
  }

  const session = await verifySessionToken(sessionToken);

  if (!session) {
    const response = redirectToSignIn(request, pathname);
    response.cookies.delete('session');
    response.cookies.delete('auth_state');
    return response;
  }

  // Pass session data via headers for downstream Server Components
  const requestHeaders = new Headers(request.headers);
  requestHeaders.set('x-user-id', session.userId);
  requestHeaders.set('x-user-role', session.role);
  requestHeaders.set('x-pathname', pathname);

  return NextResponse.next({ request: { headers: requestHeaders } });
}

function redirectToSignIn(
  request: NextRequest,
  redirectPath: string,
): NextResponse {
  const signInUrl = new URL('/auth/sign-in', request.url);
  signInUrl.searchParams.set('redirect', redirectPath);
  return NextResponse.redirect(signInUrl);
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

### 12.4 Auth Provider (Client Component)

```tsx
// components/auth-provider.tsx
'use client';

import { createContext, useContext, useCallback, useState, type ReactNode } from 'react';
import type { AuthUser } from '@/lib/auth';

interface AuthContextValue {
  user: AuthUser | null;
  isAuthenticated: boolean;
  update: (user: AuthUser | null) => void;
}

const AuthContext = createContext<AuthContextValue>({
  user: null,
  isAuthenticated: false,
  update: () => {},
});

export function AuthProvider({
  user: initialUser,
  children,
}: {
  user: AuthUser | null;
  children: ReactNode;
}) {
  const [user, setUser] = useState<AuthUser | null>(initialUser);

  const update = useCallback((newUser: AuthUser | null) => {
    setUser(newUser);
  }, []);

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        update,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### 12.5 Root Layout

```tsx
// app/layout.tsx
import { getUser } from '@/lib/auth';
import { AuthProvider } from '@/components/auth-provider';
import type { Metadata } from 'next';
import './globals.css';

export const metadata: Metadata = {
  title: 'My App',
  description: 'Production SSR auth example',
};

export default async function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await getUser();

  return (
    <html lang="en">
      <body>
        <AuthProvider user={user}>
          {children}
        </AuthProvider>
      </body>
    </html>
  );
}
```

### 12.6 Auth Actions

```typescript
// app/auth/actions.ts
'use server';

import { redirect } from 'next/navigation';
import { z } from 'zod';
import { db } from '@/lib/db';
import {
  createSessionToken,
  setSessionCookie,
  clearSessionCookies,
} from '@/lib/session';
import { hashPassword, verifyPassword } from '@/lib/crypto';

// ============================================================
// Sign In
// ============================================================
const signInSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(1, 'Password is required'),
});

export async function signIn(
  prevState: { error: string | null },
  formData: FormData,
) {
  const parsed = signInSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!parsed.success) {
    return { error: parsed.error.errors[0].message };
  }

  const user = await db.user.findUnique({
    where: { email: parsed.data.email },
  });

  if (!user || !await verifyPassword(parsed.data.password, user.passwordHash)) {
    return { error: 'Invalid email or password.' };
  }

  const token = await createSessionToken({
    userId: user.id,
    role: user.role,
    email: user.email,
  });

  await setSessionCookie(token);

  const redirectTo = formData.get('redirect')?.toString() || '/dashboard';
  redirect(redirectTo);
}

// ============================================================
// Sign Up
// ============================================================
const signUpSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export async function signUp(
  prevState: { error: string | null },
  formData: FormData,
) {
  const parsed = signUpSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!parsed.success) {
    return { error: parsed.error.errors[0].message };
  }

  const existing = await db.user.findUnique({
    where: { email: parsed.data.email },
  });

  if (existing) {
    return { error: 'An account with this email already exists.' };
  }

  const passwordHash = await hashPassword(parsed.data.password);

  const user = await db.user.create({
    data: {
      name: parsed.data.name,
      email: parsed.data.email,
      passwordHash,
      role: 'user',
    },
  });

  const token = await createSessionToken({
    userId: user.id,
    role: user.role,
    email: user.email,
  });

  await setSessionCookie(token);
  redirect('/dashboard');
}

// ============================================================
// Sign Out
// ============================================================
export async function signOut() {
  await clearSessionCookies();
  redirect('/auth/sign-in');
}
```

### 12.7 Sign-In Page

```tsx
// app/auth/sign-in/page.tsx
'use client';

import { useActionState } from 'react';
import { useSearchParams } from 'next/navigation';
import { signIn } from '../actions';
import Link from 'next/link';

export default function SignInPage() {
  const searchParams = useSearchParams();
  const redirectTo = searchParams.get('redirect') ?? '/dashboard';
  const [state, formAction, isPending] = useActionState(signIn, { error: null });

  return (
    <div className="mx-auto max-w-md mt-20 p-6">
      <h1 className="text-2xl font-bold mb-6">Sign In</h1>

      <form action={formAction} className="space-y-4">
        <input type="hidden" name="redirect" value={redirectTo} />

        <div>
          <label htmlFor="email" className="block text-sm font-medium mb-1">
            Email
          </label>
          <input
            id="email"
            name="email"
            type="email"
            required
            autoComplete="email"
            className="w-full border rounded px-3 py-2"
          />
        </div>

        <div>
          <label htmlFor="password" className="block text-sm font-medium mb-1">
            Password
          </label>
          <input
            id="password"
            name="password"
            type="password"
            required
            autoComplete="current-password"
            className="w-full border rounded px-3 py-2"
          />
        </div>

        {state.error && (
          <div role="alert" className="text-red-600 text-sm">
            {state.error}
          </div>
        )}

        <button
          type="submit"
          disabled={isPending}
          className="w-full bg-blue-600 text-white rounded py-2 font-medium disabled:opacity-50"
        >
          {isPending ? 'Signing in...' : 'Sign In'}
        </button>
      </form>

      <p className="mt-4 text-sm text-gray-600">
        Do not have an account?{' '}
        <Link href="/auth/sign-up" className="text-blue-600">
          Sign up
        </Link>
      </p>
    </div>
  );
}
```

### 12.8 Protected Dashboard Page

```tsx
// app/dashboard/page.tsx
import { requireAuth } from '@/lib/auth';
import { db } from '@/lib/db';
import { DashboardClient } from './dashboard-client';

export default async function DashboardPage() {
  const user = await requireAuth(); // Redirects to sign-in if not authenticated

  const [metrics, recentActivity] = await Promise.all([
    db.metrics.findMany({
      where: { userId: user.id },
      orderBy: { date: 'desc' },
      take: 30,
    }),
    db.activity.findMany({
      where: { userId: user.id },
      orderBy: { createdAt: 'desc' },
      take: 10,
    }),
  ]);

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold mb-6">
        Welcome back, {user.name}
      </h1>

      <DashboardClient
        metrics={metrics}
        recentActivity={recentActivity}
        userName={user.name}
      />
    </div>
  );
}
```

```tsx
// app/dashboard/dashboard-client.tsx
'use client';

import { useAuth } from '@/components/auth-provider';
import { signOut } from '@/app/auth/actions';

interface DashboardClientProps {
  metrics: Array<{ id: string; value: number; label: string; date: string }>;
  recentActivity: Array<{ id: string; description: string; createdAt: string }>;
  userName: string;
}

export function DashboardClient({
  metrics,
  recentActivity,
  userName,
}: DashboardClientProps) {
  const { user } = useAuth();

  return (
    <div>
      <div className="flex justify-between items-center mb-8">
        <p className="text-gray-600">
          Signed in as {user?.email}
        </p>
        <button
          onClick={() => signOut()}
          className="text-red-600 hover:text-red-800"
        >
          Sign out
        </button>
      </div>

      <section className="mb-8">
        <h2 className="text-xl font-semibold mb-4">Your Metrics</h2>
        <div className="grid grid-cols-3 gap-4">
          {metrics.slice(0, 6).map((metric) => (
            <div key={metric.id} className="border rounded p-4">
              <p className="text-sm text-gray-600">{metric.label}</p>
              <p className="text-2xl font-bold">{metric.value}</p>
            </div>
          ))}
        </div>
      </section>

      <section>
        <h2 className="text-xl font-semibold mb-4">Recent Activity</h2>
        <ul className="space-y-2">
          {recentActivity.map((activity) => (
            <li key={activity.id} className="border rounded p-3">
              <p>{activity.description}</p>
              <p className="text-sm text-gray-500">
                {new Date(activity.createdAt).toLocaleDateString()}
              </p>
            </li>
          ))}
        </ul>
      </section>
    </div>
  );
}
```

### 12.9 Protected API Route

```typescript
// app/api/dashboard/[userId]/route.ts
import { getSession } from '@/lib/session';
import { db } from '@/lib/db';
import { NextResponse } from 'next/server';

export async function GET(
  request: Request,
  { params }: { params: Promise<{ userId: string }> },
) {
  const session = await getSession();
  const { userId } = await params;

  if (!session) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Users can only access their own data (unless they are admin)
  if (session.userId !== userId && session.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
  }

  const metrics = await db.metrics.findMany({
    where: { userId },
    orderBy: { date: 'desc' },
    take: 30,
  });

  return NextResponse.json({ metrics });
}
```

---

## 13. COMMON SSR AUTH BUGS AND FIXES

Every bug here is one I have seen in production, multiple times, across multiple teams. Bookmark this section.

### 13.1 "User is undefined on first render"

**Symptom**: Client Components show a flash of "no user" state before showing the user's data.

**Cause**: The Client Component is initializing its own auth state instead of receiving it from the Server Component parent.

```tsx
// BAD
'use client';
export function Navbar() {
  const [user, setUser] = useState(null); // Starts null, fetches later
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(d => setUser(d.user));
  }, []);
  // Flash: null → user data
}

// GOOD
'use client';
export function Navbar() {
  const { user } = useAuth(); // From AuthProvider, which got data from Server Component
  // No flash: user data is available immediately from server-rendered props
}
```

### 13.2 "Infinite redirect loop"

**Symptom**: The browser shows "too many redirects" or you see the URL flickering between `/dashboard` and `/auth/sign-in`.

**Cause**: Your middleware redirects unauthenticated users to `/auth/sign-in`, but `/auth/sign-in` is also matched by the middleware, which checks for a session cookie, does not find one (because the user is trying to sign in!), and redirects back to `/auth/sign-in`.

**Fix**: Exclude auth paths from middleware protection.

```typescript
// middleware.ts
// BAD: no exclusion
export async function middleware(request: NextRequest) {
  const session = request.cookies.get('session');
  if (!session) {
    return NextResponse.redirect(new URL('/auth/sign-in', request.url));
    // If the user is already on /auth/sign-in, this redirects again → loop
  }
}

// GOOD: exclude public paths
export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Do not require auth for these paths
  if (pathname.startsWith('/auth/') || pathname === '/') {
    return NextResponse.next();
  }

  const session = request.cookies.get('session');
  if (!session) {
    return NextResponse.redirect(new URL('/auth/sign-in', request.url));
  }
}
```

### 13.3 "Session works locally but not on Vercel"

**Symptom**: Auth works perfectly on `localhost:3000` but cookies are not sent or not set on the deployed Vercel URL.

**Causes and fixes**:

1. **Secure flag on HTTP**: If you hardcoded `secure: true` and your preview deployment is not on HTTPS (rare with Vercel, but possible with custom domains).
   Fix: `secure: process.env.NODE_ENV === 'production'`

2. **Domain mismatch**: You set `domain: 'localhost'` in the cookie and deployed to `your-app.vercel.app`.
   Fix: Do not set `domain` at all for single-domain apps. The browser defaults to the current domain.

3. **SameSite=None without Secure**: If you set `sameSite: 'none'`, you MUST also set `secure: true`. Otherwise the browser silently ignores the cookie.

4. **Cookie path mismatch**: You set `path: /dashboard` on the cookie but are trying to read it at `/api/me`.
   Fix: Use `path: '/'` for session cookies.

### 13.4 "Auth works on web but not in mobile browser"

**Symptom**: Users clicking links from email or social media arrive at the site appearing logged out, even though they have a valid session.

**Cause**: `SameSite=Strict`. When a user clicks a link from Gmail (which loads in Gmail's context) or from a social media app (which opens a webview), the navigation is cross-site. `SameSite=Strict` cookies are not sent.

**Fix**: Use `SameSite=Lax` for session cookies. Lax allows cookies on top-level navigations (link clicks) while still blocking cross-site sub-requests.

```typescript
// GOOD
cookieStore.set('session', token, {
  sameSite: 'lax', // NOT 'strict'
  // ...
});
```

### 13.5 "CORS errors with credentials"

**Symptom**: `Access to fetch at 'https://api.example.com' from origin 'https://app.example.com' has been blocked by CORS policy: The value of the 'Access-Control-Allow-Credentials' header in the response is '' which must be 'true' when the request's credentials mode is 'include'.`

**Cause**: You are making a credentialed cross-origin request (sending cookies to a different subdomain/origin), but the server does not include the proper CORS headers.

**Fix**: Both sides need to cooperate.

Client:
```typescript
fetch('https://api.example.com/data', {
  credentials: 'include', // Send cookies cross-origin
});
```

Server (API at `api.example.com`):
```typescript
// app/api/data/route.ts
export async function GET(request: Request) {
  const data = await getData();

  return new Response(JSON.stringify(data), {
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': 'https://app.example.com', // NOT '*'
      'Access-Control-Allow-Credentials': 'true',
      'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    },
  });
}
```

**Important**: When `Access-Control-Allow-Credentials` is `true`, you CANNOT use `Access-Control-Allow-Origin: *`. You must specify the exact origin.

### 13.6 "Analytics don't track SSR page views"

**Symptom**: Your analytics dashboard shows fewer page views than your server logs. SSR pages are not counted.

**Cause**: Google Analytics, Mixpanel, and other browser-based analytics fire during client-side hydration. If JavaScript fails to load (ad blocker, network issue, slow connection), the page view is never tracked. Also, Server Components do not run browser analytics code.

**Fix**: Implement dual tracking.

```tsx
// Server-side: track in your Server Component or middleware
export async function middleware(request: NextRequest) {
  // Track all page views server-side (immune to ad blockers)
  await trackServerEvent({
    event: 'page_viewed_server',
    properties: {
      path: request.nextUrl.pathname,
      userAgent: request.headers.get('user-agent'),
    },
  });

  return NextResponse.next();
}

// Client-side: also track for richer data (screen size, referrer, etc.)
// AnalyticsProvider from section 9.2
```

### 13.7 "Hydration mismatch only for logged-in users"

**Symptom**: The app works fine for anonymous users but throws hydration warnings for authenticated users.

**Cause**: A Client Component tries to determine auth state using a method that differs between server and client. Most commonly: checking `document.cookie` in the render function.

```tsx
// BAD
'use client';
export function Header() {
  // Server: typeof window is undefined, so isLoggedIn = false
  // Client: reads cookie, isLoggedIn might be true
  const isLoggedIn = typeof window !== 'undefined'
    ? document.cookie.includes('auth_state=authenticated')
    : false;

  return isLoggedIn ? <UserMenu /> : <SignInButton />;
  // If user is logged in: server renders SignInButton, client renders UserMenu = MISMATCH
}

// GOOD
'use client';
export function Header() {
  const { isAuthenticated } = useAuth(); // Same value on server and client
  return isAuthenticated ? <UserMenu /> : <SignInButton />;
}
```

### 13.8 "Session expired but user still sees old data"

**Symptom**: The user's session expires, but they continue seeing their dashboard (from the browser cache or stale React Query data) until they take an action that hits the server.

**Fix**: Implement proactive session checking on the client.

```tsx
// hooks/use-session-check.ts
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';

export function useSessionCheck(intervalMs = 5 * 60 * 1000) {
  const router = useRouter();

  useEffect(() => {
    const check = async () => {
      try {
        const res = await fetch('/api/auth/check', { credentials: 'same-origin' });
        if (res.status === 401) {
          router.push('/auth/sign-in?reason=session_expired');
        }
      } catch {
        // Network error -- do not redirect, they might be offline
      }
    };

    const interval = setInterval(check, intervalMs);

    // Also check when the tab becomes visible again
    const onVisibilityChange = () => {
      if (document.visibilityState === 'visible') check();
    };
    document.addEventListener('visibilitychange', onVisibilityChange);

    return () => {
      clearInterval(interval);
      document.removeEventListener('visibilitychange', onVisibilityChange);
    };
  }, [intervalMs, router]);
}
```

```typescript
// app/api/auth/check/route.ts
import { getSession } from '@/lib/session';
import { NextResponse } from 'next/server';

export async function GET() {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ authenticated: false }, { status: 401 });
  }
  return NextResponse.json({ authenticated: true }, { status: 200 });
}
```

### 13.9 "State leaks between users in production"

**Symptom**: User A sometimes sees User B's data. This happens intermittently and is impossible to reproduce locally.

**Cause**: A singleton store (Zustand, a global variable, a module-level cache) is shared across requests in the Node.js server process. On the server, the same process handles multiple requests concurrently. If you store user-specific data in a module-level variable, one request's data can leak to another.

```typescript
// BAD: Module-level variable leaks between requests
let currentUser: User | null = null;

export function setCurrentUser(user: User) {
  currentUser = user; // This is shared across all concurrent requests!
}

// BAD: Singleton Zustand store
const useStore = create<State>((set) => ({
  user: null,
  setUser: (user) => set({ user }),
}));
// On the server, ALL requests share this one store instance
```

**Fix**: Use `React.cache()` for per-request data, Context providers scoped to the request, or the per-request store pattern from section 8.2.

### 13.10 "Cookie not set after Server Action"

**Symptom**: You call `cookies().set()` in a Server Action but the cookie does not appear in the browser.

**Causes**:

1. **Calling `redirect()` before `cookies().set()` takes effect**: `redirect()` throws an error internally. If you redirect before the cookie setting is processed, it might not be included in the response.

   Fix: Set cookies before redirecting, and ensure the cookie setting is awaited.

   ```typescript
   'use server';
   export async function signIn(formData: FormData) {
     const token = await createSessionToken(/* ... */);
     const cookieStore = await cookies();
     cookieStore.set('session', token, { /* ... */ }); // Set first
     redirect('/dashboard'); // Then redirect
   }
   ```

2. **Cookie size exceeds 4KB**: The browser silently drops cookies that are too large.

   Fix: Keep your JWT lean or switch to database sessions.

3. **Domain mismatch**: The cookie's domain does not match the current domain.

   Fix: Do not set `domain` at all for single-domain apps.

---

## 14. SSR AUTH DECISION TREE

When you are building an SSR app and need to decide on your auth strategy, use this decision tree:

```
1. Do you need to protect server-rendered pages?
   ├── YES → Use middleware + httpOnly cookies
   └── NO → You might not need SSR auth (client-only auth is fine)

2. Do Client Components need auth state?
   ├── YES → Pass from Server Component via AuthProvider (Section 6.1)
   └── NO → Just read cookies in Server Components

3. Do you need instant session revocation?
   ├── YES → Use database sessions (not JWT alone)
   └── NO → JWT is fine for most apps

4. Do you have multiple subdomains?
   ├── YES → Set cookie domain to .example.com (Section 11.6)
   └── NO → Do not set domain attribute

5. Do users come from external links (email, social media)?
   ├── YES → Use SameSite=Lax (NOT Strict)
   └── NO → SameSite=Strict is fine

6. Do you need server-side analytics?
   ├── YES → Use HTTP APIs in Server Components/Actions (Section 9.1)
   └── NO → Client-side analytics are sufficient

7. Do you use Zustand or other singletons?
   ├── YES → Use per-request store pattern (Section 8.2)
   └── NO → React Context via providers is sufficient
```

---

## 15. QUICK REFERENCE: WHAT WORKS WHERE

```
+----------------------------+--------+--------+--------+--------+--------+
|                            | Server | Client | Middle | Route  | Server |
|                            | Comp.  | Comp.  | ware   | Handler| Action |
+----------------------------+--------+--------+--------+--------+--------+
| Read cookies()             | YES    | NO     | YES*   | YES    | YES    |
| Set cookies()              | NO     | NO     | YES*   | YES    | YES    |
| Delete cookies()           | NO     | NO     | YES*   | YES    | YES    |
| document.cookie (read)     | NO     | YES**  | NO     | NO     | NO     |
| document.cookie (write)    | NO     | YES    | NO     | NO     | NO     |
| useAuth() / useContext     | NO     | YES    | NO     | NO     | NO     |
| React.cache()              | YES    | NO     | NO     | YES    | YES    |
| headers()                  | YES    | NO     | YES*   | YES    | YES    |
| redirect()                 | YES    | NO     | YES    | NO     | YES    |
| fetch with credentials     | YES    | YES    | YES    | YES    | YES    |
| Access environment vars    | YES    | NO***  | YES    | YES    | YES    |
| Direct DB access           | YES    | NO     | MAYBE  | YES    | YES    |
| window / document          | NO     | YES    | NO     | NO     | NO     |
| localStorage               | NO     | YES    | NO     | NO     | NO     |
+----------------------------+--------+--------+--------+--------+--------+

* Middleware uses request.cookies API (slightly different from cookies())
** Only non-httpOnly cookies are visible to document.cookie
*** Only NEXT_PUBLIC_ prefixed variables are available in Client Components
```

---

## 16. THE MENTAL CHECKLIST

Before shipping an SSR app with auth, run through this checklist:

```
Cookie Configuration:
[ ] Session cookie is httpOnly
[ ] Session cookie is Secure (in production)
[ ] Session cookie is SameSite=Lax
[ ] Session cookie has appropriate maxAge
[ ] Session cookie path is / (unless intentionally restricted)
[ ] No sensitive data in non-httpOnly cookies
[ ] Cookie domain is correct for your subdomain setup

Middleware:
[ ] Auth paths (/auth/*) are excluded from protection
[ ] Static assets (_next/*, favicons, etc.) are excluded
[ ] Invalid sessions are cleaned up (cookie deleted)
[ ] Redirect includes the original destination (?redirect=/dashboard)
[ ] Rate limiting on auth endpoints

Server Components:
[ ] Auth checks are cached with React.cache()
[ ] User data is passed to Client Components as serializable props
[ ] requireAuth() redirects (does not render unauthenticated state)

Client Components:
[ ] Auth state comes from AuthProvider props (not direct cookie reads)
[ ] No typeof window checks that change render output
[ ] No localStorage reads during initial render
[ ] No Date-dependent rendering without useEffect

Hydration:
[ ] Server and client produce identical initial HTML
[ ] Client-only UI is wrapped in useEffect/ClientOnly pattern
[ ] suppressHydrationWarning used only for cosmetic differences

State Management:
[ ] No singleton stores on the server
[ ] QueryClient created per-request on server, reused on client
[ ] Zustand stores wrapped in per-request Context providers

Security:
[ ] CSRF protection on state-changing API routes
[ ] Server Actions have Next.js automatic CSRF protection
[ ] No user data in error messages (enumeration prevention)
[ ] Logout clears ALL auth cookies

Monitoring:
[ ] Server-side error tracking (Sentry server SDK)
[ ] Client-side error tracking (Sentry client SDK)
[ ] Analytics on both server (initial load) and client (navigation)
[ ] Web Vitals collection
```

---

## Summary

SSR auth is hard. Not because you are doing it wrong, but because the model is genuinely complex. You are running the same code in two different worlds and expecting consistent results. The server has cookies but no browser APIs. The client has browser APIs but cannot read httpOnly cookies. They render at different times, and any discrepancy between them causes hydration mismatches.

The patterns in this chapter exist because thousands of engineers have hit these walls before you. The core principles are:

1. **httpOnly cookies for session tokens, always**. Security is not optional.
2. **Middleware as the first line of defense**. Validate sessions before pages render.
3. **Server Components do the auth work**. Read cookies, validate sessions, fetch user data.
4. **Pass auth data down as props**. Client Components receive auth state from server parents.
5. **`React.cache()` for per-request memoization**. Validate once, use everywhere.
6. **Never use singletons for request-scoped data on the server**. Each request is a different user.
7. **URL state is the SSR-safe state mechanism**. Both server and client can read it.
8. **When in doubt, delay client-only rendering to `useEffect`**. It prevents mismatches.

The gap between the server render and the client hydration is where every SSR auth bug lives. Close that gap by ensuring the server and client always agree on the same initial state, and you will save yourself hours of debugging.

Now go ship your SSR app. And when something breaks at 2 AM, come back to section 13.
