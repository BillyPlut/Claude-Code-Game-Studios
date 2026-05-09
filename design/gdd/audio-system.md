# Audio System

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-09
> **Implements Pillar**: All Three (atmosphere, emotional arc, Kobyz as identity mechanic)

## Overview

The Audio System manages all sound in Altyn Adam — music, ambient, SFX, and dialogue — through Unreal Engine's MetaSounds framework and Sound Class hierarchy. It organizes audio into four classes under a Master bus: **Music** (adaptive layered score), **SFX** (combat, movement, environmental), **Ambient** (wind, animals, settlement), and **Dialogue** (voiced lines and NPC speech). The defining behavior is dynamic music: a MetaSound-driven adaptive score responds to gameplay state, transitioning between a calm exploration layer, a tense suspicion layer, and a full combat layer through real-time parameter changes — not hard cuts. Sound Mixes manage reactive ducking — music fades when dialogue plays; ambient drops during cinematic sequences. 3D spatial audio with custom attenuation governs SFX placement in the world. The Audio System also provides the sound foundation for the Kobyz ritual: individual note assets triggered by input, organized so the Kobyz Music System can sequence them into melodies. Player-facing controls (master volume, music volume, SFX volume) are persisted via the Save/Load System.

## Player Fantasy

The boy lost everything but the songs. Altyn Adam sounds like a culture's instruments — kobyz, dombra, felted drum — carrying meaning that words cannot reach. When the boy plays the kobyz, the player presses each note themselves, and what rises is not a minigame but a small act of remembering: fingers finding melodies his people taught him before they were gone. Outside those moments, the steppe is mostly wind and distance — music waits, present but withheld, until combat brings drums low and bodily, or grief brings a single dombra string under a wide sky. When violence comes, the world does not get louder; it narrows — birds stop, wind thins, the boy's pulse becomes the loudest thing. When it ends, the wind returns and the player exhales with it. And when the cavalry charge finally begins, every instrument the player has heard across the journey returns at once — and the sound is not heroism, but recognition.

## Detailed Design

### Core Rules

**Rule 1 — Sound Class Hierarchy**

All audio routes through a four-class hierarchy under a Master bus:

```
Master
├── Music       — adaptive MetaSound score
├── SFX         — all gameplay sounds
│   ├── Player.Movement   (footsteps, jump, land, roll)
│   ├── Player.Combat     (bow draw, bowstring, sword impacts)
│   ├── Player.Damage     (hit grunts, death)
│   ├── Enemy.Melee       (sword clash, enemy grunts, death)
│   ├── Enemy.Ranged      (arrow whiz, impact)
│   ├── Environment.Interactive (doors, chests, objects)
│   ├── Mount.Horse       (hooves, whinny, gallop)
│   ├── UI.Menu           (confirm, cancel, scroll, ping) — 2D, non-diegetic
│   └── Ritual.Kobyz      (individual note sustains) — 2D, non-diegetic
├── Ambient     — wind, wildlife, environmental loops
└── Dialogue    — voiced NPC lines
```

Sound Class volume is driven by player-exposed settings (Rule 12). No system reads or writes Sound Class volume directly except through the Save/Load System and Sound Mix modifiers.

---

**Rule 2 — Adaptive Score Architecture**

The adaptive score is a single MetaSound asset (`MUS_Adaptive_Score_Main`) with four parameter-driven internal layers:

| Layer | Name | Instruments | Active in |
|-------|------|-------------|-----------|
| 0 | Ambient Drone | Bowed kobyz sustained tone, low wind texture | All states (volume varies) |
| 1 | Melodic Foundation | Dombra arpeggios, sparse kobyz melody fragments | Calm (full), Suspicion (20%), Combat (silent) |
| 2 | Tension Weave | Irregular felted drum pulse, kobyz harmonics | Suspicion (full), Combat (60%) |
| 3 | Rhythmic Drive | Full drum pattern, driving dombra ostinato, full kobyz | Combat (full) |

Three float parameters control layer volumes. The Audio State Manager sets all three simultaneously:

| Parameter | Calm | Suspicion | Combat |
|-----------|------|-----------|--------|
| `MusicState_Calm` | 1.0 | 0.0 | 0.0 |
| `MusicState_Suspicion` | 0.0 | 1.0 | 0.0 |
| `MusicState_Combat` | 0.0 | 0.0 | 1.0 |

A single MetaSound (not multiple crossfading assets) ensures all layers remain phase-synchronized over long play sessions.

---

**Rule 3 — Music State Machine**

The Audio State Manager (`UAudioStateManager : UGameInstanceSubsystem`) owns the state machine. It reads events from other systems and sets MetaSound parameters via `SetFloatParameter`.

**State Triggers:**

| State | Trigger | Source |
|-------|---------|--------|
| Calm | Default state; post-combat dwell elapsed | Internal timer |
| Suspicion | Any enemy's AI awareness reaches `Alert` level | AI System broadcast |
| Combat | First `OnTakeDamage` or `OnDealDamage` event within an encounter | Combat System |

Suspicion is NOT triggered by proximity alone — only confirmed enemy alert state.

**Crossfade Durations:**

| Transition | Duration |
|------------|----------|
| Calm → Suspicion | 3.0 s |
| Suspicion → Combat | 0.8 s |
| Combat → Suspicion | 4.0 s |
| Combat → Calm | 8.0 s |
| Suspicion → Calm | 5.0 s |

**Minimum Dwell (anti-ping-pong):**

| State | Minimum before downward transition |
|-------|------------------------------------|
| Suspicion | 12 s after last alert trigger |
| Combat | 20 s after last damage event AND no enemy in `Alert` state |

---

**Rule 4 — Post-Combat Exhalation Cue**

Four seconds after the Combat → Calm transition begins, a one-shot non-looping dombra phrase (`mus_resolve_postcombat_exhale_01`) plays through the Music Sound Class. Duration: 8–12 seconds. It plays once and ends; only the Calm layer continues beneath it. This is the "wind returns and the player exhales" moment — the transition from combat to calm is never merely silence returning, but a heard event.

---

**Rule 5 — Sound Mixes**

Three Sound Mixes govern dynamic ducking. All targets are multipliers against current class volume (1.0 = no change). Applied via `PushSoundMixModifier` / `PopSoundMixModifier`.

**Mix A — Dialogue Narrative** (named NPCs, cutscene lines):

| Class | Target | Ramp In | Ramp Out |
|-------|--------|---------|---------|
| Music | 0.15 | 0.4 s | 2.0 s |
| Ambient | 0.25 | 0.6 s | 3.0 s |
| SFX | 0.50 | 0.3 s | 1.5 s |

A `bNarrativeDialogueActive` flag is set `true` for the duration. While true, Mix B cannot be pushed.

**Mix B — Dialogue Chatter** (ambient NPCs, background conversation):

| Class | Target | Ramp In | Ramp Out |
|-------|--------|---------|---------|
| Music | 0.60 | 0.8 s | 1.5 s |
| Ambient | 1.0 (no change) | — | — |
| SFX | 1.0 (no change) | — | — |

**Mix C — Combat Alertness** (pushed when entering combat state):

| Class | Target | Ramp In | Ramp Out |
|-------|--------|---------|---------|
| Ambient | 0.20 | 0.4 s | 1.2 s |
| SFX | 1.0 (no change) | — | — |

Music is not touched by this mix — the MetaSound state machine handles it directly.

**Mix D — Cinematic** (Sequencer-driven cutscenes):

| Class | Target | Ramp In | Ramp Out |
|-------|--------|---------|---------|
| Ambient | 0.0 | 0.6 s | 1.0 s |
| SFX | 0.30 | 0.6 s | 1.0 s |

---

**Rule 6 — Kobyz Note Architecture**

Each kobyz pitch is a sustained instrument note with two playback phases:

- **Sustain**: looped while input is held
- **Release tail**: triggered on input release, fades over 0.8 s

