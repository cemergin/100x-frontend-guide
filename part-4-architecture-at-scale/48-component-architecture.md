<!--
  CHAPTER: 48
  TITLE: Component Library Architecture — Building UI at Scale
  PART: IV — Architecture at Scale
  PREREQS: Chapters 17, 16
  KEY_TOPICS: atomic design, compound components, headless UI, Radix primitives, composition patterns, slot pattern, render props, polymorphic components, forwardRef, generic components, controlled vs uncontrolled, component API design, Storybook driven development, component documentation, versioning, breaking changes
  DIFFICULTY: Advanced
  UPDATED: 2026-04-07
-->

# Chapter 48: Component Library Architecture — Building UI at Scale

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 17, 16 | Difficulty: Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- Component API design is architecture: fewer props, sensible defaults, and composition over configuration produce libraries that teams actually adopt instead of work around
- Compound components (Select + Select.Trigger + Select.Content + Select.Item) backed by React Context are the gold standard for complex, multi-part UI -- they give consumers full layout control without prop-drilling
- Headless UI separates behavior (a11y, keyboard nav, state) from rendering; use Radix UI or React Aria as your primitives layer, then style on top -- never reinvent the a11y wheel
- Polymorphic components (`as` prop), generic components (`<List<T>>`), and the slot pattern give library consumers escape hatches without forking; design for the 90% case but never trap the 10%
- Storybook-driven development (write the story first, then implement) produces better APIs, catches design issues early, and generates living documentation that stays in sync with code

</details>

Chapter 16 showed you design tokens, tool choices, and the foundations of a design system. This chapter is about something different: the **engineering** of the component library itself. The architecture patterns, TypeScript gymnastics, composition strategies, and development workflows that separate a component library people love from one people route around.

I've built component libraries used by three developers and component libraries used by three hundred. The failure mode is always the same: the library works fine for the first twenty components, then falls apart as edge cases pile up. A button that needs to be a link. A dropdown that needs to be controlled externally. A list that needs to render different item types. A form field that needs custom validation UI.

The difference between a library that survives these pressures and one that doesn't is not cleverness. It's the patterns you commit to on day one. This chapter is about those patterns.

### In This Chapter
- Component API Design Principles
- Atomic Design in Practice
- Compound Components with React Context
- The Headless UI Pattern
- Polymorphic Components (the `as` prop)
- The Slot Pattern
- Controlled vs Uncontrolled Components
- Generic Components with TypeScript
- forwardRef and Imperative Handles
- Storybook-Driven Development
- Component Documentation
- Versioning and Breaking Changes
- The Complete Component: Building a Production-Grade Select

### Related Chapters
- [Ch 16: Design Systems & Component Libraries] -- tokens, tool choices, and the design system foundation
- [Ch 17: Testing Strategies] -- testing your component library
- [Ch 15: Monorepo Architecture] -- where your component library lives in the repo
- [Ch 19: Frontend Architecture Patterns] -- how component libraries fit into the larger architecture

---

## 1. COMPONENT API DESIGN

The most important code in a component library is not the implementation -- it's the API. The props interface is what hundreds of developers interact with every day. A well-designed API disappears; developers use it without thinking. A poorly designed API generates Slack questions, wrapper components, and eventually a competing library maintained by the team that got frustrated enough to fork yours.

### The Pit of Success

The concept comes from Rico Mariani at Microsoft: **make the right thing easy and the wrong thing hard.** Applied to component libraries:

```typescript
// BAD: The wrong thing is easy
// Developers WILL pass hex colors because the prop accepts string
interface BadButtonProps {
  color: string;           // What colors are valid? Who knows!
  size: string;            // "small"? "sm"? "S"? pixels?
  onClick: () => void;
  disabled: boolean;       // Must be explicitly set to false
  loading: boolean;        // Must be explicitly set to false
  type: string;            // "submit"? "button"? "reset"?
  icon: any;               // What goes here?
  iconPosition: string;    // "left"? "right"? "start"? "end"?
}

// GOOD: The right thing is easy
// TypeScript guides developers to the correct values
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';  // Defaults to 'primary'
  size?: 'sm' | 'md' | 'lg';                                // Defaults to 'md'
  onPress?: () => void;
  disabled?: boolean;                                         // Defaults to false
  loading?: boolean;                                          // Defaults to false
  icon?: React.ReactElement;
  iconPosition?: 'start' | 'end';                            // Defaults to 'start'
  children: React.ReactNode;
}
```

### The Five Rules of Component API Design

**Rule 1: Fewer props = less to learn.**

Every prop you add is a decision the consumer must make. The ideal component has 2-5 props for basic usage. Advanced props exist but aren't needed for the common case.

```typescript
// Count the decisions a developer must make:

// Version A: 11 decisions
<Button
  variant="primary"
  size="md"
  disabled={false}
  loading={false}
  fullWidth={false}
  icon={undefined}
  iconPosition="start"
  type="button"
  testID="submit-btn"
  accessibilityLabel="Submit form"
  onPress={handleSubmit}
>
  Submit
</Button>

// Version B: 2 decisions (the rest are sensible defaults)
<Button onPress={handleSubmit}>Submit</Button>

// Version A and Version B render the exact same output.
// Version B is the better API.
```

**Rule 2: Sensible defaults eliminate boilerplate.**

Every optional prop needs a default that covers the 80% case:

```typescript
// tokens/defaults.ts
const BUTTON_DEFAULTS = {
  variant: 'primary',
  size: 'md',
  iconPosition: 'start',
  type: 'button',
  disabled: false,
  loading: false,
  fullWidth: false,
} as const;

function Button({
  variant = 'primary',
  size = 'md',
  iconPosition = 'start',
  disabled = false,
  loading = false,
  children,
  onPress,
  ...rest
}: ButtonProps) {
  // Implementation
}
```

**Rule 3: Composition over configuration.**

When a component starts accumulating boolean flags, it's a sign you need composition instead:

```typescript
// BAD: Configuration explosion
// Every new feature adds a prop. 2^n combinations to test.
<Card
  hasHeader={true}
  headerTitle="Settings"
  headerSubtitle="Manage your account"
  headerAction={<IconButton icon="edit" />}
  hasDivider={true}
  hasFooter={true}
  footerAlignment="right"
  footerActions={[<Button>Save</Button>, <Button>Cancel</Button>]}
>
  <CardContent />
</Card>

// GOOD: Composition
// Each piece is independent. Consumers assemble what they need.
<Card>
  <Card.Header>
    <Card.Title>Settings</Card.Title>
    <Card.Subtitle>Manage your account</Card.Subtitle>
    <Card.Action><IconButton icon="edit" /></Card.Action>
  </Card.Header>
  <Card.Divider />
  <Card.Body>
    <CardContent />
  </Card.Body>
  <Card.Footer align="right">
    <Button variant="ghost">Cancel</Button>
    <Button>Save</Button>
  </Card.Footer>
</Card>
```

The second version has more JSX, but it's *self-documenting*. You can read the structure. You can rearrange parts. You can omit parts. You never end up with `hasHeader={true}` but `headerTitle={undefined}` -- a state that shouldn't exist but the types allow.

**Rule 4: Props should not fight each other.**

If two props can create an impossible state, your API is wrong:

```typescript
// BAD: Contradictory states are possible
interface BadProps {
  loading: boolean;
  disabled: boolean;
  error: boolean;
  success: boolean;
  // What happens when loading AND error are both true?
  // What about loading AND success AND disabled?
}

// GOOD: Use a discriminated union
interface GoodProps {
  status?: 'idle' | 'loading' | 'error' | 'success';
  disabled?: boolean;
  // Only one status at a time. Impossible states are impossible.
}
```

**Rule 5: Extend native elements, don't replace them.**

Your `<Button>` should accept everything a native `<Pressable>` accepts (for React Native) or everything a `<button>` accepts (for web). Don't make consumers choose between your API and the platform's:

```typescript
import { Pressable, PressableProps } from 'react-native';

// Extend the native props, don't replace them
interface ButtonProps extends Omit<PressableProps, 'style'> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  icon?: React.ReactElement;
  iconPosition?: 'start' | 'end';
  children: React.ReactNode;
}

function Button({ variant = 'primary', size = 'md', children, ...pressableProps }: ButtonProps) {
  return (
    <Pressable
      {...pressableProps}
      style={[styles.base, styles[variant], styles[size]]}
    >
      {/* ... */}
    </Pressable>
  );
}

// Now consumers can pass ANY Pressable prop:
<Button
  variant="primary"
  onPress={handlePress}
  onLongPress={handleLongPress}       // Works! It's a Pressable prop
  accessibilityRole="button"           // Works! It's a Pressable prop
  hitSlop={10}                         // Works! It's a Pressable prop
  testID="submit-button"              // Works! It's a Pressable prop
>
  Submit
</Button>
```

### The Props Checklist

Before shipping any component, run through this:

```
┌──────────────────────────────────────────────────────────────────────┐
│  COMPONENT API REVIEW CHECKLIST                                       │
│                                                                       │
│  □ Can the component be used with just children + one handler?        │
│  □ Does every optional prop have a sensible default?                  │
│  □ Are string props replaced with union types?                        │
│  □ Are boolean flag clusters replaced with discriminated unions?      │
│  □ Does the component extend the relevant native element's props?     │
│  □ Can impossible states be expressed through the props? (They        │
│    shouldn't be)                                                      │
│  □ Is the prop naming consistent with the rest of the library?        │
│    (onPress vs onClick, size vs scale, variant vs appearance)         │
│  □ Would a new developer understand the props without reading docs?   │
│  □ Is there a composition-based alternative to any multi-prop         │
│    configuration pattern?                                             │
│  □ Has the API been tested by writing consumer code FIRST?            │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. ATOMIC DESIGN IN PRACTICE

Brad Frost's Atomic Design gives us a mental model for categorizing components by complexity. It's borrowed from chemistry: atoms combine into molecules, molecules into organisms, organisms into templates, templates into pages. The model is useful but imperfect -- let's see where it helps and where it breaks down.

### The Five Levels

```
┌──────────────────────────────────────────────────────────────────────┐
│  ATOMIC DESIGN HIERARCHY                                              │
│                                                                       │
│  ATOMS          Smallest, indivisible UI units                        │
│  │              Button, Text, Icon, Input, Badge, Checkbox, Avatar    │
│  │                                                                    │
│  MOLECULES      Small groups of atoms working together                │
│  │              SearchBar (Input + Icon + Button)                     │
│  │              FormField (Label + Input + ErrorText)                 │
│  │              UserBadge (Avatar + Name + StatusDot)                 │
│  │                                                                    │
│  ORGANISMS      Complex, distinct UI sections                         │
│  │              Header (Logo + Nav + SearchBar + UserMenu)            │
│  │              ProductCard (Image + Title + Price + AddToCart)       │
│  │              CommentThread (Comment + ReplyList + ReplyInput)      │
│  │                                                                    │
│  TEMPLATES      Page-level layouts without real data                  │
│  │              DashboardLayout (Sidebar + Header + ContentArea)      │
│  │              AuthLayout (CenteredCard + Background)                │
│  │                                                                    │
│  PAGES          Templates connected to real data and state            │
│                 DashboardPage (DashboardLayout + fetched data)        │
│                 LoginPage (AuthLayout + auth state + form logic)      │
└──────────────────────────────────────────────────────────────────────┘
```

### Implementation: Atoms

Atoms are the foundation. They have no business logic, no data dependencies, and no awareness of where they're used. They accept design tokens and render styled platform primitives.

```typescript
// atoms/Text.tsx
import { Text as RNText, TextProps as RNTextProps, TextStyle } from 'react-native';
import { useTheme } from '../theme';

type TextVariant = 'body' | 'caption' | 'heading1' | 'heading2' | 'heading3' | 'label' | 'code';
type TextColor = 'primary' | 'secondary' | 'tertiary' | 'error' | 'success' | 'inherit';

interface TextProps extends RNTextProps {
  variant?: TextVariant;
  color?: TextColor;
  align?: TextStyle['textAlign'];
  weight?: 'regular' | 'medium' | 'semibold' | 'bold';
}

const variantStyles: Record<TextVariant, TextStyle> = {
  heading1: { fontSize: 32, lineHeight: 40, fontWeight: '700' },
  heading2: { fontSize: 24, lineHeight: 32, fontWeight: '600' },
  heading3: { fontSize: 20, lineHeight: 28, fontWeight: '600' },
  body:     { fontSize: 16, lineHeight: 24, fontWeight: '400' },
  caption:  { fontSize: 12, lineHeight: 16, fontWeight: '400' },
  label:    { fontSize: 14, lineHeight: 20, fontWeight: '500' },
  code:     { fontSize: 14, lineHeight: 20, fontFamily: 'monospace' },
};

export function Text({
  variant = 'body',
  color = 'primary',
  align,
  weight,
  style,
  ...rest
}: TextProps) {
  const theme = useTheme();

  const colorMap: Record<TextColor, string> = {
    primary: theme.color.text,
    secondary: theme.color.textSecondary,
    tertiary: theme.color.textTertiary,
    error: theme.color.error,
    success: theme.color.success,
    inherit: 'inherit',
  };

  return (
    <RNText
      style={[
        variantStyles[variant],
        { color: colorMap[color] },
        align && { textAlign: align },
        weight && { fontWeight: weightMap[weight] },
        style,
      ]}
      {...rest}
    />
  );
}

const weightMap = {
  regular: '400' as const,
  medium: '500' as const,
  semibold: '600' as const,
  bold: '700' as const,
};
```

```typescript
// atoms/Button.tsx
import { forwardRef } from 'react';
import {
  Pressable,
  PressableProps,
  ActivityIndicator,
  View,
  ViewStyle,
  TextStyle,
} from 'react-native';
import { Text } from './Text';
import { useTheme } from '../theme';

type ButtonVariant = 'primary' | 'secondary' | 'ghost' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

interface ButtonProps extends Omit<PressableProps, 'children' | 'style'> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  loading?: boolean;
  icon?: React.ReactElement;
  iconPosition?: 'start' | 'end';
  fullWidth?: boolean;
  children: string; // Enforce string children for consistent typography
}

export const Button = forwardRef<View, ButtonProps>(function Button(
  {
    variant = 'primary',
    size = 'md',
    loading = false,
    icon,
    iconPosition = 'start',
    fullWidth = false,
    disabled = false,
    children,
    ...pressableProps
  },
  ref,
) {
  const theme = useTheme();
  const isDisabled = disabled || loading;

  const containerStyle: ViewStyle = {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
    gap: theme.spacing.sm,
    borderRadius: theme.borderRadius.md,
    opacity: isDisabled ? 0.5 : 1,
    ...(fullWidth && { width: '100%' }),
    ...sizeStyles[size],
    ...variantContainerStyles(theme)[variant],
  };

  const textColor = variantTextColor(theme)[variant];

  return (
    <Pressable
      ref={ref}
      disabled={isDisabled}
      style={({ pressed }) => [
        containerStyle,
        pressed && { opacity: 0.8 },
      ]}
      accessibilityRole="button"
      accessibilityState={{ disabled: isDisabled, busy: loading }}
      {...pressableProps}
    >
      {loading ? (
        <ActivityIndicator color={textColor} size="small" />
      ) : (
        <>
          {icon && iconPosition === 'start' && icon}
          <Text variant="label" style={{ color: textColor }}>
            {children}
          </Text>
          {icon && iconPosition === 'end' && icon}
        </>
      )}
    </Pressable>
  );
});

const sizeStyles: Record<ButtonSize, ViewStyle> = {
  sm: { paddingHorizontal: 12, paddingVertical: 6, minHeight: 32 },
  md: { paddingHorizontal: 16, paddingVertical: 10, minHeight: 40 },
  lg: { paddingHorizontal: 24, paddingVertical: 14, minHeight: 48 },
};

const variantContainerStyles = (theme: ReturnType<typeof useTheme>) => ({
  primary:   { backgroundColor: theme.color.primary } as ViewStyle,
  secondary: { backgroundColor: 'transparent', borderWidth: 1, borderColor: theme.color.border } as ViewStyle,
  ghost:     { backgroundColor: 'transparent' } as ViewStyle,
  danger:    { backgroundColor: theme.color.error } as ViewStyle,
});

const variantTextColor = (theme: ReturnType<typeof useTheme>) => ({
  primary:   theme.color.onPrimary,
  secondary: theme.color.text,
  ghost:     theme.color.primary,
  danger:    theme.color.onPrimary,
});
```

### Implementation: Molecules

Molecules combine atoms into functional units. They have a single responsibility but are composed of multiple primitives:

```typescript
// molecules/FormField.tsx
import { View, ViewStyle } from 'react-native';
import { Text } from '../atoms/Text';
import { useTheme } from '../theme';

interface FormFieldProps {
  label: string;
  error?: string;
  hint?: string;
  required?: boolean;
  children: React.ReactNode; // The actual input atom
}

