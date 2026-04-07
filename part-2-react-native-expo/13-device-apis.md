<!--
  CHAPTER: 13
  TITLE: Device APIs & Native Features
  PART: II — React Native & Expo
  PREREQS: Chapter 6
  KEY_TOPICS: camera, location, maps, contacts, calendar, haptics, clipboard, share sheet, barcode scanning, WebView, device info, sensors, NFC, Bluetooth, keyboard handling, media playback
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 13: Device APIs & Native Features

> **Part II — React Native & Expo** | Prerequisites: Chapter 6 | Difficulty: Intermediate

> "The best mobile apps don't feel like they're running inside a framework. They feel like they own the device."
>
> "If your app can't access the camera, the GPS, or the haptic engine, why isn't it a website?"

---

<details>
<summary><strong>TL;DR</strong></summary>

- Expo's module ecosystem gives you native device APIs through a consistent, permission-aware, TypeScript-first interface — no native code required for most use cases
- Camera (expo-camera) handles photos, video, barcode scanning, and QR codes; always request permissions before opening, and always provide a fallback UI for denied states
- Location (expo-location) supports foreground, background, and geofencing — but background location requires config plugin setup and will get your App Store submission rejected if you don't justify it
- react-native-maps is the standard for MapView; use clustering for 50+ markers, custom markers for branding, and Mapbox when you need offline maps or advanced styling
- Haptics make your app feel native — use them on button presses, success/error states, and swipe actions, but never on scroll or rapid-fire events
- WebView is for embedding web content; expo-web-browser (SafariViewController/Chrome Custom Tabs) is for OAuth flows and external links — mixing them up is a common mistake
- Keyboard handling is the silent UX killer in React Native; use react-native-keyboard-controller for production apps, or at minimum KeyboardAvoidingView with platform-specific behavior
- expo-av handles audio/video playback, expo-video provides a modern video player, and expo-image-picker gives you camera roll and capture access

</details>

Every mobile app eventually needs to talk to the hardware. The camera. The GPS. The accelerometer. The haptic engine. The share sheet. The clipboard. The keyboard. These are the things that separate a mobile app from a fancy website, and they're the reason your users downloaded a native app in the first place.

The good news: Expo has wrapped most of these device APIs into a consistent, TypeScript-first, permission-aware module system. The bad news: there are subtle platform differences, permission gotchas, and performance traps that will bite you if you don't know about them.

This chapter is your field guide to every device API you'll commonly need. We'll go deep on the ones that trip people up (camera, location, keyboard handling) and give you working patterns for the rest. By the end, you'll be able to access any device capability with confidence.

### In This Chapter
- Camera: Photos, Video, and Barcode Scanning
- Location: Foreground, Background, and Geofencing
- Maps: MapView, Markers, Clustering, and Directions
- Contacts: Reading and Creating Contact Records
- Calendar: Events and Reminders
- Haptics: Impact, Notification, and Selection Feedback
- Clipboard: Copy/Paste Text and Images
- Share Sheet: Sharing Content and Receiving Shared Content
- Barcode/QR Scanning: Custom Scanning UI
- WebView & In-App Browser: When to Use Which
- Device Info: Screen Dimensions, Platform Detection, Versions
- Sensors: Accelerometer, Gyroscope, Barometer, Shake Detection
- Keyboard Handling: The Silent UX Killer
- Media Playback: Audio, Video, and Image Picker
- Platform Differences: iOS vs Android Haptics and Vibration

### Related Chapters
- [Ch 6: EAS Mastery] — config plugins for native module setup
- [Ch 7: Navigation Architecture] — deep linking from device features
- [Ch 8: Styling & Animation] — gesture-driven interactions with device sensors
- [Ch 14: Permissions, Accessibility & Internationalization] — permission request patterns

---

## 1. CAMERA: PHOTOS, VIDEO, AND BARCODE SCANNING

The camera is probably the most-requested device API. Profile photos, document scanning, barcode reading, video recording — it all starts here.

### 1.1 Setting Up expo-camera

```bash
npx expo install expo-camera
```

The camera module requires native permissions on both platforms. Here's the `app.json` config:

```json
{
  "expo": {
    "plugins": [
      [
        "expo-camera",
        {
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera for taking photos and scanning codes.",
          "microphonePermission": "Allow $(PRODUCT_NAME) to access your microphone for video recording.",
          "recordAudioAndroid": true
        }
      ]
    ]
  }
}
```

**Why customize the permission string?** Because "App wants to access your camera" is scary. "Allow MyApp to access your camera for taking profile photos" tells the user exactly why. Apple reviews this string, and vague descriptions can lead to rejection.

### 1.2 Basic Camera View

```tsx
import { CameraView, CameraType, useCameraPermissions } from 'expo-camera';
import { useState, useRef } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

function CameraScreen() {
  const [facing, setFacing] = useState<CameraType>('back');
  const [permission, requestPermission] = useCameraPermissions();
  const cameraRef = useRef<CameraView>(null);

  // Permission not yet determined
  if (!permission) {
    return <View style={styles.container} />;
  }

  // Permission denied
  if (!permission.granted) {
    return (
      <View style={styles.container}>
        <Text style={styles.message}>
          We need camera access to take photos.
        </Text>
        <Pressable style={styles.button} onPress={requestPermission}>
          <Text style={styles.buttonText}>Grant Permission</Text>
        </Pressable>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <CameraView
        ref={cameraRef}
        style={styles.camera}
        facing={facing}
      >
        <View style={styles.controls}>
          <Pressable
            style={styles.flipButton}
            onPress={() => {
              setFacing(current => 
                current === 'back' ? 'front' : 'back'
              );
            }}
          >
            <Text style={styles.buttonText}>Flip</Text>
          </Pressable>
          
          <Pressable
            style={styles.captureButton}
            onPress={async () => {
              if (cameraRef.current) {
                const photo = await cameraRef.current.takePictureAsync({
                  quality: 0.8,
                  base64: false,
                  exif: true,
                });
                console.log('Photo taken:', photo?.uri);
                // Handle the photo — save, upload, display
              }
            }}
          >
            <View style={styles.captureInner} />
          </Pressable>
        </View>
      </CameraView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    backgroundColor: '#000',
  },
  camera: {
    flex: 1,
  },
  controls: {
    position: 'absolute',
    bottom: 40,
    left: 0,
    right: 0,
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    gap: 40,
  },
  flipButton: {
    padding: 16,
    backgroundColor: 'rgba(255,255,255,0.3)',
    borderRadius: 8,
  },
  captureButton: {
    width: 72,
    height: 72,
    borderRadius: 36,
    borderWidth: 4,
    borderColor: 'white',
    justifyContent: 'center',
    alignItems: 'center',
  },
  captureInner: {
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: 'white',
  },
  message: {
    color: 'white',
    fontSize: 16,
    textAlign: 'center',
    marginBottom: 16,
    paddingHorizontal: 32,
  },
  button: {
    alignSelf: 'center',
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 1.3 Taking Photos with Options

```tsx
interface TakePictureOptions {
  quality?: number;        // 0-1, default 1. Use 0.7-0.8 for uploads.
  base64?: boolean;        // Include base64 data. Heavy — only if needed.
  exif?: boolean;          // Include EXIF metadata (GPS, orientation, etc.)
  skipProcessing?: boolean; // Skip post-processing. Faster, lower quality.
  imageType?: 'png' | 'jpg'; // Output format
}

async function takePhotoWithOptions(cameraRef: React.RefObject<CameraView>) {
  if (!cameraRef.current) return null;

  const photo = await cameraRef.current.takePictureAsync({
    quality: 0.8,           // Good balance of quality vs file size
    base64: false,          // Don't need base64 for file upload
    exif: true,             // Include metadata for location tagging
    skipProcessing: false,  // Process for correct orientation
  });

  return photo;
  // photo = { uri: 'file:///...', width: 4032, height: 3024, exif: {...} }
}
```

**Opinion:** Always use `quality: 0.7-0.8` for photos that will be uploaded. Full quality (1.0) produces 8-12MB files that murder your user's data plan and your server's bandwidth. The visual difference between 0.8 and 1.0 is imperceptible on mobile screens.

### 1.4 Recording Video

```tsx
import { CameraView, CameraType } from 'expo-camera';
import { useState, useRef } from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';

function VideoRecorder() {
  const cameraRef = useRef<CameraView>(null);
  const [isRecording, setIsRecording] = useState(false);

  const startRecording = async () => {
    if (!cameraRef.current) return;
    
    setIsRecording(true);
    
    try {
      const video = await cameraRef.current.recordAsync({
        maxDuration: 60,        // Max 60 seconds
        maxFileSize: 50_000_000, // Max 50MB
        // quality is set via the CameraView `videoQuality` prop
      });
      
      console.log('Video recorded:', video?.uri);
      // video = { uri: 'file:///...' }
    } catch (error) {
      console.error('Recording failed:', error);
    } finally {
      setIsRecording(false);
    }
  };

  const stopRecording = () => {
    if (cameraRef.current && isRecording) {
      cameraRef.current.stopRecording();
    }
  };

  return (
    <View style={styles.container}>
      <CameraView
        ref={cameraRef}
        style={styles.camera}
        mode="video"
        videoQuality="720p"
      >
        <View style={styles.controls}>
          <Pressable
            style={[
              styles.recordButton,
              isRecording && styles.recordButtonActive,
            ]}
            onPress={isRecording ? stopRecording : startRecording}
          >
            <View style={[
              styles.recordInner,
              isRecording && styles.recordInnerActive,
            ]} />
          </Pressable>
        </View>
      </CameraView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  camera: { flex: 1 },
  controls: {
    position: 'absolute',
    bottom: 40,
    alignSelf: 'center',
  },
  recordButton: {
    width: 72,
    height: 72,
    borderRadius: 36,
    borderWidth: 4,
    borderColor: 'white',
    justifyContent: 'center',
    alignItems: 'center',
  },
  recordButtonActive: {
    borderColor: 'red',
  },
  recordInner: {
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: 'red',
  },
  recordInnerActive: {
    width: 28,
    height: 28,
    borderRadius: 4,
    backgroundColor: 'red',
  },
});
```

### 1.5 Flash and Zoom Control

```tsx
import { CameraView, FlashMode } from 'expo-camera';
import { useState, useRef } from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';
import Animated, { useSharedValue, useAnimatedProps } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const AnimatedCameraView = Animated.createAnimatedComponent(CameraView);

function CameraWithControls() {
  const [flash, setFlash] = useState<FlashMode>('off');
  const zoom = useSharedValue(0);
  const savedZoom = useSharedValue(0);
  const cameraRef = useRef<CameraView>(null);

  // Pinch-to-zoom gesture
  const pinchGesture = Gesture.Pinch()
    .onStart(() => {
      savedZoom.value = zoom.value;
    })
    .onUpdate((event) => {
      // Scale factor to zoom level (0-1)
      const newZoom = savedZoom.value + (event.scale - 1) * 0.5;
      zoom.value = Math.max(0, Math.min(1, newZoom));
    });

  const animatedProps = useAnimatedProps(() => ({
    zoom: zoom.value,
  }));

  const cycleFlash = () => {
    setFlash(current => {
      switch (current) {
        case 'off': return 'on';
        case 'on': return 'auto';
        case 'auto': return 'off';
        default: return 'off';
      }
    });
  };

  const flashLabels: Record<FlashMode, string> = {
    off: 'Flash Off',
    on: 'Flash On',
    auto: 'Flash Auto',
  };

  return (
    <View style={styles.container}>
      <GestureDetector gesture={pinchGesture}>
        <AnimatedCameraView
          ref={cameraRef}
          style={styles.camera}
          flash={flash}
          animatedProps={animatedProps}
        >
          <View style={styles.topControls}>
            <Pressable style={styles.flashButton} onPress={cycleFlash}>
              <Text style={styles.flashText}>{flashLabels[flash]}</Text>
            </Pressable>
          </View>
        </AnimatedCameraView>
      </GestureDetector>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  camera: { flex: 1 },
  topControls: {
    position: 'absolute',
    top: 60,
    left: 0,
    right: 0,
    alignItems: 'center',
  },
  flashButton: {
    padding: 12,
    backgroundColor: 'rgba(0,0,0,0.5)',
    borderRadius: 20,
  },
  flashText: {
    color: 'white',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

### 1.6 QR/Barcode Scanning with CameraView

This is one of the most common camera use cases, and expo-camera handles it natively:

```tsx
import { CameraView, BarcodeScanningResult } from 'expo-camera';
import { useState, useCallback } from 'react';
import { View, Text, StyleSheet, Dimensions, Pressable } from 'react-native';
import Animated, { FadeIn, FadeOut } from 'react-native-reanimated';

const { width: SCREEN_WIDTH } = Dimensions.get('window');
const SCAN_AREA_SIZE = SCREEN_WIDTH * 0.7;

function QRScanner({ onScanComplete }: { onScanComplete: (data: string) => void }) {
  const [scanned, setScanned] = useState(false);

  const handleBarCodeScanned = useCallback(
    (result: BarcodeScanningResult) => {
      if (scanned) return; // Prevent rapid-fire scans
      
      setScanned(true);
      
      // Validate the scanned data before acting on it
      const { data, type } = result;
      console.log(`Scanned ${type}: ${data}`);
      
      onScanComplete(data);
    },
    [scanned, onScanComplete]
  );

  return (
    <View style={styles.container}>
      <CameraView
        style={styles.camera}
        barcodeScannerSettings={{
          barcodeTypes: [
            'qr',
            'ean13',
            'ean8',
            'code128',
            'code39',
            'upc_a',
            'upc_e',
            'pdf417',
            'aztec',
            'datamatrix',
          ],
        }}
        onBarcodeScanned={scanned ? undefined : handleBarCodeScanned}
      >
        {/* Scanning overlay */}
        <View style={styles.overlay}>
          {/* Top */}
          <View style={styles.overlayDark} />
          
          {/* Middle row */}
          <View style={styles.middleRow}>
            <View style={styles.overlayDark} />
            
            {/* Scan area */}
            <View style={styles.scanArea}>
              {/* Corner markers */}
              <View style={[styles.corner, styles.topLeft]} />
              <View style={[styles.corner, styles.topRight]} />
              <View style={[styles.corner, styles.bottomLeft]} />
              <View style={[styles.corner, styles.bottomRight]} />
            </View>
            
            <View style={styles.overlayDark} />
          </View>
          
          {/* Bottom */}
          <View style={styles.overlayDark}>
            <Text style={styles.instruction}>
              Point your camera at a QR code
            </Text>
          </View>
        </View>

        {/* Scanned state */}
        {scanned && (
          <Animated.View 
            entering={FadeIn}
            exiting={FadeOut}
            style={styles.scannedOverlay}
          >
            <Text style={styles.scannedText}>Code scanned!</Text>
            <Pressable
              style={styles.scanAgainButton}
              onPress={() => setScanned(false)}
            >
              <Text style={styles.scanAgainText}>Scan Again</Text>
            </Pressable>
          </Animated.View>
        )}
      </CameraView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  camera: { flex: 1 },
  overlay: { flex: 1 },
  overlayDark: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.6)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  middleRow: {
    flexDirection: 'row',
    height: SCAN_AREA_SIZE,
  },
  scanArea: {
    width: SCAN_AREA_SIZE,
    height: SCAN_AREA_SIZE,
    position: 'relative',
  },
  corner: {
    position: 'absolute',
    width: 24,
    height: 24,
    borderColor: '#00FF00',
  },
  topLeft: {
    top: 0,
    left: 0,
    borderTopWidth: 3,
    borderLeftWidth: 3,
  },
  topRight: {
    top: 0,
    right: 0,
    borderTopWidth: 3,
    borderRightWidth: 3,
  },
  bottomLeft: {
    bottom: 0,
    left: 0,
    borderBottomWidth: 3,
    borderLeftWidth: 3,
  },
  bottomRight: {
    bottom: 0,
    right: 0,
    borderBottomWidth: 3,
    borderRightWidth: 3,
  },
  instruction: {
    color: 'white',
    fontSize: 16,
    marginTop: 24,
  },
  scannedOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(0,0,0,0.7)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  scannedText: {
    color: 'white',
    fontSize: 24,
    fontWeight: '700',
    marginBottom: 16,
  },
  scanAgainButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
  },
  scanAgainText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 1.7 Camera Best Practices

Here's what I've learned from shipping camera features in production apps:

1. **Always request permission before showing the camera.** Don't mount `CameraView` and have it show a black screen while permissions are resolving. Show a loading state or permission request screen first.

2. **Provide graceful fallbacks.** If the user denies camera permission, give them a way to open Settings and grant it manually. Never show a dead-end screen.

3. **Use `quality: 0.7-0.8` for uploads.** Full quality is a waste of bandwidth.

4. **Handle orientation carefully.** Photos taken in landscape may have EXIF orientation data that your backend needs to respect.

5. **Test on real devices.** The camera in the iOS Simulator is a static image. The Android emulator camera is a virtual scene. Neither represents real performance.

6. **Dispose of the camera when not in use.** Use `useFocusEffect` to unmount the camera when navigating away. Camera hardware is a shared resource — holding it prevents other apps from using it.

```tsx
import { useFocusEffect } from 'expo-router';
import { useCallback, useState } from 'react';

function CameraScreenWithFocus() {
  const [isFocused, setIsFocused] = useState(false);

  useFocusEffect(
    useCallback(() => {
      setIsFocused(true);
      return () => setIsFocused(false);
    }, [])
  );

  if (!isFocused) {
    return <View style={{ flex: 1, backgroundColor: '#000' }} />;
  }

  return <CameraView style={{ flex: 1 }} />;
}
```

---

## 2. LOCATION: FOREGROUND, BACKGROUND, AND GEOFENCING

Location is the second most-requested device API — and the one most likely to get your app rejected from the App Store if you handle it wrong.

### 2.1 Setting Up expo-location

```bash
npx expo install expo-location
```

```json
{
  "expo": {
    "plugins": [
      [
        "expo-location",
        {
          "locationAlwaysAndWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location to show nearby restaurants even when the app is in the background.",
          "locationAlwaysPermission": "Allow $(PRODUCT_NAME) to use your location in the background.",
          "locationWhenInUsePermission": "Allow $(PRODUCT_NAME) to use your location to show nearby restaurants.",
          "isAndroidBackgroundLocationEnabled": true,
          "isAndroidForegroundServiceEnabled": true
        }
      ]
    ]
  }
}
```

### 2.2 Foreground Location

This is the simple case: get the user's location while the app is open.

```tsx
import * as Location from 'expo-location';
import { useState, useEffect } from 'react';
import { View, Text, StyleSheet, ActivityIndicator, Pressable } from 'react-native';

interface LocationData {
  latitude: number;
  longitude: number;
  altitude: number | null;
  accuracy: number | null;
  heading: number | null;
  speed: number | null;
}

function useCurrentLocation() {
  const [location, setLocation] = useState<LocationData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let isMounted = true;

    async function getLocation() {
      try {
        // Check permission status
        const { status } = await Location.requestForegroundPermissionsAsync();
        
        if (status !== 'granted') {
          if (isMounted) {
            setError('Location permission denied');
            setLoading(false);
          }
          return;
        }

        // Check if location services are enabled
        const isEnabled = await Location.hasServicesEnabledAsync();
        if (!isEnabled) {
          if (isMounted) {
            setError('Location services are disabled. Please enable them in Settings.');
            setLoading(false);
          }
          return;
        }

        // Get current position
        const result = await Location.getCurrentPositionAsync({
          accuracy: Location.Accuracy.Balanced,
          // Balanced = ~100m accuracy, good battery life
          // High = ~10m accuracy, higher battery usage
          // BestForNavigation = GPS + Wi-Fi + cellular, highest battery
        });

        if (isMounted) {
          setLocation({
            latitude: result.coords.latitude,
            longitude: result.coords.longitude,
            altitude: result.coords.altitude,
            accuracy: result.coords.accuracy,
            heading: result.coords.heading,
            speed: result.coords.speed,
          });
          setLoading(false);
        }
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err.message : 'Failed to get location');
          setLoading(false);
        }
      }
    }

    getLocation();

    return () => {
      isMounted = false;
    };
  }, []);

  return { location, error, loading };
}

