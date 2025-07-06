---
layout: single
title: "Projectile Prediction: Part 2"
excerpt: Predictively spawning projectiles with the Gameplay Ability System.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-07-03
---

Part 2 of a series exploring and implementing projectile prediction for multiplayer games. This part walks through how to predictively spawn projectile actors using the Gameplay Ability System in Unreal Engine.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...).
{: .notice--info}

## Introduction

In the previous section of this series, we looked at the model we'll be using for our projectile prediction system, and a short overview of the system itself.

The model we chose relies on two projectiles: a "fake" projectile, on the firing client, and a "real" (a.k.a. "authoritative") projectile on the server. Once spawned, these two projectiles must be "linked" together so the authoritative projectile can "fast-forward" towards the fake projectile, and so the fake projectile can lerp towards the authoritative one and eventually synchronize with it.

To handle spawning and linking the two projectiles, we'll be using the Gameplay Ability System and creating a new **ability task**. Ability tasks are asynchronous functions that can be called inside gameplay abilities, which is where we'll usually want to spawn projectiles from (e.g. a "fire rocket" ability). 

Using an ability task will let us hook into GAS's prediction system, which we can use to destroy the projectiles if the ability is rejected. It will also make it easier to handle the different network execution paths we need: our ability task needs to run both locally (on the activating client) and on the server; on the client, it will spawn the fake projectile, and on the server, it will spawn and link the authoritative projectile. With an ability task, we can perform both of these operations with just one blueprint node in our ability script, because the ability will create two separate instances of the task for us. Lastly, GAS's "target data" replication system will also let us easily communicate between the client and server without needing to thread any RPCs through a controller or pawn, which will make things easier.

## Setting Up the Task

Let's start by creating our base projectile class. We won't be doing anything with it yet, but we want to be able to reference it in our task. This will be a subclass of `Actor` named `Projectile`.

TODO: Creating a projectile class.

Next, let's create the ability task. Each ability task node needs its own class, so we'll create a subclass of `AbilityTask` named `AbilityTask_SpawnPredictedProjectile` The `AbilityTask_` prefix is the standard naming convention for ability task classes.

TODO: Creating an ability task class.

Let's start by declaring a constructor and overriding the `Activate` function.

To call ability tasks, we don't use a traditional constructor. Instead, we define a blueprint-callable function, which we'll call `SpawnPredictedProjectile`, to create and initialize a new instance of the task, and this is the node we actually call inside ability blueprints. `Activate` is the function that's called on the instantiated task itself once it's been created.

To properly initialize our task with `SpawnPredictedProjectile`, we need to give it our projectile's parameters: its spawn location, spawn rotation, and the projectile subclass to spawn.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

UCLASS()
class GAME_API UAbilityTask_SpawnPredictedProjectile : public UAbilityTask
{
    GENERATED_BODY()

    /** Spawn a predicted projectile actor. */
    UFUNCTION(BlueprintCallable, Meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "True"), Category = "Ability|Tasks")
    static UAbilityTask_SpawnPredictedProjectile* SpawnPredictedProjectile(UGameplayAbility* OwningAbility, TSubclassOf<AProjectile> ProjectileClass, FVector SpawnLocation, FRotator SpawnRotation);
    
    /** Spawn a fake projectile on the client and an authoritative projectile on the server. */
    virtual void Activate() override;
}
{% endhighlight %}

Once we instantiate the task, we need to cache these parameters somewhere in the task instance so we can use them.

{% highlight c++ %}
    // ...
    
    // Parameters.
    
protected:
    
    /** The projectile class to spawn. */
    UPROPERTY()
    TSubclassOf<AProjectile> ProjectileClass;
    
    /** Location at which to spawn the projectile. */
    FVector SpawnLocation;
    
    /** Rotation with which to spawn the projectile. */
    FRotator SpawnRotation;
{% endhighlight %}

Lastly, we want our task node to have some output pins to continue execution depending on whether our task succeeds or fails.

