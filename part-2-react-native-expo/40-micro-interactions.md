<!--
  CHAPTER: 40
  TITLE: Micro-Interactions & Motion Design Cookbook
  PART: II — React Native & Expo
  PREREQS: Chapters 9
  KEY_TOPICS: shared element transitions, skeleton loaders, pull-to-refresh, swipe actions, bottom sheets, toast animations, page transitions, spring physics, gesture-driven interfaces, Reanimated recipes, Moti, layout animations
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 40: Micro-Interactions & Motion Design Cookbook

> **Part II — React Native & Expo** | Prerequisites: Chapter 9 | Difficulty: Intermediate → Advanced

> "The details are not the details. They make the design." — Charles Eames
>
> "If it doesn't move, it doesn't feel real." — Every user who tried your static app and then opened Instagram

---

<details>
<summary><strong>TL;DR</strong></summary>

- Micro-interactions are the difference between an app that "works" and an app that feels premium; every tap, swipe, and state change should have visual feedback
- Springs feel natural because they model real physics; use `withSpring` for almost everything, `withTiming` only for opacity fades and progress bars
- All interactive animations must run on the UI thread via Reanimated worklets; if your animation touches the JS thread, it will jank on older devices
- @gorhom/bottom-sheet is the gold standard for bottom sheets; do not build your own unless you have a very specific reason
- The cookbook at the end of this chapter has 15 copy-paste recipes; use them as starting points, then customize the spring configs to match your app's personality

</details>

## Why This Chapter Exists

Chapter 8 covered the fundamentals: Reanimated's shared values, worklets, `withSpring`, `withTiming`, Gesture Handler basics. You learned how the animation system works. You understand UI thread vs JS thread. You can make things move.

This chapter is about taste.

I have reviewed hundreds of React Native apps. The ones that feel "premium" -- the ones users describe as "smooth" or "polished" or "it just feels right" -- are not doing anything technically exotic. They are not using some secret animation library. They are using the same Reanimated and Gesture Handler that everyone else has access to.

The difference is that they animate everything. Every state change. Every user interaction. Every transition. Not with flashy, attention-grabbing animations, but with subtle, purposeful motion that gives the user constant feedback about what is happening.

Here is what "animate everything" looks like in practice:

- A button press has a scale-down effect (not just a color change)
- A list item being deleted slides out before disappearing
- A new item appearing in a list fades and slides in from below
- A bottom sheet snaps to points with spring physics, not linear timing
- A pull-to-refresh has a custom animation, not the default spinner
- A toast notification slides in from the top with a spring bounce
- A page transition has a shared element that morphs between screens
- A loading state shows content-shaped skeletons with a shimmer effect

None of these are hard individually. The challenge is doing all of them, consistently, across your entire app. That is what this chapter gives you -- patterns and recipes you can apply everywhere.

### In This Chapter
- Spring physics -- why springs feel natural and how to tune them
- Shared element transitions -- photo galleries, card-to-detail, hero animations
- Skeleton loaders -- shimmer effects, content-shaped placeholders
- Pull-to-refresh -- custom animations beyond the default spinner
- Swipe actions -- swipe-to-delete, swipe-to-archive
- Bottom sheets -- @gorhom/bottom-sheet deep dive
- Toast notifications -- animated toast system
- Page transitions -- custom navigator transitions
- Gesture-driven interfaces -- drag-to-reorder, pinch-to-zoom, card stacks
- Layout animations -- entering, exiting, layout transitions
- Lottie animations -- when and how to use Lottie
- Animation performance -- keeping everything on the UI thread
- The Cookbook -- 15 copy-paste recipes

### Related Chapters
- [Ch 8: Styling & Animation] -- Reanimated and Gesture Handler fundamentals
- [Ch 7: Navigation Architecture] -- React Navigation screen transitions
- [Ch 13: Performance Optimization] -- frame budgets and rendering performance

---

## 1. SPRING PHYSICS

### 1.1 Why Springs Feel Natural

Everything in the real world moves with inertia, friction, and elasticity. When you push a door, it doesn't move at a constant speed and then stop instantly. It accelerates, decelerates, and might bounce slightly if it hits a stop.

Linear timing animations (`withTiming` with a fixed duration) do not model this. They move at a mathematically defined curve (ease-in-out, cubic-bezier, etc.) for an exact duration. They feel "digital" -- smooth, predictable, and slightly lifeless.

Spring animations (`withSpring`) model a mass on a spring. They have three parameters:

| Parameter | What It Does | Default | Range |
|-----------|-------------|---------|-------|
| `damping` | How quickly the oscillation settles. Higher = less bouncy. | 10 | 1-100 |
| `stiffness` | How "tight" the spring is. Higher = faster. | 100 | 1-1000 |
| `mass` | How heavy the object is. Higher = slower, more momentum. | 1 | 0.1-10 |

```tsx
import Animated, { withSpring, useSharedValue, useAnimatedStyle } from 'react-native-reanimated';

function SpringDemo() {
  const translateY = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  const bounce = () => {
    translateY.value = -100;
    translateY.value = withSpring(0, {
      damping: 8,       // Low damping = more bouncy
      stiffness: 150,   // Higher stiffness = faster snap
      mass: 0.8,        // Lighter mass = more responsive
    });
  };

  return (
    <Animated.View style={[styles.box, animatedStyle]}>
      <Pressable onPress={bounce}>
        <Text>Tap me</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### 1.2 Overdamped vs Underdamped

The ratio of damping to stiffness determines the "feel":

**Underdamped (bouncy)** -- `damping` is low relative to `stiffness`. The animation overshoots and oscillates before settling. Use for playful interactions: like buttons, success states, notification badges.

```typescript
// Bouncy spring — use for playful interactions
const bouncySpring = {
  damping: 6,
  stiffness: 200,
  mass: 0.6,
};
```

**Critically damped (smooth, no overshoot)** -- The animation reaches the target as fast as possible without oscillating. Use for UI elements that need to feel precise: bottom sheets, drawer openings, panel resizes.

```typescript
// Critically damped — use for precise UI movements
const smoothSpring = {
  damping: 20,
  stiffness: 200,
  mass: 1,
};
```

**Overdamped (sluggish)** -- `damping` is high relative to `stiffness`. The animation slowly approaches the target without ever overshooting. Use for heavy elements: full-screen modals, background blurs.

```typescript
// Overdamped — use for heavy, deliberate movements
const heavySpring = {
  damping: 50,
  stiffness: 100,
  mass: 2,
};
```

### 1.3 Spring Presets

Here are the spring configs I use across every app. Define them once, use them everywhere:

```typescript
// lib/animation-config.ts
export const springs = {
  /** Snappy — for button presses, toggles, small elements */
  snappy: { damping: 15, stiffness: 400, mass: 0.5 },

  /** Bouncy — for success states, like buttons, badges */
  bouncy: { damping: 6, stiffness: 200, mass: 0.6 },

  /** Gentle — for page transitions, bottom sheets, large elements */
  gentle: { damping: 20, stiffness: 150, mass: 1 },

  /** Stiff — for drag releases, snap-to-position */
  stiff: { damping: 20, stiffness: 300, mass: 0.8 },

  /** Lazy — for background elements, parallax effects */
  lazy: { damping: 30, stiffness: 80, mass: 1.5 },

  /** Modal — for modal present/dismiss */
  modal: { damping: 25, stiffness: 200, mass: 1 },
} as const;
```

### 1.4 When to Use Timing Instead

Springs are not always the answer. Use `withTiming` for:

1. **Opacity fades** -- Fading in/out should be linear or ease-in-out. A bouncing opacity looks wrong.
2. **Progress bars** -- Progress should move linearly toward the target.
3. **Color transitions** -- Animating between colors works better with timing.
4. **Rotation loops** -- A continuously spinning loader should use `withRepeat` + `withTiming`.
5. **Exact duration requirements** -- "This animation must take exactly 300ms."

```typescript
import { withTiming, Easing } from 'react-native-reanimated';

// Fade in over 200ms
opacity.value = withTiming(1, { duration: 200, easing: Easing.inOut(Easing.ease) });

// Progress bar
progress.value = withTiming(0.75, { duration: 500, easing: Easing.out(Easing.cubic) });

// Continuous spin
rotation.value = withRepeat(
  withTiming(360, { duration: 1000, easing: Easing.linear }),
  -1, // infinite
  false // don't reverse
);
```

---

## 2. SHARED ELEMENT TRANSITIONS

### 2.1 What They Are

A shared element transition is when an element appears to "morph" from one screen to another. The classic example: you tap a photo thumbnail in a grid, and it expands smoothly into a full-screen detail view. The photo does not disappear and reappear -- it animates continuously between the two positions.

This is the single most impactful animation you can add to a mobile app. It creates a sense of spatial continuity that makes navigation feel physical rather than digital.

### 2.2 React Navigation Shared Element Transitions

React Navigation (v7+) supports shared element transitions via Reanimated's `SharedTransition` API:

```tsx
// screens/photo-grid.tsx
import Animated from 'react-native-reanimated';
import { useNavigation } from '@react-navigation/native';
import { FlatList, Pressable, Dimensions } from 'react-native';

const { width } = Dimensions.get('window');
const COLUMN_COUNT = 3;
const IMAGE_SIZE = width / COLUMN_COUNT - 2;

interface Photo {
  id: string;
  uri: string;
  title: string;
}

export function PhotoGrid({ photos }: { photos: Photo[] }) {
  const navigation = useNavigation();

  const renderPhoto = ({ item }: { item: Photo }) => (
    <Pressable
      onPress={() => navigation.navigate('PhotoDetail', { photo: item })}
    >
      <Animated.Image
        source={{ uri: item.uri }}
        style={{
          width: IMAGE_SIZE,
          height: IMAGE_SIZE,
          margin: 1,
        }}
        sharedTransitionTag={`photo-${item.id}`}
      />
    </Pressable>
  );

  return (
    <FlatList
      data={photos}
      renderItem={renderPhoto}
      keyExtractor={(item) => item.id}
      numColumns={COLUMN_COUNT}
    />
  );
}
```

```tsx
// screens/photo-detail.tsx
import Animated from 'react-native-reanimated';
import { View, Text, Dimensions } from 'react-native';

const { width, height } = Dimensions.get('window');

export function PhotoDetail({ route }: { route: any }) {
  const { photo } = route.params;

  return (
    <View style={{ flex: 1, backgroundColor: '#000' }}>
      <Animated.Image
        source={{ uri: photo.uri }}
        style={{
          width: width,
          height: width, // Square, or calculate aspect ratio
          alignSelf: 'center',
        }}
        sharedTransitionTag={`photo-${photo.id}`}
      />
      <Text style={{ color: '#fff', padding: 16, fontSize: 18 }}>
        {photo.title}
      </Text>
    </View>
  );
}
```

The `sharedTransitionTag` prop is the key. When both screens have an element with the same tag, React Navigation automatically animates between them. The element appears to move and resize from its position on the first screen to its position on the second screen.

### 2.3 Custom Shared Transition Config

You can customize the transition animation:

```tsx
import { SharedTransition, withSpring } from 'react-native-reanimated';

const customTransition = SharedTransition.custom((values) => {
  'worklet';
  return {
    originX: withSpring(values.targetOriginX, { damping: 20, stiffness: 200 }),
    originY: withSpring(values.targetOriginY, { damping: 20, stiffness: 200 }),
    width: withSpring(values.targetWidth, { damping: 20, stiffness: 200 }),
    height: withSpring(values.targetHeight, { damping: 20, stiffness: 200 }),
  };
});

// Use on the Animated component
<Animated.Image
  sharedTransitionTag={`photo-${photo.id}`}
  sharedTransitionStyle={customTransition}
  source={{ uri: photo.uri }}
  style={styles.image}
/>
```

### 2.4 Card to Full-Screen Pattern

A common pattern is a card in a list that expands to a full-screen detail view:

```tsx
// components/expandable-card.tsx
import Animated from 'react-native-reanimated';
import { Pressable, View, Text, Dimensions } from 'react-native';
import { useNavigation } from '@react-navigation/native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface CardProps {
  id: string;
  title: string;
  subtitle: string;
  imageUri: string;
  color: string;
}

