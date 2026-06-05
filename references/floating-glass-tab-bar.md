# Floating Liquid Glass Tab Bar — Optional Pattern (June 2026)

**Scope:** Universal Expo Router pattern for **any** app — not tied to a specific product or brand.

Use when you want a **floating pill tab bar** over colorful/mesh backgrounds (reference: frosted capsule with active oval highlight, icon + label stack). This is an **alternative** to system `NativeTabs`, not a replacement for every app.

---

## WHEN TO CHOOSE THIS

| Choose **Floating Glass JS tabs** | Choose **NativeTabs** |
|---|---|
| Brand needs a **floating pill** over hero/mesh UI | System Liquid Glass / Material 3 is enough |
| Custom active pill, spacing, labels, 3–5 tabs | Want scroll-to-top on re-tap, bottom accessory (iOS 26) |
| Same chrome on **Android + web** with one design | Maximum OS-native behavior with minimal JS |
| Content **scrolls behind** the bar (glass reads on variation) | Flat/system bar attached to screen edge is fine |

**Default skill recommendation remains NativeTabs for production native chrome.** Add floating glass when design explicitly needs it.

---

## VISUAL STRATEGIES (color-agnostic)

Borrow the **structure**, not a fixed palette:

1. **Floating capsule** — high `borderRadius` (≈24–32), horizontal margin from screen edges (≈12–20), small gap above home indicator.
2. **Frosted translucency** — background content (mesh, gradients, photos) **must** show through softly; flat gray screens make glass look like plastic.
3. **Hairline rim** — 1px border, high-opacity white or `foreground` at ~15–25% on light glass (defines pill edge).
4. **Diffuse float shadow** — low-opacity shadow + Android `elevation` (≈6–12); shadow color from app `shadow` token, not pure black.
5. **Active tab** — **oval pill** behind icon + label (inset from slot edges), not full-width segment or underline.
6. **Inactive tabs** — icon above label, muted `foreground` / `muted-foreground`; no background chip.
7. **Press** — subtle scale (~0.94–0.96 spring) on tab tap; haptics optional (often **off** for tab switches — see `premium-feel.md`).
8. **Background behind bar** — prefer **mesh / soft blobs / gradient fields** on tab screens so blur/frost has something to refract (same strategy as reference mockups with green/yellow wash).

**Token example (adapt to your theme):**

```ts
export const FLOATING_TAB_GLASS = {
  frost: "rgba(255, 255, 255, 0.86)",       // or theme glass light
  frostAndroid: "rgba(252, 254, 255, 0.92)", // Android: often no real blur
  frostUnderlay: "rgba(248, 250, 252, 0.75)",
  sheen: ["rgba(255,255,255,0.5)", "rgba(255,255,255,0.1)", "rgba(255,255,255,0)"] as const,
  border: "rgba(255, 255, 255, 0.82)",
  borderInner: "rgba(0, 0, 0, 0.06)",
  tintBrand: "rgba(var(--primary-rgb), 0.06)", // optional brand wash
  activePill: "rgba(var(--primary-rgb), 0.12)",
  shadow: "#0f172a", // or theme shadow
} as const;
```

---

## PLATFORM IMPLEMENTATION

### iOS (dev build)

1. **Prefer** `expo-glass-effect` `GlassView` when `isGlassEffectAPIAvailable()`.
2. **Fallback:** `expo-blur` `BlurView` with `tint="systemChromeMaterialLight"` (or dark-aware tint if app is dark-first).
3. Layer **frost underlay + top sheen** (`expo-linear-gradient`) under blur so bar never reads as empty transparent.

### Android (critical)

- **`BlurView` + system dark mode often renders solid black** — use **layered frost** (semi-opaque Views) + top sheen gradient instead of blur for the tab bar shell.
- Keep bar **light-frosted** even in dark app themes if home UI is light/mesh-first; or maintain a dedicated `tabBarGlass` token set per theme.
- `tabBarStyle`: `position: 'absolute'`, transparent background, `elevation: 0` on navigator chrome; real shadow on **pill shell** only.

### Web

