<!--
  CHAPTER: 33
  TITLE: Authentication & Authorization — From Login Screen to RBAC
  PART: V — Deployment & Operations
  PREREQS: Chapters 6, 22, 27
  KEY_TOPICS: OAuth2, PKCE, JWT, session management, refresh tokens, Clerk, Stack Auth, Supabase Auth, Firebase Auth, Auth.js, passkeys, biometric auth, RBAC, protected routes, middleware auth, token storage, social login, MFA, magic links
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 33: Authentication & Authorization — From Login Screen to RBAC

> **Part V — Deployment & Operations** | Prerequisites: Chapters 6, 22, 27 | Difficulty: Intermediate → Advanced

> "The best authentication system is one the user never thinks about and the attacker never stops thinking about."

---

<details>
<summary><strong>TL;DR</strong></summary>

- Use a managed auth provider (Clerk, Supabase Auth, or Firebase Auth) instead of rolling your own -- the security surface area of authentication is enormous, and managed providers handle password hashing, brute-force protection, token rotation, and compliance for you
- For mobile apps, OAuth2 with PKCE is mandatory; the implicit flow is deprecated and insecure. Use `expo-auth-session` with system browser -- never a WebView
- Clerk is the strongest choice for React Native + Next.js monorepos: first-class Expo SDK, Next.js middleware, organizations/multi-tenancy, and pre-built UI components on both platforms
- Store tokens in `expo-secure-store` on mobile, never in AsyncStorage; use httpOnly cookies on web. Implement automatic token refresh with API client interceptors and a mutex to prevent race conditions
- Apple Sign In is **required** by App Store policy if you offer any third-party social login; missing it is the number one auth-related rejection reason
- Protect routes at every layer: navigation guards in React Native, middleware in Next.js, `auth()` checks in Server Components, and Row Level Security or custom claims at the database level
- RBAC belongs in the backend -- conditional UI in the client is a convenience, not a security boundary

</details>

Authentication is the front door of your application. Get it right and users barely notice it exists -- they sign in once, their session persists, and they move on with their lives. Get it wrong and you face a spectrum of consequences: frustrated users who cannot log in, support tickets about expired sessions, App Store rejections for missing Apple Sign In, and -- worst case -- a breach that exposes your users' data.

This chapter is long because authentication touches everything. It is not a single feature you implement in a file and forget about. Authentication decisions affect your project structure, your API layer, your navigation flow, your database schema, your deployment configuration, and your App Store submission. Every chapter in this book eventually connects back to auth.

Here is what makes auth particularly challenging in a React Native + Next.js monorepo: you are building for two fundamentally different runtime environments. On the web, you have httpOnly cookies, server-side sessions, and middleware. On mobile, you have secure key storage, biometric prompts, and deep-link redirects. A good auth architecture bridges both worlds with shared types, shared business logic, and platform-specific implementations where necessary.

We will start with the landscape -- which auth providers exist and when to use each one. Then we will dive deep into the protocols (OAuth2 PKCE, JWT lifecycle, session management). Then we will build complete implementations with Clerk, Supabase, and Firebase. Finally, we will layer on RBAC, MFA, passkeys, and the end-to-end architecture that ties it all together.

### In This Chapter
- The Auth Landscape -- Clerk, Supabase Auth, Firebase Auth, Auth.js, Stack Auth, custom JWT: when to use each
- OAuth2 PKCE for Mobile -- why implicit flow is dead, the complete PKCE flow, expo-auth-session
- Clerk (Recommended) -- full monorepo setup for React Native + Next.js, social login, MFA, organizations
- Supabase Auth -- email/password, magic links, OAuth, Row Level Security, React Native client
- Firebase Auth -- phone auth, Google/Apple Sign-In, custom tokens, ID token verification
- Auth.js (NextAuth) -- v5 with App Router, providers, session strategies, middleware
- Token Management -- access/refresh lifecycle, secure storage, automatic refresh interceptors
- Social Login -- Apple Sign In (required), Google Sign In, platform requirements
- Passkeys & Biometric Auth -- WebAuthn/FIDO2, expo-local-authentication, when to use each
- Protected Routes -- React Native navigation guards, Next.js middleware, Server Component checks
- RBAC -- roles, permissions, database-level enforcement, conditional UI
- MFA -- TOTP, SMS OTP, email OTP with Clerk and Supabase
- Session Management -- cookies vs tokens, expiry strategies, concurrent sessions, force logout
- The Complete Architecture -- end-to-end monorepo example with everything wired together

### Related Chapters
- [Ch 6: Project Structure] -- monorepo layout that auth packages plug into
- [Ch 9: State Management] -- where auth state lives in your app
- [Ch 10: Data Fetching] -- API client interceptors for token refresh
- [Ch 22: Security & Data Protection] -- secure storage, certificate pinning, OWASP Mobile Top 10
- [Ch 27: Deployment & CI/CD] -- environment variables for auth secrets

---

## 1. THE AUTH LANDSCAPE IN 2026

### 1.1 The Case Against Rolling Your Own

Let me save you months of work and a potential security incident: **do not build your own authentication system.** I know this is a strong statement for a book about architecture. Here is why.

A "simple" email/password auth system requires you to implement:

- Password hashing (bcrypt/argon2 with proper cost factors)
- Brute-force protection (rate limiting per IP, per account, progressive delays)
- Password reset flows (secure token generation, expiry, single-use enforcement)
- Email verification (deliverability, link expiry, resend throttling)
- Token generation (JWT signing, refresh token rotation, revocation)
- Session management (concurrent session limits, force logout, device tracking)
- Account lockout (after N failed attempts, with unlock flow)
- CSRF protection (on web)
- Secure cookie configuration (httpOnly, Secure, SameSite, proper domain scoping)
- Password strength requirements (without being annoying)
- Credential stuffing detection
- Compliance (GDPR right to deletion, data export, audit logging)

That is just email/password. Add social login (Google, Apple, GitHub) and you also need:

- OAuth2 PKCE implementation
- ID token verification for each provider
- Account linking (user signs up with email, later tries Google with same email)
- Provider-specific quirks (Apple's private relay email, Google's one-tap)
- Nonce verification
- State parameter validation

Add MFA and you need TOTP generation, QR code rendering, backup codes, SMS delivery, and recovery flows. Add passkeys and you need WebAuthn ceremony handling, credential storage, and device attestation.

Each of these is a surface area for security vulnerabilities. Managed auth providers have teams of security engineers whose full-time job is hardening these systems. They run bug bounty programs. They have SOC 2 compliance. They patch vulnerabilities before you even know they exist.

**Use a managed provider.** Save your engineering time for the features that differentiate your product.

### 1.2 The Provider Landscape

Here is the honest comparison of every auth provider worth considering for a React Native + Next.js monorepo:

```
+------------------+----------+----------+----------+----------+----------+
|                  | Clerk    | Supabase | Firebase | Auth.js  | Stack    |
|                  |          | Auth     | Auth     | (v5)     | Auth     |
+------------------+----------+----------+----------+----------+----------+
| RN/Expo SDK      | Native   | JS only  | Native   | None     | JS only  |
|                  | (first-  | (@supa-  | (@react- | (API     | (new,    |
|                  | class)   | base/    | native-  | only)    | growing) |
|                  |          | ssr)     | firebase)|          |          |
+------------------+----------+----------+----------+----------+----------+
| Next.js Support  | Native   | Native   | Admin    | Native   | Native   |
|                  | middle-  | SSR      | SDK      | middle-  | middle-  |
|                  | ware     | helpers  | (server) | ware     | ware     |
+------------------+----------+----------+----------+----------+----------+
| Pre-built UI     | Yes      | Yes      | Drop-in  | No       | Yes      |
|                  | (RN +    | (Web     | (Web     |          | (Web     |
|                  | Web)     | only)    | only)    |          | only)    |
+------------------+----------+----------+----------+----------+----------+
| Social Login     | Google,  | Google,  | Google,  | Any      | Google,  |
|                  | Apple,   | Apple,   | Apple,   | OAuth2   | Apple,   |
|                  | GitHub,  | GitHub,  | GitHub,  | provider | GitHub   |
|                  | 20+      | Azure,   | Twitter, |          |          |
|                  |          | 15+      | Phone    |          |          |
+------------------+----------+----------+----------+----------+----------+
| MFA              | TOTP,    | TOTP,    | Phone,   | Custom   | TOTP     |
|                  | SMS,     | Phone    | TOTP     | (DIY)    |          |
|                  | Backup   |          |          |          |          |
|                  | codes    |          |          |          |          |
+------------------+----------+----------+----------+----------+----------+
| Organizations /  | Yes      | No       | No       | No       | Yes      |
| Multi-tenancy    | (native) | (manual) | (manual) |          | (basic)  |
+------------------+----------+----------+----------+----------+----------+
| Passkeys         | Yes      | No       | Yes      | Custom   | No       |
|                  |          |          | (limited)|          |          |
+------------------+----------+----------+----------+----------+----------+
| Pricing          | Free to  | Free to  | Free     | Free     | Free     |
| (free tier)      | 10K MAU  | 50K MAU  | (no MAU  | (OSS,    | (OSS,    |
|                  |          |          | limit,   | self-    | self-    |
|                  |          |          | then     | host)    | host)    |
|                  |          |          | pay/use) |          |          |
+------------------+----------+----------+----------+----------+----------+
| Pricing          | $0.02/   | $0.00325 | Free to  | $0       | $0       |
| (per MAU after   | MAU      | /MAU     | anon+    |          |          |
| free tier)       |          |          | phone    |          |          |
|                  |          |          | costs    |          |          |
+------------------+----------+----------+----------+----------+----------+
| Vendor Lock-in   | Medium   | Low      | High     | None     | Low      |
|                  |          | (OSS,    | (Google  | (OSS)    | (OSS)    |
|                  |          | self-    | eco-     |          |          |
|                  |          | host)    | system)  |          |          |
+------------------+----------+----------+----------+----------+----------+
| Best For         | Cross-   | Full-    | Google   | Web-     | OSS      |
|                  | platform | stack    | ecosystem| only     | self-    |
|                  | monorepo | Supabase | apps,    | Next.js  | hosted   |
|                  | apps     | apps     | phone    | apps     | apps     |
|                  |          |          | auth     |          |          |
+------------------+----------+----------+----------+----------+----------+
```

### 1.3 The Decision Matrix

**Choose Clerk if:**
- You are building a React Native + Next.js monorepo (the primary audience of this book)
- You need pre-built sign-in/sign-up UI on both mobile and web
- You want organizations / multi-tenancy out of the box
- You want MFA, passkeys, and social login without writing ceremony code
- You can accept the $0.02/MAU cost after 10K users

**Choose Supabase Auth if:**
- You are already using Supabase for your database and want auth tightly integrated
- Row Level Security is central to your authorization model
- You want the option to self-host everything
- You want a generous free tier (50K MAU)
- You are comfortable building your own sign-in UI on mobile

**Choose Firebase Auth if:**
- You are deep in the Google ecosystem (Firestore, Cloud Functions, FCM)
- Phone/SMS authentication is a primary login method
- You want anonymous auth for progressive onboarding
- You need to support a very large user base at low cost (generous free tier)

**Choose Auth.js if:**
- You are building a Next.js-only web app (no React Native)
- You want maximum flexibility over providers and session strategy
- You want zero vendor lock-in (it is open source, you control everything)
- You are comfortable not having pre-built UI components

**Choose Stack Auth if:**
- You want an open-source, self-hostable Clerk alternative
- You are early-stage and want to avoid any auth cost
- You can accept a less mature ecosystem and smaller community

**Choose custom JWT if:**
- You have a dedicated security team
- You have regulatory requirements that prevent using third-party auth
- You enjoy pain (only half joking)

For the rest of this chapter, we will go deep on Clerk (recommended for most readers of this book), Supabase Auth, and Firebase Auth, with a section on Auth.js for Next.js-only scenarios.

---

## 2. OAUTH2 PKCE FOR MOBILE

Before we dive into specific providers, you need to understand the protocol that underpins every social login on mobile: OAuth2 with PKCE (Proof Key for Code Exchange, pronounced "pixie").

### 2.1 Why the Implicit Flow Is Dead

If you have worked with OAuth on the web, you might remember the implicit flow -- the one where the access token comes back directly in the URL fragment (`#access_token=...`). It was convenient. It was also a security disaster.

The implicit flow has been formally deprecated by the OAuth 2.0 Security Best Current Practice (RFC 9700) and the IETF. Here is why:

1. **Tokens in URLs are leaked.** Browser history, referrer headers, proxy logs, and shoulder surfing all expose tokens in URL fragments.
2. **No client authentication.** There is no way to verify that the token is being received by the legitimate client.
3. **No refresh tokens.** Implicit flow cannot issue refresh tokens, so the user experience degrades (forced re-authentication).
4. **Token replay.** An intercepted token can be replayed by any party.

On mobile, the problems multiply:

5. **Custom URL scheme hijacking.** On Android, any app can register the same custom URL scheme (`myapp://callback`). A malicious app could intercept the redirect and steal the token.
6. **No secure back-channel.** Mobile apps do not have a server-side component in the traditional sense, so the token is always exposed to the client.

### 2.2 The PKCE Flow

PKCE (RFC 7636) solves these problems by adding a proof mechanism to the authorization code flow. Even if an attacker intercepts the authorization code, they cannot exchange it for tokens without the proof.

Here is the complete flow:

```
┌──────────────┐                              ┌──────────────┐
│  Mobile App  │                              │  Auth Server  │
│  (Client)    │                              │  (Provider)   │
└──────┬───────┘                              └──────┬───────┘
       │                                              │
       │  1. Generate code_verifier (random string)   │
       │     Generate code_challenge =                │
       │       BASE64URL(SHA256(code_verifier))       │
       │                                              │
       │  2. Open system browser with:                │
       │     /authorize?                              │
       │       response_type=code                     │
       │       &client_id=xxx                         │
       │       &redirect_uri=myapp://callback         │
       │       &code_challenge=yyy                    │
       │       &code_challenge_method=S256            │
       │       &state=random_state                    │
       │────────────────────────────────────────────▶│
       │                                              │
       │  3. User authenticates in system browser     │
       │     (enters credentials, consents)           │
       │                                              │
       │  4. Auth server redirects to:                │
       │     myapp://callback?code=zzz&state=...      │
       │◀────────────────────────────────────────────│
       │                                              │
       │  5. App receives code via deep link           │
       │     Verifies state matches                   │
       │                                              │
       │  6. Exchange code for tokens:                │
       │     POST /token                              │
       │       grant_type=authorization_code           │
       │       &code=zzz                              │
       │       &redirect_uri=myapp://callback         │
       │       &code_verifier=original_verifier       │
       │────────────────────────────────────────────▶│
       │                                              │
       │  7. Server verifies:                         │
       │     BASE64URL(SHA256(code_verifier))         │
       │       === stored code_challenge              │
       │                                              │
       │  8. Returns access_token + refresh_token     │
       │◀────────────────────────────────────────────│
       │                                              │
       │  9. Store tokens in Secure Store              │
       │                                              │
```

**Why this is secure:**

- The `code_verifier` is generated on the device and never sent over the network during the authorization step.
- The `code_challenge` (a hash of the verifier) is sent in step 2 but cannot be reversed to obtain the verifier.
- In step 6, only the app that generated the original `code_verifier` can complete the exchange.
- Even if a malicious app intercepts the authorization code in step 4, it does not have the `code_verifier` and cannot exchange the code for tokens.

### 2.3 System Browser vs WebView

**Always use the system browser. Never use a WebView for OAuth.**

This is not just a best practice -- it is a security requirement. Here is why:

