# Dialogue & Choice System

> **Status**: Complete
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-11
> **Implements Pillar**: Pillar 1 (clan trust built through deeds, not words) + Pillar 2 (choices define who the boy becomes) + Pillar 3 (final battle army shaped by all choices made)

## Overview

The Dialogue & Choice System manages all spoken interactions between the boy and the world: NPC conversations, quest-gating discussions, clan council moments, and the unmarked moral choices that shape the final battle. It owns two domains: **playback** (rendering the active dialogue tree, presenting choices, advancing nodes) and **consequence** (writing choice outcomes to save data, modifying clan trust levels, firing save triggers for significant decisions).

Dialogue trees are authored as `UDataAsset` subclasses (`UDialogueNode`, `UDialogueTree`) edited entirely within Unreal Editor — no external tools, no JSON. A runtime manager (`UDialogueSubsystem : UGameInstanceSubsystem`) loads the active tree, steps through nodes, presents choice lists to the UI, and resolves consequences on selection. Choice results are persisted as `DialogueChoices: TMap<FName, int32>` in `UAltynSaveGame` — a permanent record that future scenes and the final battle algorithm read to determine NPC states and army composition.

There are no moral markers. No choice is labeled right or wrong. The system never tells the player what a choice costs — only that they made it. Consequences emerge later: a trust level that opened or closed a door, a counselor who was spared or named, a name that the boy receives in the final scene. The system's job is to record choices faithfully and deliver consequences at the right moment. Its obligation to the player is that every stone they lay is real, and none of them move.

## Player Fantasy

You speak, or you do not speak. The screen does not tell you which was right. There is no glow on the word that earns trust, no warning on the one that costs it — only the moment after, where the elder looks at you a little longer than before, or does not, and you cannot tell which it was. The fantasy is this: every choice is a stone set down on the path behind you, and none of them move. The system keeps faith with what you did. It does not flatter you. It does not punish you. It only remembers.

Someone is asking you a question. They have already decided what to call you — *boy*, *survivor*, *last of them*, *the one we found* — and the name they chose tells you what they think you are. You have not given them yours. You will not. The name you had belonged to a child whose people were alive, and that child is not the one answering now. Every conversation is a small reckoning with who you are becoming. Not a character sheet being filled in. Not approval being earned. Something smaller and more true: deciding, sentence by sentence, what kind of person speaks out of your mouth when grief has thinned everything else away. Sometimes you lie. Sometimes you tell the truth too plainly and watch it land badly. Sometimes you say nothing, and the silence is the loudest thing you have said all chapter.

Trust is a slow animal. It does not come when you call it. The elders of the steppe have been lied to by emissaries for longer than you have been alive. They will not be moved by what you say — they will be moved by whether the shape of you, the deeds at your back, the silence you kept and the truth you risked telling, matches what this clan needs from a stranger today. When a counselor leans forward and answers your question instead of deflecting it, the feeling is not *I won the dialogue*. It is *they decided I was worth answering.* Some doors will close that you did not know were open. Some will open that you did not know existed. The steppe is older than you and it does not explain itself.

When the name comes, at the end — before the last charge, before the Persian line — it will not feel given. It will feel *earned in a language you did not know you were speaking.* The steppe always knew. It was waiting for you to catch up.

## Detailed Design

### Core Rules

**DC-1: Data Structure — UDialogueTree**

Each dialogue sequence is a `UDialogueTree : UPrimaryDataAsset` authored in the Unreal Editor. No external tools, no JSON.

```
UDialogueTree : UPrimaryDataAsset
  TreeID: FName                                  — unique identifier; used as save key
  DisplayName: FText                             — editor label only; never shown in-game
  SpeakerNPCID: FName                           — which NPC owns this tree
  RootNodeID: FName                             — entry point node
  Nodes: TArray<FDialogueNode>                  — all nodes, flat array
  bOneShot: bool                                — if true, cannot replay once completed
  Prerequisites: TArray<FDialoguePrecondition>  — conditions to enter this tree
  bSignificant: bool                            — author tag; trees with story-critical choices
```

Node IDs use `lowercase_snake_case` — `FName` keys are case-insensitive at creation, but canonical casing prevents authoring errors in `TMap<FName, int32>` lookups.

---

**DC-2: Data Structure — FDialogueNode**

