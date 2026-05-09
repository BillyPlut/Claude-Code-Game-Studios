# Unreal Engine — Version Reference

Last verified: 2026-05-08

| Field | Value |
|-------|-------|
| **Engine Version** | Unreal Engine 5.7 |
| **Latest Hotfix** | 5.7.4 (March 10, 2026) |
| **Release Date** | ~Early 2026 |
| **Project Pinned** | 2026-05-08 |
| **LLM Knowledge Cutoff** | May 2025 |
| **Risk Level** | HIGH — versions 5.5, 5.6, 5.7 are beyond LLM training data |

## Knowledge Gap Warning

The LLM's training data covers Unreal Engine up to approximately **~5.3 / early 5.4**.
Versions 5.5, 5.6, and 5.7 introduced breaking changes the model does NOT know about.
Always cross-reference the files in this directory before accepting any API suggestions.

## Post-Cutoff Version Timeline

| Version | Release | Risk Level | Key Theme |
|---------|---------|------------|-----------|
| 5.5 | November 2024 | HIGH | Interchange default, Enhanced Input remapping overhaul, GAS NonInstanced deprecated |
| 5.6 | Mid 2025 | HIGH | Lumen HWRT focus, UMG animation refactor, Mass parallel default, VSM changes |
| 5.7 | Early 2026 | HIGH | Substrate production-ready, GAS loose tags removed, Iris UReplicationBridge removed, Linux SDL3 |

## Files in This Directory

| File | Contents |
|------|----------|
| `VERSION.md` (this file) | Version pin, risk level, quick reference |
| `breaking-changes.md` | Version-by-version breaking changes (5.4 → 5.7) |
| `deprecated-apis.md` | "Don't use X → Use Y" reference tables |
| `current-best-practices.md` | New best practices introduced since training cutoff |

## Critical Removals (must know before writing any code)

1. **GAS**: `AddReplicatedLooseGameplayTags()` / `RemoveReplicatedLooseGameplayTags()` — **REMOVED in 5.7**
2. **Enhanced Input**: `PlayerMappableOptions` struct members — **DELETED in 5.5**
3. **Iris**: `UReplicationBridge` — **REMOVED in 5.7**; use `UObjectReplicationBridge` directly
4. **Lumen**: SWRT mesh SDF detail tracing — **deprecated**; `r.Lumen.TraceMeshSDFs` defaults to 0 in 5.7
5. **GAS**: `EGameplayAbilityInstancingPolicy::NonInstanced` — **deprecated in 5.5**

## Verified Sources

- Release notes 5.5: https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5.5-release-notes
- Release notes 5.7: https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-7-release-notes
- Breaking changes KB: https://dev.epicgames.com/community/learning/knowledge-base/oDEB/unreal-engine-ue5-breaking-and-noteworthy-changes