| System Browser | WebView (Embedded) |
|---|---|
| User sees the real URL bar and can verify they are on the real provider | App controls what the user sees; can spoof the login page |
| Shared cookie jar means SSO works (already signed into Google) | Isolated cookie jar; user must re-enter credentials every time |
| App cannot access user's credentials | App has full access to the WebView DOM and can steal credentials |
| Required by Google's OAuth policies | **Rejected by Google** since 2021 |
| Required by Apple Sign In guidelines | Can cause App Store rejection |

Google explicitly blocks OAuth requests from embedded WebViews. If you try to use a WebView for Google Sign In, the user will see an error: "This browser or app may not be secure."

### 2.4 Implementation with expo-auth-session

`expo-auth-session` handles the PKCE flow for you, including code verifier generation, system browser launching, and redirect handling.

```bash
npx expo install expo-auth-session expo-crypto expo-web-browser
```

```typescript
// packages/auth/src/oauth-pkce.ts
import * as AuthSession from 'expo-auth-session';
import * as WebBrowser from 'expo-web-browser';
import * as Crypto from 'expo-crypto';

// Required for web browser redirect to close properly
WebBrowser.maybeCompleteAuthSession();

// Discovery documents for common providers
const GOOGLE_DISCOVERY = AuthSession.useAutoDiscovery(
  'https://accounts.google.com'
);

// Define your redirect URI
// In Expo, this is automatically handled based on your app scheme
const redirectUri = AuthSession.makeRedirectUri({
  scheme: 'myapp',
  path: 'auth/callback',
});

/**
 * Complete Google OAuth2 PKCE flow.
 *
 * This function:
 * 1. Generates code_verifier and code_challenge
 * 2. Opens system browser for Google sign-in
 * 3. Receives authorization code via redirect
 * 4. Exchanges code for tokens (via your backend)
 */
export function useGoogleAuth() {
  const discovery = AuthSession.useAutoDiscovery(
    'https://accounts.google.com'
  );

  const [request, response, promptAsync] = AuthSession.useAuthRequest(
    {
      clientId: process.env.EXPO_PUBLIC_GOOGLE_CLIENT_ID!,
      redirectUri,
      scopes: ['openid', 'profile', 'email'],
      // PKCE is enabled by default in expo-auth-session
      // code_verifier and code_challenge are generated automatically
      usePKCE: true,
      // Extra params for Google-specific behavior
      extraParams: {
        // Force account selection even if user has one Google account
        prompt: 'select_account',
      },
    },
    discovery
  );

  const handleResponse = async () => {
    if (response?.type === 'success') {
      const { code } = response.params;

      // IMPORTANT: Exchange the code on your BACKEND, not on the client.
      // The client sends the code and code_verifier to your server,
      // which makes the token exchange request.
      const tokenResponse = await fetch(
        `${process.env.EXPO_PUBLIC_API_URL}/auth/google/callback`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            code,
            code_verifier: request?.codeVerifier,
            redirect_uri: redirectUri,
          }),
        }
      );

      const tokens = await tokenResponse.json();
      // Store tokens securely (see Section 7)
      return tokens;
    }

    if (response?.type === 'error') {
      throw new Error(
        response.error?.message ?? 'OAuth authentication failed'
      );
    }

    return null; // User cancelled
  };

  return {
    isReady: !!request,
    promptAsync,
    handleResponse,
  };
}
```

```typescript
// Server-side: apps/api/src/routes/auth/google-callback.ts
import { Router } from 'express';

const router = Router();

router.post('/auth/google/callback', async (req, res) => {
  const { code, code_verifier, redirect_uri } = req.body;

  // Exchange the authorization code for tokens on the server
  // This keeps your client_secret off the mobile device
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri,
      client_id: process.env.GOOGLE_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CLIENT_SECRET!,
      code_verifier,
    }),
  });

  const tokens = await tokenResponse.json();

  if (tokens.error) {
    return res.status(400).json({ error: tokens.error_description });
  }

  // Verify the ID token
  const idTokenPayload = await verifyGoogleIdToken(tokens.id_token);

  // Find or create user in your database
  const user = await findOrCreateUser({
    email: idTokenPayload.email,
    name: idTokenPayload.name,
    googleId: idTokenPayload.sub,
    avatar: idTokenPayload.picture,
  });

  // Issue your own tokens (see Section 7)
  const appTokens = await issueTokens(user);

  return res.json(appTokens);
});
```

### 2.5 Expo Deep Linking Configuration

For the OAuth redirect to work, your app needs to handle deep links:

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "ios": {
      "bundleIdentifier": "com.yourcompany.myapp",
      "associatedDomains": ["applinks:auth.yourapp.com"]
    },
    "android": {
      "package": "com.yourcompany.myapp",
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "myapp",
              "host": "auth",
              "pathPrefix": "/callback"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

> **Critical:** On Android, custom URL schemes (`myapp://`) are not secure because any app can register the same scheme. For production apps, use Universal Links (iOS) or App Links (Android) with verified domains. The `autoVerify: true` and `associatedDomains` configuration above sets this up.

---

## 3. CLERK (RECOMMENDED FOR MOST APPS)

Clerk is the auth provider I recommend for React Native + Next.js monorepos. It has first-class Expo and Next.js SDKs, pre-built UI components for both platforms, organizations/multi-tenancy, and a developer experience that just works. The trade-off is cost ($0.02/MAU after 10K) and vendor lock-in, but for most teams the time saved is worth it.

### 3.1 Installation

```bash
# In your monorepo root
# React Native (Expo)
cd apps/mobile
npx expo install @clerk/clerk-expo expo-secure-store

# Next.js
cd apps/web
pnpm add @clerk/nextjs

# Shared types (optional but recommended)
cd packages/auth
pnpm add @clerk/types
```

### 3.2 Environment Variables

```bash
# .env.local (Next.js)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxx
CLERK_SECRET_KEY=sk_test_xxxxx
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/onboarding

# .env (React Native / Expo)
EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxxxx
```

> **Never put the Clerk Secret Key in your React Native app.** The secret key is for server-side use only (Next.js API routes, Server Actions, middleware). The publishable key is safe for client-side use.

### 3.3 React Native Setup

```typescript
// apps/mobile/src/providers/auth-provider.tsx
import { ClerkProvider, ClerkLoaded } from '@clerk/clerk-expo';
import * as SecureStore from 'expo-secure-store';

/**
 * Clerk token cache using expo-secure-store.
 * This ensures auth tokens survive app restarts and are stored securely
 * in the iOS Keychain / Android Keystore.
 */
const tokenCache = {
  async getToken(key: string): Promise<string | null> {
    try {
      const item = await SecureStore.getItemAsync(key);
      return item;
    } catch (error) {
      console.error('SecureStore getToken error:', error);
      await SecureStore.deleteItemAsync(key);
      return null;
    }
  },
  async saveToken(key: string, value: string): Promise<void> {
    try {
      await SecureStore.setItemAsync(key, value);
    } catch (error) {
      console.error('SecureStore saveToken error:', error);
    }
  },
};

const publishableKey = process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!;

if (!publishableKey) {
  throw new Error(
    'Missing EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY. ' +
    'Add it to your .env file.'
  );
}

export function AuthProvider({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider publishableKey={publishableKey} tokenCache={tokenCache}>
      <ClerkLoaded>
        {children}
      </ClerkLoaded>
    </ClerkProvider>
  );
}
```

```typescript
// apps/mobile/app/_layout.tsx (Expo Router)
import { Slot } from 'expo-router';
import { AuthProvider } from '../src/providers/auth-provider';

export default function RootLayout() {
  return (
    <AuthProvider>
      <Slot />
    </AuthProvider>
  );
}
```

### 3.4 Next.js Setup

```typescript
// apps/web/app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs';

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  );
}
```

```typescript
// apps/web/middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

// Define which routes require authentication
const isProtectedRoute = createRouteMatcher([
  '/dashboard(.*)',
  '/settings(.*)',
  '/api/protected(.*)',
]);

// Define which routes are always public
const isPublicRoute = createRouteMatcher([
  '/',
  '/sign-in(.*)',
  '/sign-up(.*)',
  '/api/webhooks(.*)',
  '/api/public(.*)',
]);

export default clerkMiddleware(async (auth, req) => {
  // If the route is protected, require authentication
  if (isProtectedRoute(req)) {
    await auth.protect();
  }
});

export const config = {
  // Match all routes except static files and _next
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
};
```

### 3.5 Authentication Hooks

Clerk provides hooks that work identically on React Native and Next.js (client components):

```typescript
// packages/auth/src/hooks/use-current-user.ts
// This works on BOTH React Native and Next.js client components
import { useAuth, useUser } from '@clerk/clerk-expo'; // or @clerk/nextjs

/**
 * Unified auth hook for both platforms.
 * Wraps Clerk's hooks with consistent typing.
 */
export function useCurrentUser() {
  const { isLoaded, isSignedIn, userId, getToken, signOut } = useAuth();
  const { user } = useUser();

  return {
    isLoaded,
    isSignedIn: isSignedIn ?? false,
    userId: userId ?? null,
    user: user
      ? {
          id: user.id,
          email: user.primaryEmailAddress?.emailAddress ?? null,
          name: user.fullName ?? null,
          avatar: user.imageUrl,
          role: (user.publicMetadata?.role as string) ?? 'viewer',
        }
      : null,
    /**
     * Get a JWT token for API calls.
     * On web, Clerk handles this via cookies.
     * On mobile, this retrieves from secure storage.
     */
    getToken,
    signOut,
  };
}

// Type for the user object, shareable across the monorepo
export interface AuthUser {
  id: string;
  email: string | null;
  name: string | null;
  avatar: string;
  role: string;
}
```

### 3.6 Sign-In Screen (React Native)

Clerk provides pre-built `<SignIn />` and `<SignUp />` components, but you can also build custom UI with their hooks for full design control:

```typescript
// apps/mobile/app/(auth)/sign-in.tsx
import { useSignIn } from '@clerk/clerk-expo';
import { useRouter } from 'expo-router';
import { useState } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  Alert,
  StyleSheet,
} from 'react-native';
import * as WebBrowser from 'expo-web-browser';
import { useOAuth } from '@clerk/clerk-expo';

WebBrowser.maybeCompleteAuthSession();

export default function SignInScreen() {
  const { signIn, setActive, isLoaded } = useSignIn();
  const router = useRouter();

  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  // OAuth hooks for social login
  const { startOAuthFlow: startGoogleOAuth } = useOAuth({
    strategy: 'oauth_google',
  });
  const { startOAuthFlow: startAppleOAuth } = useOAuth({
    strategy: 'oauth_apple',
  });

  const handleEmailSignIn = async () => {
    if (!isLoaded) return;
    setLoading(true);

    try {
      const result = await signIn.create({
        identifier: email,
        password,
      });

      if (result.status === 'complete') {
        // Set the active session
        await setActive({ session: result.createdSessionId });
        // Navigate to the main app
        router.replace('/(tabs)/home');
      } else if (result.status === 'needs_second_factor') {
        // MFA is required -- navigate to MFA screen
        router.push('/(auth)/mfa');
      } else {
        // Handle other statuses (needs_identifier, needs_first_factor, etc.)
        console.log('Sign-in status:', result.status);
      }
    } catch (err: any) {
      const errorMessage =
        err.errors?.[0]?.longMessage ?? 'Sign-in failed. Please try again.';
      Alert.alert('Error', errorMessage);
    } finally {
      setLoading(false);
    }
  };

  const handleGoogleSignIn = async () => {
    try {
      const { createdSessionId, setActive: setOAuthActive } =
        await startGoogleOAuth();

      if (createdSessionId) {
        await setOAuthActive!({ session: createdSessionId });
        router.replace('/(tabs)/home');
      }
    } catch (err) {
      console.error('Google OAuth error:', err);
      Alert.alert('Error', 'Google sign-in failed. Please try again.');
    }
  };

  const handleAppleSignIn = async () => {
    try {
      const { createdSessionId, setActive: setOAuthActive } =
        await startAppleOAuth();

      if (createdSessionId) {
        await setOAuthActive!({ session: createdSessionId });
        router.replace('/(tabs)/home');
      }
    } catch (err) {
      console.error('Apple OAuth error:', err);
      Alert.alert('Error', 'Apple sign-in failed. Please try again.');
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Welcome Back</Text>

      <TextInput
        style={styles.input}
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        autoCapitalize="none"
        keyboardType="email-address"
        textContentType="emailAddress"
        autoComplete="email"
      />

      <TextInput
        style={styles.input}
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        textContentType="password"
        autoComplete="password"
      />

      <TouchableOpacity
        style={styles.button}
        onPress={handleEmailSignIn}
        disabled={loading}
      >
        {loading ? (
          <ActivityIndicator color="#fff" />
        ) : (
          <Text style={styles.buttonText}>Sign In</Text>
        )}
      </TouchableOpacity>

      <View style={styles.divider}>
        <View style={styles.dividerLine} />
        <Text style={styles.dividerText}>or continue with</Text>
        <View style={styles.dividerLine} />
      </View>

      {/* Apple Sign In MUST come first on iOS (App Store requirement) */}
      <TouchableOpacity
        style={styles.socialButton}
        onPress={handleAppleSignIn}
      >
        <Text style={styles.socialButtonText}> Sign in with Apple</Text>
      </TouchableOpacity>

      <TouchableOpacity
        style={styles.socialButton}
        onPress={handleGoogleSignIn}
      >
        <Text style={styles.socialButtonText}>Sign in with Google</Text>
      </TouchableOpacity>

      <TouchableOpacity onPress={() => router.push('/(auth)/sign-up')}>
        <Text style={styles.linkText}>
          Don't have an account? Sign up
        </Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 24,
    justifyContent: 'center',
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 28,
    fontWeight: '700',
    marginBottom: 32,
    textAlign: 'center',
  },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 12,
    padding: 16,
    fontSize: 16,
    marginBottom: 12,
    backgroundColor: '#f9f9f9',
  },
  button: {
    backgroundColor: '#000',
    borderRadius: 12,
    padding: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  divider: {
    flexDirection: 'row',
    alignItems: 'center',
    marginVertical: 24,
  },
  dividerLine: {
    flex: 1,
    height: 1,
    backgroundColor: '#ddd',
  },
  dividerText: {
    marginHorizontal: 12,
    color: '#888',
    fontSize: 14,
  },
  socialButton: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 12,
    padding: 16,
    alignItems: 'center',
    marginBottom: 12,
  },
  socialButtonText: {
    fontSize: 16,
    fontWeight: '500',
  },
  linkText: {
    textAlign: 'center',
    color: '#666',
    marginTop: 16,
    fontSize: 14,
  },
});
```

### 3.7 Protected Navigation (React Native with Expo Router)

```typescript
// apps/mobile/app/(auth)/_layout.tsx
import { useAuth } from '@clerk/clerk-expo';
import { Redirect, Stack } from 'expo-router';

export default function AuthLayout() {
  const { isSignedIn, isLoaded } = useAuth();

  // If the user is already signed in, redirect to the main app
  if (isLoaded && isSignedIn) {
    return <Redirect href="/(tabs)/home" />;
  }

  return <Stack screenOptions={{ headerShown: false }} />;
}
```

```typescript
// apps/mobile/app/(tabs)/_layout.tsx
import { useAuth } from '@clerk/clerk-expo';
import { Redirect, Tabs } from 'expo-router';
import { ActivityIndicator, View } from 'react-native';

export default function TabsLayout() {
  const { isSignedIn, isLoaded } = useAuth();

  // Show loading screen while Clerk loads the session from secure storage
  if (!isLoaded) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    );
  }

  // If not signed in, redirect to auth flow
  if (!isSignedIn) {
    return <Redirect href="/(auth)/sign-in" />;
  }

  return (
    <Tabs>
      <Tabs.Screen name="home" options={{ title: 'Home' }} />
      <Tabs.Screen name="profile" options={{ title: 'Profile' }} />
      <Tabs.Screen name="settings" options={{ title: 'Settings' }} />
    </Tabs>
  );
}
```

### 3.8 Server Component Auth (Next.js)

```typescript
// apps/web/app/dashboard/page.tsx
import { auth, currentUser } from '@clerk/nextjs/server';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  // Server-side auth check -- no client JavaScript needed
  const { userId } = await auth();

  if (!userId) {
    redirect('/sign-in');
  }

  // Get full user object (includes metadata, email, etc.)
  const user = await currentUser();

  return (
    <div>
      <h1>Welcome, {user?.firstName}</h1>
      <p>Role: {user?.publicMetadata?.role as string ?? 'viewer'}</p>
      {/* Dashboard content */}
    </div>
  );
}
```

```typescript
// apps/web/app/api/protected/route.ts
import { auth } from '@clerk/nextjs/server';
import { NextResponse } from 'next/server';

export async function GET() {
  const { userId } = await auth();

  if (!userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Fetch data scoped to this user
  const data = await db.query('SELECT * FROM items WHERE user_id = $1', [
    userId,
  ]);

  return NextResponse.json(data);
}
```

### 3.9 Authenticated API Calls from React Native

```typescript
// packages/api-client/src/client.ts
import { useAuth } from '@clerk/clerk-expo';

/**
 * Create an authenticated fetch wrapper that automatically
 * includes the Clerk session token in API requests.
 */
export function useAuthenticatedFetch() {
  const { getToken } = useAuth();

  const authFetch = async (
    url: string,
    options: RequestInit = {}
  ): Promise<Response> => {
    // Clerk's getToken() returns a short-lived JWT
    // that is automatically refreshed
    const token = await getToken();

    if (!token) {
      throw new Error('Not authenticated');
    }

    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        Authorization: `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    });
  };

  return authFetch;
}

