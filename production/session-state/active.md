# Session State — Altyn Adam

**Last updated**: 2026-05-08  
**Current task**: Designing health-stamina-system GDD (5/27)  

## Status
- [x] Engine configured: Unreal Engine 5.7
- [x] GDD migrated: design/gdd/game-concept.md
- [x] Systems index created: design/gdd/systems-index.md (27 systems)
- [x] Per-system GDDs (2/27) — input-system.md + save-load-system.md COMPLETE
- [ ] Architecture overview

## Next Action
Design next system GDD — `health-stamina-system.md` (Priority 5 in design order)

Run `/design-system health-stamina-system` to begin, or `/map-systems next` to auto-pick.

## Key Decisions
- Survival System removed from scope (water mechanics would complicate solo-dev scope)
- Armor/Fragment System moved to MVP (first fragment in Chapter 2, needs to be ready)
- Review mode: lean
- Input System: IMC_AimBow uses Scalar modifier ×0.35 on IA_Move (not a separate action)
- Input System: bPlayerMappable on IMC asset binding (not C++ struct — removed UE 5.5)
- Input System: remap persistence via USaveGame, no built-in auto-save in UE 5.7
- Input System: IA_Fire is separate from IA_AimBow Completed trigger

## Session Update — 2026-05-09 (design-system)
- Task: save-load-system GDD — COMPLETE (all 8 sections)
- File: design/gdd/save-load-system.md
- Key decisions: 2 USaveGame classes (UAltynSaveGame + UAltynSettingsSaveGame), UGameInstanceSubsystem host, async primary saves, synchronous checkpoint, SaveVersion=1, DefaultClanTrust=50.0

## Session Update — 2026-05-09 (design-system)
- Task: audio-system GDD — COMPLETE (all 8 sections)
- File: design/gdd/audio-system.md
- Key decisions: UAudioStateManager UGameInstanceSubsystem; single MetaSound asset with 4 parameter-driven layers; 3 music states (Calm/Suspicion/Combat); Combat→Calm crossfade 8.0s; Combat dwell 20s; Kobyz = sustained+release, 4-voice polyphonic, 6 pitches (C major pentatonic + octave); 4 Sound Mixes (Narrative/Chatter/Combat Alertness/Cinematic); 4 player volume settings (Master/Music/SFX/Ambient); bird scatter 20% misfire; finale scripted override via bFinalChargeBegun; post-combat Exhalation Cue fires 4s after Combat→Calm
- Per-system GDDs: 3/27 — input-system.md + save-load-system.md + audio-system.md COMPLETE
- Next: /design-system character-controller (Priority 4 in design order)

## Session Update — 2026-05-09 (design-system)
- Task: character-controller GDD — COMPLETE (all sections)
- File: design/gdd/character-controller.md
- Key decisions: AAltynCharacter:ACharacter + CMC (no custom subclass at MVP); Walk 300 UU/s, Sprint 560 UU/s (3-phase: explosive/degrading/recovery), Crouch 150 UU/s (Kazakh squat posture); Dodge = lateral step-aside 180 UU (NOT roll), DodgeCooldown 1.2s, -40% hitbox frames 5–14; JumpZVelocity 380 UU/s (~75cm clearance); HardFallThreshold 150 UU (stumble only, no HP damage); OnMovementNoisePulse delegate (Crouch Idle 0.0, CrouchWalk 0.15, Walk 0.45, Dodge 0.30, Sprint 0.85, HardLanding 0.90); turning-in-place locomotion (not free-strafing); 3 idle variants (Calm/Aware/Tense) driven by Stealth System; bControllerActive flag for horse handoff; position restore from UAltynSaveGame on OnSaveLoaded
- Per-system GDDs: 4/27 — input-system.md + save-load-system.md + audio-system.md + character-controller.md COMPLETE
- Next: /design-system health-stamina-system (Priority 5 in design order)
