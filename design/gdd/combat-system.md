# Combat System

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-11
> **Implements Pillar**: Pillar 3 (Culminating Battle — combat is the vocabulary of the final confrontation) + Pillar 2 (Personal Journey — survival, not domination)

## Overview

The Combat System owns all offensive and defensive interactions between the boy and enemies. It manages two domains: melee (light attack, heavy attack, block, parry) and ranged (bow draw, aim, arrow release). It owns the `bInCombat` flag — the boolean that gates stamina recovery in Health & Stamina, drives the music state in Audio, enables Combat Idle in the Character Controller, and tells the Stealth System the encounter is live. All damage output from the player routes through `UHealthStaminaComponent::ApplyDamage()` on enemies; all damage intake from enemies routes through the same interface on `AAltynCharacter`. Hit detection is animation-timed — active windows are defined by `UAnimNotify` events on attack Montages, not by per-tick polling. The combo system sequences light attacks (up to 3 in a chain) and gates the heavy attack behind a stamina cost. Block absorbs incoming hits at a stamina cost per hit. Parry is a narrow timing window that staggers the enemy with no stamina cost — the highest-risk, highest-reward option.

The boy is not a warrior. He has a knife, a bow, and reflexes built from a life in the steppe — not from training. Every combat action carries a cost: stamina spent, a recovery frame the enemy can punish, a window the player chose to open. The bow is the honest answer when the problem is at distance. Melee is the answer when it got close before he was ready. Stamina is the clock that runs on every fight, and when it empties, the options narrow until there is only the next dodge or the death animation. Combat in *Altyn Adam* is a problem the boy solves before it solves him, with whatever he has left.

## Player Fantasy

**"The Body Remembers What the Name Forgot"**

Every fight is a problem solved with insufficient tools.

The boy's hands are quicker than his courage — reflex outrunning thought, the body remembering hunts and wrestling matches from a childhood that no longer exists. He has never trained for war. What he has is the grass-learned patience of something that has always been smaller than what hunts it, and a knife that was meant for skinning.

The fantasy is not power. It is cold, focused improvisation that ends with him still breathing. When he survives something he had no right to survive, the feeling is *I am still here* — not *I am a warrior.* The distinction is the design contract: the mechanics must say what the story says. Stamina empties. Recovery is never free. The option tree narrows until there is only the next choice or the death animation. This is the ludonarrative promise: if the story says he is one mistake from the boy he was yesterday, the systems must agree.

**Bow and knife duality.** The bow is the first answer — uncomfortable predator competence from a distance, deliberate, technical, aimed. The knife is the failure state: combat became close before he was ready, and now everything is contingent and ugly. A player who reaches for melee should feel the cost of that distance collapsing. A player who lands a clean bow kill at range should feel that the steppe taught him something that cannot be named.

The final battle — Pillar 3 — is the test of this contract. Everything the player has learned about survival-not-mastery gets one last argument. Combat in *Altyn Adam* does not graduate the boy into a warrior. It brings him to the threshold of the last thing he has to do.

## Detailed Design

### Core Rules

**Rule CS-1 — Architecture**
The Combat System is implemented as `UCombatComponent : UActorComponent`, owned by `AAltynCharacter` via `UPROPERTY`. All combat state, input gating, and action dispatch are owned by this component. No other system writes combat state directly. External systems read via const accessors or subscribe to delegates.

**Rule CS-2 — bInCombat Ownership**
`bInCombat` is a private bool on `UCombatComponent`, written exclusively via `EnterCombat()` and `ExitCombat()`. Broadcast to all subscribers via `OnCombatStateChanged(bool)` delegate on each transition. No external system sets it.

**Rule CS-3 — bInCombat: Entry Triggers**
`EnterCombat()` fires when any of the following occur:

| Trigger | Source |
|---------|--------|
| Any enemy enters `Alert` state within `CombatActivationRange = 1500 UU` | Stealth System `OnAwarenessChanged(Alert)` → Combat subscribes |
| Player executes any melee attack | Combat System — `IA_Attack` or `IA_HeavyAttack` consumed |
| Player holds `IA_AimBow` while any enemy is at Alert within range | Combat System — proximity check on aim entry |
| Player takes damage | H&S `ApplyDamage()` fires → Combat subscribes |

Combat starts when the situation becomes combat — not when the first blow lands. The enemy committing to Alert is the point of no return.

**Rule CS-4 — bInCombat: Exit (Post-Combat Window)**
`ExitCombat()` fires only when ALL three conditions are simultaneously true:
1. No enemy is in `Alert` or `Searching` state (Stealth System aggregate tier is `Suspicious` or lower)
2. No attack Montage is executing (Combat state is `CS_CombatIdle`)
3. `PostCombatWindow = 5.0 s` timer has elapsed since condition 1 became true

If any enemy re-enters `Alert` before the timer expires, the timer resets. `ExitCombat()` broadcasts `bInCombat = false`, begins Camera blend to `CS_Exploration`, allows H&S stamina recovery on next tick, and pops Audio Combat mix.

**Rule CS-5 — Damage Interfaces**
All damage output from player routes through `IHealthInterface::ApplyDamage(float Amount)` on the hit actor — not through direct component cast. This interface must be defined before any enemies are implemented. All damage intake from enemies routes through the same interface on `AAltynCharacter`. H&S is the single write surface for both `HealthCurrent` and `StaminaCurrent`.

**Rule CS-6 — Hit Detection: AnimNotify Guard**
Hit detection windows are timed by `UAnimNotify` point notifies on attack Montages (not `UAnimNotifyState`, not per-tick polling). On each notify callback, `UCombatComponent` executes a `SweepSingleByChannel` against `ECC_Pawn`.

Critical guard: the component state — not the Montage — is the authority for whether a hit should register. If the notify fires while the component is NOT in an attack-active state (e.g., Montage was interrupted by stagger or dodge on the same frame), the sweep result is discarded. This prevents ghost hits from interrupted attacks.