/**
 * For use with React Query / TanStack Query:
 */
export function useAuthenticatedQuery<T>(
  key: string[],
  url: string
) {
  const authFetch = useAuthenticatedFetch();

  return useQuery({
    queryKey: key,
    queryFn: async (): Promise<T> => {
      const response = await authFetch(url);
      if (!response.ok) {
        throw new Error(`API error: ${response.status}`);
      }
      return response.json();
    },
  });
}
```

### 3.10 Clerk Organizations (Multi-Tenancy)

Clerk's Organizations feature gives you multi-tenancy out of the box -- teams, workspaces, or any concept where users belong to groups with different roles.

```typescript
// apps/mobile/src/screens/organization-switcher.tsx
import { useOrganizationList, useOrganization } from '@clerk/clerk-expo';
import { View, Text, FlatList, TouchableOpacity } from 'react-native';

export function OrganizationSwitcher() {
  const { organizationList, isLoaded, setActive } = useOrganizationList();
  const { organization: activeOrg } = useOrganization();

  if (!isLoaded) return null;

  return (
    <View>
      <Text style={{ fontSize: 18, fontWeight: '600', marginBottom: 12 }}>
        Switch Organization
      </Text>
      <FlatList
        data={organizationList}
        keyExtractor={(item) => item.organization.id}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={{
              padding: 16,
              borderRadius: 8,
              backgroundColor:
                item.organization.id === activeOrg?.id
                  ? '#f0f0ff'
                  : '#fff',
              marginBottom: 8,
              borderWidth: 1,
              borderColor:
                item.organization.id === activeOrg?.id
                  ? '#6366f1'
                  : '#eee',
            }}
            onPress={() => setActive({ organization: item.organization.id })}
          >
            <Text style={{ fontWeight: '600' }}>
              {item.organization.name}
            </Text>
            <Text style={{ color: '#888', fontSize: 12 }}>
              Role: {item.membership.role}
            </Text>
          </TouchableOpacity>
        )}
      />
    </View>
  );
}
```

```typescript
// apps/web/middleware.ts (with organization-scoped protection)
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isOrgRoute = createRouteMatcher(['/org/:orgId/(.*)']);

export default clerkMiddleware(async (auth, req) => {
  if (isOrgRoute(req)) {
    // Require both authentication AND an active organization
    const { orgId } = await auth();
    if (!orgId) {
      // User is signed in but has no active organization
      return Response.redirect(new URL('/select-org', req.url));
    }
  }
});
```

---

## 4. SUPABASE AUTH

Supabase Auth is tightly integrated with the Supabase platform -- when you use it, your auth users are automatically linked to Row Level Security policies in your Postgres database. This is powerful: you define authorization rules at the database level, and they apply to every query, whether it comes from your React Native app, your Next.js server, or a direct API call.

### 4.1 Setup

```bash
# React Native
npx expo install @supabase/supabase-js expo-secure-store

# Next.js
pnpm add @supabase/supabase-js @supabase/ssr
```

### 4.2 Supabase Client (React Native)

```typescript
// packages/supabase/src/client.native.ts
import 'react-native-url-polyfill/auto';
import { createClient } from '@supabase/supabase-js';
import * as SecureStore from 'expo-secure-store';
import type { Database } from './database.types';

// Custom storage adapter using expo-secure-store
// This ensures auth tokens are encrypted at rest
const secureStoreAdapter = {
  getItem: async (key: string): Promise<string | null> => {
    return SecureStore.getItemAsync(key);
  },
  setItem: async (key: string, value: string): Promise<void> => {
    await SecureStore.setItemAsync(key, value);
  },
  removeItem: async (key: string): Promise<void> => {
    await SecureStore.deleteItemAsync(key);
  },
};

export const supabase = createClient<Database>(
  process.env.EXPO_PUBLIC_SUPABASE_URL!,
  process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      storage: secureStoreAdapter,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false, // Important for React Native
    },
  }
);
```

### 4.3 Supabase Client (Next.js with SSR)

```typescript
// packages/supabase/src/server.ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';
import type { Database } from './database.types';

export async function createSupabaseServerClient() {
  const cookieStore = await cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // setAll is called from Server Components where cookies
            // cannot be set. This is fine -- the middleware handles it.
          }
        },
      },
    }
  );
}
```

```typescript
// apps/web/middleware.ts (Supabase)
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          supabaseResponse = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // IMPORTANT: Do not remove this line.
  // Refreshes the auth token and updates cookies.
  const {
    data: { user },
  } = await supabase.auth.getUser();

  // Redirect unauthenticated users from protected routes
  if (
    !user &&
    !request.nextUrl.pathname.startsWith('/login') &&
    !request.nextUrl.pathname.startsWith('/auth') &&
    !request.nextUrl.pathname.startsWith('/api/public') &&
    request.nextUrl.pathname !== '/'
  ) {
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    return NextResponse.redirect(url);
  }

  return supabaseResponse;
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

### 4.4 Email/Password Authentication

```typescript
// packages/supabase/src/auth.ts
import { supabase } from './client';

export async function signUpWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
    options: {
      // Optional: add metadata at sign-up time
      data: {
        role: 'viewer',
        onboarding_complete: false,
      },
    },
  });

  if (error) throw error;

  // Supabase sends a confirmation email by default.
  // data.user will exist but data.session will be null
  // until the email is confirmed.
  return {
    user: data.user,
    needsEmailConfirmation: !data.session,
  };
}

export async function signInWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    if (error.message === 'Email not confirmed') {
      throw new Error('Please confirm your email before signing in.');
    }
    throw error;
  }

  return data; // { user, session }
}

export async function signOut() {
  const { error } = await supabase.auth.signOut();
  if (error) throw error;
}
```

### 4.5 Magic Links

Magic links are passwordless authentication -- the user enters their email, receives a link, clicks it, and is signed in. No password to remember, no password to breach.

```typescript
// packages/supabase/src/magic-link.ts
import { supabase } from './client';
import { makeRedirectUri } from 'expo-auth-session';

export async function sendMagicLink(email: string) {
  // For React Native, you need a redirect URL that your app handles
  const redirectTo = makeRedirectUri({
    scheme: 'myapp',
    path: 'auth/callback',
  });

  const { error } = await supabase.auth.signInWithOtp({
    email,
    options: {
      emailRedirectTo: redirectTo,
      // Don't create a new user if they don't exist
      shouldCreateUser: true,
    },
  });

  if (error) throw error;

  return { message: 'Check your email for the magic link' };
}

// Handle the deep link when the user clicks the magic link
// In Expo Router, this is handled via a route:
// apps/mobile/app/auth/callback.tsx
import { useEffect } from 'react';
import { useRouter, useLocalSearchParams } from 'expo-router';
import { supabase } from '@/packages/supabase/src/client';

export default function AuthCallback() {
  const router = useRouter();
  const params = useLocalSearchParams();

  useEffect(() => {
    // The magic link redirects with tokens in the URL fragment
    // Supabase's client handles extracting and setting the session
    const handleMagicLink = async () => {
      const { error } = await supabase.auth.getSession();

      if (error) {
        console.error('Magic link error:', error);
        router.replace('/(auth)/sign-in');
        return;
      }

      router.replace('/(tabs)/home');
    };

    handleMagicLink();
  }, []);

  return null; // Or a loading spinner
}
```

### 4.6 OAuth with Supabase

```typescript
// packages/supabase/src/oauth.ts
import { supabase } from './client';
import * as WebBrowser from 'expo-web-browser';
import * as AuthSession from 'expo-auth-session';

WebBrowser.maybeCompleteAuthSession();

export async function signInWithGoogle() {
  const redirectTo = AuthSession.makeRedirectUri({
    scheme: 'myapp',
    path: 'auth/callback',
  });

  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo,
      queryParams: {
        access_type: 'offline',
        prompt: 'consent',
      },
    },
  });

  if (error) throw error;

  // Open the OAuth URL in the system browser
  if (data.url) {
    const result = await WebBrowser.openAuthSessionAsync(
      data.url,
      redirectTo
    );

    if (result.type === 'success') {
      // Extract tokens from the URL and set the session
      const url = new URL(result.url);
      const access_token = url.searchParams.get('access_token');
      const refresh_token = url.searchParams.get('refresh_token');

      if (access_token && refresh_token) {
        await supabase.auth.setSession({
          access_token,
          refresh_token,
        });
      }
    }
  }
}
```

### 4.7 Row Level Security (Authorization at the Database Level)

This is Supabase's killer feature for authorization. Instead of checking permissions in every API route, you define policies at the Postgres level:

```sql
-- Enable RLS on the table
ALTER TABLE items ENABLE ROW LEVEL SECURITY;

-- Users can only read their own items
CREATE POLICY "Users can read own items"
  ON items
  FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert items for themselves
CREATE POLICY "Users can insert own items"
  ON items
  FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Users can only update their own items
CREATE POLICY "Users can update own items"
  ON items
  FOR UPDATE
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- Users can only delete their own items
CREATE POLICY "Users can delete own items"
  ON items
  FOR DELETE
  USING (auth.uid() = user_id);

-- Admins can read all items (using JWT custom claims)
CREATE POLICY "Admins can read all items"
  ON items
  FOR SELECT
  USING (
    auth.jwt() ->> 'role' = 'admin'
    OR auth.uid() = user_id
  );

-- Organization-based access:
-- Users can see items belonging to any organization they are a member of
CREATE POLICY "Org members can read org items"
  ON items
  FOR SELECT
  USING (
    organization_id IN (
      SELECT org_id FROM organization_members
      WHERE user_id = auth.uid()
    )
  );
```

```typescript
// With RLS enabled, queries are automatically scoped:
const { data: myItems } = await supabase
  .from('items')
  .select('*');
// Returns ONLY items where user_id = current user's ID
// No WHERE clause needed -- RLS handles it

// An admin user running the same query would see all items
// because of the admin policy above
```

### 4.8 Auth State Listener (React Native)

```typescript
// packages/supabase/src/auth-provider.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import { supabase } from './client';
import type { Session, User } from '@supabase/supabase-js';

interface AuthContext {
  session: Session | null;
  user: User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContext>({
  session: null,
  user: null,
  isLoading: true,
  isAuthenticated: false,
});

export function SupabaseAuthProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  const [session, setSession] = useState<Session | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // Get the initial session from secure storage
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      setIsLoading(false);
    });

    // Listen for auth state changes
    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
      setIsLoading(false);
    });

    return () => subscription.unsubscribe();
  }, []);

  return (
    <AuthContext.Provider
      value={{
        session,
        user: session?.user ?? null,
        isLoading,
        isAuthenticated: !!session,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export const useSupabaseAuth = () => useContext(AuthContext);
```

### 4.9 Real-Time Subscriptions with Auth

Supabase's real-time subscriptions respect RLS policies:

```typescript
// Subscribe to changes on the user's items (RLS-filtered)
const channel = supabase
  .channel('my-items')
  .on(
    'postgres_changes',
    {
      event: '*',
      schema: 'public',
      table: 'items',
      // RLS will automatically filter to the current user's items
    },
    (payload) => {
      console.log('Change received:', payload);
      // Update your local state / React Query cache
    }
  )
  .subscribe();

// Clean up when the component unmounts
return () => {
  supabase.removeChannel(channel);
};
```

---

## 5. FIREBASE AUTH

Firebase Auth is the most mature mobile auth solution and excels at phone/SMS authentication -- something neither Clerk nor Supabase handles as well. It also supports anonymous auth for progressive onboarding (let users explore the app before requiring sign-up).

### 5.1 Setup

```bash
# Expo with Firebase (using @react-native-firebase)
npx expo install @react-native-firebase/app @react-native-firebase/auth

# You also need the Expo config plugin
npx expo install expo-build-properties
```

```json
// app.json
{
  "expo": {
    "plugins": [
      "@react-native-firebase/app",
      "@react-native-firebase/auth",
      [
        "expo-build-properties",
        {
          "ios": {
            "useFrameworks": "static"
          }
        }
      ]
    ],
    "ios": {
      "googleServicesFile": "./GoogleService-Info.plist"
    },
    "android": {
      "googleServicesFile": "./google-services.json"
    }
  }
}
```

> **Note:** `@react-native-firebase` requires a development build (not Expo Go). Run `npx expo prebuild` and then `npx expo run:ios` or `npx expo run:android`.

### 5.2 Email/Password Authentication

```typescript
// packages/firebase/src/auth.ts
import auth, { FirebaseAuthTypes } from '@react-native-firebase/auth';

export async function signUpWithEmail(
  email: string,
  password: string
): Promise<FirebaseAuthTypes.UserCredential> {
  try {
    const credential = await auth().createUserWithEmailAndPassword(
      email,
      password
    );

    // Send email verification
    await credential.user.sendEmailVerification();

    return credential;
  } catch (error: any) {
    switch (error.code) {
      case 'auth/email-already-in-use':
        throw new Error('An account with this email already exists.');
      case 'auth/invalid-email':
        throw new Error('Please enter a valid email address.');
      case 'auth/weak-password':
        throw new Error('Password must be at least 6 characters.');
      default:
        throw new Error('Sign-up failed. Please try again.');
    }
  }
}

export async function signInWithEmail(
  email: string,
  password: string
): Promise<FirebaseAuthTypes.UserCredential> {
  try {
    return await auth().signInWithEmailAndPassword(email, password);
  } catch (error: any) {
    switch (error.code) {
      case 'auth/invalid-credential':
        throw new Error('Invalid email or password.');
      case 'auth/user-disabled':
        throw new Error('This account has been disabled.');
      case 'auth/too-many-requests':
        throw new Error(
          'Too many failed attempts. Please try again later.'
        );
      default:
        throw new Error('Sign-in failed. Please try again.');
    }
  }
}

export async function signOut(): Promise<void> {
  await auth().signOut();
}

export async function resetPassword(email: string): Promise<void> {
  await auth().sendPasswordResetEmail(email);
}
```

### 5.3 Phone Auth (SMS OTP)

Phone auth is Firebase's strongest differentiator. It handles SMS delivery, rate limiting, and phone number verification across the world.

