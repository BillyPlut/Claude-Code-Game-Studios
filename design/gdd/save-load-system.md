# Save / Load System

> **Status**: In Design
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-08
> **Implements Pillar**: All Three (persistence of every gameplay decision)

## Overview

The Save / Load System persists all player progress across sessions using Unreal Engine's `USaveGame` framework. It writes to a single named save slot and serializes the complete game state: active chapter and player position, collected Memories, unlocked Armor fragments, per-clan trust levels (0–100), dialogue choices made, mission tracker state, world POI discovery flags, and input remapping preferences. Two write triggers are defined: a timed auto-save every 5 minutes, and an event-driven save fired by the game after key moments — significant dialogue choices, Memory collection, fragment discovery, and chapter transitions. Saves are written asynchronously so gameplay never pauses during a write. A separate checkpoint slot captures state at the start of each combat encounter and is overwritten on successful completion; if the player dies, the checkpoint is loaded rather than the full save. Save data is versioned to allow future fields to be added without corrupting existing saves.

## Player Fantasy

The player never thinks about saving. They think about what they have done — and they trust that the steppe holds it. A dialogue cannot be unspoken; a clan's trust cannot be quietly retried; a Memory once recovered cannot be misplaced. Combat may be retried, because death in battle is failure of skill, not of self — but a choice made in council is a stone laid, and stones do not move. The system's only obligation is to ensure that the boy who rides into the final charge is provably, irrefutably, the boy the player built. Failure is the moment the player notices the system at all.

## Detailed Design

### Core Rules

**Rule 1 — Two Save Objects**
All persistence is split across two `USaveGame` subclasses:
- `UAltynSaveGame` — all gameplay state (chapter, position, memories, armor, trust, choices, objectives, POIs)
- `UAltynSettingsSaveGame` — input remapping preferences only (written on remap, not on auto-save; loaded before gameplay begins)

The combat checkpoint is a full copy of `UAltynSaveGame`. `UAltynSettingsSaveGame` is never checkpointed.

**Rule 2 — Slot Names**
Slot names are `static const FString` constants in `UAltynSaveSubsystem`. Never inline string literals.

| Constant | Slot Name |
|----------|-----------|
| `PrimarySlot` | `"AltynAdam_Save"` |
| `CheckpointSlot` | `"AltynAdam_Checkpoint"` |
| `SettingsSlot` | `"AltynAdam_Settings"` |

**Rule 3 — Save Manager**
All save/load operations are owned by `UAltynSaveSubsystem : UGameInstanceSubsystem`. Accessed via `GetGameInstance()->GetSubsystem<UAltynSaveSubsystem>()`. Survives level transitions. No other system writes to disk directly.

**Rule 4 — Auto-Save Triggers**
Auto-save writes to the primary slot only. Triggers:

| Trigger | Who fires it |
|---------|-------------|
| Every 5 minutes (game-time timer) | `UAltynSaveSubsystem` internal timer |
| Significant dialogue choice made | Dialogue & Choice System |
| Memory collected | Memory System |
| Armor fragment unlocked | Armor/Fragment System |
| Chapter transition | Mission Tracker |
| Manual save from Pause Menu | UI System |

Auto-save is **suppressed** during active combat (preserves checkpoint integrity — a fight cannot be escaped via auto-save).

**Rule 5 — Async Write Protocol**
All primary slot writes use `AsyncSaveGameToSlot`. A `bSaveInProgress` bool guards against concurrent writes. If a save is requested while `bSaveInProgress == true`, the request is queued and processed when the current write's delegate fires. The delegate fires on the game thread; no state mutation occurs between kick-off and callback.

**Rule 6 — Combat Checkpoint**
Written **synchronously** (`SaveGameToSlot`) at the start of each combat encounter, before the first enemy engages. Synchronous to guarantee the checkpoint exists before combat begins. On player death: load checkpoint slot → overwrite in-memory save state → resume. Checkpoint is overwritten on successful combat completion (cannot be exploited as a manual save for pre-fight state).

**Rule 7 — Save Versioning**
`UAltynSaveGame` contains an `int32 SaveVersion` UPROPERTY, starting at `1`. On every load: compare `SaveVersion` to `CurrentSaveVersion`. If the loaded version is older, call `MigrateFromVersion()` to apply default values for new fields. **Never delete or reorder existing UPROPERTYs** — only append. If the loaded version is newer (from a future build), log a warning and continue; Unreal's serializer silently drops unrecognized properties.