---

**Rule CS-7 — Light Attack: Combo Chain**
Up to 3-hit chain. Each hit requires a new `IA_Attack` press within `ComboInputWindow`.

| Hit | Montage | Hit Frames | Damage Multiplier |
|-----|---------|-----------|-------------------|
| LA1 | `AM_KnifeSwing_L1` | 8–14 | 1.0× base |
| LA2 | `AM_KnifeSwing_L2` | 8–14 | 1.0× base |
| LA3 (Finisher) | `AM_KnifeSwing_L3` | 10–18 | 1.4× base |

`ComboInputWindow = 0.55 s` (tunable 0.4–0.8 s). Window opens at frame 8 of each Montage. `IA_Attack` pressed before frame 8 is queued one frame. Window expiring without input → combo resets, transitions to `CS_CombatIdle`.

**Rule CS-8 — Light Attack: Blocked Entry States**
`IA_Attack` is silently discarded while CC is in: Sprint, Crouch, Airborne, Dodge (`bIsDodging == true`), HardLanding stumble Montage. Sprint breaks first on `IA_Attack` press (CC GDD Rule 3) — the combat action fires on the next frame after sprint state exits.

**Rule CS-9 — Combo Reset Conditions**
Combo counter resets to 0 on: `ComboInputWindow` expiry; player takes damage; `bIsDodging` becomes true; `IA_Block` is held (block entry); `IA_AimBow` held (bow immediately exits melee).

---

**Rule CS-10 — Heavy Attack: Stamina Gate**
At `IA_HeavyAttack` press:
- If `StaminaCurrent ≥ HeavyAttackCost (25.0)`: execute `AM_KnifeSlam_H1`, immediately call `DrainStamina(25.0)`
- If `StaminaCurrent < 25.0`: input silently rejected — no animation, no UI feedback. The body does not have it.

`IA_HeavyAttack` within `ComboInputWindow` of LA1 or LA2 branches out of the combo (combo counter resets). Heavy attack is always an interrupt, never a combo extension.

| Montage | Wind-up | Hit Window | Recovery | Damage | Stamina Cost |
|---------|---------|------------|----------|--------|--------------|
| `AM_KnifeSlam_H1` | Frames 0–13 | Frames 14–22 | Frames 23–28 + 0.4 s lock | 2.2× base | 25.0 |

14-frame wind-up at 60fps = 233 ms of visible vulnerability before the hit lands. Recovery phase is authored into Montage hold frames — dodge and block available during recovery, attacks are not.

---

**Rule CS-11 — Block: Entry and Cost**
`CS_Blocking` enters on `IA_Block` held ≥ `ParryInputThreshold (0.18 s)`. No stamina cost to raise block — only to absorb a hit. When an incoming hit lands during `CS_Blocking`: `DrainStamina(BlockCostPerHit = 15.0)` is called; `ApplyDamage()` on the player is intercepted and discarded (0 HP damage). Block cannot be entered during an active attack Montage — `IA_Block` is queued until `CS_CombatIdle`.

**Rule CS-12 — Block Break**
If `StaminaCurrent == 0` when a hit lands on an active block: block breaks. Transition to `CS_Stunned` immediately. Player takes full damage on that hit. The arm gives way visually — block animation breaks.

---

**Rule CS-13 — Parry: Input Model**
Parry and block share `IA_Block`, distinguished by hold duration:
- Tap `IA_Block` (press-and-release < `ParryInputThreshold = 0.18 s`) → Parry attempt
- Hold `IA_Block` (≥ 0.18 s) → Block

`ParryInputThreshold = 0.18 s` (tunable 0.12–0.25 s). Once `CS_Blocking` is active, the parry-tap path is closed — the player must release and re-tap on the next attack.

**Rule CS-14 — Parry: Active Window**
On parry input, `UCombatComponent` opens `ParryActiveWindow = 0.22 s`. Window opens immediately on input. If enemy attack lands during this window: parry succeeds. No attack within 0.22 s: parry whiffs, character transitions to `CS_CombatIdle` with no penalty.

**Rule CS-15 — Parry: UE 5.7 Implementation Guard**
`bIsParryWindowOpen` is mirrored by a `FTimerHandle` that force-closes it after `ParryActiveWindow + 0.05 s`. Additionally, `OnMontageBlendingOut` (with `bInterrupted == true`) clears `bIsParryWindowOpen` immediately. This is required because `UAnimNotifyState::NotifyEnd()` is not guaranteed to fire on Montage interruption in UE 5.5–5.7. The timer is the authoritative closer; the Montage notify is advisory only.

**Rule CS-16 — Parry: Successful Effects**
On successful parry:
- Player: 0 stamina cost, 0 damage, additive flash Montage (`AM_Parry_Success`, 8 frames)
- Enemy: Combat System fires `OnEnemyParried(AAltynEnemy*)` delegate; Enemy AI handles the stun state (`EnemyParryStunDuration = 1.4 s` provisional — **flag for Enemy AI GDD**)
- "Staggers the enemy" = Enemy AI BehaviorTree enters `Stunned` blackboard state, blocking all attack/movement/ability tasks for the stun duration. Combat System fires the event; Enemy AI owns the stun implementation entirely.

---

**Rule CS-17 — Dodge Ownership (Resolves H&S OQ-1)**
Dodge stamina drain is owned by the **Character Controller**, applied unconditionally regardless of `bInCombat`. When `IA_Dodge` is consumed, CC calls `DrainStamina(DodgeCost = 10.0)` at Montage start. The Combat System reads `bIsDodging` from CC to block attack input during dodge frames. Combat System never owns, observes, or calls anything related to dodge stamina.

Rationale: dodge is a movement action that predates combat. A dodge outside combat costs the same as a dodge inside it — making it free outside combat creates an exploitable i-frame entry pattern.

---