function LocationDisplay() {
  const { location, error, loading } = useCurrentLocation();

  if (loading) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" />
        <Text style={styles.text}>Getting your location...</Text>
      </View>
    );
  }

  if (error) {
    return (
      <View style={styles.center}>
        <Text style={styles.error}>{error}</Text>
      </View>
    );
  }

  return (
    <View style={styles.center}>
      <Text style={styles.text}>
        Lat: {location?.latitude.toFixed(6)}
      </Text>
      <Text style={styles.text}>
        Lon: {location?.longitude.toFixed(6)}
      </Text>
      <Text style={styles.text}>
        Accuracy: {location?.accuracy?.toFixed(0)}m
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  center: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
  },
  text: {
    fontSize: 18,
    marginVertical: 4,
    color: '#1A1A1A',
  },
  error: {
    fontSize: 16,
    color: '#FF3B30',
    textAlign: 'center',
  },
});
```

### 2.3 Watching Location (Continuous Updates)

For navigation apps, ride-sharing, fitness tracking — you need continuous location updates:

```tsx
import * as Location from 'expo-location';
import { useEffect, useRef, useState } from 'react';

interface LocationWatchOptions {
  accuracy?: Location.Accuracy;
  distanceInterval?: number; // Minimum distance (meters) between updates
  timeInterval?: number;     // Minimum time (ms) between updates
}

function useLocationWatch(options: LocationWatchOptions = {}) {
  const [location, setLocation] = useState<Location.LocationObject | null>(null);
  const [locations, setLocations] = useState<Location.LocationObject[]>([]);
  const subscriptionRef = useRef<Location.LocationSubscription | null>(null);

  useEffect(() => {
    let isMounted = true;

    async function startWatching() {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') return;

      subscriptionRef.current = await Location.watchPositionAsync(
        {
          accuracy: options.accuracy ?? Location.Accuracy.High,
          distanceInterval: options.distanceInterval ?? 10, // Update every 10m
          timeInterval: options.timeInterval ?? 5000,       // Or every 5 seconds
        },
        (newLocation) => {
          if (isMounted) {
            setLocation(newLocation);
            setLocations(prev => [...prev, newLocation]);
          }
        }
      );
    }

    startWatching();

    return () => {
      isMounted = false;
      subscriptionRef.current?.remove();
    };
  }, [options.accuracy, options.distanceInterval, options.timeInterval]);

  return { location, locations };
}

// Usage in a run-tracking screen
function RunTracker() {
  const { location, locations } = useLocationWatch({
    accuracy: Location.Accuracy.BestForNavigation,
    distanceInterval: 5,   // Every 5 meters
    timeInterval: 1000,     // Or every second
  });

  // Calculate total distance
  const totalDistance = locations.reduce((total, loc, index) => {
    if (index === 0) return 0;
    const prev = locations[index - 1];
    return total + getDistanceMeters(
      prev.coords.latitude, prev.coords.longitude,
      loc.coords.latitude, loc.coords.longitude
    );
  }, 0);

  return (
    <View>
      <Text>Distance: {(totalDistance / 1000).toFixed(2)} km</Text>
      <Text>Speed: {((location?.coords.speed ?? 0) * 3.6).toFixed(1)} km/h</Text>
      <Text>Points recorded: {locations.length}</Text>
    </View>
  );
}

// Haversine formula for distance between two GPS coordinates
function getDistanceMeters(
  lat1: number, lon1: number,
  lat2: number, lon2: number
): number {
  const R = 6371000; // Earth's radius in meters
  const dLat = ((lat2 - lat1) * Math.PI) / 180;
  const dLon = ((lon2 - lon1) * Math.PI) / 180;
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos((lat1 * Math.PI) / 180) *
    Math.cos((lat2 * Math.PI) / 180) *
    Math.sin(dLon / 2) * Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
}
```

### 2.4 Background Location

This is where it gets serious. Background location means your app continues to track the user's position even when they've switched to another app or locked their phone. Both Apple and Google have strict policies about this.

**When you need it:**
- Navigation apps (turn-by-turn directions)
- Fitness tracking (recording a run or bike ride)
- Delivery driver tracking
- Geofencing (alerting when entering/leaving an area)

**When you don't need it (and shouldn't request it):**
- "Nearby" features (use foreground-only)
- Weather apps (use significant location changes instead)
- Anything where "when in use" permission is sufficient

```tsx
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const BACKGROUND_LOCATION_TASK = 'background-location-task';

// Define the background task OUTSIDE of any component
// This must be at the module level
TaskManager.defineTask(BACKGROUND_LOCATION_TASK, ({ data, error }) => {
  if (error) {
    console.error('Background location error:', error);
    return;
  }
  
  if (data) {
    const { locations } = data as { locations: Location.LocationObject[] };
    
    // Process locations — save to local DB, send to server, etc.
    for (const location of locations) {
      console.log('Background location:', {
        lat: location.coords.latitude,
        lng: location.coords.longitude,
        timestamp: location.timestamp,
      });
      
      // In a real app, you'd batch these and send to your server
      // or save to SQLite/MMKV for later sync
    }
  }
});

async function startBackgroundLocationTracking() {
  // Step 1: Request foreground permission first
  const foreground = await Location.requestForegroundPermissionsAsync();
  if (foreground.status !== 'granted') {
    throw new Error('Foreground location permission denied');
  }

  // Step 2: Request background permission
  // On iOS, this triggers the "Always Allow" prompt
  // On Android 10+, this triggers a separate permission dialog
  const background = await Location.requestBackgroundPermissionsAsync();
  if (background.status !== 'granted') {
    throw new Error('Background location permission denied');
  }

  // Step 3: Check if the task is already running
  const isTaskRunning = await TaskManager.isTaskRegisteredAsync(
    BACKGROUND_LOCATION_TASK
  );
  if (isTaskRunning) {
    console.log('Background location task already running');
    return;
  }

  // Step 4: Start background location updates
  await Location.startLocationUpdatesAsync(BACKGROUND_LOCATION_TASK, {
    accuracy: Location.Accuracy.High,
    distanceInterval: 50,         // Update every 50 meters
    timeInterval: 10000,           // Or every 10 seconds
    deferredUpdatesInterval: 60000, // Batch updates every minute (battery saving)
    deferredUpdatesDistance: 100,   // Or every 100 meters
    showsBackgroundLocationIndicator: true, // iOS: blue bar indicator
    foregroundService: {
      notificationTitle: 'MyApp is tracking your location',
      notificationBody: 'Tap to open the app',
      notificationColor: '#007AFF',
    },
    pausesUpdatesAutomatically: false, // Don't let iOS pause updates
    activityType: Location.ActivityType.Fitness, // iOS: optimize for activity type
  });
}

async function stopBackgroundLocationTracking() {
  const isTaskRunning = await TaskManager.isTaskRegisteredAsync(
    BACKGROUND_LOCATION_TASK
  );
  
  if (isTaskRunning) {
    await Location.stopLocationUpdatesAsync(BACKGROUND_LOCATION_TASK);
  }
}
```

**Critical app.json config for background location:**

```json
{
  "expo": {
    "ios": {
      "infoPlist": {
        "UIBackgroundModes": ["location"],
        "NSLocationAlwaysAndWhenInUseUsageDescription": "Allow MyApp to track your run even when the app is in the background.",
        "NSLocationAlwaysUsageDescription": "Allow MyApp to track your location in the background.",
        "NSLocationWhenInUseUsageDescription": "Allow MyApp to show your location on the map."
      }
    },
    "android": {
      "permissions": [
        "ACCESS_FINE_LOCATION",
        "ACCESS_COARSE_LOCATION",
        "ACCESS_BACKGROUND_LOCATION",
        "FOREGROUND_SERVICE",
        "FOREGROUND_SERVICE_LOCATION"
      ]
    }
  }
}
```

### 2.5 Geofencing

Geofencing alerts your app when the user enters or leaves a geographic region. It's battery-efficient because the OS handles the monitoring — your app only wakes up when a boundary is crossed.

```tsx
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const GEOFENCING_TASK = 'geofencing-task';

// Define geofence regions
const GEOFENCE_REGIONS: Location.LocationRegion[] = [
  {
    identifier: 'office',
    latitude: 37.7749,
    longitude: -122.4194,
    radius: 200, // 200 meters
    notifyOnEnter: true,
    notifyOnExit: true,
  },
  {
    identifier: 'home',
    latitude: 37.7849,
    longitude: -122.4094,
    radius: 100,
    notifyOnEnter: true,
    notifyOnExit: true,
  },
  {
    identifier: 'gym',
    latitude: 37.7649,
    longitude: -122.4294,
    radius: 150,
    notifyOnEnter: true,
    notifyOnExit: false, // Only notify on arrival
  },
];

// Define the background task
TaskManager.defineTask(GEOFENCING_TASK, ({ data, error }) => {
  if (error) {
    console.error('Geofencing error:', error);
    return;
  }

  if (data) {
    const { eventType, region } = data as {
      eventType: Location.GeofencingEventType;
      region: Location.LocationRegion;
    };

    if (eventType === Location.GeofencingEventType.Enter) {
      console.log(`Entered region: ${region.identifier}`);
      // Send notification, update server, trigger automation
    } else if (eventType === Location.GeofencingEventType.Exit) {
      console.log(`Left region: ${region.identifier}`);
    }
  }
});

async function startGeofencing() {
  const { status } = await Location.requestBackgroundPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Background location permission required for geofencing');
  }

  await Location.startGeofencingAsync(GEOFENCING_TASK, GEOFENCE_REGIONS);
}

async function stopGeofencing() {
  await Location.stopGeofencingAsync(GEOFENCING_TASK);
}

// Check which geofences are active
async function getActiveGeofences() {
  const regions = await Location.getGeofencingRegionsAsync();
  console.log('Active geofences:', regions);
  return regions;
}
```

### 2.6 Geocoding and Reverse Geocoding

Converting between addresses and coordinates:

```tsx
import * as Location from 'expo-location';

// Address → Coordinates
async function geocodeAddress(address: string) {
  const results = await Location.geocodeAsync(address);
  
  if (results.length === 0) {
    throw new Error('Address not found');
  }

  return {
    latitude: results[0].latitude,
    longitude: results[0].longitude,
    accuracy: results[0].accuracy,
  };
}

// Coordinates → Address
async function reverseGeocode(latitude: number, longitude: number) {
  const results = await Location.reverseGeocodeAsync({
    latitude,
    longitude,
  });

  if (results.length === 0) {
    throw new Error('No address found for these coordinates');
  }

  const address = results[0];
  return {
    street: address.street,
    streetNumber: address.streetNumber,
    city: address.city,
    region: address.region,        // State/province
    postalCode: address.postalCode,
    country: address.country,
    isoCountryCode: address.isoCountryCode,
    formattedAddress: address.formattedAddress,
    // Compose a display string
    displayAddress: [
      address.streetNumber,
      address.street,
      address.city,
      address.region,
      address.postalCode,
    ].filter(Boolean).join(', '),
  };
}

// Usage
async function example() {
  // Forward geocode
  const coords = await geocodeAddress('1600 Amphitheatre Parkway, Mountain View, CA');
  console.log(coords); // { latitude: 37.422, longitude: -122.084, ... }

  // Reverse geocode
  const address = await reverseGeocode(37.7749, -122.4194);
  console.log(address.displayAddress); // "1 Market St, San Francisco, CA, 94105"
}
```

### 2.7 Battery-Efficient Location Patterns

Location is the number one battery killer on mobile. Here's how to be a good citizen:

```tsx
import * as Location from 'expo-location';

// Pattern 1: Use the right accuracy level
const ACCURACY_BY_USE_CASE = {
  // "Find restaurants near me" — city block accuracy is fine
  nearby: Location.Accuracy.Balanced,    // ~100m, low battery
  
  // "Show me on the map" — good accuracy needed
  mapDisplay: Location.Accuracy.High,    // ~10m, moderate battery
  
  // "Navigate to destination" — best accuracy needed
  navigation: Location.Accuracy.BestForNavigation, // ~1m, high battery
  
  // "What city am I in?" — very rough is fine
  cityLevel: Location.Accuracy.Low,      // ~1km, minimal battery
};

// Pattern 2: Significant location changes (iOS)
// Only triggers when the user moves a significant distance (~500m)
// Uses cell tower triangulation, not GPS — very battery efficient
async function watchSignificantChanges(
  onLocationChange: (location: Location.LocationObject) => void
) {
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return;

  // Use high distance interval with low accuracy for battery efficiency
  return Location.watchPositionAsync(
    {
      accuracy: Location.Accuracy.Balanced,
      distanceInterval: 500,     // Only update every 500 meters
      timeInterval: 300000,       // Or every 5 minutes at most
    },
    onLocationChange
  );
}

// Pattern 3: One-shot location with caching
// Don't re-fetch if we have a recent location
let cachedLocation: { location: Location.LocationObject; timestamp: number } | null = null;
const CACHE_DURATION = 60000; // 1 minute

