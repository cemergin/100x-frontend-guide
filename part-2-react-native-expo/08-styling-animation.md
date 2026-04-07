<!--
  CHAPTER: 8
  TITLE: Styling & Animation
  PART: II — React Native & Expo
  PREREQS: Chapter 1
  KEY_TOPICS: NativeWind, Tamagui, Unistyles, Reanimated v3, Gesture Handler, expo-image, design tokens
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 8: Styling & Animation

> **Part II — React Native & Expo** | Prerequisites: Chapter 1 | Difficulty: Intermediate

Here's the dirty secret of React Native styling: **`StyleSheet.create` is not enough.** It was fine for prototypes and simple apps, but the moment you need a design system, responsive layouts across phone and tablet, dark mode, platform-specific adjustments, and 60fps animations — you need more. A lot more.

The React Native styling ecosystem has exploded in the last two years. NativeWind brought Tailwind to mobile. Tamagui pushed the boundaries of cross-platform design systems. Reanimated v3 made worklet-based animations accessible. Gesture Handler gave us native-quality gesture recognition. And `expo-image` solved the image performance problem that plagued React Native since its inception.

But with more options comes more confusion. I've seen teams adopt NativeWind and then fight it for weeks because they didn't understand how it maps to React Native's layout system. I've seen apps with beautiful animations that drain the battery in an hour because every animation runs on the JS thread. I've seen design token systems that add 200ms to app startup because of runtime style computation.

This chapter is about making the right choices, understanding the trade-offs, and building a styling architecture that scales from MVP to millions of users. We'll cover every major styling solution, how animations actually work under the hood, and how to build a responsive design system for mobile.

### In This Chapter
- StyleSheet.create: The Foundation and Its Limits
- NativeWind: Tailwind CSS for React Native
- Tamagui: The Cross-Platform Design System Framework
- Unistyles: The New Contender
- Reanimated v3: Worklets, Shared Values, and Layout Animations
- Gesture Handler: Native-Quality Gesture Recognition
- expo-image: The Image Component You Should Be Using
- Design Tokens and Responsive Design for Mobile
- Choosing a Styling Solution: The Decision Framework