```
FDialogueNode
  NodeID: FName
  NodeType: EDialogueNodeType    — Linear | Choice | Condition | Event
  SpeakerID: FName               — maps to NPC or "player" for player-spoken lines
  BodyText: FText
  VoiceCue: TSoftObjectPtr<USoundBase>   — soft ref; loaded on demand, not at tree load
  Choices: TArray<FDialogueChoice>       — populated for Choice nodes only
  ConditionBranches: TArray<FDialogueConditionBranch>  — populated for Condition nodes only
  NextNodeID: FName              — for Linear/Event nodes; NAME_None = tree ends
  Consequences: TArray<FDialogueConsequence>
```

Node type behavior:
- **Linear** — NPC or player line. Advances on `IA_DialogueAdvance` input or after voice cue ends.
- **Choice** — Builds visible choice list; waits for player selection. Advances to selected choice's `NextNodeID`.
- **Condition** — Evaluates `ConditionBranches` in order; first passing branch wins. Invisible to player — no pause, no UI change.
- **Event** — Fires `Consequences` and advances immediately. Used for mid-dialogue world-state changes.

---

**DC-3: Data Structure — FDialogueChoice**

```
FDialogueChoice
  ChoiceID: FName
  ChoiceText: FText
  NextNodeID: FName
  Consequences: TArray<FDialogueConsequence>
  AvailabilityConditions: TArray<FDialoguePrecondition>
  bHideIfUnavailable: bool   — false (default) = hidden when unavailable
```

---

**DC-4: Choice Visibility Rule**

When `AvailabilityConditions` fails, `bHideIfUnavailable = false` (default): the choice does not appear in the list. The player never sees a greyed-out option. The system never signals that a path was closed.

This rule is non-negotiable. It is the mechanical implementation of the "no moral markers" pillar — choices that cannot be taken simply do not exist, from the player's perspective.

`bHideIfUnavailable = true` is an explicit author override for specific cases where the presence of a locked option is intentional (e.g., a trust-gated response the player should know exists but cannot yet reach).

---

**DC-5: Data Structure — FDialogueConsequence**

```
FDialogueConsequence
  ConsequenceType: EDialogueConsequenceType
    — TrustDelta | SaveFlagWrite | MissionAdvance | MemoryUnlock | CustomEvent
  ClanID: FName          — for TrustDelta
  TrustDelta: float      — signed; applied clamped to [0.0, 100.0]
  FlagName: FName        — for SaveFlagWrite (key in DialogueChoices TMap)
  FlagValue: int32
  EventTag: FGameplayTag — for CustomEvent; broadcast via subsystem delegate
  bTriggerAutoSave: bool — calls TriggerSignificantSave() after this consequence block
```

---

**DC-6: Data Structure — FDialoguePrecondition**

```
FDialoguePrecondition
  ConditionType: EDialogueConditionType
    — FlagEquals | FlagGreaterThan | TrustAbove | TrustBelow | MemoryUnlocked | ChapterEquals
  TargetID: FName     — clan ID / flag name / memory ID depending on type
  CompareValue: int32
```

New condition types are added by extending the enum and adding an evaluation branch in `UDialogueSubsystem::EvaluateCondition()`.

---

**DC-7: Runtime Lifecycle — RequestDialogue()**

`UDialogueSubsystem::RequestDialogue(FName TreeID, AActor* SpeakerActor)` is the single entry point. Guards before starting:

1. Sync-load `UDialogueTree` asset. (Trees are small data assets — sync load is safe.)
2. Evaluate all tree `Prerequisites`. If any fail: return `false`.
3. If `bOneShot == true` and `DialogueChoices[TreeID_completed]` exists: return `false`.
4. If `CurrentState != DS_Idle`: return `false`. Dialogue cannot interrupt dialogue.
5. Add `IMC_Dialogue` at priority 2 (above `IMC_OnFoot=1`). Takes effect next input frame.
6. Broadcast `OnDialogueStarted(SpeakerActor)`. Subscribers: Camera (framing transition), Combat (suspend `bInCombat`), Stealth (pause detection gauge).
7. Set `CurrentState = DS_Playing`. Call `StepToNode(RootNodeID)`.

---

**DC-8: Runtime Lifecycle — StepToNode()**

`UDialogueSubsystem::StepToNode(FName NodeID)`:

1. Look up `NodeID` in `CurrentTree->Nodes`. If not found: call `EndDialogue()`.
2. **Condition node**: Evaluate `ConditionBranches` in order. Call `StepToNode()` on first passing branch. No state change visible to player.
3. **Event node**: Fire `Consequences` (DC-10). Call `StepToNode(NextNodeID)`.
4. **Linear node**: Set `CurrentState = DS_Playing`. Broadcast `OnNodeEntered(FDialogueNode)` → UI renders text, begins typewriter. Play `VoiceCue` if assigned (soft-load on demand). Await `IA_DialogueAdvance`.
5. **Choice node**: Filter `Choices` by `AvailabilityConditions` (hide failing with `bHideIfUnavailable == false`). Set `CurrentState = DS_AwaitingChoice`. Broadcast `OnChoicesPresented(TArray<FDialogueChoice>)`.

---

**DC-9: Runtime Lifecycle — SelectChoice()**

`UDialogueSubsystem::SelectChoice(int32 ChoiceIndex)`:

1. Validate index against visible list bounds. Ignore out-of-range input.
2. Set `CurrentState = DS_Resolving`.
3. Record the choice: write `DialogueChoices[ChoiceID] = 1` to the active `UAltynSaveGame` instance.
4. Fire all `FDialogueConsequence` entries on the selected choice, in declaration order, synchronously same frame (DC-10).
5. Call `StepToNode(SelectedChoice.NextNodeID)`.

---

**DC-10: Consequence Resolution Order**

Consequences fire synchronously, same frame, in declaration order:

1. **TrustDelta** — `UAltynSaveGame::ClanTrustLevels[ClanID] = clamp(current + delta, 0.0, 100.0)`
2. **SaveFlagWrite** — `UAltynSaveGame::DialogueChoices[FlagName] = FlagValue`
3. **MissionAdvance** — Broadcast `OnMissionAdvance(FName)`. *(Provisional — Mission Tracker GDD not yet authored.)*
4. **MemoryUnlock** — Broadcast `OnMemoryUnlock(FName)`. *(Provisional — Memory System GDD not yet authored.)*
5. **CustomEvent** — Broadcast `OnCustomDialogueEvent(FGameplayTag)`. Any system may subscribe.
6. After all consequences in the block: if any had `bTriggerAutoSave == true`, call `USaveLoadSubsystem::TriggerSignificantSave()`. Save data is fully written before this call.

---

**DC-11: Runtime Lifecycle — EndDialogue()**

`UDialogueSubsystem::EndDialogue()`:

1. Fire any remaining `Consequences` on `CurrentNode`.
2. If `bOneShot == true`: write `DialogueChoices[TreeID_completed] = 1`.
3. Remove `IMC_Dialogue` from Enhanced Input.
4. Broadcast `OnDialogueEnded()`. Subscribers: Camera (restore prior framing), Combat (re-evaluate `bInCombat`), Stealth (resume detection gauge).
5. Set `CurrentState = DS_Idle`. Clear `CurrentTree` and `CurrentNode`.

### States and Transitions

```
DS_Idle
  │  RequestDialogue() passes all guards
  ▼
DS_Playing  ◄── Condition/Event nodes resolved inline (no state change)
  │  Linear node: awaiting IA_DialogueAdvance
  │  Choice node: choices built
  ▼
DS_AwaitingChoice
  │  SelectChoice() called
  ▼
DS_Resolving  (consequences fire synchronously, same frame)
  │  StepToNode(NextNodeID) called
  ├──► DS_Playing  (more nodes remain)
  └──► DS_Complete (NextNodeID == NAME_None)
         │  EndDialogue() completes
         ▼
       DS_Idle
```

**Legal transition table:**

| From | To | Trigger |
|---|---|---|
| DS_Idle | DS_Playing | `RequestDialogue()` passes all guards |
| DS_Playing | DS_AwaitingChoice | Choice node entered |
| DS_Playing | DS_Playing | Condition/Event node resolved inline |
| DS_Playing | DS_Complete | `NextNodeID == NAME_None` |
| DS_AwaitingChoice | DS_Resolving | `SelectChoice()` called |
| DS_Resolving | DS_Playing | `StepToNode()` — more nodes remain |
| DS_Resolving | DS_Complete | `StepToNode(NAME_None)` — tree exhausted |
| DS_Complete | DS_Idle | `EndDialogue()` completes |

