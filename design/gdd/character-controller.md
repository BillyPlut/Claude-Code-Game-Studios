# Character Controller

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-09
> **Implements Pillar**: All Three (traversal as presence, movement as identity, cavalry charge as payoff)

## Overview

The Character Controller manages all on-foot movement and physical interaction for the boy — the foundational layer that every gameplay system reads. Built on Unreal Engine's `ACharacter` and `UCharacterMovementComponent`, it defines five movement modes (walk, sprint, crouch, jump, dodge), consumes Input Actions from the Input System, and exposes movement state to the Health & Stamina, Camera, Stealth, and Combat systems. Position and chapter are written to `UAltynSaveGame` and restored on load. The controller does not handle mounted movement — that belongs to the Horse System — but it owns the handoff: dismounting returns control to the Character Controller at the horse's location.

The boy moves like someone who grew up in the steppe — purposeful, quiet, fast when he needs to be. He walks by default, not because the game forces it, but because the world is big enough that walking is the right pace for reading it. Sprint is explosive and brief. Dodge is a survival reflex, not a flashy combo. Crouch is the posture of someone who learned to disappear before he learned to fight. Every movement state is legible to the player through animation — there is no mode the player is in without knowing it.

## Player Fantasy

The steppe is bigger than you. Every movement system in this game exists to make you feel that. You walk because running across this land is for horses, not boys. You sprint and the wind takes the sound of it — eight seconds of effort and then your breath makes the decision before you do. You crouch and the grass closes over you in a way that is briefly a comfort and then a reminder of how easily you disappear. You dodge and live, or you don't — there is no generous margin of error, only the one step that got you clear.

There is no leveling out of this asymmetry. Across six clans and two years of this boy's life, the steppe does not get smaller. At the end of the game you ride into a Persian army and the asymmetry is still the point — you are still one person against a force that has no reason to fear you. What changed is that you chose to be there. The cavalry charge does not feel like power because you unlocked it. It feels like power because you have walked every step of the world that led to it, and you know exactly how far it is from here to where you started.

## Detailed Design

### Core Rules

**Rule 1 — Engine Foundation**

The Character Controller is implemented as `AAltynCharacter : ACharacter`. All movement is driven by `UCharacterMovementComponent` (CMC). No custom movement component subclass at MVP. CMC property names should be verified against UE 5.7 headers before implementation — pre-5.5 names are the reference used here.

---

**Rule 2 — Walk (Default State)**

- `MaxWalkSpeed`: 300 UU/s | `MaxAcceleration`: 1200
- Entry: default state; `IA_Sprint` released; crouch released while moving
- Exit: `IA_Sprint` held + speed > 0; `IA_Crouch` triggered; `IA_Jump`; `IA_Dodge`
- Combat allowed: light melee only
- Locomotion is **turning-in-place** — the character always rotates to face the direction of travel before moving. Free strafing is not a base movement behavior. Backward movement beyond 2–3 steps forces a turn; no sustained backward walk cycle.

---

**Rule 3 — Sprint**

- `MaxWalkSpeed`: 560 UU/s | `MaxAcceleration`: 1400
- Entry: `IA_Sprint` held + `StaminaCurrent > 0` + not crouching
- Exit: `IA_Sprint` released; `StaminaCurrent == 0`; `IA_Dodge`; `IA_Jump`; `IA_Crouch` press; `IA_MeleeAttack` (sprint breaks on attack)
- Combat allowed: No — breaking sprint before attacking is by design

Sprint has three animation phases driven by elapsed sprint time:

| Phase | Duration | Animation state |
|-------|----------|----------------|
| Explosive | 0–5 s | Full stride, arms driving, clean form |
| Degrading | 5–8 s | Stride shortens, head drops, arm rhythm loses regularity |
| Recovery | > 8 s or stamina = 0 | Mandatory Sprint Recovery state: 1.5–2 s, body opens, visible breath cycles |

Sprint Recovery is a distinct animation state — not a blend to Walk. The transition communicates that the body decided to stop.

