# Altyn Adam — Systems Index

**Last updated**: 2026-05-09  
**Total systems**: 27  
**Designed**: 4 / 27  

---

## Progress Tracker

| System | Milestone | Status | GDD File |
|--------|-----------|--------|----------|
| Input System | MVP | Complete | input-system.md |
| Save / Load System | MVP | Complete | save-load-system.md |
| Audio System | MVP | Complete | audio-system.md |
| Character Controller | MVP | Complete | character-controller.md |
| Health & Stamina System | MVP | Not Started | — |
| Camera System | MVP | Not Started | — |
| Stealth System | MVP | Not Started | — |
| Enemy AI System | MVP | Not Started | — |
| Combat System | MVP | Not Started | — |
| Dialogue & Choice System | MVP | Not Started | — |
| Memory System | MVP | Not Started | — |
| Ability System | MVP | Not Started | — |
| Armor / Fragment System | MVP | Not Started | — |
| NPC Faction & Trust System | MVP | Not Started | — |
| Objective / Mission Tracker | MVP | Not Started | — |
| Cinematic / Cutscene System | MVP | Not Started | — |
| Combat & Stealth HUD | MVP | Not Started | — |
| Dialogue UI | MVP | Not Started | — |
| Open World / Level Streaming | Vertical Slice | Not Started | — |
| Horse System | Vertical Slice | Not Started | — |
| Hub System | Vertical Slice | Not Started | — |
| Map & Navigation System | Vertical Slice | Not Started | — |
| Memory Journal UI | Vertical Slice | Not Started | — |
| Map & Minimap UI | Vertical Slice | Not Started | — |
| Armor / Equipment UI | Vertical Slice | Not Started | — |
| Kobyz Music System | Alpha | Not Started | — |
| Name / Legacy System | Full Vision | Not Started | — |

---

## Recommended Design Order

Design in this order — each system's GDD must exist before dependent systems are designed.

### Layer 1 — Foundation (no dependencies)
1. `input-system.md` — MVP
2. `save-load-system.md` — MVP ⚠️ HIGH RISK (5 dependents)
3. `audio-system.md` — MVP
4. `character-controller.md` — MVP ⚠️ HIGH RISK (6 dependents)

### Layer 2 — Core (depend on Foundation)
5. `health-stamina-system.md` — MVP
6. `camera-system.md` — MVP
7. `stealth-system.md` — MVP
8. `combat-system.md` — MVP
9. `dialogue-choice-system.md` — MVP ⚠️ HIGH RISK (5 dependents)

### Layer 3 — Features (depend on Core)
10. `enemy-ai-system.md` — MVP
11. `memory-system.md` — MVP
12. `ability-system.md` — MVP
13. `armor-fragment-system.md` — MVP
14. `npc-faction-trust-system.md` — MVP
15. `objective-mission-tracker.md` — MVP
16. `cinematic-cutscene-system.md` — MVP
17. `open-world-streaming-system.md` — Vertical Slice
18. `horse-system.md` — Vertical Slice
19. `hub-system.md` — Vertical Slice
20. `map-navigation-system.md` — Vertical Slice
21. `kobyz-music-system.md` — Alpha

### Layer 4 — Narrative / Final (depend on Features)
22. `name-legacy-system.md` — Full Vision

### Layer 5 — Presentation / UI (design after gameplay system)
23. `combat-stealth-hud.md` — MVP
24. `dialogue-ui.md` — MVP
25. `memory-journal-ui.md` — Vertical Slice
26. `map-minimap-ui.md` — Vertical Slice
27. `armor-equipment-ui.md` — Vertical Slice

---

## Dependency Map

```
FOUNDATION
  Input System ──────────────────────────────────┐
  Save / Load System ─────────────────────────────┤
  Audio System ───────────────────────────────────┤
  Character Controller ───────────────────────────┘
         │
         ▼
  CORE
  Health & Stamina ◄─── Character Controller
  Camera System    ◄─── Character Controller
  Stealth System   ◄─── Character Controller, Input
  Combat System    ◄─── Character Controller, Health & Stamina
  Dialogue System  ◄─── Input, Save/Load
         │
         ▼
  FEATURES
  Enemy AI         ◄─── Combat, Stealth, Health & Stamina
  Memory System    ◄─── Dialogue, Save/Load
  Ability System   ◄─── Memory, Combat
  Armor/Fragment   ◄─── Ability, Save/Load
  NPC Faction      ◄─── Dialogue, Save/Load
  Mission Tracker  ◄─── Dialogue, NPC Faction
  Cinematic System ◄─── Camera, Dialogue, Audio
  Open World       ◄─── Character Controller
  Horse System     ◄─── Character Controller, Combat
  Hub System       ◄─── Dialogue, NPC Faction, Open World
  Map & Navigation ◄─── Open World
  Kobyz Music      ◄─── Audio, Input, Open World
         │
         ▼
  NARRATIVE / FINAL
  Name/Legacy      ◄─── NPC Faction, Dialogue, Save/Load
         │
         ▼
  PRESENTATION
  Combat & Stealth HUD  ◄─── Combat, Stealth, Health & Stamina
  Dialogue UI           ◄─── Dialogue System
  Memory Journal UI     ◄─── Memory, Ability
  Map & Minimap UI      ◄─── Map & Navigation
  Armor / Equipment UI  ◄─── Armor / Fragment
```

