# Camera System

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-10
> **Implements Pillar**: Pillar 1 (the steppe is bigger than you — camera defines world scale) + Pillar 3 (cavalry charge payoff)

## Overview

The Camera System controls all camera behavior for Altyn Adam: a third-person follow camera for on-foot exploration and combat, a lock-on mode that binds to a single enemy during combat encounters, a near-death modifier that shifts gravity and angle in response to the boy's health, and a cinematic handoff mode that yields control to the Cinematic/Cutscene System during scripted sequences. Implemented on `USpringArmComponent` + `UCameraComponent` attached to `AAltynCharacter`, with modular `UCameraModifier` subclasses for health-driven behaviors (Dutch tilt, hit impulse) and state-driven adjustments (FOV, arm length per movement mode). `APlayerCameraManager` coordinates mode transitions and modifier stack. Character Controller exposes `bIsSprinting` and `bIsCrouching`; Health & Stamina exposes `HealthCurrent` and the `OnPlayerDeath` / `OnWoundedStateExited` delegates. Camera System subscribes to these and adjusts accordingly — no polling, event-driven.

The camera has one governing philosophy: **the steppe is always bigger than the frame**. The default follow distance is set not to keep the boy centered and comfortable, but to keep him correctly small against the world around him. Sprint FOV does not widen to feel fast — it barely widens because speed in this world does not make you larger. Lock-on does not pull tight to the enemy — it holds enough distance to show the boy and the threat together, and the space between them. Every value in this system exists to serve that contract: the world is the subject; the boy is inside it.

The camera is the player's body in the world before any input is registered. A camera that jostles and cuts reads as a game asking for attention; a camera that holds still and breathes with the scene reads as a world asking to be inhabited. Altyn Adam's camera is of the second kind.

## Player Fantasy

The frame is mostly steppe. It was always going to be mostly steppe. In the prologue the boy walks toward the herd and the camera's distance says: this is a small person in a large place, and the place is older than him. After the fire it says the same thing, because the steppe does not resize itself for a tragedy. In combat the camera does not pull tight to a swordfight, because a swordfight in this world is two small people doing a small thing, and the world is still doing what the world is doing. The Dutch tilt at low health is the frame admitting that the boy can no longer hold the world straight, not the world tilting toward him. At the end you charge a Persian army and the boy occupies the same fraction of the frame he did when he was thirteen and tying his horse, and the army fills the rest, and you understand for the first time what the camera has been telling you the whole game: the steppe is always bigger than you, and the only thing that ever changes is what you are willing to ride into anyway.

> **Design rule this establishes:** The boy occupies a roughly constant fraction of the frame across all gameplay modes — exploration, combat, lock-on, near-death, cavalry charge. This is the test every tuning value in this GDD is measured against.

## Detailed Design

### Core Rules

**Rule 1 — Camera Architecture**

The camera is implemented as `USpringArmComponent` + `UCameraComponent` attached to `AAltynCharacter`. All behavioral changes — FOV, arm length, tilt, impulse — are applied via `UCameraModifier` subclasses on the `APlayerCameraManager` modifier stack. `APlayerCameraManager::SetFOV()` is not used; FOV changes are written via `ModifyCamera()` with per-frame `FMath::FInterpTo` interpolation toward a target value.

---

**Rule 2 — Governing Constraint: Constant Fraction of Frame**

The boy occupies a roughly constant fraction of the frame across all camera modes: exploration, combat, lock-on, bow aim, near-death, and the cavalry charge finale. This is the primary test for all tuning values. Any arm length, FOV, or lock-on framing adjustment that shrinks or enlarges the boy relative to the world requires explicit design justification. The cavalry charge does not receive a special camera pull-back — the army fills the rest of the frame because the frame was always this size.

---

**Rule 3 — Camera Mode and Modifier Inventory**

Six camera states and three permanent modifiers that overlay active states.

| ID | Type | Description |
|----|------|-------------|
| `CS_Exploration` | State | Default on-foot follow camera |
| `CS_Combat` | State | Slight arm shortening; active when `bInCombat == true` |
| `CS_LockOn` | State | Two-target framing; replaces `CS_Combat` when hold active |
| `CS_AimBow` | State | Tighter arm, narrowed FOV; bow-draw camera |
| `CS_Cinematic` | State | Control yielded to Cinematic System; all modifiers suspended |
| `Mod_NearDeath` | Modifier | Dutch tilt + orbit lower; overlaid on any active state |
| `Mod_HitImpulse` | Modifier | Per-hit directional displacement, critically-damped spring |
| `Mod_FOVState` | Modifier | Manages all FOV interpolation (sprint bonus, mode base, bow zoom) |