Any call that would produce an illegal transition is silently rejected.

### Interactions with Other Systems

| System | What Dialogue Needs | What Dialogue Provides |
|---|---|---|
| **Input System** | `IMC_Dialogue` context (4 bindings: `IA_DialogueAdvance`, `IA_DialoguePrevChoice`, `IA_DialogueNextChoice`, `IA_DialogueCancel`). Added/removed dynamically at priority 2. | Nothing — Input System is upstream only. |
| **Save/Load System** | `UAltynSaveGame` instance for reading/writing `DialogueChoices` + `ClanTrustLevels`. `TriggerSignificantSave()` for auto-save. | Writes `DialogueChoices` + `ClanTrustLevels`. Signals save trigger on `bTriggerAutoSave` consequences. |
| **Camera System** | `OnDialogueStarted(AActor*)` + `OnDialogueEnded()` delegates. Camera subscribes and transitions framing. | Broadcasts delegates. Does not dictate camera behavior — Camera owns its response. |
| **Combat System** | Subscribes to `OnDialogueStarted` to suspend `bInCombat`; `OnDialogueEnded` to re-evaluate. | Broadcasts `OnDialogueStarted` / `OnDialogueEnded`. |
| **Stealth System** | `UAltynStealthSubsystem::PauseGauge()` / `ResumeGauge()` called on `OnDialogueStarted` / `OnDialogueEnded`. *(Cross-system flag: Stealth GDD must expose these as public API.)* | Calls Pause/ResumeGauge via public API. No access to stealth internals. |
| **Audio System** | No direct API call. Voice cues play via `UAudioComponent` on speaker actor. Music state (Calm during dialogue) handled by Audio subscribing to `OnDialogueStarted`. | Voice cue playback on speaker actor. |
| **NPC Faction & Trust System** | Not yet authored (Layer 3). At MVP, trust lives in `UAltynSaveGame::ClanTrustLevels`. Faction system reads this at its own design time. | Writes `TrustDelta` consequences directly to `UAltynSaveGame`. |
| **Memory System** | Not yet authored (Layer 3). Subscribes to `OnMemoryUnlock(FName MemoryID)` when authored. | Broadcasts `OnMemoryUnlock` delegate. Does not unlock memories directly. |
| **Objective/Mission Tracker** | Not yet authored (Layer 3). Subscribes to `OnMissionAdvance(FName MissionID)` when authored. | Broadcasts `OnMissionAdvance` delegate. |
| **Dialogue UI** | `OnNodeEntered(FDialogueNode)` → renders text + typewriter. `OnChoicesPresented(TArray<FDialogueChoice>)` → renders choice list. `OnDialogueEnded()` → tears down. | Broadcasts all display delegates. Owns no UI logic. |

## Formulas

**Formula 1: Trust Delta Application**

```
NewTrustLevel = clamp(CurrentTrustLevel + TrustDelta, 0.0, 100.0)
```

| Variable | Type | Range | Definition |
|---|---|---|---|
| `CurrentTrustLevel` | float | [0.0, 100.0] | Value read from `UAltynSaveGame::ClanTrustLevels[ClanID]` before this consequence fires |
| `TrustDelta` | float | [-100.0, +100.0] | Signed value authored in `FDialogueConsequence.TrustDelta`; positive = trust gained, negative = trust lost |
| `NewTrustLevel` | float | [0.0, 100.0] | Value written back to `ClanTrustLevels[ClanID]` after clamping |

**Authoring guidance:** Typical single-dialogue deltas: ±5 (minor acknowledgment), ±10 (meaningful interaction), ±20 (significant deed or betrayal). Starting trust for all clans: `DefaultClanTrust = 50.0` (defined in Save/Load GDD). A clan at 0.0 or 100.0 does not clamp further — no negative trust or overflow.

**Example:**
- Current = 45.0, TrustDelta = +12.0 → New = 57.0
- Current = 95.0, TrustDelta = +12.0 → New = 100.0 (clamped)
- Current = 8.0, TrustDelta = -20.0 → New = 0.0 (clamped)

---

**Formula 2: Prerequisite Condition Evaluation**

A `TArray<FDialoguePrecondition>` passes when **all** entries evaluate to `true`. An empty array always passes.