### Related Chapters
- [Ch 1: React Native Architecture & Internals] — how styling interacts with the native rendering pipeline
- [Ch 2: Browser Rendering & Web Fundamentals] — CSS concepts that carry over (and those that don't)
- [Ch 7: Navigation Architecture] — animating navigation transitions
- [Ch 13: Performance Optimization] — style and animation performance patterns

---

## 1. STYLESHEET.CREATE: THE FOUNDATION

Before we talk about libraries, you need to understand what React Native's built-in styling actually does.

### 1.1 How StyleSheet.create Works

```tsx
import { StyleSheet, View, Text } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#FFFFFF',
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#000000',
  },
});

function Screen() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Hello</Text>
    </View>
  );
}
```

**What `StyleSheet.create` actually does:** In the old architecture, it sent the style objects across the Bridge once, assigned them numeric IDs, and subsequent renders referenced styles by ID instead of sending the full objects. In the New Architecture with JSI, this optimization is less critical because there's no serialization overhead — but `StyleSheet.create` still provides:

1. **Validation at creation time** — catches invalid style properties early
2. **Object freezing** — prevents accidental mutation
3. **A clear declaration pattern** — styles are defined outside the component

### 1.2 The Flexbox Model in React Native

React Native uses Yoga (Meta's cross-platform Flexbox implementation) for layout. It's **almost** CSS Flexbox, but with critical differences:

| Property | CSS Default | React Native Default |
|----------|-------------|---------------------|
| `flexDirection` | `row` | `column` |
| `alignContent` | `stretch` | `flex-start` |
| `flexShrink` | `1` | `0` |
| `position` | `static` | `relative` |

**`flexDirection: 'column'` by default is the most important difference.** In web CSS, children flow horizontally by default. In React Native, they stack vertically. This matches mobile UI conventions where screens are vertical scrolling lists.

```tsx
// In React Native, this stacks children vertically (no flexDirection needed)
<View>
  <Text>First</Text>
  <Text>Second</Text>
  <Text>Third</Text>
</View>

// To go horizontal, explicitly set flexDirection
<View style={{ flexDirection: 'row' }}>
  <Text>Left</Text>
  <Text>Center</Text>
  <Text>Right</Text>
</View>
```

### 1.3 What React Native Does NOT Support

React Native's style system is a subset of CSS:

**Not supported:**
- Cascading (styles don't inherit from parent to child, except text within text)
- CSS Grid (use Flexbox, which handles most cases)
- Pseudo-classes (`:hover`, `:focus`, `:active`)
- Pseudo-elements (`::before`, `::after`)
- Media queries (use `useWindowDimensions` or a library)
- CSS animations / transitions (use Reanimated)
- `calc()`, `var()`, CSS custom properties
- Most shorthand properties (`margin: 10px 20px` — use `marginVertical`/`marginHorizontal`)
- `overflow: visible` on Android (clipping is the default)

**React Native-specific properties:**
- `elevation` (Android shadow)
- `shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius` (iOS shadow)
- `marginVertical`, `marginHorizontal`, `paddingVertical`, `paddingHorizontal`

### 1.4 The Limits of StyleSheet

```tsx
// Problem 1: No responsive styles
const styles = StyleSheet.create({
  container: {
    padding: 16, // Same on phone and tablet? Same in portrait and landscape?
  },
});

// Problem 2: No theme support
const styles = StyleSheet.create({
  text: {
    color: '#000000', // What about dark mode?
  },
});

// Problem 3: No design tokens
const styles = StyleSheet.create({
  card: {
    borderRadius: 8,  // Is this consistent with other cards?
    padding: 16,      // Is this the "medium" spacing?
    gap: 12,          // Is this the "small" gap?
  },
});

// Problem 4: Verbose for common patterns
const styles = StyleSheet.create({
  centered: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  // You end up with dozens of utility styles
});
```

These limitations are why the ecosystem has developed multiple styling solutions. Let's look at each.

---

## 2. NATIVEWIND: TAILWIND CSS FOR REACT NATIVE

NativeWind brings Tailwind CSS's utility-first approach to React Native. It uses Tailwind's compiler to transform class names into React Native styles.

### 2.1 How NativeWind Works

```tsx
import { View, Text, Pressable } from 'react-native';

function ProductCard({ product }) {
  return (
    <View className="bg-white dark:bg-gray-900 rounded-xl p-4 shadow-md">
      <Image
        source={{ uri: product.image }}
        className="w-full h-48 rounded-lg"
      />
      <Text className="text-lg font-bold mt-3 text-gray-900 dark:text-white">
        {product.name}
      </Text>
      <Text className="text-sm text-gray-500 dark:text-gray-400 mt-1">
        {product.description}
      </Text>
      <View className="flex-row justify-between items-center mt-4">
        <Text className="text-xl font-bold text-blue-600">
          ${product.price}
        </Text>
        <Pressable className="bg-blue-600 active:bg-blue-700 px-4 py-2 rounded-lg">
          <Text className="text-white font-semibold">Add to Cart</Text>
        </Pressable>
      </View>
    </View>
  );
}
```

**Under the hood:** NativeWind v4 uses Tailwind CSS's compiler to generate a mapping from class names to React Native style objects. At build time, `className="bg-white p-4 rounded-xl"` is transformed into the equivalent of `style={{ backgroundColor: '#FFFFFF', padding: 16, borderRadius: 12 }}`.

### 2.2 NativeWind Setup

```bash
npx expo install nativewind tailwindcss
```

```javascript
// tailwind.config.js
module.exports = {
  content: [
    './app/**/*.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#EFF6FF',
          100: '#DBEAFE',
          200: '#BFDBFE',
          500: '#3B82F6',
          600: '#2563EB',
          700: '#1D4ED8',
          900: '#1E3A8A',
        },
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem',
      },
    },
  },
  plugins: [],
};
```

```javascript
// babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
  };
};
```

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 2.3 NativeWind Features

**Dark mode:**
```tsx
// Automatically switches based on system appearance
<View className="bg-white dark:bg-gray-900">
  <Text className="text-black dark:text-white">
    Adapts to system theme
  </Text>
</View>

// Or programmatic control
import { useColorScheme } from 'nativewind';

function ThemeToggle() {
  const { colorScheme, toggleColorScheme } = useColorScheme();
  return (
    <Pressable onPress={toggleColorScheme}>
      <Text>{colorScheme === 'dark' ? 'Light Mode' : 'Dark Mode'}</Text>
    </Pressable>
  );
}
```

**Responsive design:**
```tsx
// NativeWind supports breakpoints based on window width
<View className="flex-col md:flex-row lg:gap-8">
  <View className="w-full md:w-1/3">
    <Sidebar />
  </View>
  <View className="w-full md:w-2/3">
    <MainContent />
  </View>
</View>
```

**Platform-specific styles:**
```tsx
<View className="p-4 ios:p-6 android:p-3">
  <Text className="text-base ios:text-lg">
    Platform-specific styling
  </Text>
</View>
```

**State variants:**
```tsx
<Pressable className="bg-blue-500 active:bg-blue-700 disabled:opacity-50">
  <Text className="text-white">Press Me</Text>
</Pressable>

// Hover (web only, does nothing on mobile)
<Pressable className="hover:bg-gray-100 active:bg-gray-200">
  <Text>Hover and Press</Text>
</Pressable>
```

### 2.4 NativeWind Trade-offs

**Advantages:**
- Familiar to web developers who know Tailwind
- Rapid prototyping — less context switching between files
- Built-in dark mode, responsive breakpoints, platform variants
- Small runtime overhead (styles are resolved at build time)
- Web + mobile consistency (Tailwind on web, NativeWind on mobile)

**Disadvantages:**
- Long class strings can be hard to read
- Not all Tailwind utilities work in React Native
- IDE autocomplete requires Tailwind extension
- Custom animations still need Reanimated
- Can be confusing when RN layout differs from CSS (e.g., flexDirection)

### 2.5 When to Choose NativeWind

Choose NativeWind when:
- Your team already knows Tailwind CSS
- You're building a universal app (web + mobile)
- You want rapid prototyping speed
- You value co-location of styles and markup
- Your design system maps well to utility classes

Avoid NativeWind when:
- You need highly dynamic, computation-based styles
- Your team prefers traditional StyleSheet patterns
- You're building a component library for other teams (utility classes leak to consumers)

---

## 3. TAMAGUI: THE CROSS-PLATFORM DESIGN SYSTEM

Tamagui is a styling and UI framework that optimizes styles at compile time and works across React Native and web.

### 3.1 How Tamagui Works

Tamagui's key innovation is **compile-time optimization.** It analyzes your styled components at build time and extracts the styles into static CSS (for web) or precomputed StyleSheet calls (for native). Dynamic styles that depend on props are kept as runtime styles.

```tsx
import { styled, View, Text, Button } from 'tamagui';

const Card = styled(View, {
  backgroundColor: '$background',
  borderRadius: '$4',
  padding: '$4',
  shadowColor: '$shadowColor',
  shadowRadius: 8,
  shadowOpacity: 0.1,
  
  // Variants
  variants: {
    size: {
      small: { padding: '$2' },
      medium: { padding: '$4' },
      large: { padding: '$6' },
    },
    elevated: {
      true: {
        shadowRadius: 16,
        shadowOpacity: 0.2,
      },
    },
  } as const,
  
  // Default variant values
  defaultVariants: {
    size: 'medium',
  },
});

// Usage
function ProductCard({ product }) {
  return (
    <Card size="large" elevated>
      <Text fontSize="$6" fontWeight="700" color="$color">
        {product.name}
      </Text>
      <Text fontSize="$3" color="$gray10">
        {product.description}
      </Text>
    </Card>
  );
}
```

### 3.2 Tamagui Tokens and Themes

```typescript
// tamagui.config.ts
import { createTamagui, createTokens } from 'tamagui';

const tokens = createTokens({
  size: {
    0: 0,
    1: 4,
    2: 8,
    3: 12,
    4: 16,
    5: 20,
    6: 24,
    8: 32,
    10: 40,
    12: 48,
    true: 16, // default
  },
  space: {
    0: 0,
    1: 4,
    2: 8,
    3: 12,
    4: 16,
    5: 20,
    6: 24,
    8: 32,
    true: 16,
  },
  radius: {
    0: 0,
    1: 4,
    2: 8,
    3: 12,
    4: 16,
    true: 8,
  },
  color: {
    white: '#FFFFFF',
    black: '#000000',
    gray1: '#F8F9FA',
    gray5: '#ADB5BD',
    gray10: '#212529',
    blue5: '#3B82F6',
    blue6: '#2563EB',
    blue7: '#1D4ED8',
    red5: '#EF4444',
  },
});

const lightTheme = {
  background: tokens.color.white,
  color: tokens.color.gray10,
  borderColor: tokens.color.gray1,
  shadowColor: tokens.color.black,
  accentBackground: tokens.color.blue5,
  accentColor: tokens.color.white,
};

const darkTheme = {
  background: tokens.color.gray10,
  color: tokens.color.white,
  borderColor: tokens.color.gray5,
  shadowColor: tokens.color.black,
  accentBackground: tokens.color.blue6,
  accentColor: tokens.color.white,
};

const config = createTamagui({
  tokens,
  themes: {
    light: lightTheme,
    dark: darkTheme,
  },
  media: {
    sm: { maxWidth: 640 },
    md: { maxWidth: 768 },
    lg: { maxWidth: 1024 },
    xl: { maxWidth: 1280 },
    short: { maxHeight: 680 },
    tall: { minHeight: 1000 },
  },
});

export default config;
```

### 3.3 Tamagui Trade-offs

**Advantages:**
- Compile-time optimization (faster runtime)
- Type-safe tokens and themes
- Cross-platform (web + native) with optimal output for each
- Rich variant system
- Built-in responsive media queries
- Component library (buttons, inputs, sheets, etc.)

**Disadvantages:**
- Significant learning curve
- Large dependency footprint
- Build configuration complexity
- Opinionated about component structure
- Can be overkill for small projects

---

## 4. UNISTYLES: THE NEW CONTENDER

Unistyles (by jpudysz) is a newer styling library that provides a lightweight alternative with excellent performance.

### 4.1 How Unistyles Works

Unistyles processes styles at runtime using the C++ layer (JSI), avoiding the JavaScript thread for style computation. It provides a `StyleSheet.create`-like API with superpowers.

```tsx
import { createStyleSheet, useStyles } from 'react-native-unistyles';

const stylesheet = createStyleSheet((theme, runtime) => ({
  container: {
    flex: 1,
    backgroundColor: theme.colors.background,
    paddingTop: runtime.insets.top,
    paddingBottom: runtime.insets.bottom,
  },
  title: {
    fontSize: {
      xs: 18,
      sm: 20,
      md: 24,
    },
    fontWeight: '700',
    color: theme.colors.text,
  },
  card: {
    backgroundColor: theme.colors.card,
    borderRadius: theme.radius.md,
    padding: theme.spacing.md,
    marginHorizontal: theme.spacing.md,
    marginBottom: theme.spacing.sm,
  },
}));

function ProductList() {
  const { styles } = useStyles(stylesheet);
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Products</Text>
      {products.map(p => (
        <View key={p.id} style={styles.card}>
          <Text>{p.name}</Text>
        </View>
      ))}
    </View>
  );
}
```

### 4.2 Unistyles Themes and Breakpoints

```typescript
// unistyles.config.ts
import { UnistylesRegistry } from 'react-native-unistyles';

const lightTheme = {
  colors: {
    background: '#FFFFFF',
    text: '#1A1A1A',
    card: '#F5F5F5',
    primary: '#3B82F6',
    secondary: '#8B5CF6',
    error: '#EF4444',
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
  },
  radius: {
    sm: 4,
    md: 8,
    lg: 16,
    full: 9999,
  },
};

const darkTheme = {
  colors: {
    background: '#1A1A1A',
    text: '#FFFFFF',
    card: '#2A2A2A',
    primary: '#60A5FA',
    secondary: '#A78BFA',
    error: '#F87171',
  },
  spacing: lightTheme.spacing,
  radius: lightTheme.radius,
};

type AppThemes = {
  light: typeof lightTheme;
  dark: typeof darkTheme;
};

declare module 'react-native-unistyles' {
  export interface UnistylesThemes extends AppThemes {}
}

UnistylesRegistry
  .addThemes({
    light: lightTheme,
    dark: darkTheme,
  })
  .addBreakpoints({
    xs: 0,
    sm: 380,
    md: 768,
    lg: 1024,
  })
  .addConfig({
    adaptiveThemes: true, // Follows system appearance
  });
```

### 4.3 Unistyles Trade-offs

**Advantages:**
- Familiar `StyleSheet.create` API (minimal learning curve)
- JSI-powered (fast, avoids JS thread for style computation)
- Built-in breakpoints, themes, and safe area insets
- Lightweight (small bundle size)
- TypeScript-first with full type inference

**Disadvantages:**
- Smaller ecosystem than NativeWind or Tamagui
- No compile-time optimization (runtime only)
- Native-only (no web support currently)
- Newer library, smaller community

---

## 5. STYLING SOLUTION COMPARISON

| Feature | StyleSheet | NativeWind | Tamagui | Unistyles |
|---------|-----------|------------|---------|-----------|
| Learning curve | None | Low (if Tailwind) | High | Low |
| Dark mode | Manual | Built-in | Built-in | Built-in |
| Responsive | Manual | Built-in | Built-in | Built-in |
| Tokens/Themes | No | Via config | Built-in | Built-in |
| Web support | N/A | Yes | Yes | No |
| Compile-time optimization | No | Yes | Yes | No |
| Bundle size impact | None | Small | Medium | Small |
| Performance | Baseline | Good | Excellent (compiled) | Excellent (JSI) |
| TypeScript DX | Basic | Good | Excellent | Excellent |
| Variant system | No | Via config | Built-in | Manual |

**The recommendation by project type:**

| Project Type | Recommendation |
|-------------|----------------|
| Quick prototype | NativeWind or StyleSheet |
| Production mobile app | Unistyles or NativeWind |
| Universal app (web + mobile) | NativeWind or Tamagui |
| Design system / Component library | Tamagui |
| Performance-critical app | Unistyles |
| Team knows Tailwind | NativeWind |

---

## 6. REANIMATED V3: THE ANIMATION ENGINE

React Native Reanimated is the standard for performant animations in React Native. It runs animation logic on the **UI thread** using worklets, completely bypassing the JavaScript thread.

### 6.1 The Architecture

```
Without Reanimated:
  JS Thread: Calculate animation value → Bridge/JSI → UI Thread: Apply style
  (Every frame crosses thread boundaries)

With Reanimated:
  JS Thread: Define animation (once)
  UI Thread: Calculate + Apply every frame (via worklet)
  (No thread crossing during animation)
```

**Worklets** are JavaScript functions that execute on the UI thread. They're compiled to a format that Reanimated's C++ runtime can execute directly, without the JavaScript engine.

### 6.2 Shared Values

Shared values are the communication mechanism between the JS thread and the UI thread:

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

function AnimatedCard() {
  // Shared value — accessible from both JS and UI threads
  const scale = useSharedValue(1);
  const opacity = useSharedValue(1);
  
  // Animated style — runs on the UI thread
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    opacity: opacity.value,
  }));
  
  function handlePressIn() {
    // Update from JS thread — Reanimated syncs to UI thread
    scale.value = withSpring(0.95, {
      damping: 15,
      stiffness: 150,
    });
    opacity.value = withTiming(0.8, { duration: 100 });
  }
  
  function handlePressOut() {
    scale.value = withSpring(1);
    opacity.value = withTiming(1, { duration: 100 });
  }
  
  return (
    <Pressable onPressIn={handlePressIn} onPressOut={handlePressOut}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Press me!</Text>
      </Animated.View>
    </Pressable>
  );
}
```

### 6.3 Animation Types

**Spring animations (natural feel):**
```tsx
// Spring with configuration
scale.value = withSpring(1, {
  damping: 15,        // How quickly it settles (higher = less bounce)
  stiffness: 150,     // How "tight" the spring is (higher = faster)
  mass: 1,            // Weight of the object
  overshootClamping: false,  // Allow overshooting target
  restDisplacementThreshold: 0.01,
  restSpeedThreshold: 2,
});
```

**Timing animations (precise control):**
```tsx
// Timing with easing
opacity.value = withTiming(0, {
  duration: 300,
  easing: Easing.bezier(0.25, 0.1, 0.25, 1), // Ease-out
});
```

**Decay animations (physics-based, like a scroll flick):**
```tsx
// Continues moving based on initial velocity, then decelerates
translateX.value = withDecay({
  velocity: velocityFromGesture,
  deceleration: 0.998, // How quickly it slows down
  clamp: [-200, 200],  // Boundaries
});
```

**Sequence and parallel:**
```tsx
// Sequential animations
opacity.value = withSequence(
  withTiming(0, { duration: 200 }),
  withTiming(1, { duration: 200 }),
  withTiming(0, { duration: 200 }),
  withTiming(1, { duration: 200 }),
);