**Modifier priority stack (highest to lowest):**
1. `CS_Cinematic` — absolute override; all modifiers suspended while active
2. `Mod_NearDeath` — always layered on top of active mode
3. `Mod_HitImpulse` — per-hit, completes in 0.18 s, then idle
4. `CS_LockOn` / `CS_AimBow` — replace Exploration/Combat orbit and arm
5. `CS_Combat` — replaces Exploration arm + FOV
6. `CS_Exploration` — base state
7. Sprint FOV bonus — additive on any active state
8. Crouch arm reduction — additive on any active state

---

**Rule 4 — CS_Exploration (Default Follow Camera)**

- `TargetArmLength`: `ExplorationArmLength = 475 UU` (tunable range: 450–500)
- `FieldOfView`: `FOV_Exploration = 80°` (tunable range: 75–85°)
- `bEnableCameraLag = true`, `CameraLagSpeed = 10.0`
- `bDoCollisionTest = true`, `ProbeSize = 12 cm`
- Entry: game load; `bInCombat` → false; `CS_AimBow` exits while not in combat; `CS_Cinematic` exits (if prior state was Exploration)
- Exit: `bInCombat` → true (→ `CS_Combat`, 0.4 s blend); `IA_LockOn` held (→ `CS_LockOn`); `IA_AimBow` held (→ `CS_AimBow`); cinematic trigger (→ `CS_Cinematic`)

---

**Rule 5 — CS_Combat**

- `TargetArmLength`: `CombatArmLength = ExplorationArmLength × 0.88 ≈ 418 UU` (10–15% shorter than exploration)
- `FieldOfView`: `FOV_Combat = FOV_Exploration − 3° ≈ 77°`
- Transition in: 0.4 s `FInterpTo` blend from Exploration arm/FOV values
- Transition out: 0.5 s blend back to Exploration values when `bInCombat` → false
- Entry: `bInCombat` → true
- Exit: `bInCombat` → false (→ `CS_Exploration`); `IA_LockOn` held (→ `CS_LockOn`); `IA_AimBow` held (→ `CS_AimBow`); cinematic (→ `CS_Cinematic`)

---

**Rule 6 — CS_LockOn**

Available from `CS_Exploration` and `CS_Combat`. Not available from `CS_AimBow`.

**Target acquisition — ValidityScore:**
The highest-scoring valid enemy within `MaxLockOnRange = 1000 UU` is selected. Score is evaluated at the moment `IA_LockOn` is first held; not re-evaluated continuously.

| Component | Formula | Weight |
|-----------|---------|--------|
| Distance | `1.0 − (Distance / MaxLockOnRange)` | 0.4 |
| Angle | `1.0 − (AngleDelta / 180°)` | 0.4 |
| Line of sight | Sphere trace from `UCameraComponent` to enemy `Chest_Bone`, radius 15 UU, `ECC_Visibility` — 1.0 clear, 0.0 blocked | 0.2 |

An enemy directly in front at 400 UU scores higher than an enemy directly behind at 200 UU — the player's forward field of view takes priority over raw proximity.

**Arm length during lock-on:**
`LockOnArmLength = CombatArmLength + (EnemyDistance × LockOnArmScale)`, clamped to `[400 UU, 600 UU]`. `LockOnArmScale = 0.15`. Both the boy and the target occupy the frame at typical combat distances without the arm exceeding exploration length.

**Horizontal orbit framing:**
The camera anchor holds `MaxLockOnOrbitAngle = 25°` of horizontal orbit budget. `Mod_LockOn` (a `UCameraModifier`) adjusts `SocketOffset` each frame to keep the locked target within the inner 30% of frame width. The orbit does not exceed the 25° budget — beyond that, the spring arm slowly rotates to re-center.

**Target lost / switching:**
- Target dies mid-lock: auto-acquire next highest `ValidityScore` candidate within range; if none → `CS_Combat`, `LockOnTransitionTime = 0.25 s`.
- Target exceeds `MaxLockBreakRange = 1300 UU`: → `CS_Combat`.
- New enemy from behind while locked: no auto-switch. Lock target is honored until player releases/re-holds or target dies.
- `IA_LockOn` released: → `CS_Combat` (if `bInCombat`) or `CS_Exploration`.

**Collision during lock-on:** `LockOnProbeSize = 8 cm`, `CameraLagSpeed = 14` (faster contraction — combat demands rapid repositioning).

---

**Rule 7 — CS_AimBow**

