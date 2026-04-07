<!--
  CHAPTER: 18
  TITLE: Testing Strategy — What to Test, How to Test, When to Stop
  PART: IV — Architecture at Scale
  PREREQS: Chapters 5, 9
  KEY_TOPICS: testing trophy, Jest, RNTL, Maestro, Playwright, MSW, testing hooks, testing navigation, snapshot testing, visual regression, performance testing
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 17: Testing Strategy — What to Test, How to Test, When to Stop

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 5, 9 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- The testing trophy replaces the pyramid: invest most in integration tests (RNTL, MSW), fewer unit tests, fewer E2E tests, and almost zero snapshot tests
- React Native Testing Library tests behavior (what the user sees and does), not implementation details; query by text, role, or test ID -- never by component internals
- Mock Service Worker (MSW) intercepts network requests at the handler level, giving you realistic API mocking without stubbing `fetch` or Axios
- Maestro is winning mobile E2E because of YAML-based flows, built-in waiting, and reliable CI execution; use it for critical user journeys (login, checkout, onboarding)
- Stop testing third-party library internals, stop maintaining brittle snapshots, and stop chasing 100% coverage; test the user flows that would wake you up at 3 AM if they broke

</details>

Here's the dirty secret about testing in frontend: **most teams either test too little or test the wrong things.** The team with zero tests ships bugs to production and puts out fires every sprint. The team with 3,000 snapshot tests has a green CI badge and *still* ships bugs to production — they just also spend 45 minutes waiting for tests to run and another hour updating snapshots every time someone changes a font size.

The difference between a codebase with useful tests and a codebase with test theater is not the number of tests. It's not the coverage percentage. It's *what* gets tested, *how* it gets tested, and — critically — *when someone decides to stop writing tests and ship the damn thing.*

I've seen teams with 95% code coverage that couldn't catch a broken login flow because all their tests were unit tests on individual functions that never ran together. I've seen teams with 40% coverage and a rock-solid app because every test exercised a real user workflow end-to-end. The second team shipped faster, broke less, and slept better at night.

This chapter gives you the testing strategy that actually works for modern React Native and Next.js apps. Not the textbook answer — the battle-tested answer. We'll cover the tools, the patterns, the configuration, and most importantly, the judgment calls about what deserves a test and what doesn't.

### In This Chapter
- The Testing Trophy — why integration-heavy testing replaced the pyramid
- Jest Configuration for React Native — setup, transforms, mocking native modules
- React Native Testing Library — testing behavior, not implementation
- Mock Service Worker (MSW) — intercepting network requests without mocking fetch
- Testing TanStack Query — QueryClient wrappers, hooks, optimistic updates, cache
- Testing Zustand Stores — independent store logic and connected components
- Maestro for Mobile E2E — why it's winning, YAML flows, CI integration
- Playwright for Web E2E — Next.js setup, page objects, auth flows, screenshots
- What NOT to Test — snapshot tests, implementation details, third-party internals
- Visual Regression — Chromatic, Percy, Storybook, and when it's worth the cost
- Performance Testing — Flashlight, Lighthouse CI, benchmarking renders

### Related Chapters
- [Ch 5: Expo Platform] — Expo-specific testing considerations
- [Ch 9: State Management] — the stores and queries we're testing here
- [Ch 16: Design Systems] — Storybook as a testing and documentation surface
- [Ch 18: CI/CD] — running these tests in your pipeline

---

## 1. THE TESTING TROPHY

Forget the testing pyramid. If you learned testing from a textbook or a 2015 conference talk, you were taught the pyramid: lots of unit tests at the base, fewer integration tests in the middle, even fewer E2E tests at the top. The idea was that unit tests are fast and cheap, so write tons of them. E2E tests are slow and flaky, so write few.

**That model is wrong for frontend development.** Here's why.

### 1.1 The Pyramid's Problem

A unit test for a React component tests the component in isolation — no router, no state management, no data fetching, no other components. You render `<Button onPress={mockFn} />`, simulate a press, and assert the mock was called. Great, your Button works.

But does your *screen* work? Does the button actually trigger a mutation that invalidates the right query and navigates to the right screen? The unit test has no idea. You've tested that a button calls a function, which is roughly equivalent to testing that JavaScript's function invocation works. Congratulations.

Meanwhile, your users don't interact with isolated components. They interact with *screens.* They tap a button on a form, expect data to be submitted, expect a loading spinner, expect a success message, and expect to end up on a different screen. The interesting bugs — the ones that actually hit production — live in the *integration* between components, not inside individual components.

### 1.2 The Testing Trophy Model

Kent C. Dodds introduced the Testing Trophy (sometimes called the Testing Diamond) as a replacement. It looks like this:

```
                    ┌─────────┐
                    │  E2E    │  ← Few. Critical paths only.
                    │ (small) │    Real device / browser.
                ┌───┴─────────┴───┐
                │   Integration    │  ← MOST tests live here.
                │   (large)        │    Render screens, mock network.
                │                  │    Test user flows.
            ┌───┴──────────────────┴───┐
            │       Unit                │  ← Pure logic only.
            │       (medium)            │    Utils, hooks, stores.
            │                           │    No component rendering.
        ┌───┴───────────────────────────┴───┐
        │        Static Analysis             │  ← TypeScript, ESLint, Biome.
        │        (foundation)                │    Catches bugs before tests run.
        └───────────────────────────────────┘
```

The key insight: **integration tests give you the best return on investment.** They're fast enough to run on every commit (unlike E2E), and they catch real bugs (unlike unit tests on isolated components). They test the actual user experience — "I fill in the form, press submit, and see the success screen" — without the fragility of running on a real device or browser.

### 1.3 The ROI Breakdown

| Test Type | Speed | Confidence | Maintenance | ROI |
|-----------|-------|------------|-------------|-----|
| Static (TS/ESLint) | Instant | Low–Medium | Near-zero | Very high |
| Unit (pure logic) | Very fast | Low | Low | Medium |
| Integration (screens) | Fast | **High** | Medium | **Very high** |
| E2E (Maestro/Playwright) | Slow | Very high | High | Medium |
| Snapshot | Very fast | **Nearly zero** | **Very high** | **Negative** |

The "nearly zero confidence" on snapshots is not an exaggeration. We'll get to that in Section 9.

### 1.4 What This Means in Practice

For a typical React Native app, your test distribution should look something like:

```
Static analysis (TypeScript strict mode + ESLint/Biome):
  → Catches ~40% of bugs before any test runs
  → Cost: near-zero after initial setup

Integration tests (RNTL + MSW):
  → 60-80% of your test files
  → Test each screen's primary user flow
  → Test error states, loading states, empty states
  → Test form validation and submission
  → Mock network, render real component trees

Unit tests (Jest):
  → 15-30% of your test files
  → Pure utility functions (formatCurrency, parseDate)
  → Complex business logic (calculateShipping, validateCoupon)
  → Zustand store logic (actions, computed values)
  → Custom hooks with complex state machines

E2E tests (Maestro for mobile, Playwright for web):
  → 5-10% of your test files
  → Critical paths: sign up, sign in, purchase, core feature
  → Smoke tests for each app update before release
  → Visual regression on key screens
```

> **Shopify Mobile:** Shopify's React Native team publicly shared that they shifted from a pyramid model to a trophy model during their migration. Their integration test suite (RNTL) catches more regressions per test than their previous unit-test-heavy approach, with faster CI times because they deleted hundreds of shallow-render unit tests that were testing React's own rendering logic.

---

## 2. JEST CONFIGURATION FOR REACT NATIVE

Jest is the test runner. Getting it configured correctly for React Native is one of those things that wastes a day the first time and five minutes every time after — if you know the tricks. Here's the complete setup.

### 2.1 Base Configuration

If you're using Expo (and you should be — see Chapter 5), the starting point is `jest-expo`. It handles the worst of the React Native transform pain for you.

```bash
# Install testing dependencies
npx expo install jest-expo @testing-library/react-native @testing-library/jest-native
npm install --save-dev @types/jest
```

Your `jest.config.js` (or `jest.config.ts` if you prefer):

```js
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  // Use jest-expo preset — handles RN transforms, Hermes syntax, etc.
  preset: 'jest-expo',

  // Transform files that Jest can't handle natively
  transformIgnorePatterns: [
    // The key trick: DON'T ignore node_modules that ship untransformed ESM.
    // jest-expo handles most of this, but you may need to add packages.
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@sentry/react-native|react-native-svg|react-native-reanimated|react-native-gesture-handler|react-native-screens|react-native-safe-area-context|@gorhom/bottom-sheet|react-native-mmkv|nativewind)/)',
  ],

  // File extensions — THE ORDER MATTERS.
  // Jest resolves left-to-right. Put platform-specific extensions first.
  moduleFileExtensions: [
    'ts',
    'tsx',
    'js',
    'jsx',
    'json',
    'node',
  ],

  // Module name mapping — handle path aliases from tsconfig
  moduleNameMapper: {
    // Match your tsconfig paths
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@screens/(.*)$': '<rootDir>/src/screens/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@stores/(.*)$': '<rootDir>/src/stores/$1',
    '^@api/(.*)$': '<rootDir>/src/api/$1',
    '^@assets/(.*)$': '<rootDir>/src/assets/$1',

    // Handle static assets
    '\\.(jpg|jpeg|png|gif|webp|svg)$': '<rootDir>/__mocks__/fileMock.js',
    '\\.(css|less|scss)$': '<rootDir>/__mocks__/styleMock.js',
  },

  // Setup files that run BEFORE each test file
  setupFiles: [
    './jest.setup.js',
  ],

  // Setup files that run AFTER the testing framework is installed
  // (can use expect, jest globals, etc.)
  setupFilesAfterFramework: [],

  // Collect coverage from source files, not test files
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.{ts,tsx}',
    '!src/**/__tests__/**',
    '!src/**/*.test.{ts,tsx}',
    '!src/**/types.ts',
    '!src/**/index.ts', // Re-export barrels don't need coverage
  ],

  // Coverage thresholds — be realistic, not aspirational
  coverageThreshold: {
    global: {
      branches: 60,
      functions: 60,
      lines: 70,
      statements: 70,
    },
  },

  // Timeout — RN tests can be slower than web tests
  testTimeout: 15000,

  // Max workers — don't overwhelm CI
  maxWorkers: '50%',
};
```

### 2.2 The Setup File

This is where most teams get tripped up. React Native has native modules that don't exist in the Jest/Node.js environment. You need to mock them.

