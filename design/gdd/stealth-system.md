# Stealth System

> **Status**: Complete
> **Author**: Solo Dev + Agents
> **Last Updated**: 2026-05-10
> **Implements Pillar**: Путешествие по кланам (Clan Journey) — navigating hostile territory through skill, not power

## Overview

The Stealth System governs how enemies perceive the player and how the player manages that perception. It owns the detection model — per-enemy vision cones, hearing radii, and alert state transitions (Unaware → Suspicious → Alerted → Searching) — and exposes the mechanics through which the player shapes their detectability: crouching into cover, managing movement speed, using terrain to muffle noise, and creating distractions. The system subscribes to the Character Controller's `OnMovementNoisePulse(float NoiseLevel)` delegate and applies terrain multipliers to produce an effective noise level that enemies evaluate against their hearing threshold at range. Each enemy runs an independent perception check; detection is not global. The Stealth System aggregates individual enemy alert states into a highest-threat awareness level that drives the Character Controller's idle animation variant (Calm / Aware / Tense) and provides the Enemy AI System with the current alert tier per enemy. For the player, stealth is not a mode to activate — it is a permanent negotiation between speed, shadow, and silence. On the steppe, the question is never whether the grass hides you. The question is how long.

## Player Fantasy

The player feels that the steppe is on their side — not as magic, not as gift, but as *knowledge*. The boy knows which grasses muffle footsteps and which stone corridors echo. He knows where deer beds are and where the wind reverses at dusk. The Persians are loud, shod, armored, and foreign: they move through the land the way a river moves through a hand — over it, not with it. The boy moves through it the way water finds a channel that was already there.

Stealth in *Altyn Adam* is the inheritance grief could not strip. Everything else — his name, his people, his place in the world — is gone. What remains is what his uncles and the land taught him without words: where to be, how to be still, when to wait. Every successful traversal through an enemy patrol is not triumph. It is something smaller and more true: *They did not see me. The steppe kept its secret.*

**This is not a stealth-assassin power fantasy.** The boy does not eliminate guards from shadows. He does not feel invincible in cover. If a player feels *cool* while sneaking, the system is failing — they should feel *small, alert, and alive*. The ambient magic of the steppe (do the spirits hide him? or only the wind?) must always remain unresolved. The land is an ambivalent collaborator, not a tool.

As the boy gains allies and identity across the chapters, stealth may feel less like survival and more like fluency. By the Culminating Battle, he stops hiding. He stands in the open. That arc must be readable in the movement system before a single line of dialogue explains it.

## Detailed Design

### Core Rules

1. **Stealth is not a mode.** The system evaluates detection continuously for every enemy within a configured activation range. There is no "stealth button" — the player is always either perceived or not perceived, and the gradient between those states is the system.

2. **Detection is per-enemy and independent.** No enemy's alert state directly alters another's. Group coordination (alarm shouts, responding to an alerted companion) is the Enemy AI System's responsibility. The Stealth System is the substrate; Enemy AI is the behavior layer.

3. **The Stealth System subscribes to `AAltynCharacter::OnMovementNoisePulse(float NoiseLevel)`.** On each pulse, the system reads the PhysicalMaterial beneath the player's capsule, looks up the terrain multiplier, and fires `UAIPerceptionSystem::OnEvent()` with a `FAINoiseEvent(Loudness = EffectiveNoise, Location = PlayerLocation, Tag = "Footstep")`.

4. **Each enemy runs an independent detection gauge (0.0–1.0).** The gauge fills from combined vision + noise stimuli each evaluation tick (0.1 s fixed interval). It drains passively when stimuli cease. Gauge ≥ SuspicionThreshold (0.25) → Suspicious; gauge ≥ AlertThreshold (0.85) with active LOS → Alert.

5. **Vision is two-zone, no detection from behind.** Focused zone: ±30° forward half-angle, 1200 UU range, full fill rate. Peripheral zone: ±60° forward half-angle, 500 UU range, half fill rate. Beyond 60° off the enemy's facing direction: no vision stimulus regardless of distance.

6. **Line-of-sight uses the AI Perception System's built-in ECC_Visibility trace.** Solid static meshes blocking ECC_Visibility (stone walls, low cover, building interiors) automatically occlude sight. Foliage and tall grass do NOT provide LOS occlusion — they have no ECC_Visibility blocking collision. Tall grass reduces detection exclusively through the noise terrain multiplier.

