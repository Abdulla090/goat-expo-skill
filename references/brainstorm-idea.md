# Brainstorm Idea — Seed → Production Plan (June 2026)

**Purpose:** When the user gives a **small, vague, or one-line app idea**, expand it into a **ship-ready product brief + technical plan** before writing code.

**Trigger phrases:** `app idea`, `I want to build`, `make an app like`, `brainstorm`, `what should I build`, `turn this into an app`, `plan my app`, `MVP for`, `startup idea`, `mobile app for`.

**Agent rule:** Read this file **first**. Output the **Product Brief** (template below). Do **not** scaffold code until the user confirms or says “build it” / “start implementing”.

---

## WHEN TO USE

| User input | Action |
|---|---|
| One sentence (“Duolingo for X”, “habit tracker with AI”) | Full expansion — all sections |
| Medium brief (audience + 2–3 features named) | Skip obvious clarifiers; fill gaps with labeled assumptions |
| Detailed PRD already written | Light pass — stack map + route tree + phase cut only |
| “Just build it” after a prior brief | Proceed to `foundation.md` + `vibe-coder-prompts.md` Prompt 1 |

---

## AGENT WORKFLOW (mandatory order)

```
1. Parse seed        → extract noun (what), verb (job), who (user), why (pain)
2. Sharpen wedge     → one sentence positioning; reject “app for everyone”
3. Define MVP        → 3–5 screens users must have on day 1; cut the rest
4. Map information   → entities, relationships, offline vs server
5. Design navigation → Expo Router tree + tab/stack/modal rules
6. Pick stack         → goat-expo-skill decision rules (lists, auth, styling)
7. Phase roadmap     → MVP → v1 → v1.1 with explicit “not now” list
8. Risk & polish     → permissions, store rejection, anti-slop UI direction
9. Deliver brief     → use template below; max 2 open questions at end
```

**Do not:** jump to `create-expo-app`, invent 15 tabs, add AI because the idea is vague, or copy a famous app’s entire feature set.

**Do:** name **assumptions** explicitly (`Assumption: B2C, free + ads later`), tie every screen to a **user job**, and point to reference files for the next build step.

---

## STEP 1 — PARSE THE SEED

Extract from even a 3-word idea:

| Field | Question | Example (“meal prep app”) |
|---|---|---|
| **Job** | What painful task disappears? | Plan weekly meals without decision fatigue |
| **User** | Who pays attention daily? | Busy parents, 25–40, cooks 4+ nights/week |
| **Moment** | When do they open the app? | Sunday planning + grocery run |
| **Outcome** | Success in one line? | Grocery list + 5 dinners decided in &lt;10 min |
| **Wedge** | Why not Notes / Excel / ChatGPT? | Pantry-aware plans + one-tap list export |

If the seed is a clone (“like Duolingo but for music”):
- **Keep:** core loop structure (lesson → practice → streak → progress)
- **Replace:** content domain, metaphor, and **one** differentiator (e.g. ear training vs flashcards)
- **Cut:** clone features that don’t serve the wedge (social feed, leaderboards unless core)

---

## STEP 2 — POSITIONING (anti-generic)

Write **one positioning sentence:**

> **[App name]** helps **[specific user]** **[achieve outcome]** by **[unique mechanism]** — unlike **[alternative]**, we **[wedge]**.

**Reject these defaults unless the user insists:**
- “AI-powered” as the only differentiator
- Dashboard with 4 stat cards on home
- Chat as the entire product without a structured loop
- Social feed for non-social jobs

Cross-check: `references/anti-ai-slop.md` — pick a **visual metaphor** tied to the domain (path, timeline, shelf, map — not generic glass dashboard).

---

## STEP 3 — MVP SCOPE (ruthless)

### The MVP test

Every MVP feature must pass **all three**:
1. **Daily or weekly** use for target user
2. **Demonstrable in 30 seconds** (demo / store screenshot)
3. **Buildable in 1–2 weeks** with Expo SDK 56 + this skill stack

### Scope table (fill for every idea)

| Tier | Include | Exclude (v1+) |
|---|---|---|
| **MVP (ship)** | 3–5 core screens, auth if needed, one list/feed, one create flow, settings | Widgets, iPad layout, teams, admin panel |
| **v1** | Onboarding, notifications, paywall, search, profile depth | ML on-device, custom JSI |
| **Later** | Web parity, widgets, referrals, B2B API | “Nice to have” from brainstorm dump |