export function FormField({ label, error, hint, required, children }: FormFieldProps) {
  const theme = useTheme();

  return (
    <View style={{ gap: theme.spacing.xs }}>
      <Text variant="label" color="secondary">
        {label}
        {required && <Text color="error"> *</Text>}
      </Text>

      {children}

      {error ? (
        <Text variant="caption" color="error">{error}</Text>
      ) : hint ? (
        <Text variant="caption" color="tertiary">{hint}</Text>
      ) : null}
    </View>
  );
}

// Usage:
// <FormField label="Email" error={errors.email} required>
//   <TextInput value={email} onChangeText={setEmail} />
// </FormField>
```

```typescript
// molecules/SearchBar.tsx
import { View } from 'react-native';
import { TextInput } from '../atoms/TextInput';
import { Icon } from '../atoms/Icon';
import { Button } from '../atoms/Button';
import { useTheme } from '../theme';

interface SearchBarProps {
  value: string;
  onChangeText: (text: string) => void;
  onSubmit?: () => void;
  placeholder?: string;
}

export function SearchBar({
  value,
  onChangeText,
  onSubmit,
  placeholder = 'Search...',
}: SearchBarProps) {
  const theme = useTheme();

  return (
    <View style={{
      flexDirection: 'row',
      alignItems: 'center',
      gap: theme.spacing.sm,
      backgroundColor: theme.color.backgroundSecondary,
      borderRadius: theme.borderRadius.md,
      paddingHorizontal: theme.spacing.md,
    }}>
      <Icon name="search" size={20} color={theme.color.textTertiary} />
      <TextInput
        value={value}
        onChangeText={onChangeText}
        onSubmitEditing={onSubmit}
        placeholder={placeholder}
        style={{ flex: 1 }}
      />
      {value.length > 0 && (
        <Button variant="ghost" size="sm" onPress={() => onChangeText('')}>
          Clear
        </Button>
      )}
    </View>
  );
}
```

### Implementation: Organisms

Organisms are where things get interesting. They're large enough to have their own internal state and complex enough to benefit from compound component patterns (covered in Section 3):

```typescript
// organisms/ProductCard.tsx
import { View, Image, Pressable } from 'react-native';
import { Text } from '../atoms/Text';
import { Badge } from '../atoms/Badge';
import { Button } from '../atoms/Button';
import { useTheme } from '../theme';

interface Product {
  id: string;
  title: string;
  price: number;
  image: string;
  discount?: number;
  inStock: boolean;
}

interface ProductCardProps {
  product: Product;
  onPress?: (product: Product) => void;
  onAddToCart?: (product: Product) => void;
}

export function ProductCard({ product, onPress, onAddToCart }: ProductCardProps) {
  const theme = useTheme();
  const { title, price, image, discount, inStock } = product;

  const discountedPrice = discount ? price * (1 - discount / 100) : price;

  return (
    <Pressable
      onPress={() => onPress?.(product)}
      style={{
        borderRadius: theme.borderRadius.lg,
        backgroundColor: theme.color.surface,
        overflow: 'hidden',
        borderWidth: 1,
        borderColor: theme.color.border,
      }}
    >
      <View style={{ position: 'relative' }}>
        <Image source={{ uri: image }} style={{ width: '100%', aspectRatio: 1 }} />
        {discount && (
          <Badge
            variant="danger"
            style={{ position: 'absolute', top: theme.spacing.sm, right: theme.spacing.sm }}
          >
            {discount}% OFF
          </Badge>
        )}
      </View>

      <View style={{ padding: theme.spacing.md, gap: theme.spacing.sm }}>
        <Text variant="label" numberOfLines={2}>{title}</Text>

        <View style={{ flexDirection: 'row', alignItems: 'center', gap: theme.spacing.sm }}>
          <Text variant="heading3">${discountedPrice.toFixed(2)}</Text>
          {discount && (
            <Text
              variant="caption"
              color="tertiary"
              style={{ textDecorationLine: 'line-through' }}
            >
              ${price.toFixed(2)}
            </Text>
          )}
        </View>

        <Button
          variant={inStock ? 'primary' : 'secondary'}
          size="md"
          disabled={!inStock}
          onPress={() => onAddToCart?.(product)}
          fullWidth
        >
          {inStock ? 'Add to Cart' : 'Out of Stock'}
        </Button>
      </View>
    </Pressable>
  );
}
```

### When Atomic Design Helps (and When It's Overengineering)

```
┌──────────────────────────────────────────────────────────────────────┐
│  ATOMIC DESIGN: WHEN TO USE IT                                        │
│                                                                       │
│  HELPS WHEN:                                                          │
│  ✓ You have a shared component library consumed by multiple teams     │
│  ✓ You need a shared vocabulary between designers and engineers       │
│  ✓ Your component count exceeds ~50 and you need organizational      │
│    structure                                                          │
│  ✓ You're building a general-purpose UI kit                           │
│                                                                       │
│  OVERENGINEERING WHEN:                                                │
│  ✗ Small team, single product -- just use /components with flat      │
│    structure                                                          │
│  ✗ You spend more time debating "is this a molecule or organism?"    │
│    than building features                                             │
│  ✗ Your components are highly domain-specific (not reusable atoms)   │
│  ✗ You force every component into the hierarchy even when it does    │
│    not fit                                                            │
│                                                                       │
│  THE PRAGMATIC MIDDLE GROUND:                                         │
│  Use the mental model without the rigid folder structure:             │
│    /components                                                        │
│      /primitives    (atoms)                                           │
│      /composites    (molecules + organisms)                           │
│      /layouts       (templates)                                       │
│      /screens       (pages -- or keep in /app routes)                │
│                                                                       │
│  Three levels is enough. Don't fight over molecule vs organism.       │
└──────────────────────────────────────────────────────────────────────┘
```

The real value of atomic design is not the taxonomy. It's the thinking: **small, composable pieces that combine into larger, more specific pieces.** You can get that benefit without five folders and a classification debate.

---

## 3. COMPOUND COMPONENTS

Compound components are the single most important pattern in component library architecture. They're what separate a component library that feels like React from one that feels like jQuery wrapped in JSX.

The idea: a set of components that work together implicitly through shared context, but give the consumer full control over structure and layout.

### The Problem Compound Components Solve

Consider a Select dropdown. The naive approach:

```typescript
// BAD: Configuration-based API
<Select
  options={[
    { value: 'apple', label: 'Apple', icon: '🍎', disabled: false },
    { value: 'banana', label: 'Banana', icon: '🍌', disabled: false },
    { value: 'cherry', label: 'Cherry', icon: '🍒', disabled: true },
  ]}
  value={selected}
  onChange={setSelected}
  placeholder="Pick a fruit"
  triggerIcon="chevron-down"
  triggerVariant="outlined"
  optionRenderer={(option) => <CustomOption {...option} />}
  headerRenderer={() => <CategoryHeader />}
  footerRenderer={() => <SelectFooter />}
  emptyRenderer={() => <EmptyState />}
  searchable={true}
  searchPlaceholder="Filter fruits..."
  maxHeight={300}
  position="bottom"
  closeOnSelect={true}
/>
```

This is the configuration trap. Every new requirement adds a prop. Every new prop adds a test case. The combination space is exponential. And the `*Renderer` props are a code smell -- they're the library admitting it can't handle layout.

The compound component approach:

```typescript
// GOOD: Composition-based API
<Select value={selected} onValueChange={setSelected}>
  <Select.Trigger>
    <Select.Value placeholder="Pick a fruit" />
    <Select.Icon />
  </Select.Trigger>

  <Select.Content>
    <Select.Search placeholder="Filter fruits..." />

    <Select.Group>
      <Select.Label>Fruits</Select.Label>
      <Select.Item value="apple">
        <Select.ItemIcon>🍎</Select.ItemIcon>
        <Select.ItemText>Apple</Select.ItemText>
      </Select.Item>
      <Select.Item value="banana">
        <Select.ItemIcon>🍌</Select.ItemIcon>
        <Select.ItemText>Banana</Select.ItemText>
      </Select.Item>
      <Select.Item value="cherry" disabled>
        <Select.ItemIcon>🍒</Select.ItemIcon>
        <Select.ItemText>Cherry</Select.ItemText>
      </Select.Item>
    </Select.Group>

    <Select.Empty>No fruits found</Select.Empty>
  </Select.Content>
</Select>
```

Why this is better:
- **Consumers control layout.** Want the icon on the right instead of the left? Move the JSX. Want a divider between groups? Add a `<Select.Separator />`.
- **No render props.** The structure IS the render.
- **Self-documenting.** You can read the component tree and understand what it does.
- **Type-safe.** Each sub-component has its own props. No `optionRenderer: (option: OptionType) => ReactNode` stringly-typed APIs.
- **Extensible.** Need a new feature? Add a new sub-component. Existing consumers are unaffected.

### Implementation: The Context Pattern

The magic behind compound components is React Context. The parent provides state, the children consume it:

```typescript
// components/Select/SelectContext.tsx
import { createContext, useContext } from 'react';

interface SelectContextValue {
  // State
  value: string | undefined;
  open: boolean;
  highlightedIndex: number;
  searchQuery: string;

  // Actions
  onValueChange: (value: string) => void;
  onOpenChange: (open: boolean) => void;
  onHighlightedIndexChange: (index: number) => void;
  onSearchQueryChange: (query: string) => void;

  // Refs
  triggerRef: React.RefObject<View>;
  contentRef: React.RefObject<View>;

  // Derived
  selectedItem: SelectItemData | undefined;
  filteredItems: SelectItemData[];

  // Registration
  registerItem: (item: SelectItemData) => void;
  unregisterItem: (value: string) => void;
}

interface SelectItemData {
  value: string;
  label: string;
  disabled: boolean;
  index: number;
}

const SelectContext = createContext<SelectContextValue | null>(null);

export function useSelectContext() {
  const context = useContext(SelectContext);
  if (!context) {
    throw new Error(
      'Select compound components must be used within a <Select> parent. ' +
      'Did you forget to wrap your component in <Select>?'
    );
  }
  return context;
}

export { SelectContext };
export type { SelectContextValue, SelectItemData };
```

The error message in `useSelectContext` is important. When someone uses `<Select.Item>` without a `<Select>` parent, they'll get a helpful error instead of `Cannot read properties of null`.

### Implementation: The Root Component

```typescript
// components/Select/Select.tsx
import { useRef, useState, useCallback, useMemo } from 'react';
import { View } from 'react-native';
import { SelectContext, SelectItemData } from './SelectContext';

interface SelectProps {
  value?: string;
  defaultValue?: string;
  onValueChange?: (value: string) => void;
  open?: boolean;
  defaultOpen?: boolean;
  onOpenChange?: (open: boolean) => void;
  children: React.ReactNode;
}