export function ExpandableCard({ id, title, subtitle, imageUri, color }: CardProps) {
  const navigation = useNavigation();

  return (
    <Pressable onPress={() => navigation.navigate('CardDetail', { id, title, subtitle, imageUri, color })}>
      <Animated.View
        sharedTransitionTag={`card-bg-${id}`}
        style={{
          backgroundColor: color,
          borderRadius: 16,
          marginHorizontal: 16,
          marginBottom: 16,
          overflow: 'hidden',
        }}
      >
        <Animated.Image
          sharedTransitionTag={`card-image-${id}`}
          source={{ uri: imageUri }}
          style={{
            width: SCREEN_WIDTH - 32,
            height: 200,
          }}
        />
        <View style={{ padding: 16 }}>
          <Animated.Text
            sharedTransitionTag={`card-title-${id}`}
            style={{ fontSize: 20, fontWeight: '700', color: '#fff' }}
          >
            {title}
          </Animated.Text>
          <Text style={{ fontSize: 14, color: 'rgba(255,255,255,0.7)', marginTop: 4 }}>
            {subtitle}
          </Text>
        </View>
      </Animated.View>
    </Pressable>
  );
}
```

```tsx
// screens/card-detail.tsx
import Animated from 'react-native-reanimated';
import { View, Text, ScrollView, Dimensions } from 'react-native';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

export function CardDetail({ route }: { route: any }) {
  const { id, title, subtitle, imageUri, color } = route.params;

  return (
    <Animated.View
      sharedTransitionTag={`card-bg-${id}`}
      style={{ flex: 1, backgroundColor: color }}
    >
      <ScrollView>
        <Animated.Image
          sharedTransitionTag={`card-image-${id}`}
          source={{ uri: imageUri }}
          style={{ width: SCREEN_WIDTH, height: 300 }}
        />
        <View style={{ padding: 24 }}>
          <Animated.Text
            sharedTransitionTag={`card-title-${id}`}
            style={{ fontSize: 32, fontWeight: '700', color: '#fff' }}
          >
            {title}
          </Animated.Text>
          <Text style={{ fontSize: 16, color: 'rgba(255,255,255,0.8)', marginTop: 8, lineHeight: 24 }}>
            {subtitle}
          </Text>
          {/* Rest of detail content */}
        </View>
      </ScrollView>
    </Animated.View>
  );
}
```

Multiple elements can share transitions simultaneously. The background, image, and title all animate independently from their source position to their destination position.

---

## 3. SKELETON LOADERS

### 3.1 Why Skeletons Beat Spinners

Spinners say "wait." Skeletons say "content is coming and it will look like this." The difference matters more than you think.

Research from Google and Facebook shows that perceived load time is lower when users see content-shaped placeholders instead of a generic spinner. The user's brain starts processing the layout before the content arrives. When the content appears, it feels instant because the brain already has a mental model of the page structure.

### 3.2 Shimmer Effect with Reanimated

The shimmer is a gradient that sweeps across the skeleton, indicating activity:

```tsx
// components/skeleton.tsx
import { useEffect } from 'react';
import { View, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  Easing,
  interpolate,
} from 'react-native-reanimated';
import { LinearGradient } from 'expo-linear-gradient';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface SkeletonProps {
  width: number | string;
  height: number;
  borderRadius?: number;
  style?: any;
}

export function Skeleton({
  width,
  height,
  borderRadius = 4,
  style,
}: SkeletonProps) {
  const shimmerPosition = useSharedValue(0);

  useEffect(() => {
    shimmerPosition.value = withRepeat(
      withTiming(1, { duration: 1200, easing: Easing.inOut(Easing.ease) }),
      -1, // infinite
      false // don't reverse — sweep in one direction
    );
  }, []);

  const shimmerStyle = useAnimatedStyle(() => {
    const translateX = interpolate(
      shimmerPosition.value,
      [0, 1],
      [-SCREEN_WIDTH, SCREEN_WIDTH]
    );

    return {
      transform: [{ translateX }],
    };
  });

  return (
    <View
      style={[
        {
          width,
          height,
          borderRadius,
          backgroundColor: '#E5E7EB',
          overflow: 'hidden',
        },
        style,
      ]}
    >
      <Animated.View style={[StyleSheet.absoluteFill, shimmerStyle]}>
        <LinearGradient
          colors={['transparent', 'rgba(255,255,255,0.4)', 'transparent']}
          start={{ x: 0, y: 0 }}
          end={{ x: 1, y: 0 }}
          style={StyleSheet.absoluteFill}
        />
      </Animated.View>
    </View>
  );
}
```

### 3.3 Content-Shaped Skeletons

Do not just show rectangles. Match the shape of your actual content:

```tsx
// components/skeletons/card-skeleton.tsx
import { View } from 'react-native';
import { Skeleton } from '../skeleton';

export function CardSkeleton() {
  return (
    <View style={{
      padding: 16,
      backgroundColor: '#fff',
      borderRadius: 12,
      marginBottom: 12,
    }}>
      {/* Image placeholder */}
      <Skeleton width="100%" height={200} borderRadius={8} />

      {/* Title */}
      <Skeleton
        width="70%"
        height={20}
        borderRadius={4}
        style={{ marginTop: 16 }}
      />

      {/* Subtitle */}
      <Skeleton
        width="90%"
        height={14}
        borderRadius={4}
        style={{ marginTop: 8 }}
      />

      {/* Meta row */}
      <View style={{ flexDirection: 'row', marginTop: 16, gap: 8 }}>
        <Skeleton width={32} height={32} borderRadius={16} />
        <View style={{ flex: 1 }}>
          <Skeleton width="40%" height={12} borderRadius={4} />
          <Skeleton
            width="25%"
            height={10}
            borderRadius={4}
            style={{ marginTop: 4 }}
          />
        </View>
      </View>
    </View>
  );
}
```

```tsx
// components/skeletons/profile-skeleton.tsx
import { View } from 'react-native';
import { Skeleton } from '../skeleton';

export function ProfileSkeleton() {
  return (
    <View style={{ padding: 24, alignItems: 'center' }}>
      {/* Avatar */}
      <Skeleton width={80} height={80} borderRadius={40} />

      {/* Name */}
      <Skeleton
        width={150}
        height={22}
        borderRadius={4}
        style={{ marginTop: 16 }}
      />

      {/* Bio line 1 */}
      <Skeleton
        width={250}
        height={14}
        borderRadius={4}
        style={{ marginTop: 8 }}
      />

      {/* Bio line 2 */}
      <Skeleton
        width={200}
        height={14}
        borderRadius={4}
        style={{ marginTop: 4 }}
      />

      {/* Stats row */}
      <View style={{
        flexDirection: 'row',
        marginTop: 24,
        gap: 32,
      }}>
        {[1, 2, 3].map((i) => (
          <View key={i} style={{ alignItems: 'center' }}>
            <Skeleton width={40} height={20} borderRadius={4} />
            <Skeleton
              width={60}
              height={12}
              borderRadius={4}
              style={{ marginTop: 4 }}
            />
          </View>
        ))}
      </View>
    </View>
  );
}
```

```tsx
// components/skeletons/list-skeleton.tsx
import { View } from 'react-native';
import { Skeleton } from '../skeleton';

export function ListSkeleton({ count = 5 }: { count?: number }) {
  return (
    <View>
      {Array.from({ length: count }).map((_, i) => (
        <View
          key={i}
          style={{
            flexDirection: 'row',
            padding: 16,
            borderBottomWidth: 1,
            borderBottomColor: '#F3F4F6',
          }}
        >
          {/* Avatar */}
          <Skeleton width={48} height={48} borderRadius={24} />

          {/* Text content */}
          <View style={{ flex: 1, marginLeft: 12, justifyContent: 'center' }}>
            <Skeleton width="60%" height={16} borderRadius={4} />
            <Skeleton
              width="80%"
              height={12}
              borderRadius={4}
              style={{ marginTop: 6 }}
            />
          </View>

          {/* Right element (timestamp, etc.) */}
          <Skeleton width={40} height={12} borderRadius={4} />
        </View>
      ))}
    </View>
  );
}
```

### 3.4 Skeleton with MotiView

Moti provides a simpler API for common animations. Its skeleton helper is particularly clean:

```tsx
// Using Moti's Skeleton component
import { MotiView } from 'moti';
import { Skeleton as MotiSkeleton } from 'moti/skeleton';

export function CardSkeletonMoti({ loading }: { loading: boolean }) {
  return (
    <MotiView
      animate={{ opacity: loading ? 1 : 0 }}
      transition={{ type: 'timing', duration: 200 }}
    >
      <MotiSkeleton
        colorMode="light"
        width="100%"
        height={200}
        radius={8}
      />
      <MotiSkeleton
        colorMode="light"
        width="70%"
        height={20}
        radius={4}
      />
      <MotiSkeleton
        colorMode="light"
        width="50%"
        height={16}
        radius={4}
      />
    </MotiView>
  );
}
```

Moti's `Skeleton` component handles the shimmer animation automatically. Less code, less control.

### 3.5 Integration with TanStack Query

```tsx
// screens/feed.tsx
import { useQuery } from '@tanstack/react-query';
import { FlatList } from 'react-native';
import { CardSkeleton } from '@/components/skeletons/card-skeleton';
import { FeedCard } from '@/components/feed-card';

export function FeedScreen() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ['feed'],
    queryFn: fetchFeed,
  });

  if (isLoading) {
    return (
      <FlatList
        data={Array.from({ length: 5 })}
        renderItem={() => <CardSkeleton />}
        keyExtractor={(_, i) => `skeleton-${i}`}
        contentContainerStyle={{ padding: 16 }}
      />
    );
  }

  return (
    <FlatList
      data={data}
      renderItem={({ item }) => <FeedCard {...item} />}
      keyExtractor={(item) => item.id}
      contentContainerStyle={{ padding: 16 }}
    />
  );
}
```

---

## 4. PULL-TO-REFRESH

### 4.1 The Default Is Boring

The default `RefreshControl` on both iOS and Android is a spinner. It works. It is also what every other app uses. A custom pull-to-refresh animation is a small detail that makes your app memorable.

### 4.2 Custom Pull-to-Refresh with Reanimated

```tsx
// components/custom-refresh.tsx
import { useState, useCallback } from 'react';
import { View, FlatList, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  withSequence,
  interpolate,
  Extrapolation,
  useAnimatedScrollHandler,
  runOnJS,
} from 'react-native-reanimated';

const REFRESH_THRESHOLD = 80;
const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface CustomRefreshProps {
  onRefresh: () => Promise<void>;
  children: React.ReactNode;
}

export function CustomRefreshList({
  data,
  renderItem,
  onRefresh,
  keyExtractor,
}: {
  data: any[];
  renderItem: any;
  onRefresh: () => Promise<void>;
  keyExtractor: (item: any) => string;
}) {
  const [refreshing, setRefreshing] = useState(false);
  const scrollY = useSharedValue(0);
  const refreshProgress = useSharedValue(0);

  const handleRefresh = useCallback(async () => {
    setRefreshing(true);
    refreshProgress.value = withSpring(1, { damping: 15, stiffness: 200 });
    try {
      await onRefresh();
    } finally {
      refreshProgress.value = withTiming(0, { duration: 300 });
      setRefreshing(false);
    }
  }, [onRefresh]);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;

      if (event.contentOffset.y < -REFRESH_THRESHOLD && !refreshing) {
        runOnJS(handleRefresh)();
      }

      // Map pull distance to progress (0 to 1)
      if (event.contentOffset.y < 0) {
        refreshProgress.value = interpolate(
          -event.contentOffset.y,
          [0, REFRESH_THRESHOLD],
          [0, 1],
          Extrapolation.CLAMP
        );
      }
    },
  });

  // Custom refresh indicator animation
  const indicatorStyle = useAnimatedStyle(() => {
    const translateY = interpolate(
      refreshProgress.value,
      [0, 1],
      [-60, 0],
      Extrapolation.CLAMP
    );

    const scale = interpolate(
      refreshProgress.value,
      [0, 0.5, 1],
      [0.3, 0.8, 1],
      Extrapolation.CLAMP
    );

    const rotate = interpolate(
      refreshProgress.value,
      [0, 1],
      [0, 360]
    );

    return {
      transform: [
        { translateY },
        { scale },
        { rotate: `${rotate}deg` },
      ],
      opacity: refreshProgress.value,
    };
  });

  return (
    <View style={{ flex: 1 }}>
      {/* Custom refresh indicator */}
      <Animated.View style={[{
        position: 'absolute',
        top: 10,
        alignSelf: 'center',
        width: 40,
        height: 40,
        borderRadius: 20,
        backgroundColor: '#111827',
        justifyContent: 'center',
        alignItems: 'center',
        zIndex: 10,
      }, indicatorStyle]}>
        <Animated.Text style={{ color: '#fff', fontSize: 20 }}>
          ↻
        </Animated.Text>
      </Animated.View>

      <Animated.FlatList
        data={data}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        bounces
      />
    </View>
  );
}
```

### 4.3 Lottie Pull-to-Refresh

For more complex refresh animations (a character pulling, liquid fill, etc.), use Lottie:

```tsx
// components/lottie-refresh.tsx
import { useState, useCallback, useRef } from 'react';
import { View, FlatList } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedProps,
  interpolate,
  runOnJS,
} from 'react-native-reanimated';
import LottieView from 'lottie-react-native';

const AnimatedLottie = Animated.createAnimatedComponent(LottieView);