**Rule CS-18 — Bow: Input Model**
Two-step input. `IA_AimBow` held → enters `CS_AimBow`, draws bow. `IA_Fire` triggered (separate button, bound only in `IMC_AimBow`) → fires arrow. Arrow does NOT fire on release of `IA_AimBow`. Player can enter full draw and choose not to fire. The bow rewards patience — it requires a deliberate second input.

**Rule CS-19 — Bow: Draw Time**

| Draw State | Duration | Damage Multiplier |
|------------|----------|-------------------|
| Partial draw | 0–0.4 s | 0.5× base |
| Full draw | ≥ 0.4 s | 1.0× base |
| Overdraw | ≥ 2.4 s | 1.0× (no bonus) |

`MinDrawTime = 0.4 s` (tunable 0.25–0.6 s). No stamina cost on draw. No overdraw penalty beyond time spent. Draw animation communicates state: arm tension increases through the window, enters a held idle at full draw.

**Rule CS-20 — Bow: Arrow Flight**
Arrows are physics-simulated projectiles — not hitscan. `AArrowProjectile` spawns at fire with `ProjectileGravityScale = 0.35` (tunable 0.2–0.5). Arc is shallow at close range, pronounced at long range. Hit via overlap against `ECC_Pawn` → calls `IHealthInterface::ApplyDamage()` on the hit actor.

Rationale: hitscan makes the bow trivially accurate at any range. Arc rewards the steppe-archer's patience at distance and punishes panicked close-range shots with a 0.5× damage partial-draw penalty.

**Rule CS-21 — Bow: Movement and Lock-On**
Player can move during `CS_AimBow` at `105 UU/s` (Input System Rule 4: ×0.35 scalar). Sprint unavailable during aim. `IA_LockOn` unavailable in `CS_AimBow` (Camera GDD Rule 7). On `CS_AimBow` entry: Combat System calls `UCameraModifier_LockOn::ClearTarget()` and broadcasts `OnBowAimChanged(true)` — Camera subscribes and enters its own `CS_AimBow` transition. No shared state enum between Combat and Camera systems.

**Rule CS-22 — Arrow Count (Provisional)**
`MaxArrows = 20` provisional. On fire: decrement. At 0: `IA_Fire` rejected. No regeneration during combat. **Flag: requires Inventory/Resources GDD.**

---

### States and Transitions

`UCombatComponent` maintains a private `ECombatState` enum. All transitions execute via `SetCombatState(ECombatState)`, which validates against the legal transition table before committing.

| State | Entry | Exit | Inputs Accepted |
|-------|-------|------|----------------|
| `CS_Idle` | Load; `ExitCombat()` | `EnterCombat()` → `CS_CombatIdle` | All movement; no combat |
| `CS_CombatIdle` | `EnterCombat()`; any action Montage completes; dodge completes while `bInCombat` | `ExitCombat()` → `CS_Idle`; any combat input | All combat inputs; dodge; movement |
| `CS_LightAttack` (LA1/LA2/LA3) | `IA_Attack` from `CS_CombatIdle` or within `ComboInputWindow` | Montage completes / window expires → `CS_CombatIdle`; next `IA_Attack` in window → next LA; `IA_HeavyAttack` in window (stamina OK) → `CS_HeavyAttack`; damage → `CS_Stunned` | Chain attack; heavy branch; dodge |
| `CS_HeavyAttack` | `IA_HeavyAttack` from `CS_CombatIdle` or as combo branch; stamina ≥ 25 | Montage + recovery → `CS_CombatIdle`; damage during active frames → `CS_Stunned` | Dodge (recovery phase only); block (recovery phase only) |
| `CS_Blocking` | `IA_Block` held ≥ 0.18 s from `CS_CombatIdle` | `IA_Block` released → `CS_CombatIdle`; stamina = 0 on hit → `CS_Stunned`; `IA_Dodge` → `CS_CombatIdle` | Dodge; nothing else |
| `CS_Parrying` (Active / Recovery) | `IA_Block` tap < 0.18 s from `CS_CombatIdle` | Active window expires (0.22 s) → `CS_CombatIdle`; successful parry → Recovery → `CS_CombatIdle`; hit during whiff → `CS_Stunned` | Nothing during active window |
| `CS_AimBow` (Drawing / FullDraw / Releasing) | `IA_AimBow` held from `CS_Idle` or `CS_CombatIdle` | `IA_AimBow` released → `CS_CombatIdle`/`CS_Idle`; `IA_Fire` → Releasing → prior state; damage → `CS_Stunned` | `IA_Fire`; `IA_AimBow` release; dodge (exits bow, dodges) |
| `CS_Stunned` | Damage mid-attack; block break | `StunDuration (0.6 s)` elapses → `CS_CombatIdle` | Nothing — all combat inputs rejected |
| `CS_Dead` | `OnPlayerDeath` (H&S) | `OnSaveLoaded` (checkpoint) → `CS_Idle` | All input blocked |

**Legal transition matrix:**
```
CS_Idle         → CS_CombatIdle
CS_CombatIdle   → CS_LightAttack, CS_HeavyAttack, CS_Blocking, CS_Parrying, CS_AimBow, CS_Dead
CS_LightAttack  → CS_LightAttack (chain), CS_HeavyAttack (branch), CS_CombatIdle, CS_Stunned, CS_Dead
CS_HeavyAttack  → CS_CombatIdle, CS_Stunned, CS_Dead
CS_Blocking     → CS_CombatIdle, CS_Stunned, CS_Dead
CS_Parrying     → CS_CombatIdle, CS_Stunned, CS_Dead
CS_AimBow       → CS_CombatIdle, CS_Idle, CS_Stunned, CS_Dead
CS_Stunned      → CS_CombatIdle
CS_Dead         → CS_Idle
```
Any transition not listed is illegal and silently rejected.

