# UI & Design — Expo UI, Tailwind, Liquid Glass (June 2026)

## THE FOUR LAYERS (in priority order)

```
1. NATIVE PRIMITIVES  → Expo UI (@expo/ui) — SwiftUI / Jetpack Compose / universal
2. STYLING ENGINE     → Uniwind (recommended) OR NativeWind v5 preview — Tailwind CSS v4
3. PLATFORM CHROME    → NativeTabs, expo-glass-effect, Material 3 dynamic colors
4. CROSS-PLATFORM     → lucide-react-native, expo-image, shared layout in app/
```

**Senior rule:** Expo UI for system-native UX; Tailwind layer for branded screens; do not fight the platform with JS-only tab bars in production native apps.

**Anti–AI slop:** Use the **`anti-ai-slop`** skill (universal, any project) + `references/anti-ai-slop.md` — tokens from repo, ≤2 raised surfaces per view, rows not card stacks.

---

## LAYER 1 — EXPO UI (SDK 56 STABLE)

Included in default `create-expo-app@sdk-56` template. Works in Expo Go (SDK 56 builds) and dev builds.

### Universal components (iOS + Android + web experimental)

```tsx
import {
  Host,
  Row,
  Column,
  ScrollView,
  Text,
  TextInput,
  Button,
  Switch,
  Slider,
  Checkbox,
  BottomSheet,
} from '@expo/ui';
```

Web backs onto `react-dom` / `react-native-web`. Native uses Jetpack Compose + SwiftUI.

### Platform-specific APIs (stable)

```tsx
// iOS SwiftUI
import { Host, Text } from '@expo/ui/swift-ui';
import { glassEffect, padding } from '@expo/ui/swift-ui/modifiers';

// Android Compose
import { Host, Text } from '@expo/ui/jetpack-compose';
import { useMaterialColors } from '@expo/ui/jetpack-compose';
```

**SDK 56 highlights:**
- `useNativeState` — JS drives Swift `ObservableObject` / Compose `MutableState`
- `WorkletCallback` — synchronous worklet props (flicker-free controlled inputs)
- Extend with custom SwiftUI/Compose views + modifiers
- **Drop-in replacements** — change import only for many community libs:

```tsx
// Before
import DateTimePicker from '@react-native-community/datetimepicker';
// After
import DateTimePicker from '@expo/ui/community/datetime-picker';
```

Also: bottom-sheet, picker, slider, segmented-control, pager-view, menu, masked-view paths under `@expo/ui/community/*`.

Docs: https://docs.expo.dev/versions/latest/sdk/ui/

### When to use Expo UI vs Tailwind custom

| Use Expo UI | Use Uniwind / NativeWind |
|---|---|
| Settings, system forms, sheets | Marketing, brand-heavy screens |
| Switches, sliders, native pickers | Custom cards, game UI, dashboards |
| MD3 dynamic colors on Android | Web-first marketing layouts |

---

## LAYER 2A — UNIWIND (recommended for new apps)

From Unistyles team. **Tailwind v4 only.** Metro plugin — **no Babel preset**. Works with Expo Go (OSS tier).

```powershell
npm install uniwind tailwindcss@^4
```

```js
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config');
const { withUniwind } = require('uniwind/metro');
const config = getDefaultConfig(__dirname);
module.exports = withUniwind(config);
```

```css
/* global.css */
@import 'tailwindcss';
@theme {
  --color-primary: #0ea5e9;
}
```

```tsx
<View className="flex-1 bg-background p-4">
  <Text className="text-2xl font-bold text-primary">Hello</Text>
</View>
```

**Themes:** CSS `@theme` + runtime theme switching — no `ThemeProvider` required for Uniwind (still use Router `ThemeProvider` for navigation chrome).

**Safe area:** `p-safe`, `m-safe`, etc. with `react-native-safe-area-context` (OSS); Pro tier injects insets in C++.

**Migration from NativeWind:** https://docs.uniwind.dev/migration-from-nativewind

**Use `tailwind-merge`** — Uniwind does not dedupe conflicting classes on web.

---

## LAYER 2B — NATIVEWIND v5 (preview)

Use if you need NativeWind ecosystem/docs. Requires RN 0.81+, Reanimated 4+, Tailwind 4.1+.

```powershell
npx expo install nativewind@preview react-native-css react-native-reanimated react-native-safe-area-context
npx expo install --dev tailwindcss @tailwindcss/postcss postcss
```

```js
// metro.config.js
const { withNativewind } = require('nativewind/metro');
module.exports = withNativewind(getDefaultConfig(__dirname));
```

```json
{
  "overrides": { "lightningcss": "1.30.1" }
}
```

**Do not start new projects on NativeWind v4** (Tailwind v3).

---

## LAYER 2C — UNISTYLES 3.0 (object styles, C++ / Fabric)

**When to pick Unistyles over Uniwind/NativeWind:**

| Choose Unistyles 3 | Choose Uniwind / NativeWind |
|--------------------|----------------------------|
| Heavy **runtime** theme variants (programmatic objects) | Tailwind/design-token workflow, web+mobile shared utilities |
| Team prefers `StyleSheet`-like objects, not `className` | New greenfield with Tailwind v4 |
| Animation-heavy dynamic themes where style recalc must stay off JS thread | Max agent/vibe-coder familiarity with utility classes |

