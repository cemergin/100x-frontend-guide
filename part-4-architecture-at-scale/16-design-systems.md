<!--
  CHAPTER: 17
  TITLE: Design Systems & Component Libraries
  PART: IV — Architecture at Scale
  PREREQS: Chapter 15
  KEY_TOPICS: design tokens, shadcn/ui, Tamagui, NativeWind, Storybook, CVA, accessibility, compound components, component library, monorepo
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 16: Design Systems & Component Libraries

> **Part IV — Architecture at Scale** | Prerequisites: Chapter 15 | Difficulty: Intermediate

<details>
<summary><strong>TL;DR</strong></summary>

- Design tokens (color, spacing, typography, radii) are the atomic foundation; centralize them in a shared package and derive platform-specific values from one source of truth
- Building a full design system from scratch is almost always wrong for small teams; adopt shadcn/ui for web, Tamagui or NativeWind for native, and build only the pieces unique to your product
- The compound component pattern (composable sub-components like `<Select.Trigger>`, `<Select.Content>`) produces the most flexible and accessible component APIs
- Accessibility is a first-class concern, not a retrofit; every interactive component needs keyboard navigation, screen reader labels, focus management, and sufficient color contrast
- Storybook for React Native lets you develop and test components in isolation; wire it into CI for visual regression testing with Chromatic

</details>

Here's a truth that most teams learn the hard way: **the cost of not having a design system is invisible until it's catastrophic.** It shows up as twelve slightly different shades of blue across your app. It shows up as buttons that have 5px of padding on one screen and 8px on another. It shows up as a "quick redesign" that takes three months because every component was hand-crafted with inline styles that nobody fully understands.

A design system isn't a Figma file. It isn't a component library. It isn't a set of color variables. It's all of those things working together as a **single source of truth** that bridges design and engineering, web and mobile, one team and twenty teams.

But here's the thing most design system articles won't tell you: **building a design system from scratch is almost always a mistake for startups and small teams.** The ROI doesn't kick in until you have multiple products, multiple platforms, or multiple teams consuming the system. Before that point, you need a pragmatic approach — adopt community tools, establish conventions, and build only the pieces that are uniquely yours.

This chapter takes the pragmatic path. We'll start with design tokens (the foundation), choose the right component abstraction for your platform, and build a component library that scales from one developer to fifty — in a monorepo, with accessibility baked in, and with the tooling to keep it consistent as it grows.

### In This Chapter
- Design Tokens: The Foundation of Everything
- Cross-Platform Token Architecture (color, spacing, typography)
- shadcn/ui for Web — the anti-library library
- Tamagui — universal components for web and native
- NativeWind — Tailwind CSS for React Native
- Storybook for React Native
- CVA (Class Variance Authority) for Variant Management
- The Compound Component Pattern
- Accessibility (a11y) as a First-Class Concern
- Building a Component Library in a Monorepo
- The Design System Decision Framework

### Related Chapters
- [Ch 15: Monorepo Architecture] — where your design system lives
- [Ch 3: The Rendering Pipeline] — understanding how components render
- [Ch 13: Performance Optimization] — component performance patterns
- [Ch 17: Testing Strategies] — testing your component library

---

## 1. DESIGN TOKENS: THE FOUNDATION OF EVERYTHING

Design tokens are the atomic values of your design system: colors, spacing, typography, border radii, shadows, breakpoints, animation durations. They're called "tokens" because they're symbolic — `color.primary` means the same thing everywhere, whether "everywhere" is a React Native app, a Next.js site, a Figma file, or an email template.

### Why Tokens Matter

Without tokens:
```typescript
// Screen A (written by Dev 1)
<View style={{ padding: 16, backgroundColor: '#3B82F6' }}>
  <Text style={{ fontSize: 14, color: '#FFFFFF' }}>Hello</Text>
</View>

// Screen B (written by Dev 2)
<View style={{ padding: 15, backgroundColor: '#3B83F6' }}>  {/* Slightly different! */}
  <Text style={{ fontSize: 13, color: '#FFF' }}>Hello</Text>  {/* Also different! */}
</View>
```

With tokens:
```typescript
// Both developers use the same tokens
<View style={{ padding: tokens.spacing.md, backgroundColor: tokens.color.primary }}>
  <Text style={{ fontSize: tokens.fontSize.sm, color: tokens.color.onPrimary }}>Hello</Text>
</View>
```

### Token Architecture

```
┌────────────────────────────────────────────────────────────┐
│  TOKEN HIERARCHY                                            │
│                                                             │
│  GLOBAL TOKENS (raw values)                                 │
│  ├── blue-500: '#3B82F6'                                    │
│  ├── gray-900: '#111827'                                    │
│  ├── space-4: 16                                            │
│  └── font-size-sm: 14                                       │
│                                                             │
│  SEMANTIC TOKENS (meaning-based aliases)                    │
│  ├── color.primary: blue-500                                │
│  ├── color.text: gray-900                                   │
│  ├── spacing.md: space-4                                    │
│  └── typography.body: font-size-sm                          │
│                                                             │
│  COMPONENT TOKENS (component-specific)                      │
│  ├── button.backgroundColor: color.primary                  │
│  ├── button.paddingX: spacing.md                            │
│  ├── card.borderRadius: radius.lg                           │
│  └── input.borderColor: color.border                        │
└────────────────────────────────────────────────────────────┘
```

