# Native Modules — When to Write JSI / Turbo (June 2026)

**Default for 999 product stack:** Use **Expo modules**, **config plugins**, and published Nitro/MMKV libraries. **Do not** start with custom C++.

---

## DECISION TREE

```
Need native capability?
├─ Exists in Expo SDK (expo-camera, expo-audio, expo-file-system, …)
│  └─ Use expo install + dev build if needed
├─ Exists on npm with New Arch support (MMKV, Reanimated, FlashList, …)
│  └─ npx expo install; align Nitro peers (production-edge-cases.md)
├─ Needs thin platform API (Settings, Widget, Share extension)
│  └─ Expo config plugin + local module OR expo-modules-core template
└─ Still blocked: latency, SIMD, custom DSP, on-device model inference
   └─ Consider Turbo Module / JSI — measure first (performance.md)
```

---

## WHEN CUSTOM JSI / TURBO IS JUSTIFIED

| Use case | Why bridge/modules fail |
|----------|-------------------------|
| Real-time audio DSP (noise gate, custom codec) | JS thread + bridge too slow |
| On-device ML inference (Core ML / NNAPI) with tight buffer contracts | Need zero-copy buffers |
| Game loop / 60fps sensor fusion | Synchronous JSI reads |
| Proprietary SDK (payment hardware, BLE protocol) | Vendor ships native only |

**Not justified:** prettier tabs, “faster” `JSON.parse`, replacing FlashList, avoiding `expo-dev-client`, or copying a blog post “hello JSI” module.

---

## PREFER BEFORE CUSTOM NATIVE

1. **`expo/fetch`** + background tasks for network-heavy work  
2. **Reanimated worklets** for gesture/animation math on UI thread  
3. **Skia** for custom 2D (charts, signatures) without C++  
4. **Inline Expo modules** (`expo-type-information`) for Swift/Kotlin APIs with TS codegen  
5. **EAS Build** custom native code in `ios/` / `android/` after `prebuild` — still not JSI unless measured  

---

## NEW ARCH NOTES

- **TurboModules** lazy-load; **Fabric** for layout — your module must support New Architecture (Legacy dead SDK 55+).
- **Nitro Modules** (MMKV, some third parties) — version-pin `react-native-nitro-modules`; never mix majors.
- Expo’s **JSI path on iOS** (SDK 56) reduces ObjC++ hops for **Expo-authored** modules — you still pay maintenance for **your** C++.

---

## NITRO MODULE AUTHORING (library / heavy perf)

Published Nitro libs (MMKV, VisionCamera, quick-crypto) use **Nitrogen** codegen from TypeScript:

```powershell
npx nitrogen@latest init
```

Generates C++ JSI bindings — for **library authors**, not typical app screens.

**App teams:** consume via `npx expo install` + Nitro peer pin (`production-edge-cases.md`). Do not hand-roll Nitro unless building a reusable native package.

---

## WASM / HERMES (do not hype)

- **Hermes v1** is the default RN engine — use it; opt out only with measured reason (`foundation.md`).
- **Wasm in RN** is not “import .wasm in Metro like the browser.” Paths today: experimental Static Hermes AOT, Hermes sandbox internals, or community bridges (e.g. wasm runtimes via Turbo Module).
- **Default product path:** Expo module → Nitro consumer lib → Skia/Reanimated — not custom Wasm unless profiling proves JS/C++ bottleneck.

---

## COST CHECKLIST (before writing C++)

- [ ] Prototype in JS; profile with Hermes sampling (`performance.md`)
- [ ] Confirmed no Expo module / community package
- [ ] Plan for **iOS + Android + CI** (EAS) maintenance
- [ ] TypeScript types exported for app team
- [ ] Fallback when module missing (Expo Go / web) documented

---

## AGENT RULE

If user or reviewer says “reach 1000 by learning JSI” for a standard CRUD / learning app → **reject**. Point to `mastery-rubric.md` Pre-ship audit instead.