export function SelectRoot({
  value: controlledValue,
  defaultValue,
  onValueChange: controlledOnValueChange,
  open: controlledOpen,
  defaultOpen = false,
  onOpenChange: controlledOnOpenChange,
  children,
}: SelectProps) {
  // Support both controlled and uncontrolled modes
  const [internalValue, setInternalValue] = useState(defaultValue);
  const [internalOpen, setInternalOpen] = useState(defaultOpen);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const [searchQuery, setSearchQuery] = useState('');

  const isControlledValue = controlledValue !== undefined;
  const isControlledOpen = controlledOpen !== undefined;

  const value = isControlledValue ? controlledValue : internalValue;
  const open = isControlledOpen ? controlledOpen : internalOpen;

  const onValueChange = useCallback((newValue: string) => {
    if (!isControlledValue) setInternalValue(newValue);
    controlledOnValueChange?.(newValue);
  }, [isControlledValue, controlledOnValueChange]);

  const onOpenChange = useCallback((newOpen: boolean) => {
    if (!isControlledOpen) setInternalOpen(newOpen);
    controlledOnOpenChange?.(newOpen);
    if (!newOpen) setSearchQuery('');
  }, [isControlledOpen, controlledOnOpenChange]);

  // Item registration for keyboard navigation
  const itemsRef = useRef<Map<string, SelectItemData>>(new Map());

  const registerItem = useCallback((item: SelectItemData) => {
    itemsRef.current.set(item.value, item);
  }, []);

  const unregisterItem = useCallback((itemValue: string) => {
    itemsRef.current.delete(itemValue);
  }, []);

  const triggerRef = useRef<View>(null);
  const contentRef = useRef<View>(null);

  const items = Array.from(itemsRef.current.values());
  const selectedItem = items.find((item) => item.value === value);

  const filteredItems = useMemo(() => {
    if (!searchQuery) return items;
    return items.filter((item) =>
      item.label.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }, [items, searchQuery]);

  const contextValue = useMemo<SelectContextValue>(
    () => ({
      value,
      open,
      highlightedIndex,
      searchQuery,
      onValueChange,
      onOpenChange,
      onHighlightedIndexChange: setHighlightedIndex,
      onSearchQueryChange: setSearchQuery,
      triggerRef,
      contentRef,
      selectedItem,
      filteredItems,
      registerItem,
      unregisterItem,
    }),
    [value, open, highlightedIndex, searchQuery, selectedItem, filteredItems,
     onValueChange, onOpenChange, registerItem, unregisterItem],
  );

  return (
    <SelectContext.Provider value={contextValue}>
      {children}
    </SelectContext.Provider>
  );
}
```

### Implementation: Sub-Components

```typescript
// components/Select/SelectTrigger.tsx
import { forwardRef } from 'react';
import { Pressable, View, ViewProps } from 'react-native';
import { useSelectContext } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectTriggerProps extends Omit<ViewProps, 'children'> {
  children: React.ReactNode;
}

export const SelectTrigger = forwardRef<View, SelectTriggerProps>(
  function SelectTrigger({ children, style, ...rest }, ref) {
    const { open, onOpenChange, triggerRef } = useSelectContext();
    const theme = useTheme();

    return (
      <Pressable
        ref={triggerRef}
        onPress={() => onOpenChange(!open)}
        accessibilityRole="button"
        accessibilityState={{ expanded: open }}
        accessibilityLabel="Select option"
        style={[
          {
            flexDirection: 'row',
            alignItems: 'center',
            justifyContent: 'space-between',
            paddingHorizontal: theme.spacing.md,
            paddingVertical: theme.spacing.sm,
            borderRadius: theme.borderRadius.md,
            borderWidth: 1,
            borderColor: open ? theme.color.borderFocus : theme.color.border,
            backgroundColor: theme.color.surface,
            gap: theme.spacing.sm,
          },
          style,
        ]}
        {...rest}
      >
        {children}
      </Pressable>
    );
  },
);

// components/Select/SelectValue.tsx
import { Text } from '../../atoms/Text';
import { useSelectContext } from './SelectContext';

interface SelectValueProps {
  placeholder?: string;
}

export function SelectValue({ placeholder = 'Select...' }: SelectValueProps) {
  const { selectedItem } = useSelectContext();

  return (
    <Text
      variant="body"
      color={selectedItem ? 'primary' : 'tertiary'}
      numberOfLines={1}
      style={{ flex: 1 }}
    >
      {selectedItem?.label ?? placeholder}
    </Text>
  );
}

// components/Select/SelectItem.tsx
import { useEffect } from 'react';
import { Pressable, View } from 'react-native';
import { Text } from '../../atoms/Text';
import { useSelectContext } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectItemProps {
  value: string;
  disabled?: boolean;
  children: React.ReactNode;
}

export function SelectItem({ value, disabled = false, children }: SelectItemProps) {
  const ctx = useSelectContext();
  const theme = useTheme();
  const isSelected = ctx.value === value;

  // Register this item with the parent for keyboard navigation
  useEffect(() => {
    ctx.registerItem({ value, label: typeof children === 'string' ? children : value, disabled, index: 0 });
    return () => ctx.unregisterItem(value);
  }, [value, disabled]);

  const handlePress = () => {
    if (disabled) return;
    ctx.onValueChange(value);
    ctx.onOpenChange(false);
  };

  return (
    <Pressable
      onPress={handlePress}
      disabled={disabled}
      accessibilityRole="menuitem"
      accessibilityState={{ selected: isSelected, disabled }}
      style={({ pressed }) => [
        {
          flexDirection: 'row',
          alignItems: 'center',
          paddingHorizontal: theme.spacing.md,
          paddingVertical: theme.spacing.sm,
          gap: theme.spacing.sm,
          opacity: disabled ? 0.5 : 1,
          backgroundColor: pressed
            ? theme.color.surfaceHover
            : isSelected
              ? theme.color.backgroundSecondary
              : 'transparent',
        },
      ]}
    >
      {children}
    </Pressable>
  );
}
```

### Assembly: The Namespace Pattern

The final piece is assembling sub-components into a single namespace export:

```typescript
// components/Select/index.ts
import { SelectRoot } from './Select';
import { SelectTrigger } from './SelectTrigger';
import { SelectContent } from './SelectContent';
import { SelectValue } from './SelectValue';
import { SelectItem } from './SelectItem';
import { SelectItemText } from './SelectItemText';
import { SelectItemIcon } from './SelectItemIcon';
import { SelectIcon } from './SelectIcon';
import { SelectGroup } from './SelectGroup';
import { SelectLabel } from './SelectLabel';
import { SelectSeparator } from './SelectSeparator';
import { SelectSearch } from './SelectSearch';
import { SelectEmpty } from './SelectEmpty';

// Attach sub-components as properties of the root component
export const Select = Object.assign(SelectRoot, {
  Trigger: SelectTrigger,
  Content: SelectContent,
  Value: SelectValue,
  Item: SelectItem,
  ItemText: SelectItemText,
  ItemIcon: SelectItemIcon,
  Icon: SelectIcon,
  Group: SelectGroup,
  Label: SelectLabel,
  Separator: SelectSeparator,
  Search: SelectSearch,
  Empty: SelectEmpty,
});

// Type export for consumers who need it
export type { SelectProps } from './Select';
```

The `Object.assign` trick is what enables the `<Select.Trigger>` dot-notation syntax. TypeScript understands this pattern and provides full autocomplete for all sub-components.

---

## 4. THE HEADLESS UI PATTERN

Here's a question that every component library architect must answer: should your components own their rendering, or should they only provide behavior and let consumers bring their own JSX?

The headless pattern says: **separate the logic (state, a11y, keyboard navigation) from the rendering (styles, layout, animation).** The library provides hooks or renderless components; the consumer provides the visual layer.

### Why Headless Matters

The problem with styled component libraries (like old-school Material UI or Ant Design) is twofold:

1. **Fighting styles is painful.** You spend more time overriding defaults than building features. The library's CSS specificity wars become your problem.
2. **Accessibility is hard to bolt on.** If you build your own dropdown from scratch, you probably forget that it needs `role="listbox"`, `aria-activedescendant`, `aria-expanded`, keyboard arrow navigation, typeahead search, focus trapping, and Escape to close. That's not laziness; it's that the WAI-ARIA spec for a combobox is 47 pages.

Headless UI gives you the a11y and interaction logic -- the hard part -- and lets you own the easy part: how it looks.

### The Headless Hook Pattern

```typescript
// hooks/useToggle.ts -- simplest example
import { useState, useCallback } from 'react';

interface UseToggleOptions {
  defaultValue?: boolean;
  value?: boolean;
  onChange?: (value: boolean) => void;
}

interface UseToggleReturn {
  isOn: boolean;
  toggle: () => void;
  setOn: () => void;
  setOff: () => void;
  buttonProps: {
    role: 'switch';
    'aria-checked': boolean;
    onPress: () => void;
  };
}

export function useToggle(options: UseToggleOptions = {}): UseToggleReturn {
  const { defaultValue = false, value: controlledValue, onChange } = options;
  const [internalValue, setInternalValue] = useState(defaultValue);

  const isControlled = controlledValue !== undefined;
  const isOn = isControlled ? controlledValue : internalValue;

  const handleChange = useCallback((newValue: boolean) => {
    if (!isControlled) setInternalValue(newValue);
    onChange?.(newValue);
  }, [isControlled, onChange]);

  const toggle = useCallback(() => handleChange(!isOn), [isOn, handleChange]);
  const setOn = useCallback(() => handleChange(true), [handleChange]);
  const setOff = useCallback(() => handleChange(false), [handleChange]);

  return {
    isOn,
    toggle,
    setOn,
    setOff,
    buttonProps: {
      role: 'switch',
      'aria-checked': isOn,
      onPress: toggle,
    },
  };
}
```

The `buttonProps` pattern is key. The hook returns a bag of props that the consumer spreads onto their element. The hook handles the a11y attributes; the consumer handles the styling:

```typescript
// Consumer code
function MyFancyToggle() {
  const { isOn, buttonProps } = useToggle({ defaultValue: false });

  return (
    <Pressable
      {...buttonProps}
      style={{
        width: 50,
        height: 28,
        borderRadius: 14,
        backgroundColor: isOn ? '#22C55E' : '#D1D5DB',
        justifyContent: 'center',
        padding: 2,
      }}
    >
      <Animated.View
        style={{
          width: 24,
          height: 24,
          borderRadius: 12,
          backgroundColor: '#FFFFFF',
          transform: [{ translateX: isOn ? 22 : 0 }],
        }}
      />
    </Pressable>
  );
}
```

### A Real-World Headless Hook: useCombobox

```typescript
// hooks/useCombobox.ts
import { useState, useRef, useCallback, useMemo } from 'react';
import { TextInput } from 'react-native';

interface UseComboboxOptions<T> {
  items: T[];
  itemToString: (item: T) => string;
  selectedItem?: T | null;
  defaultSelectedItem?: T | null;
  onSelectedItemChange?: (item: T | null) => void;
  onInputValueChange?: (value: string) => void;
  filterItems?: (items: T[], inputValue: string) => T[];
}

interface UseComboboxReturn<T> {
  // State
  isOpen: boolean;
  inputValue: string;
  highlightedIndex: number;
  selectedItem: T | null;
  filteredItems: T[];

  // Prop getters (the headless pattern's core API)
  getInputProps: () => {
    value: string;
    onChangeText: (text: string) => void;
    onFocus: () => void;
    onBlur: () => void;
    role: 'combobox';
    accessibilityState: { expanded: boolean };
    'aria-activedescendant': string | undefined;
    'aria-autocomplete': 'list';
    'aria-controls': string;
    ref: React.RefObject<TextInput>;
  };

  getMenuProps: () => {
    role: 'listbox';
    accessibilityLabel: string;
    nativeID: string;
  };

  getItemProps: (options: { item: T; index: number }) => {
    role: 'option';
    nativeID: string;
    accessibilityState: { selected: boolean };
    onPress: () => void;
    style: { backgroundColor: string };
  };

  getToggleButtonProps: () => {
    onPress: () => void;
    accessibilityLabel: string;
    'aria-expanded': boolean;
  };

  // Actions
  openMenu: () => void;
  closeMenu: () => void;
  reset: () => void;
}

export function useCombobox<T>(options: UseComboboxOptions<T>): UseComboboxReturn<T> {
  const {
    items,
    itemToString,
    selectedItem: controlledSelectedItem,
    defaultSelectedItem = null,
    onSelectedItemChange,
    onInputValueChange,
    filterItems = defaultFilter,
  } = options;

  const [isOpen, setIsOpen] = useState(false);
  const [inputValue, setInputValue] = useState(
    defaultSelectedItem ? itemToString(defaultSelectedItem) : '',
  );
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const [internalSelectedItem, setInternalSelectedItem] = useState(defaultSelectedItem);

  const inputRef = useRef<TextInput>(null);
  const menuId = useRef(`combobox-menu-${Math.random().toString(36).slice(2)}`).current;

  const isControlled = controlledSelectedItem !== undefined;
  const selectedItem = isControlled ? controlledSelectedItem : internalSelectedItem;

  const filteredItems = useMemo(
    () => filterItems(items, inputValue),
    [items, inputValue, filterItems],
  );

  const selectItem = useCallback((item: T | null) => {
    if (!isControlled) setInternalSelectedItem(item);
    onSelectedItemChange?.(item);
    setInputValue(item ? itemToString(item) : '');
    setIsOpen(false);
    setHighlightedIndex(-1);
  }, [isControlled, onSelectedItemChange, itemToString]);

  function defaultFilter(allItems: T[], query: string): T[] {
    if (!query) return allItems;
    const lower = query.toLowerCase();
    return allItems.filter((item) =>
      itemToString(item).toLowerCase().includes(lower),
    );
  }

  const getInputProps = useCallback(() => ({
    value: inputValue,
    onChangeText: (text: string) => {
      setInputValue(text);
      onInputValueChange?.(text);
      if (!isOpen) setIsOpen(true);
      setHighlightedIndex(0);
    },
    onFocus: () => setIsOpen(true),
    onBlur: () => {
      // Delay to allow item press to fire first
      setTimeout(() => setIsOpen(false), 150);
    },
    role: 'combobox' as const,
    accessibilityState: { expanded: isOpen },
    'aria-activedescendant': highlightedIndex >= 0
      ? `${menuId}-item-${highlightedIndex}`
      : undefined,
    'aria-autocomplete': 'list' as const,
    'aria-controls': menuId,
    ref: inputRef,
  }), [inputValue, isOpen, highlightedIndex, menuId, onInputValueChange]);

  const getMenuProps = useCallback(() => ({
    role: 'listbox' as const,
    accessibilityLabel: 'Suggestions',
    nativeID: menuId,
  }), [menuId]);

  const getItemProps = useCallback(({ item, index }: { item: T; index: number }) => ({
    role: 'option' as const,
    nativeID: `${menuId}-item-${index}`,
    accessibilityState: { selected: item === selectedItem },
    onPress: () => selectItem(item),
    style: {
      backgroundColor: index === highlightedIndex ? '#F3F4F6' : 'transparent',
    },
  }), [menuId, selectedItem, highlightedIndex, selectItem]);

  const getToggleButtonProps = useCallback(() => ({
    onPress: () => setIsOpen((prev) => !prev),
    accessibilityLabel: isOpen ? 'Close suggestions' : 'Open suggestions',
    'aria-expanded': isOpen,
  }), [isOpen]);

  return {
    isOpen,
    inputValue,
    highlightedIndex,
    selectedItem,
    filteredItems,
    getInputProps,
    getMenuProps,
    getItemProps,
    getToggleButtonProps,
    openMenu: () => setIsOpen(true),
    closeMenu: () => setIsOpen(false),
    reset: () => {
      selectItem(null);
      setInputValue('');
    },
  };
}
```

Usage:

```typescript
// Consumer brings ALL the rendering
interface Fruit {
  name: string;
  emoji: string;
}

const fruits: Fruit[] = [
  { name: 'Apple', emoji: '🍎' },
  { name: 'Banana', emoji: '🍌' },
  { name: 'Cherry', emoji: '🍒' },
];

function FruitPicker() {
  const {
    isOpen,
    filteredItems,
    getInputProps,
    getMenuProps,
    getItemProps,
    getToggleButtonProps,
  } = useCombobox({
    items: fruits,
    itemToString: (fruit) => fruit.name,
    onSelectedItemChange: (fruit) => console.log('Selected:', fruit),
  });

  return (
    <View>
      <View style={{ flexDirection: 'row', alignItems: 'center' }}>
        <TextInput
          {...getInputProps()}
          placeholder="Search fruits..."
          style={{ flex: 1, borderWidth: 1, padding: 8, borderRadius: 8 }}
        />
        <Pressable {...getToggleButtonProps()}>
          <Icon name={isOpen ? 'chevron-up' : 'chevron-down'} />
        </Pressable>
      </View>

      {isOpen && filteredItems.length > 0 && (
        <View {...getMenuProps()} style={{ borderWidth: 1, borderRadius: 8, marginTop: 4 }}>
          {filteredItems.map((fruit, index) => (
            <Pressable
              key={fruit.name}
              {...getItemProps({ item: fruit, index })}
              style={[getItemProps({ item: fruit, index }).style, { padding: 12 }]}
            >
              <Text>{fruit.emoji} {fruit.name}</Text>
            </Pressable>
          ))}
        </View>
      )}
    </View>
  );
}
```

### Headless Primitive Libraries

Don't build headless hooks from scratch for every component. Stand on the shoulders of libraries that have already solved the hard a11y problems:

```
┌──────────────────────────────────────────────────────────────────────┐
│  HEADLESS UI LIBRARY COMPARISON                                       │
│                                                                       │
│  LIBRARY         APPROACH        PLATFORM    KEY STRENGTH             │
│  ──────────────  ──────────────  ──────────  ────────────────────     │
│  Radix UI        Unstyled        Web         Most complete a11y       │
│                  components                  implementation; great    │
│                                              compound component API   │
│                                                                       │
│  React Aria      Hooks           Web +       Adobe-backed; most      │
│  (Adobe)                         RN (some)   spec-compliant a11y;    │
│                                              internationalization     │
│                                                                       │
│  Ark UI          Unstyled        Web + RN    Built on Zag.js state   │
│  (Chakra team)   components                  machines; framework     │
│                                              agnostic core           │
│                                                                       │
│  Downshift       Hooks           Any         Best combobox/select    │
│                                              hook; small and focused │
│                                                                       │
│  Headless UI     Unstyled        Web         Tailwind Labs; simple   │
│  (Tailwind)      components                  API; fewer components   │
│                                                                       │
│  RECOMMENDATION:                                                      │
│  Web: Radix UI primitives + your styling layer                       │
│  React Native: React Aria hooks where available + custom hooks       │
│  Universal: Ark UI (Zag.js state machines work everywhere)           │
└──────────────────────────────────────────────────────────────────────┘
```

### Layering: Headless -> Styled -> Themed

The best component libraries use a three-layer architecture:

```typescript
// Layer 1: Headless (behavior only)
// From a library like Radix, React Aria, or your own hooks
import * as RadixSelect from '@radix-ui/react-select';

// Layer 2: Styled (design system defaults)
// Your library provides this
import { styled } from '../styled';

const StyledTrigger = styled(RadixSelect.Trigger, {
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'space-between',
  borderRadius: '$md',
  border: '1px solid $border',
  padding: '$sm $md',
  backgroundColor: '$surface',
  '&:focus': { borderColor: '$borderFocus', outline: 'none' },
});

const StyledContent = styled(RadixSelect.Content, {
  backgroundColor: '$surface',
  borderRadius: '$md',
  border: '1px solid $border',
  boxShadow: '$lg',
  overflow: 'hidden',
});

const StyledItem = styled(RadixSelect.Item, {
  padding: '$sm $md',
  cursor: 'pointer',
  '&[data-highlighted]': { backgroundColor: '$surfaceHover' },
  '&[data-state="checked"]': { backgroundColor: '$backgroundSecondary' },
});

// Layer 3: Themed (product-specific overrides)
// Consumers can override at the theme level
const productTheme = createTheme({
  colors: {
    surface: '#1A1A2E',        // Dark mode
    surfaceHover: '#16213E',
    border: '#0F3460',
  },
});
```

---

## 5. POLYMORPHIC COMPONENTS

The `as` prop pattern lets a component render as a different element while keeping its styling and behavior. This is one of the most useful patterns in component libraries, and one of the hardest to type correctly.

### The Problem

You have a `Button` component. It looks like a button. But sometimes you need it to:
- Be a link (`<a href="/about">`)
- Be a React Navigation `<Link>` component
- Be a `<div>` with click handling (for layout reasons)
- Be a custom wrapper component

Without polymorphism, you end up with:

```typescript
// BAD: Separate components for each case
<Button onPress={handlePress}>Click me</Button>
<LinkButton href="/about">About</LinkButton>
<ButtonAsDiv onClick={handleClick}>Wrapper</ButtonAsDiv>

// Or worse: stringly-typed escape hatches
<Button href="/about" renderAs="a">About</Button>
```

### The Solution: The `as` Prop

```typescript
// components/Box.tsx -- the foundation polymorphic component
import { ComponentPropsWithoutRef, ElementType, forwardRef, ReactNode } from 'react';

// The magic type: extract props based on the `as` element type
type PolymorphicProps<E extends ElementType, Props = {}> = Props &
  Omit<ComponentPropsWithoutRef<E>, keyof Props | 'as'> & {
    as?: E;
  };

// For forwardRef, we also need the ref type
type PolymorphicRef<E extends ElementType> =
  ComponentPropsWithoutRef<E> extends { ref?: infer R } ? R : never;

// Example: Polymorphic Button (Web)
type ButtonOwnProps = {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
  children: ReactNode;
};

type ButtonProps<E extends ElementType = 'button'> = PolymorphicProps<E, ButtonOwnProps>;

// The component with proper type inference
export const Button = forwardRef(function Button<E extends ElementType = 'button'>(
  { as, variant = 'primary', size = 'md', loading, children, ...rest }: ButtonProps<E>,
  ref: PolymorphicRef<E>,
) {
  const Component = as || 'button';

  return (
    <Component
      ref={ref}
      className={`btn btn-${variant} btn-${size}`}
      disabled={loading}
      {...rest}
    >
      {loading ? <Spinner /> : children}
    </Component>
  );
}) as <E extends ElementType = 'button'>(props: ButtonProps<E>) => JSX.Element;
```

Now TypeScript enforces the correct props based on what you pass to `as`:

```typescript
// Renders a <button> -- accepts button props
<Button variant="primary" onClick={handleClick}>
  Click me
</Button>

// Renders an <a> -- accepts anchor props
<Button as="a" href="/about" target="_blank">
  About
</Button>

// Renders a Next.js Link -- accepts Link props
<Button as={Link} href="/dashboard">
  Dashboard
</Button>

// TYPE ERROR: 'href' doesn't exist on <button>
<Button href="/about">About</Button>
//       ^^^^ Error!

// TYPE ERROR: 'onPress' doesn't exist on <a>
<Button as="a" onPress={handler}>About</Button>
//              ^^^^^^^ Error!
```

### React Native Polymorphism

React Native's component model is different from web. The `as` pattern doesn't map as cleanly because RN doesn't have HTML elements. Instead, we use a variation:

```typescript
// React Native polymorphic pattern
import { View, ViewProps, Pressable, PressableProps } from 'react-native';
import { Link, LinkProps } from 'expo-router';

// Discriminated union approach for RN
type CardProps =
  | ({ pressable?: false } & ViewProps)
  | ({ pressable: true } & PressableProps)
  | ({ pressable?: false; href: string } & Omit<LinkProps, 'children'>);

export function Card(props: CardProps & { children: React.ReactNode }) {
  if ('href' in props && props.href) {
    const { href, children, ...rest } = props;
    return (
      <Link href={href} asChild {...rest}>
        <View style={cardStyles}>{children}</View>
      </Link>
    );
  }

  if (props.pressable) {
    const { pressable, children, ...rest } = props;
    return (
      <Pressable style={cardStyles} {...rest}>
        {children}
      </Pressable>
    );
  }

  const { children, ...rest } = props;
  return (
    <View style={cardStyles} {...rest}>
      {children}
    </View>
  );
}
```

### When to Use Polymorphism

```
USE polymorphic components when:
  - The visual appearance is the same but the underlying element differs
    (button that's a link, heading that's a different tag level)
  - You're building a primitive component library (Box, Text, Button)
  - The consumer needs to integrate with routing libraries

AVOID polymorphic components when:
  - The behavior changes significantly between element types
  - You're working in React Native (prefer discriminated unions)
  - The TypeScript complexity outweighs the ergonomic benefit
  - Only one or two variants exist (just make two components)
```

---

## 6. THE SLOT PATTERN

Slots give consumers injection points into specific regions of a component. Think of them as named holes in a template that consumers fill with their own content.

### Slots via Props

```typescript
// The prop-based slot pattern
interface DialogProps {
  // Slot props
  header?: React.ReactNode;
  footer?: React.ReactNode;
  icon?: React.ReactNode;

  // Content
  title: string;
  description?: string;
  children?: React.ReactNode;

  // Behavior
  open: boolean;
  onClose: () => void;
}

function Dialog({ header, footer, icon, title, description, children, open, onClose }: DialogProps) {
  if (!open) return null;

  return (
    <View style={styles.overlay}>
      <View style={styles.container}>
        {/* Header slot: use custom header or default */}
        {header ?? (
          <View style={styles.header}>
            {icon && <View style={styles.iconContainer}>{icon}</View>}
            <Text variant="heading3">{title}</Text>
            {description && <Text variant="body" color="secondary">{description}</Text>}
          </View>
        )}

        {/* Body: always use children */}
        {children && <View style={styles.body}>{children}</View>}

        {/* Footer slot: use custom footer or default close button */}
        {footer ?? (
          <View style={styles.footer}>
            <Button variant="primary" onPress={onClose}>OK</Button>
          </View>
        )}
      </View>
    </View>
  );
}

// Usage with defaults:
<Dialog
  open={isOpen}
  onClose={() => setIsOpen(false)}
  title="Confirm Delete"
  description="This action cannot be undone."
/>

// Usage with custom slots:
<Dialog
  open={isOpen}
  onClose={() => setIsOpen(false)}
  title="Confirm Delete"
  icon={<Icon name="trash" color="red" size={32} />}
  footer={
    <View style={{ flexDirection: 'row', gap: 12 }}>
      <Button variant="ghost" onPress={() => setIsOpen(false)}>Cancel</Button>
      <Button variant="danger" onPress={handleDelete}>Delete Forever</Button>
    </View>
  }
>
  <Text>Are you sure you want to delete "Project Alpha"?</Text>
  <Text variant="caption" color="tertiary">All files and data will be permanently removed.</Text>
</Dialog>
```

### Slots via Children Composition (Compound Components)

The alternative to slot props is children composition. Instead of passing JSX into named props, you compose sub-components inside children:

```typescript
// Compound component approach (same Dialog)
<Dialog open={isOpen} onClose={() => setIsOpen(false)}>
  <Dialog.Header>
    <Dialog.Icon><Icon name="trash" color="red" size={32} /></Dialog.Icon>
    <Dialog.Title>Confirm Delete</Dialog.Title>
    <Dialog.Description>This action cannot be undone.</Dialog.Description>
  </Dialog.Header>

  <Dialog.Body>
    <Text>Are you sure you want to delete "Project Alpha"?</Text>
  </Dialog.Body>

  <Dialog.Footer>
    <Button variant="ghost" onPress={() => setIsOpen(false)}>Cancel</Button>
    <Button variant="danger" onPress={handleDelete}>Delete Forever</Button>
  </Dialog.Footer>
</Dialog>
```

### Render Callback Slots

Sometimes a slot needs data from the parent. Render callback slots (a.k.a. render props) solve this:

```typescript
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  renderEmpty?: () => React.ReactNode;
  renderSeparator?: () => React.ReactNode;
  renderHeader?: (count: number) => React.ReactNode;
}