export function LottieRefreshList({
  data,
  renderItem,
  onRefresh,
  keyExtractor,
}: {
  data: any[];
  renderItem: any;
  onRefresh: () => Promise<void>;
  keyExtractor: (item: any) => string;
}) {
  const [refreshing, setRefreshing] = useState(false);
  const pullProgress = useSharedValue(0);
  const lottieRef = useRef<LottieView>(null);

  const handleRefresh = useCallback(async () => {
    setRefreshing(true);
    lottieRef.current?.play(); // Play the full animation on refresh
    try {
      await onRefresh();
    } finally {
      lottieRef.current?.reset();
      setRefreshing(false);
    }
  }, [onRefresh]);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      if (event.contentOffset.y < 0) {
        pullProgress.value = Math.min(-event.contentOffset.y / 80, 1);
      } else {
        pullProgress.value = 0;
      }

      if (event.contentOffset.y < -80 && !refreshing) {
        runOnJS(handleRefresh)();
      }
    },
  });

  // Drive Lottie progress based on pull distance
  const lottieAnimatedProps = useAnimatedProps(() => ({
    progress: refreshing ? undefined : pullProgress.value * 0.5, // First half during pull
  }));

  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={{
        position: 'absolute',
        top: 0,
        alignSelf: 'center',
        width: 60,
        height: 60,
        zIndex: 10,
      }}>
        <AnimatedLottie
          ref={lottieRef}
          source={require('@/assets/animations/refresh.json')}
          animatedProps={lottieAnimatedProps}
          loop={refreshing}
          style={{ width: 60, height: 60 }}
        />
      </Animated.View>

      <Animated.FlatList
        data={data}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        bounces
      />
    </View>
  );
}
```

### 4.4 Integration with TanStack Query

```tsx
// Pull-to-refresh with TanStack Query
const { data, refetch, isRefetching } = useQuery({
  queryKey: ['feed'],
  queryFn: fetchFeed,
});

// Pass refetch as onRefresh
<CustomRefreshList
  data={data ?? []}
  renderItem={renderItem}
  keyExtractor={(item) => item.id}
  onRefresh={async () => { await refetch(); }}