async function getLocationCached(): Promise<Location.LocationObject> {
  if (cachedLocation && Date.now() - cachedLocation.timestamp < CACHE_DURATION) {
    return cachedLocation.location;
  }

  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Location permission denied');
  }

  // getLastKnownPositionAsync is instant — uses the last known GPS fix
  const lastKnown = await Location.getLastKnownPositionAsync({
    maxAge: CACHE_DURATION,
    requiredAccuracy: 100, // Accept if within 100m accuracy
  });

  if (lastKnown) {
    cachedLocation = { location: lastKnown, timestamp: Date.now() };
    return lastKnown;
  }

  // Fall back to getting fresh position
  const fresh = await Location.getCurrentPositionAsync({
    accuracy: Location.Accuracy.Balanced,
  });

  cachedLocation = { location: fresh, timestamp: Date.now() };
  return fresh;
}
```

---

## 3. MAPS: MAPVIEW, MARKERS, CLUSTERING, AND DIRECTIONS

Maps are one of those features that seem simple until you have 5,000 markers on screen and your app drops to 3fps.

### 3.1 Setting Up react-native-maps

```bash
npx expo install react-native-maps
```

The `react-native-maps` library uses Apple Maps on iOS and Google Maps on Android by default. If you want Google Maps on both platforms, you'll need a Google Maps API key.

```json
{
  "expo": {
    "ios": {
      "config": {
        "googleMapsApiKey": "YOUR_GOOGLE_MAPS_API_KEY"
      }
    },
    "android": {
      "config": {
        "googleMaps": {
          "apiKey": "YOUR_GOOGLE_MAPS_API_KEY"
        }
      }
    }
  }
}
```

### 3.2 Basic MapView

```tsx
import MapView, { Marker, PROVIDER_GOOGLE, Region } from 'react-native-maps';
import { StyleSheet, View, Platform } from 'react-native';
import { useState, useRef, useCallback } from 'react';

interface Place {
  id: string;
  name: string;
  latitude: number;
  longitude: number;
  description: string;
}

function MapScreen({ places }: { places: Place[] }) {
  const mapRef = useRef<MapView>(null);
  const [selectedPlace, setSelectedPlace] = useState<Place | null>(null);
  const [region, setRegion] = useState<Region>({
    latitude: 37.7749,
    longitude: -122.4194,
    latitudeDelta: 0.05,
    longitudeDelta: 0.05,
  });

  const handleRegionChange = useCallback((newRegion: Region) => {
    setRegion(newRegion);
  }, []);

  const centerOnPlace = useCallback((place: Place) => {
    mapRef.current?.animateToRegion(
      {
        latitude: place.latitude,
        longitude: place.longitude,
        latitudeDelta: 0.01,
        longitudeDelta: 0.01,
      },
      500 // animation duration in ms
    );
    setSelectedPlace(place);
  }, []);

  const fitToAllMarkers = useCallback(() => {
    if (places.length === 0) return;
    
    mapRef.current?.fitToCoordinates(
      places.map(p => ({ latitude: p.latitude, longitude: p.longitude })),
      {
        edgePadding: { top: 50, right: 50, bottom: 50, left: 50 },
        animated: true,
      }
    );
  }, [places]);

  return (
    <View style={styles.container}>
      <MapView
        ref={mapRef}
        style={styles.map}
        provider={Platform.OS === 'android' ? PROVIDER_GOOGLE : undefined}
        initialRegion={region}
        onRegionChangeComplete={handleRegionChange}
        showsUserLocation
        showsMyLocationButton
        showsCompass
        rotateEnabled={false} // Disable rotation for simpler UX
      >
        {places.map(place => (
          <Marker
            key={place.id}
            coordinate={{
              latitude: place.latitude,
              longitude: place.longitude,
            }}
            title={place.name}
            description={place.description}
            onPress={() => setSelectedPlace(place)}
          />
        ))}
      </MapView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  map: { flex: 1 },
});
```

### 3.3 Custom Markers

Default pins are boring. Custom markers let you brand your map:

```tsx
import MapView, { Marker } from 'react-native-maps';
import { View, Text, StyleSheet, Image } from 'react-native';

interface CustomMarkerProps {
  coordinate: { latitude: number; longitude: number };
  price: string;
  isSelected: boolean;
  onPress: () => void;
}

function PriceMarker({ coordinate, price, isSelected, onPress }: CustomMarkerProps) {
  return (
    <Marker
      coordinate={coordinate}
      onPress={onPress}
      // tracksViewChanges: Set to false once the marker is rendered.
      // Setting this to true continuously re-renders the marker — 
      // which kills performance with many markers.
      tracksViewChanges={false}
    >
      <View style={[
        styles.markerContainer,
        isSelected && styles.markerSelected,
      ]}>
        <Text style={[
          styles.markerText,
          isSelected && styles.markerTextSelected,
        ]}>
          {price}
        </Text>
        <View style={[
          styles.markerArrow,
          isSelected && styles.markerArrowSelected,
        ]} />
      </View>
    </Marker>
  );
}

// Image-based custom marker (better performance than View-based)
function AvatarMarker({ 
  coordinate, 
  imageUrl, 
  onPress 
}: {
  coordinate: { latitude: number; longitude: number };
  imageUrl: string;
  onPress: () => void;
}) {
  return (
    <Marker coordinate={coordinate} onPress={onPress}>
      <View style={styles.avatarContainer}>
        <Image
          source={{ uri: imageUrl }}
          style={styles.avatarImage}
          // Cache the image for performance
          // (use expo-image in your actual app)
        />
      </View>
    </Marker>
  );
}

const styles = StyleSheet.create({
  markerContainer: {
    backgroundColor: 'white',
    paddingHorizontal: 8,
    paddingVertical: 4,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: '#DDD',
    alignItems: 'center',
    // Shadow for iOS
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.15,
    shadowRadius: 4,
    // Shadow for Android
    elevation: 4,
  },
  markerSelected: {
    backgroundColor: '#1A1A1A',
    borderColor: '#1A1A1A',
  },
  markerText: {
    fontSize: 14,
    fontWeight: '700',
    color: '#1A1A1A',
  },
  markerTextSelected: {
    color: 'white',
  },
  markerArrow: {
    position: 'absolute',
    bottom: -6,
    width: 12,
    height: 12,
    backgroundColor: 'white',
    borderRightWidth: 1,
    borderBottomWidth: 1,
    borderColor: '#DDD',
    transform: [{ rotate: '45deg' }],
  },
  markerArrowSelected: {
    backgroundColor: '#1A1A1A',
    borderColor: '#1A1A1A',
  },
  avatarContainer: {
    width: 40,
    height: 40,
    borderRadius: 20,
    borderWidth: 2,
    borderColor: 'white',
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
    elevation: 4,
  },
  avatarImage: {
    width: '100%',
    height: '100%',
  },
});
```

### 3.4 Polylines and Routes

Drawing routes on the map:

```tsx
import MapView, { Polyline, Marker } from 'react-native-maps';
import { useState, useEffect } from 'react';
import { View, StyleSheet } from 'react-native';

interface Coordinate {
  latitude: number;
  longitude: number;
}

interface Route {
  coordinates: Coordinate[];
  distance: number; // meters
  duration: number; // seconds
}

// Fetch directions from a routing API
async function getDirections(
  origin: Coordinate,
  destination: Coordinate
): Promise<Route> {
  // Option 1: Google Directions API
  const response = await fetch(
    `https://maps.googleapis.com/maps/api/directions/json?` +
    `origin=${origin.latitude},${origin.longitude}&` +
    `destination=${destination.latitude},${destination.longitude}&` +
    `key=YOUR_API_KEY`
  );
  
  const data = await response.json();
  
  if (data.routes.length === 0) {
    throw new Error('No route found');
  }

  // Decode the polyline
  const route = data.routes[0];
  const coordinates = decodePolyline(route.overview_polyline.points);
  
  return {
    coordinates,
    distance: route.legs[0].distance.value,
    duration: route.legs[0].duration.value,
  };
}

// Google polyline decoder
function decodePolyline(encoded: string): Coordinate[] {
  const coordinates: Coordinate[] = [];
  let index = 0;
  let lat = 0;
  let lng = 0;

  while (index < encoded.length) {
    let shift = 0;
    let result = 0;
    let byte;

    do {
      byte = encoded.charCodeAt(index++) - 63;
      result |= (byte & 0x1f) << shift;
      shift += 5;
    } while (byte >= 0x20);

    const dlat = result & 1 ? ~(result >> 1) : result >> 1;
    lat += dlat;

    shift = 0;
    result = 0;

    do {
      byte = encoded.charCodeAt(index++) - 63;
      result |= (byte & 0x1f) << shift;
      shift += 5;
    } while (byte >= 0x20);

    const dlng = result & 1 ? ~(result >> 1) : result >> 1;
    lng += dlng;

    coordinates.push({
      latitude: lat / 1e5,
      longitude: lng / 1e5,
    });
  }

  return coordinates;
}

function RouteMap({ origin, destination }: { origin: Coordinate; destination: Coordinate }) {
  const [route, setRoute] = useState<Route | null>(null);

  useEffect(() => {
    getDirections(origin, destination)
      .then(setRoute)
      .catch(console.error);
  }, [origin.latitude, origin.longitude, destination.latitude, destination.longitude]);

  return (
    <View style={styles.container}>
      <MapView
        style={styles.map}
        initialRegion={{
          latitude: (origin.latitude + destination.latitude) / 2,
          longitude: (origin.longitude + destination.longitude) / 2,
          latitudeDelta: Math.abs(origin.latitude - destination.latitude) * 2,
          longitudeDelta: Math.abs(origin.longitude - destination.longitude) * 2,
        }}
      >
        {/* Origin marker */}
        <Marker coordinate={origin} pinColor="green" title="Start" />
        
        {/* Destination marker */}
        <Marker coordinate={destination} pinColor="red" title="End" />
        
        {/* Route polyline */}
        {route && (
          <Polyline
            coordinates={route.coordinates}
            strokeColor="#007AFF"
            strokeWidth={4}
            lineDashPattern={undefined} // Solid line; use [10, 5] for dashed
          />
        )}
      </MapView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  map: { flex: 1 },
});
```

### 3.5 Marker Clustering

When you have hundreds or thousands of markers, rendering them all individually destroys performance. Clustering groups nearby markers into a single cluster marker that shows a count.

```tsx
import MapView, { Marker, Region } from 'react-native-maps';
import { View, Text, StyleSheet } from 'react-native';
import { useState, useMemo, useCallback } from 'react';
// react-native-maps has clustering built-in via react-native-map-clustering
// Or you can use supercluster for manual control
import Supercluster from 'supercluster';

interface MapPoint {
  id: string;
  latitude: number;
  longitude: number;
  name: string;
}

function ClusteredMap({ points }: { points: MapPoint[] }) {
  const [region, setRegion] = useState<Region>({
    latitude: 37.7749,
    longitude: -122.4194,
    latitudeDelta: 0.5,
    longitudeDelta: 0.5,
  });

  // Create supercluster index
  const cluster = useMemo(() => {
    const sc = new Supercluster({
      radius: 40,        // Cluster radius in pixels
      maxZoom: 16,       // Max zoom to cluster at
      minZoom: 0,
      minPoints: 2,      // Minimum points to form a cluster
    });

    // Load points as GeoJSON features
    sc.load(
      points.map(point => ({
        type: 'Feature' as const,
        geometry: {
          type: 'Point' as const,
          coordinates: [point.longitude, point.latitude],
        },
        properties: {
          id: point.id,
          name: point.name,
        },
      }))
    );

    return sc;
  }, [points]);

  // Get clusters for current viewport
  const clusteredMarkers = useMemo(() => {
    const bbox: [number, number, number, number] = [
      region.longitude - region.longitudeDelta / 2, // west
      region.latitude - region.latitudeDelta / 2,    // south
      region.longitude + region.longitudeDelta / 2,  // east
      region.latitude + region.latitudeDelta / 2,    // north
    ];

    // Convert latitudeDelta to zoom level (approximate)
    const zoom = Math.round(Math.log2(360 / region.longitudeDelta));

    return cluster.getClusters(bbox, zoom);
  }, [cluster, region]);

  const handleRegionChange = useCallback((newRegion: Region) => {
    setRegion(newRegion);
  }, []);

  return (
    <MapView
      style={styles.map}
      initialRegion={region}
      onRegionChangeComplete={handleRegionChange}
    >
      {clusteredMarkers.map((feature, index) => {
        const [longitude, latitude] = feature.geometry.coordinates;
        const isCluster = feature.properties.cluster;

        if (isCluster) {
          const pointCount = feature.properties.point_count;
          const size = 30 + (pointCount > 100 ? 20 : pointCount > 10 ? 10 : 0);
          
          return (
            <Marker
              key={`cluster-${feature.properties.cluster_id}`}
              coordinate={{ latitude, longitude }}
              tracksViewChanges={false}
            >
              <View style={[styles.clusterMarker, { width: size, height: size, borderRadius: size / 2 }]}>
                <Text style={styles.clusterText}>{pointCount}</Text>
              </View>
            </Marker>
          );
        }

        return (
          <Marker
            key={feature.properties.id}
            coordinate={{ latitude, longitude }}
            title={feature.properties.name}
            tracksViewChanges={false}
          />
        );
      })}
    </MapView>
  );
}

const styles = StyleSheet.create({
  map: { flex: 1 },
  clusterMarker: {
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
    borderWidth: 2,
    borderColor: 'white',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.2,
    shadowRadius: 4,
    elevation: 4,
  },
  clusterText: {
    color: 'white',
    fontSize: 14,
    fontWeight: '700',
  },
});
```

### 3.6 Mapbox Alternative

When you need offline maps, advanced styling, or 3D terrain, Mapbox is the way to go. It's more expensive but more capable.

```bash
npx expo install @rnmapbox/maps
```

```tsx
import Mapbox from '@rnmapbox/maps';
import { View, StyleSheet } from 'react-native';

// Set your access token (do this once, in your app's entry point)
Mapbox.setAccessToken('YOUR_MAPBOX_ACCESS_TOKEN');

function MapboxScreen() {
  return (
    <View style={styles.container}>
      <Mapbox.MapView
        style={styles.map}
        styleURL={Mapbox.StyleURL.Dark} // Built-in dark mode!
        zoomEnabled
        rotateEnabled
      >
        <Mapbox.Camera
          centerCoordinate={[-122.4194, 37.7749]}
          zoomLevel={12}
          animationMode="flyTo"
          animationDuration={2000}
        />
        
        {/* User location */}
        <Mapbox.UserLocation visible />
        
        {/* Marker */}
        <Mapbox.PointAnnotation
          id="marker1"
          coordinate={[-122.4194, 37.7749]}
          title="San Francisco"
        >
          <View style={styles.mapboxMarker} />
        </Mapbox.PointAnnotation>
        
        {/* Route line */}
        <Mapbox.ShapeSource
          id="routeSource"
          shape={{
            type: 'Feature',
            geometry: {
              type: 'LineString',
              coordinates: [
                [-122.4194, 37.7749],
                [-122.4094, 37.7849],
                [-122.3994, 37.7949],
              ],
            },
            properties: {},
          }}
        >
          <Mapbox.LineLayer
            id="routeLayer"
            style={{
              lineColor: '#007AFF',
              lineWidth: 4,
              lineCap: 'round',
              lineJoin: 'round',
            }}
          />
        </Mapbox.ShapeSource>
      </Mapbox.MapView>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  map: { flex: 1 },
  mapboxMarker: {
    width: 20,
    height: 20,
    borderRadius: 10,
    backgroundColor: '#007AFF',
    borderWidth: 2,
    borderColor: 'white',
  },
});
```

**When to use Mapbox over react-native-maps:**
- You need offline map support (download map tiles for areas)
- You need custom map styles (branded map appearance)
- You need 3D terrain or building extrusions
- You need advanced clustering with animations
- You need turn-by-turn navigation (Mapbox Navigation SDK)

**When to stick with react-native-maps:**
- Simple marker-based maps
- You're already using Google Maps APIs
- You don't want the extra dependency cost
- You need Apple Maps on iOS (some users prefer it)

---

## 4. CONTACTS: READING AND CREATING CONTACT RECORDS

### 4.1 Setting Up expo-contacts

```bash
npx expo install expo-contacts
```

```json
{
  "expo": {
    "plugins": [
      [
        "expo-contacts",
        {
          "contactsPermission": "Allow $(PRODUCT_NAME) to access your contacts to help you invite friends."
        }
      ]
    ]
  }
}
```

### 4.2 Reading Contacts

```tsx
import * as Contacts from 'expo-contacts';
import { useState, useCallback } from 'react';
import { View, Text, FlatList, StyleSheet, Pressable, TextInput } from 'react-native';

interface ContactItem {
  id: string;
  name: string;
  phoneNumber: string | null;
  email: string | null;
  imageUri: string | null;
}