// With delay
scale.value = withDelay(300, withSpring(1.2));
```

**Repeat:**
```tsx
// Repeat animation
rotation.value = withRepeat(
  withTiming(2 * Math.PI, { duration: 1000, easing: Easing.linear }),
  -1, // Infinite
  false // Don't reverse
);
```

### 6.4 Layout Animations

Reanimated v3 includes layout animations that automatically animate elements when they enter, exit, or change position in a list:

```tsx
import Animated, {
  FadeIn,
  FadeOut,
  Layout,
  SlideInRight,
  SlideOutLeft,
  LinearTransition,
} from 'react-native-reanimated';

function TodoList({ items, onRemove }) {
  return (
    <View>
      {items.map(item => (
        <Animated.View
          key={item.id}
          entering={FadeIn.duration(300).delay(100)}
          exiting={SlideOutLeft.duration(200)}
          layout={LinearTransition.springify().damping(15)}
          style={styles.todoItem}
        >
          <Text>{item.text}</Text>
          <Pressable onPress={() => onRemove(item.id)}>
            <Text>Remove</Text>
          </Pressable>
        </Animated.View>
      ))}
    </View>
  );
}
```

**Built-in entering animations:**
- `FadeIn`, `FadeInDown`, `FadeInUp`, `FadeInLeft`, `FadeInRight`
- `SlideInLeft`, `SlideInRight`, `SlideInUp`, `SlideInDown`
- `ZoomIn`, `ZoomInRotate`
- `BounceIn`, `BounceInDown`, `BounceInUp`
- `FlipInEasyX`, `FlipInEasyY`

Each has a corresponding exiting variant (`FadeOut`, `SlideOutLeft`, etc.) and can be customized:

```tsx
// Custom entering animation
const CustomEnter = FadeIn
  .delay(200)
  .duration(400)
  .springify()
  .damping(12);
