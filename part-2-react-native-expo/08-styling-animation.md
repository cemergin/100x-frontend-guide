<!--
  CHAPTER: 9
  TITLE: Styling & Animation
  PART: II — React Native & Expo
  PREREQS: Chapter 1
  KEY_TOPICS: NativeWind, Tamagui, Unistyles, Reanimated v3, Gesture Handler, expo-image, design tokens
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 8: Styling & Animation

> **Part II — React Native & Expo** | Prerequisites: Chapter 1 | Difficulty: Intermediate

> "Design is not just what it looks like and feels like. Design is how it works." — Steve Jobs
>
> "Animation is how your app earns trust." — Every mobile designer who's watched a user flinch at a janky transition

---

<details>
<summary><strong>TL;DR</strong></summary>

- The styling decision you make in week one becomes the tax you pay every week after; choose NativeWind (Tailwind for RN), Tamagui (cross-platform with a compiler), or Unistyles (JSI-based) based on your team and platform needs
- Reanimated v3 runs animations on the UI thread via worklets, completely bypassing the JS thread; this is the only way to hit 60fps on complex gesture-driven animations
- Design tokens (color, spacing, typography) are the shared language between design and engineering; centralize them from day one or face a dark-mode retrofit nightmare
- Gesture Handler composes pan, pinch, rotation, and tap gestures declaratively; combine with Reanimated for interactions that feel truly native
- `expo-image` replaces `Image` with built-in caching, blurhash placeholders, and content-fit modes; use it for every image in your app

</details>

## The Problem Nobody Warns You About

Here's what happened to the last three React Native teams I consulted with. They started building. They used `StyleSheet.create` because that's what the docs show. They inline-styled a few things. They added some colors as constants. It worked.

Then the designer asked for dark mode. Then the PM wanted responsive layouts for tablets. Then someone said "can we animate that?" and someone else said "this needs to feel native." And suddenly the team was staring at a codebase where styles were scattered across 200 files, there was no consistent spacing scale, dark mode was a patchwork of ternaries, and animations were jittery `Animated.timing` calls running on the JS thread.

I've seen this story play out at startups and at Fortune 500 companies. The styling decision you make in week one becomes the tax you pay every week after.

This chapter gives you the information to make that decision well, and then goes deep on the options that actually work at scale: NativeWind, Tamagui, Unistyles, Reanimated, and Gesture Handler. We'll also cover `expo-image` (because image handling is half of mobile UI), design tokens, and responsive patterns.

### In This Chapter
- StyleSheet.create: When It's Enough (and When It Isn't)
- NativeWind v4: Tailwind for React Native
- Tamagui: Cross-Platform with a Compiler
- Unistyles: JSI-Based Styling
- The Decision Matrix: Choosing Your Styling Approach
- Design Tokens: The Shared Language
- Responsive Design for Mobile
- Reanimated v3: Worklets and Shared Values
- Layout Animations: Entering, Exiting, Layout Transitions
- Gesture Handler: Pan, Pinch, and Composing Gestures
- expo-image: Caching, Blurhash, and Performance

### Related Chapters
- [Ch 1: JavaScript & TypeScript Foundations] — language primitives for styling patterns
- [Ch 7: Navigation Architecture] — transition animations between screens
- [Ch 13: Performance Optimization] — rendering performance and animation frame budgets
- [Ch 3: The Rendering Pipeline] — how style changes trigger re-renders

---

## 1. STYLESHEET.CREATE: THE BASELINE

Before we talk about any library, you need to understand what you get out of the box.

### 1.1 What StyleSheet.create Actually Does

There's a persistent myth that `StyleSheet.create` "optimizes" your styles by sending them over the bridge once. That was partially true in the old architecture. In the New Architecture with JSI, the performance story has changed — but `StyleSheet.create` still gives you three things:

1. **Validation at creation time** — It catches invalid style properties early
2. **Object identity stability** — The style object reference doesn't change between renders
3. **A clear contract** — You're declaring "these are styles" rather than "this is some object"

```tsx
import { StyleSheet, View, Text } from 'react-native';

// This is fine. Don't let anyone tell you it's not.
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#FFFFFF',
  },
  title: {
    fontSize: 24,
    fontWeight: '700',
    color: '#1A1A1A',
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 16,
    fontWeight: '400',
    color: '#666666',
    lineHeight: 24,
  },
});

function Header({ title, subtitle }: { title: string; subtitle: string }) {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>{title}</Text>
      <Text style={styles.subtitle}>{subtitle}</Text>
    </View>
  );
}
```

### 1.2 When StyleSheet.create Is Enough

I'm going to say something that might be controversial in a chapter about styling libraries: **StyleSheet.create is enough for a lot of apps.**

Here's when you can stick with it:

- **Small team (1-3 devs)** who can maintain consistency through code review
- **No dark mode** requirement (or you're okay with a simple theme context)
- **No cross-platform** (RN only, no web target)
- **Simple responsive needs** (phone-only, no tablet support)
- **No design system** to implement

That's a meaningful subset of apps. If that's you, use `StyleSheet.create`, keep your colors in a constants file, and move on with your life. Don't adopt a library because a blog post told you to.

### 1.3 When StyleSheet.create Breaks Down

Here's where it falls apart:

```tsx
// The dark mode ternary hell
const styles = StyleSheet.create({
  container: {
    backgroundColor: isDark ? '#1A1A1A' : '#FFFFFF',  // Can't do this — isDark isn't available here
  },
});

// So you move to dynamic styles...
function Screen() {
  const isDark = useColorScheme() === 'dark';
  
  // Now your styles are created every render
  const styles = StyleSheet.create({
    container: {
      backgroundColor: isDark ? '#1A1A1A' : '#FFFFFF',
      padding: 16,
    },
    text: {
      color: isDark ? '#FFFFFF' : '#1A1A1A',
    },
  });
  
  return (
    <View style={styles.container}>
      <Text style={styles.text}>Hello</Text>
    </View>
  );
}

// Or you use the "function that returns styles" pattern
const createStyles = (isDark: boolean) => StyleSheet.create({
  container: {
    backgroundColor: isDark ? '#1A1A1A' : '#FFFFFF',
    padding: 16,
  },
  text: {
    color: isDark ? '#FFFFFF' : '#1A1A1A',
  },
});

// Slightly better, but still re-creates on every render unless you memoize.
// And now multiply this by 200 components.
```

Other pain points:
- **No breakpoints** — You have to roll your own `useWindowDimensions` wrapper for responsive layouts
- **No type-safe theming** — Your `colors.ts` file has no way to enforce that every component uses theme tokens instead of raw hex values
- **No platform variants** — Writing `Platform.select({})` everywhere gets old fast
- **Style composition is clunky** — Arrays of style objects are fine until you have 5 conditional styles

When you hit these walls, it's time to pick a library. Let's look at the three that matter.

---

## 2. NATIVEWIND V4: TAILWIND FOR REACT NATIVE

NativeWind brings Tailwind CSS to React Native. Not a "Tailwind-inspired" utility system — actual Tailwind CSS, compiled to React Native styles.

### 2.1 Why NativeWind Exists

If your team already knows Tailwind from web development, NativeWind is the fastest path to productive styling in React Native. The mental model transfers directly: utility classes, responsive prefixes, dark mode, the whole thing.

NativeWind v4 (the current version) is a complete rewrite from v2. It uses the Tailwind CSS compiler directly and supports the full Tailwind feature set. This is important — v2 was a subset. v4 is the real thing.

```tsx
import { View, Text, Pressable } from 'react-native';

// This is a real NativeWind component. Not pseudocode.
function ProductCard({ name, price, onPress }: ProductCardProps) {
  return (
    <Pressable 
      onPress={onPress}
      className="bg-white dark:bg-gray-900 rounded-2xl p-4 shadow-sm 
                 active:scale-95 transition-transform"
    >
      <View className="flex-row items-center justify-between">
        <Text className="text-lg font-semibold text-gray-900 dark:text-white">
          {name}
        </Text>
        <Text className="text-base font-medium text-emerald-600 dark:text-emerald-400">
          ${price}
        </Text>
      </View>
    </Pressable>
  );
}
```

### 2.2 Setting Up NativeWind v4

The setup is more involved than most RN libraries, but you do it once.

```bash
# Install NativeWind and its peer dependencies
npx expo install nativewind tailwindcss react-native-reanimated

# NativeWind v4 uses the Tailwind CSS PostCSS plugin
npm install --save-dev tailwindcss@3 postcss autoprefixer
```

**tailwind.config.js:**
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './app/**/*.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
    './components/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        // Your brand colors
        brand: {
          50: '#EEF2FF',
          100: '#E0E7FF',
          200: '#C7D2FE',
          300: '#A5B4FC',
          400: '#818CF8',
          500: '#6366F1',
          600: '#4F46E5',
          700: '#4338CA',
          800: '#3730A3',
          900: '#312E81',
          950: '#1E1B4B',
        },
      },
      fontFamily: {
        sans: ['Inter'],
        mono: ['JetBrainsMono'],
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

**babel.config.js:**
```javascript
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

**metro.config.js:**
```javascript
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);

module.exports = withNativeWind(config, { input: './global.css' });
```

**global.css:**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

**app/_layout.tsx:**
```tsx
import '../global.css';
import { Stack } from 'expo-router';

export default function RootLayout() {
  return <Stack />;
}
```

### 2.3 Platform Variants

This is one of NativeWind's killer features. You can write platform-specific styles inline:

```tsx
<View className="p-4 ios:p-6 android:p-3 web:max-w-2xl web:mx-auto">
  <Text className="text-base ios:text-lg android:text-sm">
    Platform-specific sizing
  </Text>
</View>
```

Under the hood, NativeWind evaluates these at compile time using `Platform.OS`. No runtime cost.

```tsx
// More platform variant examples
function SearchBar() {
  return (
    <View className="flex-row items-center bg-gray-100 dark:bg-gray-800 
                      rounded-xl px-4 h-12
                      ios:rounded-2xl ios:h-11
                      android:rounded-lg android:h-14">
      <SearchIcon className="w-5 h-5 text-gray-400 mr-3" />
      <TextInput
        placeholder="Search..."
        className="flex-1 text-base text-gray-900 dark:text-white
                   ios:text-[17px]
                   android:text-sm"
        placeholderTextColor="#9CA3AF"
      />
    </View>
  );
}
```

### 2.4 Dark Mode

NativeWind supports dark mode out of the box using Tailwind's `dark:` prefix. It hooks into React Native's `useColorScheme()` by default, but you can control it manually.

```tsx
import { useColorScheme } from 'nativewind';

