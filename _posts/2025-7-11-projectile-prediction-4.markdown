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

Given the complex nature of projectile prediction, we've implemented a variety of tools to help with debugging, in addition to extensive debug logging. These tools are defined in a `UDeveloperSettingsBackedByCVars` class called `UGASDeveloperSettings` (since, technically, this projectile implementation is part of the Gameplay Ability System).

These settings allow for the configuration of a variety of debugging options.

`ProjectileDebugMode` will draw the position of projectiles at regular timestamps throughout their trajectory. Depending on the setting, we can filter which types of projectiles are debugged:

 - `PredictedVersusClient`: Draws the trajectory of the fake projectile and the owning client's version of the authoritative projectile (i.e. where the authoritative projectile would appear for the client that fired the projectile, if it were not hidden).
  - `ClientVersusServer`: Draws the trajectory of the owning client's version of the authoritative projectile and the server's version of the authoritative projectile (i.e. where the authoritative projectile _actually_ is).
  - `All`: Draws The fake projectile, the owning client's version of the authoritative projectile, and the server's version of the authoritative projectile.

When making `PredictedVersusClient` draws, we also draw arrows between the projectiles' corresponding time steps, to show the difference in their positions at each point in their trajectory. When we sync the projectiles over time, we can see this distance become smaller and smaller, until the two are eventually synced, indicated by a change in color:

TODO

And if `bWaitForLinkage` is enabled, we won't start drawing until the fake and authoritative projectiles have been linked. If it's disabled, we'll always see a few unlinked fake projectile draws, since the fake projectile is always spawned earlier:

TODO

`bDrawSpawnPosition` and `bDrawFinalPosition` help us debug projectile spawns and hits, by showing the starting and ending position of each projectile:

TODO

When debugging projectile synchronization, we can log each lerp step performed by enabling `bLogCorrection`:

TODO

Lastly, we can adjust the frequency, duration, and color of each draw with the remaining `Draw` and `Color` settings:

TODO (e.g. drawing server at a high frequency)

## Movement

To move the projectile, we use Unreal's built-in `UProjectileMovementComponent` class. We don't use the built-in movement replication solution, however, since it usually looks pretty terrible, even at high net update frequencies.

Instead, we simulate the projectile's movement on each machine locally. Since we guarantee that both of our projectiles (fake and authoritative) have the exact same spawn location and rotation, they'll always follow the exact same trajectory.

The only time their trajectories may differ is if one hits a surface that the other misses. We'll examine how this, and other missed predictions and desynchronizations, can happen in more detail in the [_Reconciliation_ section](#detonation--reconciliation). What's important with regard to projectile movement, however, is that when the authoritative projectile hits something, we enable movement replication, so each projectile will be teleported to the same location to land or explode. Since projectiles are usually destroyed when they land (e.g. rockets are usually destroyed and replaced by an "explosion" particle effect), this is primarily to ensure that any "land" or "explosion" effects occur in the correct location.

To perform this movement replication, we use a variable called `ReplicatedProjectileMovement` of custom type `FRepProjectileMovement`, which is an optimized version of the `FRepMovement` type used by the built-in `ReplicatedMovement` variable.

We override `AActor`'s replication functions—`PreReplication`, `GatherCurrentMovement`, etc.—to replace `ReplicatedMovement` with our `ReplicatedProjectileMovement` variable, and to use our custom `bReplicateProjectileMovement` variable instead of `bReplicateMovement`.

When projectiles land, we enable `bReplicateProjectileMovement` to replicate their final position, to ensure every projectile lands in the same place:

TODO