```

### 6.5 `useAnimatedScrollHandler`

For scroll-linked animations that run entirely on the UI thread:

```tsx
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedStyle,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

function ParallaxHeader() {
  const scrollY = useSharedValue(0);
  
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });
  
  const headerStyle = useAnimatedStyle(() => ({
    height: interpolate(
      scrollY.value,
      [0, 200],
      [300, 100],
      Extrapolation.CLAMP
    ),
    opacity: interpolate(
      scrollY.value,
      [0, 150],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));
  
  const titleStyle = useAnimatedStyle(() => ({
    transform: [
      {
        translateY: interpolate(
          scrollY.value,
          [0, 200],
          [0, -50],
          Extrapolation.CLAMP
        ),
      },
      {
        scale: interpolate(
          scrollY.value,
          [0, 200],
          [1, 0.8],
          Extrapolation.CLAMP
        ),
      },
    ],
  }));
  
  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.header, headerStyle]}>
        <Animated.Text style={[styles.title, titleStyle]}>
          Profile
        </Animated.Text>
      </Animated.View>
      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
      >
        {/* Content */}
      </Animated.ScrollView>
    </View>
  );
}
```

### 6.6 `interpolate` and `interpolateColor`

```tsx
// Number interpolation
const opacity = interpolate(
  scrollY.value,
  [0, 100, 200],        // Input range
  [1, 0.5, 0],          // Output range
  Extrapolation.CLAMP   // Don't go beyond bounds
);

// Color interpolation
import { interpolateColor } from 'react-native-reanimated';