function List<T>({ items, renderItem, renderEmpty, renderSeparator, renderHeader }: ListProps<T>) {
  if (items.length === 0 && renderEmpty) {
    return <>{renderEmpty()}</>;
  }

  return (
    <View>
      {renderHeader?.(items.length)}
      {items.map((item, index) => (
        <Fragment key={index}>
          {index > 0 && renderSeparator?.()}
          {renderItem(item, index)}
        </Fragment>
      ))}
    </View>
  );
}

// Usage:
<List
  items={users}
  renderItem={(user) => (
    <UserCard key={user.id} name={user.name} avatar={user.avatar} />
  )}
  renderEmpty={() => <EmptyState message="No users found" />}
  renderSeparator={() => <Divider />}
  renderHeader={(count) => <Text variant="caption">{count} users</Text>}
/>
```

### The Trade-offs

```
┌──────────────────────────────────────────────────────────────────────┐
│  SLOT PATTERN COMPARISON                                              │
│                                                                       │
│  APPROACH              PROS                   CONS                    │
│  ────────────────────  ─────────────────────  ──────────────────────  │
│  Slot props            Simple to implement    Hard to enforce order   │
│  (header={<...>})      Easy to understand     No implicit context     │
│                        Good for 1-3 slots     Gets messy with >3     │
│                                                                       │
│  Children composition  Full layout control    More verbose            │
│  (compound components) Implicit context       Higher implementation   │
│                        Self-documenting       cost                    │
│                        Scales to many slots                           │
│                                                                       │
│  Render callbacks      Parent can pass data   Awkward syntax          │
│  (renderItem)          Type-safe data flow    Re-render concerns      │
│                        Flexible               Harder to read          │
│                                                                       │
│  GUIDELINE:                                                           │
│  ≤3 optional regions -> slot props                                    │
│  >3 regions or complex layout -> compound components                 │
│  Slot needs parent data -> render callback                           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 7. CONTROLLED VS UNCONTROLLED

This is the most frequently botched pattern in component libraries. Get it wrong and consumers either can't control the component when they need to, or they're forced to manage state they don't care about.

### The Spectrum

```
┌──────────────────────────────────────────────────────────────────────┐
│  THE CONTROL SPECTRUM                                                 │
│                                                                       │
│  UNCONTROLLED                                  CONTROLLED             │
│  ←──────────────────────────────────────────────────────→            │
│                                                                       │
│  Component manages       Both parent and        Parent manages        │
│  its own state.          component can           all state.            │
│  Parent has no           manage state.           Component is          │
│  control.                Component has a         stateless.            │
│                          default but defers                            │
│                          to parent.                                    │
│                                                                       │
│  <Accordion>             <Accordion               <Accordion          │
│                           defaultValue="a">        value={value}       │
│  "Just work"              "Start here,             onValueChange=      │
│                            but I can override"      {setValue}>         │
│                                                                        │
│  Simplest for consumer.  Best of both worlds.   Most flexible.        │
│  No flexibility.         Slightly more complex   Consumer MUST         │
│                          implementation.          manage state.         │
└──────────────────────────────────────────────────────────────────────┘
```

### The "Controlled with a Default" Pattern

This is the gold standard. The component works out of the box (uncontrolled) but can be fully controlled when needed:

```typescript
// hooks/useControllableState.ts
import { useState, useCallback, useRef, useEffect } from 'react';

interface UseControllableStateOptions<T> {
  value?: T;
  defaultValue: T;
  onChange?: (value: T) => void;
}

export function useControllableState<T>({
  value: controlledValue,
  defaultValue,
  onChange,
}: UseControllableStateOptions<T>): [T, (value: T | ((prev: T) => T)) => void] {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;

  // Warn if switching between controlled and uncontrolled
  const isControlledRef = useRef(isControlled);
  useEffect(() => {
    if (isControlledRef.current !== isControlled) {
      console.warn(
        'A component is changing from ' +
        (isControlledRef.current ? 'controlled' : 'uncontrolled') +
        ' to ' +
        (isControlled ? 'controlled' : 'uncontrolled') +
        '. This is likely a bug. Decide between controlled and uncontrolled for the lifetime of the component.',
      );
    }
    isControlledRef.current = isControlled;
  }, [isControlled]);

  const setValue = useCallback(
    (nextValue: T | ((prev: T) => T)) => {
      const resolvedValue =
        typeof nextValue === 'function'
          ? (nextValue as (prev: T) => T)(value)
          : nextValue;

      if (!isControlled) {
        setInternalValue(resolvedValue);
      }
      onChange?.(resolvedValue);
    },
    [isControlled, onChange, value],
  );

  return [value, setValue];
}
```

Now every component uses this hook:

```typescript
// components/Accordion.tsx
interface AccordionProps {
  // Controlled
  value?: string[];
  onValueChange?: (value: string[]) => void;
  // Uncontrolled
  defaultValue?: string[];
  // Shared
  type?: 'single' | 'multiple';
  children: React.ReactNode;
}

function Accordion({
  value,
  defaultValue = [],
  onValueChange,
  type = 'single',
  children,
}: AccordionProps) {
  const [openItems, setOpenItems] = useControllableState({
    value,
    defaultValue,
    onChange: onValueChange,
  });

  const toggleItem = useCallback((itemValue: string) => {
    setOpenItems((prev) => {
      if (type === 'single') {
        return prev.includes(itemValue) ? [] : [itemValue];
      }
      return prev.includes(itemValue)
        ? prev.filter((v) => v !== itemValue)
        : [...prev, itemValue];
    });
  }, [type, setOpenItems]);

  // ... context provider with openItems and toggleItem
}
```

Consumer experience:

```typescript
// Uncontrolled: "just work"
<Accordion defaultValue={['item-1']}>
  <Accordion.Item value="item-1">
    <Accordion.Trigger>Section 1</Accordion.Trigger>
    <Accordion.Content>Content 1</Accordion.Content>
  </Accordion.Item>
  <Accordion.Item value="item-2">
    <Accordion.Trigger>Section 2</Accordion.Trigger>
    <Accordion.Content>Content 2</Accordion.Content>
  </Accordion.Item>
</Accordion>

// Controlled: full control
const [openSections, setOpenSections] = useState(['item-1']);

<Accordion value={openSections} onValueChange={setOpenSections}>
  {/* same children */}
</Accordion>

// You can also sync with URL params, analytics, etc.
<Accordion
  value={searchParams.getAll('section')}
  onValueChange={(sections) => {
    setSearchParams({ section: sections });
    analytics.track('accordion_toggle', { sections });
  }}
>
  {/* same children */}
</Accordion>
```

### Decision Framework

```
When should a component be controlled vs uncontrolled?

ALWAYS support both (use useControllableState) for:
  - Form inputs (TextInput, Select, Checkbox, RadioGroup)
  - Toggle states (Accordion, Tabs, Disclosure, Dialog open state)
  - Any component where the parent might need to read or set the value

Default to UNCONTROLLED (with optional controlled override) for:
  - Animation states
  - Internal UI states (hover, focus, scroll position)
  - Ephemeral states that don't affect data flow

Force CONTROLLED for:
  - Components that MUST sync with external state (form libraries)
  - Components where forgetting to pass the handler is a bug
```

---

## 8. GENERIC COMPONENTS WITH TYPESCRIPT

Generic components let you build type-safe, reusable containers where the item type flows through the entire component tree. This is essential for lists, tables, selectors, and any component that renders a collection of typed data.

### The Basic Pattern

```typescript
// components/List.tsx
import { FlatList, FlatListProps, View } from 'react-native';
import { Text } from '../atoms/Text';

interface ListProps<T> extends Omit<FlatListProps<T>, 'data' | 'renderItem'> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactElement;
  keyExtractor: (item: T) => string;
  emptyMessage?: string;
  header?: React.ReactNode;
  footer?: React.ReactNode;
}

export function List<T>({
  items,
  renderItem,
  keyExtractor,
  emptyMessage = 'No items',
  header,
  footer,
  ...flatListProps
}: ListProps<T>) {
  return (
    <FlatList
      data={items}
      keyExtractor={(item, index) => keyExtractor(item)}
      renderItem={({ item, index }) => renderItem(item, index)}
      ListEmptyComponent={<Text variant="body" color="tertiary">{emptyMessage}</Text>}
      ListHeaderComponent={header ? <>{header}</> : null}
      ListFooterComponent={footer ? <>{footer}</> : null}
      {...flatListProps}
    />
  );
}
```

TypeScript infers `T` from the `items` prop, so `renderItem` is automatically typed:

```typescript
interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'member';
}

const users: User[] = [/* ... */];

// T is inferred as User. renderItem receives User, not any.
<List
  items={users}
  keyExtractor={(user) => user.id}           // user: User ✓
  renderItem={(user) => (                     // user: User ✓
    <View>
      <Text>{user.name}</Text>                {/* TypeScript knows user.name exists */}
      <Text>{user.email}</Text>               {/* TypeScript knows user.email exists */}
      <Badge>{user.role}</Badge>              {/* TypeScript knows user.role is 'admin' | 'member' */}
    </View>
  )}
/>
```

### Generic Select Component

```typescript
// components/TypedSelect.tsx
import { useState, useCallback } from 'react';

interface TypedSelectProps<T> {
  items: T[];
  value?: T | null;
  defaultValue?: T | null;
  onValueChange?: (item: T) => void;
  itemToString: (item: T) => string;
  itemToKey: (item: T) => string;
  renderItem?: (item: T, isSelected: boolean) => React.ReactNode;
  placeholder?: string;
  children?: React.ReactNode;
}

export function TypedSelect<T>({
  items,
  value: controlledValue,
  defaultValue = null,
  onValueChange,
  itemToString,
  itemToKey,
  renderItem,
  placeholder = 'Select...',
}: TypedSelectProps<T>) {
  const [internalValue, setInternalValue] = useState(defaultValue);
  const isControlled = controlledValue !== undefined;
  const value = isControlled ? controlledValue : internalValue;

  const handleSelect = useCallback((item: T) => {
    if (!isControlled) setInternalValue(item);
    onValueChange?.(item);
  }, [isControlled, onValueChange]);

  return (
    <View>
      <Pressable style={styles.trigger}>
        <Text>{value ? itemToString(value) : placeholder}</Text>
      </Pressable>

      <View style={styles.content}>
        {items.map((item) => {
          const isSelected = value !== null && itemToKey(item) === itemToKey(value);
          return (
            <Pressable
              key={itemToKey(item)}
              onPress={() => handleSelect(item)}
              style={[styles.item, isSelected && styles.selectedItem]}
            >
              {renderItem ? renderItem(item, isSelected) : (
                <Text>{itemToString(item)}</Text>
              )}
            </Pressable>
          );
        })}
      </View>
    </View>
  );
}
```

Usage with full type inference:

```typescript
interface Country {
  code: string;
  name: string;
  flag: string;
  population: number;
}

const countries: Country[] = [
  { code: 'US', name: 'United States', flag: '🇺🇸', population: 331_000_000 },
  { code: 'JP', name: 'Japan', flag: '🇯🇵', population: 125_000_000 },
  { code: 'DE', name: 'Germany', flag: '🇩🇪', population: 83_000_000 },
];

// T = Country, inferred from items
<TypedSelect
  items={countries}
  itemToString={(c) => c.name}     // c: Country ✓
  itemToKey={(c) => c.code}        // c: Country ✓
  onValueChange={(country) => {
    // country: Country ✓
    console.log(country.flag);     // Works!
    console.log(country.population); // Works!
  }}
  renderItem={(country, isSelected) => (
    // country: Country ✓
    <View style={{ flexDirection: 'row', gap: 8 }}>
      <Text>{country.flag}</Text>
      <Text weight={isSelected ? 'bold' : 'regular'}>{country.name}</Text>
      <Text variant="caption" color="tertiary">
        {(country.population / 1_000_000).toFixed(0)}M
      </Text>
    </View>
  )}
/>
```

