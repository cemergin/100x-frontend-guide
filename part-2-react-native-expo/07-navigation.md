<!--
  CHAPTER: 7
  TITLE: Navigation Architecture
  PART: II — React Native & Expo
  PREREQS: Chapter 5
  KEY_TOPICS: Expo Router v4, React Navigation 7, deep linking, type safety, lazy loading, prefetching, auth routing
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 7: Navigation Architecture

> **Part II — React Native & Expo** | Prerequisites: Chapter 5 | Difficulty: Intermediate

Navigation is the skeleton of your mobile app. Get it wrong and everything — performance, deep linking, state management, testing, analytics — gets harder. Get it right and half your app's architecture falls into place.

I've seen teams spend weeks debugging navigation state bugs because they chose the wrong navigator nesting strategy. I've seen apps that take 400ms to navigate between screens because every route loads eagerly at startup. I've seen deep link configurations that work on iOS but silently break on Android because the URI scheme wasn't configured symmetrically. And I've seen codebases where adding a new screen means modifying five files because navigation isn't type-safe.

This chapter is about building navigation architecture that handles all of this from day one. We'll focus on **Expo Router v4** (file-based routing built on React Navigation) and **React Navigation 7** (the imperative API underneath). You'll learn when to use each, how they interact, and how to build a navigation system that scales from MVP to millions of users.

### In This Chapter
- Expo Router v4: File-Based Routing for React Native
- React Navigation 7: The Imperative Layer
- Navigation Performance: Lazy Loading, Prefetching, Memoization
- Type-Safe Navigation
- Deep Linking Configuration
- Tab, Stack, Drawer, and Modal Navigation Patterns
- Auth-Protected Routes
- Navigation Architecture Patterns

### Related Chapters
- [Ch 5: Expo Platform] — Expo's build system and development workflow
- [Ch 3: The Rendering Pipeline] — how navigation transitions interact with React rendering
- [Ch 8: Styling & Animation] — transition animations between screens
- [Ch 13: Performance Optimization] — navigation-specific performance patterns

---

## 1. EXPO ROUTER V4: FILE-BASED ROUTING

Expo Router brings the file-based routing paradigm (popularized by Next.js) to React Native. Your file system IS your route configuration.

### 1.1 The Core Concept

```
app/
  _layout.tsx          → Root layout (wraps everything)
  index.tsx            → /  (home screen)
  about.tsx            → /about
  settings/
    _layout.tsx        → Settings layout (wraps settings screens)
    index.tsx          → /settings
    profile.tsx        → /settings/profile
    notifications.tsx  → /settings/notifications
  (auth)/
    _layout.tsx        → Auth group layout
    login.tsx          → /login
    register.tsx       → /register
  [id].tsx             → /anything (dynamic segment)
  users/
    [userId]/
      index.tsx        → /users/123
      posts/
        [postId].tsx   → /users/123/posts/456
  +not-found.tsx       → 404 page
```

