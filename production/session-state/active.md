# Session State — Altyn Adam

**Last updated**: 2026-05-11  
**Current task**: Enemy AI System GDD — in design (Section A)  

## Status
- [x] Engine configured: Unreal Engine 5.7
- [x] GDD migrated: design/gdd/game-concept.md
- [x] Systems index created: design/gdd/systems-index.md (27 systems)
- [x] Per-system GDDs (2/27) — input-system.md + save-load-system.md COMPLETE
- [ ] Architecture overview

## Next Action
Run /design-system enemy-ai-system (Priority 10 in design order)

File: design/gdd/dialogue-choice-system.md — COMPLETE

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
## Session Update — 2026-05-09 (design-system)
- Task: health-stamina-system GDD — COMPLETE (all sections)
- File: design/gdd/health-stamina-system.md
- Key decisions: HealthMax=100, StaminaMax=100 (provisional); SprintDrainRate=12.5 UU/s (8s exhaust target); StaminaRecoveryRate=20.0 UU/s (5s full recovery); NO mid-combat stamina regen; WoundedThreshold=30 HP (additive locomotion layer, right shoulder drop 10°, no stat debuff); HealthCurrent persists across manual saves, resets to HealthMax on death-load; StaminaCurrent always resets on any load; full campfire restore; near-death signal = breath catch + 1.5° camera tilt (NO vignette/heartbeat); OnPlayerDeath + OnStaminaExhausted + OnWoundedStateEntered/Exited delegates
- Cross-system flag: Save/Load GDD must add HealthCurrent to UAltynSaveGame schema
- Cross-system flag: Audio System GDD — block post-combat dombra phrase if campfire rest starts within 8s of combat end
- Cross-system flag: Camera System GDD — near-death camera drift must be coordinated
- Per-system GDDs: 5/27 — input + save-load + audio + character-controller + health-stamina COMPLETE
- Next: /design-system camera-system (Priority 6 in design order)

## Session Update — 2026-05-10 (design-system)
- Task: camera-system GDD — COMPLETE (all sections)
- File: design/gdd/camera-system.md
- Key decisions: ExplorationArmLength=475 UU; CombatArm=88% (~418 UU); AimBowArm=70% (~333 UU); FOV_Exploration=80°; SprintFOVBonus=5°; MaxLockOnRange=1000 UU; LockOnArmScale=0.15; MaxLockOnOrbitAngle=25°; ValidityScore formula (distance 0.4 + angle 0.4 + LOS 0.2); NearDeathCameraTilt=1.5° (clears only on OnWoundedStateExited, NOT at 10→11 HP); CameraImpulseMagnitude=3.0 UU (UCameraModifier, not CameraShake); bow aim = CS_AimBow state (Model A — lock-on melee-only); CS_Cinematic restores to prior state (stack), clears lock-on; all camera changes via UCameraModifier; IMovieScenePlayer deprecated UE5.6 — use SetViewTarget + ULevelSequencePlayer
- Cross-system flag: Input System GDD must add IA_LockOn as Bool/Ongoing hold action in IMC_OnFoot (not IMC_AimBow, not IMC_Mounted)
- Cross-system flag: Cinematic/Cutscene System GDD — use ULevelSequencePlayer, avoid IMovieScenePlayer (deprecated 5.6)
- Cross-system flag: Open World/Hub GDDs — indoor combat spaces need min ceiling clearance ~300 UU
- Per-system GDDs: 6/27 — input + save-load + audio + character-controller + health-stamina + camera COMPLETE
- Next: /design-system stealth-system (Priority 7 in design order)

## Session Update — 2026-05-11 (design-system) — Stealth System
- Task: stealth-system GDD — COMPLETE (all sections)
- File: design/gdd/stealth-system.md
- Key decisions: Per-enemy independent detection (Stealth System is substrate; Enemy AI handles group coordination); 4-state alert model (Unaware/Suspicious/Alert/Searching — Searching locked by CC GDD Rule 9); accumulation detection gauge (0.0–1.0, 0.1 s tick for vision + event-driven pulse for noise); 2-zone vision cone (Focused ±30° 1200 UU, Peripheral ±60° 500 UU); grass = noise multiplier only (no LOS occlusion through foliage); crouching reduces VisionStimulus via CrouchVisibilityMultiplier=0.6; per-enemy HearingThreshold (not global floor); stone throw = only MVP distraction (Loudness=0.75, independent of terrain); UAltynStealthSubsystem : UWorldSubsystem for global aggregate; FAINoiseEvent.Loudness carries EffectiveNoise (no custom UAISense needed); PhysicalMaterial shared with Audio System for terrain lookup; LightModifier tuning knob designed (ToD system future integration); Searching state heightened senses (FillRate ×1.5, HearingThreshold ×0.7); Player Fantasy: "The Steppe Hides Its Own" — anti-predator, grief-as-physical, inherited knowledge
- Cross-system flags: Input System GDD must add IA_Throw (Bool, hold-release, IMC_OnFoot; Gamepad R1, KB G); Audio System GDD — UAudioStateManager subscribes to OnAwarenessChanged; Cinematic/Cutscene GDD — must freeze/resume stealth gauge on CS_Cinematic; Open World GDD — all navigable surfaces must have PhysicalMaterial assigned
- Per-system GDDs: 7/27 — input + save-load + audio + character-controller + health-stamina + camera + stealth COMPLETE
- Next: /design-system combat-system (Priority 8 in design order)