**Rule 8 — Save Data Schema (`UAltynSaveGame`)**

| Field | Type | Description |
|-------|------|-------------|
| `SaveVersion` | `int32` | Schema version, starts at 1 |
| `ActiveChapter` | `FName` | Current chapter ID (e.g., `"Chapter_1"`) |
| `PlayerPosition` | `FVector` | World position at last save |
| `CollectedMemories` | `TArray<FName>` | IDs of collected Memories |
| `ArmorFragmentsUnlocked` | `TArray<bool>` | 6 elements indexed by clan order |
| `ClanTrustLevels` | `TMap<FName, float>` | Clan ID → trust level (0.0–100.0) |
| `DialogueChoices` | `TMap<FName, int32>` | Dialogue node ID → choice index |
| `CompletedObjectives` | `TArray<FName>` | IDs of completed objectives |
| `ActiveObjectives` | `TArray<FName>` | IDs of current active objectives |
| `DiscoveredPOIs` | `TArray<FName>` | IDs of discovered world POIs |
| `LastSaveTime` | `FDateTime` | Timestamp of last write |

**Rule 9 — Settings Schema (`UAltynSettingsSaveGame`)**

| Field | Type | Description |
|-------|------|-------------|
| `KeyProfile` | Serialized `UEnhancedPlayerMappableKeyProfile` | Player remap data |
| `MouseSensitivity` | `float` | 0.1–2.0 |

### States and Transitions

| State | Description | Entry | Exit |
|-------|-------------|-------|------|
| `Idle` | No I/O in progress | Start; write callback received; load complete | Save trigger; combat start; game start |
| `Saving` | Async primary slot write in progress | Auto-save or manual save trigger | Write callback received |
| `CheckpointSaving` | Synchronous checkpoint write | Combat encounter starts | Write complete (immediate) |
| `Loading` | Reading from disk | Game start; player death | Load complete; `OnSaveLoaded` broadcast |

- `CheckpointSaving` briefly blocks (synchronous). Expected duration < 16ms at < 10 KB save size.
- `Saving` and `Loading` cannot overlap. If a load is triggered while `bSaveInProgress`, load waits for the write callback first.

### Interactions with Other Systems

| System | Provides to Save | Reads from Save | Trigger |
|--------|-----------------|-----------------|---------|
| Memory System | `CollectedMemories` array | Restores unlocked Memory abilities | `OnSaveLoaded`; fires save on Memory collect |
| Armor/Fragment System | `ArmorFragmentsUnlocked` array | Restores fragment visuals + abilities | `OnSaveLoaded`; fires save on fragment unlock |
| NPC Faction & Trust System | `ClanTrustLevels` map | Restores per-clan trust | `OnSaveLoaded`; fires save after trust-changing events |
| Dialogue & Choice System | `DialogueChoices` map | Prevents replaying already-made choices | `OnSaveLoaded`; fires save after significant choice |
| Mission Tracker | `CompletedObjectives`, `ActiveObjectives` | Restores objective state | `OnSaveLoaded`; fires save on chapter transition |
| Open World System | `DiscoveredPOIs` | Restores POI discovery flags | `OnSaveLoaded` |
| Character Controller | `PlayerPosition`, `ActiveChapter` | Spawns player at saved position | `OnSaveLoaded` |
| Input System | — | `UAltynSettingsSaveGame` loaded on startup; subsystem serializes `UEnhancedPlayerMappableKeyProfile` | Game start; on remap |
| UI System | Receives `bSaveInProgress` | Shows/hides save indicator icon | On flag change |

## Formulas

The Save / Load System contains no gameplay math. The quantitative parameters are:

**F1 — Auto-Save Timer Interval**
```
AutoSaveInterval = 300 seconds (5 minutes of game time)
```
| Variable | Value | Range | Effect if Too Low | Effect if Too High |
|----------|-------|-------|-------------------|-------------------|
| `AutoSaveInterval` | 300s | 60s – 600s | Frequent disk I/O; save indicator distracts player | Player can lose up to 10 minutes of progress on crash |

**F2 — Save Version Migration**
```
if (LoadedSaveVersion < CurrentSaveVersion):
    for v in range(LoadedSaveVersion, CurrentSaveVersion):
        MigrateFromVersion(v → v+1)
```
| Variable | Starting Value | Rule |
|----------|---------------|------|
| `CurrentSaveVersion` | 1 | Increment by 1 each time a UPROPERTY is added or semantically changed |
| `LoadedSaveVersion` | Read from file | If < CurrentSaveVersion → migrate; if > CurrentSaveVersion → log warning |

