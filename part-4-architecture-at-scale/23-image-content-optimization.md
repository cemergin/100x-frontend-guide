<!--
  CHAPTER: 23
  TITLE: Image & Content Optimization — Every Screen, Every Size
  PART: IV — Architecture at Scale
  PREREQS: Chapters 9, 11
  KEY_TOPICS: responsive images, expo-image, next/image, image CDN, Cloudinary, Imgix, srcset, pixel density, WebP, AVIF, blurhash, thumbhash, progressive loading, responsive layouts, breakpoints, safe areas, foldables, tablets, dynamic type, content adaptation, responsive typography, adaptive components
  DIFFICULTY: Intermediate → Advanced
  UPDATED: 2026-04-07
-->

# Chapter 23: Image & Content Optimization — Every Screen, Every Size

> **Part IV — Architecture at Scale** | Prerequisites: Chapters 9, 11 | Difficulty: Intermediate → Advanced

<details>
<summary><strong>TL;DR</strong></summary>

- A single unoptimized 4000x3000 photo consumes 48MB of decoded RAM; load ten in a feed and you hit 480MB -- instant OOM kill on most Android devices; image optimization is not optional, it is survival
- WebP is the default image format for mobile in 2026 (25-34% smaller than JPEG at equivalent quality); AVIF offers 50% savings but encoding is slow and device support is still catching up
- expo-image (built on SDWebImage + Glide) replaces the legacy Image and FastImage components; it handles format negotiation, disk/memory caching, blurhash placeholders, and recycling in FlashList out of the box
- next/image automates format conversion, srcset generation, and lazy loading for web; pair it with a CDN (Cloudinary, Imgix, Cloudflare Images) when you need art direction or advanced transformations
- Responsive layouts require purpose-built breakpoint systems (useWindowDimensions + breakpoint hooks for RN, NativeWind/Tamagui media queries) and explicit handling for safe areas, foldables, tablets, and Dynamic Type
- Build a single universal media system -- one OptimizedImage component, one CDN URL builder, one placeholder pipeline -- that works identically across React Native and Next.js

</details>

I want to tell you about the production incident that taught me more about image optimization than any conference talk ever could.

We shipped a social feed feature. Beautiful cards, full-bleed cover images, smooth scroll. On our development iPhones, it was gorgeous. On a Samsung Galaxy A14 with 3GB of RAM in Lagos, the app was force-killed by Android's Low Memory Killer within 40 seconds of scrolling.

The post-mortem was brutal: our API returned full-resolution photos (most were 4000x3000 from iPhone 15 cameras). We displayed them in 375x200 point cards. Every image was decoded to a 48MB bitmap in memory. Ten images in the viewport meant 480MB of decoded image data -- on a device where the entire app budget was 200MB.

The fix took two weeks. The architectural lesson should have taken two minutes: **images are the single largest consumer of memory, bandwidth, and rendering time in any content-rich application.** Getting them wrong doesn't degrade the experience -- it kills the app. Literally.

This chapter covers everything you need to build a media system that works on every screen, every pixel density, every connection speed, and every device category -- from a $50 Android phone on 3G to a 27-inch Retina iMac on fiber.

### In This Chapter
- The Image Problem on Mobile (the math that explains OOM crashes)
- Image Formats: JPEG, PNG, WebP, AVIF, SVG, and Animated Formats
- expo-image for React Native
- next/image for Web
- Image CDN Services (Cloudinary, Imgix, Cloudflare, Vercel)
- Responsive Images by Pixel Density
- Responsive Layouts for Mobile (phones, tablets, foldables)
- Responsive Typography (Dynamic Type, font scaling, fluid type)
- Adaptive Content (different content per device)
- Lazy Loading and Progressive Rendering
- Video and Rich Media
- Building a Universal Media System

### Related Chapters
- [Ch 9: React Native Core Concepts] -- understanding the bridge and native modules
- [Ch 11: Navigation & Routing] -- screen-level layout considerations
- [Ch 13: Performance Optimization] -- memory management and profiling tools
- [Ch 16: Design Systems] -- responsive tokens and component variants

---

## 1. THE IMAGE PROBLEM ON MOBILE

Before we talk about solutions, you need to understand the problem at a physics level. This isn't about "best practices" -- it's about understanding why an unoptimized image can crash your app.

### The Memory Math

When a device displays an image, it must decode the compressed file (JPEG, PNG, WebP) into a raw bitmap in memory. Every pixel requires 4 bytes (one each for red, green, blue, and alpha channels).

Here is the formula:

```
Memory (bytes) = width × height × 4

A 4000×3000 photo:
4000 × 3000 × 4 = 48,000,000 bytes = 48 MB

A 6000×4000 photo (iPhone 15 Pro Max):
6000 × 4000 × 4 = 96,000,000 bytes = 96 MB
```

The compressed file size (the JPEG on disk) is irrelevant to memory consumption. A 2MB JPEG that is 4000x3000 pixels still decodes to 48MB in RAM. The file size only affects network transfer time; the decoded size is what kills your app.

Now extrapolate to a scrolling feed:

```
Scenario: Social feed with 10 visible images

Unoptimized (4000×3000 source images):
10 images × 48 MB = 480 MB decoded memory

Optimized (750×500, matching 375×250 display at 2x):
10 images × 750 × 500 × 4 = 15 MB decoded memory

That's a 32x reduction in memory usage.
```

### The Device Reality in 2026

| Device Category | RAM | Available to App | Image Budget |
|----------------|-----|-----------------|--------------|
| Low-end Android (Galaxy A14, Redmi 12C) | 3-4 GB | ~600 MB | ~100 MB |
| Mid-range Android (Pixel 8a, Galaxy A55) | 6-8 GB | ~1.5 GB | ~300 MB |
| High-end Android (Pixel 9 Pro, Galaxy S25) | 12-16 GB | ~3 GB | ~600 MB |
| iPhone SE (3rd gen) | 4 GB | ~1.2 GB | ~250 MB |
| iPhone 15 | 6 GB | ~2 GB | ~400 MB |
| iPhone 15 Pro Max | 8 GB | ~3 GB | ~600 MB |
| iPad Pro M4 | 16 GB | ~6 GB | ~1.2 GB |

The "Image Budget" column is roughly 20% of available app memory -- a reasonable ceiling for images given that your app also needs memory for JS runtime, native views, navigation stacks, network caches, and everything else.

On a low-end Android device with a 100MB image budget, you can hold **two** unoptimized 4000x3000 photos in memory before you're in the danger zone. Two photos. In a social feed. That's not a performance problem -- that's an app that doesn't work.

### The Bandwidth Math

Images are also the largest contributor to network transfer. On a typical content page:

```
Average web page (2026):     ~2.5 MB total
Images contribution:         ~1.5 MB (60%)
JavaScript contribution:     ~500 KB (20%)
Everything else:             ~500 KB (20%)

Mobile app API response with 20 feed items:
Unoptimized (2MB JPEGs):    40 MB
Optimized (100KB WebPs):     2 MB
```

For users on metered connections (most of the world), that 38MB difference is real money. In India, 1GB of mobile data costs roughly $0.17 (2026). Serving unoptimized images costs your users about $0.006 per feed load. That adds up to several dollars per month for an active user -- meaningful in markets where the average smartphone costs $120.

### The Rendering Pipeline Impact

Decoding a large image blocks the main thread (or the decoder thread, which then delays composition):

```
Image decoding timeline (4000×3000 JPEG):
┌─────────────────────────────────────────────────┐
│ Network fetch:  800ms (2MB on fast 4G)          │
│ JPEG decode:    120-250ms (CPU-bound)           │
│ Resize/scale:   50-100ms                        │
│ Compositing:    10-20ms                         │
│ Total:          ~1000-1200ms                    │
└─────────────────────────────────────────────────┘

Image decoding timeline (750×500 WebP):
┌─────────────────────────────────────────────────┐
│ Network fetch:  50ms (80KB on fast 4G)          │
│ WebP decode:    8-15ms                          │
│ Compositing:    2-5ms                           │
│ Total:          ~60-70ms                        │
└─────────────────────────────────────────────────┘
```

During that 120-250ms decode of a large JPEG, your scroll animation is frozen. Users see jank. And if you're decoding multiple images simultaneously (as happens when scrolling fast), you can saturate the CPU and freeze the entire app for perceptible durations.

**The takeaway is simple: serve the smallest image that looks good at the displayed size and pixel density. Everything in this chapter flows from that principle.**

---

## 2. IMAGE FORMATS

Choosing the right format is the first and highest-leverage optimization. Each format exists for a reason, and using the wrong one costs you 2-10x in file size.

### 2.1 JPEG

The workhorse of the internet since 1992. Lossy compression optimized for photographs.

**Strengths:**
- Universal support (every device, every browser, every image viewer)
- Excellent compression for photographs (complex scenes with gradients)
- Adjustable quality: 60-80 is the sweet spot for most photos (80% of quality, 30% of file size vs. q100)
- Progressive JPEG: renders a blurry preview first, then sharpens (great perceived performance)

**Weaknesses:**
- No transparency support
- Lossy only (no pixel-perfect mode)
- Compression artifacts visible on sharp edges, text, and flat-color areas
- Not great for graphics, logos, or illustrations

**When to use:** Photographs, complex real-world images, hero images, profile photos, product shots.

```
Quality guide (JPEG):
q100: 2.4 MB   — indistinguishable from original, overkill
q85:  680 KB   — nearly indistinguishable, good for print
q75:  420 KB   — excellent for screens, the sweet spot
q60:  280 KB   — noticeable softening on close inspection
q40:  180 KB   — visible artifacts, thumbnails only
```

### 2.2 PNG

Lossless compression, supports transparency (alpha channel).

**Strengths:**
- Pixel-perfect reproduction (no compression artifacts)
- Full alpha transparency (not just on/off, but graduated)
- Excellent for graphics with sharp edges, text, and flat colors
- Wide support

**Weaknesses:**
- File sizes 5-10x larger than JPEG for photographs
- No lossy mode (you cannot trade quality for size)
- Overkill for photographs

**When to use:** Icons with transparency, illustrations, screenshots with text, UI elements, any image where you need exact pixel reproduction.

### 2.3 WebP

Developed by Google, WebP is the **default format for mobile in 2026.** It supports both lossy and lossless compression, animation, and alpha transparency.

**Strengths:**
- 25-34% smaller than JPEG at equivalent visual quality (lossy mode)
- 26% smaller than PNG (lossless mode)
- Supports alpha transparency (unlike JPEG)
- Supports animation (unlike JPEG)
- Near-universal support: all modern browsers, iOS 14+, Android 4.2+
- The best balance of compression, quality, and compatibility in 2026

**Weaknesses:**
- Slightly slower to encode than JPEG (negligible with CDN)
- Slightly more CPU to decode than JPEG (negligible on modern hardware)
- Lossy WebP at very low quality can produce different artifacts than JPEG (blurring vs. blocking)

**When to use:** Everything. WebP is the default for mobile. Use it for photos, illustrations, icons, thumbnails. Fall back to JPEG/PNG only when you need to support ancient devices.

```
File size comparison (same source, same visual quality):
JPEG q75:   420 KB
WebP q75:   280 KB  (33% smaller)
AVIF q75:   180 KB  (57% smaller than JPEG)
```

### 2.4 AVIF

Based on the AV1 video codec, AVIF offers the best compression ratios available. It is the future, but with caveats.

**Strengths:**
- 50% smaller than JPEG at equivalent visual quality
- Superior to WebP in compression efficiency
- Supports 10-bit and 12-bit color depth (HDR)
- Supports alpha transparency
- Handles both photographs and graphics well

**Weaknesses:**
- **Slow to encode:** 10-50x slower than JPEG encoding; CDN encoding pipelines handle this, but real-time encoding is impractical
- **Growing but incomplete device support:** Safari 16.4+, Chrome 85+, Firefox 93+, Android 12+, iOS 16.4+; missing on older devices
- **Slow to decode on lower-end hardware:** can cause jank on budget phones
- **Maximum image dimensions:** some implementations cap at 8192x8192

**When to use:** High-traffic web applications where bandwidth savings justify the encoding cost. Always serve with a WebP or JPEG fallback. Best used through a CDN that negotiates format automatically.

### 2.5 SVG

Vector format that scales to any resolution without quality loss.

**Strengths:**
- Infinite scaling: looks perfect at 1x, 2x, 3x, or 100x
- Tiny file sizes for simple graphics (often under 1KB for icons)
- Can be styled with CSS (colors, strokes, fills)
- Can be animated with CSS or JavaScript
- Accessible: text in SVGs is selectable and readable by screen readers

**Weaknesses:**
- Not suitable for photographs (vector representation of a photo is enormous)
- Complex SVGs with many paths can be slow to render
- Security concerns: SVGs can contain scripts (sanitize before rendering user-uploaded SVGs)

**When to use:** Icons, logos, illustrations, data visualizations, decorative graphics, anything that needs to scale perfectly.

**React Native:** Use `react-native-svg` for rendering SVGs. `expo-image` also supports SVG rendering as of SDK 51.

```typescript
// React Native SVG rendering
import { SvgUri } from 'react-native-svg';
import { Image } from 'expo-image';

// Option 1: react-native-svg (more control)
<SvgUri
  uri="https://cdn.example.com/icons/arrow.svg"
  width={24}
  height={24}
  fill={theme.colors.text}
/>

// Option 2: expo-image (simpler, handles caching)
<Image
  source="https://cdn.example.com/icons/arrow.svg"
  style={{ width: 24, height: 24 }}
  tintColor={theme.colors.text}
  contentFit="contain"
/>
```

**Next.js (Web):** Import SVGs as React components or use `<img>` tags.

```typescript
// Option 1: Import as component (with @svgr/webpack)
import ArrowIcon from '@/assets/icons/arrow.svg';
<ArrowIcon className="h-6 w-6 text-gray-700" />

// Option 2: Use next/image (simpler, no bundler config)
import Image from 'next/image';
<Image src="/icons/arrow.svg" alt="Arrow" width={24} height={24} />
```

### 2.6 Animated Formats: GIF vs Lottie vs Animated WebP

| Feature | GIF | Animated WebP | Lottie |
|---------|-----|---------------|--------|
| **File size** | Large (500KB-5MB typical) | 3-5x smaller than GIF | Tiny (5-50KB typical) |
| **Colors** | 256 max | Millions | Vector (unlimited) |
| **Transparency** | Binary (on/off) | Full alpha | Full alpha |
| **Scalability** | Pixel-based (blurry when scaled) | Pixel-based | Vector (infinite scaling) |
| **Interactivity** | None | None | Play/pause, speed, segments |
| **Creation** | Photoshop, ffmpeg | ffmpeg, CDN conversion | After Effects + Bodymovin |
| **Best for** | Legacy compatibility, simple clips | Animated photos, short clips | UI animations, onboarding, icons |
| **Performance** | Poor (large, slow decode) | Good | Excellent (GPU-accelerated) |

**The recommendation:** Use Lottie for UI animations and illustrations. Use animated WebP for short photographic clips. Never use GIF in a new project -- it's a legacy format with no advantages over animated WebP.

```typescript
// Lottie in React Native
import LottieView from 'lottie-react-native';

<LottieView
  source={require('./animations/success.json')}
  autoPlay
  loop={false}
  style={{ width: 120, height: 120 }}
/>

// Lottie on Web
import Lottie from 'lottie-react';
import successAnimation from './animations/success.json';

<Lottie
  animationData={successAnimation}
  loop={false}
  style={{ width: 120, height: 120 }}
/>
```

