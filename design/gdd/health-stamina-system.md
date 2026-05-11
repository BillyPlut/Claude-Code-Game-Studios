# Health & Stamina System

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-09
> **Implements Pillar**: Pillar 2 (personal journey — body as resource) + Pillar 3 (cavalry charge payoff)

## Overview

The Health & Stamina System tracks the boy's two body resources: `HealthCurrent` (hit points depleted by combat damage) and `StaminaCurrent` (effort capacity drained by sustained sprint and combat exertion). Both are floats owned by this system and exposed as read-only to dependent systems — the Character Controller reads `StaminaCurrent` every tick to gate sprint entry; the Combat System reads it to gate heavy attacks and blocking. Neither resource has a visible bar during normal exploration; the player reads body state from animation and audio feedback, not numbers. The HUD shows health only during active combat encounters (owned by Combat & Stealth HUD).

Health depletes when the boy takes combat damage. At zero, the death-and-respawn flow fires: a death animation, fade-to-black, and respawn at the last checkpoint with `HealthCurrent` restored to full. There is no mid-combat HP regeneration; health restores only at rest points between encounters. Stamina drains during sprint and during combat stamina expenditure, and recovers when the relevant activity stops. Stamina reaching zero forces the Character Controller out of sprint immediately — the body decides before the player does. Hard landings (≥150 UU fall, defined in the Character Controller GDD) do not damage health.

*Provisional*: Stamina gating of heavy attacks and blocking is noted here but will be fully specified in the Combat System GDD.

## Player Fantasy

You are not the boy. You are someone who has been allowed to walk inside him for a while. He has a body that was here before you arrived and will be here after you put the controller down, and at the edges of what that body can do, it will make decisions you did not authorize.

You hold sprint and you feel the wind take the sound of it, and at second eight the body opens against your input. The breath is louder than the music. The chest rises. The hand that was on the bow drops to the knee. You wanted nine seconds. The body gave you eight, because eight is what was there. You did not fail an input check. You watched a fourteen-year-old boy run as far as he could run, and then stop because he had to.

Health works the same way. You do not regulate it. You do not patch it on the move. The body holds the wound until the world is quiet enough to put it down. If the wound is more than the body can hold, the boy goes still and the steppe keeps walking, and you wake at the last warm fire because someone or something kept you. Death is not a punishment for poor play. It is the steppe reminding you that the boy was always borrowed.

The system does not ask you to manage the body. It asks you to listen to it. Eight seconds. The cut on the rib. The shortness of the stride. The small sounds the boy makes when he sits down. These are not numbers. These are how the body talks, and the game is in the conversation.

This is the contract you build across six clans. At the end of the game you ride into a Persian army and the body gives everything it has, because by then you know exactly what everything means. The cavalry charge is not the moment the boy becomes more than himself. It is the moment he spends all that he is — and the army moves anyway.

## Detailed Design

### Core Rules

**Rule 1 — Owned Floats**
The Health & Stamina System owns two authoritative floats: `HealthCurrent` and `StaminaCurrent`. No other system writes to either float directly. All modifications go through H&S functions: `ApplyDamage(float Amount)`, `RestoreHealth(float Amount)`, `DrainStamina(float Amount)`, and `RecoverStamina(float DeltaTime)`. External systems receive read-only access only.

**Rule 2 — Health Ceiling**
`HealthMax = 100.0` (provisional). `HealthCurrent` is clamped to `[0.0, HealthMax]` at all times. `HealthMax` does not change during MVP gameplay.

**Rule 3 — Damage Intake**
`HealthCurrent` decreases only when the Combat System calls `ApplyDamage(float Amount)`. Hard falls (≥ 150 UU, per Character Controller GDD) do not apply damage. All future damage sources (environmental hazards, status effects) must route through `ApplyDamage()` — H&S is the single write surface.

**Rule 4 — Wounded State Threshold**
When `HealthCurrent` drops to or below `WoundedThreshold = 30.0` (30% of HealthMax, provisional), H&S fires `OnWoundedStateEntered`. When `HealthCurrent` rises above the threshold, H&S fires `OnWoundedStateExited`. The Character Controller subscribes and blends the Wounded locomotion layer. The Audio System subscribes and increases breathing intensity. The wounded state is a communication layer — not a gameplay debuff, not a stat penalty.

**Rule 5 — Death Trigger**
When `HealthCurrent` reaches `0.0`, H&S fires `OnPlayerDeath` on the same tick. Death flow, in order:
1. H&S broadcasts `OnPlayerDeath`
2. Character Controller and Combat System subscribe → freeze player input on the same tick
3. Death animation plays (Character Controller Animation Blueprint)
4. Save/Load System loads the combat checkpoint slot
5. H&S sets `HealthCurrent = HealthMax` on `OnSaveLoaded` (combat checkpoint load only)

**Rule 6 — Health Restoration**
`HealthCurrent` restores to `HealthMax` (full) at campfire/rest point interactions — instantaneously on rest event completion. There is no mid-combat HP regen and no timed regen during exploration. `HealthCurrent` is written to `UAltynSaveGame` on every manual and auto-save, and restored from it on manual/auto loads. On combat-checkpoint load (death-load), `HealthCurrent` is always set to `HealthMax` regardless of the saved value.