`StaminaCurrent` is owned and drained by the Health & Stamina System. Character Controller reads the float each tick — when it reaches 0, sprint terminates automatically. Character Controller does not own stamina math or drain rate.

---

**Rule 4 — Crouch (Toggle)**

- `CrouchedHalfHeight`: 60 UU (standing half-height: 88 UU)
- `MaxWalkSpeed` (crouched): 150 UU/s | `MaxAcceleration`: 600
- Entry: `IA_Crouch` toggles on (`AAltynCharacter::Crouch()`)
- Exit: `IA_Crouch` toggles off (`AAltynCharacter::UnCrouch()`)
- Sprint from crouch: No — `IA_Sprint` is ignored while crouched. Sequence: toggle crouch off → stand animation completes (interruptible at frame 8) → sprint accepted
- Sprint + crouch simultaneously: sprint ends first, then crouch activates
- Attack while crouched: No. Crouch ambush kill, if added, belongs to the Combat System via `IA_Interact` — not as a standard attack
- `bIsCrouching` boolean flipped on `IA_Crouch Triggered`. `Crouch()` / `UnCrouch()` called once per toggle, not per-frame

Crouch posture is the weight-forward Kazakh resting squat — not the tactical soldier squat. Stand-to-crouch: 6–8 frames. Crouch-to-stand: head rises first (3–4 frames before spine extension).

---

**Rule 5 — Jump**

- Single jump only: `JumpMaxCount = 1`
- `JumpZVelocity`: 380 UU/s (clears obstacles up to ~75 cm / knee–waist height)
- Air control: `MaxAcceleration` 1000 (air)
- Entry: `IA_Jump` triggered while grounded
- Exit: `bIsGrounded == true` (landing)
- Attack while airborne: No — `IA_MeleeAttack` rejected while `MovementMode == MOVE_Falling`
- Jump arc: CMC-driven. `JumpZVelocity` is a tunable data knob — do not bake it into animation root motion
- Landing recovery: normal landing plays a short absorption animation. Hard fall (≥ 150 UU) plays a distinct stumble Montage with root motion (see Rule 7)

---

**Rule 6 — Dodge**

- Root motion Montage. Distance: 180 UU. Duration: 0.35 s total
- Direction: input-relative — dodge goes where `IA_Move` axis points at the moment `IA_Dodge` fires. If `IA_Move` is zero: dodge backward
- Entry: `IA_Dodge` triggered while in Walk, Sprint, or Combat Idle (`bInCombat == true`, broadcast by Combat System). Sprint breaks immediately on dodge entry
- Entry blocked from: Crouch, Airborne, Interact, `bIsDodging == true`
- Cooldown: `DodgeCooldown = 1.2 s` after Montage completes — `IA_Dodge` rejected during cooldown. No chain-dodging
- Invulnerability (partial): collision capsule shrinks 40% during active travel phase only. Not full invincibility — attacks tracking the reduced capsule can still connect

| Phase | Duration | Hitbox |
|-------|----------|--------|
| Wind-up (frames 0–4) | ~67 ms | Full |
| Active travel (frames 5–14) | ~167 ms | −40% capsule |
| Recovery (frames 15–21) | ~117 ms | Full |

The dodge is a lateral step-aside (1–1.2 body widths). **Not a roll.** Directional variants: left, right (mirrored), forward. No roll animation at MVP.

---

**Rule 7 — Hard Fall**

Fall distance threshold: 150 UU (≈1.5 m), measured on `OnLanded()`.

- Fall distance < 150 UU: standard landing absorption animation, no consequence
- Fall distance ≥ 150 UU: hard landing stumble Montage (root motion, ~0.5 s recovery delay). No HP damage at MVP

`OnMovementNoisePulse` fires at `NoiseLevel = 0.90` on hard landing; `0.55` on soft landing.

---

**Rule 8 — Interact**

- `MaxWalkSpeed`: 0 (character locked while interacting)
- Entry: `IA_Interact` triggered while in range of an `IAltynInteractable`
- Exit: interaction completes or player cancels
- Animation: root motion Montage (character snaps to interaction point)
- Mount prompt (`IA_Interact` near horse): accepted during Walk or Sprint. **Blocked while `bIsDodging == true`**

