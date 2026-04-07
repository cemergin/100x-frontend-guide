<!--
  CHAPTER: 14
  TITLE: Permissions, Accessibility & Internationalization
  PART: II — React Native & Expo
  PREREQS: Chapter 6
  KEY_TOPICS: permissions, requesting, denied handling, settings redirect, VoiceOver, TalkBack, semantic roles, focus management, a11y testing, react-i18next, expo-localization, RTL, pluralization
  DIFFICULTY: Intermediate
  UPDATED: 2026-04-07
-->

# Chapter 14: Permissions, Accessibility & Internationalization

> **Part II — React Native & Expo** | Prerequisites: Chapter 6 | Difficulty: Intermediate

> "The measure of a great app isn't how well it works for you. It's how well it works for everyone."
>
> "Accessibility isn't a feature. It's a quality metric."

---

<details>
<summary><strong>TL;DR</strong></summary>

- Permissions are a trust conversation with your user, not a technical checkbox; explain WHY before you ask, handle denial gracefully, and never request permissions you don't actively need
- iOS has "ask once" semantics — if the user denies, you can never ask again from code; you must redirect them to Settings. Android has a "never_ask_again" state that works similarly after two denials
- Pre-permission dialogs (your own UI explaining the value before the OS prompt) dramatically increase grant rates — 80%+ vs 40-50% for cold prompts
- VoiceOver (iOS) and TalkBack (Android) read your UI aloud to visually impaired users; if your components don't have `accessibilityLabel` and `accessibilityRole`, those users experience chaos
- Focus management after navigation, modals, and dynamic content updates is what separates accessible apps from inaccessible ones with a few labels sprinkled on
- react-i18next + expo-localization is the production i18n stack; structure translations by namespace, lazy-load them, and use ICU message format for pluralization
- RTL support (Arabic, Hebrew, Urdu) requires `I18nManager.forceRTL()` plus RTL-aware styles; test it early or face a painful retrofit

</details>

This chapter covers three topics that teams consistently underinvest in: permissions, accessibility, and internationalization. They're grouped together because they share a common theme — they're all about making your app work for people who aren't you.

Permissions are about respecting user agency. Accessibility is about serving users with disabilities. Internationalization is about serving users who speak different languages. All three are easy to bolt on badly and hard to retrofit properly. All three have real business consequences: App Store rejections, lawsuits, and missing entire markets.

Let's do them right from the start.

### In This Chapter
- Permission Model: iOS vs Android Differences
- Requesting Permissions: Pre-Permission Dialogs, Handling Denial
- Common Permissions: Camera, Location, Photos, Notifications, ATT
- App Store Review Considerations
- Why Accessibility Matters: Legal, Moral, and Business Cases
- VoiceOver & TalkBack: How Screen Readers Work
- React Native Accessibility Props
- Focus Management: Order, Navigation, Modal Trapping
- Testing Accessibility
- i18n Architecture: react-i18next + expo-localization
- RTL Support
- Pluralization: ICU Message Format
- Date/Number Formatting
- Dynamic Language Switching

### Related Chapters
- [Ch 6: EAS Mastery] — config plugins for permission setup
- [Ch 13: Device APIs & Native Features] — APIs that require permissions
- [Ch 8: Styling & Animation] — responsive and RTL-aware styling
- [Ch 7: Navigation Architecture] — focus management during navigation

---

# PART 1: PERMISSIONS

## 1. THE PERMISSION MODEL: iOS VS ANDROID

Permissions on mobile are not like permissions on the web. On the web, you get a browser prompt and the user clicks "Allow" or "Block." On mobile, the system controls when and how the prompt appears, and the rules are different on each platform.

### 1.1 iOS Permission States

iOS permissions have four possible states:

```
undetermined → granted
           → denied (permanent — cannot re-prompt from code)
```

```tsx
type IOSPermissionStatus = 
  | 'undetermined'  // User has never been asked
  | 'granted'       // User approved
  | 'denied'        // User denied (redirecting to Settings is the only option)
  | 'limited';      // iOS 14+ photo library: user granted partial access
```

**The critical rule on iOS:** You get ONE chance to ask. If the user denies the system prompt, calling `requestPermissionsAsync()` again does nothing — it immediately returns `'denied'` without showing any UI. The only way to change the permission is for the user to go to Settings > Your App > toggle it on.

This is why pre-permission dialogs matter so much on iOS. You need the user to understand the value BEFORE they see the system prompt, because you don't get a second shot.

### 1.2 Android Permission States

Android permissions are more forgiving but more complex:

```
undetermined → granted
           → denied (can ask again)
           → denied (never_ask_again — cannot re-prompt)
```

```tsx
type AndroidPermissionStatus = 
  | 'undetermined'     // User has never been asked
  | 'granted'          // User approved
  | 'denied'           // User denied but can be asked again
  | 'never_ask_again'; // User checked "Don't ask again" — must go to Settings
```

**Android quirks:**
- Starting with Android 11 (API 30), if the user denies a permission twice, the system automatically treats it as "Don't ask again"
- Android 13 (API 33) introduced granular photo/video/audio permissions (READ_MEDIA_IMAGES, READ_MEDIA_VIDEO, READ_MEDIA_AUDIO) replacing READ_EXTERNAL_STORAGE
- Android 13 also requires explicit POST_NOTIFICATIONS permission for push notifications
- Background location requires a separate request AFTER foreground location is granted (you cannot request both at once)

### 1.3 The Permission Status Type in Expo

Expo normalizes these platform differences into a consistent type:

```tsx
import { PermissionStatus } from 'expo-modules-core';

// Expo's unified permission status
type PermissionStatus = 
  | 'granted'       // Permission approved
  | 'denied'        // Permission denied
  | 'undetermined'; // Not yet asked

// The full permission response
interface PermissionResponse {
  status: PermissionStatus;
  granted: boolean;        // Convenience: status === 'granted'
  canAskAgain: boolean;    // Can we show the system prompt again?
  expires: 'never' | number; // When the permission expires
}
```

The `canAskAgain` field is the key. When it's `false`, calling `requestPermissionsAsync()` will not show a system prompt. You need to direct the user to Settings.

```tsx
import { Linking } from 'react-native';

function handleDeniedPermission(response: PermissionResponse) {
  if (!response.granted && !response.canAskAgain) {
    // User has permanently denied — redirect to Settings
    Alert.alert(
      'Permission Required',
      'This feature requires camera access. Please enable it in your device settings.',
      [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Open Settings', onPress: () => Linking.openSettings() },
      ]
    );
  }
}
```

---

## 2. REQUESTING PERMISSIONS: THE RIGHT WAY

### 2.1 The Pre-Permission Dialog Pattern

This is the single most impactful thing you can do for permission grant rates. Before showing the system prompt, show your own dialog that explains WHY you need the permission and WHAT the user gets from granting it.

```
Without pre-permission dialog:
  App opens → System prompt → 40-50% grant rate

With pre-permission dialog:
  App opens → Your explanation → User taps "Continue" → System prompt → 80-90% grant rate
```

The reason is simple: context. When a system prompt appears out of nowhere, the user's instinct is to deny. When they understand the value, they grant.

```tsx
import { useState, useCallback } from 'react';
import { View, Text, Pressable, StyleSheet, Modal, Image, Alert, Linking } from 'react-native';
import { Camera } from 'expo-camera';

interface PrePermissionDialogProps {
  visible: boolean;
  onAllow: () => void;
  onDeny: () => void;
  title: string;
  description: string;
  iconSource?: any;
}

function PrePermissionDialog({
  visible,
  onAllow,
  onDeny,
  title,
  description,
}: PrePermissionDialogProps) {
  return (
    <Modal
      visible={visible}
      transparent
      animationType="fade"
      onRequestClose={onDeny}
    >
      <View style={styles.overlay}>
        <View style={styles.dialog}>
          {/* Illustration or icon */}
          <View style={styles.iconContainer}>
            <Text style={styles.iconEmoji}>📸</Text>
          </View>

          <Text style={styles.dialogTitle}>{title}</Text>
          <Text style={styles.dialogDescription}>{description}</Text>

          <View style={styles.dialogButtons}>
            <Pressable style={styles.denyButton} onPress={onDeny}>
              <Text style={styles.denyButtonText}>Not Now</Text>
            </Pressable>
            <Pressable style={styles.allowButton} onPress={onAllow}>
              <Text style={styles.allowButtonText}>Allow</Text>
            </Pressable>
          </View>
        </View>
      </View>
    </Modal>
  );
}

// Usage in a component
function ProfilePhotoButton() {
  const [showPrePermission, setShowPrePermission] = useState(false);

  const handleTakePhoto = useCallback(async () => {
    // Check current permission status without asking
    const { status, canAskAgain } = await Camera.getCameraPermissionsAsync();

    if (status === 'granted') {
      // Already have permission — open camera
      openCamera();
      return;
    }

    if (!canAskAgain) {
      // User has permanently denied — redirect to Settings
      Alert.alert(
        'Camera Access Required',
        'Please enable camera access in Settings to take a profile photo.',
        [
          { text: 'Cancel', style: 'cancel' },
          { text: 'Open Settings', onPress: () => Linking.openSettings() },
        ]
      );
      return;
    }

    // Show pre-permission dialog first
    setShowPrePermission(true);
  }, []);

  const handleAllowPermission = useCallback(async () => {
    setShowPrePermission(false);

    // NOW show the system prompt
    const { status } = await Camera.requestCameraPermissionsAsync();

    if (status === 'granted') {
      openCamera();
    }
    // If denied, we don't bug them — they said no
  }, []);

  const handleDenyPermission = useCallback(() => {
    setShowPrePermission(false);
    // User said "Not Now" on our dialog — don't show system prompt
    // Maybe show the feature with a placeholder/fallback instead
  }, []);

  return (
    <>
      <Pressable style={styles.photoButton} onPress={handleTakePhoto}>
        <Text style={styles.photoButtonText}>Take Profile Photo</Text>
      </Pressable>

      <PrePermissionDialog
        visible={showPrePermission}
        onAllow={handleAllowPermission}
        onDeny={handleDenyPermission}
        title="Allow Camera Access"
        description="We need camera access so you can take a profile photo. Your photo helps other users recognize you."
      />
    </>
  );
}

function openCamera() {
  // Navigate to camera screen
}

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
    padding: 32,
  },
  dialog: {
    backgroundColor: 'white',
    borderRadius: 20,
    padding: 24,
    width: '100%',
    maxWidth: 340,
    alignItems: 'center',
  },
  iconContainer: {
    width: 80,
    height: 80,
    borderRadius: 40,
    backgroundColor: '#F0F7FF',
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 16,
  },
  iconEmoji: {
    fontSize: 36,
  },
  dialogTitle: {
    fontSize: 20,
    fontWeight: '700',
    color: '#1A1A1A',
    textAlign: 'center',
    marginBottom: 8,
  },
  dialogDescription: {
    fontSize: 15,
    color: '#666',
    textAlign: 'center',
    lineHeight: 22,
    marginBottom: 24,
  },
  dialogButtons: {
    flexDirection: 'row',
    gap: 12,
    width: '100%',
  },
  denyButton: {
    flex: 1,
    padding: 14,
    borderRadius: 12,
    backgroundColor: '#F0F0F0',
    alignItems: 'center',
  },
  denyButtonText: {
    fontSize: 16,
    fontWeight: '600',
    color: '#666',
  },
  allowButton: {
    flex: 1,
    padding: 14,
    borderRadius: 12,
    backgroundColor: '#007AFF',
    alignItems: 'center',
  },
  allowButtonText: {
    fontSize: 16,
    fontWeight: '600',
    color: 'white',
  },
  photoButton: {
    padding: 16,
    backgroundColor: '#007AFF',
    borderRadius: 8,
    alignItems: 'center',
  },
  photoButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
});
```