**Rule 7 — Stamina Ceiling**
`StaminaMax = 100.0` (provisional). `StaminaCurrent` is clamped to `[0.0, StaminaMax]` at all times. `StaminaMax` does not change during MVP gameplay. `StaminaCurrent` is **not** saved to `UAltynSaveGame` — it resets to `StaminaMax` on any load.

**Rule 8 — Stamina Drain: Sprint**
When `bIsSprinting == true` (read from Character Controller each tick), `DrainStamina(SprintDrainRate × DeltaTime)` is called every tick. `SprintDrainRate = 12.5 UU/s` (provisional — derived from design target: `100.0 / 8.0 = 12.5`). Drain is linear; no acceleration within a sprint session.

**Rule 9 — Stamina Exhaustion Contract**
When `StaminaCurrent == 0.0`, H&S fires `OnStaminaExhausted`. Character Controller subscribes and immediately exits Sprint → Sprint_Recovery (1.5–2 s, CC-owned). Sprint re-entry requires `StaminaCurrent > SprintReentryThreshold = 10.0` (provisional), preventing immediate re-sprint after recovery begins. Stamina does not recover during Sprint_Recovery if combat is active (Rule 11). If not in combat, recovery begins on the next tick after Sprint_Recovery ends.

**Rule 10 — Stamina Drain: Combat Exertion** *(Provisional)*
H&S exposes `DrainStamina(float Amount)`. The Combat System calls it at action execution (not on button press). Provisional costs:

| Action | Drain Amount | Notes |
|--------|-------------|-------|
| Heavy attack | 25.0 | One-time cost on execution |
| Blocking an attack | 15.0 | Per blocked hit |
| Dodge (combat context) | 10.0 | Ownership TBD: CC vs. Combat System |

*Flag: Dodge stamina cost ownership must be resolved in the Combat System GDD. Recommendation: H&S applies a unified dodge drain regardless of combat context.*

**Rule 11 — No Stamina Recovery During Combat**
`StaminaCurrent` does not recover while `bInCombat == true`. Each fight is a stamina budget the player enters with and burns down. Recovery begins only after `bInCombat` transitions to `false`. There is no extended post-combat recovery lockout — recovery starts on the tick `bInCombat` becomes false.

**Rule 12 — Stamina Recovery After Combat**
When `bInCombat == false` AND `bIsSprinting == false`, `RecoverStamina(StaminaRecoveryRate × DeltaTime)` is called every tick. `StaminaRecoveryRate = 20.0 UU/s` (provisional — full recovery from empty takes 5.0 s). Recovery is linear with no startup delay. The perceptual weight of recovery is owned by animation (Sprint_Recovery Montage) and audio (breathing), not by a delay in H&S math.

**Rule 13 — HUD Visibility Contract**
H&S exposes `HealthCurrent`, `HealthMax`, `StaminaCurrent`, `StaminaMax` as read-only. It does not own or control HUD visibility. The Combat & Stealth HUD subscribes to `bInCombat` (Combat System) and shows/hides health and stamina displays based on that flag.

---

### States and Transitions

**Health States**

| State | Condition | Entered When | Exited When | Player Actions Blocked |
|-------|-----------|--------------|-------------|----------------------|
| `HS_Healthy` | `HealthCurrent > 30.0` | On load; damage ends above threshold; rest completes | `HealthCurrent ≤ 30.0` OR death | None |
| `HS_Wounded` | `0.0 < HealthCurrent ≤ 30.0` | `OnWoundedStateEntered` fires | HP rises above 30.0 OR death | None (visual/audio layer only) |
| `HS_Dead` | `HealthCurrent == 0.0` | `OnPlayerDeath` fires | `OnSaveLoaded` (combat checkpoint load) | All input |

**Stamina States**

| State | Condition | Entered When | Exited When | Player Actions Blocked |
|-------|-----------|--------------|-------------|----------------------|
| `SS_Ready` | `StaminaCurrent > 10.0` | On load; after recovery above threshold | Sprint or combat drain begins | None |
| `SS_Draining` | Drain called this tick | Sprint begins OR combat exertion action fires | Drain stops | None |
| `SS_Recovering` | `bInCombat == false AND bIsSprinting == false` | Sprint ends; combat ends | Drain resumes OR `SS_Ready` threshold crossed | None |
| `SS_Exhausted` | `StaminaCurrent == 0.0` | `OnStaminaExhausted` fires | `StaminaCurrent > SprintReentryThreshold` after recovery | Sprint entry |

---

### Interactions with Other Systems