```
TreeAccessible = ∀ condition ∈ Prerequisites: EvaluateCondition(condition) == true
ChoiceVisible  = bHideIfUnavailable == false
               ? ∀ condition ∈ AvailabilityConditions: EvaluateCondition(condition) == true
               : true  (always rendered — greyed-out mode)
```

**Per-condition evaluation:**

| Condition Type | Passes When |
|---|---|
| `FlagEquals` | `DialogueChoices[TargetID] == CompareValue` |
| `FlagGreaterThan` | `DialogueChoices[TargetID] > CompareValue` |
| `TrustAbove` | `ClanTrustLevels[TargetID] > float(CompareValue)` |
| `TrustBelow` | `ClanTrustLevels[TargetID] < float(CompareValue)` |
| `MemoryUnlocked` | Memory System confirms `TargetID` is in unlocked set *(Provisional)* |
| `ChapterEquals` | Current chapter index == `CompareValue` |

For `FlagEquals` and `FlagGreaterThan`: if `TargetID` key is absent from `DialogueChoices`, treat as value 0.

---

**Formula 3: Visible Choice Count**

```
VisibleChoices = { c ∈ Choices : c.bHideIfUnavailable == false
                                  AND ∀ cond ∈ c.AvailabilityConditions: EvaluateCondition(cond) }
               ∪ { c ∈ Choices : c.bHideIfUnavailable == true }
```

| Variable | Range | Notes |
|---|---|---|
| `Choices` | 1–4 authored per node | Hard author limit: 4 choices max per Choice node |
| `VisibleChoices` | 1–4 | Must be ≥ 1 when `DS_AwaitingChoice` is entered. If 0, system logs a warning and calls `EndDialogue()`. |

---

**Formula 4: Typewriter Character Reveal Rate**

```
RevealInterval = 1.0 / CharactersPerSecond   (seconds per character)
```

| Variable | Type | Default | Range | Definition |
|---|---|---|---|---|
| `CharactersPerSecond` | float | 40.0 | [15.0, 120.0] | Characters revealed per second |
| `RevealInterval` | float | 0.025s | — | `FTimerManager::SetTimer()` interval per character |

Implementation: timer fires each `RevealInterval` seconds, appending the next character via `UTextBlock::SetText()`. Pressing `IA_DialogueAdvance` during typewriter playback completes the text immediately (reveal all remaining characters, cancel timer) rather than advancing the node.

**Example:** 40 characters/second → 0.025s interval. A 120-character line fully reveals in 3.0 seconds.

## Edge Cases

**EC-1: VisibleChoices == 0 at DS_AwaitingChoice**
If all choices on a Choice node fail their `AvailabilityConditions` and all have `bHideIfUnavailable = false`, the visible list is empty. The system logs a warning, calls `EndDialogue()` immediately, and returns the player to normal input. This is an authoring error — every Choice node must guarantee at least one visible option under all reachable game states.

**EC-2: RequestDialogue() Called During Active Dialogue**
If `CurrentState != DS_Idle`, `RequestDialogue()` returns `false` and the new tree is silently ignored. No queuing. Any NPC interaction trigger that fires during active dialogue is discarded.

**EC-3: Player Dies During Dialogue**
`OnPlayerDeath` causes `EndDialogue()` to fire immediately. Consequences already resolved (fired synchronously at selection time) are in the in-memory save object. Consequences from choices made after the last checkpoint save are not on disk — the death-load restores to the checkpoint. The player re-plays the dialogue. No consequence is reversed on death.

**EC-4: Condition Node With No Passing Branch**
If no `ConditionBranches` entry passes, `StepToNode()` finds no target, logs a warning, and calls `EndDialogue()`. Authoring rule: every Condition node must include a fallback branch with an empty `Conditions` array (always passes) as the final entry.

**EC-5: Voice Cue Missing or Failed to Load**
If `VoiceCue` is null or soft-load fails, the node proceeds normally without audio. No error UI, no stall. Allows dialogue to function during development before voice assets exist.

**EC-6: bOneShot Tree Re-Entry Attempt**
If `bOneShot == true` and `DialogueChoices[TreeID_completed] == 1` exists, `RequestDialogue()` returns `false`. The NPC does not open dialogue. The interaction prompt is controlled by the NPC Faction system — Dialogue System does not suppress it directly (NPC Faction GDD must handle this).