`IA_LockOn` is not available while `CS_AimBow` is active. Lock-on is a melee tool; the bow is aimed with trained skill, not a targeting system.

- `TargetArmLength`: `AimBowArmLength = ExplorationArmLength × 0.70 ≈ 333 UU`
- `FieldOfView`: `FOV_AimBow = FOV_Exploration − 10° = 70°`
- Camera position: `SocketOffset` shifts 20 UU to the left of the boy's right shoulder — the bow arm occupies the left side; shifting the camera left opens the right-side sight line to the target.
- Entry: `IA_AimBow` held (from `CS_Exploration` or `CS_Combat`)
- Exit: `IA_AimBow` released → `CS_Combat` (if `bInCombat == true`) else `CS_Exploration`
- Transition blend: 0.25 s in, 0.25 s out

---

**Rule 8 — Mod_NearDeath (Dutch Tilt Overlay)**

Trigger: `HealthCurrent` drops below `NearDeathThreshold = 10.0 HP` — fires once on threshold crossing; does not re-fire on subsequent hits while active.

Effect: `Mod_NearDeath` interpolates its modifier `Alpha` from 0.0 to 1.0 over 1.5 s. At full `Alpha`: `ViewRotation.Roll += NearDeathCameraTilt = 1.5°` and `SocketOffset.Z −= 5 UU` (orbit pivot lowers). Both interpolate smoothly; no snap.

**Clear condition:** `OnWoundedStateExited` fires (`HealthCurrent` rises above `WoundedThreshold = 30.0 HP` via campfire rest). The tilt does **not** clear when HP recovers from 9 to 11 HP. Once the near-death tilt fires, it persists for the remainder of the encounter and clears only at a campfire.

Applied over all active modes. Suspended (not reset) during `CS_Cinematic` — resumes on exit if health has not been restored.

---

**Rule 9 — Mod_HitImpulse (Directional Displacement)**

Trigger: any `ApplyDamage()` call → H&S fires delegate with hit direction vector → Camera System receives it.

Effect: single displacement opposite the hit vector, magnitude `CameraImpulseMagnitude = 3.0 UU`. Returns to origin over `CameraImpulseReturn = 0.18 s` via critically-damped spring — no oscillation, no secondary bounce. If hit direction is unknown: displace downward. Multiple simultaneous hits stack additively; spring returns to origin regardless.

Implementation: `UCameraModifier_HitImpulse` stores the hit direction as state, runs a per-frame spring simulation, drives `ViewLocation` delta in `ModifyCamera()`.

Active in all modes except `CS_Cinematic`.

---

**Rule 10 — CS_Cinematic**

Entry: Cinematic/Cutscene System calls `APlayerController::SetViewTarget(CineCameraActor, FViewTargetTransitionParams)`.
Exit: `SetViewTarget()` called again pointing at `AAltynCharacter` with a blend-in time.

**Restore rule:** on cinematic exit, restore to the camera state that was active at entry time (state preserved on entry). Exception: `CS_LockOn` is never restored — if a cinematic interrupts lock-on, exit to `CS_Combat` (if `bInCombat`) or `CS_Exploration`. Cinematics always clear lock-on state.

All camera modifiers are suspended during `CS_Cinematic`. `Mod_NearDeath` pauses (does not reset) and resumes if health has not recovered above the Wounded threshold on exit.

*Implementation note:* Use `ULevelSequencePlayer` for sequence playback. `IMovieScenePlayer` is deprecated in UE 5.6 and must not be subclassed.

---

**Rule 11 — Sprint FOV Adjustment (driven by `bIsSprinting`)**

When `bIsSprinting` → true: `Mod_FOVState` target increases by `SprintFOVBonus = 5°` above the current mode-base FOV. Arm length does not change — speed does not make the boy larger; it makes the world move past him faster.

- Blend in: 0.3 s `FInterpTo`
- Blend out: 0.5 s (slower out — prevents FOV snapping when sprint exhausts mid-stride)

Applied additively over any active mode's base FOV.

---

**Rule 12 — Crouch Arm Adjustment (driven by `bIsCrouching`)**

When `bIsCrouching` → true: arm shortens by `CrouchArmDelta = 28 UU` (`StandingHalfHeight − CrouchedHalfHeight = 88 − 60`). Preserves the boy's fraction of the frame while crouched.

Camera pitch: unchanged. The camera holds its world-space pitch — it does not dip to track the boy's eyeline. The boy descends into the frame; the world does not tilt toward him.

`CrouchArmLength` floor: `max(CombatArmLength, ActiveArmLength − CrouchArmDelta)` — crouch arm cannot be shorter than combat arm (400 UU).

