[English](./README.md) | [Tieng Viet](./README.vi.md)

# Vit.SpawnKit

`Vit.SpawnKit` is a lightweight Unity spawning and pooling toolkit built around three ideas:

- author spawn definitions with `ScriptableObject` assets
- reuse instances through pools instead of repeated instantiate/destroy
- keep gameplay code short with a static `SpawnKit` facade

The package targets Unity `2022.3` and currently focuses on runtime spawning with prefab-backed pools.

## What It Includes

- `SpawnManager` runtime service for registering spawnables, pooling, spawning, and despawning
- `SpawnKit` static facade for short gameplay-facing calls
- `SpawnableSO` for defining spawn sources, pool config, lifecycle defaults, and variant selection
- `SpawnCatalogSO` for auto-registration and prewarming
- `SpawnPresetSO` for reusable spawn presets, including collider-volume spawning
- sync and async batch spawning APIs
- timed despawn and `OnDisable`-driven return-to-pool lifecycle modes
- pooled instance callbacks through `ISpawnPoolCallbacks`
- pool maintenance APIs: stats, trim, clear, release-unused

## Current Scope

This package is usable today for prefab-based runtime spawning.

Current limitations in this package version:

- `SpawnSourceType.Addressables` is present in the API, but the package only contains conditional code paths guarded by `USE_ADDRESSABLES`. No `AddressablesSpawnFactory` implementation is included in this package.
- There are no custom editor tools in the package yet. Setup is done through standard Unity inspectors.
- Sample content in this repository lives outside the package at `Assets/SpawnModule/Sample`.

## Installation

You can use the package in one of these ways:

- `Embedded package`: place the folder directly at `Packages/com.vit.spawnkit`
- `Local package`: reference a folder outside the project with a `file:` dependency
- `Git package`: reference a repository URL from Package Manager

If the folder already exists at `Packages/com.vit.spawnkit`, Unity treats it as an embedded package and you do not need a manifest entry.

For a local package stored outside the project, `manifest.json` usually looks like this:

```json
{
  "dependencies": {
    "com.vit.spawnkit": "file:../path-to/com.vit.spawnkit"
  }
}
```

If the package is hosted in another repository, use the corresponding Git URL instead.

## Core Concepts

### 1. Spawnable

`SpawnableSO` defines what can be spawned and how its pool behaves.

It contains:

- a stable `SpawnKey`
- one or more prefab variants
- per-variant `count` or `weight`
- `PoolConfig` (`prewarmCount`, `maxSize`, `growStep`, `allowGrow`)
- default `SpawnLifecycle`
- default seed behavior

Variant behavior:

- if total configured `count` across variants is greater than `0`, SpawnKit follows that explicit plan
- if all variant `count` values are `0`, SpawnKit falls back to weighted random selection

### 2. Catalog

`SpawnCatalogSO` is a list of `SpawnableSO` assets that `SpawnManager` can auto-register on startup or prewarm on demand.

### 3. Preset

`SpawnPresetSO` packages a reusable spawn request:

- target `SpawnableSO`
- max batch count
- optional lifecycle override
- seed override
- volume-placement tuning
- behavior across one or multiple colliders

This is the easiest way to let designers control spawn behavior without touching code.

### 4. Manager

`SpawnManager` is the runtime service. It owns the pools and must exist in the scene.

At startup it:

- creates a `PoolsRoot` transform if none is assigned
- registers spawnables from the assigned catalog
- becomes `SpawnManager.Instance`

## Unity Setup

### Step 1. Create assets

Create the following assets from Unity's `Assets/Create/SpawnKit` menu:

- `Spawnable`
- `Spawn Catalog`
- `Spawn Preset`

### Step 2. Configure a `SpawnableSO`

Set:

- `Key`: logical id for lookup and debugging
- `Source Type`: `Prefab`
- `Prefab Variants`: add one or more prefabs
- `Pool Config`: set prewarm and pool size
- `Default Lifecycle`: choose `Manual`, `OnDisable`, or `AfterSeconds`

### Step 3. Create a catalog

Add your `SpawnableSO` assets to a `SpawnCatalogSO`.

### Step 4. Add `SpawnManager` to the scene

Create an empty GameObject, add `SpawnManager`, then assign:

- `Catalog`: optional but recommended
- `Pools Root`: optional; if empty, the manager creates one automatically

### Step 5. Optional preset setup

Create a `SpawnPresetSO` when you want reusable designer-configured spawning, especially for collider-volume spawning.

## Quick Start

### Spawn one object at a fixed pose

```csharp
using UnityEngine;
using Vit.SpawnKit.Api;
using Vit.SpawnKit.Data;
using Vit.SpawnKit.ScriptableObjects;

public sealed class SpawnOneExample : MonoBehaviour
{
    [SerializeField] private SpawnableSO spawnable;

    private void Start()
    {
        SpawnKit.SpawnOne(
            spawnable,
            position: transform.position,
            rotation: Quaternion.identity,
            parent: null,
            seed: 0,
            lifecycle: SpawnLifecycle.Manual);
    }
}
```