### 2.2 A Reusable Permission Hook

```tsx
import { useState, useCallback } from 'react';
import { Alert, Linking, Platform } from 'react-native';

type PermissionCheckFn = () => Promise<{ status: string; canAskAgain: boolean }>;
type PermissionRequestFn = () => Promise<{ status: string }>;

interface UsePermissionOptions {
  checkPermission: PermissionCheckFn;
  requestPermission: PermissionRequestFn;
  permissionName: string;       // Human-readable name ("Camera", "Location")
  explanation: string;           // Why the user should grant it
  settingsMessage?: string;      // Message when redirecting to Settings
}

interface UsePermissionResult {
  status: 'undetermined' | 'granted' | 'denied' | 'settings_required';
  isGranted: boolean;
  request: () => Promise<boolean>;  // Returns true if granted
  check: () => Promise<void>;
}

function usePermission(options: UsePermissionOptions): UsePermissionResult {
  const [status, setStatus] = useState<UsePermissionResult['status']>('undetermined');

  const check = useCallback(async () => {
    const result = await options.checkPermission();
    if (result.status === 'granted') {
      setStatus('granted');
    } else if (!result.canAskAgain) {
      setStatus('settings_required');
    } else {
      setStatus('undetermined');
    }
  }, [options.checkPermission]);

  const request = useCallback(async (): Promise<boolean> => {
    // First, check current status
    const currentStatus = await options.checkPermission();

    if (currentStatus.status === 'granted') {
      setStatus('granted');
      return true;
    }

    if (!currentStatus.canAskAgain) {
      // Must go to Settings
      setStatus('settings_required');
      
      Alert.alert(
        `${options.permissionName} Access Required`,
        options.settingsMessage ?? 
          `Please enable ${options.permissionName.toLowerCase()} access in Settings to use this feature.`,
        [
          { text: 'Cancel', style: 'cancel' },
          { text: 'Open Settings', onPress: () => Linking.openSettings() },
        ]
      );
      return false;
    }

    // Request permission (system prompt)
    const result = await options.requestPermission();

    if (result.status === 'granted') {
      setStatus('granted');
      return true;
    }

    setStatus('denied');
    return false;
  }, [options]);

  return {
    status,
    isGranted: status === 'granted',
    request,
    check,
  };
}

// Usage
import * as Camera from 'expo-camera';
import * as Location from 'expo-location';

function MyComponent() {
  const cameraPermission = usePermission({
    checkPermission: Camera.getCameraPermissionsAsync,
    requestPermission: Camera.requestCameraPermissionsAsync,
    permissionName: 'Camera',
    explanation: 'We need camera access to scan QR codes.',
  });

  const locationPermission = usePermission({
    checkPermission: Location.getForegroundPermissionsAsync,
    requestPermission: Location.requestForegroundPermissionsAsync,
    permissionName: 'Location',
    explanation: 'We need your location to show nearby results.',
  });

  const handleScan = async () => {
    const granted = await cameraPermission.request();
    if (granted) {
      // Open scanner
    }
  };

  // ...
}
```

### 2.3 Permission Request Timing

**When to request permissions:**

```
GOOD:
  User taps "Take Photo" → Request camera permission
  User taps "Show My Location" → Request location permission
  User taps "Add to Calendar" → Request calendar permission
  
  The user's action creates context. They understand WHY you're asking.

BAD:
  App launches → Request camera + location + contacts + calendar + notifications
  
  The user hasn't done anything yet. They don't know why you need all this.
  This is called "permission carpet bombing" and it tanks grant rates.
```

**The golden rule: Request permissions at the moment of relevance.** The user should have just done something that makes the permission request make sense.

---

## 3. COMMON PERMISSIONS AND THEIR QUIRKS

### 3.1 Camera Permission

```tsx
import { useCameraPermissions, getCameraPermissionsAsync } from 'expo-camera';

// Hook-based (recommended for components)
function CameraComponent() {
  const [permission, requestPermission] = useCameraPermissions();
  
  // permission.granted — boolean
  // permission.canAskAgain — boolean
  // permission.status — PermissionStatus
  // requestPermission() — shows system prompt
}

// Imperative (for utilities)
async function checkCamera() {
  const { status, canAskAgain } = await getCameraPermissionsAsync();
  return { granted: status === 'granted', canAskAgain };
}
```

**iOS quirk:** Camera permission also covers the microphone IF you requested it in your config plugin. If you didn't include microphone permission, video recording will be silent.

**Android quirk:** On Android, camera permission also grants barcode scanning. No additional permission needed.

### 3.2 Location Permission (When-in-Use vs Always)

This is the most complex permission on both platforms.

```tsx
import * as Location from 'expo-location';

// FOREGROUND (when-in-use) — the common case
async function requestForegroundLocation() {
  const { status, canAskAgain } = await Location.requestForegroundPermissionsAsync();
  return status === 'granted';
}

// BACKGROUND (always) — requires foreground FIRST
async function requestBackgroundLocation() {
  // Step 1: Must have foreground permission first
  const foreground = await Location.requestForegroundPermissionsAsync();
  if (foreground.status !== 'granted') {
    return false;
  }

  // Step 2: Request background permission
  // iOS: Shows "Change to Always Allow" prompt
  // Android 10+: Shows a separate dialog with "Allow all the time" option
  // Android 11+: Does NOT show a dialog — redirects user to Settings
  const background = await Location.requestBackgroundPermissionsAsync();
  return background.status === 'granted';
}
```

**iOS location states:**
```
undetermined → "When In Use" → "Always" (via separate prompt)
                            → denied
           → denied (permanent)
```

**Android location states:**
```
undetermined → "While using the app" (foreground only)
             → "Only this time" (one-shot, reverts to undetermined)
             → denied
             
"While using the app" → "Allow all the time" (background)
                      → denied
```

**Android 11+ background location:** The system CANNOT show a prompt for background location. You must display your own UI telling the user to go to Settings > Location > "Allow all the time". This is intentional — Google wants developers to justify background location to the user, not just trigger a system dialog.

```tsx
import { Platform, Alert, Linking } from 'react-native';

async function requestBackgroundLocationAndroid11Plus() {
  if (Platform.OS === 'android' && Platform.Version >= 30) {
    Alert.alert(
      'Background Location Required',
      'To track your run in the background, please go to Settings and select "Allow all the time" for location access.',
      [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Open Settings', onPress: () => Linking.openSettings() },
      ]
    );
    return;
  }

  // For older Android and iOS, the system prompt works
  const { status } = await Location.requestBackgroundPermissionsAsync();
  return status === 'granted';
}
```

### 3.3 Photo Library Permission

```tsx
import * as ImagePicker from 'expo-image-picker';

async function requestPhotoLibrary() {
  const { status, canAskAgain } = 
    await ImagePicker.requestMediaLibraryPermissionsAsync();
  return { granted: status === 'granted', canAskAgain };
}
```

**iOS 14+ "Limited" access:** Users can grant access to only selected photos, not their entire library. Your app sees a subset of the photo library. You should handle this gracefully:

```tsx
import * as ImagePicker from 'expo-image-picker';

async function handlePhotoPermission() {
  const result = await ImagePicker.requestMediaLibraryPermissionsAsync();
  
  if (result.status === 'granted') {
    // Full access — can browse entire library
    return 'full';
  }
  
  if (result.accessPrivileges === 'limited') {
    // Limited access — user selected specific photos
    // The image picker will only show those photos
    return 'limited';
  }
  
  return 'denied';
}
```

### 3.4 Notification Permission

```tsx
import * as Notifications from 'expo-notifications';
import { Platform } from 'react-native';

async function requestNotificationPermission() {
  // Android 12 and below: notifications are granted by default
  // Android 13+: requires explicit permission
  // iOS: always requires explicit permission

  const { status: existingStatus } = 
    await Notifications.getPermissionsAsync();

  if (existingStatus === 'granted') return true;

  const { status } = await Notifications.requestPermissionsAsync({
    ios: {
      allowAlert: true,
      allowBadge: true,
      allowSound: true,
      allowCriticalAlerts: false, // Requires special entitlement from Apple
      allowProvisional: false,    // "Quiet" notifications (no prompt, deliver silently)
    },
  });

  return status === 'granted';
}

// Pro tip: iOS "provisional" notifications
// These deliver silently to the Notification Center without showing a prompt.
// Users can then choose to keep or turn off notifications.
// Great for low-stakes notification types.
async function requestProvisionalNotifications() {
  if (Platform.OS !== 'ios') {
    return requestNotificationPermission();
  }

  const { status } = await Notifications.requestPermissionsAsync({
    ios: {
      allowAlert: true,
      allowBadge: true,
      allowSound: true,
      allowProvisional: true, // This is the key
    },
  });

  return status === 'granted';
}
```

### 3.5 App Tracking Transparency (ATT) — iOS Only

iOS 14.5+ requires explicit permission before tracking users across apps (IDFA). This is the infamous "Allow [App] to track your activity?" prompt.

```bash
npx expo install expo-tracking-transparency
```

```tsx
import { requestTrackingPermissionsAsync, getTrackingPermissionsAsync } from 'expo-tracking-transparency';
import { Platform } from 'react-native';

async function requestTrackingPermission() {
  if (Platform.OS !== 'ios') return true; // Only iOS has ATT

  const { status } = await getTrackingPermissionsAsync();
  
  if (status === 'granted') return true;
  if (status === 'denied') return false;
  
  // Show the system ATT prompt
  const { status: newStatus } = await requestTrackingPermissionsAsync();
  return newStatus === 'granted';
}

// Usage: Request ATT before initializing analytics/ad SDKs
async function initializeAnalytics() {
  const trackingAllowed = await requestTrackingPermission();
  
  if (trackingAllowed) {
    // Initialize full analytics with IDFA
    // analytics.init({ enableTracking: true });
  } else {
    // Initialize analytics WITHOUT cross-app tracking
    // analytics.init({ enableTracking: false });
    // You can still track in-app events — just no IDFA
  }
}
```

**ATT timing:** Apple recommends showing ATT after the user has experienced value from your app, not on first launch. If the very first thing a user sees is a tracking prompt, they'll deny it AND be annoyed.

### 3.6 Microphone Permission

```tsx
import { Audio } from 'expo-av';

async function requestMicrophonePermission() {
  const { status, canAskAgain } = await Audio.requestPermissionsAsync();
  return { granted: status === 'granted', canAskAgain };
}
```

### 3.7 Contacts and Calendar

```tsx
import * as Contacts from 'expo-contacts';
import * as Calendar from 'expo-calendar';

// Contacts
async function requestContactsPermission() {
  const { status } = await Contacts.requestPermissionsAsync();
  return status === 'granted';
}

// Calendar events
async function requestCalendarPermission() {
  const { status } = await Calendar.requestCalendarPermissionsAsync();
  return status === 'granted';
}

// Calendar reminders (iOS only)
async function requestRemindersPermission() {
  const { status } = await Calendar.requestRemindersPermissionsAsync();
  return status === 'granted';
}
```

---

## 4. APP STORE REVIEW CONSIDERATIONS

### 4.1 Apple App Store

Apple is aggressive about permission usage. Here's what gets apps rejected:

**Requesting permissions you don't use:**
Apple's automated and manual review checks whether your binary contains usage of APIs matching the permissions you declare. If you declare camera permission but never actually use the camera API, your app will be flagged.

**Vague permission descriptions:**
"MyApp needs access to your location" will get rejected. "MyApp needs your location to show nearby restaurants" will pass. Be specific about WHY.

**Requesting all permissions on launch:**
Requesting camera + location + contacts + notifications before the user has done anything is a rejection risk. Request at the moment of relevance.