Blend: 0.15 s in (matches stand-to-crouch animation timing), 0.2 s out.

---

**Rule 13 — Minimum Arm Length Floor**

`MinArmLength = 150 UU` — spring arm collision contraction never goes below this. Below 150 UU the camera is uncomfortably close to the back of the boy's head.

*Level design constraint:* any indoor space containing combat must have minimum ceiling clearance of ~300 UU. Flag for Open World, Hub System, and level design guidelines.

### States and Transitions

| State | Entry | Exit | Active Modifiers |
|-------|-------|------|-----------------|
| `CS_Exploration` | Load; `bInCombat` → false; `CS_AimBow`/`CS_LockOn` exit not in combat; cinematic exit (was Exploration) | `bInCombat` → true (0.4 s blend → `CS_Combat`); `IA_LockOn` held; `IA_AimBow` held; cinematic trigger | `Mod_NearDeath` if active; `Mod_HitImpulse`; sprint FOV; crouch arm |
| `CS_Combat` | `bInCombat` → true | `bInCombat` → false (0.5 s → `CS_Exploration`); `IA_LockOn` held; `IA_AimBow` held; cinematic trigger | Same as Exploration |
| `CS_LockOn` | `IA_LockOn` held + valid target, from `CS_Combat` or `CS_Exploration` | `IA_LockOn` released; target > `MaxLockBreakRange`; all targets dead (0.25 s → `CS_Combat`); `IA_AimBow` held (→ `CS_AimBow`, lock cleared); cinematic | `Mod_NearDeath` if active; `Mod_HitImpulse`; sprint FOV; crouch arm |
| `CS_AimBow` | `IA_AimBow` held (from Exploration or Combat) | `IA_AimBow` released → `CS_Combat` or `CS_Exploration` | `Mod_NearDeath` if active; `Mod_HitImpulse` |
| `CS_Cinematic` | `SetViewTarget(CineCameraActor)` by Cinematic System | `SetViewTarget(AAltynCharacter)` by Cinematic System | None — all suspended |

### Interactions with Other Systems

| System | Camera Receives | Camera Exposes | Interface |
|--------|----------------|----------------|-----------|
| Character Controller | `bIsSprinting`, `bIsCrouching` (per-state change) | — | State flags; delegate or per-tick read |
| Health & Stamina | `HealthCurrent` float (NearDeath threshold); `OnWoundedStateExited` (clear Mod_NearDeath) | — | Float read; delegate subscription |
| Input System | `IA_LockOn` hold (gates `CS_LockOn`); `IA_AimBow` hold (gates `CS_AimBow`) | — | Input Action events |
| Combat System | `bInCombat` bool (CS_Combat trigger); hit direction vector (Mod_HitImpulse) | — | Bool broadcast; damage delegate |
| Cinematic/Cutscene System | `SetViewTarget` calls (enter/exit `CS_Cinematic`) | Active state at entry (for restore on exit) | `APlayerController::SetViewTarget()` |
| Stealth System | *(none at MVP)* | *(TBD — flag for Stealth System GDD)* | — |

**Cross-system flag — Input System GDD:** `IA_LockOn` is not currently defined in the Input System GDD. Add as a `Bool / Ongoing` hold action bound in `IMC_OnFoot` only (not `IMC_AimBow`, not `IMC_Mounted`). Required before camera implementation begins.

**Cross-system flag — Cinematic/Cutscene System GDD:** camera handoff must route through `APlayerController::SetViewTarget()` + `ULevelSequencePlayer`. `IMovieScenePlayer` is deprecated in UE 5.6 and must not be subclassed.

## Formulas

**F1 — Camera Mode Arm Lengths**

```
ExplorationArmLength = 475 UU  (tunable: 450–500)
CombatArmLength      = ExplorationArmLength × 0.88 ≈ 418 UU
AimBowArmLength      = ExplorationArmLength × 0.70 ≈ 333 UU
CrouchArmDelta       = StandingHalfHeight − CrouchedHalfHeight = 88 − 60 = 28 UU
CrouchArmLength      = max(CombatArmLength, ActiveArmLength − CrouchArmDelta)
MinArmLength         = 150 UU  (collision floor — never contracts below this)
```

| Variable | Value | Range | Effect if Too Short | Effect if Too Long |
|----------|-------|-------|--------------------|--------------------|
| `ExplorationArmLength` | 475 UU | 450–500 | Boy too large; world feels small | Boy reads as a figure in the distance; combat legibility suffers |
| `CombatArmLength` | ~418 UU | 400–440 | Combat feels claustrophobic | Boy and enemy don't share frame comfortably |
| `AimBowArmLength` | ~333 UU | 300–375 | Bow arm clips camera | Bow reticle target too far to read |