### 2.7 Format Decision Matrix

```
START
  │
  ├── Is it a photograph or complex real-world image?
  │   ├── Yes → Use WebP (lossy). Fall back to JPEG.
  │   │         Consider AVIF if your CDN supports it and you
  │   │         can serve with fallback.
  │   └── No ↓
  │
  ├── Is it an icon, logo, or simple illustration?
  │   ├── Does it need to scale to different sizes?
  │   │   ├── Yes → Use SVG
  │   │   └── No → Use WebP (lossless) or PNG
  │   └── Does it need transparency?
  │       ├── Yes → SVG or WebP (lossless) or PNG
  │       └── No → SVG for vector, WebP for raster
  │
  ├── Is it an animation?
  │   ├── UI animation (loading, success, onboarding)?
  │   │   └── Use Lottie
  │   ├── Short photographic clip?
  │   │   └── Use animated WebP or short MP4
  │   └── Legacy requirement?
  │       └── GIF (but convert to animated WebP if possible)
  │
  └── Is it a screenshot or image with text?
      └── Use WebP (lossless) or PNG
          (lossy compression creates artifacts around text)
```

---

## 3. EXPO-IMAGE (REACT NATIVE)

`expo-image` is the standard image component for React Native in 2026. It replaces both the core `Image` component and the once-popular `react-native-fast-image` (which is now unmaintained).

### 3.1 Why expo-image Over Alternatives

| Feature | RN `Image` | FastImage | expo-image |
|---------|-----------|-----------|------------|
| **Native engine** | Custom | SDWebImage (iOS) / Glide (Android) | SDWebImage (iOS) / Glide (Android) |
| **Disk cache** | Basic | Yes | Yes, configurable |
| **Memory cache** | Basic | Yes | Yes, with eviction policies |
| **Format negotiation** | No | No | Yes (WebP, AVIF auto-detection) |
| **Blurhash placeholder** | No | No | Built-in |
| **Thumbhash placeholder** | No | No | Built-in |
| **Transition animations** | No | Fade only | Cross-fade, flip, curl |
| **SVG support** | No | No | Yes (SDK 51+) |
| **Recycling support** | Poor | Medium | Excellent (FlashList-aware) |
| **Maintained** | Meta | Unmaintained | Expo (actively maintained) |
| **Content fit modes** | `resizeMode` | `resizeMode` | `contentFit` (CSS-aligned) |

The native engine matters enormously. SDWebImage and Glide are battle-tested image loading libraries used by millions of native iOS and Android apps. They handle memory management, disk caching, image decoding on background threads, and progressive loading at a level that no JavaScript-based solution can match.

### 3.2 Basic Configuration

```typescript
import { Image } from 'expo-image';

// Basic usage
<Image
  source={{ uri: 'https://cdn.example.com/photo.webp' }}
  style={{ width: 375, height: 250 }}
  contentFit="cover"
  cachePolicy="memory-disk"
  transition={200}
/>
```

### 3.3 contentFit Modes

`contentFit` replaces the old `resizeMode` prop and aligns with CSS `object-fit`:

```typescript
// cover: scales to fill the container, cropping if necessary
// (most common for feed cards, hero images, profile photos)
<Image source={source} contentFit="cover" />

// contain: scales to fit within the container, may leave empty space
// (good for product images where you need to see the whole thing)
<Image source={source} contentFit="contain" />

// fill: stretches to fill the container exactly (distorts aspect ratio)
// (rarely what you want)
<Image source={source} contentFit="fill" />

// scale-down: like contain, but never scales UP
// (good for icons and small images that shouldn't be enlarged)
<Image source={source} contentFit="scale-down" />

// none: no resizing, display at original size
<Image source={source} contentFit="none" />
```

Visual comparison:

```
Source image: 800×600 (landscape)
Container:   400×400 (square)

cover:                    contain:
┌──────────────────┐      ┌──────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│      │                  │
│▓▓▓ IMAGE ▓▓▓▓▓▓▓│      │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│▓▓▓(cropped)▓▓▓▓▓│      │▓▓▓ IMAGE ▓▓▓▓▓▓▓│
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│      │▓▓▓(full)▓▓▓▓▓▓▓▓│
└──────────────────┘      │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
Image fills container,    │                  │
edges are cropped.        └──────────────────┘
                          Image fits inside,
                          empty space on top/bottom.
```

### 3.4 Placeholder Strategies

Placeholders are critical for perceived performance. A blank space that suddenly pops into a full image feels broken. A smooth transition from a hint to the full image feels fast and intentional.

#### Blurhash

Blurhash encodes an image into a short string (~30 bytes for a 4x3 component hash) that renders as a blurred color representation. It's calculated server-side and sent with the API response.

```typescript
// The blurhash string comes from your API
// Typically 20-30 characters, e.g., "LKO2:N%2Tw=w]~RBVZRi};RPxuwH"

<Image
  source={{ uri: 'https://cdn.example.com/photo.webp' }}
  placeholder={{ blurhash: 'LKO2:N%2Tw=w]~RBVZRi};RPxuwH' }}
  contentFit="cover"
  transition={300}
  style={{ width: '100%', aspectRatio: 16 / 9 }}
/>
```

**Generating blurhash on the server:**

```typescript
// Node.js — generate blurhash at upload time
import { encode } from 'blurhash';
import sharp from 'sharp';

async function generateBlurhash(imagePath: string): Promise<string> {
  const { data, info } = await sharp(imagePath)
    .raw()
    .ensureAlpha()
    .resize(32, 32, { fit: 'inside' }) // Small size is fine for hash
    .toBuffer({ resolveWithObject: true });

  return encode(
    new Uint8ClampedArray(data),
    info.width,
    info.height,
    4, // x components (4 is the sweet spot)
    3  // y components (3 for landscape, 4 for square/portrait)
  );
}

// Store the hash alongside the image URL in your database
// { url: "https://cdn.example.com/photo.webp", blurhash: "LKO2:N%2Tw=w..." }
```

#### Thumbhash

Thumbhash is a newer alternative to blurhash that preserves more detail, including the image's aspect ratio and transparency. It produces a ~28-byte hash.

```typescript
import { Image } from 'expo-image';

<Image
  source={{ uri: 'https://cdn.example.com/photo.webp' }}
  placeholder={{ thumbhash: 'k0oGDQaSVnOIeHegh5d3d0iI+IVAV4Rx' }}
  contentFit="cover"
  transition={300}
  style={{ width: '100%', aspectRatio: 16 / 9 }}
/>
```

**Generating thumbhash on the server:**

```typescript
import { rgbaToThumbHash } from 'thumbhash';
import sharp from 'sharp';

async function generateThumbhash(imagePath: string): Promise<string> {
  const { data, info } = await sharp(imagePath)
    .raw()
    .ensureAlpha()
    .resize(100, 100, { fit: 'inside' }) // Thumbhash needs slightly more resolution
    .toBuffer({ resolveWithObject: true });

  const hash = rgbaToThumbHash(
    info.width,
    info.height,
    new Uint8Array(data)
  );

  return Buffer.from(hash).toString('base64');
}
```

#### Blurhash vs Thumbhash Comparison

| Feature | Blurhash | Thumbhash |
|---------|----------|-----------|
| **Size** | ~20-30 bytes | ~28 bytes |
| **Detail** | Color regions only | More structural detail |
| **Aspect ratio** | Not encoded | Encoded in hash |
| **Transparency** | Not supported | Supported |
| **Decode speed** | ~1ms | ~1ms |
| **Visual quality** | Good (abstract blur) | Better (recognizable shapes) |
| **Ecosystem** | Larger (more libraries) | Growing |

**Recommendation:** Use thumbhash for new projects. Use blurhash if you already have it integrated or need the larger ecosystem.

#### LQIP (Low Quality Image Placeholder)

A tiny version of the actual image (10-20px wide, heavily compressed), loaded inline as a base64 data URI:

```typescript
// LQIP from your API (base64-encoded tiny JPEG, ~200-500 bytes)
const lqipUri = 'data:image/jpeg;base64,/9j/4AAQSkZJRgABAQA...';

<Image
  source={{ uri: 'https://cdn.example.com/photo.webp' }}
  placeholder={{ uri: lqipUri }}
  contentFit="cover"
  transition={300}
  style={{ width: '100%', aspectRatio: 16 / 9 }}
/>
```

#### Dominant Color

The simplest placeholder -- just fill the space with the image's dominant color:

```typescript
<Image
  source={{ uri: 'https://cdn.example.com/photo.webp' }}
  placeholder={{ uri: '#3B82F6' }} // Or use a color string
  contentFit="cover"
  transition={300}
  style={{ width: '100%', aspectRatio: 16 / 9 }}
/>
```

### 3.5 Cache Policies

```typescript
import { Image } from 'expo-image';

// memory-disk (default): cache in both memory and disk
// Best for most use cases
<Image cachePolicy="memory-disk" ... />

// disk: cache on disk only, not in memory
// Good for large images you don't need to re-display instantly
<Image cachePolicy="disk" ... />

// memory: cache in memory only (cleared when app backgrounds)
// Good for temporary images
<Image cachePolicy="memory" ... />

// none: no caching at all
// Good for images that change frequently (live camera feeds, etc.)
<Image cachePolicy="none" ... />
```

**Cache management:**

```typescript
import { Image } from 'expo-image';

// Clear all caches (both memory and disk)
await Image.clearDiskCache();
await Image.clearMemoryCache();

// Get cache size
const diskCacheSize = await Image.getCachePathAsync('disk');

// Prefetch an image into cache (useful before navigating to a detail screen)
await Image.prefetch('https://cdn.example.com/detail-photo.webp');

// Prefetch multiple images
await Image.prefetch([
  'https://cdn.example.com/photo1.webp',
  'https://cdn.example.com/photo2.webp',
  'https://cdn.example.com/photo3.webp',
]);
```

### 3.6 Recycling in Lists with FlashList

This is where expo-image truly shines. In a FlashList (or RecyclerListView), cells are recycled -- the same view is reused with different data as the user scrolls. If image components don't handle recycling properly, you get:

1. **Stale images:** The old image is visible for a moment before the new one loads (image "flashing")
2. **Memory spikes:** Old images aren't released before new ones are loaded
3. **Redundant network requests:** Images that were already cached are re-fetched

expo-image handles all of these by coordinating with the native recycling layer:

```typescript
import { FlashList } from '@shopify/flash-list';
import { Image } from 'expo-image';

interface FeedItem {
  id: string;
  imageUrl: string;
  blurhash: string;
  title: string;
}

function FeedCard({ item }: { item: FeedItem }) {
  return (
    <View style={styles.card}>
      <Image
        source={{ uri: item.imageUrl }}
        placeholder={{ blurhash: item.blurhash }}
        contentFit="cover"
        transition={200}
        // recyclingKey is critical for FlashList recycling.
        // It tells expo-image that the image source has changed,
        // so it should cancel the previous load and start a new one.
        recyclingKey={item.id}
        style={styles.cardImage}
      />
      <Text style={styles.title}>{item.title}</Text>
    </View>
  );
}

function Feed({ items }: { items: FeedItem[] }) {
  return (
    <FlashList
      data={items}
      renderItem={({ item }) => <FeedCard item={item} />}
      estimatedItemSize={300}
      keyExtractor={(item) => item.id}
    />
  );
}

const styles = StyleSheet.create({
  card: {
    marginBottom: 16,
    borderRadius: 12,
    overflow: 'hidden',
    backgroundColor: '#F3F4F6',
  },
  cardImage: {
    width: '100%',
    aspectRatio: 16 / 9,
  },
  title: {
    padding: 12,
    fontSize: 16,
    fontWeight: '600',
  },
});
```

**Key points for list performance:**

1. **Always set `recyclingKey`** to a unique identifier for the item. Without it, recycled cells show stale images.
2. **Use `estimatedItemSize`** on FlashList for accurate recycling window calculations.
3. **Use `transition`** to smooth the placeholder-to-image swap (200-300ms is ideal).
4. **Set explicit dimensions** on the Image component (width + aspectRatio, or width + height). Avoid measuring after load.

### 3.7 Prefetching for Navigation

When you know the user is about to navigate to a screen with a large image (e.g., tapping a feed card to open a detail view), prefetch the full-resolution image:

```typescript
import { Image } from 'expo-image';
import { useCallback } from 'react';

function FeedCard({ item, onPress }: { item: FeedItem; onPress: () => void }) {
  const handlePressIn = useCallback(() => {
    // Start prefetching on press-in (before the press is released)
    // This gives you ~100-200ms head start
    Image.prefetch(item.fullResolutionUrl);
  }, [item.fullResolutionUrl]);

  return (
    <Pressable onPressIn={handlePressIn} onPress={onPress}>
      <Image
        source={{ uri: item.thumbnailUrl }}
        placeholder={{ blurhash: item.blurhash }}
        contentFit="cover"
        recyclingKey={item.id}
        style={styles.cardImage}
      />
    </Pressable>
  );
}
```

---

## 4. NEXT/IMAGE (WEB)

Next.js provides the `next/image` component that automates most image optimization for web. It's one of the strongest reasons to use Next.js over a custom React setup.

### 4.1 How next/image Works

When you use `next/image`, the following happens automatically:

1. **Format conversion:** The image is converted to WebP or AVIF based on the browser's `Accept` header
2. **Responsive srcset:** Multiple sizes are generated so the browser downloads only the size it needs
3. **Lazy loading:** Images below the fold are lazy-loaded with Intersection Observer (enabled by default)
4. **Size optimization:** Images are resized to the dimensions you specify, not the original dimensions
5. **Caching:** Optimized images are cached on the server (or at the edge with Vercel)

```typescript
import Image from 'next/image';

// Basic usage
<Image
  src="/photos/hero.jpg"
  alt="A beautiful mountain landscape"
  width={1200}
  height={630}
  quality={75}
/>
```

### 4.2 The `sizes` Prop

The `sizes` prop is the most misunderstood and most important prop on `next/image`. It tells the browser how wide the image will be at each viewport size, so the browser can choose the right image from the `srcset` *before* CSS is parsed.

Without `sizes`, the browser assumes the image is full-width (100vw) and downloads the largest version. With `sizes`, it can download a smaller version.

```typescript
// A responsive card image:
// - Full width on mobile (< 640px)
// - Half width on tablet (640-1024px)
// - One-third width on desktop (> 1024px)

<Image
  src="/photos/product.jpg"
  alt="Product photo"
  fill
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
/>
```

Here's what the browser does with this:

```
Viewport width: 375px (mobile phone)
  → sizes says: 100vw = 375px
  → At 2x pixel density: needs 750px wide image
  → Browser picks the 828px variant from srcset

Viewport width: 768px (tablet)
  → sizes says: 50vw = 384px
  → At 2x pixel density: needs 768px wide image
  → Browser picks the 828px variant from srcset

Viewport width: 1440px (desktop)
  → sizes says: 33vw = 475px
  → At 1x pixel density: needs 475px wide image
  → Browser picks the 640px variant from srcset
```

**Common sizes patterns:**

```typescript
// Full-width hero image
sizes="100vw"

// Content width (max 1200px centered)
sizes="(max-width: 1200px) 100vw, 1200px"

// Responsive grid: 1 col → 2 col → 3 col → 4 col
sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, (max-width: 1536px) 33vw, 25vw"

// Sidebar image (always 300px)
sizes="300px"

// Thumbnail (always 80px)
sizes="80px"
```