**Background location without justification:**
If you request "Always" location access, Apple wants to see that your app genuinely needs it. Navigation apps, fitness trackers, geofencing apps — yes. A restaurant finder — no (use "When in Use").

**ATT without ad tracking:**
If you request ATT permission but don't actually use the IDFA for advertising, Apple may reject. Only request ATT if you're actually doing cross-app tracking.

```json
// app.json — permission descriptions that pass Apple review
{
  "expo": {
    "ios": {
      "infoPlist": {
        "NSCameraUsageDescription": "$(PRODUCT_NAME) needs camera access to take profile photos and scan QR codes.",
        "NSPhotoLibraryUsageDescription": "$(PRODUCT_NAME) needs photo library access to select a profile photo.",
        "NSLocationWhenInUseUsageDescription": "$(PRODUCT_NAME) needs your location to show nearby restaurants on the map.",
        "NSLocationAlwaysAndWhenInUseUsageDescription": "$(PRODUCT_NAME) needs your location to send you notifications when you're near a saved restaurant.",
        "NSMicrophoneUsageDescription": "$(PRODUCT_NAME) needs microphone access to record voice messages.",
        "NSContactsUsageDescription": "$(PRODUCT_NAME) needs contacts access to help you invite friends.",
        "NSCalendarsUsageDescription": "$(PRODUCT_NAME) needs calendar access to add reservation reminders.",
        "NSUserTrackingUsageDescription": "$(PRODUCT_NAME) uses your data to provide personalized ads and measure ad effectiveness."
      }
    }
  }
}
```

### 4.2 Google Play Store

Google flags these permission patterns:

**SMS and Call Log permissions:**
Google heavily restricts SMS and call log access. Your app will be removed unless it's a default SMS/dialer app. Don't use these permissions unless absolutely necessary.

**QUERY_ALL_PACKAGES:**
Listing all installed apps requires justification. Google wants to know why your app needs to see what other apps are installed.

**Background location:**
Like Apple, Google wants justification for background location. You'll need to fill out a declaration in the Google Play Console explaining why.

**Excessive permissions:**
Requesting permissions you don't use will trigger a warning in the Play Console. Keep your permission list minimal.

```json
// app.json — Android permissions
{
  "expo": {
    "android": {
      "permissions": [
        "CAMERA",
        "ACCESS_FINE_LOCATION",
        "ACCESS_COARSE_LOCATION",
        "RECORD_AUDIO",
        "READ_CONTACTS",
        "WRITE_CONTACTS",
        "READ_CALENDAR",
        "WRITE_CALENDAR"
      ],
      "blockedPermissions": [
        "READ_PHONE_STATE",
        "READ_CALL_LOG",
        "READ_SMS"
      ]
    }
  }
}
```

The `blockedPermissions` field is useful for preventing third-party libraries from adding permissions you don't want. Some analytics/ad SDKs sneakily add permissions through their AndroidManifest merge.

---

# PART 2: ACCESSIBILITY

## 5. WHY ACCESSIBILITY MATTERS

I'm going to be blunt: most mobile apps are inaccessible. Not "could be better" inaccessible — completely unusable for people with disabilities. Buttons without labels. Screens where VoiceOver reads nonsense. Focus traps that make it impossible to navigate. Color contrast that fails WCAG guidelines.

Here's why you should care:

### 5.1 The Legal Case

- **ADA (Americans with Disabilities Act):** Courts have consistently ruled that mobile apps are "places of public accommodation" under Title III. Companies including Domino's, Target, and Netflix have faced lawsuits.
- **European Accessibility Act (EAA):** Effective June 2025, requires digital products sold in the EU to be accessible. This includes mobile apps.
- **Section 508:** Applies to US government agencies and their contractors. If you build apps for government, accessibility is a legal requirement.

### 5.2 The Market Case

- **15% of the world's population** has some form of disability (WHO)
- **1 in 4 US adults** has a disability (CDC)
- People with disabilities have **$8 trillion in disposable income** globally
- Aging populations in developed countries are growing — accessibility features serve them too

### 5.3 The Quality Case

Accessible apps are better apps. Period.

- Good accessibility means clear, semantic UI structure — which makes your code more maintainable
- Focus management means predictable navigation — which benefits power users too
- Good contrast and typography means readable UI — which helps everyone in bright sunlight
- Accessible forms have better error messages — which reduces support tickets for everyone

---

## 6. VOICEOVER (iOS) AND TALKBACK (ANDROID)

### 6.1 How Screen Readers Work

A screen reader converts visual UI into audio output. When a user swipes right on their phone, the screen reader moves to the next focusable element and reads its label, role, and state aloud.

**VoiceOver (iOS):** Built into every iPhone. Activated via Settings > Accessibility > VoiceOver, or by triple-clicking the side button (if configured). Users navigate by:
- **Swiping right:** Move to next element
- **Swiping left:** Move to previous element
- **Double-tap:** Activate the focused element
- **Three-finger swipe:** Scroll

**TalkBack (Android):** Built into most Android phones. Activated via Settings > Accessibility > TalkBack. Users navigate by:
- **Swiping right:** Move to next element
- **Swiping left:** Move to previous element
- **Double-tap:** Activate the focused element
- **Two-finger scroll:** Scroll up/down

### 6.2 The Accessibility Tree

React Native builds an accessibility tree from your component hierarchy. The screen reader reads this tree, not the visual layout. If your components don't provide accessibility information, the screen reader has nothing useful to read.

```tsx
// BAD: Screen reader says "Button" (no context)
<Pressable onPress={handleDelete}>
  <TrashIcon />
</Pressable>

// GOOD: Screen reader says "Delete item, button"
<Pressable 
  onPress={handleDelete}
  accessibilityLabel="Delete item"
  accessibilityRole="button"
>
  <TrashIcon />
</Pressable>
```

### 6.3 Testing with Screen Readers

**How to test VoiceOver (iOS):**
1. Open Settings > Accessibility > VoiceOver
2. Turn it ON (you can also triple-click the side button)
3. Navigate your app by swiping right/left and double-tapping
4. Listen to what VoiceOver reads for each element
5. Check: Does it make sense? Is the order logical? Can you reach everything?

**How to test TalkBack (Android):**
1. Open Settings > Accessibility > TalkBack
2. Turn it ON
3. Navigate your app the same way
4. Note: TalkBack has slightly different behavior than VoiceOver

**Pro tip:** Set up a keyboard shortcut to toggle VoiceOver quickly. On iOS, go to Settings > Accessibility > Accessibility Shortcut > select VoiceOver. Then triple-click the side button to toggle it on/off during development.

---

## 7. REACT NATIVE ACCESSIBILITY PROPS

### 7.1 The Essential Props

```tsx
import { View, Text, Pressable, Image, Switch, TextInput } from 'react-native';

// accessibilityLabel: What the screen reader reads aloud
// This is the most important prop. If you set nothing else, set this.
<Pressable
  accessibilityLabel="Delete message from John"
  onPress={handleDelete}
>
  <TrashIcon />
</Pressable>

// accessibilityRole: Tells the screen reader what TYPE of element this is
// This determines how the screen reader announces it ("button", "link", etc.)
<Pressable
  accessibilityLabel="View profile"
  accessibilityRole="button" // "button" | "link" | "header" | "image" | 
                              // "search" | "switch" | "tab" | "checkbox" |
                              // "radio" | "adjustable" | "imagebutton" |
                              // "text" | "none" | "alert" | "progressbar"
  onPress={handleViewProfile}
>
  <Text>Profile</Text>
</Pressable>

// accessibilityState: Current state of the element
<Pressable
  accessibilityLabel="Favorite"
  accessibilityRole="button"
  accessibilityState={{
    selected: isFavorited,    // Is this item selected?
    disabled: isLoading,      // Is this element disabled?
    checked: isChecked,       // Is this checkbox/switch checked? (true | false | 'mixed')
    busy: isSubmitting,       // Is this element loading?
    expanded: isExpanded,     // Is this accordion/dropdown expanded?
  }}
  onPress={toggleFavorite}
>
  <HeartIcon filled={isFavorited} />
</Pressable>

// accessibilityHint: Additional context for what will happen
// VoiceOver reads this after a pause. Keep it short.
<Pressable
  accessibilityLabel="Add to cart"
  accessibilityRole="button"
  accessibilityHint="Adds this item to your shopping cart"
  onPress={addToCart}
>
  <Text>Add to Cart</Text>
</Pressable>

// accessibilityValue: For adjustable elements (sliders, progress bars)
<View
  accessibilityLabel="Volume"
  accessibilityRole="adjustable"
  accessibilityValue={{
    min: 0,
    max: 100,
    now: currentVolume,
    text: `${currentVolume}%`, // Human-readable value
  }}
>
  <Slider value={currentVolume} onValueChange={setVolume} />
</View>
```

### 7.2 Grouping and Hiding Elements

```tsx
// accessible={true}: Groups children into a single focusable element
// Screen reader reads the group label instead of individual children
<View
  accessible={true}
  accessibilityLabel={`${name}, ${price}, ${rating} stars`}
  accessibilityRole="button"
>
  <Text>{name}</Text>
  <Text>{price}</Text>
  <StarRating value={rating} />
</View>

// importantForAccessibility: Control visibility in the a11y tree
<View importantForAccessibility="no-hide-descendants">
  {/* Everything in here is hidden from screen readers */}
  {/* Use for decorative elements that add no information */}
  <DecorativeBorder />
  <BackgroundAnimation />
</View>

// Individual element hiding
<Image
  source={decorativeImage}
  importantForAccessibility="no"
  accessibilityElementsHidden={true} // iOS equivalent
/>

// accessibilityElementsHidden (iOS) / importantForAccessibility (Android)
// These serve the same purpose but are platform-specific
// Use both for cross-platform:
<View
  accessibilityElementsHidden={true}    // iOS
  importantForAccessibility="no-hide-descendants" // Android
>
  <DecorativeContent />
</View>
```

### 7.3 Accessible Custom Components

Most accessibility bugs come from custom components that don't provide the right props. Here's how to build accessible custom components:

```tsx
import { Pressable, View, Text, StyleSheet, AccessibilityInfo, Platform } from 'react-native';
import { useState, useCallback } from 'react';

// Accessible toggle/switch component
interface AccessibleToggleProps {
  label: string;
  value: boolean;
  onValueChange: (value: boolean) => void;
  disabled?: boolean;
  hint?: string;
}

function AccessibleToggle({
  label,
  value,
  onValueChange,
  disabled = false,
  hint,
}: AccessibleToggleProps) {
  const handlePress = useCallback(() => {
    if (disabled) return;
    onValueChange(!value);
  }, [disabled, value, onValueChange]);

  return (
    <Pressable
      style={[styles.toggleContainer, disabled && styles.toggleDisabled]}
      onPress={handlePress}
      accessibilityLabel={label}
      accessibilityRole="switch"
      accessibilityState={{
        checked: value,
        disabled,
      }}
      accessibilityHint={hint}
    >
      <Text style={[styles.toggleLabel, disabled && styles.toggleLabelDisabled]}>
        {label}
      </Text>
      <View style={[
        styles.toggleTrack,
        value && styles.toggleTrackOn,
        disabled && styles.toggleTrackDisabled,
      ]}>
        <View style={[
          styles.toggleThumb,
          value && styles.toggleThumbOn,
        ]} />
      </View>
    </Pressable>
  );
}

// Accessible accordion component
interface AccordionProps {
  title: string;
  children: React.ReactNode;
}

function AccessibleAccordion({ title, children }: AccordionProps) {
  const [expanded, setExpanded] = useState(false);

  return (
    <View>
      <Pressable
        style={styles.accordionHeader}
        onPress={() => setExpanded(!expanded)}
        accessibilityLabel={title}
        accessibilityRole="button"
        accessibilityState={{ expanded }}
        accessibilityHint={`${expanded ? 'Collapse' : 'Expand'} ${title} section`}
      >
        <Text style={styles.accordionTitle}>{title}</Text>
        <Text style={styles.accordionIcon}>{expanded ? '−' : '+'}</Text>
      </Pressable>
      
      {expanded && (
        <View 
          style={styles.accordionContent}
          accessibilityLiveRegion="polite" // Announce content changes
        >
          {children}
        </View>
      )}
    </View>
  );
}

// Accessible card with grouped content
interface ProductCardProps {
  name: string;
  price: string;
  rating: number;
  imageUrl: string;
  onPress: () => void;
}

function AccessibleProductCard({
  name,
  price,
  rating,
  imageUrl,
  onPress,
}: ProductCardProps) {
  return (
    <Pressable
      style={styles.card}
      onPress={onPress}
      accessible={true} // Group all children into one focusable element
      accessibilityLabel={`${name}, ${price}, rated ${rating} out of 5 stars`}
      accessibilityRole="button"
      accessibilityHint="Opens product details"
    >
      <Image
        source={{ uri: imageUrl }}
        style={styles.cardImage}
        // Image is decorative when part of a grouped element
        // The card's label already describes the product
        accessibilityElementsHidden={true}
        importantForAccessibility="no"
      />
      <View style={styles.cardInfo}>
        <Text style={styles.cardName}>{name}</Text>
        <Text style={styles.cardPrice}>{price}</Text>
        <View style={styles.cardRating}>
          {Array.from({ length: 5 }, (_, i) => (
            <Text key={i} style={styles.star}>
              {i < rating ? '★' : '☆'}
            </Text>
          ))}
        </View>
      </View>
    </Pressable>
  );
}

const styles = StyleSheet.create({
  // Toggle styles
  toggleContainer: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
  },
  toggleDisabled: { opacity: 0.5 },
  toggleLabel: { fontSize: 16, color: '#1A1A1A' },
  toggleLabelDisabled: { color: '#999' },
  toggleTrack: {
    width: 50,
    height: 30,
    borderRadius: 15,
    backgroundColor: '#DDD',
    justifyContent: 'center',
    padding: 2,
  },
  toggleTrackOn: { backgroundColor: '#34C759' },
  toggleTrackDisabled: { backgroundColor: '#F0F0F0' },
  toggleThumb: {
    width: 26,
    height: 26,
    borderRadius: 13,
    backgroundColor: 'white',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.2,
    shadowRadius: 2,
    elevation: 2,
  },
  toggleThumbOn: { alignSelf: 'flex-end' },

  // Accordion styles
  accordionHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: '#F5F5F5',
    borderRadius: 8,
  },
  accordionTitle: { fontSize: 16, fontWeight: '600', color: '#1A1A1A' },
  accordionIcon: { fontSize: 20, color: '#666' },
  accordionContent: { padding: 16 },

  // Card styles
  card: {
    backgroundColor: 'white',
    borderRadius: 12,
    overflow: 'hidden',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 4,
  },
  cardImage: { width: '100%', height: 200 },
  cardInfo: { padding: 16 },
  cardName: { fontSize: 16, fontWeight: '600', color: '#1A1A1A' },
  cardPrice: { fontSize: 18, fontWeight: '700', color: '#007AFF', marginTop: 4 },
  cardRating: { flexDirection: 'row', marginTop: 4 },
  star: { fontSize: 16, color: '#FFB800' },
});
```

---

## 8. FOCUS MANAGEMENT

Focus management is the hardest part of mobile accessibility. It's what determines the ORDER in which screen readers navigate through your UI, and what happens to focus when the UI changes dynamically.

### 8.1 Controlling Focus Order

By default, VoiceOver and TalkBack move through elements in the order they appear in the component tree (top to bottom, left to right for LTR languages). Most of the time this is correct. When it's not, you need to intervene.

```tsx
import { View, Text, Pressable, AccessibilityInfo, findNodeHandle } from 'react-native';
import { useRef, useEffect } from 'react';

// Setting focus programmatically
function SearchResults({ results, query }: { results: any[]; query: string }) {
  const resultsHeaderRef = useRef<View>(null);

  useEffect(() => {
    if (results.length > 0 && resultsHeaderRef.current) {
      // After results load, move VoiceOver focus to the results header
      const handle = findNodeHandle(resultsHeaderRef.current);
      if (handle) {
        AccessibilityInfo.setAccessibilityFocus(handle);
      }
    }
  }, [results]);

  return (
    <View>
      <View 
        ref={resultsHeaderRef}
        accessible
        accessibilityRole="header"
        accessibilityLabel={`${results.length} results for ${query}`}
      >
        <Text style={styles.resultsHeader}>
          {results.length} results for "{query}"
        </Text>
      </View>

      {results.map(result => (
        <ResultItem key={result.id} result={result} />
      ))}
    </View>
  );
}
```

### 8.2 Focus After Navigation

When the user navigates to a new screen, the screen reader should focus on the screen title or the first meaningful content. Expo Router and React Navigation handle this automatically for stack navigators, but you may need to adjust for modals, bottom sheets, and custom transitions.

```tsx
import { useFocusEffect } from 'expo-router';
import { useCallback, useRef } from 'react';
import { View, Text, AccessibilityInfo, findNodeHandle } from 'react-native';

function ModalScreen() {
  const titleRef = useRef<Text>(null);

  useFocusEffect(
    useCallback(() => {
      // When this screen gains focus, set VoiceOver focus on the title
      const timeout = setTimeout(() => {
        const handle = findNodeHandle(titleRef.current);
        if (handle) {
          AccessibilityInfo.setAccessibilityFocus(handle);
        }
      }, 100); // Small delay to let the animation complete

      return () => clearTimeout(timeout);
    }, [])
  );

  return (
    <View>
      <Text
        ref={titleRef}
        accessibilityRole="header"
        style={styles.title}
      >
        Edit Profile
      </Text>
      {/* ... */}
    </View>
  );
}
```

### 8.3 Modal Focus Trapping

When a modal is open, the screen reader should NOT be able to navigate to content behind the modal. React Native's `<Modal>` component handles this automatically, but custom modals (like bottom sheets built with Reanimated) do not.

```tsx
import { Modal, View, Text, Pressable, StyleSheet } from 'react-native';
import { useRef, useEffect } from 'react';

// Using React Native's Modal (handles focus trapping automatically)
function AccessibleModal({
  visible,
  onClose,
  title,
  children,
}: {
  visible: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}) {
  const closeButtonRef = useRef<View>(null);

  return (
    <Modal
      visible={visible}
      transparent
      animationType="slide"
      onRequestClose={onClose} // Android back button
      // The Modal component automatically traps focus on iOS and Android
    >
      <View style={styles.modalOverlay}>
        <View 
          style={styles.modalContent}
          // accessibilityViewIsModal tells iOS that this view contains the modal
          // All sibling views become inaccessible
          accessibilityViewIsModal={true}
        >
          {/* Header with close button */}
          <View style={styles.modalHeader}>
            <Text
              accessibilityRole="header"
              style={styles.modalTitle}
            >
              {title}
            </Text>
            <Pressable
              ref={closeButtonRef}
              onPress={onClose}
              accessibilityLabel="Close"
              accessibilityRole="button"
              style={styles.closeButton}
            >
              <Text style={styles.closeButtonText}>X</Text>
            </Pressable>
          </View>

          {/* Modal content */}
          {children}
        </View>
      </View>
    </Modal>
  );
}

// For custom modals (like Reanimated bottom sheets), you need to manually
// hide the background content from accessibility:
function ScreenWithCustomModal() {
  const [isModalOpen, setIsModalOpen] = useState(false);

  return (
    <View style={{ flex: 1 }}>
      {/* Background content — hide from accessibility when modal is open */}
      <View
        importantForAccessibility={isModalOpen ? 'no-hide-descendants' : 'auto'}
        accessibilityElementsHidden={isModalOpen}
      >
        <Text>Background content</Text>
        <Pressable onPress={() => setIsModalOpen(true)}>
          <Text>Open Modal</Text>
        </Pressable>
      </View>

      {/* Custom modal */}
      {isModalOpen && (
        <View
          style={styles.customModal}
          accessibilityViewIsModal={true}
        >
          <Text accessibilityRole="header">Modal Title</Text>
          <Pressable onPress={() => setIsModalOpen(false)}>
            <Text>Close</Text>
          </Pressable>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'flex-end',
  },
  modalContent: {
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 24,
    maxHeight: '80%',
  },
  modalHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 16,
  },
  modalTitle: {
    fontSize: 20,
    fontWeight: '700',
    color: '#1A1A1A',
  },
  closeButton: {
    width: 32,
    height: 32,
    borderRadius: 16,
    backgroundColor: '#F0F0F0',
    justifyContent: 'center',
    alignItems: 'center',
  },
  closeButtonText: {
    fontSize: 16,
    fontWeight: '600',
    color: '#666',
  },
  customModal: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 24,
  },
});
```

### 8.4 Announcing Dynamic Content

When content changes without a page transition (loading states, error messages, live updates), you need to announce the change to screen readers:

```tsx
import { AccessibilityInfo, View, Text } from 'react-native';

// Announce a message to the screen reader
function announceForAccessibility(message: string) {
  AccessibilityInfo.announceForAccessibility(message);
}

// Example: Announce form validation errors
function FormField({ error }: { error: string | null }) {
  useEffect(() => {
    if (error) {
      announceForAccessibility(`Error: ${error}`);
    }
  }, [error]);

  return (
    <View>
      <TextInput />
      {error && (
        <Text 
          accessibilityRole="alert" // Automatically announced by screen readers
          style={styles.error}
        >
          {error}
        </Text>
      )}
    </View>
  );
}

// accessibilityLiveRegion: Automatically announce changes to this view's content
// 'polite' — announces after current speech finishes
// 'assertive' — interrupts current speech immediately
// 'none' — default, no announcements
function NotificationBanner({ message }: { message: string | null }) {
  if (!message) return null;

  return (
    <View
      accessibilityLiveRegion="polite"
      accessibilityRole="alert"
      style={styles.banner}
    >
      <Text>{message}</Text>
    </View>
  );
}

// Example: Announce loading state changes
function DataLoader() {
  const [loading, setLoading] = useState(false);
  const [data, setData] = useState(null);

  const loadData = async () => {
    setLoading(true);
    announceForAccessibility('Loading data...');
    
    try {
      const result = await fetchData();
      setData(result);
      announceForAccessibility(`Loaded ${result.length} items`);
    } catch (error) {
      announceForAccessibility('Failed to load data. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  // ...
}
```

---

## 9. TESTING ACCESSIBILITY

### 9.1 Manual Testing Checklist

Run through this checklist for every screen in your app:

```
VoiceOver / TalkBack Testing:
  [ ] Every interactive element has a label (accessibilityLabel)
  [ ] Every interactive element has a role (accessibilityRole)
  [ ] Focus order follows visual reading order
  [ ] No elements are unreachable via swipe navigation
  [ ] Decorative elements are hidden from screen readers
  [ ] Dynamic content changes are announced
  [ ] Modals trap focus (can't navigate behind them)
  [ ] After navigation, focus lands on the screen title
  [ ] Error messages are announced when they appear
  [ ] Loading states are announced
  [ ] All images have labels or are marked as decorative
  [ ] Custom gestures have accessible alternatives (e.g., swipe-to-delete has a delete button)

Visual Accessibility:
  [ ] Text contrast ratio is at least 4.5:1 (WCAG AA)
  [ ] Large text (18pt+) contrast ratio is at least 3:1
  [ ] Interactive elements have a minimum touch target of 44x44 points
  [ ] Color is not the ONLY way to convey information (also use icons, text, shapes)
  [ ] UI respects Dynamic Type / system font size settings
  [ ] UI is usable with bold text enabled
  [ ] UI is usable with reduce motion enabled

Interaction Accessibility:
  [ ] All features accessible without gestures (swipe-to-delete has an alternative)
  [ ] Double-tap works on all interactive elements
  [ ] Long-press actions have alternative access
  [ ] Timeouts give sufficient time (or have no timeout)
  [ ] Auto-playing media can be paused
```

