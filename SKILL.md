---
name: goat-expo-skill
description: >
  Ultimate senior Expo + React Native production stack (June 2026).
  ALWAYS trigger for: "react native", "expo app", "mobile app", "RN stack", "expo stack",
  "expo libraries", "react native setup", "expo setup", "best RN libraries", "expo production",
  "react native 2026", "expo sdk 56", "expo router", "reanimated", "flashlist", "legendlist",
  "nativewind", "uniwind", "expo ui", "new architecture", "EAS", "dev build", "expo-dev-client",
  "mobile app architecture", "react native performance",   "liquid glass", "ios 26 edge shading", "glass edge shading", "specular highlight",
  "fresnel rim", "liquid glass button", "native tabs",
  "floating tab bar", "glass tab bar", "frosted tab bar", "pill tab bar", "custom tab bar",
  "boxShadow", "shadow deprecated", "pointerEvents deprecated", "reanimated web",
  "useAnimatedReaction", "metro cache", "HMR crash", "expo web console",
  "what libraries should I use for RN", "senior expo stack", "rate my react native stack",
  "expo stack score", "stack audit", "mastery rubric", "expo-audio", "expo-av migration",
  "ai slop", "anti ai slop", "generic ai design", "too many cards", "box in box UI",
  "app idea", "brainstorm app", "I want to build", "plan my app", "startup idea", "MVP plan".
  Use when building or architecting any React Native + Expo app (iOS + Android + web).
  Covers: SDK 56, dev builds, Expo Router (RN fork), Expo UI, Uniwind/NativeWind v5,
  Reanimated 4.4, FlashList v2, Legend List v3, MMKV v4/Nitro, TanStack Query, EAS, agents.
---

# RN + Expo — Senior Nerd Stack (June 2026)

**Scope:** Universal guidance for **any** React Native + Expo app. No product-specific defaults — read the target repo’s tokens, routes, and branding before generating UI.

**Baseline:** Expo SDK 56 · React Native 0.85 · React 19.2 · New Architecture (mandatory) · Hermes v1 · TypeScript 6

