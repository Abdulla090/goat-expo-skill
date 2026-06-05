# 🐐 GOAT Expo Skill

**The agent skill for production-grade React Native + Expo apps (SDK 56, June 2026).**

Not another “use FlashList everywhere” blog post. A **ship-proof** decision system for Cursor, Claude Code, Codex, and any agent that reads `SKILL.md` — navigation, lists, glass UI, EAS, Sentry, anti–AI-slop, and honest stack scoring.

[![Expo SDK 56](https://img.shields.io/badge/Expo-SDK%2056-000020?style=flat-square&logo=expo&logoColor=white)](https://expo.dev/changelog/sdk-56)
[![React Native 0.85](https://img.shields.io/badge/React%20Native-0.85-61DAFB?style=flat-square&logo=react&logoColor=black)](https://reactnative.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](LICENSE)

---

## Why this exists

AI agents default to **2023 tutorials**: Expo Go, `@react-navigation/*` in app code, `estimatedItemSize` on FlashList v2, purple gradient dashboards, and “920/1000” scores from `package.json` alone.

**GOAT Expo Skill** fixes that with:

- ✅ **SDK 56** defaults — Router fork imports, `expo/fetch`, dev builds, Nitro/MMKV alignment  
- ✅ **NativeTabs vs floating liquid glass** — when to use system chrome vs custom frosted pill tab bar  
- ✅ **List discipline** — FlashList v2 / Legend List v3 / ScrollView rules per screen  
- ✅ **Production edge cases** — Sentry `traceFetch` vs `EXPO_PUBLIC_USE_RN_FETCH`, secrets in SecureStore  
- ✅ **Anti–AI-slop UI** — max 2 raised surfaces, repo tokens first, no vendor marketing chrome  
- ✅ **Honest mastery rubric** — stack knowledge ≠ your app score; pre-ship audit checklist  

**Real-world proof:** [Phingo](https://github.com/Abdulla090/Zheer_saz_doulingo) — full learning app (~38k LOC, 25+ routes, i18n, AI, EAS) built in **~1 week** with agents + this skill.

---

## Install

### Cursor (project — already in a repo)

```text
.cursor/skills/goat-expo-skill/
```

Clone or copy this repo into `.cursor/skills/goat-expo-skill/` in your Expo project.

### Cursor / agents (global)

```powershell
git clone https://github.com/Abdulla090/goat-expo-skill.git "$env:USERPROFILE\.agents\skills\goat-expo-skill"
```

Or symlink `SKILL.md` + `references/` into your global skills folder.

### Skills CLI (if your agent supports `npx skills add`)

```powershell
npx skills add Abdulla090/goat-expo-skill
```

### Manual

1. Copy `SKILL.md` and the `references/` folder into your agent’s skills directory.  
2. Tell the agent: *“Use goat-expo-skill for all Expo / RN work.”*

---

## Quick start

1. Open **`SKILL.md`** — read the reference file map.  
2. New app → `references/foundation.md` + `references/navigation.md`.  
3. UI / glass tabs → `references/ui-design.md` + `references/floating-glass-tab-bar.md`.  
4. Before ship → `references/mastery-rubric.md` pre-ship audit.  
5. Paste prompts from `references/vibe-coder-prompts.md`.

**Trigger phrases:** `expo sdk 56`, `react native stack`, `native tabs`, `liquid glass`, `floating tab bar`, `rate my expo stack`, `anti ai slop`, `production expo app`.

---

## What’s inside

| File | Topic |
|------|--------|
| `SKILL.md` | Stack at a glance, decision rules, golden rules |
| `references/foundation.md` | SDK 56, dev client, New Arch, Hermes |
| `references/navigation.md` | Expo Router, NativeTabs, import migration |
| `references/floating-glass-tab-bar.md` | Optional frosted pill tab bar over mesh UI |
| `references/ui-design.md` | Expo UI, Uniwind, Liquid Glass |
| `references/performance.md` | FlashList v2, Legend List v3, profiling |
| `references/mastery-rubric.md` | Honest scoring + pre-ship checklist |
| `references/anti-ai-slop.md` | Universal anti-slop UI rules |
| `references/vibe-coder-prompts.md` | Copy-paste agent prompts |

Full map: see **REFERENCE FILE MAP** in `SKILL.md`.

---

## GOAT vs generic AI advice

| Generic AI | GOAT Expo Skill |
|------------|-----------------|
| FlashList on every list | Audit per screen (ScrollView / FlashList / Legend List) |
| Expo Go for production | `expo-dev-client` + EAS |
| `@react-navigation/native` in app | `expo-router/react-navigation` (SDK 56) |
| Score 920 from dependencies | Split knowledge vs execution; audit repo |
| Custom JSI for “1000 score” | Native modules only when measured need |
| Black `BlurView` tab bar on Android | Layered frost + sheen pattern |

---

## Contributing

PRs welcome — especially SDK changelog updates, production postmortems, and **reproducible** perf fixes.  
Do not add hype without evidence (see `references/mastery-rubric.md` → filtering external research).

---

## Star this repo ⭐

If this skill saved you a week of wrong-stack rework, **star it** so the next dev finds SDK 56 defaults instead of 2023 tutorials.

Share with: `#ReactNative` `#Expo` `#Cursor` `#AIcoding` — tags help discovery; quality keeps stars.

---

## License

MIT — see [LICENSE](LICENSE).

Built by [@Abdulla090](https://github.com/Abdulla090).