**Key conventions:**
- `_layout.tsx` — Layout files wrap child routes
- `index.tsx` — The default route for a directory
- `[param].tsx` — Dynamic route segment
- `[...catchAll].tsx` — Catch-all route
- `(group)/` — Route group (doesn't appear in URL)
- `+not-found.tsx` — Custom 404 page
- `+html.tsx` — Custom HTML wrapper (web only)

### 1.2 Expo Router v4 Breaking Changes

Expo Router v4 (shipped with Expo SDK 52) introduced significant changes from v3:

**1. Typed Routes are the default:**
```tsx
// v3: Untyped by default
router.push('/users/123');

// v4: Fully typed routes generated from file structure
router.push('/users/[userId]', { userId: '123' });

// The route string is a union type derived from your file structure
// Typo in route name → compile error
```

**2. New `router` API:**
```tsx
import { router } from 'expo-router';

// Navigation methods
router.push('/settings/profile');           // Push onto stack
router.replace('/settings/profile');        // Replace current screen
router.back();                              // Go back
router.canGoBack();                         // Check if back is possible
router.dismiss();                           // Dismiss modal
router.dismissAll();                        // Dismiss all modals
router.navigate('/settings/profile');       // Navigate (smart: push or pop)
```

**3. New Link component API:**
```tsx
import { Link } from 'expo-router';

<Link href="/settings/profile">
  <Text>Go to Profile</Text>
</Link>

// With parameters
<Link href={{ pathname: '/users/[userId]', params: { userId: '123' } }}>
  <Text>View User</Text>
</Link>

// Replacing instead of pushing
<Link href="/login" replace>
  <Text>Login</Text>
</Link>
```

**4. Improved `useLocalSearchParams` and `useGlobalSearchParams`:**
```tsx
// useLocalSearchParams: reads params for the CURRENT route only
// Doesn't re-render when params in parent/sibling routes change
function UserProfile() {
  const { userId } = useLocalSearchParams<{ userId: string }>();
  return <Text>User: {userId}</Text>;
}

// useGlobalSearchParams: reads ALL params from the entire URL
// Re-renders when ANY param changes — use sparingly
function DebugPanel() {
  const params = useGlobalSearchParams();
  return <Text>{JSON.stringify(params)}</Text>;
}
```

### 1.3 Layout Files: The Architecture Foundation

Layout files define the navigation structure. They're the most important files in your routing architecture.

**Root Layout (Tab Navigator):**
```tsx
// app/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function RootLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: '#007AFF',
        headerShown: false,
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="explore"
        options={{
          title: 'Explore',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
      {/* Hide auth routes from tab bar */}
      <Tabs.Screen
        name="(auth)"
        options={{ href: null }}  
      />
    </Tabs>
  );
}
```

**Stack Layout:**
```tsx
// app/settings/_layout.tsx
import { Stack } from 'expo-router';

export default function SettingsLayout() {
  return (
    <Stack
      screenOptions={{
        headerBackTitle: 'Back',
        headerTintColor: '#007AFF',
      }}
    >
      <Stack.Screen
        name="index"
        options={{ title: 'Settings' }}
      />
      <Stack.Screen
        name="profile"
        options={{ title: 'Edit Profile' }}
      />
      <Stack.Screen
        name="notifications"
        options={{ title: 'Notification Preferences' }}
      />
    </Stack>
  );
}
```

**Modal presentation:**
```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen
        name="modal"
        options={{
          presentation: 'modal',
          headerTitle: 'Modal',
        }}
      />
      <Stack.Screen
        name="full-screen-modal"
        options={{
          presentation: 'fullScreenModal',
          headerShown: false,
        }}
      />
    </Stack>
  );
}
```

### 1.4 Route Groups

Route groups organize files without affecting the URL structure:

```
app/
  (tabs)/
    _layout.tsx        → Tab navigator
    index.tsx          → /
    explore.tsx        → /explore
    profile.tsx        → /profile
  (auth)/
    _layout.tsx        → Auth stack
    login.tsx          → /login
    register.tsx       → /register
  (modals)/
    _layout.tsx        → Modal group
    create-post.tsx    → /create-post
    filters.tsx        → /filters
```

The parentheses are stripped from the URL. `(tabs)/index.tsx` maps to `/`, not `/(tabs)`.

**Why groups matter:**
1. They let you apply different layouts to different sets of routes
2. They organize your file system without polluting URL paths
3. They enable conditional rendering based on auth state (more on this later)

### 1.5 Dynamic Routes

```tsx
// app/users/[userId].tsx
import { useLocalSearchParams } from 'expo-router';

export default function UserScreen() {
  const { userId } = useLocalSearchParams<{ userId: string }>();
  
  // userId is typed as string
  return <UserProfile userId={userId} />;
}

// app/posts/[...slug].tsx (catch-all)
import { useLocalSearchParams } from 'expo-router';

export default function PostScreen() {
  const { slug } = useLocalSearchParams<{ slug: string[] }>();
  
  // For /posts/2024/react/performance:
  // slug = ['2024', 'react', 'performance']
  return <Post path={slug.join('/')} />;
}
```

---

## 2. REACT NAVIGATION 7: THE IMPERATIVE LAYER

Expo Router is built on top of React Navigation. Understanding React Navigation 7 is important because:
1. Expo Router's layout components (`Stack`, `Tabs`, `Drawer`) ARE React Navigation navigators
2. You'll need React Navigation's APIs for advanced patterns
3. Debugging navigation issues requires understanding the underlying library

### 2.1 What's New in React Navigation 7

React Navigation 7 (released late 2024) brought a major API overhaul:

**Static configuration API:**
```tsx
// React Navigation 7 — static configuration
import { createStaticNavigation } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      screen: HomeScreen,
      options: { title: 'Home' },
    },
    Profile: {
      screen: ProfileScreen,
      options: { title: 'Profile' },
      linking: {
        path: 'profile/:userId',
      },
    },
  },
});

const Navigation = createStaticNavigation(RootStack);

export default function App() {
  return <Navigation />;
}
```

**Dynamic configuration (still supported):**
```tsx
// React Navigation 7 — dynamic configuration
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Profile" component={ProfileScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### 2.2 When to Use React Navigation Directly vs. Expo Router

| Scenario | Use Expo Router | Use React Navigation |
|----------|----------------|---------------------|
| New Expo project | Yes | No |
| Deep linking required | Yes (built-in) | Requires configuration |
| Web + Mobile support | Yes | Possible but harder |
| Existing React Native (non-Expo) | No | Yes |
| Custom navigator | No (use RN underneath) | Yes |
| Complex conditional navigation | Either | Sometimes easier |
| Type-safe routes | Yes (auto-generated) | Yes (manual types) |

**In 2026, the recommendation is clear: use Expo Router for new projects.** React Navigation is the foundation, but Expo Router adds file-based routing, automatic deep linking, type safety, and web support with minimal configuration.

### 2.3 Navigation State Structure

Understanding navigation state is essential for debugging:

```typescript
// Navigation state is a tree of route objects
type NavigationState = {
  key: string;          // Unique key for this navigator
  index: number;        // Currently focused route index
  routeNames: string[]; // All possible routes in this navigator
  routes: Array<{
    key: string;        // Unique key for this route instance
    name: string;       // Route name
    params?: object;    // Route parameters
    state?: NavigationState; // Nested navigator state (if any)
  }>;
  type: string;         // 'stack' | 'tab' | 'drawer'
  stale: boolean;       // Whether state needs rehydration
};
```

You can inspect the navigation state at any time:

```tsx
import { useNavigationState } from '@react-navigation/native';

function DebugNavigation() {
  const state = useNavigationState(state => state);
  console.log(JSON.stringify(state, null, 2));
  return null;
}
```

---

## 3. NAVIGATION PERFORMANCE

Navigation performance directly affects perceived app quality. A 300ms delay between tap and screen appearance makes the app feel slow.

### 3.1 Lazy Loading Screens

By default, stack navigators render screens only when they're navigated to. Tab navigators render all tabs eagerly.

```tsx
// Make tab screens lazy
<Tabs
  screenOptions={{
    lazy: true,           // Don't render until focused
    freezeOnBlur: true,   // Freeze invisible tabs (React Navigation 7)
  }}
>
```

**`freezeOnBlur`** uses React's `React.memo` and `Object.freeze` to prevent re-renders of unfocused tabs. This is a significant performance win for apps with expensive tab content.

```tsx
// For Expo Router tabs
// app/(tabs)/_layout.tsx
export default function TabsLayout() {
  return (
    <Tabs
      screenOptions={{
        lazy: true,
        // React Native Screens optimization
        freezeOnBlur: true,
      }}
    >
      {/* tab screens */}
    </Tabs>
  );
}
```

### 3.2 Screen Prefetching

Prefetch the next screen's data before the user navigates:

```tsx
import { useNavigation, useFocusEffect } from '@react-navigation/native';
import { useQueryClient } from '@tanstack/react-query';

function ProductListScreen() {
  const queryClient = useQueryClient();
  
  // Prefetch product details when the list screen is focused
  function handleProductVisible(productId: string) {
    queryClient.prefetchQuery({
      queryKey: ['product', productId],
      queryFn: () => fetchProduct(productId),
      staleTime: 30_000,
    });
  }
  
  return (
    <FlashList
      data={products}
      renderItem={({ item }) => (
        <ProductCard
          product={item}
          onVisible={() => handleProductVisible(item.id)}
        />
      )}
      viewabilityConfig={{ itemVisiblePercentThreshold: 50 }}
      onViewableItemsChanged={({ viewableItems }) => {
        viewableItems.forEach(({ item }) => handleProductVisible(item.id));
      }}
    />
  );
}
```

### 3.3 Navigation Transition Performance

Native stack navigators (`@react-navigation/native-stack` / Expo Router's `Stack`) use native platform transitions — iOS uses `UINavigationController` animations, Android uses Fragment transitions. These run on the UI thread and are not affected by JavaScript thread performance.

**JavaScript-based stack (`@react-navigation/stack`)** runs transitions on the JS thread using Reanimated. These CAN be affected by JS thread load. Avoid the JS-based stack unless you need custom transition animations that native doesn't support.

```tsx
// Use native stack (default in Expo Router)
import { Stack } from 'expo-router';

// Custom transitions (native)
<Stack.Screen
  name="details"
  options={{
    animation: 'slide_from_right',      // iOS + Android
    // animation: 'fade',                // Fade transition
    // animation: 'slide_from_bottom',   // Modal-like
    // animation: 'none',                // No animation
    
    // iOS-specific
    animationDuration: 250,
    
    // Full custom animation (native)
    customAnimationOnGesture: true,
    animationTypeForReplace: 'push',
  }}
/>
```

### 3.4 Memoizing Screen Components

Screens should be memoized to prevent unnecessary re-renders during navigation animations:

```tsx
// The screen component
const ProductDetailScreen = React.memo(function ProductDetailScreen() {
  const { productId } = useLocalSearchParams<{ productId: string }>();
  
  const { data: product } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
  });
  
  if (!product) return <LoadingScreen />;
  
  return (
    <ScrollView>
      <ProductHeader product={product} />
      <ProductDetails product={product} />
      <RelatedProducts productId={productId} />
    </ScrollView>
  );
});
```

With the React Compiler, explicit `React.memo` becomes unnecessary — but it doesn't hurt as a clear signal of intent.

### 3.5 Avoiding Navigation Re-render Cascades

A common performance bug is passing inline objects or functions as screen options:

```tsx
// BAD: Creates new options object every render, causing re-renders
<Stack.Screen
  name="profile"
  options={({ route }) => ({
    title: route.params?.name ?? 'Profile',
    headerRight: () => <SettingsButton />,  // New function every render
  })}