### Advanced: Generic Components with Constraints

```typescript
// Constrain T to types that have an 'id' field
interface Identifiable {
  id: string | number;
}

interface DataTableProps<T extends Identifiable> {
  data: T[];
  columns: Column<T>[];
  onRowPress?: (item: T) => void;
  selectedIds?: Set<T['id']>;
  onSelectionChange?: (ids: Set<T['id']>) => void;
}

interface Column<T> {
  key: keyof T & string;
  header: string;
  width?: number;
  render?: (value: T[keyof T], item: T) => React.ReactNode;
  sortable?: boolean;
}

export function DataTable<T extends Identifiable>({
  data,
  columns,
  onRowPress,
  selectedIds,
  onSelectionChange,
}: DataTableProps<T>) {
  return (
    <View>
      {/* Header row */}
      <View style={styles.headerRow}>
        {columns.map((col) => (
          <View key={col.key} style={[styles.cell, col.width ? { width: col.width } : { flex: 1 }]}>
            <Text variant="label">{col.header}</Text>
          </View>
        ))}
      </View>

      {/* Data rows */}
      {data.map((item) => (
        <Pressable
          key={item.id}
          onPress={() => onRowPress?.(item)}
          style={[
            styles.row,
            selectedIds?.has(item.id) && styles.selectedRow,
          ]}
        >
          {columns.map((col) => (
            <View key={col.key} style={[styles.cell, col.width ? { width: col.width } : { flex: 1 }]}>
              {col.render
                ? col.render(item[col.key], item)
                : <Text>{String(item[col.key])}</Text>
              }
            </View>
          ))}
        </Pressable>
      ))}
    </View>
  );
}
```

Usage:

```typescript
interface Employee {
  id: number;
  name: string;
  department: string;
  salary: number;
  startDate: string;
}

<DataTable<Employee>
  data={employees}
  columns={[
    { key: 'name', header: 'Name', width: 200 },
    { key: 'department', header: 'Department' },
    {
      key: 'salary',
      header: 'Salary',
      render: (value) => <Text>${(value as number).toLocaleString()}</Text>,
      sortable: true,
    },
    {
      key: 'startDate',
      header: 'Start Date',
      render: (value) => <Text>{new Date(value as string).toLocaleDateString()}</Text>,
    },
  ]}
  onRowPress={(employee) => navigate(`/employees/${employee.id}`)}
  selectedIds={selectedEmployees}
  onSelectionChange={setSelectedEmployees}
/>
```

### The forwardRef + Generics Problem

React's `forwardRef` erases generics. Here's the workaround:

```typescript
// The problem: forwardRef strips the generic
// This LOSES the generic T:
const List = forwardRef(<T,>(props: ListProps<T>, ref: Ref<FlatList>) => {
  // ...
});

// Solution 1: Type assertion
export const List = forwardRef(function List<T>(
  props: ListProps<T>,
  ref: React.ForwardedRef<FlatList>,
) {
  // implementation
}) as <T>(props: ListProps<T> & { ref?: React.Ref<FlatList> }) => React.ReactElement;

// Solution 2: Wrapper function (cleaner)
function ListInner<T>(props: ListProps<T>, ref: React.ForwardedRef<FlatList>) {
  // implementation
}

export const List = forwardRef(ListInner) as <T>(
  props: ListProps<T> & { ref?: React.Ref<FlatList> },
) => React.ReactElement;

// Solution 3: Skip forwardRef entirely (React 19+)
// In React 19, ref is a regular prop. No forwardRef needed.
export function List<T>({ ref, ...props }: ListProps<T> & { ref?: React.Ref<FlatList> }) {
  // Use ref directly
}
```

---

## 9. FORWARDREF AND IMPERATIVE HANDLES

Most components communicate declaratively through props. But sometimes a parent needs to imperatively tell a child to do something: scroll to a position, focus an input, reset internal state, play an animation.

### When Imperative Handles Are Appropriate

```
GOOD use cases for imperative handles:
  - Focus management (focus an input programmatically)
  - Scroll control (scrollTo, scrollToEnd, scrollToIndex)
  - Animation triggers (play, pause, reset)
  - Form resets (clear internal state without controlled mode)
  - Measurement (get layout dimensions)

BAD use cases (use props/state instead):
  - Setting values (use controlled props)
  - Toggling visibility (use state + conditional rendering)
  - Triggering data fetches (use effects)
  - Anything that can be expressed declaratively
```

### Basic forwardRef

```typescript
// components/TextInput.tsx
import { forwardRef, useRef, useImperativeHandle } from 'react';
import { TextInput as RNTextInput, TextInputProps as RNTextInputProps, View } from 'react-native';
import { useTheme } from '../theme';

interface TextInputProps extends RNTextInputProps {
  label?: string;
  error?: string;
}

// Forward the ref so parents can call .focus(), .blur(), .clear()
export const TextInput = forwardRef<RNTextInput, TextInputProps>(
  function TextInput({ label, error, style, ...rest }, ref) {
    const theme = useTheme();

    return (
      <View style={{ gap: theme.spacing.xs }}>
        {label && <Text variant="label">{label}</Text>}
        <RNTextInput
          ref={ref}
          style={[
            {
              borderWidth: 1,
              borderColor: error ? theme.color.error : theme.color.border,
              borderRadius: theme.borderRadius.md,
              paddingHorizontal: theme.spacing.md,
              paddingVertical: theme.spacing.sm,
              fontSize: theme.fontSize.md,
              color: theme.color.text,
            },
            style,
          ]}
          placeholderTextColor={theme.color.textTertiary}
          {...rest}
        />
        {error && <Text variant="caption" color="error">{error}</Text>}
      </View>
    );
  },
);

// Parent usage:
function LoginForm() {
  const emailRef = useRef<RNTextInput>(null);
  const passwordRef = useRef<RNTextInput>(null);

  return (
    <View>
      <TextInput
        ref={emailRef}
        label="Email"
        returnKeyType="next"
        onSubmitEditing={() => passwordRef.current?.focus()}  // Focus next input
      />
      <TextInput
        ref={passwordRef}
        label="Password"
        secureTextEntry
        returnKeyType="done"
        onSubmitEditing={handleLogin}
      />
    </View>
  );
}
```

### useImperativeHandle: Custom Imperative APIs

When the native ref isn't enough and you need to expose custom methods:

```typescript
// components/ScrollableList.tsx
import { forwardRef, useRef, useImperativeHandle } from 'react';
import { FlatList, FlatListProps, LayoutAnimation, View } from 'react-native';

// Define the imperative API
export interface ScrollableListHandle {
  scrollToTop: (animated?: boolean) => void;
  scrollToIndex: (index: number, animated?: boolean) => void;
  scrollToEnd: (animated?: boolean) => void;
  getVisibleItems: () => { first: number; last: number };
  refresh: () => void;
}

interface ScrollableListProps<T> extends Omit<FlatListProps<T>, 'ref'> {
  onRefresh?: () => Promise<void>;
}

export const ScrollableList = forwardRef<ScrollableListHandle, ScrollableListProps<any>>(
  function ScrollableList({ onRefresh, ...props }, ref) {
    const flatListRef = useRef<FlatList>(null);
    const visibleItemsRef = useRef({ first: 0, last: 0 });
    const [refreshing, setRefreshing] = useState(false);

    // Expose a controlled, type-safe imperative API
    useImperativeHandle(ref, () => ({
      scrollToTop(animated = true) {
        flatListRef.current?.scrollToOffset({ offset: 0, animated });
      },

      scrollToIndex(index: number, animated = true) {
        flatListRef.current?.scrollToIndex({ index, animated, viewPosition: 0 });
      },

      scrollToEnd(animated = true) {
        flatListRef.current?.scrollToEnd({ animated });
      },

      getVisibleItems() {
        return visibleItemsRef.current;
      },

      async refresh() {
        if (onRefresh) {
          setRefreshing(true);
          await onRefresh();
          setRefreshing(false);
        }
      },
    }), [onRefresh]);

    return (
      <FlatList
        ref={flatListRef}
        refreshing={refreshing}
        onRefresh={onRefresh ? async () => {
          setRefreshing(true);
          await onRefresh();
          setRefreshing(false);
        } : undefined}
        onViewableItemsChanged={({ viewableItems }) => {
          if (viewableItems.length > 0) {
            visibleItemsRef.current = {
              first: viewableItems[0].index ?? 0,
              last: viewableItems[viewableItems.length - 1].index ?? 0,
            };
          }
        }}
        viewabilityConfig={{ itemVisiblePercentThreshold: 50 }}
        {...props}
      />
    );
  },
);
```

Consumer usage:

```typescript
function ChatScreen() {
  const listRef = useRef<ScrollableListHandle>(null);

  // Scroll to bottom when new message arrives
  useEffect(() => {
    if (newMessage) {
      listRef.current?.scrollToEnd();
    }
  }, [newMessage]);

  return (
    <View style={{ flex: 1 }}>
      <ScrollableList
        ref={listRef}
        data={messages}
        renderItem={({ item }) => <MessageBubble message={item} />}
        keyExtractor={(msg) => msg.id}
        onRefresh={loadOlderMessages}
        inverted
      />

      <View style={{ flexDirection: 'row', gap: 8, padding: 16 }}>
        <Button
          variant="ghost"
          size="sm"
          onPress={() => listRef.current?.scrollToTop()}
        >
          Jump to top
        </Button>
      </View>
    </View>
  );
}
```

### Type-Safe Ref Forwarding for Compound Components

When compound components need to expose refs:

```typescript
// Type-safe compound component refs
import { forwardRef, useRef, useImperativeHandle, createContext, useContext } from 'react';

// Define what the Form exposes imperatively
export interface FormHandle {
  submit: () => void;
  reset: () => void;
  validate: () => boolean;
  getValues: () => Record<string, unknown>;
  setFieldError: (field: string, error: string) => void;
  focusField: (field: string) => void;
}

// Internal context for field registration
interface FormContextValue {
  registerField: (name: string, ref: FieldRef) => void;
  unregisterField: (name: string) => void;
  getFieldError: (name: string) => string | undefined;
}

interface FieldRef {
  focus: () => void;
  validate: () => boolean;
  getValue: () => unknown;
  reset: () => void;
}

const FormContext = createContext<FormContextValue | null>(null);

export const Form = forwardRef<FormHandle, { children: React.ReactNode; onSubmit: (values: Record<string, unknown>) => void }>(
  function Form({ children, onSubmit }, ref) {
    const fieldsRef = useRef<Map<string, FieldRef>>(new Map());
    const [errors, setErrors] = useState<Record<string, string>>({});

    const registerField = (name: string, fieldRef: FieldRef) => {
      fieldsRef.current.set(name, fieldRef);
    };

    const unregisterField = (name: string) => {
      fieldsRef.current.delete(name);
    };

    useImperativeHandle(ref, () => ({
      submit() {
        if (this.validate()) {
          onSubmit(this.getValues());
        }
      },

      reset() {
        fieldsRef.current.forEach((field) => field.reset());
        setErrors({});
      },

      validate() {
        let isValid = true;
        const newErrors: Record<string, string> = {};
        fieldsRef.current.forEach((field, name) => {
          if (!field.validate()) {
            isValid = false;
            newErrors[name] = 'Validation failed';
          }
        });
        setErrors(newErrors);
        return isValid;
      },

      getValues() {
        const values: Record<string, unknown> = {};
        fieldsRef.current.forEach((field, name) => {
          values[name] = field.getValue();
        });
        return values;
      },

      setFieldError(field: string, error: string) {
        setErrors((prev) => ({ ...prev, [field]: error }));
      },

      focusField(field: string) {
        fieldsRef.current.get(field)?.focus();
      },
    }), [onSubmit]);

    return (
      <FormContext.Provider value={{
        registerField,
        unregisterField,
        getFieldError: (name) => errors[name],
      }}>
        {children}
      </FormContext.Provider>
    );
  },
);

// Usage:
function ProfileScreen() {
  const formRef = useRef<FormHandle>(null);

  return (
    <View>
      <Form ref={formRef} onSubmit={(values) => api.updateProfile(values)}>
        <Form.Field name="name" label="Name" required />
        <Form.Field name="email" label="Email" required />
        <Form.Field name="bio" label="Bio" multiline />
      </Form>

      <View style={{ flexDirection: 'row', gap: 12 }}>
        <Button variant="ghost" onPress={() => formRef.current?.reset()}>
          Reset
        </Button>
        <Button onPress={() => formRef.current?.submit()}>
          Save Changes
        </Button>
      </View>
    </View>
  );
}
```

---

## 10. STORYBOOK-DRIVEN DEVELOPMENT

Storybook is not just a documentation tool. Used correctly, it's a development methodology: **write the story first, then build the component.** This inverts the normal flow and produces better APIs.

### The SDD Workflow

```
┌──────────────────────────────────────────────────────────────────────┐
│  STORYBOOK-DRIVEN DEVELOPMENT WORKFLOW                                │
│                                                                       │
│  1. DEFINE THE API                                                    │
│     Write the story with ideal consumer code.                         │
│     This IS your API design phase.                                    │
│                                                                       │
│  2. CREATE THE TYPES                                                  │
│     Extract the props interface from your story code.                 │
│     The story is the test for your API.                               │
│                                                                       │
│  3. IMPLEMENT THE COMPONENT                                           │
│     Build until the story renders correctly.                          │
│     Hot reload against Storybook, not your app.                       │
│                                                                       │
│  4. ADD VARIANTS                                                      │
│     Write stories for every variant, state, and edge case.            │
│     Each story is a visual test case.                                 │
│                                                                       │
│  5. ADD INTERACTION TESTS                                             │
│     Use play functions to test click, type, keyboard nav.             │
│     These run in the browser, catching real interaction bugs.         │
│                                                                       │
│  6. VISUAL REGRESSION                                                 │
│     Chromatic snapshots every story on every PR.                      │
│     Catches unintended visual changes across all variants.            │
│                                                                       │
│  7. DESIGN REVIEW                                                     │
│     Link the Storybook deploy to the PR.                              │
│     Designers review in Storybook, not Figma.                         │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Writing Stories: The Complete Example

```typescript
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { expect, fn, userEvent, within } from '@storybook/test';
import { Button } from './Button';
import { Icon } from '../Icon';

// Meta: configures the story page
const meta: Meta<typeof Button> = {
  title: 'Atoms/Button',
  component: Button,
  tags: ['autodocs'],  // Auto-generate docs from types

  // Controls: Storybook generates interactive controls from these
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'ghost', 'danger'],
      description: 'Visual style variant',
      table: { defaultValue: { summary: 'primary' } },
    },
    size: {
      control: 'radio',
      options: ['sm', 'md', 'lg'],
      description: 'Size preset',
      table: { defaultValue: { summary: 'md' } },
    },
    loading: {
      control: 'boolean',
      description: 'Shows a spinner and disables interaction',
    },
    disabled: {
      control: 'boolean',
      description: 'Prevents interaction',
    },
    fullWidth: {
      control: 'boolean',
      description: 'Stretches to fill container width',
    },
    onPress: { action: 'pressed' }, // Log presses in the Actions panel
  },

  // Default args for all stories
  args: {
    children: 'Button',
    variant: 'primary',
    size: 'md',
    onPress: fn(), // Spy function for interaction tests
  },

  // Decorators: wrap stories in providers
  decorators: [
    (Story) => (
      <ThemeProvider>
        <View style={{ padding: 20, alignItems: 'flex-start' }}>
          <Story />
        </View>
      </ThemeProvider>
    ),
  ],
};

export default meta;
type Story = StoryObj<typeof meta>;

// ─── BASIC STORIES ─────────────────────────────────────────────────

export const Primary: Story = {
  args: { variant: 'primary', children: 'Primary Button' },
};

export const Secondary: Story = {
  args: { variant: 'secondary', children: 'Secondary Button' },
};

export const Ghost: Story = {
  args: { variant: 'ghost', children: 'Ghost Button' },
};

export const Danger: Story = {
  args: { variant: 'danger', children: 'Delete Item' },
};

// ─── SIZE VARIANTS ─────────────────────────────────────────────────

export const Small: Story = {
  args: { size: 'sm', children: 'Small' },
};

export const Large: Story = {
  args: { size: 'lg', children: 'Large Button' },
};

// ─── STATE VARIANTS ────────────────────────────────────────────────

export const Loading: Story = {
  args: { loading: true, children: 'Submitting...' },
};

export const Disabled: Story = {
  args: { disabled: true, children: 'Disabled' },
};

// ─── WITH ICON ─────────────────────────────────────────────────────

export const WithIconStart: Story = {
  args: {
    icon: <Icon name="plus" size={16} />,
    iconPosition: 'start',
    children: 'Add Item',
  },
};

