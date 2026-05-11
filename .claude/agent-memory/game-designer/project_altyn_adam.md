---
name: Project — Altyn Adam (Game Designer view)
description: Design-relevant facts about Altyn Adam for the game-designer agent
type: project
---

**Game**: Altyn Adam («Золотой Человек»)
**Tone**: Realistic, brutal — not fantasy. Ghost of Tsushima atmosphere, AC2 structure.
**Protagonist**: Unnamed boy, ~14–15, last survivor of the Golden People. A "witness earning proximity to the boy" — body has its own logic.
**Three pillars**: (1) Clan journey; (2) Personal journey trauma → identity; (3) Cavalry charge payoff.

**Health & Stamina system decided facts (2026-05-09)**:
- Two floats: HealthCurrent, StaminaCurrent
- Death at 0 HP → respawn at checkpoint, full health
- No mid-combat HP regen — campfire/rest only
- Stamina drains on sprint (~8s to exhaustion) and combat exertion; at 0 → forces Sprint_Recovery
- Hard fall: NO HP damage (decided)
- No visible bars during exploration — animation + audio only
- Combat System (not yet designed) reads StaminaCurrent to gate heavy attacks + blocking

**GDD files**:
- `design/gdd/health-stamina-system.md` — In Design (skeleton only, Detailed Design section not yet authored)
- `design/gdd/character-controller.md` — Complete
- `design/gdd/systems-index.md` — 4/27 systems complete

**Why:** Realistic tone demands that body state communicates through world signals, not UI numbers. The "body has its own logic" framing is a core pillar expression.

**How to apply:** All health/stamina design decisions must serve the "you are a witness, not a god" fantasy. Systems that let the player min-max body state (regen camping, stamina farming) undermine the tone.