| System | H&S Receives | H&S Exposes | Interface Type |
|--------|-------------|-------------|----------------|
| Character Controller | `bIsSprinting` bool (sprint drain context) | `StaminaCurrent` float (read-only, sprint gate); `OnStaminaExhausted`; `OnWoundedStateEntered/Exited` | Per-tick read; delegates |
| Combat System *(provisional)* | `ApplyDamage(float)`, `DrainStamina(float)` calls | `StaminaCurrent`, `HealthCurrent` (gate heavy attacks/blocking); `OnPlayerDeath` | Function calls; read-only floats; delegate |
| Save/Load System | `OnSaveLoaded` event + checkpoint type flag | `HealthCurrent` written to `UAltynSaveGame` on save; `HealthMax` set on combat-checkpoint load | Event subscription; USaveGame field |
| Audio System | — | `OnWoundedStateEntered/Exited`; movement state context via Character Controller `OnMovementNoisePulse` | Delegate subscription |
| Character Controller (Wounded layer) | — | `OnWoundedStateEntered/Exited` → locomotion blend switch | Delegate |
| Combat & Stealth HUD | — | `HealthCurrent`, `HealthMax`, `StaminaCurrent`, `StaminaMax` (read-only) | Read-only floats |

## Formulas

**F1 — Sprint Exhaustion Time**

`T_exhaust = StaminaMax / SprintDrainRate`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Stamina maximum | `StaminaMax` | float | 80–120 | Total stamina pool; see Tuning Knobs |
| Sprint drain rate | `SprintDrainRate` | float | 10.0–16.7 | Stamina units lost per second while sprinting |
| Exhaustion time | `T_exhaust` | float | 6.0–12.0 s | Seconds of continuous sprint before exhaustion |

**Output Range**: 6.0 s (aggressive) to 12.0 s (generous) under tuning range  
**Design target**: 8.0 s  
**Example**: `100.0 / 12.5 = 8.0 s` ✓

---

**F2 — Stamina Recovery Time (from empty to full)**

`T_recover_full = StaminaMax / StaminaRecoveryRate`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Stamina maximum | `StaminaMax` | float | 80–120 | Total stamina pool |
| Recovery rate | `StaminaRecoveryRate` | float | 15.0–30.0 | Stamina units recovered per second (out of combat only) |
| Recovery time | `T_recover_full` | float | 2.7–8.0 s | Seconds to recover from 0 to StaminaMax |

**Output Range**: 2.7 s (fast) to 8.0 s (slow) under tuning range  
**Design target**: 5.0 s — recovery is shorter than a full exhaustion sprint; the body breathes faster than it runs  
**Example**: `100.0 / 20.0 = 5.0 s` ✓

---

**F3 — Sprint Re-entry Time (from exhaustion)**

`T_reentry = SprintReentryThreshold / StaminaRecoveryRate`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Re-entry threshold | `SprintReentryThreshold` | float | 5.0–20.0 | Minimum StaminaCurrent before sprint input is accepted |
| Recovery rate | `StaminaRecoveryRate` | float | 15.0–30.0 | Stamina units per second (same as F2) |
| Re-entry time | `T_reentry` | float | 0.25–1.33 s | Time from exhaustion until sprint is physically possible again |

**Output Range**: 0.25 s (permissive) to 1.33 s (strict)  
**Design target**: 0.5 s — sufficient to register the pause without feeling punishing  
**Example**: `10.0 / 20.0 = 0.5 s` — Character Controller Sprint_Recovery animation (1.5–2 s) gates sprint for longer than this in practice; threshold primarily prevents snap-re-entry outside Sprint_Recovery context.

---

**F4 — Wounded Threshold**

`WoundedThreshold = HealthMax × WoundedFraction`

| Variable | Symbol | Type | Range | Description |
|----------|--------|------|-------|-------------|
| Health maximum | `HealthMax` | float | 80–120 | Total health pool |
| Wounded fraction | `WoundedFraction` | float | 0.20–0.40 | Fraction of HealthMax below which Wounded state activates |
| Threshold value | `WoundedThreshold` | float | 16–48 | Absolute HP value triggering Wounded animation/audio layer |

**Output Range**: 16 HP (tight) to 48 HP (generous) under tuning range  
**Design target**: 30.0 HP (30% of 100)  
**Example**: `100.0 × 0.30 = 30.0` — roughly 3 average hits of buffer after wounded state activates before death, depending on enemy damage values (provisional, Combat System GDD)

---

**F5 — Stamina Combat Budget** *(Provisional — depends on Combat System GDD)*

Executions from a full stamina pool at provisional costs from Rule 10:

| Action | Cost | Executions from full stamina |
|--------|------|------------------------------|
| Heavy attack | 25.0 | 4 |
| Blocking a hit | 15.0 | 6 |
| Dodge | 10.0 | 10 |
| Mixed (2 heavy + 2 block + 1 dodge) | 80.0 | budget fully spent |

**Output Range**: 0–10 actions per encounter before depletion, depending on action mix  
**Sanity check**: A mid-length encounter at provisional values exhausts most of the stamina budget. If too many actions are possible, increase costs proportionally; if too few, decrease. Re-tune when the Combat System GDD is authored.

## Edge Cases

**E1 — Sprint Exhausts Mid-Dodge**
Dodge is active (`bIsDodging == true`) when `StaminaCurrent` reaches 0.0. `OnStaminaExhausted` fires immediately, but the Character Controller does not interrupt an active dodge Montage. Dodge completes its full 0.35 s travel. Sprint_Recovery begins after the Montage ends. Stamina recovery (if not in combat) begins the tick after Sprint_Recovery completes.