export const WithIconEnd: Story = {
  args: {
    icon: <Icon name="arrow-right" size={16} />,
    iconPosition: 'end',
    children: 'Continue',
  },
};

// ─── COMPOSITION STORY ─────────────────────────────────────────────

export const AllVariants: Story = {
  render: () => (
    <View style={{ gap: 12 }}>
      {(['primary', 'secondary', 'ghost', 'danger'] as const).map((variant) => (
        <View key={variant} style={{ flexDirection: 'row', gap: 8, alignItems: 'center' }}>
          {(['sm', 'md', 'lg'] as const).map((size) => (
            <Button key={size} variant={variant} size={size}>
              {variant} {size}
            </Button>
          ))}
        </View>
      ))}
    </View>
  ),
};

// ─── INTERACTION TESTS ─────────────────────────────────────────────

export const ClickTest: Story = {
  args: { children: 'Click me' },
  play: async ({ canvasElement, args }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole('button');

    // Verify the button renders
    await expect(button).toBeInTheDocument();

    // Click and verify handler was called
    await userEvent.click(button);
    await expect(args.onPress).toHaveBeenCalledTimes(1);
  },
};

export const DisabledClickTest: Story = {
  args: { children: 'Disabled', disabled: true },
  play: async ({ canvasElement, args }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole('button');

    // Click disabled button
    await userEvent.click(button);
    // Handler should NOT have been called
    await expect(args.onPress).not.toHaveBeenCalled();
  },
};

export const LoadingClickTest: Story = {
  args: { children: 'Loading', loading: true },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole('button');

    // Verify accessibility state
    await expect(button).toHaveAttribute('aria-busy', 'true');
    await expect(button).toBeDisabled();
  },
};
```

### Storybook Configuration for React Native

```typescript
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-native';

const config: StorybookConfig = {
  stories: [
    '../src/components/**/*.stories.@(ts|tsx)',
  ],
  addons: [
    '@storybook/addon-ondevice-controls',
    '@storybook/addon-ondevice-actions',
  ],
};

export default config;
```

For web-based Storybook with React Native Web:

```typescript
// .storybook/main.ts (web)
import type { StorybookConfig } from '@storybook/react-webpack5';

const config: StorybookConfig = {
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: [
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',
  ],
  framework: '@storybook/react-webpack5',
  webpackFinal: async (config) => {
    // Alias react-native to react-native-web
    config.resolve = {
      ...config.resolve,
      alias: {
        ...config.resolve?.alias,
        'react-native$': 'react-native-web',
      },
      extensions: ['.web.tsx', '.web.ts', '.tsx', '.ts', '.web.js', '.js'],
    };
    return config;
  },
};

export default config;
```

### Visual Regression with Chromatic

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on: pull_request

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for Chromatic to find baseline

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci

      - uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          buildScriptName: build-storybook
          exitZeroOnChanges: true  # Don't fail, just flag changes
          autoAcceptChanges: main  # Auto-accept on main branch
```

Every PR gets:
1. A Storybook deploy with all component stories
2. Visual diffs for every changed component
3. A link in the PR for designer review

---

## 11. COMPONENT DOCUMENTATION

Documentation that lives separately from code gets stale. The best component documentation is generated from the code itself and augmented with human-written examples.

### Auto-Generated Props Tables

When you use `tags: ['autodocs']` in Storybook, it generates a props table from your TypeScript types. But you can improve it with JSDoc comments:

```typescript
interface ButtonProps extends Omit<PressableProps, 'children' | 'style'> {
  /**
   * Visual style variant.
   * - `primary`: High emphasis action (submit, confirm)
   * - `secondary`: Medium emphasis (cancel, secondary action)
   * - `ghost`: Low emphasis (tertiary action, links)
   * - `danger`: Destructive action (delete, remove)
   *
   * @default 'primary'
   */
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';

  /**
   * Size preset. Affects padding, font size, and minimum height.
   *
   * | Size | Min Height | Font Size |
   * |------|------------|-----------|
   * | sm   | 32px       | 13px      |
   * | md   | 40px       | 14px      |
   * | lg   | 48px       | 16px      |
   *
   * @default 'md'
   */
  size?: 'sm' | 'md' | 'lg';

  /**
   * Shows a loading spinner and disables interaction.
   * The button width is preserved to prevent layout shift.
   *
   * @default false
   */
  loading?: boolean;

  /**
   * Icon element rendered alongside the button text.
   * Use the `iconPosition` prop to control placement.
   *
   * @example
   * ```tsx
   * <Button icon={<Icon name="plus" size={16} />}>Add Item</Button>
   * ```
   */
  icon?: React.ReactElement;

  /** Button label. Must be a string for consistent typography. */
  children: string;
}
```

### Documentation Page Pattern

Structure your Storybook docs pages with a consistent format:

```typescript
// Button.mdx
import { Meta, Story, Canvas, Controls, ArgTypes } from '@storybook/blocks';
import * as ButtonStories from './Button.stories';

<Meta of={ButtonStories} />

# Button

Buttons trigger actions. Use the `variant` prop to communicate the action's importance.

## Usage Guidelines

### Do
- Use `primary` for the single most important action on a screen
- Use `secondary` for alternative actions alongside a primary button
- Use `ghost` for low-emphasis actions in toolbars and dense UI
- Always include a text label (icon-only buttons need `accessibilityLabel`)

### Don't
- Don't use more than one `primary` button in a group
- Don't use `danger` for non-destructive actions
- Don't disable buttons without explaining why (use a tooltip or helper text)

## Interactive Demo

<Canvas of={ButtonStories.Primary} />
<Controls of={ButtonStories.Primary} />

## Variants

<Canvas>
  <Story of={ButtonStories.Primary} />
  <Story of={ButtonStories.Secondary} />
  <Story of={ButtonStories.Ghost} />
  <Story of={ButtonStories.Danger} />
</Canvas>

## Sizes

<Canvas of={ButtonStories.AllVariants} />

## With Icons

<Canvas>
  <Story of={ButtonStories.WithIconStart} />
  <Story of={ButtonStories.WithIconEnd} />
</Canvas>

## States

<Canvas>
  <Story of={ButtonStories.Loading} />
  <Story of={ButtonStories.Disabled} />
</Canvas>

## Accessibility

- Renders with `accessibilityRole="button"`
- Loading state sets `aria-busy="true"` and `disabled`
- Minimum touch target is 32x32 (sm), meeting WCAG 2.5.8
- Color contrast meets WCAG AA for all variants

## Props

<ArgTypes of={ButtonStories} />
```

### Generating API Docs from TypeScript

For documentation outside Storybook, use `react-docgen-typescript` to extract props:

```typescript
// scripts/generate-docs.ts
import { parse } from 'react-docgen-typescript';
import { writeFileSync } from 'fs';
import { glob } from 'glob';

const options = {
  savePropValueAsString: true,
  shouldExtractLiteralValuesFromEnum: true,
  shouldRemoveUndefinedFromOptional: true,
  propFilter: (prop: any) => {
    // Filter out inherited HTML/RN props unless explicitly documented
    if (prop.parent) {
      return !prop.parent.fileName.includes('node_modules');
    }
    return true;
  },
};

async function generateDocs() {
  const files = await glob('src/components/**/*.tsx', {
    ignore: ['**/*.stories.tsx', '**/*.test.tsx'],
  });

  const allDocs = files.map((file) => {
    const docs = parse(file, options);
    return docs.map((doc) => ({
      name: doc.displayName,
      description: doc.description,
      props: Object.entries(doc.props).map(([name, prop]) => ({
        name,
        type: prop.type.name,
        required: prop.required,
        defaultValue: prop.defaultValue?.value,
        description: prop.description,
      })),
    }));
  }).flat();

  writeFileSync(
    'docs/component-api.json',
    JSON.stringify(allDocs, null, 2),
  );

  console.log(`Generated docs for ${allDocs.length} components`);
}

generateDocs();
```

---

## 12. VERSIONING AND BREAKING CHANGES

A component library without versioning discipline will either stagnate (afraid to change) or break consumers constantly (changing without notice). Neither is acceptable. Here's how to manage evolution.

### Semantic Versioning for Component Libraries

```
MAJOR (X.0.0): Breaking changes
  - Removing a prop
  - Changing a prop's type
  - Changing default behavior
  - Removing a component
  - Changing the rendering output in a way that breaks consumers' styling

MINOR (0.X.0): New features, backward compatible
  - Adding a new component
  - Adding a new optional prop
  - Adding a new variant to a union type
  - Expanding the API without breaking existing usage

PATCH (0.0.X): Bug fixes, backward compatible
  - Fixing a rendering bug
  - Fixing a type error
  - Improving accessibility without changing API
  - Performance improvements
```

### The Deprecation Pattern

Never remove a prop without a deprecation period. Here's the process:

```typescript
// Step 1: Deprecate in types (MINOR version bump)
interface ButtonProps {
  /**
   * @deprecated Use `variant` instead. Will be removed in v4.0.
   */
  appearance?: 'primary' | 'secondary' | 'ghost' | 'danger';

  /** Visual style variant. */
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger';
}

// Step 2: Warn at runtime during the deprecation period
function Button({ appearance, variant: variantProp, ...rest }: ButtonProps) {
  if (appearance !== undefined) {
    if (__DEV__) {
      console.warn(
        '[Button] The `appearance` prop is deprecated. ' +
        'Use `variant` instead. ' +
        '`appearance` will be removed in v4.0.\n\n' +
        'Migration:\n' +
        '  Before: <Button appearance="primary">\n' +
        '  After:  <Button variant="primary">'
      );
    }
  }

  // Support both during transition
  const variant = variantProp ?? appearance ?? 'primary';

  // ...
}

// Step 3: Remove in MAJOR version bump
// v4.0.0: Remove `appearance` prop entirely
```

### Migration Codemods

For breaking changes that affect many files, write codemods with `jscodeshift`:

```typescript
// codemods/rename-appearance-to-variant.ts
import type { API, FileInfo, Options } from 'jscodeshift';

export default function transformer(file: FileInfo, api: API, options: Options) {
  const j = api.jscodeshift;
  const root = j(file.source);

  // Find all <Button appearance="..."> and rename to variant
  root
    .findJSXElements('Button')
    .find(j.JSXAttribute, { name: { name: 'appearance' } })
    .forEach((path) => {
      // Rename the attribute
      path.node.name = j.jsxIdentifier('variant');
    });

  return root.toSource({ quote: 'single' });
}

// Run it:
// npx jscodeshift --transform codemods/rename-appearance-to-variant.ts src/
```

### Changelog Discipline

Every PR that touches the component library must include a changeset:

```yaml
# .changeset/button-variant-rename.md
---
"@acme/ui": minor
---

Added `variant` prop to Button. Deprecated `appearance` prop (will be removed in v4.0). See migration guide.
```

Use `@changesets/cli` to automate versioning and changelog generation:

```bash
# Developer workflow:
npx changeset           # Create a changeset describing the change
git add .changeset/     # Commit the changeset with the code

# CI workflow (on merge to main):
npx changeset version   # Bump versions based on changesets
npx changeset publish   # Publish to npm
```

### Supporting Multiple Versions

When a major version bump is unavoidable, give consumers time to migrate:

```
┌──────────────────────────────────────────────────────────────────────┐
│  MAJOR VERSION MIGRATION TIMELINE                                     │
│                                                                       │
│  Week 0:  Release v4.0.0-beta.1                                      │
│           Announce in #design-system channel                          │
│           Publish migration guide                                     │
│           Publish codemod                                             │
│                                                                       │
│  Week 2:  Release v4.0.0-beta.2 (address feedback)                   │
│           Run codemod on pilot team's codebase                        │
│                                                                       │
│  Week 4:  Release v4.0.0                                              │
│           v3.x enters maintenance mode (security fixes only)          │
│           Begin team-by-team migration with codemod                   │
│                                                                       │
│  Week 8:  All teams migrated                                          │
│           v3.x deprecated                                             │
│                                                                       │
│  Week 16: v3.x end of life                                            │
│           Remove from monorepo                                        │
│                                                                       │
│  KEY RULE: v3.x and v4.x must coexist in the same app during         │
│  migration. Use package aliases if needed:                            │
│                                                                       │
│  "dependencies": {                                                    │
│    "@acme/ui": "^4.0.0",                                             │
│    "@acme/ui-v3": "npm:@acme/ui@^3.0.0"                             │
│  }                                                                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 13. THE COMPLETE COMPONENT: BUILDING A PRODUCTION-GRADE SELECT

Let's pull every pattern together. We'll build a `Select` component from scratch that includes: a headless logic hook, a compound component API, keyboard navigation, accessibility, animation, a Storybook story, and tests.

### Layer 1: The Headless Hook

```typescript
// hooks/useSelect.ts
import { useState, useRef, useCallback, useMemo, useEffect } from 'react';
import { useControllableState } from './useControllableState';

// ─── Types ──────────────────────────────────────────────────────────

export interface SelectOption {
  value: string;
  label: string;
  disabled?: boolean;
}

export interface UseSelectOptions {
  options: SelectOption[];
  value?: string;
  defaultValue?: string;
  onValueChange?: (value: string) => void;
  open?: boolean;
  defaultOpen?: boolean;
  onOpenChange?: (open: boolean) => void;
  searchable?: boolean;
}

export interface UseSelectReturn {
  // State
  isOpen: boolean;
  selectedOption: SelectOption | undefined;
  highlightedIndex: number;
  searchQuery: string;
  filteredOptions: SelectOption[];

  // Actions
  open: () => void;
  close: () => void;
  toggle: () => void;
  selectOption: (value: string) => void;
  highlightOption: (index: number) => void;
  search: (query: string) => void;
  clearSearch: () => void;

  // Keyboard handler
  handleKeyDown: (event: { key: string; preventDefault: () => void }) => void;

  // Prop getters
  getTriggerProps: () => {
    role: 'combobox';
    'aria-expanded': boolean;
    'aria-haspopup': 'listbox';
    'aria-activedescendant': string | undefined;
    'aria-controls': string;
    onPress: () => void;
  };

  getContentProps: () => {
    role: 'listbox';
    nativeID: string;
    'aria-label': string;
  };

  getOptionProps: (option: SelectOption, index: number) => {
    role: 'option';
    nativeID: string;
    'aria-selected': boolean;
    'aria-disabled': boolean | undefined;
    onPress: () => void;
  };
}

// ─── Implementation ─────────────────────────────────────────────────