---

**F2 — Lock-On Target Validity Score**

```
ValidityScore = (DistanceScore × 0.4) + (AngleScore × 0.4) + (LOSScore × 0.2)

DistanceScore     = 1.0 − (Distance / MaxLockOnRange)
AngleScore        = 1.0 − (AngleDelta / 180°)
LOSScore          = 1.0 (clear) | 0.0 (blocked)

MaxLockOnRange    = 1000 UU
MaxLockBreakRange = 1300 UU
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| `Distance` | float | 0–1000 UU | World-space distance from player to candidate |
| `AngleDelta` | float | 0°–180° | Angle between player forward and player-to-enemy vector |
| `LOSScore` | bool→float | 0.0 or 1.0 | Sphere trace (radius 15 UU) from `UCameraComponent` to enemy `Chest_Bone`, `ECC_Visibility` |
| `ValidityScore` | float | 0.0–1.0 | Highest score = selected target |

**Output range:** 0.0 (occluded/max range) to 1.0 (close, in front, unobstructed).

**Worked example:** Enemy A at 400 UU, 25° off-center, LOS clear: `0.24 + 0.344 + 0.2 = 0.784`. Enemy B at 200 UU, 140° behind, LOS clear: `0.32 + 0.089 + 0.2 = 0.609`. A wins despite being twice as far — forward visibility takes priority over proximity.

---

**F3 — Lock-On Arm Length (Dynamic)**

```
LockOnArmLength = CombatArmLength + (EnemyDistance × LockOnArmScale)
                = 418 + (EnemyDistance × 0.15)

Clamped to [400 UU, 600 UU]
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| `CombatArmLength` | float | 400–440 UU | Base arm during combat |
| `EnemyDistance` | float | 0–1000 UU | World-space distance to locked target |
| `LockOnArmScale` | float | 0.10–0.20 | Fraction of distance added to arm |
| `LockOnArmLength` | float | 400–600 UU | Final arm length during lock-on |

**Worked example:** Enemy 500 UU away → `418 + (500 × 0.15) = 493 UU`. Both characters share the frame clearly.

---

**F4 — Lock-On Horizontal Orbit**

```
OrbitOffset = clamp(TargetScreenX − TargetIdealScreenX, −MaxLockOnOrbitAngle, +MaxLockOnOrbitAngle)
```

| Variable | Type | Range | Description |
|----------|------|-------|-------------|
| `TargetScreenX` | float | 0.0–1.0 | Normalized horizontal screen position of locked target |
| `TargetIdealScreenX` | float | 0.5–0.7 | Desired screen position (right of center; right-shoulder framing) |
| `MaxLockOnOrbitAngle` | float | 15°–35° | Maximum horizontal orbit from neutral |
| `OrbitOffset` | float | −25° to +25° | Degrees added to spring arm yaw via `SocketOffset` |

**Output range:** clamped to ±`MaxLockOnOrbitAngle`. The clamp is intentional — the camera has a follow budget; it does not chase the target beyond it.

---

**F5 — FOV Per Mode**

```
FOV_Exploration = 80°
FOV_Combat      = FOV_Exploration − 3° = 77°
FOV_AimBow      = FOV_Exploration − 10° = 70°
FOV_Sprint      = ActiveModeFOV + SprintFOVBonus = ActiveModeFOV + 5°
```

All FOV values transition via `FMath::FInterpTo` per frame in `Mod_FOVState::ModifyCamera()`.

| Variable | Value | Range | Effect if Too Narrow | Effect if Too Wide |
|----------|-------|-------|---------------------|--------------------|
| `FOV_Exploration` | 80° | 75–85° | Tunnel vision on the steppe | Depth lost; world feels flat |
| `SprintFOVBonus` | 5° | 3–8° | Speed imperceptible | Fish-eye |
| `FOV_AimBow` | 70° | 65–75° | Flanking enemies invisible | Bow reticle feels imprecise |

---

**F6 — Hit Impulse Spring**

Per-tick implementation in `UCameraModifier_HitImpulse::ModifyCamera()`:

```
ImpulseVelocity   += (Origin − CurrentImpulseOffset) × SpringStiffness × DeltaTime
ImpulseVelocity   *= (1.0 − SpringDamping × DeltaTime)
CurrentImpulseOffset += ImpulseVelocity × DeltaTime
ViewLocation       += CurrentImpulseOffset
```