`CS_Dead` and the CC death state activate simultaneously via `OnPlayerDeath` — both systems subscribe independently to the same H&S event and do not write to each other.

---

### Interactions with Other Systems

**Health & Stamina System**

| Data | Direction | When |
|------|-----------|------|
| `StaminaCurrent` (read-only float) | H&S → Combat | At `IA_HeavyAttack` press; at block-break check |
| `DrainStamina(25.0)` | Combat → H&S | Heavy attack execution |
| `DrainStamina(15.0)` | Combat → H&S | Per absorbed block hit |
| `IHealthInterface::ApplyDamage()` on enemies | Combat → Enemy H&S | Hit window AnimNotify fires with valid sweep hit |
| Incoming `ApplyDamage()` on player — intercepted | Enemy → intercepted by Combat | If `CS_Blocking` active and stamina > 0: discard. Else: forward to player H&S |
| `OnPlayerDeath` delegate | H&S → Combat | Enters `CS_Dead`; freezes all input gating |
| `OnSaveLoaded` delegate | Save/Load → Combat | Resets to `CS_Idle`, `bInCombat = false` |
| `OnPlayerHit(FVector HitDirection)` | Combat broadcasts | On any `ApplyDamage()` landing on player — Camera subscribes for `Mod_HitImpulse` |

*Resolves H&S OQ-1 (dodge drain → CC), OQ-2 (Heavy 25.0, Block 15.0, Dodge 10.0 confirmed), OQ-5 (campfire gate: `IA_Interact` on campfire checks `GetbInCombat()` before returning true).*

**Character Controller**

| Data | Direction | When |
|------|-----------|------|
| `OnCombatStateChanged(bool)` | Combat → CC | `EnterCombat()` / `ExitCombat()` — enables/disables Combat Idle + dodge from combat |
| `bIsDodging` (read-only bool) | CC → Combat | Per `IA_Attack` / `IA_HeavyAttack` press — blocks input during dodge frames |
| `bIsCrouching`, `MovementMode` (read-only) | CC → Combat | Per attack input — guard checks |
| Sprint break on `IA_Attack` | CC handles (CC GDD Rule 3) | CC owns this; Combat System does not |
| `bIsInHardLanding` (read-only bool) | CC → Combat | Rejects attack input during stumble Montage (~0.5 s) |

**Audio System**

| Data | Direction | When |
|------|-----------|------|
| `bInCombat = true` | Combat → Audio | `EnterCombat()` → push Combat Alertness mix |
| `OnDealDamage` / `OnTakeDamage` delegates | Combat broadcasts | Audio `UAudioStateManager` subscribes → first event triggers Combat music state |
| `bInCombat = false` | Combat → Audio | `ExitCombat()` → pop mix; begin Combat→Calm crossfade (8.0 s); start 4 s countdown to post-combat dombra exhale |
| SFX trigger calls | Combat → Audio | `sfx_bow_draw_01`, `sfx_bowstring_release_01`, `sfx_arrow_flight_01` at Montage keyframes |

**Stealth System**

| Data | Direction | When |
|------|-----------|------|
| `OnAwarenessChanged(Alert)` within `CombatActivationRange` | Stealth → Combat | Triggers `EnterCombat()` |
| Awareness drops below Alert | Stealth → Combat | Starts `PostCombatWindow` timer for `ExitCombat()` |
| `OnEnemyDeath(AAltynEnemy*)` | Combat broadcasts | Stealth `UAltynStealthSubsystem` deregisters dead enemy from awareness aggregate |

**Camera System**

| Data | Direction | When |
|------|-----------|------|
| `OnCombatStateChanged(bool)` | Combat → Camera | `CS_Exploration` ↔ `CS_Combat` blend |
| `OnBowAimChanged(bool)` | Combat broadcasts | Camera enters/exits `CS_AimBow` |
| `OnPlayerHit(FVector)` | Combat → Camera | Camera `Mod_HitImpulse` receives hit direction |
| Lock-on clear on `CS_AimBow` entry | Combat → Camera | `ClearTarget()` on `UCameraModifier_LockOn` |

**Save/Load System**

| Data | Direction | When |
|------|-----------|------|
| `OnCombatEncounterStart()` | Combat broadcasts | Save System subscribes → writes combat checkpoint synchronously before `bInCombat` flips true |
| Auto-save suppression | Combat → Save | `UAltynSaveSubsystem` checks `GetbInCombat()` before writing primary slot |
| `OnSaveLoaded` | Save → Combat | Resets to `CS_Idle`, clears all combat state |

---

**Cross-system flags from Section C:**
1. **Input System GDD**: Confirm `IA_Block` supports hold-duration discrimination (tap < 0.18 s = parry, hold ≥ 0.18 s = block) via a single Bool action in `IMC_OnFoot`, or whether `IA_Parry` needs to be a separate action.
2. **H&S GDD**: Add `UCombatComponent::OnPlayerHit(FVector)` as the Camera hit impulse source delegate (does not modify `ApplyDamage()` signature).
3. **Save/Load GDD**: Add `OnCombatEncounterStart()` as the trigger for synchronous combat checkpoint write.
4. **Enemy AI GDD** (future): `EnemyParryStunDuration = 1.4 s` provisional; Enemy AI owns `Stunned` BT state implementation.

## Formulas

**F1 — Melee Damage**

```
MeleeDamage = LightDamage_base × DamageMultiplier

LightDamage_base = 25.0 HP
DamageMultiplier:
  LA1, LA2       = 1.0
  LA3 (Finisher) = 1.4
  Heavy attack   = 2.2
```

| Action | Formula | Output |
|--------|---------|--------|
| Light attack (LA1/LA2) | `25.0 × 1.0` | **25.0 HP** |
| Finisher (LA3) | `25.0 × 1.4` | **35.0 HP** |
| Heavy attack | `25.0 × 2.2` | **55.0 HP** |
| Full 3-hit combo | `25 + 25 + 35` | **85.0 HP** |