```js
// jest.setup.js

// Silence the warning: Animated: `useNativeDriver` is not supported
// because the native animated module is not available
jest.mock('react-native/Libraries/Animated/NativeAnimatedHelper');

// Mock react-native-reanimated
jest.mock('react-native-reanimated', () => {
  const Reanimated = require('react-native-reanimated/mock');
  Reanimated.default.call = () => {};
  return Reanimated;
});

// Mock react-native-gesture-handler
jest.mock('react-native-gesture-handler', () => {
  const View = require('react-native').View;
  return {
    Swipeable: View,
    DrawerLayout: View,
    State: {},
    ScrollView: View,
    Slider: View,
    Switch: View,
    TextInput: View,
    ToolbarAndroid: View,
    ViewPagerAndroid: View,
    DrawerLayoutAndroid: View,
    WebView: View,
    NativeViewGestureHandler: View,
    TapGestureHandler: View,
    FlingGestureHandler: View,
    ForceTouchGestureHandler: View,
    LongPressGestureHandler: View,
    PanGestureHandler: View,
    PinchGestureHandler: View,
    RotationGestureHandler: View,
    RawButton: View,
    BaseButton: View,
    RectButton: View,
    BorderlessButton: View,
    FlatList: require('react-native').FlatList,
    gestureHandlerRootHOC: (component) => component,
    Directions: {},
    GestureHandlerRootView: View,
  };
});

// Mock expo-router — critical for testing screens
jest.mock('expo-router', () => ({
  useRouter: () => ({
    push: jest.fn(),
    replace: jest.fn(),
    back: jest.fn(),
    canGoBack: jest.fn(() => true),
    navigate: jest.fn(),
  }),
  useLocalSearchParams: () => ({}),
  useGlobalSearchParams: () => ({}),
  useSegments: () => [],
  usePathname: () => '/',
  Link: ({ children, ...props }) => {
    const { Text } = require('react-native');
    return <Text {...props}>{children}</Text>;
  },
  Stack: {
    Screen: () => null,
  },
  Tabs: {
    Screen: () => null,
  },
  Redirect: () => null,
}));

// Mock expo-secure-store
jest.mock('expo-secure-store', () => ({
  getItemAsync: jest.fn(),
  setItemAsync: jest.fn(),
  deleteItemAsync: jest.fn(),
}));

// Mock react-native-mmkv
jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    set: jest.fn(),
    getString: jest.fn(),
    getNumber: jest.fn(),
    getBoolean: jest.fn(),
    delete: jest.fn(),
    contains: jest.fn(),
    getAllKeys: jest.fn(() => []),
    clearAll: jest.fn(),
  })),
}));

// Mock @react-native-async-storage/async-storage
jest.mock('@react-native-async-storage/async-storage', () =>
  require('@react-native-async-storage/async-storage/jest/async-storage-mock')
);

// Mock expo-haptics
jest.mock('expo-haptics', () => ({
  impactAsync: jest.fn(),
  notificationAsync: jest.fn(),
  selectionAsync: jest.fn(),
  ImpactFeedbackStyle: {
    Light: 'light',
    Medium: 'medium',
    Heavy: 'heavy',
  },
  NotificationFeedbackType: {
    Success: 'success',
    Warning: 'warning',
    Error: 'error',
  },
}));

// Mock expo-constants
jest.mock('expo-constants', () => ({
  default: {
    expoConfig: {
      name: 'TestApp',
      slug: 'test-app',
      version: '1.0.0',
      extra: {
        apiUrl: 'https://api.test.com',
      },
    },
    manifest: null,
  },
}));

// Global test utilities
// Silence console.warn in tests (uncomment if noisy, but be careful —
// warnings sometimes reveal real problems)
// const originalWarn = console.warn;
// console.warn = (...args) => {
//   if (args[0]?.includes?.('some noisy warning')) return;
//   originalWarn(...args);
// };
```

### 2.3 The Asset Mocks

```js
// __mocks__/fileMock.js
module.exports = 'test-file-stub';

// __mocks__/styleMock.js
module.exports = {};
```

### 2.4 The moduleFileExtensions Trick

Here's a trick that saves real debugging time. React Native resolves files with platform-specific extensions: `Component.ios.tsx`, `Component.android.tsx`, `Component.native.tsx`, `Component.tsx`. Jest doesn't know about this by default.

If you use `jest-expo`, it handles this. But if you're configuring Jest manually (e.g., in a monorepo shared package), you need to tell Jest about the resolution order:

```js
// For a React Native-specific test config
moduleFileExtensions: ['ios.ts', 'ios.tsx', 'native.ts', 'native.tsx', 'ts', 'tsx', 'js', 'jsx', 'json'],

// For a web-specific test config
moduleFileExtensions: ['web.ts', 'web.tsx', 'ts', 'tsx', 'js', 'jsx', 'json'],
```

This means if you have `api-client.native.ts` and `api-client.web.ts`, the React Native test config will pick up the `.native` version automatically, just like Metro does at runtime.

### 2.5 Monorepo Jest Configuration

In a Turborepo monorepo (Chapter 15), each package and app should have its own Jest config. The root `jest.config.js` uses `projects` to orchestrate them:

```js
// Root jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  projects: [
    '<rootDir>/apps/mobile/jest.config.js',
    '<rootDir>/apps/web/jest.config.js',
    '<rootDir>/packages/shared/jest.config.js',
    '<rootDir>/packages/ui/jest.config.js',
  ],
};
```

Each package has its own config with the appropriate preset:

```js
// packages/shared/jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  displayName: 'shared',
  preset: 'ts-jest',
  testEnvironment: 'node',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
};
```

```js
// apps/mobile/jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  displayName: 'mobile',
  preset: 'jest-expo',
  // ... full RN config from Section 2.1
};
```

```js
// apps/web/jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  displayName: 'web',
  testEnvironment: 'jsdom',
  transform: {
    '^.+\\.(ts|tsx)$': ['@swc/jest'],
  },
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
};
```

Now `npm test` runs all projects in parallel, and `npm test -- --selectProjects mobile` runs just the mobile tests.

---

## 3. REACT NATIVE TESTING LIBRARY (RNTL)

React Native Testing Library is the single most important testing tool in your stack. It embodies a philosophy that, once you internalize it, will fundamentally change how you think about testing.

### 3.1 The Philosophy

RNTL's guiding principle comes from Kent C. Dodds:

> "The more your tests resemble the way your software is used, the more confidence they can give you."

This means:

1. **Query by what the user sees, not by implementation details.** Use `getByText('Submit')`, not `getByTestId('submit-btn')`. Use `getByRole('button', { name: 'Submit' })`, not `root.findByType(TouchableOpacity)`.

2. **Interact the way the user interacts.** Use `fireEvent.press()`, not `component.props.onPress()`. Use `userEvent.type()`, not `component.setState({ text: 'hello' })`.

3. **Assert on what the user sees.** Assert that "Success!" text appears on screen, not that `someInternalState.isSuccess === true`.

4. **Don't test implementation details.** If you refactor a component from `useState` to `useReducer`, and the behavior doesn't change, no test should break. If you extract a subcomponent, no test should break. Tests that break on refactors are liabilities, not assets.

### 3.2 Query Priority

RNTL gives you several ways to find elements. Use them in this order:

```
1. getByRole          — Best. Matches accessibility role + name.
2. getByText          — Great. What the user literally reads.
3. getByPlaceholderText — Good for inputs.
4. getByDisplayValue  — Good for filled inputs.
5. getByLabelText     — Great for accessible forms.
6. getByTestID        — LAST RESORT. Invisible to users.
```

**Why `getByTestID` is last resort:** a test ID is invisible to the user. It couples your test to your implementation. If you rename the test ID, your test breaks. If you move the component, your test breaks. If you query by text or role instead, you're testing what the user actually sees — and that doesn't change when you refactor.

Use `getByTestID` only when there's genuinely no visible text or accessible label — for example, a decorative element you need to assert is rendered, or a component with dynamic content that's hard to query otherwise.

### 3.3 The Core Patterns

Let's build up from simple to complex. Every example is a complete test file you can drop into a project.

#### Pattern 1: Testing a Simple Component

```tsx
// src/components/Badge.tsx
import { View, Text, StyleSheet } from 'react-native';

interface BadgeProps {
  count: number;
  variant?: 'default' | 'warning' | 'error';
}

export function Badge({ count, variant = 'default' }: BadgeProps) {
  if (count === 0) return null;

  const displayCount = count > 99 ? '99+' : String(count);

  return (
    <View
      style={[styles.badge, styles[variant]]}
      accessibilityRole="text"
      accessibilityLabel={`${count} notifications`}
    >
      <Text style={styles.text}>{displayCount}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  badge: { borderRadius: 12, paddingHorizontal: 6, paddingVertical: 2 },
  default: { backgroundColor: '#007AFF' },
  warning: { backgroundColor: '#FF9500' },
  error: { backgroundColor: '#FF3B30' },
  text: { color: 'white', fontSize: 12, fontWeight: '700' },
});
```

```tsx
// src/components/__tests__/Badge.test.tsx
import { render, screen } from '@testing-library/react-native';
import { Badge } from '../Badge';

describe('Badge', () => {
  it('renders the count', () => {
    render(<Badge count={5} />);
    expect(screen.getByText('5')).toBeTruthy();
  });

  it('renders nothing when count is 0', () => {
    render(<Badge count={0} />);
    expect(screen.queryByRole('text')).toBeNull();
  });

  it('caps display at 99+', () => {
    render(<Badge count={150} />);
    expect(screen.getByText('99+')).toBeTruthy();
  });

  it('has correct accessibility label', () => {
    render(<Badge count={42} />);
    expect(screen.getByLabelText('42 notifications')).toBeTruthy();
  });
});
```

Notice what we're NOT testing: we're not testing the background color. We're not testing the border radius. We're not testing that StyleSheet.create was called with specific values. Those are implementation details. If someone changes the badge from blue to purple, no test should break — the *behavior* didn't change.

#### Pattern 2: Testing a Screen with User Interaction

This is where RNTL shines. We render an entire screen and interact with it like a user.

```tsx
// src/screens/LoginScreen.tsx
import { useState } from 'react';
import { View, Text, TextInput, Pressable, ActivityIndicator } from 'react-native';
import { useRouter } from 'expo-router';
import { useAuthStore } from '@stores/authStore';

export function LoginScreen() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const router = useRouter();
  const login = useAuthStore((s) => s.login);

  const handleLogin = async () => {
    setError(null);

    if (!email.trim()) {
      setError('Email is required');
      return;
    }
    if (!password.trim()) {
      setError('Password is required');
      return;
    }

    setIsLoading(true);
    try {
      await login(email, password);
      router.replace('/(tabs)/home');
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <View>
      <Text>Sign In</Text>

      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        autoCapitalize="none"
        keyboardType="email-address"
        accessibilityLabel="Email"
      />

      <TextInput
        placeholder="Password"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        accessibilityLabel="Password"
      />

      {error && (
        <Text accessibilityRole="alert">{error}</Text>
      )}

      <Pressable
        onPress={handleLogin}
        disabled={isLoading}
        accessibilityRole="button"
        accessibilityLabel="Sign in"
      >
        {isLoading ? (
          <ActivityIndicator />
        ) : (
          <Text>Sign In</Text>
        )}
      </Pressable>
    </View>
  );
}
```

```tsx
// src/screens/__tests__/LoginScreen.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { LoginScreen } from '../LoginScreen';
import { useAuthStore } from '@stores/authStore';
import { useRouter } from 'expo-router';

// Get the mocked router (from jest.setup.js)
const mockRouter = useRouter();

// Mock the auth store
jest.mock('@stores/authStore');
const mockLogin = jest.fn();

beforeEach(() => {
  jest.clearAllMocks();
  (useAuthStore as unknown as jest.Mock).mockImplementation((selector) =>
    selector({ login: mockLogin })
  );
});

describe('LoginScreen', () => {
  it('renders the login form', () => {
    render(<LoginScreen />);

    expect(screen.getByText('Sign In')).toBeTruthy();
    expect(screen.getByPlaceholderText('Email')).toBeTruthy();
    expect(screen.getByPlaceholderText('Password')).toBeTruthy();
    expect(screen.getByRole('button', { name: 'Sign in' })).toBeTruthy();
  });

  it('shows error when email is empty', () => {
    render(<LoginScreen />);

    fireEvent.press(screen.getByRole('button', { name: 'Sign in' }));

    expect(screen.getByText('Email is required')).toBeTruthy();
    expect(mockLogin).not.toHaveBeenCalled();
  });

  it('shows error when password is empty', () => {
    render(<LoginScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'user@test.com');
    fireEvent.press(screen.getByRole('button', { name: 'Sign in' }));

    expect(screen.getByText('Password is required')).toBeTruthy();
    expect(mockLogin).not.toHaveBeenCalled();
  });

  it('calls login and navigates on success', async () => {
    mockLogin.mockResolvedValueOnce(undefined);
    render(<LoginScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'password123');
    fireEvent.press(screen.getByRole('button', { name: 'Sign in' }));

    await waitFor(() => {
      expect(mockLogin).toHaveBeenCalledWith('user@test.com', 'password123');
      expect(mockRouter.replace).toHaveBeenCalledWith('/(tabs)/home');
    });
  });

  it('shows error message on login failure', async () => {
    mockLogin.mockRejectedValueOnce(new Error('Invalid credentials'));
    render(<LoginScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'wrong');
    fireEvent.press(screen.getByRole('button', { name: 'Sign in' }));

    await waitFor(() => {
      expect(screen.getByText('Invalid credentials')).toBeTruthy();
    });

    // Should NOT have navigated
    expect(mockRouter.replace).not.toHaveBeenCalled();
  });

  it('disables button while loading', async () => {
    // Login that never resolves (stays in loading state)
    mockLogin.mockReturnValue(new Promise(() => {}));
    render(<LoginScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'user@test.com');
    fireEvent.changeText(screen.getByPlaceholderText('Password'), 'password123');
    fireEvent.press(screen.getByRole('button', { name: 'Sign in' }));

    await waitFor(() => {
      const button = screen.getByRole('button', { name: 'Sign in' });
      expect(button.props.accessibilityState?.disabled).toBe(true);
    });
  });
});
```