Three randomized variants exist per pitch to prevent machine-gun repetition. Asset naming: `sfx_kobyz_note_[pitch]_0[1-3].ogg`.

**Pitch set (MVP — 6 pitches, one per clan):**

| Pitch | Clan | Note | Frequency |
|-------|------|------|-----------|
| Root | Massagetes | C4 | ~262 Hz |
| 2nd | Haumavargas | D4 | ~294 Hz |
| 3rd | Tigrakhauda | E4 | ~330 Hz |
| 5th | Paradaraya | G4 | ~392 Hz |
| 6th | Arimaspys | A4 | ~440 Hz |
| Octave | Parasugaudam | C5 | ~524 Hz |

Scale: C major pentatonic + octave closure. All combinations are musically coherent — no input combination sounds wrong.

**Polyphony**: Maximum 4 simultaneous kobyz voices. When a 5th note is triggered, the oldest active note immediately enters its release tail phase. The new note begins immediately.

**Rapid input**: Notes triggered within 0.1 s of each other crossfade — the previous note enters release tail as the new note starts. No hard cuts.

**Ambient ducking during ritual**: When kobyz ritual state is active:
- Music Layer 0 (Ambient Drone) ducks to 30% over 1.5 s
- Music Layer 1 (Melodic Foundation) ducks to 0% over 2.0 s
- Ambient Sound Class ducks to 40% over 2.0 s

---

**Rule 7 — Melody Completion Response**

When the Kobyz Music System signals that a complete melody was performed (`bMelodyComplete = true` parameter), the MetaSound graph fires two simultaneous events:

1. **Harmonic resonance**: one-shot bowed kobyz chord bloom (`sfx_kobyz_melody_resolve_01.ogg`), 2–3 s, non-looping
2. **Sub-bass presence**: barely perceptible sub-bass rumble, 4 s, fades to silence

No visual UI notification accompanies this — the audio IS the feedback.

---

**Rule 8 — Ambient Layer Behavior**

Six ambient layers exist under the Ambient Sound Class:

| Layer | Description | Behavior |
|-------|-------------|----------|
| Wind Base | Constant steppe wind | Volume driven by `WindIntensity` float (Weather System) |
| Wind Gusts | Irregular high-energy bursts | Random one-shots, 8–45 s interval |
| Distant Wildlife | Birds, insects, mammals off-screen | Spatial, randomized 3D positions |
| Close Wildlife | Animals near player | 3D, triggered by wildlife spawner proximity |
| Settlement Proximity | Fire crackle, human activity, tools | 3D audio volume, proximity-triggered |
| Weather Events | Rain, dust storms | Separate MetaSound graph, layered on Wind Base |

**Ambient response by music state:**

| State | Distant Wildlife | Close Wildlife | Wind Base | Wind Gusts |
|-------|-----------------|----------------|-----------|------------|
| Calm | Full | Full | Full | Active |
| Suspicion | Spawn rate −60% | Unchanged | Full | Active |
| Combat | Mute (0.3 s fade) | Mute (1.0 s fade) | 40% volume | Paused |
| Post-combat | Returns over 15 s | Returns over 15 s | Full over 8 s | Resumes |

**Bird scatter signal**: A bird scatter one-shot (`sfx_amb_birds_scatter_01.ogg`) fires when an enemy enters a wildlife spawn point's detection radius. This is diegetic only — no UI marker. The trigger has a 20% random misfire rate so it functions as a hint, not a radar ping. Players who learn to read it are rewarded; players who miss it are not penalized.

**Player heartbeat**: During Combat state, `sfx_player_heartbeat_combat_loop` enters the mix and loops. It exits with the Combat → Calm transition.

---

**Rule 9 — Finale Override**

The cavalry charge finale uses a scripted override that suspends the adaptive state machine:

1. Mission Tracker fires `bFinalChargeBegun = true`
2. `UAudioStateManager` sets `bAdaptiveMusicEnabled = false` — adaptive crossfade logic halts
3. `mus_finale_cavalry_charge` begins on Music slot [1] (dedicated, independent of adaptive slot [0])
4. Adaptive score on slot [0] fades to 0% over 3.0 s
5. Finale plays to completion (loops if battle is extended)
6. On mission end: finale fades over 6.0 s, `bAdaptiveMusicEnabled = true`, adaptive score resumes from Calm

**Stretch goal**: 8–12 seconds before the charge trigger, a brief solo passage recalls the kobyz prologue melody (`mus_finale_prologue_echo_01`). Requires one additional asset and one sequenced trigger. Flag for scope review.

---

**Rule 10 — Voice Budget**

| Parameter | Value |
|-----------|-------|
| Global max simultaneous voices | 32 |
| Reserved (UI + Dialogue, never stolen) | 4 |
| Available for world sounds | 28 |
| Distance culling threshold | 10,000 UU (100 m) |

Per-category limits and steal rules:

| Category | Max voices | Steal rule |
|----------|-----------|------------|
| UI.Menu | Unlimited | Never |
| Ritual.Kobyz | 4 | Steal oldest (with 0.8 s release tail) |
| Player.Movement | 4 | Steal oldest |
| Player.Combat | 3 | Steal oldest |
| Enemy.Melee | 4 | Steal quietest |
| Mount.Horse | 2 | Steal oldest |
| Ambient loops | 3 | Stop all if budget exceeded |
| Dialogue | 2 | Queue second (0.5 s delay) |

---

**Rule 11 — Attenuation Profiles**

Six named attenuation assets (Inverse/logarithmic falloff, Sphere shape):

| Profile | Min distance | Max distance | Notes |
|---------|-------------|-------------|-------|
| `Att_Player` | 200 UU | 800 UU | Player footsteps/combat; kinesthetic |
| `Att_EnemyCombat` | 300 UU | 4,000 UU | Enemy melee/ranged; spatial directionality |
| `Att_Interactive` | 500 UU | 2,000 UU | Doors, chests, world objects |
| `Att_Mount` | 400 UU | 5,000 UU | Horse carries far across open steppe |
| `Att_AmbientPoint` | 800 UU | 3,000 UU | Campfires, streams, wildlife points |
| `Att_NPCVoice` | 500 UU | 4,000 UU | All NPC dialogue, spatialized |

---

**Rule 12 — Player-Exposed Settings**

Four volume sliders, all persisted via `UAltynSettingsSaveGame`:

| Setting | Default | Range | Bound to |
|---------|---------|-------|----------|
| Master Volume | 1.0 | 0.0–1.0 | Master Sound Class |
| Music Volume | 0.8 | 0.0–1.0 | Music Sound Class |
| SFX Volume | 1.0 | 0.0–1.0 | SFX Sound Class |
| Ambient Volume | 0.9 | 0.0–1.0 | Ambient Sound Class |

Dialogue and UI volume are not exposed separately. Dialogue is sparse enough that independent control is unnecessary. UI is coupled to SFX for consistency.

### States and Transitions

| State | Description | Entry | Exit |
|-------|-------------|-------|------|
| `Calm` | Default exploration; Melodic Foundation layer active | Game start; post-combat dwell elapsed; all enemies defeated | Any enemy reaches `Alert` AI state |
| `Suspicion` | Enemy aware; Tension Weave fades in | Enemy AI state = Alert | First damage event → Combat; 12 s no alert → Calm |
| `Combat` | Active fight; Rhythmic Drive at full; heartbeat enters | First damage event | 20 s no damage AND no alert enemies → Calm |
| `Ritual` | Kobyz interaction active; world audio ducks | Kobyz ritual triggered | Ritual complete or player exits interaction |
| `Finale` | Scripted override; adaptive machine suspended | `bFinalChargeBegun` | Mission end |

State machine is owned by `UAudioStateManager`. Only one music state is active at a time. Ritual layers on top of Calm (not Combat). Finale overrides any state.

### Interactions with Other Systems