### 4.3 srcset Generation

Next.js generates the following device sizes by default (configurable in `next.config.js`):

```javascript
// next.config.js
module.exports = {
  images: {
    // deviceSizes are for layout="responsive" or fill images
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
    // imageSizes are for fixed-width images
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    // Format preference order
    formats: ['image/avif', 'image/webp'],
  },
};
```

The generated HTML looks like:

```html
<img
  srcset="
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=640&q=75  640w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=750&q=75  750w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=828&q=75  828w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=1080&q=75 1080w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=1200&q=75 1200w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=1920&q=75 1920w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=2048&q=75 2048w,
    /_next/image?url=%2Fphotos%2Fproduct.jpg&w=3840&q=75 3840w
  "
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  src="/_next/image?url=%2Fphotos%2Fproduct.jpg&w=3840&q=75"
  loading="lazy"
  decoding="async"
/>
```

### 4.4 The `priority` Prop

For the Largest Contentful Paint (LCP) image -- typically the hero image or the first visible image -- you should set `priority` to disable lazy loading and trigger a preload:

```typescript
// Hero image — always above the fold, always the LCP element
<Image
  src="/photos/hero.jpg"
  alt="Mountain landscape"
  width={1920}
  height={1080}
  priority // Disables lazy loading, adds <link rel="preload">
  quality={80}
  sizes="100vw"
/>

// Regular content image — lazy loaded by default
<Image
  src="/photos/content.jpg"
  alt="Content image"
  width={800}
  height={600}
  sizes="(max-width: 768px) 100vw, 800px"
/>
```

**Rule of thumb:** Set `priority` on exactly one image per page -- the LCP image. Setting it on multiple images defeats the purpose (everything can't be high priority).

### 4.5 Placeholder Blur

`next/image` supports blurred placeholders that show while the image loads:

```typescript
// For local images: automatic blur placeholder
import heroImage from '@/public/photos/hero.jpg';

<Image
  src={heroImage}
  alt="Hero"
  placeholder="blur" // Automatically generated at build time
/>

// For remote images: provide a blurDataURL
<Image
  src="https://cdn.example.com/photo.jpg"
  alt="Remote photo"
  width={800}
  height={600}
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..."
/>
```

**Generating blurDataURL with plaiceholder:**

```typescript
// scripts/generate-blur.ts
import { getPlaiceholder } from 'plaiceholder';

async function getBlurDataURL(imageUrl: string): Promise<string> {
  const response = await fetch(imageUrl);
  const buffer = Buffer.from(await response.arrayBuffer());

  const { base64 } = await getPlaiceholder(buffer, {
    size: 10, // Very small — just for the blur effect
  });

  return base64;
}

// In a Server Component or getStaticProps:
export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  const blurDataURL = await getBlurDataURL(product.imageUrl);

  return (
    <Image
      src={product.imageUrl}
      alt={product.name}
      width={800}
      height={600}
      placeholder="blur"
      blurDataURL={blurDataURL}
    />
  );
}
```

### 4.6 Remote Image Configuration

By default, `next/image` only optimizes local images. For remote images, you must configure allowed domains:

```javascript
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/images/**',
      },
      {
        protocol: 'https',
        hostname: '**.cloudinary.com', // Wildcard for subdomains
      },
      {
        protocol: 'https',
        hostname: 'images.unsplash.com',
      },
    ],
  },
};
```

### 4.7 When to Use an External CDN vs. Built-in next/image

| Scenario | Recommendation |
|----------|---------------|
| Static marketing site with local images | Built-in `next/image` |
| Dynamic content with user-uploaded images | External CDN (Cloudinary, Imgix) |
| Need face detection, AI cropping, or overlays | External CDN |
| High traffic (>1M images/day) | External CDN (cost-effective at scale) |
| Using Vercel for hosting | Vercel Image Optimization (built-in) |
| Need art direction (different crops per viewport) | External CDN with URL transformations |
| Simple blog or portfolio | Built-in `next/image` |

---

## 5. IMAGE CDN SERVICES

An image CDN sits between your origin storage (S3, GCS, etc.) and the user. It optimizes, resizes, reformats, and caches images on the fly via URL parameters. This is the most impactful infrastructure decision for image-heavy applications.

### 5.1 Cloudinary

The most feature-rich image CDN. URL-based transformations that cover everything from basic resize to AI-powered content-aware cropping.

```
Base URL pattern:
https://res.cloudinary.com/{cloud_name}/image/upload/{transformations}/{public_id}.{format}

Examples:

Resize to 400x300, cover crop:
https://res.cloudinary.com/demo/image/upload/w_400,h_300,c_fill/sample.webp

Auto format + auto quality (let Cloudinary decide):
https://res.cloudinary.com/demo/image/upload/f_auto,q_auto/sample

Face-aware crop (zoom into detected face):
https://res.cloudinary.com/demo/image/upload/w_200,h_200,c_thumb,g_face/portrait.webp

Blur background + overlay text:
https://res.cloudinary.com/demo/image/upload/e_blur_region:faces/l_text:Arial_30:Hello/sample

Progressive JPEG at quality 70, 800px wide:
https://res.cloudinary.com/demo/image/upload/w_800,q_70,fl_progressive/photo.jpg
```

**Key features:**
- Automatic format negotiation (`f_auto` serves WebP to Chrome, AVIF to Firefox, JPEG to Safari 15)
- AI-powered `g_auto` gravity that detects the important part of the image
- Background removal (`e_background_removal`)
- Responsive breakpoints API (automatically determines optimal image sizes)
- Video transformations (thumbnail generation, format conversion, adaptive streaming)

### 5.2 Imgix

Similar URL-based approach to Cloudinary, with a cleaner URL syntax and strong analytics.

```
Base URL pattern:
https://{source}.imgix.net/{path}?{parameters}

Examples:

Resize to 400x300, cover crop:
https://example.imgix.net/photo.jpg?w=400&h=300&fit=crop

Auto format + auto compression:
https://example.imgix.net/photo.jpg?auto=format,compress

Face-aware crop:
https://example.imgix.net/portrait.jpg?w=200&h=200&fit=facearea&facepad=1.5

Blur:
https://example.imgix.net/photo.jpg?blur=50

DPR-aware (serve 2x images):
https://example.imgix.net/photo.jpg?w=400&dpr=2
```

**Key features:**
- Cleaner URL syntax (query parameters vs. path-based)
- Real-time rendering analytics
- Client libraries for React, Vue, etc.
- PDF and SVG rendering
- Color palette extraction

### 5.3 Cloudflare Images

Simpler and cheaper, tightly integrated with the Cloudflare CDN.

```
Upload via API, serve via variants:
https://imagedelivery.net/{account_hash}/{image_id}/{variant_name}

Variants are predefined in your Cloudflare dashboard:
- "thumbnail": 150x150, cover
- "card": 400x300, cover
- "hero": 1200x630, cover
- "full": max 2400px, contain

Flexible variants (URL parameters):
https://imagedelivery.net/{account_hash}/{image_id}/w=400,h=300,fit=cover
```

**Key features:**
- Flat pricing ($5/month for 100K images stored, $1/100K unique transformations)
- No per-bandwidth charges (included in Cloudflare plan)
- Integrated with Cloudflare CDN (200+ PoPs)
- Simple API (fewer transformation options than Cloudinary, but covers 90% of use cases)

### 5.4 Vercel Image Optimization

Built into Next.js when deployed on Vercel. No additional service needed.

```
Automatically used by next/image:
/_next/image?url={encoded_url}&w={width}&q={quality}

Configuration in next.config.js:
module.exports = {
  images: {
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 60, // seconds
  },
};
```

**Key features:**
- Zero configuration when using `next/image` on Vercel
- Automatic format negotiation (AVIF, WebP, JPEG)
- Edge caching at Vercel's CDN
- Included in Vercel pricing (with limits per plan)

### 5.5 Cost Comparison (approximate, 2026)

| Service | Free Tier | Typical Cost at Scale (1M images/month) |
|---------|-----------|----------------------------------------|
| **Cloudinary** | 25 credits/month (~25K transformations) | ~$89-249/month |
| **Imgix** | 1,000 origin images | ~$100-300/month |
| **Cloudflare Images** | None | ~$5-50/month |
| **Vercel Image Optimization** | Included in Hobby plan (1000 images) | Included in Pro plan ($20/month), then $5/1000 images |

Cloudflare Images is dramatically cheaper at scale. Cloudinary is more expensive but offers the most transformation features (AI cropping, background removal, video). Choose based on your feature needs, not just price.

### 5.6 Building a Universal Image URL Utility

This utility generates optimized image URLs that work for both React Native and web, from a single CDN configuration:

```typescript
// lib/image-url.ts

type ImageFit = 'cover' | 'contain' | 'fill' | 'scale-down';
type ImageFormat = 'auto' | 'webp' | 'avif' | 'jpeg' | 'png';

interface ImageTransformOptions {
  width: number;
  height?: number;
  fit?: ImageFit;
  quality?: number; // 1-100
  format?: ImageFormat;
  dpr?: number; // 1, 2, or 3
  blur?: number; // 0-100
}

interface ImageCDNConfig {
  provider: 'cloudinary' | 'imgix' | 'cloudflare';
  baseUrl: string;
  cloudName?: string; // Cloudinary only
  accountHash?: string; // Cloudflare only
}

const defaultConfig: ImageCDNConfig = {
  provider: 'cloudinary',
  baseUrl: 'https://res.cloudinary.com',
  cloudName: 'your-cloud',
};

export function getImageUrl(
  imageId: string,
  options: ImageTransformOptions,
  config: ImageCDNConfig = defaultConfig
): string {
  const {
    width,
    height,
    fit = 'cover',
    quality = 75,
    format = 'auto',
    dpr = 1,
    blur,
  } = options;

  const actualWidth = Math.round(width * dpr);
  const actualHeight = height ? Math.round(height * dpr) : undefined;

  switch (config.provider) {
    case 'cloudinary': {
      const transforms: string[] = [];
      transforms.push(`w_${actualWidth}`);
      if (actualHeight) transforms.push(`h_${actualHeight}`);
      transforms.push(`c_${fitToCloudinary(fit)}`);
      transforms.push(`q_${quality}`);
      transforms.push(`f_${format === 'auto' ? 'auto' : format}`);
      if (blur) transforms.push(`e_blur:${blur * 10}`);
      return `${config.baseUrl}/${config.cloudName}/image/upload/${transforms.join(',')}/${imageId}`;
    }

    case 'imgix': {
      const params = new URLSearchParams();
      params.set('w', String(actualWidth));
      if (actualHeight) params.set('h', String(actualHeight));
      params.set('fit', fitToImgix(fit));
      params.set('q', String(quality));
      if (format === 'auto') {
        params.set('auto', 'format,compress');
      } else {
        params.set('fm', format);
      }
      if (blur) params.set('blur', String(blur * 5));
      return `${config.baseUrl}/${imageId}?${params.toString()}`;
    }

    case 'cloudflare': {
      const params: string[] = [];
      params.push(`w=${actualWidth}`);
      if (actualHeight) params.push(`h=${actualHeight}`);
      params.push(`fit=${fitToCloudflare(fit)}`);
      params.push(`q=${quality}`);
      if (format !== 'auto') params.push(`f=${format}`);
      if (blur) params.push(`blur=${blur * 5}`);
      return `https://imagedelivery.net/${config.accountHash}/${imageId}/${params.join(',')}`;
    }
  }
}

function fitToCloudinary(fit: ImageFit): string {
  const map: Record<ImageFit, string> = {
    cover: 'fill',
    contain: 'fit',
    fill: 'scale',
    'scale-down': 'limit',
  };
  return map[fit];
}

function fitToImgix(fit: ImageFit): string {
  const map: Record<ImageFit, string> = {
    cover: 'crop',
    contain: 'fit',
    fill: 'scale',
    'scale-down': 'max',
  };
  return map[fit];
}

function fitToCloudflare(fit: ImageFit): string {
  return fit; // Cloudflare uses the same terminology
}

// --- Responsive sizes helper ---

interface ResponsiveImageUrls {
  small: string;   // 375px display width
  medium: string;  // 768px display width
  large: string;   // 1200px display width
  thumbnail: string; // 80px display width
}

export function getResponsiveImageUrls(
  imageId: string,
  aspectRatio: number = 16 / 9,
  config?: ImageCDNConfig
): ResponsiveImageUrls {
  const sizes = [
    { key: 'thumbnail', width: 80 },
    { key: 'small', width: 375 },
    { key: 'medium', width: 768 },
    { key: 'large', width: 1200 },
  ] as const;

  const urls = {} as ResponsiveImageUrls;
  for (const { key, width } of sizes) {
    urls[key] = getImageUrl(
      imageId,
      {
        width,
        height: Math.round(width / aspectRatio),
        dpr: key === 'thumbnail' ? 1 : 2, // 2x for non-thumbnails
        quality: key === 'thumbnail' ? 60 : 75,
      },
      config
    );
  }
  return urls;
}
```

Usage across platforms:

```typescript
// React Native
import { Image } from 'expo-image';
import { getImageUrl } from '@/lib/image-url';
import { PixelRatio } from 'react-native';

const dpr = PixelRatio.get(); // 2 on iPhone SE, 3 on iPhone 15 Pro

<Image
  source={{ uri: getImageUrl('photo123.jpg', {
    width: 375,
    height: 250,
    dpr: Math.min(dpr, 3), // Cap at 3x
  }) }}
  contentFit="cover"
  style={{ width: '100%', aspectRatio: 375 / 250 }}
/>

// Next.js (using a custom loader)
// next.config.js
module.exports = {
  images: {
    loader: 'custom',
    loaderFile: './lib/image-loader.ts',
  },
};

// lib/image-loader.ts
import { getImageUrl } from './image-url';

export default function cloudinaryLoader({
  src,
  width,
  quality,
}: {
  src: string;
  width: number;
  quality?: number;
}) {
  return getImageUrl(src, {
    width,
    quality: quality || 75,
    format: 'auto',
  });
}
```

---

## 6. RESPONSIVE IMAGES BY PIXEL DENSITY

Pixel density is the most commonly misunderstood aspect of responsive images. Getting it wrong means either serving blurry images (too few pixels) or wasting bandwidth (too many pixels).

### 6.1 Understanding Pixel Density

A "point" (pt) or "CSS pixel" is a logical unit. A "device pixel" is a physical pixel on the screen. The ratio between them is the pixel density (or device pixel ratio, DPR).

```
Device                        Screen Width (pt)   DPR    Actual Pixels
─────────────────────────────────────────────────────────────────────
iPhone SE (3rd gen)           375 pt              2x     750 px
iPhone 15                     393 pt              3x     1179 px
iPhone 15 Pro Max             430 pt              3x     1290 px
Samsung Galaxy A14            360 pt              2x     720 px
Samsung Galaxy S25 Ultra      412 pt              3.5x   1440 px
Pixel 9                       412 pt              2.625x 1080 px
iPad Pro 13" M4               1024 pt             2x     2048 px
MacBook Pro 14"               1512 pt             2x     3024 px
1080p Desktop Monitor         1920 pt             1x     1920 px
4K Desktop Monitor            3840 pt             1x*    3840 px
```

*Note: 4K monitors typically run at 1x DPR but with more screen real estate, or at 2x DPR with the same logical resolution as 1080p. It depends on the user's display scaling settings.

### 6.2 The Formula

```
Actual pixels needed = displayed size (pt) × pixel density (DPR)