```typescript
// packages/firebase/src/phone-auth.ts
import auth from '@react-native-firebase/auth';
import { useState, useRef } from 'react';

export function usePhoneAuth() {
  const [verificationId, setVerificationId] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  /**
   * Step 1: Send verification code to phone number.
   * Firebase handles the SMS delivery.
   */
  const sendVerificationCode = async (phoneNumber: string) => {
    setLoading(true);
    try {
      // Phone number must be in E.164 format: +1234567890
      const confirmation = await auth().signInWithPhoneNumber(phoneNumber);
      setVerificationId(confirmation.verificationId);
      return confirmation;
    } catch (error: any) {
      switch (error.code) {
        case 'auth/invalid-phone-number':
          throw new Error('Please enter a valid phone number.');
        case 'auth/too-many-requests':
          throw new Error(
            'Too many attempts. Please try again later.'
          );
        case 'auth/quota-exceeded':
          throw new Error(
            'SMS quota exceeded. Please try again tomorrow.'
          );
        default:
          throw new Error('Failed to send verification code.');
      }
    } finally {
      setLoading(false);
    }
  };

  /**
   * Step 2: Verify the code entered by the user.
   */
  const verifyCode = async (code: string) => {
    if (!verificationId) {
      throw new Error('No verification in progress.');
    }

    setLoading(true);
    try {
      const credential = auth.PhoneAuthProvider.credential(
        verificationId,
        code
      );
      const result = await auth().signInWithCredential(credential);
      return result;
    } catch (error: any) {
      switch (error.code) {
        case 'auth/invalid-verification-code':
          throw new Error('Invalid code. Please try again.');
        case 'auth/session-expired':
          throw new Error(
            'Verification expired. Please request a new code.'
          );
        default:
          throw new Error('Verification failed. Please try again.');
      }
    } finally {
      setLoading(false);
    }
  };

  return {
    sendVerificationCode,
    verifyCode,
    verificationId,
    loading,
  };
}
```

```typescript
// apps/mobile/app/(auth)/phone-sign-in.tsx
import { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, Alert } from 'react-native';
import { useRouter } from 'expo-router';
import { usePhoneAuth } from '@/packages/firebase/src/phone-auth';

export default function PhoneSignInScreen() {
  const router = useRouter();
  const { sendVerificationCode, verifyCode, verificationId, loading } =
    usePhoneAuth();

  const [phoneNumber, setPhoneNumber] = useState('');
  const [code, setCode] = useState('');

  const handleSendCode = async () => {
    try {
      await sendVerificationCode(phoneNumber);
    } catch (err: any) {
      Alert.alert('Error', err.message);
    }
  };

  const handleVerify = async () => {
    try {
      await verifyCode(code);
      router.replace('/(tabs)/home');
    } catch (err: any) {
      Alert.alert('Error', err.message);
    }
  };

  if (!verificationId) {
    // Step 1: Enter phone number
    return (
      <View style={{ flex: 1, padding: 24, justifyContent: 'center' }}>
        <Text style={{ fontSize: 24, fontWeight: '700', marginBottom: 24 }}>
          Enter your phone number
        </Text>
        <TextInput
          value={phoneNumber}
          onChangeText={setPhoneNumber}
          placeholder="+1 (555) 123-4567"
          keyboardType="phone-pad"
          textContentType="telephoneNumber"
          autoComplete="tel"
          style={{
            borderWidth: 1,
            borderColor: '#ddd',
            borderRadius: 12,
            padding: 16,
            fontSize: 18,
            marginBottom: 16,
          }}
        />
        <TouchableOpacity
          onPress={handleSendCode}
          disabled={loading}
          style={{
            backgroundColor: '#000',
            borderRadius: 12,
            padding: 16,
            alignItems: 'center',
          }}
        >
          <Text style={{ color: '#fff', fontSize: 16, fontWeight: '600' }}>
            {loading ? 'Sending...' : 'Send Code'}
          </Text>
        </TouchableOpacity>
      </View>
    );
  }

  // Step 2: Enter verification code
  return (
    <View style={{ flex: 1, padding: 24, justifyContent: 'center' }}>
      <Text style={{ fontSize: 24, fontWeight: '700', marginBottom: 8 }}>
        Enter verification code
      </Text>
      <Text style={{ color: '#666', marginBottom: 24 }}>
        We sent a code to {phoneNumber}
      </Text>
      <TextInput
        value={code}
        onChangeText={setCode}
        placeholder="123456"
        keyboardType="number-pad"
        textContentType="oneTimeCode"
        autoComplete="sms-otp"
        maxLength={6}
        style={{
          borderWidth: 1,
          borderColor: '#ddd',
          borderRadius: 12,
          padding: 16,
          fontSize: 24,
          letterSpacing: 8,
          textAlign: 'center',
          marginBottom: 16,
        }}
      />
      <TouchableOpacity
        onPress={handleVerify}
        disabled={loading}
        style={{
          backgroundColor: '#000',
          borderRadius: 12,
          padding: 16,
          alignItems: 'center',
        }}
      >
        <Text style={{ color: '#fff', fontSize: 16, fontWeight: '600' }}>
          {loading ? 'Verifying...' : 'Verify'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}
```

### 5.4 Google Sign-In with Firebase

```bash
# Install the Google Sign-In library
npx expo install @react-native-google-signin/google-signin
```

```typescript
// packages/firebase/src/google-auth.ts
import auth from '@react-native-firebase/auth';
import {
  GoogleSignin,
  statusCodes,
} from '@react-native-google-signin/google-signin';

// Configure Google Sign-In (call this once at app startup)
GoogleSignin.configure({
  webClientId: process.env.EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID,
  // iOS client ID is automatically read from GoogleService-Info.plist
});

export async function signInWithGoogle() {
  try {
    // Check if play services are available (Android)
    await GoogleSignin.hasPlayServices({
      showPlayServicesUpdateDialog: true,
    });

    // Sign in with Google
    const signInResult = await GoogleSignin.signIn();

    // Get the ID token
    const idToken = signInResult.data?.idToken;
    if (!idToken) {
      throw new Error('No ID token received from Google.');
    }

    // Create a Firebase credential with the Google ID token
    const googleCredential = auth.GoogleAuthProvider.credential(idToken);

    // Sign in to Firebase with the Google credential
    const userCredential = await auth().signInWithCredential(
      googleCredential
    );

    return userCredential;
  } catch (error: any) {
    switch (error.code) {
      case statusCodes.SIGN_IN_CANCELLED:
        // User cancelled the sign-in flow
        return null;
      case statusCodes.IN_PROGRESS:
        throw new Error('Sign-in already in progress.');
      case statusCodes.PLAY_SERVICES_NOT_AVAILABLE:
        throw new Error(
          'Google Play Services are not available on this device.'
        );
      default:
        throw new Error('Google sign-in failed. Please try again.');
    }
  }
}
```

### 5.5 Apple Sign-In with Firebase

```bash
npx expo install expo-apple-authentication
```

```typescript
// packages/firebase/src/apple-auth.ts
import auth from '@react-native-firebase/auth';
import * as AppleAuthentication from 'expo-apple-authentication';
import * as Crypto from 'expo-crypto';

export async function signInWithApple() {
  // Generate a nonce for security
  const rawNonce = Math.random().toString(36).substring(2, 10);
  const hashedNonce = await Crypto.digestStringAsync(
    Crypto.CryptoDigestAlgorithm.SHA256,
    rawNonce
  );

  try {
    const appleCredential = await AppleAuthentication.signInAsync({
      requestedScopes: [
        AppleAuthentication.AppleAuthenticationScope.EMAIL,
        AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
      ],
      nonce: hashedNonce,
    });

    const { identityToken } = appleCredential;
    if (!identityToken) {
      throw new Error('No identity token received from Apple.');
    }

    // Create a Firebase credential with the Apple ID token
    const firebaseCredential = auth.AppleAuthProvider.credential(
      identityToken,
      rawNonce
    );

    // Sign in to Firebase
    const userCredential = await auth().signInWithCredential(
      firebaseCredential
    );

    // Apple only provides the user's name on the FIRST sign-in.
    // Store it immediately because you will never get it again.
    if (appleCredential.fullName?.givenName) {
      await userCredential.user.updateProfile({
        displayName: `${appleCredential.fullName.givenName} ${
          appleCredential.fullName.familyName ?? ''
        }`.trim(),
      });
    }

    return userCredential;
  } catch (error: any) {
    if (error.code === 'ERR_REQUEST_CANCELED') {
      return null; // User cancelled
    }
    throw new Error('Apple sign-in failed. Please try again.');
  }
}
```

### 5.6 Anonymous Auth (Progressive Onboarding)

Anonymous auth lets users explore your app without creating an account. When they decide to sign up, you link their anonymous account to a permanent one, preserving all their data.

```typescript
// packages/firebase/src/anonymous-auth.ts
import auth from '@react-native-firebase/auth';

/**
 * Sign in anonymously. Creates a temporary user account.
 * All data created by this user can be preserved when they
 * later link to a permanent account.
 */
export async function signInAnonymously() {
  const { user } = await auth().signInAnonymously();
  return user;
}

/**
 * Link an anonymous account to an email/password account.
 * This preserves the user's UID and all associated data.
 */
export async function linkAnonymousToEmail(
  email: string,
  password: string
) {
  const currentUser = auth().currentUser;
  if (!currentUser?.isAnonymous) {
    throw new Error('Current user is not anonymous.');
  }

  const credential = auth.EmailAuthProvider.credential(email, password);
  const result = await currentUser.linkWithCredential(credential);

  // Send verification email
  await result.user.sendEmailVerification();

  return result;
}

/**
 * Link an anonymous account to a Google account.
 */
export async function linkAnonymousToGoogle(idToken: string) {
  const currentUser = auth().currentUser;
  if (!currentUser?.isAnonymous) {
    throw new Error('Current user is not anonymous.');
  }

  const credential = auth.GoogleAuthProvider.credential(idToken);
  return currentUser.linkWithCredential(credential);
}
```

### 5.7 Auth State Listener

```typescript
// packages/firebase/src/auth-provider.tsx
import { createContext, useContext, useEffect, useState } from 'react';
import auth, { FirebaseAuthTypes } from '@react-native-firebase/auth';

interface FirebaseAuthContext {
  user: FirebaseAuthTypes.User | null;
  isLoading: boolean;
  isAuthenticated: boolean;
  isAnonymous: boolean;
}

const AuthContext = createContext<FirebaseAuthContext>({
  user: null,
  isLoading: true,
  isAuthenticated: false,
  isAnonymous: false,
});

export function FirebaseAuthProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  const [user, setUser] = useState<FirebaseAuthTypes.User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // onAuthStateChanged fires immediately with the current auth state,
    // then again whenever it changes
    const unsubscribe = auth().onAuthStateChanged((user) => {
      setUser(user);
      setIsLoading(false);
    });

    return unsubscribe;
  }, []);

  return (
    <AuthContext.Provider
      value={{
        user,
        isLoading,
        isAuthenticated: !!user,
        isAnonymous: user?.isAnonymous ?? false,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export const useFirebaseAuth = () => useContext(AuthContext);
```

### 5.8 ID Token Verification on Server

When your React Native app calls your backend, you send the Firebase ID token and verify it server-side:

```typescript
// apps/api/src/middleware/firebase-auth.ts
import { getAuth } from 'firebase-admin/auth';
import { initializeApp, cert, getApps } from 'firebase-admin/app';

// Initialize Firebase Admin (once)
if (getApps().length === 0) {
  initializeApp({
    credential: cert({
      projectId: process.env.FIREBASE_PROJECT_ID,
      clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
      privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    }),
  });
}

/**
 * Express middleware to verify Firebase ID tokens.
 */
export async function firebaseAuthMiddleware(
  req: any,
  res: any,
  next: any
) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing authorization header' });
  }

  const idToken = authHeader.split('Bearer ')[1];

  try {
    // Verify the ID token and check if it has been revoked
    const decodedToken = await getAuth().verifyIdToken(idToken, true);
    req.user = decodedToken;
    next();
  } catch (error: any) {
    if (error.code === 'auth/id-token-revoked') {
      return res.status(401).json({ error: 'Token has been revoked' });
    }
    if (error.code === 'auth/id-token-expired') {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

```typescript
// On the React Native client, get the ID token for API calls:
import auth from '@react-native-firebase/auth';

async function callProtectedApi(endpoint: string) {
  const currentUser = auth().currentUser;
  if (!currentUser) throw new Error('Not authenticated');

  // getIdToken(true) forces a refresh if the token is expired
  const idToken = await currentUser.getIdToken(true);

  return fetch(`${API_URL}${endpoint}`, {
    headers: {
      Authorization: `Bearer ${idToken}`,
      'Content-Type': 'application/json',
    },
  });
}
```

### 5.9 Custom Claims for RBAC

Firebase supports custom claims -- metadata attached to the user's ID token that is verified server-side:

```typescript
// Server-side: Set custom claims
import { getAuth } from 'firebase-admin/auth';

export async function setUserRole(uid: string, role: string) {
  await getAuth().setCustomUserClaims(uid, { role });
  // Note: The user's existing ID tokens will still have the old claims
  // until they are refreshed. Force a token refresh on the client:
}

// Server-side: Check custom claims in middleware
export function requireRole(role: string) {
  return (req: any, res: any, next: any) => {
    if (req.user?.role !== role) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
}

// Client-side: Force token refresh after role change
import auth from '@react-native-firebase/auth';

async function refreshClaims() {
  const user = auth().currentUser;
  if (user) {
    // Force refresh to get updated custom claims
    await user.getIdToken(true);
    const idTokenResult = await user.getIdTokenResult();
    console.log('User role:', idTokenResult.claims.role);
  }
}
```

---

## 6. AUTH.JS (NEXTAUTH) FOR NEXT.JS

Auth.js (formerly NextAuth.js) is the standard open-source auth solution for Next.js. It is the right choice when you are building a web-only application or when you want maximum control over your auth flow without vendor lock-in. Version 5 brought major changes: native App Router support, Edge compatibility, and a simpler API.

### 6.1 Setup

```bash
pnpm add next-auth@beta
```

```typescript
// apps/web/auth.ts
import NextAuth from 'next-auth';
import Google from 'next-auth/providers/google';
import GitHub from 'next-auth/providers/github';
import Apple from 'next-auth/providers/apple';
import Credentials from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from './lib/prisma';

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    Apple({
      clientId: process.env.APPLE_CLIENT_ID!,
      clientSecret: process.env.APPLE_CLIENT_SECRET!,
    }),
    Credentials({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.password) {
          return null;
        }

        const user = await prisma.user.findUnique({
          where: { email: credentials.email as string },
        });

        if (!user || !user.hashedPassword) {
          return null;
        }

        const bcrypt = await import('bcryptjs');
        const isValid = await bcrypt.compare(
          credentials.password as string,
          user.hashedPassword
        );

        if (!isValid) return null;

        return {
          id: user.id,
          email: user.email,
          name: user.name,
          image: user.image,
          role: user.role,
        };
      },
    }),
  ],
  session: {
    // JWT strategy is faster (no DB lookup per request)
    // Database strategy is more secure (revocable sessions)
    strategy: 'jwt',
  },
  callbacks: {
    // Add custom fields to the JWT
    async jwt({ token, user, trigger, session }) {
      if (user) {
        token.role = (user as any).role ?? 'viewer';
        token.userId = user.id;
      }
      // Handle session updates (e.g., role change)
      if (trigger === 'update' && session?.role) {
        token.role = session.role;
      }
      return token;
    },
    // Expose custom fields in the session
    async session({ session, token }) {
      if (session.user) {
        session.user.role = token.role as string;
        session.user.id = token.userId as string;
      }
      return session;
    },
    // Control which users can sign in
    async signIn({ user, account }) {
      // Block sign-in for disabled users
      if (user.id) {
        const dbUser = await prisma.user.findUnique({
          where: { id: user.id },
        });
        if (dbUser?.disabled) return false;
      }
      return true;
    },
  },
  pages: {
    signIn: '/sign-in',
    error: '/auth-error',
  },
});
```

### 6.2 Route Handler

```typescript
// apps/web/app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth';