export function useSelect(opts: UseSelectOptions): UseSelectReturn {
  const {
    options,
    value: controlledValue,
    defaultValue,
    onValueChange,
    open: controlledOpen,
    defaultOpen = false,
    onOpenChange,
    searchable = false,
  } = opts;

  // Controllable state for value and open
  const [value, setValue] = useControllableState({
    value: controlledValue,
    defaultValue: defaultValue ?? '',
    onChange: onValueChange,
  });

  const [isOpen, setIsOpen] = useControllableState({
    value: controlledOpen,
    defaultValue: defaultOpen,
    onChange: onOpenChange,
  });

  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const [searchQuery, setSearchQuery] = useState('');

  // Unique IDs for a11y
  const idRef = useRef(`select-${Math.random().toString(36).slice(2, 8)}`);
  const listboxId = `${idRef.current}-listbox`;

  // Derived state
  const filteredOptions = useMemo(() => {
    if (!searchable || !searchQuery) return options;
    return options.filter((opt) =>
      opt.label.toLowerCase().includes(searchQuery.toLowerCase()),
    );
  }, [options, searchQuery, searchable]);

  const selectedOption = useMemo(
    () => options.find((opt) => opt.value === value),
    [options, value],
  );

  // Reset highlight when filtered list changes
  useEffect(() => {
    setHighlightedIndex(filteredOptions.length > 0 ? 0 : -1);
  }, [filteredOptions.length]);

  // ─── Actions ───────────────────────────────────────────────────

  const open = useCallback(() => {
    setIsOpen(true);
    // Highlight the selected item, or the first item
    const selectedIndex = filteredOptions.findIndex((opt) => opt.value === value);
    setHighlightedIndex(selectedIndex >= 0 ? selectedIndex : 0);
  }, [setIsOpen, filteredOptions, value]);

  const close = useCallback(() => {
    setIsOpen(false);
    setSearchQuery('');
    setHighlightedIndex(-1);
  }, [setIsOpen]);

  const toggle = useCallback(() => {
    if (isOpen) close();
    else open();
  }, [isOpen, open, close]);

  const selectOption = useCallback(
    (optionValue: string) => {
      const option = options.find((o) => o.value === optionValue);
      if (option && !option.disabled) {
        setValue(optionValue);
        close();
      }
    },
    [options, setValue, close],
  );

  const highlightOption = useCallback(
    (index: number) => {
      const clampedIndex = Math.max(0, Math.min(index, filteredOptions.length - 1));
      // Skip disabled options
      const option = filteredOptions[clampedIndex];
      if (option && !option.disabled) {
        setHighlightedIndex(clampedIndex);
      }
    },
    [filteredOptions],
  );

  const search = useCallback((query: string) => {
    setSearchQuery(query);
  }, []);

  const clearSearch = useCallback(() => {
    setSearchQuery('');
  }, []);

  // ─── Keyboard Handler ──────────────────────────────────────────

  const handleKeyDown = useCallback(
    (event: { key: string; preventDefault: () => void }) => {
      switch (event.key) {
        case 'ArrowDown': {
          event.preventDefault();
          if (!isOpen) {
            open();
          } else {
            // Move highlight down, skip disabled
            let nextIndex = highlightedIndex + 1;
            while (nextIndex < filteredOptions.length && filteredOptions[nextIndex]?.disabled) {
              nextIndex++;
            }
            if (nextIndex < filteredOptions.length) {
              setHighlightedIndex(nextIndex);
            }
          }
          break;
        }

        case 'ArrowUp': {
          event.preventDefault();
          if (isOpen) {
            let prevIndex = highlightedIndex - 1;
            while (prevIndex >= 0 && filteredOptions[prevIndex]?.disabled) {
              prevIndex--;
            }
            if (prevIndex >= 0) {
              setHighlightedIndex(prevIndex);
            }
          }
          break;
        }

        case 'Enter':
        case ' ': {
          event.preventDefault();
          if (!isOpen) {
            open();
          } else if (highlightedIndex >= 0) {
            const option = filteredOptions[highlightedIndex];
            if (option && !option.disabled) {
              selectOption(option.value);
            }
          }
          break;
        }

        case 'Escape': {
          event.preventDefault();
          close();
          break;
        }

        case 'Home': {
          event.preventDefault();
          if (isOpen) setHighlightedIndex(0);
          break;
        }

        case 'End': {
          event.preventDefault();
          if (isOpen) setHighlightedIndex(filteredOptions.length - 1);
          break;
        }

        default: {
          // Type-ahead: jump to first option matching typed character
          if (event.key.length === 1 && !searchable) {
            const char = event.key.toLowerCase();
            const startIndex = highlightedIndex + 1;
            const matchIndex = filteredOptions.findIndex(
              (opt, i) => i >= startIndex && opt.label.toLowerCase().startsWith(char),
            );
            if (matchIndex >= 0) {
              setHighlightedIndex(matchIndex);
            } else {
              // Wrap around
              const wrapIndex = filteredOptions.findIndex(
                (opt) => opt.label.toLowerCase().startsWith(char),
              );
              if (wrapIndex >= 0) setHighlightedIndex(wrapIndex);
            }
          }
          break;
        }
      }
    },
    [isOpen, open, close, selectOption, highlightedIndex, filteredOptions, searchable],
  );

  // ─── Prop Getters ──────────────────────────────────────────────

  const getTriggerProps = useCallback(
    () => ({
      role: 'combobox' as const,
      'aria-expanded': isOpen,
      'aria-haspopup': 'listbox' as const,
      'aria-activedescendant':
        highlightedIndex >= 0
          ? `${listboxId}-option-${highlightedIndex}`
          : undefined,
      'aria-controls': listboxId,
      onPress: toggle,
    }),
    [isOpen, highlightedIndex, listboxId, toggle],
  );

  const getContentProps = useCallback(
    () => ({
      role: 'listbox' as const,
      nativeID: listboxId,
      'aria-label': 'Options',
    }),
    [listboxId],
  );

  const getOptionProps = useCallback(
    (option: SelectOption, index: number) => ({
      role: 'option' as const,
      nativeID: `${listboxId}-option-${index}`,
      'aria-selected': option.value === value,
      'aria-disabled': option.disabled || undefined,
      onPress: () => selectOption(option.value),
    }),
    [listboxId, value, selectOption],
  );

  return {
    isOpen,
    selectedOption,
    highlightedIndex,
    searchQuery,
    filteredOptions,
    open,
    close,
    toggle,
    selectOption,
    highlightOption,
    search,
    clearSearch,
    handleKeyDown,
    getTriggerProps,
    getContentProps,
    getOptionProps,
  };
}
```

### Layer 2: The Compound Component

```typescript
// components/Select/SelectContext.ts
import { createContext, useContext } from 'react';
import type { UseSelectReturn, SelectOption } from '../../hooks/useSelect';

interface SelectContextValue extends UseSelectReturn {
  size: 'sm' | 'md' | 'lg';
}

const SelectContext = createContext<SelectContextValue | null>(null);

export function useSelectCtx(): SelectContextValue {
  const ctx = useContext(SelectContext);
  if (!ctx) {
    throw new Error(
      'Select compound components must be rendered inside <Select>. ' +
      'Did you forget to wrap your Select.Trigger, Select.Content, etc. in a <Select>?'
    );
  }
  return ctx;
}

export { SelectContext };
export type { SelectContextValue };
```

```typescript
// components/Select/Select.tsx
import { useMemo } from 'react';
import { View } from 'react-native';
import { useSelect, UseSelectOptions, SelectOption } from '../../hooks/useSelect';
import { SelectContext, SelectContextValue } from './SelectContext';

// Sub-component imports
import { SelectTrigger } from './SelectTrigger';
import { SelectContent } from './SelectContent';
import { SelectValue } from './SelectValue';
import { SelectItem } from './SelectItem';
import { SelectItemText } from './SelectItemText';
import { SelectIcon } from './SelectIcon';
import { SelectGroup } from './SelectGroup';
import { SelectLabel } from './SelectLabel';
import { SelectSeparator } from './SelectSeparator';
import { SelectSearch } from './SelectSearch';
import { SelectEmpty } from './SelectEmpty';

// ─── Root Props ──────────────────────────────────────────────────

interface SelectRootProps extends UseSelectOptions {
  size?: 'sm' | 'md' | 'lg';
  children: React.ReactNode;
}

// ─── Root Component ──────────────────────────────────────────────

function SelectRoot({
  size = 'md',
  children,
  ...selectOptions
}: SelectRootProps) {
  const select = useSelect(selectOptions);

  const contextValue = useMemo<SelectContextValue>(
    () => ({ ...select, size }),
    [select, size],
  );

  return (
    <SelectContext.Provider value={contextValue}>
      <View style={{ position: 'relative' }}>
        {children}
      </View>
    </SelectContext.Provider>
  );
}

// ─── Namespace Assembly ──────────────────────────────────────────

export const Select = Object.assign(SelectRoot, {
  Trigger: SelectTrigger,
  Content: SelectContent,
  Value: SelectValue,
  Item: SelectItem,
  ItemText: SelectItemText,
  Icon: SelectIcon,
  Group: SelectGroup,
  Label: SelectLabel,
  Separator: SelectSeparator,
  Search: SelectSearch,
  Empty: SelectEmpty,
});

export type { SelectRootProps as SelectProps };
```

```typescript
// components/Select/SelectTrigger.tsx
import { forwardRef } from 'react';
import { Pressable, View, ViewStyle } from 'react-native';
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectTriggerProps {
  children: React.ReactNode;
  style?: ViewStyle;
}

const sizePadding = {
  sm: { paddingHorizontal: 8, paddingVertical: 4, minHeight: 32 },
  md: { paddingHorizontal: 12, paddingVertical: 8, minHeight: 40 },
  lg: { paddingHorizontal: 16, paddingVertical: 12, minHeight: 48 },
} as const;

export const SelectTrigger = forwardRef<View, SelectTriggerProps>(
  function SelectTrigger({ children, style }, ref) {
    const { isOpen, size, getTriggerProps, handleKeyDown } = useSelectCtx();
    const theme = useTheme();
    const triggerProps = getTriggerProps();

    return (
      <Pressable
        ref={ref}
        {...triggerProps}
        onKeyPress={(e) => handleKeyDown({ key: e.nativeEvent.key, preventDefault: () => {} })}
        style={({ pressed }) => [
          {
            flexDirection: 'row',
            alignItems: 'center',
            justifyContent: 'space-between',
            gap: theme.spacing.sm,
            borderWidth: 1,
            borderColor: isOpen ? theme.color.borderFocus : theme.color.border,
            borderRadius: theme.borderRadius.md,
            backgroundColor: theme.color.surface,
            ...sizePadding[size],
          },
          pressed && { backgroundColor: theme.color.surfaceHover },
          style,
        ]}
      >
        {children}
      </Pressable>
    );
  },
);
```

```typescript
// components/Select/SelectContent.tsx
import { useEffect, useRef } from 'react';
import { View, Modal, Pressable, ScrollView, Animated, ViewStyle, Dimensions } from 'react-native';
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectContentProps {
  children: React.ReactNode;
  maxHeight?: number;
  style?: ViewStyle;
}

export function SelectContent({ children, maxHeight = 256, style }: SelectContentProps) {
  const { isOpen, close, getContentProps } = useSelectCtx();
  const theme = useTheme();
  const fadeAnim = useRef(new Animated.Value(0)).current;
  const scaleAnim = useRef(new Animated.Value(0.95)).current;

  useEffect(() => {
    if (isOpen) {
      Animated.parallel([
        Animated.timing(fadeAnim, {
          toValue: 1,
          duration: 150,
          useNativeDriver: true,
        }),
        Animated.spring(scaleAnim, {
          toValue: 1,
          tension: 300,
          friction: 20,
          useNativeDriver: true,
        }),
      ]).start();
    } else {
      Animated.timing(fadeAnim, {
        toValue: 0,
        duration: 100,
        useNativeDriver: true,
      }).start();
      scaleAnim.setValue(0.95);
    }
  }, [isOpen, fadeAnim, scaleAnim]);

  if (!isOpen) return null;

  const contentProps = getContentProps();

  return (
    <Modal transparent visible={isOpen} onRequestClose={close}>
      {/* Backdrop */}
      <Pressable
        style={{ flex: 1 }}
        onPress={close}
        accessibilityLabel="Close select"
      >
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <Pressable onPress={(e) => e.stopPropagation()}>
            <Animated.View
              style={[
                {
                  backgroundColor: theme.color.surface,
                  borderRadius: theme.borderRadius.md,
                  borderWidth: 1,
                  borderColor: theme.color.border,
                  maxHeight,
                  minWidth: 200,
                  shadowColor: '#000',
                  shadowOffset: { width: 0, height: 4 },
                  shadowOpacity: 0.15,
                  shadowRadius: 12,
                  elevation: 8,
                  overflow: 'hidden',
                  opacity: fadeAnim,
                  transform: [{ scale: scaleAnim }],
                },
                style,
              ]}
              {...contentProps}
            >
              <ScrollView bounces={false} keyboardShouldPersistTaps="handled">
                {children}
              </ScrollView>
            </Animated.View>
          </Pressable>
        </View>
      </Pressable>
    </Modal>
  );
}
```

```typescript
// components/Select/SelectItem.tsx
import { Pressable, View, ViewStyle } from 'react-native';
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectItemProps {
  value: string;
  disabled?: boolean;
  children: React.ReactNode;
  style?: ViewStyle;
}

export function SelectItem({ value, disabled = false, children, style }: SelectItemProps) {
  const ctx = useSelectCtx();
  const theme = useTheme();

  // Find this option's index in the filtered list
  const index = ctx.filteredOptions.findIndex((opt) => opt.value === value);
  const isHighlighted = index === ctx.highlightedIndex;
  const isSelected = ctx.selectedOption?.value === value;

  const optionProps = ctx.getOptionProps(
    { value, label: '', disabled },
    index,
  );

  return (
    <Pressable
      {...optionProps}
      disabled={disabled}
      style={({ pressed }) => [
        {
          flexDirection: 'row',
          alignItems: 'center',
          gap: theme.spacing.sm,
          paddingHorizontal: theme.spacing.md,
          paddingVertical: theme.spacing.sm,
          opacity: disabled ? 0.4 : 1,
          backgroundColor: isHighlighted
            ? theme.color.surfaceHover
            : isSelected
              ? theme.color.backgroundSecondary
              : 'transparent',
        },
        pressed && !disabled && { backgroundColor: theme.color.surfaceHover },
        style,
      ]}
    >
      {children}

      {isSelected && (
        <View style={{ marginLeft: 'auto' }}>
          <CheckIcon size={16} color={theme.color.primary} />
        </View>
      )}
    </Pressable>
  );
}

// components/Select/SelectItemText.tsx
import { Text, TextProps } from '../../atoms/Text';

interface SelectItemTextProps extends TextProps {
  children: string;
}

export function SelectItemText({ children, ...rest }: SelectItemTextProps) {
  return <Text variant="body" {...rest}>{children}</Text>;
}

// components/Select/SelectValue.tsx
import { Text } from '../../atoms/Text';
import { useSelectCtx } from './SelectContext';

interface SelectValueProps {
  placeholder?: string;
}

export function SelectValue({ placeholder = 'Select...' }: SelectValueProps) {
  const { selectedOption } = useSelectCtx();

  return (
    <Text
      variant="body"
      color={selectedOption ? 'primary' : 'tertiary'}
      numberOfLines={1}
      style={{ flex: 1 }}
    >
      {selectedOption?.label ?? placeholder}
    </Text>
  );
}

// components/Select/SelectIcon.tsx
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

export function SelectIcon() {
  const { isOpen } = useSelectCtx();
  const theme = useTheme();

  return (
    <ChevronIcon
      direction={isOpen ? 'up' : 'down'}
      size={16}
      color={theme.color.textTertiary}
    />
  );
}

// components/Select/SelectGroup.tsx
import { View, ViewStyle } from 'react-native';

interface SelectGroupProps {
  children: React.ReactNode;
  style?: ViewStyle;
}

export function SelectGroup({ children, style }: SelectGroupProps) {
  return <View style={style}>{children}</View>;
}

// components/Select/SelectLabel.tsx
import { View } from 'react-native';
import { Text } from '../../atoms/Text';
import { useTheme } from '../../theme';

interface SelectLabelProps {
  children: string;
}

export function SelectLabel({ children }: SelectLabelProps) {
  const theme = useTheme();

  return (
    <View style={{
      paddingHorizontal: theme.spacing.md,
      paddingVertical: theme.spacing.xs,
      paddingTop: theme.spacing.sm,
    }}>
      <Text variant="caption" color="tertiary" weight="semibold">
        {children}
      </Text>
    </View>
  );
}

// components/Select/SelectSeparator.tsx
import { View } from 'react-native';
import { useTheme } from '../../theme';

export function SelectSeparator() {
  const theme = useTheme();

  return (
    <View style={{
      height: 1,
      backgroundColor: theme.color.border,
      marginVertical: theme.spacing.xs,
    }} />
  );
}

// components/Select/SelectSearch.tsx
import { TextInput, View } from 'react-native';
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectSearchProps {
  placeholder?: string;
}

export function SelectSearch({ placeholder = 'Search...' }: SelectSearchProps) {
  const { searchQuery, search } = useSelectCtx();
  const theme = useTheme();

  return (
    <View style={{
      paddingHorizontal: theme.spacing.sm,
      paddingVertical: theme.spacing.xs,
      borderBottomWidth: 1,
      borderBottomColor: theme.color.border,
    }}>
      <TextInput
        value={searchQuery}
        onChangeText={search}
        placeholder={placeholder}
        autoFocus
        style={{
          paddingHorizontal: theme.spacing.sm,
          paddingVertical: theme.spacing.xs,
          fontSize: theme.fontSize.sm,
          color: theme.color.text,
        }}
        placeholderTextColor={theme.color.textTertiary}
      />
    </View>
  );
}

// components/Select/SelectEmpty.tsx
import { View } from 'react-native';
import { Text } from '../../atoms/Text';
import { useSelectCtx } from './SelectContext';
import { useTheme } from '../../theme';

interface SelectEmptyProps {
  children: React.ReactNode;
}

export function SelectEmpty({ children }: SelectEmptyProps) {
  const { filteredOptions } = useSelectCtx();
  const theme = useTheme();

  if (filteredOptions.length > 0) return null;

  return (
    <View style={{
      padding: theme.spacing.lg,
      alignItems: 'center',
    }}>
      {typeof children === 'string' ? (
        <Text variant="body" color="tertiary">{children}</Text>
      ) : (
        children
      )}
    </View>
  );
}
```

### Layer 3: The Storybook Story

```typescript
// components/Select/Select.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { expect, fn, userEvent, within } from '@storybook/test';
import { useState } from 'react';
import { View, Text } from 'react-native';
import { Select } from './';
import { ThemeProvider } from '../../theme';

const meta: Meta<typeof Select> = {
  title: 'Components/Select',
  component: Select,
  tags: ['autodocs'],
  decorators: [
    (Story) => (
      <ThemeProvider>
        <View style={{ padding: 40, minHeight: 400, alignItems: 'flex-start' }}>
          <Story />
        </View>
      </ThemeProvider>
    ),
  ],
};

