# Web + RN New Architecture Pitfalls (June 2026)

Production lessons from **Expo Router web** + **react-native-web** + **Reanimated 4** on RN 0.83+ / SDK 56.

**When to read:** Console shows `shadow* deprecated`, `pointerEvents` deprecated, Reanimated easing warnings, `ReferenceError: X is not defined` after HMR, or tab bar crashes only on web.

---

## 1. Deprecated style props (RN 0.83+ web)

React Native Web (New Architecture) warns and will remove:

| Deprecated | Use instead |
|---|---|
| `shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius` | `boxShadow` (web) — keep `shadow*` on native only |
| `pointerEvents` **as a View prop** | `style={{ pointerEvents: 'none' }}` (or in `StyleSheet`) |

### `crossShadow()` utility (repo pattern)

Never spread raw `shadow*` in shared `StyleSheet.create` if the style runs on web.

```ts
// src/utils/shadows.ts
import { Platform, ViewStyle } from "react-native";

export function crossShadow({
  color = "#000",
  offsetX = 0,
  offsetY = 4,
  blur = 12,
  spread = 0,
  opacity = 0.1,
  elevation = 4,
} = {}): ViewStyle {
  if (Platform.OS === "web") {
    const rgba = hexToRgba(color, opacity);
    return { boxShadow: `${offsetX}px ${offsetY}px ${blur}px ${spread}px ${rgba}` } as ViewStyle;
  }
  return {
    shadowColor: color,
    shadowOffset: { width: offsetX, height: offsetY },
    shadowOpacity: opacity,
    shadowRadius: blur,
    elevation,
  };
}
```

**Usage in StyleSheet:**

```ts
activeChip: {
  borderRadius: 999,
  ...crossShadow({ color: "#2B59F3", offsetY: 2, blur: 6, opacity: 0.12 }),
},
```

**Multi-layer web shadows** (liquid glass shell): build a `boxShadow` string with comma-separated layers + `inset` — see `liquidGlassShellShadow()` in `references/ios-26-liquid-glass-edge-shading.md`.

### `pointerEvents` migration

```tsx
// ❌ Web deprecation warning
<View pointerEvents="none" style={styles.overlay} />

// ✅ Preferred everywhere (web + native)
<View style={[styles.overlay, { pointerEvents: "none" }]} />

// For decorative glass edge layers — put pointerEvents in StyleSheet once:
specularArc: { position: "absolute", pointerEvents: "none", /* ... */ },
```

Apply to: tab bar host (`box-none`), glass edge gradients, mesh backgrounds, animated indicators that must not steal taps.

---

## 2. Reanimated on web

### Custom spring easing is not supported

Console: `[Reanimated] Selected easing is not currently supported on web. Using linear easing instead.`

**Fix:** Branch animations — `withSpring` on iOS/Android, `withTiming` on web.

```ts
// src/utils/motion-spring.ts
import { Platform } from "react-native";
import { Easing, withSpring, withTiming, type WithSpringConfig } from "react-native-reanimated";

export function springMotion(to: number, config?: WithSpringConfig) {
  if (Platform.OS === "web") {
    return withTiming(to, { duration: 220, easing: Easing.out(Easing.cubic) });
  }
  return withSpring(to, config);
}
```

Use for: floating tab active circle, path switcher chip, any `withSpring(Motion.soft)` on shared navigators.

**Do not** call `Platform.OS` inside worklets for this pattern — assign from JS thread (`pillX.value = springMotion(x)` in `useEffect` / press handlers).

### Tab indicator: prefer `useEffect` + optimistic press — not `useAnimatedReaction`

`useAnimatedReaction` is valid Reanimated API but **overkill** for bottom-tab index sync and has caused agent mistakes (missing import → runtime crash).

**Canonical pattern (floating glass tab bar):**