---

## High-Risk Systems

These are bottlenecks — errors in their architecture block many other systems.
Design these carefully and get architecture review before implementation.

| System | Risk | Reason |
|--------|------|--------|
| Character Controller | ⚠️ HIGH | 6 systems depend on it directly |
| Save / Load System | ⚠️ HIGH | 5 systems depend on it; wrong architecture = data loss |
| Dialogue & Choice System | ⚠️ HIGH | 5 systems depend on it; choices must persist correctly |

---

## System Descriptions

### MVP Systems

**Input System**  
Handles keyboard/mouse and gamepad input. Routes actions to gameplay systems. Supports remapping.

**Save / Load System**  
Persists player progress: choices, memories, armor fragments, faction trust levels, world state. Must support mid-chapter checkpoints.

**Audio System**  
Music, SFX, ambient. Critical for Kobyz mechanic and atmospheric tone. Must support dynamic music layers (calm → tense → combat).

**Character Controller**  
Player movement: walking, running, crouching, climbing, rolling. Foundation for all interaction and traversal.

**Health & Stamina System**  
Player resource management. Health depletes from combat damage. Stamina gates sprinting, heavy attacks, blocking. Death and respawn flow.

**Camera System**  
Third-person follow camera. Combat lock-on mode. Cinematic control for cutscenes. Dynamic distance based on context.

**Stealth System**  
Detection model: vision cones, hearing radius, alert states (unaware → suspicious → alerted). Cover mechanics. Distractions.

**Enemy AI System**  
Patrol behavior, alert state machine, group coordination, combat AI. Responds to stealth detection. Horse-mounted variants for Chapter 3+.

**Combat System**  
Melee (light attack, heavy attack, block, parry, dodge) + Ranged (bow aiming, draw speed, arrow arc). Combo system. Hit detection.

**Dialogue & Choice System**  
Dialogue tree authoring and playback. Player choice selection. Choice consequence tracking. NPC state changes based on choices.

**Memory System**  
Collectible «Memories» found through dialogue, rituals, exploration. Each Memory grants a passive buff or unlocks an ability. Stored in the Memory Journal.

**Ability System**  
Unified container for active and passive abilities from Memories and Armor fragments. Manages activation, cooldowns, and passive application.

**Armor / Fragment System**  
Six clan armor fragments, each with a visual model change and linked unique ability. Fragment unlock flow. Full armor visual state in final battle.

**NPC Faction & Trust System**  
Tracks trust level per clan (0–100). Modified by quest outcomes and dialogue choices. Determines allied army composition in the final battle.

**Objective / Mission Tracker**  
Chapter-level task list. Tracks active objectives, completion state, optional tasks. AC2-style notification on completion.

**Cinematic / Cutscene System**  
Scripted camera sequences and animation playback for key story moments (mother's death, haoma ritual, final battle opening). Blends in/out of gameplay.

**Combat & Stealth HUD**  
Health bar, stamina bar, stealth detection meter, enemy alert icons, bow reticle, ability slot display.

**Dialogue UI**  
Dialogue box, speaker name, NPC portrait or scene focus. Choice list with 2–4 options. Fade in/out transitions.

### Vertical Slice Systems

**Open World / Level Streaming**  
Per-chapter small open world. Unreal World Partition for streaming. LOD management. Point of interest spawning and state.

**Horse System**  
Mounted movement (walk, trot, gallop). Mounted combat: melee swings and bow. Horse stamina. Dismount/mount flow. Horse AI (stays nearby).

**Hub System**  
Tomyris camp between chapters. NPC conversations, training interactions, preparation. Hub state updates as story progresses.

**Map & Navigation System**  
World map per chapter region. POI discovery and marking. Objective markers. Fast travel points (limited — preserves exploration pacing).

**Memory Journal UI**  
Full Memory log with lore text and portrait. Ability unlock display. Progress tracking per chapter.

**Map & Minimap UI**  
In-HUD minimap during exploration. Full map screen. Objective and POI markers. Player direction indicator.

**Armor / Equipment UI**  
Visual armor doll showing equipped fragments. Fragment ability description. Locked/unlocked state per clan.

### Alpha Systems

**Kobyz Music System**  
Melody input interface (hold + directional input per note). World activation points. Melody effects: open paths, calm spirits, buff allies. Library of 6 clan melodies built over the game. Final composite melody.

### Full Vision Systems

**Name / Legacy System**  
Tracks all player actions: choices made, clan trust levels, enemies spared or killed, memories collected. Generates a name from a predefined table based on weighted score. Presented as the final scene before the last battle.

---

*Systems Index · Altyn Adam · Generated 2026-05-08*
