# Input System

> **Status**: In Design
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-08
> **Implements Pillar**: All Three (foundation for all gameplay interaction)

## Overview

The Input System translates physical input — keyboard, mouse, and gamepad — into typed game actions consumed by all other systems. It is built on Unreal Engine's Enhanced Input framework and manages three layers: **Input Actions** (named, typed signals such as `IA_Move`, `IA_Attack`, `IA_Crouch`), **Input Mapping Contexts** (sets of key→action bindings active in a given gameplay mode), and **context switching** (adding/removing contexts as the player transitions between modes: on-foot, mounted, dialogue, kobyz ritual). All gameplay systems bind to Input Actions, never to raw keys. Key remapping is supported via `UEnhancedInputUserSettings`; the system exposes player-mappable actions through the settings menu without touching Mapping Context assets directly.

## Player Fantasy

The steppe does not forgive hesitation. A mounted archer who pulls half a second late loses the shot; a warrior who shifts weight wrong takes the blade. This is the first thing the input system must communicate — that action in this world has weight, that the half-second between intent and response is the gap between a boy and a warrior.

The player does not control a character. They control a becoming. Every mounted charge, every cautious crouch in tall grass, every trembling note drawn from the kobyz strings — these are moments that shape who the unnamed boy is turning into. The input system is the mechanism of that transformation: if it responds with precision and confidence, the player believes in the boy's growing capability. If it falters — if a dodge feels sluggish, if a context switch lags — the sense of becoming falters with it.

The fantasy is not power. It is clarity. The player should feel that they know exactly what the boy will do before they act, and that the world responds honestly to that action. No animation lock that swallows inputs. No gesture that fires the wrong ability. No input that silences intent. The steppe is unforgiving — but the controls must be fair.

## Detailed Design

### Core Rules

**Rule 1 — Input Action Supremacy**
All gameplay systems bind to named Input Actions (`IA_*`), never to raw keys or axis values. Raw key bindings exist only in IMC assets; no gameplay code reads `FKey` directly.

**Rule 2 — Three-Layer Pipeline**
```
Physical Input → UEnhancedInputLocalPlayerSubsystem → Input Action value
Input Action value → UEnhancedInputComponent trigger → game system handler
Game system handler → gameplay system response
```

**Rule 3 — Input Mapping Contexts**

| IMC | Priority | When Active |
|-----|----------|-------------|
| `IMC_OnFoot` | 0 | Default; player on foot |
| `IMC_Mounted` | 0 | Player on horseback — replaces OnFoot |
| `IMC_AimBow` | 1 | Holding aim — layered over current base |
| `IMC_Dialogue` | 2 | Active dialogue (Post-MVP) |
| `IMC_KobyzRitual` | 2 | Kobyz ritual (Post-MVP) |

**Rule 4 — Input Actions (MVP)**

| Action | Type | Consuming System | Notes |
|--------|------|------------------|-------|
| `IA_Move` | Axis2D | Character Controller, Mount System | In `IMC_AimBow`: Scalar modifier ×0.35 for slow-walk during aim |
| `IA_Look` | Axis2D | Camera System | Active in all base contexts |
| `IA_Jump` | Bool/Triggered | Character Controller | Not bound in `IMC_Mounted` |
| `IA_Sprint` | Bool/Ongoing | Character Controller | Hold; becomes gallop in `IMC_Mounted` |
| `IA_Crouch` | Bool/Triggered | Character Controller | Toggle |
| `IA_Dodge` | Bool/Triggered | Character Controller | Only in `IMC_OnFoot` |
| `IA_Attack` | Bool/Triggered | Combat System | Light melee |
| `IA_HeavyAttack` | Bool/Triggered | Combat System | Heavy melee |
| `IA_Block` | Bool/Ongoing | Combat System | Hold-to-block |
| `IA_AimBow` | Bool/Triggered | Input System (IMC switch) | Hold = enter aim; release = exit aim. Does NOT fire the arrow. |
| `IA_Fire` | Bool/Triggered | Combat System (bow) | Bound only in `IMC_AimBow`. Separate from `IA_AimBow` — fires on explicit press, not on release of aim. |
| `IA_Interact` | Bool/Triggered | Interaction System | Context-sensitive |
| `IA_Dismount` | Bool/Triggered | Mount System | Bound only in `IMC_Mounted` |
| `IA_OpenMap` | Bool/Triggered | UI System | Active in all base contexts |
| `IA_Pause` | Bool/Triggered | UI System | Active in all base contexts |

**Rule 5 — Consume Input**
Every binding in `IMC_AimBow` that shadows a binding in the current base context must have `Consume Input = true`. Without this, both IMCs fire for the same physical key in the same frame.