/>

// GOOD: Define options statically or use navigation.setOptions
<Stack.Screen
  name="profile"
  options={{ title: 'Profile' }}
/>

// Then in the screen component:
function ProfileScreen() {
  const navigation = useNavigation();
  const { name } = useLocalSearchParams();
  
  useLayoutEffect(() => {
    navigation.setOptions({
      title: name ?? 'Profile',
      headerRight: () => <SettingsButton />,
    });
  }, [navigation, name]);
}
```

---

## 4. TYPE-SAFE NAVIGATION

Type safety in navigation prevents an entire category of bugs — wrong route names, missing parameters, mismatched parameter types.

### 4.1 Expo Router Type Generation

Expo Router v4 automatically generates route types from your file structure:

```tsx
// After running `npx expo start`, types are generated at:
// .expo/types/router.d.ts

// These types are available globally:
import { router, useLocalSearchParams, Link } from 'expo-router';

// Route string is a union of all valid routes
router.push('/settings/profile');  // OK
router.push('/settings/profle');   // ERROR: typo caught at compile time

// Parameters are inferred from the route
router.push({
  pathname: '/users/[userId]',
  params: { userId: '123' },
});
// Missing userId → compile error
// Wrong type → compile error
```

### 4.2 Manual Type Configuration (React Navigation)

When using React Navigation directly, you define types manually:

```typescript
// types/navigation.ts
import { NativeStackScreenProps } from '@react-navigation/native-stack';
import { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import { CompositeScreenProps, NavigatorScreenParams } from '@react-navigation/native';

// Define param lists for each navigator
type RootStackParamList = {
  Main: NavigatorScreenParams<MainTabParamList>;
  Modal: { title: string; content: string };
  Auth: NavigatorScreenParams<AuthStackParamList>;
};

type MainTabParamList = {
  Home: undefined;
  Explore: { category?: string };
  Profile: { userId: string };
};

type AuthStackParamList = {
  Login: undefined;
  Register: { referralCode?: string };
  ForgotPassword: { email?: string };
};

// Screen props types
type HomeScreenProps = CompositeScreenProps<
  BottomTabScreenProps<MainTabParamList, 'Home'>,
  NativeStackScreenProps<RootStackParamList>
>;

type ProfileScreenProps = CompositeScreenProps<
  BottomTabScreenProps<MainTabParamList, 'Profile'>,
  NativeStackScreenProps<RootStackParamList>
>;

// Usage in screens
function HomeScreen({ navigation, route }: HomeScreenProps) {
  // navigation.navigate is fully typed
  navigation.navigate('Profile', { userId: '123' }); // OK
  navigation.navigate('Profile');                      // ERROR: userId required
  navigation.navigate('Profle', { userId: '123' });  // ERROR: typo
}

// Global type registration (React Navigation 7)
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

### 4.3 Type-Safe Deep Links

```tsx
// app/_layout.tsx (Expo Router handles this automatically)
// For React Navigation, configure the linking config:

const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Main: {
        screens: {
          Home: '',
          Explore: 'explore/:category?',
          Profile: 'users/:userId',
        },
      },
      Modal: 'modal',
      Auth: {
        screens: {
          Login: 'login',
          Register: 'register/:referralCode?',
        },
      },
    },
  },
};
```

---

## 5. DEEP LINKING

Deep linking is what makes your app a first-class citizen of the mobile ecosystem. It's how users get from a notification, email, or web link directly to the right screen in your app.

### 5.1 Expo Router Deep Linking (Automatic)

Expo Router generates deep linking configuration from your file structure. Every route is automatically linkable:

```
File: app/users/[userId]/posts/[postId].tsx
URL:  myapp://users/123/posts/456
Web:  https://myapp.com/users/123/posts/456
```

Configuration is minimal:

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "web": {
      "bundler": "metro"
    },
    "ios": {
      "associatedDomains": ["applinks:myapp.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "https",
              "host": "myapp.com",
              "pathPrefix": "/"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

### 5.2 Universal Links (iOS) and App Links (Android)

URL scheme links (`myapp://...`) are custom protocols. Universal Links (iOS) and App Links (Android) use HTTPS URLs that open in your app when it's installed, or fall back to the website when it's not.

**Setting up Universal Links:**

1. Host an `apple-app-site-association` file:
```json
// https://myapp.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.mycompany.myapp",
        "paths": ["*"]
      }
    ]
  }
}
```

2. Host an `assetlinks.json` file:
```json
// https://myapp.com/.well-known/assetlinks.json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.mycompany.myapp",
      "sha256_cert_fingerprints": ["YOUR_SHA256_FINGERPRINT"]
    }
  }
]
```

### 5.3 Deep Link Testing

```bash
# iOS Simulator
npx uri-scheme open "myapp://users/123" --ios

# Android Emulator
npx uri-scheme open "myapp://users/123" --android

# Or using adb directly
adb shell am start -W -a android.intent.action.VIEW -d "myapp://users/123" com.mycompany.myapp

# Expo development
npx expo start
# Then open: exp://192.168.1.100:8081/--/users/123
```

### 5.4 Handling Deep Links in the App

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Linking } from 'react-native';
import { router, useRootNavigationState } from 'expo-router';

export default function RootLayout() {
  const navigationState = useRootNavigationState();
  
  useEffect(() => {
    // Handle deep links that arrive while the app is running
    const subscription = Linking.addEventListener('url', (event) => {
      // Expo Router handles this automatically for file-based routes
      // Custom handling for non-standard links:
      const url = new URL(event.url);
      if (url.pathname.startsWith('/invite/')) {
        const code = url.pathname.split('/invite/')[1];
        handleInviteCode(code);
      }
    });
    
    return () => subscription.remove();
  }, []);
  
  // Handle the initial deep link (app opened via link)
  useEffect(() => {
    Linking.getInitialURL().then(url => {
      if (url) {
        // Process initial URL
        // Expo Router handles route-based links automatically
      }
    });
  }, []);
  
  return <Stack />;
}
```

### 5.5 Deep Link Gotchas

**1. Auth state and deep links:** A deep link to `/users/123/settings` should redirect to login if the user isn't authenticated, then redirect back after login:

```tsx
// See the auth-protected routes section below
```

**2. Stale deep links on Android:** Android can hold deep link intents even after the app is killed. Check `getInitialURL` on every cold start.

**3. Deep link testing in production:** Use deferred deep links (via Branch, Firebase Dynamic Links, or a custom solution) for marketing campaigns — these work even if the user needs to install the app first.

---

## 6. NAVIGATION PATTERNS

### 6.1 Tab + Stack Pattern (The Standard)

The most common pattern: tabs at the root, each tab containing a stack:

```
app/
  _layout.tsx              → Stack (root)
  (tabs)/
    _layout.tsx            → Tabs
    home/
      _layout.tsx          → Stack
      index.tsx            → Home screen
      [postId].tsx         → Post detail
    explore/
      _layout.tsx          → Stack
      index.tsx            → Explore screen
      search-results.tsx   → Search results
    profile/
      _layout.tsx          → Stack
      index.tsx            → Profile screen
      edit.tsx             → Edit profile
      settings.tsx         → Settings
```

```tsx
// app/_layout.tsx
export default function RootLayout() {
  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="(tabs)" />
      <Stack.Screen 
        name="modal" 
        options={{ presentation: 'modal' }} 
      />
    </Stack>
  );
}

// app/(tabs)/_layout.tsx
export default function TabsLayout() {
  return (
    <Tabs screenOptions={{ headerShown: false }}>
      <Tabs.Screen name="home" options={{ title: 'Home' }} />
      <Tabs.Screen name="explore" options={{ title: 'Explore' }} />
      <Tabs.Screen name="profile" options={{ title: 'Profile' }} />
    </Tabs>
  );
}

// app/(tabs)/home/_layout.tsx
export default function HomeLayout() {
  return <Stack />;
}
```

### 6.2 Drawer Navigation

```tsx
// app/_layout.tsx
import { Drawer } from 'expo-router/drawer';
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Drawer
        screenOptions={{
          drawerType: 'front',
          headerShown: true,
        }}
      >
        <Drawer.Screen
          name="index"
          options={{ title: 'Home', drawerLabel: 'Home' }}
        />
        <Drawer.Screen
          name="settings"
          options={{ title: 'Settings', drawerLabel: 'Settings' }}
        />
      </Drawer>
    </GestureHandlerRootView>
  );
}
```

### 6.3 Modal Patterns

**Full-screen modal:**
```tsx
// app/_layout.tsx
<Stack>
  <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
  <Stack.Screen
    name="create-post"
    options={{
      presentation: 'fullScreenModal',
      headerShown: true,
      title: 'Create Post',
      headerLeft: () => (
        <TouchableOpacity onPress={() => router.dismiss()}>
          <Text>Cancel</Text>
        </TouchableOpacity>
      ),
    }}
  />