**This is what a good integration test looks like.** We rendered the full screen. We interacted with it through the user-facing API (text inputs, buttons). We asserted on user-visible outcomes (error messages, navigation). We tested the happy path, validation, and error handling. If someone refactors the internal state management or changes the component structure, these tests still pass.

#### Pattern 3: Testing a List Screen

```tsx
// src/screens/__tests__/ProductListScreen.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { ProductListScreen } from '../ProductListScreen';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// We'll cover MSW in the next section, but here's the pattern with simple mocking
jest.mock('@api/products', () => ({
  useProducts: jest.fn(),
}));

import { useProducts } from '@api/products';
const mockUseProducts = useProducts as jest.Mock;

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false, // Don't retry in tests — fail fast
        gcTime: 0,    // Don't cache between tests
      },
    },
  });

  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}

describe('ProductListScreen', () => {
  const mockProducts = [
    { id: '1', name: 'Widget Pro', price: 29.99, inStock: true },
    { id: '2', name: 'Gadget Max', price: 49.99, inStock: true },
    { id: '3', name: 'Thingamajig', price: 9.99, inStock: false },
  ];

  it('shows loading state initially', () => {
    mockUseProducts.mockReturnValue({
      data: undefined,
      isLoading: true,
      isError: false,
    });

    renderWithProviders(<ProductListScreen />);
    expect(screen.getByText('Loading products...')).toBeTruthy();
  });

  it('renders product list', () => {
    mockUseProducts.mockReturnValue({
      data: mockProducts,
      isLoading: false,
      isError: false,
    });

    renderWithProviders(<ProductListScreen />);

    expect(screen.getByText('Widget Pro')).toBeTruthy();
    expect(screen.getByText('$29.99')).toBeTruthy();
    expect(screen.getByText('Gadget Max')).toBeTruthy();
    expect(screen.getByText('Thingamajig')).toBeTruthy();
  });

  it('shows out-of-stock badge', () => {
    mockUseProducts.mockReturnValue({
      data: mockProducts,
      isLoading: false,
      isError: false,
    });

    renderWithProviders(<ProductListScreen />);
    expect(screen.getByText('Out of Stock')).toBeTruthy();
  });

  it('shows error state with retry', () => {
    const refetch = jest.fn();
    mockUseProducts.mockReturnValue({
      data: undefined,
      isLoading: false,
      isError: true,
      error: new Error('Network error'),
      refetch,
    });

    renderWithProviders(<ProductListScreen />);

    expect(screen.getByText('Something went wrong')).toBeTruthy();
    fireEvent.press(screen.getByText('Try Again'));
    expect(refetch).toHaveBeenCalled();
  });

  it('shows empty state when no products', () => {
    mockUseProducts.mockReturnValue({
      data: [],
      isLoading: false,
      isError: false,
    });

    renderWithProviders(<ProductListScreen />);
    expect(screen.getByText('No products found')).toBeTruthy();
  });
});
```

#### Pattern 4: Testing Forms with React Hook Form

```tsx
// src/screens/__tests__/CreateProfileScreen.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { CreateProfileScreen } from '../CreateProfileScreen';

// Assume this screen uses React Hook Form + Zod internally
// We test the BEHAVIOR, not the form library

describe('CreateProfileScreen', () => {
  it('shows validation errors for empty required fields', async () => {
    render(<CreateProfileScreen />);

    // Try to submit without filling anything
    fireEvent.press(screen.getByText('Create Profile'));

    await waitFor(() => {
      expect(screen.getByText('Name is required')).toBeTruthy();
      expect(screen.getByText('Email is required')).toBeTruthy();
    });
  });

  it('shows validation error for invalid email', async () => {
    render(<CreateProfileScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Full name'), 'John Doe');
    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'not-an-email');
    fireEvent.press(screen.getByText('Create Profile'));

    await waitFor(() => {
      expect(screen.getByText('Invalid email address')).toBeTruthy();
    });
  });

  it('submits valid form data', async () => {
    render(<CreateProfileScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Full name'), 'John Doe');
    fireEvent.changeText(screen.getByPlaceholderText('Email'), 'john@example.com');
    fireEvent.changeText(screen.getByPlaceholderText('Bio'), 'Hello world');
    fireEvent.press(screen.getByText('Create Profile'));

    await waitFor(() => {
      expect(screen.getByText('Profile created!')).toBeTruthy();
    });
  });

  it('shows character count for bio field', () => {
    render(<CreateProfileScreen />);

    fireEvent.changeText(screen.getByPlaceholderText('Bio'), 'Hello');
    expect(screen.getByText('5/280')).toBeTruthy();
  });
});
```

### 3.4 Async Utilities

RNTL provides several utilities for handling asynchronous operations. Here's when to use each:

```tsx
// waitFor — wait for an assertion to pass (polls until timeout)
await waitFor(() => {
  expect(screen.getByText('Success')).toBeTruthy();
});

// waitFor with options
await waitFor(
  () => { expect(screen.getByText('Loaded')).toBeTruthy(); },
  { timeout: 5000, interval: 100 }
);

// findBy* — shorthand for waitFor + getBy (returns a promise)
const element = await screen.findByText('Success');
expect(element).toBeTruthy();

// waitForElementToBeRemoved — wait for something to disappear
await waitForElementToBeRemoved(() => screen.getByText('Loading...'));
expect(screen.getByText('Content')).toBeTruthy();

// act() — wrap manual state updates (rarely needed with RNTL)
// RNTL wraps fireEvent in act() automatically.
// Only use act() when you're triggering state updates manually:
import { act } from '@testing-library/react-native';
await act(async () => {
  jest.advanceTimersByTime(3000);
});
```

**Common pitfall:** "act() warnings." If you see "An update to Component inside a test was not wrapped in act(...)," it usually means an async operation completed after your test ended. The fix is almost always to `await waitFor()` for the final state, not to wrap random things in `act()`.

---

## 4. MOCK SERVICE WORKER (MSW)

Mock Service Worker is the best way to mock network requests in tests. Period. Here's why, and how to set it up for React Native.

### 4.1 Why MSW Over jest.mock(fetch)

The traditional approach is to mock `fetch` or `axios` directly:

```tsx
// The BAD way
jest.mock('axios');
const mockAxios = axios as jest.Mocked<typeof axios>;
mockAxios.get.mockResolvedValue({ data: { user: { name: 'John' } } });
```

Problems with this approach:

1. **It couples your test to your HTTP client.** If you switch from `axios` to `fetch` to `ky`, every test breaks.
2. **It doesn't test your actual request logic.** Your test doesn't verify that you're hitting the right URL, sending the right headers, or handling the response format correctly.
3. **It's hard to test request/response combinations.** Setting up mocks for sequential requests (fetch user, then fetch user's orders) gets messy fast.
4. **It doesn't test error responses properly.** You're mocking the client's behavior, not the server's behavior. A real 404 response has a status code, headers, and a body — not just a rejected promise.

MSW intercepts at the *network level*. Your code makes a real `fetch` call. MSW intercepts it before it leaves the process and returns a mocked response. Your code doesn't know it's being mocked — it processes the response exactly as it would in production.

### 4.2 Setting Up MSW for React Native

```bash
npm install --save-dev msw
```

MSW v2 uses a different setup for Node.js (tests) vs. browser (Storybook). For Jest tests, you use the `node` setup:

```ts
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```ts
// src/test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

const API_URL = 'https://api.test.com';

export const handlers = [
  // GET /users/me
  http.get(`${API_URL}/users/me`, () => {
    return HttpResponse.json({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com',
      avatar: 'https://example.com/avatar.jpg',
    });
  }),

  // GET /products
  http.get(`${API_URL}/products`, ({ request }) => {
    const url = new URL(request.url);
    const category = url.searchParams.get('category');

    const products = [
      { id: '1', name: 'Widget Pro', price: 29.99, category: 'widgets' },
      { id: '2', name: 'Gadget Max', price: 49.99, category: 'gadgets' },
      { id: '3', name: 'Widget Lite', price: 9.99, category: 'widgets' },
    ];

    const filtered = category
      ? products.filter((p) => p.category === category)
      : products;

    return HttpResponse.json({ products: filtered, total: filtered.length });
  }),

  // POST /auth/login
  http.post(`${API_URL}/auth/login`, async ({ request }) => {
    const body = await request.json() as { email: string; password: string };

    if (body.email === 'user@test.com' && body.password === 'password123') {
      return HttpResponse.json({
        token: 'mock-jwt-token',
        user: { id: '1', name: 'John Doe', email: body.email },
      });
    }

    return HttpResponse.json(
      { message: 'Invalid credentials' },
      { status: 401 }
    );
  }),

  // POST /products
  http.post(`${API_URL}/products`, async ({ request }) => {
    const body = await request.json() as { name: string; price: number };
    return HttpResponse.json(
      { id: 'new-1', ...body, createdAt: new Date().toISOString() },
      { status: 201 }
    );
  }),

  // DELETE /products/:id
  http.delete(`${API_URL}/products/:id`, ({ params }) => {
    return HttpResponse.json({ deleted: params.id });
  }),
];
```

Wire it into your Jest setup:

```js
// jest.setup.js (add to existing setup)
import { server } from './src/test/mocks/server';

// Start the server before all tests
beforeAll(() => server.listen({ onUnhandledRequest: 'warn' }));

// Reset handlers after each test (removes per-test overrides)
afterEach(() => server.resetHandlers());

// Close the server after all tests
afterAll(() => server.close());
```

The `onUnhandledRequest: 'warn'` option is gold during development — it tells you when your code makes a request that doesn't have a handler, which usually means you forgot to mock something.

### 4.3 Testing Loading, Error, and Success States

Here's the power of MSW — testing all three states of an async operation with clean, readable tests:

```tsx
// src/screens/__tests__/ProfileScreen.test.tsx
import { render, screen, waitFor } from '@testing-library/react-native';
import { http, HttpResponse } from 'msw';
import { server } from '@test/mocks/server';
import { ProfileScreen } from '../ProfileScreen';
import { createTestQueryClient, TestProviders } from '@test/utils';

const API_URL = 'https://api.test.com';