**Rule 6 — Player Remapping**
- Actions are marked `bPlayerMappable = true` in the IMC asset (this field is on the binding, not in C++ structs — those were removed in UE 5.5).
- The Settings Menu reads and writes remap data through `UEnhancedInputUserSettings`, obtained via `ULocalPlayer::GetSubsystem<UEnhancedInputUserSettings>()`.
- Remap data is serialized inside the game's `USaveGame` object. There is no built-in file-write — the game owns persistence.
- IMC assets are never modified at runtime.

### States and Transitions

| State | Active IMCs | Entry Trigger | Exit Trigger |
|-------|------------|---------------|--------------|
| `OnFoot` | IMC_OnFoot | Game start; successful dismount | Mount success; Dialogue start; Kobyz start |
| `Mounted` | IMC_Mounted | Successful mount | Dismount; player death |
| `AimBow_OnFoot` | IMC_OnFoot + IMC_AimBow | Hold `IA_AimBow` on foot | Release `IA_AimBow`; player death |
| `AimBow_Mounted` | IMC_Mounted + IMC_AimBow | Hold `IA_AimBow` mounted | Release `IA_AimBow`; dismount |
| `Dialogue` | IMC_Dialogue (Post-MVP) | Dialogue trigger from world | Dialogue end event |
| `KobyzRitual` | IMC_KobyzRitual (Post-MVP) | Kobyz ritual action | Ritual end; interruption |

**Transition rules:**
- Only one base IMC (`IMC_OnFoot` or `IMC_Mounted`) is active at a time.
- `IMC_AimBow` is always layered; it never replaces the base IMC.
- `IMC_Dialogue` is layered over the current base (blocks combat without removing the base).
- `IMC_KobyzRitual` replaces the current base entirely.
- If a context swap is triggered while a previous swap is in progress, the request is queued for one tick.

### Interactions with Other Systems

| System | Receives from Input | Sends to Input | Interface Owner |
|--------|--------------------|--------------------|-----------------|
| Character Controller | `IA_Move`, `IA_Look`, `IA_Jump`, `IA_Sprint`, `IA_Crouch`, `IA_Dodge` | — | Character Controller binds to IA events |
| Combat System | `IA_Attack`, `IA_HeavyAttack`, `IA_Block`, `IA_Fire` | — | Combat System binds |
| Mount System | `IA_Sprint` (gallop), `IA_Dismount` | Context swap commands (mount/dismount) | Mount System calls `AddMappingContext` / `RemoveMappingContext` |
| Camera System | `IA_Look` | — | Camera binds to `IA_Look` |
| Interaction System | `IA_Interact` | — | Interaction System binds |
| Dialogue System | Navigation, confirm (Post-MVP) | `AddMappingContext(IMC_Dialogue)` on trigger | Dialogue System calls swap |
| Kobyz System | Kobyz-specific actions (Post-MVP) | `RemoveMappingContext(base)` → `AddMappingContext(IMC_KobyzRitual)` | Kobyz System calls swap |
| UI System | `IA_OpenMap`, `IA_Pause` | — | UI System binds |
| Save/Load System | — | Serialized `UEnhancedInputUserSettings` remap data | Save System owns serialization |

## Formulas

The Input System defines no gameplay math (damage, XP, etc.) but specifies quantitative thresholds that directly affect control feel.

**F1 — Aim Slow-Walk Scalar**
```
MoveSpeed_Aim = MoveSpeed_Base × AimScalar
```
| Variable | Value | Range | Effect |
|----------|-------|-------|--------|
| `AimScalar` | 0.35 | 0.1 – 0.6 | Multiplier applied via IMC_AimBow Scalar modifier on IA_Move |
| `MoveSpeed_Base` | defined in Character Controller GDD | — | Input value before scaling |
| `MoveSpeed_Aim` | `MoveSpeed_Base × 0.35` | — | Movement speed while aiming bow |

Example: Full stick input (1.0) during aim produces 0.35 — a slow shuffle, not a freeze.

**F2 — Gamepad Stick Dead Zone**
```
EffectiveInput = (RawStickValue - DeadZone) / (1.0 - DeadZone)   [if |RawStickValue| > DeadZone]
EffectiveInput = 0                                                 [if |RawStickValue| ≤ DeadZone]
```
| Variable | Value | Range | Effect |
|----------|-------|-------|--------|
| `DeadZone` | 0.15 | 0.05 – 0.25 | Below this threshold, stick input is zeroed to prevent drift |

Configured as a Dead Zone Threshold trigger on `IA_Move` and `IA_Look` bindings inside the IMC.