**Cap navigation:** max **4 tabs** (Miller’s law — `ux-psychology.md`). Hidden stack screens for detail/create/modals.

---

## STEP 4 — INFORMATION ARCHITECTURE

Sketch entities before routes:

```text
User ──< Item
User ──< Session / Log
Item ──< Tag (optional)
```

For each entity list:
- **Fields** (id, createdAt, title, status, …)
- **Source** — local only / Supabase / REST / on-device
- **Persist** — MMKV (prefs), SecureStore (tokens), SQLite (relational offline), TanStack Query (server)

| Pattern | When |
|---|---|
| Local-only MVP | Zustand + MMKV; no backend until v1 |
| Auth + cloud sync | Supabase or custom API + TanStack Query + SecureStore |
| Offline-first | expo-sqlite WAL + Query persistence |
| AI features | Structured prompts + server proxy; never raw API keys in app |

Read `references/data-state.md` before finalizing.

---

## STEP 5 — EXPO ROUTER MAP

Produce a **route tree** for every brief:

```text
app/
├── _layout.tsx              # providers: Query, SafeArea, GestureHandler, Sentry
├── (auth)/
│   ├── login.tsx
│   └── register.tsx
├── (tabs)/
│   ├── _layout.tsx          # NativeTabs OR custom floating glass (see decision)
│   ├── index.tsx            # Home / primary loop
│   ├── [domain].tsx         # e.g. library, path, inbox
│   └── profile.tsx
├── [entity]/[id].tsx        # detail stack
├── create.tsx               # modal or stack
└── settings/
    └── index.tsx
```

**Tab decision** (`SKILL.md` + `navigation.md`):
- System chrome, standard app → **NativeTabs** (dev build)
- Hero/mesh background, brand-forward → **floating glass tab bar** (`floating-glass-tab-bar.md`)
- Expo Go demo only → JS `<Tabs>`

**List decision per screen** (`performance.md`):

| Screen type | Component |
|---|---|
| Settings, &lt;30 static rows | ScrollView / FlatList |
| Feed, grid, catalog | FlashList v2 |
| Chat, threads, variable rows | Legend List v3 |

---

## STEP 6 — STACK AUTO-PICK (from seed)

Default **unless the brief requires otherwise**:

| Layer | Choice |
|---|---|
| Foundation | Expo SDK 56, dev-client, New Arch, React Compiler |
| Navigation | Expo Router, no app-level `@react-navigation/*` |
| Styling | Uniwind + Tailwind v4 (new app) |
| UI primitives | Expo UI + expo-image |
| Motion | Reanimated 4 + Gesture Handler; Moti for skeletons |
| Server state | TanStack Query when backend exists |
| UI state | Zustand |
| Persist | MMKV (non-secrets), SecureStore (tokens) |
| Ship | EAS Build/Update, Sentry (`traceFetch` if `expo/fetch`) |
| E2E | Maestro smoke flows (`agentic-workflows.md`) |

**Monetization (pick one primary for v1, note in brief):**

| Model | Stack hint |
|---|---|
| Free + subscription | RevenueCat (`build-deploy.md`) |
| One-time unlock | RevenueCat or IAP module |
| B2B seats | Clerk orgs or Supabase RLS + invite |
| Ads (last resort) | Note ad SDK only if user asks — not default |

---

## STEP 7 — PHASE ROADMAP

Always output three phases with **week estimates** (solo dev + agents):

```text
Phase 0 — Foundation (2–3 days)
  create-expo-app@sdk-56, dev build, Router shell, theme tokens, AGENTS.md

Phase 1 — MVP (1–2 weeks)
  [list screens + data + one happy path E2E]

Phase 2 — v1 polish (1–2 weeks)
  onboarding, empty states, haptics, Sentry, store assets, paywall if any

Phase 3 — Post-launch
  OTA fixes, analytics, widgets, i18n — only items user validated
```

Each phase ends with a **demoable checkpoint** — not “50% of backend done.”

---

## STEP 8 — UX & ANTI-SLOP DIRECTION

Before any UI mock, specify:

| Topic | Decision |
|---|---|
| **Home metaphor** | path / inbox / calendar / map / shelf — one dominant pattern |
| **Primary color** | one brand hue + neutral scale — no purple AI default |
| **Raised surfaces** | ≤ 2 per screen (`anti-ai-slop.md`) |
| **Empty states** | one illustration or icon + one action — not 3 cards |
| **Copy tone** | friendly / professional / playful — match audience |
| **Locales** | single language MVP or i18n from day 1? (`i18n-rtl.md`) |

