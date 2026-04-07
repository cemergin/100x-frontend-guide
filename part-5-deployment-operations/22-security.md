<!--
  CHAPTER: 22
  TITLE: Security & Data Protection
  PART: V — Deployment & Operations
  PREREQS: Chapters 5, 6
  KEY_TOPICS: secure storage, API keys, certificate pinning, biometrics, OWASP Mobile Top 10, OAuth2 PKCE, token management, code obfuscation, ProGuard, jailbreak detection
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 22: Security & Data Protection

> **Part V — Deployment & Operations** | Prerequisites: Chapters 5, 6 | Difficulty: Intermediate

Here is the uncomfortable truth that most React Native tutorials skip entirely: **your app is not secure by default.** Your JavaScript bundle is readable. Your app binary is decompilable. Every API key you embed in your code is extractable. Every piece of data you store in AsyncStorage is sitting in plain text on the filesystem. Every network request can be intercepted by anyone with a proxy tool.

Most web developers coming to mobile carry an assumption that "the client is a black box" -- that because users cannot right-click "View Source" on a mobile app, the code is somehow hidden. It is not. On Android, anyone can download your APK, unzip it, and read your JavaScript bundle in a text editor. On iOS, it takes slightly more effort, but the result is the same. Assume that everything in your app binary is public.

This chapter is about building security into your React Native app from the ground up. Not security theater -- actual defenses that protect your users' data, your API keys, your authentication tokens, and your business logic. We will cover the OWASP Mobile Top 10, implement secure storage, build proper authentication flows, set up certificate pinning, handle biometrics, and deal with the reality that some users will run your app on jailbroken or rooted devices.

The goal is not to make your app impenetrable -- that is impossible. The goal is to make it expensive enough to attack that most adversaries move on to an easier target, and to ensure that when (not if) a breach occurs, the blast radius is as small as possible.

### In This Chapter
- The Mobile Security Mindset -- what makes mobile different from web
- Secure Storage -- expo-secure-store, react-native-keychain, MMKV, what goes where
- API Key Protection -- never bundle secrets in JS, the BFF pattern, EAS environment variables
- Authentication -- OAuth2 PKCE, token management, automatic refresh with interceptors
- Certificate Pinning -- what it is, how to implement it, the maintenance burden
- OWASP Mobile Top 10 -- each vulnerability with React Native examples and mitigations
- Code Obfuscation -- ProGuard/R8, Hermes bytecode, iOS bitcode
- Biometric Authentication -- Face ID, Touch ID, fingerprint, when to use it
- Jailbreak/Root Detection -- why it matters, available libraries, the cat-and-mouse game

### Related Chapters
- [Ch 5: Expo Platform] -- Expo Secure Store and environment variable handling
- [Ch 6: Project Structure] -- where sensitive configuration lives in your project
- [Ch 9: State Management] -- what should and should not be in global state
- [Ch 10: Data Fetching] -- API client setup relevant to token management
- [Ch 21: Firebase Console Mastery] -- securing Firebase credentials

---

## 1. THE MOBILE SECURITY MINDSET

### 1.1 Your JS Bundle Is Readable

Let me show you exactly how exposed your code is. When you build a React Native app, your JavaScript is bundled into a single file. On Android, it lives at `assets/index.android.bundle` inside the APK. Anyone can extract it:

```bash
# Extract an APK (it's just a ZIP file)
unzip app-release.apk -d extracted/

# If NOT using Hermes, the bundle is plain JavaScript:
cat extracted/assets/index.android.bundle | head -100
# You'll see your entire app's JavaScript, minified but readable

# If using Hermes, the bundle is bytecode:
file extracted/assets/index.android.bundle
# "Hermes JavaScript bytecode"
# Harder to read, but NOT encrypted -- tools exist to decompile it
```

**What this means for you:**
- Every string literal in your code is visible: API URLs, error messages, feature flag names
- Every API key hardcoded in JavaScript is extractable
- Your business logic is reverse-engineerable
- Your validation logic is bypassable (an attacker can see exactly what you check)

### 1.2 Your App Binary Is Decompilable

Beyond JavaScript, the native binary itself is decompilable:

- **Android:** Tools like `jadx` or `apktool` can decompile your APK to readable Java/Kotlin code. Every native module, every configuration file, every resource is exposed.
- **iOS:** While harder due to code signing and encryption, a jailbroken device can decrypt the binary, and tools like `class-dump` or `Hopper` can reverse-engineer the Objective-C/Swift code.

### 1.3 The Three Rules

Build every security decision on these three rules:

1. **Never trust the client.** Any data that comes from the app -- form inputs, tokens, device info, even the app version -- can be fabricated. Validate everything server-side.
2. **Never store secrets in the bundle.** API keys, encryption keys, signing secrets -- if it is in your JavaScript or native code, it is public.
3. **Minimize blast radius.** When (not if) something is compromised, limit what an attacker can access. Short-lived tokens, scoped permissions, and principle of least privilege.

---

## 2. SECURE STORAGE

The number one security mistake in React Native apps is storing sensitive data in the wrong place. Here is the definitive guide to what goes where.

### 2.1 The Storage Hierarchy

```
+-------------------------------+------------------+-------------------+
| Storage                       | Encrypted?       | Use For           |
+-------------------------------+------------------+-------------------+
| expo-secure-store /           | Yes (Keychain/   | Auth tokens,      |
| react-native-keychain         | Keystore)        | refresh tokens,   |
|                               |                  | encryption keys,  |
|                               |                  | biometric secrets |
+-------------------------------+------------------+-------------------+
| MMKV                          | Optional         | User preferences, |
|                               | (can enable      | cached UI state,  |
|                               | encryption)      | non-sensitive      |
|                               |                  | settings           |
+-------------------------------+------------------+-------------------+
| AsyncStorage                  | NO               | Non-sensitive      |
|                               |                  | data only.         |
|                               |                  | Never tokens.      |
|                               |                  | Never PII.         |
+-------------------------------+------------------+-------------------+
| In-memory only                | N/A (volatile)   | Decrypted secrets  |
| (React state / JS variables)  |                  | during active use  |
+-------------------------------+------------------+-------------------+
```

**The rule is simple: if the data would be a problem in a breach, it goes in Secure Store or Keychain. Everything else can go in MMKV. AsyncStorage is for things you would be comfortable printing on a billboard.**

### 2.2 expo-secure-store

`expo-secure-store` wraps the iOS Keychain and Android Keystore -- the operating system's built-in secure storage mechanisms. Data stored here is encrypted by the OS, protected by the device lock screen, and (on iOS) can be further protected by requiring biometric authentication to access.

```bash
npx expo install expo-secure-store
```

```typescript
// src/services/secure-storage.ts
import * as SecureStore from 'expo-secure-store';

const KEYS = {
  ACCESS_TOKEN: 'auth_access_token',
  REFRESH_TOKEN: 'auth_refresh_token',
  USER_ID: 'auth_user_id',
  BIOMETRIC_KEY: 'biometric_encryption_key',
  PIN_HASH: 'pin_hash',
} as const;

type StorageKey = (typeof KEYS)[keyof typeof KEYS];

class SecureStorageService {
  /**
   * Store a value securely.
   * On iOS: stored in the Keychain
   * On Android: encrypted with the Keystore
   */
  async set(key: StorageKey, value: string): Promise<void> {
    try {
      await SecureStore.setItemAsync(key, value, {
        // Require the device to be unlocked to access this value.
        // Options:
        // - WHEN_UNLOCKED: accessible when device is unlocked
        // - AFTER_FIRST_UNLOCK: accessible after first unlock
        //   (even if device is locked later -- good for background)
        // - WHEN_UNLOCKED_THIS_DEVICE_ONLY: same as WHEN_UNLOCKED
        //   but not included in iCloud backups
        keychainAccessible:
          SecureStore.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
      });
    } catch (error) {
      console.error(`SecureStore set failed for "${key}":`, error);
      throw new SecureStorageError(
        `Failed to store value for "${key}"`,
        error
      );
    }
  }

  /**
   * Retrieve a value from secure storage.
   * Returns null if the key does not exist.
   */
  async get(key: StorageKey): Promise<string | null> {
    try {
      return await SecureStore.getItemAsync(key);
    } catch (error) {
      console.error(`SecureStore get failed for "${key}":`, error);
      return null;
    }
  }

  /**
   * Delete a value from secure storage.
   */
  async delete(key: StorageKey): Promise<void> {
    try {
      await SecureStore.deleteItemAsync(key);
    } catch (error) {
      console.error(
        `SecureStore delete failed for "${key}":`,
        error
      );
    }
  }

  /**
   * Store auth tokens after login.
   */
  async setAuthTokens(tokens: {
    accessToken: string;
    refreshToken: string;
    userId: string;
  }): Promise<void> {
    await Promise.all([
      this.set(KEYS.ACCESS_TOKEN, tokens.accessToken),
      this.set(KEYS.REFRESH_TOKEN, tokens.refreshToken),
      this.set(KEYS.USER_ID, tokens.userId),
    ]);
  }

  /**
   * Retrieve auth tokens.
   */
  async getAuthTokens(): Promise<{
    accessToken: string | null;
    refreshToken: string | null;
    userId: string | null;
  }> {
    const [accessToken, refreshToken, userId] = await Promise.all([
      this.get(KEYS.ACCESS_TOKEN),
      this.get(KEYS.REFRESH_TOKEN),
      this.get(KEYS.USER_ID),
    ]);

    return { accessToken, refreshToken, userId };
  }

  /**
   * Clear all auth tokens. Call on logout.
   */
  async clearAuthTokens(): Promise<void> {
    await Promise.all([
      this.delete(KEYS.ACCESS_TOKEN),
      this.delete(KEYS.REFRESH_TOKEN),
      this.delete(KEYS.USER_ID),
    ]);
  }

  /**
   * Check if auth tokens exist (without reading them).
   */
  async hasAuthTokens(): Promise<boolean> {
    const token = await this.get(KEYS.ACCESS_TOKEN);
    return token !== null;
  }
}

class SecureStorageError extends Error {
  constructor(
    message: string,
    public readonly cause?: unknown
  ) {
    super(message);
    this.name = 'SecureStorageError';
  }
}

export const secureStorage = new SecureStorageService();
export { KEYS as SECURE_STORAGE_KEYS };
```

