# Performance — Lists, Storage, Images, Profiling (June 2026)

## FIX IN THIS ORDER

```
1. New Architecture + Hermes v1     (SDK 56 default)
2. React Compiler                   (experiments.reactCompiler)
3. Correct list component           (FlashList v2 / Legend List v3)
4. expo-image + blurhash            (not RN Image)
5. MMKV v4 for hot KV paths         (dev build)
6. expo-sqlite WAL for relational   (offline)
7. Route-level code splitting       (expo-router lazy routes)
8. Profile on low-end Android       (then optimize)
```

---

## LISTS — 2026 DECISION MATRIX

| Scenario | Component | Notes |
|---|---|---|
| &lt; ~100 static rows | `FlatList` | Fine for settings, short lists |
| Large feed, grid, masonry | **FlashList v2** | New Arch **required**; no size estimates |
| Chat, AI, variable height | **Legend List v3** | `@legendapp/list/react-native` |
| Bidirectional infinite | **Legend List v3** | `maintainVisibleContentPosition`, anchoring |
| Web virtual list | `@legendapp/list/react` | Same mental model |

---

## FLASHLIST v2 (Shopify)

**New Architecture only.** JS-only rewrite — no `estimatedItemSize`.

```powershell
npx expo install @shopify/flash-list
```

```tsx
import { FlashList } from '@shopify/flash-list';

<FlashList
  data={items}
  renderItem={({ item }) => <ItemRow item={item} />}
  keyExtractor={(item) => item.id}
  masonry={false}
  onEndReached={fetchNextPage}
  onEndReachedThreshold={0.5}
/>
```

**v2 changes vs v1:**
- No `estimatedItemSize`, `overrideItemLayout` for sizing hacks
- `masonry` is a prop (not `MasonryFlashList`)
- `maintainVisibleContentPosition` default on — great for chat-style feeds
- `inverted` deprecated — use MCP / anchoring patterns
- No `CellContainer` — use `View`

Error `FlashList v2 is only supported on new architecture` → enable New Arch, rebuild dev client.

Docs: https://shopify.github.io/flash-list/docs/v2-migration/

---

## LEGEND LIST v3 (LegendApp)

100% TypeScript, **no native linking** — works in more environments; best for dynamic sizing.

```powershell
npm install @legendapp/list
```

```tsx
import { LegendList } from '@legendapp/list/react-native';

<LegendList
  data={items}
  renderItem={({ item }) => <ItemRow item={item} />}
  keyExtractor={(item) => item.id}
  recycleItems
  maintainVisibleContentPosition
/>
```

**Chat / keyboard:**
```tsx
import { KeyboardChatLegendList } from '@legendapp/list/keyboard-chat';
```

**Reanimated list:**
```tsx
import { AnimatedLegendList } from '@legendapp/list/reanimated';
```

Migrate from FlashList: rename component + add `recycleItems` for similar recycling behavior.

---

## NEVER DO THIS

```tsx
{items.map((item) => <Row key={item.id} />)}  // unbounded lists
<FlatList data={thousands} />                  // measured jank at scale
<FlashList ... />  // missing keyExtractor
```

---

## MMKV v4 (Nitro Modules)

```powershell
npx expo install react-native-mmkv react-native-nitro-modules
```

Requires **dev build**. **Nitro major must match MMKV peer** — Nitro is not forwards-compatible (MMKV on 0.33+ will not build against 0.31).

**Version discipline:**
- Install only via `npx expo install react-native-mmkv react-native-nitro-modules`
- After install, verify `peerDependencies` in `react-native-mmkv/package.json`
- Align **all** Nitro consumers (`@rive-app/react-native`, etc.) to the same major
- Monorepos: hoist one `react-native-nitro-modules` at workspace root
- EAS error `NitroModules cannot be found` → bump Nitro to MMKV minimum, `prebuild --clean`, rebuild

Full checklist: `references/production-edge-cases.md`

```tsx
import { createMMKV } from 'react-native-mmkv';

export const storage = createMMKV();

storage.set('settings.theme', 'dark');
const theme = storage.getString('settings.theme');
```

**v4 notes (Marc Rousavy):**
- Nitro Module — faster native boundary
- Install `react-native-nitro-modules` explicitly
- Not for **auth tokens** — use `expo-secure-store`

### Zustand persist adapter

```tsx
import { createMMKV } from 'react-native-mmkv';
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

const mmkv = createMMKV();
const mmkvStorage = {
  getItem: (name: string) => mmkv.getString(name) ?? null,
  setItem: (name: string, value: string) => { mmkv.set(name, value); },
  removeItem: (name: string) => { mmkv.remove(name); },
};

export const useSettingsStore = create(
  persist(
    (set) => ({ /* ... */ }),
    { name: 'settings', storage: createJSONStorage(() => mmkvStorage) }
  )
);
```

---

## EXPO SQLITE (SDK 56)

```powershell
npx expo install expo-sqlite
```