*Provisional — calibrate against enemy HP values when Enemy AI GDD is authored.*

---

**F2 — Bow Damage**

```
ArrowDamage = ArrowDamage_base × DrawMultiplier

ArrowDamage_base = 25.0 HP (provisional — equals LightDamage_base)
DrawMultiplier:
  Partial draw (< MinDrawTime 0.4 s) = 0.5
  Full draw (≥ MinDrawTime)          = 1.0
  Overdraw (≥ 2.4 s)                 = 1.0 (no bonus)
```

| Draw State | Formula | Output |
|------------|---------|--------|
| Partial draw | `25.0 × 0.5` | **12.5 HP** |
| Full draw | `25.0 × 1.0` | **25.0 HP** |

Arrow gravity arc: `ProjectileGravityScale = 0.35` (no damage formula dependency — arc is visual and requires aim compensation at range).

---

**F3 — Block Damage Interception**

```
PlayerDamageTaken = EnemyDamage × (1 − BlockInterception)

BlockInterception:
  CS_Blocking active AND StaminaCurrent > 0  →  BlockInterception = 1.0  →  PlayerDamageTaken = 0
  CS_Blocking active AND StaminaCurrent == 0 →  BlockInterception = 0.0  →  PlayerDamageTaken = EnemyDamage
  Any other state                             →  BlockInterception = 0.0  →  PlayerDamageTaken = EnemyDamage

StaminaAfterBlock = clamp(StaminaCurrent − BlockCostPerHit, 0.0, StaminaMax)
BlockCostPerHit = 15.0
```

Block break condition: `StaminaCurrent == 0` at moment of hit → block breaks, full damage, `CS_Stunned`.

---

**F4 — Parry Window Evaluation**

```
ParrySuccess = (EnemyAttackTimestamp ∈ [ParryWindowOpen, ParryWindowClose])

ParryActiveWindow = 0.22 s
ParryWindowOpen   = timestamp of IA_Block tap input
ParryWindowClose  = min(ParryWindowOpen + ParryActiveWindow, ForceCloseTimer)
ForceCloseTimer   = ParryWindowOpen + ParryActiveWindow + 0.05 s
```

On `ParrySuccess`: `PlayerDamageTaken = 0`, `DrainStamina = 0`, fire `OnEnemyParried(Enemy)`.
On `ParryWhiff`: no cost, transition to `CS_CombatIdle`.

---

**F5 — Heavy Attack Stamina Gate**

```
HeavyAttackAllowed = (StaminaCurrent ≥ HeavyAttackCost)

HeavyAttackCost   = 25.0
StaminaAfterHeavy = StaminaCurrent − 25.0
```

If `HeavyAttackAllowed == false`: input silently rejected, no state change.

---

**F6 — Combo Timing Window**

```
ComboAccepted = (IA_Attack received during [ComboWindowOpen, ComboWindowClose])

ComboWindowOpen  = frame 8 of current Montage ≈ Montage_Start + 0.133 s (at 60fps)
ComboWindowClose = ComboWindowOpen + ComboInputWindow
ComboInputWindow = 0.55 s

EarlyInputQueued = (IA_Attack received before ComboWindowOpen) → queued for 1 frame
LateInput        = (IA_Attack received after ComboWindowClose) → discarded, combo resets
```

---

**F7 — Post-Combat Exit Timer**

```
CombatExitAllowed = (AwarenessAggregateTier ≤ Suspicious)
                    AND (ECombatState == CS_CombatIdle)
                    AND (PostCombatElapsed ≥ PostCombatWindow)

PostCombatWindow  = 5.0 s
PostCombatElapsed = time since CombatExitAllowed first became true
                    (resets if awareness re-enters Alert)
```

---

**F8 — Stun Duration**

```
PlayerStunEnd = CS_Stunned_Entry + StunDuration

StunDuration = 0.6 s (all stun triggers: damage-mid-attack, block break)
```

During `StunDuration`: all combat inputs rejected, state cannot transition.

---

**Example calculations:**

*Scenario A — Player fights a single enemy (100 HP):*
- LA1 (25) + LA2 (25) + LA3 (35) = 85 HP. Enemy at 15 HP — one light attack to finish.
- Heavy (55) + LA1 (25) = 80 HP. Enemy at 20 HP. Stamina spent: 25.0 (25% of budget).
- ~4 heavy attacks before stamina exhaustion in a single encounter with no recovery.

*Scenario B — Player blocks 3 enemy hits (StaminaCurrent = 80):*
- 80 − 15 = 65 → 65 − 15 = 50 → 50 − 15 = 35. Still blocking.
- 4th hit on StaminaCurrent = 5: 5 − 15 < 0 → block breaks, full damage, `CS_Stunned` (0.6 s).

*Scenario C — Bow at partial draw vs. full draw:*
- Panicked fire at 0.2 s: 25.0 × 0.5 = **12.5 HP**. Enemy at 87.5 HP.
- Patient fire at 0.5 s: 25.0 × 1.0 = **25.0 HP**. Same timing cost, double damage.

## Edge Cases

**E1 — Stamina exhaustion mid-combo**
Player executes LA1 (no stamina cost), then attempts heavy attack with `StaminaCurrent < 25.0`. Heavy attack is silently rejected. Combo window is still open — player can continue with LA2 if they press `IA_Attack` within `ComboInputWindow`. The heavy gate does not close the combo.

**E2 — Dodge interrupts combo**
Player is mid-combo (LA1 or LA2 delivered). `IA_Dodge` pressed. Dodge fires immediately — Montage interrupts, combo counter resets to 0, `bIsDodging` becomes true. Combat System blocks attack input for dodge duration. Stamina drain: CC applies `DrainStamina(10.0)` at dodge Montage start. On dodge completion, state transitions to `CS_CombatIdle` (since `bInCombat == true`).