</Stack>
```

**Bottom sheet modal:**
```tsx
// Using @gorhom/bottom-sheet with Expo Router
// app/_layout.tsx
<Stack>
  <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
  <Stack.Screen
    name="filter-sheet"
    options={{
      presentation: 'transparentModal',
      animation: 'fade',
      headerShown: false,
    }}
  />
</Stack>

// app/filter-sheet.tsx
import BottomSheet from '@gorhom/bottom-sheet';

export default function FilterSheet() {
  return (
    <View style={{ flex: 1, backgroundColor: 'rgba(0,0,0,0.5)' }}>
      <Pressable
        style={{ flex: 1 }}
        onPress={() => router.dismiss()}
      />
      <BottomSheet snapPoints={['50%', '90%']} index={0}>
        <FilterContent />
      </BottomSheet>
    </View>
  );
}
```

### 6.4 Nested Navigators: The Nesting Rules

When nesting navigators, follow these rules:

**1. Each navigator manages its own history:**
```
Root Stack
  ├── Tab Navigator
  │   ├── Home Stack
  │   │   ├── Home Screen
  │   │   └── Post Detail
  │   └── Profile Stack
  │       ├── Profile Screen
  │       └── Settings
  └── Auth Stack
      ├── Login
      └── Register

// router.back() goes back within the INNERMOST navigator
// From Post Detail → Home Screen (within Home Stack)
// From Home Screen → nothing (can't go back past tab root)
```

**2. Navigating across navigators:**
```tsx
// From Home tab, navigate to Profile tab's Settings screen
router.push('/(tabs)/profile/settings');