describe('ProfileScreen', () => {
  // SUCCESS STATE — uses the default handler from handlers.ts
  it('displays the user profile', async () => {
    render(
      <TestProviders>
        <ProfileScreen />
      </TestProviders>
    );

    // Initially shows loading
    expect(screen.getByText('Loading...')).toBeTruthy();

    // Then shows the profile
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeTruthy();
      expect(screen.getByText('john@example.com')).toBeTruthy();
    });
  });

  // ERROR STATE — override the handler for this test
  it('shows error state when API fails', async () => {
    server.use(
      http.get(`${API_URL}/users/me`, () => {
        return HttpResponse.json(
          { message: 'Internal Server Error' },
          { status: 500 }
        );
      })
    );

    render(
      <TestProviders>
        <ProfileScreen />
      </TestProviders>
    );

    await waitFor(() => {
      expect(screen.getByText('Something went wrong')).toBeTruthy();
    });
  });

  // NETWORK ERROR — simulate total network failure
  it('shows error on network failure', async () => {
    server.use(
      http.get(`${API_URL}/users/me`, () => {
        return HttpResponse.error();
      })
    );

    render(
      <TestProviders>
        <ProfileScreen />
      </TestProviders>
    );

    await waitFor(() => {
      expect(screen.getByText('Something went wrong')).toBeTruthy();
    });
  });

  // SLOW RESPONSE — test loading state persistence
  it('shows loading indicator while fetching', async () => {
    server.use(
      http.get(`${API_URL}/users/me`, async () => {
        // Delay the response
        await new Promise((resolve) => setTimeout(resolve, 100));
        return HttpResponse.json({ id: '1', name: 'John', email: 'john@test.com' });
      })
    );

    render(
      <TestProviders>
        <ProfileScreen />
      </TestProviders>
    );

    // Loading state should be visible immediately
    expect(screen.getByText('Loading...')).toBeTruthy();

    // Then it should resolve
    await waitFor(() => {
      expect(screen.queryByText('Loading...')).toBeNull();
      expect(screen.getByText('John')).toBeTruthy();
    });
  });

  // EMPTY STATE
  it('shows empty state when user has no data', async () => {
    server.use(
      http.get(`${API_URL}/users/me`, () => {
        return HttpResponse.json({
          id: '1',
          name: null,
          email: 'john@test.com',
          avatar: null,
        });
      })
    );

    render(
      <TestProviders>
        <ProfileScreen />
      </TestProviders>
    );

    await waitFor(() => {
      expect(screen.getByText('Complete your profile')).toBeTruthy();
    });
  });
});
```

### 4.4 The TestProviders Utility

You'll notice every test wraps the component in `<TestProviders>`. This is a standard pattern — a single wrapper that provides all the context your components need:

```tsx
// src/test/utils.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

export function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,        // Fail fast in tests
        gcTime: 0,           // No caching between tests
        staleTime: 0,        // Always refetch
      },
      mutations: {
        retry: false,
      },
    },
    // Suppress error logging in tests
    logger: {
      log: console.log,
      warn: console.warn,
      error: () => {},
    },
  });
}

interface TestProvidersProps {
  children: React.ReactNode;
  queryClient?: QueryClient;
}

export function TestProviders({
  children,
  queryClient = createTestQueryClient(),
}: TestProvidersProps) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

// Custom render function that wraps with providers automatically
import { render, RenderOptions } from '@testing-library/react-native';

export function renderWithProviders(
  ui: React.ReactElement,
  options?: RenderOptions & { queryClient?: QueryClient }
) {
  const { queryClient, ...renderOptions } = options ?? {};

  return render(
    <TestProviders queryClient={queryClient}>{ui}</TestProviders>,
    renderOptions
  );
}
```

### 4.5 Testing Request Bodies and Headers

MSW lets you assert on what your code *sent*, not just what it received:

```tsx
it('sends the correct auth header', async () => {
  let capturedHeaders: Headers | undefined;

  server.use(
    http.get(`${API_URL}/users/me`, ({ request }) => {
      capturedHeaders = request.headers;
      return HttpResponse.json({ id: '1', name: 'John', email: 'john@test.com' });
    })
  );

  renderWithProviders(<ProfileScreen />);

  await waitFor(() => {
    expect(capturedHeaders?.get('Authorization')).toBe('Bearer mock-jwt-token');
  });
});

it('sends the correct request body on mutation', async () => {
  let capturedBody: unknown;

  server.use(
    http.post(`${API_URL}/products`, async ({ request }) => {
      capturedBody = await request.json();
      return HttpResponse.json({ id: 'new-1' }, { status: 201 });
    })
  );

  renderWithProviders(<CreateProductScreen />);

  fireEvent.changeText(screen.getByPlaceholderText('Product name'), 'New Widget');
  fireEvent.changeText(screen.getByPlaceholderText('Price'), '19.99');
  fireEvent.press(screen.getByText('Create'));

  await waitFor(() => {
    expect(capturedBody).toEqual({
      name: 'New Widget',
      price: 19.99,
    });
  });
});
```

---

## 5. TESTING TANSTACK QUERY

TanStack Query is your server state manager (Chapter 9). Testing components that use it requires a bit of setup, but the patterns are clean once you know them.

### 5.1 The QueryClient Wrapper (Revisited)

Every test that involves TanStack Query needs its own `QueryClient`. This is critical — if tests share a client, cached data from one test leaks into the next:

```tsx
// WRONG — shared client
const queryClient = new QueryClient();

describe('tests', () => {
  it('test 1', () => {
    // This fetches user data and caches it
    render(<QueryClientProvider client={queryClient}><Profile /></QueryClientProvider>);
  });

  it('test 2', () => {
    // This gets the cached data from test 1 — WRONG!
    render(<QueryClientProvider client={queryClient}><Profile /></QueryClientProvider>);
  });
});

// RIGHT — fresh client per test
describe('tests', () => {
  it('test 1', () => {
    const queryClient = createTestQueryClient();
    render(<QueryClientProvider client={queryClient}><Profile /></QueryClientProvider>);
  });

  it('test 2', () => {
    const queryClient = createTestQueryClient();
    render(<QueryClientProvider client={queryClient}><Profile /></QueryClientProvider>);
  });
});
```

### 5.2 Testing Custom Query Hooks

Sometimes you want to test a custom hook in isolation, without rendering a full screen. Use `renderHook`:

```tsx
// src/hooks/useUser.ts
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`https://api.test.com/users/${userId}`);
  if (!response.ok) throw new Error('Failed to fetch user');
  return response.json();
}

export function useUser(userId: string) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    enabled: !!userId,
  });
}
```

```tsx
// src/hooks/__tests__/useUser.test.tsx
import { renderHook, waitFor } from '@testing-library/react-native';
import { http, HttpResponse } from 'msw';
import { server } from '@test/mocks/server';
import { useUser } from '../useUser';
import { TestProviders, createTestQueryClient } from '@test/utils';

const API_URL = 'https://api.test.com';

function wrapper({ children }: { children: React.ReactNode }) {
  return <TestProviders>{children}</TestProviders>;
}

describe('useUser', () => {
  it('fetches and returns user data', async () => {
    server.use(
      http.get(`${API_URL}/users/123`, () => {
        return HttpResponse.json({
          id: '123',
          name: 'Jane Smith',
          email: 'jane@example.com',
        });
      })
    );

    const { result } = renderHook(() => useUser('123'), { wrapper });

    // Initially loading
    expect(result.current.isLoading).toBe(true);
    expect(result.current.data).toBeUndefined();

    // Eventually resolves
    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual({
      id: '123',
      name: 'Jane Smith',
      email: 'jane@example.com',
    });
  });

  it('handles error state', async () => {
    server.use(
      http.get(`${API_URL}/users/999`, () => {
        return HttpResponse.json({ message: 'Not found' }, { status: 404 });
      })
    );

    const { result } = renderHook(() => useUser('999'), { wrapper });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error?.message).toBe('Failed to fetch user');
  });

  it('does not fetch when userId is empty', () => {
    const { result } = renderHook(() => useUser(''), { wrapper });

    // Should not be loading because query is disabled
    expect(result.current.isLoading).toBe(false);
    expect(result.current.fetchStatus).toBe('idle');
  });
});
```

### 5.3 Testing Mutations

Mutations are where things get interesting. You need to test that the mutation fires, handles success/error, and invalidates the right queries.

```tsx
// src/hooks/useCreateProduct.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface CreateProductInput {
  name: string;
  price: number;
}

interface Product {
  id: string;
  name: string;
  price: number;
  createdAt: string;
}