function SettingsScreen() {
  const { colorScheme, setColorScheme, toggleColorScheme } = useColorScheme();

  return (
    <View className="flex-1 bg-white dark:bg-gray-950 p-4">
      <Text className="text-2xl font-bold text-gray-900 dark:text-white mb-6">
        Settings
      </Text>
      
      <View className="flex-row items-center justify-between 
                        bg-gray-50 dark:bg-gray-900 rounded-xl p-4">
        <Text className="text-base text-gray-700 dark:text-gray-300">
          Dark Mode
        </Text>
        <Switch
          value={colorScheme === 'dark'}
          onValueChange={toggleColorScheme}
          trackColor={{ false: '#D1D5DB', true: '#818CF8' }}
          thumbColor="#FFFFFF"
        />
      </View>

      {/* Force light mode for a specific section */}
      <View className="light mt-4 bg-white rounded-xl p-4">
        <Text className="text-gray-900">
          This section is always in light mode
        </Text>
      </View>
    </View>
  );
}
```

### 2.5 Responsive Breakpoints

NativeWind supports Tailwind's responsive breakpoints, but in the context of mobile, you'll mostly use them for tablet layouts:

```tsx
// Phone: single column. Tablet: two columns. Web: three columns.
function ProductGrid({ products }: { products: Product[] }) {
  return (
    <View className="flex-row flex-wrap p-4 gap-4">
      {products.map((product) => (
        <View
          key={product.id}
          className="w-full md:w-[48%] lg:w-[31%]"
        >
          <ProductCard product={product} />
        </View>
      ))}
    </View>
  );
}
```

Custom breakpoints for mobile-specific needs:

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      // Mobile-first breakpoints
      'sm': '380px',     // Larger phones (iPhone 15 Pro Max)
      'md': '768px',     // Tablets
      'lg': '1024px',    // iPad Pro / web
      'xl': '1280px',    // Large tablets / desktop web
    },
  },
};
```

### 2.6 Custom Utility Classes with @apply

For components that repeat the same utility combination, extract them:

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .card {
    @apply bg-white dark:bg-gray-900 rounded-2xl p-4 shadow-sm;
  }
  
  .card-title {
    @apply text-lg font-semibold text-gray-900 dark:text-white;
  }
  
  .btn-primary {
    @apply bg-brand-600 rounded-xl px-6 py-3 items-center justify-center
           active:bg-brand-700 active:scale-[0.98];
  }
  
  .btn-primary-text {
    @apply text-white font-semibold text-base;
  }
  
  .input-field {
    @apply bg-gray-50 dark:bg-gray-800 rounded-xl px-4 h-12
           text-base text-gray-900 dark:text-white
           border border-gray-200 dark:border-gray-700
           focus:border-brand-500 dark:focus:border-brand-400;
  }
}
```

```tsx
// Now your components are clean
function LoginForm() {
  return (
    <View className="card">
      <Text className="card-title mb-4">Sign In</Text>
      <TextInput className="input-field mb-3" placeholder="Email" />
      <TextInput className="input-field mb-4" placeholder="Password" secureTextEntry />
      <Pressable className="btn-primary">
        <Text className="btn-primary-text">Sign In</Text>
      </Pressable>
    </View>
  );
}
```

### 2.7 NativeWind Gotchas

After using NativeWind on several production apps, here are the traps:

**1. Not all Tailwind utilities work in RN.**

React Native doesn't support all CSS properties. Things like `grid`, `backdrop-blur`, most `filter` utilities, and complex selectors don't work. NativeWind gracefully ignores unsupported utilities, which means your style silently does nothing. This is the most common debugging frustration.

```tsx
// These WON'T work on native (they work on web through NativeWind)
<View className="grid grid-cols-2">        {/* No CSS Grid in RN */}
<View className="backdrop-blur-lg">        {/* No backdrop-filter in RN */}
<View className="hover:bg-gray-100">       {/* No hover on mobile */}
<View className="overflow-x-scroll">       {/* Use ScrollView instead */}
```

**2. TypeScript className prop.**

Out of the box, React Native components don't have a `className` prop. NativeWind adds it via its JSX import source, but you might get TypeScript errors in custom components:

```tsx
// Problem: your custom component doesn't accept className
function Card({ children }: { children: React.ReactNode }) {
  return <View>{children}</View>;
}

// Solution: use the cssInterop helper or just forward the className
import { cssInterop } from 'nativewind';

// For third-party components
import { BlurView } from 'expo-blur';
cssInterop(BlurView, { className: 'style' });

// Now this works:
<BlurView className="rounded-xl overflow-hidden" />
```

**3. Animation classes have limits.**

NativeWind's `transition-*` and `animate-*` classes work, but for complex animations you'll want Reanimated. Don't try to build a carousel with Tailwind animation classes.

---

## 3. TAMAGUI: CROSS-PLATFORM WITH A COMPILER

Tamagui takes a fundamentally different approach. Instead of mapping utility classes to RN styles, it provides a complete component library with a compiler that extracts styles at build time. If you're building for both React Native and web, Tamagui is worth serious consideration.

### 3.1 The Core Idea

Tamagui's pitch is: write once, get optimized output for both web (CSS) and native (StyleSheet). The compiler flattens your styled components into direct `View` and `Text` elements with pre-computed styles, removing the runtime overhead.

```tsx
import { Button, H1, Paragraph, YStack, XStack, Theme } from 'tamagui';

function WelcomeScreen() {
  return (
    <Theme name="light">
      <YStack flex={1} padding="$4" backgroundColor="$background" gap="$4">
        <H1 color="$color">Welcome Back</H1>
        <Paragraph color="$colorSubtle" size="$5">
          Pick up where you left off
        </Paragraph>
        
        <XStack gap="$3" marginTop="$6">
          <Button theme="active" flex={1} size="$5">
            Sign In
          </Button>
          <Button variant="outlined" flex={1} size="$5">
            Create Account
          </Button>
        </XStack>
      </YStack>
    </Theme>
  );
}
```

### 3.2 The Token System

Tamagui's design token system is first-class. You define tokens once, and they flow through everything — components, themes, media queries.

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
    7: 28,
    8: 32,
    9: 36,
    10: 40,
    true: 16,  // Default size
  },
  space: {
    0: 0,
    1: 4,
    2: 8,
    3: 12,
    4: 16,
    5: 20,
    6: 24,
    7: 28,
    8: 32,
    9: 48,
    10: 64,
    true: 16,
  },
  radius: {
    0: 0,
    1: 4,
    2: 8,
    3: 12,
    4: 16,
    5: 20,
    6: 24,
    true: 8,
  },
  color: {
    white: '#FFFFFF',
    black: '#000000',
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
    brand500: '#6366F1',
    brand600: '#4F46E5',
    brand700: '#4338CA',
  },
});

const lightTheme = {
  background: tokens.color.white,
  backgroundSubtle: tokens.color.gray50,
  color: tokens.color.gray900,
  colorSubtle: tokens.color.gray500,
  borderColor: tokens.color.gray200,
  brandColor: tokens.color.brand600,
};

const darkTheme = {
  background: tokens.color.gray900,
  backgroundSubtle: tokens.color.gray800,
  color: tokens.color.white,
  colorSubtle: tokens.color.gray400,
  borderColor: tokens.color.gray700,
  brandColor: tokens.color.brand500,
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
    // Interaction-based media queries
    pointerCoarse: { pointer: 'coarse' },  // Touch devices
    pointerFine: { pointer: 'fine' },       // Mouse devices
  },
  shorthands: {
    // Shorthand props for convenience
    px: 'paddingHorizontal',
    py: 'paddingVertical',
    mx: 'marginHorizontal',
    my: 'marginVertical',
    br: 'borderRadius',
    bg: 'backgroundColor',
    w: 'width',
    h: 'height',
  } as const,
});

export type AppConfig = typeof config;

declare module 'tamagui' {
  interface TamaguiCustomConfig extends AppConfig {}
}

export default config;
```

### 3.3 Creating Custom Styled Components

Tamagui's `styled()` function creates optimizable components. The compiler can flatten these:

```tsx
import { styled, Stack, Text, GetProps } from 'tamagui';

// The compiler will optimize this into a flat View + StyleSheet
const Card = styled(Stack, {
  backgroundColor: '$background',
  borderRadius: '$4',
  padding: '$4',
  borderWidth: 1,
  borderColor: '$borderColor',
  
  // Variants — type-safe and optimizable
  variants: {
    elevated: {
      true: {
        shadowColor: '$color',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 8,
        elevation: 3,
      },
    },
    size: {
      small: {
        padding: '$2',
        borderRadius: '$2',
      },
      medium: {
        padding: '$4',
        borderRadius: '$4',
      },
      large: {
        padding: '$6',
        borderRadius: '$5',
      },
    },
  } as const,
  
  // Default variants
  defaultVariants: {
    size: 'medium',
  },
});

// Type-safe props are generated automatically
type CardProps = GetProps<typeof Card>;

const CardTitle = styled(Text, {
  fontSize: 18,
  fontWeight: '700',
  color: '$color',
  marginBottom: '$2',
});

const CardDescription = styled(Text, {
  fontSize: 14,
  color: '$colorSubtle',
  lineHeight: 22,
});

// Usage
function FeatureCard({ title, description }: { title: string; description: string }) {
  return (
    <Card elevated size="medium">
      <CardTitle>{title}</CardTitle>
      <CardDescription>{description}</CardDescription>
    </Card>
  );
}
```

### 3.4 Media Queries and Responsive Styles

Tamagui handles responsive styles through its media query system, and the compiler can optimize these:

```tsx
import { Stack, Text, styled } from 'tamagui';

const ResponsiveGrid = styled(Stack, {
  flexDirection: 'column',
  gap: '$3',
  padding: '$4',
  
  // At medium breakpoint, switch to row
  $md: {
    flexDirection: 'row',
    flexWrap: 'wrap',
  },
});

const GridItem = styled(Stack, {
  width: '100%',
  
  $md: {
    width: '48%',
  },
  $lg: {
    width: '31%',
  },
});

// Or use the hook for imperative logic
import { useMedia } from 'tamagui';

function AdaptiveLayout({ children }: { children: React.ReactNode }) {
  const media = useMedia();
  
  if (media.lg) {
    return <SidebarLayout>{children}</SidebarLayout>;
  }
  return <TabLayout>{children}</TabLayout>;
}
```

### 3.5 The Compiler: What It Actually Does

This is Tamagui's secret weapon and its biggest source of confusion. Let's be clear about what it does.

**Before compilation:**
```tsx
<Stack padding="$4" backgroundColor="$background" borderRadius="$3">
  <Text color="$color" fontSize={18}>Hello</Text>
</Stack>
```

**After compilation (native):**
```tsx
// The compiler flattens Stack → View, Text → Text
// Token references → resolved values
// Style objects → pre-computed StyleSheet references
<View style={_sheet["0"]}>
  <Text style={_sheet["1"]}>Hello</Text>
</View>
```

This means:
- No runtime token resolution
- No component wrapper overhead
- Pre-computed `StyleSheet.create` calls
- Smaller bundle because the styled component code is removed

