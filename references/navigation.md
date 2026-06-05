# Navigation — Expo Router SDK 56 (June 2026)

## MENTAL MODEL

Expo Router = file-based routes + layouts. SDK 56 **forked** React Navigation packages — app code must import from **`expo-router/*`**, not `@react-navigation/*`.

React Navigation (Satya Sahoo) remains the ecosystem standard for non-Expo apps. Expo Router and RN can share ideas; import paths diverged in SDK 56.

---

## SDK 56 BREAKING CHANGE — IMPORT MIGRATION

**Before (SDK 55):**
```tsx
import { ThemeProvider, DarkTheme } from '@react-navigation/native';
import { createMaterialTopTabNavigator } from '@react-navigation/material-top-tabs';
```

**After (SDK 56):**
```tsx
import { ThemeProvider, DarkTheme } from 'expo-router/react-navigation';
import { createMaterialTopTabNavigator } from 'expo-router/js-top-tabs';
```

| Old import | New import |
|---|---|
| `@react-navigation/native` | `expo-router/react-navigation` |
| `@react-navigation/core` | `expo-router/react-navigation` |
| `@react-navigation/elements` | `expo-router/react-navigation` |
| `@react-navigation/bottom-tabs` | `expo-router/js-tabs` |
| `@react-navigation/material-top-tabs` | `expo-router/js-top-tabs` |
| `@react-navigation/stack` | Prefer Expo Router `Stack` layout |
| `@react-navigation/drawer` | Prefer Expo Router `Drawer` layout |

**Automate:**
```powershell
npx expo-codemod sdk-56-expo-router-react-navigation-replace app
```

`expo-doctor` warns if both `expo-router` and `@react-navigation/*` are direct dependencies — remove unused RN packages from `package.json` after migration.

CLI rewrites `@react-navigation/core` in **node_modules** temporarily; app code must still migrate.

Opt out of CLI check: `EXPO_ROUTER_DISABLE_RN_NAVIGATION_CHECK=1` (not recommended).

Guide: https://docs.expo.dev/router/migrate/sdk-55-to-56/

---

## ROOT LAYOUT — PROVIDER ORDER

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ThemeProvider, DarkTheme, DefaultTheme } from 'expo-router/react-navigation';
import { useColorScheme } from 'react-native';
import { StatusBar } from 'expo-status-bar';

const queryClient = new QueryClient();

export default function RootLayout() {
  const scheme = useColorScheme();
  return (
    <SafeAreaProvider>
      <QueryClientProvider client={queryClient}>
        <GestureHandlerRootView style={{ flex: 1 }}>
          <ThemeProvider value={scheme === 'dark' ? DarkTheme : DefaultTheme}>
            <StatusBar style="auto" />
            <Stack />
          </ThemeProvider>
        </GestureHandlerRootView>
      </QueryClientProvider>
    </SafeAreaProvider>
  );
}
```

`ThemeProvider` from `expo-router/react-navigation` — required for stable native tab/header theming (Liquid Glass dark mode flicker without it).

---

## NATIVE TABS (production native chrome)

**Import:** `expo-router/unstable-native-tabs`  
**Status:** Unstable API — expect breaking changes.  
**Requires:** Dev build (not store Expo Go for full native behavior).

### SDK 55+ compound API

```tsx
// app/(tabs)/_layout.tsx
import { NativeTabs } from 'expo-router/unstable-native-tabs';

