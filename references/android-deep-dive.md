# Android Deep Dive — Safe Areas, Edge-to-Edge, APK/AAB, Play Store

## WHY ANDROID NEEDS ITS OWN FILE

Android is not "iOS but cheaper." It has fundamentally different:
- Safe area anatomy (status bar + navigation bar + gesture zones + notches)
- Screen fragmentation (foldables, tablets, ultra-wide, ultra-tall)
- Build output formats (APK vs AAB)
- Store distribution tracks (internal → closed → open → production)
- Navigation paradigm (predictive back, back stack, task management)
- Material Design 3 language vs iOS HIG

Getting Android right is a separate discipline. This file covers it completely.

---

## SAFE AREA ANATOMY — EVERY ANDROID DEVICE TYPE

### The 4 inset zones on Android

```
┌─────────────────────────────────────┐
│  STATUS BAR (top inset)             │ ← insets.top (24–48dp)
├─────────────────────────────────────┤
│                                     │
│         YOUR CONTENT                │
│                                     │
├─────────────────────────────────────┤
│  NAV BAR or GESTURE ZONE (bottom)   │ ← insets.bottom (0–48dp)
└─────────────────────────────────────┘
         ↑ possible side insets for curved screens / foldables
```

### Status bar heights by device class

| Device | Status bar height |
|---|---|
| Standard Android (XXHDPI) | 24dp |
| Devices with notch/punch-hole | 28–48dp (varies) |
| Foldable (outer screen) | 24–32dp |
| Android tablet | 24dp |

**Never hardcode status bar height.** Always use `insets.top`.

### Navigation bar variants

| Nav type | `insets.bottom` | How to detect |
|---|---|---|
| Gesture navigation (Android 10+) | 0–20dp (thin) | Default on modern devices |
| 3-button nav bar | 48dp | Legacy, still common |
| 2-button nav bar | 32dp | Transitional (rare) |
| None (fullscreen/foldable) | 0 | Gaming / fullscreen mode |

**The navigation bar can change at runtime** (user can switch between gesture and button nav in settings).
Always listen for inset changes — never assume static values.

### The correct safe area implementation

```tsx
// ✅ CORRECT — handles all device variants dynamically
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Screen() {
  const insets = useSafeAreaInsets();

  return (
    <View style={{
      flex: 1,
      paddingTop: insets.top,
      paddingBottom: insets.bottom,
      paddingLeft: insets.left,    // foldables, curved screens
      paddingRight: insets.right,  // foldables, curved screens
    }}>
      <Content />
    </View>
  );
}

// ✅ With NativeWind (cleaner)
<View className="flex-1 pt-safe pb-safe pl-safe pr-safe">
```

### SafeAreaView vs useSafeAreaInsets

```tsx
// SafeAreaView — wraps the entire screen
// Good for: screens where ALL content needs inset padding
import { SafeAreaView } from 'react-native-safe-area-context';

<SafeAreaView style={{ flex: 1 }} edges={['top', 'bottom']}>
  <Content />
</SafeAreaView>

// useSafeAreaInsets — granular control
// Good for: screens where header is transparent / content bleeds under status bar
const insets = useSafeAreaInsets();

<View style={{ flex: 1 }}>
  {/* Image bleeds under status bar */}
  <Image source={hero} style={{ height: 300 + insets.top }} />

  {/* Content starts below status bar */}
  <View style={{ paddingTop: insets.top }}>
    <BackButton />
  </View>
</View>
```

### The `edges` prop — control which sides apply insets

```tsx
// Only apply top and bottom (most common for screens)
<SafeAreaView edges={['top', 'bottom']}>

// Only bottom (screen has custom header that handles top)
<SafeAreaView edges={['bottom']}>

// Only top (screen has bottom nav bar handled elsewhere)
<SafeAreaView edges={['top']}>

// All sides (foldable / tablet safe)
<SafeAreaView edges={['top', 'bottom', 'left', 'right']}>
```

---

## EDGE-TO-EDGE — THE FULL PICTURE

Mandatory from SDK 54 targeting Android 16. **Cannot be disabled.**

### What edge-to-edge means in practice