**E2 — HealthCurrent Crosses WoundedThreshold and 0 in the Same Tick**
Two damage events in the same tick (or one hit dropping HP from 50 to 0) cause `HealthCurrent` to pass 30.0 on its way to 0.0. Both `OnWoundedStateEntered` and `OnPlayerDeath` fire in the same tick. `OnPlayerDeath` takes precedence: death flow begins immediately. The Wounded locomotion layer and audio state are never applied. Order: Wounded delegate first, Death delegate second; downstream subscribers must handle immediate Death cancellation without side effects.

**E3 — `bInCombat` Becomes True During Sprint_Recovery**
Enemy enters alert state and sets `bInCombat = true` while the Character Controller is in Sprint_Recovery. Stamina recovery — if it had begun — is immediately suppressed. Sprint_Recovery animation continues normally (Character Controller-owned; H&S does not interrupt it). Player enters combat with whatever `StaminaCurrent` existed at the start of Sprint_Recovery. Sprint remains unavailable until `StaminaCurrent > SprintReentryThreshold` AND `bInCombat` returns to false.

**E4 — Save While in Wounded State**
Player manually saves with `HealthCurrent = 12.0` (Wounded state active). H&S writes `12.0` to `UAltynSaveGame`. On next manual load, player returns at `HealthCurrent = 12.0`; Wounded state re-activates immediately via `OnWoundedStateEntered`. No health is granted for the act of saving or loading. Campfire rest is the only restoration path.

**E5 — Campfire Access During Active Combat**
Player reaches a campfire interaction point while `bInCombat == true`. H&S does not gate campfire access — it has no `bInCombat` flag of its own. Blocking campfire during combat is the responsibility of the Combat System or level design gate logic. H&S applies `RestoreHealth(HealthMax)` unconditionally if the rest event fires. Design intent: campfires must not be reachable mid-combat through level geometry.

**E6 — `StaminaCurrent` Exactly Equals `SprintReentryThreshold`**
`StaminaCurrent == 10.0` exactly. Sprint entry check is `StaminaCurrent > SprintReentryThreshold` (strictly greater than). Sprint is not allowed at exactly 10.0. Player must reach 10.01 or above. This closes the boundary-condition exploit of snapping back to sprint at the threshold floor.

**E7 — Dodge Drain Applied at `StaminaCurrent = 0`**
`DrainStamina(10.0)` (dodge cost) fires while `StaminaCurrent` is already 0.0. Clamp holds at 0.0 — stamina cannot go negative. `OnStaminaExhausted` does not re-fire (system is already exhausted). No double penalty, no duplicate delegate.

**E8 — Multiple `ApplyDamage` Calls in One Tick**
Two enemies hit simultaneously, each calling `ApplyDamage()` in the same tick. Both are processed sequentially within the tick. `HealthCurrent` is clamped after each call. If the first hit drops HP to 0, `OnPlayerDeath` fires immediately. The second `ApplyDamage` call is received but `HealthCurrent` is already 0 and clamped; no second death event fires.

## Dependencies

**Upstream (H&S depends on these):**

| System | Dependency | Interface |
|--------|-----------|-----------|
| Character Controller | `bIsSprinting` bool (sprint drain gate) | Per-tick read |
| Character Controller | `bIsDodging` bool (dodge drain trigger — provisional, see Rule 10 flag) | On dodge fire |
| Save/Load System | `OnSaveLoaded` event + checkpoint type flag (combat vs. manual) | Event subscription |
| Combat System *(provisional)* | `ApplyDamage(float)` calls (damage intake); `DrainStamina(float)` calls (combat exertion costs) | Function calls into H&S |

**Downstream (depend on H&S):**

| System | What They Need | Interface |
|--------|---------------|-----------|
| Character Controller | `StaminaCurrent` (sprint gate per tick); `OnStaminaExhausted` (exit sprint); `OnWoundedStateEntered/Exited` (locomotion blend) | Read-only float; delegates |
| Combat System *(provisional)* | `HealthCurrent`, `StaminaCurrent` (gate heavy attacks, blocking); `OnPlayerDeath` (freeze input, trigger death flow) | Read-only floats; delegate |
| Audio System | `OnWoundedStateEntered/Exited` (breathing intensity change) | Delegate subscription |
| Combat & Stealth HUD | `HealthCurrent`, `HealthMax`, `StaminaCurrent`, `StaminaMax` (display during combat) | Read-only floats |
| Save/Load System | `HealthCurrent` written to `UAltynSaveGame` on save | USaveGame field write |