const backgroundColor = interpolateColor(
  scrollY.value,
  [0, 200],
  ['#FFFFFF', '#000000']
);
```

### 6.7 Reanimated Performance Tips

**1. Keep worklets pure:**
```tsx
// BAD: Accessing JS-thread values in a worklet
const someJSValue = getSomething(); // JS thread
useAnimatedStyle(() => ({
  opacity: someJSValue, // Can't reliably access JS values in worklet
}));

// GOOD: Use shared values
const opacity = useSharedValue(1);
useAnimatedStyle(() => ({
  opacity: opacity.value, // Shared value is accessible on UI thread
}));
```

**2. Avoid creating objects in animated styles:**
```tsx
// BAD: Creates new transform array every frame
useAnimatedStyle(() => ({
  transform: [{ translateX: x.value }, { translateY: y.value }],
}));

// This is actually fine — Reanimated handles this efficiently
// The worklet is compiled to avoid allocations
// But avoid complex computations inside the worklet
```

**3. Use `cancelAnimation` for cleanup:**
```tsx
useEffect(() => {
  opacity.value = withRepeat(withTiming(0, { duration: 1000 }), -1, true);
  
  return () => {
    cancelAnimation(opacity);
  };
}, []);
```

---

## 7. GESTURE HANDLER: NATIVE-QUALITY GESTURES

React Native Gesture Handler provides gesture recognition that runs on the native thread, not the JS thread.

### 7.1 Basic Gestures

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);
  
  const panGesture = Gesture.Pan()
    .onStart(() => {
      scale.value = withSpring(1.05);
    })
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY;
    })
    .onEnd(() => {
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
      scale.value = withSpring(1);
    });
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));
  
  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Drag me!</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 7.2 Composing Gestures

```tsx
function ZoomableImage({ source }) {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const savedTranslateX = useSharedValue(0);
  const savedTranslateY = useSharedValue(0);
  
  // Pinch to zoom
  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = savedScale.value * event.scale;
    })
    .onEnd(() => {
      if (scale.value < 1) {
        scale.value = withSpring(1);
        savedScale.value = 1;
      } else {
        savedScale.value = scale.value;
      }
    });
  
  // Pan to move (only when zoomed in)
  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      if (scale.value > 1) {
        translateX.value = savedTranslateX.value + event.translationX;
        translateY.value = savedTranslateY.value + event.translationY;
      }
    })
    .onEnd(() => {
      savedTranslateX.value = translateX.value;
      savedTranslateY.value = translateY.value;
    });
  
  // Double tap to zoom
  const doubleTapGesture = Gesture.Tap()
    .numberOfTaps(2)
    .onStart(() => {
      if (scale.value > 1) {
        scale.value = withSpring(1);
        savedScale.value = 1;
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        savedTranslateX.value = 0;
        savedTranslateY.value = 0;
      } else {
        scale.value = withSpring(2);
        savedScale.value = 2;
      }
    });
  
  // Compose: pinch and pan run simultaneously, double tap is exclusive
  const composed = Gesture.Simultaneous(
    pinchGesture,
    panGesture
  );
  const gesture = Gesture.Exclusive(doubleTapGesture, composed);
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));
  
  return (
    <GestureDetector gesture={gesture}>
      <Animated.Image source={source} style={[styles.image, animatedStyle]} />
    </GestureDetector>
  );
}
```

### 7.3 Gesture Composition Strategies

```tsx
// Simultaneous: Both gestures run at the same time
Gesture.Simultaneous(gestureA, gestureB);

// Exclusive: First gesture that activates wins, others are cancelled
Gesture.Exclusive(gestureA, gestureB);

// Race: First gesture to start activating wins
Gesture.Race(gestureA, gestureB);

// Manual activation control
const panGesture = Gesture.Pan()
  .activateAfterLongPress(500)  // Requires long press before pan activates
  .minDistance(10)               // Minimum distance before activation
  .activeOffsetX([-20, 20])     // Activate when horizontal movement exceeds 20
  .failOffsetY([-5, 5]);        // Fail if vertical movement exceeds 5
```

### 7.4 Swipe-to-Delete Pattern

```tsx
function SwipeableRow({ item, onDelete }) {
  const translateX = useSharedValue(0);
  const itemHeight = useSharedValue(70);
  const opacity = useSharedValue(1);
  
  const DELETE_THRESHOLD = -100;
  
  const panGesture = Gesture.Pan()
    .activeOffsetX([-10, 10])
    .failOffsetY([-5, 5])
    .onUpdate((event) => {
      translateX.value = Math.min(0, event.translationX);
    })
    .onEnd(() => {
      if (translateX.value < DELETE_THRESHOLD) {
        // Trigger delete animation
        translateX.value = withTiming(-400, { duration: 200 });
        itemHeight.value = withTiming(0, { duration: 200 });
        opacity.value = withTiming(0, { duration: 200 }, () => {
          runOnJS(onDelete)(item.id);
        });
      } else {
        // Snap back
        translateX.value = withSpring(0);
      }
    });
  
  const rowStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
    height: itemHeight.value,
    opacity: opacity.value,
  }));
  
  const deleteButtonStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [-100, 0],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));
  
  return (
    <View style={styles.rowContainer}>
      <Animated.View style={[styles.deleteButton, deleteButtonStyle]}>
        <Text style={styles.deleteText}>Delete</Text>
      </Animated.View>
      <GestureDetector gesture={panGesture}>
        <Animated.View style={[styles.row, rowStyle]}>
          <Text>{item.text}</Text>
        </Animated.View>
      </GestureDetector>
    </View>
  );
}
```

---

## 8. EXPO-IMAGE: THE IMAGE COMPONENT YOU SHOULD BE USING

`expo-image` is a drop-in replacement for React Native's `Image` component, built on top of platform-specific high-performance image loaders (SDWebImage on iOS, Glide on Android).

### 8.1 Why expo-image Over Image

| Feature | RN Image | expo-image |
|---------|----------|------------|
| Caching | Basic, in-memory | Disk + memory, cross-session |
| Formats | JPEG, PNG, GIF | JPEG, PNG, GIF, WebP, AVIF, SVG, animated |
| Blurhash/Thumbhash | No | Yes (placeholder while loading) |
| Transitions | No | Built-in fade, cross-dissolve |
| Recycling | No | Yes (memory efficient in lists) |
| Content fit | `resizeMode` | CSS-like `contentFit` + `contentPosition` |
| Performance | Adequate | Significantly better in lists |

### 8.2 Basic Usage

```tsx
import { Image } from 'expo-image';