**E3 — Player takes damage during heavy attack wind-up (frames 0–13)**
Player is committed to `CS_HeavyAttack` before the hit window opens. Enemy attack lands. Transition to `CS_Stunned`. Heavy attack Montage aborts. Stamina was already drained (drain fires at Montage start). The stamina cost is paid even if the attack is interrupted — the boy committed the effort.

**E4 — Block with exactly 0 stamina remaining before hit**
`StaminaCurrent == 0.0` when `IA_Block` is pressed. `CS_Blocking` DOES enter (no cost to raise block). Next incoming hit: `StaminaCurrent == 0`, block break fires immediately — player takes full damage and enters `CS_Stunned`. The player sees the block raise and immediately break. Correct: the body attempted it, but there was nothing left to absorb with.

**E5 — Parry Montage interrupted before enemy attacks**
Player taps `IA_Block`, `CS_Parrying` enters, `bIsParryWindowOpen = true`. Before any enemy attack lands, player takes damage from a second enemy. On `OnMontageBlendingOut` with `bInterrupted == true`, Combat System clears `bIsParryWindowOpen` immediately. Force-close timer also fires 0.05 s later. No ghost-parry: player is in `CS_Stunned`, parry window definitively closed.

**E6 — Multiple enemies attack simultaneously during block**
Two enemies land hits in the same frame on an active block. Each hit is processed sequentially. Stamina drains twice. If `StaminaCurrent = 25.0` entering the frame: first hit drains to 10.0 (damage intercepted), second hit evaluates at 10.0 > 0 → drains to 0.0 (damage intercepted). Block holds for both hits. Next hit after this frame: `StaminaCurrent == 0` → block breaks. Being cornered by multiple enemies should feel precarious.

**E7 — Parry during multi-attacker encounter**
`ParrySuccess` fires on the first enemy attack that lands within `ParryActiveWindow`. If two enemies both attack within the window, the first to resolve triggers the parry. The second attack resolves against the now-expired window — player takes full damage and enters `CS_Stunned`. Parry is not a tool for multi-attacker situations.

**E8 — Sprint breaks and IA_Attack fires same frame**
Player holds sprint, presses `IA_Attack`. CC exits sprint on `IA_Attack` input (CC GDD Rule 3). On the same frame: sprint exits, then `IA_Attack` is evaluated. CC state at evaluation is no longer Sprint — attack is accepted. No one-frame delay; the attack fires immediately. Sprint-to-attack transition feels responsive.

**E9 — HardLanding during combat**
Player falls ≥ 150 UU while `bInCombat == true`. CC fires stumble Montage. Combat System reads `bIsInHardLanding == true` from CC and rejects all combat inputs for ~0.5 s. `bInCombat` remains true. After stumble completes, input resumes. HardLanding does NOT trigger `CS_Stunned` — it is CC-owned recovery, not a damage-interrupt.

**E10 — Arrow fired, player dies mid-flight**
`AArrowProjectile` is a spawned actor with its own velocity. `OnPlayerDeath` fires on player. The projectile continues its trajectory — it is not owned by `AAltynCharacter`. If it hits an enemy after the player dies, `IHealthInterface::ApplyDamage()` fires on the enemy. The arrow was already released.

**E11 — IA_Fire pressed at arrow count = 0**
`UCombatComponent` checks `ArrowCount` before `IA_Fire` resolves. At 0: input silently rejected. `CS_AimBow` state remains active. The draw animation switches to the "empty quiver" variant (hand reaches back, finds nothing). Player must release aim to exit.

**E12 — bInCombat rapid toggle (enemy dies immediately after going Alert)**
Enemy goes Alert → `EnterCombat()` fires → player fires arrow → enemy dies instantly. `OnEnemyDeath` fires → awareness aggregate drops to Unaware. All three `ExitCombat()` conditions become true. `PostCombatWindow = 5.0 s` starts. Music Combat state begins 8.0 s crossfade. Net: a very brief encounter, clean recovery. No jarring snap.

**E13 — Death during CS_AimBow**
Player is in `CS_AimBow` at full draw. Enemy attack lands. `OnPlayerDeath` fires. `CS_Dead` entered — bow Montage aborts, no arrow spawned. The commitment cost of draw time is lost.

**E14 — Campfire interact while PostCombatWindow is ticking**
`bInCombat` is still true (PostCombatWindow running). Player approaches campfire. `IA_Interact` received. Campfire `CanInteract()` checks `GetbInCombat()` → true → prompt does not render, interaction blocked. Player cannot rest until combat fully exits. Stamina recovery and resting are not available while the encounter is still live.

## Dependencies

**Upstream (this GDD depends on):**
- `input-system.md` — `IA_Attack`, `IA_HeavyAttack`, `IA_Block`, `IA_AimBow`, `IA_Fire`, `IA_Dodge`; IMC_OnFoot hold-duration discrimination for tap-parry; IMC_AimBow for bow movement scalar
- `save-load-system.md` — `OnSaveLoaded` delegate for combat state reset; auto-save suppression contract; `OnCombatEncounterStart()` checkpoint write trigger
- `audio-system.md` — `UAudioStateManager` subscribes to `OnDealDamage`/`OnTakeDamage`/`OnCombatStateChanged`; bow SFX assets in `Player.Combat` Sound Class
- `character-controller.md` — `bIsDodging`, `bIsCrouching`, `MovementMode`, `bIsInHardLanding` read-only access; `OnCombatStateChanged` delegate enables Combat Idle + dodge; sprint break on `IA_Attack`; `DrainStamina(10.0)` for dodge owned by CC
- `health-stamina-system.md` — `ApplyDamage()` and `DrainStamina()` as the only write surfaces; `StaminaCurrent` read-only gate for heavy attack; `OnPlayerDeath` delegate; `bInCombat` gates stamina recovery
- `camera-system.md` — `CS_Combat`, `CS_AimBow`, `CS_Exploration` states driven by Combat System delegates; `Mod_HitImpulse` receives `OnPlayerHit(FVector)`; lock-on cleared on bow aim entry
- `stealth-system.md` — `OnAwarenessChanged(Alert)` triggers `EnterCombat()`; awareness drop below Alert starts PostCombatWindow; `UAltynStealthSubsystem` deregisters dead enemies via `OnEnemyDeath`

