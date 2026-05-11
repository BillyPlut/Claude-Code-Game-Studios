---
name: Project Altyn Adam — Camera System Design Context
description: Upstream GDD decisions locked before camera authoring; key cross-system constraints the camera GDD must respect
type: project
---

Camera System GDD is in active design as of 2026-05-10. The skeleton file exists at design/gdd/camera-system.md.

Upstream locked values the camera GDD depends on:
- Character Controller exposes: bIsSprinting (bool), bIsCrouching (bool)
- StandingHalfHeight = 88 UU, CrouchedHalfHeight = 60 UU (32% height reduction)
- Health & Stamina: NearDeathThreshold = 10.0 HP; WoundedThreshold = 30.0 HP
- Near-death camera: Dutch tilt 1.5°, orbit lowers 4–6 UU, over 1.5s, one-shot on crossing, clears on OnWoundedStateExited
- Hit micro-impulse: 2–4 UU opposite hit vector, returns 0.18s, critically-damped spring, no oscillation
- Delegates available: OnWoundedStateExited, OnPlayerDeath, OnWoundedStateEntered, OnStaminaExhausted
- Input System: IA_Look, IA_LockOn not yet in input GDD (needs adding); IA_AimBow uses IMC_AimBow layered over base
- No IA_LockOn entry exists in the Input System GDD — must be added there as a cross-system fact

**Why:** These values flow from Character Controller GDD (Complete) and Health & Stamina GDD (Complete). Camera GDD must not contradict them.
**How to apply:** Before writing any camera formula that touches health states, movement states, or input actions, verify against these locked values.
