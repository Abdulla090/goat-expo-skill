# Production Edge Cases — 945 → 999 (June 2026)

Senior-stack refinements that separate “accurate SDK 56 docs” from “ship-proof production config.”

**999 skill bar:** Sentry/fetch/Nitro/Query split + rejection rules for AI code. **Custom JSI is not part of this file** — see `native-modules.md`.

---

## 1. Sentry + `expo/fetch` + `traceFetch`

### Background (SDK 56)

- `globalThis.fetch` defaults to **`expo/fetch`** (WinterTC, compression, better OTA/Sentry story on paper).
- Some libraries or APM setups still expect the legacy RN fetch/XHR path.

### The rule

| App fetch mode | Sentry config |
|---|---|
| Default (`expo/fetch`, no env override) | `reactNativeTracingIntegration({ traceFetch: true })` + `breadcrumbsIntegration({ fetch: true })` |
| Legacy (`EXPO_PUBLIC_USE_RN_FETCH=1`) | **`traceFetch: false`** (or omit integration) — RN fetch is already instrumented differently; `traceFetch: true` can **duplicate** HTTP spans and breadcrumbs |

### Recommended init (conditional)

```tsx
import * as Sentry from "@sentry/react-native";

const useLegacyRnFetch = process.env.EXPO_PUBLIC_USE_RN_FETCH === "1";

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,
  environment: __DEV__ ? "development" : "production",
  tracesSampleRate: __DEV__ ? 1 : 0.15,
  integrations: [
    Sentry.reactNativeTracingIntegration({
      traceFetch: !useLegacyRnFetch,
    }),
    Sentry.breadcrumbsIntegration({ fetch: true }),
  ],
});
```

### Debugging matrix

| Symptom | Likely cause | Fix |
|---|---|---|
| No `http.client` spans, using `expo/fetch` | Sentry not tracing expo fetch | `traceFetch: true` |
| Duplicate network spans | `traceFetch: true` **and** `EXPO_PUBLIC_USE_RN_FETCH=1` | Disable `traceFetch` when on RN fetch |
| GraphQL/API visible in dev but not prod | DSN missing, sampling, or wrong fetch mode | Verify env + integration table above |

**Do not** set both “force RN fetch” and “force traceFetch for expo” without understanding they target different stacks.

---

## 2. MMKV v4 + `react-native-nitro-modules`

### Background

- **MMKV v4** is a **Nitro Module** (Marc Rousavy / Margelo).
- **`react-native-nitro-modules`** is a **required peer**, not optional.
- Nitro is **backwards compatible** within a major line but **not forwards compatible**: MMKV built on Nitro 0.33+ will not run with Nitro 0.31 in `node_modules`.

### Install (only correct path)

```powershell
npx expo install react-native-mmkv react-native-nitro-modules
```

Never hand-pick Nitro from a blog post — read **MMKV’s `peerDependencies`** after install.

### Version alignment checklist

- [ ] `npx expo install` resolved both packages (Expo SDK lockfile)
- [ ] `react-native-nitro-modules` satisfies MMKV peer range
- [ ] Other Nitro consumers agree (e.g. **`@rive-app/react-native`** often requires `>=0.35.0 <0.36`)
- [ ] One Nitro major across the **whole** app — no duplicate versions from nested deps
- [ ] New **dev build** after adding MMKV (not Expo Go)

### Monorepos / workspaces

- Hoist **`react-native-nitro-modules`** to the workspace root `package.json`.
- Single `node_modules` resolution for Nitro — duplicate majors cause “NitroModules cannot be found” or link errors.
- Run native build from the **app package** that owns `android/` / `ios/` after prebuild.

### Android EAS failures

| Error | Fix |
|---|---|
| `NitroModules cannot be found` | Bump `react-native-nitro-modules` to MMKV’s required minimum (often 0.33+) |
| Worked last week, fails now | Nitro **not forwards compatible** — align to MMKV peer, don’t downgrade Nitro alone |
| Clean build still fails | `npx expo prebuild --clean`, clear Gradle cache, rebuild; enable `EAS_GRADLE_CACHE=1` for speed not as first fix |

```json
// eas.json — build perf (optional)
{
  "build": {
    "production": {
      "env": { "EAS_GRADLE_CACHE": "1" }
    }
  }
}
```

### Quick-linking / custom native dirs

- After `expo prebuild`, open Android Studio once if autolinking looks stale.
- Custom native modules + Nitro: keep **one** `react-native-nitro-modules` version in the dependency tree; document it in the app README for CI.