/>
```

---

## 5. SWIPE ACTIONS

### 5.1 Swipe-to-Delete / Swipe-to-Archive

This is the interaction email apps made standard. Swipe right to archive, swipe left to delete. Here is a complete implementation:

```tsx
// components/swipeable-row.tsx
import { View, Text, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  runOnJS,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';
import {
  Gesture,
  GestureDetector,
} from 'react-native-gesture-handler';

const { width: SCREEN_WIDTH } = Dimensions.get('window');
const SWIPE_THRESHOLD = SCREEN_WIDTH * 0.3;

interface SwipeableRowProps {
  children: React.ReactNode;
  onDelete: () => void;
  onArchive: () => void;
}

export function SwipeableRow({ children, onDelete, onArchive }: SwipeableRowProps) {
  const translateX = useSharedValue(0);
  const rowHeight = useSharedValue(70); // Adjust to your row height
  const rowOpacity = useSharedValue(1);

  const dismissRow = (action: 'delete' | 'archive') => {
    const direction = action === 'delete' ? -SCREEN_WIDTH : SCREEN_WIDTH;
    translateX.value = withTiming(direction, { duration: 200 }, () => {
      rowHeight.value = withTiming(0, { duration: 200 });
      rowOpacity.value = withTiming(0, { duration: 200 });
    });

    // Call the action after animation
    setTimeout(() => {
      if (action === 'delete') onDelete();
      else onArchive();
    }, 400);
  };

  const panGesture = Gesture.Pan()
    .activeOffsetX([-10, 10]) // Only activate after 10px horizontal movement
    .failOffsetY([-5, 5])     // Fail if vertical movement > 5px (for scrolling)
    .onUpdate((event) => {
      translateX.value = event.translationX;
    })
    .onEnd((event) => {
      const { translationX, velocityX } = event;

      // Swipe left far enough or fast enough → delete
      if (translationX < -SWIPE_THRESHOLD || velocityX < -500) {
        runOnJS(dismissRow)('delete');
        return;
      }

      // Swipe right far enough or fast enough → archive
      if (translationX > SWIPE_THRESHOLD || velocityX > 500) {
        runOnJS(dismissRow)('archive');
        return;
      }

      // Snap back
      translateX.value = withSpring(0, { damping: 20, stiffness: 300 });
    });

  const rowAnimatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  const containerAnimatedStyle = useAnimatedStyle(() => ({
    height: rowHeight.value,
    opacity: rowOpacity.value,
  }));

  // Left background (archive — green)
  const leftActionStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      translateX.value,
      [0, SWIPE_THRESHOLD],
      [0, 1],
      Extrapolation.CLAMP
    );
    const scale = interpolate(
      translateX.value,
      [0, SWIPE_THRESHOLD],
      [0.5, 1],
      Extrapolation.CLAMP
    );

    return { opacity, transform: [{ scale }] };
  });

  // Right background (delete — red)
  const rightActionStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      translateX.value,
      [-SWIPE_THRESHOLD, 0],
      [1, 0],
      Extrapolation.CLAMP
    );
    const scale = interpolate(
      translateX.value,
      [-SWIPE_THRESHOLD, 0],
      [1, 0.5],
      Extrapolation.CLAMP
    );

    return { opacity, transform: [{ scale }] };
  });

  return (
    <Animated.View style={[containerAnimatedStyle, { overflow: 'hidden' }]}>
      {/* Background actions */}
      <View style={StyleSheet.absoluteFill}>
        {/* Archive (left swipe reveals right) */}
        <View style={[styles.actionBackground, styles.archiveBackground]}>
          <Animated.Text style={[styles.actionText, leftActionStyle]}>
            Archive
          </Animated.Text>
        </View>

        {/* Delete (right swipe reveals left) */}
        <View style={[styles.actionBackground, styles.deleteBackground]}>
          <Animated.Text style={[styles.actionText, rightActionStyle]}>
            Delete
          </Animated.Text>
        </View>
      </View>

      {/* Foreground row */}
      <GestureDetector gesture={panGesture}>
        <Animated.View style={[styles.row, rowAnimatedStyle]}>
          {children}
        </Animated.View>
      </GestureDetector>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  row: {
    backgroundColor: '#ffffff',
  },
  actionBackground: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'center',
  },
  archiveBackground: {
    backgroundColor: '#059669',
    alignItems: 'flex-start',
    paddingLeft: 24,
  },
  deleteBackground: {
    backgroundColor: '#DC2626',
    alignItems: 'flex-end',
    paddingRight: 24,
  },
  actionText: {
    color: '#ffffff',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

**Usage:**

```tsx
<FlatList
  data={emails}
  renderItem={({ item }) => (
    <SwipeableRow
      onDelete={() => deleteEmail(item.id)}
      onArchive={() => archiveEmail(item.id)}
    >
      <EmailRow email={item} />
    </SwipeableRow>
  )}
/>
```

---

## 6. BOTTOM SHEETS

### 6.1 @gorhom/bottom-sheet

This library is the gold standard. Do not build your own bottom sheet. Do not use some random npm package with 200 stars. Use @gorhom/bottom-sheet. It handles snap points, backdrop, keyboard avoidance, scrollable content, and gesture interaction correctly -- which is much harder than it sounds.

```bash
npm install @gorhom/bottom-sheet
```

**Note**: @gorhom/bottom-sheet requires react-native-reanimated and react-native-gesture-handler, which you should already have if you are following this book.

### 6.2 Basic Bottom Sheet

```tsx
// components/basic-bottom-sheet.tsx
import { useCallback, useMemo, useRef } from 'react';
import { View, Text, StyleSheet, Button } from 'react-native';
import BottomSheet, { BottomSheetBackdrop } from '@gorhom/bottom-sheet';

export function BasicBottomSheetExample() {
  const bottomSheetRef = useRef<BottomSheet>(null);

  // Define snap points — the heights the sheet can snap to
  const snapPoints = useMemo(() => ['25%', '50%', '90%'], []);

  // Open the sheet
  const handleOpen = useCallback(() => {
    bottomSheetRef.current?.snapToIndex(1); // Snap to 50%
  }, []);

  // Close the sheet
  const handleClose = useCallback(() => {
    bottomSheetRef.current?.close();
  }, []);

  // Custom backdrop
  const renderBackdrop = useCallback(
    (props: any) => (
      <BottomSheetBackdrop
        {...props}
        disappearsOnIndex={-1}
        appearsOnIndex={0}
        opacity={0.5}
      />
    ),
    []
  );

  return (
    <View style={styles.container}>
      <Button title="Open Sheet" onPress={handleOpen} />

      <BottomSheet
        ref={bottomSheetRef}
        index={-1} // Start closed (-1 = closed)
        snapPoints={snapPoints}
        enablePanDownToClose
        backdropComponent={renderBackdrop}
        backgroundStyle={styles.sheetBackground}
        handleIndicatorStyle={styles.handleIndicator}
      >
        <View style={styles.sheetContent}>
          <Text style={styles.sheetTitle}>Bottom Sheet</Text>
          <Text style={styles.sheetBody}>
            Swipe down to dismiss, or snap to different heights.
          </Text>
          <Button title="Close" onPress={handleClose} />
        </View>
      </BottomSheet>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  sheetBackground: {
    backgroundColor: '#ffffff',
    borderTopLeftRadius: 24,
    borderTopRightRadius: 24,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -4 },
    shadowOpacity: 0.1,
    shadowRadius: 12,
    elevation: 8,
  },
  handleIndicator: {
    backgroundColor: '#D1D5DB',
    width: 40,
    height: 4,
    borderRadius: 2,
  },
  sheetContent: { flex: 1, padding: 24 },
  sheetTitle: { fontSize: 22, fontWeight: '700', color: '#111827', marginBottom: 8 },
  sheetBody: { fontSize: 15, color: '#6B7280', lineHeight: 22 },
});
```

### 6.3 Bottom Sheet with Scrollable Content

One of the hardest problems with bottom sheets is scrollable content inside them. When the user scrolls up on a list inside the sheet, should the sheet expand or should the list scroll? @gorhom/bottom-sheet handles this:

```tsx
import BottomSheet, { BottomSheetFlatList } from '@gorhom/bottom-sheet';

export function ScrollableBottomSheet({ items }: { items: any[] }) {
  const bottomSheetRef = useRef<BottomSheet>(null);
  const snapPoints = useMemo(() => ['40%', '90%'], []);

  const renderItem = useCallback(({ item }: { item: any }) => (
    <View style={{
      padding: 16,
      borderBottomWidth: 1,
      borderBottomColor: '#F3F4F6',
    }}>
      <Text style={{ fontSize: 16, fontWeight: '600' }}>{item.title}</Text>
      <Text style={{ fontSize: 14, color: '#6B7280', marginTop: 4 }}>
        {item.description}
      </Text>
    </View>
  ), []);

  return (
    <BottomSheet
      ref={bottomSheetRef}
      index={0}
      snapPoints={snapPoints}
      enablePanDownToClose
    >
      {/* BottomSheetFlatList — NOT regular FlatList */}
      <BottomSheetFlatList
        data={items}
        renderItem={renderItem}
        keyExtractor={(item) => item.id}
        contentContainerStyle={{ paddingBottom: 24 }}
      />
    </BottomSheet>
  );
}
```

**Critical**: Use `BottomSheetFlatList`, `BottomSheetScrollView`, or `BottomSheetSectionList` inside a bottom sheet -- not the regular React Native versions. The @gorhom versions handle the gesture coordination between sheet dragging and content scrolling.

### 6.4 Bottom Sheet with Keyboard

When a bottom sheet contains a text input, the keyboard interaction needs special handling:

```tsx
import BottomSheet, { BottomSheetTextInput } from '@gorhom/bottom-sheet';

export function InputBottomSheet() {
  const bottomSheetRef = useRef<BottomSheet>(null);
  const snapPoints = useMemo(() => ['30%', '60%'], []);

  return (
    <BottomSheet
      ref={bottomSheetRef}
      index={0}
      snapPoints={snapPoints}
      keyboardBehavior="interactive"  // Sheet moves with keyboard
      keyboardBlurBehavior="restore"   // Restore snap point when keyboard dismisses
      android_keyboardInputMode="adjustResize" // Android keyboard handling
    >
      <View style={{ padding: 16 }}>
        <Text style={{ fontSize: 18, fontWeight: '700', marginBottom: 12 }}>
          Add a comment
        </Text>
        {/* BottomSheetTextInput — NOT regular TextInput */}
        <BottomSheetTextInput
          placeholder="Write your comment..."
          style={{
            borderWidth: 1,
            borderColor: '#D1D5DB',
            borderRadius: 8,
            padding: 12,
            fontSize: 16,
            minHeight: 100,
            textAlignVertical: 'top',
          }}
          multiline
        />
      </View>
    </BottomSheet>
  );
}
```

### 6.5 Dynamic Snap Points

Sometimes the sheet height should depend on its content:

```tsx
import BottomSheet, { BottomSheetView } from '@gorhom/bottom-sheet';

export function DynamicBottomSheet({ children }: { children: React.ReactNode }) {
  const bottomSheetRef = useRef<BottomSheet>(null);

  return (
    <BottomSheet
      ref={bottomSheetRef}
      enableDynamicSizing // Let the sheet size to its content
      enablePanDownToClose
      maxDynamicContentSize={600} // Don't grow beyond 600px
    >
      <BottomSheetView>
        {children}
      </BottomSheetView>
    </BottomSheet>
  );
}
```

---

## 7. TOAST NOTIFICATIONS

### 7.1 Building a Custom Toast System

Most apps need toast notifications: brief messages that appear at the top or bottom of the screen and auto-dismiss. Here is a complete implementation:

```tsx
// lib/toast.tsx
import { createContext, useCallback, useContext, useState, useRef } from 'react';
import { View, Text, StyleSheet, Pressable, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  withDelay,
  withSequence,
  runOnJS,
  SlideInUp,
  SlideOutUp,
  Layout,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

type ToastType = 'success' | 'error' | 'info' | 'warning';

interface Toast {
  id: string;
  type: ToastType;
  title: string;
  message?: string;
  duration?: number;
}

const toastConfig: Record<ToastType, { backgroundColor: string; icon: string }> = {
  success: { backgroundColor: '#059669', icon: '✓' },
  error: { backgroundColor: '#DC2626', icon: '✕' },
  info: { backgroundColor: '#2563EB', icon: 'i' },
  warning: { backgroundColor: '#D97706', icon: '!' },
};

interface ToastContextType {
  show: (toast: Omit<Toast, 'id'>) => void;
}

const ToastContext = createContext<ToastContextType | null>(null);

export function useToast() {
  const context = useContext(ToastContext);
  if (!context) throw new Error('useToast must be used within ToastProvider');
  return context;
}

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  const counterRef = useRef(0);

  const show = useCallback((toast: Omit<Toast, 'id'>) => {
    const id = `toast-${++counterRef.current}`;
    setToasts((prev) => [...prev, { ...toast, id }]);

    // Auto-dismiss
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, toast.duration ?? 3000);
  }, []);

  const dismiss = useCallback((id: string) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  }, []);

  return (
    <ToastContext.Provider value={{ show }}>
      {children}
      <View style={styles.toastContainer} pointerEvents="box-none">
        {toasts.map((toast) => (
          <ToastItem key={toast.id} toast={toast} onDismiss={dismiss} />
        ))}
      </View>
    </ToastContext.Provider>
  );
}

function ToastItem({ toast, onDismiss }: { toast: Toast; onDismiss: (id: string) => void }) {
  const config = toastConfig[toast.type];
  const progress = useSharedValue(1);

  // Auto-dismiss progress bar
  const duration = toast.duration ?? 3000;
  progress.value = withTiming(0, { duration });

  const progressStyle = useAnimatedStyle(() => ({
    width: `${progress.value * 100}%`,
  }));

  return (
    <Animated.View
      entering={SlideInUp.springify().damping(15).stiffness(200)}
      exiting={SlideOutUp.duration(200)}
      layout={Layout.springify()}
    >
      <Pressable onPress={() => onDismiss(toast.id)}>
        <View style={[styles.toast, { backgroundColor: config.backgroundColor }]}>
          <View style={styles.toastIconContainer}>
            <Text style={styles.toastIcon}>{config.icon}</Text>
          </View>
          <View style={styles.toastContent}>
            <Text style={styles.toastTitle}>{toast.title}</Text>
            {toast.message && (
              <Text style={styles.toastMessage}>{toast.message}</Text>
            )}
          </View>
          {/* Progress bar */}
          <Animated.View style={[styles.progressBar, progressStyle]} />
        </View>
      </Pressable>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  toastContainer: {
    position: 'absolute',
    top: 60, // Below status bar
    left: 16,
    right: 16,
    zIndex: 9999,
    gap: 8,
  },
  toast: {
    borderRadius: 12,
    padding: 14,
    flexDirection: 'row',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.15,
    shadowRadius: 12,
    elevation: 6,
    overflow: 'hidden',
  },
  toastIconContainer: {
    width: 28,
    height: 28,
    borderRadius: 14,
    backgroundColor: 'rgba(255,255,255,0.2)',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  toastIcon: {
    color: '#ffffff',
    fontSize: 14,
    fontWeight: '700',
  },
  toastContent: {
    flex: 1,
  },
  toastTitle: {
    color: '#ffffff',
    fontSize: 15,
    fontWeight: '600',
  },
  toastMessage: {
    color: 'rgba(255,255,255,0.8)',
    fontSize: 13,
    marginTop: 2,
  },
  progressBar: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    height: 3,
    backgroundColor: 'rgba(255,255,255,0.3)',
    borderBottomLeftRadius: 12,
  },
});
```

**Usage:**

```tsx
// Wrap your app
export default function App() {
  return (
    <ToastProvider>
      <NavigationContainer>
        {/* ... */}
      </NavigationContainer>
    </ToastProvider>
  );
}

// Use anywhere in your app
function SomeScreen() {
  const toast = useToast();

  const handleSave = async () => {
    try {
      await saveData();
      toast.show({
        type: 'success',
        title: 'Saved!',
        message: 'Your changes have been saved.',
      });
    } catch (error) {
      toast.show({
        type: 'error',
        title: 'Save failed',
        message: 'Please check your connection and try again.',
        duration: 5000,
      });
    }
  };
}
```

---

## 8. PAGE TRANSITIONS

### 8.1 Custom Stack Navigator Transitions

React Navigation lets you customize screen transitions. Here are the most common patterns:

```tsx
// navigation/transitions.ts
import { TransitionPresets } from '@react-navigation/stack';
import type { StackCardInterpolationProps } from '@react-navigation/stack';

// Fade transition
export const fadeTransition = {
  cardStyleInterpolator: ({ current }: StackCardInterpolationProps) => ({
    cardStyle: {
      opacity: current.progress,
    },
  }),
  transitionSpec: {
    open: { animation: 'timing', config: { duration: 250 } },
    close: { animation: 'timing', config: { duration: 200 } },
  },
};

// Slide from right (default iOS, customized)
export const slideFromRight = {
  cardStyleInterpolator: ({ current, next, layouts }: StackCardInterpolationProps) => ({
    cardStyle: {
      transform: [
        {
          translateX: current.progress.interpolate({
            inputRange: [0, 1],
            outputRange: [layouts.screen.width, 0],
          }),
        },
      ],
    },
    overlayStyle: {
      opacity: current.progress.interpolate({
        inputRange: [0, 1],
        outputRange: [0, 0.5],
      }),
    },
  }),
};

// Slide from bottom (modal)
export const slideFromBottom = {
  cardStyleInterpolator: ({ current, layouts }: StackCardInterpolationProps) => ({
    cardStyle: {
      transform: [
        {
          translateY: current.progress.interpolate({
            inputRange: [0, 1],
            outputRange: [layouts.screen.height, 0],
          }),
        },
      ],
      borderTopLeftRadius: current.progress.interpolate({
        inputRange: [0, 1],
        outputRange: [20, 0],
      }),
      borderTopRightRadius: current.progress.interpolate({
        inputRange: [0, 1],
        outputRange: [20, 0],
      }),
    },
    overlayStyle: {
      opacity: current.progress.interpolate({
        inputRange: [0, 1],
        outputRange: [0, 0.5],
      }),
    },
  }),
};

// Scale + fade (like iOS App Store card expand)
export const scaleFromCenter = {
  cardStyleInterpolator: ({ current }: StackCardInterpolationProps) => ({
    cardStyle: {
      opacity: current.progress.interpolate({
        inputRange: [0, 0.5, 1],
        outputRange: [0, 0.5, 1],
      }),
      transform: [
        {
          scale: current.progress.interpolate({
            inputRange: [0, 1],
            outputRange: [0.9, 1],
          }),
        },
      ],
    },
  }),
  transitionSpec: {
    open: { animation: 'spring', config: { damping: 20, stiffness: 200 } },
    close: { animation: 'timing', config: { duration: 200 } },
  },
};
```

**Usage in navigator:**

```tsx
import { createStackNavigator } from '@react-navigation/stack';
import { fadeTransition, slideFromBottom, scaleFromCenter } from './transitions';

const Stack = createStackNavigator();

function AppNavigator() {
  return (
    <Stack.Navigator>
      <Stack.Screen
        name="Home"
        component={HomeScreen}
      />
      <Stack.Screen
        name="Detail"
        component={DetailScreen}
        options={scaleFromCenter}
      />
      <Stack.Screen
        name="Settings"
        component={SettingsScreen}
        options={{
          presentation: 'modal',
          ...slideFromBottom,
        }}
      />
    </Stack.Navigator>
  );
}
```

### 8.2 Per-Screen Transitions

You can also set transitions dynamically:

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={({ route }) => ({
    ...(route.params?.transition === 'fade' ? fadeTransition : scaleFromCenter),
  })}
/>
```

---

## 9. GESTURE-DRIVEN INTERFACES

### 9.1 Drag-to-Reorder List

```tsx
// components/draggable-list.tsx
import { useState, useCallback } from 'react';
import { View, Text, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  useAnimatedReaction,
  runOnJS,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const ROW_HEIGHT = 64;

interface DraggableItem {
  id: string;
  label: string;
}

export function DraggableList({
  items: initialItems,
  onReorder,
}: {
  items: DraggableItem[];
  onReorder: (items: DraggableItem[]) => void;
}) {
  const [items, setItems] = useState(initialItems);

  const handleReorder = useCallback((fromIndex: number, toIndex: number) => {
    setItems((prev) => {
      const newItems = [...prev];
      const [removed] = newItems.splice(fromIndex, 1);
      newItems.splice(toIndex, 0, removed);
      onReorder(newItems);
      return newItems;
    });
  }, [onReorder]);

  return (
    <View style={{ height: items.length * ROW_HEIGHT }}>
      {items.map((item, index) => (
        <DraggableRow
          key={item.id}
          item={item}
          index={index}
          totalItems={items.length}
          onReorder={handleReorder}
        />
      ))}
    </View>
  );
}

function DraggableRow({
  item,
  index,
  totalItems,
  onReorder,
}: {
  item: DraggableItem;
  index: number;
  totalItems: number;
  onReorder: (fromIndex: number, toIndex: number) => void;
}) {
  const translateY = useSharedValue(0);
  const isActive = useSharedValue(false);
  const startIndex = useSharedValue(index);

  const panGesture = Gesture.Pan()
    .activateAfterLongPress(200) // Long press to start dragging
    .onStart(() => {
      isActive.value = true;
      startIndex.value = index;
    })
    .onUpdate((event) => {
      translateY.value = event.translationY;
    })
    .onEnd(() => {
      const newIndex = Math.round(
        Math.min(
          Math.max(index + translateY.value / ROW_HEIGHT, 0),
          totalItems - 1
        )
      );

      if (newIndex !== index) {
        runOnJS(onReorder)(index, newIndex);
      }

      translateY.value = withSpring(0, { damping: 20, stiffness: 300 });
      isActive.value = false;
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
    zIndex: isActive.value ? 100 : 0,
    shadowOpacity: isActive.value ? 0.15 : 0,
    backgroundColor: isActive.value ? '#F9FAFB' : '#FFFFFF',
    elevation: isActive.value ? 8 : 0,
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.row, { top: index * ROW_HEIGHT }, animatedStyle]}>
        <Text style={styles.dragHandle}>☰</Text>
        <Text style={styles.rowLabel}>{item.label}</Text>
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  row: {
    position: 'absolute',
    left: 0,
    right: 0,
    height: ROW_HEIGHT,
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#F3F4F6',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowRadius: 8,
  },
  dragHandle: {
    fontSize: 20,
    color: '#9CA3AF',
    marginRight: 16,
  },
  rowLabel: {
    fontSize: 16,
    color: '#111827',
    fontWeight: '500',
  },
});
```

### 9.2 Pinch-to-Zoom Image

```tsx
// components/zoomable-image.tsx
import { Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';
import { Gesture, GestureDetector, GestureHandlerRootView } from 'react-native-gesture-handler';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');

export function ZoomableImage({ uri }: { uri: string }) {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const savedTranslateX = useSharedValue(0);
  const savedTranslateY = useSharedValue(0);
  const focalX = useSharedValue(0);
  const focalY = useSharedValue(0);

  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = savedScale.value * event.scale;
      focalX.value = event.focalX;
      focalY.value = event.focalY;
    })
    .onEnd(() => {
      if (scale.value < 1) {
        scale.value = withSpring(1, { damping: 15, stiffness: 200 });
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        savedScale.value = 1;
        savedTranslateX.value = 0;
        savedTranslateY.value = 0;
      } else if (scale.value > 4) {
        scale.value = withSpring(4, { damping: 15, stiffness: 200 });
        savedScale.value = 4;
      } else {
        savedScale.value = scale.value;
      }
    });

  const panGesture = Gesture.Pan()
    .minPointers(1)
    .maxPointers(2)
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

  const doubleTapGesture = Gesture.Tap()
    .numberOfTaps(2)
    .onEnd((event) => {
      if (scale.value > 1) {
        // Zoom out
        scale.value = withSpring(1, { damping: 15, stiffness: 200 });
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        savedScale.value = 1;
        savedTranslateX.value = 0;
        savedTranslateY.value = 0;
      } else {
        // Zoom in to 2x at tap point
        scale.value = withSpring(2, { damping: 15, stiffness: 200 });
        savedScale.value = 2;
        const offsetX = (SCREEN_WIDTH / 2 - event.x) * 1;
        const offsetY = (SCREEN_HEIGHT / 2 - event.y) * 1;
        translateX.value = withSpring(offsetX);
        translateY.value = withSpring(offsetY);
        savedTranslateX.value = offsetX;
        savedTranslateY.value = offsetY;
      }
    });

  const composedGesture = Gesture.Simultaneous(
    pinchGesture,
    panGesture,
    doubleTapGesture
  );

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
        style={[
          { width: SCREEN_WIDTH, height: SCREEN_HEIGHT },
          animatedStyle,
        ]}
        resizeMode="contain"
      />
    </GestureDetector>
  );
}
```

### 9.3 Swipe Card Stack (Tinder-like)

```tsx
// components/swipe-card-stack.tsx
import { useState } from 'react';
import { View, Text, Dimensions, StyleSheet, Image } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  interpolate,
  runOnJS,
  Extrapolation,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');