async function createProduct(input: CreateProductInput): Promise<Product> {
  const response = await fetch('https://api.test.com/products', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(input),
  });
  if (!response.ok) throw new Error('Failed to create product');
  return response.json();
}

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createProduct,
    onSuccess: () => {
      // Invalidate the products list so it refetches
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

```tsx
// src/hooks/__tests__/useCreateProduct.test.tsx
import { renderHook, waitFor, act } from '@testing-library/react-native';
import { http, HttpResponse } from 'msw';
import { server } from '@test/mocks/server';
import { useCreateProduct } from '../useCreateProduct';
import { TestProviders, createTestQueryClient } from '@test/utils';
import { QueryClient } from '@tanstack/react-query';

const API_URL = 'https://api.test.com';

describe('useCreateProduct', () => {
  it('creates a product and invalidates the cache', async () => {
    const queryClient = createTestQueryClient();
    const invalidateSpy = jest.spyOn(queryClient, 'invalidateQueries');

    function wrapper({ children }: { children: React.ReactNode }) {
      return <TestProviders queryClient={queryClient}>{children}</TestProviders>;
    }

    server.use(
      http.post(`${API_URL}/products`, async ({ request }) => {
        const body = await request.json() as { name: string; price: number };
        return HttpResponse.json(
          { id: 'new-1', ...body, createdAt: '2026-04-07T00:00:00Z' },
          { status: 201 }
        );
      })
    );

    const { result } = renderHook(() => useCreateProduct(), { wrapper });

    act(() => {
      result.current.mutate({ name: 'Test Product', price: 19.99 });
    });

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });

    expect(result.current.data).toEqual({
      id: 'new-1',
      name: 'Test Product',
      price: 19.99,
      createdAt: '2026-04-07T00:00:00Z',
    });

    // Verify cache invalidation
    expect(invalidateSpy).toHaveBeenCalledWith({ queryKey: ['products'] });
  });

  it('handles mutation error', async () => {
    server.use(
      http.post(`${API_URL}/products`, () => {
        return HttpResponse.json(
          { message: 'Validation error' },
          { status: 422 }
        );
      })
    );

    function wrapper({ children }: { children: React.ReactNode }) {
      return <TestProviders>{children}</TestProviders>;
    }

    const { result } = renderHook(() => useCreateProduct(), { wrapper });

    act(() => {
      result.current.mutate({ name: '', price: -1 });
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error?.message).toBe('Failed to create product');
  });
});
```

### 5.4 Testing Optimistic Updates

Optimistic updates are the ultimate test of your mutation setup. You need to verify three things: (1) the UI updates immediately, (2) it stays updated on success, (3) it rolls back on failure.

```tsx
// src/hooks/useToggleFavorite.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface Product {
  id: string;
  name: string;
  isFavorite: boolean;
}

export function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (productId: string) => {
      const response = await fetch(
        `https://api.test.com/products/${productId}/favorite`,
        { method: 'POST' }
      );
      if (!response.ok) throw new Error('Failed to toggle favorite');
      return response.json();
    },

    // Optimistic update
    onMutate: async (productId) => {
      await queryClient.cancelQueries({ queryKey: ['products'] });

      const previousProducts = queryClient.getQueryData<Product[]>(['products']);

      queryClient.setQueryData<Product[]>(['products'], (old) =>
        old?.map((p) =>
          p.id === productId ? { ...p, isFavorite: !p.isFavorite } : p
        )
      );

      return { previousProducts };
    },

    onError: (_err, _productId, context) => {
      // Rollback on error
      if (context?.previousProducts) {
        queryClient.setQueryData(['products'], context.previousProducts);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

```tsx
// src/hooks/__tests__/useToggleFavorite.test.tsx
import { renderHook, waitFor, act } from '@testing-library/react-native';
import { http, HttpResponse } from 'msw';
import { server } from '@test/mocks/server';
import { useToggleFavorite } from '../useToggleFavorite';
import { TestProviders, createTestQueryClient } from '@test/utils';

const API_URL = 'https://api.test.com';

describe('useToggleFavorite', () => {
  const products = [
    { id: '1', name: 'Widget', isFavorite: false },
    { id: '2', name: 'Gadget', isFavorite: true },
  ];

  it('optimistically updates the cache, then confirms', async () => {
    const queryClient = createTestQueryClient();
    queryClient.setQueryData(['products'], products);

    function wrapper({ children }: { children: React.ReactNode }) {
      return <TestProviders queryClient={queryClient}>{children}</TestProviders>;
    }

    server.use(
      http.post(`${API_URL}/products/1/favorite`, () => {
        return HttpResponse.json({ id: '1', name: 'Widget', isFavorite: true });
      })
    );

    const { result } = renderHook(() => useToggleFavorite(), { wrapper });

    act(() => {
      result.current.mutate('1');
    });

    // Check optimistic update happened immediately
    const updatedProducts = queryClient.getQueryData<typeof products>(['products']);
    expect(updatedProducts?.[0].isFavorite).toBe(true);

    await waitFor(() => {
      expect(result.current.isSuccess).toBe(true);
    });
  });

  it('rolls back on server error', async () => {
    const queryClient = createTestQueryClient();
    queryClient.setQueryData(['products'], products);

    function wrapper({ children }: { children: React.ReactNode }) {
      return <TestProviders queryClient={queryClient}>{children}</TestProviders>;
    }

    server.use(
      http.post(`${API_URL}/products/1/favorite`, () => {
        return HttpResponse.json({ message: 'Server error' }, { status: 500 });
      })
    );

    const { result } = renderHook(() => useToggleFavorite(), { wrapper });

    act(() => {
      result.current.mutate('1');
    });

    // Optimistic update happens immediately
    expect(queryClient.getQueryData<typeof products>(['products'])?.[0].isFavorite).toBe(true);

    // After error, it should roll back
    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    const rolledBack = queryClient.getQueryData<typeof products>(['products']);
    expect(rolledBack?.[0].isFavorite).toBe(false); // Back to original
  });
});
```

---

## 6. TESTING ZUSTAND STORES

Zustand stores (Chapter 9) are pure JavaScript objects with functions. This makes them extremely testable — you can test them without rendering any components at all.

### 6.1 Testing Store Logic Independently

```ts
// src/stores/cartStore.ts
import { create } from 'zustand';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  total: () => number;
  itemCount: () => number;
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],

  addItem: (item) =>
    set((state) => {
      const existing = state.items.find((i) => i.id === item.id);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...item, quantity: 1 }] };
    }),

  removeItem: (id) =>
    set((state) => ({
      items: state.items.filter((i) => i.id !== id),
    })),

  updateQuantity: (id, quantity) =>
    set((state) => ({
      items: quantity <= 0
        ? state.items.filter((i) => i.id !== id)
        : state.items.map((i) => (i.id === id ? { ...i, quantity } : i)),
    })),

  clearCart: () => set({ items: [] }),

  total: () =>
    get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),

  itemCount: () =>
    get().items.reduce((sum, item) => sum + item.quantity, 0),
}));
```

```ts
// src/stores/__tests__/cartStore.test.ts
import { useCartStore } from '../cartStore';

// Reset store between tests — CRITICAL
beforeEach(() => {
  useCartStore.setState({ items: [] });
});

describe('cartStore', () => {
  const widget = { id: '1', name: 'Widget', price: 10 };
  const gadget = { id: '2', name: 'Gadget', price: 25 };

  describe('addItem', () => {
    it('adds a new item with quantity 1', () => {
      useCartStore.getState().addItem(widget);

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1);
      expect(items[0]).toEqual({ ...widget, quantity: 1 });
    });

    it('increments quantity when adding existing item', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(widget);

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1);
      expect(items[0].quantity).toBe(2);
    });

    it('handles multiple different items', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(gadget);

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(2);
    });
  });

  describe('removeItem', () => {
    it('removes an item by id', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(gadget);
      useCartStore.getState().removeItem('1');

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1);
      expect(items[0].id).toBe('2');
    });

    it('does nothing when item not found', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().removeItem('nonexistent');

      expect(useCartStore.getState().items).toHaveLength(1);
    });
  });

  describe('updateQuantity', () => {
    it('updates quantity for an item', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().updateQuantity('1', 5);

      expect(useCartStore.getState().items[0].quantity).toBe(5);
    });

    it('removes item when quantity set to 0', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().updateQuantity('1', 0);

      expect(useCartStore.getState().items).toHaveLength(0);
    });

    it('removes item when quantity is negative', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().updateQuantity('1', -1);

      expect(useCartStore.getState().items).toHaveLength(0);
    });
  });

  describe('clearCart', () => {
    it('removes all items', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(gadget);
      useCartStore.getState().clearCart();

      expect(useCartStore.getState().items).toHaveLength(0);
    });
  });

  describe('computed values', () => {
    it('calculates total correctly', () => {
      useCartStore.getState().addItem(widget); // 10 * 1
      useCartStore.getState().addItem(gadget); // 25 * 1
      useCartStore.getState().addItem(widget); // 10 * 2

      expect(useCartStore.getState().total()).toBe(45); // 20 + 25
    });

    it('calculates item count correctly', () => {
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(widget);
      useCartStore.getState().addItem(gadget);

      expect(useCartStore.getState().itemCount()).toBe(3); // 2 + 1
    });

    it('returns 0 for empty cart', () => {
      expect(useCartStore.getState().total()).toBe(0);
      expect(useCartStore.getState().itemCount()).toBe(0);
    });
  });
});
```

Notice how clean this is. No `render`, no `screen`, no async waiting. Zustand stores are synchronous state machines. You call actions, you check state. This is the *correct* use of unit tests — pure logic with no UI.

### 6.2 Testing Components That Use Stores

When testing a component that reads from a Zustand store, you have two options:

**Option 1: Pre-populate the store (preferred)**

```tsx
// src/screens/__tests__/CartScreen.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { CartScreen } from '../CartScreen';
import { useCartStore } from '@stores/cartStore';

beforeEach(() => {
  // Reset store to known state
  useCartStore.setState({
    items: [
      { id: '1', name: 'Widget', price: 10, quantity: 2 },
      { id: '2', name: 'Gadget', price: 25, quantity: 1 },
    ],
  });
});

afterEach(() => {
  useCartStore.setState({ items: [] });
});

describe('CartScreen', () => {
  it('displays cart items', () => {
    render(<CartScreen />);

    expect(screen.getByText('Widget')).toBeTruthy();
    expect(screen.getByText('Gadget')).toBeTruthy();
    expect(screen.getByText('$45.00')).toBeTruthy(); // total
  });

  it('removes item when delete button is pressed', () => {
    render(<CartScreen />);

    // Find the remove button for Widget
    const removeButtons = screen.getAllByLabelText('Remove item');
    fireEvent.press(removeButtons[0]);

    expect(screen.queryByText('Widget')).toBeNull();
    expect(screen.getByText('Gadget')).toBeTruthy();
  });

  it('updates quantity', () => {
    render(<CartScreen />);

    const incrementButtons = screen.getAllByLabelText('Increase quantity');
    fireEvent.press(incrementButtons[0]); // Increment Widget from 2 to 3

    expect(screen.getByText('3')).toBeTruthy(); // New quantity
    expect(screen.getByText('$55.00')).toBeTruthy(); // New total: 30 + 25
  });

  it('shows empty state', () => {
    useCartStore.setState({ items: [] });
    render(<CartScreen />);

    expect(screen.getByText('Your cart is empty')).toBeTruthy();
  });
});
```

**Option 2: Mock the store (when you need to isolate)**

```tsx
jest.mock('@stores/cartStore');
import { useCartStore } from '@stores/cartStore';

const mockStore = {
  items: [{ id: '1', name: 'Widget', price: 10, quantity: 1 }],
  removeItem: jest.fn(),
  total: () => 10,
  itemCount: () => 1,
};

(useCartStore as unknown as jest.Mock).mockImplementation((selector) =>
  selector(mockStore)
);
```

**When to use which:** Option 1 (real store) is better for integration tests — you're testing the real store behavior. Option 2 (mock store) is better when you specifically want to test the component's rendering logic in isolation, or when the store has complex dependencies (like MMKV persistence) that are hard to set up in tests.

---

## 7. MAESTRO FOR MOBILE E2E

If you need to test your app on actual devices (or simulators), Maestro is the tool. It has won the React Native E2E testing battle against Detox, and it's not even close.

### 7.1 Why Maestro Won

Let's be direct about why Detox lost:

| Factor | Detox | Maestro |
|--------|-------|---------|
| Setup complexity | Extremely complex. Needs native build config, gray-box sync, custom bespoke setup per project. | Install CLI. Point at app. Done. |
| Test language | JavaScript. Requires understanding Detox internals and sync mechanisms. | YAML. A QA engineer can write tests in 10 minutes. |
| Flakiness | Notorious. The gray-box synchronization model breaks in creative ways with animations, timers, network requests. | Very stable. Black-box approach with smart waiting. |
| Speed | Moderate. Build + test cycle is slow. | Fast. No recompilation needed between test changes. |
| React Native support | Good but fragile. Breaking changes with RN upgrades. | Excellent. Platform-agnostic. |
| CI integration | Complex. Needs Android emulator + iOS simulator in CI. | Simple. First-class Maestro Cloud for CI. |
| Recording | No built-in recording. | `maestro record` — records your manual interaction as YAML. |
| Debugging | Painful. Log files, manual inspection. | `maestro studio` — live interactive test builder in the browser. |
| Maintenance cost | High. Tests break frequently on RN upgrades. | Low. YAML tests are stable across versions. |

The fundamental difference: Detox uses a **gray-box** approach — it hooks into your app's internals to synchronize with animations, network requests, and timers. This gives more control but creates enormous coupling between the test framework and your app's runtime behavior. Every React Native upgrade, every new animation library, every change to your timer usage can break Detox synchronization.

Maestro uses a **black-box** approach — it drives the app through the accessibility layer, just like a real user would. It waits for elements by polling the screen, not by hooking into internals. This is simpler, more stable, and works identically on iOS and Android.

### 7.2 Installation and Setup

```bash
# Install Maestro CLI (macOS)
curl -Ls "https://get.maestro.mobile.dev" | bash

# Verify installation
maestro --version

# Start Maestro Studio (interactive test builder)
maestro studio
```

Build your app for testing:

```bash
# For iOS simulator
npx expo run:ios

# For Android emulator
npx expo run:android

# Or use a development build
npx expo start --dev-client
```

### 7.3 Writing Maestro Tests

Maestro tests are YAML files. No JavaScript, no compilation, no babel transforms. A QA engineer who's never written code can learn to write Maestro tests in an afternoon.

```yaml
# e2e/login-flow.yaml
appId: com.yourapp.mobile
---
# Test: User can log in with valid credentials

# Launch the app
- launchApp

# Wait for the login screen
- assertVisible: "Sign In"

# Enter email
- tapOn: "Email"
- inputText: "user@test.com"

# Enter password
- tapOn: "Password"
- inputText: "password123"

# Tap sign in
- tapOn: "Sign In"

# Assert we're on the home screen
- assertVisible: "Welcome back"
- assertVisible: "Home"
```

```yaml
# e2e/login-validation.yaml
appId: com.yourapp.mobile
---
# Test: Login form shows validation errors

- launchApp
- assertVisible: "Sign In"

# Try to submit without entering anything
- tapOn:
    text: "Sign In"
    index: 1  # The button, not the header

# Assert validation errors
- assertVisible: "Email is required"

# Enter email but not password
- tapOn: "Email"
- inputText: "user@test.com"
- tapOn:
    text: "Sign In"
    index: 1

- assertVisible: "Password is required"
```

```yaml
# e2e/add-to-cart.yaml
appId: com.yourapp.mobile
---
# Test: User can add a product to cart and see it in cart screen

- launchApp

# Navigate to a product (assumes user is logged in)
- tapOn: "Products"
- tapOn: "Widget Pro"

# Add to cart
- tapOn: "Add to Cart"
- assertVisible: "Added to cart"

# Go to cart
- tapOn: "Cart"

# Verify product is in cart
- assertVisible: "Widget Pro"
- assertVisible: "$29.99"
- assertVisible: "Qty: 1"

# Increase quantity
- tapOn: "+"
- assertVisible: "Qty: 2"
- assertVisible: "$59.98"
```

```yaml
# e2e/onboarding-flow.yaml
appId: com.yourapp.mobile
---
# Test: New user completes onboarding

- launchApp:
    clearState: true  # Fresh install state

# Onboarding screen 1
- assertVisible: "Welcome to MyApp"
- tapOn: "Next"

# Onboarding screen 2
- assertVisible: "Track your progress"
- tapOn: "Next"

# Onboarding screen 3
- assertVisible: "Get started"
- tapOn: "Create Account"

# Registration
- tapOn: "Full name"
- inputText: "Test User"
- tapOn: "Email"
- inputText: "test@example.com"
- tapOn: "Password"
- inputText: "secureP4ss!"
- tapOn: "Create Account"

# Should land on home screen
- assertVisible: "Welcome, Test User"
```

### 7.4 Advanced Maestro Patterns

```yaml
# Scrolling
- scrollUntilVisible:
    element: "Load More"
    direction: DOWN
    timeout: 10000

# Conditional logic
- runFlow:
    when:
      visible: "Accept Cookies"
    commands:
      - tapOn: "Accept"

# Screenshots for visual verification
- takeScreenshot: "home-screen-loaded"

# Wait for specific duration (use sparingly)
- wait: 2000

# Swipe gestures
- swipe:
    direction: LEFT
    duration: 500

# Back navigation
- pressKey: back

# Run a sub-flow
- runFlow: "e2e/flows/login.yaml"

# Repeat actions
- repeat:
    times: 3
    commands:
      - tapOn: "+"

# Environment variables
- inputText: "${EMAIL}"  # Pass via: maestro test -e EMAIL=user@test.com
```

### 7.5 Organizing Maestro Tests

```
e2e/
├── flows/                    # Reusable sub-flows
│   ├── login.yaml           # Login flow (reused by many tests)
│   ├── logout.yaml
│   └── navigate-to-cart.yaml
├── auth/
│   ├── login-success.yaml
│   ├── login-validation.yaml
│   ├── signup-success.yaml
│   └── forgot-password.yaml
├── cart/
│   ├── add-to-cart.yaml
│   ├── remove-from-cart.yaml
│   └── checkout.yaml
├── profile/
│   ├── edit-profile.yaml
│   └── change-password.yaml
└── smoke/
    └── critical-paths.yaml   # Runs all critical flows in sequence
```

### 7.6 Running Maestro Tests

```bash
# Run a single test
maestro test e2e/auth/login-success.yaml

# Run all tests in a directory
maestro test e2e/auth/

# Run with environment variables
maestro test -e EMAIL=user@test.com -e PASSWORD=pass123 e2e/auth/login-success.yaml

# Record a test (manually interact, Maestro writes YAML)
maestro record

# Start Maestro Studio (browser-based test builder)
maestro studio

# Run on specific device
maestro test --device emulator-5554 e2e/auth/login-success.yaml
```

### 7.7 Maestro in CI

For CI, you have two options: run locally on a simulator/emulator, or use Maestro Cloud.

**Option 1: Local CI (GitHub Actions)**

```yaml
# .github/workflows/e2e-mobile.yml
name: Mobile E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  e2e-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Maestro
        run: curl -Ls "https://get.maestro.mobile.dev" | bash

      - name: Build iOS app
        run: npx expo run:ios --configuration Release

      - name: Boot iOS Simulator
        run: |
          xcrun simctl boot "iPhone 16"

      - name: Run E2E tests
        run: |
          export PATH="$PATH":"$HOME/.maestro/bin"
          maestro test e2e/smoke/

      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: maestro-screenshots
          path: ~/.maestro/tests/
```

**Option 2: Maestro Cloud (recommended for teams)**

```yaml
# .github/workflows/e2e-cloud.yml
name: Mobile E2E (Maestro Cloud)

on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build Android APK
        run: |
          npx expo prebuild --platform android
          cd android && ./gradlew assembleRelease

      - name: Upload to Maestro Cloud
        uses: mobile-dev-inc/action-maestro-cloud@v1
        with:
          api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
          app-file: android/app/build/outputs/apk/release/app-release.apk
          workspace: e2e/
```

Maestro Cloud runs your tests on real devices in the cloud, provides video recordings of each test, and integrates with GitHub PR checks. For teams, this is the path of least resistance.

---

## 8. PLAYWRIGHT FOR WEB E2E

If your monorepo includes a Next.js web app (Chapter 25), Playwright is the E2E tool. It's fast, reliable, and has the best API of any browser automation tool ever built.

### 8.1 Setup for Next.js

```bash
npm install --save-dev @playwright/test
npx playwright install
```

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'github' : 'html',

  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',       // Capture trace on failure
    screenshot: 'only-on-failure', // Screenshot on failure
    video: 'retain-on-failure',    // Video on failure
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'mobile-chrome',
      use: { ...devices['Pixel 7'] },
    },
    {
      name: 'mobile-safari',
      use: { ...devices['iPhone 15'] },
    },
  ],

  // Start the Next.js dev server before running tests
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