### 9.2 Automated Testing with eslint-plugin-react-native-a11y

```bash
npm install --save-dev eslint-plugin-react-native-a11y
```

```js
// .eslintrc.js
module.exports = {
  plugins: ['react-native-a11y'],
  rules: {
    'react-native-a11y/has-accessibility-props': 'warn',
    'react-native-a11y/has-valid-accessibility-role': 'error',
    'react-native-a11y/has-valid-accessibility-state': 'error',
    'react-native-a11y/has-valid-accessibility-value': 'error',
    'react-native-a11y/no-nested-touchables': 'error',
    'react-native-a11y/has-valid-accessibility-actions': 'error',
    'react-native-a11y/has-valid-accessibility-live-region': 'error',
    'react-native-a11y/has-valid-important-for-accessibility': 'error',
  },
};
```

### 9.3 Testing with Detox/Maestro

```yaml
# maestro/accessibility-test.yaml
appId: com.myapp
---
# Test that VoiceOver labels are present
- assertVisible:
    text: "Delete item"
    
# Test focus order
- tapOn:
    accessibilityLabel: "Search"
- assertFocused:
    accessibilityLabel: "Search input"

# Test that modal traps focus
- tapOn:
    accessibilityLabel: "Open settings"
- assertVisible:
    accessibilityLabel: "Settings"
- assertNotVisible: 
    accessibilityLabel: "Home"  # Should be hidden behind modal
```

### 9.4 Checking Screen Reader Behavior in Code

```tsx
import { AccessibilityInfo, Platform } from 'react-native';
import { useState, useEffect } from 'react';

function useScreenReaderStatus() {
  const [isScreenReaderEnabled, setIsScreenReaderEnabled] = useState(false);

  useEffect(() => {
    AccessibilityInfo.isScreenReaderEnabled().then(setIsScreenReaderEnabled);

    const subscription = AccessibilityInfo.addEventListener(
      'screenReaderChanged',
      setIsScreenReaderEnabled
    );

    return () => subscription.remove();
  }, []);

  return isScreenReaderEnabled;
}

function useReduceMotion() {
  const [reduceMotion, setReduceMotion] = useState(false);

  useEffect(() => {
    AccessibilityInfo.isReduceMotionEnabled().then(setReduceMotion);

    const subscription = AccessibilityInfo.addEventListener(
      'reduceMotionChanged',
      setReduceMotion
    );

    return () => subscription.remove();
  }, []);

  return reduceMotion;
}

// Usage: Adapt UI for accessibility settings
function AnimatedComponent() {
  const reduceMotion = useReduceMotion();
  const isScreenReader = useScreenReaderStatus();

  // Skip animations if reduce motion is enabled
  const animationDuration = reduceMotion ? 0 : 300;

  // Provide text alternatives if screen reader is active
  if (isScreenReader) {
    return (
      <View accessible accessibilityLabel="Progress: 75%">
        <Text>Progress: 75%</Text>
      </View>
    );
  }

  return (
    <Animated.View style={/* animated progress bar */}>
      {/* Visual progress bar */}
    </Animated.View>
  );
}
```

---

# PART 3: INTERNATIONALIZATION

## 10. I18N ARCHITECTURE: REACT-I18NEXT + EXPO-LOCALIZATION

### 10.1 The Stack

The production i18n stack for React Native is:

- **expo-localization** — Detects the device's language, region, and preferences
- **react-i18next** — The translation framework (wraps i18next for React)
- **i18next** — The core internationalization engine
- **i18next-resources-to-backend** — Lazy-loads translation files (optional, for large apps)

```bash
npx expo install expo-localization
npm install react-i18next i18next
```

### 10.2 Setting Up i18n

```tsx
// src/i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import { getLocales } from 'expo-localization';

// Import translation files
import en from './locales/en.json';
import es from './locales/es.json';
import fr from './locales/fr.json';
import ar from './locales/ar.json';
import ja from './locales/ja.json';

// Get the device's preferred language
const deviceLanguage = getLocales()[0]?.languageCode ?? 'en';

// Supported languages
const SUPPORTED_LANGUAGES = ['en', 'es', 'fr', 'ar', 'ja'] as const;
type SupportedLanguage = typeof SUPPORTED_LANGUAGES[number];

function getDefaultLanguage(): SupportedLanguage {
  if (SUPPORTED_LANGUAGES.includes(deviceLanguage as SupportedLanguage)) {
    return deviceLanguage as SupportedLanguage;
  }
  return 'en'; // Fallback to English
}

i18n
  .use(initReactI18next)
  .init({
    resources: {
      en: { translation: en },
      es: { translation: es },
      fr: { translation: fr },
      ar: { translation: ar },
      ja: { translation: ja },
    },
    lng: getDefaultLanguage(),
    fallbackLng: 'en',
    
    interpolation: {
      escapeValue: false, // React already escapes
    },
    
    // React-specific options
    react: {
      useSuspense: false, // Disable Suspense for React Native
    },
    
    // Key separator for nested translations
    keySeparator: '.',
    
    // Namespace separator
    nsSeparator: ':',
    
    // Default namespace
    defaultNS: 'translation',
    
    // Enable debug in development
    debug: __DEV__,
  });

export default i18n;
export { SUPPORTED_LANGUAGES, type SupportedLanguage };
```

### 10.3 Translation File Structure

```
src/i18n/
  index.ts              — i18n configuration
  locales/
    en.json             — English translations
    es.json             — Spanish translations
    fr.json             — French translations
    ar.json             — Arabic translations
    ja.json             — Japanese translations
```

```json
// src/i18n/locales/en.json
{
  "common": {
    "loading": "Loading...",
    "error": "Something went wrong",
    "retry": "Try Again",
    "cancel": "Cancel",
    "save": "Save",
    "delete": "Delete",
    "done": "Done",
    "back": "Back",
    "next": "Next",
    "search": "Search",
    "noResults": "No results found"
  },
  "auth": {
    "login": "Log In",
    "signup": "Sign Up",
    "logout": "Log Out",
    "email": "Email",
    "password": "Password",
    "forgotPassword": "Forgot Password?",
    "welcomeBack": "Welcome back, {{name}}!",
    "loginError": "Invalid email or password. Please try again."
  },
  "home": {
    "greeting": "Good {{timeOfDay}}, {{name}}",
    "subtitle": "Here's what's happening today",
    "recentActivity": "Recent Activity",
    "viewAll": "View All"
  },
  "profile": {
    "title": "Profile",
    "editProfile": "Edit Profile",
    "settings": "Settings",
    "memberSince": "Member since {{date}}",
    "postsCount_one": "{{count}} post",
    "postsCount_other": "{{count}} posts"
  },
  "notifications": {
    "title": "Notifications",
    "empty": "No notifications yet",
    "markAllRead": "Mark All as Read",
    "newMessage": "{{name}} sent you a message",
    "newFollower": "{{name}} started following you",
    "itemCount_zero": "No items",
    "itemCount_one": "{{count}} item",
    "itemCount_other": "{{count}} items"
  },
  "settings": {
    "title": "Settings",
    "language": "Language",
    "theme": "Theme",
    "notifications": "Notifications",
    "privacy": "Privacy",
    "about": "About",
    "version": "Version {{version}}"
  },
  "errors": {
    "network": "Network error. Please check your connection.",
    "server": "Server error. Please try again later.",
    "timeout": "Request timed out. Please try again.",
    "notFound": "The requested resource was not found."
  }
}
```

```json
// src/i18n/locales/es.json
{
  "common": {
    "loading": "Cargando...",
    "error": "Algo sali\u00f3 mal",
    "retry": "Intentar de nuevo",
    "cancel": "Cancelar",
    "save": "Guardar",
    "delete": "Eliminar",
    "done": "Listo",
    "back": "Atr\u00e1s",
    "next": "Siguiente",
    "search": "Buscar",
    "noResults": "No se encontraron resultados"
  },
  "auth": {
    "login": "Iniciar sesi\u00f3n",
    "signup": "Registrarse",
    "logout": "Cerrar sesi\u00f3n",
    "email": "Correo electr\u00f3nico",
    "password": "Contrase\u00f1a",
    "forgotPassword": "\u00bfOlvidaste tu contrase\u00f1a?",
    "welcomeBack": "\u00a1Bienvenido de vuelta, {{name}}!",
    "loginError": "Correo o contrase\u00f1a inv\u00e1lidos. Por favor intenta de nuevo."
  },
  "home": {
    "greeting": "{{timeOfDay}}, {{name}}",
    "subtitle": "Esto es lo que pasa hoy",
    "recentActivity": "Actividad reciente",
    "viewAll": "Ver todo"
  },
  "profile": {
    "title": "Perfil",
    "editProfile": "Editar perfil",
    "settings": "Configuraci\u00f3n",
    "memberSince": "Miembro desde {{date}}",
    "postsCount_one": "{{count}} publicaci\u00f3n",
    "postsCount_other": "{{count}} publicaciones"
  },
  "notifications": {
    "title": "Notificaciones",
    "empty": "A\u00fan no hay notificaciones",
    "markAllRead": "Marcar todo como le\u00eddo",
    "newMessage": "{{name}} te envi\u00f3 un mensaje",
    "newFollower": "{{name}} comenz\u00f3 a seguirte",
    "itemCount_zero": "Sin elementos",
    "itemCount_one": "{{count}} elemento",
    "itemCount_other": "{{count}} elementos"
  },
  "settings": {
    "title": "Configuraci\u00f3n",
    "language": "Idioma",
    "theme": "Tema",
    "notifications": "Notificaciones",
    "privacy": "Privacidad",
    "about": "Acerca de",
    "version": "Versi\u00f3n {{version}}"
  }
}
```

### 10.4 Using Translations in Components

```tsx
import { useTranslation } from 'react-i18next';
import { View, Text, Pressable, StyleSheet } from 'react-native';

function HomeScreen() {
  const { t } = useTranslation();

  const timeOfDay = getTimeOfDay(); // "morning" | "afternoon" | "evening"
  const user = useCurrentUser();

  return (
    <View style={styles.container}>
      {/* Simple translation */}
      <Text style={styles.title}>
        {t('home.greeting', { timeOfDay, name: user.name })}
      </Text>
      
      <Text style={styles.subtitle}>
        {t('home.subtitle')}
      </Text>

      {/* Pluralization */}
      <Text style={styles.postCount}>
        {t('profile.postsCount', { count: user.postCount })}
        {/* English: "1 post" or "5 posts" */}
        {/* Spanish: "1 publicacion" or "5 publicaciones" */}
      </Text>

      {/* Section header */}
      <Text style={styles.sectionHeader}>
        {t('home.recentActivity')}
      </Text>

      <Pressable style={styles.viewAllButton}>
        <Text style={styles.viewAllText}>{t('home.viewAll')}</Text>
      </Pressable>
    </View>
  );
}

function getTimeOfDay(): string {
  const hour = new Date().getHours();
  if (hour < 12) return 'morning';
  if (hour < 17) return 'afternoon';
  return 'evening';
}

// Error handling with translations
function ErrorBoundary({ error }: { error: Error }) {
  const { t } = useTranslation();

  const errorMessage = error.message.includes('network')
    ? t('errors.network')
    : error.message.includes('timeout')
    ? t('errors.timeout')
    : t('common.error');

  return (
    <View style={styles.errorContainer}>
      <Text style={styles.errorText}>{errorMessage}</Text>
      <Pressable style={styles.retryButton}>
        <Text style={styles.retryText}>{t('common.retry')}</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 28, fontWeight: '700', color: '#1A1A1A' },
  subtitle: { fontSize: 16, color: '#666', marginTop: 4 },
  postCount: { fontSize: 14, color: '#999', marginTop: 8 },
  sectionHeader: { fontSize: 20, fontWeight: '700', color: '#1A1A1A', marginTop: 24 },
  viewAllButton: { padding: 8 },
  viewAllText: { fontSize: 14, color: '#007AFF', fontWeight: '600' },
  errorContainer: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 32 },
  errorText: { fontSize: 16, color: '#666', textAlign: 'center', marginBottom: 16 },
  retryButton: { padding: 16, backgroundColor: '#007AFF', borderRadius: 8 },
  retryText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

### 10.5 Namespaces for Large Apps

For large apps, split translations into namespaces to keep files manageable:

```
src/i18n/locales/
  en/
    common.json       — Shared strings (buttons, labels, errors)
    auth.json         — Authentication screens
    home.json         — Home screen
    profile.json      — Profile screens
    settings.json     — Settings screens
    notifications.json — Notification strings