function Avatar({ uri, size = 48 }) {
  return (
    <Image
      source={uri}
      style={{ width: size, height: size, borderRadius: size / 2 }}
      contentFit="cover"
      placeholder={{ blurhash: 'LGF5]+Yk^6#M@-5c,1J5@[or[Q6.' }}
      transition={200}
      recyclingKey={uri}
    />
  );
}
```

### 8.3 Blurhash and Thumbhash Placeholders

```tsx
// Generate blurhash on the server when processing uploads
// Then store it alongside the image URL

function ProductImage({ product }) {
  return (
    <Image
      source={product.imageUrl}
      style={styles.productImage}
      contentFit="cover"
      placeholder={{ blurhash: product.imageBlurhash }}
      transition={300}
    />
  );
}

// ThumbHash (newer, more accurate than BlurHash)
function ProductImage({ product }) {
  return (
    <Image
      source={product.imageUrl}
      style={styles.productImage}
      contentFit="cover"
      placeholder={{ thumbhash: product.imageThumbhash }}
      transition={300}
    />
  );
}
```

### 8.4 expo-image in Lists

```tsx
import { Image } from 'expo-image';
import { FlashList } from '@shopify/flash-list';

function ImageGrid({ images }) {
  return (
    <FlashList
      data={images}
      numColumns={3}
      estimatedItemSize={130}
      renderItem={({ item }) => (
        <Image
          source={item.url}
          style={styles.gridImage}
          contentFit="cover"
          placeholder={{ blurhash: item.blurhash }}
          recyclingKey={item.id}  // Critical for list performance
          transition={200}
        />
      )}
    />
  );
}
```

**`recyclingKey`** tells `expo-image` to recycle the native image view when the item changes. This is critical in lists — without it, the image component is destroyed and recreated for each item, causing visible flicker and wasted memory.

---

## 9. DESIGN TOKENS AND RESPONSIVE DESIGN

### 9.1 Building a Token System

Design tokens are the atomic values of your design system — colors, spacing, typography, shadows, radii. They bridge the gap between design and code.

```typescript
// tokens/spacing.ts
export const spacing = {
  '2xs': 2,
  xs: 4,
  sm: 8,
  md: 16,
  lg: 24,
  xl: 32,
  '2xl': 48,
  '3xl': 64,
} as const;

// tokens/colors.ts
export const palette = {
  // Neutral
  gray50: '#F9FAFB',
  gray100: '#F3F4F6',
  gray200: '#E5E7EB',
  gray300: '#D1D5DB',
  gray400: '#9CA3AF',
  gray500: '#6B7280',
  gray600: '#4B5563',
  gray700: '#374151',
  gray800: '#1F2937',
  gray900: '#111827',
  gray950: '#030712',
  
  // Primary
  blue50: '#EFF6FF',
  blue100: '#DBEAFE',
  blue500: '#3B82F6',
  blue600: '#2563EB',
  blue700: '#1D4ED8',
  blue900: '#1E3A8A',
  
  // Semantic
  white: '#FFFFFF',
  black: '#000000',
} as const;

export const lightColors = {
  background: palette.white,
  surface: palette.gray50,
  text: palette.gray900,
  textSecondary: palette.gray500,
  textTertiary: palette.gray400,
  border: palette.gray200,
  primary: palette.blue600,
  primaryText: palette.white,
  error: '#EF4444',
  success: '#22C55E',
  warning: '#F59E0B',
};

export const darkColors = {
  background: palette.gray950,
  surface: palette.gray900,
  text: palette.white,
  textSecondary: palette.gray400,
  textTertiary: palette.gray500,
  border: palette.gray800,
  primary: palette.blue500,
  primaryText: palette.white,
  error: '#F87171',
  success: '#4ADE80',
  warning: '#FBBF24',
};

// tokens/typography.ts
export const typography = {
  displayLarge: {
    fontSize: 32,
    lineHeight: 40,
    fontWeight: '700' as const,
    letterSpacing: -0.5,
  },
  displayMedium: {
    fontSize: 28,
    lineHeight: 36,
    fontWeight: '700' as const,
    letterSpacing: -0.3,
  },
  headlineLarge: {
    fontSize: 24,
    lineHeight: 32,
    fontWeight: '600' as const,
  },
  headlineMedium: {
    fontSize: 20,
    lineHeight: 28,
    fontWeight: '600' as const,
  },
  bodyLarge: {
    fontSize: 16,
    lineHeight: 24,
    fontWeight: '400' as const,
  },
  bodyMedium: {
    fontSize: 14,
    lineHeight: 20,
    fontWeight: '400' as const,
  },
  bodySmall: {
    fontSize: 12,
    lineHeight: 16,
    fontWeight: '400' as const,
  },
  labelLarge: {
    fontSize: 14,
    lineHeight: 20,
    fontWeight: '600' as const,
  },
  labelMedium: {
    fontSize: 12,
    lineHeight: 16,
    fontWeight: '600' as const,
  },
};

// tokens/shadows.ts
import { Platform } from 'react-native';

export const shadows = {
  sm: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.05,
      shadowRadius: 2,
    },
    android: {
      elevation: 2,
    },
  }),
  md: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.1,
      shadowRadius: 8,
    },
    android: {
      elevation: 4,
    },
  }),
  lg: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 8 },
      shadowOpacity: 0.15,
      shadowRadius: 16,
    },
    android: {
      elevation: 8,
    },
  }),
};
```

### 9.2 Theme Provider

```tsx
// theme/ThemeContext.tsx
import { createContext, useContext, useMemo } from 'react';
import { useColorScheme } from 'react-native';
import { lightColors, darkColors } from '@/tokens/colors';
import { spacing } from '@/tokens/spacing';
import { typography } from '@/tokens/typography';
import { shadows } from '@/tokens/shadows';

const ThemeContext = createContext<Theme | null>(null);