### Spawn a batch

```csharp
using UnityEngine;
using Vit.SpawnKit.Api;
using Vit.SpawnKit.Algorithms;
using Vit.SpawnKit.ScriptableObjects;

public sealed class BatchSpawnExample : MonoBehaviour
{
    [SerializeField] private SpawnableSO spawnable;

    private void Start()
    {
        var handle = SpawnKit.Spawn(
            spawnable,
            count: 20,
            parent: transform,
            algorithm: new SimplePointAlgorithm(transform.position, 5f));

        // Keep the handle if you want to despawn the whole batch later.
        Debug.Log(handle.IsEmpty ? "Nothing spawned" : $"Spawned {handle.Instances.Count} objects");
    }
}
```

### Spawn from a preset into one or more colliders

```csharp
using UnityEngine;
using Vit.SpawnKit.Api;
using Vit.SpawnKit.ScriptableObjects;

public sealed class PresetVolumeSpawnExample : MonoBehaviour
{
    [SerializeField] private SpawnPresetSO preset;
    [SerializeField] private Collider[] volumes;

    private void Start()
    {
        SpawnKit.Spawn(preset, volumes, transform);
    }
}
```

### Async spawn for large batches

```csharp
using System.Threading.Tasks;
using UnityEngine;
using Vit.SpawnKit.Api;
using Vit.SpawnKit.Algorithms;
using Vit.SpawnKit.ScriptableObjects;

public sealed class AsyncSpawnExample : MonoBehaviour
{
    [SerializeField] private SpawnableSO spawnable;

    private async void Start()
    {
        var handle = await SpawnKit.SpawnAsync(
            spawnable,
            count: 500,
            parent: transform,
            algorithm: new SimplePointAlgorithm(transform.position, 20f),
            maxPerFrame: 32);

        Debug.Log($"Spawned {handle.Instances.Count} pooled objects asynchronously");
    }
}
```

## API

The main public facade is `Vit.SpawnKit.Api.SpawnKit`.

### Spawn

- `Spawn(SpawnableSO spawnable, int count = 1, Transform parent = null, ISpawnAlgorithm algorithm = null, uint seed = 0, SpawnLifecycle? lifecycle = null)`
- `SpawnAsync(SpawnableSO spawnable, int count = 1, Transform parent = null, ISpawnAlgorithm algorithm = null, uint seed = 0, SpawnLifecycle? lifecycle = null, int maxPerFrame = 32, CancellationToken cancellationToken = default)`
- `Spawn(ISpawnRequestSource source)`
- `SpawnAsync(ISpawnRequestSource source, int maxPerFrame = 32, CancellationToken cancellationToken = default)`
- `Spawn(SpawnPresetSO preset, Collider volume, Transform parent = null)`
- `SpawnAsync(SpawnPresetSO preset, Collider volume, Transform parent = null, int maxPerFrame = 32, CancellationToken cancellationToken = default)`
- `Spawn(SpawnPresetSO preset, Collider[] volumes, Transform parent = null)`
- `SpawnAsync(SpawnPresetSO preset, Collider[] volumes, Transform parent = null, int maxPerFrame = 32, CancellationToken cancellationToken = default)`
- `Spawn(SpawnPresetSO preset, Transform parent = null)`
- `SpawnAsync(SpawnPresetSO preset, Transform parent = null, int maxPerFrame = 32, CancellationToken cancellationToken = default)`
- `SpawnOne(SpawnableSO spawnable, Vector3 position, Quaternion rotation, Transform parent = null, uint seed = 0, SpawnLifecycle? lifecycle = null)`

### Despawn

- `Despawn(GameObject instance)`
- `SpawnHandle.DespawnAll()`
- `SpawnHandle.Dispose()`

Use these when the lifecycle mode is `Manual`, or when you want to return a whole spawned batch explicitly.

### Prewarm And Registration

- `Prewarm(SpawnableSO spawnable)`
- `Prewarm(SpawnCatalogSO catalog)`

These methods force pool creation and prewarm inactive instances using each spawnable's `PoolConfig`.

### Pool Stats

- `GetPoolStats(SpawnableSO spawnable)`
- `TryGetPoolStats(SpawnableSO spawnable, out PoolStats stats)`
- `GetPoolStats(SpawnPresetSO preset)`

`PoolStats` contains:

- `active`
- `inactive`
- `total`
- `max`

### Pool Maintenance

- `Trim(SpawnableSO spawnable, int keepInactive = 0)`
- `Clear(SpawnableSO spawnable)`
- `ReleaseUnused(SpawnableSO spawnable)`
- `Trim(SpawnPresetSO preset, int keepInactive = 0)`
- `Clear(SpawnPresetSO preset)`
- `ReleaseUnused(SpawnPresetSO preset)`