```

```tsx
// i18n configuration with namespaces
i18n.init({
  resources: {
    en: {
      common: require('./locales/en/common.json'),
      auth: require('./locales/en/auth.json'),
      home: require('./locales/en/home.json'),
      profile: require('./locales/en/profile.json'),
    },
    es: {
      common: require('./locales/es/common.json'),
      auth: require('./locales/es/auth.json'),
      home: require('./locales/es/home.json'),
      profile: require('./locales/es/profile.json'),
    },
  },
  defaultNS: 'common',
  fallbackNS: 'common',
});

// Using namespaces in components
function AuthScreen() {
  const { t } = useTranslation('auth'); // Use 'auth' namespace
  
  return (
    <View>
      <Text>{t('login')}</Text>           {/* auth:login */}
      <Text>{t('common:cancel')}</Text>    {/* Explicit namespace */}
    </View>
  );
}
```

### 10.6 Lazy Loading Translations

For apps with many languages, loading all translations upfront wastes memory. Lazy-load them:

```bash
npm install i18next-resources-to-backend
```

```tsx
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import resourcesToBackend from 'i18next-resources-to-backend';
import { getLocales } from 'expo-localization';

i18n
  .use(initReactI18next)
  .use(
    resourcesToBackend((language: string, namespace: string) => {
      // Dynamic import — only loads the language when needed
      return import(`./locales/${language}/${namespace}.json`);
    })
  )
  .init({
    lng: getLocales()[0]?.languageCode ?? 'en',
    fallbackLng: 'en',
    defaultNS: 'common',
    ns: ['common'], // Only load 'common' namespace initially
    react: { useSuspense: false },
  });
```

---

## 11. RTL SUPPORT

### 11.1 Right-to-Left Languages

Arabic, Hebrew, Urdu, Farsi, and other RTL languages read from right to left. When your app switches to an RTL language, the entire layout should mirror:

- Text aligns to the right
- Flex row items reverse (start is now the right side)
- Padding/margin start/end flip
- Icons that imply direction (arrows, chevrons) flip
- Back buttons move to the right
- Tab bars stay in the same position (LTR) but the selected indicator may need adjustment

### 11.2 Enabling RTL

```tsx
import { I18nManager, Platform } from 'react-native';
import * as Updates from 'expo-updates';
import i18n from './i18n';

const RTL_LANGUAGES = ['ar', 'he', 'ur', 'fa'];

async function setLanguage(languageCode: string) {
  const isRTL = RTL_LANGUAGES.includes(languageCode);
  
  // Change the language
  await i18n.changeLanguage(languageCode);
  
  // Update RTL setting
  if (I18nManager.isRTL !== isRTL) {
    I18nManager.forceRTL(isRTL);
    I18nManager.allowRTL(isRTL);
    
    // RTL changes require a restart to take effect
    if (Platform.OS !== 'web') {
      // Trigger an app restart
      await Updates.reloadAsync();
    }
  }
}
```

**The restart problem:** `I18nManager.forceRTL()` only takes effect after the JavaScript bundle restarts. This means the user sees LTR layout until the app restarts. There's no way around this in React Native currently. The best UX is to show a "Restarting..." message and call `Updates.reloadAsync()`.

### 11.3 RTL-Aware Styles

```tsx
import { StyleSheet, I18nManager, Platform } from 'react-native';

// Use 'start' and 'end' instead of 'left' and 'right'
const styles = StyleSheet.create({
  // GOOD: RTL-aware
  container: {
    paddingStart: 16,    // Left in LTR, Right in RTL
    paddingEnd: 8,       // Right in LTR, Left in RTL
    marginStart: 12,
    marginEnd: 12,
  },
  
  // GOOD: Flex direction respects RTL automatically
  row: {
    flexDirection: 'row', // Items flow left-to-right in LTR, right-to-left in RTL
    alignItems: 'center',
  },
  
  // GOOD: Text alignment follows RTL
  text: {
    textAlign: 'left', // In RTL, React Native automatically flips this to 'right'
    writingDirection: 'auto', // Follows the language direction
  },
  
  // BAD: Hard-coded left/right doesn't flip in RTL
  badContainer: {
    paddingLeft: 16,   // Always left, even in RTL — WRONG
    paddingRight: 8,   // Always right, even in RTL — WRONG
  },
});

// For directional icons (arrows, chevrons)
function DirectionalIcon({ direction }: { direction: 'forward' | 'back' }) {
  const isRTL = I18nManager.isRTL;
  
  // 'forward' means "toward the next screen" — right in LTR, left in RTL
  const rotation = direction === 'forward'
    ? isRTL ? '180deg' : '0deg'
    : isRTL ? '0deg' : '180deg';

  return (
    <ChevronRightIcon style={{ transform: [{ rotate: rotation }] }} />
  );
}

// Helper for RTL-conditional styling
function rtlStyle<T>(ltrValue: T, rtlValue: T): T {
  return I18nManager.isRTL ? rtlValue : ltrValue;
}

// Usage
const dynamicStyles = StyleSheet.create({
  arrow: {
    transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }],
  },
  absolutePositioned: {
    position: 'absolute',
    [I18nManager.isRTL ? 'left' : 'right']: 16, // Pin to trailing edge
  },
});
```

### 11.4 Testing RTL

```tsx
// Force RTL for testing (add to your dev tools)
function RTLToggle() {
  const [isRTL, setIsRTL] = useState(I18nManager.isRTL);

  const toggleRTL = async () => {
    const newValue = !isRTL;
    I18nManager.forceRTL(newValue);
    I18nManager.allowRTL(newValue);
    setIsRTL(newValue);
    
    // Restart to apply
    await Updates.reloadAsync();
  };

  return (
    <Pressable onPress={toggleRTL} style={styles.devButton}>
      <Text>RTL: {isRTL ? 'ON' : 'OFF'}</Text>
    </Pressable>
  );
}
```

**RTL testing checklist:**
- [ ] All text aligns correctly (right-aligned for RTL)
- [ ] Flex rows display in the correct order
- [ ] Navigation back button appears on the correct side
- [ ] Directional icons (arrows, chevrons) are flipped
- [ ] Absolute-positioned elements are on the correct side
- [ ] Padding/margin start/end are applied correctly
- [ ] Swipe gestures work in the expected direction
- [ ] No text is truncated or overflowing
- [ ] Numbers remain LTR (Arabic numerals or Eastern Arabic numerals)

---

## 12. PLURALIZATION: ICU MESSAGE FORMAT

### 12.1 The Problem

English has two plural forms: singular and plural (1 item, 2 items). But many languages have more:

- **Arabic:** 6 plural forms (zero, one, two, few, many, other)
- **Russian:** 3 forms (one, few, many)
- **Japanese:** 1 form (no plural distinction)
- **French:** 2 forms (one, other — but "0" uses plural, unlike English)
- **Polish:** 3 forms (one, few, many)

If you're concatenating strings like `${count} item${count === 1 ? '' : 's'}`, you're going to have a bad time when translating.

### 12.2 i18next Built-in Pluralization

i18next handles pluralization based on the language's rules:

```json
// en.json — English (two forms)
{
  "cart": {
    "itemCount_one": "{{count}} item in your cart",
    "itemCount_other": "{{count}} items in your cart"
  },
  "followers": {
    "count_zero": "No followers yet",
    "count_one": "{{count}} follower",
    "count_other": "{{count}} followers"
  }
}

// ar.json — Arabic (six forms)
{
  "cart": {
    "itemCount_zero": "لا عناصر في سلتك",
    "itemCount_one": "عنصر واحد في سلتك",
    "itemCount_two": "عنصران في سلتك",
    "itemCount_few": "{{count}} عناصر في سلتك",
    "itemCount_many": "{{count}} عنصرًا في سلتك",
    "itemCount_other": "{{count}} عنصر في سلتك"
  }
}

// ja.json — Japanese (one form, no plural distinction)
{
  "cart": {
    "itemCount_other": "カートに{{count}}個の商品"
  }
}

// ru.json — Russian (three forms)
{
  "cart": {
    "itemCount_one": "{{count}} товар в корзине",
    "itemCount_few": "{{count}} товара в корзине",
    "itemCount_many": "{{count}} товаров в корзине"
  }
}
```

```tsx
// Usage — i18next automatically picks the correct form
const { t } = useTranslation();

t('cart.itemCount', { count: 0 });  // English: "0 items in your cart"
                                     // Arabic: "لا عناصر في سلتك"
t('cart.itemCount', { count: 1 });  // English: "1 item in your cart"
                                     // Arabic: "عنصر واحد في سلتك"
t('cart.itemCount', { count: 2 });  // English: "2 items in your cart"
                                     // Arabic: "عنصران في سلتك"
t('cart.itemCount', { count: 5 });  // English: "5 items in your cart"
                                     // Arabic: "5 عناصر في سلتك"
```

### 12.3 ICU Message Format for Complex Cases

For more complex translations (gender, selections, nested plurals), use the ICU MessageFormat:

```bash
npm install i18next-icu
```

```tsx
import i18n from 'i18next';
import ICU from 'i18next-icu';

i18n
  .use(ICU)
  .use(initReactI18next)
  .init({
    // ... same config as before
  });
```

```json
// en.json with ICU format
{
  "greeting": "{gender, select, male {He} female {She} other {They}} liked your post.",
  
  "notification": "{name} sent you {count, plural, one {a message} other {# messages}}.",
  
  "lastSeen": "{name} was last seen {when, select, today {today at {time}} yesterday {yesterday} other {{date}}}.",
  
  "orderStatus": "Your order has {count, plural, =0 {no items} one {# item} other {# items}} and is {status, select, pending {being prepared} shipped {on its way} delivered {delivered} other {processing}}."
}
```

```tsx
// Usage
t('greeting', { gender: 'female' });
// "She liked your post."

t('notification', { name: 'Sarah', count: 3 });
// "Sarah sent you 3 messages."

t('lastSeen', { name: 'John', when: 'today', time: '3:42 PM' });
// "John was last seen today at 3:42 PM."
```

---

## 13. DATE/NUMBER FORMATTING

### 13.1 Using the Intl API

React Native supports the `Intl` API for locale-aware formatting on both platforms. Combined with expo-localization for locale detection, you get proper formatting everywhere.

```tsx
import { getLocales } from 'expo-localization';