| System | Provides to Audio | Audio Provides | Trigger |
|--------|------------------|----------------|---------|
| AI System | Enemy `Alert` state change | — | Broadcast on awareness change → `UAudioStateManager` sets Suspicion |
| Combat System | `OnTakeDamage` / `OnDealDamage` events | — | First damage → Combat; timestamp for dwell timer |
| Kobyz Music System | Note trigger (pitch index), `bMelodyComplete` | Individual note audio assets; melody resolution response | Per-button input |
| Dialogue & Choice System | Dialogue tier (Narrative or Chatter) | Mix A or Mix B pushed/popped | Dialogue start/end events |
| Mission Tracker | `bFinalChargeBegun` | — | Finale override on flag |
| Save/Load System | Loads `UAltynSettingsSaveGame` at startup | 4 volume settings persisted on change | `OnSaveLoaded`; on slider change |
| Weather System | `WindIntensity` float | — | Wind Base volume driven by this parameter |

## Formulas

The Audio System contains no gameplay math. Its quantitative parameters are:

**F1 — Music Crossfade Durations**

```
Calm → Suspicion:    3.0 s
Suspicion → Combat:  0.8 s
Combat → Suspicion:  4.0 s
Combat → Calm:       8.0 s
Suspicion → Calm:    5.0 s
```

| Variable | Value | Range | Effect if Too Short | Effect if Too Long |
|----------|-------|-------|--------------------|--------------------|
| `Calm_to_Suspicion` | 3.0 s | 1.0–5.0 s | Jarring cut; breaks immersion | Tension doesn't build fast enough |
| `Suspicion_to_Combat` | 0.8 s | 0.3–2.0 s | Instant cut feels electronic | Combat eruption loses its lurch |
| `Combat_to_Calm` | 8.0 s | 4.0–15.0 s | Exhale moment is lost | Player in next encounter before score resets |

**F2 — Minimum State Dwell Times**

```
SuspicionDwell = 12 s (after last alert trigger)
CombatDwell    = 20 s (after last damage AND no alert enemies)
```

| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|--------------------|
| `SuspicionDwell` | 12 s | 5–30 s | Score ping-pongs on patrol edge cases | Suspicion music outlasts the threat |
| `CombatDwell` | 20 s | 10–45 s | Score drops mid-combat during lulls | Player waits too long for calm return after a short fight |

**F3 — Sound Mix Targets**

```
Dialogue Narrative: Music × 0.15 | Ambient × 0.25 | SFX × 0.50
Dialogue Chatter:   Music × 0.60 | Ambient × 1.0  | SFX × 1.0
Combat Alertness:   Ambient × 0.20
Cinematic:          Ambient × 0.0 | SFX × 0.30
```

| Mix | Parameter | Default | Safe Range | Effect if Too Low | Effect if Too High |
|-----|-----------|---------|------------|-------------------|--------------------|
| Dialogue Narrative | Music target | 0.15 | 0.0–0.4 | Score inaudible during speech | Score competes with dialogue |
| Dialogue Narrative | Ambient target | 0.25 | 0.0–0.5 | World vanishes during story | Ambient distracts from speech |
| Combat Alertness | Ambient target | 0.20 | 0.0–0.5 | Steppe silence feels artificial | World-narrowing effect is lost |

**F4 — Kobyz Polyphony and Timing**

```
MaxKobyzVoices    = 4
ReleaseTime       = 0.8 s
RapidInputWindow  = 0.1 s  (notes within this window crossfade rather than stack)
VariantsPerPitch  = 3
```

**F5 — Voice Budget**

```
GlobalVoiceMax   = 32
ReservedVoices   = 4   (UI.Menu + Dialogue — never stolen)
WorldVoices      = 28
CullDistance     = 10,000 UU
```

**F6 — Kobyz Ritual Ambient Duck**

```
On ritual entry:
  Layer0_Volume = 0.30  (over 1.5 s)
  Layer1_Volume = 0.0   (over 2.0 s)
  AmbientClass  = 0.40  (over 2.0 s)

On ritual exit:
  All values return to pre-ritual levels over 2.0 s
```