### 2.3 react-native-keychain

If you need more control than `expo-secure-store` provides -- like specifying the authentication policy per item, or using biometric-protected access -- `react-native-keychain` gives you lower-level access:

```bash
npx expo install react-native-keychain
```

```typescript
// src/services/keychain.ts
import * as Keychain from 'react-native-keychain';

class KeychainService {
  /**
   * Store credentials with biometric protection.
   * The user must authenticate with Face ID / Touch ID /
   * fingerprint to access these credentials.
   */
  async setBiometricProtectedCredentials(
    username: string,
    password: string
  ): Promise<boolean> {
    try {
      await Keychain.setGenericPassword(username, password, {
        // Require biometric authentication to access
        accessControl:
          Keychain.ACCESS_CONTROL.BIOMETRY_CURRENT_SET,
        // Store in the Secure Enclave (iOS) or StrongBox (Android)
        accessible:
          Keychain.ACCESSIBLE.WHEN_UNLOCKED_THIS_DEVICE_ONLY,
        // Custom service name for namespacing
        service: 'com.myapp.biometric-credentials',
        // Security level for Android
        securityLevel: Keychain.SECURITY_LEVEL.SECURE_HARDWARE,
      });
      return true;
    } catch (error) {
      console.error('Failed to store biometric credentials:', error);
      return false;
    }
  }

  /**
   * Retrieve credentials (will trigger biometric prompt).
   */
  async getBiometricProtectedCredentials(): Promise<{
    username: string;
    password: string;
  } | null> {
    try {
      const credentials = await Keychain.getGenericPassword({
        service: 'com.myapp.biometric-credentials',
        authenticationPrompt: {
          title: 'Authentication Required',
          subtitle: 'Verify your identity to continue',
          cancel: 'Cancel',
        },
      });

      if (credentials) {
        return {
          username: credentials.username,
          password: credentials.password,
        };
      }
      return null;
    } catch (error) {
      console.error(
        'Failed to retrieve biometric credentials:',
        error
      );
      return null;
    }
  }

  /**
   * Check what biometric types are available.
   */
  async getSupportedBiometryType(): Promise<string | null> {
    return Keychain.getSupportedBiometryType();
  }

  /**
   * Reset all stored credentials.
   */
  async resetCredentials(): Promise<void> {
    await Keychain.resetGenericPassword({
      service: 'com.myapp.biometric-credentials',
    });
  }
}

export const keychainService = new KeychainService();
```

### 2.4 What Goes Where: The Complete Guide

```typescript
// CORRECT: Tokens in Secure Store
await secureStorage.set('auth_access_token', token);

// CORRECT: Preferences in MMKV
mmkvStorage.set('theme', 'dark');
mmkvStorage.set('onboarding_complete', true);
mmkvStorage.set('last_viewed_tab', 'home');

// CORRECT: Cache in MMKV (non-sensitive)
mmkvStorage.set('cached_user_name', user.name);
mmkvStorage.set('cached_avatar_url', user.avatarUrl);

// WRONG: Token in AsyncStorage (not encrypted!)
// await AsyncStorage.setItem('token', accessToken);

// WRONG: Token in MMKV (not encrypted by default!)
// mmkvStorage.set('access_token', accessToken);

// WRONG: API key in JavaScript code
// const API_KEY = 'sk-1234567890abcdef';

// WRONG: User password stored anywhere on the device
// await secureStorage.set('password', user.password);
// (Use tokens, never store passwords)

// CORRECT: Sensitive data in memory only (never persisted)
const decryptedPayload = decrypt(encryptedData, key);
// Use decryptedPayload, then let it be garbage collected
```

### 2.5 MMKV with Encryption

If you need encrypted storage with better performance than Secure Store (for larger datasets), MMKV supports encryption:

```typescript
// src/services/encrypted-storage.ts
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';

/**
 * Create an encrypted MMKV instance.
 * The encryption key is stored in Secure Store (Keychain/Keystore).
 */
async function createEncryptedStorage(): Promise<MMKV> {
  // Get or generate an encryption key
  let encryptionKey = await SecureStore.getItemAsync(
    'mmkv_encryption_key'
  );

  if (!encryptionKey) {
    // Generate a random encryption key
    // In production, use a proper random generator:
    encryptionKey = generateRandomBase64(32);
    await SecureStore.setItemAsync(
      'mmkv_encryption_key',
      encryptionKey
    );
  }

  return new MMKV({
    id: 'encrypted-storage',
    encryptionKey,
  });
}

function generateRandomBase64(length: number): string {
  const chars =
    'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/';
  let result = '';
  const values = new Uint8Array(length);
  crypto.getRandomValues(values);
  for (let i = 0; i < length; i++) {
    result += chars[values[i] % chars.length];
  }
  return result;
}

// Usage:
// const encryptedStore = await createEncryptedStorage();
// encryptedStore.set('sensitive_but_large_data', jsonString);
```

---

## 3. API KEY PROTECTION

### 3.1 The Problem

This is the single most common security vulnerability in React Native apps:

```typescript
// DO NOT DO THIS
const STRIPE_SECRET_KEY = 'sk_live_abc123...';
const GOOGLE_MAPS_KEY = 'AIzaSy...';
const FIREBASE_API_KEY = 'AIzaSy...';
const INTERNAL_API_KEY = 'secret-key-12345';

// Every one of these is extractable from your bundle.
// On Android, literally: unzip the APK and grep for the string.
```

"But I use environment variables!" -- that does not help if those variables are bundled into the JavaScript at build time:

```typescript
// This is ALSO exposed -- it is replaced at build time
const API_KEY = process.env.EXPO_PUBLIC_API_KEY;
// After bundling, this becomes:
const API_KEY = 'the-actual-key-value';
```

**Any environment variable that starts with `EXPO_PUBLIC_` is embedded in the JavaScript bundle and is public.** The name literally tells you this.

### 3.2 The Solution: Never Bundle Secrets in JS

There are exactly two safe patterns for handling secrets:

**Pattern 1: Server-side proxy (BFF)**

Your mobile app never talks directly to third-party APIs that require secret keys. Instead, it talks to *your* backend, which holds the secrets and proxies the requests.

```
Mobile App                Your Backend (BFF)           Third-Party API
    |                          |                            |
    |-- POST /api/charge ----->|                            |
    |   { amount: 2999,       |                            |
    |     paymentMethodId }    |                            |
    |                          |-- POST /v1/charges ------->|
    |                          |   Authorization:            |
    |                          |   Bearer sk_live_abc123    |
    |                          |                            |
    |                          |<-- 200 OK -----------------|
    |<-- 200 OK ---------------|                            |
```

```typescript
// Mobile app -- NO secret keys anywhere
async function createCharge(
  amount: number,
  paymentMethodId: string
): Promise<ChargeResult> {
  // Talk to YOUR backend, not Stripe directly
  const response = await api.post('/api/payments/charge', {
    amount,
    paymentMethodId,
  });
  return response.data;
}
```

```typescript
// Backend (Node.js/Express example)
// The secret key lives here, in server-side environment variables
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

app.post('/api/payments/charge', async (req, res) => {
  const { amount, paymentMethodId } = req.body;

  const charge = await stripe.paymentIntents.create({
    amount,
    currency: 'usd',
    payment_method: paymentMethodId,
    confirm: true,
  });

  res.json({ id: charge.id, status: charge.status });
});
```

**Pattern 2: Restricted client-side keys**

Some keys are designed to be on the client (Firebase API keys, Google Maps keys for mobile). These are safe *if properly restricted*:

- **Firebase API keys:** Not actually secret. They identify your project but do not grant access. Security is enforced by Firebase Security Rules and App Check.
- **Google Maps keys:** Restrict to your app's bundle identifier (iOS) and package name + SHA-1 (Android) in the Google Cloud Console.
- **Stripe publishable key:** Designed to be public. Only allows creating tokens, not charging cards.

```typescript
// These are OKAY to include in the bundle:
const FIREBASE_API_KEY = 'AIzaSy...'; // Not a secret
const GOOGLE_MAPS_KEY = 'AIzaSy...'; // Restricted to your app
const STRIPE_PUBLISHABLE_KEY = 'pk_live_...'; // Designed to be public

// These are NEVER okay in the bundle:
// const STRIPE_SECRET_KEY = 'sk_live_...';    // SECRET
// const DATABASE_URL = 'postgres://...';       // SECRET
// const JWT_SIGNING_KEY = 'secret123';         // SECRET
// const INTERNAL_API_KEY = 'key-abc123';       // SECRET
```

### 3.3 EAS Environment Variables

EAS Build supports environment variables that are injected at build time but are NOT embedded in the JavaScript bundle (they are available to native code only):

```bash
# Set a secret in EAS (stored encrypted on Expo servers)
eas secret:create --name GOOGLE_MAPS_API_KEY \
  --value "AIzaSy..." \
  --scope project

# Set environment-specific secrets
eas secret:create --name API_BASE_URL \
  --value "https://api.myapp.com" \
  --scope project \
  --type string
```

```typescript
// app.config.ts -- these are available at build time, not runtime JS
const config: ExpoConfig = {
  // ...
  ios: {
    config: {
      googleMapsApiKey: process.env.GOOGLE_MAPS_API_KEY,
    },
  },
  android: {
    config: {
      googleMaps: {
        apiKey: process.env.GOOGLE_MAPS_API_KEY,
      },
    },
  },
};
```

The key distinction: `process.env.GOOGLE_MAPS_API_KEY` in `app.config.ts` is evaluated at **build time** and injected into native configuration files (Info.plist, AndroidManifest.xml), not into the JavaScript bundle. This is better than having the key in JS, though a determined attacker can still extract it from the native binary.

### 3.4 The BFF as API Key Vault

The Backend for Frontend (BFF) pattern is the gold standard for API key protection:

```typescript
// src/api/client.ts
import axios from 'axios';
import { secureStorage } from '../services/secure-storage';

const api = axios.create({
  // Your BFF URL -- the only server your app talks to
  baseURL: 'https://api.myapp.com',
  timeout: 30000,
});

// Attach auth token to every request
api.interceptors.request.use(async (config) => {
  const { accessToken } = await secureStorage.getAuthTokens();
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

export { api };
```

The BFF holds all the secrets:

```
Your BFF Server
├── STRIPE_SECRET_KEY (env var)
├── SENDGRID_API_KEY (env var)
├── OPENAI_API_KEY (env var)
├── DATABASE_URL (env var)
├── JWT_SIGNING_SECRET (env var)
└── ... every other secret

Your Mobile App
├── BFF base URL (public, not a secret)
├── Firebase config (not actually secret)
├── Google Maps key (restricted to your app)
└── ... that is it. No secrets.
```

---

## 4. AUTHENTICATION

### 4.1 OAuth2 PKCE: The Only Correct Flow for Mobile

**Do not use the implicit flow.** It was deprecated by OAuth 2.1 for good reason. The implicit flow sends tokens in URL fragments, which are exposed in browser history, HTTP logs, and can be intercepted by malicious apps that register the same redirect URI scheme.

**Do not use the authorization code flow without PKCE.** On mobile, a malicious app can intercept the authorization code by registering the same custom URL scheme.

**Use the authorization code flow with PKCE (Proof Key for Code Exchange).**

Here is how PKCE works:

```
1. App generates a random "code_verifier" (43-128 chars)
2. App creates a "code_challenge" = SHA256(code_verifier)
3. App opens the auth URL with code_challenge
4. User authenticates in the browser
5. Auth server redirects back with an authorization code
6. App exchanges the code + code_verifier for tokens
7. Auth server verifies SHA256(code_verifier) == code_challenge
8. If match, issues tokens
```

The key insight: even if a malicious app intercepts the authorization code in step 5, it cannot exchange it for tokens because it does not have the `code_verifier` (which was generated in-memory and never transmitted).

### 4.2 Implementing OAuth2 PKCE

```typescript
// src/services/auth.ts
import * as AuthSession from 'expo-auth-session';
import * as Crypto from 'expo-crypto';
import * as WebBrowser from 'expo-web-browser';
import { secureStorage } from './secure-storage';

// Enable web browser dismissal on iOS
WebBrowser.maybeCompleteAuthSession();

const AUTH_CONFIG = {
  clientId: 'your-client-id', // Public client ID (not a secret)
  authorizationEndpoint: 'https://auth.myapp.com/authorize',
  tokenEndpoint: 'https://auth.myapp.com/token',
  revocationEndpoint: 'https://auth.myapp.com/revoke',
  redirectUri: AuthSession.makeRedirectUri({
    scheme: 'myapp',
    path: 'auth/callback',
  }),
  scopes: ['openid', 'profile', 'email', 'offline_access'],
};

interface TokenResponse {
  access_token: string;
  refresh_token: string;
  id_token: string;
  expires_in: number;
  token_type: string;
}

class AuthService {
  private codeVerifier: string | null = null;

  /**
   * Start the OAuth2 PKCE login flow.
   * Opens the system browser for authentication.
   */
  async login(): Promise<{ success: boolean; error?: string }> {
    try {
      // 1. Generate PKCE code verifier and challenge
      this.codeVerifier = this.generateCodeVerifier();
      const codeChallenge = await this.generateCodeChallenge(
        this.codeVerifier
      );

      // 2. Build the authorization URL
      const authUrl = this.buildAuthorizationUrl(codeChallenge);

      // 3. Open the browser for user authentication
      const result = await WebBrowser.openAuthSessionAsync(
        authUrl,
        AUTH_CONFIG.redirectUri
      );

      if (result.type !== 'success' || !result.url) {
        return {
          success: false,
          error: 'Authentication was cancelled',
        };
      }

      // 4. Extract the authorization code from the redirect URL
      const code = this.extractCodeFromUrl(result.url);
      if (!code) {
        return {
          success: false,
          error: 'No authorization code received',
        };
      }

      // 5. Exchange the code + code_verifier for tokens
      const tokens = await this.exchangeCodeForTokens(code);

      // 6. Store tokens securely
      await secureStorage.setAuthTokens({
        accessToken: tokens.access_token,
        refreshToken: tokens.refresh_token,
        userId: this.extractUserIdFromIdToken(tokens.id_token),
      });

      // 7. Store token expiry for proactive refresh
      const expiresAt = Date.now() + tokens.expires_in * 1000;
      await secureStorage.set(
        'auth_access_token',
        JSON.stringify({
          token: tokens.access_token,
          expiresAt,
        })
      );

      return { success: true };
    } catch (error) {
      return {
        success: false,
        error:
          error instanceof Error
            ? error.message
            : 'Authentication failed',
      };
    } finally {
      // Clear the code verifier from memory
      this.codeVerifier = null;
    }
  }

  /**
   * Logout: clear tokens and revoke them server-side.
   */
  async logout(): Promise<void> {
    const { refreshToken } = await secureStorage.getAuthTokens();

    // Revoke the refresh token server-side
    if (refreshToken) {
      try {
        await fetch(AUTH_CONFIG.revocationEndpoint, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
          },
          body: new URLSearchParams({
            token: refreshToken,
            token_type_hint: 'refresh_token',
            client_id: AUTH_CONFIG.clientId,
          }).toString(),
        });
      } catch (error) {
        // Revocation failed -- tokens will expire on their own
        console.warn('Token revocation failed:', error);
      }
    }

    // Clear local tokens
    await secureStorage.clearAuthTokens();
  }

  // --- Private helpers ---

  private generateCodeVerifier(): string {
    // RFC 7636: 43-128 characters from [A-Z] / [a-z] / [0-9] / "-" / "." / "_" / "~"
    const array = new Uint8Array(32);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private async generateCodeChallenge(
    verifier: string
  ): Promise<string> {
    const digest = await Crypto.digestStringAsync(
      Crypto.CryptoDigestAlgorithm.SHA256,
      verifier,
      { encoding: Crypto.CryptoEncoding.BASE64 }
    );
    // Convert standard Base64 to Base64URL
    return digest
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }

  private buildAuthorizationUrl(codeChallenge: string): string {
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: AUTH_CONFIG.clientId,
      redirect_uri: AUTH_CONFIG.redirectUri,
      scope: AUTH_CONFIG.scopes.join(' '),
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      // State parameter for CSRF protection
      state: this.generateRandomState(),
    });

    return `${AUTH_CONFIG.authorizationEndpoint}?${params.toString()}`;
  }

  private extractCodeFromUrl(url: string): string | null {
    const parsed = new URL(url);
    return parsed.searchParams.get('code');
  }

  private async exchangeCodeForTokens(
    code: string
  ): Promise<TokenResponse> {
    if (!this.codeVerifier) {
      throw new Error('Code verifier not found');
    }

    const response = await fetch(AUTH_CONFIG.tokenEndpoint, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        redirect_uri: AUTH_CONFIG.redirectUri,
        client_id: AUTH_CONFIG.clientId,
        code_verifier: this.codeVerifier,
      }).toString(),
    });

    if (!response.ok) {
      const errorBody = await response.text();
      throw new Error(
        `Token exchange failed: ${response.status} ${errorBody}`
      );
    }

    return response.json();
  }

  private extractUserIdFromIdToken(idToken: string): string {
    // Decode the JWT payload (without verification -- server already verified)
    const payload = idToken.split('.')[1];
    const decoded = JSON.parse(atob(payload));
    return decoded.sub;
  }

  private generateRandomState(): string {
    const array = new Uint8Array(16);
    crypto.getRandomValues(array);
    return this.base64UrlEncode(array);
  }

  private base64UrlEncode(buffer: Uint8Array): string {
    const base64 = btoa(String.fromCharCode(...buffer));
    return base64
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }
}

export const authService = new AuthService();
```