Example: A 375×250 pt image on an iPhone 15 (3x):
375 × 3 = 1125 px wide
250 × 3 = 750 px tall

You need a 1125×750 image. Serving a 375×250 image looks blurry.
Serving a 4000×3000 image wastes 95% of the data transferred.
```

### 6.3 React Native: PixelRatio API

```typescript
import { PixelRatio, Dimensions } from 'react-native';

// Get the device pixel ratio
const dpr = PixelRatio.get();
// iPhone SE: 2
// iPhone 15 Pro: 3
// Some Android: 2.625 (round up to 3)

// Round to nearest whole pixel to avoid sub-pixel rendering
const imageWidth = PixelRatio.roundToNearestPixel(375);

// Get the physical pixel count needed
const physicalWidth = PixelRatio.getPixelSizeForLayoutSize(375);
// iPhone SE: 750
// iPhone 15 Pro: 1125

// Use this to request the right image size from your CDN
function getOptimalImageWidth(displayWidth: number): number {
  const dpr = Math.min(PixelRatio.get(), 3); // Cap at 3x
  const physicalWidth = Math.ceil(displayWidth * dpr);

  // Round up to nearest standard size to improve cache hit rates
  const standardSizes = [320, 480, 640, 750, 828, 1080, 1200, 1440, 1920];
  return standardSizes.find((size) => size >= physicalWidth) || standardSizes[standardSizes.length - 1];
}
```

### 6.4 React Native Asset Suffixes

For local assets (bundled with the app), React Native automatically selects the right file based on the device's pixel density:

```
assets/
  logo.png        ← Used on 1x screens (mdpi Android)
  logo@2x.png     ← Used on 2x screens (xhdpi Android, non-Pro iPhones)
  logo@3x.png     ← Used on 3x screens (xxhdpi Android, Pro iPhones)
```

```typescript
// React Native automatically picks the right variant
import logo from '@/assets/logo.png';
<Image source={logo} style={{ width: 100, height: 40 }} />
// On iPhone 15 Pro (3x): loads logo@3x.png (300×120 physical pixels)
// On iPhone SE (2x): loads logo@2x.png (200×80 physical pixels)
```

### 6.5 Web: srcset with Pixel Density Descriptors

For the web, you have two srcset strategies:

**1. Pixel density descriptors (for fixed-size images):**

```html
<!-- An avatar that's always 48x48 CSS pixels -->
<img
  srcset="
    /avatars/user.jpg?w=48   1x,
    /avatars/user.jpg?w=96   2x,
    /avatars/user.jpg?w=144  3x
  "
  src="/avatars/user.jpg?w=96"
  width="48"
  height="48"
  alt="User avatar"
/>
```

**2. Width descriptors (for responsive images):**

```html
<!-- A responsive image that changes size with the viewport -->
<img
  srcset="
    /photos/hero.jpg?w=640   640w,
    /photos/hero.jpg?w=750   750w,
    /photos/hero.jpg?w=1080  1080w,
    /photos/hero.jpg?w=1200  1200w,
    /photos/hero.jpg?w=1920  1920w
  "
  sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
  src="/photos/hero.jpg?w=1200"
  alt="Hero photo"
/>
```

Width descriptors with `sizes` are more flexible because they account for both viewport width AND pixel density. The browser does the math:

```
Viewport: 375px, DPR: 2x
sizes says: 100vw = 375px
Effective width needed: 375 × 2 = 750px
Browser picks: 750w variant

Viewport: 1440px, DPR: 1x
sizes says: 33vw = 475px
Effective width needed: 475 × 1 = 475px
Browser picks: 640w variant (next size up)
```

### 6.6 Building a Universal Density-Aware Image Component

```typescript
// components/OptimizedImage/index.tsx
// Works on both React Native and Web (via NativeWind or platform-specific code)

import { Platform, PixelRatio } from 'react-native';
import { Image as ExpoImage } from 'expo-image';
import NextImage from 'next/image';
import { getImageUrl } from '@/lib/image-url';

interface OptimizedImageProps {
  imageId: string;
  alt: string;
  width: number;  // Display width in points/CSS pixels
  height: number; // Display height in points/CSS pixels
  fit?: 'cover' | 'contain';
  priority?: boolean;
  blurhash?: string;
  quality?: number;
}

export function OptimizedImage({
  imageId,
  alt,
  width,
  height,
  fit = 'cover',
  priority = false,
  blurhash,
  quality = 75,
}: OptimizedImageProps) {
  if (Platform.OS === 'web') {
    return (
      <NextImage
        src={imageId}
        alt={alt}
        width={width}
        height={height}
        quality={quality}
        priority={priority}
        style={{ objectFit: fit }}
        sizes={`${width}px`}
        placeholder={blurhash ? 'blur' : 'empty'}
        blurDataURL={blurhash ? decodeBlurhashToDataURL(blurhash) : undefined}
      />
    );
  }

  // React Native
  const dpr = Math.min(Math.ceil(PixelRatio.get()), 3);
  const uri = getImageUrl(imageId, {
    width,
    height,
    dpr,
    quality,
    format: 'auto',
  });

  return (
    <ExpoImage
      source={{ uri }}
      alt={alt}
      contentFit={fit}
      placeholder={blurhash ? { blurhash } : undefined}
      transition={200}
      style={{ width, height }}
      cachePolicy="memory-disk"
    />
  );
}
```

---

## 7. RESPONSIVE LAYOUTS FOR MOBILE

The screen landscape in 2026 is extraordinarily diverse. Your layout must work across all of these:

```
Category          Width (pt)    Examples
────────────────────────────────────────────────────────
Small phone       320-360       iPhone SE, Galaxy A-series
Medium phone      375-393       iPhone 15, Pixel 9
Large phone       412-430       iPhone 15 Pro Max, Galaxy S25 Ultra
Foldable (folded) 360-380       Galaxy Z Fold (outer)
Foldable (open)   585-673       Galaxy Z Fold (inner)
Small tablet      744-768       iPad Mini, Galaxy Tab A
Medium tablet     820-834       iPad Air, iPad 10th gen
Large tablet      1024-1194     iPad Pro 11", iPad Pro 13"
Desktop web       1024-1920+    Laptop/desktop browsers
```

### 7.1 React Native Responsive Patterns

#### The useWindowDimensions Hook

```typescript
import { useWindowDimensions } from 'react-native';

function MyComponent() {
  const { width, height } = useWindowDimensions();
  // Re-renders when dimensions change (rotation, split view, etc.)

  const isTablet = width >= 768;
  const columns = width >= 1024 ? 3 : width >= 768 ? 2 : 1;

  return (
    <FlatList
      data={items}
      numColumns={columns}
      key={columns} // Force re-mount when column count changes
      renderItem={({ item }) => (
        <View style={{ width: width / columns - 16 }}>
          <Card item={item} variant={isTablet ? 'expanded' : 'compact'} />
        </View>
      )}
    />
  );
}
```

#### Building a Breakpoint System

```typescript
// hooks/useBreakpoint.ts
import { useWindowDimensions } from 'react-native';

type Breakpoint = 'xs' | 'sm' | 'md' | 'lg' | 'xl' | '2xl';

const BREAKPOINTS: Record<Breakpoint, number> = {
  xs: 0,
  sm: 640,
  md: 768,
  lg: 1024,
  xl: 1280,
  '2xl': 1536,
};

export function useBreakpoint(): Breakpoint {
  const { width } = useWindowDimensions();

  if (width >= BREAKPOINTS['2xl']) return '2xl';
  if (width >= BREAKPOINTS.xl) return 'xl';
  if (width >= BREAKPOINTS.lg) return 'lg';
  if (width >= BREAKPOINTS.md) return 'md';
  if (width >= BREAKPOINTS.sm) return 'sm';
  return 'xs';
}

export function useIsAbove(breakpoint: Breakpoint): boolean {
  const { width } = useWindowDimensions();
  return width >= BREAKPOINTS[breakpoint];
}

export function useResponsiveValue<T>(values: Partial<Record<Breakpoint, T>>): T | undefined {
  const breakpoint = useBreakpoint();
  const orderedBreakpoints: Breakpoint[] = ['2xl', 'xl', 'lg', 'md', 'sm', 'xs'];
  const currentIndex = orderedBreakpoints.indexOf(breakpoint);

  for (let i = currentIndex; i < orderedBreakpoints.length; i++) {
    const bp = orderedBreakpoints[i];
    if (values[bp] !== undefined) return values[bp];
  }
  return undefined;
}

// Usage:
function ProductGrid() {
  const columns = useResponsiveValue({
    xs: 1,
    sm: 2,
    md: 2,
    lg: 3,
    xl: 4,
  });

  const cardVariant = useResponsiveValue({
    xs: 'compact' as const,
    md: 'expanded' as const,
  });

  return (
    <FlatList
      data={products}
      numColumns={columns}
      key={columns}
      renderItem={({ item }) => <ProductCard product={item} variant={cardVariant} />}
    />
  );
}
```

### 7.2 NativeWind Breakpoints

NativeWind brings Tailwind CSS breakpoints to React Native, making responsive design feel natural:

```typescript
import { View, Text } from 'react-native';

// NativeWind responsive classes (Tailwind syntax)
function ResponsiveCard({ title, description }: { title: string; description: string }) {
  return (
    <View className="
      flex-col p-4
      sm:flex-row sm:p-6
      md:p-8
      lg:max-w-4xl lg:mx-auto
    ">
      <View className="
        w-full mb-4
        sm:w-1/3 sm:mb-0 sm:mr-6
        md:w-1/4
      ">
        <Image
          source={{ uri: imageUrl }}
          className="w-full aspect-square rounded-lg sm:aspect-[3/4]"
        />
      </View>
      <View className="flex-1">
        <Text className="
          text-lg font-bold
          sm:text-xl
          md:text-2xl
        ">
          {title}
        </Text>
        <Text className="
          text-sm text-gray-600 mt-2
          sm:text-base
          md:text-lg md:leading-relaxed
        ">
          {description}
        </Text>
      </View>
    </View>
  );
}

// Platform-specific classes
function PlatformAwareComponent() {
  return (
    <View className="
      p-4
      ios:pt-12
      android:pt-8
      web:max-w-7xl web:mx-auto
    ">
      <Text className="
        text-base
        ios:font-sf-pro
        android:font-roboto
      ">
        Platform-aware text
      </Text>
    </View>
  );
}
```

### 7.3 Tamagui Media Queries

Tamagui provides responsive props directly on every component:

```typescript
import { XStack, YStack, Text, Image } from 'tamagui';

function ResponsiveLayout() {
  return (
    <XStack
      flexDirection="column"
      padding="$4"
      gap="$4"
      $gtMd={{
        flexDirection: 'row',
        padding: '$6',
      }}
      $gtLg={{
        maxWidth: 1200,
        marginHorizontal: 'auto',
        padding: '$8',
      }}
    >
      <YStack
        width="100%"
        $gtMd={{ width: '33%' }}
        $gtLg={{ width: '25%' }}
      >
        <Image
          source={{ uri: imageUrl }}
          width="100%"
          aspectRatio={1}
          borderRadius="$4"
          $gtSm={{ aspectRatio: 3 / 4 }}
        />
      </YStack>
      <YStack flex={1}>
        <Text
          fontSize="$6"
          fontWeight="bold"
          $gtSm={{ fontSize: '$7' }}
          $gtMd={{ fontSize: '$8' }}
        >
          {title}
        </Text>
      </YStack>
    </XStack>
  );
}
```

### 7.4 Adaptive Column Layouts

The most common responsive pattern: varying the number of columns based on screen width.

```typescript
// components/ResponsiveGrid.tsx
import { useWindowDimensions, View, StyleSheet } from 'react-native';

interface ResponsiveGridProps<T> {
  data: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
  minItemWidth?: number; // Minimum width per item in points
  gap?: number;
  padding?: number;
}

export function ResponsiveGrid<T>({
  data,
  renderItem,
  minItemWidth = 300,
  gap = 16,
  padding = 16,
}: ResponsiveGridProps<T>) {
  const { width } = useWindowDimensions();
  const availableWidth = width - padding * 2;
  const columns = Math.max(1, Math.floor((availableWidth + gap) / (minItemWidth + gap)));
  const itemWidth = (availableWidth - gap * (columns - 1)) / columns;

  return (
    <View style={[styles.grid, { padding }]}>
      {data.map((item, index) => (
        <View
          key={index}
          style={{
            width: itemWidth,
            marginRight: (index + 1) % columns === 0 ? 0 : gap,
            marginBottom: gap,
          }}
        >
          {renderItem(item, index)}
        </View>
      ))}
    </View>
  );
}

const styles = StyleSheet.create({
  grid: {
    flexDirection: 'row',
    flexWrap: 'wrap',
  },
});

// Usage:
<ResponsiveGrid
  data={products}
  minItemWidth={280}
  renderItem={(product) => <ProductCard product={product} />}
/>
// 375px phone: 1 column
// 768px tablet: 2 columns
// 1200px desktop: 3 columns
// 1600px wide desktop: 4 columns
```

### 7.5 Safe Areas

Safe areas account for hardware intrusions: the Dynamic Island, camera notch, home indicator, status bar, and rounded screen corners.

```typescript
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function ScreenLayout({ children }: { children: React.ReactNode }) {
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

// Common inset values (2026):
// iPhone 15 Pro (Dynamic Island): top=59, bottom=34, left=0, right=0
// iPhone SE: top=20, bottom=0, left=0, right=0
// Pixel 9: top=24, bottom=0, left=0, right=0
// iPad Pro: top=24, bottom=20, left=0, right=0

// With NativeWind:
function ScreenLayout({ children }) {
  return (
    <View className="flex-1 pt-safe pb-safe px-safe">
      {children}
    </View>
  );
}
```

**Critical rule:** Never put interactive elements (buttons, inputs) in the bottom safe area. The home indicator lives there, and tapping a button overlapping it is frustrating and may trigger the home gesture.

```typescript
// BAD: Button in the safe area zone
<View style={{ position: 'absolute', bottom: 0 }}>
  <Button title="Submit" />
</View>

// GOOD: Button above the safe area
function FloatingButton() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{
      position: 'absolute',
      bottom: insets.bottom + 16, // 16pt above the safe area
      left: 16,
      right: 16,
    }}>
      <Button title="Submit" />
    </View>
  );
}
```

### 7.6 Orientation Changes

```typescript
import { useWindowDimensions } from 'react-native';

function useOrientation() {
  const { width, height } = useWindowDimensions();
  return {
    isPortrait: height >= width,
    isLandscape: width > height,
  };
}

// Lock orientation per screen (Expo)
import * as ScreenOrientation from 'expo-screen-orientation';

function VideoPlayerScreen() {
  useEffect(() => {
    // Allow landscape for video
    ScreenOrientation.unlockAsync();

    return () => {
      // Lock back to portrait when leaving
      ScreenOrientation.lockAsync(
        ScreenOrientation.OrientationLock.PORTRAIT_UP
      );
    };
  }, []);

  return <VideoPlayer />;
}
```

### 7.7 Foldable Devices

Foldable phones (Samsung Galaxy Z Fold, Pixel Fold) introduce a new layout dimension: the fold. When open, the screen is tablet-sized but divided by a physical hinge.

```typescript
// hooks/useFoldableState.ts
import { useWindowDimensions, Platform } from 'react-native';

interface FoldableState {
  isFoldable: boolean;
  isFolded: boolean;     // Using the narrow outer screen
  isUnfolded: boolean;   // Using the wide inner screen
  isTabletop: boolean;   // Partially folded (like a laptop)
}