**EC-7: Save Object Not Available**
If `UAltynSaveGame` is null when a consequence fires (startup race), the consequence is skipped and a warning is logged. `bTriggerAutoSave` is also suppressed. Dialogue continues. This is a startup-only case; the save reference is stable for the session lifetime once initialized.

**EC-8: TrustDelta ClanID Not in Save Object**
If `ClanTrustLevels[ClanID]` key is absent, treat `CurrentTrustLevel = 50.0` (DefaultClanTrust from Save/Load GDD) and insert the key. Handles dialogue that fires before all clan trust entries have been initialized.

**EC-9: IMC_Dialogue Added Twice**
`RequestDialogue()` guards ensure dialogue cannot be entered while active, so `IMC_Dialogue` is never added twice through normal use. If added externally by error, the duplicate is harmless — Enhanced Input applies highest-priority active mapping; same-priority duplicates do not cause double input.

**EC-10: Dialogue Started While bInCombat == true**
`RequestDialogue()` does not check combat state. The Combat System is responsible for suspending combat behavior on `OnDialogueStarted`. If the Combat System fails to respond, combat continues alongside dialogue. This is an integration error, not a Dialogue System error.

## Dependencies

**Upstream (systems this GDD depends on):**

| System | GDD | What this system consumes |
|---|---|---|
| **Input System** | `input-system.md` ✓ | `IMC_Dialogue` context with 4 bindings (`IA_DialogueAdvance`, `IA_DialoguePrevChoice`, `IA_DialogueNextChoice`, `IA_DialogueCancel`). Dynamic add/remove via Enhanced Input API at priority 2. |
| **Save/Load System** | `save-load-system.md` ✓ | `UAltynSaveGame::DialogueChoices: TMap<FName, int32>` and `ClanTrustLevels: TMap<FName, float>`. `USaveLoadSubsystem::TriggerSignificantSave()` for auto-save. `DefaultClanTrust = 50.0` as initial trust value. |

**Downstream (systems that depend on this GDD):**

| System | GDD | What that system consumes |
|---|---|---|
| **NPC Faction & Trust System** | Not yet authored | Reads `ClanTrustLevels` from `UAltynSaveGame`. Must handle suppressing NPC interaction prompts when `bOneShot` trees are completed (EC-6). |
| **Memory System** | Not yet authored | Subscribes to `OnMemoryUnlock(FName MemoryID)`. Owns unlock logic — Dialogue fires the event only. |
| **Objective/Mission Tracker** | Not yet authored | Subscribes to `OnMissionAdvance(FName MissionID)`. Owns mission state — Dialogue fires the event only. |
| **Dialogue UI** | `dialogue-ui.md` (Layer 5, not yet authored) | Subscribes to `OnNodeEntered`, `OnChoicesPresented`, `OnDialogueEnded`. Owns all rendering. |
| **Cinematic/Cutscene System** | Not yet authored | Must not activate sequences during `DS_Playing` or `DS_AwaitingChoice`. Must define behavior if `CS_Cinematic` is triggered while dialogue is active. |

**Cross-system flags generated by this GDD:**

| Target GDD | Required Change |
|---|---|
| **Input System** | Add `IMC_Dialogue` with 4 bindings: `IA_DialogueAdvance` (Gamepad A / KB Space or Enter), `IA_DialoguePrevChoice` (D-pad Up / KB Arrow Up), `IA_DialogueNextChoice` (D-pad Down / KB Arrow Down), `IA_DialogueCancel` (Gamepad B / KB Escape). Priority 2. |
| **Stealth System** | Expose `PauseGauge()` and `ResumeGauge()` as public API on `UAltynStealthSubsystem`. |
| **Camera System** | Subscribe to `OnDialogueStarted(AActor*)` and `OnDialogueEnded()`. Camera GDD must define its framing response. |
| **Save/Load System** | Confirm `DialogueChoices: TMap<FName, int32>` and `ClanTrustLevels: TMap<FName, float>` are in the `UAltynSaveGame` schema under their canonical field names. |

## Tuning Knobs