| Variable | Value | Range | Description |
|----------|-------|-------|-------------|
| `CameraImpulseMagnitude` | 3.0 UU | 1.5–5.0 | Initial displacement in hit-opposite direction |
| `CameraImpulseReturn` | 0.18 s | 0.12–0.25 | Time to return to origin |
| `SpringStiffness` | ~150 | 80–250 | Return speed (derive from `CameraImpulseReturn`) |
| `SpringDamping` | ~0.9 | 0.8–1.0 | Critically-damped; no oscillation below 1.0 |

**Output range:** 0 UU (at rest) to `CameraImpulseMagnitude` UU at hit. Returns within `CameraImpulseReturn` seconds with no oscillation.

## Edge Cases

**E1 — `IA_LockOn` Held: No Valid Target Exists**

Player holds `IA_LockOn` but `ValidityScore` is 0.0 for all candidates (none within `MaxLockOnRange`, or all blocked). Camera does not enter `CS_LockOn`; remains in `CS_Combat` or `CS_Exploration`. If a valid target appears while the input is still held, lock-on activates immediately without requiring a re-press.

**E2 — Locked Target Dies, No Valid Successor**

Camera evaluates for valid successors — none found. Transitions to `CS_Combat` over `LockOnTransitionTime = 0.25 s`. Does not transition to `CS_Exploration` — `bInCombat` may still be true. `CS_Exploration` is entered only after `bInCombat` → false.

**E3 — All Enemies Die Simultaneously**

`OnTargetDeath` and all remaining enemy deaths fire in the same tick. Camera transitions to `CS_Combat` per E2, then to `CS_Exploration` when `bInCombat` → false fires. Both transitions may resolve within one frame; blend times prevent visual discontinuity.

**E4 — `bInCombat` Fires While `CS_AimBow` Active**

Player draws bow during exploration; an enemy triggers `bInCombat` → true. Camera stays in `CS_AimBow`. On `IA_AimBow` released: exits to `CS_Combat` (not `CS_Exploration`), because `bInCombat` is now true.

**E5 — `CS_Cinematic` Entry While `Mod_NearDeath` Active**

`Mod_NearDeath` is suspended (paused, not reset) during `CS_Cinematic`. `Alpha` is preserved. On cinematic exit: if health is still below `WoundedThreshold`, modifier resumes from preserved `Alpha`. If `OnWoundedStateExited` fired during the cinematic (campfire rest), the modifier checks its clear condition on exit and deactivates.

**E6 — Multiple Hits in Rapid Succession**

Two `ApplyDamage` calls within the same tick. Hit direction vectors are averaged; combined impulse is the sum of both, capped at `CameraImpulseMagnitude × 2.0 = 6.0 UU`. Spring returns from the combined offset.

**E7 — Spring Arm Collision Compresses Below `MinArmLength`**

Camera clamps at `MinArmLength = 150 UU` — does not compress further. Camera may clip geometry at this extreme. Fix is in level geometry, not the floor value.

**E8 — `OnSaveLoaded` While `CS_LockOn` or `CS_AimBow` Active**

All camera states reset on `OnSaveLoaded`. `CS_LockOn` cleared (no lock target persists across loads). Camera restores to `CS_Exploration`. `Mod_NearDeath` is re-evaluated from loaded `HealthCurrent` — if health is below `NearDeathThreshold` at load time, the modifier activates.

**E9 — Camera During Hard Landing Stumble**

Hard-landing stumble Montage (~0.5 s) plays. No special camera state — camera is in `CS_Exploration` or `CS_Combat` and follows character position through the stumble via spring arm lag. If damage is applied on impact (not MVP), `Mod_HitImpulse` fires as normal.

## Dependencies

**Upstream (Camera System depends on these):**

| System | Dependency | Interface |
|--------|-----------|-----------|
| Character Controller | `bIsSprinting`, `bIsCrouching` booleans (drive FOV bonus and arm adjustment) | Per-state change event |
| Health & Stamina System | `HealthCurrent` float (NearDeath threshold check); `OnWoundedStateExited` delegate (clear `Mod_NearDeath`) | Float read; delegate subscription |
| Input System | `IA_LockOn` hold action (gates `CS_LockOn`); `IA_AimBow` hold action (gates `CS_AimBow`) | Input Action events |
| Combat System | `bInCombat` bool (gates `CS_Combat`); hit direction vector on `ApplyDamage` delegate (`Mod_HitImpulse`) | Bool broadcast; delegate |
| Save/Load System | `OnSaveLoaded` event (resets all camera state) | Event subscription |

**Downstream (depend on Camera System):**