**F3 — Mouse Sensitivity**
```
LookDelta = RawMouseDelta × MouseSensitivity
```
| Variable | Default | Range | Effect |
|----------|---------|-------|--------|
| `MouseSensitivity` | 0.5 | 0.1 – 2.0 | Exposed in Settings Menu; persisted via SaveGame |

Single scalar for MVP (separate X/Y sensitivity not required until post-MVP accessibility pass).

## Edge Cases

**E1 — Death / Interruption While Aiming**
If the player dies or is force-dismounted while `IMC_AimBow` is active, `IMC_AimBow` must be removed before the base context swap occurs. Order: remove `IMC_AimBow` → then perform the base context swap. If this order is violated, `IMC_AimBow` remains active with no base context beneath it, and `IA_Move` fires with an orphaned Scalar modifier.

**E2 — Simultaneous Base Contexts (Bug Guard)**
Only one base IMC (`IMC_OnFoot` or `IMC_Mounted`) may be active at a time. If both are detected active (e.g., due to a double-mount event), the higher-priority context wins and the lower is immediately removed. Log a warning: `"Input: duplicate base IMC detected — removing IMC_OnFoot"`.

**E3 — Input During Context Swap (Queued Request)**
If `IA_AimBow` is pressed while a mount/dismount swap is in progress, the request is queued for one tick. If the base swap completes before the next frame, `IMC_AimBow` is added normally. If the base swap has not completed after two ticks, the aim request is discarded and the player must press again.

**E4 — Gamepad Disconnection Mid-Session**
If the gamepad disconnects while `IMC_Mounted` or `IMC_AimBow` is active, the game treats disconnection as releasing all held buttons (clearing `IA_Sprint`, `IA_Block`, `IA_AimBow` ongoing triggers). The player is frozen until reconnection. Display a "Controller disconnected" UI overlay. The Input System does not crash; it simply stops receiving gamepad events.

**E5 — Keyboard + Gamepad Simultaneously**
Enhanced Input handles mixed device input natively — both sources feed the same Input Actions. Whichever device last sent non-zero input determines which button-prompt icons the UI displays. The Input System does not own device-icon switching; that belongs to the UI System.

**E6 — Remap Conflict (Two Actions on the Same Key)**
`UEnhancedInputUserSettings` allows multiple actions to be mapped to the same key. When this occurs during remapping, the Settings Menu displays a warning: `"[Key] is already used by [ActionName]"`. The remap is allowed — the player's intent is valid. Both actions fire on that key. The Input System does not enforce uniqueness.

**E7 — Aim Cancel During Arrow Nock (Animation Lock)**
If the player releases `IA_AimBow` before the nock animation completes, `IMC_AimBow` is removed immediately on release regardless of animation state. The Combat System is responsible for cancelling the nock animation when it receives `IA_AimBow Completed`; the Input System does not wait for animation completion before processing the context removal.

## Dependencies

**Upstream (Input System depends on these):**

| System | Dependency | Notes |
|--------|-----------|-------|
| UE5 Enhanced Input Plugin | Must be enabled in project — `EnhancedInput` module in `Build.cs` | Engine feature; not a separate game system |
| Save/Load System | Persists `UEnhancedInputUserSettings` remap data via `USaveGame` | Input System provides the data; Save/Load System owns serialization |

**Downstream (these systems depend on Input System):**

| System | What They Consume | Note |
|--------|------------------|------|
| Character Controller | `IA_Move`, `IA_Look`, `IA_Jump`, `IA_Sprint`, `IA_Crouch`, `IA_Dodge` | Their GDD must reference Input System as a dependency |
| Combat System | `IA_Attack`, `IA_HeavyAttack`, `IA_Block`, `IA_Fire` | — |
| Mount System | `IA_Sprint` (gallop), `IA_Dismount`; triggers context swaps | Bidirectional: Mount System also calls Input System to swap IMCs |
| Camera System | `IA_Look` | — |
| Interaction System | `IA_Interact` | — |
| UI System | `IA_OpenMap`, `IA_Pause` | — |
| Dialogue System | `IMC_Dialogue` context (Post-MVP) | — |
| Kobyz System | `IMC_KobyzRitual` context (Post-MVP) | — |

**Bidirectionality note:** Each downstream system's GDD must list the Input System under its Dependencies section. When authoring those GDDs, reference `design/gdd/input-system.md` as the source of the Input Actions they consume.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|-------------------|
| `AimScalar` | 0.35 | 0.1 – 0.6 | Player nearly frozen while aiming — frustrating in mounted combat | Player moves almost at full speed while aiming — trivializes mounted archery skill |
| `DeadZone` | 0.15 | 0.05 – 0.25 | Stick drift visible on older gamepads — camera creeps in menus | Large dead zone makes fine movement feel stepped — snapping into motion noticeably |
| `MouseSensitivity` (default) | 0.5 | 0.1 – 2.0 | Camera sluggish, painful to look around | Camera overshoots targets on casual movement |
| Input queue window | 2 ticks | 1 – 4 ticks | Queued aim press during a dismount is nearly always lost | Stale queued inputs trigger actions the player did not intend |

