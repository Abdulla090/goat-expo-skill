# iOS 26 Liquid Glass — Edge Shading & RN Strategy (June 2026)

**Scope:** Universal Expo / React Native guidance for reproducing **iOS 26 default edge shading** on floating controls, tab bars, and neutral buttons — without fighting the system on native iOS.

**Authority:** [WWDC25 — Meet Liquid Glass](https://developer.apple.com/videos/play/wwdc2025/219/), [Build a SwiftUI app with the new design](https://developer.apple.com/videos/play/wwdc2025/323/), [Build a UIKit app with the new design](https://developer.apple.com/videos/play/wwdc2025/284/), `expo-glass-effect` docs.

---

## WHAT iOS 26 DOES (SYSTEM DEFAULT)

Liquid Glass is **not** “blur + white overlay.” Apple composes multiple optical layers:

| Layer | What the user sees | System behavior |
|---|---|---|
| **Refraction / lensing** | Content behind warps slightly | Real-time rendering; strongest on `GlassEffect` / `UIGlassEffect` |
| **Specular highlights** | Bright arc along top + rim catch lights | Virtual **overhead light**; geometry-aware (capsule curves catch more) |
| **Fresnel rim** | Left/right edges glow at grazing angles | More reflection, less transmission at shallow view angles |
| **Bottom inner shade** | Subtle darkening along lower inner edge | Reads as glass **thickness** / lens profile |
| **Adaptive tint** | Glass lightens or darkens with wallpaper & content | `colorScheme` + content behind |
| **Adaptive shadow** | Float shadow stronger over text/busy areas, softer on flat white | Separation without muddying light UIs |
| **Scroll edge effect** | Content under bars dissolves (blur + fade) | **Separate** from glass material — `UIScrollEdgeElementContainerInteraction` / SwiftUI scroll edge |

**Critical distinction:**

- **Glass edge shading** = on the **control surface** (pill, FAB, chip).
- **Scroll edge effect** = on **content scrolling under** floating chrome (system applies to scroll views under toolbars).

Do **not** stack opaque backgrounds behind glass bars — Apple explicitly says extra backgrounds **break** scroll edge legibility.

---

## WHEN TO USE NATIVE vs CUSTOM EDGE

```
iOS 26+ dev build + isGlassEffectAPIAvailable()?
  yes → GlassView / Expo UI glassEffect
        → System draws specular + rim
        → DO NOT duplicate 6 custom gradient rims on top (kills “magic”)
        → Optional: float shadow only
  no  → Custom LiquidGlassEdgeShading stack (this doc)
```

| Platform | Material | Edge shading |
|---|---|---|
| **iOS 26+** (`expo-glass-effect`) | `GlassView` `glassEffectStyle="regular"` | **System default** — skip custom rims |
| **iOS fallback** | `BlurView` `systemChromeMaterialLight` | **Custom** edge stack |
| **Android** | Layered frost (no `BlurView` black bar) | **Custom** edge stack (required) |
| **Web** | `backdrop-filter` + pearl-slate tint | **Custom** edge stack + **inset box-shadow** |

Always guard: `isGlassEffectAPIAvailable()` — API missing on some betas/devices.

Respect `AccessibilityInfo.isReduceTransparencyEnabled()` → solid fallback surface.

---

## CUSTOM EDGE SHADING — LAYER STACK (Z-ORDER)

Apply **only** on web, Android, and iOS without Glass API. Bottom → top:

```
┌─────────────────────────────────────────────┐
│ 6. Content (icons, labels)                  │ z: 5
│ 5. Top sheen gradient (soft, optional)      │ z: 1
│ 4. Dual rim (outer slate + inner white)     │ z: 3–4
│ 3. Top hairline (1px catch light)           │ z: 4
│ 2. Bottom inner shade gradient              │ z: 2
│ 1b. Lateral Fresnel (left + right 15–20%)   │ z: 2
│ 1a. Specular top arc (38–45% height)        │ z: 2
│ 0. Backdrop (blur / frost / GlassView)     │ z: 0
└─────────────────────────────────────────────┘
+ Outer float shadow (not inset) on shell
+ Web: inset highlight top + inset shade bottom in box-shadow
```

### Token targets (light chrome on white apps)

Pearl-slate frost — **not** pure white (invisible on `#FFF` screens):

```ts
export const LIQUID_GLASS = {
  frostWeb: "rgba(226, 232, 240, 0.58)",
  frostUnderlay: "rgba(203, 213, 225, 0.38)",
  edgeSpecular: [
    "rgba(255, 255, 255, 0.92)",
    "rgba(255, 255, 255, 0.32)",
    "rgba(255, 255, 255, 0)",
  ],
  edgeTopLine: "rgba(255, 255, 255, 0.98)",
  edgeFresnel: "rgba(255, 255, 255, 0.38)",
  edgeBottomShade: "rgba(100, 116, 139, 0.16)",
  border: "rgba(148, 163, 184, 0.44)",
  borderInner: "rgba(255, 255, 255, 0.58)",
} as const;
```

### Web backdrop (true glass, not plastic)

```tsx
{
  backgroundColor: LIQUID_GLASS.frostWeb,
  backdropFilter: "blur(24px) saturate(165%) contrast(1.04)",
  WebkitBackdropFilter: "blur(24px) saturate(165%) contrast(1.04)",
}
```

`expo-blur` on web stacks opaque `backgroundColor` — prefer thin custom tint + `backdrop-filter`.

### Web inset edge (iOS 26–style in one paint)

```ts
boxShadow: [
  "0 0 0 1px rgba(148, 163, 184, 0.28)",      // outer rim ring
  "0 8px 28px rgba(15, 23, 42, 0.1)",         // float
  "inset 0 1.5px 0 rgba(255, 255, 255, 0.95)", // top catch
  "inset 0 -1px 0 rgba(100, 116, 139, 0.1)",  // bottom shade
].join(", ")
```

---

## REACT NATIVE COMPONENT PATTERN

**Reference implementation (any app):**

```
src/constants/liquid-glass.ts          — tokens + shellShadow()
src/components/LiquidGlassSurface.tsx  — shell + edge stack
src/components/LiquidGlassEdgeShading.tsx (or export from Surface)
src/utils/liquid-glass-color.ts        — prefersLiquidGlass() / isPrimaryBlue()
```

```tsx
export function LiquidGlassSurface({
  borderRadius,
  children,
  edgeShading, // default: !nativeGlass
}: Props) {
  const nativeGlass = Platform.OS === 'ios' && isGlassEffectAPIAvailable();

  return (
    <View style={[shell, liquidGlassShellShadow('button')]}>
      <FrostWash minimal={nativeGlass || Platform.OS === 'web'} />
      {Platform.OS === 'web' && <WebLiquidBackdrop />}
      {nativeGlass && <GlassView glassEffectStyle="regular" colorScheme="light" isInteractive />}
      {edgeShading !== false && !nativeGlass && (
        <LiquidGlassEdgeShading borderRadius={borderRadius} />
      )}
      {children}
    </View>
  );
}
```

### Wire to buttons (not blue CTAs)

| Component | Strategy |
|---|---|
| **Neutral** `SoftPressableButton` / chips | Auto `prefersLiquidGlass(faceColor)` → `LiquidGlassSurface` |
| **Primary blue CTA** | Solid fill + top sheen only (`HomeLiquidButton` pattern) — **no** glass stack |
| **Tab bar pill + FAB** | `LiquidGlassSurface` `shadowDepth="tab"` |
| **PremiumPressable** | `glass` prop for icon circles |
| **PressableScale** | `glass` + `glassRadius` for settings rows |

```ts
export function isPrimaryBlueButton(color: string): boolean {
  // #2B59F3, #0A84FF, rgba(43,89,243,…)
}
export function prefersLiquidGlass(faceColor: string): boolean {
  return !isPrimaryBlueButton(faceColor) && isNeutralFaceColor(faceColor);
}
```

---

## SCROLL EDGE EFFECT (FLOATING CHROME + LISTS)

When a **floating glass tab bar** or bottom FAB row sits over scrolling content:

1. **Do not** add opaque `backgroundColor` on the navigator `tabBarStyle` — keep transparent.
2. Add **scroll padding** so last items clear the float: `tabBarScrollPadding(insets.bottom)`.
3. On **native** iOS 26, prefer system scroll edge where possible; for custom bottom float clusters, UIKit uses `UIScrollEdgeElementContainerInteraction` with `edge = .bottom`.
4. RN custom equivalent (approximation): subtle **content fade** near bottom safe area is optional; do not double-blur entire list.

Dense UIs (many floats): Apple offers **`.hard` scroll edge style** — closer to iOS 18 solid bar; use sparingly.

---

## ACTIVE INDICATOR (TAB BAR)

iOS 26 floating tabs often use a **circle** or **capsule** active chip inside the glass pill:

- Animate with `useSharedValue` + **`springMotion` on press** (optimistic) + **`useEffect`** when `state.index` catches up (`web-rn-pitfalls.md` — avoid `useAnimatedReaction` for tab sync).
- Keep indicator **mounted**; fade `opacity` when on non-pill routes (e.g. profile FAB).
- Snap position on `slotWidth` reflow (rotation) without spring.

---

## ANTI-PATTERNS

| Don't | Why |
|---|---|
| 6+ gradient layers **on top of** native `GlassView` | Apple: “sacrifices the magic” |
| Pure `rgba(255,255,255,0.9)` on white app bg | Reads as invisible / plastic |
| `BlurView` on Android tab bar without dark-mode test | Solid black bar |
| Glass inside FlashList / LegendList rows | GPU death |
| Liquid Glass on **primary content** cards | Reserved for **navigation & controls** per HIG |
| Only `backdrop-filter: blur(10px)` | Flat smudge — no Fresnel, no rim, not iOS 26 |

---

## CHECKLIST (AGENT PRE-SHIP)

- [ ] iOS 26 device: `GlassView` path tested; custom rims disabled when native
- [ ] Web: `backdrop-filter` + inset shadows visible on white background
- [ ] Android: frost layers + edge rims (no black blur)
- [ ] Blue primary buttons still solid brand fill
- [ ] Neutral buttons use shared `LiquidGlassSurface`
- [ ] Tab indicator moves on **first** tap (optimistic `pillX` + `springMotion`)
- [ ] Web: no `shadow*` deprecation warnings (`crossShadow` / `boxShadow`)
- [ ] Decorative glass layers use `style.pointerEvents: 'none'`
- [ ] ≤3 glass surfaces visible per screen
- [ ] Reduce Transparency → opaque fallback

---

## RELATED

- `references/ui-design.md` — Liquid Glass layer order
- `references/floating-glass-tab-bar.md` — pill + FAB layout
- `references/premium-feel.md` — press scale, haptics
- `references/platform-polish.md` — safe area, edge-to-edge

**Phingo reference files:** `src/components/LiquidGlassSurface.tsx`, `src/constants/liquid-glass.ts`