**Bidirectionality note**: Character Controller and Combat System both appear as upstream AND downstream. The Character Controller GDD must list H&S as a dependent. The Save/Load GDD must add `HealthCurrent` to `UAltynSaveGame` schema (not currently present — flag for save-load-system.md update).

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|--------------------|
| `HealthMax` | 100.0 | 80–150 | Cheap one-shots; death feels unfair | Encounters trivial; no tension |
| `WoundedFraction` | 0.30 | 0.20–0.45 | Wounded state entered too late; no signal before death | Constantly in Wounded state; desensitizes player |
| `SprintDrainRate` | 12.5 | 10.0–16.7 | Sprint becomes a traversal tool; combat approach trivial | Sprint unusable for combat approach |
| `StaminaMax` | 100.0 | 80–120 | Scales F1 and F2 proportionally (see Formulas) | — |
| `StaminaRecoveryRate` | 20.0 | 15.0–30.0 | Recovery between fights too long; pacing suffers | Camping trivially restores stamina in 3 s |
| `SprintReentryThreshold` | 10.0 | 5.0–20.0 | Player snaps back into sprint immediately after exhaustion | Long forced walk window before re-sprint allowed |
| Heavy attack drain | 25.0 | 15.0–35.0 | Heavy attacks spammable; stamina feels meaningless | Single heavy exhausts stamina budget |
| Block drain | 15.0 | 8.0–25.0 | Blocking is free; no pressure in defensive fights | One blocked hit exhausts stamina; discourages blocking |
| Dodge drain | 10.0 | 5.0–20.0 | Dodge-spam trivializes combat; CC cooldown is the only gate | Single dodge too costly; dodge unused |

**Note on interdependence**: `SprintDrainRate`, `StaminaMax`, and `StaminaRecoveryRate` form a coupled system — changing one changes the feel of all three formulas (F1–F3). Always tune as a group, not independently.

## Visual/Audio Requirements

### Governing Principle

The body is the only UI. Every specification in this section exists to answer one question: how does the player know what state the boy is in, without numbers, bars, or icons? The answer is always the same — the same way a person knows. You watch how someone breathes. You hear how they land. You see whether they move like they are managing something they will not name. The boy communicates body state the way someone who has been watched by the steppe his whole life does: with the minimum necessary signal, clearly, without performance.

The design error to avoid is melodrama. A vignette that pulses like a heartbeat, a red flash on hit, a screen shake that says "you are being punished" — these are the visual language of arcade games. Altyn Adam's visual language is closer to documentary. You do not see a red vignette. You see a fourteen-year-old boy carrying a wound, and you read it in his shoulder.

---

### 1. Wounded Locomotion Layer (HS_Wounded: HealthCurrent ≤ 30.0)

The Wounded locomotion layer is an additive animation layer blended over the standard locomotion state machine. It activates when `OnWoundedStateEntered` fires and deactivates when `OnWoundedStateExited` fires. It does not replace any locomotion state — it sits on top of all of them (Walk, Sprint, Crouch Walk, Idle variants).

**Blend-in**: 20–30 frames from `OnWoundedStateEntered`. Not instantaneous — the wound has weight that arrives.  
**Blend-out**: 40–60 frames from `OnWoundedStateExited`. Longer out than in — the body does not immediately forget.

**Posture changes (all must be visible at standard third-person camera distance ~3.5 m):**

- **Right shoulder drops 8–12°** (right side holds the wound). This is the primary wound signal — not a clutch, not a grab, but a shoulder the boy is no longer holding up the same way.
- **Right arm swing amplitude reduced 60%**. Left arm swings normally; right arm barely moves in Walk. In Sprint, right arm drive is muted — left arm compensates slightly. Asymmetry reads immediately at distance.
- **Head carriage drops 5–8°** — chin down, not dramatically. The posture of someone looking at the ground 2 m ahead instead of the horizon. Reads as fatigue rather than injury.
- **Stride asymmetry**: right step 8–10% shorter than left; footfall slightly heavier (less controlled push-off). Visible in Walk cycle; less legible in Sprint.
- **Breathing rhythm visible at Idle**: a shallow secondary chest movement — 0.6 s microcycle layered on base Idle. Authored as a procedural additive or looping blend tree layer, not a one-shot Montage. The chest does not heave; it catches.
- **Right hand at rest**: in Calm Idle, right hand drifts 10–15° toward torso — not touching, not clutching, but orbiting the wound. One hand loose, one hand slightly inward. If it reads as dramatic, pull it back.

**What the Wounded layer must NOT do:**
- Add a stumble or limp that breaks the locomotion blend space. The boy is wounded, not disabled — he can still run, still dodge.
- Add blood, dust, or debris texture. Those are VFX decisions; this layer is pure skeletal animation.
- Override Tense or Aware idle adjustments. Wounded blends additively over them; neither cancels the other.

**Sprint_Recovery in Wounded state**: an additional shallow catch-breath is layered at the end of Sprint_Recovery — approximately 6–8 frames of the chest rising and not fully settling (0.2–0.3 s). Must not delay Sprint_Recovery's return to Walk.

---

### 2. Near-Death Signal (HealthCurrent 5–10 HP)

There is no visible HP bar. The near-death signal must be unambiguous but not melodramatic.

**The signal is a sound, not a screen effect.**

When `HealthCurrent` drops below `NearDeathThreshold = 10.0` HP (provisional), a single audio event plays: a short, involuntary exhale with a slight catch — not a gasp, not a groan, but the specific sound of someone's breath breaking at the moment before they understand the scope of the damage. Duration: 0.6–0.8 s. Plays once on crossing the threshold, not on subsequent hits. Asset: `sfx_player_neardeath_catch_01.ogg`.