---

**Rule 9 — Idle Variants**

Three idle states driven by Stealth System enemy awareness:

| Idle | Trigger | Body language |
|------|---------|---------------|
| Calm | No nearby threat | Weight on one hip, slow head scan every 4–6 s, hands loose |
| Aware | Nearby enemy `Searching` | Weight centered, deliberate head scan, arms drop, breath held |
| Tense | Nearby enemy `Alert` or `bInCombat` | Two-footed, arms raised 10–15°, head still, gaze fixed |

Calm → Aware: 20–30 frame blend. Aware → Tense: 8–12 frame blend. Tense → Calm only after Combat System clears `bInCombat` and no enemy remains in `Alert`.

---

**Rule 10 — Horse System Handoff**

**On mount:**
1. `bControllerActive = false`
2. `ACharacter` attaches to horse `Socket_Rider_Pelvis`
3. CMC set to `MOVE_None`
4. All active states cleared (sprint ends, crouch releases, dodge cooldown cancels)
5. `IMC_OnFoot` removed, `IMC_Mounted` added

**On dismount:**
1. Character detaches from socket
2. `SetMovementMode(MOVE_Walking)`, `bControllerActive = true`
3. `IMC_Mounted` removed, `IMC_OnFoot` added
4. Character placed at horse `Socket_Dismount_Left`
5. Character enters **Walk Idle (Calm)** regardless of prior state

Dismount behavior above speed threshold (> 400 UU/s) is owned by the Horse System GDD.

---

**Rule 11 — Movement Noise Signal**

Character Controller fires `OnMovementNoisePulse(float NoiseLevel)` delegate. Stealth System and Audio System subscribe. Not polled per tick.

| State | NoiseLevel | Pattern |
|-------|-----------|---------|
| Crouch Idle | 0.0 | Not fired |
| Crouch Walk | 0.15 | Per footstep event |
| Walk | 0.45 | Per footstep event |
| Sprint | 0.85 | Per footstep event |
| Dodge (active travel) | 0.30 | One-shot at phase start |
| Landing (soft, < 150 UU) | 0.55 | One-shot on `OnLanded()` |
| Landing (hard, ≥ 150 UU) | 0.90 | One-shot on `OnLanded()` |
| Airborne / Interact | 0.0 | Not fired |

Terrain multipliers (grass ×0.6, stone ×1.4, etc.) are applied by the Stealth System, not the Character Controller.

---

**Rule 12 — Position Restore**

On `OnSaveLoaded`: Character Controller reads `PlayerPosition` (FVector) and `ActiveChapter` (FName) from `UAltynSaveGame`, calls `SetActorLocation(PlayerPosition)`. Character spawns at Walk Idle (Calm). Teleport is immediate — no interpolation.

### States and Transitions

| State | Entry | Exit | Attack? |
|-------|-------|------|---------|
| `Walk_Idle` | Default; sprint/dodge/jump ends; dismount; load | `IA_Move` non-zero; `IA_Sprint`; `IA_Jump`; `IA_Dodge`; `IA_Crouch`; `IA_Interact` | Yes (light melee) |
| `Walk_Moving` | `IA_Move` non-zero while not sprinting | `IA_Move` zero; `IA_Sprint`; `IA_Jump`; `IA_Dodge`; `IA_Crouch` | Yes (light melee) |
| `Sprint` | `IA_Sprint` + `StaminaCurrent > 0` + not crouching | `IA_Sprint` released; stamina = 0; `IA_Dodge`; `IA_Jump`; `IA_Crouch`; `IA_Attack` | No |
| `Sprint_Recovery` | Sprint exits while stamina = 0 | Recovery anim completes (1.5–2 s) → `Walk_Idle` | No |
| `Crouch_Idle` | `IA_Crouch` toggled while `IA_Move` zero | `IA_Move` non-zero; `IA_Crouch` off; `IA_Jump` (stand-then-jump) | No |
| `Crouch_Walk` | Crouched + `IA_Move` non-zero | `IA_Move` zero; `IA_Crouch` off | No |
| `Airborne` | `IA_Jump` triggered or walked off edge | `OnLanded()` → `Walk_Idle` or `Hard_Landing` | No |
| `Hard_Landing` | `OnLanded()` + fall ≥ 150 UU | Stumble Montage completes → `Walk_Idle` | No |
| `Dodge` | `IA_Dodge` from Walk, Sprint, or Combat Idle | Montage completes → `Walk_Idle`; cooldown 1.2 s | No |
| `Interact` | `IA_Interact` in range of `IAltynInteractable` | Interaction completes or cancelled → `Walk_Idle` | No |
| `Mounted` | Horse System takes control | Horse System relinquishes → `Walk_Idle` | No |