7. **Crouching reduces vision stimulus, not cone geometry.** When `bIsCrouching == true`, the player's vision stimulus strength is multiplied by `CrouchVisibilityMultiplier` (0.6, a 40% reduction). The enemy's cone angle is unchanged. Standing behind solid cover while crouching provides both LOS occlusion and the visibility reduction — the boy knows to get low.

8. **Noise stimulus uses linear distance attenuation within each enemy's `HearingRadius`.** Each enemy has an independent `HearingThreshold` (floor). Noise stimuli below this floor are ignored regardless of distance — this allows per-enemy sensitivity variation (scouts hear more than standard guards).

9. **Terrain multipliers apply at the PhysicalMaterial level.** The same PhysicalMaterial that the Audio System reads for footstep SFX selection determines the stealth terrain multiplier. Both systems read from one surface definition — no duplicate surface authoring.

10. **Stone throw is the sole MVP distraction mechanic.** The player holds `IA_Throw` to aim (parabolic arc preview), releases to throw. On impact, the stone fires `FAINoiseEvent(Loudness = 0.75, Tag = "Distraction")` at the impact location. All Unaware and Suspicious enemies within `HearingRadius` of the impact investigate. Alert enemies ignore distractions.

11. **The Stealth System aggregates all active enemy alert tiers into a single highest-threat awareness level** via `UAltynStealthSubsystem : UWorldSubsystem`. This aggregate drives the Character Controller's idle animation variant and the HUD detection display. Enemies register and deregister with the subsystem on `AAltynEnemyAIController::BeginPlay/EndPlay`.

12. **Enemy vision range is modulated by ambient light.** `AdjustedFocusedRange = FocusedRange × LightModifier`. Until a Time-of-Day System exists, `LightModifier = 1.0` (permanent full daylight). The parameter is available as a tuning knob for future integration.

---

### States and Transitions

| State | Entry Condition | Exit Conditions | Enemy Behavior | CC Idle Variant |
|-------|----------------|-----------------|----------------|-----------------|
| **Unaware** | Default; `SuspicionDecayTime` elapsed from Suspicious / `SearchDuration` elapsed from Searching with no re-stimulus | → Suspicious: gauge ≥ 0.25 | Patrol route; ambient scan animation | Calm |
| **Suspicious** | Gauge ≥ 0.25 | → Unaware: gauge drains to 0, `SuspicionDecayTime` (8 s) expires; → Alert: gauge ≥ 0.85 with active LOS; → Searching: gauge ≥ 0.85 but LOS already broken | Pauses patrol; turns toward stimulus source; advances slowly to last-known position; suspicion vocalisation | Aware |
| **Alert** | Gauge ≥ 0.85 with active LOS; OR single noise hit `EffectiveNoise ≥ 0.90` within 300 UU (point-blank sprint/land) | → Searching: active LOS lost for > `AlertLOSBreakTime` (3 s); never → Unaware directly | Alarm call; charges player's last known position at full speed; attacks on sight | Tense |
| **Searching** | Alert enemy loses LOS for > 3 s; OR gauge reaches 0.85 but LOS was already broken at threshold crossing | → Alert: regains LOS or noise stimulus (distance-attenuated strength > 0.60) while searching; → Unaware: `SearchDuration` (30 s) expires with no re-stimulus | Investigates last known position; follows distraction impact points; heightened perception active (FillRate ×1.5, HearingThreshold ×0.7) | Aware |

**Deescalation rule**: Alert → Unaware requires two steps: Alert → Searching first (break LOS for 3 s), then Searching → Unaware (30 s with no re-stimulus). Direct Alert → Unaware is not possible. The near-miss must be earned.

---

### Interactions with Other Systems

**Character Controller → Stealth System**
- Stealth subscribes to `OnMovementNoisePulse(float NoiseLevel)`. On each pulse: reads PhysicalMaterial, applies terrain multiplier, fires `FAINoiseEvent`.
- Stealth reads `bIsCrouching` each evaluation tick for `CrouchVisibilityMultiplier` application.

**Stealth System → Character Controller**
- `UAltynStealthSubsystem` broadcasts `OnAwarenessChanged(EAltynAlertTier)`. Character Controller subscribes in `BeginPlay`. Mapping: Unaware / Suspicious → Calm idle; Searching → Aware idle; Alert → Tense idle. (Implements CC GDD Rule 9.)