export default function TabLayout() {
  return (
    <NativeTabs>
      <NativeTabs.Trigger name="index">
        <NativeTabs.Trigger.Label>Home</NativeTabs.Trigger.Label>
        <NativeTabs.Trigger.Icon sf="house.fill" md="home" />
      </NativeTabs.Trigger>
      <NativeTabs.Trigger name="explore">
        <NativeTabs.Trigger.Label>Explore</NativeTabs.Trigger.Label>
        <NativeTabs.Trigger.Icon sf="magnifyingglass" md="search" />
      </NativeTabs.Trigger>
    </NativeTabs>
  );
}
```

**Rules:**
- One `NativeTabs.Trigger` per tab route — **not** auto-added from filesystem
- `name` must match route (including groups like `(home)`)
- Tab set must be **static** — dynamic add/remove remounts navigator
- iOS: `sf` SF Symbols · Android: `md` Material Symbols
- iOS 26: Liquid Glass tab bar, scroll-to-top on tab re-tap, bottom accessory (`NativeTabs.BottomAccessory`)
- Android: Material 3 dynamic colors, safe area integrated
- Max ~5 tabs on Android (platform constraint)

Docs: https://docs.expo.dev/router/advanced/native-tabs/

### Web + native split (recommended)

```
app/(tabs)/_layout.tsx        → re-exports or simple wrapper
app/(tabs)/_layout.native.tsx → NativeTabs
app/(tabs)/_layout.web.tsx    → expo-router/ui or custom tabs
```

Expo blog: use platform files instead of one layout for all platforms.

---

## JS TABS (fallback)

Use when: Expo Go learning, web-first, or custom tab bar design.

```tsx
import { Tabs } from 'expo-router';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ headerShown: false }}>
      <Tabs.Screen name="index" options={{ title: 'Home' }} />
    </Tabs>
  );
}
```

---

## STACK — SDK 56 FEATURES

- Experimental **Stack v5** / Material headers / **predictive back** (with `react-native-screens`)
- **Stack.Toolbar** on Android (experimental, same API direction as iOS)
- iOS 26: native **zoom** stack transitions (system fidelity — prefer over JS mimic)
- **SplitView** (experimental) — tablet/foldable multi-pane native layouts
- **SuspenseFallback** export in layouts (like ErrorBoundary pattern)
- Web: streaming SSR behind `unstable_useServerRendering`, `generateMetadata`

```tsx
// app/_layout.tsx — loading UI per layout
export function SuspenseFallback() {
  return <ActivityIndicator style={{ flex: 1 }} />;
}
```

---

## DATA LOADING (Router helpers)

```tsx
import { createServerLoader, createStaticLoader } from 'expo-router';
// createStaticLoader — params only (static)
// createServerLoader — always has Request; errors if used in static context
```

Pair with TanStack Query for client cache; use loaders for SSR/web or critical path data.

---

## DEEP LINKING

Configure in `app.config.ts`:

```ts
export default {
  expo: {
    scheme: 'myapp',
    ios: { associatedDomains: ['applinks:myapp.com'] },
    android: { intentFilters: [/* app links */] },
  },
};
```

Validate params with Zod before navigation — see `references/universal-links.md`.

---

## NATIVE TABS VS JS TABS — DECISION

| Choose NativeTabs | Choose JS `<Tabs>` (default) | Choose **Floating Glass** JS tab bar |
|---|---|---|
| App Store quality native feel | Standard bottom bar, web parity | Floating **pill** over mesh/hero UI |
| iOS 26 Liquid Glass tabs | Primary target is web | Custom active oval + frosted shell |
| Material 3 Android tabs | Rapid prototype in limited Go build | Same branded chrome Android + iOS + web |
| Dev build in CI | Need dynamic tab count at runtime | Static 3–5 tabs, design-led product |

**Floating glass implementation:** `references/floating-glass-tab-bar.md` (universal pattern — adapt colors to repo theme).

---

## ANTI-PATTERNS

- Importing `@react-navigation/native` in `app/` with Expo Router 56+
- Custom scroll-to-top hacks when NativeTabs provides it
- Putting Liquid Glass layers inside every tab screen (GPU cost)
- `BackHandler` returning `true` everywhere (breaks Android predictive back)
- Forgetting new dev build after adding native tab / screen native deps

---

## FURTHER READING

- https://expo.dev/blog/expo-router-v6 (Native Tabs / Liquid Glass era)
- https://expo.dev/blog/expo-router-v55-more-native-navigation-more-powerful-web
- Expo skills `building-native-ui` tabs reference (GitHub: expo/skills)