const SWIPE_THRESHOLD = SCREEN_WIDTH * 0.35;

interface Card {
  id: string;
  title: string;
  subtitle: string;
  imageUri: string;
}

export function SwipeCardStack({
  cards: initialCards,
  onSwipeLeft,
  onSwipeRight,
}: {
  cards: Card[];
  onSwipeLeft: (card: Card) => void;
  onSwipeRight: (card: Card) => void;
}) {
  const [cards, setCards] = useState(initialCards);

  const handleSwipe = (direction: 'left' | 'right') => {
    const card = cards[0];
    if (!card) return;

    if (direction === 'left') onSwipeLeft(card);
    else onSwipeRight(card);

    setCards((prev) => prev.slice(1));
  };

  return (
    <View style={styles.stackContainer}>
      {cards.slice(0, 3).reverse().map((card, index) => {
        const actualIndex = Math.min(cards.length - 1, 2) - index;
        return (
          <SwipeCard
            key={card.id}
            card={card}
            index={actualIndex}
            isTop={actualIndex === 0}
            onSwipe={handleSwipe}
          />
        );
      })}
    </View>
  );
}

function SwipeCard({
  card,
  index,
  isTop,
  onSwipe,
}: {
  card: Card;
  index: number;
  isTop: boolean;
  onSwipe: (direction: 'left' | 'right') => void;
}) {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .enabled(isTop)
    .onUpdate((event) => {
      translateX.value = event.translationX;
      translateY.value = event.translationY * 0.5; // Dampen vertical movement
    })
    .onEnd((event) => {
      const { translationX, velocityX } = event;

      if (translationX > SWIPE_THRESHOLD || velocityX > 500) {
        translateX.value = withTiming(SCREEN_WIDTH * 1.5, { duration: 300 });
        runOnJS(onSwipe)('right');
        return;
      }

      if (translationX < -SWIPE_THRESHOLD || velocityX < -500) {
        translateX.value = withTiming(-SCREEN_WIDTH * 1.5, { duration: 300 });
        runOnJS(onSwipe)('left');
        return;
      }

      // Snap back
      translateX.value = withSpring(0, { damping: 15, stiffness: 200 });
      translateY.value = withSpring(0, { damping: 15, stiffness: 200 });
    });

  const cardAnimatedStyle = useAnimatedStyle(() => {
    const rotate = interpolate(
      translateX.value,
      [-SCREEN_WIDTH / 2, 0, SCREEN_WIDTH / 2],
      [-15, 0, 15],
      Extrapolation.CLAMP
    );

    // Back cards: scale up as front card moves away
    const backScale = isTop
      ? 1
      : interpolate(
          Math.abs(translateX.value),
          [0, SWIPE_THRESHOLD],
          [1 - index * 0.05, 1 - (index - 1) * 0.05],
          Extrapolation.CLAMP
        );

    const backTranslateY = isTop
      ? translateY.value
      : interpolate(
          Math.abs(translateX.value),
          [0, SWIPE_THRESHOLD],
          [index * 10, (index - 1) * 10],
          Extrapolation.CLAMP
        );

    return {
      transform: [
        { translateX: isTop ? translateX.value : 0 },
        { translateY: backTranslateY },
        { rotate: isTop ? `${rotate}deg` : '0deg' },
        { scale: backScale },
      ],
    };
  });

  // Like / Nope labels
  const likeOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [0, SWIPE_THRESHOLD / 2],
      [0, 1],
      Extrapolation.CLAMP
    ),
  }));

  const nopeOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [-SWIPE_THRESHOLD / 2, 0],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, cardAnimatedStyle]}>
        <Image source={{ uri: card.imageUri }} style={styles.cardImage} />

        {/* "LIKE" stamp */}
        <Animated.View style={[styles.stamp, styles.likeStamp, likeOpacity]}>
          <Text style={[styles.stampText, { color: '#059669' }]}>LIKE</Text>
        </Animated.View>

        {/* "NOPE" stamp */}
        <Animated.View style={[styles.stamp, styles.nopeStamp, nopeOpacity]}>
          <Text style={[styles.stampText, { color: '#DC2626' }]}>NOPE</Text>
        </Animated.View>

        <View style={styles.cardInfo}>
          <Text style={styles.cardTitle}>{card.title}</Text>
          <Text style={styles.cardSubtitle}>{card.subtitle}</Text>
        </View>
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  stackContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  card: {
    position: 'absolute',
    width: SCREEN_WIDTH - 32,
    height: SCREEN_HEIGHT * 0.6,
    borderRadius: 16,
    backgroundColor: '#fff',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.1,
    shadowRadius: 12,
    elevation: 6,
    overflow: 'hidden',
  },
  cardImage: {
    width: '100%',
    height: '75%',
  },
  cardInfo: {
    padding: 16,
  },
  cardTitle: {
    fontSize: 24,
    fontWeight: '700',
    color: '#111827',
  },
  cardSubtitle: {
    fontSize: 16,
    color: '#6B7280',
    marginTop: 4,
  },
  stamp: {
    position: 'absolute',
    top: 40,
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderWidth: 3,
    borderRadius: 8,
  },
  likeStamp: {
    left: 24,
    borderColor: '#059669',
    transform: [{ rotate: '-15deg' }],
  },
  nopeStamp: {
    right: 24,
    borderColor: '#DC2626',
    transform: [{ rotate: '15deg' }],
  },
  stampText: {
    fontSize: 32,
    fontWeight: '900',
    letterSpacing: 2,
  },
});
```

---

## 10. LAYOUT ANIMATIONS

### 10.1 Entering and Exiting Animations

Reanimated provides built-in entering and exiting animations for any component:

```tsx
import Animated, {
  FadeIn,
  FadeOut,
  FadeInDown,
  FadeOutUp,
  SlideInRight,
  SlideOutLeft,
  ZoomIn,
  ZoomOut,
  BounceIn,
  Layout,
} from 'react-native-reanimated';

// Fade in from below when appearing, fade out upward when disappearing
<Animated.View
  entering={FadeInDown.duration(400).springify()}
  exiting={FadeOutUp.duration(200)}
>
  <Text>I animate in and out!</Text>
</Animated.View>

// Zoom in with bounce when appearing
<Animated.View
  entering={BounceIn.delay(200)}
  exiting={ZoomOut.duration(200)}
>
  <Text>Bounce!</Text>
</Animated.View>

// Slide in from right
<Animated.View
  entering={SlideInRight.springify().damping(15)}
  exiting={SlideOutLeft.duration(200)}
>
  <Text>Slide!</Text>
</Animated.View>
```

### 10.2 Staggered List Animations

Make list items appear one after another:

```tsx
// components/staggered-list.tsx
import Animated, { FadeInDown } from 'react-native-reanimated';
import { FlatList, View, Text } from 'react-native';

export function StaggeredList({ data }: { data: any[] }) {
  const renderItem = ({ item, index }: { item: any; index: number }) => (
    <Animated.View
      entering={FadeInDown
        .delay(index * 80) // Each item delayed by 80ms
        .springify()
        .damping(15)
        .stiffness(200)
      }
      style={{
        padding: 16,
        marginBottom: 8,
        backgroundColor: '#fff',
        borderRadius: 12,
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.05,
        shadowRadius: 4,
        elevation: 2,
      }}
    >
      <Text style={{ fontSize: 16, fontWeight: '600' }}>{item.title}</Text>
      <Text style={{ fontSize: 14, color: '#6B7280', marginTop: 4 }}>
        {item.description}
      </Text>
    </Animated.View>
  );

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      contentContainerStyle={{ padding: 16 }}
    />
  );
}
```

### 10.3 Layout Transitions

When items change position (e.g., removing an item from a list), the remaining items should animate to their new positions:

```tsx
import Animated, {
  Layout,
  FadeInDown,
  FadeOutRight,
  LinearTransition,
} from 'react-native-reanimated';

function AnimatedList({
  items,
  onRemove,
}: {
  items: any[];
  onRemove: (id: string) => void;
}) {
  return (
    <View style={{ padding: 16 }}>
      {items.map((item, index) => (
        <Animated.View
          key={item.id}
          entering={FadeInDown.delay(index * 50)}
          exiting={FadeOutRight.duration(200)}
          layout={LinearTransition.springify().damping(15)} // Smooth repositioning
          style={{
            padding: 16,
            marginBottom: 8,
            backgroundColor: '#fff',
            borderRadius: 12,
            flexDirection: 'row',
            justifyContent: 'space-between',
            alignItems: 'center',
          }}
        >
          <Text style={{ fontSize: 16 }}>{item.label}</Text>
          <Pressable onPress={() => onRemove(item.id)}>
            <Text style={{ color: '#DC2626', fontWeight: '600' }}>Remove</Text>
          </Pressable>
        </Animated.View>
      ))}
    </View>
  );
}
```

### 10.4 Accordion Expand/Collapse

```tsx
// components/accordion.tsx
import { useState } from 'react';
import { Pressable, View, Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  interpolate,
  useAnimatedRef,
  measure,
  runOnUI,
} from 'react-native-reanimated';

interface AccordionProps {
  title: string;
  children: React.ReactNode;
}