### Interactions with Other Systems

| System | Provides to Character Controller | Character Controller Provides | Trigger |
|--------|----------------------------------|------------------------------|---------|
| Input System | All IA_* actions (Move, Sprint, Jump, Dodge, Crouch, Interact) | — | Per-frame input events |
| Health & Stamina System | `StaminaCurrent` float (read each tick) | `bIsSprinting` bool; movement state for stamina drain context | Per-tick read |
| Save/Load System | `PlayerPosition`, `ActiveChapter` on `OnSaveLoaded` | `PlayerPosition` for `UAltynSaveGame` write on auto-save | `OnSaveLoaded`; save events |
| Audio System | — | `OnMovementNoisePulse`; movement state for footstep SFX selection | Per-footstep event |
| Stealth System | — | `OnMovementNoisePulse`; idle variant trigger (enemy awareness level read) | Per-footstep event; per-state change |
| Combat System | `bInCombat` bool (broadcast — gates dodge from Combat Idle) | `bIsDodging` bool (blocks mount + attack during dodge) | On combat state change |
| Camera System | — | `bIsSprinting`, `bIsCrouching` (camera adjusts FOV/offset per state) | State changes |
| Horse System | Control transfer on mount; dismount position | Control relinquish; `Socket_Dismount_Left` position | `IA_Interact` near horse; dismount event |

## Formulas

**F1 — Movement Speed Parameters**

```
Walk:         MaxWalkSpeed = 300 UU/s,  MaxAcceleration = 1200
Sprint:       MaxWalkSpeed = 560 UU/s,  MaxAcceleration = 1400
Crouch Walk:  MaxWalkSpeed = 150 UU/s,  MaxAcceleration = 600
Sprint ratio: 560 / 300 = 1.87×
```

| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|--------------------|
| `WalkSpeed` | 300 | 200–400 | Player frustrated by slowness | World feels small; walk loses meaning |
| `SprintSpeed` | 560 | 400–700 | Sprint feels like a jog | Sprint trivializes world scale |
| `CrouchWalkSpeed` | 150 | 80–220 | Stealth traversal painfully slow | Crouch loses distinction from walk |
| `MaxAcceleration_Walk` | 1200 | 800–2000 | Sluggish direction response | Twitchy, feels weightless |
| `BrakingDeceleration_Sprint` | 1000 | 600–1500 | Sprint slides on stop | Sprint stops instantly; no weight |

**F2 — Jump Parameters**

```
JumpZVelocity  = 380 UU/s
JumpMaxCount   = 1
GravityScale   = 1.0  (CMC default)
Max clearance  ≈ 75 cm (knee–waist height)
```

| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|--------------------|
| `JumpZVelocity` | 380 | 300–500 | Can't clear intended obstacles | Floaty; breaks grounded feel |

**F3 — Dodge Parameters**

```
DodgeDistance   = 180 UU  (baked into root motion Montage)
DodgeDuration   = 0.35 s  (total Montage length)
DodgeCooldown   = 1.2 s
HitboxReduction = 40%  (active travel phase only)

Phase timing:
  Wind-up:       ~67 ms  (frames 0–4)
  Active travel: ~167 ms (frames 5–14)
  Recovery:      ~117 ms (frames 15–21)
```

| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|--------------------|
| `DodgeCooldown` | 1.2 s | 0.8–2.0 s | Dodge becomes spammable | Punishing; single miss = death |
| `HitboxReduction` | 40% | 20–60% | Dodge never saves you | Dodge becomes reliable i-frame button |
| `DodgeDistance` | 180 UU | 120–250 UU | Dodge doesn't clear attacks | Dodge becomes a traversal tool |

**F4 — Hard Fall Threshold**

```
HardFallThreshold = 150 UU  (≈1.5 m vertical distance)
```

| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|--------------------|
| `HardFallThreshold` | 150 UU | 100–250 UU | Normal landings trigger stumble | Tall falls have no consequence |

**F5 — Movement Noise Levels**

```
Crouch_Idle   = 0.0  (not emitted)
Crouch_Walk   = 0.15
Dodge         = 0.30 (one-shot at active travel start)
Walk          = 0.45
Landing_Soft  = 0.55 (one-shot on OnLanded())
Sprint        = 0.85
Landing_Hard  = 0.90 (one-shot on OnLanded())
```

Output range: 0.0 (silent) to 1.0 (loudest). Terrain multipliers applied downstream by Stealth System. Not polled — emitted via `OnMovementNoisePulse` delegate.

**F6 — Crouch Capsule**

```
StandingHalfHeight  = 88 UU
CrouchedHalfHeight  = 60 UU
Height reduction    = 32%
```

## Edge Cases

**E1 — Sprint Stamina Depletes Mid-Stride**
Character is sprinting when `StaminaCurrent` reaches 0. Character Controller exits Sprint to Sprint_Recovery on the same tick. No position snap — blend to recovery starts immediately from current velocity. Sprint_Recovery plays 1.5–2 s before Walk_Idle.

**E2 — Dodge Triggered During Sprint_Recovery**
Sprint_Recovery is interruptible by dodge after frame 8 (body has partially settled). If `IA_Dodge` fires before frame 8: dodge is queued and executes at frame 8. If after frame 8: dodge executes immediately.

**E3 — Jump Triggered While Crouching**
`IA_Jump` while crouched triggers stand-then-jump: `UnCrouch()` fires, stand animation begins (interruptible at frame 8), then `Jump()` fires. Character does not jump from the crouched capsule. If ceiling clearance is insufficient for the capsule to expand, stand is blocked and the jump is cancelled silently.

**E4 — Mount Attempted During Dodge**
`IA_Interact` (horse mount) fires while `bIsDodging == true`. Mount is rejected. Player must re-press `IA_Interact` after `DodgeCooldown` clears.

**E5 — Hard Landing During Active Combat**
Player falls ≥ 150 UU during an encounter. Hard_Landing stumble Montage plays (~0.5 s). `IA_MeleeAttack` and `IA_Dodge` are rejected until Montage completes. The player is briefly vulnerable — by design.

**E6 — Dismount at Speed**
Horse System owns the behavior for dismount above 400 UU/s. Character Controller receives control in Walk_Idle regardless of how the Horse System terminates mounted state.

**E7 — OnSaveLoaded While Airborne**
Save loaded mid-jump. `SetActorLocation(PlayerPosition)` teleports character to saved ground position. CMC resets to `MOVE_Walking`. No fall physics carry over from the interrupted airborne state.

**E8 — Dodge Cooldown Expires During Combat**
`DodgeCooldown` timer expires while `bInCombat == true`. `IA_Dodge` re-enables immediately. Dodge cooldown and combat state are independent — no special handling.

## Dependencies

**Upstream (Character Controller depends on these):**

| System | Dependency | Notes |
|--------|-----------|-------|
| UE5 `ACharacter` + `UCharacterMovementComponent` | Movement physics, jump, crouch, fall | Engine API |
| UE5 Animation Blueprint | State machine for locomotion, idles, Montages | Engine API |
| Input System | All IA_* actions (Move, Sprint, Jump, Dodge, Crouch, Interact) | Input System owns bindings; Character Controller owns responses |
| Health & Stamina System | `StaminaCurrent` float (read to gate sprint) | H&S owns stamina math; Character Controller reads only |
| Save/Load System | `PlayerPosition`, `ActiveChapter` on `OnSaveLoaded` | Defined in save-load-system.md |