| System | What They Need | Interface |
|--------|---------------|-----------|
| Cinematic/Cutscene System | Camera control handoff via `SetViewTarget`; saved camera state restored on exit | `APlayerController::SetViewTarget()` |

**Cross-system flags:**
- **Input System GDD**: `IA_LockOn` must be added as a `Bool / Ongoing` hold action in `IMC_OnFoot`. Not currently present.
- **Cinematic/Cutscene System GDD**: Use `ULevelSequencePlayer` for sequences; `IMovieScenePlayer` deprecated UE 5.6 — do not subclass.
- **Stealth System GDD**: Camera has no Stealth System interface at MVP. If stealth kill animations or detection-alert transitions need camera responses, define the delegate in the Stealth System GDD.
- **Open World / Hub System GDDs**: All indoor combat spaces must have minimum ceiling clearance of ~300 UU to keep the camera above `MinArmLength`.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|--------------------|
| `ExplorationArmLength` | 475 UU | 450–500 | Boy too prominent; world scale diminished | Boy reads as distant silhouette; combat legibility suffers |
| `CombatArmRatio` | 0.88 | 0.82–0.94 | Combat claustrophobic | Exploration and combat feel identical; mode undifferentiated |
| `AimBowArmRatio` | 0.70 | 0.60–0.80 | Bow arm clips camera view | Target too small to read |
| `CrouchArmDelta` | 28 UU | 20–35 | Boy shrinks in frame while crouching | Arm change jarring on stand-to-crouch |
| `MinArmLength` | 150 UU | 100–200 | Camera clips geometry at tight corners | Too much minimum; camera too far in close spaces |
| `FOV_Exploration` | 80° | 75–85° | Tunnel vision | Depth lost; world flattens |
| `SprintFOVBonus` | 5° | 3–8° | Speed imperceptible | Fish-eye |
| `FOV_AimBow` | 70° | 65–75° | Peripheral enemies invisible | Bow feel imprecise |
| `MaxLockOnRange` | 1000 UU | 800–1200 | Lock-on useless at range | Player locks onto distant archers during stealth |
| `MaxLockBreakRange` | 1300 UU | 1000–1600 | Lock breaks during normal backstep | Lock persists through absurdly distant target |
| `LockOnArmScale` | 0.15 | 0.10–0.20 | Distant enemy exits frame | Arm extends past exploration distance |
| `MaxLockOnOrbitAngle` | 25° | 15–35° | Locked target drifts to frame edge | Camera spins awkwardly around the boy |
| `LockOnTransitionTime` | 0.25 s | 0.15–0.4 | Acquisition snaps; feels digital | Sluggish; player can't reacquire quickly |
| `NearDeathCameraTilt` | 1.5° | 0.5–3.0° | Imperceptible | Reads as cinematic affectation |
| `NearDeathOrbitDrop` | 5 UU | 3–8 UU | Signal too subtle | Orbit feels broken |
| `CameraImpulseMagnitude` | 3.0 UU | 1.5–5.0 | Hits feel weightless | Reads as screen shake; breaks tone |
| `CameraImpulseReturn` | 0.18 s | 0.12–0.25 | Displacement lingers | Camera bounces; oscillation visible |
| `SprintFOVBlendIn` | 0.3 s | 0.2–0.5 | FOV snaps on sprint start | Delayed speed read |
| `SprintFOVBlendOut` | 0.5 s | 0.3–0.8 | FOV snaps when sprint exhausts | FOV lingers after sprint ends |
| `CameraLagSpeed_Exploration` | 10.0 | 6–15 | Camera floats ahead of character | Camera nailed to character; no lag feel |
| `CameraLagSpeed_LockOn` | 14.0 | 10–18 | Camera slow to follow mid-combat | Jittery during lock-on orbit |

## Visual/Audio Requirements

The Camera System owns no visual or audio assets. It does not play sounds, render overlays, or spawn particles. All visual responses to camera state changes are owned by downstream systems:

- **Near-death audio signal** (`sfx_player_neardeath_catch_01.ogg`) — owned by Health & Stamina System
- **Hit audio response** (grunt, wound exhale) — owned by Health & Stamina System
- **Sprint Recovery breathing** — owned by Character Controller
- **Combat HUD visibility** (health/stamina bars) — owned by Combat & Stealth HUD System

The only camera-visible output is camera position, rotation, and FOV — all driven by the rules in Detailed Design. No screen-space overlays, vignettes, blur, color grading, or chromatic aberration from this system. These are explicitly prohibited per the Health & Stamina GDD governing principle: the governing rule is whether the boy would experience this visually — he does not see the world in red, does not lose vision when wounded, does not see the world pulse.

