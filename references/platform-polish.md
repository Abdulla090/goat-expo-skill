# Platform Polish — iOS 26, Android 16, Tabs, Haptics (June 2026)

## PRINCIPLE

Use **native primitives** (Expo UI, NativeTabs, platform symbols) so behavior matches OS expectations without `Platform.OS` spaghetti.

---

## NATIVE TABS

See `references/navigation.md` for full API.

**Quick rules:**
- `expo-router/unstable-native-tabs` + compound `NativeTabs.Trigger.*`
- `ThemeProvider` from `expo-router/react-navigation` wrapping tabs
- iOS: `sf` props · Android: `md` Material Symbols
- Dev build required for production-native tab bar
- Static tab list only

**iOS 26:** Liquid Glass tab bar, scroll-to-top on re-tap, `NativeTabs.BottomAccessory` (floating above bar).

**Android:** Material 3 dynamic colors via system theme; safe area built into NativeTabs (SDK 55+).

**Optional:** Floating frosted pill over mesh UI → `references/floating-glass-tab-bar.md` (JS custom `tabBar`, not NativeTabs).

---

## LIQUID GLASS

**Primary:** Expo UI `glassEffect` modifier  
**RN tree:** `expo-glass-effect` + `isGlassEffectAPIAvailable()`

```tsx
import { GlassView, isGlassEffectAPIAvailable } from 'expo-glass-effect';
```

Max ~3 glass layers per screen. Not in list rows.

**Icons:** `.icon` asset + Icon Composer for iOS 26 home screen treatment.

---

## ANDROID EDGE-TO-EDGE

Mandatory from SDK 54+ targeting Android 16. Cannot disable — design for insets.

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Screen() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ flex: 1, paddingTop: insets.top, paddingBottom: insets.bottom }}>
      {/* content */}
    </View>
  );
}
```

Uniwind: `pt-safe`, `pb-safe`, `p-safe` (with safe area context).

`SafeAreaProvider` at root — see navigation.md.

---

## PREDICTIVE BACK (Android)

Enabled by default with modern `react-native-screens` + Router. **Do not** block with `BackHandler` returning `true` unless necessary.

---

## HAPTICS

| Context | Choice |
|---|---|
| Default / Expo Go / cross-platform | `expo-haptics` (SDK 56: web Safari support) |
| Lowest latency iOS in dev build | `react-native-haptics` (optional) |
| Presets + worklet-safe patterns in dev build | `react-native-pulsar` (optional; see `premium-feel.md`) |

**Rule:** Haptics on **state changes** (submit, toggle, error) — not tab taps or scroll. See `premium-feel.md`.

```tsx
import * as Haptics from 'expo-haptics';

await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Success);
await Haptics.selectionAsync();
```

| Action | Feedback |
|---|---|
| Primary tap | Light impact |
| Success | Notification success |
| Error | Notification error |
| Toggle / picker step | Selection |
| Destructive confirm | Heavy impact |

---

## STATUS BAR & NAVIGATION BAR (SDK 56)

Declarative components merge in mount order:

```tsx
import { StatusBar } from 'expo-status-bar';
import { NavigationBar } from 'expo-navigation-bar';

<StatusBar style="auto" hidden={false} />
<NavigationBar style="auto" hidden={false} />
```

Config plugins available for both — aligned options.

---

## SPLASH SCREEN

```tsx
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();
SplashScreen.setOptions({ duration: 300, fade: true });
// hide after fonts + auth + critical data
```

---

## KEYBOARD

```tsx
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : undefined}
  style={{ flex: 1 }}
>
  <ScrollView keyboardShouldPersistTaps="handled">{/* form */}</ScrollView>
</KeyboardAvoidingView>
```

**Better:** `react-native-keyboard-controller` or **Legend List keyboard-chat** for chat UIs.

```powershell
npx expo install react-native-keyboard-controller
```

---

## WIDGETS (iOS)

`expo-widgets` stable SDK 56 — Live Activities, timelines, config plugin improvements.

---

## NOTIFICATIONS

```powershell
npx expo install expo-notifications
```

Request permissions in context (after user enables a feature), not at first launch.

---

## PLATFORM FILES

```
Component.tsx          # shared types
Component.ios.tsx
Component.android.tsx
Component.web.tsx
```

Prefer over long `Platform.select` blocks.

---

## ACCESSIBILITY (App Store gate)

```tsx
<Pressable
  accessibilityRole="button"
  accessibilityLabel="Submit order"
  accessibilityHint="Double tap to confirm"
>
```

- Minimum touch **44×44 pt** (iOS) / **48×48 dp** (Android)
- Don't rely on color alone for state
- Support Dynamic Type / font scaling where possible
- Respect reduce motion and reduce transparency