**Downstream (depend on Character Controller):**

| System | What They Need | Notes |
|--------|---------------|-------|
| Health & Stamina System | `bIsSprinting` bool (sprint context for drain rate) | Character Controller broadcasts on state change |
| Camera System | `bIsSprinting`, `bIsCrouching` (FOV and offset adjustment) | Camera System reads; Character Controller exposes |
| Stealth System | `OnMovementNoisePulse` delegate; idle variant trigger | One-way: Character Controller → Stealth |
| Audio System | `OnMovementNoisePulse`; movement state for footstep SFX | Subscribed to same delegate as Stealth System |
| Combat System | `bIsDodging` bool (blocks attack input during dodge) | Combat System reads; Character Controller owns the flag |
| Horse System | Control transfer contract; `Socket_Dismount_Left` world position | Both GDDs must agree on handoff protocol |
| Open World System | Traversal capabilities define what terrain level design can use | Design constraint, not a runtime dependency |

**Bidirectionality note**: Health & Stamina is both upstream (provides `StaminaCurrent`) and downstream (reads `bIsSprinting`). Both GDDs must list each other.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|--------------------|
| `WalkSpeed` | 300 UU/s | 200–400 | Frustratingly slow | World feels small |
| `SprintSpeed` | 560 UU/s | 400–700 | Sprint feels like a jog | Trivializes world scale |
| `CrouchWalkSpeed` | 150 UU/s | 80–220 | Stealth traversal painful | Crouch loses stealth feel |
| `MaxAcceleration_Walk` | 1200 | 800–2000 | Sluggish direction response | Twitchy, feels weightless |
| `BrakingDeceleration_Sprint` | 1000 | 600–1500 | Sprint slides through stops | No momentum on stop |
| `JumpZVelocity` | 380 UU/s | 300–500 | Can't clear intended obstacles | Floaty; breaks grounded feel |
| `DodgeCooldown` | 1.2 s | 0.8–2.0 | Dodge becomes spammable | Punishing; single miss = death |
| `HitboxReduction_Dodge` | 40% | 20–60% | Dodge never saves you | Reliable i-frame button |
| `DodgeDistance` | 180 UU | 120–250 | Dodge clips attacks | Becomes traversal tool |
| `HardFallThreshold` | 150 UU | 100–250 | Normal landings trigger stumble | Falls never stagger |
| `CrouchedHalfHeight` | 60 UU | 40–70 | May conflict with ceiling geometry | Capsule barely smaller than standing |
| Noise: Crouch Walk | 0.15 | 0.0–0.3 | Stealth trivially silent | Crouch offers no stealth advantage |
| Noise: Walk | 0.45 | 0.3–0.6 | Normal walk never heard | Walk always triggers AI |
| Noise: Sprint | 0.85 | 0.7–1.0 | Sprint is inaudible to AI | — |

**Note on Sprint recovery duration**: Sprint_Recovery animation length (1.5–2 s) is controlled by the animation asset, not a code knob. To tune recovery feel, adjust the Montage playback rate in the Animation Blueprint.

## Visual/Audio Requirements

### Animation Style Direction

The governing principle: **the boy moves like someone who has been watched by the steppe his whole life.** He scans without turning his head. His center of gravity is low and forward. He does not waste motion. Every animation state must communicate this — not as performance, but as habit.

**Walk**: Forward lean from the hips (not the shoulders), weight over the balls of the feet, head level and nearly still, arms hanging and swinging lightly with fingers slightly curled. Low heel strike, short stride. The gaze is slightly down-range — reading the world while moving. NOT: military stride, action-hero chest-out, wide-legged swagger.

**Sprint — three animation phases**:
1. *Explosive (0–5 s)*: full stride, arms driving, chin slightly tucked. Form is controlled but not athletic — this is survival speed, not sport.
2. *Degrading (5–8 s)*: stride shortens, head drops slightly, arm rhythm loses regularity. Degradation accelerates in the final 2 seconds.
3. *Sprint_Recovery (mandatory state)*: body opens, chest expands, 1–2 visible breath cycles, 1.5–2 s before walk cycle resumes. The recovery communicates that the body decided to stop. NOT a snap to walk.