// Get the user's locale
const locale = getLocales()[0]?.languageTag ?? 'en-US';
// "en-US", "fr-FR", "ar-SA", "ja-JP", etc.

// Date formatting
function formatDate(date: Date, style: 'short' | 'medium' | 'long' = 'medium'): string {
  const options: Intl.DateTimeFormatOptions = {
    short: { month: 'numeric', day: 'numeric', year: '2-digit' },
    medium: { month: 'short', day: 'numeric', year: 'numeric' },
    long: { month: 'long', day: 'numeric', year: 'numeric', weekday: 'long' },
  }[style];

  return new Intl.DateTimeFormat(locale, options).format(date);
}

// Results by locale:
// en-US: "Apr 7, 2026" | "4/7/26" | "Tuesday, April 7, 2026"
// fr-FR: "7 avr. 2026" | "07/04/26" | "mardi 7 avril 2026"
// ja-JP: "2026年4月7日" | "2026/04/07" | "2026年4月7日火曜日"
// ar-SA: "٧ أبريل ٢٠٢٦" | ...

// Time formatting
function formatTime(date: Date): string {
  return new Intl.DateTimeFormat(locale, {
    hour: 'numeric',
    minute: 'numeric',
    hour12: undefined, // Use the locale's default (12h for US, 24h for most of Europe)
  }).format(date);
}

// Number formatting
function formatNumber(value: number): string {
  return new Intl.NumberFormat(locale).format(value);
}
// en-US: "1,234,567.89"
// de-DE: "1.234.567,89"
// fr-FR: "1 234 567,89"

// Currency formatting
function formatCurrency(amount: number, currency: string = 'USD'): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
}
// en-US: "$1,234.56"
// de-DE: "1.234,56 $"
// ja-JP: "$1,234.56" or "￥1,234" for JPY

// Percentage formatting
function formatPercent(value: number): string {
  return new Intl.NumberFormat(locale, {
    style: 'percent',
    minimumFractionDigits: 0,
    maximumFractionDigits: 1,
  }).format(value);
}
// 0.156 → "15.6%" (en-US) | "15,6 %" (fr-FR)

// Relative time formatting
function formatRelativeTime(date: Date): string {
  const now = new Date();
  const diffMs = date.getTime() - now.getTime();
  const diffSeconds = Math.round(diffMs / 1000);
  const diffMinutes = Math.round(diffMs / 60000);
  const diffHours = Math.round(diffMs / 3600000);
  const diffDays = Math.round(diffMs / 86400000);

  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });

  if (Math.abs(diffSeconds) < 60) return rtf.format(diffSeconds, 'second');
  if (Math.abs(diffMinutes) < 60) return rtf.format(diffMinutes, 'minute');
  if (Math.abs(diffHours) < 24) return rtf.format(diffHours, 'hour');
  if (Math.abs(diffDays) < 30) return rtf.format(diffDays, 'day');
  
  // Fall back to absolute date for older dates
  return formatDate(date);
}
// en-US: "2 hours ago", "yesterday", "in 3 days"
// es-ES: "hace 2 horas", "ayer", "dentro de 3 dias"
```

### 13.2 A Complete Formatting Utility

```tsx
// src/utils/format.ts
import { getLocales, getCalendars } from 'expo-localization';

class Formatter {
  private locale: string;
  private timeZone: string;
  private uses24HourClock: boolean;

  constructor() {
    const localeInfo = getLocales()[0];
    const calendarInfo = getCalendars()[0];
    
    this.locale = localeInfo?.languageTag ?? 'en-US';
    this.timeZone = calendarInfo?.timeZone ?? 'UTC';
    this.uses24HourClock = localeInfo?.uses24HourClock ?? false;
  }

  date(date: Date | string | number, style: 'short' | 'medium' | 'long' = 'medium'): string {
    const d = new Date(date);
    const options: Record<string, Intl.DateTimeFormatOptions> = {
      short: { month: 'numeric', day: 'numeric' },
      medium: { month: 'short', day: 'numeric', year: 'numeric' },
      long: { month: 'long', day: 'numeric', year: 'numeric', weekday: 'long' },
    };
    return new Intl.DateTimeFormat(this.locale, {
      ...options[style],
      timeZone: this.timeZone,
    }).format(d);
  }

  time(date: Date | string | number): string {
    const d = new Date(date);
    return new Intl.DateTimeFormat(this.locale, {
      hour: 'numeric',
      minute: 'numeric',
      hour12: !this.uses24HourClock,
      timeZone: this.timeZone,
    }).format(d);
  }

  dateTime(date: Date | string | number): string {
    return `${this.date(date)} ${this.time(date)}`;
  }

  number(value: number, decimals?: number): string {
    return new Intl.NumberFormat(this.locale, {
      minimumFractionDigits: decimals,
      maximumFractionDigits: decimals,
    }).format(value);
  }

  currency(amount: number, currency: string = 'USD'): string {
    return new Intl.NumberFormat(this.locale, {
      style: 'currency',
      currency,
    }).format(amount);
  }

  percent(value: number): string {
    return new Intl.NumberFormat(this.locale, {
      style: 'percent',
      maximumFractionDigits: 1,
    }).format(value);
  }

  relativeTime(date: Date | string | number): string {
    const d = new Date(date);
    const now = new Date();
    const diffMs = d.getTime() - now.getTime();
    const absDiff = Math.abs(diffMs);
    
    const rtf = new Intl.RelativeTimeFormat(this.locale, { numeric: 'auto' });

    if (absDiff < 60_000) {
      return rtf.format(Math.round(diffMs / 1000), 'second');
    }
    if (absDiff < 3_600_000) {
      return rtf.format(Math.round(diffMs / 60_000), 'minute');
    }
    if (absDiff < 86_400_000) {
      return rtf.format(Math.round(diffMs / 3_600_000), 'hour');
    }
    if (absDiff < 2_592_000_000) {
      return rtf.format(Math.round(diffMs / 86_400_000), 'day');
    }
    
    return this.date(d);
  }

  compact(value: number): string {
    return new Intl.NumberFormat(this.locale, {
      notation: 'compact',
      compactDisplay: 'short',
    }).format(value);
  }
  // 1234 → "1.2K" (en-US) | "1,2 k" (fr-FR) | "1234" (ja-JP)
}

// Singleton instance
export const format = new Formatter();

// Usage:
// format.date(new Date())          → "Apr 7, 2026"
// format.time(new Date())          → "3:42 PM"
// format.currency(29.99)           → "$29.99"
// format.compact(1_500_000)        → "1.5M"
// format.relativeTime(yesterday)   → "yesterday"
```

---

## 14. DYNAMIC LANGUAGE SWITCHING

### 14.1 Changing Language at Runtime

Users want to change the language from within your app, not from the device settings. Here's how to do it properly:

```tsx
// src/i18n/useLanguage.ts
import { useState, useCallback, useEffect } from 'react';
import { I18nManager } from 'react-native';
import i18n from './index';
import * as Updates from 'expo-updates';
import AsyncStorage from '@react-native-async-storage/async-storage';

const LANGUAGE_KEY = 'user_language';
const RTL_LANGUAGES = ['ar', 'he', 'ur', 'fa'];

interface LanguageOption {
  code: string;
  name: string;         // In English
  nativeName: string;   // In the language itself
  isRTL: boolean;
}

const LANGUAGES: LanguageOption[] = [
  { code: 'en', name: 'English', nativeName: 'English', isRTL: false },
  { code: 'es', name: 'Spanish', nativeName: 'Espa\u00f1ol', isRTL: false },
  { code: 'fr', name: 'French', nativeName: 'Fran\u00e7ais', isRTL: false },
  { code: 'de', name: 'German', nativeName: 'Deutsch', isRTL: false },
  { code: 'ja', name: 'Japanese', nativeName: '\u65e5\u672c\u8a9e', isRTL: false },
  { code: 'ar', name: 'Arabic', nativeName: '\u0627\u0644\u0639\u0631\u0628\u064a\u0629', isRTL: true },
  { code: 'he', name: 'Hebrew', nativeName: '\u05e2\u05d1\u05e8\u05d9\u05ea', isRTL: true },
  { code: 'pt', name: 'Portuguese', nativeName: 'Portugu\u00eas', isRTL: false },
  { code: 'ko', name: 'Korean', nativeName: '\ud55c\uad6d\uc5b4', isRTL: false },
  { code: 'zh', name: 'Chinese', nativeName: '\u4e2d\u6587', isRTL: false },
];

function useLanguage() {
  const [currentLanguage, setCurrentLanguage] = useState(i18n.language);
  const [isChanging, setIsChanging] = useState(false);

  // Load persisted language on mount
  useEffect(() => {
    AsyncStorage.getItem(LANGUAGE_KEY).then(savedLanguage => {
      if (savedLanguage && savedLanguage !== i18n.language) {
        i18n.changeLanguage(savedLanguage);
        setCurrentLanguage(savedLanguage);
      }
    });
  }, []);

  const changeLanguage = useCallback(async (languageCode: string) => {
    setIsChanging(true);

    try {
      // Persist the choice
      await AsyncStorage.setItem(LANGUAGE_KEY, languageCode);

      // Change i18next language
      await i18n.changeLanguage(languageCode);
      setCurrentLanguage(languageCode);

      // Handle RTL change
      const isRTL = RTL_LANGUAGES.includes(languageCode);
      if (I18nManager.isRTL !== isRTL) {
        I18nManager.forceRTL(isRTL);
        I18nManager.allowRTL(isRTL);
        
        // RTL change requires app restart
        await Updates.reloadAsync();
        return; // App will restart
      }
    } catch (error) {
      console.error('Failed to change language:', error);
    } finally {
      setIsChanging(false);
    }
  }, []);

  return {
    currentLanguage,
    changeLanguage,
    isChanging,
    languages: LANGUAGES,
  };
}

export { useLanguage, LANGUAGES, type LanguageOption };
```

### 14.2 Language Selection Screen

```tsx
import { View, Text, FlatList, Pressable, StyleSheet, ActivityIndicator } from 'react-native';
import { useTranslation } from 'react-i18next';
import { useLanguage, LanguageOption } from '../i18n/useLanguage';

