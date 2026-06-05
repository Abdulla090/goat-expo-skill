# Foundation — Expo SDK 56 + New Architecture (June 2026)

## THE BASELINE

Every new production app in June 2026:

```
Expo SDK 56          (released 2026-05-21)
React Native 0.85.x
React 19.2.x
Hermes v1            (default)
New Architecture     (mandatory — no Legacy Arch in SDK 55+)
TypeScript 6.x       (template default)
Node.js ≥ 20.19.4
Xcode ≥ 26.4         (iOS builds)
iOS deployment ≥ 16.4
```

Legacy Architecture is dead (frozen SDK 55+, removed from Expo SDK 56 templates). If a library does not support New Arch + Fabric, do not adopt it.

---

## WHAT NEW ARCHITECTURE GIVES YOU

| Legacy | New Architecture |
|---|---|
| Async JSON bridge | JSI — synchronous native access |
| JS-thread animations | UI-thread worklets (Reanimated) |
| Eager module load | TurboModules — lazy init |
| Old renderer | Fabric — concurrent layout |

**SDK 56 Android wins (zero app code):**
- Kotlin compiler plugin for Expo Modules → ~40% faster cold start, ~33% faster first render
- Optional `android.usePrecompiledHeaders` in `expo-build-properties` → up to ~2.8× faster CMake debug

**SDK 56 iOS wins:**
- Prebuilt XCFrameworks for heavy Expo modules (~16% faster clean builds)
- EAS precompiles Reanimated + react-native-screens (~20% more on EAS)
- New Swift/C++ JSI path for iOS modules (fewer ObjC++ hops)

---

## METRO & NODE (SDK 56)

| Change | Detail |
|--------|--------|
| **Node.js** | **≥ 20.19.4** (EOL versions dropped) |
| **File watcher** | Default **Node built-in watcher**; Watchman optional via Metro `resolver.useWatchman` |
| **On-demand FS** | `experiment.onDemandFilesystem` default — drops `watchFolders` as mandatory; better pnpm/bun monorepos |
| **Bundler warmup** | Faster `expo start` (monorepos reported up to ~30% improvement) |
| **HTTPS dev server** | Metro `server.tls` for APIs requiring secure origin locally |

```js
// metro.config.js — local HTTPS (when API requires secure origin)
module.exports = {
  server: {
    tls: {
      cert: './path/to/cert.pem',
      key: './path/to/key.pem',
    },
  },
};
```

---

## HERMES V1

Default in SDK 56. Faster startup, lower memory, fewer Metro transforms.

Opt out only with a measured reason:

```json
{
  "expo": {
    "plugins": [
      ["expo-build-properties", {
        "ios": { "useHermesV1": false },
        "android": { "useHermesV1": false }
      }]
    ]
  }
}
```

---

## REACT COMPILER (enable day one)

Expo SDK 54+ templates enable React Compiler. Turn on in existing apps:

```json
{
  "expo": {
    "experiments": {
      "reactCompiler": true
    }
  }
}
```

Run `npx expo lint` — `eslint-config-expo` includes compiler rules (SDK 55+).

**After enabling:** remove most `useMemo`, `useCallback`, `React.memo` unless profiling proves they help.

Opt out per file: `"use no memo"` directive.

Docs: https://docs.expo.dev/guides/react-compiler/

---

## REACT 19.2 — USE ON MOBILE

| API | Use for |
|---|---|
| `useOptimistic()` | Likes, toggles, instant feedback before server |
| `useActionState()` | Form submit loading/error with less boilerplate |
| `use()` | Promise/context in render (with Suspense boundaries) |
| React Compiler | Automatic memoization |

Pair optimistic UI with TanStack Query `onMutate` rollback for server-backed data.

---

## PROJECT CREATION

```powershell
npx create-expo-app@latest MyApp --template default@sdk-56
cd MyApp
npx expo install expo-dev-client
```

**Do not** rely on `create-expo-app@latest` without `--template default@sdk-56` during SDK transitions.

Verify `package.json`: `"expo": "^56.x"`, `"react-native": "0.85.x"`.

---

## PRODUCTION FOLDER STRUCTURE

```
MyApp/
├── app/                      # Expo Router routes
│   ├── _layout.tsx           # Providers (see navigation.md)
│   ├── (tabs)/
│   │   ├── _layout.tsx       # NativeTabs or Tabs
│   │   └── index.tsx
│   └── +not-found.tsx
├── components/
│   ├── ui/
│   └── features/
├── hooks/
├── lib/                      # api, storage, cn, supabase
├── stores/                   # Zustand
├── services/
├── assets/
├── global.css                # Tailwind v4 / Uniwind or NativeWind v5
├── app.config.ts             # Typed config plugins (SDK 56)
├── AGENTS.md                 # Agent instructions (template)
└── eas.json
```