**When the compiler CAN'T optimize:**
- Dynamic values (props that aren't known at build time)
- Spread props (`{...rest}`)
- Conditional expressions in style props
- Components with complex animation states

```tsx
// CAN be optimized — all values are static or tokens
<Stack padding="$4" backgroundColor="$background" />

// CANNOT be optimized — dynamic value
<Stack padding={isLarge ? '$6' : '$4'} backgroundColor="$background" />

// The compiler will gracefully fall back to runtime evaluation
// Your app still works, it's just not as fast
```

### 3.6 Tamagui Gotchas

**1. Setup complexity.** Tamagui's configuration is more involved than NativeWind. The `tamagui.config.ts` file can easily grow to hundreds of lines. This is fine for a team building a design system; it's overkill for a weekend project.

**2. The compiler is optional but important.** Tamagui works without the compiler (everything is evaluated at runtime), but you lose the main performance benefit. Make sure your build pipeline is set up correctly.

**3. Bundle size.** Tamagui ships a lot of code. If you're only using a few components, consider importing from `@tamagui/core` instead of the full `tamagui` package. Tree shaking helps, but it's not perfect.

**4. Learning curve.** Tamagui has its own vocabulary: themes, tokens, shorthands, variants, media queries, animations. It's a lot to learn upfront compared to "just write Tailwind classes."

---

## 4. UNISTYLES: JSI-BASED STYLING

Unistyles is the newcomer that's been gaining serious momentum. It's a JSI-based styling library — meaning it runs on the native thread via the JavaScript Interface, not through the bridge. It gives you a `StyleSheet.create`-like API with superpowers.

### 4.1 Why Unistyles Is Interesting

If NativeWind is "Tailwind for RN" and Tamagui is "a cross-platform design system," then Unistyles is "StyleSheet.create that doesn't suck." It keeps the familiar API but adds:

- **Theming** with instant, synchronous switching (no React re-render required)
- **Breakpoints** that are evaluated on the native side
- **Dynamic functions** in your stylesheets
- **Runtime** that lives entirely in C++ (via JSI)

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
    fontSize: 24,
    fontWeight: '700',
    color: theme.colors.text,
    marginBottom: theme.spacing.md,
  },
  grid: {
    flexDirection: runtime.screen.width > 768 ? 'row' : 'column',
    gap: theme.spacing.md,
    padding: theme.spacing.lg,
  },
}));

function HomeScreen() {
  const { styles, theme } = useStyles(stylesheet);
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Home</Text>
      <View style={styles.grid}>
        {/* Content */}
      </View>
    </View>
  );
}
```

### 4.2 Setting Up Unistyles

```bash
npx expo install react-native-unistyles
```

```typescript
// unistyles.ts — your theme and breakpoint configuration
import { UnistylesRegistry } from 'react-native-unistyles';

const lightTheme = {
  colors: {
    background: '#FFFFFF',
    surface: '#F9FAFB',
    text: '#111827',
    textSecondary: '#6B7280',
    border: '#E5E7EB',
    brand: '#6366F1',
    brandDark: '#4F46E5',
    error: '#EF4444',
    success: '#10B981',
    warning: '#F59E0B',
  },
  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
    xxl: 48,
  },
  radius: {
    sm: 6,
    md: 12,
    lg: 16,
    xl: 24,
    full: 9999,
  },
  typography: {
    h1: { fontSize: 32, fontWeight: '800' as const, lineHeight: 40 },
    h2: { fontSize: 24, fontWeight: '700' as const, lineHeight: 32 },
    h3: { fontSize: 20, fontWeight: '600' as const, lineHeight: 28 },
    body: { fontSize: 16, fontWeight: '400' as const, lineHeight: 24 },
    caption: { fontSize: 13, fontWeight: '400' as const, lineHeight: 18 },
  },
};

const darkTheme: typeof lightTheme = {
  colors: {
    background: '#0F172A',
    surface: '#1E293B',
    text: '#F1F5F9',
    textSecondary: '#94A3B8',
    border: '#334155',
    brand: '#818CF8',
    brandDark: '#6366F1',
    error: '#F87171',
    success: '#34D399',
    warning: '#FBBF24',
  },
  spacing: lightTheme.spacing,  // Same spacing in both themes
  radius: lightTheme.radius,
  typography: lightTheme.typography,
};

// Breakpoints for responsive design
const breakpoints = {
  xs: 0,      // Phones
  sm: 380,    // Large phones
  md: 768,    // Tablets
  lg: 1024,   // Large tablets / web
} as const;

type AppBreakpoints = typeof breakpoints;
type AppThemes = {
  light: typeof lightTheme;
  dark: typeof darkTheme;
};

// Register with TypeScript augmentation
declare module 'react-native-unistyles' {
  export interface UnistylesBreakpoints extends AppBreakpoints {}
  export interface UnistylesThemes extends AppThemes {}
}

UnistylesRegistry
  .addBreakpoints(breakpoints)
  .addThemes({
    light: lightTheme,
    dark: darkTheme,
  })
  .addConfig({
    adaptiveThemes: true,  // Auto-switch with system dark mode
  });
```

### 4.3 Dynamic Styles and the Runtime Object

The `runtime` object gives you access to device information without importing multiple hooks:

```tsx
const stylesheet = createStyleSheet((theme, runtime) => ({
  safeArea: {
    paddingTop: runtime.insets.top,
    paddingBottom: runtime.insets.bottom,
    paddingLeft: runtime.insets.left,
    paddingRight: runtime.insets.right,
  },
  
  // Responsive styles using breakpoints
  header: {
    height: {
      xs: 56,
      md: 64,
      lg: 72,
    },
    paddingHorizontal: theme.spacing.lg,
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: theme.colors.surface,
    borderBottomWidth: 1,
    borderBottomColor: theme.colors.border,
  },
  
  // Dynamic styles based on props
  avatar: (size: number) => ({
    width: size,
    height: size,
    borderRadius: size / 2,
    backgroundColor: theme.colors.brand,
  }),
  
  // Landscape vs portrait
  content: {
    flexDirection: runtime.orientation === 'landscape' ? 'row' : 'column',
    padding: theme.spacing.lg,
  },
}));

function ProfileScreen() {
  const { styles } = useStyles(stylesheet);
  
  return (
    <View style={styles.safeArea}>
      <View style={styles.header}>
        {/* styles.avatar is a function — call it with the size */}
        <View style={styles.avatar(48)} />
      </View>
      <View style={styles.content}>
        {/* Content adapts to orientation */}
      </View>
    </View>
  );
}
```

### 4.4 Theme Switching

Because Unistyles runs at the JSI level, theme switching is synchronous. No React state update, no re-render cascade:

```tsx
import { UnistylesRuntime } from 'react-native-unistyles';

// Switch theme — this is instant, no re-render required
// Unistyles updates all mounted components synchronously via JSI
UnistylesRuntime.setTheme('dark');

// Or toggle
function toggleTheme() {
  const current = UnistylesRuntime.themeName;
  UnistylesRuntime.setTheme(current === 'light' ? 'dark' : 'light');
}

// Adaptive themes (follow system setting)
UnistylesRuntime.setAdaptiveThemes(true);
```

This is a genuine performance advantage. In a NativeWind or Tamagui app, toggling dark mode causes a re-render from the top of the tree. With Unistyles, the style values update at the native level and the UI refreshes without React being involved.

### 4.5 Unistyles Gotchas

**1. No web support (yet).** If you're targeting web, Unistyles is not an option today. It's purely native (iOS + Android) because it relies on JSI.

**2. Expo compatibility.** Unistyles requires a custom dev client (it has native code). You can't use Expo Go. This is fine for production apps but adds friction during early development.

**3. Smaller ecosystem.** Fewer blog posts, fewer StackOverflow answers, fewer example repos than NativeWind or Tamagui. You're more on your own when you hit edge cases.

**4. It's a styling library, not a component library.** Unistyles doesn't give you pre-built components like Tamagui does. You style your own components (or use it alongside a component library).

---

## 5. THE DECISION MATRIX

Stop reading blog posts that say "Library X is the best." The right choice depends on your context. Here's the matrix:

### 5.1 At a Glance

| Factor | StyleSheet.create | NativeWind v4 | Tamagui | Unistyles |
|--------|------------------|---------------|---------|-----------|
| **Learning curve** | None | Low (if you know Tailwind) | High | Low-Medium |
| **Dark mode** | Manual | Built-in (`dark:`) | Built-in (themes) | Built-in (JSI) |
| **Responsive** | Manual | Breakpoints | Media queries | Breakpoints (JSI) |
| **Web support** | No | Yes | Yes (primary use case) | No |
| **Compiler optimization** | N/A | Tailwind compiler | Tamagui compiler | N/A (JSI) |
| **Component library** | No | No | Yes (extensive) | No |
| **TypeScript** | Basic | Good | Excellent | Excellent |
| **Bundle size impact** | None | ~15KB | ~50-100KB | ~8KB |
| **Expo Go compatible** | Yes | Yes (with setup) | Yes | No |
| **Theme switching perf** | N/A | Re-render | Re-render | Instant (JSI) |

### 5.2 Pick This If...

**StyleSheet.create** — You're building a small app, no dark mode, no tablet, no web. Or you're prototyping and want zero dependencies.

**NativeWind v4** — Your team knows Tailwind. You want rapid iteration with a familiar API. You might target web later. You value developer experience over peak runtime performance.

**Tamagui** — You're building a cross-platform app (RN + web) with a real design system. You want a component library included. You're willing to invest in setup for long-term payoff.

**Unistyles** — You want maximum native performance. You're only targeting iOS and Android. You like the `StyleSheet.create` mental model but need theming and responsiveness. You're using a custom dev client anyway.

### 5.3 The Combinations That Work

Some teams combine these. The combinations that actually work:

- **Unistyles + NativeWind** — Don't. Pick one.
- **Tamagui + NativeWind** — Don't. They solve the same problem.
- **Unistyles + a component library (like React Native Paper)** — This works well. Unistyles for your custom styles, the component library for standard UI elements.
- **NativeWind + Reanimated** — Great combination. NativeWind for layout/theming, Reanimated for complex animations.
- **Tamagui + Reanimated** — Works, but Tamagui has its own animation system. Check if Tamagui's animations are sufficient before adding Reanimated.

---

## 6. DESIGN TOKENS: THE SHARED LANGUAGE

Regardless of which styling library you pick, you need design tokens. Tokens are the atomic values — colors, spacing, typography, radii, shadows — that make your app visually consistent.

### 6.1 What Tokens Are (and Aren't)

Tokens are **not** "a colors.ts file." That's where most teams stop, and it's not enough.

Tokens are a hierarchical system:

```
Global Tokens → Alias Tokens → Component Tokens
  blue-500       brand-primary     button-bg-primary
  16px           spacing-md        card-padding
  700            font-weight-bold  heading-weight
```

**Global tokens** are raw values: `blue-500 = #6366F1`, `space-4 = 16`.
**Alias tokens** give semantic meaning: `brand-primary = blue-500`, `spacing-md = space-4`.
**Component tokens** are scoped: `button-bg-primary = brand-primary`, `card-padding = spacing-md`.

Why this hierarchy? Because when the brand color changes from indigo to teal, you change one alias token and every component updates. If you used `#6366F1` directly in 200 components, good luck.

### 6.2 A Production Token System

