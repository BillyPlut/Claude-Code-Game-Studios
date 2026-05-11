# Enemy AI System

> **Status**: In Design
> **Author**: Solo Dev + Claude Code Game Studios
> **Last Updated**: 2026-05-11
> **Implements Pillar**: Pillar 1 (clan trust through deeds — enemies are the stakes that make deeds meaningful) + Pillar 3 (final battle army — enemy behavior defines the quality of the culminating confrontation)

## Overview

The Enemy AI System governs how enemies move, perceive, and act within each encounter space. It owns three behavioral domains: **patrol** (how enemies move when unalerted, including waypoint routing and idle scan behavior), **alert response** (investigation, alarm-calling, and the group coordination logic that notifies nearby allies when any enemy enters Alert), and **combat behavior** (attack patterns, positioning, stagger response, and retreat thresholds). The system does not own detection — perception gauges and alert-state thresholds belong to the Stealth System. It does not own damage — hit detection and damage application belong to the Combat System. What it owns is the behavioral consequence of those inputs: what an enemy *does* when the Stealth System writes `AlertTier = Alert` to its Blackboard, or when the Combat System fires `OnEnemyParried`.

Each enemy is driven by `AAltynEnemyAIController : AAIController` running a Behavior Tree (BT) that branches on the Blackboard key `AlertTier`. Patrol, Investigate, Alarm+Engage, and Search are the four root branches — one per alert state. Group coordination is explicit and author-free: when any enemy transitions to Alert, it broadcasts a `FAINoiseEvent` (an alarm shout) at its own position; Suspicious and Unaware enemies within `AlarmRange` receive it and escalate proportionally. MVP enemy types are **StandardGuard** (grounded melee patrol) and **Archer** (ranged, repositions under fire). Scout and horse-mounted variants are out of MVP scope. Enemy HP and damage values are specified here, calibrated against the Combat System's `LightDamage_base = 25.0` so that standard encounters feel dangerous but survivable with skill. The player experiences this system as opponents who read like soldiers — alert, organized, credible — not as patrol animations with collision boxes.

## Player Fantasy

**"The Land Knows Them as Strangers"**

The Persians are credible soldiers. They are not credible to *this place*. They wear too much. They move in formations the steppe does not reward. Their eyes are on the path, because in the Zagros mountains that is where the danger is. The boy's eyes are on the ridgelines — always the ridgelines — because that is where wolves come from, where weather comes from, and where the boy with a knife would go if he were you.

The system must make the player feel the gap. A guard who walks past a clear silhouette on the ridge because he was scanning the trail is not stupid — he is *not from here*. An archer who chooses a high stone outcrop for cover instead of a fold in the grass is choosing the cover that *looks like cover*, not the cover that hides you from a child of the steppe. An alarm call that travels in a straight line from soldier to soldier along the path is the disciplined communication of a professional military unit — and it is slower and less intelligent than the way news travels in grass.

The boy is not faster than them. He is not stronger. He does not outfight them. What he has is locality — the same inheritance the land gave him for stealth, applied to the other side of the encounter. He knows what they will do because he knows what the land punishes and what it rewards, and they do not know yet what this land is.

When group coordination triggers — when the alarm shout goes up and soldiers converge on his last known position — the player must feel two things simultaneously: *they are organized and they are coming*, and *they are coming to the wrong place*. The patrol becomes a unit; the unit remains foreign. The more competent they act, the more misplaced their competence is. That tension is the system.

This is not about enemy incompetence. They will kill the boy if he gets it wrong. The fantasy is *displaced*, not inferior. The failure mode to avoid: if the player ever thinks "this enemy AI is bad," the system has failed. What they should feel is: "these soldiers were trained for somewhere else."

The enemy AI system is the negative space of the steppe's generosity. The land is on your side as knowledge — this system is the consequence of the enemy not having that knowledge, applied to every patrol route, every cover choice, every alarm propagation, every attack pattern. Together, the two systems make a single claim: *the steppe has been here longer than Persia, and it remembers its own.*

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
