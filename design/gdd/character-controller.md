# Character Controller

> **Status**: In Design
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