function LanguageSelectionScreen() {
  const { t } = useTranslation();
  const { currentLanguage, changeLanguage, isChanging, languages } = useLanguage();

  const renderLanguage = ({ item }: { item: LanguageOption }) => {
    const isSelected = item.code === currentLanguage;

    return (
      <Pressable
        style={[styles.languageRow, isSelected && styles.languageRowSelected]}
        onPress={() => changeLanguage(item.code)}
        disabled={isChanging}
        accessibilityLabel={`${item.name}, ${item.nativeName}`}
        accessibilityRole="radio"
        accessibilityState={{ selected: isSelected }}
      >
        <View style={styles.languageInfo}>
          <Text style={[
            styles.languageName,
            isSelected && styles.languageNameSelected,
          ]}>
            {item.nativeName}
          </Text>
          <Text style={styles.languageSubtext}>{item.name}</Text>
        </View>

        {isSelected && (
          <Text style={styles.checkmark}>✓</Text>
        )}

        {item.isRTL && (
          <Text style={styles.rtlBadge}>RTL</Text>
        )}
      </Pressable>
    );
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{t('settings.language')}</Text>
      
      {isChanging && (
        <View style={styles.loadingOverlay}>
          <ActivityIndicator size="large" />
          <Text style={styles.loadingText}>Changing language...</Text>
        </View>
      )}

      <FlatList
        data={languages}
        keyExtractor={item => item.code}
        renderItem={renderLanguage}
        ItemSeparatorComponent={() => <View style={styles.separator} />}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: 'white' },
  title: {
    fontSize: 28,
    fontWeight: '700',
    color: '#1A1A1A',
    padding: 16,
    paddingBottom: 8,
  },
  languageRow: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 16,
  },
  languageRowSelected: {
    backgroundColor: '#F0F7FF',
  },
  languageInfo: { flex: 1 },
  languageName: {
    fontSize: 17,
    fontWeight: '500',
    color: '#1A1A1A',
  },
  languageNameSelected: {
    fontWeight: '700',
    color: '#007AFF',
  },
  languageSubtext: {
    fontSize: 14,
    color: '#999',
    marginTop: 2,
  },
  checkmark: {
    fontSize: 18,
    color: '#007AFF',
    fontWeight: '700',
    marginLeft: 12,
  },
  rtlBadge: {
    fontSize: 10,
    fontWeight: '700',
    color: '#666',
    backgroundColor: '#F0F0F0',
    paddingHorizontal: 6,
    paddingVertical: 2,
    borderRadius: 4,
    marginLeft: 8,
    overflow: 'hidden',
  },
  separator: {
    height: 1,
    backgroundColor: '#F0F0F0',
    marginLeft: 16,
  },
  loadingOverlay: {
    ...StyleSheet.absoluteFillObject,
    backgroundColor: 'rgba(255,255,255,0.9)',
    justifyContent: 'center',
    alignItems: 'center',
    zIndex: 1,
  },
  loadingText: {
    marginTop: 12,
    fontSize: 16,
    color: '#666',
  },
});
```

### 14.3 Initializing Language on App Start

```tsx
// app/_layout.tsx
import '../src/i18n'; // Initialize i18n before rendering
import { Stack } from 'expo-router';
import { useEffect, useState } from 'react';
import AsyncStorage from '@react-native-async-storage/async-storage';
import i18n from '../src/i18n';

export default function RootLayout() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    async function loadLanguage() {
      const savedLanguage = await AsyncStorage.getItem('user_language');
      if (savedLanguage) {
        await i18n.changeLanguage(savedLanguage);
      }
      setIsReady(true);
    }
    loadLanguage();
  }, []);

  if (!isReady) {
    return null; // Or a splash screen
  }

  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
    </Stack>
  );
}
```

---

## 15. I18N + A11Y: WHERE THEY INTERSECT

Accessibility labels need to be translated too. This is something teams often miss — they add `accessibilityLabel` in English and forget to translate them.

```tsx
import { useTranslation } from 'react-i18next';
import { Pressable, Text, View } from 'react-native';

function TranslatedAccessibleButton() {
  const { t } = useTranslation();

  return (
    <Pressable
      accessibilityLabel={t('common.delete')}  // Translated!
      accessibilityRole="button"
      accessibilityHint={t('accessibility.deleteHint')} // Also translated!
      onPress={handleDelete}
    >
      <TrashIcon />
    </Pressable>
  );
}

// Translation file should include accessibility-specific strings
// en.json
{
  "accessibility": {
    "deleteHint": "Removes this item permanently",
    "closeModal": "Close this dialog",
    "goBack": "Go to the previous screen",
    "openMenu": "Opens the navigation menu",
    "loading": "Content is loading, please wait",
    "searchResults": "{{count}} search results found"
  }
}

// es.json
{
  "accessibility": {
    "deleteHint": "Elimina este elemento permanentemente",
    "closeModal": "Cerrar este di\u00e1logo",
    "goBack": "Ir a la pantalla anterior",
    "openMenu": "Abre el men\u00fa de navegaci\u00f3n",
    "loading": "El contenido est\u00e1 cargando, por favor espere",
    "searchResults": "{{count}} resultados de b\u00fasqueda encontrados"
  }
}
```

---

## 16. PUTTING IT ALL TOGETHER: A FULLY ACCESSIBLE, INTERNATIONALIZED SETTINGS SCREEN

```tsx
import { View, Text, ScrollView, Pressable, Switch, StyleSheet, Linking } from 'react-native';
import { useTranslation } from 'react-i18next';
import { useLanguage } from '../i18n/useLanguage';
import { useState, useCallback } from 'react';
import Constants from 'expo-constants';
import { format } from '../utils/format';

function SettingsScreen() {
  const { t } = useTranslation();
  const { currentLanguage, languages } = useLanguage();
  const [notifications, setNotifications] = useState(true);
  const [darkMode, setDarkMode] = useState(false);

  const currentLanguageLabel = languages.find(l => l.code === currentLanguage)?.nativeName ?? 'English';
  const appVersion = Constants.expoConfig?.version ?? '1.0.0';

  return (
    <ScrollView 
      style={styles.container}
      contentContainerStyle={styles.content}
    >
      {/* Screen title */}
      <Text
        style={styles.screenTitle}
        accessibilityRole="header"
      >
        {t('settings.title')}
      </Text>

      {/* Preferences section */}
      <Text style={styles.sectionHeader} accessibilityRole="header">
        {t('settings.preferences')}
      </Text>

      {/* Language row — navigates to language picker */}
      <Pressable
        style={styles.settingRow}
        onPress={() => {/* navigate to language selection */}}
        accessibilityLabel={`${t('settings.language')}: ${currentLanguageLabel}`}
        accessibilityRole="button"
        accessibilityHint={t('accessibility.openLanguageSelection')}
      >
        <Text style={styles.settingLabel}>{t('settings.language')}</Text>
        <View style={styles.settingValue}>
          <Text style={styles.settingValueText}>{currentLanguageLabel}</Text>
          <Text style={styles.chevron}>›</Text>
        </View>
      </Pressable>

      {/* Notifications toggle */}
      <View
        style={styles.settingRow}
        accessible={true}
        accessibilityLabel={t('settings.notifications')}
        accessibilityRole="switch"
        accessibilityState={{ checked: notifications }}
      >
        <Text style={styles.settingLabel}>{t('settings.notifications')}</Text>
        <Switch
          value={notifications}
          onValueChange={setNotifications}
          // Don't add accessibility props to the Switch itself 
          // since the parent View handles it
        />
      </View>

      {/* Dark mode toggle */}
      <View
        style={styles.settingRow}
        accessible={true}
        accessibilityLabel={t('settings.theme')}
        accessibilityRole="switch"
        accessibilityState={{ checked: darkMode }}
      >
        <Text style={styles.settingLabel}>{t('settings.theme')}</Text>
        <Switch
          value={darkMode}
          onValueChange={setDarkMode}
        />
      </View>

      {/* About section */}
      <Text style={styles.sectionHeader} accessibilityRole="header">
        {t('settings.about')}
      </Text>

      <View style={styles.settingRow}>
        <Text style={styles.settingLabel}>{t('settings.version', { version: appVersion })}</Text>
      </View>

      {/* Privacy & Terms */}
      <Pressable
        style={styles.settingRow}
        onPress={() => Linking.openURL('https://myapp.com/privacy')}
        accessibilityLabel={t('settings.privacy')}
        accessibilityRole="link"
      >
        <Text style={styles.settingLabel}>{t('settings.privacy')}</Text>
        <Text style={styles.chevron}>›</Text>
      </Pressable>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#F5F5F5' },
  content: { paddingBottom: 40 },
  screenTitle: {
    fontSize: 34,
    fontWeight: '700',
    color: '#1A1A1A',
    paddingHorizontal: 16,
    paddingTop: 16,
    paddingBottom: 8,
  },
  sectionHeader: {
    fontSize: 13,
    fontWeight: '600',
    color: '#666',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
    paddingHorizontal: 16,
    paddingTop: 24,
    paddingBottom: 8,
  },
  settingRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    backgroundColor: 'white',
    borderBottomWidth: StyleSheet.hairlineWidth,
    borderBottomColor: '#E5E5E5',
  },
  settingLabel: {
    fontSize: 17,
    color: '#1A1A1A',
  },
  settingValue: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 4,
  },
  settingValueText: {
    fontSize: 17,
    color: '#999',
  },
  chevron: {
    fontSize: 20,
    color: '#CCC',
    marginStart: 4, // RTL-aware!
  },
});
```

---

## 17. COMPREHENSIVE CHECKLISTS

### Permissions Checklist
- [ ] Every permission has a specific, descriptive usage string
- [ ] Pre-permission dialogs explain value before system prompts
- [ ] Denied permissions show "Open Settings" option
- [ ] Permissions are requested at the moment of relevance, not on launch
- [ ] `canAskAgain` is checked before calling `requestPermissionsAsync`
- [ ] No unnecessary permissions declared in app.json
- [ ] `blockedPermissions` used to prevent library-injected permissions (Android)
- [ ] Background location has a legitimate, user-visible reason
- [ ] ATT (iOS) is requested after the user has experienced app value
- [ ] All permission flows tested on both iOS and Android

### Accessibility Checklist
- [ ] Every interactive element has `accessibilityLabel`
- [ ] Every interactive element has `accessibilityRole`
- [ ] `accessibilityState` set on toggles, checkboxes, and expandable elements
- [ ] Decorative elements hidden with `importantForAccessibility="no"`
- [ ] Dynamic content changes announced via `accessibilityLiveRegion` or `announceForAccessibility`
- [ ] Modals trap focus (`accessibilityViewIsModal`)
- [ ] Focus management after navigation and content updates
- [ ] Touch targets at least 44x44 points
- [ ] Color contrast meets WCAG AA (4.5:1 for text)
- [ ] Tested with VoiceOver on iOS and TalkBack on Android
- [ ] eslint-plugin-react-native-a11y configured
- [ ] Respects system `reduceMotion` and `boldText` settings

### i18n Checklist
- [ ] react-i18next + expo-localization configured
- [ ] Translation files organized by namespace
- [ ] Pluralization uses i18next plural forms (not manual logic)
- [ ] Date/number formatting uses `Intl` with detected locale
- [ ] RTL supported via `paddingStart/End` instead of `paddingLeft/Right`
- [ ] RTL tested with Arabic or Hebrew
- [ ] Directional icons flip in RTL
- [ ] User language preference persisted with AsyncStorage
- [ ] Language switchable at runtime without losing state
- [ ] Accessibility labels are translated
- [ ] All user-visible strings are in translation files (no hard-coded strings)
- [ ] Screenshots/screenshots tested in longest supported language (German often breaks layouts)

---

## Key Takeaways

1. **The pre-permission dialog is your most powerful tool.** Explaining WHY before the system prompt doubles or triples grant rates. On iOS, you only get one shot — make it count.

2. **Accessibility is not optional.** It's a legal requirement (ADA, EAA), it serves 15% of the population, and it makes your app better for everyone. Start with `accessibilityLabel` and `accessibilityRole` on every interactive element.

3. **Focus management is the hard part of accessibility.** Labels and roles are the easy win. Controlling focus order, managing focus after navigation, and trapping focus in modals — that's where the real work is.

4. **Translate everything, including accessibility labels.** If your `accessibilityLabel` is in English and your user's VoiceOver is set to Spanish, they'll hear English labels in a Spanish app. Every user-facing string goes in the translation files.

5. **RTL is not a feature — it's a layout direction.** Use `paddingStart/End` instead of `paddingLeft/Right`. Use `flexDirection: 'row'` and let the system handle the direction. Test early with Arabic/Hebrew, because retrofitting RTL into a large codebase is painful.

6. **Use the `Intl` API for date/number formatting.** Don't build your own date formatter. The browser/runtime's `Intl` object knows how every locale formats dates, numbers, currencies, and relative times. Use it.

7. **Persist the user's language choice.** Don't assume the device language is the user's preferred app language. Store their choice in AsyncStorage, load it on app start, and let them change it from within the app.

---

**Next up:** [Chapter 15: Performance Optimization] — where we go deep on rendering performance, frame budgets, memory profiling, and making your app feel fast on every device. Because all the accessibility and i18n in the world doesn't matter if your app stutters.