**F7 — Post-Combat Exhalation Trigger**

```
ExhalationDelay = 4.0 s after Combat → Calm transition begins
```

## Edge Cases

**E1 — Combat Starts During Kobyz Ritual**

Player is at a kobyz ritual interaction point when an enemy enters `Alert` state. The ritual state takes priority; the music state machine does not transition to Suspicion while `bRitualActive == true`. If damage is taken during ritual, the ritual is forcibly interrupted: ritual state exits over 1.0 s, world audio returns to pre-ritual levels, and the music state machine transitions to Combat immediately.

**E2 — Multiple Simultaneous Dialogue Speakers**

Two NPCs speak simultaneously. Mix B (Chatter) uses a ref-count — the mix pops only when all active chatter sources end. If a Narrative line triggers while chatter is active, Mix A overrides immediately. `bNarrativeDialogueActive = true` blocks Mix B for its duration.

**E3 — Enemy Alert During Combat Dwell**

A second encounter begins while the 20-second Combat dwell is counting down from a previous fight. The dwell timer resets to 20 s from the new alert event. The Combat state remains active. No transition occurs.

**E4 — Finale Triggered While In Combat**

`bFinalChargeBegun` fires while the music state machine is in Combat state. The finale override fires immediately — `bAdaptiveMusicEnabled = false`, Combat heartbeat fades out over 1.0 s, `mus_finale_cavalry_charge` begins. The adaptive score does not wait for the Combat dwell to expire.

**E5 — Volume Set to Zero by Player**

Player sets Music Volume to 0.0. The MetaSound asset continues running internally (maintains sync and state). Only the Sound Class volume multiplier is zero. When the player restores volume, music resumes at the correct state without re-entry artifacts. Do NOT pause or stop the MetaSound graph when volume is muted.

**E6 — Audio Device Disconnected at Runtime**

Player unplugs headphones mid-session. Unreal Engine 5 handles device rerouting at the engine level. No game-level handling required. If Unreal fails to reroute, audio may silence, but no crash or save corruption occurs — audio state is in memory, not on disk.

**E7 — Kobyz Note Retriggered Before Release Tail Completes**

Player rapidly presses the same pitch button. Each press is a new `UAudioComponent` on the `Ritual.Kobyz` Sound Class, subject to the 4-voice limit. If 4 voices are active, the oldest enters its 0.8 s release tail and the new component begins immediately. Brief overlap of the same pitch is musically acceptable and physically realistic for a bowed instrument.

**E8 — Scene Load Clears All Audio**

Level transition interrupts all active audio. On load completion: `UAudioStateManager` re-evaluates current music state and restarts `MUS_Adaptive_Score_Main` from the beginning of the current state's loop. Ambient layers restart their loops. No audio state is persisted to disk — all state is re-derived from gameplay state on load.

## Dependencies

**Upstream (Audio System depends on these):**

| System | Dependency | Notes |
|--------|-----------|-------|
| UE5 MetaSounds | MetaSound node graph for adaptive score; `SetFloatParameter`, `SetBoolParameter` at runtime | Engine API |
| UE5 Sound Classes & Mixes | `USoundClass`, `USoundMix`, `PushSoundMixModifier` / `PopSoundMixModifier` | Engine API |
| UE5 AudioComponent | `UAudioComponent` for kobyz notes, Exhalation Cue, ambient point sources | Engine API |
| UE5 Sound Attenuation | `USoundAttenuation` assets (6 profiles defined in Rule 11) | Engine API |
| Save/Load System | `UAltynSettingsSaveGame` stores 4 volume settings; loaded before first audio plays | Defined in save-load-system.md |
| Input System | Kobyz ritual note triggers come from IMC_KobyzRitual (IA_KobyzNote_[Clan] per pitch) | Input System owns the bindings; Audio System owns the audio response |

**Downstream (depend on Audio System):**

