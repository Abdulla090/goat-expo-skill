# AI Prompts — Expo SDK 56 Senior Stack (June 2026)

Copy a prompt, fill `[BRACKETS]`, paste into Cursor / Claude Code / Codex.

**Before generating code:** run `npx skills add expo/skills` and prefer official Expo skills for upgrades.

---

## PROMPT 1 — NEW PROJECT (FULL SKELETON)

```
Create a production Expo SDK 56 app.

App: [APP_NAME]
Android package: [com.company.app]
iOS bundle ID: [com.company.app]

FOUNDATION:
- Expo SDK 56, RN 0.85, React 19.2, TypeScript strict, New Architecture, Hermes v1
- experiments.reactCompiler: true in app.json
- Path alias @/* → ./*
- expo-dev-client (dev builds — not store Expo Go as primary workflow)

NAVIGATION:
- expo-router file-based routing
- NO @react-navigation/* imports in app code — use expo-router/react-navigation, expo-router/js-tabs
- app/(tabs)/_layout.tsx: NativeTabs (expo-router/unstable-native-tabs) with NativeTabs.Trigger.*
- ThemeProvider from expo-router/react-navigation wrapping tabs
- Tabs: [LIST TABS] — sf icons iOS, md icons Android

STYLING (pick one):
- Uniwind + Tailwind v4 (withUniwind metro) OR NativeWind v5 preview + react-native-css
- cn() via clsx + tailwind-merge
- expo-image for all remote images

UI NATIVE:
- @expo/ui for system controls where appropriate
- expo-glass-effect only with isGlassEffectAPIAvailable() guard on iOS 26

ANIMATIONS:
- react-native-reanimated 4.4 + react-native-worklets (expo install)
- react-native-gesture-handler — GestureHandlerRootView at root
- Moti for skeletons / presence

DATA:
- TanStack Query (staleTime 5m, gcTime 24h, networkMode offlineFirst optional)
- global fetch = expo/fetch (SDK 56 default)
- Zustand for UI state
- MMKV v4 + react-native-nitro-modules for non-secret persist (dev build)
- expo-secure-store for auth tokens ONLY
- React Hook Form + Zod

LISTS:
- @shopify/flash-list v2 for feeds (no estimatedItemSize)
- @legendapp/list/react-native for chat if needed

PLATFORM:
- SafeAreaProvider, StatusBar style auto
- expo-haptics on interactions

MONITORING:
- @sentry/react-native: traceFetch true ONLY if not using EXPO_PUBLIC_USE_RN_FETCH=1
- ErrorBoundary + Sentry.wrap on root

OUTPUT FILES:
- app/_layout.tsx (provider nesting order)
- app/(tabs)/_layout.tsx (NativeTabs)
- lib/storage.ts (MMKV), lib/secure.ts (SecureStore), lib/api.ts, lib/cn.ts
- stores/auth.store.ts, hooks/useNetworkStatus.ts
- components/ErrorBoundary.tsx, components/OfflineBanner.tsx
- global.css, metro.config.js, app.config.ts, eas.json, .env.example, AGENTS.md stub
```

---

## PROMPT 2 — DATA SCREEN (ALL STATES)

```
Build [SCREEN_NAME] for Expo SDK 56.

API: GET [ENDPOINT]
Shape: [DESCRIBE]

STACK:
- Uniwind or NativeWind v5 + TanStack Query useQuery
- placeholderData / keepPreviousData for stale-while-revalidate
- FlashList v2 (no estimatedItemSize) OR LegendList v3 if variable height
- Moti skeletons matching final layout
- Reanimated press scale 0.96 + expo-haptics Light on pressIn
- expo-image + blurhash
- useSafeAreaInsets

STATES: loading skeleton, error+retry, empty+CTA, success list, background refresh indicator

Prefetch: onPressIn start query for detail route

Dark mode: dark: classes
Pull-to-refresh: refetch + medium haptic

Output: screen, item, skeleton, types
```

---

## PROMPT 3 — FORM (RHF + ZOD)

```
Form [FORM_NAME] — fields: [LIST]
POST [ENDPOINT] → success: [NAVIGATE]

Stack: React Hook Form + Zod + TanStack useMutation + Uniwind/NativeWind v5
Validate on blur. Moti error entrance. Keyboard controller or KeyboardAvoidingView.
Submit states: idle / loading / success. Haptics success/error.
Shake on API error (Reanimated). secureTextEntry + textContentType on passwords.
Zod .trim() on strings. No logging secrets.
```