```typescript
// tokens/primitives.ts — Global tokens (raw values, never used directly in components)
export const primitives = {
  colors: {
    white: '#FFFFFF',
    black: '#000000',
    
    gray: {
      50: '#F9FAFB', 100: '#F3F4F6', 200: '#E5E7EB', 300: '#D1D5DB',
      400: '#9CA3AF', 500: '#6B7280', 600: '#4B5563', 700: '#374151',
      800: '#1F2937', 900: '#111827', 950: '#030712',
    },
    indigo: {
      50: '#EEF2FF', 100: '#E0E7FF', 200: '#C7D2FE', 300: '#A5B4FC',
      400: '#818CF8', 500: '#6366F1', 600: '#4F46E5', 700: '#4338CA',
      800: '#3730A3', 900: '#312E81', 950: '#1E1B4B',
    },
    red: {
      50: '#FEF2F2', 100: '#FEE2E2', 400: '#F87171',
      500: '#EF4444', 600: '#DC2626', 700: '#B91C1C',
    },
    green: {
      50: '#F0FDF4', 100: '#DCFCE7', 400: '#4ADE80',
      500: '#22C55E', 600: '#16A34A', 700: '#15803D',
    },
    amber: {
      50: '#FFFBEB', 100: '#FEF3C7', 400: '#FBBF24',
      500: '#F59E0B', 600: '#D97706', 700: '#B45309',
    },
  },
  
  spacing: {
    0: 0, 1: 4, 2: 8, 3: 12, 4: 16, 5: 20, 6: 24, 8: 32, 10: 40, 12: 48, 16: 64,
  },
  
  radius: {
    none: 0, sm: 6, md: 10, lg: 16, xl: 24, full: 9999,
  },
  
  fontSize: {
    xs: 12, sm: 14, base: 16, lg: 18, xl: 20, '2xl': 24, '3xl': 30, '4xl': 36,
  },
  
  fontWeight: {
    normal: '400' as const,
    medium: '500' as const,
    semibold: '600' as const,
    bold: '700' as const,
    extrabold: '800' as const,
  },
  
  lineHeight: {
    tight: 1.25,
    normal: 1.5,
    relaxed: 1.75,
  },
} as const;
```

```typescript
// tokens/semantic.ts — Alias tokens (semantic meaning, references primitives)
import { primitives as p } from './primitives';

export const lightTheme = {
  // Surfaces
  background: p.colors.white,
  backgroundSubtle: p.colors.gray[50],
  surface: p.colors.white,
  surfaceRaised: p.colors.white,
  
  // Text
  text: p.colors.gray[900],
  textSecondary: p.colors.gray[500],
  textTertiary: p.colors.gray[400],
  textInverse: p.colors.white,
  
  // Brand
  brandPrimary: p.colors.indigo[600],
  brandPrimaryHover: p.colors.indigo[700],
  brandSubtle: p.colors.indigo[50],
  brandText: p.colors.indigo[600],
  
  // Borders
  border: p.colors.gray[200],
  borderSubtle: p.colors.gray[100],
  borderStrong: p.colors.gray[300],
  
  // Feedback
  error: p.colors.red[600],
  errorSubtle: p.colors.red[50],
  errorText: p.colors.red[700],
  success: p.colors.green[600],
  successSubtle: p.colors.green[50],
  successText: p.colors.green[700],
  warning: p.colors.amber[500],
  warningSubtle: p.colors.amber[50],
  warningText: p.colors.amber[700],
  
  // Interactive
  inputBackground: p.colors.gray[50],
  inputBorder: p.colors.gray[300],
  inputBorderFocus: p.colors.indigo[500],
  inputText: p.colors.gray[900],
  inputPlaceholder: p.colors.gray[400],
};

export const darkTheme: typeof lightTheme = {
  background: p.colors.gray[950],
  backgroundSubtle: p.colors.gray[900],
  surface: p.colors.gray[900],
  surfaceRaised: p.colors.gray[800],
  
  text: p.colors.gray[50],
  textSecondary: p.colors.gray[400],
  textTertiary: p.colors.gray[500],
  textInverse: p.colors.gray[900],
  
  brandPrimary: p.colors.indigo[500],
  brandPrimaryHover: p.colors.indigo[400],
  brandSubtle: p.colors.indigo[950],
  brandText: p.colors.indigo[400],
  
  border: p.colors.gray[700],
  borderSubtle: p.colors.gray[800],
  borderStrong: p.colors.gray[600],
  
  error: p.colors.red[500],
  errorSubtle: '#2D1B1B',
  errorText: p.colors.red[400],
  success: p.colors.green[500],
  successSubtle: '#1B2D1E',
  successText: p.colors.green[400],
  warning: p.colors.amber[400],
  warningSubtle: '#2D2A1B',
  warningText: p.colors.amber[400],
  
  inputBackground: p.colors.gray[800],
  inputBorder: p.colors.gray[600],
  inputBorderFocus: p.colors.indigo[400],
  inputText: p.colors.gray[50],
  inputPlaceholder: p.colors.gray[500],
};
```

```typescript
// tokens/component.ts — Component tokens (scoped to specific components)
import { primitives as p } from './primitives';

export const componentTokens = {
  button: {
    primary: {
      bg: 'brandPrimary',         // References semantic token
      text: 'textInverse',
      bgPressed: 'brandPrimaryHover',
      height: 48,
      paddingH: p.spacing[6],
      borderRadius: p.radius.lg,
      fontSize: p.fontSize.base,
      fontWeight: p.fontWeight.semibold,
    },
    secondary: {
      bg: 'transparent',
      text: 'brandPrimary',
      borderColor: 'border',
      bgPressed: 'brandSubtle',
      height: 48,
      paddingH: p.spacing[6],
      borderRadius: p.radius.lg,
      fontSize: p.fontSize.base,
      fontWeight: p.fontWeight.semibold,
    },
    ghost: {
      bg: 'transparent',
      text: 'text',
      bgPressed: 'backgroundSubtle',
      height: 44,
      paddingH: p.spacing[4],
      borderRadius: p.radius.md,
      fontSize: p.fontSize.base,
      fontWeight: p.fontWeight.medium,
    },
  },
  
  card: {
    bg: 'surface',
    borderColor: 'border',
    borderRadius: p.radius.xl,
    padding: p.spacing[4],
    shadowOpacity: 0.05,
    shadowRadius: 8,
  },
  
  input: {
    bg: 'inputBackground',
    text: 'inputText',
    placeholder: 'inputPlaceholder',
    border: 'inputBorder',
    borderFocus: 'inputBorderFocus',
    height: 48,
    borderRadius: p.radius.lg,
    paddingH: p.spacing[4],
    fontSize: p.fontSize.base,
  },
};
```

### 6.3 Using Tokens with Each Library

**With NativeWind:**
Your tokens go in `tailwind.config.js`. The primitives map to Tailwind's color/spacing scale. The semantic tokens map to CSS custom properties.

**With Tamagui:**
Tokens plug directly into `createTokens()` and `createTamagui()`. Tamagui was designed for this pattern.

**With Unistyles:**
Tokens become your theme object, passed to `UnistylesRegistry.addThemes()`.

**With plain StyleSheet.create:**
Use a React Context to provide the current theme's semantic tokens, and create a `useTheme()` hook.

---

## 7. RESPONSIVE DESIGN FOR MOBILE

"Responsive design" on mobile is different from web. You're not dealing with browser resize events. You're dealing with:

1. **Different phone sizes** — iPhone SE (375pt) to iPhone 15 Pro Max (430pt)
2. **Tablets** — iPad Mini (744pt) to iPad Pro 12.9" (1024pt)
3. **Foldables** — Samsung Fold in folded (344dp) vs unfolded (884dp) state
4. **Orientation** — Portrait vs landscape
5. **Dynamic Type** — Users who've changed their system font size

### 7.1 The Phone Size Problem

Most "responsive" RN apps don't actually need breakpoints for phones. The difference between a 375pt and 430pt phone is small enough that flex layouts handle it. Where teams get into trouble is **hard-coded dimensions**.

```tsx
// BAD: Hard-coded widths that don't adapt
const styles = StyleSheet.create({
  card: {
    width: 350,  // This overflows on iPhone SE, wastes space on Max
    height: 200,
  },
});

// GOOD: Relative sizing
const styles = StyleSheet.create({
  card: {
    width: '100%',    // Or use flex
    aspectRatio: 16 / 9,  // Maintain proportions
    maxWidth: 500,     // Cap on tablets
  },
});
```

### 7.2 Tablet Layouts

Tablets are where responsive design actually matters in mobile. A phone layout stretched to iPad looks terrible.

```tsx
import { useWindowDimensions } from 'react-native';

// A simple but effective responsive hook
function useLayout() {
  const { width, height } = useWindowDimensions();
  
  return {
    isPhone: width < 768,
    isTablet: width >= 768 && width < 1024,
    isLargeTablet: width >= 1024,
    isLandscape: width > height,
    screenWidth: width,
    screenHeight: height,
    // Useful for grid calculations
    columns: width < 768 ? 1 : width < 1024 ? 2 : 3,
    contentWidth: Math.min(width, 1200), // Cap content width
  };
}

// Master-detail layout for tablets
function InboxScreen() {
  const { isPhone, isTablet } = useLayout();
  const [selectedId, setSelectedId] = useState<string | null>(null);

  if (isPhone) {
    // Phone: show list OR detail, not both
    return selectedId 
      ? <EmailDetail id={selectedId} onBack={() => setSelectedId(null)} />
      : <EmailList onSelect={setSelectedId} />;
  }

  // Tablet: side-by-side master-detail
  return (
    <View style={{ flexDirection: 'row', flex: 1 }}>
      <View style={{ width: isTablet ? 320 : 380, borderRightWidth: 1, borderColor: '#E5E7EB' }}>
        <EmailList onSelect={setSelectedId} selectedId={selectedId} />
      </View>
      <View style={{ flex: 1 }}>
        {selectedId 
          ? <EmailDetail id={selectedId} /> 
          : <EmptyState message="Select an email" />
        }
      </View>
    </View>
  );
}
```

### 7.3 Dynamic Type / Font Scaling

Users set custom font sizes in their OS settings. Your app should respect this. By default, React Native respects Dynamic Type on iOS and font scale on Android, but you need to test with extreme values.

```tsx
import { PixelRatio, useWindowDimensions } from 'react-native';

// Get the current font scale
const fontScale = PixelRatio.getFontScale();
// 1.0 = default, 1.5 = 150%, 2.0 = 200%

// PROBLEM: Components break at large font scales
// because text overflows fixed containers

// SOLUTION 1: Use flexShrink and numberOfLines
<View style={{ flexDirection: 'row', alignItems: 'center' }}>
  <Text 
    style={{ fontSize: 16, flex: 1, flexShrink: 1 }} 
    numberOfLines={2}
  >
    This text will wrap and truncate if needed
  </Text>
  <Text style={{ fontSize: 16, marginLeft: 8 }}>$99</Text>
</View>

// SOLUTION 2: Cap the font scale for specific elements
// (use sparingly — accessibility matters)
import { Text, TextProps, StyleSheet } from 'react-native';

function CappedText({ style, ...props }: TextProps) {
  return (
    <Text 
      {...props} 
      style={style}
      maxFontSizeMultiplier={1.3}  // Cap at 130% of default
      allowFontScaling={true}       // Still allow some scaling
    />
  );
}

// SOLUTION 3: Test at extreme scales
// In iOS Simulator: Settings → Accessibility → Display & Text Size → Larger Text
// In Android Emulator: Settings → Display → Font size (drag to max)
```

### 7.4 Safe Areas and Notches

Every modern phone has insets — the notch, Dynamic Island, home indicator, status bar. Handle them:

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function ScreenContainer({ children }: { children: React.ReactNode }) {
  const insets = useSafeAreaInsets();
  
  return (
    <View style={{
      flex: 1,
      paddingTop: insets.top,
      paddingBottom: insets.bottom,
      paddingLeft: insets.left,   // Important for landscape
      paddingRight: insets.right,  // Important for landscape
    }}>
      {children}
    </View>
  );
}