### 4.3 Token Management

Access tokens should be short-lived (5-15 minutes). Refresh tokens should be long-lived (days to weeks) and stored in Secure Store. Here is the automatic refresh pattern:

```typescript
// src/api/token-manager.ts
import { secureStorage } from '../services/secure-storage';

interface TokenData {
  token: string;
  expiresAt: number;
}

class TokenManager {
  private refreshPromise: Promise<string> | null = null;

  /**
   * Get a valid access token.
   * Automatically refreshes if expired or about to expire.
   */
  async getValidAccessToken(): Promise<string | null> {
    const { accessToken, refreshToken } =
      await secureStorage.getAuthTokens();

    if (!accessToken || !refreshToken) {
      return null;
    }

    // Check if token is still valid (with 60-second buffer)
    const isExpiringSoon = await this.isTokenExpiringSoon(
      accessToken
    );

    if (!isExpiringSoon) {
      return accessToken;
    }

    // Token is expired or expiring soon -- refresh it
    return this.refreshAccessToken(refreshToken);
  }

  /**
   * Refresh the access token using the refresh token.
   * Uses a mutex to prevent concurrent refresh requests.
   */
  private async refreshAccessToken(
    refreshToken: string
  ): Promise<string> {
    // If a refresh is already in progress, wait for it
    if (this.refreshPromise) {
      return this.refreshPromise;
    }

    this.refreshPromise = this.doRefresh(refreshToken);

    try {
      const newToken = await this.refreshPromise;
      return newToken;
    } finally {
      this.refreshPromise = null;
    }
  }

  private async doRefresh(refreshToken: string): Promise<string> {
    try {
      const response = await fetch(
        'https://auth.myapp.com/token',
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
          },
          body: new URLSearchParams({
            grant_type: 'refresh_token',
            refresh_token: refreshToken,
            client_id: 'your-client-id',
          }).toString(),
        }
      );

      if (!response.ok) {
        // Refresh token is invalid or expired
        // Force logout
        await secureStorage.clearAuthTokens();
        throw new TokenRefreshError('Refresh token expired');
      }

      const data = await response.json();

      // Store the new tokens
      await secureStorage.setAuthTokens({
        accessToken: data.access_token,
        refreshToken: data.refresh_token ?? refreshToken,
        userId:
          (await secureStorage.get('auth_user_id')) ?? '',
      });

      return data.access_token;
    } catch (error) {
      if (error instanceof TokenRefreshError) throw error;
      throw new TokenRefreshError(
        'Failed to refresh token',
        error
      );
    }
  }

  private async isTokenExpiringSoon(
    token: string
  ): Promise<boolean> {
    try {
      // Decode JWT to check expiration
      const payload = token.split('.')[1];
      const decoded = JSON.parse(atob(payload));
      const expiresAt = decoded.exp * 1000; // Convert to ms
      const bufferMs = 60000; // 60-second buffer
      return Date.now() >= expiresAt - bufferMs;
    } catch {
      // If we cannot decode the token, assume it is expired
      return true;
    }
  }
}

class TokenRefreshError extends Error {
  constructor(
    message: string,
    public readonly cause?: unknown
  ) {
    super(message);
    this.name = 'TokenRefreshError';
  }
}

export const tokenManager = new TokenManager();
```

### 4.4 API Client with Automatic Token Refresh

```typescript
// src/api/client.ts
import axios, { AxiosError, InternalAxiosRequestConfig } from 'axios';
import { tokenManager } from './token-manager';
import { secureStorage } from '../services/secure-storage';

const api = axios.create({
  baseURL: 'https://api.myapp.com',
  timeout: 30000,
});

// Request interceptor: attach access token
api.interceptors.request.use(
  async (config: InternalAxiosRequestConfig) => {
    const token = await tokenManager.getValidAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  }
);

// Response interceptor: handle 401 by refreshing token
api.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as InternalAxiosRequestConfig & {
      _retry?: boolean;
    };

    // If we get a 401 and have not already retried
    if (
      error.response?.status === 401 &&
      !originalRequest._retry
    ) {
      originalRequest._retry = true;

      try {
        const { refreshToken } =
          await secureStorage.getAuthTokens();
        if (!refreshToken) {
          // No refresh token -- user needs to log in again
          throw error;
        }

        // The token manager will handle the refresh
        const newToken =
          await tokenManager.getValidAccessToken();
        if (newToken) {
          originalRequest.headers.Authorization =
            `Bearer ${newToken}`;
          return api(originalRequest);
        }
      } catch (refreshError) {
        // Refresh failed -- clear tokens and redirect to login
        await secureStorage.clearAuthTokens();
        // Emit an event or navigate to login
        // eventBus.emit('auth:logout');
        throw refreshError;
      }
    }

    return Promise.reject(error);
  }
);

export { api };
```

### 4.5 Token Storage Security Checklist

```
+----------------------------------+---+-------------------------------+
| Practice                         |   | Why                           |
+----------------------------------+---+-------------------------------+
| Store tokens in Secure Store     | Y | Encrypted by OS               |
| Short-lived access tokens        | Y | Limits exposure window        |
| Refresh tokens with rotation     | Y | Old tokens become invalid     |
| Revoke tokens on logout          | Y | Prevents token reuse          |
| Clear tokens on auth failure     | Y | Forces re-authentication      |
| Never log tokens                 | Y | Logs are often insecure       |
| Never send tokens in URLs        | Y | URLs are logged by servers    |
| Use HTTPS for all token exchange | Y | Prevents interception         |
| Store token expiry separately    | Y | Allows proactive refresh      |
+----------------------------------+---+-------------------------------+
```

---

## 5. CERTIFICATE PINNING

### 5.1 What It Is

Certificate pinning ensures that your app only trusts specific certificates for your server, not any certificate signed by any CA (Certificate Authority). Without pinning, an attacker who compromises a CA -- or who installs a custom CA certificate on the device -- can intercept all HTTPS traffic between your app and your server (a man-in-the-middle attack).

```
Without pinning:
  App trusts ANY certificate signed by ANY CA
  Attacker with a rogue CA cert can intercept traffic

With pinning:
  App trusts ONLY your server's specific certificate
  Rogue CA certs are rejected -> MITM attack fails
```

### 5.2 When Certificate Pinning Matters

**Use certificate pinning when:**
- Your app handles financial transactions
- Your app stores sensitive health or personal data
- You operate in a regulated industry (HIPAA, PCI-DSS, SOX)
- Your app is a high-value target (banking, enterprise)

**Certificate pinning may be overkill when:**
- Your app does not handle sensitive data
- You are an early-stage startup and engineering bandwidth is limited
- You do not have a plan for certificate rotation

### 5.3 Implementation with TrustKit

For React Native, `react-native-ssl-pinning` or native TrustKit integration are the main approaches:

```typescript
// Using react-native-ssl-pinning with fetch replacement
// Install: npx expo install react-native-ssl-pinning

import { fetch as pinnedFetch } from 'react-native-ssl-pinning';

const PINNING_CONFIG = {
  // Your server's certificate public key hashes (SHA-256)
  // Get these by running:
  // openssl s_client -connect api.myapp.com:443 | \
  //   openssl x509 -pubkey -noout | \
  //   openssl pkey -pubin -outform DER | \
  //   openssl dgst -sha256 -binary | base64
  'api.myapp.com': {
    includeSubdomains: true,
    publicKeyHashes: [
      // Current certificate hash
      'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
      // Backup certificate hash (always pin at least 2!)
      'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',
    ],
  },
};

async function secureFetch(
  url: string,
  options?: RequestInit
): Promise<Response> {
  const hostname = new URL(url).hostname;
  const pinConfig = PINNING_CONFIG[hostname];

  if (!pinConfig) {
    // No pinning configured for this host -- use regular fetch
    return fetch(url, options);
  }

  try {
    const response = await pinnedFetch(url, {
      ...options,
      sslPinning: {
        certs: pinConfig.publicKeyHashes,
      },
    });
    return response;
  } catch (error) {
    // Pinning validation failed -- possible MITM attack
    console.error('Certificate pinning validation failed:', error);
    throw new CertificatePinningError(
      'Connection rejected: certificate validation failed'
    );
  }
}

class CertificatePinningError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'CertificatePinningError';
  }
}
```

