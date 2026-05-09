# Unreal Engine Breaking Changes — 5.4 → 5.7

Last verified: 2026-05-08

## UE 5.5 (November 2024)

| Area | Change | Action Required |
|------|--------|-----------------|
| **FBX Import** | Interchange enabled by default; legacy FBX importer disabled | Validate all FBX asset imports; some 5.4 files may need re-export from DCC |
| **Enhanced Input** | `PlayerMappableOptions` struct members on `FEnhancedActionKeyMapping` fully **deleted** | Migrate to `UEnhancedInputUserSettings` + `UEnhancedPlayerMappableKeyProfile` |
| **GAS** | `EGameplayAbilityInstancingPolicy::NonInstanced` deprecated, no further development | Migrate to `InstancedPerActor` or `InstancedPerExecution` |
| **Threading** | `FTicker` deprecated | Replace with `FTSTicker` (thread-safe, same interface) |
| **Lumen** | 60 Hz target, new scalability assumptions | Review `r.Lumen.ScreenProbeGather` settings if tuned for 5.4 |

---

## UE 5.6 (Mid 2025)

| Area | Change | Action Required |
|------|--------|-----------------|
| **Lumen MaxRayIntensity** | `r.Lumen.ScreenProbeGather.MaxRayIntensity` default changed **40 → 10** | Review scenes lit with old default; may appear darker |
| **Lumen SWRT** | SWRT designated deprecated — no future development | Begin migrating to HWRT pipeline |
| **Virtual Shadow Maps** | `r.Shadow.Virtual.Cache 0` behavior changed (now fast uncached, not full disable) | Update any configs using this CVar as a "disable" |
| **VSM HZB mode** | Mode "1" removed | Remove references to this HZB mode |
| **UMG Animations** | `UUMGSequencePlayer` refactored out; lightweight runner structs used | Remove direct references/subclasses of `UUMGSequencePlayer` |
| **IMovieScenePlayer** | Interface being deprecated | Avoid new dependencies; migrate away |
| **Virtual Texture** | `r.VT.PageFreeThreshold` default changed **60 → 15** | May affect VT streaming behavior |
| **Mass Entity** | `FMassProcessingPhase.bRunInParallelMode` now defaults to `true` | Audit all Mass Processors for thread safety |
| **Profiler modules** | `Profiler*` modules deprecated | Remove `Profiler` deps from `.Build.cs`; use UnrealInsights |
| **GPU Profiler** | In-editor `ProfileGPU` UI popup removed | Use log dump or UnrealInsights |

---

## UE 5.7 (Early 2026)

| Area | Change | Action Required |
|------|--------|-----------------|
| **Linux platform** | SDL2 → SDL3 migration | Rework SDL2 customizations for SDL3 API (if targeting Linux) |
| **Lumen SWRT** | `r.Lumen.TraceMeshSDFs` now defaults to **0**; SWRT fully removed from active development | Remove all SWRT/SDF-based Lumen tuning; use HWRT |
| **Clustered Deferred** | `r.UseClusteredDeferredShading` and `r.Mobile.UseClusteredDeferredShading` deprecated | Stop using these CVars |
| **Iris Replication** | `UReplicationBridge` base class **removed** | Update class hierarchies to `UObjectReplicationBridge` |
| **GAS Loose Tags** | `AddReplicatedLooseGameplayTags()` and `RemoveReplicatedLooseGameplayTags()` **REMOVED** | Use GE or Ability Set tag replication |
| **Nanite MinLOD** | `r.Nanite.Culling.MinLOD` now enabled by default | Review heavily-tuned Nanite LOD projects |

---

## Migration Checklist (5.4 → 5.7)

- [ ] Enhanced Input: Remove `PlayerMappableOptions` → `UEnhancedInputUserSettings`
- [ ] GAS Loose Tags: Replace `AddReplicatedLooseGameplayTags` / `RemoveReplicatedLooseGameplayTags` → GE/Ability Set tags
- [ ] GAS NonInstanced: `EGameplayAbilityInstancingPolicy::NonInstanced` → `InstancedPerActor`
- [ ] Iris: Remove `UReplicationBridge` subclassing → `UObjectReplicationBridge`
- [ ] Lumen: Remove SWRT tuning; configure HWRT; check `MaxRayIntensity` (now 10)
- [ ] VSM: Remove `r.Shadow.Virtual.Cache 0` "disable" usage
- [ ] UMG: Remove `UUMGSequencePlayer` direct references; avoid `IMovieScenePlayer`
- [ ] Threading: Replace `FTicker` with `FTSTicker`
- [ ] FBX: Validate imports under Interchange; re-export from DCC if needed
- [ ] Linux: Audit SDL2 customizations for SDL3 (if targeting Linux)
- [ ] Mass: Audit all Processors for thread safety (parallel default since 5.6)
- [ ] Build: Remove `Profiler*` module deps from `.Build.cs`