// Or with NativeWind — it integrates with safe area context
function Screen() {
  return (
    <View className="flex-1 pt-safe pb-safe px-safe">
      {/* Content respects safe areas */}
    </View>
  );
}
```

---

## 8. REANIMATED V3: THE ANIMATION ENGINE

React Native Reanimated is the animation library for React Native. Not an option — the library. If you need animations that run at 60fps (or 120fps on ProMotion devices), Reanimated is non-negotiable.

### 8.1 Why Not the Built-in Animated API?

The built-in `Animated` API has a fundamental problem: most operations run on the JavaScript thread. When the JS thread is busy (rendering, computing, running your business logic), animations stutter.

Reanimated solves this by running animations on the **UI thread** via "worklets" — small JavaScript functions that are compiled and executed on the native side.

```tsx
// Built-in Animated — runs on JS thread
import { Animated } from 'react-native';

const fadeAnim = new Animated.Value(0);
Animated.timing(fadeAnim, {
  toValue: 1,
  duration: 300,
  useNativeDriver: true,  // This helps, but only for opacity/transform
}).start();

// Reanimated v3 — runs entirely on UI thread
import Animated, { 
  useSharedValue, 
  useAnimatedStyle, 
  withTiming, 
  withSpring 
} from 'react-native-reanimated';

function FadeIn({ children }: { children: React.ReactNode }) {
  const opacity = useSharedValue(0);
  
  useEffect(() => {
    opacity.value = withTiming(1, { duration: 300 });
  }, []);
  
  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
  }));
  
  return (
    <Animated.View style={animatedStyle}>
      {children}
    </Animated.View>
  );
}
```

### 8.2 Shared Values: The Core Primitive

A `SharedValue` is Reanimated's fundamental building block. It's a value that exists on both the JS thread and the UI thread simultaneously, and it can be read and written from either side.

```tsx
import { useSharedValue, withTiming, withSpring, withDelay } from 'react-native-reanimated';

function AnimatedCard() {
  // Shared values — these live on both threads
  const scale = useSharedValue(1);
  const translateY = useSharedValue(0);
  const opacity = useSharedValue(1);
  const rotation = useSharedValue(0);
  
  // Animate with timing (ease in/out)
  const handlePress = () => {
    scale.value = withTiming(0.95, { duration: 100 }, () => {
      // Callback runs on UI thread when animation completes
      scale.value = withTiming(1, { duration: 100 });
    });
  };
  
  // Animate with spring (physics-based)
  const handleDrag = (translationY: number) => {
    translateY.value = withSpring(translationY, {
      damping: 15,        // Higher = less bouncy
      stiffness: 150,     // Higher = faster
      mass: 1,            // Higher = more inertia
    });
  };
  
  // Sequence with delays
  const handleAppear = () => {
    opacity.value = withDelay(200, withTiming(1, { duration: 400 }));
    translateY.value = withDelay(200, withSpring(0, { damping: 12 }));
  };
  
  // Animated styles — computed on UI thread
  const cardStyle = useAnimatedStyle(() => ({
    transform: [
      { scale: scale.value },
      { translateY: translateY.value },
      { rotate: `${rotation.value}deg` },
    ],
    opacity: opacity.value,
  }));
  
  return (
    <Animated.View style={[styles.card, cardStyle]}>
      <Pressable onPress={handlePress}>
        <Text>Tap me</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### 8.3 Worklets: Functions That Run on the UI Thread

A worklet is any function marked with the `'worklet'` directive. It gets compiled by Reanimated's Babel plugin and can run on the UI thread.

```tsx
import { runOnJS, runOnUI, useAnimatedStyle } from 'react-native-reanimated';

// This runs on the UI thread — no JS thread involvement
function clampValue(value: number, min: number, max: number) {
  'worklet';
  return Math.min(Math.max(value, min), max);
}

// More complex worklet example
function interpolateColor(progress: number) {
  'worklet';
  // This computation happens on the UI thread at 60fps
  const r = Math.round(255 * (1 - progress) + 99 * progress);
  const g = Math.round(255 * (1 - progress) + 102 * progress);
  const b = Math.round(255 * (1 - progress) + 241 * progress);
  return `rgb(${r}, ${g}, ${b})`;
}

function ProgressBar({ progress }: { progress: SharedValue<number> }) {
  const animatedStyle = useAnimatedStyle(() => {
    const clampedProgress = clampValue(progress.value, 0, 1);
    
    return {
      width: `${clampedProgress * 100}%`,
      backgroundColor: interpolateColor(clampedProgress),
    };
  });
  
  return (
    <View style={styles.progressTrack}>
      <Animated.View style={[styles.progressFill, animatedStyle]} />
    </View>
  );
}

// When you need to call JS functions from a worklet
function MyComponent() {
  const position = useSharedValue(0);
  
  const updateAnalytics = (pos: number) => {
    // This runs on the JS thread
    analytics.track('scroll_position', { position: pos });
  };
  
  const animatedStyle = useAnimatedStyle(() => {
    // This runs on the UI thread
    if (position.value > 500) {
      // Can't call JS functions directly from a worklet!
      // Use runOnJS to bridge back:
      runOnJS(updateAnalytics)(position.value);
    }
    
    return {
      transform: [{ translateY: -position.value * 0.5 }],
    };
  });
  
  return <Animated.View style={animatedStyle} />;
}
```

### 8.4 Interpolation

`interpolate` maps a value from one range to another. This is how you create scroll-driven animations, parallax effects, and complex transitions.

```tsx
import { interpolate, Extrapolation, useAnimatedStyle, useAnimatedScrollHandler } from 'react-native-reanimated';

function ParallaxHeader() {
  const scrollY = useSharedValue(0);
  
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });
  
  const headerStyle = useAnimatedStyle(() => {
    const height = interpolate(
      scrollY.value,
      [-100, 0, 200],    // Input range
      [400, 300, 100],    // Output range
      Extrapolation.CLAMP  // Don't go beyond the output range
    );
    
    const opacity = interpolate(
      scrollY.value,
      [0, 150, 200],
      [1, 0.5, 0],
      Extrapolation.CLAMP
    );
    
    const scale = interpolate(
      scrollY.value,
      [-100, 0],
      [1.3, 1],
      Extrapolation.CLAMP
    );
    
    return {
      height,
      opacity,
      transform: [{ scale }],
    };
  });
  
  const titleStyle = useAnimatedStyle(() => {
    const translateY = interpolate(
      scrollY.value,
      [0, 200],
      [0, -60],
      Extrapolation.CLAMP
    );
    
    const fontSize = interpolate(
      scrollY.value,
      [0, 200],
      [32, 20],
      Extrapolation.CLAMP
    );
    
    return {
      transform: [{ translateY }],
      fontSize,
    };
  });
  
  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.header, headerStyle]}>
        <Animated.Text style={[styles.title, titleStyle]}>
          Discover
        </Animated.Text>
      </Animated.View>
      
      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        contentContainerStyle={{ paddingTop: 300 }}
      >
        {/* Scrollable content */}
      </Animated.ScrollView>
    </View>
  );
}
```

### 8.5 Spring Animations: Getting the Feel Right

Spring animations are what make mobile UI feel "native." Timing animations feel robotic. Spring animations feel physical. Here are the spring presets you need:

```tsx
import { withSpring, WithSpringConfig } from 'react-native-reanimated';

// Preset spring configurations for common use cases
const springs = {
  // Snappy — for button presses, toggles, small state changes
  snappy: {
    damping: 15,
    stiffness: 300,
    mass: 0.8,
  } satisfies WithSpringConfig,
  
  // Smooth — for page transitions, card expansions
  smooth: {
    damping: 20,
    stiffness: 150,
    mass: 1,
  } satisfies WithSpringConfig,
  
  // Bouncy — for success states, celebrations, playful UI
  bouncy: {
    damping: 8,
    stiffness: 200,
    mass: 0.8,
  } satisfies WithSpringConfig,
  
  // Gentle — for subtle movements, background elements
  gentle: {
    damping: 25,
    stiffness: 80,
    mass: 1.2,
  } satisfies WithSpringConfig,
  
  // Stiff — for drag snapping, drawer open/close
  stiff: {
    damping: 20,
    stiffness: 400,
    mass: 1,
  } satisfies WithSpringConfig,
};

// Usage
const handleToggle = () => {
  isExpanded.value = !isExpanded.value;
  height.value = withSpring(
    isExpanded.value ? 200 : 0, 
    springs.snappy
  );
};
```

### 8.6 Scroll-Driven Animations

One of Reanimated's most powerful patterns. You connect scroll position to any visual property:

```tsx
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedStyle,
  interpolate,
  interpolateColor,
  Extrapolation,
} from 'react-native-reanimated';

function AnimatedFeedScreen() {
  const scrollY = useSharedValue(0);
  
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });
  
  // Navigation bar that fades in as you scroll
  const navBarStyle = useAnimatedStyle(() => {
    const backgroundColor = interpolateColor(
      scrollY.value,
      [0, 100],
      ['transparent', 'rgba(255, 255, 255, 0.95)']
    );
    
    const borderBottomColor = interpolateColor(
      scrollY.value,
      [0, 100],
      ['transparent', '#E5E7EB']
    );
    
    return {
      backgroundColor,
      borderBottomColor,
      borderBottomWidth: 1,
    };
  });
  
  const navTitleStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      scrollY.value,
      [60, 100],
      [0, 1],
      Extrapolation.CLAMP
    );
    
    return { opacity };
  });
  
  // Floating action button that hides on scroll down, shows on scroll up
  const lastScrollY = useSharedValue(0);
  const fabTranslateY = useSharedValue(0);
  
  const fabScrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      const currentScrollY = event.contentOffset.y;
      const diff = currentScrollY - lastScrollY.value;
      
      // Scrolling down — hide FAB
      if (diff > 0 && currentScrollY > 50) {
        fabTranslateY.value = withSpring(100, springs.stiff);
      }
      // Scrolling up — show FAB
      else if (diff < -5) {
        fabTranslateY.value = withSpring(0, springs.snappy);
      }
      
      lastScrollY.value = currentScrollY;
    },
  });
  
  const fabStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: fabTranslateY.value }],
  }));
  
  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.navBar, navBarStyle]}>
        <Animated.Text style={[styles.navTitle, navTitleStyle]}>
          Feed
        </Animated.Text>
      </Animated.View>
      
      <Animated.FlatList
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        data={feedItems}
        renderItem={renderFeedItem}
      />
      
      <Animated.View style={[styles.fab, fabStyle]}>
        <PlusIcon />
      </Animated.View>
    </View>
  );
}
```

---

## 9. LAYOUT ANIMATIONS

Reanimated's layout animations let you animate components entering, exiting, and rearranging in a list. This is what makes your app feel polished.

### 9.1 Entering Animations

```tsx
import Animated, { 
  FadeIn, FadeInDown, FadeInUp, FadeInLeft, FadeInRight,
  SlideInDown, SlideInUp, SlideInLeft, SlideInRight,
  ZoomIn, BounceIn, FlipInXUp, LightSpeedInLeft,
  StretchInX, StretchInY,
} from 'react-native-reanimated';

// Basic fade in
<Animated.View entering={FadeIn.duration(300)}>
  <Text>I fade in</Text>
</Animated.View>

// Slide up with fade
<Animated.View entering={FadeInUp.duration(400).springify()}>
  <Text>I slide up with a spring</Text>
</Animated.View>

// Staggered list items
function StaggeredList({ items }: { items: Item[] }) {
  return (
    <View>
      {items.map((item, index) => (
        <Animated.View
          key={item.id}
          entering={FadeInDown
            .delay(index * 80)        // 80ms stagger between items
            .duration(400)
            .springify()
            .damping(15)
          }
        >
          <ListItem item={item} />
        </Animated.View>
      ))}
    </View>
  );
}

// Custom entering animation
import { EntryAnimationsValues, withTiming, withSpring } from 'react-native-reanimated';

const customEntering = (values: EntryAnimationsValues) => {
  'worklet';
  const animations = {
    opacity: withTiming(1, { duration: 300 }),
    transform: [
      { translateY: withSpring(0, { damping: 12 }) },
      { scale: withSpring(1, { damping: 10 }) },
    ],
  };
  const initialValues = {
    opacity: 0,
    transform: [
      { translateY: 50 },
      { scale: 0.9 },
    ],
  };
  return { initialValues, animations };
};

<Animated.View entering={customEntering}>
  <Card />
</Animated.View>
```

### 9.2 Exiting Animations

```tsx
import Animated, { 
  FadeOut, FadeOutDown, FadeOutUp,
  SlideOutDown, SlideOutLeft, SlideOutRight,
  ZoomOut, 
} from 'react-native-reanimated';

// Swipe-to-delete with exit animation
function DeletableItem({ item, onDelete }: DeletableItemProps) {
  const [isDeleting, setIsDeleting] = useState(false);
  
  if (isDeleting) return null;  // Reanimated handles the exit animation
  
  return (
    <Animated.View 
      entering={FadeInRight.duration(300)}
      exiting={SlideOutLeft.duration(250).withCallback((finished) => {
        // Called on UI thread when animation completes
        if (finished) {
          runOnJS(onDelete)(item.id);
        }
      })}
    >
      <Pressable onLongPress={() => setIsDeleting(true)}>
        <Text>{item.name}</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### 9.3 Layout Transitions

When items in a list reorder, Reanimated can animate the position change automatically:

```tsx
import Animated, { Layout, LinearTransition, SequencedTransition } from 'react-native-reanimated';

function SortableList() {
  const [items, setItems] = useState(initialItems);
  const [sortBy, setSortBy] = useState<'name' | 'date'>('name');
  
  const sortedItems = useMemo(() => {
    return [...items].sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return b.date.getTime() - a.date.getTime();
    });
  }, [items, sortBy]);
  
  return (
    <View>
      <View style={styles.sortButtons}>
        <Button title="By Name" onPress={() => setSortBy('name')} />
        <Button title="By Date" onPress={() => setSortBy('date')} />
      </View>
      
      {sortedItems.map((item) => (
        <Animated.View
          key={item.id}
          // Layout animation — items smoothly move to new positions
          layout={LinearTransition.springify().damping(15).stiffness(150)}
          entering={FadeInDown}
          exiting={FadeOutDown}
        >
          <ListItem item={item} />
        </Animated.View>
      ))}
    </View>
  );
}
```

### 9.4 Shared Element Transitions

Shared element transitions make items appear to morph from one screen to another. Reanimated v3 introduced this as a first-class feature:

```tsx
import Animated, { SharedTransition, withSpring } from 'react-native-reanimated';