function useContacts() {
  const [contacts, setContacts] = useState<ContactItem[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadContacts = useCallback(async () => {
    setLoading(true);
    setError(null);

    try {
      const { status } = await Contacts.requestPermissionsAsync();
      
      if (status !== 'granted') {
        setError('Contacts permission denied');
        return;
      }

      const { data } = await Contacts.getContactsAsync({
        fields: [
          Contacts.Fields.Name,
          Contacts.Fields.PhoneNumbers,
          Contacts.Fields.Emails,
          Contacts.Fields.Image,
        ],
        sort: Contacts.SortTypes.FirstName,
        pageSize: 500,
        pageOffset: 0,
      });

      const formattedContacts: ContactItem[] = data.map(contact => ({
        id: contact.id ?? Math.random().toString(),
        name: contact.name ?? 'Unknown',
        phoneNumber: contact.phoneNumbers?.[0]?.number ?? null,
        email: contact.emails?.[0]?.email ?? null,
        imageUri: contact.image?.uri ?? null,
      }));

      setContacts(formattedContacts);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load contacts');
    } finally {
      setLoading(false);
    }
  }, []);

  return { contacts, loading, error, loadContacts };
}

function ContactPicker({ onSelect }: { onSelect: (contact: ContactItem) => void }) {
  const { contacts, loading, error, loadContacts } = useContacts();
  const [search, setSearch] = useState('');

  const filteredContacts = contacts.filter(contact =>
    contact.name.toLowerCase().includes(search.toLowerCase()) ||
    contact.phoneNumber?.includes(search) ||
    contact.email?.toLowerCase().includes(search.toLowerCase())
  );

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.searchInput}
        placeholder="Search contacts..."
        value={search}
        onChangeText={setSearch}
        autoCapitalize="none"
      />
      
      {contacts.length === 0 && !loading && (
        <Pressable style={styles.loadButton} onPress={loadContacts}>
          <Text style={styles.loadButtonText}>Load Contacts</Text>
        </Pressable>
      )}

      {error && <Text style={styles.error}>{error}</Text>}

      <FlatList
        data={filteredContacts}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <Pressable
            style={styles.contactRow}
            onPress={() => onSelect(item)}
          >
            <View style={styles.avatar}>
              <Text style={styles.avatarText}>
                {item.name.charAt(0).toUpperCase()}
              </Text>
            </View>
            <View style={styles.contactInfo}>
              <Text style={styles.contactName}>{item.name}</Text>
              {item.phoneNumber && (
                <Text style={styles.contactDetail}>{item.phoneNumber}</Text>
              )}
              {item.email && (
                <Text style={styles.contactDetail}>{item.email}</Text>
              )}
            </View>
          </Pressable>
        )}
        ItemSeparatorComponent={() => <View style={styles.separator} />}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: 'white' },
  searchInput: {
    padding: 16,
    fontSize: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#E5E5E5',
  },
  loadButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    margin: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  loadButtonText: { color: 'white', fontSize: 16, fontWeight: '600' },
  error: { color: 'red', padding: 16, textAlign: 'center' },
  contactRow: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
    paddingHorizontal: 16,
  },
  avatar: {
    width: 40,
    height: 40,
    borderRadius: 20,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  avatarText: { color: 'white', fontSize: 16, fontWeight: '700' },
  contactInfo: { flex: 1 },
  contactName: { fontSize: 16, fontWeight: '600', color: '#1A1A1A' },
  contactDetail: { fontSize: 14, color: '#666', marginTop: 2 },
  separator: { height: 1, backgroundColor: '#F0F0F0', marginLeft: 68 },
});
```

### 4.3 Creating a Contact

```tsx
import * as Contacts from 'expo-contacts';
import { Alert, Platform, Linking } from 'react-native';

async function createContact(data: {
  firstName: string;
  lastName: string;
  phoneNumber?: string;
  email?: string;
  company?: string;
}) {
  const { status } = await Contacts.requestPermissionsAsync();
  if (status !== 'granted') {
    Alert.alert(
      'Permission Required',
      'Please enable contacts access in Settings to save contacts.',
      [
        { text: 'Cancel', style: 'cancel' },
        { 
          text: 'Open Settings', 
          onPress: () => Linking.openSettings() 
        },
      ]
    );
    return;
  }

  const contact: Contacts.Contact = {
    [Contacts.Fields.FirstName]: data.firstName,
    [Contacts.Fields.LastName]: data.lastName,
    contactType: Contacts.ContactTypes.Person,
  };

  if (data.phoneNumber) {
    contact[Contacts.Fields.PhoneNumbers] = [
      {
        number: data.phoneNumber,
        label: 'mobile',
        isPrimary: true,
      },
    ];
  }

  if (data.email) {
    contact[Contacts.Fields.Emails] = [
      {
        email: data.email,
        label: 'work',
        isPrimary: true,
      },
    ];
  }

  if (data.company) {
    contact[Contacts.Fields.Company] = data.company;
  }

  try {
    const contactId = await Contacts.addContactAsync(contact);
    console.log('Contact created with ID:', contactId);
    return contactId;
  } catch (error) {
    console.error('Failed to create contact:', error);
    throw error;
  }
}
```

---

## 5. CALENDAR: EVENTS AND REMINDERS

### 5.1 Setting Up expo-calendar

```bash
npx expo install expo-calendar
```

```json
{
  "expo": {
    "plugins": [
      [
        "expo-calendar",
        {
          "calendarPermission": "Allow $(PRODUCT_NAME) to access your calendar to add event reminders.",
          "remindersPermission": "Allow $(PRODUCT_NAME) to access your reminders to set task due dates."
        }
      ]
    ]
  }
}
```

### 5.2 Reading Calendars and Events

```tsx
import * as Calendar from 'expo-calendar';
import { Platform } from 'react-native';

// Get all calendars
async function getCalendars() {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Calendar permission denied');
  }

  const calendars = await Calendar.getCalendarsAsync(
    Calendar.EntityTypes.EVENT
  );

  // Filter to writable calendars
  const writableCalendars = calendars.filter(cal => cal.allowsModifications);
  
  return writableCalendars;
}

// Get events within a date range
async function getEvents(
  startDate: Date,
  endDate: Date,
  calendarIds?: string[]
) {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Calendar permission denied');
  }

  // If no calendar IDs specified, use all calendars
  const ids = calendarIds ?? (await Calendar.getCalendarsAsync(
    Calendar.EntityTypes.EVENT
  )).map(cal => cal.id);

  const events = await Calendar.getEventsAsync(
    ids,
    startDate,
    endDate
  );

  return events.map(event => ({
    id: event.id,
    title: event.title,
    startDate: new Date(event.startDate),
    endDate: new Date(event.endDate),
    location: event.location,
    notes: event.notes,
    allDay: event.allDay,
    calendarId: event.calendarId,
  }));
}

// Usage
async function getThisWeeksEvents() {
  const now = new Date();
  const endOfWeek = new Date();
  endOfWeek.setDate(now.getDate() + 7);
  
  const events = await getEvents(now, endOfWeek);
  console.log(`You have ${events.length} events this week`);
  return events;
}
```

### 5.3 Creating Events

```tsx
import * as Calendar from 'expo-calendar';
import { Platform } from 'react-native';

// Get the default calendar ID (or create one for your app)
async function getDefaultCalendarId(): Promise<string> {
  if (Platform.OS === 'ios') {
    const defaultCalendar = await Calendar.getDefaultCalendarAsync();
    return defaultCalendar.id;
  }
  
  // On Android, find the primary Google calendar or create one
  const calendars = await Calendar.getCalendarsAsync(Calendar.EntityTypes.EVENT);
  const primaryCalendar = calendars.find(
    cal => cal.isPrimary && cal.allowsModifications
  );
  
  if (primaryCalendar) return primaryCalendar.id;
  
  // If no primary calendar, find any writable calendar
  const writableCalendar = calendars.find(cal => cal.allowsModifications);
  if (writableCalendar) return writableCalendar.id;
  
  // Last resort: create a local calendar
  const newCalendarId = await Calendar.createCalendarAsync({
    title: 'MyApp Events',
    color: '#007AFF',
    entityType: Calendar.EntityTypes.EVENT,
    sourceId: calendars[0]?.source?.id,
    source: {
      name: calendars[0]?.source?.name ?? 'Local',
      isLocalAccount: true,
      type: calendars[0]?.source?.type,
    },
    name: 'myapp-events',
    ownerAccount: 'personal',
    accessLevel: Calendar.CalendarAccessLevel.OWNER,
  });
  
  return newCalendarId;
}

// Create a calendar event
async function createEvent(data: {
  title: string;
  startDate: Date;
  endDate: Date;
  location?: string;
  notes?: string;
  allDay?: boolean;
  alarms?: number[]; // Minutes before event to trigger alarm
}) {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Calendar permission denied');
  }

  const calendarId = await getDefaultCalendarId();

  const eventId = await Calendar.createEventAsync(calendarId, {
    title: data.title,
    startDate: data.startDate,
    endDate: data.endDate,
    location: data.location,
    notes: data.notes,
    allDay: data.allDay ?? false,
    timeZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
    alarms: data.alarms?.map(minutes => ({
      relativeOffset: -minutes, // Negative = before event
      method: Calendar.AlarmMethod.ALERT,
    })) ?? [{ relativeOffset: -15, method: Calendar.AlarmMethod.ALERT }],
  });

  console.log('Event created with ID:', eventId);
  return eventId;
}

// Usage
async function scheduleAppointment() {
  const tomorrow = new Date();
  tomorrow.setDate(tomorrow.getDate() + 1);
  tomorrow.setHours(10, 0, 0, 0);
  
  const endTime = new Date(tomorrow);
  endTime.setHours(11, 0, 0, 0);

  const eventId = await createEvent({
    title: 'Team Standup',
    startDate: tomorrow,
    endDate: endTime,
    location: 'Conference Room A',
    notes: 'Weekly sync meeting',
    alarms: [15, 5], // Alert 15 min and 5 min before
  });
  
  return eventId;
}
```

### 5.4 Reminders (iOS Only)

```tsx
import * as Calendar from 'expo-calendar';
import { Platform } from 'react-native';

async function createReminder(data: {
  title: string;
  dueDate?: Date;
  notes?: string;
}) {
  if (Platform.OS !== 'ios') {
    console.warn('Reminders are only supported on iOS');
    return;
  }

  const { status } = await Calendar.requestRemindersPermissionsAsync();
  if (status !== 'granted') {
    throw new Error('Reminders permission denied');
  }

  const calendars = await Calendar.getCalendarsAsync(Calendar.EntityTypes.REMINDER);
  const defaultCalendar = calendars.find(cal => cal.allowsModifications);
  
  if (!defaultCalendar) {
    throw new Error('No writable reminder calendar found');
  }

  const reminderId = await Calendar.createReminderAsync(defaultCalendar.id, {
    title: data.title,
    dueDate: data.dueDate,
    notes: data.notes,
    completed: false,
    alarms: data.dueDate
      ? [{ relativeOffset: -60 }] // 1 hour before
      : undefined,
  });

  return reminderId;
}
```

---

## 6. HAPTICS: IMPACT, NOTIFICATION, AND SELECTION FEEDBACK

Haptics are what separate a "meh" app from one that feels native. Every time your user taps a button, completes an action, or swipes something away — that little buzz says "your input was received."

### 6.1 Setting Up expo-haptics

```bash
npx expo install expo-haptics
```

No permissions required. No config plugin needed. Just install and use.

### 6.2 The Three Types of Haptic Feedback

```tsx
import * as Haptics from 'expo-haptics';

// 1. IMPACT — Physical tap feedback
// Use for: button presses, toggles, selections
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);

// iOS also supports:
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Soft);
await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Rigid);

// 2. NOTIFICATION — Outcome feedback
// Use for: success, warning, error states
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);

// 3. SELECTION — Tick feedback
// Use for: scrolling through options, moving between items
await Haptics.selectionAsync();
```

### 6.3 When to Use Each Type

Here's my opinionated guide to haptic feedback:

```tsx
import * as Haptics from 'expo-haptics';
import { Platform } from 'react-native';

// Wrap haptics in a safe helper
// (does nothing on Android if haptics aren't supported, 
// does nothing on web)
const haptic = {
  // Button press — light impact
  tap: () => {
    if (Platform.OS !== 'web') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    }
  },
  
  // Toggle switch, segment control — medium impact
  toggle: () => {
    if (Platform.OS !== 'web') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    }
  },
  
  // Destructive action confirmation — heavy impact
  heavy: () => {
    if (Platform.OS !== 'web') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
    }
  },
  
  // Action succeeded (payment processed, item saved)
  success: () => {
    if (Platform.OS !== 'web') {
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
    }
  },
  
  // Something went wrong (validation error, network failure)
  error: () => {
    if (Platform.OS !== 'web') {
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Error);
    }
  },
  
  // Warning state (low battery, nearing limit)
  warning: () => {
    if (Platform.OS !== 'web') {
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
    }
  },
  
  // Scrolling through options (picker, time selector)
  selection: () => {
    if (Platform.OS !== 'web') {
      Haptics.selectionAsync();
    }
  },
};

export { haptic };
```

### 6.4 Haptics in Practice

```tsx
import { Pressable, Text, Switch, View, StyleSheet } from 'react-native';
import { haptic } from './haptics';
import { useState } from 'react';

// Button with haptic feedback
function HapticButton({ 
  title, 
  onPress, 
  variant = 'primary' 
}: {
  title: string;
  onPress: () => void;
  variant?: 'primary' | 'destructive';
}) {
  return (
    <Pressable
      style={[styles.button, variant === 'destructive' && styles.destructive]}
      onPress={() => {
        haptic.tap();
        onPress();
      }}
    >
      <Text style={styles.buttonText}>{title}</Text>
    </Pressable>
  );
}

// Toggle with haptic feedback
function HapticToggle({ 
  value, 
  onValueChange 
}: {
  value: boolean;
  onValueChange: (value: boolean) => void;
}) {
  return (
    <Switch
      value={value}
      onValueChange={(newValue) => {
        haptic.toggle();
        onValueChange(newValue);
      }}
    />
  );
}