```
WITHOUT edge-to-edge:
┌─────────────────────────────┐
│ [system status bar]         │  ← system-controlled, black/white
│─────────────────────────────│
│ YOUR APP CONTENT            │
│─────────────────────────────│
│ [system nav bar]            │  ← system-controlled, black/white
└─────────────────────────────┘

WITH edge-to-edge (SDK 54+):
┌─────────────────────────────┐
│ YOUR APP (status bar zone)  │  ← your content renders here
│ YOUR APP CONTENT            │
│ YOUR APP (nav bar zone)     │  ← your content renders here
└─────────────────────────────┘
System overlays are transparent — your content shows through
```

### The 3 things you must handle for edge-to-edge

```tsx
// 1. Status bar — make it transparent and set icon color
import { StatusBar } from 'expo-status-bar';
<StatusBar style="auto" translucent backgroundColor="transparent" />

// 2. Content — don't let it hide under system bars
const insets = useSafeAreaInsets();
<View style={{ paddingTop: insets.top, paddingBottom: insets.bottom }}>

// 3. Floating elements (FAB, snackbar) — position above nav bar
<View style={{ bottom: insets.bottom + 16 }}>
  <FAB />
</View>
```

### WindowInsets listener (runtime inset changes)

```tsx
// When user switches between gesture and button nav mid-session
import { useSafeAreaInsets } from 'react-native-safe-area-context';

// useSafeAreaInsets() automatically re-renders when insets change
// This is why you must use it — never cache inset values in state
```

---

## ANDROID DEVICE FRAGMENTATION — HANDLE ALL OF THESE

### Foldables (Samsung Galaxy Fold, Pixel Fold)

```tsx
import { useWindowDimensions } from 'react-native';

function FoldableAwareLayout() {
  const { width, height } = useWindowDimensions();

  // Detect folded vs unfolded state by aspect ratio
  const isUnfolded = width > height;  // landscape/wide = unfolded
  const isFolded = height > width * 1.5;  // very tall = outer (folded) screen

  return (
    <View className={cn(
      "flex-1",
      isUnfolded && "flex-row",  // two-column layout when unfolded
    )}>
      {isUnfolded && <Sidebar className="w-80" />}
      <MainContent className="flex-1" />
    </View>
  );
}
```

### Punch-hole cameras (most modern Android)

```tsx
// useSafeAreaInsets handles this automatically
// insets.top will be larger on devices with punch-hole
const insets = useSafeAreaInsets();

// Never assume insets.top is 24dp on Android — punch-hole devices return 44-48dp
```

### Ultra-wide / landscape tablets

```tsx
// Side insets matter on tablets in landscape
const insets = useSafeAreaInsets();

<View style={{
  paddingLeft: Math.max(insets.left, 16),   // at least 16dp padding
  paddingRight: Math.max(insets.right, 16),
}}>
```

### Large screen / tablet layout (Android adaptive UI)

```tsx
import { useWindowDimensions } from 'react-native';

// Android window size classes (Material 3)
function useWindowSizeClass() {
  const { width } = useWindowDimensions();
  if (width < 600) return 'compact';    // phone
  if (width < 840) return 'medium';     // small tablet, foldable unfolded
  return 'expanded';                    // large tablet, desktop
}

function AdaptiveScreen() {
  const sizeClass = useWindowSizeClass();

  return (
    <View className="flex-1">
      {sizeClass === 'compact' && <PhoneLayout />}
      {sizeClass === 'medium' && <TabletLayout sidebar={false} />}
      {sizeClass === 'expanded' && <TabletLayout sidebar={true} />}
    </View>
  );
}
```

---

## MATERIAL DESIGN 3 — ANDROID UI PATTERNS

### Color system (M3 dynamic color)

```tsx
// Android 12+ supports dynamic color — pulls palette from wallpaper
// In React Native, use platform-aware colors
import { Platform, useColorScheme } from 'react-native';

const colors = {
  // M3 surface colors
  surface: Platform.select({
    android: '#FFFBFE',   // M3 surface
    ios: '#FFFFFF',
  }),
  surfaceVariant: Platform.select({
    android: '#E7E0EC',   // M3 surface variant
    ios: '#F2F2F7',       // iOS grouped background
  }),
};
```

### Android-specific UI components