export function useFoldableState(): FoldableState {
  const { width, height } = useWindowDimensions();

  // Heuristic: Galaxy Z Fold outer screen is ~360pt,
  // inner screen is ~585pt. The fold is detected by
  // a significant width change.
  const isLikelyFoldableWidth = width >= 580 && width <= 680;
  const isNarrowPhone = width < 400;

  // For more accurate detection, use react-native-device-info
  // or the Jetpack WindowManager API on Android
  return {
    isFoldable: Platform.OS === 'android' && (isLikelyFoldableWidth || isNarrowPhone),
    isFolded: isNarrowPhone,
    isUnfolded: isLikelyFoldableWidth,
    isTabletop: false, // Requires native module for hinge angle detection
  };
}

// Layout that adapts to fold state
function AdaptiveFeedLayout() {
  const { isUnfolded } = useFoldableState();
  const { width } = useWindowDimensions();

  if (isUnfolded) {
    // Two-pane layout: list on left, detail on right
    return (
      <View style={{ flexDirection: 'row', flex: 1 }}>
        <View style={{ width: width / 2 }}>
          <FeedList />
        </View>
        <View style={{ width: width / 2 }}>
          <FeedDetail />
        </View>
      </View>
    );
  }

  // Single-pane layout for folded state or regular phones
  return <FeedList />;
}
```

### 7.8 iPad Split View and Multitasking

On iPad, your app can run side-by-side with another app. The available width changes dynamically:

```typescript
// iPad multitasking widths (approximate, iPad Pro 11"):
// Full screen:        834 pt
// 2/3 split:          556 pt
// 1/2 split:          417 pt
// 1/3 split:          278 pt
// Slide Over:         320 pt

// useWindowDimensions automatically reflects the current size.
// Your responsive breakpoint system handles this automatically
// if you base layouts on window width, not screen width.

function IPadAwareLayout({ children }: { children: React.ReactNode }) {
  const { width } = useWindowDimensions();

  // This correctly handles both:
  // - iPad at 1/3 split view (278pt → phone layout)
  // - iPad at full screen (834pt → tablet layout)
  // - iPad at 2/3 split (556pt → depends on your breakpoints)
  const layout = width >= 768 ? 'tablet' : width >= 500 ? 'compact-tablet' : 'phone';

  return (
    <LayoutContext.Provider value={layout}>
      {children}
    </LayoutContext.Provider>
  );
}
```

---

## 8. RESPONSIVE TYPOGRAPHY

Typography that doesn't respect the user's text size preferences is both an accessibility failure and, on iOS, a reason for App Store rejection.

### 8.1 Dynamic Type (iOS) and Font Scale (Android)

Both platforms allow users to set a preferred text size in system settings. On iOS this is called Dynamic Type; on Android it's Font Scale.

```
iOS Dynamic Type sizes:
xSmall    → 0.82x
Small     → 0.88x
Medium    → 0.94x
Large     → 1.0x  (default)
xLarge    → 1.12x
xxLarge   → 1.24x
xxxLarge  → 1.35x
AX1-AX5  → 1.6x to 3.1x (accessibility sizes)

Android Font Scale:
Small     → 0.85x
Default   → 1.0x
Large     → 1.15x
Largest   → 1.3x
```

React Native respects these settings automatically -- `fontSize: 16` on a device set to "Large" (default) renders at 16pt, but on a device set to "xxxLarge" it renders at 21.6pt (16 * 1.35).

### 8.2 Why You Must Support Dynamic Type

1. **Apple will reject your app.** Since iOS 15, Apple's App Store Review Guidelines (section 4.1) require apps to support Dynamic Type for text that is user-facing. Reviewers test at various sizes.

2. **It's the right thing to do.** Over 25% of iPhone users use a non-default text size. Many have low vision or other accessibility needs.

3. **It's the law in many jurisdictions.** ADA (US), EAA (EU), and AODA (Canada) require digital accessibility.

### 8.3 maxFontSizeMultiplier

While you must support Dynamic Type, you also need to prevent layouts from breaking at extreme sizes. `maxFontSizeMultiplier` caps the scaling:

```typescript
import { Text, TextProps } from 'react-native';

// Create a base Text component that limits maximum scaling
function AppText({ style, maxFontSizeMultiplier = 1.5, ...props }: TextProps) {
  return (
    <Text
      maxFontSizeMultiplier={maxFontSizeMultiplier}
      style={style}
      {...props}
    />
  );
}

// Usage with different limits based on context:

// Body text: allow generous scaling (1.5x is reasonable)
<AppText maxFontSizeMultiplier={1.5} style={{ fontSize: 16 }}>
  This is body text that can scale up to 24pt.
</AppText>

// Button labels: more restricted to prevent layout breakage
<AppText maxFontSizeMultiplier={1.2} style={{ fontSize: 14 }}>
  Submit
</AppText>

// Tab bar labels: most restricted (very tight layout)
<AppText maxFontSizeMultiplier={1.1} style={{ fontSize: 10 }}>
  Home
</AppText>

// Never set maxFontSizeMultiplier to 1.0 (disables scaling entirely)
// This will fail accessibility audits and App Store review.
```

**Testing Dynamic Type:**

```
iOS Simulator:
Settings → Accessibility → Display & Text Size → Larger Text → drag slider

Xcode:
Environment Overrides → Dynamic Type → select size

React Native Dev Menu:
Cannot be changed in dev menu; use device/simulator settings.
```

### 8.4 Building Scalable Layouts for Dynamic Type

The key challenge with Dynamic Type is that text can become much larger, pushing layouts beyond their bounds. Here are patterns that work:

```typescript
// BAD: Fixed height that will clip large text
<View style={{ height: 44 }}>
  <Text style={{ fontSize: 16 }}>This might get clipped</Text>
</View>

// GOOD: minHeight with flexible growth
<View style={{ minHeight: 44, justifyContent: 'center' }}>
  <Text style={{ fontSize: 16 }}>This can grow</Text>
</View>

// BAD: Horizontal layout that will overflow
<View style={{ flexDirection: 'row' }}>
  <Text style={{ fontSize: 16 }}>Long label</Text>
  <Text style={{ fontSize: 16 }}>Another label</Text>
</View>

// GOOD: Wrap to vertical at large text sizes
function AdaptiveRow({ children }: { children: React.ReactNode }) {
  const fontScale = PixelRatio.getFontScale();
  const isLargeText = fontScale > 1.2;

  return (
    <View style={{
      flexDirection: isLargeText ? 'column' : 'row',
      gap: isLargeText ? 4 : 16,
    }}>
      {children}
    </View>
  );
}

// BAD: Truncating text that the user explicitly made larger
<Text numberOfLines={1} style={{ fontSize: 16 }}>
  The user made text large because they need to read it...
</Text>

// GOOD: Allow wrapping, truncate only non-essential text
<Text
  numberOfLines={fontScale > 1.3 ? undefined : 2} // Unwrap at large sizes
  style={{ fontSize: 16 }}
>
  Important content should always be fully visible.
</Text>
```

### 8.5 Fluid Typography on Web

On the web, use `clamp()` for typography that scales smoothly between viewport sizes:

```css
/* Fluid type scale using clamp() */
:root {
  /* clamp(minimum, preferred, maximum) */
  --font-size-xs:   clamp(0.75rem,  0.7rem  + 0.25vw, 0.875rem);  /* 12-14px */
  --font-size-sm:   clamp(0.875rem, 0.8rem  + 0.375vw, 1rem);     /* 14-16px */
  --font-size-base: clamp(1rem,     0.9rem  + 0.5vw,   1.125rem);  /* 16-18px */
  --font-size-lg:   clamp(1.125rem, 1rem    + 0.625vw, 1.375rem);  /* 18-22px */
  --font-size-xl:   clamp(1.5rem,   1.25rem + 1.25vw,  2rem);      /* 24-32px */
  --font-size-2xl:  clamp(2rem,     1.5rem  + 2.5vw,   3rem);      /* 32-48px */
  --font-size-3xl:  clamp(2.5rem,   1.75rem + 3.75vw,  4rem);      /* 40-64px */
}

/* Usage */
h1 { font-size: var(--font-size-3xl); }
h2 { font-size: var(--font-size-2xl); }
h3 { font-size: var(--font-size-xl); }
p  { font-size: var(--font-size-base); }
```

With Tailwind CSS (Next.js):

```typescript
// tailwind.config.ts
export default {
  theme: {
    fontSize: {
      xs:   ['clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem)',   { lineHeight: '1.5' }],
      sm:   ['clamp(0.875rem, 0.8rem + 0.375vw, 1rem)',     { lineHeight: '1.5' }],
      base: ['clamp(1rem, 0.9rem + 0.5vw, 1.125rem)',       { lineHeight: '1.6' }],
      lg:   ['clamp(1.125rem, 1rem + 0.625vw, 1.375rem)',   { lineHeight: '1.5' }],
      xl:   ['clamp(1.5rem, 1.25rem + 1.25vw, 2rem)',       { lineHeight: '1.3' }],
      '2xl': ['clamp(2rem, 1.5rem + 2.5vw, 3rem)',          { lineHeight: '1.2' }],
      '3xl': ['clamp(2.5rem, 1.75rem + 3.75vw, 4rem)',      { lineHeight: '1.1' }],
    },
  },
};
```

### 8.6 Cross-Platform Typography Tokens

```typescript
// tokens/typography.ts

import { Platform, PixelRatio } from 'react-native';

const baseFontSize = Platform.OS === 'web' ? 16 : 16;

export const typeScale = {
  xs:   { fontSize: baseFontSize * 0.75, lineHeight: baseFontSize * 0.75 * 1.5 },   // 12
  sm:   { fontSize: baseFontSize * 0.875, lineHeight: baseFontSize * 0.875 * 1.5 },  // 14
  base: { fontSize: baseFontSize, lineHeight: baseFontSize * 1.6 },                   // 16
  lg:   { fontSize: baseFontSize * 1.125, lineHeight: baseFontSize * 1.125 * 1.5 },  // 18
  xl:   { fontSize: baseFontSize * 1.5, lineHeight: baseFontSize * 1.5 * 1.3 },      // 24
  '2xl': { fontSize: baseFontSize * 2, lineHeight: baseFontSize * 2 * 1.2 },         // 32
  '3xl': { fontSize: baseFontSize * 2.5, lineHeight: baseFontSize * 2.5 * 1.1 },     // 40
} as const;

export const fontWeights = {
  normal: '400' as const,
  medium: '500' as const,
  semibold: '600' as const,
  bold: '700' as const,
};

export const fontFamilies = Platform.select({
  ios: {
    sans: 'System', // SF Pro on iOS
    mono: 'Menlo',
  },
  android: {
    sans: 'Roboto',
    mono: 'monospace',
  },
  web: {
    sans: 'Inter, system-ui, -apple-system, sans-serif',
    mono: 'JetBrains Mono, Fira Code, monospace',
  },
})!;

// Typography presets combining scale, weight, and family
export const typography = {
  headingLarge: {
    ...typeScale['3xl'],
    fontWeight: fontWeights.bold,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.3,
  },
  headingMedium: {
    ...typeScale['2xl'],
    fontWeight: fontWeights.bold,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.3,
  },
  headingSmall: {
    ...typeScale.xl,
    fontWeight: fontWeights.semibold,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.4,
  },
  bodyLarge: {
    ...typeScale.lg,
    fontWeight: fontWeights.normal,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.5,
  },
  bodyDefault: {
    ...typeScale.base,
    fontWeight: fontWeights.normal,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.5,
  },
  bodySmall: {
    ...typeScale.sm,
    fontWeight: fontWeights.normal,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.5,
  },
  caption: {
    ...typeScale.xs,
    fontWeight: fontWeights.normal,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.3,
  },
  label: {
    ...typeScale.sm,
    fontWeight: fontWeights.medium,
    fontFamily: fontFamilies.sans,
    maxFontSizeMultiplier: 1.2,
  },
  code: {
    ...typeScale.sm,
    fontWeight: fontWeights.normal,
    fontFamily: fontFamilies.mono,
    maxFontSizeMultiplier: 1.3,
  },
} as const;
```

---

## 9. ADAPTIVE CONTENT

"Mobile-first content" does not mean "desktop content squeezed into a small screen." It means showing different content, different amounts of content, and different layouts per device category.

### 9.1 Adaptive Components

The same data, presented differently based on screen size:

```typescript
// components/ProductCard/index.tsx

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  const breakpoint = useBreakpoint();

  switch (breakpoint) {
    case 'xs':
    case 'sm':
      return <CompactProductCard product={product} />;
    case 'md':
      return <MediumProductCard product={product} />;
    case 'lg':
    case 'xl':
    case '2xl':
      return <ExpandedProductCard product={product} />;
  }
}

// Phone: compact horizontal card
function CompactProductCard({ product }: ProductCardProps) {
  return (
    <View style={styles.compactCard}>
      <Image
        source={{ uri: product.thumbnailUrl }}
        style={{ width: 80, height: 80, borderRadius: 8 }}
        contentFit="cover"
      />
      <View style={{ flex: 1, marginLeft: 12 }}>
        <Text style={typography.bodyDefault} numberOfLines={1}>
          {product.name}
        </Text>
        <Text style={typography.caption} numberOfLines={1}>
          {product.category}
        </Text>
        <Text style={typography.label}>${product.price}</Text>
      </View>
    </View>
  );
}

// Tablet: medium vertical card with more detail
function MediumProductCard({ product }: ProductCardProps) {
  return (
    <View style={styles.mediumCard}>
      <Image
        source={{ uri: product.imageUrl }}
        style={{ width: '100%', aspectRatio: 4 / 3, borderRadius: 12 }}
        contentFit="cover"
        placeholder={{ blurhash: product.blurhash }}
      />
      <View style={{ padding: 12 }}>
        <Text style={typography.bodyLarge}>{product.name}</Text>
        <Text style={typography.bodySmall} numberOfLines={2}>
          {product.description}
        </Text>
        <View style={{ flexDirection: 'row', justifyContent: 'space-between', marginTop: 8 }}>
          <Text style={typography.label}>${product.price}</Text>
          <StarRating rating={product.rating} />
        </View>
      </View>
    </View>
  );
}

// Desktop: expanded card with full description and actions
function ExpandedProductCard({ product }: ProductCardProps) {
  return (
    <View style={styles.expandedCard}>
      <Image
        source={{ uri: product.imageUrl }}
        style={{ width: '100%', aspectRatio: 16 / 9, borderRadius: 16 }}
        contentFit="cover"
        placeholder={{ blurhash: product.blurhash }}
      />
      <View style={{ padding: 16 }}>
        <Text style={typography.headingSmall}>{product.name}</Text>
        <Text style={typography.bodyDefault} numberOfLines={3}>
          {product.description}
        </Text>
        <View style={{ flexDirection: 'row', alignItems: 'center', marginTop: 12, gap: 16 }}>
          <Text style={typography.headingSmall}>${product.price}</Text>
          <StarRating rating={product.rating} showCount />
          <Text style={typography.caption}>{product.reviewCount} reviews</Text>
        </View>
        <View style={{ flexDirection: 'row', gap: 12, marginTop: 16 }}>
          <Button title="Add to Cart" variant="primary" />
          <Button title="Save" variant="outline" />
        </View>
      </View>
    </View>
  );
}
```

### 9.2 Responsive Data Fetching

Don't fetch the same payload for a phone and a desktop. A phone showing 10 compact cards doesn't need the same data depth as a desktop showing 30 expanded cards:

```typescript
// hooks/useResponsiveFeed.ts
import { useBreakpoint } from './useBreakpoint';
import { useInfiniteQuery } from '@tanstack/react-query';