// This pushes onto the current stack, then navigates
// React Navigation handles the cross-navigator resolution
```

**3. Reset navigation state:**
```tsx
// Reset to a specific route (useful after login/logout)
router.replace('/');

// Or for complex resets with React Navigation:
import { CommonActions } from '@react-navigation/native';

navigation.dispatch(
  CommonActions.reset({
    index: 0,
    routes: [{ name: 'Main' }],
  })
);
```

---

## 7. AUTH-PROTECTED ROUTES

Authentication flow is a navigation architecture problem. Here's how to handle it properly.

### 7.1 The Redirect Pattern (Expo Router)

```tsx
// app/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

export default function RootLayout() {
  return (
    <AuthProvider>
      <Stack>
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="(auth)" options={{ headerShown: false }} />
      </Stack>
    </AuthProvider>
  );
}

// app/(tabs)/_layout.tsx
import { Redirect } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

export default function TabsLayout() {
  const { isAuthenticated, isLoading } = useAuth();
  
  if (isLoading) {
    return <LoadingScreen />;
  }
  
  if (!isAuthenticated) {
    return <Redirect href="/login" />;
  }
  
  return (
    <Tabs>
      {/* tab screens */}
    </Tabs>
  );
}

// app/(auth)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

export default function AuthLayout() {
  const { isAuthenticated } = useAuth();
  
  if (isAuthenticated) {
    return <Redirect href="/" />;
  }
  
  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="login" />
      <Stack.Screen name="register" />
    </Stack>
  );
}
```

### 7.2 Handling Deep Links with Auth

When a deep link arrives for an auth-protected route:

```tsx
// hooks/useDeepLinkRedirect.ts
import { useEffect, useRef } from 'react';
import { router, usePathname } from 'expo-router';
import { useAuth } from './useAuth';