**Input System → Stealth System**
- `IA_Crouch` (existing, IMC_OnFoot) enables crouch; CC exposes via `bIsCrouching`.
- `IA_Throw` (new — must be added to Input System GDD, IMC_OnFoot) triggers the stone throw arc-aim and release. Binding: Gamepad — hold R1 + aim stick, release to throw; Keyboard — hold G, release.

**Stealth System → Enemy AI System**
- Alert state per enemy is available via `AAltynEnemyAIController::GetAlertState()` and written to Blackboard key `AlertTier` on each transition. Enemy AI reads this for BehaviorTree branching. Group coordination is Enemy AI's responsibility — Stealth does not broadcast to other enemies.
- Stone throw impact location is written to Blackboard key `PointOfInterest` (FVector) when a distraction event reaches an enemy's perception component.

**Stealth System → Audio System**
- `UAltynStealthSubsystem` fires `OnAwarenessChanged(EAltynAlertTier)` — the Audio System's `UAudioStateManager` subscribes and maps tiers to music states: Unaware → Calm layer; Suspicious → Suspicion layer; Alert or Searching → Combat layer. (Cross-reference: Audio System GDD Section C, music state transitions.)

**Stealth System → Combat & Stealth HUD**
- Global awareness aggregate (highest tier) from `UAltynStealthSubsystem` drives the HUD detection meter. No additional interface from this system is needed at MVP.

**Time-of-Day System → Stealth System (future)**
- When implemented, the Time-of-Day System sets `UAltynStealthSubsystem::SetLightModifier(float)`. Until then, the subsystem holds `LightModifier = 1.0`.

## Formulas

All formulas reference values defined in Section C. Noise and vision stimuli are handled as separate input types — noise is event-driven (fires on movement pulse), vision is continuous (evaluated every 0.1 s tick).

---

**F1 — Effective Noise**

Terrain-adjusted noise level after PhysicalMaterial lookup.

```
EffectiveNoise = clamp(NoiseLevel × TerrainMultiplier, 0.0, 1.0)
```

| Variable | Type | Range | Source |
|----------|------|-------|--------|
| NoiseLevel | float | 0.0–1.0 | `OnMovementNoisePulse` delegate from Character Controller |
| TerrainMultiplier | float | 0.45–1.50 | Terrain multiplier table (below) |
| EffectiveNoise | float | 0.0–1.0 | Output; fed into F2 and fired as `FAINoiseEvent.Loudness` |

**Terrain Multiplier Table:**

| Terrain / PhysicalMaterial | Multiplier | Walk (0.45) | Sprint (0.85) |
|---------------------------|-----------|------------|--------------|
| Tall Grass | 0.45 | 0.20 | 0.38 |
| Sand / Loose Soil | 0.70 | 0.32 | 0.60 |
| Packed Earth Path (baseline) | 1.00 | 0.45 | 0.85 |
| Shallow Water | 1.25 | 0.56 | 1.00 (clamped) |
| Stone / Rock | 1.35 | 0.61 | 1.00 (clamped) |
| Wooden Floor | 1.50 | 0.68 | 1.00 (clamped) |

*Values above 1.0 before clamping are intentional — sprint on hard surfaces always reaches maximum detectability.*

**Example**: CrouchWalk (0.15) in Tall Grass → 0.15 × 0.45 = 0.068.

---

**F2 — Noise Stimulus Strength** (distance-attenuated)

How loud a noise event is to a specific enemy at a specific distance.