### 5.4 Native Configuration Approach

For a more robust approach, configure pinning at the native level:

**iOS (Info.plist via Expo config plugin):**

```typescript
// app.config.ts
const config: ExpoConfig = {
  // ...
  ios: {
    infoPlist: {
      // Network Security Configuration
      NSAppTransportSecurity: {
        NSAllowsArbitraryLoads: false, // Enforce HTTPS
        NSPinnedDomains: {
          'api.myapp.com': {
            NSIncludesSubdomains: true,
            NSPinnedLeafIdentities: [
              {
                'SPKI-SHA256-BASE64':
                  'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
              },
              {
                'SPKI-SHA256-BASE64':
                  'BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=',
              },
            ],
          },
        },
      },
    },
  },
};
```

**Android (Network Security Config):**

Create an Expo config plugin that generates the Android network security configuration:

```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">api.myapp.com</domain>
        <pin-set expiration="2027-01-01">
            <!-- Current certificate -->
            <pin digest="SHA-256">
                AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
            </pin>
            <!-- Backup certificate -->
            <pin digest="SHA-256">
                BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=
            </pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

### 5.5 The Maintenance Burden

Certificate pinning has a significant maintenance cost that many teams underestimate:

**Problem: Certificate rotation**

When your server's certificate changes (typically yearly), your pinned certificate hash no longer matches. If you only pinned one certificate, every app that has not updated is now locked out -- it cannot connect to your server at all.

**Mitigation strategies:**

1. **Always pin at least two certificates** -- the current one and the next one (or a backup CA).
2. **Pin the intermediate CA certificate** instead of the leaf certificate. Intermediate CAs change less frequently.
3. **Include certificate hashes in Remote Config** so you can update them without an app update.
4. **Set up monitoring** to alert you well before a certificate expires.
5. **Have a force-update mechanism** (Remote Config kill switch from Chapter 21) so you can push users to update if pinning breaks.

```
+----------------------------------+------+------+
| Pinning Strategy                 | Pros | Cons |
+----------------------------------+------+------+
| Pin leaf certificate             | Most | Must |
|                                  | secure | update on |
|                                  |      | every rotation |
+----------------------------------+------+------+
| Pin intermediate CA              | Less | Slightly |
|                                  | maintenance | less secure |
+----------------------------------+------+------+
| Pin multiple (current + backup)  | Best | More complex |
|                                  | balance | setup |
+----------------------------------+------+------+
| Dynamic pins via Remote Config   | Most | Complex, |
|                                  | flexible | bootstrap issue |
+----------------------------------+------+------+
```

---

## 6. OWASP MOBILE TOP 10

The OWASP Mobile Top 10 is the industry standard list of the most critical security risks in mobile applications. Here is each one with React Native-specific context and mitigations.

### M1: Improper Credential Usage

**What it is:** Using hardcoded credentials, insecure credential storage, or transmitting credentials insecurely.

**React Native context:**

```typescript
// VULNERABLE: Hardcoded API key in JavaScript
const API_KEY = 'sk-prod-abc123xyz';

// VULNERABLE: Credentials in AsyncStorage
await AsyncStorage.setItem('auth_token', token);

// VULNERABLE: Logging tokens
console.log('Token:', accessToken);
```

**Mitigation:**

```typescript
// SECURE: Server-side proxy for secret keys
// (see Section 3 of this chapter)

// SECURE: Tokens in Secure Store
await SecureStore.setItemAsync('auth_token', token);

// SECURE: Never log sensitive data
if (__DEV__) {
  console.log('Token exists:', !!accessToken);
}
```

### M2: Inadequate Supply Chain Security

**What it is:** Using vulnerable or malicious third-party dependencies.

**React Native context:** React Native apps typically have 500-1500+ npm dependencies. Any one of them could be compromised.

**Mitigation:**

```bash
# Audit dependencies regularly
npm audit
npx expo-doctor

# Use lockfiles and verify integrity
# package-lock.json / yarn.lock should be committed

# Pin exact versions for security-critical packages
# In package.json:
# "@react-native-firebase/app": "20.4.0"  (exact, not ^20.4.0)
```

```json
// .npmrc -- enforce strict dependency resolution
engine-strict=true
```

### M3: Insecure Authentication/Authorization

**What it is:** Weak authentication mechanisms, missing authorization checks, or bypassable auth.

**React Native context:**

```typescript
// VULNERABLE: Client-side only role check
if (user.role === 'admin') {
  showAdminPanel();
}
// An attacker can modify the user object in memory

// VULNERABLE: Sending user ID in request
// (attacker can change it to another user's ID)
api.get(`/users/${userId}/private-data`);
```

**Mitigation:**

```typescript
// SECURE: Server validates authorization on every request
// The server extracts user ID from the JWT, not from the request body
api.get('/users/me/private-data');
// Server: const userId = jwt.verify(token).sub;

// Client-side role checks are for UI only, never for security
// The server must enforce all authorization rules
```

### M4: Insufficient Input/Output Validation

**What it is:** Accepting unvalidated input or rendering unsanitized output.

**React Native context:**

```typescript
// VULNERABLE: Rendering HTML from user input
<WebView source={{ html: userProvidedContent }} />
// XSS via injected script tags

// VULNERABLE: Using user input in deep links
Linking.openURL(userProvidedUrl);
// Could open malicious URLs or trigger unintended actions

// VULNERABLE: SQL injection via raw queries (if using local DB)
db.executeSql(`SELECT * FROM users WHERE name = '${userInput}'`);
```

**Mitigation:**

```typescript
// SECURE: Validate and sanitize all inputs
import { z } from 'zod';

const UserInputSchema = z.object({
  name: z.string().min(1).max(100).trim(),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
});

function handleUserInput(raw: unknown) {
  const result = UserInputSchema.safeParse(raw);
  if (!result.success) {
    throw new ValidationError(result.error);
  }
  return result.data; // Validated and typed
}

// SECURE: Validate URLs before opening
function safeOpenUrl(url: string): void {
  const parsed = new URL(url);
  const allowedHosts = ['myapp.com', 'www.myapp.com'];
  if (!allowedHosts.includes(parsed.hostname)) {
    throw new Error('URL not allowed');
  }
  Linking.openURL(url);
}

// SECURE: Parameterized queries
db.executeSql(
  'SELECT * FROM users WHERE name = ?',
  [userInput]
);
```

### M5: Insecure Communication

**What it is:** Transmitting data over unencrypted channels, or failing to validate certificates.

**React Native context:**

```typescript
// VULNERABLE: HTTP instead of HTTPS
const API_URL = 'http://api.myapp.com'; // No encryption!

// VULNERABLE: Disabling SSL verification in development
// and forgetting to re-enable in production
```

**Mitigation:**

```typescript
// SECURE: Always HTTPS
const API_URL = 'https://api.myapp.com';

// SECURE: Certificate pinning (see Section 5)

// SECURE: iOS App Transport Security (enabled by default in Expo)
// Android: network_security_config.xml to prevent cleartext traffic
```

### M6: Inadequate Privacy Controls

**What it is:** Collecting, storing, or transmitting more data than necessary, or failing to handle data deletion requests.

**React Native context:**

```typescript
// VULNERABLE: Logging PII
console.log('User:', JSON.stringify(user));
// This might end up in crash logs, analytics, or system logs

// VULNERABLE: Caching sensitive data unnecessarily
mmkvStorage.set('user_ssn', user.socialSecurityNumber);
```

**Mitigation:**

```typescript
// SECURE: Scrub PII from logs
function sanitizeForLogging(user: User): Record<string, unknown> {
  return {
    id: user.id,
    role: user.role,
    // Omit: email, phone, name, SSN, etc.
  };
}

// SECURE: Only store what you need
// Do not cache sensitive data locally
// Implement data deletion when user requests it
```

### M7: Insufficient Binary Protections

**What it is:** Lack of code obfuscation, tampering detection, or reverse engineering protections.

**Mitigation:** See Section 7 (Code Obfuscation) of this chapter.

### M8: Security Misconfiguration

**What it is:** Insecure default settings, unnecessary permissions, or debug features left enabled.

**React Native context:**

```typescript
// VULNERABLE: Debug menu accessible in production
if (__DEV__) {
  // This is fine -- only available in dev
}