| Knob | Default | Safe Range | What It Affects |
|---|---|---|---|
| `CharactersPerSecond` | 40.0 | [15.0, 120.0] | Typewriter reveal speed. Lower = more deliberate, cinematic feel. Higher = faster pacing. Below 15 feels unbearably slow; above 120 text pops in effectively instantaneously. |
| `DefaultClanTrust` | 50.0 | [0.0, 100.0] | Starting trust level for all clans (owned by Save/Load GDD). Changing shifts the entire trust economy; adjust only at balance pass. |
| `MaxChoicesPerNode` | 4 | [2, 4] | Hard cap on choices per Choice node. Affects Dialogue UI layout. Do not raise above 4 without updating the Dialogue UI spec. |
| `IMC_DialoguePriority` | 2 | [1, 5] | Enhanced Input priority for `IMC_Dialogue`. Must be above `IMC_OnFoot` (1) and below any system-pause UI. Changing can cause input conflicts with other active IMCs. |
| `bDefaultHideUnavailable` | false | true / false | Project-wide default for `FDialogueChoice.bHideIfUnavailable`. `false` = hide unavailable choices (no moral markers, design default). `true` = show greyed-out options everywhere. Changing after authoring begins requires auditing all existing trees. |
| `TrustDeltaMinor` | 5.0 | [1.0, 15.0] | Authoring reference for minor trust interactions. Design guideline for content authors, not a system constant. |
| `TrustDeltaMajor` | 20.0 | [10.0, 50.0] | Authoring reference for significant trust interactions. Design guideline for content authors, not a system constant. |

## Visual/Audio Requirements

**Voice:**
- All NPC dialogue lines reference a `TSoftObjectPtr<USoundBase>` voice cue. Loaded on demand when the node is entered; unloaded when `EndDialogue()` completes.
- Player response lines are silent — `SpeakerID == "player"` nodes have null `VoiceCue` by design.
- Voice cue naming convention: `VO_{NPC_ID}_{TreeID}_{NodeID}` (e.g., `VO_Elder_SakhaTreeIntro_greet_01`)

**Audio state:**
- Audio System receives `OnDialogueStarted` and transitions music to Calm if not already. Music transitions are the Audio System's responsibility; this system owns only the broadcast.
- Ambient SFX (wind, birds, fire) continue during dialogue. Attenuation during narrative passages is owned by the Audio System's Narrative Sound Mix.

**No visual transitions owned by this system:**
- Camera transitions are owned by the Camera System (subscribes to `OnDialogueStarted` / `OnDialogueEnded`).
- Fade-to-black, letterbox, or cinematic bars are owned by the Cinematic/Cutscene System if used.
- The Dialogue System does not modify world lighting, post-processing, or character animation directly.

## UI Requirements

The Dialogue UI is a downstream system (`dialogue-ui.md`, Layer 5 — not yet authored). This GDD defines the data contract; the UI GDD defines the visual implementation.

**Data the Dialogue System provides to the UI:**

| Delegate | Payload | UI Responsibility |
|---|---|---|
| `OnNodeEntered(FDialogueNode)` | Full node struct: `SpeakerID`, `BodyText`, `VoiceCue` | Renders speaker name label + begins typewriter effect on `BodyText` |
| `OnChoicesPresented(TArray<FDialogueChoice>)` | Filtered visible choices only (1–4) | Renders choice list; no unavailable choices are ever in this array |
| `OnDialogueEnded()` | No payload | Tears down the dialogue UI panel |

**Constraints the Dialogue UI GDD must satisfy:**
- Choice list must support 1–4 visible entries without layout breaking.
- No choice is visually distinguished as "correct," "moral," or "optimal." All choices render identically (same color, same weight, same styling).
- Player line attribution: speaker name must not display the player's name. Player lines use a blank label or a contextual identifier (e.g., no speaker label). The Dialogue UI GDD must specify the exact treatment.
- Typewriter completion-on-advance must be instant — reveal the full string and cancel the timer in one operation with no animation frame between.
- Both gamepad and keyboard navigation must work for choice selection (`IA_DialoguePrevChoice` / `IA_DialogueNextChoice` to cycle; `IA_DialogueAdvance` to confirm).

## Acceptance Criteria

**AC-1: Dialogue starts correctly**
Given an NPC with a valid `UDialogueTree` and all `Prerequisites` passing, when the player interacts, `RequestDialogue()` opens dialogue: the UI appears, root node text displays, `IMC_Dialogue` is active (confirmed via Input Debugger), and `IMC_OnFoot` input is suppressed.