```
NoiseStimulusStrength = EffectiveNoise × max(0, 1 − Distance / HearingRadius)
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| EffectiveNoise | float | 0.0–1.0 | Output of F1 |
| Distance | float | 0–HearingRadius (UU) | 3D distance: noise source → enemy |
| HearingRadius | float | 800–1400 UU | Per-enemy maximum hearing range |
| NoiseStimulusStrength | float | 0.0–1.0 | Output; fed into F3 |

**Example**: Sprint pulse (0.85) on Packed Earth at 600 UU from enemy with HearingRadius 1000 UU:
- EffectiveNoise = 0.85
- NoiseStimulusStrength = 0.85 × (1 − 600/1000) = 0.85 × 0.40 = 0.34

At Distance ≥ HearingRadius: NoiseStimulusStrength = 0 (inaudible).

---

**F3 — Noise Detection Gauge Increment** (applied on pulse event receipt)

One-shot gauge increase when a noise event reaches the enemy's AI perception component.

```
ΔG_noise = NoiseStimulusStrength × NoisePulseWeight     [if NoiseStimulusStrength > HearingThreshold]
ΔG_noise = 0                                             [if NoiseStimulusStrength ≤ HearingThreshold]
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| NoiseStimulusStrength | float | 0.0–1.0 | Output of F2 |
| NoisePulseWeight | float | 0.2–0.5 | Scales how much a single noise pulse moves the gauge; tunable |
| HearingThreshold | float | 0.05–0.25 | Per-enemy minimum audible strength; sub-threshold stimuli are ignored |
| ΔG_noise | float | 0.0–NoisePulseWeight | Immediate gauge increment; applied outside the 0.1 s tick cycle |

**Example**: Sprint pulse NoiseStimulusStrength = 0.34. NoisePulseWeight = 0.30. HearingThreshold = 0.10 (standard guard).
- 0.34 > 0.10 → ΔG_noise = 0.34 × 0.30 = 0.10. Guard's detection gauge increases by 0.10.

**Example (inaudible)**: CrouchWalk in Tall Grass at 200 UU: NoiseStimulusStrength = 0.068 × 0.80 = 0.054. HearingThreshold = 0.10. 0.054 < 0.10 → ΔG_noise = 0. Enemy hears nothing.

---

**F4 — Vision Stimulus Strength** (evaluated per 0.1 s tick)

How visible the player is to an enemy from its current position.

```
VisionStimulus = ZoneModifier × (1 − CoverOcclusion) × CrouchModifier
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| ZoneModifier | float | {0.0, 0.5, 1.0} | 1.0 in Focused zone (≤30°, ≤AdjustedFocusedRange); 0.5 in Peripheral zone (≤60°, ≤500 UU); 0.0 behind enemy |
| CoverOcclusion | float | {0.0, 1.0} | 0.0 if LOS trace clear; 1.0 if ECC_Visibility trace blocked by geometry (binary at MVP — single ray) |
| CrouchModifier | float | {0.6, 1.0} | 0.6 (`CrouchVisibilityMultiplier`) if `bIsCrouching`; 1.0 if standing |
| VisionStimulus | float | 0.0–1.0 | Output; fed into F5 |

**Example**: Player standing in Focused zone, no cover: VisionStimulus = 1.0 × (1 − 0) × 1.0 = 1.0.

**Example**: Player crouching behind a low wall (LOS fully blocked): CoverOcclusion = 1.0 → VisionStimulus = 0.0 regardless of zone.

**Example**: Player crouching in Peripheral zone, no cover: VisionStimulus = 0.5 × 1.0 × 0.6 = 0.30.

---

**F5 — Detection Gauge Tick Update** (vision accumulation, every 0.1 s)

Continuous vision-driven gauge fill and decay.

```
G(t+1) = clamp(G(t) + FillRate × VisionStimulus − DecayRate × (1 − IsVisionStimulated), 0.0, 1.0)
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| G(t) | float | 0.0–1.0 | Current gauge value |
| FillRate | float | 0.3–1.0 per second | How fast the gauge fills under continuous vision; per-enemy tunable |
| VisionStimulus | float | 0.0–1.0 | Output of F4 this tick |
| DecayRate | float | 0.2–0.6 per second | How fast the gauge drains when no vision stimulus; per-enemy tunable |
| IsVisionStimulated | bool | {0, 1} | 1 if VisionStimulus > 0 this tick; 0 otherwise |
| G(t+1) | float | 0.0–1.0 | Updated gauge value, clamped |

*Note: Noise increments (F3) are applied directly to G outside this tick cycle. G at any moment = vision-accumulated value + cumulative noise increments.*

**Gauge thresholds**: SuspicionThreshold = 0.25 (enter Suspicious). AlertThreshold = 0.85 (enter Alert, requires active LOS).

**Example**: Enemy in Focused zone with clear LOS. FillRate = 0.6/s, VisionStimulus = 1.0 (standing, no cover).
- Per tick (0.1 s): G increases by 0.06.
- Reaches SuspicionThreshold 0.25 after ~4 ticks (0.4 s).
- Reaches AlertThreshold 0.85 after ~14 ticks (1.4 s). Enemy detects player in ~1.4 s if player stands in full view.