export const { GET, POST } = handlers;
```

### 6.3 Middleware Protection

```typescript
// apps/web/middleware.ts
import { auth } from './auth';

export default auth((req) => {
  const { pathname } = req.nextUrl;
  const isLoggedIn = !!req.auth;

  // Define protected routes
  const protectedPaths = ['/dashboard', '/settings', '/admin'];
  const isProtected = protectedPaths.some((path) =>
    pathname.startsWith(path)
  );

  if (isProtected && !isLoggedIn) {
    const signInUrl = new URL('/sign-in', req.url);
    signInUrl.searchParams.set('callbackUrl', pathname);
    return Response.redirect(signInUrl);
  }

  // Admin-only routes
  if (pathname.startsWith('/admin') && req.auth?.user?.role !== 'admin') {
    return Response.redirect(new URL('/dashboard', req.url));
  }
});

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
};
```

### 6.4 Server Component Auth

```typescript
// apps/web/app/dashboard/page.tsx
import { auth } from '@/auth';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await auth();

  if (!session) {
    redirect('/sign-in');
  }

  return (
    <div>
      <h1>Welcome, {session.user.name}</h1>
      <p>Role: {session.user.role}</p>
    </div>
  );
}
```

### 6.5 Client Component Auth

```typescript
// apps/web/components/user-menu.tsx
'use client';

import { useSession, signOut } from 'next-auth/react';

export function UserMenu() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <div>Loading...</div>;
  if (status === 'unauthenticated') return <a href="/sign-in">Sign In</a>;

  return (
    <div>
      <span>{session?.user?.name}</span>
      <button onClick={() => signOut({ callbackUrl: '/' })}>
        Sign Out
      </button>
    </div>
  );
}
```

### 6.6 Sharing Auth.js Sessions with React Native

Auth.js is web-only, but you can bridge it to React Native through your API:

```typescript
// apps/api/src/routes/auth/mobile-token.ts
// This endpoint lets your RN app authenticate using the same user database

import { Router } from 'express';
import { prisma } from '../../lib/prisma';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';

const router = Router();

router.post('/auth/mobile/sign-in', async (req, res) => {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({ where: { email } });
  if (!user?.hashedPassword) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const valid = await bcrypt.compare(password, user.hashedPassword);
  if (!valid) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // Issue tokens for the mobile app
  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );

  const refreshToken = jwt.sign(
    { userId: user.id, type: 'refresh' },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: '30d' }
  );

  // Store refresh token in database for revocation
  await prisma.refreshToken.create({
    data: {
      token: refreshToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    },
  });

  return res.json({ accessToken, refreshToken, user: { id: user.id, email: user.email, role: user.role } });
});

export default router;
```

### 6.7 Type Augmentation

Auth.js uses TypeScript module augmentation for custom fields:

```typescript
// apps/web/types/next-auth.d.ts
import 'next-auth';
import 'next-auth/jwt';

declare module 'next-auth' {
  interface Session {
    user: {
      id: string;
      role: string;
      email: string;
      name: string;
      image: string;
    };
  }

  interface User {
    role?: string;
  }
}

declare module 'next-auth/jwt' {
  interface JWT {
    role?: string;
    userId?: string;
  }
}
```

---

## 7. TOKEN MANAGEMENT

Token management is where most auth implementations fall apart. The sign-in flow works, the user is authenticated, and then three weeks later they open the app and get a 401 error because nobody thought about token refresh. This section ensures that does not happen.

### 7.1 The Two-Token Architecture

Every modern auth system uses a two-token pattern:

```
┌─────────────────────────────────────────────────────────┐
│ ACCESS TOKEN                                             │
│                                                          │
│ Purpose: Authorize API requests                          │
│ Lifetime: Short (15 minutes)                             │
│ Storage: Memory (preferred) or Secure Store              │
│ Contains: User ID, role, permissions                     │
│ Sent: In Authorization header on every API request       │
│                                                          │
│ If stolen: Attacker has 15 minutes of access             │
│ If lost: User re-authenticates or uses refresh token     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ REFRESH TOKEN                                            │
│                                                          │
│ Purpose: Obtain new access tokens without re-login       │
│ Lifetime: Long (7-30 days)                               │
│ Storage: Secure Store (iOS Keychain / Android Keystore)  │
│ Contains: User ID, token family (for rotation)           │
│ Sent: Only to the /token/refresh endpoint                │
│                                                          │
│ If stolen: Attacker can generate new access tokens       │
│            until the refresh token is revoked or rotated │
│ If lost: User must re-authenticate                       │
└─────────────────────────────────────────────────────────┘
```

**Why short-lived access tokens?**

If an access token is intercepted (network logging, compromised proxy, leaked in error reporting), the damage window is 15 minutes. Compare this to a long-lived token: an attacker could silently access the user's data for weeks.

**Why long-lived refresh tokens?**

So the user does not have to re-enter their credentials every 15 minutes. The refresh token is stored in the most secure storage available and is only ever sent to a single endpoint.

### 7.2 Token Rotation

**Refresh token rotation** is a critical security mechanism: every time a refresh token is used, a new one is issued and the old one is invalidated. If an attacker steals a refresh token and both the attacker and the legitimate user try to use it, the second use triggers a revocation of the entire token family.

```typescript
// apps/api/src/routes/auth/refresh.ts
import jwt from 'jsonwebtoken';
import { prisma } from '../../lib/prisma';
import crypto from 'crypto';

export async function handleTokenRefresh(refreshToken: string) {
  // 1. Verify the refresh token's signature
  let payload: any;
  try {
    payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!);
  } catch {
    throw new Error('Invalid refresh token');
  }

  // 2. Look up the token in the database
  const storedToken = await prisma.refreshToken.findUnique({
    where: { token: refreshToken },
    include: { user: true },
  });

  // 3. If the token doesn't exist, it was already rotated.
  //    This means either it was used legitimately (normal),
  //    or it was stolen and the legitimate token was already rotated.
  //    Either way, revoke the ENTIRE token family as a precaution.
  if (!storedToken) {
    // Revoke all refresh tokens for this user
    await prisma.refreshToken.deleteMany({
      where: { userId: payload.userId },
    });
    throw new Error('Token reuse detected. All sessions revoked.');
  }

  // 4. Check if the token is expired
  if (storedToken.expiresAt < new Date()) {
    await prisma.refreshToken.delete({ where: { id: storedToken.id } });
    throw new Error('Refresh token expired');
  }

  // 5. Delete the used refresh token (rotation)
  await prisma.refreshToken.delete({ where: { id: storedToken.id } });

  // 6. Issue new tokens
  const newAccessToken = jwt.sign(
    { userId: storedToken.userId, role: storedToken.user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );

  const newRefreshToken = jwt.sign(
    {
      userId: storedToken.userId,
      type: 'refresh',
      family: payload.family ?? crypto.randomUUID(),
    },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: '30d' }
  );

  // 7. Store the new refresh token
  await prisma.refreshToken.create({
    data: {
      token: newRefreshToken,
      userId: storedToken.userId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
    },
  });

  return { accessToken: newAccessToken, refreshToken: newRefreshToken };
}
```

### 7.3 Secure Storage

```typescript
// packages/auth/src/token-storage.ts
import * as SecureStore from 'expo-secure-store';
import { Platform } from 'react-native';

const KEYS = {
  ACCESS_TOKEN: 'auth_access_token',
  REFRESH_TOKEN: 'auth_refresh_token',
  TOKEN_EXPIRY: 'auth_token_expiry',
} as const;

/**
 * NEVER use AsyncStorage for tokens.
 *
 * AsyncStorage stores data in:
 * - iOS: An unencrypted plist file
 * - Android: An unencrypted SQLite database
 *
 * expo-secure-store uses:
 * - iOS: Keychain (encrypted, hardware-backed on devices with Secure Enclave)
 * - Android: Keystore (encrypted, hardware-backed on most modern devices)
 */
export const tokenStorage = {
  async saveTokens(accessToken: string, refreshToken: string): Promise<void> {
    await Promise.all([
      SecureStore.setItemAsync(KEYS.ACCESS_TOKEN, accessToken),
      SecureStore.setItemAsync(KEYS.REFRESH_TOKEN, refreshToken),
      SecureStore.setItemAsync(
        KEYS.TOKEN_EXPIRY,
        // Calculate when the access token expires (15 min from now)
        String(Date.now() + 15 * 60 * 1000)
      ),
    ]);
  },

  async getAccessToken(): Promise<string | null> {
    return SecureStore.getItemAsync(KEYS.ACCESS_TOKEN);
  },

  async getRefreshToken(): Promise<string | null> {
    return SecureStore.getItemAsync(KEYS.REFRESH_TOKEN);
  },

  async isAccessTokenExpired(): Promise<boolean> {
    const expiry = await SecureStore.getItemAsync(KEYS.TOKEN_EXPIRY);
    if (!expiry) return true;
    // Consider it expired 60 seconds early to account for clock skew
    // and network latency
    return Date.now() > Number(expiry) - 60_000;
  },

  async clearTokens(): Promise<void> {
    await Promise.all([
      SecureStore.deleteItemAsync(KEYS.ACCESS_TOKEN),
      SecureStore.deleteItemAsync(KEYS.REFRESH_TOKEN),
      SecureStore.deleteItemAsync(KEYS.TOKEN_EXPIRY),
    ]);
  },
};
```

### 7.4 Automatic Token Refresh with Interceptors

This is the code that makes auth invisible to the user. When an API call fails with a 401, the interceptor automatically refreshes the token and retries the request. A mutex prevents multiple concurrent refreshes.

```typescript
// packages/api-client/src/authenticated-client.ts
import { tokenStorage } from '@/packages/auth/src/token-storage';

const API_URL = process.env.EXPO_PUBLIC_API_URL!;

/**
 * Mutex to prevent concurrent token refresh.
 *
 * Without this, if 5 requests fail with 401 simultaneously,
 * all 5 would try to refresh the token, using the same refresh token
 * 5 times. With rotation, the first would succeed and invalidate
 * the refresh token, causing the other 4 to fail and potentially
 * triggering a "token reuse detected" security event.
 */
let isRefreshing = false;
let refreshPromise: Promise<boolean> | null = null;

async function refreshTokens(): Promise<boolean> {
  // If a refresh is already in progress, wait for it
  if (isRefreshing && refreshPromise) {
    return refreshPromise;
  }

  isRefreshing = true;
  refreshPromise = (async () => {
    try {
      const refreshToken = await tokenStorage.getRefreshToken();
      if (!refreshToken) return false;

      const response = await fetch(`${API_URL}/auth/refresh`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken }),
      });

      if (!response.ok) {
        // Refresh failed -- clear tokens and force re-auth
        await tokenStorage.clearTokens();
        return false;
      }

      const { accessToken, refreshToken: newRefreshToken } =
        await response.json();

      await tokenStorage.saveTokens(accessToken, newRefreshToken);
      return true;
    } catch {
      return false;
    } finally {
      isRefreshing = false;
      refreshPromise = null;
    }
  })();

  return refreshPromise;
}

/**
 * Authenticated fetch wrapper with automatic token refresh.
 */
export async function authFetch(
  endpoint: string,
  options: RequestInit = {}
): Promise<Response> {
  // Check if the access token is about to expire
  const isExpired = await tokenStorage.isAccessTokenExpired();
  if (isExpired) {
    const refreshed = await refreshTokens();
    if (!refreshed) {
      // Emit an event that the auth provider listens to
      // to trigger sign-out and redirect to login
      authEventEmitter.emit('SESSION_EXPIRED');
      throw new Error('Session expired. Please sign in again.');
    }
  }

  const accessToken = await tokenStorage.getAccessToken();

  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    },
  });

  // Handle 401 (token expired between the check and the request)
  if (response.status === 401) {
    const refreshed = await refreshTokens();
    if (!refreshed) {
      authEventEmitter.emit('SESSION_EXPIRED');
      throw new Error('Session expired. Please sign in again.');
    }

    // Retry with the new token
    const newAccessToken = await tokenStorage.getAccessToken();
    return fetch(`${API_URL}${endpoint}`, {
      ...options,
      headers: {
        ...options.headers,
        Authorization: `Bearer ${newAccessToken}`,
        'Content-Type': 'application/json',
      },
    });
  }

  return response;
}

// Simple event emitter for auth events
class AuthEventEmitter {
  private listeners: Map<string, Set<() => void>> = new Map();

  on(event: string, callback: () => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);
    return () => this.listeners.get(event)?.delete(callback);
  }

  emit(event: string) {
    this.listeners.get(event)?.forEach((cb) => cb());
  }
}

export const authEventEmitter = new AuthEventEmitter();
```

### 7.5 Integration with React Query

```typescript
// packages/api-client/src/query-client.ts
import { QueryClient } from '@tanstack/react-query';
import { authEventEmitter } from './authenticated-client';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error: any) => {
        // Don't retry auth errors
        if (error?.status === 401 || error?.status === 403) {
          return false;
        }
        return failureCount < 3;
      },
    },
  },
});

// When the session expires, clear all cached queries
authEventEmitter.on('SESSION_EXPIRED', () => {
  queryClient.clear();
});
```

---

## 8. SOCIAL LOGIN

### 8.1 Apple Sign In (REQUIRED on iOS)

This is the single most important thing in this section: **if your iOS app offers any third-party login (Google, Facebook, GitHub, etc.), you MUST also offer Sign in with Apple.** This is Apple's App Store Review Guideline 4.8, and violating it is the most common auth-related App Store rejection.

The requirement applies if:
- You offer Google Sign In
- You offer Facebook Login
- You offer any OAuth-based social login
- You offer "Sign in with X" for any third-party service

The requirement does NOT apply if:
- You only offer email/password (no social login at all)
- You only offer phone number authentication
- Your app uses a company-specific SSO (enterprise apps)

**Common rejection reasons and how to avoid them:**

| Rejection Reason | Fix |
|---|---|
| "Apple Sign In not offered" | Add Apple Sign In button above all other social login buttons |
| "Apple Sign In button not prominent enough" | Make it the same size as other social buttons; use Apple's official design guidelines |
| "User cannot delete account" | Add account deletion in settings (also required by App Store policy since 2022) |
| "Name/email requested after Apple Sign In" | Apple provides name and email (if user consents). Do not ask again |

```typescript
// Implementation with Clerk (simplest)
import { useOAuth } from '@clerk/clerk-expo';

const { startOAuthFlow } = useOAuth({ strategy: 'oauth_apple' });

// Implementation with Supabase
const { data, error } = await supabase.auth.signInWithOAuth({
  provider: 'apple',
});

// Implementation with Firebase (see Section 5.5 above)
// Uses expo-apple-authentication for the native dialog
```

### 8.2 Google Sign In

Google Sign In works differently on each platform:

- **iOS/Android (React Native):** Use `@react-native-google-signin/google-signin` for native UI, or `expo-auth-session` for a browser-based flow
- **Web (Next.js):** Use Google's OAuth2 flow via your auth provider (Clerk, Auth.js, etc.)

```typescript
// The native approach (recommended for mobile -- smoother UX):
import {
  GoogleSignin,
  GoogleSigninButton,
} from '@react-native-google-signin/google-signin';

// Configure once at app startup
GoogleSignin.configure({
  webClientId: process.env.EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID,
  offlineAccess: true,
  scopes: ['profile', 'email'],
});