To define output pins for a task, we need to define a delegates for each pin that we want. To trigger the output pin, we simply broadcast that delegate. We can also add output parameters by adding parameters to the delegate.

It might be useful to have a reference to the new projectile once it's spawned, so let's define a new delegate signature that provides an `AProjectile` pointer. (Put this right above the class, below the include list.)

{% highlight c++ %}
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FSpawnPredictedProjectileDelegate, AProjectile*, SpawnedProjectile);
{% endhighlight %}

Now, we can define our output pins using this signature.

{% highlight c++ %}
    /**
     * Called locally when the projectile is spawned.
     *
     * On clients, this will return the fake projectile actor. On the server, this will return the authoritative
     * projectile actor.
     */
    UPROPERTY(BlueprintAssignable)
    FSpawnPredictedProjectileDelegate Success;

    /** Called on the client and server if either failed to spawn their projectile actor, usually because the ability's
     * prediction key was rejected. The ability should likely be cancelled (on both machines) at this point. */
    UPROPERTY(BlueprintAssignable)
    FSpawnPredictedProjectileDelegate FailedToSpawn;
{% endhighlight %}

Due to latency, `Success` will be triggered at unpredictable times. That means we usually shouldn't use it for gameplay logic; we really only want to use this for local cosmetic effects and validation. For logic that should have deterministic timing, like spawning multiple subsequent projectiles with a fixed delay between each, we should use the normal execution output pin and `Wait Delay` tasks instead.
{: .notice--info}

Now that we have everything our task needs, we can start implementing the logic.

For our "constructor," `SpawnPredictedProjectile`, all we need to do is create a new task object and initialize its paramters:

{% highlight c++ %}
UAbilityTask_SpawnPredictedProjectile* UAbilityTask_SpawnPredictedProjectile::SpawnPredictedProjectile(UGameplayAbility* OwningAbility, TSubclassOf<AProjectile> ProjectileClass, FVector SpawnLocation, FRotator SpawnRotation)
{
    if (!ensureAlwaysMsgf(OwningAbility->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted, TEXT("SpawnPredictedProjectile ability task activated in ability (%s), which does not have a net execution policy of Local Predicted. SpawnPredictedProjectile should only be used in predicted abilities. Use AbilityTask_SpawnActor otherwise."), *GetNameSafe(OwningAbility)))
    {
        return nullptr;
    }

	if (!ensureAlwaysMsgf(IsValid(ProjectileClass), TEXT("SpawnPredictedProjectile ability task activated in ability (%s) without a valid projectile class set."), *GetNameSafe(OwningAbility)))
	{
		return nullptr;
	}

	UAbilityTask_SpawnPredictedProjectile* Task = NewAbilityTask<UAbilityTask_SpawnPredictedProjectile>(OwningAbility);
	Task->ProjectileClass = ProjectileClass;
	Task->SpawnLocation = SpawnLocation;
	Task->SpawnRotation = SpawnRotation;
	return Task;
}
{% endhighlight %}

We've also added some data validation here. Since this task is performing client-side prediction, we should only use it in gameplay abilities that are `Local Predicted` (as opposed to abilities that only execute on the client or the server). Otherwise, there's no point in predicting anything. We're also checking to make sure we provided a valid projectile class in our parameters, so we actually have something to spawn.

## Configuring Prediction

Before we start spawning projectiles, we need to configure some additional parameters for prediction.

We're going to use our player controller as the owner of our projectiles, since it's replicated, easily accessible, and won't be destroyed if our player dies (as opposed to our player character), so I'm choosing to put our prediction configuration properties here too.

Let's quickly break down the purpose of each of these values before we add them.

### Maximum Prediction Ping

The first value we need is our `MaxPredictionPing`. I mentioned in the previous section that we should put a limit on how far our projectile is forward-predicted, so it doesn't jump too far at high latencies. `MaxPredictionPing` will be the maximum ping with which we'll forward-predict our projectiles. If the client has a higher ping than this, then we'll _delay_ spawning our projectile, so it doesn't get fast-forwarded further than this value.

