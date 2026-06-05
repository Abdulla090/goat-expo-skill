# Field Bug Playbook — Document Fixes in the Skill (June 2026)

**Agent rule:** After you fix a bug that is **non-obvious**, **platform-specific**, or a **repeatable AI mistake**, update this file (and the matching reference) before ending the session.

**Do not document:** one-off typos, missing imports in a single file with no pattern, product copy, or business-logic bugs unique to one screen’s data.

**Do document:** bugs that would bite the **next agent** or **next SDK bump** — especially traps LLMs reintroduce.

---

## WHEN TO UPDATE THE SKILL

| Signal | Action |
|---|---|
| Fix took >1 attempt because of a **non-documented** RN/Expo/Reanimated quirk | Add row to playbook + link from `production-edge-cases.md` or `web-rn-pitfalls.md` |
| Bug is a **classic AI regression** (wrong hook, glass on tiny control, stale HMR, wrong tab animation) | Add **anti-pattern** + **canonical fix** |
| You added a **reusable util** (`crossShadow`, `springMotion`) | Document in playbook + `agentic-workflows.md` workflow |
| Crash only on **one platform** (Android press + navigate, web bundle stale) | Platform column in table below |

### Workflow (mandatory after non-trivial fix)

1. **Fix in code** — prefer shared util when pattern repeats twice.
2. **Add one playbook entry** — symptom → root cause → fix → test step.
3. **Update the right reference** — web → `web-rn-pitfalls.md`; tabs/glass → `floating-glass-tab-bar.md`; navigation → `navigation.md`.
4. **Sync** project `.cursor/skills/goat-expo-skill/` → global `~/.agents/skills/rn-expo-stack/` if the user uses both.
5. **Mention in PR/commit** — one line: `docs(skill): field-bug-playbook — <short title>`.

---

## PLAYBOOK ENTRIES (field-proven)

### Web — stale Metro shows removed API

| | |
|---|---|
| **Symptom** | `ReferenceError: useAnimatedReaction is not defined` but `grep src/` is clean |
| **Root cause** | Stale HMR bundle after refactor; “Expo CLI and web client are out of sync” |
| **Fix** | `npx expo start --web --clear` + hard refresh — not a code change |
| **AI trap** | Re-adding `useAnimatedReaction` or random imports instead of clearing cache |
| **Reference** | `web-rn-pitfalls.md` §3 |

### Web — RN 0.83+ deprecations

| | |
|---|---|
| **Symptom** | `shadow* style props are deprecated`; `props.pointerEvents is deprecated` |
| **Root cause** | Shared `StyleSheet` uses legacy shadow props / `pointerEvents` prop on web |
| **Fix** | `crossShadow()` for shadows; `style={{ pointerEvents: 'none' }}` on decorative layers |
| **Reference** | `web-rn-pitfalls.md` §1 |

### Web — Reanimated spring easing

| | |
|---|---|
| **Symptom** | `[Reanimated] Selected easing is not currently supported on web` |
| **Root cause** | `withSpring(customConfig)` on web |
| **Fix** | `springMotion()` — `withTiming` on web, `withSpring` on native |
| **Reference** | `web-rn-pitfalls.md` §2, `animations.md` |

### Tab bar — indicator animation pattern

| | |
|---|---|
| **Symptom** | Indicator doesn’t move on first tab tap; or crash from missing hook import |
| **Root cause** | Syncing only in `useEffect` without optimistic press; or `useAnimatedReaction` without import |
| **Fix** | `optimisticPress` ref + `moveIndicator` on press + `useEffect` when `state.index` catches up |
| **AI trap** | Defaulting to `useAnimatedReaction` for tab index |
| **Reference** | `floating-glass-tab-bar.md`, `web-rn-pitfalls.md` §2 |

### Android — Guidebook chip crash on press

| | |
|---|---|
| **Symptom** | App crashes tapping small “Guidebook” chip on blue unit header (Android) |
| **Root cause** | `SoftPressableButton` with translucent white → **liquid glass** on tiny control; Reanimated **CSS press transition** + immediate `router.push`; global tab `animation: "shift"` on hidden `guidebook` route |
| **Fix** | Solid semi-opaque `Pressable` chip (min width/height, centered row, `numberOfLines={1}`); `guidebook` screen `animation: "fade"`; avoid glass on chips &lt; ~44pt on saturated backgrounds |
| **AI trap** | Applying liquid glass to every neutral button regardless of size/context |
| **Test** | Path tab → tap Guidebook → screen opens, tab bar hidden, back works |
| **Reference** | `ui-design.md` (small controls), `navigation.md` (hidden route animation) |

### Android — liquid glass on small controls (general)

| | |
|---|---|
| **Symptom** | Clipped label, misaligned icon, unreadable chip on colored header |
| **Root cause** | `LiquidGlassSurface` + `overflow: hidden`; `contentStyle` flex not applied inside glass shell |
| **Fix** | Chips under ~48×120pt: **solid** `rgba(255,255,255,0.2–0.35)` + border; reserve glass for bars, cards, large buttons |
| **AI trap** | `prefersLiquidGlass()` true for all non-blue buttons |

### Navigation — hidden tab routes

| | |
|---|---|
| **Symptom** | Crash or jank pushing to `href: null` / hidden tab (`guidebook`, `lesson`) |
| **Root cause** | Android `shift` animation on routes not in tab bar |
| **Fix** | Per-screen `animation: 'fade'` (or `none`) on hidden stack tabs |
| **Reference** | `navigation.md` |

---

## TEMPLATE (copy for new entries)

```markdown
### [Platform] — Short title

| | |
|---|---|
| **Symptom** | What the user/console shows |
| **Root cause** | Why it happens (1–2 sentences) |
| **Fix** | Concrete code/strategy |
| **AI trap** | What LLMs typically do wrong |
| **Test** | How to verify |
| **Reference** | `other-file.md` §section |
```

---

## RELATED FILES

- `references/agentic-workflows.md` — agent workflow summary
- `references/web-rn-pitfalls.md` — web-specific subset
- `references/production-edge-cases.md` — production index
- `references/error-boundaries.md` — stale bundle vs real crash triage