// VULNERABLE: Excessive permissions in app.json
{
  "android": {
    "permissions": [
      "CAMERA",
      "RECORD_AUDIO",
      "READ_CONTACTS",    // Do you actually need this?
      "ACCESS_FINE_LOCATION", // Do you actually need this?
    ]
  }
}
```

**Mitigation:**

```typescript
// SECURE: Request only necessary permissions
{
  "android": {
    "permissions": [
      "CAMERA"  // Only what you actually use
    ]
  }
}

// SECURE: Strip debug code from production builds
// Expo and React Native do this automatically with __DEV__
// But verify your custom debug tools are also stripped
```

### M9: Insecure Data Storage

**What it is:** Storing sensitive data in insecure locations (files, databases, preferences, logs, clipboard).

**Mitigation:** See Section 2 (Secure Storage) of this chapter.

### M10: Insufficient Cryptography

**What it is:** Using weak algorithms, hardcoded keys, or improper implementations.

```typescript
// VULNERABLE: Hardcoded encryption key
const ENCRYPTION_KEY = 'my-secret-key-123';

// VULNERABLE: Using MD5 or SHA1 for security purposes
const hash = md5(password);

// VULNERABLE: Rolling your own crypto
function myEncrypt(data: string, key: string): string {
  // DO NOT DO THIS
}
```

**Mitigation:**

```typescript
// SECURE: Use platform-provided crypto
import * as Crypto from 'expo-crypto';

// SECURE: Use strong algorithms
const hash = await Crypto.digestStringAsync(
  Crypto.CryptoDigestAlgorithm.SHA256,
  data
);

// SECURE: Key stored in Keychain/Keystore (see Section 2)
// SECURE: Use established crypto libraries, never roll your own
```

---

## 7. CODE OBFUSCATION

### 7.1 The Reality of Obfuscation

Let me set expectations: obfuscation makes reverse engineering harder, not impossible. A determined attacker with enough time and skill will eventually understand your code. But obfuscation raises the cost, which is often enough to deter casual attackers and script kiddies.

### 7.2 Hermes Bytecode: Implicit Obfuscation

If you are using React Native with Hermes (which you should be -- see Chapter 1), your JavaScript is already compiled to Hermes bytecode at build time. This is not plain JavaScript -- it is a binary format that requires specialized tools to decompile.

```bash
# Without Hermes: your bundle is readable JavaScript
cat index.android.bundle
# function authenticateUser(e,t){var n=apiKey+"secret"...

# With Hermes: your bundle is bytecode
hexdump -C index.android.bundle | head
# 1f 3f c2 00  48 65 72 6d  65 73 ...
```

Hermes bytecode is not encryption -- tools like `hbcdump` and `hermes-dec` can decompile it. But it is significantly harder to read than plain JavaScript, and the decompiled output loses variable names, comments, and structure.

**This is free obfuscation -- you get it just by using Hermes.** And you should be using Hermes for performance reasons anyway.

### 7.3 ProGuard / R8 for Android

ProGuard (now replaced by R8, which is backwards-compatible with ProGuard rules) shrinks, optimizes, and obfuscates your Android native code. For React Native, this affects the Java/Kotlin layer.

```groovy
// android/app/build.gradle
// (Managed by Expo config plugins, but here's what it does)
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                'proguard-rules.pro'
        }
    }
}
```

**Key ProGuard rules for React Native:**

```
# proguard-rules.pro

# Keep React Native bridge classes
-keep class com.facebook.react.** { *; }
-keep class com.facebook.hermes.** { *; }

# Keep native modules (your custom ones)
-keep class com.myapp.nativemodules.** { *; }

# Keep classes referenced via reflection
-keepclassmembers class * {
    @com.facebook.react.uimanager.annotations.ReactProp <methods>;
}

# Keep Firebase Crashlytics for proper stack traces
-keepattributes SourceFile,LineNumberTable
-keep public class * extends java.lang.Exception
```

Expo handles most of this automatically. In a managed Expo project, ProGuard/R8 is enabled for release builds by default with sensible rules.

### 7.4 iOS Bitcode and Compilation

On iOS, your app is compiled to native ARM machine code. The original Objective-C/Swift source is not included in the binary. This provides a base level of obfuscation for the native layer.

**Additional iOS protections:**
- App Store binaries are encrypted (FairPlay DRM). They are decrypted at runtime on the device.
- Swift code includes less metadata than Objective-C, making it harder to reverse engineer.
- `strip` removes symbol information from the binary.

### 7.5 Additional Obfuscation with react-native-obfuscating-transformer

For the JavaScript layer (beyond Hermes bytecode), you can use Metro bundler plugins to obfuscate your code:

```bash
npm install --save-dev react-native-obfuscating-transformer
```

```javascript
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const obfuscatingTransformer = require('react-native-obfuscating-transformer');

const config = getDefaultConfig(__dirname);

// Only obfuscate in production builds
if (process.env.NODE_ENV === 'production') {
  config.transformer.babelTransformerPath =
    require.resolve('react-native-obfuscating-transformer');

  config.transformer.obfuscatingTransformerOptions = {
    // Rename variables and functions
    compact: true,
    // Control flow flattening (makes logic harder to follow)
    controlFlowFlattening: true,
    controlFlowFlatteningThreshold: 0.75,
    // Dead code injection
    deadCodeInjection: true,
    deadCodeInjectionThreshold: 0.4,
    // String array encoding
    stringArray: true,
    stringArrayEncoding: ['base64'],
    stringArrayThreshold: 0.75,
    // Files to exclude from obfuscation
    // (obfuscation can sometimes break things)
    excludeNodeModules: true,
  };
}

module.exports = config;
```

> **Warning:** Heavy obfuscation can impact performance (larger bundle, slower execution due to control flow flattening) and make debugging production issues much harder. Use it judiciously and test thoroughly.

### 7.6 The Obfuscation Trade-Off Table

```
+------------------------------+----------+------------+-----------+
| Technique                    | Security | Perf Cost  | Effort    |
+------------------------------+----------+------------+-----------+
| Hermes bytecode              | Medium   | None       | None      |
| (already using it)           |          | (faster)   | (default) |
+------------------------------+----------+------------+-----------+
| ProGuard/R8 (Android)        | Medium   | None       | Low       |
|                               |          | (smaller)  | (default) |
+------------------------------+----------+------------+-----------+
| iOS compilation              | Medium   | None       | None      |
| (already compiled)           |          |            | (default) |
+------------------------------+----------+------------+-----------+
| JS obfuscation transformer   | Med-High | Medium     | Medium    |
|                               |          | (5-15%     |           |
|                               |          |  bundle)   |           |
+------------------------------+----------+------------+-----------+
| Server-side business logic   | High     | Network    | High      |
| (move logic to backend)     |          | latency    |           |
+------------------------------+----------+------------+-----------+
```

**My recommendation:** Use Hermes (you already are), enable ProGuard/R8 for release builds (Expo does this by default), and move any truly sensitive business logic to your server. Only add JS obfuscation if you are in a high-security domain (banking, healthcare, enterprise).

---

## 8. BIOMETRIC AUTHENTICATION

### 8.1 When to Use Biometrics

Biometric authentication (Face ID, Touch ID, fingerprint) provides a convenient way for users to verify their identity without typing a password. But it is important to understand what biometrics are and are not:

**Biometrics are:** A convenient way to unlock something that is already secured (like a token stored in the Keychain).

**Biometrics are not:** A replacement for proper authentication. You cannot send a "face scan" to your server. Biometrics are a local-only verification that the person holding the device is authorized to access locally stored credentials.

### 8.2 The Pattern

The standard pattern is:
1. User logs in with username/password (or OAuth2 PKCE)
2. Store the auth token in Keychain with biometric protection
3. On subsequent app opens, prompt for biometrics
4. If biometric check passes, retrieve the token from Keychain
5. Use the token to make authenticated API calls

```typescript
// src/services/biometric-auth.ts
import * as LocalAuthentication from 'expo-local-authentication';
import { Platform } from 'react-native';
import { keychainService } from './keychain';
import { secureStorage } from './secure-storage';

interface BiometricCapabilities {
  isAvailable: boolean;
  biometryType: 'fingerprint' | 'facial' | 'iris' | null;
  isEnrolled: boolean;
}

class BiometricAuthService {
  /**
   * Check if biometric authentication is available and enrolled.
   */
  async getCapabilities(): Promise<BiometricCapabilities> {
    const isAvailable =
      await LocalAuthentication.hasHardwareAsync();
    const isEnrolled =
      await LocalAuthentication.isEnrolledAsync();
    const supportedTypes =
      await LocalAuthentication.supportedAuthenticationTypesAsync();

    let biometryType: BiometricCapabilities['biometryType'] = null;

    if (
      supportedTypes.includes(
        LocalAuthentication.AuthenticationType.FACIAL_RECOGNITION
      )
    ) {
      biometryType = 'facial';
    } else if (
      supportedTypes.includes(
        LocalAuthentication.AuthenticationType.FINGERPRINT
      )
    ) {
      biometryType = 'fingerprint';
    } else if (
      supportedTypes.includes(
        LocalAuthentication.AuthenticationType.IRIS
      )
    ) {
      biometryType = 'iris';
    }

    return { isAvailable, biometryType, isEnrolled };
  }