For example, if we set `MaxPredictionPing` to `100ms`, but our client has `150ms` of ping, we'll wait `50ms` before spawning either projectile, so the authoritative projectile only has to be fast-forwarded `100ms` to catch up to the fake one (assuming we're not doing partial fast-forwarding, detailed in the previous section). This is important for two reasons.

First, this keeps things fair for other players. We learned how partial fast-forwarding helps keep things fair for other players by preventing projectiles from teleporting a considerable distance ahead of where they should be. But since this distance is determinant on clients' ping, we could still end up forwarding projectiles extreme distances if clients have high enough ping. Setting this limit on the amount of ping with which we'll actually _allow_ clients to forward-predict mitigates this issue.

Second, this helps communicate latency to our player. If we're playing with `150ms` of ping and everything looks perfect, but we aren't hitting any shots, we'd be a little confused. _Forcing_ the player to see some noticeable lag helps tell them, _"You're playing with a lot of latency; you should expect some issues."_

### Client Bias

On the topic of fast-forwarding, remember that we're only doing _partial_ forward-prediction, so we're only forwarding projectiles a portion of the distance to where the client "wants" them. In the previous section, I said we'd forward-predict them "about halfway."

To make this more configurable, we'll actually make this a variable called `ClientBiasPct`, which will represent the amount, as a percentage, that we'll favor where the client wants the projectile, and where the server wants the projectile. Setting `ClientBiasPct` to `1.0` will have the effect of "full" fast-forwarding: placing the authoritative projectile exactly where the fake one is; setting it to `0.0` will disable fast-forwarding entirely, leaving the projectile where the server spawns it.

### Fudge Factor

Due to factors like processing time and tick rate, the server's "perceived" ping of a connected client is usually around `10-20ms` higher than it really is. To account for this, we'll also need a small value, which I'll call `PredictionLatencyReduction`, to help "fudge" the server's estimation of clients' ping to a more accurate value.

### Prediction Values

Lastly, we'll need to define two functions: `GetForwardPredictionTime` and `GetProjectileSleepTime`.

`GetForwardPredictionTime` will determine how far we'll actually forward-predict our authoritative projectile, taking into account our client bias, and using `PredictionLatencyReduction` to fudge the server's value.

`GetProjectileSleepTime` will return the amount of time, if any, that we should _delay_ spawning our projectiles, in case our ping exceeds `MaxPredictionPing`.

{% highlight c++ %}
// MyPlayerController.h

public:

    /** Aka "fudge factor." Used to "fudge" the server's estimate of a client's ping to get a more accurate guess of
     * their exact ping. */
    UPROPERTY(BlueprintReadOnly, Config, Category = Network)
    float PredictionLatencyReduction;

    /** How much (from 0-1) to favor the client when determining the real position of predicted projectiles. A greater
     * value will spawn authoritative projectiles closer to where the client wants; a smaller value will spawn
     * authoritative projectiles closer to where the server wants (forwarding them less). */
    UPROPERTY(BlueprintReadOnly, Config, Category = Network)
    float ClientBiasPct;

    /** Max amount of ping to predict ahead for. If the client's ping exceeds this, we'll delay spawning the projectile
     * so it doesn't spawn further ahead than this. */
    UPROPERTY(BlueprintReadOnly, Config, Category = Network)
    float MaxPredictionPing;

    /** The amount of time, in seconds, to tick or simulate to make up for network lag. (1/2 player's ping) - prediction
     * latency reduction. */
    float GetForwardPredictionTime() const;

    /** How long to wait before spawning the projectile if the client's ping exceeds MaxPredictionPing, so we don't
     * forward-predict too far. */
    float GetProjectileSleepTime() const;
{% endhighlight %}

To be able to tweak these values in `DefaultGame.ini` (which is the purpose of the `Config` specifier), remember to put `Config = Game` in your class's `UCLASS` macro (i.e. `UCLASS(Config = Game)`).
{: .notice--info}