export default meta;
type Story = StoryObj<typeof meta>;

// ─── Basic Usage ────────────────────────────────────────────────

export const Default: Story = {
  render: () => (
    <Select
      options={[
        { value: 'apple', label: 'Apple' },
        { value: 'banana', label: 'Banana' },
        { value: 'cherry', label: 'Cherry' },
        { value: 'dragonfruit', label: 'Dragon Fruit' },
      ]}
      defaultValue="banana"
    >
      <Select.Trigger>
        <Select.Value placeholder="Choose a fruit..." />
        <Select.Icon />
      </Select.Trigger>

      <Select.Content>
        <Select.Item value="apple">
          <Select.ItemText>Apple</Select.ItemText>
        </Select.Item>
        <Select.Item value="banana">
          <Select.ItemText>Banana</Select.ItemText>
        </Select.Item>
        <Select.Item value="cherry">
          <Select.ItemText>Cherry</Select.ItemText>
        </Select.Item>
        <Select.Item value="dragonfruit">
          <Select.ItemText>Dragon Fruit</Select.ItemText>
        </Select.Item>
      </Select.Content>
    </Select>
  ),
};

// ─── Controlled ─────────────────────────────────────────────────

export const Controlled: Story = {
  render: () => {
    const [value, setValue] = useState('cherry');

    return (
      <View style={{ gap: 16 }}>
        <Text>Selected: {value}</Text>

        <Select
          options={[
            { value: 'apple', label: 'Apple' },
            { value: 'banana', label: 'Banana' },
            { value: 'cherry', label: 'Cherry' },
          ]}
          value={value}
          onValueChange={setValue}
        >
          <Select.Trigger>
            <Select.Value />
            <Select.Icon />
          </Select.Trigger>
          <Select.Content>
            <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
            <Select.Item value="banana"><Select.ItemText>Banana</Select.ItemText></Select.Item>
            <Select.Item value="cherry"><Select.ItemText>Cherry</Select.ItemText></Select.Item>
          </Select.Content>
        </Select>
      </View>
    );
  },
};

// ─── With Groups ────────────────────────────────────────────────

export const WithGroups: Story = {
  render: () => (
    <Select
      options={[
        { value: 'apple', label: 'Apple' },
        { value: 'banana', label: 'Banana' },
        { value: 'cherry', label: 'Cherry' },
        { value: 'carrot', label: 'Carrot' },
        { value: 'broccoli', label: 'Broccoli' },
        { value: 'spinach', label: 'Spinach' },
      ]}
    >
      <Select.Trigger>
        <Select.Value placeholder="Choose food..." />
        <Select.Icon />
      </Select.Trigger>

      <Select.Content>
        <Select.Group>
          <Select.Label>Fruits</Select.Label>
          <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
          <Select.Item value="banana"><Select.ItemText>Banana</Select.ItemText></Select.Item>
          <Select.Item value="cherry"><Select.ItemText>Cherry</Select.ItemText></Select.Item>
        </Select.Group>

        <Select.Separator />

        <Select.Group>
          <Select.Label>Vegetables</Select.Label>
          <Select.Item value="carrot"><Select.ItemText>Carrot</Select.ItemText></Select.Item>
          <Select.Item value="broccoli"><Select.ItemText>Broccoli</Select.ItemText></Select.Item>
          <Select.Item value="spinach"><Select.ItemText>Spinach</Select.ItemText></Select.Item>
        </Select.Group>
      </Select.Content>
    </Select>
  ),
};

// ─── With Disabled Items ────────────────────────────────────────

export const WithDisabledItems: Story = {
  render: () => (
    <Select
      options={[
        { value: 'apple', label: 'Apple' },
        { value: 'banana', label: 'Banana', disabled: true },
        { value: 'cherry', label: 'Cherry' },
      ]}
    >
      <Select.Trigger>
        <Select.Value placeholder="Choose a fruit..." />
        <Select.Icon />
      </Select.Trigger>

      <Select.Content>
        <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
        <Select.Item value="banana" disabled><Select.ItemText>Banana (sold out)</Select.ItemText></Select.Item>
        <Select.Item value="cherry"><Select.ItemText>Cherry</Select.ItemText></Select.Item>
      </Select.Content>
    </Select>
  ),
};

// ─── Searchable ─────────────────────────────────────────────────

export const Searchable: Story = {
  render: () => (
    <Select
      options={Array.from({ length: 50 }, (_, i) => ({
        value: `item-${i}`,
        label: `Option ${i + 1}`,
      }))}
      searchable
    >
      <Select.Trigger>
        <Select.Value placeholder="Search options..." />
        <Select.Icon />
      </Select.Trigger>

      <Select.Content maxHeight={300}>
        <Select.Search placeholder="Type to filter..." />

        {Array.from({ length: 50 }, (_, i) => (
          <Select.Item key={i} value={`item-${i}`}>
            <Select.ItemText>{`Option ${i + 1}`}</Select.ItemText>
          </Select.Item>
        ))}

        <Select.Empty>No matching options</Select.Empty>
      </Select.Content>
    </Select>
  ),
};

// ─── Size Variants ──────────────────────────────────────────────

export const Sizes: Story = {
  render: () => (
    <View style={{ gap: 16 }}>
      {(['sm', 'md', 'lg'] as const).map((size) => (
        <Select
          key={size}
          size={size}
          options={[
            { value: 'apple', label: 'Apple' },
            { value: 'banana', label: 'Banana' },
          ]}
          defaultValue="apple"
        >
          <Select.Trigger>
            <Select.Value />
            <Select.Icon />
          </Select.Trigger>
          <Select.Content>
            <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
            <Select.Item value="banana"><Select.ItemText>Banana</Select.ItemText></Select.Item>
          </Select.Content>
        </Select>
      ))}
    </View>
  ),
};
```

### Layer 4: Tests

```typescript
// components/Select/__tests__/Select.test.tsx
import { render, fireEvent, screen, waitFor } from '@testing-library/react-native';
import { Select } from '../';
import { ThemeProvider } from '../../../theme';

function renderSelect(props = {}) {
  const defaultOptions = [
    { value: 'apple', label: 'Apple' },
    { value: 'banana', label: 'Banana' },
    { value: 'cherry', label: 'Cherry' },
  ];

  return render(
    <ThemeProvider>
      <Select options={defaultOptions} {...props}>
        <Select.Trigger>
          <Select.Value placeholder="Pick a fruit" />
          <Select.Icon />
        </Select.Trigger>
        <Select.Content>
          <Select.Item value="apple">
            <Select.ItemText>Apple</Select.ItemText>
          </Select.Item>
          <Select.Item value="banana">
            <Select.ItemText>Banana</Select.ItemText>
          </Select.Item>
          <Select.Item value="cherry">
            <Select.ItemText>Cherry</Select.ItemText>
          </Select.Item>
        </Select.Content>
      </Select>
    </ThemeProvider>,
  );
}

describe('Select', () => {
  // ─── Rendering ───────────────────────────────────────────────

  it('renders the trigger with placeholder', () => {
    renderSelect();
    expect(screen.getByText('Pick a fruit')).toBeTruthy();
  });

  it('renders the trigger with selected value', () => {
    renderSelect({ defaultValue: 'banana' });
    expect(screen.getByText('Banana')).toBeTruthy();
  });

  it('does not render content when closed', () => {
    renderSelect();
    expect(screen.queryByText('Apple')).toBeNull();
  });

  // ─── Opening and Closing ─────────────────────────────────────

  it('opens on trigger press', async () => {
    renderSelect();
    fireEvent.press(screen.getByText('Pick a fruit'));
    await waitFor(() => {
      expect(screen.getByText('Apple')).toBeTruthy();
      expect(screen.getByText('Banana')).toBeTruthy();
      expect(screen.getByText('Cherry')).toBeTruthy();
    });
  });

  it('closes when an item is selected', async () => {
    renderSelect();
    fireEvent.press(screen.getByText('Pick a fruit'));
    await waitFor(() => expect(screen.getByText('Apple')).toBeTruthy());

    fireEvent.press(screen.getByText('Apple'));
    await waitFor(() => {
      expect(screen.getByText('Apple')).toBeTruthy(); // Now shows as selected value
      expect(screen.queryByText('Banana')).toBeNull(); // Content closed
    });
  });

  // ─── Selection ───────────────────────────────────────────────

  it('calls onValueChange when an item is selected', async () => {
    const onValueChange = jest.fn();
    renderSelect({ onValueChange });

    fireEvent.press(screen.getByText('Pick a fruit'));
    await waitFor(() => expect(screen.getByText('Cherry')).toBeTruthy());

    fireEvent.press(screen.getByText('Cherry'));
    expect(onValueChange).toHaveBeenCalledWith('cherry');
  });

  it('does not select disabled items', async () => {
    const onValueChange = jest.fn();

    render(
      <ThemeProvider>
        <Select
          options={[
            { value: 'apple', label: 'Apple' },
            { value: 'banana', label: 'Banana', disabled: true },
          ]}
          onValueChange={onValueChange}
        >
          <Select.Trigger>
            <Select.Value placeholder="Pick" />
            <Select.Icon />
          </Select.Trigger>
          <Select.Content>
            <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
            <Select.Item value="banana" disabled>
              <Select.ItemText>Banana</Select.ItemText>
            </Select.Item>
          </Select.Content>
        </Select>
      </ThemeProvider>,
    );

    fireEvent.press(screen.getByText('Pick'));
    await waitFor(() => expect(screen.getByText('Banana')).toBeTruthy());

    fireEvent.press(screen.getByText('Banana'));
    expect(onValueChange).not.toHaveBeenCalled();
  });

  // ─── Controlled Mode ─────────────────────────────────────────

  it('works in controlled mode', () => {
    const { rerender } = renderSelect({ value: 'apple' });
    expect(screen.getByText('Apple')).toBeTruthy();

    rerender(
      <ThemeProvider>
        <Select
          options={[
            { value: 'apple', label: 'Apple' },
            { value: 'banana', label: 'Banana' },
            { value: 'cherry', label: 'Cherry' },
          ]}
          value="cherry"
        >
          <Select.Trigger>
            <Select.Value placeholder="Pick a fruit" />
            <Select.Icon />
          </Select.Trigger>
          <Select.Content>
            <Select.Item value="apple"><Select.ItemText>Apple</Select.ItemText></Select.Item>
            <Select.Item value="banana"><Select.ItemText>Banana</Select.ItemText></Select.Item>
            <Select.Item value="cherry"><Select.ItemText>Cherry</Select.ItemText></Select.Item>
          </Select.Content>
        </Select>
      </ThemeProvider>,
    );

    expect(screen.getByText('Cherry')).toBeTruthy();
  });

  // ─── Accessibility ───────────────────────────────────────────

  it('has correct accessibility attributes on trigger', () => {
    renderSelect();
    const trigger = screen.getByRole('combobox');
    expect(trigger).toBeTruthy();
    expect(trigger.props.accessibilityState?.expanded).toBe(false);
  });

  it('sets aria-expanded when open', async () => {
    renderSelect();
    const trigger = screen.getByRole('combobox');

    fireEvent.press(trigger);
    await waitFor(() => {
      expect(trigger.props.accessibilityState?.expanded).toBe(true);
    });
  });

  // ─── Context Error ───────────────────────────────────────────

  it('throws when sub-components are used outside Select', () => {
    // Suppress console.error for expected error
    const spy = jest.spyOn(console, 'error').mockImplementation(() => {});

    expect(() => {
      render(<Select.Trigger><Select.Value /></Select.Trigger>);
    }).toThrow('Select compound components must be rendered inside <Select>');

    spy.mockRestore();
  });
});
```

### The Architecture Recap

```
┌──────────────────────────────────────────────────────────────────────┐
│  THE COMPLETE COMPONENT: ARCHITECTURE LAYERS                          │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ STORYBOOK STORIES + TESTS                                    │    │
│  │ Visual documentation, interaction tests, visual regression   │    │
│  └─────────────────────────┬────────────────────────────────────┘    │
│                             │                                         │
│  ┌──────────────────────────┴───────────────────────────────────┐    │
│  │ COMPOUND COMPONENT (Select.Trigger, Select.Item, etc.)       │    │
│  │ React Context, namespace pattern, forwardRef                 │    │
│  │ Consumer-facing API: composition, layout control             │    │
│  └─────────────────────────┬────────────────────────────────────┘    │
│                             │                                         │
│  ┌──────────────────────────┴───────────────────────────────────┐    │
│  │ HEADLESS HOOK (useSelect)                                    │    │
│  │ State, keyboard nav, a11y attributes, prop getters           │    │
│  │ Zero rendering. Reusable across any UI framework.            │    │
│  └─────────────────────────┬────────────────────────────────────┘    │
│                             │                                         │
│  ┌──────────────────────────┴───────────────────────────────────┐    │
│  │ UTILITY HOOKS (useControllableState, useId)                  │    │
│  │ Controlled/uncontrolled, unique IDs, keyboard handling       │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                       │
│  EACH LAYER IS INDEPENDENTLY TESTABLE AND REUSABLE                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## THE COMPONENT LIBRARY ARCHITECT'S CHECKLIST

Before you ship a component library, run through this:

```
┌──────────────────────────────────────────────────────────────────────┐
│  COMPONENT LIBRARY SHIP CHECKLIST                                     │
│                                                                       │
│  API DESIGN                                                           │
│  □ Every component works with minimal props (pit of success)          │
│  □ Every optional prop has a documented default                       │
│  □ Union types replace all string/boolean flag patterns               │
│  □ Impossible states cannot be expressed through props                │
│  □ Components extend native element props (Pressable, View, etc.)    │
│                                                                       │
│  PATTERNS                                                             │
│  □ Complex multi-part components use compound pattern                 │
│  □ Logic is separated from rendering (headless hooks)                 │
│  □ All interactive components support controlled + uncontrolled       │
│  □ Collection components use TypeScript generics                      │
│  □ Components that wrap native elements forward refs                  │
│                                                                       │
│  ACCESSIBILITY                                                        │
│  □ Every interactive component has correct ARIA roles                 │
│  □ Keyboard navigation works (arrows, enter, escape, tab)            │
│  □ Focus management is correct (trapping, restoration)               │
│  □ Screen reader announces state changes                             │
│  □ Touch targets meet minimum size (44x44 on mobile)                 │
│                                                                       │
│  DOCUMENTATION                                                        │
│  □ Every component has a Storybook story                              │
│  □ Every variant and state has a dedicated story                      │
│  □ Props tables are auto-generated from TypeScript                    │
│  □ Usage guidelines include do/don't examples                         │
│  □ Accessibility notes are documented per component                   │
│                                                                       │
│  TESTING                                                              │
│  □ Unit tests cover rendering, interaction, and edge cases            │
│  □ Interaction tests in Storybook cover real user flows               │
│  □ Visual regression catches unintended style changes                 │
│  □ a11y audit runs in CI (axe, jest-axe)                              │
│                                                                       │
│  VERSIONING                                                           │
│  □ Semantic versioning is followed strictly                           │
│  □ Breaking changes have deprecation warnings first                   │
│  □ Codemods exist for breaking changes that affect >10 files          │
│  □ Changelog is maintained with changesets                            │
│  □ Migration guides exist for every major version                     │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## SUMMARY

Building a component library is not about building components. It's about building a **system** that other developers use to build components. The distinction matters because it changes every decision:

- You don't optimize for the component author's convenience; you optimize for the **consumer's** convenience.
- You don't design the API around your implementation; you design the implementation around the **API you wish existed**.
- You don't ship when the component works; you ship when the component works, is documented, is tested, is accessible, and can be upgraded without breaking consumers.

The patterns in this chapter -- compound components, headless hooks, polymorphism, controllable state, generic types, imperative handles -- are not theoretical. They are the specific engineering techniques used by Radix UI, React Aria, shadcn/ui, and every production-grade component library that has survived contact with real consumers.

The 100x architect knows that a well-built component library is a **force multiplier**. Every hour invested in getting the patterns right saves ten hours of consumer confusion, Slack questions, wrapper components, and forks. Build the system right, and the components build themselves.

### Key Takeaways

1. **API design is architecture.** The props interface is the most important code in your library. Design it by writing consumer code first.

2. **Compound components are the gold standard** for complex, multi-part UI. Context + composition = flexible, self-documenting APIs.

3. **Headless first.** Separate behavior from rendering. Use Radix, React Aria, or Ark UI for the hard a11y work.

4. **Always support controlled + uncontrolled.** Use `useControllableState` in every stateful component. No exceptions.

5. **TypeScript generics make collection components type-safe.** The `<List<T>>` pattern should be second nature.

6. **Storybook is a development tool, not just documentation.** Write the story first, build the component second.

7. **Version with discipline.** Semver, changesets, deprecation warnings, codemods. The boring stuff is what earns trust.