## Session Update — 2026-05-11 (design-system) — Combat System
- Task: combat-system GDD — COMPLETE (all sections)
- File: design/gdd/combat-system.md
- Key decisions: UCombatComponent:UActorComponent (NOT GAS); bInCombat owned by Combat, triggers on enemy Alert within 1500 UU; LightDamage_base=25.0; HeavyAttackCost=25.0 (×2.2 multiplier); BlockCostPerHit=15.0; parry=tap IA_Block <0.18s (no stamina cost, staggers enemy 1.4s); ComboInputWindow=0.55s; 3-hit combo (LA1/LA2/LA3 with 1.4× finisher); ParryActiveWindow=0.22s with FTimerHandle force-close (UE 5.7 UAnimNotifyState guard); Dodge stamina owned by CC unconditionally (resolves H&S OQ-1); Arrow=AArrowProjectile physics projectile, gravity 0.35, draw time 0.4s gates 0.5×/1.0× multiplier; PostCombatWindow=5.0s; StunDuration=0.6s; IHealthInterface for damage (not direct cast); OnBowAimChanged delegate (no shared enum with Camera); 9 combat states (CS_Idle/CombatIdle/LightAttack/HeavyAttack/Blocking/Parrying/AimBow/Stunned/Dead)
- Cross-system flags: Input System GDD — confirm IA_Block hold-duration discrimination OR add IA_Parry; H&S GDD — add OnPlayerHit(FVector) delegate; Save/Load GDD — add OnCombatEncounterStart() checkpoint trigger; Enemy AI GDD — EnemyParryStunDuration=1.4s provisional
- Per-system GDDs: 8/27 — input + save-load + audio + character-controller + health-stamina + camera + stealth + combat COMPLETE
- Next: /design-system dialogue-choice-system (Priority 9 in design order)

## Session Update — 2026-05-11 (design-system) — Dialogue & Choice System
- Task: dialogue-choice-system GDD — COMPLETE (all sections)
- File: design/gdd/dialogue-choice-system.md
- Key decisions: UDialogueTree:UPrimaryDataAsset (no external tools); UDialogueSubsystem:UGameInstanceSubsystem; 4 node types (Linear/Choice/Condition/Event); FDialogueConsequence 5 types (TrustDelta/SaveFlagWrite/MissionAdvance/MemoryUnlock/CustomEvent); bHideIfUnavailable=false (hidden not greyed — non-negotiable, implements "no moral markers"); consequences synchronous same-frame, data written before save trigger; IMC_Dialogue priority 2 (add at RequestDialogue, remove at EndDialogue); TypewriterCharactersPerSecond=40.0 via FTimerManager (NOT UUMGSequencePlayer — deprecated 5.6); TrustDelta clamped to [0.0,100.0]; bOneShot trees record TreeID_completed flag; NewTrustLevel = clamp(CurrentTrustLevel + TrustDelta, 0.0, 100.0); 5-state machine (DS_Idle/Playing/AwaitingChoice/Resolving/Complete); lowercase_snake_case FName convention for node IDs; Player lines are silent (SpeakerID=="player" → null VoiceCue)
- Cross-system flags: Input System GDD — add IMC_Dialogue with 4 bindings (IA_DialogueAdvance/PrevChoice/NextChoice/Cancel, priority 2); Stealth System GDD — expose PauseGauge()/ResumeGauge() as public API on UAltynStealthSubsystem; Camera System GDD — subscribe to OnDialogueStarted(AActor*)/OnDialogueEnded(); Save/Load System GDD — confirm DialogueChoices:TMap<FName,int32> and ClanTrustLevels:TMap<FName,float> in UAltynSaveGame schema
- Per-system GDDs: 9/27 — input + save-load + audio + character-controller + health-stamina + camera + stealth + combat + dialogue COMPLETE
- Next: /design-system enemy-ai-system (Priority 10 in design order)