  /**
   * Get a human-readable name for the biometric type.
   */
  getBiometryDisplayName(
    biometryType: BiometricCapabilities['biometryType']
  ): string {
    switch (biometryType) {
      case 'facial':
        return Platform.OS === 'ios' ? 'Face ID' : 'Face Unlock';
      case 'fingerprint':
        return Platform.OS === 'ios'
          ? 'Touch ID'
          : 'Fingerprint';
      case 'iris':
        return 'Iris Scan';
      default:
        return 'Biometric Authentication';
    }
  }

  /**
   * Prompt the user for biometric authentication.
   * Returns true if authentication succeeded.
   */
  async authenticate(
    reason?: string
  ): Promise<{ success: boolean; error?: string }> {
    try {
      const result = await LocalAuthentication.authenticateAsync({
        promptMessage:
          reason ?? 'Verify your identity to continue',
        cancelLabel: 'Cancel',
        disableDeviceFallback: false, // Allow PIN/password fallback
        fallbackLabel: 'Use Passcode',
      });

      if (result.success) {
        return { success: true };
      }

      return {
        success: false,
        error: result.error ?? 'Authentication failed',
      };
    } catch (error) {
      return {
        success: false,
        error:
          error instanceof Error
            ? error.message
            : 'Biometric authentication error',
      };
    }
  }

  /**
   * Enable biometric login for the current user.
   * Stores the refresh token with biometric protection.
   */
  async enableBiometricLogin(): Promise<boolean> {
    const capabilities = await this.getCapabilities();
    if (!capabilities.isAvailable || !capabilities.isEnrolled) {
      return false;
    }

    // Get the current refresh token
    const { refreshToken, userId } =
      await secureStorage.getAuthTokens();
    if (!refreshToken || !userId) {
      return false;
    }

    // Store in Keychain with biometric protection
    const stored =
      await keychainService.setBiometricProtectedCredentials(
        userId,
        refreshToken
      );

    return stored;
  }

  /**
   * Attempt biometric login.
   * Retrieves the stored refresh token after biometric verification.
   */
  async biometricLogin(): Promise<{
    success: boolean;
    refreshToken?: string;
    userId?: string;
    error?: string;
  }> {
    try {
      // This will trigger the biometric prompt
      const credentials =
        await keychainService.getBiometricProtectedCredentials();

      if (!credentials) {
        return {
          success: false,
          error: 'No biometric credentials found',
        };
      }

      return {
        success: true,
        refreshToken: credentials.password,
        userId: credentials.username,
      };
    } catch (error) {
      return {
        success: false,
        error:
          error instanceof Error
            ? error.message
            : 'Biometric login failed',
      };
    }
  }

  /**
   * Disable biometric login.
   */
  async disableBiometricLogin(): Promise<void> {
    await keychainService.resetCredentials();
  }
}

