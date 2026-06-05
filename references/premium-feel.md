# Premium Feel — Perception Stack (June 2026)

Distilled from industry practice (including [Beto Moedano — *How I Make My Apps Feel Premium*](https://codewithbeto.dev/blog/how-i-make-apps-feel-premium), Jun 2026). **Not** a separate library stack — map each detail onto your existing RN + Expo choices.

**Core idea:** Premium ≠ flashy. It is many small, invisible decisions (press, motion, haptics, keyboard, loading) that compound.

---

## THE 5 PILLARS → STACK MAPPING

| Pillar | Senior rule | Default in this skill |
|--------|-------------|------------------------|
| **Press feedback** | ~100ms scale/spring before action fires; buttons feel physical | Reanimated `withSpring` ~0.96 on `onPressIn` (see `animations.md`) |
| **Subtle motion** | Animate only when motion answers “what changed?”; **150–300ms** | Reanimated / Moti; avoid parallax-everything |
| **Haptics** | Confirm **state changes** (submit, toggle, delete) — not scroll/nav noise | `expo-haptics` default; optional upgrades below |
| **Keyboard** | Layout tracks keyboard; submit stays visible; intentional dismiss | `react-native-keyboard-controller` (`platform-polish.md`) |
| **Loading / empty** | Skeleton/shimmer shape, not lone spinners; empty states teach next step | TanStack placeholder + Moti skeletons (`speed-perception.md`) |

---

## PRESS STATES

**iOS 26 + Liquid Glass:** System chrome (`expo-glass-effect`, Expo UI, NativeTabs) already gives strong press on supported builds — still add explicit press on custom CTAs.

**Android + older iOS:** Do not ship flat `Pressable` with zero feedback.

```tsx
// Default pattern (already in skill)
scale.value = withSpring(0.96, { damping: 20, stiffness: 300 });
```

**Optional shortcut:** [`pressto`](https://github.com/alantoa/pressto) — Reanimated + GH wrapper for consistent press animations on many buttons (use when you would otherwise skip press feedback).

---

## SUBTLE ANIMATIONS

- If you cannot name the user question the animation answers, **remove it**.
- Prefer **150–300ms** for UI transitions; longer reads as “showing off”.
- **Do not** default to `react-native-ease` for greenfield Expo apps (niche; Reanimated/Moti + docs win — see `animations.md`).
- iOS 26 Liquid Glass tab transitions: prefer **NativeTabs** in dev builds (`navigation.md`). Use **floating glass JS tab bar** (`floating-glass-tab-bar.md`) only when design needs a frosted pill over mesh/hero UI — still add press scale on tab items.

---

## HAPTICS

| When | Do |
|------|-----|
| Form success / error | `notificationAsync` Success / Error |
| Toggle, picker commit, destructive confirm | `impactAsync` Light or Medium |
| Tab change, list scroll, idle taps | **No** haptic |

**Defaults:** `expo-haptics` (Expo Go + cross-platform, SDK 56 web where supported).

**Optional dev-build upgrades** (Nitro/native; not Expo Go):

| Library | When |
|---------|------|
| `react-native-haptics` | Lower latency on iOS |
| `react-native-pulsar` (Software Mansion) | Presets, custom patterns, worklet-safe triggers — ex-Expo teams use for “physical” apps |

Never haptic every interaction — noise feels cheap.

---

## KEYBOARD

Treat keyboard as a first-class layout concern:

```powershell
npx expo install react-native-keyboard-controller
```

- `KeyboardAwareScrollView` / provider patterns from library docs
- `softwareKeyboardLayoutMode: "resize"` in `app.json` (Android)
- Drag-to-dismiss: `react-native-gesture-handler` + blur/focus on pan end (worklet → `scheduleOnRN` for focus)

See `platform-polish.md`.

---

## LOADING & EMPTY STATES

- **Skeleton** with shimmer ≈ final layout (FlashList `ListEmptyComponent` skeleton, not spinner-only home).
- **Empty:** headline + why empty + primary CTA (never blank `FlatList`).
- AI/long waits: cycling status copy + shimmer text beats a static spinner.

See `speed-perception.md` and `ux-psychology.md` (Peak-End Rule).

---

## CHECKLIST BEFORE SHIP

- [ ] Primary buttons have press scale or native glass press
- [ ] No animation without a user-visible state change
- [ ] Haptics only on commits / errors / destructive actions
- [ ] Chat/forms use keyboard controller; submit visible while typing
- [ ] First paint uses skeleton; empty routes have CTA
- [ ] NativeTabs on iOS/Android dev builds unless deliberate JS tab design

---

## WHAT NOT TO COPY BLINDLY

- Promoting a single animation library as the whole stack — stay on Reanimated 4 + GH baseline.
- Replacing `expo-haptics` in Expo Go workflows — Pulsar/Haptics need dev builds.
- Heavy blur stacks on lists — performance + Liquid Glass layer limits (`ui-design.md`).