```tsx
// FAB (Floating Action Button) — Android primary action
// Position: bottom right, above nav bar
function AndroidFAB({ onPress }: { onPress: () => void }) {
  const insets = useSafeAreaInsets();

  return (
    <Pressable
      onPress={onPress}
      style={{
        position: 'absolute',
        bottom: insets.bottom + 16,
        right: 16,
        width: 56,
        height: 56,
        borderRadius: 16,     // M3 FAB has 16dp radius, not circle
        backgroundColor: '#6750A4',
        alignItems: 'center',
        justifyContent: 'center',
        elevation: 6,          // Android shadow
      }}
    >
      <Icon name="add" size={24} color="white" />
    </Pressable>
  );
}

// Android elevation (shadow system)
// iOS uses shadowColor/shadowOffset — Android uses elevation
const cardStyle = Platform.select({
  android: { elevation: 2 },
  ios: {
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
  },
});
```

### Android ripple effect (required for native feel)

```tsx
import { Pressable, Platform } from 'react-native';

// Android ripple — native ink splash on press
<Pressable
  android_ripple={{
    color: 'rgba(0,0,0,0.1)',
    borderless: false,
    radius: -1,  // -1 = fill the view
  }}
  onPress={handlePress}
>
  <Content />
</Pressable>

// For circular buttons (icon buttons)
<Pressable
  android_ripple={{
    color: 'rgba(0,0,0,0.1)',
    borderless: true,   // ripple extends beyond bounds (circle)
    radius: 28,
  }}
  style={{ width: 48, height: 48, borderRadius: 24 }}
>
  <Icon />
</Pressable>
```

---

## APK vs AAB — WHEN TO USE WHICH

| Format | Use for | Size | Store compatible |
|---|---|---|---|
| **AAB** (.aab) | Play Store production | Smaller (Google optimizes per device) | ✅ Required for Play Store |
| **APK** (.apk) | Direct install, testing, sideload | Larger (all architectures) | ❌ Not for store |

```json
// eas.json — correct build types
{
  "build": {
    "development": {
      "developmentClient": true,
      "android": {
        "buildType": "apk"  // APK for dev — direct install on device
      }
    },
    "preview": {
      "android": {
        "buildType": "apk"  // APK for preview — share with testers directly
      }
    },
    "production": {
      "android": {
        "buildType": "app-bundle"  // AAB for Play Store — mandatory
      }
    }
  }
}
```

### APK architectures

```json
// For preview builds — include all architectures for maximum device compatibility
{
  "build": {
    "preview": {
      "android": {
        "buildType": "apk",
        "gradleCommand": ":app:assembleRelease"
      }
    }
  }
}
```

### AAB size optimization

```json
// app.json — strip unused resources, enable R8
{
  "expo": {
    "android": {
      "enableProguardInReleaseBuilds": true,
      "enableR8FullMode": true,
      "shrinkResources": true  // removes unused drawables, strings
    }
  }
}
```

Expected size reduction: ~20–35% smaller AAB with all three enabled.

---

## ANDROID APP SIGNING

### EAS managed signing (recommended — zero setup)

```json
// eas.json
{
  "build": {
    "production": {
      "android": {
        "credentialsSource": "remote"  // EAS stores your keystore securely
      }
    }
  }
}
```

EAS creates and stores your Android keystore automatically.
Download a backup immediately after first production build:

```bash
eas credentials --platform android
# Choose: "Download keystore"
# Store this file OFFLINE — losing it = you can never update your app
```

### The keystore rule
**If you lose your Android keystore, you cannot publish updates to your existing app.**
You would have to publish a new app with a new package name and lose all reviews/ratings.
Back it up to: encrypted cloud storage + offline drive.

### Key hash for Facebook/Google SDK (if needed)

```bash
# Get SHA-1 fingerprint from EAS credentials
eas credentials --platform android
# Or from keystore directly:
keytool -list -v -keystore your.keystore -alias your-alias
```

---

## PLAY STORE DISTRIBUTION TRACKS

Play Store has 4 tracks. Use them in order — never skip to production.

```
Internal testing  → Closed testing → Open testing → Production
    (< 100)           (invite only)    (opt-in)       (everyone)
```

### Track strategy