```tsx
const db = await SQLite.openDatabaseAsync('app.db');
await db.execAsync(`PRAGMA journal_mode = WAL;`);
```

SDK 56: native `ArrayBuffer` blobs, session changesets, better large-file hashing.

Use for offline-first relational data — not for simple key-value (use MMKV).

---

## NATIVE MODULE PERFORMANCE

| Tool | When |
|---|---|
| **Inline Expo modules** (SDK 56) | Small native features beside TS |
| **Expo Modules** | Default custom native; Kotlin compiler plugin speeds Android |
| **Nitro Modules** | MMKV-class sync hot paths; third-party perf libs |

Do not reach for Nitro until Expo Modules path is profiled and insufficient.

---

## EXPO IMAGE

```powershell
npx expo install expo-image
```

- SDWebImage / Glide backends
- Memory-disk cache, blurhash, priority, transitions
- Use for every remote asset in lists (with fixed dimensions to prevent layout thrash)

---

## NETWORK — `expo/fetch` (SDK 56)

Default `globalThis.fetch` — faster, WinterTC-aligned.

**Sentry + fetch (do not combine blindly):**
- Default `expo/fetch` → `traceFetch: true` if spans missing
- `EXPO_PUBLIC_USE_RN_FETCH=1` → disable `traceFetch` (duplicate spans)

See `references/production-edge-cases.md` for conditional init.

---

## STARTUP

```tsx
import * as SplashScreen from 'expo-splash-screen';
SplashScreen.preventAutoHideAsync();
// hide only after: fonts + auth hydrate + critical queries
await SplashScreen.hideAsync();
```

Hermes bytecode diffing (EAS Update default SDK 56) improves **update** download size, not cold start — still optimize splash path.

---

## REACT COMPILER

```json
{ "expo": { "experiments": { "reactCompiler": true } } }
```

Removes most manual memoization — biggest win for re-render heavy screens.

---

## METRO / BUNDLE

```js
// production — drop console
config.transformer.minifierConfig = {
  compress: { drop_console: true },
};
```

SDK 56: on-demand filesystem default, native Node watcher (Watchman optional), fewer Hermes transforms → faster bundling.

---

## PROFILING

| Tool | Use |
|---|---|
| React Native Perf Monitor | JS/UI frame time |
| Reanimated dev tools | Animation jank |
| Hermes sampling profiler | CPU hotspots |
| Hermes heap snapshot | Retained objects / leak suspects after long sessions |
| Sentry performance | Production traces |
| EAS Observe | Coming — real-user metrics |

**Truth device:** low-end Android (Pixel 4a class). iOS is rarely the bottleneck.

### Hermes sampling profiler (CPU)

1. Dev build on device (not web).
2. Open in-app dev menu → **Open React DevTools** / perf tools available in your RN version, or use Metro-connected profiling.
3. Reproduce jank (scroll list 30s, tab switch loop, animation loop).
4. Capture sample during jank — look for heavy JS functions in list `renderItem`, JSON parse in scroll, or sync storage on scroll.

### Hermes heap snapshot (memory)

Use when: memory grows after navigating away from a heavy screen, or long voice/AI sessions.

1. Reproduce leak path (open screen → back → repeat 10×).
2. Take heap snapshot **before** and **after** in Hermes-compatible tooling (React Native DevTools / Flipper where supported for your SDK).
3. Compare retained objects — common RN leaks:
   - Timers / subscriptions not cleared in `useEffect` cleanup
   - Closures holding full chat history on unmounted screens
   - `Audio` / `expo-audio` players not `release()`’d
   - Global listeners (NetInfo, AppState) duplicated per mount
4. Fix root cause; re-snapshot — do not ship “we profiled once” without a fixed issue.

**Not required for 999 product stack** — required when you have a **symptom**. See `mastery-rubric.md`.

### List choice audit (per screen)

| Screen pattern | Component | Anti-pattern |
|----------------|-----------|--------------|
| Settings, legal, &lt; 20 rows static | `ScrollView` | FlashList “because senior” |
| 100–500 uniform cards | `FlashList` v2 | `ScrollView` + `.map` |
| Leaderboard / chat / variable height | `LegendList` v3 | FlashList + wrong height hacks |
| Small picker list in modal | `FlatList` or map | Nested FlashList inside FlashList |

**Agent rule:** Grep the screen file before recommending migration — credit correct list choice.

---

## BUILD-TIME PERF (SDK 56)

- iOS: `EXPO_USE_PRECOMPILED_MODULES=0` to opt out of prebuilt Expo modules
- Android: `usePrecompiledHeaders: true` in expo-build-properties (experimental)
- EAS: precompiled Reanimated/screens on iOS cloud builds

---

## GOLDEN RULES

1. **FlashList v2** for big lists — not FlatList by default at scale.
2. **Legend List** for chat/variable height — not inverted FlatList hacks.
3. **MMKV** for hot KV — not AsyncStorage.
4. **Measure** before Skia/Nitro/custom C++.
5. **expo-image** for all network images in lists.