// Form submission with success/error haptics
function SubmitForm() {
  const [loading, setLoading] = useState(false);

  const handleSubmit = async () => {
    setLoading(true);
    try {
      await submitToServer();
      haptic.success(); // Buzz on success
      // Show success UI
    } catch (error) {
      haptic.error(); // Different buzz on error
      // Show error UI
    } finally {
      setLoading(false);
    }
  };

  return (
    <Pressable style={styles.button} onPress={handleSubmit}>
      <Text style={styles.buttonText}>
        {loading ? 'Submitting...' : 'Submit'}
      </Text>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
  },
  destructive: {
    backgroundColor: '#FF3B30',
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 6.5 When NOT to Use Haptics

This is important. Overusing haptics is worse than not using them at all:

- **Never on scroll events.** Your user will feel like they're holding a vibrating phone.
- **Never on every keystroke.** The keyboard already has its own haptic feedback (if enabled).
- **Never on rapid-fire events.** If something fires more than once per 100ms, skip the haptic.
- **Never as the ONLY feedback.** Haptics should supplement visual/audio feedback, not replace it. Some users have haptics disabled.
- **Be careful on Android.** Android haptic support varies wildly by device. The Pixel 6+ has excellent haptics. A budget Samsung from 2020 has a vibration motor that feels like a washing machine.

---

## 7. CLIPBOARD: COPY/PASTE TEXT AND IMAGES

### 7.1 Setting Up expo-clipboard

```bash
npx expo install expo-clipboard
```

### 7.2 Basic Copy/Paste

```tsx
import * as Clipboard from 'expo-clipboard';
import { useState } from 'react';
import { View, Text, Pressable, TextInput, StyleSheet, Alert } from 'react-native';
import { haptic } from './haptics';

function CopyableText({ text }: { text: string }) {
  const [copied, setCopied] = useState(false);

  const handleCopy = async () => {
    await Clipboard.setStringAsync(text);
    haptic.success();
    setCopied(true);
    
    // Reset after 2 seconds
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <Pressable style={styles.copyContainer} onPress={handleCopy}>
      <Text style={styles.copyText} numberOfLines={1}>{text}</Text>
      <Text style={styles.copyButton}>
        {copied ? 'Copied!' : 'Copy'}
      </Text>
    </Pressable>
  );
}

// Paste from clipboard
function PasteInput() {
  const [value, setValue] = useState('');

  const handlePaste = async () => {
    const hasString = await Clipboard.hasStringAsync();
    if (!hasString) {
      Alert.alert('Nothing to paste', 'Your clipboard is empty.');
      return;
    }

    const text = await Clipboard.getStringAsync();
    setValue(text);
    haptic.tap();
  };

  return (
    <View style={styles.pasteContainer}>
      <TextInput
        style={styles.input}
        value={value}
        onChangeText={setValue}
        placeholder="Type or paste here..."
      />
      <Pressable style={styles.pasteButton} onPress={handlePaste}>
        <Text style={styles.pasteButtonText}>Paste</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  copyContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
    backgroundColor: '#F5F5F5',
    borderRadius: 8,
    gap: 8,
  },
  copyText: {
    flex: 1,
    fontSize: 14,
    fontFamily: Platform.OS === 'ios' ? 'Menlo' : 'monospace',
    color: '#1A1A1A',
  },
  copyButton: {
    fontSize: 14,
    fontWeight: '600',
    color: '#007AFF',
  },
  pasteContainer: {
    flexDirection: 'row',
    gap: 8,
  },
  input: {
    flex: 1,
    padding: 12,
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 8,
    fontSize: 16,
  },
  pasteButton: {
    padding: 12,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    justifyContent: 'center',
  },
  pasteButtonText: {
    color: 'white',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

### 7.3 Clipboard with Images

```tsx
import * as Clipboard from 'expo-clipboard';
import { Image as ExpoImage } from 'expo-image';
import { useState } from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';

// Copy an image to clipboard
async function copyImage(base64Image: string) {
  await Clipboard.setImageAsync(base64Image);
}

// Check and paste an image from clipboard
function ClipboardImageViewer() {
  const [imageUri, setImageUri] = useState<string | null>(null);

  const checkClipboard = async () => {
    const hasImage = await Clipboard.hasImageAsync();
    
    if (hasImage) {
      const image = await Clipboard.getImageAsync({
        format: 'png',
      });
      
      if (image) {
        setImageUri(image.data); // base64 data URI
      }
    }
  };

  return (
    <View style={styles.container}>
      <Pressable style={styles.button} onPress={checkClipboard}>
        <Text style={styles.buttonText}>Paste Image from Clipboard</Text>
      </Pressable>
      
      {imageUri && (
        <ExpoImage
          source={{ uri: imageUri }}
          style={styles.pastedImage}
          contentFit="contain"
        />
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 16 },
  button: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
    marginBottom: 16,
  },
  buttonText: { color: 'white', fontSize: 16, fontWeight: '600' },
  pastedImage: { width: '100%', height: 300, borderRadius: 8 },
});
```

### 7.4 Clipboard Listener

Watch for clipboard changes (useful for OTP auto-fill):

```tsx
import * as Clipboard from 'expo-clipboard';
import { useEffect, useState } from 'react';

function useClipboardListener(
  pattern?: RegExp // Optional pattern to match (e.g., /^\d{6}$/ for 6-digit OTP)
) {
  const [clipboardContent, setClipboardContent] = useState<string | null>(null);

  useEffect(() => {
    const subscription = Clipboard.addClipboardListener(
      ({ contentTypes }) => {
        if (contentTypes.includes(Clipboard.ContentType.PLAIN_TEXT)) {
          Clipboard.getStringAsync().then(text => {
            if (pattern && !pattern.test(text)) return;
            setClipboardContent(text);
          });
        }
      }
    );

    return () => subscription.remove();
  }, [pattern]);

  return clipboardContent;
}

// Usage: Auto-detect OTP codes
function OTPInput() {
  const clipboardOTP = useClipboardListener(/^\d{6}$/);
  const [otp, setOTP] = useState('');

  useEffect(() => {
    if (clipboardOTP) {
      setOTP(clipboardOTP);
      // Auto-submit if you want
    }
  }, [clipboardOTP]);

  return (
    <TextInput
      value={otp}
      onChangeText={setOTP}
      placeholder="Enter 6-digit code"
      keyboardType="number-pad"
      maxLength={6}
    />
  );
}
```

---

## 8. SHARE SHEET: SHARING CONTENT AND RECEIVING SHARED CONTENT

### 8.1 Sharing with the React Native Share API

The simplest way to share content:

```tsx
import { Share, Platform } from 'react-native';

// Share text/URL
async function shareContent(data: {
  title?: string;
  message: string;
  url?: string;
}) {
  try {
    const result = await Share.share(
      {
        title: data.title,
        message: Platform.OS === 'ios' 
          ? data.message  // iOS: message and url are separate
          : `${data.message}${data.url ? `\n${data.url}` : ''}`, // Android: combine
        url: data.url,    // iOS only — ignored on Android
      },
      {
        // iOS options
        subject: data.title,              // Email subject
        excludedActivityTypes: [          // Hide specific share targets
          'com.apple.UIKit.activity.Print',
          'com.apple.UIKit.activity.AssignToContact',
        ],
        // Android options
        dialogTitle: data.title,
      }
    );

    if (result.action === Share.sharedAction) {
      if (result.activityType) {
        // iOS: shared via specific activity (e.g., 'com.apple.UIKit.activity.Mail')
        console.log('Shared via:', result.activityType);
      } else {
        // Android: shared (no activity type available)
        console.log('Shared successfully');
      }
    } else if (result.action === Share.dismissedAction) {
      // iOS only: user dismissed the share sheet
      console.log('Share dismissed');
    }
  } catch (error) {
    console.error('Share failed:', error);
  }
}

// Usage
shareContent({
  title: 'Check out this app!',
  message: 'I found this amazing restaurant on FoodApp.',
  url: 'https://foodapp.com/restaurant/123',
});
```

### 8.2 Sharing Files with expo-sharing

For sharing files (images, PDFs, etc.), use `expo-sharing`:

```bash
npx expo install expo-sharing
```

```tsx
import * as Sharing from 'expo-sharing';
import * as FileSystem from 'expo-file-system';

// Share a file
async function shareFile(fileUri: string) {
  // Check if sharing is available (not available on all platforms)
  const isAvailable = await Sharing.isAvailableAsync();
  if (!isAvailable) {
    alert('Sharing is not available on this device');
    return;
  }

  await Sharing.shareAsync(fileUri, {
    mimeType: 'application/pdf', // Specify MIME type for better share target filtering
    dialogTitle: 'Share Document',
    UTI: 'com.adobe.pdf', // iOS: Uniform Type Identifier
  });
}

// Download and share an image
async function downloadAndShare(imageUrl: string) {
  // Download to a temporary file
  const fileUri = FileSystem.cacheDirectory + 'shared-image.jpg';
  
  const downloadResult = await FileSystem.downloadAsync(
    imageUrl,
    fileUri
  );

  if (downloadResult.status !== 200) {
    throw new Error('Failed to download image');
  }

  await Sharing.shareAsync(downloadResult.uri, {
    mimeType: 'image/jpeg',
    UTI: 'public.jpeg',
  });
}

// Share a generated file (e.g., export data as CSV)
async function shareCSV(data: Array<Record<string, string>>) {
  const headers = Object.keys(data[0]).join(',');
  const rows = data.map(row => Object.values(row).join(','));
  const csv = [headers, ...rows].join('\n');

  const fileUri = FileSystem.cacheDirectory + 'export.csv';
  await FileSystem.writeAsStringAsync(fileUri, csv);

  await Sharing.shareAsync(fileUri, {
    mimeType: 'text/csv',
    dialogTitle: 'Export Data',
  });
}
```

### 8.3 Receiving Shared Content (Share Extensions)

This is more advanced — receiving content when a user shares TO your app from another app:

```tsx
// For Expo, receiving shared content requires a config plugin
// and native code. The expo-share-extension community package helps:

// app.json
{
  "expo": {
    "plugins": [
      // Community package for share extensions
      // Note: This requires custom native code
    ]
  }
}

// The basic pattern for handling incoming shared content
// in your app's entry point:

import { Linking } from 'react-native';
import { useEffect } from 'react';

function App() {
  useEffect(() => {
    // Handle incoming URLs (deep links from share extensions)
    const subscription = Linking.addEventListener('url', ({ url }) => {
      // Parse the shared content from the URL
      const sharedData = parseSharedUrl(url);
      if (sharedData) {
        // Navigate to the appropriate screen
        // e.g., create a new post with the shared content
      }
    });

    // Check for initial URL (app was launched via share)
    Linking.getInitialURL().then(url => {
      if (url) {
        const sharedData = parseSharedUrl(url);
        // Handle initial shared content
      }
    });

    return () => subscription.remove();
  }, []);

  // ...
}
```

**Honest take:** Receiving shared content (share extensions) in React Native is still painful. On iOS, you need a native Share Extension target, which means ejecting or using a config plugin. On Android, you need intent filters. If this is a core feature of your app, plan for native code. If it's nice-to-have, consider skipping it for v1.

---

## 9. BARCODE/QR SCANNING: CUSTOM SCANNING UI

We covered basic barcode scanning in Section 1.6 with the camera. Let's go deeper with a production-ready scanner.

### 9.1 Supported Barcode Formats

```tsx
// expo-camera supports these barcode types:
type BarcodeType =
  | 'aztec'        // Aztec codes
  | 'ean13'        // EAN-13 (product barcodes)
  | 'ean8'         // EAN-8
  | 'qr'           // QR codes
  | 'pdf417'       // PDF417 (driver's licenses)
  | 'upc_e'        // UPC-E (smaller product barcodes)
  | 'datamatrix'   // Data Matrix
  | 'code39'       // Code 39 (warehouse, manufacturing)
  | 'code93'       // Code 93
  | 'itf14'        // ITF-14 (shipping containers)
  | 'codabar'      // Codabar (libraries, blood banks)
  | 'code128'      // Code 128 (logistics, shipping)
  | 'upc_a';       // UPC-A (retail products)
```

### 9.2 Production QR Scanner with Validation

```tsx
import { CameraView, useCameraPermissions, BarcodeScanningResult } from 'expo-camera';
import { useState, useCallback, useRef } from 'react';
import { View, Text, StyleSheet, Pressable, Linking, Alert } from 'react-native';
import Animated, { 
  useSharedValue, 
  useAnimatedStyle, 
  withRepeat, 
  withTiming,
  FadeIn,
} from 'react-native-reanimated';
import { haptic } from './haptics';

type ScanResult = 
  | { type: 'url'; value: string }
  | { type: 'text'; value: string }
  | { type: 'product'; value: string; format: string };

function parseScanResult(result: BarcodeScanningResult): ScanResult {
  const { data, type } = result;
  
  // URL detection
  if (data.startsWith('http://') || data.startsWith('https://')) {
    return { type: 'url', value: data };
  }
  
  // Product barcode formats
  if (['ean13', 'ean8', 'upc_a', 'upc_e'].includes(type)) {
    return { type: 'product', value: data, format: type };
  }
  
  // Default: treat as text
  return { type: 'text', value: data };
}

function ProductionQRScanner({ 
  onScan 
}: { 
  onScan: (result: ScanResult) => void 
}) {
  const [permission, requestPermission] = useCameraPermissions();
  const [scannedResult, setScannedResult] = useState<ScanResult | null>(null);
  const [isProcessing, setIsProcessing] = useState(false);
  const lastScanRef = useRef<string>('');
  const scanLineY = useSharedValue(0);

  // Animated scan line
  scanLineY.value = withRepeat(
    withTiming(1, { duration: 2000 }),
    -1,   // Infinite repeats
    true   // Reverse
  );

  const scanLineStyle = useAnimatedStyle(() => ({
    top: `${scanLineY.value * 100}%`,
  }));

  const handleBarCodeScanned = useCallback(
    async (result: BarcodeScanningResult) => {
      // Debounce: don't process the same code twice within 3 seconds
      if (result.data === lastScanRef.current) return;
      if (isProcessing) return;

      setIsProcessing(true);
      lastScanRef.current = result.data;
      
      haptic.success();

      const parsed = parseScanResult(result);
      setScannedResult(parsed);
      onScan(parsed);

      // Reset after 3 seconds to allow re-scanning
      setTimeout(() => {
        lastScanRef.current = '';
        setIsProcessing(false);
      }, 3000);
    },
    [isProcessing, onScan]
  );

  const handleResultAction = useCallback(async () => {
    if (!scannedResult) return;

    switch (scannedResult.type) {
      case 'url':
        const canOpen = await Linking.canOpenURL(scannedResult.value);
        if (canOpen) {
          Alert.alert(
            'Open Link?',
            scannedResult.value,
            [
              { text: 'Cancel', style: 'cancel' },
              { 
                text: 'Open', 
                onPress: () => Linking.openURL(scannedResult.value) 
              },
            ]
          );
        }
        break;
      case 'product':
        // Look up product by barcode
        console.log(`Product barcode: ${scannedResult.value}`);
        break;
      case 'text':
        // Copy to clipboard or process
        console.log(`Scanned text: ${scannedResult.value}`);
        break;
    }
  }, [scannedResult]);

  if (!permission) return <View style={styles.loading} />;

  if (!permission.granted) {
    return (
      <View style={styles.permissionContainer}>
        <Text style={styles.permissionTitle}>Camera Access Required</Text>
        <Text style={styles.permissionText}>
          We need camera access to scan QR codes and barcodes.
        </Text>
        {permission.canAskAgain ? (
          <Pressable style={styles.permissionButton} onPress={requestPermission}>
            <Text style={styles.permissionButtonText}>Allow Camera Access</Text>
          </Pressable>
        ) : (
          <Pressable style={styles.permissionButton} onPress={() => Linking.openSettings()}>
            <Text style={styles.permissionButtonText}>Open Settings</Text>
          </Pressable>
        )}
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <CameraView
        style={styles.camera}
        barcodeScannerSettings={{
          barcodeTypes: ['qr', 'ean13', 'ean8', 'code128', 'upc_a'],
        }}
        onBarcodeScanned={handleBarCodeScanned}
      >
        {/* Scan overlay with animated line */}
        <View style={styles.scanArea}>
          <Animated.View style={[styles.scanLine, scanLineStyle]} />
        </View>
      </CameraView>

      {/* Result display */}
      {scannedResult && (
        <Animated.View entering={FadeIn} style={styles.resultContainer}>
          <Text style={styles.resultType}>{scannedResult.type.toUpperCase()}</Text>
          <Text style={styles.resultValue} numberOfLines={3}>
            {scannedResult.value}
          </Text>
          <Pressable style={styles.actionButton} onPress={handleResultAction}>
            <Text style={styles.actionButtonText}>
              {scannedResult.type === 'url' ? 'Open Link' : 'Copy'}
            </Text>
          </Pressable>
        </Animated.View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  loading: { flex: 1, backgroundColor: '#000' },
  camera: { flex: 1 },
  scanArea: {
    position: 'absolute',
    top: '25%',
    left: '15%',
    right: '15%',
    bottom: '35%',
    borderWidth: 2,
    borderColor: 'rgba(255,255,255,0.5)',
    borderRadius: 12,
    overflow: 'hidden',
  },
  scanLine: {
    position: 'absolute',
    left: 0,
    right: 0,
    height: 2,
    backgroundColor: '#007AFF',
  },
  permissionContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 32,
    backgroundColor: '#000',
  },
  permissionTitle: {
    fontSize: 20,
    fontWeight: '700',
    color: 'white',
    marginBottom: 12,
  },
  permissionText: {
    fontSize: 16,
    color: '#AAA',
    textAlign: 'center',
    marginBottom: 24,
  },
  permissionButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
  },
  permissionButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  resultContainer: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    padding: 24,
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
  },
  resultType: {
    fontSize: 12,
    fontWeight: '700',
    color: '#007AFF',
    marginBottom: 8,
    letterSpacing: 1,
  },
  resultValue: {
    fontSize: 16,
    color: '#1A1A1A',
    marginBottom: 16,
  },
  actionButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
  },
  actionButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

---

## 10. WEBVIEW & IN-APP BROWSER: WHEN TO USE WHICH

This is one of the most confused topics in React Native. There are two fundamentally different things people call "in-app browser," and using the wrong one will either break your OAuth flow or make your app feel janky.

### 10.1 The Two Options

| Feature | `react-native-webview` | `expo-web-browser` |
|---------|----------------------|-------------------|
| What it is | Embedded web content in your app | System browser (SafariViewController / Chrome Custom Tabs) |
| Use for | Rendering HTML, forms, rich content | OAuth flows, external links, PDFs |
| Control | Full (inject JS, intercept requests) | None (it's the system browser) |
| Cookies | Isolated per WebView instance | Shared with Safari/Chrome |
| Look & feel | Custom (your app wraps it) | System UI (done button, address bar) |
| OAuth | DO NOT use (security risk) | Use this (best practice) |

### 10.2 react-native-webview: Embedding Web Content

```bash
npx expo install react-native-webview
```

```tsx
import { WebView } from 'react-native-webview';
import { View, StyleSheet, ActivityIndicator, Text, Pressable } from 'react-native';
import { useState, useRef } from 'react';

function EmbeddedWebContent({ url }: { url: string }) {
  const webViewRef = useRef<WebView>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [canGoBack, setCanGoBack] = useState(false);

  return (
    <View style={styles.container}>
      {/* Navigation controls */}
      <View style={styles.toolbar}>
        <Pressable
          disabled={!canGoBack}
          onPress={() => webViewRef.current?.goBack()}
          style={[styles.navButton, !canGoBack && styles.navButtonDisabled]}
        >
          <Text style={styles.navButtonText}>Back</Text>
        </Pressable>
        
        <Pressable
          onPress={() => webViewRef.current?.reload()}
          style={styles.navButton}
        >
          <Text style={styles.navButtonText}>Reload</Text>
        </Pressable>
      </View>

      <WebView
        ref={webViewRef}
        source={{ uri: url }}
        style={styles.webview}
        
        // Loading state
        onLoadStart={() => setLoading(true)}
        onLoadEnd={() => setLoading(false)}
        renderLoading={() => (
          <ActivityIndicator size="large" style={styles.loader} />
        )}
        startInLoadingState
        
        // Error handling
        onError={(syntheticEvent) => {
          const { nativeEvent } = syntheticEvent;
          setError(nativeEvent.description);
        }}
        onHttpError={(syntheticEvent) => {
          const { nativeEvent } = syntheticEvent;
          console.warn('HTTP Error:', nativeEvent.statusCode);
        }}
        
        // Navigation state
        onNavigationStateChange={(navState) => {
          setCanGoBack(navState.canGoBack);
        }}
        
        // Security
        javaScriptEnabled
        domStorageEnabled
        
        // Prevent opening external links inside the WebView
        onShouldStartLoadWithRequest={(request) => {
          // Allow same-domain navigation
          if (request.url.startsWith(url)) return true;
          
          // Open external links in system browser
          Linking.openURL(request.url);
          return false;
        }}
      />

      {loading && (
        <View style={styles.loadingOverlay}>
          <ActivityIndicator size="large" color="#007AFF" />
        </View>
      )}

      {error && (
        <View style={styles.errorContainer}>
          <Text style={styles.errorText}>Failed to load page</Text>
          <Pressable
            style={styles.retryButton}
            onPress={() => {
              setError(null);
              webViewRef.current?.reload();
            }}
          >
            <Text style={styles.retryText}>Retry</Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  toolbar: {
    flexDirection: 'row',
    padding: 8,
    gap: 8,
    backgroundColor: '#F5F5F5',
    borderBottomWidth: 1,
    borderBottomColor: '#DDD',
  },
  navButton: { padding: 8 },
  navButtonDisabled: { opacity: 0.3 },
  navButtonText: { fontSize: 14, color: '#007AFF', fontWeight: '600' },
  webview: { flex: 1 },
  loader: {
    position: 'absolute',
    top: '50%',
    alignSelf: 'center',
  },
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'rgba(255,255,255,0.8)',
  },
  errorContainer: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: 'white',
  },
  errorText: { fontSize: 16, color: '#666', marginBottom: 16 },
  retryButton: { padding: 12, backgroundColor: '#007AFF', borderRadius: 8 },
  retryText: { color: 'white', fontWeight: '600' },
});
```

### 10.3 Communication Between WebView and React Native

This is the killer feature of `react-native-webview` — bidirectional communication:

```tsx
import { WebView, WebViewMessageEvent } from 'react-native-webview';
import { useRef, useCallback } from 'react';

// Types for messages between RN and WebView
type WebViewMessage = 
  | { type: 'FORM_SUBMITTED'; data: { name: string; email: string } }
  | { type: 'BUTTON_CLICKED'; data: { buttonId: string } }
  | { type: 'HEIGHT_CHANGED'; data: { height: number } };

type RNToWebMessage =
  | { type: 'SET_THEME'; data: { mode: 'light' | 'dark' } }
  | { type: 'SET_USER'; data: { name: string; token: string } };

function CommunicatingWebView() {
  const webViewRef = useRef<WebView>(null);

  // Receive messages FROM the WebView
  const handleMessage = useCallback((event: WebViewMessageEvent) => {
    try {
      const message: WebViewMessage = JSON.parse(event.nativeEvent.data);
      
      switch (message.type) {
        case 'FORM_SUBMITTED':
          console.log('Form data:', message.data);
          // Process form submission in RN
          break;
        case 'BUTTON_CLICKED':
          console.log('Button clicked:', message.data.buttonId);
          break;
        case 'HEIGHT_CHANGED':
          // Auto-size WebView based on content height
          break;
      }
    } catch (error) {
      console.error('Failed to parse WebView message:', error);
    }
  }, []);

  // Send messages TO the WebView
  const sendToWebView = useCallback((message: RNToWebMessage) => {
    const script = `
      window.dispatchEvent(
        new CustomEvent('rnMessage', { detail: ${JSON.stringify(message)} })
      );
      true; // Required for Android
    `;
    webViewRef.current?.injectJavaScript(script);
  }, []);

  // JavaScript to inject into the WebView for message passing
  const injectedJavaScript = `
    // Set up the message bridge
    window.sendToRN = function(type, data) {
      window.ReactNativeWebView.postMessage(JSON.stringify({ type, data }));
    };
    
    // Listen for messages from RN
    window.addEventListener('rnMessage', function(event) {
      var message = event.detail;
      // Handle message from RN
      if (message.type === 'SET_THEME') {
        document.body.className = message.data.mode;
      }
    });
    
    // Notify RN when content height changes (for auto-sizing)
    var observer = new ResizeObserver(function(entries) {
      var height = document.body.scrollHeight;
      window.sendToRN('HEIGHT_CHANGED', { height: height });
    });
    observer.observe(document.body);
    
    // Intercept form submissions
    document.addEventListener('submit', function(e) {
      e.preventDefault();
      var formData = new FormData(e.target);
      var data = Object.fromEntries(formData.entries());
      window.sendToRN('FORM_SUBMITTED', data);
    });
    
    true; // Required
  `;

  return (
    <WebView
      ref={webViewRef}
      source={{ uri: 'https://your-web-content.com' }}
      onMessage={handleMessage}
      injectedJavaScript={injectedJavaScript}
      javaScriptEnabled
    />
  );
}
```

### 10.4 expo-web-browser: System Browser for OAuth and External Links

```bash
npx expo install expo-web-browser
```

```tsx
import * as WebBrowser from 'expo-web-browser';
import { makeRedirectUri, useAuthRequest } from 'expo-auth-session';

// Simple link opening (SafariViewController / Chrome Custom Tabs)
async function openExternalLink(url: string) {
  const result = await WebBrowser.openBrowserAsync(url, {
    // iOS options
    presentationStyle: WebBrowser.WebBrowserPresentationStyle.FULL_SCREEN,
    controlsColor: '#007AFF',
    dismissButtonStyle: 'close',
    
    // Shared options
    toolbarColor: '#FFFFFF',        // Top bar color
    enableBarCollapsing: true,      // Allow collapsing on scroll
    showTitle: true,                // Show page title
  });

  if (result.type === 'cancel') {
    console.log('User closed the browser');
  }
}

// OAuth flow with expo-web-browser
// This is the CORRECT way to handle OAuth in a mobile app
WebBrowser.maybeCompleteAuthSession(); // Required for web

function OAuthLogin() {
  const [request, response, promptAsync] = useAuthRequest(
    {
      clientId: 'YOUR_CLIENT_ID',
      scopes: ['openid', 'profile', 'email'],
      redirectUri: makeRedirectUri({
        scheme: 'myapp',
        path: 'auth/callback',
      }),
    },
    {
      authorizationEndpoint: 'https://accounts.google.com/o/oauth2/v2/auth',
      tokenEndpoint: 'https://oauth2.googleapis.com/token',
    }
  );

  useEffect(() => {
    if (response?.type === 'success') {
      const { code } = response.params;
      // Exchange code for token on your backend
      exchangeCodeForToken(code);
    }
  }, [response]);

  return (
    <Pressable
      disabled={!request}
      onPress={() => promptAsync()}
      style={styles.oauthButton}
    >
      <Text style={styles.oauthText}>Sign in with Google</Text>
    </Pressable>
  );
}
```

**Critical rule:** Never use `react-native-webview` for OAuth. The WebView shares no cookies with the system browser, so single sign-on won't work. Users will have to re-enter credentials even if they're already logged into Google/Apple in Safari. Always use `expo-web-browser` for OAuth flows.

---

## 11. DEVICE INFO: SCREEN DIMENSIONS, PLATFORM DETECTION, VERSIONS

### 11.1 expo-device

```bash
npx expo install expo-device
```

```tsx
import * as Device from 'expo-device';
import { Platform, Dimensions } from 'react-native';

// Device information
const deviceInfo = {
  // Device model
  brand: Device.brand,                    // "Apple" | "Samsung" | "Google" | etc.
  modelName: Device.modelName,            // "iPhone 15 Pro" | "Pixel 8" | etc.
  modelId: Device.modelId,               // "iPhone16,1" | etc.
  designName: Device.designName,          // Android only: "raven" etc.
  
  // OS information
  osName: Device.osName,                  // "iOS" | "Android" | "iPadOS"
  osVersion: Device.osVersion,            // "17.4" | "14.0"
  osBuildId: Device.osBuildId,           // "21E219" etc.
  
  // Device type
  deviceType: Device.deviceType,          // PHONE | TABLET | DESKTOP | TV | UNKNOWN
  isDevice: Device.isDevice,              // true if physical device, false if simulator
  
  // Manufacturer
  manufacturer: Device.manufacturer,       // "Apple" | "Samsung" etc.
  
  // Memory (Android only, returns null on iOS)
  totalMemory: Device.totalMemory,         // Total RAM in bytes
  
  // Uptime
  // Device.getUptimeAsync() returns device uptime in ms
};

// Useful utility functions
function isTablet(): boolean {
  return Device.deviceType === Device.DeviceType.TABLET;
}

function isPhysicalDevice(): boolean {
  return Device.isDevice;
}

function getDeviceLabel(): string {
  return `${Device.brand} ${Device.modelName} (${Device.osName} ${Device.osVersion})`;
}
```

### 11.2 expo-constants

```tsx
import Constants from 'expo-constants';

const appInfo = {
  // App version (from app.json/app.config.js)
  appVersion: Constants.expoConfig?.version,           // "1.2.3"
  
  // Build number
  buildNumber: Platform.select({
    ios: Constants.expoConfig?.ios?.buildNumber,        // "42"
    android: Constants.expoConfig?.android?.versionCode?.toString(), // "42"
  }),
  
  // Expo SDK version
  sdkVersion: Constants.expoConfig?.sdkVersion,        // "52.0.0"
  
  // Runtime version (for OTA updates)
  runtimeVersion: Constants.expoConfig?.runtimeVersion, // "1.0.0"
  
  // Unique installation ID
  installationId: Constants.installationId,              // UUID per install
  
  // Session ID (changes each app launch)
  sessionId: Constants.sessionId,
  
  // Status bar height
  statusBarHeight: Constants.statusBarHeight,            // Number in dp/pt
  
  // Is running in Expo Go?
  isExpoGo: Constants.appOwnership === 'expo',
};
```

### 11.3 Screen Dimensions and Safe Areas

```tsx
import { Dimensions, Platform, StatusBar } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { useWindowDimensions } from 'react-native';

// Static dimensions (don't update on rotation)
const { width: SCREEN_WIDTH, height: SCREEN_HEIGHT } = Dimensions.get('screen');
const { width: WINDOW_WIDTH, height: WINDOW_HEIGHT } = Dimensions.get('window');
// On Android, "window" excludes the status bar; "screen" includes it

// Dynamic dimensions (update on rotation, split view, etc.)
function ResponsiveComponent() {
  const { width, height } = useWindowDimensions();
  const insets = useSafeAreaInsets();

  const isLandscape = width > height;
  const isTablet = width >= 768;
  const isSmallPhone = width < 375;

  // Usable area (minus safe areas)
  const usableWidth = width - insets.left - insets.right;
  const usableHeight = height - insets.top - insets.bottom;

  return (
    <View style={{
      paddingTop: insets.top,
      paddingBottom: insets.bottom,
      paddingLeft: insets.left,
      paddingRight: insets.right,
    }}>
      {/* Content */}
    </View>
  );
}

// Platform detection utilities
const platform = {
  isIOS: Platform.OS === 'ios',
  isAndroid: Platform.OS === 'android',
  isWeb: Platform.OS === 'web',
  
  // iOS version check
  isIOS17Plus: Platform.OS === 'ios' && parseInt(Platform.Version as string, 10) >= 17,
  
  // Android API level check
  isAndroid13Plus: Platform.OS === 'android' && (Platform.Version as number) >= 33,
  
  // Dynamic Island detection (rough heuristic)
  hasDynamicIsland: Platform.OS === 'ios' && 
    StatusBar.currentHeight !== undefined &&
    StatusBar.currentHeight > 50,
};
```

---

## 12. SENSORS: ACCELEROMETER, GYROSCOPE, BAROMETER, SHAKE DETECTION

### 12.1 Setting Up expo-sensors

```bash
npx expo install expo-sensors
```

### 12.2 Accelerometer

```tsx
import { Accelerometer, AccelerometerMeasurement } from 'expo-sensors';
import { useState, useEffect, useRef } from 'react';
import { View, Text, StyleSheet } from 'react-native';

function useAccelerometer(updateInterval: number = 100) {
  const [data, setData] = useState<AccelerometerMeasurement>({ x: 0, y: 0, z: 0 });
  const [isAvailable, setIsAvailable] = useState(false);

  useEffect(() => {
    Accelerometer.isAvailableAsync().then(setIsAvailable);
  }, []);

  useEffect(() => {
    if (!isAvailable) return;

    Accelerometer.setUpdateInterval(updateInterval);
    
    const subscription = Accelerometer.addListener(setData);

    return () => subscription.remove();
  }, [isAvailable, updateInterval]);

  return { data, isAvailable };
}

// Level / spirit level component
function SpiritLevel() {
  const { data } = useAccelerometer(50);
  
  // x: tilt left/right (-1 to 1)
  // y: tilt forward/backward (-1 to 1)
  // z: gravity direction (-1 to 1; ~1 when flat)
  
  const bubbleX = data.x * 100; // Convert to pixel offset
  const bubbleY = data.y * 100;
  const isLevel = Math.abs(data.x) < 0.02 && Math.abs(data.y) < 0.02;

  return (
    <View style={styles.levelContainer}>
      <View style={styles.levelCircle}>
        <View
          style={[
            styles.bubble,
            {
              transform: [
                { translateX: bubbleX },
                { translateY: -bubbleY }, // Invert Y for natural feel
              ],
            },
            isLevel && styles.bubbleLevel,
          ]}
        />
        {/* Crosshair */}
        <View style={styles.crosshairH} />
        <View style={styles.crosshairV} />
      </View>
      <Text style={styles.levelText}>
        {isLevel ? 'Level!' : `X: ${data.x.toFixed(2)} Y: ${data.y.toFixed(2)}`}
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  levelContainer: { alignItems: 'center', padding: 24 },
  levelCircle: {
    width: 200,
    height: 200,
    borderRadius: 100,
    borderWidth: 2,
    borderColor: '#DDD',
    justifyContent: 'center',
    alignItems: 'center',
    overflow: 'hidden',
  },
  bubble: {
    width: 24,
    height: 24,
    borderRadius: 12,
    backgroundColor: '#FF3B30',
  },
  bubbleLevel: {
    backgroundColor: '#34C759',
  },
  crosshairH: {
    position: 'absolute',
    width: '100%',
    height: 1,
    backgroundColor: '#DDD',
  },
  crosshairV: {
    position: 'absolute',
    width: 1,
    height: '100%',
    backgroundColor: '#DDD',
  },
  levelText: {
    marginTop: 16,
    fontSize: 16,
    fontWeight: '600',
    color: '#1A1A1A',
  },
});
```

### 12.3 Shake Detection

Detecting device shake is useful for "shake to undo," "shake to report bug," or debug menus:

```tsx
import { Accelerometer, AccelerometerMeasurement } from 'expo-sensors';
import { useEffect, useRef, useCallback } from 'react';

const SHAKE_THRESHOLD = 2.5; // Acceleration threshold (in Gs)
const SHAKE_TIMEOUT = 500;   // Minimum time between shakes
const SHAKE_COUNT = 2;       // Number of shakes required

function useShakeDetection(onShake: () => void) {
  const lastShakeTime = useRef(0);
  const shakeCount = useRef(0);
  const lastAcceleration = useRef<AccelerometerMeasurement>({ x: 0, y: 0, z: 0 });

  const handleAccelerometer = useCallback(
    (data: AccelerometerMeasurement) => {
      const { x, y, z } = data;
      const { x: lx, y: ly, z: lz } = lastAcceleration.current;
      
      // Calculate acceleration delta
      const deltaX = Math.abs(x - lx);
      const deltaY = Math.abs(y - ly);
      const deltaZ = Math.abs(z - lz);
      
      const acceleration = Math.sqrt(deltaX ** 2 + deltaY ** 2 + deltaZ ** 2);
      
      lastAcceleration.current = data;
      
      if (acceleration > SHAKE_THRESHOLD) {
        const now = Date.now();
        
        if (now - lastShakeTime.current > SHAKE_TIMEOUT) {
          shakeCount.current = 1;
        } else {
          shakeCount.current += 1;
        }
        
        lastShakeTime.current = now;
        
        if (shakeCount.current >= SHAKE_COUNT) {
          shakeCount.current = 0;
          onShake();
        }
      }
    },
    [onShake]
  );

  useEffect(() => {
    Accelerometer.setUpdateInterval(100);
    const subscription = Accelerometer.addListener(handleAccelerometer);
    return () => subscription.remove();
  }, [handleAccelerometer]);
}

// Usage
function DebugScreen() {
  useShakeDetection(() => {
    // Show debug menu on shake
    console.log('Shake detected!');
    // Show your debug/feedback UI
  });

  return <View>{/* ... */}</View>;
}
```

### 12.4 Gyroscope and Other Sensors

```tsx
import { Gyroscope, Barometer, Magnetometer, Pedometer } from 'expo-sensors';
import { useState, useEffect } from 'react';

// Gyroscope — rotation rate
function useGyroscope(updateInterval = 100) {
  const [data, setData] = useState({ x: 0, y: 0, z: 0 });

  useEffect(() => {
    Gyroscope.setUpdateInterval(updateInterval);
    const sub = Gyroscope.addListener(setData);
    return () => sub.remove();
  }, [updateInterval]);

  return data; // x, y, z = rotation rate in rad/s
}

// Barometer — atmospheric pressure (altitude estimation)
function useBarometer(updateInterval = 1000) {
  const [data, setData] = useState<{ pressure: number; relativeAltitude?: number } | null>(null);

  useEffect(() => {
    Barometer.setUpdateInterval(updateInterval);
    const sub = Barometer.addListener(setData);
    return () => sub.remove();
  }, [updateInterval]);

  return data; // pressure in hPa, relativeAltitude in meters (iOS only)
}

// Magnetometer — compass heading
function useCompass(updateInterval = 100) {
  const [heading, setHeading] = useState(0);

  useEffect(() => {
    Magnetometer.setUpdateInterval(updateInterval);
    const sub = Magnetometer.addListener(data => {
      // Calculate compass heading from magnetometer data
      const { x, y } = data;
      let angle = Math.atan2(y, x) * (180 / Math.PI);
      if (angle < 0) angle += 360;
      setHeading(Math.round(angle));
    });
    return () => sub.remove();
  }, [updateInterval]);

  return heading; // 0-360 degrees, 0 = North
}

// Pedometer — step counting
function useStepCount() {
  const [steps, setSteps] = useState(0);
  const [isAvailable, setIsAvailable] = useState(false);

  useEffect(() => {
    Pedometer.isAvailableAsync().then(setIsAvailable);
  }, []);

  useEffect(() => {
    if (!isAvailable) return;

    // Watch live steps
    const subscription = Pedometer.watchStepCount(result => {
      setSteps(result.steps);
    });

    return () => subscription.remove();
  }, [isAvailable]);

  return { steps, isAvailable };
}

// Get historical step data
async function getStepHistory(days: number) {
  const end = new Date();
  const start = new Date();
  start.setDate(start.getDate() - days);

  const isAvailable = await Pedometer.isAvailableAsync();
  if (!isAvailable) return null;

  const result = await Pedometer.getStepCountAsync(start, end);
  return result.steps;
}
```

---

## 13. KEYBOARD HANDLING: THE SILENT UX KILLER

I cannot overstate how much pain keyboard handling causes in React Native apps. It's the thing that makes users think your app is buggy, even if everything else works perfectly. The keyboard slides up, covers the input field, and the user is typing into the void.

### 13.1 The Problem

When a text input gets focus, the system keyboard slides up from the bottom. If the input is in the lower half of the screen, the keyboard covers it. React Native's built-in `KeyboardAvoidingView` tries to solve this but has platform-specific quirks that make it unreliable.

### 13.2 KeyboardAvoidingView (Built-in)

```tsx
import { 
  KeyboardAvoidingView, 
  Platform, 
  ScrollView, 
  TextInput, 
  View, 
  StyleSheet 
} from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function FormWithKeyboardAvoidance() {
  const insets = useSafeAreaInsets();

  return (
    <KeyboardAvoidingView
      style={styles.container}
      // The magic behavior prop:
      // 'padding' — adds padding at the bottom (works on iOS)
      // 'height' — reduces the height (use with caution)
      // 'position' — adjusts position (least reliable)
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      
      // keyboardVerticalOffset: account for any fixed headers
      keyboardVerticalOffset={Platform.select({
        ios: insets.top + 44, // Status bar + navigation bar
        android: 0,
      })}
    >
      <ScrollView
        contentContainerStyle={styles.scrollContent}
        keyboardShouldPersistTaps="handled" // Allow tapping buttons while keyboard is open
        keyboardDismissMode="interactive"    // iOS: drag to dismiss
      >
        <TextInput style={styles.input} placeholder="Name" />
        <TextInput style={styles.input} placeholder="Email" keyboardType="email-address" />
        <TextInput style={styles.input} placeholder="Phone" keyboardType="phone-pad" />
        <TextInput
          style={[styles.input, styles.multilineInput]}
          placeholder="Message"
          multiline
          numberOfLines={4}
        />
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  scrollContent: { padding: 16, gap: 12 },
  input: {
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
  },
  multilineInput: {
    height: 120,
    textAlignVertical: 'top',
  },
});
```

**The problem with KeyboardAvoidingView:** It works okay for simple forms. It falls apart when you have:
- Tab bars
- Bottom sheets
- Complex nested layouts
- Mixed scrollable/non-scrollable content
- Android + iOS both needing to work

### 13.3 react-native-keyboard-controller (The Better Solution)

This is the library I recommend for production apps. It uses native keyboard APIs directly and handles all the edge cases:

```bash
npx expo install react-native-keyboard-controller
```

```tsx
import { KeyboardProvider, KeyboardAwareScrollView } from 'react-native-keyboard-controller';
import { TextInput, View, StyleSheet, Text, Pressable } from 'react-native';

// Wrap your app with KeyboardProvider (in your root layout)
function RootLayout() {
  return (
    <KeyboardProvider>
      {/* Your app */}
    </KeyboardProvider>
  );
}

// Use KeyboardAwareScrollView in your forms
function BetterForm() {
  return (
    <KeyboardAwareScrollView
      style={styles.container}
      contentContainerStyle={styles.content}
      bottomOffset={20} // Extra padding below focused input
    >
      <Text style={styles.label}>Name</Text>
      <TextInput style={styles.input} placeholder="John Doe" />
      
      <Text style={styles.label}>Email</Text>
      <TextInput
        style={styles.input}
        placeholder="john@example.com"
        keyboardType="email-address"
        autoCapitalize="none"
      />
      
      <Text style={styles.label}>Password</Text>
      <TextInput
        style={styles.input}
        placeholder="********"
        secureTextEntry
      />
      
      <Text style={styles.label}>Bio</Text>
      <TextInput
        style={[styles.input, styles.multiline]}
        placeholder="Tell us about yourself..."
        multiline
        numberOfLines={6}
      />
      
      {/* This button will be properly visible above the keyboard */}
      <Pressable style={styles.submitButton}>
        <Text style={styles.submitText}>Create Account</Text>
      </Pressable>
    </KeyboardAwareScrollView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1 },
  content: { padding: 16, gap: 8 },
  label: { fontSize: 14, fontWeight: '600', color: '#1A1A1A', marginTop: 8 },
  input: {
    borderWidth: 1,
    borderColor: '#DDD',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    backgroundColor: 'white',
  },
  multiline: { height: 120, textAlignVertical: 'top' },
  submitButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 16,
  },
  submitText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

### 13.4 Keyboard Events and Animated Responses

```tsx
import { 
  useKeyboardAnimation, 
  useKeyboardHandler 
} from 'react-native-keyboard-controller';
import Animated, { useAnimatedStyle } from 'react-native-reanimated';

// Animated bottom bar that moves with the keyboard
function AnimatedBottomBar({ children }: { children: React.ReactNode }) {
  const { height, progress } = useKeyboardAnimation();

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: -height.value }],
  }));

  return (
    <Animated.View style={[styles.bottomBar, animatedStyle]}>
      {children}
    </Animated.View>
  );
}