export function useDeepLinkRedirect() {
  const { isAuthenticated } = useAuth();
  const pendingRedirect = useRef<string | null>(null);
  const pathname = usePathname();
  
  // Store the intended destination if not authenticated
  useEffect(() => {
    if (!isAuthenticated && pathname !== '/login' && pathname !== '/register') {
      pendingRedirect.current = pathname;
    }
  }, [isAuthenticated, pathname]);
  
  // Redirect after authentication
  useEffect(() => {
    if (isAuthenticated && pendingRedirect.current) {
      const destination = pendingRedirect.current;
      pendingRedirect.current = null;
      router.replace(destination);
    }
  }, [isAuthenticated]);
}
```

### 7.3 Role-Based Access

```tsx
// middleware/withAuth.tsx
import { Redirect } from 'expo-router';
import { useAuth } from '@/hooks/useAuth';

type Role = 'admin' | 'user' | 'viewer';

export function withAuth(
  WrappedComponent: React.ComponentType,
  requiredRoles?: Role[]
) {
  return function AuthenticatedComponent(props: any) {
    const { user, isAuthenticated } = useAuth();
    
    if (!isAuthenticated) {
      return <Redirect href="/login" />;
    }
    
    if (requiredRoles && !requiredRoles.includes(user!.role)) {
      return <Redirect href="/unauthorized" />;
    }
    
    return <WrappedComponent {...props} />;
  };
}