Unistyles 3 uses a **C++ core via JSI** (Fabric path) — object-based API, variants/breakpoints, not string utilities.

```powershell
npx expo install react-native-unistyles
```

**Default for this skill remains Uniwind** for new apps. Unistyles is the **performance-purist** branch when Tailwind is the wrong fit.

Do not run **both** Uniwind and Unistyles as competing global styling engines.

---

## `cn()` UTILITY (both stacks)

```ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## LAYER 3 — LIQUID GLASS (iOS 26)

**Official path (use first):**

1. **Expo UI** `glassEffect` modifier (SwiftUI-backed)
2. **`expo-glass-effect`** for `UIVisualEffectView` in RN tree

```powershell
npx expo install expo-glass-effect
```

```tsx
import { GlassView, isGlassEffectAPIAvailable } from 'expo-glass-effect';

export function GlassHeader() {
  if (!isGlassEffectAPIAvailable()) {
    return <View style={styles.fallbackHeader} />;
  }
  return <GlassView style={styles.header} glassEffectStyle="regular" />;
}
```

Some iOS 26 betas lack API — always guard with `isGlassEffectAPIAvailable()`.

Respect `AccessibilityInfo.isReduceTransparencyEnabled()`.

**Third-party (`@callstack/liquid-glass`):** only if you need Callstack-specific APIs — not the 2026 default.

### Floating tab bar (optional product chrome)

When the app needs a **floating frosted pill** over colorful/mesh backgrounds (not system edge tabs):

- Read `references/floating-glass-tab-bar.md`
- **iOS:** `GlassView` / `BlurView` + frost underlay + top sheen gradient
- **Android:** layered semi-opaque frost (avoid `BlurView` black bar in dark mode)
- Pair with **mesh/gradient hero** behind content so glass has depth to refract
- Wire via Expo Router `tabBar={(props) => <CustomBar {...props} />}` + transparent `tabBarStyle`

### Performance rules
- Max **~3** stacked glass layers per screen
- Never in FlashList/LegendList rows
- Test FPS on device with Reanimated + glass combined

### App icons (iOS 26)

```json
{
  "expo": {
    "ios": { "icon": "./assets/icon.icon" }
  }
}
```

Create `.icon` with Apple Icon Composer.

---

## THEMING & NAVIGATION

```tsx
import { ThemeProvider, DarkTheme, DefaultTheme } from 'expo-router/react-navigation';
import { useColorScheme } from 'react-native';

export default function RootLayout() {
  const scheme = useColorScheme();
  return (
    <ThemeProvider value={scheme === 'dark' ? DarkTheme : DefaultTheme}>
      {/* Stack or NativeTabs */}
    </ThemeProvider>
  );
}
```

Tailwind dark: `dark:` variants (Uniwind/NativeWind) + system `useColorScheme`.

---

## ICONS (2026)

| Platform | Package | Notes |
|---|---|---|
| iOS | `expo-symbols` | SF Symbols, Dynamic Type |
| Android MD3 | `@expo/material-symbols` + Expo UI `Icon` | Full Material Symbols catalog |
| Cross-platform | `lucide-react-native` | Consistent vectors |

**Deprecation:** `@expo/vector-icons` → `@react-native-vector-icons/*`  
Codemod: `npx @react-native-vector-icons/codemod`

NativeTabs: `sf` on iOS, `md` on Android triggers.

---

## IMAGES — EXPO IMAGE

```powershell
npx expo install expo-image
```

```tsx
import { Image } from 'expo-image';

<Image
  source={{ uri: photoUrl }}
  placeholder={{ blurhash: 'L6PZfSi_.AyE_3t7t7R**0o#DgR4' }}
  contentFit="cover"
  transition={200}
  cachePolicy="memory-disk"
  priority="high"
/>
```

Never use RN `<Image>` for remote URLs in production.

---

## SKELETONS

```powershell
npm install moti
```

Match layout dimensions to real content — no generic spinners for full screens.

---

## COMPONENT KIT DECISION TREE

```
Need native system UI?
  → Expo UI

Need Tailwind design system?
  → Uniwind (new) OR NativeWind v5 preview

Need many prebuilt RN components?
  → gluestack-ui v3 OR build with Uniwind

Need maximum styled perf + design tokens at scale?
  → Tamagui (heavier; justify with team size)

Material-only legacy?
  → react-native-paper v6 (MD3) — prefer Expo UI on SDK 56
```

---

## RESPONSIVE / TABLET

```tsx
const { width } = useWindowDimensions();
const isTablet = width >= 768;
```

Uniwind/NativeWind: `md:`, `lg:` breakpoints.

Test: iPhone SE, Pro Max, iPad, low-end Android narrow width.

---

## WEB (Expo Router)

Expo UI universal components work on web (experimental). Use `*.web.tsx` for divergent tab/navigation chrome.

NativeTabs are **native-only** — always split layouts for web.