interface FeedParams {
  pageSize: number;
  fields: string[];
  imageWidth: number;
}

function getFeedParams(breakpoint: string): FeedParams {
  switch (breakpoint) {
    case 'xs':
    case 'sm':
      return {
        pageSize: 10,
        fields: ['id', 'name', 'price', 'thumbnailUrl', 'category'],
        imageWidth: 160, // 80pt × 2x DPR
      };
    case 'md':
      return {
        pageSize: 20,
        fields: ['id', 'name', 'price', 'imageUrl', 'blurhash', 'category', 'description', 'rating'],
        imageWidth: 600,
      };
    default:
      return {
        pageSize: 30,
        fields: ['id', 'name', 'price', 'imageUrl', 'blurhash', 'category', 'description', 'rating', 'reviewCount'],
        imageWidth: 1200,
      };
  }
}

export function useFeed() {
  const breakpoint = useBreakpoint();
  const params = getFeedParams(breakpoint);

  return useInfiniteQuery({
    queryKey: ['feed', params],
    queryFn: ({ pageParam = 0 }) =>
      fetchFeed({
        offset: pageParam,
        limit: params.pageSize,
        fields: params.fields,
        imageWidth: params.imageWidth,
      }),
    getNextPageParam: (lastPage, pages) =>
      lastPage.hasMore ? pages.length * params.pageSize : undefined,
  });
}
```

### 9.3 Image Art Direction

Art direction means serving different crops of the same image depending on the device. A landscape hero on desktop becomes a square or portrait crop on mobile:

```typescript
// Web: <picture> element with art direction
function HeroImage({ desktopSrc, mobileSrc, alt }: {
  desktopSrc: string;
  mobileSrc: string;
  alt: string;
}) {
  return (
    <picture>
      {/* Mobile: portrait crop */}
      <source
        media="(max-width: 768px)"
        srcSet={`${mobileSrc}?w=750&h=1000&fit=crop 1x, ${mobileSrc}?w=1500&h=2000&fit=crop 2x`}
      />
      {/* Desktop: landscape crop */}
      <source
        media="(min-width: 769px)"
        srcSet={`${desktopSrc}?w=1200&h=630&fit=crop 1x, ${desktopSrc}?w=2400&h=1260&fit=crop 2x`}
      />
      <img
        src={`${desktopSrc}?w=1200&h=630&fit=crop`}
        alt={alt}
        style={{ width: '100%', height: 'auto' }}
      />
    </picture>
  );
}

// React Native: conditional source based on screen
function HeroImage({ imageId, alt }: { imageId: string; alt: string }) {
  const { width, height } = useWindowDimensions();
  const isPortrait = height > width;
  const dpr = Math.min(Math.ceil(PixelRatio.get()), 3);

  const uri = getImageUrl(imageId, {
    width: isPortrait ? width : Math.min(width, 1200),
    height: isPortrait ? width * 1.33 : Math.min(width, 1200) * 0.525,
    fit: 'cover',
    dpr,
  });

  return (
    <ExpoImage
      source={{ uri }}
      alt={alt}
      contentFit="cover"
      style={{
        width: '100%',
        aspectRatio: isPortrait ? 3 / 4 : 16 / 9,
      }}
    />
  );
}
```

### 9.4 Content Truncation Patterns

```typescript
// Show a preview on mobile, full content on tablet/desktop
function ArticleContent({ article }: { article: Article }) {
  const isCompact = !useIsAbove('md');
  const [isExpanded, setIsExpanded] = useState(false);

  if (!isCompact) {
    // Tablet/desktop: show full content
    return <MarkdownRenderer content={article.body} />;
  }

  // Phone: show preview with "Read more"
  return (
    <View>
      <Text
        style={typography.bodyDefault}
        numberOfLines={isExpanded ? undefined : 4}
      >
        {article.body}
      </Text>
      {!isExpanded && (
        <Pressable onPress={() => setIsExpanded(true)}>
          <Text style={[typography.label, { color: theme.colors.primary }]}>
            Read more
          </Text>
        </Pressable>
      )}
    </View>
  );
}
```

### 9.5 Platform-Specific Features

Some features only make sense on certain platforms. Don't show them where they can't work:

```typescript
import { Platform } from 'react-native';
import * as Haptics from 'expo-haptics';

function ActionMenu({ item }: { item: Item }) {
  const isWeb = Platform.OS === 'web';

  return (
    <Menu>
      <Menu.Item
        title="Share"
        onPress={() => share(item)}
        // Keyboard shortcut only shown on web
        shortcut={isWeb ? '⌘S' : undefined}
      />

      {/* Camera feature only on mobile */}
      {!isWeb && (
        <Menu.Item
          title="Scan Barcode"
          onPress={async () => {
            await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
            openBarcodeScanner();
          }}
        />
      )}

      {/* Keyboard shortcuts only on web */}
      {isWeb && (
        <Menu.Item
          title="Keyboard Shortcuts"
          onPress={() => showShortcutsDialog()}
          shortcut="⌘/"
        />
      )}
    </Menu>
  );
}
```

---

## 10. LAZY LOADING AND PROGRESSIVE RENDERING

Loading everything upfront is the fastest way to a slow app. Lazy loading defers work until it's needed. Progressive rendering shows something immediately and improves it over time.

### 10.1 Intersection Observer for Web

```typescript
// hooks/useLazyLoad.ts (Web)
import { useRef, useState, useEffect } from 'react';

export function useLazyLoad(options?: IntersectionObserverInit) {
  const ref = useRef<HTMLDivElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect(); // Only trigger once
        }
      },
      {
        rootMargin: '200px', // Start loading 200px before visible
        threshold: 0,
        ...options,
      }
    );

    observer.observe(element);
    return () => observer.disconnect();
  }, []);

  return { ref, isVisible };
}

// Usage in a component:
function LazySection({ children }: { children: React.ReactNode }) {
  const { ref, isVisible } = useLazyLoad();

  return (
    <div ref={ref}>
      {isVisible ? children : <Skeleton height={300} />}
    </div>
  );
}
```

### 10.2 FlashList Viewability Callbacks

```typescript
import { FlashList, ViewToken } from '@shopify/flash-list';
import { useCallback, useRef } from 'react';

function Feed({ items }: { items: FeedItem[] }) {
  const visibleItems = useRef<Set<string>>(new Set());

  const onViewableItemsChanged = useCallback(
    ({ viewableItems, changed }: { viewableItems: ViewToken[]; changed: ViewToken[] }) => {
      // Track which items are currently visible
      const newVisible = new Set(viewableItems.map((v) => v.key as string));
      visibleItems.current = newVisible;

      // Items that just became visible — start loading high-res images
      changed
        .filter((item) => item.isViewable)
        .forEach((item) => {
          // Prefetch the next screen's worth of images
          const index = item.index ?? 0;
          for (let i = index + 1; i <= index + 5 && i < items.length; i++) {
            Image.prefetch(items[i].imageUrl);
          }
        });
    },
    [items]
  );

  return (
    <FlashList
      data={items}
      renderItem={({ item }) => <FeedCard item={item} />}
      estimatedItemSize={300}
      onViewableItemsChanged={onViewableItemsChanged}
      viewabilityConfig={{
        itemVisiblePercentThreshold: 50,
        minimumViewTime: 100,
      }}
    />
  );
}
```

### 10.3 Three-Stage Progressive Loading

The gold standard for image loading: blurhash (instant) -> LQIP (fast) -> full resolution (complete).

```typescript
// components/ProgressiveImage.tsx
import { Image } from 'expo-image';
import { useState, useCallback } from 'react';

interface ProgressiveImageProps {
  blurhash: string;
  lqipUrl?: string; // Low-quality image placeholder URL (~200 bytes)
  fullUrl: string;
  style: any;
  contentFit?: 'cover' | 'contain';
}

export function ProgressiveImage({
  blurhash,
  lqipUrl,
  fullUrl,
  style,
  contentFit = 'cover',
}: ProgressiveImageProps) {
  // expo-image handles this natively with the placeholder prop.
  // The transition between placeholder and full image is smooth.
  // If you want a 3-stage load (blurhash → LQIP → full), you need
  // to manage it manually:

  const [stage, setStage] = useState<'blurhash' | 'lqip' | 'full'>('blurhash');

  const handleLqipLoad = useCallback(() => {
    setStage('lqip');
  }, []);

  const handleFullLoad = useCallback(() => {
    setStage('full');
  }, []);

  return (
    <View style={[style, { overflow: 'hidden' }]}>
      {/* Stage 1: Blurhash (rendered instantly from hash string) */}
      {stage === 'blurhash' && (
        <Image
          placeholder={{ blurhash }}
          style={StyleSheet.absoluteFill}
          contentFit={contentFit}
        />
      )}

      {/* Stage 2: LQIP (tiny real image, loads fast) */}
      {lqipUrl && stage !== 'full' && (
        <Image
          source={{ uri: lqipUrl }}
          style={StyleSheet.absoluteFill}
          contentFit={contentFit}
          onLoad={handleLqipLoad}
          blurRadius={2}
        />
      )}

      {/* Stage 3: Full resolution */}
      <Image
        source={{ uri: fullUrl }}
        style={StyleSheet.absoluteFill}
        contentFit={contentFit}
        onLoad={handleFullLoad}
        transition={300}
        placeholder={lqipUrl ? undefined : { blurhash }}
      />
    </View>
  );
}
```

In practice, expo-image's built-in `placeholder` + `transition` handles the common case (blurhash -> full) beautifully. The three-stage approach is only needed when you want the intermediate LQIP step for very large images that take a long time to load.

### 10.4 Skeleton Screens

Skeleton screens are better than spinners because they communicate the shape of the incoming content, setting user expectations:

```typescript
// components/Skeleton.tsx
import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withRepeat,
  withTiming,
  interpolateColor,
} from 'react-native-reanimated';
import { useEffect } from 'react';

interface SkeletonProps {
  width: number | string;
  height: number;
  borderRadius?: number;
}

export function Skeleton({ width, height, borderRadius = 8 }: SkeletonProps) {
  const progress = useSharedValue(0);

  useEffect(() => {
    progress.value = withRepeat(
      withTiming(1, { duration: 1000 }),
      -1, // infinite
      true  // reverse
    );
  }, []);

  const animatedStyle = useAnimatedStyle(() => ({
    backgroundColor: interpolateColor(
      progress.value,
      [0, 1],
      ['#E5E7EB', '#F3F4F6'] // gray-200 to gray-100
    ),
  }));

  return (
    <Animated.View
      style={[
        {
          width,
          height,
          borderRadius,
        },
        animatedStyle,
      ]}
    />
  );
}

// Skeleton for a feed card
function FeedCardSkeleton() {
  return (
    <View style={styles.card}>
      <Skeleton width="100%" height={200} borderRadius={12} />
      <View style={{ padding: 12, gap: 8 }}>
        <Skeleton width="60%" height={20} />
        <Skeleton width="100%" height={14} />
        <Skeleton width="80%" height={14} />
        <View style={{ flexDirection: 'row', gap: 12, marginTop: 8 }}>
          <Skeleton width={60} height={24} borderRadius={12} />
          <Skeleton width={80} height={24} borderRadius={12} />
        </View>
      </View>
    </View>
  );
}

// Usage in a loading state
function FeedScreen() {
  const { data, isLoading } = useFeed();

  if (isLoading) {
    return (
      <ScrollView>
        {Array.from({ length: 5 }).map((_, i) => (
          <FeedCardSkeleton key={i} />
        ))}
      </ScrollView>
    );
  }

  return <FeedList data={data} />;
}
```

### 10.5 Above-the-Fold Priority

Load critical images first, defer everything below the fold:

```typescript
// Determine if an item is "above the fold" based on its position
function FeedCard({ item, index }: { item: FeedItem; index: number }) {
  // First 3 items are likely above the fold on most devices
  const isAboveFold = index < 3;

  return (
    <View style={styles.card}>
      <Image
        source={{ uri: item.imageUrl }}
        placeholder={isAboveFold ? undefined : { blurhash: item.blurhash }}
        // Priority items: don't show placeholder, load immediately
        // Below-fold items: show placeholder, lazy load
        priority={isAboveFold ? 'high' : 'low'}
        cachePolicy="memory-disk"
        contentFit="cover"
        transition={isAboveFold ? 0 : 200}
        style={styles.cardImage}
      />
    </View>
  );
}

// Next.js: priority prop for LCP image
<Image
  src={heroImageUrl}
  alt="Hero"
  width={1200}
  height={630}
  priority // Adds <link rel="preload">, disables lazy loading
  sizes="100vw"
/>
```

### 10.6 Bandwidth-Aware Loading

Detect slow connections and serve lower quality images:

```typescript
// hooks/useNetworkQuality.ts
import NetInfo, { NetInfoStateType } from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';

type NetworkQuality = 'high' | 'medium' | 'low' | 'offline';

export function useNetworkQuality(): NetworkQuality {
  const [quality, setQuality] = useState<NetworkQuality>('high');

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      if (!state.isConnected) {
        setQuality('offline');
        return;
      }

      switch (state.type) {
        case NetInfoStateType.wifi:
        case NetInfoStateType.ethernet:
          setQuality('high');
          break;
        case NetInfoStateType.cellular:
          // Check cellular generation
          const details = state.details;
          if (details?.cellularGeneration === '5g' || details?.cellularGeneration === '4g') {
            setQuality('medium');
          } else {
            setQuality('low');
          }
          break;
        default:
          setQuality('medium');
      }
    });

    return () => unsubscribe();
  }, []);

  return quality;
}

// Use network quality to adjust image loading
function NetworkAwareImage({ imageId, width, height, ...props }: ImageProps) {
  const quality = useNetworkQuality();

  const imageQuality = {
    high: 80,
    medium: 60,
    low: 40,
    offline: 30,
  }[quality];

  const dprCap = {
    high: 3,
    medium: 2,
    low: 1,
    offline: 1,
  }[quality];

  const uri = getImageUrl(imageId, {
    width,
    height,
    quality: imageQuality,
    dpr: Math.min(PixelRatio.get(), dprCap),
    format: quality === 'low' ? 'jpeg' : 'auto', // JPEG decodes faster
  });

  return (
    <ExpoImage
      source={{ uri }}
      {...props}
    />
  );
}

