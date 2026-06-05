# Stack Maturity Rubric — 999 / 1000 (June 2026)

**Purpose:** Calibrate “how good is this RN/Expo stack?” without inflated AI scores (e.g. blind **920/1000**). Use for **self-review**, **PR review**, and **agent audits**.

**999 means:** Documentation + decision rules are **ship-proof** and **anti-hallucination** for agents — not that every app in the repo scores 999.

---

## SCORE BANDS (honest)

| Score | Meaning |
|-------|---------|
| **600–750** | Tutorial stack: Expo Go only, Context for everything, FlatList at scale, no dev build, generic AI UI |
| **750–850** | Solid mid–senior: Expo Router, Zustand, some Reanimated, mixed lists, partial native polish |
| **850–920** | Senior **choices**: SDK 56, dev client, FlashList/Legend List where needed, MMKV, glass/tabs direction, anti-slop awareness |
| **920–960** | Senior **execution**: list discipline per screen, haptics/motion system, Query for server state, Sentry/fetch aligned, profiled on low-end Android |
| **960–999** | Lead **repeatable**: production incidents handled, OTA/binary discipline, monorepo Nitro pinned, Maestro smoke, measured perf fixes |
| **1000** | Reserved — sustained production excellence across **multiple** shipped apps, not one skill file |

**Split every review:**

```
Stack knowledge (libraries + docs)     → often 880–950 if using this skill
App execution vs that knowledge        → often 150–120 pts lower until audit passes
Native platform engineer (custom JSI)  → separate axis; NOT required for 960+ product apps
```

---

## SKILL VS APP (do not conflate)

| Layer | What “999 skill” means | What it does NOT mean |
|-------|------------------------|----------------------|
| **`rn-expo-stack` skill** | Agents get correct SDK 56 defaults, list rules, Sentry/fetch, anti-slop, audit checklists | Any repo is automatically 999 without an audit |
| **Your app** | Passes **Pre-ship audit** below on critical paths | You wrote custom C++ JSI |

**Rule for agents:** If the skill says FlashList and the repo uses `ScrollView` on a 500-row feed, **fix the app** — do not raise the score.

---

## PRE-SHIP AUDIT (execution → 960+)

Run before calling perf “done” or claiming senior stack.

### Lists (see `performance.md`)

- [ ] **&lt; ~30 static rows, no scroll perf issue** → `ScrollView` / `FlatList` OK — document why in file comment if non-obvious
- [ ] **Large uniform feed / grid** → `FlashList` v2 (no `estimatedItemSize`)
- [ ] **Chat, league boards, variable row height, scroll anchor** → `LegendList` v3 + `recycleItems` where appropriate
- [ ] **Never** `LayoutAnimation` on long lists
- [ ] List rows: `expo-image`, memoized row component, stable `keyExtractor`
- [ ] **Do not** recommend FlashList for a screen already correctly on Legend List

### State & network (`data-state.md`, `production-edge-cases.md`)

- [ ] **TanStack Query** for server/API state (when app has a backend)
- [ ] **Zustand** for UI / session UX state only — not a duplicate of Query cache
- [ ] **MMKV** for persisted non-secrets (dev build); **SecureStore** for tokens
- [ ] **No** `access_token` in MMKV / AsyncStorage
- [ ] Sentry **`traceFetch`** matches fetch mode (`expo/fetch` vs `EXPO_PUBLIC_USE_RN_FETCH=1`)

### Motion & feel (`premium-feel.md`, `platform-polish.md`)

- [ ] Custom CTAs: Reanimated press (~0.96 spring), not inert `Pressable`
- [ ] Haptics on **state change** (success/error/toggle/destructive) via one helper + settings flag
- [ ] **≤ 2** raised card surfaces per screen (`anti-ai-slop.md`)
- [ ] No vendor “Powered by …” marketing chrome in product UI

### Native & build (`foundation.md`, `build-deploy.md`)

- [ ] **`expo-dev-client`** for NativeTabs, MMKV, Nitro, full audio APIs
- [ ] **Nitro** single major across MMKV + Rive + other Nitro consumers
- [ ] Profiled on **low-end Android** (Pixel 4a class), not only iPhone simulator
- [ ] OTA (EAS Update) for JS-only fixes; new binary for native/config bumps

### Observability

- [ ] Sentry (or equivalent) in production builds
- [ ] One **documented** perf pass: Hermes sampling or RN Perf Monitor on worst screen (`performance.md`)

---

## WHAT IS **NOT** REQUIRED FOR 999 PRODUCT STACK

Reject advice that treats these as default gaps:

| Topic | Reality |
|-------|---------|
| **Custom C++ JSI / TurboModules** | Only for on-device ML, DSP, game engines, or APIs no Expo module covers — see `native-modules.md` |
| **Hermes heap snapshots daily** | Use when investigating leaks — not a badge |
| **FlashList on every screen** | Wrong tool for short static content; Legend List often better for variable height |
| **TanStack Query with zero backend** | Add when you have server state; local-only apps can skip until then |
| **Score 1000** | Marketing number — use checklist pass/fail instead |

---

## COMMON AI REVIEW MISTAKES (fix the reviewer)

1. **“Migrate league to FlashList”** — Wrong if board has variable height / avatars; **Legend List** may already be correct.
2. **“920 because package.json has FlashList”** — Installing ≠ using on hot paths.
3. **“Need JSI for 1000”** — Conflates **platform engineer** with **product engineer**.
4. **“Bleeding edge” without SDK lock** — Run `npx expo-doctor`; cite SDK 56 changelog, not blog posts.
5. **Praise for Zustand while tokens sit in MMKV** — Automatic fail regardless of animation quality.

---

## AGENT WORKFLOW (when user asks “rate my stack”)

1. Read `package.json` + `app.json` + 2–3 hot screens (home, main list, tabs layout).
2. Score **knowledge** and **execution** separately (table above).
3. Output **3 concrete gaps** from Pre-ship audit (file paths if possible).
4. Do **not** output a single inflated number without the split.
5. Point to reference files for fixes — do not invent `estimatedItemSize` on FlashList v2.

---

## FILTERING EXTERNAL “2026 RESEARCH” (Gemini, blogs, LinkedIn)

**Accept into skill only if:** SDK changelog, official docs, or reproducible repo evidence.

| Often repeated online | Skill treatment |
|----------------------|-----------------|
| Expo Router, Hermes v1, Nitro, Expo UI | Already core — verify version pins |
| FlashList for every list | **Reject** — use list audit (`performance.md`) |
| JSI/Wasm for “1000 score” | **Reject** for typical apps — `native-modules.md` |
| Agent Device / Argent | **Optional** — `agentic-workflows.md`; Maestro stays CI |
| RevenueCat churn / paywall stats | **Product** analytics — not stack architecture |
| Case study apps / boilerplate ads | Inspiration only |
| OTA “75% smaller” | Use Expo figure **~58% avg** unless citing specific changelog experiment |

When user pastes a long research doc: extract **checklist items**, discard narrative and numeric scores.

---

## REFERENCES

- `performance.md` — lists + Hermes profiling steps
- `agentic-workflows.md` — Maestro vs Agent Device vs Argent
- `file-system-sdk56.md` — async copy/move migration
- `native-modules.md` — when to write native code
- `production-edge-cases.md` — Sentry, Nitro, AI code rejection
- `anti-ai-slop.md` — UI execution bar