### 8.2 The Page Object Pattern

Page objects encapsulate page-specific selectors and actions. They make tests readable and maintainable.

```ts
// e2e/pages/LoginPage.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly signInButton: Locator;
  readonly errorMessage: Locator;
  readonly forgotPasswordLink: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.signInButton = page.getByRole('button', { name: 'Sign In' });
    this.errorMessage = page.getByRole('alert');
    this.forgotPasswordLink = page.getByRole('link', { name: 'Forgot password?' });
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.signInButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }

  async expectRedirectTo(path: string) {
    await expect(this.page).toHaveURL(new RegExp(path));
  }
}
```

```ts
// e2e/pages/DashboardPage.ts
import { type Page, type Locator, expect } from '@playwright/test';

export class DashboardPage {
  readonly page: Page;
  readonly welcomeMessage: Locator;
  readonly navLinks: Locator;
  readonly userMenu: Locator;
  readonly logoutButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.welcomeMessage = page.getByText(/Welcome back/);
    this.navLinks = page.getByRole('navigation').getByRole('link');
    this.userMenu = page.getByRole('button', { name: /user menu/i });
    this.logoutButton = page.getByRole('menuitem', { name: 'Log out' });
  }

  async expectLoaded() {
    await expect(this.welcomeMessage).toBeVisible();
  }

  async logout() {
    await this.userMenu.click();
    await this.logoutButton.click();
  }

  async navigateTo(linkText: string) {
    await this.page.getByRole('link', { name: linkText }).click();
  }
}
```

### 8.3 Writing Tests with Page Objects

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test.describe('Authentication', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboard = new DashboardPage(page);

    await loginPage.goto();
    await loginPage.login('user@test.com', 'password123');
    await loginPage.expectRedirectTo('/dashboard');
    await dashboard.expectLoaded();
  });

  test('invalid credentials show error', async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.goto();
    await loginPage.login('user@test.com', 'wrongpassword');
    await loginPage.expectError('Invalid credentials');
  });

  test('empty form shows validation errors', async ({ page }) => {
    const loginPage = new LoginPage(page);

    await loginPage.goto();
    await loginPage.signInButton.click();

    await expect(page.getByText('Email is required')).toBeVisible();
    await expect(page.getByText('Password is required')).toBeVisible();
  });

  test('logout returns to login page', async ({ page }) => {
    const loginPage = new LoginPage(page);
    const dashboard = new DashboardPage(page);

    // Login first
    await loginPage.goto();
    await loginPage.login('user@test.com', 'password123');
    await dashboard.expectLoaded();

    // Logout
    await dashboard.logout();
    await loginPage.expectRedirectTo('/login');
  });
});
```

### 8.4 Testing with Authentication State

Most E2E tests need a logged-in user. Logging in through the UI for every test is slow. Playwright lets you save and reuse authentication state:

```ts
// e2e/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import path from 'path';

const authFile = path.join(__dirname, '.auth/user.json');

setup('authenticate', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Sign In' }).click();

  // Wait for redirect to confirm login worked
  await expect(page).toHaveURL('/dashboard');

  // Save authentication state (cookies, localStorage, etc.)
  await page.context().storageState({ path: authFile });
});
```

```ts
// playwright.config.ts — add to projects
projects: [
  // Setup project — runs first, saves auth state
  { name: 'setup', testMatch: /.*\.setup\.ts/ },

  {
    name: 'chromium',
    use: {
      ...devices['Desktop Chrome'],
      // Use saved auth state
      storageState: 'e2e/.auth/user.json',
    },
    dependencies: ['setup'],
  },
],
```

Now every test in the `chromium` project starts already logged in — no need to go through the login flow each time.

### 8.5 Visual Comparison (Screenshot Testing)

Playwright has built-in screenshot comparison:

```ts
// e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test('dashboard matches visual snapshot', async ({ page }) => {
  await page.goto('/dashboard');

  // Wait for all content to load
  await expect(page.getByText('Welcome back')).toBeVisible();

  // Full page screenshot comparison
  await expect(page).toHaveScreenshot('dashboard.png', {
    maxDiffPixelRatio: 0.01, // Allow 1% pixel difference
    animations: 'disabled',   // Freeze animations for consistency
  });
});

test('product card renders correctly', async ({ page }) => {
  await page.goto('/products');

  // Element-level screenshot
  const productCard = page.getByTestId('product-card').first();
  await expect(productCard).toHaveScreenshot('product-card.png');
});

test('responsive layout on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/dashboard');

  await expect(page).toHaveScreenshot('dashboard-mobile.png');
});
```

First run creates the baseline screenshots. Subsequent runs compare against them. If there's a difference, the test fails and generates a diff image showing exactly what changed.

```bash
# Update baselines after intentional visual changes
npx playwright test --update-snapshots
```

### 8.6 CI Integration

```yaml
# .github/workflows/e2e-web.yml
name: Web E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Build Next.js app
        run: npm run build

      - name: Run E2E tests
        run: npx playwright test
        env:
          CI: true

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7

      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-screenshots
          path: test-results/
```

---

## 9. WHAT NOT TO TEST

This section might save you more time than all the others combined. Knowing what *not* to test is a senior engineer skill. Junior engineers test everything. Senior engineers test the things that matter.

### 9.1 Snapshot Tests (Almost Always Worthless)

I'm going to say something that might be controversial in some circles: **snapshot tests are almost always net-negative for your codebase.**

Here's why. A snapshot test does this:

```tsx
// DON'T DO THIS
it('renders correctly', () => {
  const tree = renderer.create(<ProductCard product={mockProduct} />).toJSON();
  expect(tree).toMatchSnapshot();
});
```

This creates a snapshot file — a JSON serialization of the entire rendered component tree, including every style, every prop, every nested component. The first time it runs, it creates the snapshot. Every subsequent run, it compares against the saved snapshot.

**The problem:** any change to the component breaks the test. Changed the font size? Test fails. Reordered the styles? Test fails. Added a wrapper `<View>`? Test fails. Refactored child components? Test fails. Upgraded a library that changes default props? Test fails.

And what happens when a snapshot test fails? The developer looks at the diff, sees a 200-line JSON change, can't tell if it's intentional or a bug, and runs `npx jest --updateSnapshot`. **Every. Single. Time.**

I have never, in my career, seen a snapshot test catch a real bug that wouldn't have been caught by a behavioral test. I have seen snapshot tests waste thousands of engineering hours on meaningless updates.

**The one exception:** snapshot tests on *small, stable, serializable data structures* — like API response transformers or Zod schema outputs — can occasionally be useful. But for components? No.

```tsx
// This is acceptable — testing a data transformation
it('transforms API response to app model', () => {
  const apiResponse = { user_name: 'John', created_at: '2026-01-01' };
  const result = transformUser(apiResponse);
  expect(result).toMatchInlineSnapshot(`
    {
      "createdAt": "2026-01-01",
      "userName": "John",
    }
  `);
});
```

Inline snapshots are slightly better because the expected value is visible in the test file, making it easier to review changes. But for components, just write behavioral assertions.

### 9.2 Implementation Details

If your test breaks when you refactor but the behavior stays the same, your test is testing implementation details.

```tsx
// BAD — testing implementation details
it('calls setState with the right value', () => {
  const setState = jest.spyOn(React, 'useState');
  render(<Counter />);
  fireEvent.press(screen.getByText('+'));
  expect(setState).toHaveBeenCalledWith(1);
});