**Secondary signal — camera behavior only:**
A single, slow camera drift: over 1.5 s, the camera orbit point lowers 4–6 UU and tilts 1–2° toward the ground (Dutch tilt, barely perceptible). This communicates that the boy's center of gravity has shifted. Does not pulse, does not repeat. Does not reverse until `OnWoundedStateExited` fires (health restored above 30 HP).

**Explicitly rejected:**  
No vignette. No desaturation. No red overlay. No heartbeat sound. None of these.

*Cross-system flag*: the near-death camera drift must be specified in the Camera System GDD as a health-driven camera behavior.

---

### 3. Audio Direction Across Health States

#### Breathing

| State | Breathing audio | Asset |
|-------|----------------|-------|
| HS_Healthy, exploration | Inaudible — ambient and footsteps cover it | None played |
| Sprint_Recovery (any health) | Full breath cycle, 1–2 breaths, fades at Walk resumption | Specified in Character Controller GDD |
| HS_Wounded, Idle | Shallow, slightly irregular. One breath cycle every 8–10 s at ~25% of Sprint_Recovery volume | `sfx_player_breath_wound_idle_0[1-3].ogg` |
| HS_Wounded, Sprint_Recovery | Full recovery breath + wounded catch layered on top | `sfx_player_breath_wound_catch_01.ogg` |
| HS_Wounded, post-hit | Short suppressed exhale — the boy is managing. 0.3–0.4 s | `sfx_player_hit_wound_exhale_0[1-2].ogg` |
| Near-death threshold crossed | Involuntary catch-exhale — single trigger only | `sfx_player_neardeath_catch_01.ogg` |

**No heartbeat audio.** The heartbeat-under-low-health mechanic is correct for horror and action-thriller. It is wrong here. The boy grew up in the steppe; his own heartbeat in silence is not a signal, it is background. A heartbeat sound reads as melodrama. Use breathing depth and rhythm instead.

**No additional ambient narrowing in health state.** Mix C (Combat Alertness) already ducks ambient when combat begins. H&S audio changes must not add a second ambient duck. The world stays the world; the boy's body changes within it.

#### Footsteps in Wounded State

Right-side footstep assets in Walk receive a −2 dB level reduction and a 5–8% pitch-down shift — the audible correlate of stride asymmetry. Applied as a MetaSound parameter (`bPlayerWounded` bool driving a right-step gain modifier), not a separate asset set.

---

### 4. Screen and Camera Effects

**Appropriate:**
- Near-death camera Dutch tilt (Section 2): 1–2°, one-shot, non-pulsing.
- Hit camera micro-impulse (Section 5): single directional displacement, 2–4 UU.

**Not appropriate:**
- **Red vignette**: visual language of 2005–2015 console action games. Wrong tone.
- **Desaturation toward grayscale**: Altyn Adam's Saka gold / steppe blue-green-ochre palette is load-bearing. Washing it out damages the visual language when the player most needs to read the world clearly.
- **Screen blur or depth-of-field shift**: implies vision impairment. The boy does not lose vision when wounded; he sees the world exactly as it is.
- **Pulsing effects of any kind**: pulsing is the heartbeat metaphor in visual form. Rejected for the same reason.
- **Chromatic aberration on hit**: reads as a technical effect, not a body experience.

**The governing rule**: ask whether the boy would experience this visually. A wounded fourteen-year-old does not see the world in red. He sees it exactly as it is — and that clarity is part of the game's contract with the player.

---

### 5. Hit Feedback (any ApplyDamage call)

**5a. Camera micro-impulse:**
A single directional displacement — 2–4 UU in the direction opposite the hit vector — then returns over 0.15–0.2 s with a critically-damped spring (no oscillation). If hit direction is unknown, displace downward. Communicates direction and weight, not chaos.

**5b. Wound audio response by health state:**

| Health state | Hit audio | Asset |
|---|---|---|
| HS_Healthy (> 30 HP) | Short suppressed grunt — breath pushed out | `sfx_player_hit_grunt_0[1-3].ogg` |
| HS_Wounded (≤ 30 HP) | Slightly longer controlled exhale — adding to a weight | `sfx_player_hit_wound_exhale_0[1-2].ogg` |
| Near-death (≤ 10 HP) | First crossing: catch-exhale (Section 2). Subsequent hits: HS_Wounded exhale — do not retrigger the catch | `sfx_player_neardeath_catch_01.ogg` / `sfx_player_hit_wound_exhale_0[1-2].ogg` |

**5c. Impact animation response (Character Controller / Animation Blueprint):**
- Light hit (< 25 damage): single-frame head/shoulder recoil from impact direction, recovers in 4–6 frames.
- Heavy hit (≥ 25 damage): torso-level disruption, 8–10 frames of upper body displacement, 4-frame recovery. In Wounded state, heavy hit recovery extends by 2–3 frames via Wounded layer blend.

**Not included:** No red flash on screen or character model. No controller rumble spec here (Input System GDD). No blood particles at MVP (VFX spec in Polish phase).

---