---

## TYPESCRIPT

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "paths": { "@/*": ["./*"] }
  }
}
```

SDK 56 improves TS 6 support and monorepo `paths` resolution.

---

## LIBRARY COMPATIBILITY CHECK

Before adding any dependency:

1. https://reactnative.directory — filter New Architecture
2. GitHub: issues/PRs mentioning Fabric / TurboModules in last 12 months
3. `npx expo-doctor@latest` after install

**Avoid for new work:**
- NativeBase (unmaintained vs Expo UI / gluestack v3)
- Direct `@react-navigation/*` in app code with Expo Router 56+
- `react-native-navigation` (Wix) — use Expo Router
- FlashList v1 / Old Arch-only list libs

---

## EXPO GO VS DEV BUILD (2026 reality)

| Capability | Store Expo Go SDK 56 | Dev build (`expo-dev-client`) |
|---|---|---|
| On App Store / Play | **No** (SDK 56) | Your binary |
| Expo UI | Yes (in SDK 56 Go builds) | Yes |
| NativeTabs | No | Yes |
| MMKV v4 / Nitro | No | Yes |
| Custom native modules / plugins | No | Yes |
| Matches production `app.json` native config | No | Yes |
| Liquid Glass full fidelity | Limited | Yes |

**Senior rule:** Ship and develop with **dev builds**. Use Expo Go only for tutorials or Android CLI-installed SDK 56 Go.

iOS SDK 56 Go: TestFlight external beta or `eas go` → your TestFlight team.

---

## INLINE EXPO MODULES (SDK 56)

Write Swift/Kotlin next to TS — autolinked at prebuild:

```
modules/
  my-feature/
    MyModule.swift
    MyModule.ts          # stable interface
    MyModule.generated.ts
```

CLI: `expo-type-information` — `module-interface`, `inline-modules-interface` (watch mode).

Revamped `create-expo-module` — Windows support, `addPlatformSupport`, non-interactive mode.

---

## AI / AGENT SETUP

```powershell
npx skills add expo/skills
```

Template includes **AGENTS.md**, **CLAUDE.md**. Use Expo upgrade skill for SDK bumps.

**Evan Bacon / Expo direction (2026):** Expo Agent (prompt → native app), **serve-sim** (iOS sim in browser for MCP agents). Mobile verification should mirror how agents test web (screenshots, DOM-like inspection).

**Extended:** Maestro (CI E2E) + optional **Agent Device** (exploratory QA) + **Argent** (sim profiling) — see `references/agentic-workflows.md`. Do not skip Maestro for agent-only testing.

---

## BROWNFIELD (embed Expo in native app)

SDK 56 `expo-brownfield`:
- `multipleFrameworks: true` — multiple isolated Expo apps in one host
- Host `turboModuleClasses` registration
- iOS prebuilds by default (faster)

---

## AUDIO (SDK 56)

- **`expo-audio`** replaces **`expo-av`** for playback + recording in new work.
- Install: `npx expo install expo-audio` · dev build for full native behavior.
- Always **release/stop** players on screen exit (voice tutors, games) — prevents Hermes heap growth.
- Migration table + patterns: `references/production-edge-cases.md` §6.

---

## UPGRADE PATH (54 → 56)

1. Enable New Arch on SDK 54, stabilize
2. Upgrade to SDK 55, then 56
3. `npx expo install expo@^56.0.0 --fix`
4. Run Router codemod + `expo-doctor`
5. **Explicit devDependency:** `babel-preset-expo` — silent Metro/babel failures if missing after upgrade
6. Delete `ios`/`android` if using CNG; rebuild dev client
7. Fix `expo/fetch`, **file-system async copy/move** (`references/file-system-sdk56.md`), vector icons migration
8. `npx expo start --clear` after babel/metro config changes

```powershell
npx expo install --dev babel-preset-expo
```

Changelog: https://expo.dev/changelog/sdk-56

---

## WHEN TO LEAVE MANAGED EXPO

Almost never. Valid exceptions:
- Host app brownfield (`expo-brownfield`)
- Native module with no config plugin and no inline module path
- Rare manual native surgery (prefer inline modules + config plugins first)

99% of apps: **managed workflow + dev builds + EAS**.
