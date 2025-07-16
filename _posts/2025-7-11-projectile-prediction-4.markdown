---
layout: single
title: "Projectile Prediction: Part 4"
excerpt: A breakdown of the base projectile's features and reconciliation techniques.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-07-15
---

The fourth and final part of a series exploring and implementing projectile prediction for multiplayer games. This part breaks down the implementation of a base `Projectile` actor class, which can be subclassed into projectiles that can be spawned by our `SpawnPredictedProjectile` task.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...).
{: .notice--info}

## Introduction

In the last section, we walked through the "linking" step of initializing projectiles. As mentioned before, in this part, we're going to go through the features of the `Projectile` class, breaking down how each one works, without going too far into the code.

## Fast-Forwarding

In [part 1](https://sreitich.github.io/projectile-prediction-1/#partia-fast-forwarding-with-synchronization-and-resimulation), we decided that we would (partially) fast-forward the authoritative projectile such that it spawns closer to where the owning client wants it.

In the `SpawnPredictedProjectile` class, we fast-forward the projectile in `OnSpawnDataReplicated`, after it's spawned, by the `ForwardPredictionTime` given to us by our player controller. We call `TickActor` to tick the actor's components (for things like animations or VFX), and `TickComponent` on the projectile's `ProjectileMovement` component (see [_Movement_](#movement)) to actually move it forward.

We also call `SetLifeSpan` to reduce the projectile's lifespan by the amount it was forwarded in time. All projectiles should have a default `InitialLifeSpan` set, just to ensure we don't end up with lingering actors wasting resources if they don't hit anything.

We didn't implement this when we created the `SpawnPredictedProjectile` class because we didn't have a `ProjectileMovement` component to tick yet.
{: .notice--info}

Forwarding the projectile like this can sometimes cause issues with hit detection. Since we're essentially teleporting the projectile forward in time, if the projectile should have hit something close to its spawn location, there's a chance it will be teleported right through it, missing it entirely:

TODO

To fix this, we need to enable `bForceSubStepping` on our `ProjectileMovement` component, decrease `MaxSimulationTimeStep`, and increase `MaxSimulationIterations`. This forces the projectile to break its movement into discrete steps, so hit detection can still be performed while forwarding it.

With this, we can see the authoritative projectile being fast-forwarded on spawn, without the risk of missing collisions:

TODO

## Synchronization

Since we're only _partially_ fast-forwarding our projectile (again, see [part 1](https://sreitich.github.io/projectile-prediction-1/#partia-fast-forwarding-with-synchronization-and-resimulation)), the predicted/"fake" projectile will still not be synced with the authoritative projectile.

We'll talk more about why this desynchronization can cause issues in the [_Reconciliation_ section](#detonation--reconciliation).
{: .notice--info}

To synchronize our projectiles, we slowly lerp the fake projectile towards the authoritative one. We do this by calling `CorrectionLerpTick` on the client's version of the authoritative actor each tick, which sets the linked fake projectile's location one step towards the authoritative one. To actually move the actor, we hack into the `ReplicatedMovement` property, which can safely handle the movement update.

The rate at which we lerp the projectile is `0.05%` of the projectile's initial speed each tick. This is an arbitrary value that I've found to be big enough to synchronize the projectiles quickly, but small enough to be completely unnoticeable to clients.

The replicated authoritative projectile is hidden for the client with the fake projectile (since we don't want to see two different projectiles). But if we unhide it, we can see how the projectiles synhronize over time:

TODO

### Rewinding

On remote clients (the clients that didn't fire the projectile, so they don't have a fake one), the projectile will first appear a noticeable distance ahead of where it spawned, due to the time it takes to replicate from the server, and because of our fast-forwarding:

TODO

To make this less noticeable, when the projectile is first replicated to remote clients, we rewind it back to its spawn transform. This is done at the end of `BeginPlay`, using a replicated property called `SpawnTransform.` By doing this, we ensure that remote clients see the entire trajectory and lifetime of the projectile:

TODO

To learn why this, unintuitively, does _not_ cause synchronization issues, see [part 1](https://sreitich.github.io/projectile-prediction-1/#partia-fast-forwarding-with-synchronization-and-resimulation). The reconciliation techniques we'll cover below also help with mitigate potential issues caused by this desynchronization.
{: .notice--info}

## Debugging

Given the complex nature of projectile prediction, we've implemented a variety of tools to help with debugging, in addition to extensive debug logging. These tools are defined in a `UDeveloperSettingsBackedByCVars` class called `GASDeveloperSettings` (since, technically, this projectile implementation is part of the Gameplay Ability System).

These settings allow for the configuration of a variety of debugging options.

`ProjectileDebugMode` will draw the position of projectiles at regular timestamps throughout their trajectory. Depending on the setting, we can filter which types of projectiles are debugged:

 - `PredictedVersusClient`: Draws the trajectory of the fake projectile and the owning client's version of the authoritative projectile (i.e. where the authoritative projectile would appear for the client that fired the projectile, if it were not hidden).
  - `ClientVersusServer`: Draws the trajectory of the owning client's version of the authoritative projectile and the server's version of the authoritative projectile (i.e. where the authoritative projectile _actually_ is).
  - `All`: Draws The fake projectile, the owning client's version of the authoritative projectile, and the server's version of the authoritative projectile.

When making `PredictedVersusClient` draws, we also draw arrows between the projectiles' corresponding time steps, to show the difference in their positions at each point in their trajectory. When we sync the projectiles over time, we can see this distance become smaller and smaller, until the two are eventually synced, indicated by a change in color:

TODO

And if `bWaitForLinkage` is enabled, we won't start drawing until the fake and authoritative projectiles have been linked. If it's disabled, we'll always see a few unlinked fake projectile draws, since the fake projectile is always spawned earlier:

TODO

`bDrawSpawnPosition` and `bDrawFinalPosition` help us debug projectile spawns and hits, by showing the starting and ending position of each projectile, including the AOE radius of projectiles with area effects:

TODO

When debugging projectile synchronization, we can log each lerp step performed by enabling `bLogCorrection`:

TODO

Lastly, we can adjust the frequency, duration, and color of each draw with the remaining `Draw` and `Color` settings:

TODO (e.g. drawing server at a high frequency)

## Movement

## Hit Detection

## Effects

## Detonation & Reconciliation

Since we're simulating our projectile's movement locally (as opposed to replicating the projectile movement), having two projectiles that aren't synchronized could cause them to hit different targets. For example, if one projectile is behind the other, an enemy could move into the path of the projectile after the first one has passed it, but still get hit by the second one).

### Resimulation for Remote Proxies