```bash
# 1. Internal testing (instant review, < 100 testers)
# Use for: every build during development
eas submit --platform android --track internal

# 2. Closed testing (alpha) — requires invitation
# Use for: beta with specific users
eas submit --platform android --track alpha

# 3. Open testing (beta) — anyone can opt in
# Use for: public beta, collecting reviews before full launch
eas submit --platform android --track beta

# 4. Production — staged rollout
# Start at 10%, watch crash rates, expand
eas submit --platform android --track production
```

### Staged rollout (production)

```bash
# Start at 10% of users
eas submit --platform android --track production --rollout 0.1

# If no crashes after 24h, expand
# Do this in Play Console UI: Production → Manage rollout → Increase percentage
```

**Never launch at 100% production.** Always stage: 10% → 25% → 50% → 100%.
A critical bug at 10% = 10% of users affected. At 100% = everyone.

---

## ANDROID PERMISSIONS — RUNTIME REQUESTING

Android 6+ requires runtime permission requests. Declare + request.

```json
// app.json — declare permissions (required)
{
  "expo": {
    "android": {
      "permissions": [
        "CAMERA",
        "READ_MEDIA_IMAGES",  // Android 13+ (replaces READ_EXTERNAL_STORAGE)
        "RECORD_AUDIO",
        "ACCESS_FINE_LOCATION",
        "POST_NOTIFICATIONS"  // Android 13+ — required for push notifications
      ]
    }
  }
}
```

```tsx
// Request at the right moment (not at launch)
import { PermissionsAndroid, Platform } from 'react-native';
import * as Notifications from 'expo-notifications';

async function requestNotificationPermission() {
  if (Platform.OS === 'android' && Platform.Version >= 33) {
    // Android 13+ requires explicit notification permission
    const result = await PermissionsAndroid.request(
      PermissionsAndroid.PERMISSIONS.POST_NOTIFICATIONS
    );
    return result === PermissionsAndroid.RESULTS.GRANTED;
  }
  // Android < 13: notifications granted by default
  return true;
}
```

### Android 16 permission changes

```json
// Precise vs approximate location (Android 12+)
"permissions": [
  "ACCESS_FINE_LOCATION",      // precise (requires justification)
  "ACCESS_COARSE_LOCATION"     // approximate (easier to grant)
]
// Always request coarse first — most apps don't need precise
```

---

## ANDROID-SPECIFIC DEBUGGING

### View the safe area insets in real time

```tsx
// Debug overlay — shows actual inset values on device
function InsetDebugger() {
  const insets = useSafeAreaInsets();
  if (!__DEV__) return null;

  return (
    <View style={{ position: 'absolute', top: insets.top + 8, right: 8, zIndex: 9999 }}>
      <Text style={{ backgroundColor: 'red', color: 'white', fontSize: 10, padding: 4 }}>
        T:{insets.top} B:{insets.bottom} L:{insets.left} R:{insets.right}
      </Text>
    </View>
  );
}
```

### Flipper for Android debugging

```bash
# Android debugging tools:
# 1. ADB logcat — raw device logs
adb logcat | grep -i "myapp"

# 2. Android Studio Logcat — visual, filterable
# 3. Flipper — React Native inspector, network, layout
# 4. Chrome DevTools — Hermes JS debugging
```

---

## ANDROID BUILD CHECKLIST

```
Before first Play Store submission:
- [ ] Package name set and FINAL (cannot change after publish)
- [ ] Version code starts at 1 (auto-increments with EAS autoIncrement)
- [ ] Keystore backed up offline (EAS + your own backup)
- [ ] AAB format (not APK) for production build
- [ ] Proguard + R8 full mode enabled
- [ ] shrinkResources enabled
- [ ] All required permissions declared in app.json
- [ ] POST_NOTIFICATIONS permission for Android 13+
- [ ] App icon: 512x512 PNG, no transparency, no rounded corners (Play Store adds them)
- [ ] Feature graphic: 1024x500 PNG (required for Play Store listing)
- [ ] Tested on: Android 10, Android 13, Android 16
- [ ] Tested with: gesture nav, 3-button nav
- [ ] Tested with: normal screen, large screen (tablet or emulator)
- [ ] Edge-to-edge verified: no content hidden behind status/nav bar
- [ ] Ripple effects on all Pressable elements
- [ ] No hardcoded status bar height (using insets only)
```