**Downstream (depends on this GDD):**
- `enemy-ai-system.md` — receives `OnEnemyParried` event; owns `Stunned` BT state; defines enemy HP, damage values, and attack timing that calibrate all provisional combat formulas
- `ability-system.md` — builds on `UCombatComponent` to add active abilities; uses `ECombatState` transitions as triggers
- `combat-stealth-hud.md` — reads `HealthCurrent`, `StaminaCurrent`, `bInCombat`, `ECombatState`, arrow count from Combat and H&S systems
- `cinematic-cutscene-system.md` — must disable Combat System input gating during `CS_Cinematic`
- `horse-system.md` — mounted combat extends this system; `UCombatComponent` must support a mounted combat mode flag

## Tuning Knobs

| Knob | Default | Safe Range | Affects |
|------|---------|-----------|---------|
| `LightDamage_base` | 25.0 HP | 15–35 | All melee and bow damage (pivot value) |
| `HeavyAttackCost` | 25.0 stamina | 15–35 | Stamina budget per encounter; how often heavy is usable |
| `BlockCostPerHit` | 15.0 stamina | 10–25 | Number of hits player can absorb before stamina exhaustion |
| `ComboInputWindow` | 0.55 s | 0.4–0.8 s | Combo responsiveness vs. button-mash tolerance |
| `ParryInputThreshold` | 0.18 s | 0.12–0.25 s | How precisely the player must tap to parry; lower = harder |
| `ParryActiveWindow` | 0.22 s | 0.16–0.30 s | Parry timing skill ceiling; lower = harder to land |
| `EnemyParryStunDuration` | 1.4 s | 0.8–2.5 s | Parry reward window; tune against enemy attack frequency |
| `PostCombatWindow` | 5.0 s | 3.0–8.0 s | How long after last Alert enemy before combat state clears |
| `CombatActivationRange` | 1500 UU | 800–2000 UU | How close an Alert enemy must be to trigger bInCombat |
| `MinDrawTime` | 0.4 s | 0.25–0.6 s | Patience requirement for full-damage bow shot |
| `ProjectileGravityScale` | 0.35 | 0.2–0.5 | Arrow arc; lower = flatter trajectory, easier at range |
| `StunDuration` | 0.6 s | 0.4–1.0 s | Enemy exploitation window after player is hit mid-attack |
| `HeavyAttackRecoveryLock` | 0.4 s | 0.2–0.7 s | Post-heavy vulnerability window; higher = more risk |
| `MaxArrows` | 20 | 10–30 | Bow resource pressure; calibrate against resupply cadence |

## Visual/Audio Requirements

**Melee animations (Art/Animation):**
- `AM_KnifeSwing_L1`, `AM_KnifeSwing_L2` — light attack Montages with AnimNotify at frame 8 (hit window open) and frame 14 (hit window close)
- `AM_KnifeSwing_L3` — finisher Montage with AnimNotify at frame 10 and frame 18; visible commitment (wider arc, longer follow-through)
- `AM_KnifeSlam_H1` — heavy attack Montage: 14-frame wind-up must be visually readable as a commitment; AnimNotify at frame 14 and frame 22; hold frames 23–28 + authored recovery
- `AM_Parry_Success` — 8-frame additive flash; brief, sharp — a near-miss exhale, not a triumph
- Block hold: idle blend, weight-forward; the arm holding block looks effortful, not confident

**Bow animations (Art/Animation):**
- `AM_Bow_Draw_Start` — 0.4 s, arm tension increases visibly through partial → full draw
- `AM_Bow_Draw_Hold` — held idle at full draw; subtle breathing micro-movement
- `AM_Bow_Draw_Empty` — identical to Hold but hand reaches back to empty quiver; used when `ArrowCount == 0`
- `AM_Bow_Release` — fire animation; arrow spawns at release frame

**Hit feedback (Art/Animation):**
- Light hit reaction: brief stagger (8–10 frames)
- Heavy hit reaction: full stumble (20–24 frames), visible weight transfer
- Block impact: arm absorbs hit with visible deformation; block break = arm gives way to collapse animation
- `CS_Stunned` stagger: 36-frame window (0.6 s at 60fps); visibly off-balance, not ragdolled

**SFX requirements (Audio):**
- `sfx_bow_draw_01` — draw sound; peaks at full draw hold
- `sfx_bowstring_release_01` — release twang; fires at arrow spawn frame
- `sfx_arrow_flight_01` — arrow whistle; attached to `AArrowProjectile`
- `sfx_arrow_impact_flesh_01` — plays on projectile impact with `ECC_Pawn`
- `sfx_knife_swing_light_01`, `_02` — light attack whoosh variants (randomized)
- `sfx_knife_swing_heavy_01` — heavy attack; heavier air displacement
- `sfx_knife_impact_01` — blade contact; plays at hit window AnimNotify
- `sfx_block_impact_01` — hit absorbed by block
- `sfx_block_break_01` — block break; distinct from absorbed impact
- `sfx_parry_success_01` — sharp deflection; brief, high transient — not a victory sound
- `sfx_stagger_grunt_01` — player vocalization on `CS_Stunned` entry
- All SFX in `Player.Combat` Sound Class under Audio System's 4-level hierarchy

## UI Requirements

*(Full UI specification owned by `combat-stealth-hud.md`. This section defines the data contract.)*