interface Theme {
  colors: typeof lightColors;
  spacing: typeof spacing;
  typography: typeof typography;
  shadows: typeof shadows;
  isDark: boolean;
}

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const colorScheme = useColorScheme();
  const isDark = colorScheme === 'dark';
  
  const theme = useMemo<Theme>(() => ({
    colors: isDark ? darkColors : lightColors,
    spacing,
    typography,
    shadows,
    isDark,
  }), [isDark]);
  
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme(): Theme {
  const theme = useContext(ThemeContext);
  if (!theme) throw new Error('useTheme must be used within ThemeProvider');
  return theme;
}
```

### 9.3 Responsive Design for Mobile

Mobile responsive design is different from web responsive design. On web, you respond to viewport width. On mobile, you respond to:
- **Device form factor** (phone vs. tablet)
- **Orientation** (portrait vs. landscape)
- **Dynamic Type / font scaling** (accessibility)
- **Safe areas** (notch, home indicator, status bar)

```tsx
import { useWindowDimensions, PixelRatio } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function useResponsive() {
  const { width, height } = useWindowDimensions();
  const insets = useSafeAreaInsets();
  const fontScale = PixelRatio.getFontScale();
  
  const isTablet = width >= 768;
  const isLandscape = width > height;
  const isLargeFont = fontScale > 1.2;
  
  return {
    width,
    height,
    insets,
    fontScale,
    isTablet,
    isLandscape,
    isLargeFont,
    // Responsive spacing
    horizontalPadding: isTablet ? 32 : 16,
    // Content width (with max-width on tablet)
    contentWidth: isTablet ? Math.min(width - 64, 720) : width - 32,
    // Grid columns
    gridColumns: isTablet ? (isLandscape ? 4 : 3) : 2,
  };
}

function ProductGrid() {
  const { gridColumns, contentWidth, horizontalPadding } = useResponsive();
  const itemWidth = (contentWidth - (gridColumns - 1) * 12) / gridColumns;
  
  return (
    <View style={{ paddingHorizontal: horizontalPadding }}>
      <FlashList
        data={products}
        numColumns={gridColumns}
        estimatedItemSize={itemWidth + 100}
        renderItem={({ item }) => (
          <ProductCard product={item} width={itemWidth} />
        )}
      />
    </View>
  );
}
```

### 9.4 Safe Areas

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Screen({ children }) {
  const insets = useSafeAreaInsets();
  
  return (
    <View
      style={{
        flex: 1,
        paddingTop: insets.top,
        paddingBottom: insets.bottom,
        paddingLeft: insets.left,
        paddingRight: insets.right,
      }}
    >
      {children}
    </View>
  );
}

// For scroll views, apply insets differently
function ScrollScreen({ children }) {
  const insets = useSafeAreaInsets();
  
  return (
    <ScrollView
      contentContainerStyle={{
        paddingTop: insets.top,
        paddingBottom: insets.bottom + 20, // Extra space at bottom
      }}
      scrollIndicatorInsets={{ bottom: insets.bottom }}
    >
      {children}
    </ScrollView>
  );
}
```

---

## 10. PLATFORM-SPECIFIC STYLING PATTERNS

### 10.1 Shadows

Shadows work differently on iOS and Android:

```tsx
// Platform-unified shadow helper
function createShadow(elevation: number) {
  return Platform.select({
    ios: {
      shadowColor: '#000000',
      shadowOffset: {
        width: 0,
        height: Math.round(elevation * 0.5),
      },
      shadowOpacity: 0.05 + elevation * 0.015,
      shadowRadius: elevation * 1.5,
    },
    android: {
      elevation,
    },
    default: {}, // Web fallback
  });
}

const styles = StyleSheet.create({
  card: {
    backgroundColor: 'white',
    borderRadius: 12,
    padding: 16,
    ...createShadow(4),
  },
});
```

### 10.2 Haptic Feedback

```tsx
import * as Haptics from 'expo-haptics';

function HapticButton({ onPress, children }) {
  function handlePress() {
    Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    onPress?.();
  }
  
  return (
    <Pressable onPress={handlePress}>
      {children}
    </Pressable>
  );
}

// Selection feedback (for toggles, checkboxes)
function Toggle({ value, onChange }) {
  function handleToggle() {
    Haptics.selectionAsync();
    onChange(!value);
  }
  
  return (
    <Pressable onPress={handleToggle}>
      {/* toggle UI */}
    </Pressable>
  );
}

// Notification feedback (for success/error states)
function SubmitButton({ onSubmit }) {
  async function handleSubmit() {
    try {
      await onSubmit();
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
    } catch {
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
    }
  }
}
```

### 10.3 StatusBar Styling

```tsx
import { StatusBar } from 'expo-status-bar';

function Screen({ lightContent = false, children }) {
  return (
    <>
      <StatusBar style={lightContent ? 'light' : 'dark'} />
      {children}
    </>
  );
}

// Per-screen status bar
function DarkScreen() {
  return (
    <View style={{ flex: 1, backgroundColor: '#000' }}>
      <StatusBar style="light" />
      {/* Dark background content */}
    </View>
  );
}
```

---

## 11. ANIMATION PATTERNS COOKBOOK

### 11.1 Skeleton Loading

```tsx
function Skeleton({ width, height, borderRadius = 4 }) {
  const opacity = useSharedValue(0.3);
  
  useEffect(() => {
    opacity.value = withRepeat(
      withTiming(1, { duration: 800 }),
      -1,
      true
    );
  }, []);
  
  const style = useAnimatedStyle(() => ({
    opacity: opacity.value,
  }));
  
  return (
    <Animated.View
      style={[
        {
          width,
          height,
          borderRadius,
          backgroundColor: '#E5E7EB',
        },
        style,
      ]}
    />
  );
}

function ProductCardSkeleton() {
  return (
    <View style={styles.card}>
      <Skeleton width="100%" height={200} borderRadius={8} />
      <View style={{ marginTop: 12 }}>
        <Skeleton width="70%" height={20} />
      </View>
      <View style={{ marginTop: 8 }}>
        <Skeleton width="40%" height={16} />
      </View>
    </View>
  );
}
```

### 11.2 Pull-to-Refresh Animation