// Listen for keyboard events
function KeyboardAwareComponent() {
  useKeyboardHandler({
    onStart: (e) => {
      'worklet';
      console.log('Keyboard starting to show, target height:', e.height);
    },
    onMove: (e) => {
      'worklet';
      // e.height = current keyboard height (animated)
      // e.progress = 0 to 1 animation progress
    },
    onEnd: (e) => {
      'worklet';
      console.log('Keyboard animation complete');
    },
  });

  return <View>{/* ... */}</View>;
}
```

### 13.5 Dismissing the Keyboard

```tsx
import { Keyboard, Pressable, View } from 'react-native';

// Dismiss on tap outside
function DismissKeyboardView({ children }: { children: React.ReactNode }) {
  return (
    <Pressable
      style={{ flex: 1 }}
      onPress={Keyboard.dismiss}
    >
      {children}
    </Pressable>
  );
}

// Dismiss on scroll
// Just use keyboardDismissMode on ScrollView:
<ScrollView keyboardDismissMode="on-drag">
  {/* Content */}
</ScrollView>

// Dismiss programmatically
function handleSubmit() {
  Keyboard.dismiss();
  // ... submit form
}

// Prevent keyboard from dismissing on tap
// (useful for chat input that should stay focused)
<ScrollView keyboardShouldPersistTaps="always">
  {/* Content */}