Migration is additive only: each step sets default values for newly added fields. No migration step may modify or remove data from a previous version.

**F3 — Checkpoint Eligibility**
```
WriteCheckpoint = (CombatEncounterStarted == true) AND (bSaveInProgress == false)
```
The checkpoint is only written if no async save is currently in progress. If `bSaveInProgress == true` when combat starts, delay the checkpoint write by one tick and retry.

## Edge Cases

**E1 — Game Crash During Async Write**
If the process crashes while `AsyncSaveGameToSlot` is in progress, the `.sav` file may be partially written and corrupt. On next load, `LoadGameFromSlot` returns `nullptr` for a corrupt file. Detection: after load, check that `UAltynSaveGame` is non-null and `SaveVersion >= 1`. If null or invalid: display "Save data could not be loaded" and offer to start a new game. Do NOT silently load default state without informing the player.

**E2 — Player Quits During Auto-Save**
If the player closes the game while `bSaveInProgress == true`, the write may not complete. The 5-minute timer and event-driven triggers ensure the last completed save is at most 5 minutes old. The partial write detection from E1 applies on next launch. No additional handling required.

**E3 — Combat Checkpoint Conflicts With In-Progress Auto-Save**
If `bSaveInProgress == true` when a combat encounter starts, retry the checkpoint write each tick for up to 10 ticks (167ms at 60fps). If still blocked after 10 ticks, write the checkpoint anyway. Log: `"Checkpoint delayed — save in progress at combat start"`.

**E4 — Player Dies Before Checkpoint Is Written**
If the player dies before a delayed checkpoint write (E3) completes, there is no valid checkpoint. Behavior: fall back to the last primary slot save. Display: "Resuming from last save point." Do NOT spawn the player in an active combat state.

**E5 — No Save File Exists (First Run)**
`DoesSaveGameExist("AltynAdam_Save")` returns `false`. Behavior: initialize a fresh `UAltynSaveGame` with defaults (`SaveVersion = 1`, chapter = Prologue, all arrays empty, all trust levels = 50.0). Write this default state to disk immediately so subsequent loads always find a valid file.

**E6 — Save File From an Incompatible Future Build**
`LoadedSaveVersion > CurrentSaveVersion`. Behavior: log a warning, load the file (Unreal drops unknown fields), and continue. Display a non-blocking notification: "Save file is from a newer version — some progress may not be restored." Do NOT block the player from loading their save.

**E7 — Player Reloads Checkpoint to Escape a Narrative Choice**
A player makes a significant dialogue choice, triggering an auto-save. They then die in the subsequent combat. On death, the checkpoint (from before the choice) is loaded for combat state — but the auto-save has already recorded the choice. The checkpoint restores gameplay state (position, health) only. Narrative state (dialogue choices) is always read from the last completed primary save write, not from the checkpoint. This is by design: combat may be retried; dialogue cannot be unspoken.

## Dependencies

**Upstream (Save/Load depends on these):**

| System | Dependency | Notes |
|--------|-----------|-------|
| UE5 `USaveGame` framework | `SaveGame`, `GameplayStatics` module in `Build.cs` | Engine API; not a game system |
| UE5 `UGameInstanceSubsystem` | `Engine` module | Subsystem host; survives level transitions |
| Input System | `UEnhancedPlayerMappableKeyProfile` serialized in `UAltynSettingsSaveGame` | Input System established this contract in its GDD |

**Downstream (depend on Save/Load):**

| System | What They Need | Notes |
|--------|---------------|-------|
| Memory System | `CollectedMemories` restored on load | Their GDD must list Save/Load as dependency |
| Armor/Fragment System | `ArmorFragmentsUnlocked` restored on load | — |
| NPC Faction & Trust System | `ClanTrustLevels` restored on load | — |
| Dialogue & Choice System | `DialogueChoices` restored; fires save after significant choices | — |
| Mission Tracker | `CompletedObjectives`, `ActiveObjectives` restored | — |
| Open World System | `DiscoveredPOIs` restored | — |
| Character Controller | `PlayerPosition`, `ActiveChapter` restored | — |
| Name/Legacy System | All of the above — the final name is computed from the complete save state | High dependency: if any field is lost, the final name calculation breaks |
| UI System | `bSaveInProgress` flag for save indicator | — |

