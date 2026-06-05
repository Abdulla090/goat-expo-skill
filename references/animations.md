# Animations — Reanimated 4.4 + Worklets (June 2026)

## THE RULE

**JS-thread animation = jank on real devices.**  
**UI-thread worklets / native drivers = production quality.**

RN 0.85 adds a **new animation backend** for New Architecture — it improves core RN animations but **does not replace** Reanimated for gestures and complex UI.

---

## LIBRARY DECISION MAP

```
Gesture-driven (pan, pinch, scroll-linked)     → Reanimated 4 + Gesture Handler
Mount / unmount / presence                     → Moti (Reanimated under the hood)
Designer motion (After Effects)                → lottie-react-native
Screen hero / shared transitions               → Reanimated SharedTransition
Custom drawing (particles, charts)             → @shopify/react-native-skia
Simple mount fade (no gestures)                → Moti or Reanimated withTiming
```

**Do not default to `react-native-ease`** for new apps — niche; Reanimated/Moti cover 95% of cases with better Expo docs and community.

**Duration:** Most UI transitions **150–300ms**; longer feels decorative.

**Optional:** [`pressto`](https://github.com/alantoa/pressto) for consistent press-scale on many buttons if you skip per-component Reanimated press styles.

**Philosophy:** See `premium-feel.md` — animate only when motion clarifies what changed.

---

## REANIMATED 4.4 + WORKLETS (mandatory pair)

Reanimated 4.x = **New Architecture only**. v3 is frozen.

```powershell
npx expo install react-native-reanimated react-native-worklets
```

Expo SDK 50+ includes **worklets Babel plugin** in `babel-preset-expo`. If customizing Babel, **`react-native-worklets/plugin` must be last** (Reanimated docs).

### Core pattern

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

function PressableCard() {
  const scale = useSharedValue(1);
  const style = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <Animated.View style={[styles.card, style]}>
      <Pressable onPress={() => { scale.value = withSpring(0.96); }}>
        <Text>Press</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### Reanimated 4.4 highlights (Software Mansion)
- **`useTimestamp`** — frame-synced shared value for time-based animations
- iOS **Core Animation CSS engine** (feature-flagged) — platform-backed CSS animations
- **Animation Backend** integration with shadow tree updates (flagged, evolving)
- Android **precompiled headers** — faster native builds

Pin versions with `npx expo install` — do not float major versions independently.

Docs: https://docs.swmansion.com/react-native-reanimated/

---

## GESTURE HANDLER

Version is **SDK-pinned** (e.g. ~2.31 with Expo 56) — not a separate "v3 product name"; use hook API (`GestureDetector`, `Gesture.Pan()`).

```tsx
// app/_layout.tsx — required at root
import { GestureHandlerRootView } from 'react-native-gesture-handler';

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Stack />
    </GestureHandlerRootView>
  );
}
```

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';

function Draggable() {
  const x = useSharedValue(0);
  const gesture = Gesture.Pan()
    .onUpdate((e) => { x.value = e.translationX; })
    .onEnd(() => { x.value = withSpring(0); });
  const style = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }));
  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={style} />
    </GestureDetector>
  );
}
```

Always pair gestures with Reanimated shared values — never `Animated` from `react-native` for new code.

---

## EXPO UI + WORKLETS

Expo UI supports **`WorkletCallback`** props for synchronous native-controlled inputs (e.g. `TextField` value) — use for glass-native forms without flicker.

---

## MOTI — PRESENCE

```powershell
npm install moti
```

```tsx
import { MotiView } from 'moti';
import { AnimatePresence } from 'moti';

<AnimatePresence>
  {visible && (
    <MotiView
      from={{ opacity: 0, translateY: 12 }}
      animate={{ opacity: 1, translateY: 0 }}
      exit={{ opacity: 0 }}
      transition={{ type: 'timing', duration: 200 }}
    />
  )}
</AnimatePresence>
```

Stagger list entrances with `delay: index * 40` — cap total delay for long lists.

---

## LOTTIE

```powershell
npx expo install lottie-react-native
```

Keep JSON **&lt; ~100KB** when possible. Runs on native Lottie engines — not JS thread during playback.

---

## SKIA

```powershell
npx expo install @shopify/react-native-skia
```

**Use when standard Views hit a ceiling:**

| Use Skia (GPU canvas) | Stay on Views + Reanimated |
|-----------------------|----------------------------|
| Real-time blur/glow shaders, particles | Standard lists, forms, navigation |
| Large interactive charts (1000s of points) | Simple SVG icons (prefer vector icons / expo-image) |
| Gesture-driven strokes (velocity → width) | Press scale / tab transitions |
| Custom data viz not available in Expo UI | Anything inside FlashList/LegendList **rows** (too heavy) |

Skia draws on a **secondary GPU path** alongside normal layout — keep nav/lists native; reserve Skia for hero visuals, charts, illustrations.

Skia reads Reanimated shared values directly — pair for 120fps custom draw, not for every card border radius.

**Not for:** standard buttons, lists, remote images.

---

## SPRING PRESETS

```tsx
withSpring(v, { damping: 15, stiffness: 150 }); // snappy UI
withSpring(v, { damping: 20, stiffness: 90 });  // smooth, low bounce
withSpring(v, { damping: 10, stiffness: 100 }); // playful
```

Exit animations: **shorter duration than enter** (speed perception).

---

## SHARED ELEMENT TRANSITIONS

```tsx
<Animated.Image sharedTransitionTag={`photo-${id}`} source={{ uri }} />
```

Tags must be unique per item. Test on physical device — simulators lie about GPU load.

---

## EXPO UI GLASS + ANIMATION

Limit stacked `glassEffect` / `GlassView` layers when running scroll-linked Reanimated — profile FPS.

---

## CHECKLIST BEFORE SHIP

- [ ] No `useNativeDriver: false` on legacy `Animated` API
- [ ] `GestureHandlerRootView` at root
- [ ] Worklets/Reanimated Babel plugin order correct
- [ ] Glass layers ≤ 3 per screen
- [ ] Lottie files optimized
- [ ] Tested on low-end Android + iOS device
- [ ] `__DEV__` FPS monitor only: Reanimated dev tools

---

## ANTI-PATTERNS

- `LayoutAnimation` on large lists (use Reanimated layout or list-optimized patterns)
- Animating width/height of 50+ list items every frame
- Running Moti on every cell in a 1000-row list without virtualization
- Ignoring `useReducedMotion()` / accessibility reduce motion

```tsx
import { useReducedMotion } from 'react-native-reanimated';
const reduced = useReducedMotion();
// Skip decorative motion when true
```