**Player-exposed settings (Settings Menu):**
- Mouse Sensitivity: slider maps 0.1–2.0, displayed as a 1–10 scale
- (Post-MVP) Gamepad stick sensitivity and dead zone (accessibility pass)
- (Post-MVP) Toggle vs Hold for `IA_AimBow` (accessibility)

## Visual/Audio Requirements

None. The Input System is a Foundation-layer infrastructure system with no direct visual or audio output. All visual feedback for input events (attack animations, aim zoom, mount transitions) is owned by the downstream systems that consume Input Actions.

## UI Requirements

- **Key Remapping Screen** (Settings Menu): Displays all player-mappable Input Actions with current keyboard and gamepad bindings. Allows rebinding by pressing the desired key. Shows conflict warning inline (`"[Key] is already used by [ActionName]"`) — non-blocking.
- **Controller Disconnect Overlay**: Displays a "Controller disconnected — please reconnect" modal when gamepad disconnects mid-session. Blocks gameplay input until reconnected or the player switches to keyboard.
- **Sensitivity Slider**: Mouse Sensitivity exposed as a slider (1–10 scale, maps to 0.1–2.0 internally). Persisted to SaveGame.

## Acceptance Criteria

| # | Criterion | How to Verify | Pass Condition |
|---|-----------|--------------|----------------|
| AC-1 | All gameplay systems bind to `IA_*` actions only | Code review of Character Controller, Combat System, Camera System | Zero occurrences of `FKey` or raw axis reads in gameplay code |
| AC-2 | `IMC_OnFoot` and `IMC_Mounted` are mutually exclusive | QA: mount a horse, check active IMC list via UE debugger (`showdebug enhancedinput`) | Only one base IMC active at any time |
| AC-3 | `IMC_AimBow` layers correctly over `IMC_OnFoot` | QA: aim bow on foot; check active IMC list via debugger | Two IMCs active simultaneously: base + AimBow |
| AC-4 | `IA_Move` Scalar modifier reduces speed to ~35% while aiming | QA: walk full speed, hold aim, measure movement distance per second | Speed in aim mode ≈ 35% of normal walk speed (±5%) |
| AC-5 | No double-fire of `IA_Move` during aim mode | QA: bind a movement counter to `IA_Move`; aim and move; count fires per frame | Exactly one `IA_Move` event per frame during aim, not two |
| AC-6 | Player remapping persists after save/load | QA: remap one action, save game, reload, verify binding | Remapped key is active after reload |
| AC-7 | Gamepad disconnection releases all held inputs | QA: hold sprint, disconnect gamepad; confirm no ongoing sprint after reconnect | No `IA_Sprint` ongoing trigger fires after disconnect |
| AC-8 | Death while aiming does not leave `IMC_AimBow` active | QA: aim bow, trigger player death, check active IMC list after respawn | `IMC_AimBow` is not in active IMC list after respawn |
| AC-9 | Mouse sensitivity setting affects camera speed | QA: set sensitivity to 0.1 and 2.0; compare camera rotation per cm of mouse movement | Sensitivity 2.0 produces ≥10× more rotation than sensitivity 0.1 |
| AC-10 | `IA_Fire` fires the arrow; `IA_AimBow` release only exits aim mode | QA: hold aim, release without pressing fire; confirm no arrow fired | No arrow spawned on aim release; arrow spawned only on explicit fire button press |

## Open Questions

- **Toggle vs Hold for `IA_AimBow`**: Should aim-mode be Hold-to-aim (current default) or optionally toggled? Toggle mode is an accessibility requirement for some players. Defer to post-MVP accessibility pass.
- **Separate X/Y mouse sensitivity**: Single scalar is sufficient for MVP. Separate horizontal/vertical sensitivity is a common PC player expectation — add in post-MVP settings pass.
- **Kobyz input mapping**: How do `IMC_KobyzRitual` inputs map to musical notes? This requires a design decision from the Kobyz System GDD before `IMC_KobyzRitual` can be fully specified.
- **IMC_Dialogue movement scope**: Should `IMC_Dialogue` block only combat inputs, or also movement (`IA_Move`, `IA_Look`)? Depends on whether dialogue plays with the player locked in place or walking. Decision belongs in the Dialogue System GDD.