The Combat System exposes the following to the HUD:
- `bInCombat` (bool) — controls HUD visibility; health and stamina bars appear only during combat
- `ECombatState` (enum) — HUD reads for bow aim reticle visibility (`CS_AimBow` = show reticle)
- `ArrowCount` (int) — displayed during `CS_AimBow`; quiver icon or count
- `bIsBlocking` (bool) — HUD may show a brief blocking indicator (confirm in HUD GDD)
- `bIsParryWindowOpen` — no HUD element; parry timing is read from enemy wind-up animation only. A parry prompt would undermine skill-based timing.

The Combat System does NOT own or control HUD visibility, layout, or animations. The HUD subscribes to `OnCombatStateChanged(bool)` to show/hide itself.

## Acceptance Criteria

**AC-01 — bInCombat triggers on enemy Alert**
*Given* an enemy enters `Alert` within 1500 UU, *when* `OnAwarenessChanged(Alert)` fires, *then* `bInCombat == true` and `OnCombatStateChanged(true)` fires once on the same tick.

**AC-02 — bInCombat exits after PostCombatWindow**
*Given* all enemies are below `Alert` and `ECombatState == CS_CombatIdle`, *when* 5.0 s elapses without re-entry, *then* `bInCombat == false` and `OnCombatStateChanged(false)` fires once.

**AC-03 — Light attack combo chains correctly**
*Given* `IA_Attack` from `CS_CombatIdle`, *when* pressed again within 0.55 s of each Montage start, *then* LA1 → LA2 → LA3 execute in sequence. LA3 applies 1.4× multiplier. Pressing after window resets to `CS_CombatIdle`.

**AC-04 — Heavy attack gated by stamina**
*Given* `StaminaCurrent < 25.0`, *when* `IA_HeavyAttack` pressed, *then* no animation plays and `StaminaCurrent` is unchanged. *Given* `StaminaCurrent ≥ 25.0`, Montage fires and stamina decreases by exactly 25.0.

**AC-05 — Block absorbs hits at stamina cost**
*Given* `CS_Blocking` active and `StaminaCurrent > 0`, *when* enemy attack lands, *then* `HealthCurrent` is unchanged and `StaminaCurrent` decreases by exactly 15.0.

**AC-06 — Block breaks when stamina is empty**
*Given* `CS_Blocking` active and `StaminaCurrent == 0`, *when* enemy attack lands, *then* `HealthCurrent` decreases by full enemy damage and player enters `CS_Stunned`.

**AC-07 — Parry succeeds within ParryActiveWindow**
*Given* `IA_Block` tap < 0.18 s, *when* enemy attack lands within 0.22 s of tap, *then* `HealthCurrent` unchanged, `DrainStamina` not called, `OnEnemyParried` fires.

**AC-08 — bIsParryWindowOpen closes on interrupt**
*Given* `bIsParryWindowOpen == true` and parry Montage is interrupted, *then* `bIsParryWindowOpen == false` before the next tick — via `OnMontageBlendingOut` or force-close timer.

**AC-09 — Dodge stamina drains via CC only**
*Given* `IA_Dodge` consumed in any state, *then* `DrainStamina(10.0)` called exactly once by Character Controller; Combat System makes no stamina call.

**AC-10 — Bow damage scales with draw time**
*Given* `IA_Fire` at t = 0.2 s (partial draw), *then* `ApplyDamage` receives 12.5. *Given* `IA_Fire` at t ≥ 0.4 s, *then* `ApplyDamage` receives 25.0.

**AC-11 — Arrow follows physics arc**
*Given* player fires at a target 1500 UU away, *then* the arrow follows a gravity-affected curved trajectory requiring aim compensation above target center (`ProjectileGravityScale = 0.35`).

**AC-12 — Campfire blocked during bInCombat**
*Given* `bInCombat == true`, *when* player presses `IA_Interact` at a campfire, *then* campfire interaction does not trigger.

**AC-13 — CS_Dead on OnPlayerDeath**
*Given* `HealthCurrent` reaches 0, *when* `OnPlayerDeath` fires, *then* Combat System enters `CS_Dead` on the same tick and all combat inputs are blocked until `OnSaveLoaded` fires.

**AC-14 — Attack inputs blocked in illegal states**
*Given* player is in Sprint, Crouch, Airborne, or Dodge, *when* `IA_Attack` pressed, *then* no attack Montage plays and `ECombatState` is unchanged.

## Open Questions

**CS-OQ-1 — IA_Block hold-duration discrimination**
Tap-parry/hold-block design requires `IA_Block` to distinguish tap (< 0.18 s) from hold (≥ 0.18 s) on a single Bool action in `IMC_OnFoot`. Confirm with Input System implementation whether Enhanced Input's `Ongoing` trigger supports this, or whether `IA_Parry` needs to be a dedicated action with its own Gamepad binding.

**CS-OQ-2 — Arrow CCD on fast projectiles**
`AArrowProjectile` uses overlap against `ECC_Pawn`. Confirm whether `bUseCCD == true` is required to prevent tunnelling at close range or through thin enemies. Verify with UE 5.7 Chaos Physics projectile settings before implementation.

**CS-OQ-3 — Crouch ambush kill (post-MVP)**
CC GDD defers crouching ambush kill to the Combat System via `IA_Interact`. Confirmed out of MVP scope. When designed: requires a new input path through `UCombatComponent` bypassing the standard attack state machine and integrating with Stealth System's `Unaware` state detection.

**CS-OQ-4 — Stagger priority during parry flash**
If the player parries one enemy while a second enemy's attack lands, does `CS_Stunned` interrupt `AM_Parry_Success` additive layer? Intended: yes — stagger takes priority. Confirm AnimMontage blend priority in UE 5.7 ABP.

**CS-OQ-5 — Horse System mounted combat interface**
`bInCombat` controls behavior in CC, Audio, Camera, and H&S. Mounted combat may require a separate flag or a `UCombatComponent` extension on `AAltynHorse`. Defer to Horse System GDD; design Combat System delegate architecture to support extension without a rewrite.