---

## PROMPT 4 — AUTH (SECURE)

```
Auth flow: login/register/logout/refresh
Tokens in expo-secure-store ONLY (WHEN_UNLOCKED)
User profile in Zustand; never persist access token in MMKV
expo-router groups: (auth) vs (tabs)
Splash hides after SecureStore auth check
401 → logout + replace /login
Axios or fetch interceptor
Social: [GOOGLE/APPLE] via expo-auth-session / expo-apple-authentication
Deep link return path after login
```

---

## PROMPT 5 — OFFLINE-FIRST

```
Screen [NAME] offline-capable for [DATA].

TanStack Query: offlineFirst, persist cache to MMKV optional
NetInfo: isConnected && isInternetReachable
OfflineBanner in root layout
classifyNetworkError: offline vs 5xx vs 4xx
Optimistic mutations + queue in Zustand/MMKV if writes offline
LegendList v3 if chat with maintainVisibleContentPosition
```

---

## PROMPT 6 — SCROLL HEADER

```
Collapsing header for [SCREEN].

Reanimated 4: useAnimatedScrollHandler, interpolate + Extrapolation.CLAMP
expo-blur or expo-glass-effect (guarded) on sticky header
iOS blur / Android elevation fallback
useSafeAreaInsets — hero under status bar, back button padded
No setState in scroll handler — shared values only
```

---

## PROMPT 7 — BOTTOM SHEET

```
Prefer @expo/ui BottomSheet (SDK 56) if fits requirements.
Else @gorhom/bottom-sheet with BottomSheetFlashList / BottomSheetTextInput
Reanimated + GH, haptic on snap, safe area bottom padding
enablePanDownToClose, keyboard aware
```

---

## PROMPT 8 — PUSH NOTIFICATIONS

```
expo-notifications SDK 56
Request permission in context (not launch)
POST token to [ENDPOINT]
Foreground handler + tap → expo-router navigate by data.type
Cold start notification open handling
Android POST_NOTIFICATIONS
```

---

## PROMPT 9 — SHARED ELEMENT TRANSITION

```
List → detail with Reanimated sharedTransitionTag including item.id
FlashList v2 or LegendList v3 list
Detail hero + safe area back button
Press scale before router.push
```

---

## PROMPT 10 — RTL / i18n

```
expo-localization + i18n-js
Locales: en + [ku/ar]
I18nManager.forceRTL before render; reload on switch if needed
paddingStart/End, textAlign auto, flip directional icons
Locale in MMKV via Zustand
Font: [FONT] via expo-font
```

---

## PROMPT 11 — ERRORS + SENTRY

```
Sentry.init:
- traceFetch: true only when expo/fetch is default (no EXPO_PUBLIC_USE_RN_FETCH=1)
- if EXPO_PUBLIC_USE_RN_FETCH=1 → traceFetch: false (avoid duplicate HTTP spans)
Sentry.wrap root; ErrorBoundary with Updates.reloadAsync()
captureError utility; 401 logout in API layer
Screen-level boundaries for risky widgets
__DEV__ show error detail
```

---

## PROMPT 12 — STORE SUBMISSION

```
App [NAME] — permissions: [LIST]
Privacy manifest NSPrivacy* for MMKV/Sentry APIs
app.config.ts permission strings (why not what)
Play Data Safety + POST_NOTIFICATIONS
ASO title/subtitle/keywords/screenshot brief
Pre-submit: dev build tested, no secrets in EXPO_PUBLIC_, OTA channel production
SDK 56: note Expo Go not on stores — reviewers use TestFlight/internal track
```

---

## QUICK INSTALL (POWERSHELL)

```powershell
npx create-expo-app@latest MyApp --template default@sdk-56
cd MyApp

npx expo install expo-dev-client
npx expo install react-native-reanimated react-native-worklets react-native-gesture-handler
npx expo install @shopify/flash-list @legendapp/list
npx expo install react-native-mmkv react-native-nitro-modules
npx expo install expo-image expo-secure-store expo-sqlite
npx expo install @sentry/react-native @react-native-community/netinfo
npx expo install react-native-safe-area-context expo-status-bar expo-haptics

npm install @tanstack/react-query zustand react-hook-form zod @hookform/resolvers
npm install uniwind tailwindcss@^4
# OR: npx expo install nativewind@preview react-native-css

npx skills add expo/skills
npx eas-cli init
```