**AC-2: Linear node advances on input**
During a Linear node, pressing `IA_DialogueAdvance` advances to `NextNodeID`. If the typewriter is still playing, first press completes the text; second press advances the node.

**AC-3: Choice node presents filtered options**
A Choice node with 3 authored choices where 1 fails `AvailabilityConditions` and has `bHideIfUnavailable = false` shows exactly 2 choices in the UI. The unavailable choice does not appear in any form.

**AC-4: Choice consequence writes to save object**
After selecting a choice with a `TrustDelta` consequence, `UAltynSaveGame::ClanTrustLevels[ClanID]` reflects the updated value in the same frame. Verified via debug output or save-state inspection.

**AC-5: Trust delta clamps correctly**
Trust 95.0 + TrustDelta +20.0 = 100.0 (not 115.0). Trust 5.0 + TrustDelta -20.0 = 0.0 (not -15.0).

**AC-6: bOneShot tree cannot replay**
After completing a `bOneShot = true` tree, interacting with the same NPC again does not open dialogue. `RequestDialogue()` returns false and no UI appears.

**AC-7: bTriggerAutoSave fires after consequence block**
A consequence with `bTriggerAutoSave = true` calls `TriggerSignificantSave()` after all consequences in the block have resolved. The save object contains the updated values at the moment of the save call.

**AC-8: Player death during dialogue ends cleanly**
If `OnPlayerDeath` fires during `DS_AwaitingChoice`, `EndDialogue()` is called, `IMC_Dialogue` is removed, and the dialogue UI is not left on screen after the respawn flow completes.

**AC-9: No greyed-out options by default**
A choice with failing `AvailabilityConditions` and `bHideIfUnavailable = false` does not appear in the choice list in any form — no greyed, faded, or visually distinct option is visible.

**AC-10: Condition node is invisible to the player**
A Condition node in a tree does not produce a UI pause, blank frame, or perceptible delay. From the player's perspective, the preceding node resolves directly to the branch target.

**AC-11: Empty VisibleChoices fallback**
A Choice node where all choices fail conditions and all have `bHideIfUnavailable = false` causes the system to call `EndDialogue()`, close the UI, restore normal gameplay input, and emit a warning log entry.

**AC-12: Typewriter rate matches CharactersPerSecond**
At `CharactersPerSecond = 40.0`, a 40-character line takes approximately 1.0 second to fully reveal. Advance input during typewriter playback completes the text immediately.

**AC-13: Dialogue does not start during active dialogue**
While `DS_AwaitingChoice` is active, a second `RequestDialogue()` call returns false. The first dialogue continues uninterrupted.

**AC-14: Voice cue absent does not stall node**
A Linear node with a null `VoiceCue` plays no audio but displays text and accepts advance input normally. No error state, no hang.

## Open Questions

**OQ-1: IMC_Dialogue cancel behavior**
`IA_DialogueCancel` is bound in `IMC_Dialogue` but has no defined behavior yet. Options: (a) skip to the next node, (b) close dialogue entirely for non-significant trees, (c) do nothing (cancel not exposed in-game). Must be resolved before the Dialogue UI GDD is authored.

**OQ-2: Simultaneous trust delta consequences for same ClanID**
Two `TrustDelta` consequences for the same `ClanID` on a single node fire sequentially — the second reads the value written by the first. This is correct behavior but requires a documented test case to confirm it isn't treated as a race condition.

**OQ-3: MissionAdvance and MemoryUnlock FName canonical IDs**
`MissionAdvance` and `MemoryUnlock` consequence types broadcast delegates to systems not yet authored. When those GDDs are designed, confirm the `FName` identifiers used in `FDialogueConsequence` match the canonical IDs those systems expect. A registry entry may be needed to prevent cross-GDD naming drift.

**OQ-4: Chapter condition authority**
`ChapterEquals` compares against a current chapter index. No system has been designated as the authority for this value. Resolve when the Mission Tracker or global game state GDD is authored.

**OQ-5: NPC interaction prompt suppression after bOneShot completion**
EC-6 defers prompt suppression to the NPC Faction & Trust System GDD. That GDD must define whether prompts are hidden based on `DialogueChoices[TreeID_completed]` or whether a separate visibility flag is needed.