// Define a custom transition
const customTransition = SharedTransition.custom((values) => {
  'worklet';
  return {
    width: withSpring(values.targetWidth, { damping: 15 }),
    height: withSpring(values.targetHeight, { damping: 15 }),
    originX: withSpring(values.targetOriginX, { damping: 15 }),
    originY: withSpring(values.targetOriginY, { damping: 15 }),
    borderRadius: withSpring(values.targetBorderRadius ?? 0, { damping: 15 }),
  };
});

// List screen — the thumbnail
function ProductListItem({ product, onPress }: ProductListItemProps) {
  return (
    <Pressable onPress={() => onPress(product)}>
      <Animated.Image
        source={{ uri: product.image }}
        style={styles.thumbnail}
        // sharedTransitionTag must be unique per element
        sharedTransitionTag={`product-image-${product.id}`}
        sharedTransitionStyle={customTransition}
      />
      <Text>{product.name}</Text>
    </Pressable>
  );
}

// Detail screen — the hero image
function ProductDetail({ product }: { product: Product }) {
  return (
    <ScrollView>
      <Animated.Image
        source={{ uri: product.image }}
        style={styles.heroImage}
        // Same tag as the list item — Reanimated connects them
        sharedTransitionTag={`product-image-${product.id}`}
        sharedTransitionStyle={customTransition}
      />
      <Text style={styles.title}>{product.name}</Text>
    </ScrollView>
  );
}
```

---

## 10. GESTURE HANDLER

React Native Gesture Handler is the companion to Reanimated. Where Reanimated handles "make things move," Gesture Handler handles "respond to touch." Together, they let you build interactions that feel indistinguishable from native.

### 10.1 Why Not onPanResponder?

React Native's built-in gesture system (`PanResponder`, `onTouchStart`, etc.) runs on the JS thread. The same problem as the built-in Animated API — when the JS thread is busy, gestures stutter or drop frames.

Gesture Handler runs on the native thread. Gestures are recognized and processed before the JS thread is even involved. This is why a swipe-to-delete built with Gesture Handler feels smooth even when your FlatList is rendering 500 items.

### 10.2 The New Gesture API (v2)

Gesture Handler v2 introduced a declarative gesture API that composes beautifully with Reanimated:

```tsx
import { Gesture, GestureDetector, GestureHandlerRootView } from 'react-native-gesture-handler';
import Animated, { 
  useSharedValue, 
  useAnimatedStyle, 
  withSpring,
  runOnJS,
} from 'react-native-reanimated';