// In your sign-in screen
function GoogleButton() {
  const handleGoogleSignIn = async () => {
    try {
      await GoogleSignin.hasPlayServices();
      const userInfo = await GoogleSignin.signIn();
      // Send userInfo.idToken to your backend for verification
    } catch (error) {
      // Handle errors (see Section 5.4 for error codes)
    }
  };

  return (
    <GoogleSigninButton
      size={GoogleSigninButton.Size.Wide}
      color={GoogleSigninButton.Color.Dark}
      onPress={handleGoogleSignIn}
    />
  );
}
```

### 8.3 Account Linking

When a user signs up with email, then later tries to sign in with Google using the same email, you need to handle account linking. Each provider handles this differently:

- **Clerk:** Automatically links accounts with the same email (configurable)
- **Supabase:** Links accounts if the email is verified on both sides
- **Firebase:** Requires explicit linking via `linkWithCredential()`
- **Auth.js:** Configurable via the `allowDangerousEmailAccountLinking` option (name says it all -- be careful)

```typescript
// Firebase account linking example
import auth from '@react-native-firebase/auth';

async function linkGoogleAccount(idToken: string) {
  const currentUser = auth().currentUser;
  if (!currentUser) throw new Error('Not signed in');

  const googleCredential = auth.GoogleAuthProvider.credential(idToken);

  try {
    await currentUser.linkWithCredential(googleCredential);
  } catch (error: any) {
    if (error.code === 'auth/credential-already-in-use') {
      // Another account already uses this Google account.
      // You might want to merge the accounts.
      throw new Error(
        'This Google account is already linked to another user.'
      );
    }
    throw error;
  }
}
```

---

## 9. PASSKEYS & BIOMETRIC AUTH

### 9.1 Passkeys (WebAuthn/FIDO2)

Passkeys are the future of authentication. They replace passwords entirely with cryptographic key pairs stored on the user's device. The private key never leaves the device. Authentication is done via biometric (Face ID, fingerprint) or device PIN.

**Why passkeys matter:**
- Phishing-proof: the key is bound to the domain, so a fake site cannot use it
- No passwords to breach, leak, or stuff
- Faster than typing a password
- Cross-device sync via iCloud Keychain or Google Password Manager

**Current support (as of 2026):**
- iOS 16+, Android 9+ (via Google Play Services), Chrome, Safari, Firefox
- React Native: Limited. The best approach is to use passkeys via your auth provider (Clerk supports them natively) or through a WebView-based ceremony

```typescript
// Passkey registration with Clerk (simplest path)
import { useUser } from '@clerk/clerk-expo';

function PasskeyRegistration() {
  const { user } = useUser();

  const registerPasskey = async () => {
    try {
      // Clerk handles the entire WebAuthn ceremony
      await user?.createPasskey();
      Alert.alert('Success', 'Passkey registered successfully.');
    } catch (err) {
      console.error('Passkey registration failed:', err);
    }
  };

  return (
    <TouchableOpacity onPress={registerPasskey}>
      <Text>Register Passkey</Text>
    </TouchableOpacity>
  );
}
```

```typescript
// Web (Next.js) passkey authentication with Clerk
// Clerk's <SignIn /> component handles passkeys automatically.
// For custom UI:
import { useSignIn } from '@clerk/nextjs';

function PasskeySignIn() {
  const { signIn, setActive } = useSignIn();

  const handlePasskeySignIn = async () => {
    try {
      const result = await signIn!.authenticateWithPasskey();
      if (result.status === 'complete') {
        await setActive!({ session: result.createdSessionId });
      }
    } catch (err) {
      console.error('Passkey auth failed:', err);
    }
  };

  return <button onClick={handlePasskeySignIn}>Sign in with Passkey</button>;
}
```

### 9.2 Biometric Gating with expo-local-authentication

Biometric auth is different from passkeys. Biometrics gate access to something already on the device (like a stored token); passkeys are a complete authentication mechanism.

Use biometric gating when:
- The user opens the app and you want to verify it is them before showing data
- The user performs a sensitive action (transfer money, change email)
- You want to protect access to stored credentials

```bash
npx expo install expo-local-authentication
```

```typescript
// packages/auth/src/biometric-gate.ts
import * as LocalAuthentication from 'expo-local-authentication';
import { Platform, Alert } from 'react-native';

export async function checkBiometricSupport() {
  const hasHardware = await LocalAuthentication.hasHardwareAsync();
  const isEnrolled = await LocalAuthentication.isEnrolledAsync();
  const supportedTypes =
    await LocalAuthentication.supportedAuthenticationTypesAsync();

  return {
    isAvailable: hasHardware && isEnrolled,
    types: supportedTypes.map((type) => {
      switch (type) {
        case LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION:
          return 'face';
        case LocalAuthentication.AuthenticationType.FINGERPRINT:
          return 'fingerprint';
        case LocalAuthentication.AuthenticationType.IRIS:
          return 'iris';
        default:
          return 'unknown';
      }
    }),
  };
}

/**
 * Prompt for biometric authentication.
 * Returns true if the user authenticated successfully.
 */
export async function authenticateWithBiometrics(
  reason: string = 'Verify your identity'
): Promise<boolean> {
  const { isAvailable } = await checkBiometricSupport();

  if (!isAvailable) {
    // Fall back to PIN/password on devices without biometrics
    return true; // Or show a PIN entry screen
  }

  const result = await LocalAuthentication.authenticateAsync({
    promptMessage: reason,
    // On iOS, show "Use Passcode" fallback
    fallbackLabel: 'Use Passcode',
    // Cancel label
    cancelLabel: 'Cancel',
    // Disable the device passcode fallback if you want biometric-only
    disableDeviceFallback: false,
  });

  return result.success;
}

/**
 * Biometric gate for sensitive operations.
 * Wraps an action with biometric verification.
 */
export async function withBiometricGate<T>(
  action: () => Promise<T>,
  reason?: string
): Promise<T> {
  const authenticated = await authenticateWithBiometrics(reason);

  if (!authenticated) {
    throw new Error('Biometric authentication failed');
  }

  return action();
}

// Usage:
// const balance = await withBiometricGate(
//   () => fetchAccountBalance(),
//   'Authenticate to view your balance'
// );
```

### 9.3 Biometric-Protected Token Access

```typescript
// packages/auth/src/biometric-token-storage.ts
import * as SecureStore from 'expo-secure-store';
import { authenticateWithBiometrics } from './biometric-gate';

/**
 * Store a token that requires biometric auth to access.
 * On iOS, this uses Keychain access control flags.
 */
export async function saveBiometricProtectedToken(
  key: string,
  value: string
) {
  await SecureStore.setItemAsync(key, value, {
    // On iOS: requires Face ID/Touch ID to read this item
    requireAuthentication: true,
    // Keychain accessibility: only when device is unlocked
    keychainAccessible:
      SecureStore.WHEN_PASSCODE_SET_THIS_DEVICE_ONLY,
  });
}

export async function getBiometricProtectedToken(
  key: string
): Promise<string | null> {
  try {
    // On iOS, this automatically triggers the biometric prompt
    // On Android, you may need to manually prompt first
    return await SecureStore.getItemAsync(key, {
      requireAuthentication: true,
    });
  } catch {
    return null;
  }
}
```

---

## 10. PROTECTED ROUTES

### 10.1 React Native: Auth-Gated Navigation

The pattern for React Native is to use layout routes in Expo Router that check auth state and redirect accordingly:

```typescript
// apps/mobile/app/_layout.tsx
import { useEffect, useState } from 'react';
import { Slot, useRouter, useSegments } from 'expo-router';
import { useAuth } from '@clerk/clerk-expo'; // or your auth provider
import * as SplashScreen from 'expo-splash-screen';

// Keep the splash screen visible while we check auth
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const { isLoaded, isSignedIn } = useAuth();
  const segments = useSegments();
  const router = useRouter();
  const [isMounted, setIsMounted] = useState(false);

  useEffect(() => {
    setIsMounted(true);
  }, []);

  useEffect(() => {
    if (!isLoaded || !isMounted) return;

    // Hide splash screen once auth state is determined
    SplashScreen.hideAsync();

    const inAuthGroup = segments[0] === '(auth)';

    if (!isSignedIn && !inAuthGroup) {
      // Not signed in and not on an auth screen -- redirect to sign-in
      router.replace('/(auth)/sign-in');
    } else if (isSignedIn && inAuthGroup) {
      // Signed in but on an auth screen -- redirect to main app
      router.replace('/(tabs)/home');
    }
  }, [isSignedIn, isLoaded, segments, isMounted]);

  if (!isLoaded) {
    // Return null while loading -- splash screen is still visible
    return null;
  }

  return <Slot />;
}
```

### 10.2 React Native: Role-Based Navigation

```typescript
// apps/mobile/app/(tabs)/admin/_layout.tsx
import { useUser } from '@clerk/clerk-expo';
import { Redirect, Stack } from 'expo-router';

export default function AdminLayout() {
  const { user } = useUser();
  const role = user?.publicMetadata?.role as string;

  if (role !== 'admin') {
    // Non-admin users cannot access admin routes
    return <Redirect href="/(tabs)/home" />;
  }

  return <Stack />;
}
```

### 10.3 Next.js: Middleware-Based Protection

The middleware approach is the most robust for Next.js because it runs before any rendering:

```typescript
// apps/web/middleware.ts
// See Section 3.4 for the Clerk middleware example
// See Section 4.3 for the Supabase middleware example
// See Section 6.3 for the Auth.js middleware example
```

### 10.4 Next.js: Server Component Auth Checks

For data fetching in Server Components, check auth at the component level:

```typescript
// apps/web/app/dashboard/page.tsx
import { auth } from '@clerk/nextjs/server'; // or your auth provider
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const { userId } = await auth();

  if (!userId) {
    redirect('/sign-in');
  }

  // Fetch data with the authenticated user's context
  const items = await db.item.findMany({
    where: { userId },
  });

  return <ItemList items={items} />;
}
```

### 10.5 Next.js: Protecting Server Actions

```typescript
// apps/web/app/actions/items.ts
'use server';

import { auth } from '@clerk/nextjs/server';
import { revalidatePath } from 'next/cache';

export async function createItem(formData: FormData) {
  const { userId } = await auth();

  if (!userId) {
    throw new Error('Unauthorized');
  }

  const title = formData.get('title') as string;

  await db.item.create({
    data: {
      title,
      userId,
    },
  });

  revalidatePath('/dashboard');
}
```

### 10.6 Next.js: Protecting API Route Handlers

```typescript
// apps/web/app/api/items/route.ts
import { auth } from '@clerk/nextjs/server';
import { NextResponse } from 'next/server';

export async function GET() {
  const { userId } = await auth();

  if (!userId) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }

  const items = await db.item.findMany({ where: { userId } });
  return NextResponse.json(items);
}

export async function POST(request: Request) {
  const { userId } = await auth();

  if (!userId) {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 401 }
    );
  }

  const body = await request.json();

  const item = await db.item.create({
    data: {
      ...body,
      userId,
    },
  });

  return NextResponse.json(item, { status: 201 });
}
```

---

## 11. ROLE-BASED ACCESS CONTROL (RBAC)

### 11.1 The Three Layers of RBAC

RBAC is not a single implementation -- it is a defense-in-depth strategy that operates at three layers:

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: UI (Conditional Rendering)                │
│                                                      │
│  - Show/hide UI elements based on role               │
│  - Convenience for users, NOT a security boundary    │
│  - "The admin panel button is hidden" does not mean   │
│    the admin API is protected                         │
│                                                      │
│  If bypassed: User sees UI they shouldn't, but       │
│  cannot do anything because Layers 2 and 3 block it  │
├─────────────────────────────────────────────────────┤
│  Layer 2: API / Middleware (Server-side checks)      │
│                                                      │
│  - Check role/permissions on every API request        │
│  - This is the real security boundary                │
│  - Runs in Next.js middleware, API routes, or         │
│    Server Actions                                    │
│                                                      │
│  If bypassed: Database layer still prevents access    │
├─────────────────────────────────────────────────────┤
│  Layer 3: Database (RLS / Custom Claims)             │
│                                                      │
│  - Supabase: Row Level Security policies              │
│  - Firebase: Security Rules + Custom Claims           │
│  - SQL: WHERE clauses enforced at the ORM level       │
│                                                      │
│  If bypassed: You have a database vulnerability,      │
│  which is a much bigger problem                      │
└─────────────────────────────────────────────────────┘
```

**Rule: RBAC in the UI is a convenience. RBAC in the API is a requirement. RBAC in the database is insurance.**

### 11.2 Defining Roles and Permissions

```typescript
// packages/auth/src/rbac.ts

/**
 * Define roles as a hierarchy.
 * Each role includes all permissions of the roles below it.
 */
export const ROLES = {
  viewer: {
    level: 0,
    permissions: [
      'read:items',
      'read:own_profile',
      'update:own_profile',
    ],
  },
  editor: {
    level: 1,
    permissions: [
      'read:items',
      'create:items',
      'update:own_items',
      'delete:own_items',
      'read:own_profile',
      'update:own_profile',
    ],
  },
  admin: {
    level: 2,
    permissions: [
      'read:items',
      'create:items',
      'update:items',       // any item, not just own
      'delete:items',       // any item, not just own
      'read:users',
      'update:users',
      'manage:roles',
      'read:own_profile',
      'update:own_profile',
      'read:analytics',
    ],
  },
  owner: {
    level: 3,
    permissions: [
      'read:items',
      'create:items',
      'update:items',
      'delete:items',
      'read:users',
      'update:users',
      'delete:users',
      'manage:roles',
      'manage:billing',
      'manage:organization',
      'read:own_profile',
      'update:own_profile',
      'read:analytics',
    ],
  },
} as const;

export type Role = keyof typeof ROLES;
export type Permission = (typeof ROLES)[Role]['permissions'][number];

/**
 * Check if a role has a specific permission.
 */
export function hasPermission(role: Role, permission: Permission): boolean {
  return (ROLES[role]?.permissions as readonly string[])?.includes(permission) ?? false;
}

/**
 * Check if a role meets a minimum level.
 */
export function hasMinimumRole(userRole: Role, requiredRole: Role): boolean {
  return (ROLES[userRole]?.level ?? 0) >= (ROLES[requiredRole]?.level ?? 0);
}
```

### 11.3 RBAC in React Native (Conditional UI)

```typescript
// packages/auth/src/components/require-permission.tsx
import { useCurrentUser } from '../hooks/use-current-user';
import { hasPermission, type Permission } from '../rbac';

interface RequirePermissionProps {
  permission: Permission;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

/**
 * Conditionally render children based on the user's permissions.
 *
 * IMPORTANT: This is a UI convenience, not a security boundary.
 * Always enforce permissions on the server.
 */
export function RequirePermission({
  permission,
  children,
  fallback = null,
}: RequirePermissionProps) {
  const { user } = useCurrentUser();

  if (!user || !hasPermission(user.role as any, permission)) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}

// Usage:
// <RequirePermission permission="manage:billing">
//   <BillingSettingsButton />
// </RequirePermission>
```

```typescript
// packages/auth/src/hooks/use-permission.ts
import { useMemo } from 'react';
import { useCurrentUser } from './use-current-user';
import { hasPermission, hasMinimumRole, type Permission, type Role } from '../rbac';

export function usePermission(permission: Permission): boolean {
  const { user } = useCurrentUser();
  return useMemo(
    () => !!user && hasPermission(user.role as Role, permission),
    [user, permission]
  );
}

export function useMinimumRole(role: Role): boolean {
  const { user } = useCurrentUser();
  return useMemo(
    () => !!user && hasMinimumRole(user.role as Role, role),
    [user, role]
  );
}

// Usage:
// const canManageBilling = usePermission('manage:billing');
// const isAtLeastEditor = useMinimumRole('editor');
```