// Web: Network Information API
function useWebNetworkQuality(): NetworkQuality {
  const [quality, setQuality] = useState<NetworkQuality>('high');

  useEffect(() => {
    const connection = (navigator as any).connection;
    if (!connection) return;

    function update() {
      const { effectiveType, saveData } = connection;
      if (saveData) {
        setQuality('low');
      } else if (effectiveType === '4g') {
        setQuality('high');
      } else if (effectiveType === '3g') {
        setQuality('medium');
      } else {
        setQuality('low');
      }
    }

    update();
    connection.addEventListener('change', update);
    return () => connection.removeEventListener('change', update);
  }, []);

  return quality;
}
```

---

## 11. VIDEO AND RICH MEDIA

Images are just one part of the media story. Modern apps include video, audio, and complex animations.

### 11.1 Video Optimization

**The golden rule:** Never serve a single video file. Always use adaptive bitrate streaming (HLS or DASH) that adjusts quality based on the user's bandwidth.

```
Single file approach (DON'T DO THIS):
- 1080p video at 5 Mbps → user on 3G sees buffering
- 360p video at 1 Mbps → user on Wi-Fi sees potato quality

Adaptive bitrate (HLS):
- Server provides multiple quality levels
- Player automatically switches based on bandwidth
- 360p (1 Mbps) → 720p (3 Mbps) → 1080p (5 Mbps)
```

**Generating HLS with ffmpeg:**

```bash
# Generate HLS with multiple quality levels
ffmpeg -i input.mp4 \
  -filter_complex "[0:v]split=3[v1][v2][v3]; \
    [v1]scale=640:360[v1out]; \
    [v2]scale=1280:720[v2out]; \
    [v3]scale=1920:1080[v3out]" \
  -map "[v1out]" -c:v h264 -b:v 1M -maxrate 1.2M -bufsize 2M \
    -hls_time 6 -hls_list_size 0 -f hls output_360p.m3u8 \
  -map "[v2out]" -c:v h264 -b:v 3M -maxrate 3.6M -bufsize 6M \
    -hls_time 6 -hls_list_size 0 -f hls output_720p.m3u8 \
  -map "[v3out]" -c:v h264 -b:v 5M -maxrate 6M -bufsize 10M \
    -hls_time 6 -hls_list_size 0 -f hls output_1080p.m3u8
```

Most teams use a service (Mux, Cloudflare Stream, AWS MediaConvert, Cloudinary video) instead of managing HLS encoding themselves.

### 11.2 expo-video Player

```typescript
import { useVideoPlayer, VideoView } from 'expo-video';

function VideoPlayerScreen({ videoUrl }: { videoUrl: string }) {
  const player = useVideoPlayer(videoUrl, (player) => {
    player.loop = false;
    player.muted = false;
    player.playbackRate = 1.0;
  });

  return (
    <VideoView
      player={player}
      style={{ width: '100%', aspectRatio: 16 / 9 }}
      allowsFullscreen
      allowsPictureInPicture
      contentFit="contain"
    />
  );
}
```

### 11.3 Video in Lists (Autoplay on Viewability)

The pattern: autoplay video when it's in the viewport, pause when it scrolls away. This prevents multiple videos from playing simultaneously (which destroys performance and confuses the user).

```typescript
import { useVideoPlayer, VideoView } from 'expo-video';
import { FlashList, ViewToken } from '@shopify/flash-list';
import { useState, useCallback, useRef, createContext, useContext } from 'react';

// Context to track which video is currently "active"
const ActiveVideoContext = createContext<{
  activeId: string | null;
  setActiveId: (id: string | null) => void;
}>({ activeId: null, setActiveId: () => {} });

function VideoFeed({ items }: { items: FeedItem[] }) {
  const [activeId, setActiveId] = useState<string | null>(null);

  const onViewableItemsChanged = useCallback(
    ({ viewableItems }: { viewableItems: ViewToken[] }) => {
      // Find the most centered viewable video item
      const videoItems = viewableItems.filter(
        (v) => v.item.type === 'video' && v.isViewable
      );

      if (videoItems.length > 0) {
        setActiveId(videoItems[0].key as string);
      } else {
        setActiveId(null);
      }
    },
    []
  );

  return (
    <ActiveVideoContext.Provider value={{ activeId, setActiveId }}>
      <FlashList
        data={items}
        renderItem={({ item }) =>
          item.type === 'video' ? (
            <VideoCard item={item} />
          ) : (
            <ImageCard item={item} />
          )
        }
        estimatedItemSize={400}
        onViewableItemsChanged={onViewableItemsChanged}
        viewabilityConfig={{
          itemVisiblePercentThreshold: 60,
          minimumViewTime: 500,
        }}
      />
    </ActiveVideoContext.Provider>
  );
}

function VideoCard({ item }: { item: FeedItem }) {
  const { activeId } = useContext(ActiveVideoContext);
  const isActive = activeId === item.id;

  const player = useVideoPlayer(item.videoUrl, (player) => {
    player.loop = true;
    player.muted = true; // Muted autoplay is the standard
  });

  useEffect(() => {
    if (isActive) {
      player.play();
    } else {
      player.pause();
    }
  }, [isActive, player]);

  return (
    <View style={styles.videoCard}>
      <VideoView
        player={player}
        style={{ width: '100%', aspectRatio: 16 / 9 }}
        nativeControls={false}
      />
      {/* Thumbnail overlay when paused */}
      {!isActive && (
        <Image
          source={{ uri: item.thumbnailUrl }}
          style={StyleSheet.absoluteFill}
          contentFit="cover"
        />
      )}
    </View>
  );
}
```

### 11.4 Audio

```typescript
import { useAudioPlayer, AudioPlayer } from 'expo-audio';

function AudioPlayerComponent({ audioUrl, title }: { audioUrl: string; title: string }) {
  const player = useAudioPlayer(audioUrl);

  return (
    <View style={styles.audioPlayer}>
      <Text style={typography.label}>{title}</Text>
      <View style={{ flexDirection: 'row', alignItems: 'center', gap: 12 }}>
        <Pressable onPress={() => (player.playing ? player.pause() : player.play())}>
          <Icon name={player.playing ? 'pause' : 'play'} size={24} />
        </Pressable>
        <ProgressBar
          progress={player.currentTime / player.duration}
          style={{ flex: 1 }}
        />
        <Text style={typography.caption}>
          {formatDuration(player.currentTime)} / {formatDuration(player.duration)}
        </Text>
      </View>
    </View>
  );
}
```

### 11.5 Lottie Animations

Lottie renders After Effects animations as JSON, producing smooth 60fps animations at a fraction of the file size of video or GIF.

```typescript
import LottieView from 'lottie-react-native';
import { useRef } from 'react';

// Basic usage
function SuccessAnimation() {
  return (
    <LottieView
      source={require('@/assets/animations/success.json')}
      autoPlay
      loop={false}
      style={{ width: 150, height: 150 }}
    />
  );
}

// Controlled playback (e.g., pull-to-refresh animation that tracks scroll)
function PullToRefreshAnimation({ progress }: { progress: number }) {
  const animationRef = useRef<LottieView>(null);

  useEffect(() => {
    // progress is 0-1 based on pull distance
    // Map to animation frames
    animationRef.current?.play(0, Math.floor(progress * 60));
  }, [progress]);

  return (
    <LottieView
      ref={animationRef}
      source={require('@/assets/animations/pull-refresh.json')}
      style={{ width: 60, height: 60 }}
      loop={false}
      autoPlay={false}
    />
  );
}

// Performance considerations:
// 1. Keep Lottie JSON under 50KB (complex animations should be simplified in After Effects)
// 2. Avoid Lottie in lists — use it for one-off animations (onboarding, success, empty states)
// 3. Use renderMode="HARDWARE" on Android for GPU acceleration
// 4. Stop or remove Lottie animations when not visible

<LottieView
  source={animationData}
  renderMode={Platform.OS === 'android' ? 'HARDWARE' : undefined}
  autoPlay
  loop
/>
```

---

## 12. BUILDING A UNIVERSAL MEDIA SYSTEM

This section ties everything together into a cohesive system. Instead of ad-hoc image handling scattered across your codebase, you build a centralized media system that handles optimization, format selection, responsive sizing, placeholders, and caching in one place.

### 12.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    UNIVERSAL MEDIA SYSTEM                        │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  <OptimizedImage> │  │  <ProgressiveImg>│  │  <VideoPlayer>│  │
│  │  (RN + Web)       │  │  (3-stage load)  │  │  (RN + Web)  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬───────┘  │
│           │                      │                    │          │
│  ┌────────▼──────────────────────▼────────────────────▼───────┐ │
│  │              IMAGE URL BUILDER                              │ │
│  │  getImageUrl(id, { width, height, dpr, quality, format })  │ │
│  │  Provider-agnostic: Cloudinary / Imgix / Cloudflare         │ │
│  └────────────────────────────┬────────────────────────────────┘ │
│                               │                                  │
│  ┌────────────────────────────▼────────────────────────────────┐ │
│  │              PLACEHOLDER PIPELINE                            │ │
│  │  sharp → blurhash/thumbhash → stored in DB with image URL   │ │
│  │  Served alongside image URL in API responses                 │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              RESPONSIVE CONFIG                               │ │
│  │  Breakpoints + Pixel density + Network quality → image size  │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 The Configuration Layer

```typescript
// config/media.ts

export const mediaConfig = {
  cdn: {
    provider: 'cloudinary' as const,
    baseUrl: 'https://res.cloudinary.com',
    cloudName: process.env.CLOUDINARY_CLOUD_NAME!,
  },

  // Standard image sizes (width in points)
  sizes: {
    thumbnail: 80,
    small: 200,
    medium: 400,
    large: 800,
    hero: 1200,
    full: 2400,
  },

  // Quality per network condition
  quality: {
    high: 80,
    medium: 65,
    low: 45,
  },

  // DPR caps per network condition
  maxDpr: {
    high: 3,
    medium: 2,
    low: 1,
  },

  // Default placeholder strategy
  placeholder: 'blurhash' as 'blurhash' | 'thumbhash' | 'lqip' | 'color',

  // Cache durations
  cache: {
    memory: 50 * 1024 * 1024,   // 50MB memory cache
    disk: 200 * 1024 * 1024,     // 200MB disk cache
    ttl: 7 * 24 * 60 * 60,       // 7 days
  },

  // Format preference (first supported format wins)
  formatPreference: ['avif', 'webp', 'jpeg'] as const,

  // Breakpoints (matches Tailwind)
  breakpoints: {
    sm: 640,
    md: 768,
    lg: 1024,
    xl: 1280,
    '2xl': 1536,
  },
} as const;
```

### 12.3 The OptimizedImage Component (Full Implementation)

```typescript
// components/OptimizedImage/index.tsx

import { Platform, PixelRatio, StyleSheet, View } from 'react-native';
import { Image as ExpoImage, ImageProps as ExpoImageProps } from 'expo-image';
import NextImage, { ImageProps as NextImageProps } from 'next/image';
import { getImageUrl } from '@/lib/image-url';
import { useNetworkQuality } from '@/hooks/useNetworkQuality';
import { mediaConfig } from '@/config/media';

// Unified props that work on both platforms
export interface OptimizedImageProps {
  /** Image identifier (path or public ID in your CDN) */
  imageId: string;
  /** Alt text for accessibility */
  alt: string;
  /** Display width in points (required for optimization) */
  width: number;
  /** Display height in points. If omitted, uses aspectRatio. */
  height?: number;
  /** Aspect ratio (e.g., 16/9). Used when height is not specified. */
  aspectRatio?: number;
  /** How the image fits its container */
  fit?: 'cover' | 'contain';
  /** Blurhash string for placeholder */
  blurhash?: string;
  /** Thumbhash string for placeholder */
  thumbhash?: string;
  /** Whether this is an above-the-fold / LCP image */
  priority?: boolean;
  /** Override quality (1-100) */
  quality?: number;
  /** Unique key for recycling in lists */
  recyclingKey?: string;
  /** Additional styles */
  style?: any;
  /** Responsive sizes string for web (e.g., "(max-width: 768px) 100vw, 50vw") */
  sizes?: string;
  /** Callback when image loads */
  onLoad?: () => void;
  /** Callback on error */
  onError?: (error: any) => void;
}

export function OptimizedImage({
  imageId,
  alt,
  width,
  height,
  aspectRatio,
  fit = 'cover',
  blurhash,
  thumbhash,
  priority = false,
  quality,
  recyclingKey,
  style,
  sizes,
  onLoad,
  onError,
}: OptimizedImageProps) {
  const networkQuality = useNetworkQuality();

  // Calculate actual height from aspect ratio if not provided
  const displayHeight = height || (aspectRatio ? Math.round(width / aspectRatio) : width);

  // Determine quality and DPR based on network conditions
  const effectiveQuality = quality || mediaConfig.quality[networkQuality === 'offline' ? 'low' : networkQuality];
  const maxDpr = mediaConfig.maxDpr[networkQuality === 'offline' ? 'low' : networkQuality];

  if (Platform.OS === 'web') {
    return (
      <WebOptimizedImage
        imageId={imageId}
        alt={alt}
        width={width}
        height={displayHeight}
        fit={fit}
        blurhash={blurhash}
        priority={priority}
        quality={effectiveQuality}
        sizes={sizes || `${width}px`}
        style={style}
        onLoad={onLoad}
      />
    );
  }

  // React Native
  const dpr = Math.min(Math.ceil(PixelRatio.get()), maxDpr);
  const uri = getImageUrl(imageId, {
    width,
    height: displayHeight,
    dpr,
    quality: effectiveQuality,
    format: 'auto',
  }, mediaConfig.cdn);

  // Build placeholder
  const placeholder = thumbhash
    ? { thumbhash }
    : blurhash
      ? { blurhash }
      : undefined;

  return (
    <ExpoImage
      source={{ uri }}
      alt={alt}
      contentFit={fit}
      placeholder={priority ? undefined : placeholder}
      transition={priority ? 0 : 200}
      cachePolicy="memory-disk"
      recyclingKey={recyclingKey}
      priority={priority ? 'high' : 'normal'}
      style={[
        {
          width,
          height: height || undefined,
          aspectRatio: !height && aspectRatio ? aspectRatio : undefined,
        },
        style,
      ]}
      onLoad={onLoad}
      onError={onError}
    />
  );
}

// Web-specific implementation using next/image
function WebOptimizedImage({
  imageId,
  alt,
  width,
  height,
  fit,
  blurhash,
  priority,
  quality,
  sizes,
  style,
  onLoad,
}: {
  imageId: string;
  alt: string;
  width: number;
  height: number;
  fit: 'cover' | 'contain';
  blurhash?: string;
  priority: boolean;
  quality: number;
  sizes: string;
  style?: any;
  onLoad?: () => void;
}) {
  return (
    <NextImage
      src={imageId}
      alt={alt}
      width={width}
      height={height}
      quality={quality}
      priority={priority}
      sizes={sizes}
      style={{ objectFit: fit, ...style }}
      placeholder={blurhash ? 'blur' : 'empty'}
      blurDataURL={blurhash ? blurhashToDataURL(blurhash) : undefined}
      onLoad={onLoad}
    />
  );
}

// Convert blurhash to a data URL for next/image's blurDataURL
function blurhashToDataURL(blurhash: string): string {
  // In practice, you'd use the blurhash library to decode and render
  // to a small canvas, then export as data URL. Or compute this server-side.
  // For brevity, returning a placeholder here.
  // See: https://github.com/woltapp/blurhash/tree/master/TypeScript
  return `data:image/svg+xml;charset=utf-8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 4 3'></svg>`;
}
```

### 12.4 Placeholder Generation Pipeline

Run this as part of your image upload pipeline (e.g., in a serverless function triggered by S3 upload):

```typescript
// services/image-pipeline.ts

import sharp from 'sharp';
import { encode as encodeBlurhash } from 'blurhash';
import { rgbaToThumbHash } from 'thumbhash';
import { getImageUrl } from '@/lib/image-url';

interface ImageMetadata {
  url: string;
  width: number;
  height: number;
  blurhash: string;
  thumbhash: string;
  dominantColor: string;
  aspectRatio: number;
  // Pre-computed responsive URLs
  urls: {
    thumbnail: string;
    small: string;
    medium: string;
    large: string;
  };
}

export async function processUploadedImage(
  imageBuffer: Buffer,
  publicId: string
): Promise<ImageMetadata> {
  const image = sharp(imageBuffer);
  const metadata = await image.metadata();
  const width = metadata.width!;
  const height = metadata.height!;

  // Generate blurhash
  const blurhashBuffer = await image
    .raw()
    .ensureAlpha()
    .resize(32, 32, { fit: 'inside' })
    .toBuffer({ resolveWithObject: true });

  const blurhash = encodeBlurhash(
    new Uint8ClampedArray(blurhashBuffer.data),
    blurhashBuffer.info.width,
    blurhashBuffer.info.height,
    4,
    3
  );

  // Generate thumbhash
  const thumbhashBuffer = await image
    .raw()
    .ensureAlpha()
    .resize(100, 100, { fit: 'inside' })
    .toBuffer({ resolveWithObject: true });

  const thumbhashBytes = rgbaToThumbHash(
    thumbhashBuffer.info.width,
    thumbhashBuffer.info.height,
    new Uint8Array(thumbhashBuffer.data)
  );
  const thumbhash = Buffer.from(thumbhashBytes).toString('base64');

  // Extract dominant color
  const { dominant } = await image.stats();
  const dominantColor = `#${dominant.r.toString(16).padStart(2, '0')}${dominant.g.toString(16).padStart(2, '0')}${dominant.b.toString(16).padStart(2, '0')}`;

  // Pre-compute responsive URLs
  const aspectRatio = width / height;
  const urls = {
    thumbnail: getImageUrl(publicId, { width: 80, height: Math.round(80 / aspectRatio), quality: 60 }),
    small: getImageUrl(publicId, { width: 400, height: Math.round(400 / aspectRatio), quality: 75 }),
    medium: getImageUrl(publicId, { width: 800, height: Math.round(800 / aspectRatio), quality: 75 }),
    large: getImageUrl(publicId, { width: 1200, height: Math.round(1200 / aspectRatio), quality: 80 }),
  };

  return {
    url: getImageUrl(publicId, { width, height, quality: 85 }),
    width,
    height,
    blurhash,
    thumbhash,
    dominantColor,
    aspectRatio,
    urls,
  };
}
```

### 12.5 API Response Structure

Your API should return image metadata alongside the image URL:

```typescript
// Example API response for a feed item
interface FeedItemResponse {
  id: string;
  title: string;
  description: string;
  image: {
    id: string;           // CDN public ID
    width: number;        // Original width
    height: number;       // Original height
    aspectRatio: number;  // width / height
    blurhash: string;     // "LKO2:N%2Tw=w]~RBVZRi};RPxuwH"
    thumbhash: string;    // "k0oGDQaSVnOIeHegh5d3d0iI+IVAV4Rx"
    dominantColor: string; // "#3B82F6"
    urls: {
      thumbnail: string;  // 80px wide
      small: string;      // 400px wide
      medium: string;     // 800px wide
      large: string;      // 1200px wide
    };
  };
  createdAt: string;
}
```

Usage in a component:

```typescript
function FeedCard({ item }: { item: FeedItemResponse }) {
  const isTablet = useIsAbove('md');

  return (
    <View style={styles.card}>
      <OptimizedImage
        imageId={item.image.id}
        alt={item.title}
        width={isTablet ? 400 : 375}
        aspectRatio={item.image.aspectRatio}
        blurhash={item.image.blurhash}
        recyclingKey={item.id}
        priority={false}
      />
      <View style={styles.cardContent}>
        <Text style={typography.bodyDefault}>{item.title}</Text>
        {isTablet && (
          <Text style={typography.bodySmall} numberOfLines={2}>
            {item.description}
          </Text>
        )}
      </View>
    </View>
  );
}
```

### 12.6 Image Preloading Strategy

```typescript
// hooks/useImagePreloader.ts

import { Image } from 'expo-image';
import { useEffect, useRef } from 'react';

/**
 * Preloads images for upcoming screens based on user behavior patterns.
 * Call this in your navigation container or screen wrapper.
 */
export function useImagePreloader(currentRoute: string, feedItems?: FeedItemResponse[]) {
  const preloadedRef = useRef<Set<string>>(new Set());

  useEffect(() => {
    if (!feedItems) return;

    // Strategy: Preload the first 5 items' images when the feed loads
    const imagesToPreload = feedItems
      .slice(0, 5)
      .map((item) => item.image.urls.medium)
      .filter((url) => !preloadedRef.current.has(url));

    if (imagesToPreload.length > 0) {
      Image.prefetch(imagesToPreload);
      imagesToPreload.forEach((url) => preloadedRef.current.add(url));
    }
  }, [feedItems]);

  // When user is likely to tap a card (long press, etc.), preload the detail image
  const preloadDetailImage = (item: FeedItemResponse) => {
    const url = item.image.urls.large;
    if (!preloadedRef.current.has(url)) {
      Image.prefetch(url);
      preloadedRef.current.add(url);
    }
  };

  return { preloadDetailImage };
}
```

### 12.7 Complete Example: A Feed Screen

Putting it all together:

```typescript
// screens/FeedScreen.tsx

import { FlashList, ViewToken } from '@shopify/flash-list';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import { Image } from 'expo-image';
import { useCallback, useState } from 'react';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { useRouter } from 'expo-router';

import { OptimizedImage } from '@/components/OptimizedImage';
import { Skeleton } from '@/components/Skeleton';
import { useFeed } from '@/hooks/useFeed';
import { useBreakpoint, useIsAbove } from '@/hooks/useBreakpoint';
import { useImagePreloader } from '@/hooks/useImagePreloader';
import { typography } from '@/tokens/typography';

export default function FeedScreen() {
  const insets = useSafeAreaInsets();
  const router = useRouter();
  const isTablet = useIsAbove('md');
  const breakpoint = useBreakpoint();
  const { data, isLoading, fetchNextPage, hasNextPage } = useFeed();

  const items = data?.pages.flatMap((page) => page.items) ?? [];
  const { preloadDetailImage } = useImagePreloader('feed', items);

  // Determine grid columns based on breakpoint
  const columns = breakpoint === 'xs' || breakpoint === 'sm' ? 1
    : breakpoint === 'md' ? 2
    : 3;

  const handlePressIn = useCallback((item: FeedItemResponse) => {
    preloadDetailImage(item);
  }, [preloadDetailImage]);

  const handlePress = useCallback((item: FeedItemResponse) => {
    router.push(`/feed/${item.id}`);
  }, [router]);

  const renderItem = useCallback(({ item, index }: { item: FeedItemResponse; index: number }) => {
    const isAboveFold = index < (columns * 2); // First 2 rows are "above the fold"

    return (
      <Pressable
        onPressIn={() => handlePressIn(item)}
        onPress={() => handlePress(item)}
        style={[styles.cardWrapper, { flex: 1 / columns }]}
      >
        <View style={styles.card}>
          <OptimizedImage
            imageId={item.image.id}
            alt={item.title}
            width={isTablet ? 400 : 375}
            aspectRatio={item.image.aspectRatio}
            blurhash={item.image.blurhash}
            priority={isAboveFold}
            recyclingKey={item.id}
          />
          <View style={styles.cardContent}>
            <Text style={typography.bodyDefault} numberOfLines={isTablet ? 2 : 1}>
              {item.title}
            </Text>
            {isTablet && (
              <Text style={typography.bodySmall} numberOfLines={2}>
                {item.description}
              </Text>
            )}
            <Text style={[typography.caption, styles.timestamp]}>
              {formatRelativeTime(item.createdAt)}
            </Text>
          </View>
        </View>
      </Pressable>
    );
  }, [columns, isTablet, handlePressIn, handlePress]);

  if (isLoading) {
    return (
      <View style={[styles.container, { paddingTop: insets.top }]}>
        {Array.from({ length: 6 }).map((_, i) => (
          <View key={i} style={styles.cardWrapper}>
            <View style={styles.card}>
              <Skeleton width="100%" height={200} borderRadius={12} />
              <View style={{ padding: 12, gap: 8 }}>
                <Skeleton width="70%" height={18} />
                <Skeleton width="100%" height={14} />
                <Skeleton width="40%" height={12} />
              </View>
            </View>
          </View>
        ))}
      </View>
    );
  }

  return (
    <View style={[styles.container, { paddingTop: insets.top }]}>
      <FlashList
        data={items}
        renderItem={renderItem}
        estimatedItemSize={320}
        numColumns={columns}
        keyExtractor={(item) => item.id}
        onEndReached={() => hasNextPage && fetchNextPage()}
        onEndReachedThreshold={0.5}
        contentContainerStyle={{ padding: 8 }}
        ItemSeparatorComponent={() => <View style={{ height: 8 }} />}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#F9FAFB',
  },
  cardWrapper: {
    padding: 8,
  },
  card: {
    backgroundColor: '#FFFFFF',
    borderRadius: 16,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 3,
    elevation: 2,
  },
  cardContent: {
    padding: 12,
    gap: 4,
  },
  timestamp: {
    color: '#9CA3AF',
    marginTop: 4,
  },
});