## UI Requirements

No UI elements are owned by the Camera System. The system exposes no data to any HUD or UI subsystem. If a lock-on target indicator is used (targeting bracket, reticle over the locked enemy), it is owned by the Combat & Stealth HUD, not the Camera System. The Camera System notifies the Combat System of the current locked target; the HUD subscribes from there.

## Acceptance Criteria

| # | Given | When | Then | Pass Condition |
|---|-------|------|------|----------------|
| AC-1 | Game loads at any chapter | Player observes the boy in the steppe | Boy occupies a consistent fraction of the frame | Boy appears correctly small against the steppe — not centered and dominant |
| AC-2 | Player sprints for 3+ seconds | Observe FOV | FOV widens by approximately 5° | Perceptible but not dramatic; no fish-eye; arm length unchanged |
| AC-3 | Player crouches | Camera adjusts | Boy occupies same relative frame size as when standing | Arm shortens ~28 UU; pitch unchanged |
| AC-4 | `bInCombat` becomes true | Observe camera transition | Arm shortens 10–15%, FOV narrows ~3° | 0.4 s smooth blend; no snap |
| AC-5 | Player holds `IA_LockOn` near an enemy | Camera enters lock-on | Camera orbits to show both boy and enemy in frame | Target within inner 30% of frame width; arm extends with enemy distance |
| AC-6 | Locked enemy at 45° to player's right | Observe orbit | Camera adjusts orbit to keep target in frame | Target visible; orbit does not exceed 25° |
| AC-7 | Locked target dies, one other enemy in range | Target death | Camera auto-acquires next valid target | New lock-on within 0.25 s; no manual re-press required |
| AC-8 | Locked target dies, no other enemies | Target death | Camera transitions to `CS_Combat` | Does not pull back to exploration; arm/FOV at combat values |
| AC-9 | Player holds `IA_AimBow` | Camera enters bow aim | Arm ~333 UU, FOV ~70°, camera left of right shoulder | Right-side sight line opens; `IA_LockOn` rejected during aim |
| AC-10 | Health drops below 10 HP | Observe camera | Dutch tilt 1.5° applied over 1.5 s; orbit lowers 4–6 UU | One-shot on crossing; no pulsing; no re-trigger on subsequent hits |
| AC-11 | Player takes a hit from the right | Observe camera impulse | Camera displaces ~3 UU left, returns in ~0.18 s | No oscillation; returns cleanly; direction matches hit-opposite |
| AC-12 | Campfire rest restores health above 30 HP | Observe camera after rest | Dutch tilt clears | Tilt persists at 11 HP; only clears at campfire completion |
| AC-13 | Cinematic triggers during lock-on | Observe camera | CS_Cinematic takes control; lock-on cleared | On exit: CS_Combat (if in combat) or CS_Exploration; no lock-on restoration |
| AC-14 | Save and reload at any state | Check camera after load | Camera in CS_Exploration | Lock-on cleared; AimBow cleared; Mod_NearDeath re-evaluated from loaded health |

## Open Questions

**OQ-1 — `IA_LockOn` Input System update** *(Blocking for implementation)*
`IA_LockOn` is not in the Input System GDD. Must be added before camera implementation begins. Recommended spec: `Bool / Ongoing` hold action, bound in `IMC_OnFoot` only. Not available in `IMC_AimBow` (Model A decision) or `IMC_Mounted`.

**OQ-2 — Lock-on target indicator**
Whether to display a visual indicator over the locked target is a Combat & Stealth HUD decision. If used: minimal — a small bracket at most, consistent with the game's anti-melodrama philosophy. The Camera System does not own this element.

**OQ-3 — Stealth camera response**
The Stealth System GDD (Priority 7) should define whether the camera needs any response to detection state changes (e.g., hold-still during stealth kill animations, micro-adjustment on `Alerted` transition). No interface is defined here. Flag for Stealth System GDD.

**OQ-4 — Horse System camera**
The Camera System has no `CS_Mounted` state. Mounted camera behavior (wider FOV, longer arm for horse scale, gallop FOV adjustment) will be specified in the Horse System GDD. The `UCameraModifier` stack architecture is compatible with a mounted modifier without requiring structural changes.

**OQ-5 — Lock-on during bow aim in later chapters**
Model A (lock-on melee-only) was chosen for MVP. The Tigrahauda armor ability ("Взгляд сокола" — falcon's gaze) may create a case for bow aim assist without full lock-on. Flag for Ability System GDD when designing that fragment's ability.