</ScrollView>
```

### 13.6 Keyboard Handling Checklist

- [ ] All forms use `KeyboardAwareScrollView` or `KeyboardAvoidingView`
- [ ] `keyboardShouldPersistTaps="handled"` on ScrollViews with buttons
- [ ] Submit buttons are visible when keyboard is open
- [ ] `keyboardType` is set correctly (email, phone, numeric, etc.)
- [ ] `returnKeyType` guides the user (next, done, send, search)
- [ ] `autoCapitalize` is set appropriately (none for email, words for names)
- [ ] `secureTextEntry` for password fields
- [ ] Tab order works logically (use `ref` and `focus()` for "Next" button)
- [ ] Keyboard dismisses when tapping outside inputs
- [ ] Tested on both iOS and Android physical devices

---

## 14. MEDIA: AUDIO, VIDEO, AND IMAGE PICKER

### 14.1 Image Picker (Camera Roll & Capture)

```bash
npx expo install expo-image-picker
```

```tsx
import * as ImagePicker from 'expo-image-picker';
import { View, Pressable, Text, StyleSheet } from 'react-native';
import { Image } from 'expo-image';
import { useState } from 'react';

function ImagePickerExample() {
  const [image, setImage] = useState<string | null>(null);

  // Pick from camera roll
  const pickImage = async () => {
    // Request permission
    const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
    if (status !== 'granted') {
      alert('Sorry, we need camera roll access to pick photos.');
      return;
    }

    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ['images'],
      allowsEditing: true,           // Show crop UI
      aspect: [1, 1],                // Square crop
      quality: 0.8,                  // Compress slightly
      exif: false,                   // Don't include EXIF data
      base64: false,                 // Don't include base64
      // allowsMultipleSelection: true, // Allow picking multiple images
    });

    if (!result.canceled && result.assets.length > 0) {
      setImage(result.assets[0].uri);
    }
  };

  // Take a photo with the camera
  const takePhoto = async () => {
    const { status } = await ImagePicker.requestCameraPermissionsAsync();
    if (status !== 'granted') {
      alert('Sorry, we need camera access to take photos.');
      return;
    }

    const result = await ImagePicker.launchCameraAsync({
      allowsEditing: true,
      aspect: [4, 3],
      quality: 0.8,
    });

    if (!result.canceled && result.assets.length > 0) {
      setImage(result.assets[0].uri);
    }
  };

  return (
    <View style={styles.container}>
      {image ? (
        <Image
          source={{ uri: image }}
          style={styles.preview}
          contentFit="cover"
        />
      ) : (
        <View style={styles.placeholder}>
          <Text style={styles.placeholderText}>No image selected</Text>
        </View>
      )}

      <View style={styles.buttonRow}>
        <Pressable style={styles.button} onPress={pickImage}>
          <Text style={styles.buttonText}>Pick from Library</Text>
        </Pressable>
        <Pressable style={styles.button} onPress={takePhoto}>
          <Text style={styles.buttonText}>Take Photo</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 16, gap: 16 },
  preview: { width: '100%', height: 300, borderRadius: 12 },
  placeholder: {
    width: '100%',
    height: 300,
    borderRadius: 12,
    backgroundColor: '#F5F5F5',
    justifyContent: 'center',
    alignItems: 'center',
  },
  placeholderText: { color: '#999', fontSize: 16 },
  buttonRow: { flexDirection: 'row', gap: 12 },
  button: {
    flex: 1,
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonText: { color: 'white', fontSize: 14, fontWeight: '600' },
});
```

### 14.2 Audio Playback with expo-av

```tsx
import { Audio, AVPlaybackStatus } from 'expo-av';
import { useState, useEffect, useRef } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

function useAudioPlayer(uri: string) {
  const sound = useRef<Audio.Sound | null>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [duration, setDuration] = useState(0);
  const [position, setPosition] = useState(0);

  useEffect(() => {
    return () => {
      // Cleanup on unmount
      sound.current?.unloadAsync();
    };
  }, []);

  const load = async () => {
    const { sound: newSound } = await Audio.Sound.createAsync(
      { uri },
      { shouldPlay: false },
      onPlaybackStatusUpdate
    );
    sound.current = newSound;
  };

  const onPlaybackStatusUpdate = (status: AVPlaybackStatus) => {
    if (!status.isLoaded) return;
    
    setIsPlaying(status.isPlaying);
    setDuration(status.durationMillis ?? 0);
    setPosition(status.positionMillis);
    
    if (status.didJustFinish) {
      setIsPlaying(false);
      setPosition(0);
    }
  };

  const play = async () => {
    if (!sound.current) await load();
    await sound.current?.playAsync();
  };

  const pause = async () => {
    await sound.current?.pauseAsync();
  };

  const seek = async (positionMs: number) => {
    await sound.current?.setPositionAsync(positionMs);
  };

  const toggle = async () => {
    if (isPlaying) {
      await pause();
    } else {
      await play();
    }
  };

  return {
    isPlaying,
    duration,
    position,
    progress: duration > 0 ? position / duration : 0,
    play,
    pause,
    seek,
    toggle,
  };
}

function AudioPlayer({ uri, title }: { uri: string; title: string }) {
  const { isPlaying, duration, position, progress, toggle, seek } = useAudioPlayer(uri);

  const formatTime = (ms: number) => {
    const seconds = Math.floor(ms / 1000);
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <View style={styles.playerContainer}>
      <Pressable style={styles.playButton} onPress={toggle}>
        <Text style={styles.playButtonText}>
          {isPlaying ? 'Pause' : 'Play'}
        </Text>
      </Pressable>
      
      <View style={styles.playerInfo}>
        <Text style={styles.playerTitle} numberOfLines={1}>{title}</Text>
        <View style={styles.progressBar}>
          <View style={[styles.progressFill, { width: `${progress * 100}%` }]} />
        </View>
        <View style={styles.timeRow}>
          <Text style={styles.timeText}>{formatTime(position)}</Text>
          <Text style={styles.timeText}>{formatTime(duration)}</Text>
        </View>
      </View>
    </View>
  );
}

// Audio recording setup
async function configureAudioSession() {
  await Audio.setAudioModeAsync({
    allowsRecordingIOS: true,
    playsInSilentModeIOS: true,
    staysActiveInBackground: false,
    // For playback (when not recording):
    // allowsRecordingIOS: false,
    // interruptionModeIOS: Audio.INTERRUPTION_MODE_IOS_DUCK_OTHERS,
    // interruptionModeAndroid: Audio.INTERRUPTION_MODE_ANDROID_DUCK_OTHERS,
  });
}

const styles = StyleSheet.create({
  playerContainer: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
    backgroundColor: 'white',
    borderRadius: 12,
    gap: 12,
  },
  playButton: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: '#007AFF',
    justifyContent: 'center',
    alignItems: 'center',
  },
  playButtonText: { color: 'white', fontSize: 12, fontWeight: '700' },
  playerInfo: { flex: 1 },
  playerTitle: { fontSize: 14, fontWeight: '600', color: '#1A1A1A', marginBottom: 8 },
  progressBar: {
    height: 4,
    backgroundColor: '#E5E5E5',
    borderRadius: 2,
    overflow: 'hidden',
  },
  progressFill: {
    height: '100%',
    backgroundColor: '#007AFF',
    borderRadius: 2,
  },
  timeRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginTop: 4,
  },
  timeText: { fontSize: 12, color: '#999' },
});
```

### 14.3 Audio Recording

```tsx
import { Audio } from 'expo-av';
import { useState, useRef } from 'react';