### Security reminder

MMKV is for **non-secret** KV (settings, caches, flags). **Auth tokens → `expo-secure-store` only.**

---

## 3. Data-layer split (why this scores 945+)

This stack intentionally separates:

```
TanStack Query  → server state (network)
Zustand         → UI / transient client state
MMKV v4         → fast persisted non-secrets (native)
expo-secure-store → secrets / session material
expo-sqlite     → relational offline data
```

That split prevents the most common production bugs: tokens in MMKV, server state duplicated in Zustand, or “one AsyncStorage blob” corruption taking down the whole app.

---

## 4. When reviewing AI-generated code

Reject or fix if you see:

- `traceFetch: true` in the same project doc as `EXPO_PUBLIC_USE_RN_FETCH=1` without conditional logic
- MMKV for `access_token` / `refresh_token`
- `estimatedItemSize` on **FlashList v2**
- `@react-navigation/native` imports with Expo Router 56+ app code
- `LayoutAnimation` on long lists
- Store Expo Go as the only test environment for NativeTabs / MMKV / Nitro

---

## 5. Haptics helper (one gate for settings)

Do not scatter raw `expo-haptics` without respecting user preference.

```tsx
// lib/haptics.ts
import * as Haptics from "expo-haptics";
import { Platform } from "react-native";

let enabled = true;
export function setHapticsEnabled(v: boolean) {
  enabled = v;
}

export function hapticImpact(style: Haptics.ImpactFeedbackStyle = Haptics.ImpactFeedbackStyle.Light) {
  if (!enabled || Platform.OS === "web") return;
  void Haptics.impactAsync(style);
}

export function hapticNotification(type: Haptics.NotificationFeedbackType) {
  if (!enabled || Platform.OS === "web") return;
  void Haptics.notificationAsync(type);
}
```

Wire `setHapticsEnabled` from Zustand/settings on app boot. Use on submit/toggle/destructive only (`premium-feel.md`).

---

## 6. `expo-audio` (SDK 56 — replace `expo-av`)

| Old (`expo-av`) | New (`expo-audio`) |
|-----------------|-------------------|
| `Audio.Sound.createAsync` | `createAudioPlayer` / `useAudioPlayer` |
| Recording via `Audio.Recording` | `AudioModule`, `requestRecordingPermissionsAsync` |
| Global audio mode | `setAudioModeAsync` |

```powershell
npx expo install expo-audio
# Remove expo-av from dependencies when migration complete
```

- Voice tutors, TTS playback, lesson SFX → **`expo-audio`** in dev build.
- **Release players** on blur/unmount to avoid Hermes heap growth (`performance.md`).
- Web: verify recording APIs — may need feature flag or web-specific path.

---

## 7. Execution audit pointer

Full checklist (lists, Query, anti-slop, profiling): **`references/mastery-rubric.md`**.

Agents scoring “920+” must run that checklist against the **repo**, not `package.json` alone.

---

## 8. Expo web — RN deprecations + stale Metro

**Full guide:** `references/web-rn-pitfalls.md`

| Production symptom | Senior fix |
|---|---|
| `shadow* style props are deprecated` | `crossShadow()` utility — `boxShadow` on web only |
| `props.pointerEvents is deprecated` | `style.pointerEvents` on decorative overlays / glass layers |
| Reanimated easing not supported on web | `springMotion()` — `withTiming` on web, `withSpring` native |
| `useAnimatedReaction is not defined` (grep clean) | **Stale HMR** — `expo start --web --clear`, hard refresh |
| Tab indicator dead on first tap | Optimistic press + `useEffect`; don’t rely on reaction-only sync |

**Agent workflow:** After fixing web console issues in a repo, update `web-rn-pitfalls.md` + linked references so the next session doesn’t reintroduce the bug.

**All platforms:** Non-obvious or AI-repeatable bugs → append `field-bug-playbook.md` (`agentic-workflows.md`).

---

## References

- Expo SDK 56 changelog — `expo/fetch` default
- `expo-audio` — https://docs.expo.dev/versions/latest/sdk/audio/
- [Sentry React Native — Expo fetch discussion](https://github.com/getsentry/sentry-react-native/issues/6212)
- [MMKV v4 + Nitro upgrade guide](https://github.com/mrousavy/react-native-mmkv/blob/main/docs/V4_UPGRADE_GUIDE.md)
- Nitro troubleshooting: https://nitro.margelo.com/docs/troubleshooting