export function Accordion({ title, children }: AccordionProps) {
  const [expanded, setExpanded] = useState(false);
  const height = useSharedValue(0);
  const rotation = useSharedValue(0);
  const contentRef = useAnimatedRef<Animated.View>();

  const toggleAccordion = () => {
    if (expanded) {
      height.value = withSpring(0, { damping: 15, stiffness: 200 });
      rotation.value = withSpring(0, { damping: 15, stiffness: 200 });
    } else {
      // Measure the content height
      runOnUI(() => {
        'worklet';
        const measurement = measure(contentRef);
        if (measurement) {
          height.value = withSpring(measurement.height, {
            damping: 15,
            stiffness: 200,
          });
        }
      })();
      rotation.value = withSpring(1, { damping: 15, stiffness: 200 });
    }
    setExpanded(!expanded);
  };

  const containerStyle = useAnimatedStyle(() => ({
    height: height.value,
    overflow: 'hidden',
  }));

  const arrowStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${interpolate(rotation.value, [0, 1], [0, 180])}deg` }],
  }));

  return (
    <View style={styles.accordionContainer}>
      <Pressable onPress={toggleAccordion} style={styles.accordionHeader}>
        <Text style={styles.accordionTitle}>{title}</Text>
        <Animated.Text style={[styles.arrow, arrowStyle]}>▼</Animated.Text>
      </Pressable>

      <Animated.View style={containerStyle}>
        <Animated.View ref={contentRef} style={styles.accordionContent}>
          {children}
        </Animated.View>
      </Animated.View>
    </View>
  );
}

const styles = StyleSheet.create({
  accordionContainer: {
    backgroundColor: '#fff',
    borderRadius: 12,
    marginBottom: 8,
    overflow: 'hidden',
    borderWidth: 1,
    borderColor: '#E5E7EB',
  },
  accordionHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
  },
  accordionTitle: {
    fontSize: 16,
    fontWeight: '600',
    color: '#111827',
  },
  arrow: {
    fontSize: 12,
    color: '#6B7280',
  },
  accordionContent: {
    position: 'absolute',
    width: '100%',
    padding: 16,
    paddingTop: 0,
  },
});
```

---

## 11. LOTTIE ANIMATIONS

### 11.1 When to Use Lottie vs Reanimated

| Use Lottie | Use Reanimated |
|-----------|---------------|
| Complex illustrations (characters, scenes) | Interactive / gesture-driven animations |
| Detailed icon animations (checkmark, loading) | Layout transitions (entering, exiting) |
| Onboarding animations | Spring-based UI feedback (bounces, snaps) |
| Celebration effects (confetti, fireworks) | Value-driven animations (scroll-linked) |
| Designer-created animations (After Effects) | Developer-created programmatic animations |

The rule of thumb: if a designer made it in After Effects, use Lottie. If a developer is building interactive motion, use Reanimated.

### 11.2 Setting Up Lottie

```bash
npx expo install lottie-react-native
```

```tsx
// components/lottie-example.tsx
import LottieView from 'lottie-react-native';
import { useRef } from 'react';
import { View, Button } from 'react-native';

export function LottieExample() {
  const animationRef = useRef<LottieView>(null);

  return (
    <View style={{ alignItems: 'center' }}>
      <LottieView
        ref={animationRef}
        source={require('@/assets/animations/success.json')}
        style={{ width: 200, height: 200 }}
        autoPlay={false}
        loop={false}
      />
      <Button
        title="Play"
        onPress={() => animationRef.current?.play()}
      />
    </View>
  );
}
```

### 11.3 Lottie with Scroll Progress

Drive a Lottie animation based on scroll position:

```tsx
import LottieView from 'lottie-react-native';
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedProps,
  interpolate,
} from 'react-native-reanimated';
import { Dimensions, ScrollView, View } from 'react-native';

const AnimatedLottie = Animated.createAnimatedComponent(LottieView);
const { height: SCREEN_HEIGHT } = Dimensions.get('window');

export function ScrollLinkedLottie() {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  const lottieProps = useAnimatedProps(() => ({
    progress: interpolate(
      scrollY.value,
      [0, SCREEN_HEIGHT * 2], // Over 2 screen heights of scroll
      [0, 1]
    ),
  }));

  return (
    <View style={{ flex: 1 }}>
      <AnimatedLottie
        source={require('@/assets/animations/progress.json')}
        animatedProps={lottieProps}
        style={{ width: 100, height: 100, position: 'absolute', top: 50, alignSelf: 'center', zIndex: 10 }}
      />
      <Animated.ScrollView onScroll={scrollHandler} scrollEventThrottle={16}>
        {/* Long scrollable content */}
        <View style={{ height: SCREEN_HEIGHT * 3 }} />
      </Animated.ScrollView>
    </View>
  );
}
```

### 11.4 Where to Get Lottie Files

- [LottieFiles.com](https://lottiefiles.com) -- Largest free library
- [IconScout](https://iconscout.com/lottie-animations) -- Premium animations
- Export from After Effects using the Bodymovin plugin
- Export from Figma using LottieFiles Figma plugin

---

## 12. ANIMATION PERFORMANCE

### 12.1 The Golden Rule

**All interactive animations must run on the UI thread.**

The JS thread handles your React rendering, state updates, and business logic. The UI thread handles native view operations and animations. They run in parallel. If your animation depends on the JS thread, every time the JS thread is busy (re-rendering a list, processing data, running a network callback), your animation will stutter.

Reanimated's worklets run on the UI thread. This is the entire reason Reanimated exists.

### 12.2 What Runs on the UI Thread

| Runs on UI thread (good) | Runs on JS thread (bad) |
|---------------------------|------------------------|
| `useAnimatedStyle` with worklet | `useAnimatedStyle` that reads React state |
| `withSpring`, `withTiming` | `Animated.timing` (old API) |
| Gesture Handler callbacks with `'worklet'` | `onPanResponderMove` |
| Shared values (`useSharedValue`) | `useState` for animation values |
| `runOnUI` code | `useEffect` with `Animated.Value` |

### 12.3 Common Performance Mistakes

**Mistake 1: Reading React state in animated styles**

```tsx
// BAD — runs on JS thread, will jank
const [position, setPosition] = useState(0);
const style = { transform: [{ translateX: position }] };

// GOOD — runs on UI thread
const position = useSharedValue(0);
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{ translateX: position.value }],
}));
```

**Mistake 2: Heavy computation in worklets**

```tsx
// BAD — complex math in every frame
const animatedStyle = useAnimatedStyle(() => {
  // Don't do expensive calculations here
  const result = someExpensiveCalculation(scrollY.value);
  return { transform: [{ translateY: result }] };
});

// GOOD — use interpolate for simple mappings
const animatedStyle = useAnimatedStyle(() => ({
  transform: [{
    translateY: interpolate(scrollY.value, [0, 100], [0, -50]),
  }],
}));
```

**Mistake 3: Too many animated views**

```tsx
// BAD — animating 100 list items simultaneously
{items.map((item, index) => (
  <Animated.View
    key={item.id}
    entering={FadeInDown.delay(index * 50)} // 100 × 50ms = 5 seconds of staggered animations
    // ...
  />
))}

// GOOD — only animate visible items, cap stagger
{items.map((item, index) => (
  <Animated.View
    key={item.id}
    entering={index < 10 ? FadeInDown.delay(index * 50) : undefined} // Only first 10
    // ...
  />
))}
```

### 12.4 Measuring Animation Performance

Use Reanimated's FPS monitor in development:

```tsx
// In your app root during development
import { enableLayoutAnimations } from 'react-native-reanimated';

if (__DEV__) {
  // Enable the performance overlay
  // This shows JS thread FPS and UI thread FPS
  // Both should stay at 60fps during animations
}
```

You can also use React Native's built-in performance monitor:

```tsx
// In development, shake device → "Show Perf Monitor"
// Watch the UI thread FPS during animations
// If it drops below 60fps, your animation is running on the JS thread
```

The key metric: **UI thread FPS should never drop below 55fps during animations.** If it does, something is running expensive computation on the UI thread. If the JS thread drops but the UI thread stays at 60fps, your animations are fine -- the JS thread drop is from React re-renders, which do not affect animation smoothness.

---

## 13. THE COOKBOOK

Here are 15 copy-paste animation recipes. Each one is self-contained and ready to use.

### Recipe 1: Like Button Bounce

```tsx
// components/animations/like-button.tsx
import { useState } from 'react';
import { Pressable, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSequence,
  withSpring,
  withTiming,
} from 'react-native-reanimated';

export function LikeButton({ onLike }: { onLike: (liked: boolean) => void }) {
  const [liked, setLiked] = useState(false);
  const scale = useSharedValue(1);

  const handlePress = () => {
    const newLiked = !liked;
    setLiked(newLiked);
    onLike(newLiked);

    // Bounce sequence: shrink → overshoot → settle
    scale.value = withSequence(
      withTiming(0.7, { duration: 80 }),
      withSpring(1.2, { damping: 4, stiffness: 300 }),
      withSpring(1, { damping: 10, stiffness: 200 })
    );
  };

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Pressable onPress={handlePress}>
      <Animated.Text style={[styles.heart, animatedStyle]}>
        {liked ? '❤️' : '🤍'}
      </Animated.Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  heart: { fontSize: 32 },
});
```

### Recipe 2: FAB Menu Expand

```tsx
// components/animations/fab-menu.tsx
import { useState } from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withTiming,
  interpolate,
} from 'react-native-reanimated';

interface FabAction {
  icon: string;
  label: string;
  onPress: () => void;
}

export function FabMenu({ actions }: { actions: FabAction[] }) {
  const [open, setOpen] = useState(false);
  const progress = useSharedValue(0);

  const toggleMenu = () => {
    setOpen(!open);
    progress.value = withSpring(open ? 0 : 1, { damping: 12, stiffness: 200 });
  };

  const mainButtonStyle = useAnimatedStyle(() => ({
    transform: [{ rotate: `${interpolate(progress.value, [0, 1], [0, 45])}deg` }],
  }));

  return (
    <View style={styles.fabContainer}>
      {/* Action buttons */}
      {actions.map((action, index) => (
        <FabActionButton
          key={index}
          action={action}
          index={index}
          total={actions.length}
          progress={progress}
        />
      ))}

      {/* Main FAB */}
      <Pressable onPress={toggleMenu} style={styles.mainFab}>
        <Animated.Text style={[styles.fabIcon, mainButtonStyle]}>+</Animated.Text>
      </Pressable>
    </View>
  );
}

function FabActionButton({
  action,
  index,
  total,
  progress,
}: {
  action: FabAction;
  index: number;
  total: number;
  progress: Animated.SharedValue<number>;
}) {
  const animatedStyle = useAnimatedStyle(() => {
    const offset = (index + 1) * 64;
    return {
      transform: [
        { translateY: interpolate(progress.value, [0, 1], [0, -offset]) },
        { scale: interpolate(progress.value, [0, 0.5, 1], [0.3, 0.8, 1]) },
      ],
      opacity: progress.value,
    };
  });

  return (
    <Animated.View style={[styles.actionFab, animatedStyle]}>
      <Pressable onPress={action.onPress} style={styles.actionFabButton}>
        <Text style={styles.actionIcon}>{action.icon}</Text>
      </Pressable>
      <Animated.View style={[styles.actionLabel, useAnimatedStyle(() => ({ opacity: progress.value }))]}>
        <Text style={styles.actionLabelText}>{action.label}</Text>
      </Animated.View>
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  fabContainer: {
    position: 'absolute',
    bottom: 32,
    right: 24,
    alignItems: 'center',
  },
  mainFab: {
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#111827',
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.2,
    shadowRadius: 8,
    elevation: 6,
  },
  fabIcon: { color: '#fff', fontSize: 28, fontWeight: '300' },
  actionFab: {
    position: 'absolute',
    bottom: 0,
    flexDirection: 'row',
    alignItems: 'center',
  },
  actionFabButton: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: '#374151',
    justifyContent: 'center',
    alignItems: 'center',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 4,
    elevation: 4,
  },
  actionIcon: { fontSize: 20 },
  actionLabel: {
    position: 'absolute',
    right: 60,
    backgroundColor: '#374151',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 6,
  },
  actionLabelText: { color: '#fff', fontSize: 13, fontWeight: '500' },
});
```

### Recipe 3: Card Flip

```tsx
// components/animations/card-flip.tsx
import { useState } from 'react';
import { View, Pressable, Text, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  interpolate,
  Easing,
} from 'react-native-reanimated';

const { width } = Dimensions.get('window');
const CARD_WIDTH = width - 64;
const CARD_HEIGHT = CARD_WIDTH * 0.6;

export function FlippableCard({
  front,
  back,
}: {
  front: React.ReactNode;
  back: React.ReactNode;
}) {
  const [flipped, setFlipped] = useState(false);
  const rotation = useSharedValue(0);

  const flip = () => {
    setFlipped(!flipped);
    rotation.value = withTiming(flipped ? 0 : 180, {
      duration: 500,
      easing: Easing.inOut(Easing.ease),
    });
  };

  const frontStyle = useAnimatedStyle(() => {
    const rotateY = interpolate(rotation.value, [0, 180], [0, 180]);
    return {
      transform: [{ perspective: 1000 }, { rotateY: `${rotateY}deg` }],
      backfaceVisibility: 'hidden' as const,
      opacity: rotation.value < 90 ? 1 : 0,
    };
  });

  const backStyle = useAnimatedStyle(() => {
    const rotateY = interpolate(rotation.value, [0, 180], [180, 360]);
    return {
      transform: [{ perspective: 1000 }, { rotateY: `${rotateY}deg` }],
      backfaceVisibility: 'hidden' as const,
      opacity: rotation.value >= 90 ? 1 : 0,
    };
  });

  return (
    <Pressable onPress={flip}>
      <View style={styles.flipContainer}>
        <Animated.View style={[styles.flipCard, frontStyle]}>
          {front}
        </Animated.View>
        <Animated.View style={[styles.flipCard, styles.flipCardBack, backStyle]}>
          {back}
        </Animated.View>
      </View>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  flipContainer: {
    width: CARD_WIDTH,
    height: CARD_HEIGHT,
    alignSelf: 'center',
  },
  flipCard: {
    position: 'absolute',
    width: '100%',
    height: '100%',
    borderRadius: 16,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#111827',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.15,
    shadowRadius: 12,
    elevation: 6,
  },
  flipCardBack: {
    backgroundColor: '#1E40AF',
  },
});
```

### Recipe 4: Number Counter

```tsx
// components/animations/number-counter.tsx
import { useEffect } from 'react';
import { Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedProps,
  withTiming,
  useDerivedValue,
} from 'react-native-reanimated';

const AnimatedText = Animated.createAnimatedComponent(Text);

export function AnimatedCounter({
  value,
  duration = 1000,
  prefix = '',
  suffix = '',
  decimals = 0,
}: {
  value: number;
  duration?: number;
  prefix?: string;
  suffix?: string;
  decimals?: number;
}) {
  const animatedValue = useSharedValue(0);

  useEffect(() => {
    animatedValue.value = withTiming(value, { duration });
  }, [value]);

  const displayText = useDerivedValue(() => {
    return `${prefix}${animatedValue.value.toFixed(decimals)}${suffix}`;
  });

  const animatedProps = useAnimatedProps(() => ({
    text: displayText.value,
    defaultValue: displayText.value,
  }));

  return (
    <AnimatedText
      animatedProps={animatedProps}
      style={styles.counter}
    />
  );
}

const styles = StyleSheet.create({
  counter: {
    fontSize: 48,
    fontWeight: '800',
    color: '#111827',
    fontVariant: ['tabular-nums'],
  },
});
```

**Usage:**

```tsx
<AnimatedCounter value={1234} prefix="$" decimals={2} />
<AnimatedCounter value={99} suffix="%" />
```

### Recipe 5: Progress Ring

```tsx
// components/animations/progress-ring.tsx
import { useEffect } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedProps,
  withTiming,
  Easing,
} from 'react-native-reanimated';
import Svg, { Circle } from 'react-native-svg';

const AnimatedCircle = Animated.createAnimatedComponent(Circle);

export function ProgressRing({
  progress, // 0 to 1
  size = 120,
  strokeWidth = 8,
  color = '#111827',
  backgroundColor = '#E5E7EB',
}: {
  progress: number;
  size?: number;
  strokeWidth?: number;
  color?: string;
  backgroundColor?: string;
}) {
  const animatedProgress = useSharedValue(0);
  const radius = (size - strokeWidth) / 2;
  const circumference = 2 * Math.PI * radius;

  useEffect(() => {
    animatedProgress.value = withTiming(progress, {
      duration: 800,
      easing: Easing.out(Easing.cubic),
    });
  }, [progress]);

  const animatedProps = useAnimatedProps(() => ({
    strokeDashoffset: circumference * (1 - animatedProgress.value),
  }));

  const percentage = Math.round(progress * 100);

  return (
    <View style={{ width: size, height: size, justifyContent: 'center', alignItems: 'center' }}>
      <Svg width={size} height={size}>
        {/* Background circle */}
        <Circle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          stroke={backgroundColor}
          strokeWidth={strokeWidth}
          fill="none"
        />
        {/* Progress circle */}
        <AnimatedCircle
          cx={size / 2}
          cy={size / 2}
          r={radius}
          stroke={color}
          strokeWidth={strokeWidth}
          fill="none"
          strokeDasharray={circumference}
          animatedProps={animatedProps}
          strokeLinecap="round"
          transform={`rotate(-90, ${size / 2}, ${size / 2})`}
        />
      </Svg>
      <Text style={[styles.percentageText, { position: 'absolute' }]}>
        {percentage}%
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  percentageText: {
    fontSize: 24,
    fontWeight: '700',
    color: '#111827',
  },
});
```

### Recipe 6: Typing Indicator

```tsx
// components/animations/typing-indicator.tsx
import { useEffect } from 'react';
import { View, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withSequence,
  withTiming,
  withDelay,
} from 'react-native-reanimated';

export function TypingIndicator() {
  const dot1 = useSharedValue(0);
  const dot2 = useSharedValue(0);
  const dot3 = useSharedValue(0);

  useEffect(() => {
    const animation = withRepeat(
      withSequence(
        withTiming(-6, { duration: 300 }),
        withTiming(0, { duration: 300 })
      ),
      -1
    );

    dot1.value = animation;
    dot2.value = withDelay(150, animation);
    dot3.value = withDelay(300, animation);
  }, []);

  const style1 = useAnimatedStyle(() => ({
    transform: [{ translateY: dot1.value }],
  }));
  const style2 = useAnimatedStyle(() => ({
    transform: [{ translateY: dot2.value }],
  }));
  const style3 = useAnimatedStyle(() => ({
    transform: [{ translateY: dot3.value }],
  }));

  return (
    <View style={styles.container}>
      <Animated.View style={[styles.dot, style1]} />
      <Animated.View style={[styles.dot, style2]} />
      <Animated.View style={[styles.dot, style3]} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 4,
    backgroundColor: '#F3F4F6',
    paddingHorizontal: 16,
    paddingVertical: 12,
    borderRadius: 20,
    alignSelf: 'flex-start',
  },
  dot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    backgroundColor: '#9CA3AF',
  },
});
```

### Recipe 7: Confetti Burst

```tsx
// components/animations/confetti.tsx
import { useEffect } from 'react';
import { View, Dimensions, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withDelay,
  withSequence,
  Easing,
  runOnJS,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('window');
const CONFETTI_COUNT = 30;
const COLORS = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#FDCB6E', '#6C5CE7', '#A8E6CE'];

function randomBetween(min: number, max: number) {
  return Math.random() * (max - min) + min;
}

function ConfettiPiece({ index, onFinish }: { index: number; onFinish?: () => void }) {
  const translateX = useSharedValue(SCREEN_WIDTH / 2);
  const translateY = useSharedValue(SCREEN_HEIGHT / 2);
  const rotation = useSharedValue(0);
  const opacity = useSharedValue(1);
  const scale = useSharedValue(0);

  const color = COLORS[index % COLORS.length];
  const size = randomBetween(6, 12);
  const isSquare = Math.random() > 0.5;

  useEffect(() => {
    const targetX = randomBetween(-SCREEN_WIDTH * 0.3, SCREEN_WIDTH * 1.3);
    const targetY = randomBetween(SCREEN_HEIGHT * 0.6, SCREEN_HEIGHT * 1.2);
    const duration = randomBetween(1500, 2500);
    const delay = randomBetween(0, 300);

    scale.value = withDelay(delay, withSequence(
      withTiming(1, { duration: 100 }),
      withDelay(duration - 500, withTiming(0, { duration: 400 }))
    ));

    translateX.value = withDelay(delay, withTiming(targetX, {
      duration,
      easing: Easing.out(Easing.quad),
    }));

    translateY.value = withDelay(delay, withTiming(targetY, {
      duration,
      easing: Easing.in(Easing.quad),
    }));

    rotation.value = withDelay(delay, withTiming(
      randomBetween(360, 720) * (Math.random() > 0.5 ? 1 : -1),
      { duration, easing: Easing.linear }
    ));

    opacity.value = withDelay(delay + duration - 400, withTiming(0, { duration: 400 }));
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { rotate: `${rotation.value}deg` },
      { scale: scale.value },
    ],
    opacity: opacity.value,
  }));

  return (
    <Animated.View
      style={[
        {
          position: 'absolute',
          width: size,
          height: isSquare ? size : size * 0.4,
          backgroundColor: color,
          borderRadius: isSquare ? 2 : size,
        },
        animatedStyle,
      ]}
    />
  );
}

export function Confetti({ onComplete }: { onComplete?: () => void }) {
  useEffect(() => {
    if (onComplete) {
      const timer = setTimeout(onComplete, 3000);
      return () => clearTimeout(timer);
    }
  }, [onComplete]);

  return (
    <View style={StyleSheet.absoluteFill} pointerEvents="none">
      {Array.from({ length: CONFETTI_COUNT }).map((_, i) => (
        <ConfettiPiece key={i} index={i} />
      ))}
    </View>
  );
}
```

**Usage:**

```tsx
const [showConfetti, setShowConfetti] = useState(false);

// Trigger on success
{showConfetti && <Confetti onComplete={() => setShowConfetti(false)} />}
```

### Recipe 8: Parallax Scroll Header

```tsx
// components/animations/parallax-header.tsx
import { View, Text, Dimensions, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useAnimatedScrollHandler,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

const { width: SCREEN_WIDTH } = Dimensions.get('window');
const HEADER_HEIGHT = 300;
const COLLAPSED_HEIGHT = 100;

export function ParallaxHeader({
  imageUri,
  title,
  children,
}: {
  imageUri: string;
  title: string;
  children: React.ReactNode;
}) {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  const headerImageStyle = useAnimatedStyle(() => {
    const translateY = interpolate(
      scrollY.value,
      [-HEADER_HEIGHT, 0, HEADER_HEIGHT],
      [-HEADER_HEIGHT / 2, 0, HEADER_HEIGHT * 0.5], // Parallax effect
      Extrapolation.CLAMP
    );

    const scale = interpolate(
      scrollY.value,
      [-HEADER_HEIGHT, 0],
      [2, 1],
      Extrapolation.CLAMP
    );

    return {
      transform: [{ translateY }, { scale }],
    };
  });

  const titleStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT - COLLAPSED_HEIGHT],
      [1, 0],
      Extrapolation.CLAMP
    );

    const translateY = interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT - COLLAPSED_HEIGHT],
      [0, -30],
      Extrapolation.CLAMP
    );

    return { opacity, transform: [{ translateY }] };
  });

  const collapsedTitleStyle = useAnimatedStyle(() => {
    const opacity = interpolate(
      scrollY.value,
      [HEADER_HEIGHT - COLLAPSED_HEIGHT - 20, HEADER_HEIGHT - COLLAPSED_HEIGHT],
      [0, 1],
      Extrapolation.CLAMP
    );

    return { opacity };
  });

  return (
    <View style={{ flex: 1 }}>
      {/* Collapsed header (sticky) */}
      <Animated.View style={[styles.collapsedHeader, collapsedTitleStyle]}>
        <Text style={styles.collapsedTitle}>{title}</Text>
      </Animated.View>

      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        contentContainerStyle={{ paddingTop: HEADER_HEIGHT }}
      >
        {children}
      </Animated.ScrollView>

      {/* Parallax image header */}
      <View style={[styles.headerContainer, { height: HEADER_HEIGHT }]}>
        <Animated.Image
          source={{ uri: imageUri }}
          style={[styles.headerImage, headerImageStyle]}
        />
        <Animated.View style={[styles.headerTitleContainer, titleStyle]}>
          <Text style={styles.headerTitle}>{title}</Text>
        </Animated.View>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  headerContainer: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    overflow: 'hidden',
  },
  headerImage: {
    width: SCREEN_WIDTH,
    height: HEADER_HEIGHT,
    resizeMode: 'cover',
  },
  headerTitleContainer: {
    position: 'absolute',
    bottom: 24,
    left: 24,
    right: 24,
  },
  headerTitle: {
    fontSize: 32,
    fontWeight: '800',
    color: '#fff',
    textShadowColor: 'rgba(0,0,0,0.5)',
    textShadowOffset: { width: 0, height: 2 },
    textShadowRadius: 8,
  },
  collapsedHeader: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    height: COLLAPSED_HEIGHT,
    backgroundColor: '#fff',
    justifyContent: 'flex-end',
    paddingHorizontal: 24,
    paddingBottom: 12,
    zIndex: 100,
    borderBottomWidth: 1,
    borderBottomColor: '#E5E7EB',
  },
  collapsedTitle: {
    fontSize: 18,
    fontWeight: '700',
    color: '#111827',
  },
});
```

### Recipe 9: Sticky Header Collapse

```tsx
// components/animations/collapsing-header.tsx
import { View, Text, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useAnimatedScrollHandler,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

const EXPANDED_HEIGHT = 180;
const COLLAPSED_HEIGHT = 64;
const SCROLL_DISTANCE = EXPANDED_HEIGHT - COLLAPSED_HEIGHT;

export function CollapsingHeader({
  title,
  subtitle,
  children,
}: {
  title: string;
  subtitle: string;
  children: React.ReactNode;
}) {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  const headerStyle = useAnimatedStyle(() => {
    const height = interpolate(
      scrollY.value,
      [0, SCROLL_DISTANCE],
      [EXPANDED_HEIGHT, COLLAPSED_HEIGHT],
      Extrapolation.CLAMP
    );

    return { height };
  });

  const subtitleStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      scrollY.value,
      [0, SCROLL_DISTANCE * 0.5],
      [1, 0],
      Extrapolation.CLAMP
    ),
    transform: [{
      translateY: interpolate(
        scrollY.value,
        [0, SCROLL_DISTANCE * 0.5],
        [0, -10],
        Extrapolation.CLAMP
      ),
    }],
  }));

  const titleFontSize = useAnimatedStyle(() => {
    const fontSize = interpolate(
      scrollY.value,
      [0, SCROLL_DISTANCE],
      [32, 18],
      Extrapolation.CLAMP
    );

    return { fontSize };
  });

  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.header, headerStyle]}>
        <Animated.Text style={[styles.title, titleFontSize]}>
          {title}
        </Animated.Text>
        <Animated.Text style={[styles.subtitle, subtitleStyle]}>
          {subtitle}
        </Animated.Text>
      </Animated.View>

      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        contentContainerStyle={{ paddingTop: EXPANDED_HEIGHT }}
      >
        {children}
      </Animated.ScrollView>
    </View>
  );
}

const styles = StyleSheet.create({
  header: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    backgroundColor: '#fff',
    justifyContent: 'flex-end',
    paddingHorizontal: 24,
    paddingBottom: 12,
    zIndex: 100,
    borderBottomWidth: 1,
    borderBottomColor: '#E5E7EB',
  },
  title: {
    fontWeight: '800',
    color: '#111827',
  },
  subtitle: {
    fontSize: 15,
    color: '#6B7280',
    marginTop: 4,
  },
});
```

### Recipe 10: Image Zoom Viewer

```tsx
// components/animations/image-zoom.tsx
// See Section 9.2 — ZoomableImage component above
// (Pinch-to-zoom with double-tap and pan)
```

### Recipe 11: Drag Handle Indicator

```tsx
// components/animations/drag-handle.tsx
import { View, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withSequence,
  withTiming,
  withDelay,
} from 'react-native-reanimated';
import { useEffect } from 'react';

export function DragHandle({ animate = true }: { animate?: boolean }) {
  const translateY = useSharedValue(0);

  useEffect(() => {
    if (animate) {
      translateY.value = withDelay(
        1000,
        withRepeat(
          withSequence(
            withTiming(-4, { duration: 400 }),
            withTiming(4, { duration: 400 }),
            withTiming(0, { duration: 200 })
          ),
          3, // Repeat 3 times then stop — just a hint
          false
        )
      );
    }
  }, [animate]);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.View style={[styles.handle, animatedStyle]}>
      <View style={styles.handleBar} />
    </Animated.View>
  );
}

const styles = StyleSheet.create({
  handle: {
    width: '100%',
    alignItems: 'center',
    paddingVertical: 12,
  },
  handleBar: {
    width: 40,
    height: 4,
    borderRadius: 2,
    backgroundColor: '#D1D5DB',
  },
});
```

### Recipe 12: Loading Dots

```tsx
// components/animations/loading-dots.tsx
import { View, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  withDelay,
  withSequence,
} from 'react-native-reanimated';
import { useEffect } from 'react';

export function LoadingDots({
  color = '#111827',
  size = 10,
  count = 3,
}: {
  color?: string;
  size?: number;
  count?: number;
}) {
  return (
    <View style={styles.container}>
      {Array.from({ length: count }).map((_, i) => (
        <Dot key={i} index={i} color={color} size={size} />
      ))}
    </View>
  );
}

function Dot({ index, color, size }: { index: number; color: string; size: number }) {
  const opacity = useSharedValue(0.3);
  const scale = useSharedValue(0.8);

  useEffect(() => {
    opacity.value = withDelay(
      index * 200,
      withRepeat(
        withSequence(
          withTiming(1, { duration: 400 }),
          withTiming(0.3, { duration: 400 })
        ),
        -1
      )
    );

    scale.value = withDelay(
      index * 200,
      withRepeat(
        withSequence(
          withTiming(1, { duration: 400 }),
          withTiming(0.8, { duration: 400 })
        ),
        -1
      )
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [{ scale: scale.value }],
  }));

  return (
    <Animated.View
      style={[
        {
          width: size,
          height: size,
          borderRadius: size / 2,
          backgroundColor: color,
          marginHorizontal: size * 0.3,
        },
        animatedStyle,
      ]}
    />
  );
}

const styles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'center',
  },
});
```

### Recipe 13: Success Checkmark

```tsx
// components/animations/success-checkmark.tsx
import { useEffect } from 'react';
import { View, StyleSheet } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSequence,
  withTiming,
  withSpring,
  withDelay,
  interpolate,
  Easing,
} from 'react-native-reanimated';
import Svg, { Circle, Path } from 'react-native-svg';

const AnimatedCircle = Animated.createAnimatedComponent(Circle);
const AnimatedPath = Animated.createAnimatedComponent(Path);

export function SuccessCheckmark({ size = 80 }: { size?: number }) {
  const circleProgress = useSharedValue(0);
  const checkProgress = useSharedValue(0);
  const containerScale = useSharedValue(0);

  const circleCircumference = 2 * Math.PI * (size / 2 - 4);
  const checkPath = `M${size * 0.25} ${size * 0.5} L${size * 0.45} ${size * 0.65} L${size * 0.75} ${size * 0.35}`;
  const checkLength = size * 0.7; // Approximate

  useEffect(() => {
    // 1. Scale in the container
    containerScale.value = withSpring(1, { damping: 8, stiffness: 200 });

    // 2. Draw the circle
    circleProgress.value = withDelay(
      200,
      withTiming(1, { duration: 500, easing: Easing.out(Easing.cubic) })
    );

    // 3. Draw the checkmark
    checkProgress.value = withDelay(
      600,
      withTiming(1, { duration: 400, easing: Easing.out(Easing.cubic) })
    );
  }, []);

  const containerStyle = useAnimatedStyle(() => ({
    transform: [{ scale: containerScale.value }],
  }));

  const circleProps = useAnimatedStyle(() => ({
    strokeDashoffset: circleCircumference * (1 - circleProgress.value),
  }));

  const checkProps = useAnimatedStyle(() => ({
    strokeDashoffset: checkLength * (1 - checkProgress.value),
  }));

  return (
    <Animated.View style={[{ width: size, height: size }, containerStyle]}>
      <Svg width={size} height={size}>
        {/* Background circle */}
        <Circle
          cx={size / 2}
          cy={size / 2}
          r={size / 2 - 4}
          fill="#ECFDF5"
          stroke="#D1FAE5"
          strokeWidth={2}
        />
        {/* Animated circle border */}
        <AnimatedCircle
          cx={size / 2}
          cy={size / 2}
          r={size / 2 - 4}
          fill="none"
          stroke="#059669"
          strokeWidth={3}
          strokeDasharray={circleCircumference}
          animatedProps={circleProps}
          strokeLinecap="round"
          transform={`rotate(-90, ${size / 2}, ${size / 2})`}
        />
        {/* Animated checkmark */}
        <AnimatedPath
          d={checkPath}
          fill="none"
          stroke="#059669"
          strokeWidth={3}
          strokeDasharray={checkLength}
          animatedProps={checkProps}
          strokeLinecap="round"
          strokeLinejoin="round"
        />
      </Svg>
    </Animated.View>
  );
}
```

### Recipe 14: Shake on Error

```tsx
// components/animations/shake.tsx
import { useCallback } from 'react';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSequence,
  withTiming,
} from 'react-native-reanimated';

export function useShakeAnimation() {
  const translateX = useSharedValue(0);

  const shake = useCallback(() => {
    translateX.value = withSequence(
      withTiming(10, { duration: 50 }),
      withTiming(-10, { duration: 50 }),
      withTiming(8, { duration: 50 }),
      withTiming(-8, { duration: 50 }),
      withTiming(4, { duration: 50 }),
      withTiming(-4, { duration: 50 }),
      withTiming(0, { duration: 50 })
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return { shake, animatedStyle };
}

// Usage:
function LoginForm() {
  const { shake, animatedStyle } = useShakeAnimation();

  const handleLogin = async () => {
    try {
      await login(email, password);
    } catch {
      shake(); // Shake the form on error
    }
  };

  return (
    <Animated.View style={animatedStyle}>
      {/* Your form fields */}
    </Animated.View>
  );
}
```

### Recipe 15: Slide-In Notification Banner

```tsx
// components/animations/notification-banner.tsx
import { useEffect } from 'react';
import { View, Text, Pressable, StyleSheet, Dimensions } from 'react-native';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  withDelay,
  withTiming,
  runOnJS,
} from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const { width: SCREEN_WIDTH } = Dimensions.get('window');

interface NotificationBannerProps {
  title: string;
  message: string;
  onPress?: () => void;
  onDismiss?: () => void;
  duration?: number;
}

export function NotificationBanner({
  title,
  message,
  onPress,
  onDismiss,
  duration = 4000,
}: NotificationBannerProps) {
  const translateY = useSharedValue(-120);
  const translateX = useSharedValue(0);
  const opacity = useSharedValue(1);

  useEffect(() => {
    // Slide in
    translateY.value = withSpring(0, { damping: 15, stiffness: 200 });

    // Auto-dismiss
    const timer = setTimeout(() => {
      dismiss();
    }, duration);

    return () => clearTimeout(timer);
  }, []);

  const dismiss = () => {
    translateY.value = withTiming(-120, { duration: 200 });
    opacity.value = withTiming(0, { duration: 200 });
    if (onDismiss) {
      setTimeout(onDismiss, 200);
    }
  };

  // Swipe up to dismiss
  const swipeGesture = Gesture.Pan()
    .onUpdate((event) => {
      if (event.translationY < 0) {
        translateY.value = event.translationY;
      }
      if (Math.abs(event.translationX) > 0) {
        translateX.value = event.translationX;
      }
    })
    .onEnd((event) => {
      if (event.translationY < -30 || event.velocityY < -500) {
        // Dismiss upward
        translateY.value = withTiming(-120, { duration: 200 });
        if (onDismiss) runOnJS(onDismiss)();
        return;
      }
      if (Math.abs(event.translationX) > SCREEN_WIDTH * 0.3) {
        // Dismiss sideways
        const direction = event.translationX > 0 ? SCREEN_WIDTH : -SCREEN_WIDTH;
        translateX.value = withTiming(direction, { duration: 200 });
        if (onDismiss) runOnJS(onDismiss)();
        return;
      }
      // Snap back
      translateY.value = withSpring(0, { damping: 15, stiffness: 200 });
      translateX.value = withSpring(0, { damping: 15, stiffness: 200 });
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateY: translateY.value },
      { translateX: translateX.value },
    ],
    opacity: opacity.value,
  }));

  return (
    <GestureDetector gesture={swipeGesture}>
      <Animated.View style={[styles.banner, animatedStyle]}>
        <Pressable onPress={onPress} style={styles.bannerContent}>
          <View style={styles.bannerTextContainer}>
            <Text style={styles.bannerTitle} numberOfLines={1}>
              {title}
            </Text>
            <Text style={styles.bannerMessage} numberOfLines={2}>
              {message}
            </Text>
          </View>
          <Text style={styles.bannerDismiss}>×</Text>
        </Pressable>
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  banner: {
    position: 'absolute',
    top: 50,
    left: 16,
    right: 16,
    backgroundColor: '#111827',
    borderRadius: 14,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 8 },
    shadowOpacity: 0.2,
    shadowRadius: 16,
    elevation: 8,
    zIndex: 9999,
  },
  bannerContent: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
  },
  bannerTextContainer: {
    flex: 1,
  },
  bannerTitle: {
    color: '#fff',
    fontSize: 15,
    fontWeight: '600',
    marginBottom: 2,
  },
  bannerMessage: {
    color: 'rgba(255,255,255,0.7)',
    fontSize: 13,
    lineHeight: 18,
  },
  bannerDismiss: {
    color: 'rgba(255,255,255,0.5)',
    fontSize: 24,
    fontWeight: '300',
    marginLeft: 12,
  },
});
```

---

## Key Takeaways

1. **Animate everything with purpose.** Every state change and user interaction should have visual feedback. Not flashy -- purposeful.

2. **Use springs by default.** `withSpring` feels natural because it models physics. `withTiming` is for opacity, progress, and rotation loops.

3. **Define spring presets.** Create a `springs` config object with named presets (snappy, bouncy, gentle) and use them consistently across your app.

4. **Shared element transitions are the highest-impact animation.** A photo grid that morphs into a detail view makes your app feel ten times more polished.

5. **Skeletons beat spinners.** Content-shaped placeholders reduce perceived load time. Match the shape of your actual content, not generic rectangles.

6. **Use @gorhom/bottom-sheet.** Do not build your own. Use `BottomSheetFlatList` and `BottomSheetTextInput` inside sheets.

7. **Keep animations on the UI thread.** Use `useSharedValue`, `useAnimatedStyle`, and worklets. Never use `useState` for animation values.

8. **Cap staggered animations.** Do not animate 100 list items sequentially. Animate the first 10, then let the rest appear immediately.

9. **Measure performance.** Watch UI thread FPS during animations. If it drops below 55fps, something is wrong.

10. **The cookbook recipes are starting points.** Copy them, then tune the spring configs to match your app's personality. The difference between "generic" and "polished" is in the spring parameters.

---

*Next chapter: We will cover offline support, caching strategies, and building apps that work without a network connection -- because the best micro-interaction in the world means nothing if your app shows a blank screen in an elevator.*