function useAudioRecorder() {
  const recording = useRef<Audio.Recording | null>(null);
  const [isRecording, setIsRecording] = useState(false);
  const [recordingUri, setRecordingUri] = useState<string | null>(null);
  const [duration, setDuration] = useState(0);

  const startRecording = async () => {
    try {
      // Request permission
      const { status } = await Audio.requestPermissionsAsync();
      if (status !== 'granted') {
        throw new Error('Microphone permission denied');
      }

      // Configure audio mode for recording
      await Audio.setAudioModeAsync({
        allowsRecordingIOS: true,
        playsInSilentModeIOS: true,
      });

      // Start recording
      const { recording: newRecording } = await Audio.Recording.createAsync(
        Audio.RecordingOptionsPresets.HIGH_QUALITY,
        (status) => {
          if (status.isRecording) {
            setDuration(status.durationMillis);
          }
        },
        100 // Update interval in ms
      );

      recording.current = newRecording;
      setIsRecording(true);
      setRecordingUri(null);
    } catch (error) {
      console.error('Failed to start recording:', error);
    }
  };

  const stopRecording = async () => {
    if (!recording.current) return;

    try {
      await recording.current.stopAndUnloadAsync();
      
      // Reset audio mode
      await Audio.setAudioModeAsync({
        allowsRecordingIOS: false,
      });

      const uri = recording.current.getURI();
      setRecordingUri(uri);
      setIsRecording(false);
      recording.current = null;

      return uri;
    } catch (error) {
      console.error('Failed to stop recording:', error);
    }
  };

  return {
    isRecording,
    duration,
    recordingUri,
    startRecording,
    stopRecording,
  };
}
```

### 14.4 Video Playback with expo-video

```bash
npx expo install expo-video
```

```tsx
import { useVideoPlayer, VideoView } from 'expo-video';
import { View, Pressable, Text, StyleSheet } from 'react-native';
import { useEvent } from 'expo';
import { useState } from 'react';

function VideoPlayerScreen({ videoUri }: { videoUri: string }) {
  const player = useVideoPlayer(videoUri, (player) => {
    player.loop = false;
    player.playbackRate = 1.0;
    player.volume = 1.0;
  });

  const { isPlaying } = useEvent(player, 'playingChange', {
    isPlaying: player.playing,
  });

  return (
    <View style={styles.container}>
      <VideoView
        style={styles.video}
        player={player}
        allowsFullscreen
        allowsPictureInPicture
        contentFit="contain"
        nativeControls  // Use platform native controls
      />
      
      {/* Custom controls (if nativeControls is false) */}
      <View style={styles.controls}>
        <Pressable
          style={styles.controlButton}
          onPress={() => {
            if (isPlaying) {
              player.pause();
            } else {
              player.play();
            }
          }}
        >
          <Text style={styles.controlText}>
            {isPlaying ? 'Pause' : 'Play'}
          </Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  video: { flex: 1 },
  controls: {
    position: 'absolute',
    bottom: 40,
    left: 0,
    right: 0,
    alignItems: 'center',
  },
  controlButton: {
    padding: 16,
    backgroundColor: 'rgba(0,0,0,0.5)',
    borderRadius: 8,
  },
  controlText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

---

## 15. HAPTICS, VIBRATION PATTERNS: PLATFORM DIFFERENCES

### 15.1 iOS vs Android Haptic Support

The haptic experience is radically different across platforms:

**iOS (iPhone 8+):**
- Taptic Engine provides precise, nuanced feedback
- Three intensity levels for impact (Light, Medium, Heavy)
- Two additional iOS-only styles (Soft, Rigid)
- Notification feedback (Success, Warning, Error) each feel distinct
- Selection feedback provides a subtle tick

**Android:**
- Haptic quality varies enormously by device
- Pixel phones (6+) have excellent haptics comparable to iPhone
- Many Android phones only have a basic vibration motor
- Some cheap devices have no haptic hardware at all
- `expo-haptics` falls back to basic vibration on devices without haptic hardware

### 15.2 Vibration Patterns (Android-specific)

For more control on Android, you can use the Vibration API directly:

```tsx
import { Vibration, Platform } from 'react-native';

// Simple vibration (works on both platforms)
Vibration.vibrate();

// Duration in ms (Android only — iOS ignores the duration)
Vibration.vibrate(500); // Vibrate for 500ms

// Pattern: [wait, vibrate, wait, vibrate, ...]
// Android only — iOS only does one short vibrate
Vibration.vibrate([0, 200, 100, 200]); // buzz-pause-buzz

// Repeat pattern until cancelled
Vibration.vibrate([0, 200, 100, 200, 100, 400], true);

// Cancel vibration
Vibration.cancel();

// Cross-platform vibration utility
const vibrate = {
  short: () => {
    if (Platform.OS === 'ios') {
      // Use haptics on iOS for a better feel
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    } else {
      Vibration.vibrate(50);
    }
  },
  
  medium: () => {
    if (Platform.OS === 'ios') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    } else {
      Vibration.vibrate(100);
    }
  },
  
  heavy: () => {
    if (Platform.OS === 'ios') {
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Heavy);
    } else {
      Vibration.vibrate(200);
    }
  },
  
  pattern: (pattern: number[]) => {
    if (Platform.OS === 'android') {
      Vibration.vibrate(pattern);
    } else {
      // iOS: Fall back to a single haptic feedback
      Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
    }
  },
  
  cancel: () => {
    Vibration.cancel();
  },
};
```

### 15.3 Haptic Design Guidelines

```
Good Haptic Usage:
  Button press          → Light impact
  Toggle switch         → Medium impact
  Delete confirmation   → Heavy impact
  Success (payment)     → Success notification
  Error (validation)    → Error notification
  Picker scroll         → Selection tick
  Long press activate   → Medium impact
  Swipe action trigger  → Medium impact
  Pull-to-refresh       → Light impact at threshold

Bad Haptic Usage:
  Every list item render → Too frequent
  Scroll events          → Way too frequent
  Keyboard typing        → System handles this
  Background events      → User can't feel it
  Animation frames       → Overwhelming
  Network request start  → Meaningless
```

---

## 16. PUTTING IT ALL TOGETHER: A DEVICE-API-RICH SCREEN

Let's build a practical screen that combines multiple device APIs — a "New Post" screen that uses the camera, location, haptics, and the share sheet:

```tsx
import { useState, useCallback } from 'react';
import { 
  View, Text, TextInput, Pressable, StyleSheet, 
  Alert, ScrollView, Platform 
} from 'react-native';
import { Image } from 'expo-image';
import * as ImagePicker from 'expo-image-picker';
import * as Location from 'expo-location';
import * as Sharing from 'expo-sharing';
import { haptic } from './haptics';

interface PostDraft {
  text: string;
  images: string[];
  location: { name: string; lat: number; lng: number } | null;
}

function NewPostScreen() {
  const [draft, setDraft] = useState<PostDraft>({
    text: '',
    images: [],
    location: null,
  });
  const [isSubmitting, setIsSubmitting] = useState(false);

  // Add photo from camera or library
  const addPhoto = useCallback(async (source: 'camera' | 'library') => {
    haptic.tap();

    const permissionMethod = source === 'camera'
      ? ImagePicker.requestCameraPermissionsAsync
      : ImagePicker.requestMediaLibraryPermissionsAsync;

    const { status } = await permissionMethod();
    if (status !== 'granted') {
      Alert.alert('Permission needed', `Please allow ${source} access in Settings.`);
      return;
    }

    const launcher = source === 'camera'
      ? ImagePicker.launchCameraAsync
      : ImagePicker.launchImageLibraryAsync;

    const result = await launcher({
      mediaTypes: ['images'],
      quality: 0.8,
      allowsMultipleSelection: source === 'library',
    });

    if (!result.canceled) {
      const newUris = result.assets.map(a => a.uri);
      setDraft(prev => ({
        ...prev,
        images: [...prev.images, ...newUris].slice(0, 4), // Max 4 images
      }));
      haptic.success();
    }
  }, []);

  // Add current location
  const addLocation = useCallback(async () => {
    haptic.tap();

    const { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== 'granted') {
      Alert.alert('Permission needed', 'Please allow location access in Settings.');
      return;
    }

    const location = await Location.getCurrentPositionAsync({
      accuracy: Location.Accuracy.Balanced,
    });

    const [address] = await Location.reverseGeocodeAsync({
      latitude: location.coords.latitude,
      longitude: location.coords.longitude,
    });

    const locationName = [address?.city, address?.region]
      .filter(Boolean)
      .join(', ') || 'Unknown location';

    setDraft(prev => ({
      ...prev,
      location: {
        name: locationName,
        lat: location.coords.latitude,
        lng: location.coords.longitude,
      },
    }));
    haptic.success();
  }, []);

  // Submit post
  const handleSubmit = useCallback(async () => {
    if (!draft.text.trim() && draft.images.length === 0) {
      haptic.error();
      Alert.alert('Empty post', 'Please add some text or images.');
      return;
    }

    setIsSubmitting(true);
    haptic.tap();

    try {
      // Upload images, create post, etc.
      await new Promise(resolve => setTimeout(resolve, 1500)); // Simulate API call
      
      haptic.success();
      Alert.alert('Posted!', 'Your post has been published.');
      
      // Reset draft
      setDraft({ text: '', images: [], location: null });
    } catch (error) {
      haptic.error();
      Alert.alert('Error', 'Failed to publish post. Please try again.');
    } finally {
      setIsSubmitting(false);
    }
  }, [draft]);

  return (
    <ScrollView style={styles.container} keyboardShouldPersistTaps="handled">
      {/* Text input */}
      <TextInput
        style={styles.textInput}
        placeholder="What's on your mind?"
        value={draft.text}
        onChangeText={text => setDraft(prev => ({ ...prev, text }))}
        multiline
        maxLength={500}
      />
      
      <Text style={styles.charCount}>
        {draft.text.length}/500
      </Text>

      {/* Image previews */}
      {draft.images.length > 0 && (
        <ScrollView horizontal style={styles.imageRow}>
          {draft.images.map((uri, index) => (
            <View key={uri} style={styles.imageWrapper}>
              <Image source={{ uri }} style={styles.imagePreview} contentFit="cover" />
              <Pressable
                style={styles.removeImage}
                onPress={() => {
                  haptic.tap();
                  setDraft(prev => ({
                    ...prev,
                    images: prev.images.filter((_, i) => i !== index),
                  }));
                }}
              >
                <Text style={styles.removeImageText}>x</Text>
              </Pressable>
            </View>
          ))}
        </ScrollView>
      )}

      {/* Location tag */}
      {draft.location && (
        <View style={styles.locationTag}>
          <Text style={styles.locationText}>{draft.location.name}</Text>
          <Pressable
            onPress={() => {
              haptic.tap();
              setDraft(prev => ({ ...prev, location: null }));
            }}
          >
            <Text style={styles.removeText}>Remove</Text>
          </Pressable>
        </View>
      )}

      {/* Action buttons */}
      <View style={styles.actions}>
        <Pressable style={styles.actionButton} onPress={() => addPhoto('library')}>
          <Text style={styles.actionText}>Gallery</Text>
        </Pressable>
        <Pressable style={styles.actionButton} onPress={() => addPhoto('camera')}>
          <Text style={styles.actionText}>Camera</Text>
        </Pressable>
        <Pressable style={styles.actionButton} onPress={addLocation}>
          <Text style={styles.actionText}>Location</Text>
        </Pressable>
      </View>

      {/* Submit */}
      <Pressable
        style={[styles.submitButton, isSubmitting && styles.submitDisabled]}
        onPress={handleSubmit}
        disabled={isSubmitting}
      >
        <Text style={styles.submitText}>
          {isSubmitting ? 'Posting...' : 'Post'}
        </Text>
      </Pressable>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: 'white' },
  textInput: {
    fontSize: 18,
    minHeight: 120,
    textAlignVertical: 'top',
    lineHeight: 26,
  },
  charCount: { fontSize: 12, color: '#999', textAlign: 'right', marginTop: 4 },
  imageRow: { marginTop: 16, marginBottom: 8 },
  imageWrapper: { marginRight: 8, position: 'relative' },
  imagePreview: { width: 100, height: 100, borderRadius: 8 },
  removeImage: {
    position: 'absolute',
    top: 4,
    right: 4,
    width: 24,
    height: 24,
    borderRadius: 12,
    backgroundColor: 'rgba(0,0,0,0.6)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  removeImageText: { color: 'white', fontSize: 12, fontWeight: '700' },
  locationTag: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    padding: 12,
    backgroundColor: '#F0F7FF',
    borderRadius: 8,
    marginVertical: 8,
  },
  locationText: { fontSize: 14, color: '#007AFF' },
  removeText: { fontSize: 14, color: '#FF3B30' },
  actions: {
    flexDirection: 'row',
    gap: 8,
    marginVertical: 16,
    borderTopWidth: 1,
    borderTopColor: '#F0F0F0',
    paddingTop: 16,
  },
  actionButton: {
    flex: 1,
    padding: 12,
    backgroundColor: '#F5F5F5',
    borderRadius: 8,
    alignItems: 'center',
  },
  actionText: { fontSize: 14, fontWeight: '600', color: '#1A1A1A' },
  submitButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 8,
  },
  submitDisabled: { opacity: 0.5 },
  submitText: { color: 'white', fontSize: 16, fontWeight: '700' },
});
```

---

## 17. DEVICE API CHECKLIST

Before you ship any device API integration, run through this:

### Permissions
- [ ] Permission string is descriptive and specific (says WHY you need it)
- [ ] Pre-permission dialog explains the value before the system prompt
- [ ] Graceful fallback when permission is denied
- [ ] "Open Settings" option when permission is permanently denied
- [ ] Only request permissions when they're actually needed (not on app launch)

### Camera
- [ ] Camera unmounts when screen loses focus (`useFocusEffect`)
- [ ] Photo quality is appropriate (0.7-0.8 for uploads, 1.0 for local storage)
- [ ] Barcode scanning has debounce to prevent rapid-fire scans
- [ ] Both front and back camera work correctly

### Location
- [ ] Using the right accuracy level for the use case
- [ ] Background location has justified usage description
- [ ] Subscription is cleaned up on unmount
- [ ] Location caching to avoid redundant GPS requests

### Maps
- [ ] `tracksViewChanges={false}` on custom markers after initial render
- [ ] Clustering enabled for 50+ markers
- [ ] Region change callbacks are debounced

### Keyboard
- [ ] `KeyboardAwareScrollView` or `KeyboardAvoidingView` on all form screens
- [ ] `keyboardShouldPersistTaps="handled"` on all scrollable forms
- [ ] Submit buttons are accessible when keyboard is open
- [ ] Tested on both iOS and Android physical devices

### Haptics
- [ ] Haptic feedback on user actions (tap, toggle, complete)
- [ ] No haptics on rapid-fire events (scroll, typing)
- [ ] Platform check before calling haptic API
- [ ] Haptics supplement visual feedback, don't replace it

### Media
- [ ] Audio session configured correctly (silent mode, background audio)
- [ ] Sound resources cleaned up on unmount
- [ ] Video player handles orientation changes
- [ ] Image picker respects user-selected quality/crop

---

## Key Takeaways

1. **The permission request is a UI moment, not a technical detail.** Explain WHY before asking. Show value. Handle denial gracefully. Never dead-end the user.

2. **expo modules give you 80% of what you need.** Camera, location, haptics, clipboard, share sheet, sensors — they're all one `npx expo install` away. Don't reach for native code until you've exhausted the Expo module system.

3. **Background location will get your app rejected** if you don't have a legitimate, user-visible reason. Apple and Google both flag unnecessary background location requests. Only request "Always" if your app genuinely needs it.

4. **Haptics are the difference between "good" and "native."** Light impact on taps, success/error on outcomes, selection on pickers. But don't overdo it — too many haptics is worse than none.

5. **Keyboard handling is where React Native UX goes to die.** Use `react-native-keyboard-controller` for production apps. Test every form on both platforms, on physical devices, with different keyboard sizes.

6. **WebView is not an in-app browser.** Use `react-native-webview` for embedding web content with JS communication. Use `expo-web-browser` for OAuth and external links. Mixing them up creates security and UX problems.

7. **Always clean up.** Camera refs, location subscriptions, sensor listeners, audio players — they all hold device resources. Unmount them when the screen loses focus, and remove them when the component unmounts.

---

**Next up:** [Chapter 14: Permissions, Accessibility & Internationalization] — where we go deep on the permission patterns mentioned in this chapter, plus accessibility (VoiceOver, TalkBack) and internationalization (i18n, RTL, pluralization). Because your app isn't done until everyone can use it.