```tsx
function CustomRefreshControl({ onRefresh }) {
  const translateY = useSharedValue(0);
  const isRefreshing = useSharedValue(false);
  const rotation = useSharedValue(0);
  
  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      if (event.translationY > 0 && !isRefreshing.value) {
        translateY.value = Math.min(event.translationY * 0.5, 100);
        rotation.value = (translateY.value / 100) * 360;
      }
    })
    .onEnd(() => {
      if (translateY.value > 60) {
        isRefreshing.value = true;
        translateY.value = withSpring(60);
        rotation.value = withRepeat(
          withTiming(rotation.value + 360, { duration: 800 }),
          -1,
          false
        );
        runOnJS(onRefresh)(() => {
          isRefreshing.value = false;
          translateY.value = withSpring(0);
          cancelAnimation(rotation);
        });
      } else {
        translateY.value = withSpring(0);
      }
    });
  
  const spinnerStyle = useAnimatedStyle(() => ({
    transform: [
      { translateY: translateY.value - 40 },
      { rotate: `${rotation.value}deg` },
    ],
    opacity: interpolate(translateY.value, [0, 40], [0, 1], Extrapolation.CLAMP),
  }));
  
  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={{ flex: 1 }}>
        <Animated.View style={[styles.spinner, spinnerStyle]}>
          {/* Spinner icon */}
        </Animated.View>
        {/* Content */}
      </Animated.View>
    </GestureDetector>
  );
}
```

### 11.3 Bottom Sheet

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  interpolate,
  Extrapolation,
  runOnJS,
} from 'react-native-reanimated';

function BottomSheet({ snapPoints = [0, 300, 600], children, onClose }) {
  const translateY = useSharedValue(0);
  const context = useSharedValue(0);
  const activeSnapPoint = useSharedValue(0);
  
  const maxTranslate = -snapPoints[snapPoints.length - 1];
  
  const panGesture = Gesture.Pan()
    .onStart(() => {
      context.value = translateY.value;
    })
    .onUpdate((event) => {
      translateY.value = Math.max(
        maxTranslate,
        Math.min(0, context.value + event.translationY)
      );
    })
    .onEnd((event) => {
      // Find nearest snap point
      const currentPos = -translateY.value;
      let nearestSnap = snapPoints[0];
      let minDistance = Math.abs(currentPos - snapPoints[0]);
      
      for (const snap of snapPoints) {
        const distance = Math.abs(currentPos - snap);
        if (distance < minDistance) {
          minDistance = distance;
          nearestSnap = snap;
        }
      }
      
      // Factor in velocity
      if (event.velocityY < -500 && activeSnapPoint.value < snapPoints.length - 1) {
        nearestSnap = snapPoints[activeSnapPoint.value + 1];
      } else if (event.velocityY > 500 && activeSnapPoint.value > 0) {
        nearestSnap = snapPoints[activeSnapPoint.value - 1];
      }
      
      activeSnapPoint.value = snapPoints.indexOf(nearestSnap);
      translateY.value = withSpring(-nearestSnap, {
        damping: 20,
        stiffness: 200,
      });
      
      if (nearestSnap === 0) {
        runOnJS(onClose)();
      }
    });
  
  const sheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));
  
  const backdropStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateY.value,
      [maxTranslate, 0],
      [0.5, 0],
      Extrapolation.CLAMP
    ),
  }));
  
  return (
    <>
      <Animated.View
        style={[styles.backdrop, backdropStyle]}
        pointerEvents={translateY.value < 0 ? 'auto' : 'none'}
      />
      <GestureDetector gesture={panGesture}>
        <Animated.View style={[styles.sheet, sheetStyle]}>
          <View style={styles.handle} />
          {children}
        </Animated.View>
      </GestureDetector>
    </>
  );
}

const styles = StyleSheet.create({
  backdrop: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'black',
  },
  sheet: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    height: 700,
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingHorizontal: 16,
  },
  handle: {
    width: 36,
    height: 4,
    borderRadius: 2,
    backgroundColor: '#D1D5DB',
    alignSelf: 'center',
    marginVertical: 12,
  },
});
```

---

## 12. CHAPTER SUMMARY

Styling and animation in React Native require a deliberate architectural approach. The right choices depend on your team, your app, and your constraints.

**The key decisions:**

1. **Choose your styling solution based on your team and project.** NativeWind for Tailwind teams and universal apps. Unistyles for performance-focused native apps. Tamagui for cross-platform design systems. StyleSheet for simple apps or when you want zero dependencies.

2. **Use Reanimated v3 for all animations.** The built-in `Animated` API should be considered legacy. Reanimated's worklet-based architecture runs animations on the UI thread, which means 60fps even when your JS thread is busy. Layout animations and entering/exiting animations are free performance wins.

3. **Use Gesture Handler for all gestures.** Native-thread gesture recognition means gestures respond in under 16ms. Compose gestures (simultaneous, exclusive, race) for complex interactions. Combine with Reanimated shared values for gesture-driven animations.

4. **Use expo-image instead of Image.** Disk caching, blurhash placeholders, image recycling in lists, and support for modern formats. The performance difference in image-heavy lists is dramatic.

5. **Build a design token system from day one.** Spacing, colors, typography, shadows, and radii should be tokens, not magic numbers. This ensures consistency and makes dark mode, responsive design, and design changes manageable.

6. **Responsive mobile design is not web responsive design.** Respond to device form factor (phone vs. tablet), orientation, dynamic type scaling, and safe areas — not just viewport width. Use `useWindowDimensions` and `useSafeAreaInsets`.

7. **Haptic feedback is part of styling.** It's the tactile equivalent of visual feedback. Use it for button presses (impact), toggles (selection), and success/error states (notification). It makes your app feel native.

8. **Shadows are platform-specific.** iOS uses `shadowColor/Offset/Opacity/Radius`. Android uses `elevation`. Build a helper that abstracts this. Or use a library like NativeWind or Tamagui that handles it.

The best mobile apps don't just look good — they *feel* good. That feeling comes from 60fps animations, responsive gestures, instant visual feedback, and haptic confirmation. This chapter gave you the tools to build it.

---

> **Next:** [Chapter 9: Data Fetching & Caching] covers how to get data into your app efficiently — TanStack Query, SWR, offline-first patterns, and cache architecture.