**Bidirectionality note:** Each downstream system's GDD must list the Save/Load System as an upstream dependency and specify which `UAltynSaveGame` fields it reads.

## Tuning Knobs

| Knob | Default | Safe Range | Effect if Too Low | Effect if Too High |
|------|---------|-----------|-------------------|-------------------|
| `AutoSaveInterval` | 300s | 60s – 600s | Save icon appears too often; minor I/O overhead | Player loses up to `interval` seconds of progress on crash |
| `CheckpointRetryTicks` | 10 ticks | 1 – 30 ticks | Checkpoint may write while async save still in progress (ordering race) | Combat start delayed up to 500ms — noticeable stutter |
| `DefaultClanTrustLevel` | 50.0 | 0.0 – 75.0 | All clans start hostile — frustrating early game | All clans start as allies — removes stakes of trust-building |
| `SaveVersion` | 1 (initial) | — | Not tunable — increment only when schema changes |

**Player-exposed settings:**
- Manual Save: available from Pause Menu at any time outside combat
- (Post-MVP) Save indicator display duration

## Visual/Audio Requirements

None. The Save / Load System is Foundation-layer infrastructure with no direct visual or audio output. The save indicator icon is owned by the UI System and Combat & Stealth HUD GDD.

## UI Requirements

- **Save Indicator**: Small non-intrusive icon during active saves. Must not block gameplay. Disappears when `bSaveInProgress` returns to `false`. Visual design owned by Combat & Stealth HUD GDD.
- **"Save data could not be loaded" Dialog**: Blocking modal if corrupt save detected (E1). Options: "Start New Game" / "Quit to Desktop." No "Retry" — a corrupt file will not self-repair.
- **Manual Save**: Accessible from Pause Menu at any time outside combat. Single slot only — no slot management UI.
- **"Resuming from last save point" Notification**: Non-blocking toast shown when falling back to primary save after a missing checkpoint (E4).

## Acceptance Criteria

| # | Criterion | How to Verify | Pass Condition |
|---|-----------|--------------|----------------|
| AC-1 | Auto-save fires every 5 minutes | QA: play for 10+ minutes, check `.sav` file timestamp | File timestamp updates at ~5-minute intervals |
| AC-2 | Auto-save fires after collecting a Memory | QA: collect a Memory, immediately force-quit, relaunch | Memory is present in save after relaunch |
| AC-3 | No auto-save during active combat | QA: enter combat, wait 6+ minutes, monitor file writes | No auto-save write occurs during combat; timer fires after combat ends |
| AC-4 | Checkpoint written at combat start | QA: enter combat, die, verify spawn position | Player spawns at pre-combat position, not at last primary save position |
| AC-5 | Checkpoint does NOT revert dialogue choices | QA: make a dialogue choice, enter combat, die | Dialogue choice is preserved after checkpoint load |
| AC-6 | Corrupt save detected and handled | QA: manually corrupt `.sav` file, launch game | "Save data could not be loaded" message shown; no silent default state loaded |
| AC-7 | First run initializes default save | QA: delete all save files, launch game | Fresh `UAltynSaveGame` with `SaveVersion = 1` written to disk before main menu |
| AC-8 | Remap data persists across sessions | QA: remap one key, quit, relaunch | Remapped key is active after relaunch |
| AC-9 | Save migration preserves existing data | QA: manually set `SaveVersion = 0` in save file, launch | Existing fields read correctly; new fields have defaults; no data loss |
| AC-10 | No concurrent async writes | QA: trigger two rapid save events within 1 second | Only one write completes at a time; second write queues and fires after first |

## Open Questions

- **Rolling save backup**: Should the system keep a backup of the previous primary save (`"AltynAdam_Save_Backup"`) to protect against a corrupt overwrite? Adds I/O and complexity. Defer to post-MVP.
- **Cloud save**: Steam Cloud sync of `.sav` files. Requires Steamworks SDK integration. Out of scope for MVP — add to Full Vision milestone.
- **Dialogue choice replay**: The "no unspoken words" design prevents undoing choices. Should a "last dialogue branch rewind" option exist in Pause Menu? Decision deferred to Dialogue System GDD — it conflicts with this GDD's Player Fantasy.
- **Chapter-start save**: Should each chapter transition write a special "chapter start" snapshot (separate from rolling primary) to enable chapter replay? Decision deferred to Mission Tracker GDD.