{% highlight c++ %}
// MyPlayerController.cpp

AMyController::AMyController()
{
    // Constructor code...

    PredictionLatencyReduction = 20.0f;
    ClientBiasPct = 0.5f;
    MaxPredictionPing = 120.0f;
}

float AMyController::GetForwardPredictionTime() const
{
    // Divide by 1000 to convert ping from ms to s.
    return (PlayerState && (GetNetMode() != NM_Standalone)) ? (0.001f * ClientBiasPct * FMath::Clamp(PlayerState->ExactPing - (IsLocalController() ? 0.0f : PredictionLatencyReduction), 0.0f, MaxPredictionPing)) : 0.f;
}

float AMyController::GetProjectileSleepTime() const
{
    // At high latencies, projectiles won't be spawned until they can be forward-predicted at the maximum prediction ping.
    return 0.001f * FMath::Max(0.0f, PlayerState->ExactPing - PredictionLatencyReduction - MaxPredictionPing);
}
{% endhighlight %}

## Tracking Projectiles

One other thing we should set up in our player controller is a way to track projectiles; we need a way to identify our projectiles so they can be linked together once they're spawned.

An easy way to do this is by assigning each projectile a replicated "ID." Each time we fire a projectile, we can assign the same ID to both the fake and authoritative one. We can cache the new fake projectile in our controller (since it will always be spawned first), and once the authoritative projectile is spawned, it can search the list of spawned fake projectiles for the one with a matching ID, and link to it.

Let's start by reserving a value to represent projectiles _without_ an ID (i.e. any projectiles on other clients, and projectiles fired by listen servers, since they won't have a fake projectile, and thus have no need to link to one).

{% highlight c++ %}
// MyPlayerController.h, above the class definition

#define NULL_PROJECTILE_ID 0
{% endhighlight %}

I might switch to using a `TOptional` to represent IDs in the future so my professors won't get upset. If I do, this post will be updated.
{: .notice--info}

Now we can add variables for storing our projectiles and tracking our IDs, and a function to generate new IDs.

{% highlight c++ %}
// MyPlayerController.h

public:

    /** List of this client's fake projectiles (client-side predicted projectiles) that haven't been linked to an
     * authoritative projectile yet. */
    UPROPERTY()
    TMap<uint32, TObjectPtr<AProjectile>> FakeProjectiles;

    /** Generates a unique ID for a new fake projectile. */
    uint32 GenerateNewFakeProjectileId();

private:

    /** Internal counter for projectile IDs. Starts at 1 because 0 is reversed for non-predicted projectiles. */
    uint32 FakeProjectileIdCounter;
{% endhighlight %}

{% highlight c++ %}
// MyPlayerController.cpp

AMyController::AMyController()
{
    // Constructor code...

    PredictionLatencyReduction = 20.0f;
    ClientBiasPct = 0.5f;
    MaxPredictionPing = 120.0f;
    FakeProjectileIdCounter = 1;
}

uint32 ACrashPlayerController::GenerateNewFakeProjectileId()
{
    const uint32 NextId = FakeProjectileIdCounter;
    FakeProjectileIdCounter = FakeProjectileIdCounter < UINT32_MAX ? FakeProjectileIdCounter + 1 : 1;
    checkf(!FakeProjectiles.Contains(NextId), TEXT("Generated invalid projectile ID! ID (%i) already used. Fake projectile map size: (%i)."), NextId, FakeProjectiles.Num());
    return NextId;
}
{% endhighlight %}

We'll see later on that, although this isn't the _most_ efficient method for linking projectiles, giving each set of projectiles a unique identifier is extremely useful when debugging.
{: .notice--info}

## Spawning the Fake Projectile

Spawn fake projectile

### Deferring the Spawn

Delayed spawn

## Spawning the Authoritative Projectile

### Sending Spawn Data to the Server

Creating target data
Send data to the server

### Receiving Spawn Data on the Server

Listen for data
Spawn real projectile

### Handling Listen Servers/Standalone

Spawn non-predicted authoritative projectile