Using a custom movement replication variable may seem like a wild micro-optimization (it kinda is), but there are a few other changes we want to make to the built-in movement replication code, which are made easier by this. For example, we ignore the movement replication on non-owning simulated proxies (the clients that didn't fire the projectile), since their projectile will be intentionally behind due to being rewound.

## Hit Detection

For hit detection, two different colliders are used: a `Collision` collider and a `Hitbox` collider. The former is used to detect hits against the environment, and the latter is used to detect hits against enemies, allies (e.g. for healing projectiles), and other damageable actors (like destructibles).

This allows us to define two different hitboxes for each projectile, which is extremely useful. For example, if we have a "spear" projectile, we'd want it to have a very small hitbox against the environment, accurate to its size, so players can throw it through small gaps. But we might want it to have a more generous hitbox against enemies, so it isn't difficult to use:

TODO

We use the `Collision` collider as the projectile movement component's `UpdatedComponent`, since it's usually accurate to what the projectile's actual physics body would be, meaning it also has to be the projectile actor's root component. This does, unfortunately, lead to some restrictions on how projectiles can be configured, but I find this setup to be the most flexible without complicating code or requiring multiple base classes.

### Detonation

When a projectile hits a valid target (how "valid" targets are determined is detailed in [_Effects_](#effects)), we call this a "direct impact" or a "successful hit." This is triggered by the `Hitbox` collider overlapping an actor that passes the "valid target" check.

When a projectile hits an _invalid_ target (i.e. the environment, like a static mesh actor), this is called a "missed impact." Projectiles that can bounce off the environment won't have a "missed impact" until they hit the environment after running out of bounces, though bouncing projectiles will always detonate when hitting a valid target, regardless of their remaining bounces. This event is triggered when the `Collision` collider hits a blocking surface; since this collider is the movement component's `UpdatedComponent`, this will trigger the `OnStop` event, which is what invokes the "missed impact."

Both a direct impact _and_ a missed impact will cause the projectile to "detonate." This is the final event of a projectile's lifetime, at which point the projectile usually applies its gameplay effects, triggers any desired FX, and destroys itself.

This is the single most important event of a projectile's lifetime, and, as such, it's crucial that it's synchronized across all machines. All the methods used to ensure this synchronization and handle missed predictions are detailed in the [_Reconciliation section_](#detonation--reconciliation).
{: .notice--info}

## Effects

### Gameplay Effects

Projectiles can be configured to deal direct impact damage, AOE damage, or both. When a projectile detonates from a direct impact, the actor that was hit is referred to as the "direct target," as opposed to an "AOE target."

A "direct impact" detonation is only triggered when hitting a valid target. To define what a "valid target" actually is, we use our colliders' collision settings (e.g. configuring the `Hitbox` collider to only detect pawns) and a `Filter` property.

`Filter` is a variable of type `FCrashTargetDataFilter`, which is our project-specific `FGameplayTargetDataFilter`. This variable can be configured by projectiles to define whether an actor should be hit by the projectile depending on its team, its gameplay tags (e.g. actors with an `Invulnerable` tag are usually ignored), whether it's the owning actor (e.g. if we want to allow self-damage), whether it has an ability system component, and other parameters:

TODO (showing filter property in projectile archetype)

When the `Hitbox` collider overlaps an actor, the projectile will only detonate if that actor "passes" this filter (though this can be disabled with `bUseFilter`). This filter is also used when determining whether to apply AOE effects to nearby actors.

When a projectile detonates (either because it hit a valid target or because it landed against a surface and can't bounce), the `ImpactGameplayEffect` is applied to the target it hit (if there was one), using an enumerator called `ImpactEffectDirection` to determine the direction of the effect (e.g. for knockback).

Upon detonating, `AreaGameplayEffect` is applied to all valid targets (actors that both have line-of-sight and pass the `Filter`) within the `AreaRadius`. The `AreaOffset` vector can be used to adjust where the center of the radius will be, relative to the `Collision` collider (e.g. if the collider is at the tip of the projectile, but we want the AOE effect to originate at its center).

The `bSkipAreaEffectForImpactTarget` variable can be set to ignore the target of `ImpactGameplayEffect` when applying the AOE effect, if we want an AOE projectile to apply a different effect to any targets it hits directly.

The exact radius of a projectile's area of effect can be visualized by enabling the `bDrawFinalPosition` debug setting:

TODO

### FX

There are four different events that can trigger FX: a detonation, a direct impact, a missed impact, and a bounce (when a bouncing projectile ricochets off a surface).

Both the "detonation" event and the "impact" events are triggered when a projectile detonates, so most projectiles only use one of these events. For example, a rocket may only use the "detonation" event to trigger explosion FX, while a spear may only use the "impact" events to trigger appropriate "hit" or "miss" FX.

Each of these events can trigger a collection of particle effects, sound cues, force feedback, and decals, configurable in the archetype of each projectile class.

To compartmentalize these effects, we define a custom struct called `FProjectileFX`. This represents a collection of FX that can be triggered as a group. Projectiles have an instance of this struct defined for each of the aforementioned events, which will be collectively triggered when the event occurs:

TODO: (showing FX properties in projectile archetype)

This struct is _not_ used, however, for FX triggered by the "direct impact" event. This event triggers the `ImpactGameplayEffect`, so we instead add a gameplay cue to this gameplay effect and put any FX we want inside that cue, to help improve compartmentalization.

### Predicting FX

Because projectiles simulate their movement locally on each machine, the detonation, impact, and bounce events are triggered locally, and thus "predicted," since we don't wait for them to occur on the authoritative projectile.

This feels and looks great for small, fast, or inconsequential (e.g. no AOE or lingering effects) projectiles, like small bullets. Missed predictions and corrections are extremely rare and barely noticeable. But for larger or slower projectiles with big AOE effects, like a rocket, the risk of missed predictions increases, and correcting those predictions looks a lot more jarring, like having to resimulate an entire explosion in the correct location:

TODO

To fix this, projectiles have an option called `bPredictFX`. If `bPredictFX` is enabled, detonation and impact FX will be predicted (with plenty of reconciliation to correct any missed predictions that occur). If `bPredictFX` is disabled, these events will only be triggered by the authoritative projectile, and will be replicated to other machines when they occur.

Bounce FX are always predicted, since predicting and correcting physics simulations is more complex, and these FX aren't as important as the detonation or impacts.
{: .notice--info}

It's up to the discretion of designers whether a projectile should predict its FX. Usually, we want smaller, quick, direct-hit projectiles, like bullets, to predict FX, while big, slow, AOE projectiles, like rockets, shouldn't:

TODO

Regardless of `bPredictFX`, we never predict gameplay effects. Since we use a gameplay cue tied to the `ImpactGameplayEffect` for "direct impact" FX, this means these FX are never predicted. This is intentional, since we want our damage (which we also don't predict), hitmarker, FX, and reaction animations to be synced together, and because hit-impact missed predictions are really irritating for players. From what I've seen, this is a fairly conventional approach in games: predicting FX except for ones that indicate damage (blood splatters, star particles, etc.).
{: .notice--info}

## Detonation & Reconciliation

Since we're simulating our projectile's movement locally (as opposed to replicating the projectile movement), having two projectiles that aren't synchronized could cause them to hit different targets. For example, if the fake projectile is ahead the real one (since it's fired first), an enemy could move into the path of the projectile after the first one has passed it, but still get hit by the second one):

TODO

### Resimulation for Remote Proxies