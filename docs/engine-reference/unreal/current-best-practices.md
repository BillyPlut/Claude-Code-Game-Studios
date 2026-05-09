# Unreal Engine Current Best Practices — 5.5 → 5.7

Last verified: 2026-05-08

Follow these instead of anything the model suggests from pre-5.5 patterns.

---

## Rendering

### Lumen
- Use **Hardware Ray Tracing (HWRT)** as the primary Lumen path. Enable: `r.Lumen.HardwareRayTracing 1`
- SWRT (Software RT) is deprecated and defaults to **off** in 5.7.
- `r.Lumen.ScreenProbeGather.MaxRayIntensity` default is now **10** (was 40). Start tuning from 10.
- Lumen targets 60 Hz on high-end hardware (5.5+). Design scene complexity accordingly.

### Nanite
- **Do not use Nanite Foliage in production** — still Experimental in 5.7.
- For production foliage: use HISM (Hierarchical Instanced Static Mesh) with LODs.
- `r.Nanite.Culling.MinLOD` is enabled by default (5.7). Review MinLOD settings on heavily-tuned projects.

### Substrate Materials (5.7 — Production Ready)
- Use Substrate for new physically-based materials. Use `Substrate Slab BSDF` nodes.
- Enables multi-layer materials without Material Function hacks.
- Legacy material model still supported but Substrate is the forward path.

### MegaLights (5.7 Beta)
- Usable but not fully production-proven. Enable: `r.MegaLights 1`
- Use for scenes with hundreds of dynamic shadow-casting lights.
- Tune quality vs performance with `r.MegaLights.DownsampleFactor`.

---

## Gameplay Ability System (GAS)

- **Tag application**: Always use Gameplay Effects or Ability Set Granted Tags. Never use removed loose tag functions.
- **Instancing**: Default to `InstancedPerActor`. Use `InstancedPerExecution` only for fire-and-forget single-frame abilities.
- **Input binding**: Canonical pattern is `UInputAction` → `ASC->TryActivateAbilitiesByTag()`.
- **Tag hierarchies**: Use `UGameplayTagsManager::Get().RequestGameplayTagChildren()` for parent tag queries.

---

## Enhanced Input

- All remapping goes through `UEnhancedInputUserSettings` + `UEnhancedPlayerMappableKeyProfile`.
- Mark mappings with `bPlayerMappable = true` in the Input Mapping Context asset.
- Save/load remapping: serialize `UEnhancedInputUserSettings` state in your SaveGame object.

---

## Asset Import

- **Interchange** is the default importer (5.5+). Test all FBX and glTF assets.
- Configure Interchange pipelines in: `Project Settings → Plugins → Interchange → Import Pipelines`
- Prefer **glTF 2.0** for new assets — first-class Interchange support. FBX is a legacy path.

---

## Profiling

- **GPU**: Use `UnrealInsights` (Trace). Record with `Trace.Start file` console command.
- **CPU**: `stat unit`, `stat game`, then UnrealInsights for detail.
- **Build times**: Enable `Unreal Build Accelerator` (5.5+) for distributed C++ compilation.
- **Shader stutter**: `Automatic PSO Precaching` is enabled by default (5.5). Do not disable.

---

## Animation

- **Choosers**: Production-ready (5.5). Use for data-driven animation selection instead of large AnimBP switch logic.
- **Nondestructive Animation Layers**: Stable (5.5+). Prefer over AnimBP montage stacking for layered locomotion.
- **PCG**: Production-ready (5.7). Use PCG Editor Mode for procedural level decoration.

---

## Threading

- Replace `FTicker` with `FTSTicker` (same interface, thread-safe).
- Mass Entity processors must be thread-safe — `bRunInParallelMode` defaults to `true` since 5.6.
- For new async tasks: prefer `UE::Tasks::Launch()` over raw `FAsyncTask`.