function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);
  const savedX = useSharedValue(0);
  const savedY = useSharedValue(0);
  
  const panGesture = Gesture.Pan()
    .onStart(() => {
      // Scale up when user starts dragging
      scale.value = withSpring(1.05, { damping: 15 });
    })
    .onUpdate((event) => {
      // Move the card with the finger
      translateX.value = savedX.value + event.translationX;
      translateY.value = savedY.value + event.translationY;
    })
    .onEnd((event) => {
      // Snap to position or spring back
      const shouldDismiss = Math.abs(event.translationX) > 150;
      
      if (shouldDismiss) {
        const direction = event.translationX > 0 ? 1 : -1;
        translateX.value = withSpring(direction * 500);
        scale.value = withSpring(0.8);
      } else {
        // Spring back to saved position
        translateX.value = withSpring(savedX.value);
        translateY.value = withSpring(savedY.value);
        scale.value = withSpring(1);
        savedX.value = 0;
        savedY.value = 0;
      }
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
        <Text>Drag me</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 10.3 Pinch to Zoom

```tsx
function PinchableImage({ uri }: { uri: string }) {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const savedTranslateX = useSharedValue(0);
  const savedTranslateY = useSharedValue(0);
  
  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = savedScale.value * event.scale;
    })
    .onEnd(() => {
      if (scale.value < 1) {
        // Snap back to 1x if zoomed out
        scale.value = withSpring(1);
        savedScale.value = 1;
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        savedTranslateX.value = 0;
        savedTranslateY.value = 0;
      } else if (scale.value > 4) {
        // Cap at 4x zoom
        scale.value = withSpring(4);
        savedScale.value = 4;
      } else {
        savedScale.value = scale.value;
      }
    });
  
  const panGesture = Gesture.Pan()
    .minPointers(2)  // Only pan with 2 fingers (during pinch)
    .onUpdate((event) => {
      translateX.value = savedTranslateX.value + event.translationX;
      translateY.value = savedTranslateY.value + event.translationY;
    })
    .onEnd(() => {
      savedTranslateX.value = translateX.value;
      savedTranslateY.value = translateY.value;
    });
  
  // Compose gestures — pinch and pan run simultaneously
  const composedGesture = Gesture.Simultaneous(pinchGesture, panGesture);
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));
  
  return (
    <GestureDetector gesture={composedGesture}>
      <Animated.Image
        source={{ uri }}
        style={[{ width: '100%', aspectRatio: 1 }, animatedStyle]}
        resizeMode="contain"
      />
    </GestureDetector>
  );
}
```

### 10.4 Composing Gestures

Gesture Handler provides three composition modes:

```tsx
import { Gesture } from 'react-native-gesture-handler';

// SIMULTANEOUS — both gestures can be active at the same time
// Use for: pinch + pan (photo viewer), rotate + scale
const simultaneous = Gesture.Simultaneous(pinchGesture, panGesture);

// EXCLUSIVE — only one gesture wins (the first to activate)
// Use for: swipe left vs swipe right, tap vs long press
const exclusive = Gesture.Exclusive(longPressGesture, tapGesture);

// RACE — the first gesture to meet its activation criteria wins
// Use for: scroll vs swipe-to-dismiss
const race = Gesture.Race(swipeGesture, scrollGesture);
```

A real-world example — a card that can be tapped, long-pressed, or swiped:

```tsx
function InteractiveCard({ item, onTap, onLongPress, onDismiss }: InteractiveCardProps) {
  const translateX = useSharedValue(0);
  const opacity = useSharedValue(1);
  const cardScale = useSharedValue(1);
  
  const tapGesture = Gesture.Tap()
    .onBegin(() => {
      cardScale.value = withSpring(0.97, { damping: 15 });
    })
    .onFinalize(() => {
      cardScale.value = withSpring(1, { damping: 15 });
    })
    .onEnd(() => {
      runOnJS(onTap)(item.id);
    });
  
  const longPressGesture = Gesture.LongPress()
    .minDuration(500)
    .onStart(() => {
      cardScale.value = withSpring(0.95, { damping: 15 });
      runOnJS(onLongPress)(item.id);
    })
    .onEnd(() => {
      cardScale.value = withSpring(1, { damping: 15 });
    });
  
  const swipeGesture = Gesture.Pan()
    .activeOffsetX([-20, 20])  // Don't activate until 20px horizontal movement
    .failOffsetY([-10, 10])    // Fail if vertical movement > 10px (let scroll win)
    .onUpdate((event) => {
      translateX.value = event.translationX;
      opacity.value = interpolate(
        Math.abs(event.translationX),
        [0, 150],
        [1, 0.5],
        Extrapolation.CLAMP
      );
    })
    .onEnd((event) => {
      if (Math.abs(event.translationX) > 120) {
        const direction = event.translationX > 0 ? 1 : -1;
        translateX.value = withTiming(direction * 400, { duration: 200 });
        opacity.value = withTiming(0, { duration: 200 }, () => {
          runOnJS(onDismiss)(item.id);
        });
      } else {
        translateX.value = withSpring(0);
        opacity.value = withSpring(1);
      }
    });
  
  // Compose: swipe is exclusive with tap/longPress
  // (if you start swiping, tap/longPress won't fire)
  // Tap and longPress are exclusive with each other
  // (long press cancels tap)
  const composed = Gesture.Race(
    swipeGesture,
    Gesture.Exclusive(longPressGesture, tapGesture)
  );
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { scale: cardScale.value },
    ],
    opacity: opacity.value,
  }));
  
  return (
    <GestureDetector gesture={composed}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>{item.title}</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 10.5 Bottom Sheet (The Classic Pattern)

Every app needs a bottom sheet. Here's a production-quality one built with Gesture Handler and Reanimated:

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  interpolate,
  runOnJS,
} from 'react-native-reanimated';
import { Dimensions } from 'react-native';

const SCREEN_HEIGHT = Dimensions.get('window').height;
const SNAP_POINTS = [0, SCREEN_HEIGHT * 0.5, SCREEN_HEIGHT * 0.85];
// 0 = closed, 50% = half open, 85% = nearly full

function BottomSheet({ children, onClose }: BottomSheetProps) {
  const translateY = useSharedValue(SCREEN_HEIGHT);
  const savedTranslateY = useSharedValue(SCREEN_HEIGHT);
  const currentSnapIndex = useSharedValue(0);
  
  // Open the sheet to the first snap point
  const open = () => {
    translateY.value = withSpring(
      SCREEN_HEIGHT - SNAP_POINTS[1],
      { damping: 20, stiffness: 150 }
    );
    savedTranslateY.value = SCREEN_HEIGHT - SNAP_POINTS[1];
    currentSnapIndex.value = 1;
  };
  
  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      const newY = savedTranslateY.value + event.translationY;
      // Allow slight overscroll at top, free scroll at bottom
      translateY.value = Math.max(
        SCREEN_HEIGHT - SNAP_POINTS[SNAP_POINTS.length - 1] - 20,
        newY
      );
    })
    .onEnd((event) => {
      const currentY = translateY.value;
      const velocity = event.velocityY;
      
      // Find the nearest snap point based on position and velocity
      let targetSnap = 0;
      let minDistance = Infinity;
      
      for (let i = 0; i < SNAP_POINTS.length; i++) {
        const snapY = SCREEN_HEIGHT - SNAP_POINTS[i];
        const distance = Math.abs(currentY - snapY) - velocity * 0.1;
        if (distance < minDistance) {
          minDistance = distance;
          targetSnap = i;
        }
      }
      
      // Fast downward swipe = close
      if (velocity > 1000 && currentSnapIndex.value <= 1) {
        targetSnap = 0;
      }
      
      const targetY = SCREEN_HEIGHT - SNAP_POINTS[targetSnap];
      translateY.value = withSpring(targetY, { damping: 20, stiffness: 150 });
      savedTranslateY.value = targetY;
      currentSnapIndex.value = targetSnap;
      
      if (targetSnap === 0) {
        runOnJS(onClose)();
      }
    });
  
  const sheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));
  
  const backdropStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      translateY.value,
      [SCREEN_HEIGHT, SCREEN_HEIGHT - SNAP_POINTS[1]],
      [0, 0.5],
      Extrapolation.CLAMP
    );
    return { opacity };
  });
  
  return (
    <>
      <Animated.View style={[styles.backdrop, backdropStyle]} />
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
    left: 0,
    right: 0,
    height: SCREEN_HEIGHT,
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingTop: 8,
  },
  handle: {
    width: 36,
    height: 4,
    backgroundColor: '#D1D5DB',
    borderRadius: 2,
    alignSelf: 'center',
    marginBottom: 8,
  },
});
```

> **Pro tip:** In production, use `@gorhom/bottom-sheet` which is built on Gesture Handler and Reanimated. The code above is educational — it shows you what's happening under the hood. But the library handles edge cases you don't want to debug yourself (keyboard avoidance, nested scroll views, VirtualizedList compatibility).

---

## 11. EXPO-IMAGE: THE IMAGE COMPONENT YOU SHOULD BE USING

If you're still using React Native's built-in `<Image>` component, stop. `expo-image` is faster, has better caching, supports modern formats, and provides placeholder/transition features out of the box.

### 11.1 Why expo-image?

The built-in `<Image>` component has known issues:
- Flickering when images reload
- No disk caching on Android (only memory cache)
- No placeholder support
- No modern format support (WebP on iOS, AVIF)
- No progressive loading
- Memory management issues with large lists

`expo-image` is built on top of `SDWebImage` (iOS) and `Glide` (Android) — the same battle-tested native image libraries that Instagram and Twitter use.

```bash
npx expo install expo-image
```

### 11.2 Basic Usage

```tsx
import { Image } from 'expo-image';

// Drop-in replacement for RN Image (mostly)
function Avatar({ uri, size = 48 }: { uri: string; size?: number }) {
  return (
    <Image
      source={{ uri }}
      style={{ width: size, height: size, borderRadius: size / 2 }}
      contentFit="cover"           // Like resizeMode but better named
      transition={200}              // 200ms crossfade when image loads
      cachePolicy="memory-disk"     // Cache in memory AND on disk
      recyclingKey={uri}            // Helps with list recycling
    />
  );
}
```

### 11.3 Blurhash Placeholders

Blurhash is a compact representation of a placeholder for an image. It's a ~30 character string that produces a blurred preview of the image. You generate it on the server, store it alongside the image URL, and display it while the full image loads.

```tsx
import { Image } from 'expo-image';

function ProductImage({ product }: { product: Product }) {
  return (
    <Image
      source={{ uri: product.imageUrl }}
      placeholder={{ blurhash: product.imageBlurhash }}
      // product.imageBlurhash is something like "LKO2?U%2Tw=w]~RBVZRi};RPxuwH"
      
      contentFit="cover"
      transition={300}
      style={styles.productImage}
    />
  );
}

// In a FlatList — use recyclingKey for better performance
function ProductList({ products }: { products: Product[] }) {
  return (
    <FlatList
      data={products}
      renderItem={({ item }) => (
        <Image
          source={{ uri: item.imageUrl }}
          placeholder={{ blurhash: item.blurhash }}
          contentFit="cover"
          transition={200}
          recyclingKey={item.id}     // Prevents image flickering during list scroll
          style={styles.listImage}
          cachePolicy="memory-disk"
        />
      )}
      keyExtractor={(item) => item.id}
    />
  );
}
```

### 11.4 Generating Blurhash on the Server

You generate blurhash strings on your backend when images are uploaded:

```typescript
// Server-side (Node.js) — run when image is uploaded
import { encode } from 'blurhash';
import sharp from 'sharp';

async function generateBlurhash(imagePath: string): Promise<string> {
  const { data, info } = await sharp(imagePath)
    .raw()
    .ensureAlpha()
    .resize(32, 32, { fit: 'inside' })  // Small size is fine — it's a blur
    .toBuffer({ resolveWithObject: true });
  
  return encode(
    new Uint8ClampedArray(data),
    info.width,
    info.height,
    4,  // x components
    3,  // y components
  );
  // Returns something like: "LKO2?U%2Tw=w]~RBVZRi};RPxuwH"
}

// Store the blurhash alongside the image URL in your database
// { imageUrl: "https://cdn.example.com/product-123.jpg", blurhash: "LKO2?U%2..." }
```

### 11.5 expo-image Performance Tips

```tsx
// 1. ALWAYS set explicit dimensions
// expo-image is much faster when it doesn't have to calculate layout
<Image
  source={{ uri }}
  style={{ width: 200, height: 200 }}  // Explicit > flex
  contentFit="cover"
/>

// 2. Use cachePolicy wisely
// "memory-disk" — default, best for most cases
// "memory" — for large images you don't want to cache on disk
// "disk" — for images that are rarely accessed but should persist
// "none" — for one-off images (like camera captures)

// 3. Prefetch images before they're needed
import { Image } from 'expo-image';

// Prefetch the next page of images
async function prefetchImages(urls: string[]) {
  await Promise.all(
    urls.map((url) => Image.prefetch(url))
  );
}

// 4. Use the correct contentFit
// "cover" — fills the container, may crop (profile pics, backgrounds)
// "contain" — fits inside the container, may letterbox (product photos)
// "fill" — stretches to fill (almost never what you want)
// "none" — natural size, may overflow

// 5. Clear cache when needed (e.g., after a user logs out)
import { Image } from 'expo-image';
await Image.clearDiskCache();
await Image.clearMemoryCache();
```

### 11.6 Image Optimization Strategy

A complete image strategy for a production app:

```typescript
// utils/image.ts

import { ImageSource } from 'expo-image';

// Generate responsive image URLs (assumes your CDN supports transforms)
function getResponsiveImageUrl(
  baseUrl: string, 
  width: number,
  format: 'webp' | 'avif' | 'jpg' = 'webp'
): string {
  // Example for Cloudinary
  return baseUrl.replace(
    '/upload/',
    `/upload/w_${width},f_${format},q_auto/`
  );
  
  // Example for Imgix
  // return `${baseUrl}?w=${width}&fm=${format}&q=75&auto=format`;
}

// Create an image source with appropriate sizing
function createImageSource(
  url: string,
  displayWidth: number,
  blurhash?: string,
): { source: ImageSource; placeholder?: { blurhash: string } } {
  const pixelRatio = PixelRatio.get();
  const actualWidth = Math.round(displayWidth * pixelRatio);
  
  // Cap at reasonable max (no need for 4000px images)
  const clampedWidth = Math.min(actualWidth, 1200);
  
  return {
    source: { uri: getResponsiveImageUrl(url, clampedWidth) },
    ...(blurhash ? { placeholder: { blurhash } } : {}),
  };
}
```

---

## 12. PUTTING IT ALL TOGETHER: A STYLED, ANIMATED SCREEN

Let's build a real screen that combines styling (NativeWind), animation (Reanimated), gestures (Gesture Handler), and image handling (expo-image). This is a product detail screen with a parallax header, gesture-driven image viewer, and animated content sections.

```tsx
import { View, Text, Pressable, ScrollView, Dimensions } from 'react-native';
import { Image } from 'expo-image';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useAnimatedScrollHandler,
  interpolate,
  Extrapolation,
  FadeInDown,
  withSpring,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const HEADER_HEIGHT = 400;
const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface ProductDetailProps {
  product: {
    id: string;
    name: string;
    price: number;
    description: string;
    images: { url: string; blurhash: string }[];
    rating: number;
    reviewCount: number;
  };
}

function ProductDetailScreen({ product }: ProductDetailProps) {
  const scrollY = useSharedValue(0);
  const imageIndex = useSharedValue(0);
  const imageTranslateX = useSharedValue(0);
  
  // Scroll handler for parallax
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });
  
  // Parallax header
  const headerStyle = useAnimatedStyle(() => {
    const scale = interpolate(
      scrollY.value,
      [-100, 0],
      [1.3, 1],
      Extrapolation.CLAMP
    );
    const translateY = interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT],
      [0, -HEADER_HEIGHT * 0.5],
      Extrapolation.CLAMP
    );
    return {
      transform: [{ scale }, { translateY }],
    };
  });
  
  // Swipe between images
  const swipeGesture = Gesture.Pan()
    .activeOffsetX([-15, 15])
    .failOffsetY([-5, 5])
    .onUpdate((event) => {
      imageTranslateX.value = 
        -imageIndex.value * SCREEN_WIDTH + event.translationX;
    })
    .onEnd((event) => {
      const threshold = SCREEN_WIDTH * 0.3;
      let newIndex = imageIndex.value;
      
      if (event.translationX < -threshold && newIndex < product.images.length - 1) {
        newIndex++;
      } else if (event.translationX > threshold && newIndex > 0) {
        newIndex--;
      }
      
      imageIndex.value = newIndex;
      imageTranslateX.value = withSpring(-newIndex * SCREEN_WIDTH, {
        damping: 20,
        stiffness: 200,
      });
    });
  
  const imageCarouselStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: imageTranslateX.value }],
  }));
  
  // Nav bar opacity based on scroll
  const navBarStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      scrollY.value,
      [HEADER_HEIGHT - 120, HEADER_HEIGHT - 60],
      [0, 1],
      Extrapolation.CLAMP
    );
    return { opacity };
  });
  
  return (
    <View className="flex-1 bg-white dark:bg-gray-950">
      {/* Floating nav bar */}
      <Animated.View 
        style={navBarStyle}
        className="absolute top-0 left-0 right-0 z-10 bg-white/95 dark:bg-gray-950/95
                   pt-safe px-4 pb-3 border-b border-gray-200 dark:border-gray-800"
      >
        <Text className="text-lg font-semibold text-gray-900 dark:text-white text-center">
          {product.name}
        </Text>
      </Animated.View>
      
      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        className="flex-1"
      >
        {/* Parallax image header */}
        <Animated.View style={[{ height: HEADER_HEIGHT, overflow: 'hidden' }, headerStyle]}>
          <GestureDetector gesture={swipeGesture}>
            <Animated.View style={[{ flexDirection: 'row', width: SCREEN_WIDTH * product.images.length }, imageCarouselStyle]}>
              {product.images.map((img, index) => (
                <Image
                  key={index}
                  source={{ uri: img.url }}
                  placeholder={{ blurhash: img.blurhash }}
                  contentFit="cover"
                  transition={200}
                  style={{ width: SCREEN_WIDTH, height: HEADER_HEIGHT }}
                />
              ))}
            </Animated.View>
          </GestureDetector>
        </Animated.View>
        
        {/* Product info — staggered entrance */}
        <View className="px-4 pt-4 pb-32">
          <Animated.View entering={FadeInDown.delay(100).springify()}>
            <Text className="text-2xl font-bold text-gray-900 dark:text-white">
              {product.name}
            </Text>
            <View className="flex-row items-center mt-2 gap-2">
              <Text className="text-xl font-semibold text-emerald-600 dark:text-emerald-400">
                ${product.price}
              </Text>
              <View className="flex-row items-center bg-amber-50 dark:bg-amber-950 px-2 py-1 rounded-lg">
                <Text className="text-amber-600 dark:text-amber-400 text-sm font-medium">
                  {product.rating} ({product.reviewCount})
                </Text>
              </View>
            </View>
          </Animated.View>
          
          <Animated.View entering={FadeInDown.delay(200).springify()}>
            <Text className="text-base text-gray-600 dark:text-gray-300 mt-4 leading-6">
              {product.description}
            </Text>
          </Animated.View>
          
          <Animated.View entering={FadeInDown.delay(300).springify()}>
            <Pressable className="bg-brand-600 rounded-2xl py-4 items-center mt-6
                                  active:bg-brand-700 active:scale-[0.98]">
              <Text className="text-white font-semibold text-lg">
                Add to Cart
              </Text>
            </Pressable>
          </Animated.View>
        </View>
      </Animated.ScrollView>
    </View>
  );
}
```

This is what a 100x frontend architect's screen looks like. Not because it's clever, but because every decision is intentional: the parallax creates depth, the gesture-driven carousel avoids a third-party dependency, the staggered entrances draw the eye, the spring animations feel physical, and the image component handles caching and placeholders automatically.

---

## 13. COMMON ANIMATION RECIPES

Here are the animations you'll build most often, ready to copy:

### 13.1 Skeleton Loading

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  interpolateColor,
} from 'react-native-reanimated';
import { useEffect } from 'react';

function Skeleton({ width, height, borderRadius = 8 }: SkeletonProps) {
  const progress = useSharedValue(0);
  
  useEffect(() => {
    progress.value = withRepeat(
      withTiming(1, { duration: 1000 }),
      -1,    // Infinite repeats
      true   // Reverse on each iteration
    );
  }, []);
  
  const animatedStyle = useAnimatedStyle(() => {
    const backgroundColor = interpolateColor(
      progress.value,
      [0, 1],
      ['#E5E7EB', '#F3F4F6']  // Gray-200 to Gray-100
    );
    return { backgroundColor };
  });
  
  return (
    <Animated.View
      style={[{ width, height, borderRadius }, animatedStyle]}
    />
  );
}

// Usage
function ProductCardSkeleton() {
  return (
    <View className="bg-white dark:bg-gray-900 rounded-2xl p-4">
      <Skeleton width="100%" height={200} borderRadius={12} />
      <Skeleton width="70%" height={20} borderRadius={6} style={{ marginTop: 12 }} />
      <Skeleton width="40%" height={16} borderRadius={6} style={{ marginTop: 8 }} />
    </View>
  );
}
```

### 13.2 Pull to Refresh with Custom Animation

```tsx
function AnimatedRefreshControl({ refreshing, onRefresh }: RefreshProps) {
  const rotation = useSharedValue(0);
  const scale = useSharedValue(0.8);
  
  useEffect(() => {
    if (refreshing) {
      rotation.value = withRepeat(
        withTiming(360, { duration: 800 }),
        -1
      );
      scale.value = withSpring(1);
    } else {
      scale.value = withSpring(0.8);
    }
  }, [refreshing]);
  
  const spinnerStyle = useAnimatedStyle(() => ({
    transform: [
      { rotate: `${rotation.value}deg` },
      { scale: scale.value },
    ],
  }));
  
  return (
    <RefreshControl
      refreshing={refreshing}
      onRefresh={onRefresh}
      tintColor="transparent"  // Hide default spinner
    >
      <Animated.View style={[styles.spinner, spinnerStyle]}>
        <LoadingIcon />
      </Animated.View>
    </RefreshControl>
  );
}
```

### 13.3 Animated Counter

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  useAnimatedProps,
} from 'react-native-reanimated';
import { TextInput } from 'react-native';

const AnimatedTextInput = Animated.createAnimatedComponent(TextInput);

function AnimatedCounter({ value }: { value: number }) {
  const animatedValue = useSharedValue(0);
  
  useEffect(() => {
    animatedValue.value = withTiming(value, { duration: 800 });
  }, [value]);
  
  const animatedProps = useAnimatedProps(() => ({
    text: `${Math.round(animatedValue.value)}`,
    defaultValue: `${Math.round(animatedValue.value)}`,
  }));
  
  return (
    <AnimatedTextInput
      animatedProps={animatedProps}
      editable={false}
      style={styles.counter}
    />
  );
}
```

### 13.4 Swipeable Row (Inbox Style)

```tsx
function SwipeableRow({ children, onDelete, onArchive }: SwipeableRowProps) {
  const translateX = useSharedValue(0);
  
  const panGesture = Gesture.Pan()
    .activeOffsetX([-10, 10])
    .failOffsetY([-5, 5])
    .onUpdate((event) => {
      translateX.value = event.translationX;
    })
    .onEnd((event) => {
      const velocity = event.velocityX;
      
      // Swipe left past threshold — delete
      if (translateX.value < -150 || velocity < -800) {
        translateX.value = withTiming(-400, { duration: 200 });
        runOnJS(onDelete)();
      }
      // Swipe right past threshold — archive
      else if (translateX.value > 150 || velocity > 800) {
        translateX.value = withTiming(400, { duration: 200 });
        runOnJS(onArchive)();
      }
      // Spring back
      else {
        translateX.value = withSpring(0, { damping: 15 });
      }
    });
  
  const rowStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));
  
  const deleteActionStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      translateX.value,
      [-150, -50, 0],
      [1, 0.5, 0],
      Extrapolation.CLAMP
    );
    return { opacity };
  });
  
  const archiveActionStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      translateX.value,
      [0, 50, 150],
      [0, 0.5, 1],
      Extrapolation.CLAMP
    );
    return { opacity };
  });
  
  return (
    <View>
      {/* Background actions */}
      <View style={StyleSheet.absoluteFill}>
        <Animated.View style={[styles.deleteAction, deleteActionStyle]}>
          <TrashIcon color="white" />
        </Animated.View>
        <Animated.View style={[styles.archiveAction, archiveActionStyle]}>
          <ArchiveIcon color="white" />
        </Animated.View>
      </View>
      
      {/* Foreground content */}
      <GestureDetector gesture={panGesture}>
        <Animated.View style={rowStyle}>
          {children}
        </Animated.View>
      </GestureDetector>
    </View>
  );
}
```

---

## 14. PERFORMANCE CHECKLIST

Before you ship your styled, animated app, run through this checklist:

### Styling Performance
- [ ] No dynamic `StyleSheet.create` calls inside render functions (unless using Unistyles)
- [ ] Theme context doesn't cause full-tree re-renders (use memoization or Unistyles JSI)
- [ ] Images use `expo-image` with appropriate `cachePolicy`
- [ ] All images have explicit dimensions (width/height, not just flex)
- [ ] Blurhash strings are stored alongside image URLs in your API

### Animation Performance
- [ ] All animations use Reanimated (not built-in `Animated` for anything complex)
- [ ] No `console.log` inside worklets (kills performance)
- [ ] `useAnimatedStyle` callbacks don't access React state directly
- [ ] Spring configs are defined outside components (not re-created each render)
- [ ] `scrollEventThrottle={16}` on all animated ScrollViews
- [ ] Shared values are not being read on the JS thread in hot paths

### Gesture Performance
- [ ] `GestureHandlerRootView` wraps your app (usually in root layout)
- [ ] Gestures use `activeOffset` and `failOffset` to avoid conflicts with scroll
- [ ] Pan gestures on lists use `failOffsetY` to not steal vertical scroll
- [ ] Complex gesture compositions use `Simultaneous`, `Exclusive`, or `Race` appropriately

### Responsive Design
- [ ] No hard-coded widths that overflow on small phones (test on 375pt width)
- [ ] Tablet layouts are tested (test on iPad Mini 744pt)
- [ ] Dynamic Type is tested at 200% scale
- [ ] Safe areas are handled (notch, home indicator, landscape)
- [ ] Orientation changes don't break layout

---

## Key Takeaways

1. **Pick a styling approach based on your constraints, not trends.** NativeWind if you know Tailwind. Tamagui if you need web + native. Unistyles if you want peak native performance. StyleSheet.create if you don't need any of the extras.

2. **Design tokens are not optional at scale.** The three-tier system (primitives, semantic, component) pays for itself the first time a brand color changes.

3. **Reanimated v3 + Gesture Handler v2 is the standard.** Don't fight it. Learn worklets, shared values, and gesture composition. Every mobile interaction worth building uses them.

4. **expo-image is a drop-in upgrade.** Switch from `<Image>` to `expo-image` today. Add blurhash placeholders. Your users will notice.

5. **Animations earn trust.** A well-animated app feels professional. A janky app feels broken. Invest in the 60fps frame budget — spring animations on the UI thread, always.

6. **Test on real devices.** Animations that look smooth in the simulator may stutter on a mid-range Android phone. Responsive layouts that work on your iPhone 15 Pro may overflow on an iPhone SE. Test on the devices your users actually have.

---

**Next up:** [Chapter 9: Data Layer Foundations] — where we tackle state management, API communication, and offline-first patterns. Because the prettiest UI is useless if it's showing stale data.
