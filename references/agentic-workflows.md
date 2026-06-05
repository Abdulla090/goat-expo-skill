# Agentic Workflows — Agents, QA, Debug (June 2026)

**Default stack:** `AGENTS.md` + `npx skills add expo/skills` + **Maestro** for deterministic CI E2E.

**Optional augment (not replacements):** agent-driven exploration and deep sim debugging.

---

## BASELINE (every project)

```powershell
npx skills add expo/skills
```

- Template ships **AGENTS.md** / **CLAUDE.md** — project-specific rules beat generic LLM guesses.
- SDK bumps: use Expo upgrade skill + `npx expo-doctor` + Router codemod (`foundation.md`).
- **Expo serve-sim** — browser-hosted iOS sim for MCP-style verification (Expo direction).

---

## DETERMINISTIC E2E — MAESTRO (primary)

```powershell
# https://maestro.mobile.dev
maestro test flows/login.yaml
```

Use for: release gates, paywall paths, auth, tab navigation smoke.

**Maestro stays the CI source of truth.** Agent tools explore edge cases; they do not replace replayable flows.

---

## AGENT DEVICE (Callstack) — exploratory QA

**What it is:** CLI + thin MCP router for AI agents to drive simulators/emulators, capture accessibility snapshots (`@e1` refs), RN component trees, screenshots, logs.

**When to add:**
- Agent verifies a screen after implementation
- Exploratory tap-through before writing Maestro YAML
- Physical device + TV targets (beyond Maestro’s sweet spot)

**Setup:**

```powershell
npm install -g agent-device@<pinned-version>
npx skills add callstackincubator/agent-device
```

MCP (discovery only — automation runs via CLI):

```json
{
  "mcpServers": {
    "agent-device": {
      "command": "npx",
      "args": ["-y", "agent-device@<pinned-version>", "mcp"]
    }
  }
}
```

**Agent rule:** Run `agent-device help workflow` and `agent-device help react-native` before device commands. MCP exposes `status` / `install` / `help` — not full shell.

Docs: https://agent-device.dev

---

## ARGENT (Software Mansion) — deep debug / profile

**What it is:** MCP toolkit for simulators — tap/swipe, RN component tree, logs, HTTP (JS + native layer), React + Instruments profiling.

**When to add:**
- Reproduce jank / memory growth with agent-assisted profiling
- Debug voice/audio/session leaks after Hermes heap snapshot (`performance.md`)

**Setup:**

```powershell
npx @swmansion/argent init
```

Registers MCP for Cursor/Claude/etc. and copies workspace skills.

**Agent rule:** Argent complements Sentry + manual profiling — not a substitute for production monitoring (`build-deploy.md`).

Docs: https://argent.swmansion.com

---

## DECISION TABLE

| Need | Tool |
|------|------|
| CI release smoke | **Maestro** |
| SDK upgrade / config plugins | **expo/skills** + `expo-doctor` |
| Agent implements UI, needs tap-verify | **Agent Device** (optional) |
| Perf regression on sim | **Argent** or Hermes profiler (`performance.md`) |
| Production crashes | **Sentry** |

---

## REJECT FROM GENERIC “2026 RESEARCH” REPORTS

Do **not** treat these as stack requirements:

- **React Native Evals / Apex model** — benchmarks for model vendors, not app architecture
- **Commercial boilerplates** (Ship templates, NativeLaunch) — optional scaffolds, not defaults
- **RevenueCat aggregate stats** (Day-0 churn %, hard paywall multiples) — product analytics, not RN stack
- **Case study apps** (Inkigo, Callie) — inspiration only, not dependencies
- **“Agents replace Maestro”** — wrong; agents augment humans, Maestro gates releases

Filter external research through `mastery-rubric.md` Pre-ship audit.

---

## KEEP SKILLS CURRENT (field learnings) — **mandatory after non-trivial fixes**

When you fix a bug that is **non-obvious**, **platform-specific**, or a **repeatable AI mistake** (not a one-line typo):

1. **Fix in code** — add shared utils (`crossShadow`, `springMotion`) when the pattern appears twice.
2. **Append to `references/field-bug-playbook.md`** — symptom → root cause → fix → AI trap → test step (use the template at bottom of that file).
3. **Update the focused reference** — web → `web-rn-pitfalls.md`; tabs → `floating-glass-tab-bar.md`; nav → `navigation.md`; crashes → `error-boundaries.md`.
4. **Sync** `.cursor/skills/goat-expo-skill/` → `~/.agents/skills/rn-expo-stack/` when the user maintains both copies.

**Document if:** next agent would likely reintroduce it (glass on tiny chips, `useAnimatedReaction` for tabs, navigate mid-Reanimated press, stale Metro phantom errors).

**Skip if:** product-only logic, copy, or a single-file typo with no pattern.

**Examples already in playbook:** Guidebook Android crash, web shadow/pointerEvents deprecations, tab optimistic animation, hidden-route `fade` on Android.
