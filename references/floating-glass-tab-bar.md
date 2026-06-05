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

Borrow the **structure**, not a fixed palette. **iOS 26 edge shading detail:** `references/ios-26-liquid-glass-edge-shading.md`.

1. **Floating capsule** — high `borderRadius` (≈height/2), horizontal margin (≈12–20), gap above home indicator; optional **detached FAB** same height as pill.
2. **Frosted translucency** — pearl-slate tint on **white apps** (`rgba(226,232,240,0.55)` + blur); pure white frost disappears on `#FFF`.
3. **iOS 26 edge stack** (custom fallback) — specular top arc → Fresnel left/right rims → bottom inner shade → top hairline → dual rim → inset web shadows.
4. **Native iOS 26** — `GlassView` draws system rims; **skip** duplicate custom edge layers.
5. **Diffuse float shadow** — adaptive: stronger over busy content, softer on flat white.
6. **Active tab** — **circle** or capsule chip inside pill; **`springMotion` on press** (optimistic) + `useEffect` when `state.index` catches up — **not** `useAnimatedReaction` (easy to break imports; see `web-rn-pitfalls.md`).
7. **Inactive tabs** — icon-only or icon + label; muted foreground.
8. **Press** — scale ~0.92–0.96 spring; tab haptics usually off (`premium-feel.md`).
9. **Scroll edge** — transparent `tabBarStyle`; list `paddingBottom` clears float; no opaque bar backgrounds behind glass.

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
- **RN 0.83+ web:** `boxShadow` via `crossShadow()` — never raw `shadow*` in shared styles; `pointerEvents` on **style**, not prop (`web-rn-pitfalls.md`).
- **Reanimated:** `springMotion()` (web `withTiming`, native `withSpring`) for indicator slide — avoids easing warnings.
- After tab bar refactors: `npx expo start --web --clear` + hard refresh — stale HMR can throw `useAnimatedReaction is not defined` even when source removed it.

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
├── host (absolute bottom, style.pointerEvents: 'box-none')
├── floatWrap (horizontal margin, bottom margin)
└── GlassShell (pill)
    ├── FrostLayers (underlay + frost + optional brand tint + LinearGradient sheen)
    ├── [iOS] GlassView or BlurView
    ├── border hairline overlay
    └── row
        ├── Animated active circle/chip (translateX via springMotion / optimistic press)
        └── tabs[] → Pressable + Animated.View scale
            ├── Icon
            └── Label
```

**Active pill motion:** `useSharedValue` + **`springMotion`** sliding chip to `index * slotWidth` (Reanimated 4). On press: move immediately; `useEffect` syncs when `state.index` updates (skip if optimistic press already moved).

```ts
// utils/motion-spring.ts — web uses withTiming, native withSpring
pillX.value = springMotion(pillTargetX(index));
```

**Press pattern (reliable):** `Pressable` wrapping `Animated.View` for scale — avoid `Animated.createAnimatedComponent(Pressable)` if it returns `undefined` in some builds.

---

## MESH / HERO BACKGROUND (pairs with glass)

On tab root screens, use a **full-bleed mesh or gradient mesh** behind scroll content so the floating bar has visual depth:

- 2–4 soft radial gradients or blurred color blobs (`expo-linear-gradient`, Skia, or static image).
- Keep mesh `style={{ pointerEvents: 'none' }}` so taps pass through to content.
- Do **not** put heavy blur stacks inside list rows (`ui-design.md` glass limits).

---

## ANTI-PATTERNS

- Solid opaque `#000` or `#fff` tab bar pretending to be glass
- `import * as NavigationBar` then `<NavigationBar />` — use `import { NavigationBar } from 'expo-navigation-bar'`
- `BlurView` on Android tab bar without testing dark mode (black bar)
- Dynamic tab count at runtime (remounts navigator — keep static triggers)
- Glass layers inside FlashList/LegendList rows
- More than **~3** stacked glass surfaces visible at once on one screen
- `useAnimatedReaction` for tab index when `useEffect` + optimistic press suffices (missing import crashes web)
- Raw `shadowColor` / `shadowOffset` on styles that render on web (use `crossShadow`)

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

- `references/web-rn-pitfalls.md` — **boxShadow, pointerEvents, springMotion, Metro HMR**
- `references/ios-26-liquid-glass-edge-shading.md` — **specular/Fresnel rim, web inset shadows, button wiring**
- `references/navigation.md` — NativeTabs vs JS tabs
- `references/ui-design.md` — Liquid Glass layers, performance caps
- `references/premium-feel.md` — press scale, haptics on tabs
- `references/platform-polish.md` — safe area, edge-to-edge