Use them to shrink pools, clear inactive instances, or reduce memory back to the configured prewarm level.

## Volume Spawning

`SpawnPresetSO` uses collider-aware algorithms for placement.

Available behaviors:

- `ColliderVolumeAlgorithm`: place objects inside one collider
- `MultiColliderVolumeAlgorithm`: distribute a shared count across multiple colliders
- `PerVolumeColliderVolumeAlgorithm`: apply count per collider

Useful preset fields:

- `MaxCount`: max objects to spawn for one request
- `multiVolumeCountMode`
- `maxTryPerPoint`
- `candidatesPerPoint`
- `minDistance`
- `placementBufferCapacity`

When no collider is supplied, the preset falls back to a fixed pose at the chosen parent transform or world origin.

For direct `Spawn(...)` calls without a custom algorithm, the manager falls back to a zero-radius `SimplePointAlgorithm` at world origin. In practice, pass an algorithm whenever batch placement matters.

## Pooling Lifecycle

Each spawn can use a lifecycle policy:

- `Manual`: return explicitly with `SpawnKit.Despawn(...)` or `SpawnHandle.DespawnAll()`
- `OnDisable`: object returns automatically when disabled
- `AfterSeconds`: object returns after a delay, with scaled or unscaled time

You can define the lifecycle:

- on `SpawnableSO` as the default
- on `SpawnPresetSO` as an override
- per request via API overloads that accept `SpawnLifecycle`

## Pool Callbacks

If a spawned prefab needs reset hooks, implement `ISpawnPoolCallbacks` on any component in the prefab hierarchy.

```csharp
using UnityEngine;
using Vit.SpawnKit.Pooling;

public sealed class BulletView : MonoBehaviour, ISpawnPoolCallbacks
{
    public void OnSpawnedFromPool()
    {
        // Reset runtime state here.
    }

    public void OnDespawnedToPool()
    {
        // Stop effects, clear references, etc.
    }
}
```

## Pool Management API

You can inspect and maintain pools through the facade:

```csharp
using UnityEngine;
using Vit.SpawnKit.Api;
using Vit.SpawnKit.Data;
using Vit.SpawnKit.ScriptableObjects;

public sealed class PoolMaintenanceExample : MonoBehaviour
{
    [SerializeField] private SpawnableSO spawnable;

    private void Start()
    {
        SpawnKit.Prewarm(spawnable);

        PoolStats stats = SpawnKit.GetPoolStats(spawnable);
        Debug.Log($"Active={stats.active}, Inactive={stats.inactive}, Total={stats.total}, Max={stats.max}");

        SpawnKit.ReleaseUnused(spawnable);
        SpawnKit.Trim(spawnable, keepInactive: 8);
        SpawnKit.Clear(spawnable);
    }
}
```

Available helpers:

- `SpawnKit.Prewarm(SpawnableSO spawnable)`
- `SpawnKit.Prewarm(SpawnCatalogSO catalog)`
- `SpawnKit.GetPoolStats(...)`
- `SpawnKit.TryGetPoolStats(...)`
- `SpawnKit.Trim(...)`
- `SpawnKit.Clear(...)`
- `SpawnKit.ReleaseUnused(...)`

## Using `SpawnRequest` Directly

For advanced cases you can build a `SpawnRequest` or implement `ISpawnRequestSource`.

This is useful when:

- you want a custom `ISpawnAlgorithm`
- you want to spawn by `SpawnKey`
- you need request-level lifecycle overrides
- you want to provide request data from another gameplay system

## Namespaces

Common namespaces you will use:

```csharp
using Vit.SpawnKit.Api;
using Vit.SpawnKit.Algorithms;
using Vit.SpawnKit.Data;
using Vit.SpawnKit.Pooling;
using Vit.SpawnKit.ScriptableObjects;
```

## Sample In This Repository

This repository includes a minimal demo outside the package:

- `Assets/SpawnModule/Sample/Scene/SpawnTest.unity`
- `Assets/SpawnModule/Sample/SomethingZone_Spawner_Demo.cs`

That demo calls:

```csharp
SpawnKit.Spawn(data, volume);
```

where `data` is a `SpawnPresetSO` and `volume` is a collider array.

## Recommended Usage Pattern

For most projects, the cleanest setup is:

1. Define reusable spawn data in `SpawnableSO`.
2. Put those assets into a `SpawnCatalogSO`.
3. Add one `SpawnManager` to the active scene.
4. Use `SpawnPresetSO` for designer-authored spawn rules.
5. Use `SpawnKit` facade calls from gameplay code.

## Status

The package is still early-stage, but the current runtime flow is already coherent for:

- prefab-based pooled spawning
- batch spawning
- volume spawning
- async batched spawning
- lifecycle-driven despawn

If you plan to publish this package more broadly, the next obvious improvements are:

- bundled samples inside the package
- editor tooling for faster asset setup
- tests for pool behavior and async spawning
- actual Addressables integration when `USE_ADDRESSABLES` is enabled