**Searching state modifier** (applies to F3 threshold and F5 FillRate):

```
FillRate_search   = FillRate_base × 1.5
HearingThreshold_search = HearingThreshold_base × 0.7
```

An enemy in Searching state fills its detection gauge 50% faster and can hear sounds 30% quieter than normal.

---

**F6 — Adjusted Focused Vision Range** (light modifier)

Enemy's maximum focused vision range scaled by ambient light level.

```
AdjustedFocusedRange = FocusedRange_base × LightModifier
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| FocusedRange_base | float | 1200 UU | Default focused zone range at full daylight |
| LightModifier | float | 0.4–1.0 | Ambient light scale from Time-of-Day System; 1.0 until ToD system is implemented |
| AdjustedFocusedRange | float | 480–1200 UU | Effective focused vision range |

**LightModifier reference bands** (pending Time-of-Day System):

| Condition | LightModifier | AdjustedFocusedRange |
|-----------|-------------|----------------------|
| Full Day | 1.0 | 1200 UU |
| Dusk / Dawn | 0.7 | 840 UU |
| Night | 0.4 | 480 UU |

*Peripheral zone range is not modified by LightModifier — enemies react to nearby movement in any light level.*

## Edge Cases

**EC-1: Cutscene / Cinematic transition during active detection**
When `CS_Cinematic` is entered (Camera System handoff to the Cinematic System), the Stealth System must suppress all gauge accumulation for the duration. Enemy gauges freeze at their current values — they do not fill or drain — and no `FAINoiseEvent` is fired from `OnMovementNoisePulse` while cinematic is active. On cinematic exit, gauges resume from their frozen values. Enemies do not teleport back to Unaware because a cutscene played.

**EC-2: Player death mid-detection**
On `OnPlayerDeath` (from Health & Stamina System), `UAltynStealthSubsystem` broadcasts a reset: all enemy detection gauges drop to 0.0 and all enemies transition to Unaware (or to their patrol-route starting positions, per Enemy AI System rules). Detection state does not persist through death and respawn. The load restores the world to its checkpoint state, where enemy positions and patrol routes are also reset.

**EC-3: Save/Load while enemies are Suspicious or Alerted**
Enemy detection state is not serialized to `UAltynSaveGame`. On any load (manual save, checkpoint, death-load), all enemies initialize to Unaware. The game world restores to the saved point, and enemies restart their patrol routes. Players cannot "freeze" enemies in Alert or Searching state by saving and loading.

**EC-4: Stone throw impact Loudness is independent of terrain multiplier**
`FAINoiseEvent.Loudness = 0.75` for the distraction throw is a fixed value — it represents the impact sound of the stone, not the player's movement. The terrain multiplier table (F1) applies only to `OnMovementNoisePulse` events. A stone thrown onto Wooden Floor is exactly as loud as one thrown onto Packed Earth from the enemy's perspective.

**EC-5: Enemy killed while Suspicious or Searching**
When an enemy is killed (Combat System fires OnEnemyDeath), `AAltynEnemyAIController::EndPlay` deregisters the enemy from `UAltynStealthSubsystem`. The subsystem recalculates `GetHighestAlertTier()` immediately. If the killed enemy was the sole source of the Tense/Aware tier, the Character Controller idle variant may change on the next `OnAwarenessChanged` broadcast. No special case required — the deregistration handles it.

**EC-6: Multiple enemies with different alert states simultaneously**
Enemy A is Alert; Enemy B is Suspicious; Enemy C is Unaware. The WorldSubsystem holds the highest tier: Alert. Character Controller idle variant = Tense. Enemy A attacks; Enemy B investigates; Enemy C patrols. When Enemy A transitions to Searching (loses LOS), the WorldSubsystem highest tier becomes Suspicious — Character Controller drops to Aware. Each enemy's gauge is independent; the aggregate reflects the current worst case.

**EC-7: Alert enemy ignores distraction, Suspicious enemy in the same area does not**
Player throws a stone near two enemies: Enemy A (Alert, charging the player's last known position) and Enemy B (Suspicious, advancing slowly). Enemy A ignores the `Tag = "Distraction"` event per Rule 10. Enemy B reacts — its Suspicious state redirects toward the stone impact point. The two enemies may now diverge spatially (Enemy A charges forward, Enemy B investigates the stone). This is correct and intentional: Alert is fully committed, Suspicious is still investigative.

**EC-8: Detection gauge at AlertThreshold (0.85) but LOS broken at the same tick**
The guard's gauge fills to exactly 0.85 on a tick where VisionStimulus = 1.0, but the same tick, CoverOcclusion transitions to 1.0 (player steps behind a wall). Alert requires gauge ≥ 0.85 AND active LOS. Since CoverOcclusion = 1.0 means VisionStimulus = 0 this tick, the threshold check finds: gauge ≥ 0.85 but no LOS → transitions to **Searching**, not Alert. The player slipped behind cover at the last moment.

**EC-9: Terrain surface untagged or missing PhysicalMaterial**
If the PhysicalMaterial under the player's capsule cannot be read (untagged geometry, prototype level), `TerrainMultiplier = 1.0` (Packed Earth baseline). Untagged geometry is treated as neutral. Level design must ensure all navigable surfaces have a PhysicalMaterial assigned — this is a level design quality gate, not an error state.

**EC-10: Player stands in tall grass with enemy on elevated ground overhead**
Tall grass has no ECC_Visibility blocking collision — it cannot occlude the LOS trace from above. An enemy on a cliff above the player sees through the grass: CoverOcclusion = 0.0. The player's only concealment advantage in tall grass is the noise terrain multiplier. Elevated LOS is a known and intended advantage for enemies on high ground — level design should use solid geometry (rock overhangs, roof geometry) to create genuine cover from above.

**EC-11: Cinematic System exits while enemy gauge is frozen at Alert threshold**
On cinematic exit, gauges resume from frozen values. If a gauge was frozen at 0.84 (just below AlertThreshold), it resumes accumulating from 0.84. If the player remained in the enemy's Focused zone during the cinematic, the first post-cinematic tick may immediately push the gauge to Alert. This is correct — the cinematic did not save the player from a detection they were about to receive.

## Dependencies

**Upstream (this system depends on):**
- **Character Controller GDD** — `OnMovementNoisePulse(float NoiseLevel)` delegate (subscribed); `bIsCrouching` bool (read per tick); noise level constants: CrouchIdle=0.0, CrouchWalk=0.15, Dodge=0.30, Walk=0.45, LandingSoft=0.55, Sprint=0.85, LandingHard=0.90 (locked — owned by CC GDD); `AAltynCharacter` idle variant (Calm/Aware/Tense) driven by this system's `OnAwarenessChanged` broadcast.
- **Input System GDD** — `IA_Crouch` (Bool/Triggered, toggle, IMC_OnFoot) enables crouching; `IA_Throw` (Bool, hold-and-release, IMC_OnFoot) triggers stone throw. **Cross-system flag**: `IA_Throw` is not yet in the Input System GDD and must be added.

**Downstream (systems that depend on this one):**
- **Enemy AI System GDD** — reads per-enemy alert state via `AAltynEnemyAIController::GetAlertState()` and Blackboard key `AlertTier`; reads `PointOfInterest` (FVector) Blackboard key for distraction navigation. Group coordination (alarm propagation between enemies) is Enemy AI's responsibility — not defined here.
- **Combat & Stealth HUD GDD** — reads `UAltynStealthSubsystem::GetHighestAlertTier()` for detection meter display.
- **Audio System GDD** — `UAudioStateManager` subscribes to `UAltynStealthSubsystem::OnAwarenessChanged`; maps Unaware→Calm layer, Suspicious→Suspicion layer, Alert/Searching→Combat layer. (Cross-system: verify Audio GDD Section C references this delegate.)
- **Cinematic/Cutscene System GDD** — must call `UAltynStealthSubsystem::SetStealthActive(false/true)` (or equivalent) on CS_Cinematic entry/exit to freeze/resume gauge accumulation.

**Cross-system flags generated by this GDD:**
- Input System GDD: add `IA_Throw` (Bool, hold-and-release) to `IMC_OnFoot` — Gamepad: Hold R1 + aim stick, release to throw; Keyboard: Hold G, release.
- Open World / Level Streaming GDD: all navigable surfaces must have PhysicalMaterial assigned (quality gate for Stealth + Audio terrain surface reads).

## Tuning Knobs

All values should be externalized to a `UStealthSystemConfig` data asset or per-enemy `UPROPERTY(EditAnywhere)` on `AAltynEnemy`. Safe ranges indicate values that preserve intended gameplay without breaking the system's tone or formulas.

| Knob | Default | Safe Range | Affects |
|------|---------|-----------|---------|
| `SuspicionThreshold` | 0.25 | 0.15–0.40 | How much stimulus before enemy becomes alert to player presence |
| `AlertThreshold` | 0.85 | 0.70–0.95 | Full detection threshold; lower = faster alert, less grace |
| `FillRate_base` (per-enemy) | 0.60/s | 0.30–1.0/s | How fast gauge fills under continuous vision |
| `DecayRate` (per-enemy) | 0.40/s | 0.20–0.60/s | How fast gauge drains when player breaks line-of-sight |
| `SuspicionDecayTime` | 8 s | 4–15 s | Time from gauge = 0 to Unaware transition; how long Suspicious "lingers" |
| `AlertLOSBreakTime` | 3 s | 1.5–5 s | Time enemy must lose LOS to transition Alert → Searching |
| `SearchDuration` | 30 s | 20–60 s | Max time in Searching before returning to Unaware with no re-stimulus |
| `FocusedZoneHalfAngle` | 30° | 20–45° | Narrower = easier to approach from flanks |
| `FocusedZoneRange_base` | 1200 UU | 800–1500 UU | Longer = harder to approach across open steppe |
| `PeripheralZoneHalfAngle` | 60° | 45–80° | Wider = harder to sneak past behind enemy |
| `PeripheralZoneRange` | 500 UU | 300–700 UU | Close-range peripheral sensitivity |
| `CrouchVisibilityMultiplier` | 0.6 | 0.4–0.8 | Lower = crouching more effective; below 0.4 makes crouching near-invisible |
| `HearingRadius` (standard guard) | 1000 UU | 600–1400 UU | Hearing range ceiling |
| `HearingThreshold` (standard guard) | 0.10 | 0.05–0.25 | Minimum audible noise; lower = hears more |
| `NoisePulseWeight` | 0.30 | 0.15–0.50 | How much a single noise pulse moves the gauge |
| `SearchingFillRateMultiplier` | 1.5× | 1.2–2.0× | Heightened perception fill rate during Searching |
| `SearchingHearingThresholdMultiplier` | 0.7× | 0.5–0.9× | Lower HearingThreshold during Searching |
| `DistractionThrowLoudness` | 0.75 | 0.50–0.90 | Fixed loudness of stone impact; independent of terrain |
| `DistractionThrowRangeMin` | 600 UU | 400–800 UU | Minimum throw range (light hold) |
| `DistractionThrowRangeMax` | 1200 UU | 900–1500 UU | Maximum throw range (full hold) |
| `PointBlankNoiseAlertThreshold` | 0.90 | 0.80–1.0 | EffectiveNoise required for instant Alert within 300 UU (no gauge fill needed) |
| `LightModifier` | 1.0 (no ToD) | 0.4–1.0 | Enemy focused range scale; activated by Time-of-Day system |

## Visual/Audio Requirements

- **Enemy alert state transitions** require audio cues per transition: ? vocalisation on Suspicious entry; ! alarm call on Alert entry; "lost player" vocalisation on Searching entry; brief deescalation audio on Unaware return. Enemy audio assets are defined in Audio System / Enemy AI GDD scope.
- **Music layer transitions** driven by `OnAwarenessChanged` are defined in the Audio System GDD (UAudioStateManager). No additional music specification here.
- **Stone throw SFX**: throw release (whoosh), stone impact (thud on earth, crack on stone, splash in water — differentiated by impact PhysicalMaterial). Impact audio uses the Audio System's surface SFX selection logic — same PhysicalMaterial tag as the terrain multiplier.
- **No player character audio for stealth posture**: crouching does not produce a specific enter/exit sound. The absence of sprint/walk SFX when crouching is the audio feedback.
- **No screen vignette, heartbeat, or distortion effects for near-detection**: the tone requires the player to feel tension without UI assistance. HUD detection meter (in Combat & Stealth HUD GDD) is the only visual feedback.

## UI Requirements

The Stealth System provides data to the HUD; it does not own HUD implementation.

- **Provides to Combat & Stealth HUD**: `UAltynStealthSubsystem::GetHighestAlertTier()` (enum) — drives the detection meter display.
- **Stone throw arc preview**: a parabolic arc indicator while `IA_Throw` is held. Minimal — a dotted arc line or landing indicator only. No complex aiming reticle. Arc parameters are derived from `DistractionThrowRangeMin/Max` and hold duration. Visual implementation is the HUD GDD's scope.
- **No in-world enemy awareness indicators**: enemy question/exclamation marks are a Combat & Stealth HUD design decision. The Stealth System fires `AlertTier` changes; the HUD decides how to visualize them.

## Acceptance Criteria

- [ ] **AC-01**: Player sprinting on Packed Earth at 600 UU from a standard guard triggers guard's Suspicious state within 2 `OnMovementNoisePulse` events.
- [ ] **AC-02**: Player crouching and walking through Tall Grass at 200 UU from a standard guard produces no state change (EffectiveNoise = 0.068, below HearingThreshold 0.10).
- [ ] **AC-03**: Player standing in Focused zone with clear LOS triggers enemy Alert state in ≤ 1.5 s (at FillRate 0.60/s: ~14 ticks × 0.1 s).
- [ ] **AC-04**: Player stepping behind solid geometry (ECC_Visibility blocking) sets VisionStimulus to 0 within 1 evaluation tick (0.1 s).
- [ ] **AC-05**: Alert enemy with active LOS cannot transition to Unaware directly — must break LOS for > 3 s to reach Searching first.
- [ ] **AC-06**: Stone throw to a location 800 UU away transitions a nearby Unaware guard to Suspicious and redirects the guard to the impact location.
- [ ] **AC-07**: Alert enemy does not redirect toward stone throw impact — continues charging player's last known position.
- [ ] **AC-08**: `AAltynCharacter` idle variant changes to Tense when any enemy is in Alert; drops to Aware when highest tier is Searching; drops to Calm when all enemies are Unaware.
- [ ] **AC-09**: On save/load (any type), all enemies initialize to Unaware with no residual detection gauge.
- [ ] **AC-10**: Searching enemy reaches SuspicionThreshold (0.25) 1.5× faster than an Unaware enemy under identical vision conditions.
- [ ] **AC-11**: An enemy on an elevated platform has unobstructed LOS to a player crouching in tall grass directly below — CoverOcclusion = 0.0 (grass does not block ECC_Visibility).
- [ ] **AC-12**: Sprint on Stone/Rock surface produces EffectiveNoise = 1.0 (0.85 × 1.35 = 1.148, clamped to 1.0).
- [ ] **AC-13**: All enemy detection gauges reset to 0.0 within 1 frame of `OnPlayerDeath` firing.
- [ ] **AC-14**: PhysicalMaterial-less (untagged) surface uses `TerrainMultiplier = 1.0` (Packed Earth baseline) — no crash, no silent failure.

## Open Questions

- **OQ-1 — Deep water terrain**: No PhysicalMaterial / TerrainMultiplier defined for swimming or deep river crossings. When the Open World GDD defines river navigation, this table needs a "Deep Water" entry. Recommend 1.50 (same as Wooden Floor), but confirm with Open World / Horse System GDDs.
- **OQ-2 — Mounted stealth**: When the player is on horseback (Horse System — Vertical Slice), does the Stealth System receive horse movement noise via the same `OnMovementNoisePulse` delegate, or does the Horse System define its own? Defer to Horse System GDD.
- **OQ-3 — Stealth kill camera response**: Camera System OQ-3 noted: if stealth-kill animations need a camera response, this GDD must define the trigger delegate. At MVP, no stealth-kill mechanic exists. Revisit when Combat System GDD is authored.
- **OQ-4 — Per-enemy-type cone presets**: StandardGuard, Archer, Scout likely need different FocusedZoneRange and HearingThreshold values. This GDD defines the knob ranges; the Enemy AI GDD assigns values to enemy types. Coordinate at Enemy AI GDD authoring time.
- **OQ-5 — Visual grass concealment**: Current design provides no LOS concealment in tall grass (noise reduction only). If playtest reveals the steppe fantasy requires visual hiding in grass, a "concealment zone radius" system would be needed as a separate overlay on the detection formula. Defer to Polish phase; flag for playtest evaluation.