---

## SDK 56 MIGRATION PROMPT

```
Upgrade project to Expo SDK 56:
1. npx expo install expo@^56.0.0 --fix
2. npx expo-doctor@latest
3. npx expo-codemod sdk-56-expo-router-react-navigation-replace app
4. Remove @react-navigation/* from app imports and package.json if unused
5. Audit expo/fetch vs Sentry (traceFetch only without USE_RN_FETCH=1)
6. Pin react-native-nitro-modules to MMKV + Rive peer range
6. Migrate @expo/vector-icons → @react-native-vector-icons/*
7. Rebuild expo-dev-client after native changes
Follow https://expo.dev/changelog/sdk-56
```

---

## PROMPT — ANTI–AI SLOP SCREEN PASS (ANY PROJECT)

```
Redesign [SCREEN_PATH] in [PROJECT_NAME]. Mandatory: references/anti-ai-slop.md (+ premium-feel.md if RN/Expo).

First: scan repo for theme tokens, shared components (e.g. components/ui, theme/, design-system/), and i18n — list what you will reuse.

Reject:
- Generic dark-purple "AI" dashboards, gradient orbs, Sparkles/wand/bot icons
- "Powered by [LLM vendor]", fake hardcoded metrics, monolingual copy in localized apps
- Card ⊃ card layouts; identical card carousels for few options
- Duplicate CTAs for the same user goal

Require:
- Project tokens + existing components only — no parallel design system
- ≤ 2 raised surfaces per view; lists = rows + dividers
- One primary CTA per goal; real or skeleton data
- Strings via project's l10n pattern ([LOCALES])

Deliver: minimal diff on [SCREEN_PATH] (+ locale files if needed).
```

---

## PROMPT — FLOATING LIQUID GLASS TAB BAR (OPTIONAL)

```
Add a floating frosted pill tab bar to [APP_NAME]. Follow rn-expo-stack references/floating-glass-tab-bar.md (universal — use THIS repo's theme tokens, not example hex values).

Tabs (static, left→right): [TAB_1] · [TAB_2] · [TAB_3 optional] · [TAB_4 optional]

Visual strategies (structure, not fixed colors):
- Floating capsule with horizontal margin + soft shadow
- Frosted translucency — tab screens use mesh/gradient hero so glass refracts background
- Hairline rim on pill; top sheen gradient layer
- Active tab: sliding oval pill behind icon+label; inactive: muted icon+label only
- Press: Pressable + Animated.View scale ~0.94 (no undefined AnimatedPressable)

Platform:
- Expo Router Tabs with custom tabBar + transparent tabBarStyle
- iOS dev build: expo-glass-effect GlassView when isGlassEffectAPIAvailable(), else BlurView + frost layers
- Android: layered frost Views + LinearGradient sheen — NO BlurView on tab bar (dark mode black bar)
- Web: frost + optional backdrop-filter; _layout.web.tsx if split from NativeTabs

Imports: named exports only (e.g. import { NavigationBar } from 'expo-navigation-bar' — never import * as X for JSX).

Deliver: FloatingGlassTabBar component, tab layout wiring, mesh background on tab root screens if missing.
```

---

## PROMPT — STACK AUDIT (HONEST 999 RUBRIC)

```
Audit [PROJECT_NAME] React Native / Expo stack. Use rn-expo-stack references/mastery-rubric.md — do NOT give a single inflated score.

1. Read package.json, app.json, and hot paths: [HOME_SCREEN], [MAIN_LIST_SCREEN], app/(tabs)/_layout*.
2. Score separately:
   - Stack knowledge (library choices) /1000
   - App execution vs skill bar /1000
   - Native/JSI depth (optional axis) /1000
3. Run Pre-ship checklist from mastery-rubric.md — cite file paths for each fail/pass.
4. Lists: credit Legend List on variable-height boards; do not recommend FlashList blindly; allow ScrollView for short static screens.
5. Reject "learn JSI for 1000" unless a measured native bottleneck exists (native-modules.md).
6. Output exactly 3 prioritized fixes with reference file links (performance.md, production-edge-cases.md, anti-ai-slop.md).

No praise without evidence. No 920+ unless execution checklist mostly passes.
```