### 11.4 RBAC in Next.js (Middleware + Server Components)

```typescript
// packages/auth/src/server/require-role.ts
import { auth } from '@clerk/nextjs/server';
import { redirect } from 'next/navigation';
import { hasMinimumRole, type Role } from '../rbac';

/**
 * Server-side role check for Server Components.
 * Redirects if the user doesn't have the required role.
 */
export async function requireRole(minimumRole: Role) {
  const { userId } = await auth();

  if (!userId) {
    redirect('/sign-in');
  }

  const user = await currentUser();
  const userRole = (user?.publicMetadata?.role as Role) ?? 'viewer';

  if (!hasMinimumRole(userRole, minimumRole)) {
    redirect('/unauthorized');
  }

  return { userId, role: userRole };
}

// Usage in a Server Component:
// export default async function AdminPage() {
//   const { userId, role } = await requireRole('admin');
//   // ... render admin content
// }
```

### 11.5 RBAC at the Database Level

**Supabase (Row Level Security):**

```sql
-- Role-based RLS policies
-- Users have a 'role' field in their JWT metadata

-- Only admins can read all users
CREATE POLICY "Admins can read all users"
  ON users
  FOR SELECT
  USING (
    auth.jwt() ->> 'role' = 'admin'
    OR auth.uid() = id
  );

-- Only owners can delete users
CREATE POLICY "Owners can delete users"
  ON users
  FOR DELETE
  USING (
    auth.jwt() ->> 'role' = 'owner'
  );

-- Editors can create items
CREATE POLICY "Editors can create items"
  ON items
  FOR INSERT
  WITH CHECK (
    auth.jwt() ->> 'role' IN ('editor', 'admin', 'owner')
  );

-- Function to check organization membership + role
CREATE OR REPLACE FUNCTION has_org_role(org_id uuid, required_role text)
RETURNS boolean AS $$
  SELECT EXISTS (
    SELECT 1 FROM organization_members
    WHERE organization_id = org_id
      AND user_id = auth.uid()
      AND role = required_role
  );
$$ LANGUAGE sql SECURITY DEFINER;

-- Use in policies:
CREATE POLICY "Org admins can update org items"
  ON items
  FOR UPDATE
  USING (
    has_org_role(organization_id, 'admin')
  );
```

**Firebase (Security Rules + Custom Claims):**

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Helper function to check role
    function hasRole(role) {
      return request.auth != null
        && request.auth.token.role == role;
    }

    function isOwner(userId) {
      return request.auth != null
        && request.auth.uid == userId;
    }

    match /items/{itemId} {
      allow read: if request.auth != null;
      allow create: if hasRole('editor') || hasRole('admin');
      allow update: if hasRole('admin')
        || (hasRole('editor') && isOwner(resource.data.userId));
      allow delete: if hasRole('admin');
    }

    match /users/{userId} {
      allow read: if isOwner(userId) || hasRole('admin');
      allow update: if isOwner(userId);
      allow delete: if hasRole('owner');
    }
  }
}
```

---

## 12. MFA (MULTI-FACTOR AUTHENTICATION)

### 12.1 MFA Methods

| Method | Security | UX | Implementation Effort |
|---|---|---|---|
| TOTP (authenticator app) | High | Medium (requires app setup) | Low with managed providers |
| SMS OTP | Medium (SIM swap attacks) | Good (users know how) | Medium (SMS delivery) |
| Email OTP | Medium | Good | Low |
| Push notification | High | Excellent | High (requires FCM/APNs) |
| Hardware key (FIDO2) | Highest | Poor (requires physical device) | Medium |

**Recommendation:** Offer TOTP as the primary MFA method and SMS as a fallback. TOTP is more secure and does not depend on carrier SMS delivery.

### 12.2 TOTP with Clerk

```typescript
// apps/mobile/src/screens/mfa-setup.tsx
import { useUser } from '@clerk/clerk-expo';
import { useState, useEffect } from 'react';
import {
  View,
  Text,
  Image,
  TextInput,
  TouchableOpacity,
  Alert,
  StyleSheet,
} from 'react-native';