// BAD — testing internal state shape
it('has the right internal state', () => {
  const { result } = renderHook(() => useCounter());
  act(() => result.current.increment());
  expect(result.current._internalCount).toBe(1); // Don't access internals
});

// BAD — testing that specific child components are rendered
it('renders a TouchableOpacity', () => {
  const tree = render(<Button title="Click" />);
  expect(tree.root.findByType(TouchableOpacity)).toBeTruthy();
});

// GOOD — test behavior
it('increments the displayed count', () => {
  render(<Counter />);
  fireEvent.press(screen.getByText('+'));
  expect(screen.getByText('1')).toBeTruthy();
});
```

### 9.3 Third-Party Library Internals

Don't test that React Navigation navigates. Don't test that TanStack Query caches. Don't test that Zustand updates state. These libraries have their own test suites.

```tsx
// BAD — testing that react-query works
it('caches the response', async () => {
  render(<Profile />);
  await waitFor(() => expect(screen.getByText('John')).toBeTruthy());
  // Unmount and remount
  cleanup();
  render(<Profile />);
  // Assert it didn't fetch again — WRONG, you're testing react-query, not your code
});

// GOOD — test YOUR behavior that depends on the library
it('shows cached data while refreshing', async () => {
  render(<Profile />);
  await waitFor(() => expect(screen.getByText('John')).toBeTruthy());
  // Pull to refresh
  // Assert loading indicator appears while data stays visible
  // This tests YOUR loading/refresh UX, not the cache itself
});
```

### 9.4 Obvious CRUD Operations

If your component is a thin wrapper around a form that calls an API, and the form library handles validation, and the API client handles the request... what are you testing?

```tsx
// QUESTIONABLE — if this is a standard form with no custom logic
it('submits the form', async () => {
  render(<EditNameForm />);
  fireEvent.changeText(screen.getByPlaceholderText('Name'), 'John');
  fireEvent.press(screen.getByText('Save'));
  await waitFor(() => expect(screen.getByText('Saved!')).toBeTruthy());
});
```

This test is fine, but if you have 50 similar CRUD forms and each has this exact test, you're spending testing time on the least risky part of your app. Invest that time in testing the complex flows instead — checkout, onboarding, state machines, permission gates.

### 9.5 The Decision Framework

Ask yourself: "If this test didn't exist and someone introduced a bug here, how bad would it be, and how quickly would we find out?"

| Scenario | Skip Test? | Why |
|----------|-----------|-----|
| Login/auth flow breaks | Never skip | Users literally can't use the app |
| Payment flow breaks | Never skip | You lose money |
| A badge shows wrong color | Probably skip | Low impact, caught in visual review |
| A list sort order changes | Test if user-facing | Depends on whether sort order matters |
| Utility function returns wrong value | Test it | Pure logic, easy to test, hard to catch otherwise |
| CSS padding changes | Skip | Visual review catches this |
| Navigation to wrong screen | Test it | Broken navigation is a terrible UX |
| Third-party library renders correctly | Skip | Not your problem |

---

## 10. VISUAL REGRESSION TESTING

Visual regression testing catches UI changes that behavioral tests miss — the button that moved 3px, the color that's slightly wrong, the layout that breaks on a specific viewport. It's powerful, but it comes with real maintenance costs.

### 10.1 The Tools

| Tool | Type | Cost | Best For |
|------|------|------|----------|
| Chromatic | Storybook-based | Free tier, paid for teams | Component-level visual testing, design systems |
| Percy | Screenshot service | Paid | Full-page visual testing, CI integration |
| Playwright screenshots | Built-in | Free | Page-level comparison, part of E2E suite |
| Storybook + test-runner | Storybook-based | Free | Component testing + visual snapshots |
| Loki | Storybook-based | Free | Open-source alternative to Chromatic |
| Applitools | AI-powered | Paid | Smart visual comparison with AI ignore regions |

### 10.2 Chromatic with Storybook

If you have a design system with Storybook (Chapter 16), Chromatic is the most natural choice. It captures a screenshot of every story on every commit and flags visual changes for review.

```bash
npm install --save-dev chromatic
```

```json
// package.json
{
  "scripts": {
    "chromatic": "chromatic --project-token=$CHROMATIC_PROJECT_TOKEN"
  }
}
```

```yaml
# .github/workflows/chromatic.yml
name: Visual Regression

on:
  pull_request:
    branches: [main]

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Full git history for accurate baselines

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run Chromatic
        uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitZeroOnChanges: true # Don't fail CI, just flag changes
```

Chromatic provides a web UI where reviewers can accept or reject visual changes. This integrates into your PR review workflow — a visual change that wasn't reviewed gets flagged.

### 10.3 When Visual Regression Is Worth It

**Worth it:**
- Design system / component library (shared across teams)
- Marketing pages / landing pages (pixel-perfect matters)
- Complex data visualization (charts, graphs, dashboards)
- Responsive layouts that must work across specific viewports
- Apps with strict brand guidelines

**Not worth it:**
- Early-stage products where UI changes constantly
- Small teams (<5 engineers) where visual review in PRs is sufficient
- Apps with dynamic/personalized content (too many false positives)
- Teams without time to maintain baselines and review diffs

The maintenance cost is real. Every time you intentionally change the UI, you need to update baselines. If you have 200 stories and change the primary color, you have 200 baselines to update. Chromatic makes this manageable with batch approvals, but it's still work.

### 10.4 DIY Visual Regression with Playwright

If you don't want to pay for Chromatic, Playwright's built-in screenshot comparison is a solid free alternative:

```ts
// e2e/visual-regression.spec.ts
import { test, expect } from '@playwright/test';

const viewports = [
  { name: 'desktop', width: 1440, height: 900 },
  { name: 'tablet', width: 768, height: 1024 },
  { name: 'mobile', width: 375, height: 812 },
];

for (const viewport of viewports) {
  test(`homepage renders correctly on ${viewport.name}`, async ({ page }) => {
    await page.setViewportSize({ width: viewport.width, height: viewport.height });
    await page.goto('/');

    // Wait for content to load
    await expect(page.getByRole('heading', { level: 1 })).toBeVisible();

    // Mask dynamic content to reduce false positives
    await expect(page).toHaveScreenshot(`homepage-${viewport.name}.png`, {
      mask: [
        page.locator('[data-testid="current-time"]'),
        page.locator('[data-testid="random-hero-image"]'),
      ],
      maxDiffPixelRatio: 0.01,
      animations: 'disabled',
    });
  });
}
```

The `mask` option is critical — it blacks out elements that change between runs (timestamps, random images, user avatars) so they don't cause false positives.

---

## 11. PERFORMANCE TESTING

Performance testing is the least-adopted testing discipline in frontend, which is ironic because performance bugs are the ones users *feel* most acutely. A broken feature gets a bug report. A slow feature gets an uninstall.

### 11.1 Flashlight for Mobile Performance

Flashlight (formerly `reassure` from Callstack, now evolved into a broader tool) measures React Native rendering performance in a repeatable way.

```bash
npm install --save-dev @callstack/reassure
```

```ts
// reassure.config.ts
import { configure } from '@callstack/reassure';

configure({
  runs: 20,          // Run each test 20 times for statistical significance
  warmupRuns: 5,     // Discard first 5 runs (JIT warmup)
});
```

```tsx
// src/screens/__tests__/ProductList.perf.test.tsx
import { measurePerformance } from '@callstack/reassure';
import { ProductListScreen } from '../ProductListScreen';
import { TestProviders } from '@test/utils';

// Mock data
jest.mock('@api/products', () => ({
  useProducts: () => ({
    data: Array.from({ length: 100 }, (_, i) => ({
      id: String(i),
      name: `Product ${i}`,
      price: Math.random() * 100,
      inStock: Math.random() > 0.2,
    })),
    isLoading: false,
    isError: false,
  }),
}));

test('ProductList renders within performance budget', async () => {
  const scenario = async (screen) => {
    // Simulate user scrolling
    const list = screen.getByTestId('product-list');
    // The measurement captures render time
  };

  await measurePerformance(
    <TestProviders>
      <ProductListScreen />
    </TestProviders>,
    { scenario }
  );
});

test('ProductList handles re-renders efficiently', async () => {
  const scenario = async (screen) => {
    // Simulate a state update that triggers re-render
    fireEvent.press(screen.getByText('Sort by Price'));
  };

  const { results } = await measurePerformance(
    <TestProviders>
      <ProductListScreen />
    </TestProviders>,
    { scenario }
  );

  // Assert render count stays within bounds
  expect(results.meanRenderCount).toBeLessThan(5);
});
```

Reassure runs your component multiple times, measures render duration, and compares against a baseline. In CI, it posts a comment on your PR showing if performance regressed:

```
| Test                    | Baseline | Current | Change   |
|------------------------|----------|---------|----------|
| ProductList render     | 12.3ms   | 11.8ms  | -4.1%    |
| ProductList re-render  | 3.2ms    | 8.7ms   | +171.9%  | <-- REGRESSION
```

**CI integration:**

```yaml
# .github/workflows/perf.yml
name: Performance Tests

on:
  pull_request:
    branches: [main]

jobs:
  perf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run performance tests (baseline)
        run: |
          git checkout main
          npm ci
          npx reassure --baseline

      - name: Run performance tests (current)
        run: |
          git checkout -
          npm ci
          npx reassure

      - name: Post results
        uses: callstack/reassure-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### 11.2 Lighthouse CI for Web

Lighthouse CI runs Lighthouse audits on every commit and fails your build if performance budgets are exceeded.

```bash
npm install --save-dev @lhci/cli
```

```js
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      // Run against the built Next.js app
      startServerCommand: 'npm run start',
      url: [
        'http://localhost:3000/',
        'http://localhost:3000/products',
        'http://localhost:3000/dashboard',
      ],
      numberOfRuns: 3, // Run 3 times, take median
      settings: {
        preset: 'desktop',
        // Chrome flags for consistent results
        chromeFlags: '--no-sandbox --headless',
      },
    },
    assert: {
      assertions: {
        // Performance budgets
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.95 }],
        'categories:best-practices': ['warn', { minScore: 0.9 }],
        'categories:seo': ['warn', { minScore: 0.9 }],

        // Specific metrics
        'first-contentful-paint': ['error', { maxNumericValue: 1500 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
        'interactive': ['error', { maxNumericValue: 3000 }],

        // Bundle size
        'resource-summary:script:size': ['error', { maxNumericValue: 350000 }], // 350KB JS budget
      },
    },
    upload: {
      target: 'temporary-public-storage', // Free, or use your own LHCI server
    },
  },
};
```

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run Lighthouse CI
        run: npx @lhci/cli autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

### 11.3 Benchmarking Render Performance

For micro-level render benchmarking, you can use a simple pattern with `React.Profiler`:

```tsx
// src/test/perf-utils.tsx
import React from 'react';

interface RenderTiming {
  id: string;
  phase: 'mount' | 'update';
  actualDuration: number;
  baseDuration: number;
  startTime: number;
  commitTime: number;
}

export function measureRenders(
  ui: React.ReactElement,
  options?: { runs?: number }
) {
  const timings: RenderTiming[] = [];
  const runs = options?.runs ?? 10;

  function onRender(
    id: string,
    phase: 'mount' | 'update',
    actualDuration: number,
    baseDuration: number,
    startTime: number,
    commitTime: number
  ) {
    timings.push({ id, phase, actualDuration, baseDuration, startTime, commitTime });
  }

  const WrappedComponent = (
    <React.Profiler id="perf-test" onRender={onRender}>
      {ui}
    </React.Profiler>
  );

  return {
    component: WrappedComponent,
    getTimings: () => timings,
    getAverageDuration: () => {
      const durations = timings.map((t) => t.actualDuration);
      return durations.reduce((a, b) => a + b, 0) / durations.length;
    },
    getMountTime: () => {
      const mount = timings.find((t) => t.phase === 'mount');
      return mount?.actualDuration ?? 0;
    },
  };
}
```

```tsx
// Usage in a test
import { render, fireEvent } from '@testing-library/react-native';
import { measureRenders } from '@test/perf-utils';

it('ProductCard renders efficiently on re-render', () => {
  const { component, getTimings, getMountTime } = measureRenders(
    <TestProviders>
      <ProductCard product={mockProduct} />
    </TestProviders>
  );

  const { rerender } = render(component);

  // Trigger a re-render
  rerender(component);

  const mountTime = getMountTime();
  expect(mountTime).toBeLessThan(16); // Under one frame (16ms at 60fps)

  const timings = getTimings();
  const updateTimings = timings.filter((t) => t.phase === 'update');
  const avgUpdate = updateTimings.reduce((a, t) => a + t.actualDuration, 0) / updateTimings.length;
  expect(avgUpdate).toBeLessThan(8); // Updates should be faster than mounts
});
```

### 11.4 The Performance Testing Matrix

| What to Measure | Tool | Where to Run | Budget |
|----------------|------|-------------|--------|
| Component render time | Reassure | CI (every PR) | <16ms initial, <8ms update |
| Screen load time (mobile) | Flashlight / Maestro perf | Nightly CI | <500ms P75 |
| Page load (web) | Lighthouse CI | CI (every PR) | LCP <2.5s, TBT <300ms |
| Bundle size | size-limit / bundlesize | CI (every PR) | <350KB JS (web), <15MB (mobile) |
| FPS during interaction | Flashlight | Weekly CI | >55fps P10 |
| Memory usage | Xcode Instruments / Android Studio | Manual / quarterly | No leaks, <200MB peak |

### 11.5 The Pragmatic Approach to Performance Testing

Don't try to performance-test everything. Focus on the screens and interactions that matter most:

1. **The critical path.** App launch to first meaningful screen. Login to dashboard. Search to results.
2. **The heavy screens.** Long lists. Complex forms. Data-heavy dashboards. Screens with many images.
3. **The repeated interactions.** Scrolling a feed. Typing in a search box. Switching tabs.

If your app launch takes 800ms, you don't need a performance test on your settings page. Fix the 800ms first.

---

## 12. PUTTING IT ALL TOGETHER — THE TESTING STRATEGY

Here's the complete testing strategy for a production React Native + Next.js app in a monorepo:

### 12.1 Test Structure

```
apps/
├── mobile/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Badge.tsx
│   │   │   └── __tests__/
│   │   │       └── Badge.test.tsx
│   │   ├── screens/
│   │   │   ├── LoginScreen.tsx
│   │   │   └── __tests__/
│   │   │       └── LoginScreen.test.tsx
│   │   ├── hooks/
│   │   │   ├── useUser.ts
│   │   │   └── __tests__/
│   │   │       └── useUser.test.ts
│   │   ├── stores/
│   │   │   ├── cartStore.ts
│   │   │   └── __tests__/
│   │   │       └── cartStore.test.ts
│   │   ├── utils/
│   │   │   ├── format.ts
│   │   │   └── __tests__/
│   │   │       └── format.test.ts
│   │   └── test/
│   │       ├── mocks/
│   │       │   ├── handlers.ts
│   │       │   └── server.ts
│   │       └── utils.tsx
│   ├── e2e/                    # Maestro tests
│   │   ├── flows/
│   │   ├── auth/
│   │   ├── cart/
│   │   └── smoke/
│   └── jest.config.js
├── web/
│   ├── src/
│   │   └── ... (similar structure)
│   ├── e2e/                    # Playwright tests
│   │   ├── pages/
│   │   ├── auth.spec.ts
│   │   ├── visual.spec.ts
│   │   └── auth.setup.ts
│   ├── jest.config.js
│   └── playwright.config.ts
└── packages/
    └── shared/
        ├── src/
        │   ├── utils/
        │   │   └── __tests__/
        │   └── validators/
        │       └── __tests__/
        └── jest.config.js
```

### 12.2 CI Pipeline

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  # LAYER 1: Static analysis (fastest)
  lint-and-type:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx biome check .
      - run: npx tsc --noEmit

  # LAYER 2: Unit + Integration tests
  test-unit:
    runs-on: ubuntu-latest
    needs: lint-and-type
    strategy:
      matrix:
        project: [mobile, web, shared]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx jest --selectProjects ${{ matrix.project }} --ci --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  # LAYER 3: E2E (slowest, most expensive)
  e2e-web:
    runs-on: ubuntu-latest
    needs: test-unit
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build --workspace=apps/web
      - run: npx playwright test

  e2e-mobile:
    runs-on: ubuntu-latest
    needs: test-unit
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - name: Run Maestro Cloud
        uses: mobile-dev-inc/action-maestro-cloud@v1
        with:
          api-key: ${{ secrets.MAESTRO_CLOUD_API_KEY }}
          app-file: ${{ steps.build.outputs.apk-path }}
          workspace: apps/mobile/e2e/smoke/

  # LAYER 4: Performance (optional, on main only or nightly)
  performance:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: [e2e-web, e2e-mobile]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npm run build --workspace=apps/web
      - run: npx @lhci/cli autorun
```

### 12.3 The Coverage Question

"What coverage percentage should we target?"

The honest answer: **it doesn't matter.** Coverage is a lagging indicator, not a goal. A codebase with 50% coverage on the right code is better than 90% coverage on the wrong code.

That said, here are reasonable minimums that prevent "we'll add tests later" from turning into "we have no tests":

| Package | Reasonable Minimum | Notes |
|---------|-------------------|-------|
| `packages/shared` | 80% | Pure logic, easy to test |
| `apps/mobile` | 50-60% | Screens + stores + hooks |
| `apps/web` | 50-60% | Pages + stores + hooks |
| Design system | 40-50% | Behavioral tests, not visual |

The right question isn't "what's our coverage?" but "are the critical paths tested?" If login, signup, checkout, and the core feature all have integration tests, and your stores and utils have unit tests, you're in good shape — even if coverage says 55%.

### 12.4 When to Write Tests

The most practical timing:

1. **Before fixing a bug.** Write a test that reproduces the bug, then fix it. You'll never have that bug again.
2. **For complex business logic.** Anything with branching, calculations, or state machines.
3. **For screens with multiple states.** Loading, error, empty, success, partial data.
4. **Before a major refactor.** Write tests that capture current behavior, refactor, verify tests still pass.
5. **For anything that's broken before.** Once bitten, twice tested.

And when *not* to write tests:

1. **During a spike/prototype.** You're going to throw this code away. Don't test it.
2. **For trivial wrappers.** A component that just renders another component with different styles.
3. **When the cost exceeds the risk.** A test that takes 2 hours to write for a feature that takes 10 minutes to verify manually and has low risk.

### 12.5 The Testing Checklist

Before shipping a PR, run through this mental checklist:

```
[ ] Static analysis passes (TypeScript + linter)
[ ] Critical user flows have integration tests
[ ] Complex business logic has unit tests
[ ] Error states are tested (not just happy path)
[ ] Loading states are tested
[ ] Edge cases in pure functions are tested
[ ] Store mutations have isolated tests
[ ] New API endpoints have MSW handlers
[ ] E2E tests cover the critical path (login, core feature, checkout)
[ ] No snapshot tests were added (unless for data structures)
[ ] No implementation details are being tested
[ ] Tests pass on CI, not just locally
```

---

## 13. COMMON TESTING MISTAKES AND HOW TO FIX THEM

Let's close with the mistakes I see most often. These are the patterns that turn test suites into liabilities.

### 13.1 Testing the Mock, Not the Code

```tsx
// BAD — this test proves nothing
jest.mock('@api/products', () => ({
  fetchProducts: jest.fn().mockResolvedValue([{ id: '1', name: 'Widget' }]),
}));

it('fetches products', async () => {
  const result = await fetchProducts();
  expect(result).toEqual([{ id: '1', name: 'Widget' }]); // You're testing your mock!
});

// GOOD — test the component that USES the API
it('displays fetched products', async () => {
  // MSW handles the mock at the network level
  render(<TestProviders><ProductList /></TestProviders>);
  await waitFor(() => expect(screen.getByText('Widget')).toBeTruthy());
});
```

### 13.2 Over-Mocking

```tsx
// BAD — everything is mocked, nothing is tested
jest.mock('@stores/cartStore');
jest.mock('@hooks/useProducts');
jest.mock('@components/ProductCard');
jest.mock('react-native');

// At this point you're testing an empty shell.
```

The point of integration tests is to test things *working together*. If you mock every dependency, you're back to unit tests with extra steps. Mock the boundary (network, native modules), not the internals.

### 13.3 Flaky Async Tests

```tsx
// BAD — timing-dependent
it('shows success after delay', async () => {
  render(<SlowComponent />);
  await new Promise(resolve => setTimeout(resolve, 2000)); // Arbitrary sleep
  expect(screen.getByText('Done')).toBeTruthy();
});

// GOOD — wait for the condition
it('shows success after delay', async () => {
  render(<SlowComponent />);
  await waitFor(() => expect(screen.getByText('Done')).toBeTruthy(), {
    timeout: 5000,
  });
});

// EVEN BETTER — use fake timers for timer-dependent code
it('shows success after delay', () => {
  jest.useFakeTimers();
  render(<SlowComponent />);
  jest.advanceTimersByTime(2000);
  expect(screen.getByText('Done')).toBeTruthy();
  jest.useRealTimers();
});
```

### 13.4 Not Cleaning Up Between Tests

```tsx
// BAD — tests leak state into each other
describe('CartScreen', () => {
  it('adds item', () => {
    useCartStore.getState().addItem(widget);
    // ... test passes
  });

  it('shows empty cart', () => {
    render(<CartScreen />);
    // FAILS — cart still has the widget from the previous test!
  });
});

// GOOD — clean up
beforeEach(() => {
  useCartStore.setState({ items: [] });
  jest.clearAllMocks();
});
```

### 13.5 Test Descriptions That Don't Describe

```tsx
// BAD
it('works', () => { ... });
it('test 1', () => { ... });
it('should render correctly', () => { ... }); // Correctly HOW?

// GOOD — describe the behavior, not the implementation
it('shows validation error when email is empty', () => { ... });
it('navigates to dashboard after successful login', () => { ... });
it('rolls back optimistic update on server error', () => { ... });
it('disables submit button while mutation is pending', () => { ... });
```

A good test description reads like a specification. When it fails in CI, you should know what's broken without looking at the code.

---

## CLOSING THOUGHTS

Testing is not about achieving a number. It's not about a green badge on your README. It's about **confidence** — confidence that the thing you're shipping works for the people who use it.

The testing trophy gives you the most confidence per hour of engineering time invested. Integration tests with RNTL and MSW catch the bugs that actually matter. Maestro and Playwright catch the critical paths that must never break. Unit tests on pure logic catch the edge cases that sneak past everything else. And static analysis catches the typos and type errors that no human should have to catch manually.

Write the tests that let you deploy on a Friday afternoon without a knot in your stomach. Skip the tests that just make a coverage report look good. And when in doubt, write the test that would have caught the last bug that woke you up at 3 AM.

That's the testing strategy of a 100x frontend architect.

---

> **Next up:** [Chapter 18 — CI/CD for Mobile & Web](./18-cicd.md) — where we wire all these tests into a pipeline that catches problems before your users do.