The three levels serve different audiences:
- **Global tokens**: Designers define these (the palette, the type scale, the spacing scale)
- **Semantic tokens**: Designers + engineers map meaning to values (what's "primary"?)
- **Component tokens**: Engineers define these (how does "primary" apply to a button?)

### Implementing Tokens in TypeScript

```typescript
// tokens/colors.ts
export const palette = {
  blue: {
    50: '#EFF6FF',
    100: '#DBEAFE',
    200: '#BFDBFE',
    300: '#93C5FD',
    400: '#60A5FA',
    500: '#3B82F6',
    600: '#2563EB',
    700: '#1D4ED8',
    800: '#1E40AF',
    900: '#1E3A8A',
    950: '#172554',
  },
  gray: {
    50: '#F9FAFB',
    100: '#F3F4F6',
    200: '#E5E7EB',
    300: '#D1D5DB',
    400: '#9CA3AF',
    500: '#6B7280',
    600: '#4B5563',
    700: '#374151',
    800: '#1F2937',
    900: '#111827',
    950: '#030712',
  },
  // ... other colors
} as const;

// tokens/semantic.ts
export const lightTheme = {
  color: {
    primary: palette.blue[600],
    primaryHover: palette.blue[700],
    onPrimary: '#FFFFFF',

    background: '#FFFFFF',
    backgroundSecondary: palette.gray[50],
    surface: '#FFFFFF',
    surfaceHover: palette.gray[50],

    text: palette.gray[900],
    textSecondary: palette.gray[500],
    textTertiary: palette.gray[400],

    border: palette.gray[200],
    borderFocus: palette.blue[500],

    error: '#DC2626',
    warning: '#F59E0B',
    success: '#16A34A',
    info: palette.blue[500],
  },

  spacing: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
    '2xl': 48,
    '3xl': 64,
  },

  borderRadius: {
    sm: 4,
    md: 8,
    lg: 12,
    xl: 16,
    full: 9999,
  },

  fontSize: {
    xs: 12,
    sm: 14,
    md: 16,
    lg: 18,
    xl: 20,
    '2xl': 24,
    '3xl': 30,
    '4xl': 36,
  },

  fontWeight: {
    normal: '400' as const,
    medium: '500' as const,
    semibold: '600' as const,
    bold: '700' as const,
  },

  lineHeight: {
    tight: 1.25,
    normal: 1.5,
    relaxed: 1.75,
  },

  shadow: {
    sm: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 1 },
      shadowOpacity: 0.05,
      shadowRadius: 2,
      elevation: 1,
    },
    md: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 4 },
      shadowOpacity: 0.1,
      shadowRadius: 6,
      elevation: 3,
    },
    lg: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 10 },
      shadowOpacity: 0.15,
      shadowRadius: 15,
      elevation: 5,
    },
  },
} as const;

export const darkTheme = {
  ...lightTheme,
  color: {
    ...lightTheme.color,
    primary: palette.blue[500],
    primaryHover: palette.blue[400],

    background: palette.gray[950],
    backgroundSecondary: palette.gray[900],
    surface: palette.gray[900],
    surfaceHover: palette.gray[800],

    text: palette.gray[50],
    textSecondary: palette.gray[400],
    textTertiary: palette.gray[500],

    border: palette.gray[800],
  },
} as const;

export type Theme = typeof lightTheme;
```

### Making Tokens Available App-Wide

```typescript
// providers/ThemeProvider.tsx
import { createContext, useContext, useMemo } from 'react';
import { useColorScheme } from 'react-native';
import { lightTheme, darkTheme, type Theme } from '../tokens/semantic';

const ThemeContext = createContext<Theme>(lightTheme);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const colorScheme = useColorScheme();
  const theme = useMemo(
    () => (colorScheme === 'dark' ? darkTheme : lightTheme),
    [colorScheme]
  );

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme(): Theme {
  return useContext(ThemeContext);
}

// Usage in components
function MyComponent() {
  const theme = useTheme();

  return (
    <View style={{ padding: theme.spacing.md, backgroundColor: theme.color.surface }}>
      <Text style={{ color: theme.color.text, fontSize: theme.fontSize.md }}>
        Themed content
      </Text>
    </View>
  );
}
```

---

## 2. SHADCN/UI FOR WEB

shadcn/ui is not a component library. It's a collection of re-usable components that you copy into your project and own. This distinction matters enormously.

### Why shadcn/ui Won

Traditional component libraries (Material UI, Chakra UI, Ant Design) give you npm packages. You install them, import components, and pass props. The components are a black box — you can't easily modify their internals, their styling approach is baked in, and upgrading means hoping nothing breaks.

shadcn/ui takes a different approach:

```
Traditional Library:
  npm install @chakra-ui/react → node_modules → import → use
  Problem: You can't easily change the button's internal structure
  Problem: Library's styling decisions conflict with your design

shadcn/ui:
  npx shadcn@latest add button → copies button.tsx to your project
  Your code: Full control, modify anything, no dependency
  Updates: You choose when and what to update
```

### Setting Up shadcn/ui

```bash
# In a Next.js project
npx shadcn@latest init

# Add components as needed
npx shadcn@latest add button
npx shadcn@latest add card
npx shadcn@latest add input
npx shadcn@latest add dialog
```

This copies component files into your project (typically `components/ui/`). Each component uses:
- **Tailwind CSS** for styling
- **Radix UI** primitives for behavior (accessibility, keyboard navigation)
- **CVA** (Class Variance Authority) for variant management

### Anatomy of a shadcn/ui Component

```typescript
// components/ui/button.tsx (generated by shadcn, then customized by you)
import { Slot } from '@radix-ui/react-slot';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  // Base styles (applied to all variants)
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button';
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  }
);
Button.displayName = 'Button';

export { Button, buttonVariants };
```

### Why This Matters for Cross-Platform Teams

If your web app uses shadcn/ui, your React Native app can follow the same design token architecture and variant patterns — just with different rendering primitives. The *design decisions* (what variants exist, what sizes are available, what colors mean) stay consistent. The *implementation* differs.

---

## 3. TAMAGUI: UNIVERSAL COMPONENTS

Tamagui is a UI framework that generates optimized styles for both web and React Native from a single component definition. It's the closest thing to "write once, render everywhere" that actually works well.

### Why Tamagui

```
┌──────────────────────────────────────────────────────────┐
│  THE CROSS-PLATFORM STYLING PROBLEM                       │
│                                                           │
│  Web:     CSS classes, flexbox, grid, media queries       │
│  Native:  StyleSheet.create, Yoga flexbox, no CSS         │
│                                                           │
│  Options:                                                 │
│  1. Two codebases (web + native) → inconsistency          │
│  2. React Native Web → compromises on web performance     │
│  3. Tamagui → compiles to optimal output per platform     │
│                                                           │
│  Tamagui on Web:   → extracts to CSS at build time        │
│  Tamagui on Native: → generates optimized StyleSheet      │
│  Result: Same API, optimal performance on both            │
└──────────────────────────────────────────────────────────┘
```

### Tamagui Setup

```typescript
// tamagui.config.ts
import { createTamagui, createTokens } from 'tamagui';

const tokens = createTokens({
  color: {
    primary: '#3B82F6',
    primaryHover: '#2563EB',
    background: '#FFFFFF',
    backgroundDark: '#030712',
    text: '#111827',
    textDark: '#F9FAFB',
    // ... more colors
  },
  space: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 32,
    '2xl': 48,
  },
  size: {
    sm: 32,
    md: 40,
    lg: 48,
    xl: 56,
  },
  radius: {
    sm: 4,
    md: 8,
    lg: 12,
    xl: 16,
    full: 9999,
  },
  // ... more token categories
});

const config = createTamagui({
  tokens,
  themes: {
    light: {
      background: tokens.color.background,
      color: tokens.color.text,
      // ... map semantic names to token values
    },
    dark: {
      background: tokens.color.backgroundDark,
      color: tokens.color.textDark,
    },
  },
  // ... fonts, media queries, shorthands
});

export default config;
export type AppConfig = typeof config;

// Ensure types work
declare module 'tamagui' {
  interface TamaguiCustomConfig extends AppConfig {}
}
```

### Building Universal Components with Tamagui

```typescript
// components/Card.tsx
import { styled, YStack, XStack, Text, type GetProps } from 'tamagui';

const CardFrame = styled(YStack, {
  name: 'Card',
  backgroundColor: '$background',
  borderRadius: '$lg',
  padding: '$md',
  borderWidth: 1,
  borderColor: '$borderColor',

  // Variants
  variants: {
    elevated: {
      true: {
        shadowColor: '$shadowColor',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 8,
        elevation: 3,
      },
    },
    pressable: {
      true: {
        pressStyle: {
          scale: 0.98,
          opacity: 0.9,
        },
        cursor: 'pointer',
      },
    },
    size: {
      sm: { padding: '$sm' },
      md: { padding: '$md' },
      lg: { padding: '$lg' },
    },
  } as const,

  defaultVariants: {
    size: 'md',
  },
});

const CardHeader = styled(XStack, {
  name: 'CardHeader',
  justifyContent: 'space-between',
  alignItems: 'center',
  marginBottom: '$sm',
});

const CardTitle = styled(Text, {
  name: 'CardTitle',
  fontSize: '$lg',
  fontWeight: '$semibold',
  color: '$color',
});

const CardContent = styled(YStack, {
  name: 'CardContent',
  gap: '$sm',
});

// Export as a compound component
export const Card = Object.assign(CardFrame, {
  Header: CardHeader,
  Title: CardTitle,
  Content: CardContent,
});

// Type export
export type CardProps = GetProps<typeof CardFrame>;

// Usage (works on web AND native)
function ProductCard({ product }: { product: Product }) {
  return (
    <Card elevated pressable onPress={() => navigate(product.id)}>
      <Card.Header>
        <Card.Title>{product.name}</Card.Title>
        <PriceTag price={product.price} />
      </Card.Header>
      <Card.Content>
        <Text color="$textSecondary">{product.description}</Text>
      </Card.Content>
    </Card>
  );
}
```

### Tamagui's Compiler Optimization

The magic of Tamagui is its compiler. At build time, it analyzes your components and extracts static styles:

```typescript
// What you write:
<YStack padding="$md" backgroundColor="$background" borderRadius="$lg">
  <Text color="$text" fontSize="$md">Hello</Text>
</YStack>

// What Tamagui compiles to on web:
<div className="t_YStack _padding-md _bg-background _br-lg">
  <span className="t_Text _color-text _fs-md">Hello</span>
</div>
// With corresponding CSS classes (no runtime style computation!)

// What Tamagui compiles to on native:
<View style={compiledStyles.container}>
  <Text style={compiledStyles.text}>Hello</Text>
</View>
// With pre-computed StyleSheet objects
```

This means Tamagui components have near-zero runtime overhead on both platforms.

---

## 4. NATIVEWIND: TAILWIND CSS FOR REACT NATIVE

NativeWind brings Tailwind CSS to React Native. If your team already knows Tailwind from web development, NativeWind lets them use the same mental model on mobile.

### NativeWind v4

```typescript
// NativeWind v4 uses the `className` prop directly
import { View, Text, Pressable } from 'react-native';

function ProductCard({ product }: { product: Product }) {
  return (
    <View className="bg-white dark:bg-gray-900 rounded-xl p-4 shadow-md">
      <Text className="text-lg font-semibold text-gray-900 dark:text-white">
        {product.name}
      </Text>
      <Text className="text-sm text-gray-500 dark:text-gray-400 mt-1">
        {product.description}
      </Text>
      <Pressable className="mt-4 bg-blue-600 rounded-lg py-2 px-4 active:bg-blue-700">
        <Text className="text-white text-center font-medium">
          Add to Cart
        </Text>
      </Pressable>
    </View>
  );
}
```

### NativeWind vs Tamagui

```
┌────────────────────────────┬─────────────────────────────────┐
│  NATIVEWIND                 │  TAMAGUI                        │
├────────────────────────────┼─────────────────────────────────┤
│  Tailwind CSS syntax        │  Custom token/theme system       │
│  className strings          │  Props-based styling             │
│  Great for web devs         │  Great for RN devs               │
│  No build-time extraction   │  Build-time CSS extraction       │
│  Simpler mental model       │  More powerful theming           │
│  Community Tailwind plugins │  Built-in animation support      │
│  Best if: web-first team    │  Best if: performance-critical   │
│  using Tailwind already     │  cross-platform app              │
└────────────────────────────┴─────────────────────────────────┘
```

### NativeWind with CVA

You can combine NativeWind with CVA for variant management:

```typescript
// components/Button.tsx
import { cva, type VariantProps } from 'class-variance-authority';
import { Pressable, Text, type PressableProps } from 'react-native';
import { cn } from '@/lib/utils';

const buttonVariants = cva(
  'flex-row items-center justify-center rounded-lg font-medium',
  {
    variants: {
      variant: {
        default: 'bg-blue-600 active:bg-blue-700',
        secondary: 'bg-gray-100 dark:bg-gray-800 active:bg-gray-200',
        destructive: 'bg-red-600 active:bg-red-700',
        outline: 'border border-gray-300 dark:border-gray-700 active:bg-gray-50',
        ghost: 'active:bg-gray-100 dark:active:bg-gray-800',
      },
      size: {
        sm: 'h-9 px-3',
        md: 'h-11 px-4',
        lg: 'h-13 px-6',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'md',
    },
  }
);

const textVariants = cva('font-medium', {
  variants: {
    variant: {
      default: 'text-white',
      secondary: 'text-gray-900 dark:text-gray-100',
      destructive: 'text-white',
      outline: 'text-gray-900 dark:text-gray-100',
      ghost: 'text-gray-900 dark:text-gray-100',
    },
    size: {
      sm: 'text-sm',
      md: 'text-base',
      lg: 'text-lg',
    },
  },
  defaultVariants: {
    variant: 'default',
    size: 'md',
  },
});

interface ButtonProps extends PressableProps, VariantProps<typeof buttonVariants> {
  children: string;
  className?: string;
}

export function Button({ children, variant, size, className, ...props }: ButtonProps) {
  return (
    <Pressable
      className={cn(buttonVariants({ variant, size }), className)}
      {...props}
    >
      <Text className={textVariants({ variant, size })}>
        {children}
      </Text>
    </Pressable>
  );
}
```

---

## 5. STORYBOOK FOR REACT NATIVE

Storybook is the industry standard for developing, documenting, and testing components in isolation. The React Native integration has matured significantly.

### Setup

```bash
npx storybook@latest init --type react_native
```

### Writing Stories

```typescript
// components/Button/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  argTypes: {
    variant: {
      control: 'select',
      options: ['default', 'secondary', 'destructive', 'outline', 'ghost'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
    disabled: {
      control: 'boolean',
    },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = {
  args: {
    children: 'Button',
    variant: 'default',
    size: 'md',
  },
};

export const AllVariants: Story = {
  render: () => (
    <View style={{ gap: 12, padding: 16 }}>
      <Button variant="default">Default</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="destructive">Destructive</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="ghost">Ghost</Button>
    </View>
  ),
};

export const AllSizes: Story = {
  render: () => (
    <View style={{ gap: 12, padding: 16 }}>
      <Button size="sm">Small</Button>
      <Button size="md">Medium</Button>
      <Button size="lg">Large</Button>
    </View>
  ),
};

export const Disabled: Story = {
  args: {
    children: 'Disabled Button',
    disabled: true,
  },
};
```

### Storybook as Documentation

The best design system documentation is the living code, not a static document:

```typescript
// components/Card/Card.stories.tsx
const meta: Meta<typeof Card> = {
  title: 'Components/Card',
  component: Card,
  parameters: {
    docs: {
      description: {
        component: `
Cards are used to group related content.

## Usage Guidelines
- Use elevated cards for content that should stand out
- Use the pressable variant for interactive cards
- Card.Header is optional for simple cards

## Accessibility
- Interactive cards should have an accessible label
- Use Card.Title for semantic heading structure
        `,
      },
    },
  },
};
```

---

## 6. CVA: CLASS VARIANCE AUTHORITY

We've already seen CVA in the shadcn/ui and NativeWind examples, but it deserves its own section because variant management is a core design system concern.

### The Problem CVA Solves

Without CVA, variant logic becomes a mess of conditionals:

```typescript
// WITHOUT CVA: conditional class hell
function Button({ variant, size, disabled }: ButtonProps) {
  let classes = 'base-button';

  if (variant === 'primary') classes += ' bg-blue-600 text-white';
  else if (variant === 'secondary') classes += ' bg-gray-100 text-gray-900';
  else if (variant === 'destructive') classes += ' bg-red-600 text-white';

  if (size === 'sm') classes += ' h-9 px-3 text-sm';
  else if (size === 'md') classes += ' h-11 px-4 text-base';
  else if (size === 'lg') classes += ' h-13 px-6 text-lg';

  if (disabled) classes += ' opacity-50 pointer-events-none';

  return <Pressable className={classes}>...</Pressable>;
}
```

### CVA: Declarative Variant Management

```typescript
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  // Base classes (always applied)
  'inline-flex items-center justify-center rounded-lg font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-blue-600 text-white hover:bg-blue-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        destructive: 'bg-red-600 text-white hover:bg-red-700',
        ghost: 'text-gray-600 hover:bg-gray-100',
      },
      size: {
        sm: 'h-9 px-3 text-sm',
        md: 'h-11 px-4 text-base',
        lg: 'h-13 px-6 text-lg',
      },
      fullWidth: {
        true: 'w-full',
      },
    },

    // Compound variants: when specific variant combinations need special styling
    compoundVariants: [
      {
        variant: 'primary',
        size: 'lg',
        className: 'text-lg font-bold', // Large primary buttons are bolder
      },
      {
        variant: 'ghost',
        size: 'sm',
        className: 'px-2', // Small ghost buttons need less padding
      },
    ],

    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// TypeScript types are inferred automatically
type ButtonVariants = VariantProps<typeof buttonVariants>;
// { variant?: 'primary' | 'secondary' | 'destructive' | 'ghost'; size?: 'sm' | 'md' | 'lg'; fullWidth?: boolean }
```

### CVA with React Native StyleSheet (No Tailwind)

CVA works with any styling approach, not just Tailwind classes:

```typescript
// lib/style-variants.ts
import { StyleSheet, type ViewStyle, type TextStyle } from 'react-native';

// A CVA-like pattern for React Native StyleSheet
type VariantConfig<V extends Record<string, Record<string, ViewStyle | TextStyle>>> = {
  base: ViewStyle | TextStyle;
  variants: V;
  defaultVariants?: { [K in keyof V]?: keyof V[K] };
};

function createVariants<V extends Record<string, Record<string, ViewStyle | TextStyle>>>(
  config: VariantConfig<V>
) {
  return (selectedVariants: Partial<{ [K in keyof V]: keyof V[K] }> = {}) => {
    const styles: (ViewStyle | TextStyle)[] = [config.base];

    for (const [variantKey, variantValue] of Object.entries(selectedVariants)) {
      const variant = config.variants[variantKey];
      if (variant && variantValue) {
        const style = variant[variantValue as string];
        if (style) styles.push(style);
      }
    }

    // Apply defaults for unspecified variants
    if (config.defaultVariants) {
      for (const [key, value] of Object.entries(config.defaultVariants)) {
        if (!(key in selectedVariants) && value) {
          const variant = config.variants[key];
          if (variant) {
            const style = variant[value as string];
            if (style) styles.push(style);
          }
        }
      }
    }

    return StyleSheet.flatten(styles);
  };
}

// Usage
const getButtonStyles = createVariants({
  base: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    borderRadius: 12,
  },
  variants: {
    variant: {
      primary: { backgroundColor: '#3B82F6' },
      secondary: { backgroundColor: '#F3F4F6' },
      destructive: { backgroundColor: '#DC2626' },
    },
    size: {
      sm: { height: 36, paddingHorizontal: 12 },
      md: { height: 44, paddingHorizontal: 16 },
      lg: { height: 52, paddingHorizontal: 24 },
    },
  },
  defaultVariants: {
    variant: 'primary',
    size: 'md',
  },
});

// In component
const style = getButtonStyles({ variant: 'secondary', size: 'lg' });
```

---

## 7. THE COMPOUND COMPONENT PATTERN

Compound components are groups of components that work together to form a complete UI element. Think of `<select>` and `<option>` in HTML — they're meaningless alone but powerful together.

### Why Compound Components

```typescript
// WITHOUT compound components: prop explosion
<Card
  title="Product Name"
  subtitle="Category"
  description="A great product..."
  image="https://..."
  footer={<Button>Buy Now</Button>}
  headerRight={<Badge>New</Badge>}
  onPress={() => {}}
  bordered
  elevated
/>
// 10+ props, inflexible layout, hard to customize

// WITH compound components: composable and flexible
<Card elevated onPress={() => {}}>
  <Card.Header>
    <Card.Title>Product Name</Card.Title>
    <Badge>New</Badge>
  </Card.Header>
  <Card.Image source={{ uri: 'https://...' }} />
  <Card.Content>
    <Card.Description>A great product...</Card.Description>
  </Card.Content>
  <Card.Footer>
    <Button>Buy Now</Button>
  </Card.Footer>
</Card>
// Flexible, composable, easy to understand
```

### Implementing Compound Components

```typescript
// components/Accordion/Accordion.tsx
import { createContext, useContext, useState, useCallback } from 'react';
import { View, Text, Pressable } from 'react-native';
import Animated, {
  useAnimatedStyle,
  withTiming,
  useSharedValue,
} from 'react-native-reanimated';

// Context for sharing state between compound components
interface AccordionContextType {
  expandedItems: Set<string>;
  toggleItem: (id: string) => void;
  multiple: boolean;
}

const AccordionContext = createContext<AccordionContextType | null>(null);

function useAccordion() {
  const context = useContext(AccordionContext);
  if (!context) {
    throw new Error('Accordion components must be used within <Accordion>');
  }
  return context;
}

// Root component
interface AccordionProps {
  children: React.ReactNode;
  multiple?: boolean;
  defaultExpanded?: string[];
}

function AccordionRoot({ children, multiple = false, defaultExpanded = [] }: AccordionProps) {
  const [expandedItems, setExpandedItems] = useState<Set<string>>(
    new Set(defaultExpanded)
  );

  const toggleItem = useCallback((id: string) => {
    setExpandedItems(prev => {
      const next = new Set(prev);
      if (next.has(id)) {
        next.delete(id);
      } else {
        if (!multiple) next.clear();
        next.add(id);
      }
      return next;
    });
  }, [multiple]);

  return (
    <AccordionContext.Provider value={{ expandedItems, toggleItem, multiple }}>
      <View style={{ gap: 4 }}>
        {children}
      </View>
    </AccordionContext.Provider>
  );
}

// Item component
interface AccordionItemContextType {
  id: string;
  isExpanded: boolean;
}

const AccordionItemContext = createContext<AccordionItemContextType | null>(null);

function useAccordionItem() {
  const context = useContext(AccordionItemContext);
  if (!context) {
    throw new Error('AccordionItem components must be used within <Accordion.Item>');
  }
  return context;
}

function AccordionItem({ id, children }: { id: string; children: React.ReactNode }) {
  const { expandedItems } = useAccordion();
  const isExpanded = expandedItems.has(id);

  return (
    <AccordionItemContext.Provider value={{ id, isExpanded }}>
      <View style={{ borderWidth: 1, borderColor: '#E5E7EB', borderRadius: 8 }}>
        {children}
      </View>
    </AccordionItemContext.Provider>
  );
}

// Trigger component
function AccordionTrigger({ children }: { children: React.ReactNode }) {
  const { toggleItem } = useAccordion();
  const { id, isExpanded } = useAccordionItem();

  return (
    <Pressable
      onPress={() => toggleItem(id)}
      style={{ padding: 16, flexDirection: 'row', justifyContent: 'space-between' }}
      accessibilityRole="button"
      accessibilityState={{ expanded: isExpanded }}
    >
      {children}
      <ChevronIcon rotated={isExpanded} />
    </Pressable>
  );
}

// Content component (animated)
function AccordionContent({ children }: { children: React.ReactNode }) {
  const { isExpanded } = useAccordionItem();
  const height = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    height: withTiming(isExpanded ? undefined : 0, { duration: 200 }),
    opacity: withTiming(isExpanded ? 1 : 0, { duration: 200 }),
    overflow: 'hidden',
  }));

  return (
    <Animated.View style={animatedStyle}>
      <View style={{ padding: 16, paddingTop: 0 }}>
        {children}
      </View>
    </Animated.View>
  );
}

// Export as compound component
export const Accordion = Object.assign(AccordionRoot, {
  Item: AccordionItem,
  Trigger: AccordionTrigger,
  Content: AccordionContent,
});

// Usage
function FAQ() {
  return (
    <Accordion multiple defaultExpanded={['faq-1']}>
      <Accordion.Item id="faq-1">
        <Accordion.Trigger>
          <Text>What is your return policy?</Text>
        </Accordion.Trigger>
        <Accordion.Content>
          <Text>You can return any item within 30 days...</Text>
        </Accordion.Content>
      </Accordion.Item>

      <Accordion.Item id="faq-2">
        <Accordion.Trigger>
          <Text>How long does shipping take?</Text>
        </Accordion.Trigger>
        <Accordion.Content>
          <Text>Standard shipping takes 5-7 business days...</Text>
        </Accordion.Content>
      </Accordion.Item>
    </Accordion>
  );
}
```

---

## 8. ACCESSIBILITY (A11Y) AS A FIRST-CLASS CONCERN

Accessibility is not a feature. It's a requirement. And building it in from the start is 10x easier than retrofitting it later.

### React Native Accessibility Essentials

```typescript
// THE BIG THREE: role, label, state

// 1. accessibilityRole — what is this element?
<Pressable accessibilityRole="button">
<Pressable accessibilityRole="link">
<View accessibilityRole="header">
<Switch accessibilityRole="switch">
<TextInput accessibilityRole="search">

// 2. accessibilityLabel — what does it do? (screen reader reads this)
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Add to cart"
  // NOT: accessibilityLabel="Button" (that's useless)
  // NOT: accessibilityLabel="Cart icon" (describe the action, not the visual)
>
  <CartIcon />
</Pressable>

// 3. accessibilityState — what's its current state?
<Pressable
  accessibilityRole="checkbox"
  accessibilityLabel="Accept terms"
  accessibilityState={{ checked: isChecked }}
>
  <Checkbox checked={isChecked} />
</Pressable>
```

### Building Accessible Components into Your Design System

```typescript
// components/ui/Switch.tsx
import { Switch as RNSwitch, type SwitchProps as RNSwitchProps, View, Text } from 'react-native';

interface SwitchProps extends Omit<RNSwitchProps, 'accessibilityRole'> {
  label: string;          // Required! Every switch needs a label
  description?: string;
}

export function Switch({ label, description, value, onValueChange, ...props }: SwitchProps) {
  return (
    <View
      style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}
      accessible={true}
      accessibilityRole="switch"
      accessibilityLabel={label}
      accessibilityState={{ checked: value }}
      accessibilityHint={description}
    >
      <View style={{ flex: 1 }}>
        <Text style={{ fontSize: 16, fontWeight: '500' }}>{label}</Text>
        {description && (
          <Text style={{ fontSize: 14, color: '#6B7280', marginTop: 2 }}>
            {description}
          </Text>
        )}
      </View>
      <RNSwitch
        value={value}
        onValueChange={onValueChange}
        // Remove accessibility from the native switch since the parent handles it
        accessible={false}
        {...props}
      />
    </View>
  );
}

// Usage — accessibility is baked in, developers can't forget it
<Switch
  label="Push Notifications"
  description="Receive alerts for new messages"
  value={pushEnabled}
  onValueChange={setPushEnabled}
/>
```

### Accessibility Checklist for Component Libraries

```
┌─────────────────────────────────────────────────────────────┐
│  A11Y COMPONENT CHECKLIST                                    │
│                                                              │
│  INTERACTIVE ELEMENTS:                                       │
│  ☐ Has accessibilityRole (button, link, checkbox, etc.)     │
│  ☐ Has accessibilityLabel (meaningful description)           │
│  ☐ Has accessibilityHint for non-obvious actions             │
│  ☐ Has accessibilityState (selected, checked, expanded)      │
│  ☐ Minimum touch target 44x44pt (iOS) / 48x48dp (Android)  │
│  ☐ Visual focus indicator for keyboard navigation            │
│                                                              │
│  TEXT CONTENT:                                                │
│  ☐ Supports dynamic type / font scaling                      │
│  ☐ Minimum contrast ratio 4.5:1 (AA) or 7:1 (AAA)          │
│  ☐ Does not convey meaning through color alone               │
│                                                              │
│  IMAGES:                                                     │
│  ☐ Decorative images: accessible={false}                     │
│  ☐ Informative images: accessibilityLabel describes content  │
│                                                              │
│  STRUCTURE:                                                   │
│  ☐ Logical reading order (accessibilityOrder if needed)      │
│  ☐ Group related elements (accessible={true} on container)   │
│  ☐ Live regions for dynamic content (accessibilityLiveRegion)│
│                                                              │
│  MOTION:                                                      │
│  ☐ Respects "Reduce Motion" setting                          │
│  ☐ No auto-playing animations that can't be paused           │
└─────────────────────────────────────────────────────────────┘
```

### Respecting Reduce Motion

```typescript
import { useReducedMotion } from 'react-native-reanimated';
// Or: import { AccessibilityInfo } from 'react-native';

function AnimatedCard({ children }: { children: React.ReactNode }) {
  const reduceMotion = useReducedMotion();

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      {
        scale: reduceMotion
          ? 1 // No animation
          : withSpring(pressed ? 0.98 : 1),
      },
    ],
  }));

  return <Animated.View style={animatedStyle}>{children}</Animated.View>;
}
```

### Testing Accessibility

```bash
# iOS: Accessibility Inspector
# Xcode → Open Developer Tool → Accessibility Inspector
# Point at elements to see their a11y properties

# Android: TalkBack
# Settings → Accessibility → TalkBack → On
# Navigate with swipe gestures, listen to announcements

# Automated testing
npm install -D jest-native @testing-library/react-native
```

```typescript
// __tests__/Button.test.tsx
import { render, screen } from '@testing-library/react-native';
import { Button } from '../components/ui/Button';

test('button has correct accessibility properties', () => {
  render(<Button onPress={() => {}}>Add to Cart</Button>);

  const button = screen.getByRole('button', { name: 'Add to Cart' });
  expect(button).toBeTruthy();
});

test('disabled button has correct accessibility state', () => {
  render(<Button disabled onPress={() => {}}>Submit</Button>);

  const button = screen.getByRole('button', { name: 'Submit' });
  expect(button).toBeDisabled();
});
```

---

## 9. BUILDING A COMPONENT LIBRARY IN A MONOREPO

If you're working in a monorepo (and Chapter 15 argues you should be), your component library should be a shared package consumed by all your apps.

### Monorepo Structure

```
packages/
  ui/                          ← Your component library
    src/
      components/
        Button/
          Button.tsx
          Button.stories.tsx
          Button.test.tsx
          index.ts
        Card/
          Card.tsx
          Card.stories.tsx
          Card.test.tsx
          index.ts
        Input/
        ...
      tokens/
        colors.ts
        spacing.ts
        typography.ts
        semantic.ts
        index.ts
      providers/
        ThemeProvider.tsx
      utils/
        cn.ts
      index.ts                 ← Public API
    package.json
    tsconfig.json
    
apps/
  mobile/                      ← React Native app
    package.json               ← depends on @repo/ui
  web/                         ← Next.js app
    package.json               ← depends on @repo/ui
```

### Package Configuration

```json
// packages/ui/package.json
{
  "name": "@repo/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./tokens": "./src/tokens/index.ts",
    "./providers": "./src/providers/ThemeProvider.tsx"
  },
  "peerDependencies": {
    "react": ">=18",
    "react-native": ">=0.73"
  },
  "devDependencies": {
    "@repo/typescript-config": "workspace:*",
    "@storybook/react-native": "^7.0.0",
    "@testing-library/react-native": "^12.0.0"
  }
}
```

### The Public API

Be deliberate about what you export. Every export is a contract:

```typescript
// packages/ui/src/index.ts

// Components
export { Button, type ButtonProps } from './components/Button';
export { Card, type CardProps } from './components/Card';
export { Input, type InputProps } from './components/Input';
export { Switch, type SwitchProps } from './components/Switch';
export { Text, type TextProps } from './components/Text';
export { Badge, type BadgeProps } from './components/Badge';
export { Avatar, type AvatarProps } from './components/Avatar';

// Compound components
export { Accordion } from './components/Accordion';
export { Dialog } from './components/Dialog';
export { Tabs } from './components/Tabs';

// Providers
export { ThemeProvider, useTheme } from './providers/ThemeProvider';

// Tokens (for advanced use cases)
export { lightTheme, darkTheme, type Theme } from './tokens/semantic';

// Utilities
export { cn } from './utils/cn';
```

### Consuming in Apps

```typescript
// apps/mobile/src/screens/ProductScreen.tsx
import {
  Button,
  Card,
  Text,
  Badge,
  useTheme,
} from '@repo/ui';

function ProductScreen({ product }: { product: Product }) {
  const theme = useTheme();

  return (
    <Card elevated>
      <Card.Header>
        <Card.Title>{product.name}</Card.Title>
        <Badge variant={product.inStock ? 'success' : 'error'}>
          {product.inStock ? 'In Stock' : 'Out of Stock'}
        </Badge>
      </Card.Header>
      <Card.Content>
        <Text variant="body">{product.description}</Text>
        <Text variant="price">${product.price}</Text>
      </Card.Content>
      <Button
        variant="primary"
        size="lg"
        fullWidth
        disabled={!product.inStock}
        onPress={() => addToCart(product)}
      >
        Add to Cart
      </Button>
    </Card>
  );
}
```

### Platform-Specific Components

Some components need different implementations for web and native:

```typescript
// packages/ui/src/components/DatePicker/index.ts
export { DatePicker, type DatePickerProps } from './DatePicker';

// packages/ui/src/components/DatePicker/DatePicker.tsx
import { Platform } from 'react-native';

export function DatePicker(props: DatePickerProps) {
  if (Platform.OS === 'web') {
    return <WebDatePicker {...props} />;
  }
  return <NativeDatePicker {...props} />;
}

// Or use file extensions for Metro resolution:
// DatePicker.web.tsx   → Web implementation
// DatePicker.native.tsx → iOS/Android implementation
// DatePicker.tsx       → Shared types and re-export
```

---

## 10. THE DESIGN SYSTEM DECISION FRAMEWORK

Choosing the right tools depends on your team, your platform targets, and your scale.

```
┌─────────────────────────────────────────────────────────────┐
│  DECISION 1: What platforms do you target?                   │
│                                                              │
│  Web only → shadcn/ui + Tailwind CSS + CVA                  │
│  Native only → NativeWind or Tamagui + CVA                  │
│  Both → Tamagui (shared components) or                      │
│         shared tokens + platform-specific components         │
│                                                              │
│  DECISION 2: What does your team know?                       │
│                                                              │
│  Tailwind experts → NativeWind (mobile) + Tailwind (web)    │
│  CSS-in-JS team → Tamagui                                   │
│  New team → Start with NativeWind (lowest learning curve)   │
│                                                              │
│  DECISION 3: How many consumers?                             │
│                                                              │
│  1 app → Components in the app, not a separate package      │
│  2-3 apps → Shared package in monorepo                      │
│  5+ apps/teams → Full design system with Storybook, docs,   │
│                  versioning, migration guides                │
│                                                              │
│  DECISION 4: How mature is your design?                      │
│                                                              │
│  Still iterating → Don't over-abstract, use shadcn/ui-style │
│                    copy-paste components                      │
│  Established → Build a proper component library with         │
│                 tokens, variants, and documentation           │
│  Mature → Invest in Storybook, visual regression testing,   │
│           automated accessibility audits                     │
└─────────────────────────────────────────────────────────────┘
```

---

## CHAPTER SUMMARY

A design system is not a luxury for large teams — it's a foundation that pays dividends from day one if you approach it pragmatically:

- **Design tokens** are your foundation: define colors, spacing, typography, and shadows as semantic values that map to global primitives
- **shadcn/ui** for web gives you beautiful, accessible components that you own and customize
- **Tamagui** compiles universal components to optimal output on both web and native
- **NativeWind** brings the Tailwind mental model to React Native for teams that already think in utility classes
- **CVA** manages component variants declaratively, keeping your variant logic clean and type-safe
- **Compound components** provide composable APIs that avoid prop explosion
- **Accessibility** must be baked into every component, not bolted on later
- **Monorepo component libraries** let multiple apps share the same design system while keeping implementation flexible

The most important principle: **start simple and grow.** A well-organized set of design tokens and a handful of well-built components beats a comprehensive but unmaintained design system every time.

---

*Next: [Chapter 17: Testing Strategies]*
*Previous: [Chapter 15: Monorepo Architecture]*