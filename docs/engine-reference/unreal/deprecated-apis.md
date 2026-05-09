# Unreal Engine Deprecated & Removed APIs — 5.4 → 5.7

Last verified: 2026-05-08

## REMOVED (Hard Errors — Will Not Compile)

| Removed API | Version | Replacement | Notes |
|-------------|---------|-------------|-------|
| `AddReplicatedLooseGameplayTags()` | 5.7 | Gameplay Effect or Ability Set tag replication | GAS |
| `RemoveReplicatedLooseGameplayTags()` | 5.7 | GE / Ability Set tag replication | GAS |
| `UReplicationBridge` (as base class) | 5.7 | `UObjectReplicationBridge` (direct dep) | Iris networking |
| `PlayerMappableOptions` struct members on `FEnhancedActionKeyMapping` | 5.5 | `UEnhancedInputUserSettings`, `UEnhancedPlayerMappableKeyProfile` | Enhanced Input |
| VSM HZB mode "1" | 5.6 | N/A — simplified code path | Remove references |

---

## DEPRECATED (Compile Warnings — Will Be Removed)

| Deprecated API | Since | Replacement | Notes |
|----------------|-------|-------------|-------|
| `EGameplayAbilityInstancingPolicy::NonInstanced` | 5.5 | `InstancedPerActor` or `InstancedPerExecution` | GAS |
| `FTicker` | 5.5 | `FTSTicker` | Same interface |
| `UUMGSequencePlayer` (direct subclass/reference) | 5.6 | New lightweight runner structs | UMG |
| `IMovieScenePlayer` interface | 5.6 | New animation player architecture | UMG |
| `r.UseClusteredDeferredShading` (CVar) | 5.7 | Current deferred path (default) | Rendering |
| `r.Mobile.UseClusteredDeferredShading` (CVar) | 5.7 | Current mobile deferred path | Rendering |
| Legacy FBX Import pipeline | 5.5 | Interchange plugin | Soft deprecation |
| `Profiler*` modules in Build.cs | 5.6 | `TraceLog`, `TraceAnalysis`, UnrealInsights | Build |

---

## BEHAVIOR CHANGED (Same API, Different Result)

| API / CVar | Change | Version |
|------------|--------|---------|
| `r.Lumen.ScreenProbeGather.MaxRayIntensity` | Default **40 → 10** | 5.6 |
| `r.VT.PageFreeThreshold` | Default **60 → 15** | 5.6 |
| `r.Shadow.Virtual.Cache 0` | Now "fast uncached path", not "full disable" | 5.6 |
| `r.Lumen.TraceMeshSDFs` | Default **1 → 0** (SWRT off) | 5.7 |
| `FMassProcessingPhase.bRunInParallelMode` | Default **false → true** | 5.6 |
| `r.Nanite.Culling.MinLOD` | Now **enabled by default** | 5.7 |

---

## Code Migration Snippets

### Enhanced Input (5.5+)
```cpp
// BEFORE — DELETED in 5.5:
FEnhancedActionKeyMapping Mapping;
Mapping.PlayerMappableOptions.Name = FName("Jump");  // ERROR

// AFTER:
UEnhancedInputUserSettings* Settings = GetEnhancedInputUserSettings();
UEnhancedPlayerMappableKeyProfile* Profile = Settings->GetCurrentKeyProfile();
Profile->SetPlayerMappedKey(FName("Jump"), FKey("Space"));
```

### GAS Loose Tags (5.7+)
```cpp
// BEFORE — REMOVED in 5.7:
AbilitySystemComponent->AddReplicatedLooseGameplayTags(Tags);    // ERROR
AbilitySystemComponent->RemoveReplicatedLooseGameplayTags(Tags); // ERROR

// AFTER: Apply via Gameplay Effect with Granted Tags configured in the GE asset.
FGameplayEffectContextHandle Context = ASC->MakeEffectContext();
FGameplayEffectSpecHandle Spec = ASC->MakeOutgoingSpec(TagGrantGE, 1.0f, Context);
ASC->ApplyGameplayEffectSpecToSelf(*Spec.Data.Get());
```

### GAS Instancing (5.5+)
```cpp
// BEFORE — deprecated in 5.5:
InstancingPolicy = EGameplayAbilityInstancingPolicy::NonInstanced; // WARNING

// AFTER:
InstancingPolicy = EGameplayAbilityInstancingPolicy::InstancedPerActor;
```

### Threading (5.5+)
```cpp
// BEFORE — deprecated in 5.5:
FTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateUObject(this, &UMyClass::Tick));

// AFTER:
FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateUObject(this, &UMyClass::Tick));
```