export const biometricAuth = new BiometricAuthService();
```

### 8.3 The Biometric Login Flow

```typescript
// src/screens/LoginScreen.tsx
import { useEffect, useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { biometricAuth } from '../services/biometric-auth';
import { tokenManager } from '../api/token-manager';
import { secureStorage } from '../services/secure-storage';

function LoginScreen({ navigation }) {
  const [biometricAvailable, setBiometricAvailable] =
    useState(false);
  const [biometricType, setBiometricType] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    checkBiometrics();
  }, []);

  async function checkBiometrics() {
    const capabilities =
      await biometricAuth.getCapabilities();
    setBiometricAvailable(
      capabilities.isAvailable && capabilities.isEnrolled
    );
    setBiometricType(
      biometricAuth.getBiometryDisplayName(
        capabilities.biometryType
      )
    );

    // Auto-prompt for biometric login if available
    if (capabilities.isAvailable && capabilities.isEnrolled) {
      handleBiometricLogin();
    }
  }

  async function handleBiometricLogin() {
    setIsLoading(true);
    try {
      const result = await biometricAuth.biometricLogin();

      if (result.success && result.refreshToken) {
        // Use the refresh token to get a fresh access token
        await secureStorage.setAuthTokens({
          accessToken: '', // Will be refreshed
          refreshToken: result.refreshToken,
          userId: result.userId ?? '',
        });

        const token =
          await tokenManager.getValidAccessToken();
        if (token) {
          navigation.replace('Home');
        }
      }
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <View style={styles.container}>
      {/* Regular login form */}
      <LoginForm onSuccess={() => navigation.replace('Home')} />

      {/* Biometric login button */}
      {biometricAvailable && (
        <TouchableOpacity
          style={styles.biometricButton}
          onPress={handleBiometricLogin}
          disabled={isLoading}
        >
          <Text style={styles.biometricText}>
            Login with {biometricType}
          </Text>
        </TouchableOpacity>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    padding: 24,
  },
  biometricButton: {
    marginTop: 16,
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 12,
    alignItems: 'center',
  },
  biometricText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 8.4 Biometrics vs PIN

| Scenario | Use Biometrics | Use PIN |
|----------|---------------|---------|
| Unlocking the app | Yes | Fallback |
| Confirming a purchase | Yes | Fallback |
| Viewing sensitive data | Yes | Fallback |
| Changing security settings | No | Yes (+ current password) |
| Initial login | No | No (use full auth) |
| Device without biometrics | N/A | Yes |

**Always provide a PIN/password fallback.** Not all users can use biometrics (accessibility, hardware limitations), and biometric enrollment can change (new face registered, fingerprint removed).

### 8.5 Biometric Gotchas

1. **Biometric enrollment changes.** If a user adds a new fingerprint or re-enrolls Face ID, the Keychain item protected by `BIOMETRY_CURRENT_SET` becomes inaccessible. You need to handle this gracefully (fall back to password login).

2. **Background authentication.** On iOS, if your app goes to background and returns, the system may prompt for biometrics again. Handle the `AppState` change event appropriately.

3. **Simulator limitations.** Face ID/Touch ID can be simulated in Xcode Simulator (Hardware menu), but the behavior is not identical to a real device. Always test on physical hardware.

4. **Privacy.** Biometric data never leaves the device. You never see or store the user's face or fingerprint. You are only asking the OS "is this the device owner?" and getting a yes/no answer.

---

## 9. JAILBREAK / ROOT DETECTION

### 9.1 Why It Matters

Jailbroken (iOS) and rooted (Android) devices have elevated privileges that bypass OS security mechanisms. On a jailbroken device:

- The Keychain can be dumped
- SSL pinning can be bypassed with tools like Frida or SSL Kill Switch
- The app's sandbox is not enforced
- Dynamic instrumentation tools can modify app behavior at runtime
- File system protections are weakened

For most consumer apps, jailbreak detection is not worth the effort and false positives it causes. But for high-security apps (banking, healthcare, enterprise), it is a compliance requirement.

### 9.2 When to Implement It

**Implement jailbreak/root detection when:**
- Your app handles financial transactions
- Compliance requires it (PCI-DSS, HIPAA, SOC 2)
- Your app manages DRM-protected content
- Your app is an enterprise MDM-managed app

**Do not implement it when:**
- You are building a consumer app without sensitive data
- You prioritize user experience over marginal security gains
- You do not have the engineering bandwidth to maintain it

### 9.3 Implementation

```typescript
// src/services/device-security.ts
import { Platform } from 'react-native';

interface SecurityCheckResult {
  isCompromised: boolean;
  checks: {
    jailbroken: boolean;
    debugger: boolean;
    emulator: boolean;
    hookingFramework: boolean;
  };
  details: string[];
}

class DeviceSecurityService {
  /**
   * Run all security checks.
   * Returns a comprehensive result.
   */
  async checkDeviceSecurity(): Promise<SecurityCheckResult> {
    const details: string[] = [];
    let isCompromised = false;

    // Use a native library for reliable detection
    // Options: jail-monkey, react-native-device-integrity
    try {
      const JailMonkey =
        require('jail-monkey').default;

      const jailbroken = JailMonkey.isJailBroken();
      const debugged = JailMonkey.isDebuggedMode();
      const emulator =
        Platform.OS === 'android'
          ? JailMonkey.isOnExternalStorage()
          : false;

      if (jailbroken) {
        details.push('Device appears to be jailbroken/rooted');
        isCompromised = true;
      }
      if (debugged) {
        details.push('Debugger detected');
        isCompromised = true;
      }
      if (emulator) {
        details.push('Running on emulator/external storage');
      }

      return {
        isCompromised,
        checks: {
          jailbroken,
          debugger: debugged,
          emulator,
          hookingFramework: false,
        },
        details,
      };
    } catch (error) {
      // Library not installed -- fall back to basic checks
      return this.basicSecurityCheck();
    }
  }

  /**
   * Basic security checks without a native library.
   * Less reliable but better than nothing.
   */
  private async basicSecurityCheck(): Promise<SecurityCheckResult> {
    const details: string[] = [];
    let isCompromised = false;

    if (__DEV__) {
      details.push('Running in development mode');
    }

    return {
      isCompromised,
      checks: {
        jailbroken: false,
        debugger: __DEV__,
        emulator: false,
        hookingFramework: false,
      },
      details,
    };
  }
}

export const deviceSecurity = new DeviceSecurityService();
```

### 9.4 What to Do When a Compromised Device Is Detected

This is the policy decision. You have several options:

```typescript
// Option 1: Block the app entirely (strictest)
async function enforceDeviceSecurity(): Promise<void> {
  const result = await deviceSecurity.checkDeviceSecurity();
  if (result.isCompromised) {
    // Show a blocking screen
    // The app cannot be used on this device
    showSecurityBlockScreen(result.details);
  }
}

// Option 2: Limit functionality (balanced)
async function adjustForDeviceSecurity(): Promise<void> {
  const result = await deviceSecurity.checkDeviceSecurity();
  if (result.isCompromised) {
    // Disable sensitive features
    featureFlags.disablePayments = true;
    featureFlags.disableBiometrics = true;

    // Warn the user
    showSecurityWarning(
      'Some features are disabled for security reasons.'
    );

    // Report to your server
    api.post('/security/device-compromised', {
      checks: result.checks,
      details: result.details,
    });
  }
}

// Option 3: Report only (least disruptive)
async function reportDeviceSecurity(): Promise<void> {
  const result = await deviceSecurity.checkDeviceSecurity();
  if (result.isCompromised) {
    // Log to Crashlytics / analytics
    crashlyticsService.setCustomKeys({
      device_compromised: true,
      jailbroken: result.checks.jailbroken,
    });

    // Report to your server for monitoring
    api.post('/security/device-report', {
      checks: result.checks,
    });
  }
}
```

### 9.5 The Cat-and-Mouse Game

Here is the honest truth about jailbreak/root detection: **it is an arms race you will never win.** For every detection method, there is a bypass. Jailbreak detection libraries check for common indicators (Cydia installed, certain file paths exist, certain processes running), and jailbreak tools respond by hiding those indicators.

```
Detection: Check if /Applications/Cydia.app exists
Bypass: JailbreakBypassTool hides the Cydia app

Detection: Check if the app can write outside its sandbox
Bypass: Hooking tool intercepts the write and returns failure

Detection: Check for Frida or Cycript processes
Bypass: Process name is randomized

Detection: Use attestation APIs (DeviceCheck, Play Integrity)
Bypass: Harder, but possible with modified attestation responses
```

**The pragmatic approach:**
1. Use a detection library as a first layer
2. Combine with server-side device attestation (Apple DeviceCheck, Google Play Integrity API) for higher assurance
3. Accept that sophisticated attackers will bypass your checks
4. Focus your energy on server-side protections (input validation, authorization, rate limiting) that cannot be bypassed from the client

### 9.6 Google Play Integrity and Apple DeviceCheck

For the strongest device integrity verification, use platform attestation APIs:

```typescript
// src/services/device-attestation.ts
import { Platform } from 'react-native';

class DeviceAttestationService {
  /**
   * Get a device integrity token.
   * This token is verified by YOUR SERVER against
   * Google/Apple APIs, not on the device.
   */
  async getIntegrityToken(): Promise<string | null> {
    if (Platform.OS === 'android') {
      return this.getPlayIntegrityToken();
    }
    if (Platform.OS === 'ios') {
      return this.getDeviceCheckToken();
    }
    return null;
  }

  private async getPlayIntegrityToken(): Promise<string | null> {
    try {
      // Use @react-native-google-play-integrity
      // or a native module wrapping PlayIntegrity API
      const PlayIntegrity =
        require('react-native-play-integrity').default;
      const token = await PlayIntegrity.requestIntegrityToken(
        'your-cloud-project-number'
      );
      return token;
    } catch (error) {
      console.warn('Play Integrity check failed:', error);
      return null;
    }
  }

  private async getDeviceCheckToken(): Promise<string | null> {
    try {
      // Use a native module wrapping DCDevice
      const DeviceCheck =
        require('react-native-device-check').default;
      const token = await DeviceCheck.generateToken();
      return token;
    } catch (error) {
      console.warn('DeviceCheck failed:', error);
      return null;
    }
  }
}

export const deviceAttestation = new DeviceAttestationService();
```

The token is sent to your server, which verifies it against Google's/Apple's servers. This is much harder to fake than client-side checks because the attestation involves hardware-backed security (Secure Enclave, StrongBox) and is verified server-side.

---

## 10. SECURITY AUDIT CHECKLIST

Use this checklist before every release:

```
STORAGE
[ ] Auth tokens stored in Secure Store / Keychain
[ ] No sensitive data in AsyncStorage
[ ] No sensitive data in MMKV (unless encrypted)
[ ] No PII logged to console or crash reports

API KEYS
[ ] No secret keys in JavaScript bundle
[ ] Client-side keys are restricted (bundle ID, SHA)
[ ] Server-side proxy for third-party APIs with secrets
[ ] EXPO_PUBLIC_ variables contain no secrets

AUTHENTICATION
[ ] OAuth2 PKCE flow (not implicit, not plain auth code)
[ ] Short-lived access tokens (5-15 min)
[ ] Refresh token rotation enabled
[ ] Token revocation on logout
[ ] Auto-refresh with interceptors
[ ] Proper 401 handling (clear tokens, redirect to login)

NETWORK
[ ] HTTPS everywhere (no HTTP endpoints)
[ ] Certificate pinning (if applicable)
[ ] API responses validated (Zod schemas)
[ ] No sensitive data in URL parameters

CODE PROTECTION
[ ] Hermes enabled (bytecode, not plain JS)
[ ] ProGuard/R8 enabled for Android release builds
[ ] No debug tools in production builds
[ ] __DEV__ guards on all debug code

PERMISSIONS
[ ] Only necessary permissions requested
[ ] Permissions requested at point of use, not at startup
[ ] Graceful handling of permission denial

DEVICE
[ ] Jailbreak/root detection (if required by compliance)
[ ] Device attestation (if high-security)
[ ] Biometric authentication available with PIN fallback

DEPENDENCIES
[ ] npm audit clean (no critical/high vulnerabilities)
[ ] Dependencies updated regularly
[ ] Lock file committed
[ ] No unnecessary dependencies
```

---

## 11. DEFENSE IN DEPTH: THE LAYERED APPROACH

Security is not a single feature. It is a series of layers, each making it harder for an attacker to reach the sensitive data:

```
Layer 1: Network Security
   HTTPS + Certificate Pinning
   |
   v
Layer 2: Authentication
   OAuth2 PKCE + Short-lived tokens
   |
   v
Layer 3: Authorization
   Server-side role checks + Scoped permissions
   |
   v
Layer 4: Data Protection
   Secure Store + Encrypted storage
   |
   v
Layer 5: Code Protection
   Hermes bytecode + ProGuard + Server-side logic
   |
   v
Layer 6: Device Integrity
   Jailbreak detection + Platform attestation
   |
   v
Layer 7: Monitoring
   Crashlytics + Analytics + Security event logging
```

Each layer is imperfect. Certificate pinning can be bypassed. Tokens can be stolen. Jailbreak detection can be evaded. But an attacker who needs to bypass ALL seven layers is facing a much harder challenge than one who only needs to bypass one.

The key insight: **you are not building an impenetrable fortress. You are building a series of locked doors, each requiring a different key.** Most attackers give up after the second or third door.

---

## CHAPTER SUMMARY

Mobile security is fundamentally different from web security because the client is an untrusted, decompilable, inspectable environment. Everything in your app binary is public, and every client-side protection can be bypassed given enough effort.

The response to this is not despair -- it is defense in depth:

1. **Secure Storage** -- tokens in Keychain/Keystore, never in AsyncStorage or plain MMKV.
2. **API Key Protection** -- no secrets in the bundle, ever. Use the BFF pattern.
3. **Proper Authentication** -- OAuth2 PKCE, short-lived tokens, automatic refresh, token revocation.
4. **Certificate Pinning** -- for high-security apps, with a rotation plan.
5. **OWASP Top 10 Awareness** -- validate inputs, enforce authorization server-side, minimize data collection.
6. **Code Obfuscation** -- Hermes bytecode, ProGuard, and server-side business logic for the crown jewels.
7. **Biometric Authentication** -- convenient re-authentication with proper fallbacks.
8. **Device Integrity** -- jailbreak detection and platform attestation when compliance requires it.

The teams that do security well do not try to make their app unhackable. They make it expensive to hack, they minimize the blast radius of a successful attack, they monitor for suspicious activity, and they have incident response plans for when things go wrong. Security is not a feature you ship once -- it is a practice you maintain.

### What's Next

The next part of the book moves into **Vercel & Web** -- taking the knowledge we have built around React Native and applying it to web deployment, server components, and the full-stack React ecosystem. The security principles from this chapter -- especially around authentication, token management, and API key protection -- apply directly to web applications as well.