**Crouch**: Weight-forward Kazakh resting squat — **not the tactical soldier squat**. Weight on the balls of the feet, stable and sustainable. Slight forward curl at the upper back; chin slightly down, eyes up. He crouches the way he breathes — it's a thing he does, not a thing he performs. Crouch Idle: micro-movements (weight shift, slow head scan, occasional hand-to-ground touch). Crouch Walk: purposeful foot placement, one hand slightly forward. Stand-to-crouch: 6–8 frames. Crouch-to-stand: head comes up 3–4 frames before spine extension.

**Dodge**: Lateral step-aside (1–1.2 body widths). NOT a roll, NOT a vault. 12–16 frames total. A single explosive lateral step with a weight shift, ending in a low momentary stance before returning to idle or walk. Directional variants: left, right (mirrored), forward step-through. Roll is a polish-phase stretch goal only.

**Jump**: 2–3 frame gather (squat-then-push), then extension upward. Arms come up naturally but do not windmill. The apex has a brief quality of the body finding the landing before it arrives.

**Idle variants**:
- *Calm*: weight on one hip (only state with asymmetric weight), slow head scan every 4–6 s, hands loosely hanging.
- *Aware*: weight centered on both feet, deliberate directional head scan, arms drop fully, breathing becomes less visible.
- *Tense*: two-footed, arms raised 10–15° from body (silhouette widens at shoulder), head still, gaze fixed.

**Age specificity**: Stride length 10–15% shorter than an adult's equivalent speed. Recovery from stumble and direction changes are faster (less mass to redirect). Slightly more upper-body involvement in locomotion. Do NOT animate hesitation, trembling, or scared-looking-around as base behavior — the boy is physically capable; his competence is one of the few things not taken from him.

**Locomotion blend model**: Turning-in-place (not free strafe). 2D blend space: axis 1 = speed (0–2), axis 2 = turn direction (−1 to +1). Backward movement: 2–3 back steps allowed; any further backward travel forces a turn.

### Silhouette Legibility

All states must be readable at 10+ m camera distance without UI indicators:

| State | Key silhouette signal |
|-------|----------------------|
| Walk | Upright, slight forward lean, arms swinging low |
| Sprint | Steep forward lean, arms driving — steepest lean of all states |
| Crouch Idle | Lower + rounded, knees wide, weight-forward squat |
| Crouch Walk | Low profile + foot displacement, narrower than Crouch Idle |
| Jump (apex) | Vertical — only moment character is fully tall |
| Dodge | Horizontal burst, brief low lateral stance |
| Idle Calm | One hip (asymmetric weight — unique to this state) |
| Idle Aware | Two-footed, no hip shift, arms at sides |
| Idle Tense | Two-footed, arms 10–15° raised (shoulder silhouette wider) |

### Audio

Footstep SFX selection is driven by movement state and surface material. Character Controller provides movement state to the Audio System via `OnMovementNoisePulse`. Audio System owns footstep asset selection and SFX playback. Sprint_Recovery requires a distinct audio state (audible breath, heavier footfalls fading into normal walk). Hard Landing fires a distinct impact SFX at `NoiseLevel = 0.90`.

## UI Requirements

No UI elements are required from the Character Controller. Movement modes are communicated entirely through animation — this is confirmed by AC-11 (idle variants) and AC-6 (dodge legibility). There is no on-screen movement mode indicator, stamina bar (owned by Health & Stamina System HUD), or sprint meter. The Character Controller exposes no data to any UI system directly.

## Acceptance Criteria