export default function MFASetupScreen() {
  const { user } = useUser();
  const [totpUri, setTotpUri] = useState<string | null>(null);
  const [qrCodeUrl, setQrCodeUrl] = useState<string | null>(null);
  const [backupCodes, setBackupCodes] = useState<string[]>([]);
  const [verificationCode, setVerificationCode] = useState('');

  useEffect(() => {
    setupTOTP();
  }, []);

  const setupTOTP = async () => {
    try {
      // Create a TOTP instance
      const totp = await user!.createTOTP();

      // The URI can be used to generate a QR code
      // or entered manually in an authenticator app
      setTotpUri(totp.uri);
      setQrCodeUrl(totp.uri); // Use a QR code library to render this

      // Backup codes for recovery
      if (totp.backupCodes) {
        setBackupCodes(totp.backupCodes);
      }
    } catch (err) {
      console.error('TOTP setup failed:', err);
      Alert.alert('Error', 'Failed to set up MFA.');
    }
  };

  const verifyAndEnable = async () => {
    try {
      // Verify the code from the authenticator app
      await user!.verifyTOTP({ code: verificationCode });
      Alert.alert('Success', 'MFA is now enabled on your account.');
    } catch (err) {
      Alert.alert('Error', 'Invalid code. Please try again.');
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Set Up Two-Factor Authentication</Text>

      <Text style={styles.instructions}>
        Scan this QR code with your authenticator app (Google Authenticator,
        1Password, Authy, etc.)
      </Text>

      {qrCodeUrl && (
        <View style={styles.qrContainer}>
          {/* Use a QR code library like react-native-qrcode-svg */}
          {/* <QRCode value={qrCodeUrl} size={200} /> */}
          <Text style={styles.manualCode}>
            Or enter this code manually:{'\n'}
            {totpUri}
          </Text>
        </View>
      )}

      <TextInput
        style={styles.input}
        placeholder="Enter 6-digit code"
        value={verificationCode}
        onChangeText={setVerificationCode}
        keyboardType="number-pad"
        maxLength={6}
        textContentType="oneTimeCode"
      />

      <TouchableOpacity style={styles.button} onPress={verifyAndEnable}>
        <Text style={styles.buttonText}>Verify & Enable MFA</Text>
      </TouchableOpacity>

      {backupCodes.length > 0 && (
        <View style={styles.backupCodesContainer}>
          <Text style={styles.backupCodesTitle}>Backup Codes</Text>
          <Text style={styles.backupCodesWarning}>
            Save these codes in a safe place. Each can be used once if you
            lose access to your authenticator app.
          </Text>
          {backupCodes.map((code, i) => (
            <Text key={i} style={styles.backupCode}>
              {code}
            </Text>
          ))}
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24 },
  title: { fontSize: 24, fontWeight: '700', marginBottom: 16 },
  instructions: { fontSize: 14, color: '#666', marginBottom: 24, lineHeight: 20 },
  qrContainer: { alignItems: 'center', marginBottom: 24 },
  manualCode: { fontSize: 12, color: '#888', marginTop: 12, textAlign: 'center' },
  input: {
    borderWidth: 1, borderColor: '#ddd', borderRadius: 12,
    padding: 16, fontSize: 24, letterSpacing: 8, textAlign: 'center',
    marginBottom: 16,
  },
  button: {
    backgroundColor: '#000', borderRadius: 12, padding: 16,
    alignItems: 'center',
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  backupCodesContainer: {
    marginTop: 32, padding: 16, backgroundColor: '#fff9db',
    borderRadius: 12,
  },
  backupCodesTitle: { fontWeight: '700', marginBottom: 8 },
  backupCodesWarning: { fontSize: 12, color: '#666', marginBottom: 12 },
  backupCode: { fontFamily: 'monospace', fontSize: 14, marginBottom: 4 },
});
```

### 12.3 MFA Verification During Sign-In

```typescript
// apps/mobile/app/(auth)/mfa.tsx
import { useSignIn } from '@clerk/clerk-expo';
import { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, Alert } from 'react-native';
import { useRouter } from 'expo-router';

export default function MFAScreen() {
  const { signIn, setActive } = useSignIn();
  const router = useRouter();
  const [code, setCode] = useState('');
  const [loading, setLoading] = useState(false);
  const [useBackupCode, setUseBackupCode] = useState(false);

  const handleVerify = async () => {
    if (!signIn) return;
    setLoading(true);

    try {
      const result = await signIn.attemptSecondFactor({
        strategy: useBackupCode ? 'backup_code' : 'totp',
        code,
      });

      if (result.status === 'complete') {
        await setActive!({ session: result.createdSessionId });
        router.replace('/(tabs)/home');
      }
    } catch (err: any) {
      Alert.alert(
        'Error',
        err.errors?.[0]?.longMessage ?? 'Invalid code.'
      );
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={{ flex: 1, padding: 24, justifyContent: 'center' }}>
      <Text style={{ fontSize: 24, fontWeight: '700', marginBottom: 8 }}>
        Two-Factor Authentication
      </Text>
      <Text style={{ color: '#666', marginBottom: 24 }}>
        {useBackupCode
          ? 'Enter one of your backup codes'
          : 'Enter the code from your authenticator app'}
      </Text>

      <TextInput
        value={code}
        onChangeText={setCode}
        placeholder={useBackupCode ? 'Backup code' : '000000'}
        keyboardType={useBackupCode ? 'default' : 'number-pad'}
        maxLength={useBackupCode ? 20 : 6}
        textContentType="oneTimeCode"
        autoComplete="one-time-code"
        style={{
          borderWidth: 1,
          borderColor: '#ddd',
          borderRadius: 12,
          padding: 16,
          fontSize: useBackupCode ? 16 : 24,
          letterSpacing: useBackupCode ? 0 : 8,
          textAlign: 'center',
          marginBottom: 16,
        }}
      />

      <TouchableOpacity
        onPress={handleVerify}
        disabled={loading}
        style={{
          backgroundColor: '#000',
          borderRadius: 12,
          padding: 16,
          alignItems: 'center',
        }}
      >
        <Text style={{ color: '#fff', fontSize: 16, fontWeight: '600' }}>
          {loading ? 'Verifying...' : 'Verify'}
        </Text>
      </TouchableOpacity>

      <TouchableOpacity
        onPress={() => setUseBackupCode(!useBackupCode)}
        style={{ marginTop: 16, alignItems: 'center' }}
      >
        <Text style={{ color: '#666' }}>
          {useBackupCode
            ? 'Use authenticator app instead'
            : 'Lost your authenticator? Use backup code'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}
```

### 12.4 MFA with Supabase

```typescript
// Supabase MFA (TOTP)
import { supabase } from './client';

// Enroll in MFA
export async function enrollMFA() {
  const { data, error } = await supabase.auth.mfa.enroll({
    factorType: 'totp',
    friendlyName: 'Authenticator App',
  });

  if (error) throw error;

  // data.totp.qr_code contains a base64-encoded QR code image
  // data.totp.uri contains the TOTP URI for manual entry
  return {
    qrCode: data.totp.qr_code,
    uri: data.totp.uri,
    factorId: data.id,
  };
}

// Verify and activate MFA
export async function verifyMFA(factorId: string, code: string) {
  const { data: challenge, error: challengeError } =
    await supabase.auth.mfa.challenge({ factorId });

  if (challengeError) throw challengeError;

  const { data, error } = await supabase.auth.mfa.verify({
    factorId,
    challengeId: challenge.id,
    code,
  });

  if (error) throw error;
  return data;
}

// Check if MFA is required during sign-in
export async function checkMFAStatus() {
  const { data, error } =
    await supabase.auth.mfa.getAuthenticatorAssuranceLevel();

  if (error) throw error;

  return {
    currentLevel: data.currentLevel, // 'aal1' or 'aal2'
    nextLevel: data.nextLevel,       // what level is needed
    needsMFA: data.currentLevel === 'aal1' && data.nextLevel === 'aal2',
  };
}
```

---

## 13. SESSION MANAGEMENT

### 13.1 Cookie-Based vs Token-Based

| Aspect | Cookie-Based (Web) | Token-Based (Mobile) |
|---|---|---|
| Storage | httpOnly cookie | Secure Store (Keychain/Keystore) |
| Sent automatically? | Yes (browser handles it) | No (must add Authorization header) |
| CSRF vulnerable? | Yes (need CSRF tokens) | No (not sent automatically) |
| Cross-domain? | Limited (SameSite, domain rules) | Unlimited |
| Revocable? | Yes (clear cookie, invalidate session) | Yes (revoke refresh token) |
| Works offline? | Only for cached pages | Yes (token is on device) |
| SSR compatible? | Yes (cookies sent with every request) | No (tokens are in app memory) |

**For your monorepo:**
- **Next.js (web):** Use httpOnly cookies. Clerk and Auth.js handle this automatically.
- **React Native (mobile):** Use token-based auth stored in Secure Store. Clerk's Expo SDK handles this automatically.

### 13.2 Session Duration Strategies

**Sliding expiry:** The session is extended every time the user is active. A 30-minute session that resets on each request means the user is only logged out after 30 minutes of inactivity.

**Absolute expiry:** The session expires at a fixed time regardless of activity. A 24-hour session expires 24 hours after creation, even if the user is actively using the app.

**Recommended approach:** Use both.

```typescript
// Session configuration example
const SESSION_CONFIG = {
  // Access token: absolute expiry, short-lived
  accessToken: {
    expiresIn: '15m',    // Absolute: dies after 15 minutes
    // No sliding -- issue a new one via refresh token
  },

  // Refresh token: absolute expiry, long-lived
  refreshToken: {
    expiresIn: '30d',    // Absolute: dies after 30 days
    // No sliding -- user must re-authenticate after 30 days
    // This is a security requirement for sensitive apps
  },

  // Web session cookie: sliding + absolute
  sessionCookie: {
    maxAge: 7 * 24 * 60 * 60, // Absolute: 7 days max
    rolling: true,              // Sliding: reset on each request
    // User is logged out after 7 days OR after a period of inactivity
    // (determined by the cookie's maxAge resetting)
  },
};
```

### 13.3 Concurrent Session Handling

Should users be able to sign in from multiple devices simultaneously? The answer depends on your app:

```typescript
// packages/auth/src/session-manager.ts

interface SessionConfig {
  // Maximum number of concurrent sessions
  maxSessions: number;
  // What to do when the limit is exceeded
  strategy: 'reject_new' | 'revoke_oldest' | 'revoke_all_except_new';
}

const SESSION_CONFIGS: Record<string, SessionConfig> = {
  // Social media app: allow many devices
  social: {
    maxSessions: 10,
    strategy: 'revoke_oldest',
  },
  // Banking app: strict, one session at a time
  banking: {
    maxSessions: 1,
    strategy: 'revoke_all_except_new',
  },
  // Standard SaaS: reasonable limit
  saas: {
    maxSessions: 5,
    strategy: 'revoke_oldest',
  },
};

// Server-side: enforce session limits
export async function enforceSessionLimit(
  userId: string,
  config: SessionConfig
) {
  const activeSessions = await prisma.refreshToken.findMany({
    where: {
      userId,
      expiresAt: { gt: new Date() },
    },
    orderBy: { createdAt: 'asc' },
  });

  if (activeSessions.length >= config.maxSessions) {
    switch (config.strategy) {
      case 'reject_new':
        throw new Error(
          'Maximum number of active sessions reached. ' +
          'Please sign out from another device.'
        );

      case 'revoke_oldest':
        // Delete the oldest session to make room
        await prisma.refreshToken.delete({
          where: { id: activeSessions[0].id },
        });
        break;

      case 'revoke_all_except_new':
        // Delete all existing sessions
        await prisma.refreshToken.deleteMany({
          where: { userId },
        });
        break;
    }
  }
}
```

### 13.4 Force Logout (Remote Session Revocation)

```typescript
// Server-side: force logout a user from all devices
export async function forceLogoutAll(userId: string) {
  // Delete all refresh tokens
  await prisma.refreshToken.deleteMany({
    where: { userId },
  });

  // With Clerk:
  // await clerkClient.users.deleteUser(userId); // or use sessions API

  // With Supabase:
  // await supabaseAdmin.auth.admin.signOut(userId, 'global');

  // With Firebase:
  // await getAuth().revokeRefreshTokens(userId);
}

// Server-side: force logout a specific session
export async function forceLogoutSession(sessionId: string) {
  await prisma.refreshToken.delete({
    where: { id: sessionId },
  });
}

// Client-side: React Native -- detect force logout
// The authEventEmitter pattern from Section 7.4 handles this.
// When the refresh token is revoked, the next refresh attempt fails,
// triggering SESSION_EXPIRED and redirecting to sign-in.
```

### 13.5 Session Display (Active Sessions Screen)

```typescript
// apps/mobile/src/screens/active-sessions.tsx
import { useUser } from '@clerk/clerk-expo';
import { useSessions } from '@clerk/clerk-expo';
import { View, Text, FlatList, TouchableOpacity, Alert } from 'react-native';

export function ActiveSessionsScreen() {
  const { user } = useUser();
  const { sessions } = useSessions();

  const handleRevokeSession = async (sessionId: string) => {
    Alert.alert(
      'Revoke Session',
      'This will sign out the device associated with this session.',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Revoke',
          style: 'destructive',
          onPress: async () => {
            try {
              await user?.sessions.find(
                (s) => s.id === sessionId
              )?.revoke();
            } catch (err) {
              Alert.alert('Error', 'Failed to revoke session.');
            }
          },
        },
      ]
    );
  };

  return (
    <View style={{ flex: 1, padding: 24 }}>
      <Text style={{ fontSize: 24, fontWeight: '700', marginBottom: 16 }}>
        Active Sessions
      </Text>
      <FlatList
        data={sessions}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View
            style={{
              padding: 16,
              borderBottomWidth: 1,
              borderColor: '#eee',
              flexDirection: 'row',
              justifyContent: 'space-between',
              alignItems: 'center',
            }}
          >
            <View>
              <Text style={{ fontWeight: '600' }}>
                {item.latestActivity?.deviceType ?? 'Unknown Device'}
              </Text>
              <Text style={{ color: '#888', fontSize: 12 }}>
                Last active: {new Date(item.lastActiveAt).toLocaleDateString()}
              </Text>
              <Text style={{ color: '#888', fontSize: 12 }}>
                {item.latestActivity?.city}, {item.latestActivity?.country}
              </Text>
            </View>
            <TouchableOpacity
              onPress={() => handleRevokeSession(item.id)}
            >
              <Text style={{ color: 'red' }}>Revoke</Text>
            </TouchableOpacity>
          </View>
        )}
      />
    </View>
  );
}
```

---

## 14. THE COMPLETE AUTH ARCHITECTURE

Here is the end-to-end architecture for a production React Native + Next.js monorepo using Clerk:

### 14.1 Monorepo Structure

```
monorepo/
├── apps/
│   ├── mobile/                    # Expo / React Native
│   │   ├── app/
│   │   │   ├── _layout.tsx        # Root layout with ClerkProvider
│   │   │   ├── (auth)/
│   │   │   │   ├── _layout.tsx    # Auth layout (redirect if signed in)
│   │   │   │   ├── sign-in.tsx    # Sign-in screen
│   │   │   │   ├── sign-up.tsx    # Sign-up screen
│   │   │   │   └── mfa.tsx        # MFA verification
│   │   │   └── (tabs)/
│   │   │       ├── _layout.tsx    # Tabs layout (redirect if NOT signed in)
│   │   │       ├── home.tsx
│   │   │       ├── settings.tsx
│   │   │       └── admin/
│   │   │           └── _layout.tsx # Admin-only layout (role check)
│   │   └── src/
│   │       └── providers/
│   │           └── auth-provider.tsx
│   │
│   └── web/                       # Next.js
│       ├── app/
│       │   ├── layout.tsx         # Root layout with ClerkProvider
│       │   ├── (auth)/
│       │   │   ├── sign-in/
│       │   │   │   └── [[...sign-in]]/page.tsx
│       │   │   └── sign-up/
│       │   │       └── [[...sign-up]]/page.tsx
│       │   ├── dashboard/
│       │   │   └── page.tsx       # Protected (middleware + Server Component)
│       │   └── api/
│       │       ├── webhooks/
│       │       │   └── clerk/route.ts  # Clerk webhook handler
│       │       └── protected/
│       │           └── route.ts   # Auth-protected API route
│       └── middleware.ts           # Clerk middleware
│
├── packages/
│   ├── auth/                      # Shared auth utilities
│   │   └── src/
│   │       ├── rbac.ts            # Role/permission definitions
│   │       ├── hooks/
│   │       │   ├── use-current-user.ts
│   │       │   └── use-permission.ts
│   │       ├── components/
│   │       │   └── require-permission.tsx
│   │       └── types.ts           # Shared auth types
│   │
│   ├── api-client/                # Shared API client
│   │   └── src/
│   │       ├── client.ts          # Base client with auth headers
│   │       └── authenticated-client.ts  # Token refresh interceptor
│   │
│   └── supabase/ (or firebase/)   # If using Supabase/Firebase
│       └── src/
│           ├── client.ts
│           ├── client.native.ts
│           └── server.ts
│
└── .env.local                     # Environment variables
```

### 14.2 Shared Auth Types

```typescript
// packages/auth/src/types.ts

export interface AuthUser {
  id: string;
  email: string | null;
  name: string | null;
  avatar: string | null;
  role: UserRole;
  organizationId: string | null;
  organizationRole: OrgRole | null;
}

export type UserRole = 'viewer' | 'editor' | 'admin' | 'owner';
export type OrgRole = 'member' | 'admin' | 'owner';

export interface AuthSession {
  user: AuthUser;
  accessToken: string;
  expiresAt: number;
}

export interface AuthState {
  isLoaded: boolean;
  isSignedIn: boolean;
  user: AuthUser | null;
  session: AuthSession | null;
}

// Webhook event types (for Clerk webhooks)
export interface UserCreatedEvent {
  type: 'user.created';
  data: {
    id: string;
    email_addresses: Array<{
      email_address: string;
      verification: { status: string };
    }>;
    first_name: string | null;
    last_name: string | null;
    image_url: string;
  };
}

export interface UserUpdatedEvent {
  type: 'user.updated';
  data: UserCreatedEvent['data'];
}

export interface SessionCreatedEvent {
  type: 'session.created';
  data: {
    id: string;
    user_id: string;
  };
}
```

### 14.3 Clerk Webhook Handler (Syncing Users to Your Database)

```typescript
// apps/web/app/api/webhooks/clerk/route.ts
import { Webhook } from 'svix';
import { headers } from 'next/headers';
import { prisma } from '@/lib/prisma';
import type { UserCreatedEvent, UserUpdatedEvent } from '@repo/auth/types';

export async function POST(req: Request) {
  const WEBHOOK_SECRET = process.env.CLERK_WEBHOOK_SECRET;
  if (!WEBHOOK_SECRET) {
    throw new Error('Missing CLERK_WEBHOOK_SECRET');
  }

  // Get the Svix headers for verification
  const headerPayload = await headers();
  const svix_id = headerPayload.get('svix-id');
  const svix_timestamp = headerPayload.get('svix-timestamp');
  const svix_signature = headerPayload.get('svix-signature');

  if (!svix_id || !svix_timestamp || !svix_signature) {
    return new Response('Missing svix headers', { status: 400 });
  }

  const payload = await req.json();
  const body = JSON.stringify(payload);

  // Verify the webhook signature
  const wh = new Webhook(WEBHOOK_SECRET);
  let event: any;

  try {
    event = wh.verify(body, {
      'svix-id': svix_id,
      'svix-timestamp': svix_timestamp,
      'svix-signature': svix_signature,
    });
  } catch (err) {
    console.error('Webhook verification failed:', err);
    return new Response('Verification failed', { status: 400 });
  }

  // Handle events
  switch (event.type) {
    case 'user.created': {
      const data = event.data as UserCreatedEvent['data'];
      await prisma.user.create({
        data: {
          clerkId: data.id,
          email: data.email_addresses[0]?.email_address ?? '',
          name: `${data.first_name ?? ''} ${data.last_name ?? ''}`.trim(),
          avatar: data.image_url,
          role: 'viewer', // Default role for new users
        },
      });
      break;
    }

    case 'user.updated': {
      const data = event.data as UserUpdatedEvent['data'];
      await prisma.user.update({
        where: { clerkId: data.id },
        data: {
          email: data.email_addresses[0]?.email_address ?? '',
          name: `${data.first_name ?? ''} ${data.last_name ?? ''}`.trim(),
          avatar: data.image_url,
        },
      });
      break;
    }

    case 'user.deleted': {
      await prisma.user.delete({
        where: { clerkId: event.data.id },
      });
      break;
    }
  }

  return new Response('OK', { status: 200 });
}
```

### 14.4 The Complete Auth Flow (Sequence)

Here is everything that happens from app launch to authenticated API call:

```
┌──────────────────────────────────────────────────────────────────┐
│ APP LAUNCH                                                        │
│                                                                    │
│ 1. Splash screen shown (SplashScreen.preventAutoHideAsync)       │
│ 2. ClerkProvider initializes                                      │
│ 3. Clerk reads cached session from expo-secure-store              │
│ 4. If valid session found → skip to step 11                       │
│ 5. If no session → continue to sign-in                           │
│                                                                    │
│ SIGN-IN                                                           │
│                                                                    │
│ 6. User enters email/password OR taps social login                │
│ 7. For social login: PKCE flow via system browser                │
│ 8. Clerk verifies credentials, checks MFA requirement            │
│ 9. If MFA required → navigate to MFA screen → verify code        │
│ 10. Clerk issues session, stores tokens in secure storage         │
│                                                                    │
│ AUTHENTICATED STATE                                               │
│                                                                    │
│ 11. Splash screen hidden                                          │
│ 12. Root layout detects isSignedIn → shows (tabs) layout         │
│ 13. App renders authenticated UI                                  │
│ 14. RBAC components check role → show/hide admin features         │
│                                                                    │
│ API CALLS                                                         │
│                                                                    │
│ 15. Component needs data → calls authFetch(endpoint)             │
│ 16. authFetch checks if access token expired                     │
│ 17. If expired → refresh via Clerk's getToken() (automatic)      │
│ 18. Request sent with Authorization: Bearer <token>               │
│ 19. Server verifies token (Clerk middleware or manual verify)     │
│ 20. Server checks permissions (RBAC layer 2)                     │
│ 21. Database enforces RLS (RBAC layer 3)                          │
│ 22. Response returned to client                                   │
│                                                                    │
│ SESSION EXPIRY                                                    │
│                                                                    │
│ 23. Refresh token expires (30 days) or is revoked                │
│ 24. Next API call fails with 401                                  │
│ 25. authEventEmitter emits SESSION_EXPIRED                       │
│ 26. Auth provider catches event, clears state                    │
│ 27. Root layout detects !isSignedIn → redirects to sign-in       │
│ 28. Query cache cleared (queryClient.clear())                    │
└──────────────────────────────────────────────────────────────────┘
```

### 14.5 Auth Checklist for Production

Before shipping your auth implementation to production, verify every item:

**Security:**
- [ ] Tokens stored in `expo-secure-store` (never AsyncStorage)
- [ ] Web cookies are `httpOnly`, `Secure`, and `SameSite=Lax` (or `Strict`)
- [ ] Access tokens expire in 15 minutes or less
- [ ] Refresh token rotation is enabled
- [ ] PKCE is used for all OAuth flows (no implicit flow)
- [ ] OAuth uses system browser, never WebView
- [ ] Server validates tokens on every protected request (not just client-side checks)
- [ ] Rate limiting on auth endpoints (sign-in, sign-up, token refresh)
- [ ] Account lockout after N failed attempts
- [ ] CSRF protection on web forms

**Social Login:**
- [ ] Apple Sign In is implemented if any social login is offered
- [ ] Apple Sign In button follows Apple's HIG (design guidelines)
- [ ] Apple Sign In appears as prominently as other social login options
- [ ] Google Sign In uses native SDK for mobile, OAuth for web
- [ ] Account linking handles same-email across providers

**MFA:**
- [ ] TOTP enrollment flow with QR code and manual entry option
- [ ] Backup codes generated and shown to user (one time)
- [ ] MFA verification during sign-in
- [ ] Recovery flow for lost authenticator device

**RBAC:**
- [ ] Roles and permissions defined in shared package
- [ ] UI checks permissions (Layer 1 -- convenience)
- [ ] API/middleware checks permissions (Layer 2 -- security boundary)
- [ ] Database enforces access control (Layer 3 -- RLS/security rules)
- [ ] Default role assigned to new users (principle of least privilege)

**Session Management:**
- [ ] Session persistence across app restarts (mobile)
- [ ] Concurrent session limits enforced (if applicable)
- [ ] Force logout capability (admin action)
- [ ] Account deletion (required by App Store and Google Play)
- [ ] Session displayed with device info and revoke option

**User Experience:**
- [ ] Splash screen shown while auth state loads
- [ ] Smooth redirect: unauthenticated users go to sign-in, authenticated users skip it
- [ ] Token refresh is invisible (no random sign-outs)
- [ ] Error messages are helpful but do not leak information ("Invalid credentials" not "User not found")
- [ ] Loading states on all auth actions (sign-in, sign-up, MFA verify)
- [ ] Keyboard avoidance on sign-in forms
- [ ] Password visibility toggle
- [ ] Email and password fields use correct `textContentType` and `autoComplete` for autofill

**Webhooks:**
- [ ] Clerk/Supabase/Firebase webhooks configured for user sync
- [ ] Webhook signatures verified
- [ ] Webhook handler is idempotent (same event processed multiple times = same result)

---

## DECISION GUIDE: QUICK REFERENCE

**"I'm building a React Native + Next.js app and want auth that just works."**
Use Clerk. Follow Section 3.

**"I'm already using Supabase for my database."**
Use Supabase Auth. Follow Section 4. The RLS integration is too good to ignore.

**"I need phone/SMS authentication as a primary login method."**
Use Firebase Auth. Follow Section 5. Firebase's phone auth is the most reliable.

**"I'm building a web-only Next.js app."**
Use Auth.js. Follow Section 6. Zero cost, maximum flexibility.

**"I need to self-host everything."**
Use Supabase Auth (self-hosted) or Stack Auth. Both are open source.

**"My boss says we need to build auth from scratch."**
Show your boss Sections 1.1 and 7, explain the security surface area, and suggest a managed provider. If they insist, use the token management patterns from Section 7 and protect yourself with the security checklist in Section 14.5.

---

## FURTHER READING

- **RFC 7636 (PKCE):** https://datatracker.ietf.org/doc/html/rfc7636
- **RFC 9700 (OAuth 2.0 Security BCP):** https://datatracker.ietf.org/doc/html/rfc9700
- **Clerk Docs:** https://clerk.com/docs
- **Supabase Auth Docs:** https://supabase.com/docs/guides/auth
- **Firebase Auth Docs:** https://firebase.google.com/docs/auth
- **Auth.js Docs:** https://authjs.dev
- **Apple Sign In Guidelines:** https://developer.apple.com/sign-in-with-apple/get-started/
- **OWASP Mobile Top 10:** https://owasp.org/www-project-mobile-top-10/
- **WebAuthn Guide:** https://webauthn.guide
- **Expo Auth Session:** https://docs.expo.dev/versions/latest/sdk/auth-session/
- **Expo Secure Store:** https://docs.expo.dev/versions/latest/sdk/securestore/