### 6. Campfire / Rest Restoration

**Phase 1 — Settling (0–2 s):**
Idle transitions to campfire sitting via 15–20 frame blend. Weight sinks, one hand to knee, the other arm settles. He sits down the way someone exhausted sits — without choosing how. Audio: MetaSound Calm state continues uninterrupted. No sting. Campfire ambient loop already present as 3D spatial source. Cloth/equipment settle sounds from animation.

**Phase 2 — Rest (2 s through animation completion):**
Wounded locomotion layer blends out over 2 s (longer than standard 40–60 frame blend-out). Posture straightens fractionally, shoulder rises, chest settles to slower rhythm — the visual read of health restoring.

The rest breath: `sfx_player_breath_rest_01.ogg` — one slow, full breath cycle with a complete exhale that releases entirely. Duration: 2.5–3.0 s. Volume: 60–70% of Sprint_Recovery level. Plays once, clearly. It is the sound of putting something down.

`RestoreHealth(HealthMax)` fires at the frame the rest animation completes — not during, not after a delay.

*Cross-system flag*: the post-combat dombra phrase (`mus_resolve_postcombat_exhale_01`, Audio System) must be blocked if campfire rest begins within 8 s of combat ending. Overlap ruins the rest breath. Note in Audio System GDD.

**Phase 3 — Return:**
Boy stands at Walk_Idle (Calm). Stand animation 10–15 frames, interruptible at frame 8. Input returns at frame 8. No visual flash, no screen wipe, no "Rested" UI text.

**Campfire lighting note**: campfire rest positions must always be lit from the fire source (orange-amber from below), not from generic ambient sky. Communicate to environment artist.

---

### 7. Stamina Recovery Visual/Audio (SS_Recovering, post-combat)

**Primary signal: environmental audio restoration.** Mix C (Combat Alertness) releasing ambient to full volume over 1.2 s after combat ends is the perceptual signal that the fight is over and the body is returning. Stamina recovery rides this ambient restoration — no dedicated stamina-recovery audio.

**No dedicated stamina-recovery animation.** The SS_Recovering state has no locomotion modifier.

**Sprint re-entry as confirmation.** Smooth transition into the Explosive sprint phase after combat tells the player stamina is restored — no bar needed.

**Sprint rejection sound**: if sprint is attempted before `StaminaCurrent > SprintReentryThreshold`, a short breath-catch plays — `sfx_player_sprint_reject_01.ogg` (0.2–0.3 s, −28 dBFS — the quietest asset in the set, almost private). The body says *not yet*. No visual feedback.

---

### Asset Specifications

**Audio assets:**
- `sfx_player_breath_wound_idle_01–03.ogg` — 3 variants, 1.5–2.5 s, −24 dBFS peak
- `sfx_player_breath_wound_catch_01.ogg` — 0.4–0.6 s, −20 dBFS
- `sfx_player_hit_grunt_01–03.ogg` — 3 variants, 0.2–0.4 s, −18 dBFS
- `sfx_player_hit_wound_exhale_01–02.ogg` — 2 variants, 0.3–0.5 s, −20 dBFS
- `sfx_player_neardeath_catch_01.ogg` — 0.6–0.8 s, −16 dBFS (slightly prominent)
- `sfx_player_breath_rest_01.ogg` — 2.5–3.0 s, −22 dBFS
- `sfx_player_sprint_reject_01.ogg` — 0.2–0.3 s, −28 dBFS

All: mono, 44.1 kHz / 24-bit WAV delivery → converted to .ogg by pipeline. No reverb baked in.

**Animation assets:**
- Wounded locomotion additive layer (all locomotion states, 30 fps, authored in Animation Blueprint)
- `char_boy_hitreact_light_01` (additive Montage, 4–6 frames)
- `char_boy_hitreact_heavy_01` (additive Montage, 10–14 frames)
- `char_boy_campfire_settle_01` (root motion, 15–20 frame blend-in)
- `char_boy_campfire_sit_idle_01` (looping, min 4 s)
- `char_boy_campfire_stand_01` (15 frames, interruptible at frame 8)

**MetaSound parameters added to `MUS_Adaptive_Score_Main`:**
- `bPlayerWounded` (bool): drives right-footstep gain modifier (−2 dB, −6% pitch). Subscribes to `OnWoundedStateEntered/Exited`.
- `fNearDeathThreshold` (float, 10.0): data-only knob, no code change needed.

---