Premium checklist preview: `references/premium-feel.md` — press scale, haptics on commit, skeletons not spinners.

---

## STEP 9 — RISKS & STORE REALITY

Flag early:

| Risk | Mitigation |
|---|---|
| Health / finance / kids | Compliance, disclaimers, age gate — `store-aso.md` |
| Mic / camera / location | Permission copy + fallback UI — `ux-psychology.md` |
| User-generated content | Report flow, moderation plan |
| OAuth / deep links | `universal-links.md` |
| “AI” claims in store listing | Accurate screenshots; no fake metrics |

---

## PRODUCT BRIEF — OUTPUT TEMPLATE

**Copy this structure into the chat.** Replace bracketed sections. Keep it scannable.

```markdown
# [App Name] — Production Brief

## Seed
> User said: “[exact quote]”

## One-liner
[Positioning sentence]

## Target user
- **Who:** …
- **Job:** …
- **Alternatives today:** …
- **Wedge:** …

## Assumptions (confirm or correct)
1. …
2. …

## MVP features (in scope)
| # | Feature | User job | Primary screen |
|---|---|---|---|
| 1 | … | … | … |

## Explicitly NOT in MVP
- …

## Route map (Expo Router)
[tree]

## Data model
[entities + storage per entity]

## Stack choices
| Layer | Choice | Why |
|---|---|---|
| … | … | … |

## Screen list (MVP)
| Screen | Route | List component | Notes |
|---|---|---|---|
| … | … | … | … |

## Visual direction (anti-slop)
- Metaphor: …
- Primary CTA pattern: …
- Tab style: NativeTabs / floating glass / JS tabs

## Monetization
[model + v1 timing or “none for MVP”]

## Phases
### Phase 0 — …
### Phase 1 — MVP …
### Phase 2 — v1 …

## Open questions (max 2)
1. …
2. …

## Next step
Reply **“build it”** to scaffold with Prompt 1 in `vibe-coder-prompts.md`, or correct assumptions above.
```

---

## EXAMPLE — SEED → BRIEF (abbreviated)

**Seed:** “app that helps me remember what I planted in my garden”

**One-liner:** Plotmark helps home gardeners remember what they planted where by linking bed maps to dated plant logs — unlike a notes app, beds are visual and season-aware.

**MVP (4 tabs max):** Home (today’s tasks), Beds (grid), Log (add entry), Profile  
**NOT MVP:** weather API, social, plant ID AI, iPad  

**Routes:** `(tabs)/index`, `(tabs)/beds`, `(tabs)/log`, `(tabs)/profile`, `bed/[id]`, `plant/[id]`  
**Data:** SQLite local MVP; MMKV for prefs; v1 Supabase sync  
**Lists:** Beds grid → FlashList; Log → Legend List (variable photo rows)  
**Visual:** top-down bed map metaphor; earthy green accent; no stat dashboard  

---

## AGENT QUALITY BAR

Before sending the brief, verify:

- [ ] MVP has **≤ 5** core screens and **≤ 4** tabs
- [ ] Every feature maps to a **user job**, not “because competitors have it”
- [ ] Route tree is **implementable** with Expo Router file conventions
- [ ] List component chosen **per screen**, not FlashList everywhere
- [ ] Assumptions labeled; **≤ 2** questions — not a 20-question interview
- [ ] “NOT in MVP” list is non-empty (proves scope discipline)
- [ ] No code files generated unless user confirmed

---

## HANDOFF AFTER APPROVAL

When user says **build / implement / start**:

1. `references/foundation.md` — create project + dev build  
2. `references/vibe-coder-prompts.md` — **Prompt 1** with brief values filled in  
3. `references/navigation.md` — wire route tree  
4. `references/data-state.md` — storage + Query setup  
5. `references/mastery-rubric.md` — pre-ship audit before “done”

---

## REFERENCES

- `vibe-coder-prompts.md` — Prompt: Seed Idea → Production Plan  
- `anti-ai-slop.md` — visual direction  
- `ux-psychology.md` — cognitive load, permissions  
- `mastery-rubric.md` — don’t over-scope for “999 stack”  
- `store-aso.md` — naming, categories, rejection patterns
