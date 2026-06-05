# Anti–AI Slop (Universal)

**Canonical skill (all stacks):** `anti-ai-slop` — read `SKILL.md` + `reference.md` in that skill folder.

This file is a **mirror** for Expo/RN workflows that only open `rn-expo-stack` references. Content is project-agnostic (no single-app components or colors).

---

## Expo / RN addendum

When building with this stack, also read:

| File | Topic |
|------|--------|
| `premium-feel.md` | Press, haptics, motion timing |
| `ui-design.md` | Expo UI, Uniwind, Liquid Glass |
| `ux-psychology.md` | Empty states, touch targets |
| `speed-perception.md` | Skeletons, loading |
| `i18n-rtl.md` | Locales, RTL |

**RN-specific checks (in addition to universal checklist):**

- Reuse components under the **current repo’s** `components/`, `theme/`, or design-system path
- Do not add a second glass/card system beside the one already in the project
- `expo-haptics` on commits, not every scroll
- NativeTabs / tab order for directional transitions when using platform tabs

For full red-flag tables, token template, pseudocode, and prompts → **`anti-ai-slop/reference.md`**.