function formatRelativeTime(isoDate: string): string {
  const diff = Date.now() - new Date(isoDate).getTime();
  const minutes = Math.floor(diff / 60000);
  if (minutes < 60) return `${minutes}m ago`;
  const hours = Math.floor(minutes / 60);
  if (hours < 24) return `${hours}h ago`;
  const days = Math.floor(hours / 24);
  return `${days}d ago`;
}
```

---

## 13. PERFORMANCE CHEAT SHEET

A quick reference for the decisions covered in this chapter.

### Image Format Selection

| Content Type | Format | Quality | Expected Size (800px wide) |
|-------------|--------|---------|---------------------------|
| Photograph | WebP | 75 | 60-120 KB |
| Photograph (HDR) | AVIF | 75 | 40-80 KB |
| Screenshot with text | WebP lossless or PNG | Lossless | 100-300 KB |
| Icon/logo | SVG | N/A | 1-10 KB |
| Simple illustration | SVG or WebP lossless | Lossless | 5-50 KB |
| UI animation | Lottie JSON | N/A | 5-50 KB |
| Short video clip | MP4 (H.264) or animated WebP | N/A | 200-800 KB |

### Image Size by Context

| Context | Display Width (pt) | At 2x DPR | At 3x DPR |
|---------|-------------------|-----------|-----------|
| Thumbnail / avatar | 40-80 | 80-160 px | 120-240 px |
| List card image | 100-375 | 200-750 px | 300-1125 px |
| Half-width card (tablet) | 350-500 | 700-1000 px | 1050-1500 px |
| Full-width hero (phone) | 375-430 | 750-860 px | 1125-1290 px |
| Full-width hero (tablet) | 768-1024 | 1536-2048 px | N/A (tablets are 2x) |
| Full-width hero (desktop) | 1200-1920 | 2400-3840 px | N/A (desktops are 1-2x) |

### Memory Budget Guidelines

| Device Tier | Total Image Memory Budget | Max Concurrent Full Images | Strategy |
|-------------|--------------------------|---------------------------|----------|
| Low-end Android (3-4 GB) | 80-100 MB | 5-8 at 750x500 | Aggressive caching, lower DPR, smaller sizes |
| Mid-range (6-8 GB) | 200-300 MB | 15-20 at 750x500 | Standard optimization |
| High-end (12+ GB) | 400-600 MB | 30-40 at 750x500 | Can afford higher quality |

### Quick Wins (Ordered by Impact)

1. **Serve the right size** (32x memory reduction from 4000x3000 to 750x500)
2. **Use WebP format** (30% file size reduction over JPEG)
3. **Set quality to 75** (50% smaller than q100, visually identical on screens)
4. **Use expo-image with blurhash placeholders** (perceived instant loading)
5. **Cap DPR at 2x on slow connections** (2.25x bandwidth savings vs 3x)
6. **Lazy load below-fold images** (faster initial paint, less memory)
7. **Set recyclingKey on list images** (prevents stale images and memory leaks)
8. **Use the sizes prop on next/image** (prevents over-fetching on web)
9. **Prefetch images on navigation intent** (detail screens feel instant)
10. **Generate placeholders at upload time** (zero runtime cost)

---

## 14. DECISION MATRICES

### Choosing a Placeholder Strategy

| Factor | Blurhash | Thumbhash | LQIP | Dominant Color |
|--------|----------|-----------|------|----------------|
| Visual quality | Good (abstract) | Better (shapes) | Best (actual image) | Minimal |
| Payload size | ~30 bytes | ~28 bytes | 200-500 bytes | ~7 bytes |
| Implementation effort | Low | Low | Medium | Trivial |
| Decode speed | ~1ms | ~1ms | ~10ms (network) | Instant |
| Best for | Most use cases | New projects | Hero images | Performance-critical |

### Choosing an Image CDN

| Factor | Cloudinary | Imgix | Cloudflare Images | Vercel Image Opt |
|--------|-----------|-------|-------------------|-----------------|
| **Feature depth** | Highest | High | Medium | Basic |
| **AI features** | Yes (face, bg removal) | Limited | No | No |
| **Price at scale** | $$$ | $$ | $ | $ (with Vercel) |
| **Setup complexity** | Medium | Medium | Low | None (with Next.js) |
| **Video support** | Yes | No | Yes (Stream) | No |
| **Best for** | Feature-rich apps | Analytics-heavy | Cost-sensitive | Next.js on Vercel |

### Choosing a Responsive Layout Approach

| Factor | useWindowDimensions + hooks | NativeWind | Tamagui |
|--------|---------------------------|------------|---------|
| Learning curve | Low | Medium (Tailwind knowledge) | Medium |
| Type safety | Full | Partial | Full |
| Web compatibility | Manual | Good (Tailwind on web) | Excellent |
| Performance | Good (JS-driven) | Good (compiled styles) | Best (compiled) |
| Community size | Largest | Large | Medium |
| Best for | Custom breakpoint logic | Tailwind teams | Universal apps |

---

## CHAPTER SUMMARY

Images and responsive content are the difference between an app that works and an app that crashes, between a page that loads in 200ms and one that loads in 5 seconds, between a user who stays and one who leaves.

The principles are straightforward:

1. **Serve the smallest image that looks good.** This means the right dimensions (display size times pixel density), the right format (WebP with AVIF and JPEG fallbacks), and the right quality (75 is the sweet spot for most photos).

2. **Show something immediately.** Blurhash or thumbhash placeholders cost 30 bytes in your API response and make every image feel instant.

3. **Use the platform's best tools.** expo-image on React Native, next/image on the web. They handle caching, format negotiation, lazy loading, and responsive sizing.

4. **Centralize your media system.** One image URL builder, one placeholder pipeline, one set of responsive breakpoints. Scattered ad-hoc optimization is worse than no optimization because it creates inconsistency and maintenance burden.

5. **Adapt content to the device.** Not just the layout -- the content itself. Fewer items on a phone feed, more detail on a tablet card, different crops for portrait and landscape.

6. **Measure on the worst device your users have.** If your app works on a $50 Android phone on 3G, it works everywhere. If it only works on your development iPhone on Wi-Fi, it doesn't work.

The investment in building a proper media system pays for itself within weeks. The first time you add a new image-heavy feature and it just works -- correct size, correct format, placeholder, lazy loaded, responsive -- you'll understand why this chapter exists.

Build the system once. Use it everywhere. Your users (and your P95 latency metrics) will thank you.

---

**Next up:** [Chapter 24] continues our Architecture at Scale journey. The foundation you've built here -- responsive components, adaptive content, platform-aware rendering -- will be the bedrock for the patterns in the next chapter.