// Usage
// app/admin/dashboard.tsx
import { withAuth } from '@/middleware/withAuth';

function AdminDashboard() {
  return <DashboardContent />;
}

export default withAuth(AdminDashboard, ['admin']);
```

---

## 8. NAVIGATION STATE PERSISTENCE

For long-lived apps, persisting navigation state improves the user experience when the app is killed and restarted.

### 8.1 Basic State Persistence

```tsx
// app/_layout.tsx
import { useEffect, useState } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { NavigationContainer, InitialState } from '@react-navigation/native';

const NAVIGATION_STATE_KEY = 'NAVIGATION_STATE';

export default function RootLayout() {
  const [isReady, setIsReady] = useState(false);
  const [initialState, setInitialState] = useState<InitialState>();
  
  useEffect(() => {
    async function restoreState() {
      try {
        const savedState = await AsyncStorage.getItem(NAVIGATION_STATE_KEY);
        if (savedState) {
          setInitialState(JSON.parse(savedState));
        }
      } finally {
        setIsReady(true);
      }
    }
    
    if (!isReady) {
      restoreState();
    }
  }, [isReady]);
  
  if (!isReady) return <SplashScreen />;
  
  return (
    <Stack
      initialRouteName={initialState ? undefined : 'index'}
      screenListeners={{
        state: (e) => {
          // Save navigation state on every change
          AsyncStorage.setItem(
            NAVIGATION_STATE_KEY,
            JSON.stringify(e.data.state)
          );
        },
      }}
    >
      {/* screens */}
    </Stack>
  );
}
```

### 8.2 Selective Persistence

Don't persist everything — modals, auth screens, and transient states shouldn't be restored:

```tsx
function shouldPersistRoute(routeName: string): boolean {
  const NON_PERSISTENT_ROUTES = [
    'modal',
    'login',
    'register',
    'loading',
    'onboarding',
  ];
  return !NON_PERSISTENT_ROUTES.includes(routeName);
}