**Authority sources:** [Expo SDK 56 changelog](https://expo.dev/changelog/sdk-56), Expo Router migration, Software Mansion (Reanimated), Shopify (FlashList), LegendApp (Legend List), Unistyles team (Uniwind).

Read this file first, then open the reference file for the layer you are implementing.

---

## REFERENCE FILE MAP

| File | When to read it |
|---|---|
| `references/foundation.md` | New project, SDK 56 upgrade, New Arch, Hermes, React Compiler, dev builds |
| `references/navigation.md` | Expo Router, SDK 56 RN import fork, NativeTabs, Stack v5, deep links |
| `references/floating-glass-tab-bar.md` | **Optional** floating pill + frosted glass custom tab bar over mesh/hero UI |
| `references/ios-26-liquid-glass-edge-shading.md` | **iOS 26 default** specular/Fresnel rim, platform strategy, RN `LiquidGlassSurface` |
| `references/ui-design.md` | Expo UI, Uniwind/NativeWind, Liquid Glass, icons, theming |
| `references/animations.md` | Reanimated 4.4, worklets, Gesture Handler, Moti, Lottie, Skia |
| `references/performance.md` | FlashList v2, Legend List v3, MMKV v4, images, profiling |
| `references/data-state.md` | TanStack Query, Zustand, forms, SQLite, Supabase, `expo/fetch` |
| `references/platform-polish.md` | iOS 26, Android edge-to-edge, haptics, keyboard, widgets |
| `references/build-deploy.md` | EAS Build/Update/Submit, OTA diffing, Sentry, RevenueCat |
| `references/security.md` | SecureStore, pinning, screen capture, deep link validation |
| `references/speed-perception.md` | Perceived speed, prefetch, skeletons, loading psychology |
| `references/offline-resilience.md` | NetInfo, offline queue, TanStack Query offline, SQLite |
| `references/ux-psychology.md` | Optimistic UI, empty states, touch targets, permissions |
| `references/premium-feel.md` | Press/motion/haptics/keyboard/loading “feels premium” checklist |
| `references/anti-ai-slop.md` | **Universal** anti–AI-slop UI (any app/stack): tokens, cards, copy, agents |
| `references/error-boundaries.md` | ErrorBoundary, Sentry, crash recovery |
| `references/universal-links.md` | Universal Links, App Links, OAuth deep links |
| `references/i18n-rtl.md` | expo-localization, RTL, Arabic/Kurdish |
| `references/android-deep-dive.md` | Safe areas, AAB, signing, Play tracks |
| `references/store-aso.md` | App Store / Play submission, ASO, rejections |
| `references/brainstorm-idea.md` | **Seed idea → production brief** — expand vague app ideas before coding |
| `references/vibe-coder-prompts.md` | Copy-paste AI prompts + official Expo skills |
| `references/field-bug-playbook.md` | **After fixing non-obvious bugs** — symptom/fix/AI-trap entries agents must append |
| `references/web-rn-pitfalls.md` | **Web console errors** — `boxShadow`, `style.pointerEvents`, Reanimated web, stale HMR |
| `references/production-edge-cases.md` | Sentry + `expo/fetch`, Nitro/MMKV, haptics gate, `expo-audio` (945→999) |
| `references/mastery-rubric.md` | **Honest scoring**, skill-vs-app split, pre-ship audit, filter hype research |
| `references/native-modules.md` | When to write JSI/Turbo, Nitrogen, Wasm reality check |
| `references/agentic-workflows.md` | AGENTS.md, Maestro CI, optional Agent Device + Argent |
| `references/file-system-sdk56.md` | `expo-file-system` async copy/move, uploads, AbortSignal |

**Skill maturity target:** **999 / 1000** — agents must not confuse this with “your app scores 999.”

---

## THE COMPLETE STACK — ONE GLANCE

### Foundation
- **Expo SDK 56** (May 2026) — RN 0.85, React 19.2, Node ≥ 20.19.4, Xcode ≥ 26.4, iOS ≥ 16.4
- **New Architecture only** (Legacy removed SDK 55+)
- **Hermes v1** default · **React Compiler** on in templates (`experiments.reactCompiler`)
- **TypeScript 6** in templates
- Create: `npx create-expo-app@latest MyApp --template default@sdk-56`

### Development workflow (2026 default)
- **`expo-dev-client`** — production-parity daily driver (not store Expo Go)
- SDK 56 **Expo Go not on App/Play Store** — Android via CLI; iOS via TestFlight / `eas go`
- **Official Expo skills:** `npx skills add expo/skills` · project **AGENTS.md** in template
- **Agent tooling:** Expo Agent (beta), `serve-sim` for browser-hosted iOS sim + MCP

### Navigation
- **Expo Router** (version tied to SDK, e.g. `~56.x`) — **forked from app-level `@react-navigation/*`**
- Migrate: `npx expo-codemod sdk-56-expo-router-react-navigation-replace app`
- **`NativeTabs`** — `expo-router/unstable-native-tabs` (unstable API; dev build)
- Fallback tabs: `expo-router` JS `<Tabs>` or `expo-router/ui` on web
- Theme: `expo-router/react-navigation` (`ThemeProvider`, `DarkTheme`)

### UI & design (layer order)
1. **Expo UI** (`@expo/ui`) — stable SwiftUI / Jetpack Compose + universal components
2. **Tailwind on RN** — **Uniwind** (recommended, Tailwind v4, Metro-only) OR **NativeWind v5 preview**
3. **Liquid Glass** — `expo-glass-effect` + Expo UI `glassEffect` modifier (not third-party first)
4. **Icons** — `@expo/material-symbols` (Android MD3), `expo-symbols` (iOS), `lucide-react-native` (cross-platform)
5. **Images** — `expo-image` (blurhash, caching) — never RN `<Image>` for remote URLs

### Animations
- **Reanimated 4.4.x** + **`react-native-worklets`** (required; New Arch only)
- **Gesture Handler** — SDK-pinned (~2.31+); always pair with Reanimated
- **Moti** — mount/presence · **Lottie** — designer assets · **Skia** — custom 2D
- RN 0.85 **animation backend** complements Reanimated (do not drop Reanimated)

### Lists
- **FlashList v2** — default for large feeds (New Arch only, **no `estimatedItemSize`**)
- **Legend List v3** — chat, variable height, bidirectional infinite (`@legendapp/list/react-native`)
- **FlatList** — small/static lists only

### Data & state
- **TanStack Query** — server state · **Zustand** — UI state
- **Zustand + MMKV v4** — persisted non-secret state
- **`expo/fetch`** — default `globalThis.fetch` (SDK 56); opt out: `EXPO_PUBLIC_USE_RN_FETCH=1`
- **React Hook Form + Zod**
- **expo-sqlite** — relational / offline-first
- **expo-secure-store** — auth tokens (never MMKV for secrets)

### Storage & native perf
- **MMKV v4** + **`react-native-nitro-modules`** (dev build; align Nitro versions with MMKV)
- **Inline Expo modules** + **`expo-type-information`** (Swift → TS codegen)
- **EAS precompiled** Reanimated/screens + Expo XCFrameworks (faster iOS builds)

### Platform polish
- **NativeTabs** — Liquid Glass tabs (iOS 26), Material 3 (Android)
- Android **edge-to-edge** mandatory — `react-native-safe-area-context` / Uniwind `p-safe`
- **expo-haptics** — default (web haptics in SDK 56); **`react-native-haptics`** or **`react-native-pulsar`** optional in dev builds (presets/worklets)
- **expo-widgets** — iOS widgets stable SDK 56

### Ship
- **EAS Build / Update / Submit**
- **Hermes bytecode diffing** on by default (SDK 56 OTA ~58% smaller patches)
- **Sentry** — enable fetch tracing if using default `expo/fetch`
- **PostHog** — product analytics · **RevenueCat** — IAP
- **Maestro** — E2E · **RNTL** — components

---

## DECISION RULES

**Vague app idea / one-line concept?**
→ Read **`references/brainstorm-idea.md`** — output Product Brief first; no code until user confirms
→ Use **`vibe-coder-prompts.md`** Prompt: Seed Idea → Production Plan

**List component?**
→ Static / &lt; ~100 rows → `FlatList`
→ Feeds, grids, masonry, 500+ uniform rows → **FlashList v2**
→ Chat, AI threads, variable height, scroll anchoring → **Legend List v3** + `recycleItems`

**Styling?**
→ New app, Tailwind v4, max perf → **Uniwind**
→ Need NativeWind ecosystem / docs → **NativeWind v5 preview** + `react-native-css`
→ Heavy runtime object themes, anti-`className` → **Unistyles 3** (not with Uniwind globally)
→ System settings / pickers → **Expo UI** first
→ Do **not** start new projects on NativeWind v4 (Tailwind v3)

**Tabs?**
→ Production iOS/Android native chrome → **NativeTabs** (dev build, unstable API)
→ Floating frosted **pill** over mesh/hero backgrounds → **JS custom `tabBar`** (`references/floating-glass-tab-bar.md`)
→ Expo Go / quick web → JS `<Tabs>` or platform files (`_layout.web.tsx`)

**Animation?**
→ Gestures, scroll-linked, shared transitions → **Reanimated 4 + GH**
→ Mount/unmount presence → **Moti**
→ After Effects → **Lottie**
→ Particles / charts / custom draw → **Skia**

**Storage?**
→ Auth / payment tokens → **expo-secure-store**
→ Fast KV (settings, cache keys) → **MMKV v4** (dev build)
→ Relational offline data → **expo-sqlite**
→ One-time legacy / Expo Go only → AsyncStorage acceptable, not hot paths

**HTTP?**
→ Default: global **`expo/fetch`** (SDK 56)
→ Legacy transport only → `EXPO_PUBLIC_USE_RN_FETCH=1` (then **disable** Sentry `traceFetch` — see `references/production-edge-cases.md`)
→ Missing Sentry HTTP spans with `expo/fetch` → `traceFetch: true` (not both RN fetch + traceFetch)

---

## STARTUP CHECKLIST (new project)

```powershell
# 1. Create SDK 56 app
npx create-expo-app@latest MyApp --template default@sdk-56
cd MyApp

# 2. Dev build (required for MMKV, NativeTabs, full native stack)
npx expo install expo-dev-client

# 3. Core stack (use expo install for version alignment)
npx expo install react-native-reanimated react-native-worklets react-native-gesture-handler
npx expo install @shopify/flash-list @legendapp/list
npx expo install react-native-mmkv react-native-nitro-modules
npx expo install @tanstack/react-query zustand
npx expo install react-hook-form zod
npx expo install expo-image expo-secure-store
npx expo install @sentry/react-native

# 4. Styling — pick one
npm install uniwind tailwindcss@^4
# OR: npx expo install nativewind@preview react-native-css tailwindcss @tailwindcss/postcss postcss

# 5. Enable React Compiler (if not in template)
# app.json → "experiments": { "reactCompiler": true }

# 6. EAS
npx eas-cli init
npx eas build:configure
npx eas build --profile development --platform android

# 7. Agent skills + optional agentic QA
npx skills add expo/skills
# CI E2E: Maestro — optional agent verify: agentic-workflows.md

# 8. EAS env (recommended)
# EAS_USE_CACHE=1  — compiler ccache (~30% on hit)
# EAS_GRADLE_CACHE=1 — Android Gradle cache
```

---

## SDK 56 UPGRADE (from 55)

```powershell
npx expo install expo@^56.0.0 --fix
npx expo-doctor@latest
npx expo-codemod sdk-56-expo-router-react-navigation-replace app
# New dev build after native changes
npx expo prebuild --clean
```

Breaking highlights: `expo/fetch` default · async `expo-file-system` copy/move · `@expo/vector-icons` → `@react-native-vector-icons/*` · remove duplicate `@react-navigation/*` from app deps.

---

## GOLDEN RULES (senior nerd edition)

1. **Dev builds are the default environment** — Expo Go is for learning, not shipping or NativeTabs/MMKV/Nitro.
2. **Expo UI before random UI kits** — SwiftUI/Compose-backed primitives are stable SDK 56.
3. **Never animate on the JS thread** — Reanimated worklets / native drivers only.
4. **FlashList v2 needs no size estimates** — wrong skill advice was `estimatedItemSize` (v1 only).
5. **Secrets in SecureStore, speed in MMKV** — never swap them.
6. **Migrate off `@react-navigation/*` in app code** when using Expo Router 56+.
7. **React Compiler on** — delete manual `useMemo`/`useCallback`/`memo` unless profiling says keep.
8. **Profile on low-end Android** (Pixel 4a class) before calling perf done.
9. **OTA via EAS Update** for JS fixes; native/config changes need new binary.
10. **Check `expo-doctor` and changelog** every SDK bump — Expo moves fast in 2026.
11. **Pin `react-native-nitro-modules` to MMKV’s peer range** — Nitro is not forwards-compatible across majors (Rive, MMKV, etc. must agree).
12. **Sentry `traceFetch` matches your fetch implementation** — only with default `expo/fetch`, not with `EXPO_PUBLIC_USE_RN_FETCH=1`.
13. **No AI slop UI** — `references/anti-ai-slop.md` applies to **any** product UI; scan repo tokens/components first, max 2 raised surfaces per view, no vendor marketing chrome.
14. **Score knowledge and execution separately** — `references/mastery-rubric.md`; never rate 920+ from `package.json` alone.
15. **Lists per screen** — FlashList / Legend List / ScrollView by rule in `performance.md`; do not blanket-migrate leaderboards to FlashList.
16. **Custom native last** — `references/native-modules.md`; JSI is not the path from 920→999 for typical apps.
17. **Web RN deprecations** — `shadow*` → `boxShadow`, `pointerEvents` on **style**; Reanimated springs → `withTiming` on web (`references/web-rn-pitfalls.md`).
18. **Stale Metro first** — `ReferenceError` for removed symbols + “out of sync” → `expo start --clear` before chasing phantom bugs.
19. **Document non-obvious fixes** — after solving a non-common or AI-repeatable bug, append `references/field-bug-playbook.md` before ending the session (`agentic-workflows.md`).

---

## STACK MATURITY 999 (skill bar, not your app)

| Layer | File |
|-------|------|
| Honest rubric + pre-ship checklist | `references/mastery-rubric.md` |
| Sentry, Nitro, haptics, expo-audio | `references/production-edge-cases.md` |
| Hermes CPU + heap + list audit | `references/performance.md` |
| JSI only when measured | `references/native-modules.md` |

**What 999 adds beyond 945:** anti-inflated AI reviews, execution audit, expo-audio + file-system SDK 56, haptics gate, Hermes heap steps, agentic QA map (Maestro > agents), EAS ccache/workflows, Unistyles branch, research-filter rubric.

**The remaining 1 point:** reserved for field-proven updates (SDK bumps, incident postmortems) — edit the skill when production teaches something new.

**Recent field updates:** `field-bug-playbook.md` (Guidebook Android crash, glass-on-small-chip, tab animation); `web-rn-pitfalls.md` (crossShadow, springMotion, Metro HMR).

---

## PRODUCTION EDGE CASES (945 → 999)

Senior review gaps worth documenting:

- **`expo/fetch` + Sentry:** `traceFetch: true` when using SDK 56 default fetch; **turn off** if you opt into `EXPO_PUBLIC_USE_RN_FETCH=1` to avoid duplicate HTTP spans/breadcrumbs.
- **Nitro + MMKV:** install with `npx expo install react-native-mmkv react-native-nitro-modules`; lock Nitro major to MMKV **and** any other Nitro consumer (`@rive-app/react-native`, etc.); monorepos need one Nitro version at the workspace root.
- **Rate my stack / 920 feedback:** run `mastery-rubric.md` Pre-ship audit on the repo — reject FlashList-everywhere and JSI-as-1000 advice.

Full detail: `references/production-edge-cases.md`

---

## WHO TO TRUST (community map)

| Source | Handle / org | Trust for |
|---|---|---|
| Expo | @expo, @baconbrix (Evan Bacon) | SDK, Router, EAS, Expo UI, Agent |
| Software Mansion | @swmansion | Reanimated, Gesture Handler, worklets |
| Shopify | @Shopify | FlashList v2 |
| LegendApp | @LegendApp | Legend List v3 |
| Marc Rousavy | @mrousavy | MMKV v4, Nitro Modules |
| Unistyles / Uniwind | uniwind.dev | Uniwind vs NativeWind |

---

Read the matching `references/*.md` file before implementing that layer.