### Tuning Knobs (Visual/Audio)

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|--------------------|
| `WoundedLayerBlendIn` | 25 frames | 15–40 | Wound snaps in; reads as game state | Wound arrives too slowly to notice |
| `WoundedLayerBlendOut` | 50 frames | 30–80 | Body forgets wound immediately on heal | Wounded posture lingers after campfire |
| `ShoulderDropDegrees` | 10° | 6–14° | Too subtle; reads only as fatigue | Reads as dramatic injury; breaks realism |
| `RightArmSwingReduction` | 60% | 40–75% | Asymmetry imperceptible at camera distance | One arm looks broken |
| `NearDeathThreshold` | 10.0 HP | 5.0–15.0 | Signal fires too late; no time to respond | Fires frequently; desensitizes |
| `CameraImpulseMagnitude` | 3.0 UU | 1.5–5.0 | Hit feels weightless | Reads as screen shake; breaks tone |
| `CameraImpulseReturn` | 0.18 s | 0.12–0.25 | Camera bounces; oscillation visible | Displacement lingers; smeared |
| `NearDeathCameraTilt` | 1.5° | 0.5–3.0° | Imperceptible | Reads as cinematic affectation |
| `WoundedBreathInterval` | 9.0 s | 6.0–15.0 | Too frequent; sounds like panic | Too sparse; Wounded state feels normal |
| `RestBreathVolume` | 0.65 | 0.5–0.8 | Inaudible against campfire crackle | Dominant; draws attention to itself |

## UI Requirements

No UI elements are owned by the Health & Stamina System. `HealthCurrent`, `HealthMax`, `StaminaCurrent`, and `StaminaMax` are exposed as read-only floats; all display logic belongs to the Combat & Stealth HUD system. During exploration, no health or stamina indicator appears. During active combat (`bInCombat == true`), the Combat & Stealth HUD subscribes to those floats and renders the health bar and stamina bar. This system exposes data; it does not render anything.

## Acceptance Criteria

| # | Given | When | Then | Pass Condition |
|---|-------|------|------|----------------|
| AC-1 | Character sprinting at full stamina | Sprint held continuously | Sprint_Recovery begins | Approximately 8 s of continuous sprint before forced exit |
| AC-2 | StaminaCurrent = 0 after exhaustion | Player immediately presses sprint | Sprint does not re-engage | Sprint blocked until StaminaCurrent > 10.0 |
| AC-3 | HealthCurrent at 50 HP | Single hit drops HP to 25 | Wounded locomotion layer activates; breathing audio intensifies | Wounded state active at ≤ 30 HP; no debuff on player actions |
| AC-4 | HealthCurrent at 15 HP | Single hit drops HP to 0 | OnPlayerDeath fires once; death animation plays | Exactly one death event; no duplicate delegate calls |
| AC-5 | Player injured (HealthCurrent = 40) | Campfire rest completes | HealthCurrent = 100.0 | Full restore; no partial regen |
| AC-6 | Player at 22 HP, manually saves, reloads | Load the manual save | HealthCurrent = 22.0 | Wounded state re-activates on load; no free health |
| AC-7 | Player sprints to exhaustion, then saves and reloads | Load any save type | StaminaCurrent = StaminaMax (100.0) | Stamina always resets on any load |
| AC-8 | Player in combat (bInCombat = true), StaminaCurrent = 50 | Player stands still 5 s | StaminaCurrent = 50 (unchanged) | Zero recovery during combat |
| AC-9 | Player exits combat (bInCombat = false), StaminaCurrent = 0 | Wait 5 s (not sprinting) | StaminaCurrent ≈ 100.0 | Full recovery in approximately 5 s post-combat |
| AC-10 | HealthCurrent = 50 | Single hit deals 60 damage (crosses 30 and 0 in one tick) | OnWoundedStateEntered then OnPlayerDeath fire in same tick | No Wounded animation visible; death takes precedence; no flicker |
| AC-11 | HealthCurrent = 5 | Two enemies hit simultaneously, each dealing 10 damage | OnPlayerDeath fires once | No double death event; second hit clamped at 0 |
| AC-12 | Player falls 2 m (≥ 150 UU) | Hard landing occurs | Stumble Montage plays; HealthCurrent unchanged | Zero HP change from hard fall confirmed via debug log |

## Open Questions

**OQ-1 — Dodge stamina cost ownership** *(Blocking for Combat System GDD)*
Rule 10 flags that dodge stamina cost ownership is unresolved: does H&S drain on `bIsDodging == true` (Character Controller event), or does the Combat System call `DrainStamina(10.0)` as a combat action? Recommendation: H&S applies a unified dodge drain regardless of combat context. Confirm in Combat System GDD.

**OQ-2 — All provisional combat stamina costs**
Heavy attack (25.0), block (15.0), and dodge (10.0) are provisional estimates. Re-visit F5 and Rule 10 with real design values when Combat System GDD is authored.

**OQ-3 — Mounted travel and stamina**
Horse System GDD not yet designed. Current spec: H&S tracks only the boy's stamina. If mounted travel should drain the boy's stamina (prolonged galloping is exhausting), the Horse System GDD must call `DrainStamina()` into H&S. Otherwise the boy's stamina is fully preserved during mounted travel.

**OQ-4 — Environmental damage types**
MVP scope excludes drowning, poison, fire, and status effects. If added, they must route through `ApplyDamage()`. Consider whether the signature needs a `DamageType` enum parameter (e.g., to drive different audio/animation responses for poison vs. slash). Flag when Combat System and Enemy AI GDDs are designed.

**OQ-5 — Campfire access gate during combat**
Rule E5 places campfire gating on the Combat System / level design. The exact implementation (collision gate on trigger zone, interactable disabled while `bInCombat`, or level geometry separation) should be specified in the Combat System GDD or per-level encounter design guidelines.