| System | What They Need | Notes |
|--------|---------------|-------|
| Kobyz Music System | Individual note `UAudioComponent` playback, `bMelodyComplete` trigger, ritual ambient duck | Audio provides sound; Kobyz Music System provides sequencing logic |
| Dialogue & Choice System | Calls `UAudioStateManager` to push/pop Sound Mixes on dialogue start/end | Audio exposes `PushDialogueMix(ETier)` and `PopDialogueMix()` |
| Mission Tracker | Sets `bFinalChargeBegun` flag on `UAudioStateManager` | One-way: Mission Tracker → Audio |
| Combat System | Broadcasts damage events; Audio listens for first damage to enter Combat state | One-way: Combat → Audio |
| AI System | Broadcasts enemy alert state changes; Audio listens | One-way: AI → Audio |
| Weather System | Provides `WindIntensity` float to `UAudioStateManager` for Wind Base volume | One-way: Weather → Audio |

**Bidirectionality note**: The Audio System is a listener. No system reads state from the Audio System at runtime.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|--------------------|
| `Calm_to_Suspicion` crossfade | 3.0 s | 1.0–5.0 s | Jarring cut into tension | Threat goes unacknowledged too long |
| `Suspicion_to_Combat` crossfade | 0.8 s | 0.3–2.0 s | Hard electronic cut | Violence loses its sudden lurch |
| `Combat_to_Calm` crossfade | 8.0 s | 4.0–15.0 s | Exhale moment is absent | Player in next fight before score resets |
| `SuspicionDwell` | 12 s | 5–30 s | Score ping-pongs at patrol edges | Suspicion music outlasts the threat |
| `CombatDwell` | 20 s | 10–45 s | Score drops during mid-combat lulls | Long wait for calm after short fights |
| `ExhalationDelay` | 4.0 s | 0–8.0 s | Exhalation overlaps combat decay | Feels disconnected from combat end |
| Dialogue Narrative music target | 0.15 | 0.0–0.4 | Score inaudible | Score competes with NPC speech |
| Dialogue Narrative ambient target | 0.25 | 0.0–0.5 | World disappears during story | Ambient distracts from speech |
| Combat Alertness ambient target | 0.20 | 0.0–0.5 | Steppe feels artificially silent | World-narrowing effect is lost |
| `MaxKobyzVoices` | 4 | 2–6 | Rapid playing sounds thin | Voice count strains budget |
| `KobyzReleaseTime` | 0.8 s | 0.3–2.0 s | Notes feel percussive, not bowed | Muddy overlap during fast playing |
| `RapidInputWindow` | 0.1 s | 0.05–0.3 s | Stacked notes sound choppy | Flowing melodies hard to play cleanly |
| `GlobalVoiceMax` | 32 | 16–64 | Sounds drop in busy scenes | CPU overhead on mid-range hardware |
| `CullDistance` | 10,000 UU | 5,000–20,000 UU | Nearby sounds disappear | Distant sounds waste voice budget |
| Wind Base at Combat | 0.40 | 0.1–0.6 | World feels dead during fights | World-narrowing effect is lost |
| Bird scatter misfire rate | 20% | 0–40% | Scatter becomes reliable radar ping | Scatter becomes meaningless noise |
| Master Volume default | 1.0 | — | — | — |
| Music Volume default | 0.8 | — | — | — |
| SFX Volume default | 1.0 | — | — | — |
| Ambient Volume default | 0.9 | — | — | — |

**Note on the 8-second Combat → Calm crossfade**: This will feel slow in early testing. Hold the line — the exhale only works if the player has time to feel it.

## Visual/Audio Requirements

The Audio System is the audio authority for the entire game. Its own production requirements are:

- All audio assets must have a dedicated directory structure under `assets/audio/`:
  - `assets/audio/music/` — adaptive score MetaSound, exhalation cue, finale cue
  - `assets/audio/sfx/player/`, `sfx/enemy/`, `sfx/mount/`, `sfx/environment/`, `sfx/ui/`
  - `assets/audio/ritual/kobyz/` — 6 pitches × 3 variants = 18 WAV files
  - `assets/audio/ambient/` — wind, wildlife, settlement loops
