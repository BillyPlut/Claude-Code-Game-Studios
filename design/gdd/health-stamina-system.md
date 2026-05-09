# Health & Stamina System

> **Status**: In Design
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

[To be designed]

### States and Transitions

[To be designed]

### Interactions with Other Systems

[To be designed]

## Formulas

[To be designed]

## Edge Cases

[To be designed]

## Dependencies

[To be designed]

## Tuning Knobs

[To be designed]

## Visual/Audio Requirements

[To be designed]

## UI Requirements

[To be designed]

## Acceptance Criteria

[To be designed]

## Open Questions

[To be designed]