```tsx
const optimisticPress = useRef(false);
const prevFocusedIndex = useRef(focusedIndex);

const moveIndicator = useCallback((index: number, animated: boolean) => {
  const x = pillTargetX(index);
  pillX.value = animated ? springMotion(x) : x;
}, [pillTargetX, pillX]);

// On tab press — animate immediately (feels instant)
const navigate = (route: string, isFocused: boolean) => {
  if (!isFocused) {
    const nextIdx = pillTabs.findIndex((t) => t.route === route);
    if (nextIdx >= 0) {
      optimisticPress.current = true;
      moveIndicator(nextIdx, true);
      indicatorOpacity.value = 1;
    }
    navigation.navigate(route);
  }
};

// When navigation state catches up — skip double animation if we already moved optimistically
useEffect(() => {
  indicatorOpacity.value = springMotion(focusedIndex >= 0 ? 1 : 0);
  if (optimisticPress.current) {
    optimisticPress.current = false;
    prevFocusedIndex.current = focusedIndex;
    return;
  }
  const indexChanged = prevFocusedIndex.current !== focusedIndex;
  prevFocusedIndex.current = focusedIndex;
  moveIndicator(focusedIndex, indexChanged);
}, [focusedIndex, moveIndicator]);
```

**Why:** First tap animates on press; return visits animate when index changes; no reaction hook import to forget.

---

## 3. Stale Metro / HMR false crashes

### Symptoms

| Error | Often means |
|---|---|
| `ReferenceError: useAnimatedReaction is not defined` | Source **removed** the hook but web bundle is **stale** |
| `ReferenceError: TAB_BAR_TOP_PADDING is not defined` | Renamed/removed constant; HMR partial update |
| `Expo CLI and the web client are out of sync. Reload to reconnect.` | Metro restarted or failed mid-bundle |
| `Cannot record touch end without a touch start` | Usually crash/HMR side effect — retest after clean load |

**Before debugging code:** confirm grep shows zero matches in `src/` for the missing symbol.

### Recovery (Windows / PowerShell)

```powershell
# Stop dev server (Ctrl+C), then:
npx expo start --web --clear
# Or project script:
.\scripts\clear-metro-cache.ps1
```

Browser: **hard refresh** (`Ctrl+Shift+R`) or close tab and reopen `localhost:8081`.

**Agent rule:** If error references removed API and user saw HMR/out-of-sync messages → prescribe cache clear **first**, then code fix.

---

## 4. Removed / deprecated layout constants

When renaming tab-bar layout tokens:

- Re-export deprecated names as `0` or alias **or** grep entire repo before delete.
- HMR can leave old closures referencing deleted module bindings.

```ts
// layout.ts — safe deprecation
export const TAB_BAR_TOP_PADDING = 0; // deprecated; FAB aligns via TAB_BAR_INNER_HEIGHT
```

---

## 5. Debugging matrix (web console)

| Console message | Fix |
|---|---|
| `shadow*" style props are deprecated` | `crossShadow()` or `boxShadow` on web branch |
| `props.pointerEvents is deprecated` | Move to `style.pointerEvents` |
| Reanimated easing not supported on web | `springMotion()` / `withTiming` on web |
| `useAnimatedReaction is not defined` | Stale bundle **or** missing import — grep `src/` |
| Touch bank empty on touch end | Retest after clean reload; check overlapping `pointerEvents` |

---

## 6. Pre-ship checklist (web)

- [ ] Grep `src/` for `pointerEvents="` on Views — migrate decorative layers to style
- [ ] Grep `shadowColor` in shared styles — wrap with `crossShadow` or platform branch
- [ ] Tab/path sliding indicators use `springMotion` (or explicit web `withTiming`)
- [ ] Floating tab bar uses optimistic press + `useEffect`, not orphaned `useAnimatedReaction`
- [ ] `liquidGlassShellShadow()` uses `boxShadow` on web (inset + float layers)
- [ ] Clean `expo start --web --clear` smoke test after tab bar / glass changes

---

## RELATED FILES

- `references/floating-glass-tab-bar.md` — tab bar structure + animation
- `references/ios-26-liquid-glass-edge-shading.md` — inset `boxShadow` stack
- `references/animations.md` — Reanimated 4 + web branch
- `references/error-boundaries.md` — crash vs stale bundle triage
- `references/production-edge-cases.md` — production edge case index

---

## AGENT WORKFLOW (keep skill current)

When fixing a **web-only** RN deprecation or Reanimated quirk in a repo:

1. Fix in code with a **reusable util** (`crossShadow`, `springMotion`) when pattern repeats.
2. Update **this file** with symptom → fix row if new.
3. Update `floating-glass-tab-bar.md` / `animations.md` if pattern is tab/motion specific.
4. Sync copy to global `rn-expo-stack` skill if project mirrors it.