- All looping ambient assets must loop with < 50 ms silence (no audible gap)
- Kobyz note assets must be recorded from a real kobyz instrument or high-quality virtual instrument. The specific timbre of the instrument is load-bearing to the Player Fantasy — synthesized notes are not acceptable.
- The adaptive score must have all 4 layers composed and recorded before integration. All layers must be aligned to the same BPM grid so they remain in phase sync across runtime.

## UI Requirements

- **Volume sliders**: Four sliders in the Settings menu (Master, Music, SFX, Ambient). Range displayed as 0–100; stored internally as 0.0–1.0. Slider change applies immediately. Persisted to `UAltynSettingsSaveGame` on slider release.
- No other audio UI is required at MVP. The save indicator icon is owned by the UI/HUD GDD.

## Acceptance Criteria

| # | Criterion | How to Verify | Pass Condition |
|---|-----------|--------------|----------------|
| AC-1 | Music transitions from Calm to Suspicion when enemy reaches Alert | QA: walk into patrol alert radius without triggering combat | Score crossfades over ~3 s; no Rhythmic Drive audible |
| AC-2 | Music transitions to Combat on first damage | QA: take or deal first hit in an encounter | Rhythmic Drive layer audible within 0.8 s of damage event |
| AC-3 | Combat dwell prevents premature calm return | QA: kill all enemies, wait 10 s, verify music | Score does not return to Calm until 20 s after last damage |
| AC-4 | Post-combat exhalation cue plays | QA: end combat, wait 4 s, listen | Dombra phrase audible ~4 s after combat ends |
| AC-5 | Dialogue Narrative ducks music to ~15% | QA: trigger a named NPC line, monitor audio levels | Music drops noticeably; speech fully intelligible |
| AC-6 | Combat Alertness ducks ambient to ~20% | QA: enter combat, listen for ambient | Bird/wind sounds drop significantly; player heartbeat audible |
| AC-7 | Kobyz ritual ducks world audio | QA: enter kobyz ritual interaction, listen for steppe | Wind and music drop; kobyz notes heard in near-silence |
| AC-8 | Melody completion triggers harmonic resonance | QA: play a complete kobyz melody sequence | Chord bloom and sub-bass present on final note; no visual UI notification |
| AC-9 | Finale override suspends adaptive score | QA: trigger `bFinalChargeBegun` via debug flag | Adaptive score fades; finale cue plays; adaptive does not resume until mission end |
| AC-10 | Volume sliders persist across sessions | QA: set Ambient to 0.2, quit, relaunch | Ambient volume remains at 0.2 on next session |
| AC-11 | Voice budget enforced in busy scene | QA: 8+ enemies active simultaneously, run `stat soundwaves` console command | Active voice count ≤ 32; UI sounds never drop |
| AC-12 | Bird scatter fires diegetically with misfire rate | QA: observe enemy patrol approaching wildlife spawner across 10 approaches | Scatter plays on approximately 8/10 approaches (not every time) |

## Open Questions

- **Kobyz pitch tuning**: The 6-pitch set (C4–A4–C5) is a placeholder. A composer must validate or revise the exact frequencies before audio production begins.
- **Finale prologue echo**: The optional pre-charge prologue echo (`mus_finale_prologue_echo_01`) is a stretch goal. Confirm scope inclusion after adaptive score composition is complete.
- **Weather System contract**: The `WindIntensity` float driving Wind Base volume assumes a Weather System broadcasts this parameter. If Weather System is cut, Wind Base needs a static volume fallback.
- **Voiced dialogue assets**: The Dialogue Sound Class assumes voiced NPC lines. If voice acting is cut from budget, the Sound Class remains empty and no dialogue ducking occurs in practice. Test ducking mixes with silent placeholder audio.
- **Settlement Proximity ambient**: Proximity detection requires a trigger volume or query from the Open World System. If Open World System does not provide this, proximity ambient falls back to hand-placed audio volumes per level.