- Frost layers + optional `backdrop-filter: blur(20px)` on a wrapper; test Safari.
- Split layout: `_layout.web.tsx` → JS floating tabs; `_layout.ios.tsx` → NativeTabs or same JS bar per product choice.

---

## EXPO ROUTER WIRING

```tsx
// app/(tabs)/_layout.tsx (or shared JsTabsLayout module)
import { Tabs } from 'expo-router';
import { Platform, View } from 'react-native';
import { FloatingGlassTabBar } from '@/components/navigation/FloatingGlassTabBar';

export default function TabLayout() {
  return (
    <Tabs
      tabBar={(props) => <FloatingGlassTabBar {...props} />}
      screenOptions={{
        headerShown: false,
        tabBarShowLabel: false, // custom bar renders labels
        animation: Platform.OS === 'android' ? 'shift' : 'fade',
        tabBarBackground: () => (
          <View style={{ flex: 1, backgroundColor: 'transparent' }} />
        ),
        tabBarStyle: {
          position: 'absolute',
          left: 0,
          right: 0,
          bottom: 0,
          backgroundColor: 'transparent',
          borderTopWidth: 0,
          elevation: 0,
          shadowOpacity: 0,
        },
      }}
    >
      <Tabs.Screen name="index" />
      {/* static tab list — max ~5 */}
    </Tabs>
  );
}
```

**Types:** `BottomTabBarProps` from `expo-router/js-tabs`.

**Safe area:** `useSafeAreaInsets()` inside custom bar; add bottom padding for home indicator. Navigator still wraps `tabBar` in `SafeAreaInsetsContext.Consumer` — ensure every child component is a **named import**, not `import * as X` used as JSX tag (common `undefined` crash).

**Hide on nested routes:** `href: null` on non-tab screens + pathname guard returning `null` from custom bar.

---

## COMPONENT STRUCTURE

```
FloatingGlassTabBar
├── host (absolute bottom, pointerEvents box-none)
├── floatWrap (horizontal margin, bottom margin)
└── GlassShell (pill)
    ├── FrostLayers (underlay + frost + optional brand tint + LinearGradient sheen)
    ├── [iOS] GlassView or BlurView
    ├── border hairline overlay
    └── row
        ├── Animated active oval (translateX withSpring on tab index)
        └── tabs[] → Pressable + Animated.View scale
            ├── Icon
            └── Label
```

**Active pill motion:** `useSharedValue` + `withSpring` sliding oval to `index * slotWidth` (Reanimated 4).

**Press pattern (reliable):** `Pressable` wrapping `Animated.View` for scale — avoid `Animated.createAnimatedComponent(Pressable)` if it returns `undefined` in some builds.

---

## MESH / HERO BACKGROUND (pairs with glass)

On tab root screens, use a **full-bleed mesh or gradient mesh** behind scroll content so the floating bar has visual depth:

- 2–4 soft radial gradients or blurred color blobs (`expo-linear-gradient`, Skia, or static image).
- Keep mesh `pointerEvents="none"` so taps pass through to content.
- Do **not** put heavy blur stacks inside list rows (`ui-design.md` glass limits).

---

## ANTI-PATTERNS

- Solid opaque `#000` or `#fff` tab bar pretending to be glass
- `import * as NavigationBar` then `<NavigationBar />` — use `import { NavigationBar } from 'expo-navigation-bar'`
- `BlurView` on Android tab bar without testing dark mode (black bar)
- Dynamic tab count at runtime (remounts navigator — keep static triggers)
- Glass layers inside FlashList/LegendList rows
- More than **~3** stacked glass surfaces visible at once on one screen

---

## DECISION FLOW

```
Need system-native tabs?
  yes → NativeTabs (navigation.md)
  no → Need floating frosted pill over mesh/hero UI?
    yes → This file (JS custom tabBar)
    no → Default JS <Tabs> or expo-router/ui on web
```

---

## RELATED FILES

- `references/navigation.md` — NativeTabs vs JS tabs
- `references/ui-design.md` — Liquid Glass layers, performance caps
- `references/premium-feel.md` — press scale, haptics on tabs
- `references/platform-polish.md` — safe area, edge-to-edge