| # | Criterion | How to Verify | Pass Condition |
|---|-----------|--------------|----------------|
| AC-1 | Walk pace feels appropriate for world scale | QA: walk between two landmarks in Chapter 1 open area | Walking feels deliberate, not frustrating; steppe conveys distance |
| AC-2 | Sprint exhausts in ~8 s of continuous running | QA: sprint on open ground, count seconds | Sprint_Recovery begins at approximately 8 s |
| AC-3 | Sprint Recovery animation is distinct from Walk | QA: exhaust sprint, observe animation | Recovery state visually clear (breath, form break) before clean walk cycle |
| AC-4 | Crouch lowers capsule and enables cover navigation | QA: crouch under a low obstacle the standing character cannot pass | Character passes while crouched; obstacle blocks when standing |
| AC-5 | Dodge cannot be chained (1.2 s cooldown) | QA: spam `IA_Dodge` rapidly | Second dodge fires no earlier than 1.2 s after first |
| AC-6 | Dodge is a step-aside, not a roll | QA: observe dodge animation against melee attack | Character displaces ~180 UU; no roll animation |
| AC-7 | Hard fall triggers stumble, no HP damage | QA: drop from ≥ 1.5 m; observe HP and animation | Stumble Montage plays; HP unchanged |
| AC-8 | Jump height clears knee–waist obstacles, not chest-height | QA: jump over 70 cm rock; attempt 120 cm wall | Rock cleared; wall not cleared |
| AC-9 | Noise delegate fires correct levels per state | QA: attach debug listener; walk, sprint, crouch, dodge | Delegate values match F5 table for each state |
| AC-10 | Position restores from save at correct world location | QA: save at a specific position, reload game | Character spawns at saved position in Walk Idle |
| AC-11 | Idle variants respond to enemy awareness | QA: approach patrol from stealth; observe idle | Calm → Aware transition as enemy enters Searching range |
| AC-12 | Mount prompt blocked during dodge | QA: dodge next to a horse and press `IA_Interact` mid-dodge | Mount does not trigger until after dodge and cooldown complete |

## Open Questions

**OQ-1 — Stamina drain rate and recovery curve** (Provisional: ~8 s to exhaustion)
Exact stamina drain per second and recovery rate are not yet defined. These values will be specified in the Health & Stamina System GDD. The 8 s exhaustion window here is a design target, not a formula. If H&S math produces a different number, this GDD should be updated to match.

**OQ-2 — Motion Matching vs. blend spaces (final choice)**
The Technical Feasibility Brief flagged Motion Matching as MEDIUM risk (post-training-cutoff UE 5.3+). The current spec recommends Motion Matching for from-scratch animation, blend spaces + Nondestructive Anim Layers for asset packs. The final choice depends on the animation production approach confirmed at implementation time. This GDD's functional requirements (silhouette legibility, state transitions) are valid either way.

**OQ-3 — CMC property name verification against UE 5.7 headers**
All `UCharacterMovementComponent` property names used in this GDD (e.g., `MaxWalkSpeed`, `MaxAcceleration`, `CrouchedHalfHeight`, `JumpZVelocity`) are pre-5.5 names. These must be verified against the UE 5.7 API before writing any implementation code. See `docs/engine-reference/unreal/breaking-changes.md`.

**OQ-4 — Horse System dismount speed threshold**
The Character Controller hands off control from mounted to on-foot at `Walk_Idle` regardless of horse speed (Horse System owns all speed-to-dismount behavior). The precise speed threshold at which the Horse System allows dismount, and any stagger or stumble that fires if dismount happens above a safe speed, will be defined in the Horse System GDD.

**OQ-5 — Movement confidence evolution across chapters**
The Player Fantasy section notes the steppe does not get smaller across the game. This means the boy's movement *capability* does not improve — but could his movement *confidence* (idle posture, micro-animations) evolve visually as the story progresses? This is a post-MVP animation layer question, not a mechanics question. Flag for the animation director at Vertical Slice.

**OQ-6 — Climb / ledge-grab ownership**
Jump handles vertical clearance up to ~75 cm. Anything requiring a ledge grab or shimmy (window sills, rooftops, cliff edges) is a separate Interact mechanic, not an extension of Jump. This GDD does not specify climb behavior. Confirm in Combat System or a dedicated Traversal spec whether climbing is in scope for MVP.