function cleanNavigationState(state: NavigationState): NavigationState {
  return {
    ...state,
    routes: state.routes
      .filter(route => shouldPersistRoute(route.name))
      .map(route => ({
        ...route,
        state: route.state ? cleanNavigationState(route.state) : undefined,
      })),
    index: Math.min(
      state.index,
      state.routes.filter(r => shouldPersistRoute(r.name)).length - 1
    ),
  };
}
```

---

## 9. ANALYTICS AND NAVIGATION

### 9.1 Screen Tracking

```tsx
// Expo Router: usePathname for screen tracking
import { usePathname, useSegments } from 'expo-router';
import { useEffect, useRef } from 'react';

function useScreenTracking() {
  const pathname = usePathname();
  const segments = useSegments();
  const previousPathname = useRef(pathname);
  
  useEffect(() => {
    if (pathname !== previousPathname.current) {
      analytics.screen(pathname, {
        segments,
        previousScreen: previousPathname.current,
        timestamp: Date.now(),
      });
      previousPathname.current = pathname;
    }
  }, [pathname, segments]);
}

// Use in root layout
export default function RootLayout() {
  useScreenTracking();
  return <Stack />;
}
```

### 9.2 Navigation Timing

```tsx
function useNavigationTiming() {
  const pathname = usePathname();
  const startTime = useRef<number>();
  
  useEffect(() => {
    if (startTime.current) {
      const duration = performance.now() - startTime.current;
      analytics.track('navigation_complete', {
        to: pathname,
        duration,
      });
    }
    startTime.current = performance.now();
  }, [pathname]);
}
```

---

## 10. CHAPTER SUMMARY

Navigation architecture is foundational — it affects every aspect of your app from performance to testing to analytics.

**The key decisions:**

1. **Use Expo Router v4** for new Expo projects. File-based routing, automatic deep linking, type generation, and web support are too valuable to pass up.

2. **Understand React Navigation 7** underneath. Expo Router is built on it, and you'll need its APIs for advanced patterns (state manipulation, custom navigators, complex auth flows).

3. **Lazy load aggressively.** Tabs should be lazy. Heavy screens should defer their content. Data should be prefetched before navigation, not after.

4. **Type your routes.** With Expo Router, this is automatic. With React Navigation, invest the time in a proper `ParamList` type hierarchy. A wrong route name should be a compile error, not a runtime crash.

5. **Design deep linking from day one.** Retrofit deep linking is painful. With Expo Router, your file structure IS your deep link configuration. Verify it works on both platforms.

6. **Auth routing is a layout concern.** Use `<Redirect>` in layout files, not imperative navigation in effects. This makes the auth flow declarative and easier to reason about.

7. **The standard pattern is Tab + Stack.** Tabs at the root, each tab with its own stack. Modals are presented from the root stack above the tabs. This covers 90% of apps.

8. **Measure navigation performance.** Track time from tap to screen interactive. Under 300ms is good, under 100ms is excellent. Use React Navigation's screen listeners and React Profiler to identify bottlenecks.

---

> **Next:** [Chapter 8: Styling & Animation] covers how to style React Native apps for production — from NativeWind to Reanimated to gesture-driven interactions.
