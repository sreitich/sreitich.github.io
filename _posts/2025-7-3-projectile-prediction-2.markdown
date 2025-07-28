---
layout: single
title: "Projectile Prediction: Part 2"
excerpt: Predictively spawning projectiles with the Gameplay Ability System.
header:
    teaser: /assets/images/per-post/projectile-prediction-2/projectile-prediction-2-teaser.png
author: Meta
last_modified_at: 2025-07-26
---

Part 2 of a series exploring and implementing projectile prediction for multiplayer games. This part walks through how to predictively spawn projectile actors using the Gameplay Ability System in Unreal Engine.

The code used for this series can be found on [Unreal Engine's Learning site](https://dev.epicgames.com/community/learning/tutorials/LZ66). This section is a step-by-step walkthrough for programming the `UAbilityTask_SpawnPredictedProjectile` class. If you don't want a detailed walkthrough, you can copy the code directly (it's the same thing you'll get from following along). The code itself is extremely well-documented, and this page serves as an additional resource for documentation. Alternatively, a more concise documentation page [can be found here](https://docs.google.com/document/d/1VBhB41mwQWksoPLgx-G8YSelnWQ7_FYu4-QGueqY2lY/edit?usp=sharing).
{: .notice--info}

This code was written for a game called [_Cloud Crashers_](https://store.steampowered.com/app/2995940/Cloud_Crashers/), and uses project-specific classes named as such. For your game, you'll need to replace the `ACrashPlayerController` and `UCrashAbilitySystemComponent` classes with your game's respective player controller and ASC classes.
{: .notice--info}

## Introduction

In the previous section of this series, we looked at the model we'll be using for our projectile prediction system, and a short overview of the system itself.

The model we chose relies on two projectiles: a "fake" projectile, on the firing client, and a "real" (a.k.a. "authoritative") projectile on the server. Once spawned, these two projectiles must be "linked" together so the authoritative projectile can "fast-forward" towards the fake projectile, and so the fake projectile can lerp towards the authoritative one and eventually synchronize with it.

To handle spawning and linking the two projectiles, we'll be using the [Gameplay Ability System](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-system-for-unreal-engine) and creating a new [**ability task**](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-ability-tasks-in-unreal-engine). Ability tasks are asynchronous functions that can be called inside gameplay abilities, which is where we'll usually want to spawn projectiles from (e.g. a "fire rocket" ability). 

Using an ability task will let us hook into GAS's prediction system, which we can use to destroy the projectiles if the ability is rejected. It will also make it easier to handle the different network execution paths we need: our ability task needs to run both locally (on the activating client) and on the server; on the client, it will spawn the fake projectile, and on the server, it will spawn and link the authoritative projectile. With an ability task, we can perform both of these operations with just one blueprint node in our ability script, because the ability will create two separate instances of the task for us. Lastly, GAS's "target data" replication system will also let us easily communicate between the client and server without needing to thread any RPCs through a controller or pawn, which will make things easier.

## Setting Up the Task

Let's start by creating our base projectile class. We won't be doing anything with it yet, but we want to be able to reference it in our task. This will be a subclass of `Actor` named `Projectile`.

![Projectile class wizard]({{ '/' | absolute_url }}/assets/images/per-post/projectile-prediction-2/projectile-class-wizard.png){: .align-center}

Next, let's create the ability task. Each ability task node needs its own class, so we'll create a subclass of `AbilityTask` named `AbilityTask_SpawnPredictedProjectile` The `AbilityTask_` prefix is the standard naming convention for ability task classes.

![Ability task class wizard]({{ '/' | absolute_url }}/assets/images/per-post/projectile-prediction-2/ability-task-class-wizard.png){: .align-center}

Unreal's class wizard won't let you make a class with a name this long. To work around this, name the class something like `AbilityTask_Spawn`, then rename the class _and_ the file to `AbilityTask_SpawnPredictedProjectile` after it's added.
<br>
<br>
Unreal itself has several class names that break this character limit, so don't worry about this causing any issues.
{: .notice--info}

Let's start by declaring a constructor and overriding the `Activate` function.

To call ability tasks, we don't use a traditional constructor. Instead, we define a blueprint-callable function, which we'll call `SpawnPredictedProjectile`, to instantiate and initialize a new instance of the task, and this is the node we actually call inside ability blueprints. `Activate` is the function that's called on the instantiated task itself once it's been created.

To properly initialize our task, we need to define any parameters we want our task to have as parameters of the function. In our case, that will be the projectile class we want to spawn, as well as its spawn location and spawn rotation.

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

For our "constructor," `SpawnPredictedProjectile`, all we need to do is instantiate a new task object and initialize its parameters:

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

We're going to use our player controller as the owner of our projectiles, since it's replicated, easily accessible, and won't be destroyed if our player dies (as opposed to our player character), so it also serves as a good place to store our prediction configuration properties.

Let's quickly break down the purpose of each of these values before we add them.

### Maximum Prediction Ping

The first value we need is our `MaxPredictionPing`. I mentioned in the [previous section](https://sreitich.github.io/projectile-prediction-1/#partial-fast-forwarding) that we should put a limit on how far our projectile is forward-predicted, so it doesn't jump too far at high latencies. `MaxPredictionPing` will be the maximum ping with which we'll forward-predict our projectiles. If the client has a higher ping than this, then we'll _delay_ spawning our projectile, so it doesn't get fast-forwarded further than this value.

For example, if we set `MaxPredictionPing` to `100ms`, but our client has `150ms` of ping, we'll wait `50ms` before spawning either projectile, so the authoritative projectile only has to be fast-forwarded `100ms` to catch up to the fake one (though since we're actually doing _partial_ fast-forwarding, detailed in the previous section, we wouldn't really fast-forward that far). This is important for two reasons.

First, this keeps things fair for other players. We learned how partial fast-forwarding helps keep things fair for other players by preventing projectiles from teleporting a considerable distance ahead of where they should be. But since this distance is determinant on clients' ping, we could still end up forwarding projectiles extreme distances if clients have high enough ping. Setting this limit on the amount of ping with which we'll actually _allow_ clients to forward-predict mitigates this issue.

Second, this helps communicate latency to our player. If we're playing with `150ms` of ping and everything looks perfect, but we aren't hitting any shots, we'd be a little confused. _Forcing_ the player to see some noticeable lag helps tell them, _"You're playing with a lot of latency; you should expect some issues."_

### Client Bias

On the topic of fast-forwarding, remember that we're only doing _partial_ forward-prediction, so we're only forwarding projectiles a portion of the distance to where the client "wants" them. In the previous section, I said we'd forward-predict them "about halfway."

To make this more configurable, we'll actually make this a variable called `ClientBiasPct`, which will represent the amount, as a percentage, that we'll favor where the client wants the projectile over where the server wants the projectile. Setting `ClientBiasPct` to `1.0` will have the effect of "full" fast-forwarding: placing the authoritative projectile exactly where the fake one is; setting it to `0.0` will disable fast-forwarding entirely, leaving the projectile where the server spawns it.

### Fudge Factor

Due to factors like processing time and tick rate, the server's "perceived" ping of a connected client is usually around `10-20ms` higher than it really is. To account for this, we'll also need a small value, which I'll call `PredictionLatencyReduction`, to help "fudge" the server's estimation of clients' ping to a more accurate value.

### Prediction Values

Lastly, we'll need to define two functions: `GetForwardPredictionTime` and `GetProjectileSleepTime`.

`GetForwardPredictionTime` will determine how far we'll actually forward-predict our authoritative projectile, taking into account our client bias, and using `PredictionLatencyReduction` to fudge the server's value.

`GetProjectileSleepTime` will return the amount of time, if any, that we should _delay_ spawning our projectiles, in case our ping exceeds `MaxPredictionPing`.

{% highlight c++ %}
// CrashPlayerController.h

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
// CrashPlayerController.cpp

ACrashPlayerController::ACrashPlayerController()
{
    // Constructor code...

    PredictionLatencyReduction = 20.0f;
    ClientBiasPct = 0.5f;
    MaxPredictionPing = 120.0f;
}

float ACrashPlayerController::GetForwardPredictionTime() const
{
    // Divide by 1000 to convert ping from ms to s.
    return (PlayerState && (GetNetMode() != NM_Standalone)) ? (0.001f * ClientBiasPct * FMath::Clamp(PlayerState->ExactPing - (IsLocalController() ? 0.0f : PredictionLatencyReduction), 0.0f, MaxPredictionPing)) : 0.f;
}

float ACrashPlayerController::GetProjectileSleepTime() const
{
    // At high latencies, projectiles won't be spawned until they can be forward-predicted at the maximum prediction ping.
    return 0.001f * FMath::Max(0.0f, PlayerState->ExactPing - PredictionLatencyReduction - MaxPredictionPing);
}
{% endhighlight %}
<br>
## Tracking Projectiles

One other thing we should set up in our player controller is a way to track projectiles; we need a way to identify our projectiles so they can be linked together once they're spawned.

An easy way to do this is by assigning each projectile a replicated "ID." Each time we fire a projectile, we can assign the same ID to both the fake and authoritative one. We can cache the new fake projectile in our controller (since it will always be spawned first), and once the authoritative projectile is spawned, it can search the list of spawned fake projectiles for the one with a matching ID, and link to it.

Let's start by reserving a value to represent projectiles _without_ an ID (i.e. any projectiles on other clients, and projectiles fired by listen servers, since they won't have a fake projectile, and thus have no need to link to one).

{% highlight c++ %}
// CrashPlayerController.h, above the class definition

#define NULL_PROJECTILE_ID 0
{% endhighlight %}

I might switch to using a `TOptional` to represent IDs in the future so my professors won't get upset. If I do, this post will be updated.
{: .notice--info}

Now we can add variables for storing our projectiles and tracking our IDs, and a function to generate new IDs.

{% highlight c++ %}
// CrashPlayerController.h

public:

    /** List of this client's fake projectiles (client-side predicted projectiles) that haven't been linked to an
     * authoritative projectile yet. */
    UPROPERTY()
    TMap<uint32, TObjectPtr<AProjectile>> FakeProjectiles;

    /** Generates a unique ID for a new fake projectile. */
    uint32 GenerateNewFakeProjectileId();

private:

    /** Internal counter for projectile IDs. Starts at 1 because 0 is reserved for non-predicted projectiles. */
    uint32 FakeProjectileIdCounter;
{% endhighlight %}

{% highlight c++ %}
// CrashPlayerController.cpp

ACrashPlayerController::ACrashPlayerController()
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

This may not be the _most_ efficient method for linking projectiles, but giving each pair of projectiles a unique identifier happens to be extremely useful when debugging.
{: .notice--info}

Now that we have the configuration parameters and utilities we need, we can finally start spawning projectiles.

## Spawning the Fake Projectile

In our `Activate` function, let's start by retrieving our player controller from the gameplay ability.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::Activate()
{
    Super::Activate();
    
    if (Ability && Ability->GetCurrentActorInfo())
    {
        if (ACrashPlayerController* CrashPC = Ability->GetCurrentActorInfo()->PlayerController.IsValid() ? Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController.Get()) : nullptr)
        {
        }
    }
}
{% endhighlight %}

Now, we need to gather a few pieces of data that will help us figure out the context of our current task. For example, whether this is the locally predicted execution of the task, or the server's authoritative execution of the task.

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::Activate()
{
    Super::Activate();
    
    if (Ability && Ability->GetCurrentActorInfo())
    {
        if (ACrashPlayerController* CrashPC = Ability->GetCurrentActorInfo()->PlayerController.IsValid() ? Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController.Get()) : nullptr)
        {
            const float ForwardPredictionTime = CrashPC->GetForwardPredictionTime();
            const bool bShouldPredict = (ForwardPredictionTime > 0.0f);
            const bool bIsNetAuthority = Ability->GetCurrentActorInfo()->IsNetAuthority();
            const bool bShouldUseServerInfo = IsLocallyControlled();
        }
    }
}
{% endhighlight %}

We only need to spawn a fake projectile if we're a local client (i.e. `bIsNetAuthority` is false) and our `ForwardPredictionTime` is greater than `0.0` (i.e. our ping is greater than `0.0`, to account for LAN servers).

To properly spawn our fake projectile, we need a struct of type `FActorSpawnParameters`. These parameters will be re-used a few times, so we can make some helper functions to construct them when needed, with the necessary parameters. (We'll skip the pre-spawn initialization code for now, and come back to it in the next section, when we implement our ID code in `Projectile`).

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

protected:

    /** Helper to make spawn parameters for a projectile. */
    FActorSpawnParameters GenerateSpawnParams() const;
    
    /** Helper to make spawn parameters for a fake projectile. */
    FActorSpawnParameters GenerateSpawnParamsForFake(const uint32 ProjectileId) const;
    
    /** Helper to make spawn parameters for an authoritative projectile. */
    FActorSpawnParameters GenerateSpawnParamsForAuth(const uint32 ProjectileId) const;
{% endhighlight %}

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParams() const
{
    FActorSpawnParameters Params;
    Params.Instigator = Ability->GetCurrentActorInfo()->AvatarActor.IsValid() ? Cast<APawn>(Ability->GetCurrentActorInfo()->AvatarActor) : nullptr;
    Params.Owner = AbilitySystemComponent->GetOwnerActor();
    Params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
    return Params;
}

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParamsForFake(const uint32 ProjectileId) const
{
    FActorSpawnParameters Params = GenerateSpawnParams();
    ACrashPlayerController* CrashPC = Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController);
    Params.CustomPreSpawnInitalization = [ProjectileId, CrashPC](AActor* Actor)
    {
        // TODO: Initialize projectile ID.
    };
    return Params;
}

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParamsForAuth(const uint32 ProjectileId) const
{
    FActorSpawnParameters Params = GenerateSpawnParams();
    Params.CustomPreSpawnInitalization = [ProjectileId](AActor* Actor)
    {
        // TODO: Initialize projectile ID.
    };
    return Params;
}
{% endhighlight %}

No, `CustomPreSpawnInitalization` is not a mistake; it's a typo in Unreal's source code.
{: .notice--info}

Note that we're making the `Instigator` our ASC's avatar (the player's pawn) and the `Owner` our ASC's owner (which is usually the player state). Having access to these objects will be important for our `Projectile` code.
{: .notice--info}

Now, back in `Activate`, we can use these parameters to spawn our fake projectile, after generating a new ID for it. (For now, we're assuming that our ping is low-enough to forward-predict without delaying our spawn.)

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::Activate()
{
    Super::Activate();
    
    if (Ability && Ability->GetCurrentActorInfo())
    {
        if (ACrashPlayerController* CrashPC = Ability->GetCurrentActorInfo()->PlayerController.IsValid() ? Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController.Get()) : nullptr)
        {
            const float ForwardPredictionTime = CrashPC->GetForwardPredictionTime();
            const bool bShouldPredict = (ForwardPredictionTime > 0.0f);
            const bool bIsNetAuthority = Ability->GetCurrentActorInfo()->IsNetAuthority();
            const bool bShouldUseServerInfo = IsLocallyControlled();
            
            if (!bIsNetAuthority && bShouldPredict)
            {
                // If our ping is low enough to forward-predict, immediately spawn and initialize the fake projectile.
                const uint32 FakeProjectileId = CrashPC->GenerateNewFakeProjectileId();
                if (AProjectile* NewProjectile = GetWorld()->SpawnActor<AProjectile>(ProjectileClass, SpawnLocation, SpawnRotation, GenerateSpawnParamsForFake(FakeProjectileId)))
                {
                    if (ShouldBroadcastAbilityTaskDelegates())
                    {
                        Success.Broadcast(NewProjectile);
                    }
                    
                    // We don't end the task here because we need to keep listening for possible rejection.
                    
                    return;
                }
            }
        }
    }
}
{% endhighlight %}

We need this return statement because we'll handle our fail-cases afterwards.
{: .notice--info}

If this task—or the ability that activated it—is eventually rejected by the server (since we're doing this predictively), we'll need to reconcile by destroying the fake projectile, so we should cache a reference to it.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

protected:

    /** The fake projectile spawned by this task on the client. Used to destroy the spawned projectile if this task is
     * rejected. */
    TWeakObjectPtr<AProjectile> SpawnedFakeProj;
{% endhighlight %}

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

                    // ...

                    // Cache the projectile in case the server rejects this task, and we have to destroy it.
                    SpawnedFakeProj = NewProjectile;

                    if (ShouldBroadcastAbilityTaskDelegates())
                    {
                        Success.Broadcast(NewProjectile);
                    }
                    
                    // We don't end the task here because we need to keep listening for possible rejection.
                    
                    return;

                    // ...
{% endhighlight %}

Lastly, we should start adding some debug information. Let's create a new log category called `LogProjectiles` to write our debug information. You can put this code anywhere, but a good place is a dedicated `Logging` file.

{% highlight c++ %}
// CrashLogging.h

/** Log channel for the projectile prediction system. */
GAME_API DECLARE_LOG_CATEGORY_EXTERN(LogProjectiles, Log, All);

/** Projectile log channel shorthand. */
#define PROJECTILE_LOG(Verbosity, Format, ...) \
{ \
    UE_LOG(LogProjectiles, Verbosity, Format, ##__VA_ARGS__); \
}
{% endhighlight %}

{% highlight c++ %}
// CrashLogging.cpp

DEFINE_LOG_CATEGORY(LogProjectiles);
{% endhighlight %}

Now, we can start adding debug writes to our projectile code.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::Activate()
{
                    // ...

                    PROJECTILE_LOG(Verbose, TEXT("(%i:%i.%i) (ID: %i): Successfully spawned fake projectile (%s) on time. Attempting to forward-predict (%fms) with ping (%fms). Client bias: (%i%%)."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), FakeProjectileId, *GetNameSafe(NewProjectile), ForwardPredictionTime * 1000.0f, CrashPC->PlayerState->ExactPing, (uint32)(CrashPC->ClientBiasPct * 100.0f));

                    // Cache the projectile in case the server rejects this task, and we have to destroy it.
                    SpawnedFakeProj = NewProjectile;

                    if (ShouldBroadcastAbilityTaskDelegates())
                    {
                        Success.Broadcast(NewProjectile);
                    }
                    
                    // We don't end the task here because we need to keep listening for possible rejection.
                    
                    return;

                    // ...
}
{% endhighlight %}

We'll be using the `Verbose` verbosity so our output log doesn't get spammed with projectile information when we're not actively trying to debug it. To see `Verbose` or `VeryVerbose` messages, you need to raise the log channel's verbosity level using the `log LogProjectiles verbose` or `log LogProjectiles veryverbose` console commands.
{: .notice--info}

### Deferring the Spawn

As noted earlier, if our client's ping is _higher_ than `MaxPredictionPing`, we need to delay spawning the projectile, so it isn't forward-predicted too far.

To do this, we need to cache the projectile's spawn data, set a timer, and then spawn the projectile with that data once it ends.

Let's start by defining a struct to store our spawn data.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

/**
 * Cached spawn information for predicted projectiles that are spawned after a delay. Projectile spawning is delayed
 * when the client's ping is higher than the maximum forward prediction ping to prevent projectiles from being
 * forward-predicted too far.
 */
USTRUCT()
struct FDelayedProjectileInfo
{
    GENERATED_BODY()
    
    UPROPERTY()
    TSubclassOf<AProjectile> ProjectileClass;
    
    UPROPERTY()
    FVector SpawnLocation;
    
    UPROPERTY()
    FRotator SpawnRotation;
    
    UPROPERTY()
    TWeakObjectPtr<ACrashPlayerController> CrashPC;
    
    UPROPERTY()
    uint32 ProjectileId;
    
    FDelayedProjectileInfo() :
        ProjectileClass(nullptr),
        SpawnLocation(ForceInit),
        SpawnRotation(ForceInit), 
        CrashPC(nullptr),
        ProjectileId(0)
    {}
};
{% endhighlight %}

Next, add a variable for storing this data to our task class.

{% highlight c++ %}
protected:

    /** Cached spawn info for spawning a fake projectile after a delay. */
    UPROPERTY()
    FDelayedProjectileInfo DelayedProjectileInfo;
{% endhighlight %}

While we're here, let's also define the handle we'll use for our delay timer, and the function we'll call when that timer ends.

{% highlight c++ %}
protected:
    
    /** Handle for spawning a fake projectile after a delay, when the client's ping is higher than the
     * forward-prediction limit. */
    FTimerHandle SpawnDelayedFakeProjHandle;
    
    /** Cached spawn info for spawning a fake projectile after a delay. */
    UPROPERTY()
    FDelayedProjectileInfo DelayedProjectileInfo;

    /** Spawns a fake projectile using the DelayedProjectileInfo. */
    void SpawnDelayedFakeProjectile();
{% endhighlight %}

Now, in our `Activate` function, before we try spawning the fake projectile, check if our ping is high enough to delay the spawn instead.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

            // ...

            if (!bIsNetAuthority && bShouldPredict)
            {
                /* On clients, if our ping is too high to forward-predict, delay spawning the projectile so we don't
                 * forward-predict further than MaxPredictionPing. */
                float SleepTime = CrashPC->GetProjectileSleepTime();
                if (SleepTime > 0.0f)
                {
                    if (!GetWorld()->GetTimerManager().IsTimerActive(SpawnDelayedFakeProjHandle))
                    {
                        // Set a timer to spawn the predicted projectile after a delay.
                        DelayedProjectileInfo.ProjectileClass = ProjectileClass;
                        DelayedProjectileInfo.SpawnLocation = SpawnLocation;
                        DelayedProjectileInfo.SpawnRotation = SpawnRotation;
                        DelayedProjectileInfo.CrashPC = CrashPC;
                        DelayedProjectileInfo.ProjectileId = CrashPC->GenerateNewFakeProjectileId();
                        GetWorld()->GetTimerManager().SetTimer(SpawnDelayedFakeProjHandle, this, &UAbilityTask_SpawnPredictedProjectile::SpawnDelayedFakeProjectile, SleepTime, false);
                        
                        PROJECTILE_LOG(Verbose, TEXT("(%i:%i.%i) (ID: %i): Spawning fake projectile delayed. Ping (%fms) exceeds maximum prediction time. Sleeping for (%fms) to forward-predict with maximum time (%fms) and latency reduction (%fms)."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), DelayedProjectileInfo.ProjectileId, CrashPC->PlayerState->ExactPing, SleepTime * 1000.0f, ForwardPredictionTime * 1000.0f, CrashPC->PredictionLatencyReduction);
                    }
                    
                    return;
                }
                
                // If our ping is low enough to forward-predict, immediately spawn and initialize the fake projectile.
                const uint32 FakeProjectileId = CrashPC->GenerateNewFakeProjectileId();
    
                // ...
{% endhighlight %}

When that timer ends, all we have to do is spawn the projectile using our cached info.

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::SpawnDelayedFakeProjectile()
{
    if (Ability && Ability->GetCurrentActorInfo() && DelayedProjectileInfo.CrashPC.IsValid())
    {
        if (AProjectile* NewProjectile = GetWorld()->SpawnActor<AProjectile>(DelayedProjectileInfo.ProjectileClass, DelayedProjectileInfo.SpawnLocation, DelayedProjectileInfo.SpawnRotation, GenerateSpawnParamsForFake(DelayedProjectileInfo.ProjectileId, DelayedProjectileInfo.CrashPC.Get())))
        {
            PROJECTILE_LOG(Verbose, TEXT("(%i:%i.%i) (ID: %i): Successfully spawned fake projectile (%s) delayed. Attempting to forward-predict (%fms) with ping (%fms)."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), DelayedProjectileInfo.ProjectileId, *GetNameSafe(NewProjectile), DelayedProjectileInfo.CrashPC->GetForwardPredictionTime() * 1000.0f, DelayedProjectileInfo.CrashPC->PlayerState->ExactPing);

            SpawnedFakeProj = NewProjectile;

            if (ShouldBroadcastAbilityTaskDelegates())
            {
                Success.Broadcast(NewProjectile);
            }

            // We don't end the task here because we need to keep listening for possible rejection.

            return;
        }
    }
}
{% endhighlight %}

Now, we can set up our ability script with something like this:

![Gameplay ability script]({{ '/' | absolute_url }}/assets/images/per-post/projectile-prediction-2/script.png){: .align-center}

We haven't actually implemented the `Projectile` class yet, so if you want to test this, you can subclass `Projectile` into a blueprint, add a mesh component, and use that blueprint as the `Projectile Class` parameter, so you can actaully see the projectile.
{: .notice--info}

... and with that, our fake projectile should be getting spawned successfully:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/projectile-prediction-2/spawning-fake-projectile.mp4" type="video/mp4">
    Video tag not supported.
</video>
<br>

I don't have animations for these abilities, so I'm using an input debugger to indicate exactly when the input is pressed, to show that the projectile appears instantly for the player when emulating low latency.
{: .notice--info}

And if you emulate network conditions with higher ping (greater than whatever you set `MaxPredictionPing` to), you should see a short delay before your fake projectile appears.

## Spawning the Authoritative Projectile

### Sending Spawn Data to the Server

To spawn our authoritative projectile, we need to send the spawn information from the client to the server.

If we were to use the same spawn parameters given in our constructor, our authoritative projectile would spawn in a different location than our fake projectile. This is because `Local Predicted` abilities are executed once on the local client, then again on the server once the client's "activate" input is replicated.

That means that if we were to set up our task like the image above, the parameters (determined by `GetPlayerViewPoint`) would be different on the client and the server; if we had `60ms` of ping, then the server would read those values `30ms` later, and thus end up with different values than the client read when it activated `30ms` prior.

We _could_ just use the server's values for the authoritative projectile, but it's better if we use the client's values, so both projectiles start with the same transform. This way, regardless of any fast-forwarding or rewinding, both projectiles will always follow the same trajectory, which makes reconciliation much easier and makes the game feel more accurate for the player firing the projectile.

Lucky for us, ability tasks give us an easy way to send our client's spawn information to the server without having to worry about RPCs: target data. Target data is a type of data structure that can be replicated between the client and the server using a collection of built-in methods in the ability system component.

To replicate our data, we first need to define a new target data struct to hold it:

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

/**
 * Target data for spawning projectiles. Used to send spawn information from the client to the server.
 */
USTRUCT()
struct FGameplayAbilityTargetData_ProjectileSpawnInfo : public FGameplayAbilityTargetData
{
    GENERATED_BODY()
    
    /** Location to spawn projectile at. */
    UPROPERTY()
    FVector SpawnLocation;
    
    /** Rotation with which to spawn projectile. */
    UPROPERTY()
    FRotator SpawnRotation;
    
    /** The projectile's ID. Used to link fake and authoritative projectiles. */
    UPROPERTY()
    uint32 ProjectileId;
    
    FGameplayAbilityTargetData_ProjectileSpawnInfo() :
        SpawnLocation(ForceInit),
        SpawnRotation(ForceInit),
        ProjectileId(0)
    {}
    
    virtual UScriptStruct* GetScriptStruct() const override
    {
        return FGameplayAbilityTargetData_ProjectileSpawnInfo::StaticStruct();
    }
    
    virtual FString ToString() const override
    {
        return FString::Printf(TEXT("FGameplayAbilityTargetData_ProjectileSpawnInfo: (%i)"), ProjectileId);
    }
    
    static FGameplayAbilityTargetDataHandle MakeProjectileSpawnInfoTargetData(const FVector& SpawnLocation, const FRotator& SpawnRotation, const uint32 ProjectileId)
    {
        FGameplayAbilityTargetData_ProjectileSpawnInfo* TargetData = new FGameplayAbilityTargetData_ProjectileSpawnInfo();
        TargetData->SpawnLocation = SpawnLocation;
        TargetData->SpawnRotation = SpawnRotation;
        TargetData->ProjectileId = ProjectileId;
        FGameplayAbilityTargetDataHandle Handle;
        Handle.Data.Add(TSharedPtr<FGameplayAbilityTargetData_ProjectileSpawnInfo>(TargetData));
        return Handle;
    }
    
    bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
    {
        Ar << SpawnLocation;
        Ar << SpawnRotation;
        Ar << ProjectileId;
        
        bOutSuccess = true;
        return true;
    }
};

template<>
struct TStructOpsTypeTraits<FGameplayAbilityTargetData_ProjectileSpawnInfo> : public TStructOpsTypeTraitsBase2<FGameplayAbilityTargetData_ProjectileSpawnInfo>
{
    enum
    {
        WithNetSerializer = true
    };
};
{% endhighlight %}

Now that we have a way to store our spawn data, let's make a helper function to send it to from the client to the server (since there are a couple of different places we may need to do so).

{% highlight c++ %}
protected:

    /** Replicates the client's spawn data to the server, so the server can spawn the authoritative projectile. */
    void SendSpawnDataToServer(const FVector& InLocation, const FRotator& InRotation, uint32 InProjectileId);
{% endhighlight %}

To replicate this data from the client to the server, all we need to do is create a new prediction window, and use it to call `CallServerSetReplicatedTargetData` on our owning ability system component, passing in our target data.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::SendSpawnDataToServer(const FVector& InLocation, const FRotator& InRotation, uint32 InProjectileId)
{
    const bool bGenerateNewKey = !AbilitySystemComponent->ScopedPredictionKey.IsValidForMorePrediction();
    FScopedPredictionWindow ScopedPrediction(AbilitySystemComponent.Get(), bGenerateNewKey);
    FGameplayAbilityTargetDataHandle Handle = FGameplayAbilityTargetData_ProjectileSpawnInfo::MakeProjectileSpawnInfoTargetData(InLocation, InRotation, InProjectileId);
    AbilitySystemComponent->CallServerSetReplicatedTargetData(GetAbilitySpecHandle(), GetActivationPredictionKey(), Handle, FGameplayTag(), AbilitySystemComponent->ScopedPredictionKey);
}
{% endhighlight %}

Now, we need to call this function in two places: after spawning our fake projectile in `Activate`, and after spawning our _delayed_ fake projectile in `SpawnDelayedFakeProjectile`:

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::Activate()
{
                    // ...

                    // Send the spawn info to the server so it can spawn the authoritative projectile.
                    SendSpawnDataToServer(SpawnLocation, SpawnRotation, FakeProjectileId);
                    
                    // Cache the projectile in case the server rejects this task, and we have to destroy it.
                    SpawnedFakeProj = NewProjectile;

                    // ...
}

void UAbilityTask_SpawnPredictedProjectile::SpawnDelayedFakeProjectile()
{
            // ...

            // Send the spawn info to the server so it can spawn the authoritative projectile.
            SendSpawnDataToServer(DelayedProjectileInfo.SpawnLocation, DelayedProjectileInfo.SpawnRotation, DelayedProjectileInfo.ProjectileId);

            SpawnedFakeProj = NewProjectile;

            // ...
}
{% endhighlight %}

### Listening for Spawn Data on the Server

On the server, we need to set up a listener to retrieve the data sent by the client. We need two callback functions:

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

protected:

    /** Spawns the authoritative projectile on the server when the spawn data is received from the client. */
    void OnSpawnDataReplicated(const FGameplayAbilityTargetDataHandle& Data, FGameplayTag Activation);
    
    /** Cancels this task on the server if the client failed to spawn their version of the projectile. */
    void OnSpawnDataCancelled();
{% endhighlight %}

Back in the `Activate` function, when the server activates their version of the task, instead of spawning the projectile (like we did on the client), we need to bind these callbacks.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

            // ...

            const bool bShouldUseServerInfo = IsLocallyControlled();

            // On the server (if it's not a listen server), wait for the client to send us the projectile's spawn info.
            if (bIsNetAuthority && !bShouldUseServerInfo)
            {
                const FGameplayAbilitySpecHandle& SpecHandle = GetAbilitySpecHandle();
                const FPredictionKey& ActivationPredictionKey = GetActivationPredictionKey();
                
                AbilitySystemComponent->AbilityTargetDataSetDelegate(SpecHandle, ActivationPredictionKey).AddUObject(this, &UAbilityTask_SpawnPredictedProjectile::OnSpawnDataReplicated);
                AbilitySystemComponent->AbilityTargetDataCancelledDelegate(SpecHandle, ActivationPredictionKey).AddUObject(this, &UAbilityTask_SpawnPredictedProjectile::OnSpawnDataCancelled);
                
                // Check if the client already sent the data.
                AbilitySystemComponent->CallReplicatedTargetDataDelegatesIfSet(SpecHandle, ActivationPredictionKey);
                
                // Kill the ability if we never receive the data.
                SetWaitingOnRemotePlayerData();
                
                return;
            }
            
            if (!bIsNetAuthority && bShouldPredict)

            // ...
}
{% endhighlight %}

Now, the server should successfully receive the client's spawn data when the task is activated.

### Spawning the Projectile

Once our data is replicated, we can use it to spawn the authoritative projectile. But first, we need to define a new function to _consume_ the data. This will prevent tasks from spawning multiple projectiles if we make multiple calls in a single ability script (for example, if we wanted to have a burst-projectile ability).

This function will be called `TryConsumeClientReplicatedTargetData`, and we'll put it in our game's ability system component class. This function is the same as `UAbilitySystemComponent::ConsumeClientReplicatedTargetData`, but it also returns whether the data was actually consumed. This allows us to only proceed with spawning the projectile if the data hasn't _already_ been consumed. In other words, it prevents us from using a single target data multiple times, ensuring each target data generated by each instance of the task is only used to spawn exactly one projectile.

This is something we have to account for just because of how target data replication, specifically, is programmed, so don't worry if you don't understand this on a technical level. If you _do_ want a more technical explanation:
<br>
<br>
When we "send" target data to the server, we're not actually sending it, like we would with an RPC. We're instead adding it to an `FGameplayAbilityReplicatedDataContainer`, which is basically a replicated map that holds runtime ability data (which helps prevent data races, which is why we don't have to worry about waiting for the server before sending our target data).
<br>
<br>
On the server, the listener we set up tell us when this container changes; i.e. when the client adds some new piece of data to it, which gets replicated to us. If the new data has a matching ability spec and prediction key (meaning it was sent by the same ability that our task is in), the listener will fire a callback function with the new data.
<br>
<br>
But say we call this task twice in quick succession (such as a "burst-fire" ability). Both tasks will begin listening for the data container to update, but once it does, it will trigger _both_ tasks' listeners, since these tasks use the same ability spec _and_ the same prediction key, because they were both executed in the same ability. As a result, the first task would execute successfully, but then the second task would re-use the first task's data, instead of waiting for _its_ respective data.
<br>
<br>
We can solve this by making the first task "consume" the data, in order to prevent any other tasks from re-using it, by calling `ConsumeClientReplicatedTargetData`. This function searches the replicated data container for any data with a matching ability spec and prediction key and removes it.
<br>
<br>
Unfortunately, this function doesn't actually solve our problem, because it doesn't tell us if anything was _actually_ removed. Since the replicated data gets copied for each invocation, both instances of our task will still trigger their respective callbacks with the same data.
<br>
<br>
So, to solve this, we create a new function called `TryConsumeClientReplicatedTargetData`, which does the same thing as `ConsumeClientReplicatedTargetData`, but also returns whether any data was actually removed. This way, when our first task triggers the callback, it will successfully remove the data and continue. But when the _second_ task triggers its callback, it will try to remove the same data and _fail_. When this happens, we can exit out of the callback function early, and keep listening for _our_ data to get replicated, which we'll be able to successfully consume.
{: .notice--info}

{% highlight c++ %}
// CrashAbilitySystemComponent.h

public:

    /** Consumes cached TargetData from client (only TargetData) and returns whether any data was actually consumed. */
    bool TryConsumeClientReplicatedTargetData(FGameplayAbilitySpecHandle AbilityHandle, FPredictionKey AbilityOriginalPredictionKey);
{% endhighlight %}

{% highlight c++ %}
// CrashAbilitySystemComponent.cpp

bool UCrashAbilitySystemComponent::TryConsumeClientReplicatedTargetData(FGameplayAbilitySpecHandle AbilityHandle, FPredictionKey AbilityOriginalPredictionKey)
{
    TSharedPtr<FAbilityReplicatedDataCache> CachedData = AbilityTargetDataMap.Find(FGameplayAbilitySpecHandleAndPredictionKey(AbilityHandle, AbilityOriginalPredictionKey));
    if (CachedData.IsValid())
    {
        const bool bConsumed = CachedData->TargetData.Num() > 0;
        CachedData->TargetData.Clear();
        CachedData->bTargetConfirmed = false;
        CachedData->bTargetCancelled = false;
        return bConsumed;
    }
    return false;
}
{% endhighlight %}

Now, we can use this new function to consume the replicated data before we spawn the authoritative projectile.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::OnSpawnDataReplicated(const FGameplayAbilityTargetDataHandle& Data, FGameplayTag Activation)
{
    // Copy the target data before we consume it.
    const FGameplayAbilityTargetData* TargetData = Data.Get(0);

    // Consume the client's data. Ensures each server task only spawns one projectile for each client task.
    if (!Cast<UCrashAbilitySystemComponent>(AbilitySystemComponent)->TryConsumeClientReplicatedTargetData(GetAbilitySpecHandle(), GetActivationPredictionKey()))
    {
        return;
    }

    if (TargetData)
    {
        if (const FGameplayAbilityTargetData_ProjectileSpawnInfo* SpawnInfo = static_cast<const FGameplayAbilityTargetData_ProjectileSpawnInfo*>(TargetData))
        {
            ACrashPlayerController* CrashPC = Ability->GetCurrentActorInfo()->PlayerController.IsValid() ? Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController.Get()) : nullptr;
            const float ForwardPredictionTime = CrashPC->GetForwardPredictionTime();
            
            if (AProjectile* NewProjectile = GetWorld()->SpawnActor<AProjectile>(ProjectileClass, SpawnInfo->SpawnLocation, SpawnInfo->SpawnRotation, GenerateSpawnParamsForAuth(SpawnInfo->ProjectileId)))
            {
                // Note that there will be a discrepancy between the server's perceived ping and the client's.
                PROJECTILE_LOG(Verbose, TEXT("(%i:%i.%i) (ID: %i): Successfully spawned authoritative projectile (%s). Forwarded (%fms) for perceived ping (%fms). Latency reduction: (%fms) Client bias: (%i%%)"), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), SpawnInfo->ProjectileId, *GetNameSafe(NewProjectile), ForwardPredictionTime * 1000.0f, CrashPC->PlayerState->ExactPing, CrashPC->PredictionLatencyReduction, (uint32)(CrashPC->ClientBiasPct * 100.0f));
            
                if (NewProjectile->ProjectileMovement)
                {
                    // Tick the actor (e.g. animations, VFX).
                    if (NewProjectile->PrimaryActorTick.IsTickFunctionEnabled())
                    {
                        NewProjectile->TickActor(ForwardPredictionTime * NewProjectile->CustomTimeDilation, LEVELTICK_All, NewProjectile->PrimaryActorTick);
                    }
                    
                    // Tick the movement component (to actually move the projectile).
                    NewProjectile->ProjectileMovement->TickComponent(ForwardPredictionTime * NewProjectile->CustomTimeDilation, LEVELTICK_All, nullptr);
                    if (NewProjectile->GetLifeSpan() > 0.0f)
                    {
                        /* Since we're fast-forwarding this actor, we need to subtract the forward prediction time from
                         * its lifespan. Clamp at 0.2 so we have enough time to replicate. */
                        NewProjectile->SetLifeSpan(FMath::Max(0.2f, NewProjectile->GetLifeSpan() - ForwardPredictionTime));
                    }
                }

                if (ShouldBroadcastAbilityTaskDelegates())
                {
                    Success.Broadcast(NewProjectile);
                }
                
                EndTask();
                
                return;
            }
        }
    }

    if (ShouldBroadcastAbilityTaskDelegates())
    {
        FailedToSpawn.Broadcast(nullptr);
    }
    
    EndTask();
}
{% endhighlight %}

You can see that this is also where we fast-forward the projectile. We don't have a reference to our projectile's movement component, since we haven't created it yet, so you'll have an error for now. If you want to test this code before implementing the projectile class, you can just comment this part out.
{: .notice--info}

If our data _fails_ to replicate (because the ability was cancelled or rejected), we need to cancel the server's version of the task.

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::OnSpawnDataCancelled()
{
    if (ShouldBroadcastAbilityTaskDelegates())
    {
        FailedToSpawn.Broadcast(nullptr);
    }
    
    EndTask();
}
{% endhighlight %}

### Handling Failure

`OnSpawnDataCancelled` accounts for situations where our data doesn't get replicated, but we've also handled another fail-case at the end of `OnSpawnDataReplicated`: if the server fails to spawn the projectile, we're canceling the task and executing the `FailedToSpawn` output pin. Together, these account for possible failures on the server side. But we also need to handle failures on the client side.

When a client fails to spawn their projectile, or when their task is rejected (since any rejection on the server will get replicated to clients), we need to do the same thing: cancel the task and execute `FailedToSpawn`. But we _also_ need to make sure we cancel our target data replication, so the server doesn't try to spawn the authoritative projectile.

Let's make a helper function to cancel our target data:

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

protected:

    /** Sends the task cancellation to the server if the client failed. */
    void CancelServerSpawn();
{% endhighlight %}

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::CancelServerSpawn()
{
    const bool bGenerateNewKey = !AbilitySystemComponent->ScopedPredictionKey.IsValidForMorePrediction();
    FScopedPredictionWindow ScopedPrediction(AbilitySystemComponent.Get(), bGenerateNewKey);
    AbilitySystemComponent->ServerSetReplicatedTargetDataCancelled(GetAbilitySpecHandle(), GetActivationPredictionKey(), AbilitySystemComponent->ScopedPredictionKey);
}
{% endhighlight %}

There are two places we need to cancel our task: the end of `Activate` and the end of `SpawnDelayedFakeProjectile` (hence why we put `return` statements after each successful branch).

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::Activate()
{
    // ...
    
    // Cancel the task on the server if the client failed.
    CancelServerSpawn();
    
    // Failed to spawn. Ability should usually be cancelled at this point.
    if (ShouldBroadcastAbilityTaskDelegates())
    {
        FailedToSpawn.Broadcast(nullptr);
    }
    
    EndTask();
}

void UAbilityTask_SpawnPredictedProjectile::SpawnDelayedFakeProjectile()
{
    // ...

    // Cancel the task on the server if the client failed.
    CancelServerSpawn();

    // Failed to spawn. Ability should usually be cancelled at this point.
    if (ShouldBroadcastAbilityTaskDelegates())
    {
        FailedToSpawn.Broadcast(nullptr);
    }
    
    EndTask();
}
{% endhighlight %}

Now, if either the client _or_ server fail to spawn their projectile, the entire task will be canceled, and `FailedToSpawn` will be triggered in both the client _and_ the server's ability. This lets us reliably handle these failures in our ability script (usually just canceling the ability), with the guarantee that `FailedToSpawn` will always be executed on both versions of the ability.

The last possible failure situation we need to handle is if our client's task is rejected (usually because the ability was rejected on the server). When this happens, we need to destroy our fake projectile, and remove it from our player controller's list of unlinked fake projectiles. Let's create one more callback function to handle that.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.h

protected:

    /** Destroys the client's fake projectile if this task is rejected (e.g. the server rejects the ability
     * activation). */
    UFUNCTION()
    void OnTaskRejected();
{% endhighlight %}

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

void UAbilityTask_SpawnPredictedProjectile::OnTaskRejected()
{
    PROJECTILE_LOG(Warning, TEXT("(ID: %i): SpawnPredictedProjectile task in ability (%s) rejected. Destroying fake projectile (%s)..."), *GetNameSafe(SpawnedFakeProj.Get()), *GetNameSafe(Ability), *GetNameSafe(SpawnedFakeProj.Get()));
    
    ACrashPlayerController* CrashPC = (Ability && Ability->GetCurrentActorInfo()) ? Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController) : nullptr;
    
    // If we've spawned a fake projectile on the client, destroy it.
    if (SpawnedFakeProj.IsValid())
    {
        // The fake projectile will still be lingering on the PC's list of unlinked projectiles; we need to remove it.
        if (CrashPC)
        {
            if (const uint32* Key = CrashPC->FakeProjectiles.FindKey(SpawnedFakeProj.Get()))
            {
                UE_LOG(LogProjectiles, Error, TEXT("Removed %i"), *Key);
                CrashPC->FakeProjectiles.Remove(*Key);
            }
        }
        
        SpawnedFakeProj.Get()->Destroy();
    }
    
    // If we didn't spawn the fake projectile yet (because we're waiting for a delayed spawn), cancel it.
    GetWorld()->GetTimerManager().ClearAllTimersForObject(this);
}
{% endhighlight %}

Finally, we can just bind this function to GAS's built-in prediction system, at the start of `Activate`.

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::Activate()
{
    Super::Activate();
    
    // On the client, listen for if this task is rejected. If it is, we need to destroy our fake projectile.
    if (IsPredictingClient())
    {
        GetActivationPredictionKey().NewRejectedDelegate().BindUObject(this, &UAbilityTask_SpawnPredictedProjectile::OnTaskRejected);
    }

    // ...
{% endhighlight %}

### Handling Listen Servers/Standalone

There's one situation we haven't accounted for yet: listen servers. If a player on a listen server spawns a projectile, we don't want to spawn a fake projectile (since they don't have to worry about latency), and we don't want to send spawn data to the server (since they _are_ the server). Instead, we can simply spawn the authoritative projectile.

{% highlight c++ %}
void UAbilityTask_SpawnPredictedProjectile::Activate()
{
            // ...

            if (!bIsNetAuthority && bShouldPredict)
            {
                // ...
            }
            /* On listen servers or in standalone, spawn the authoritative projectile. No prediction is needed in this
             * case. Remote servers don't spawn authoritative actors until OnTargetDataReplicated. */
            else if (bIsNetAuthority && bShouldUseServerInfo)
            {
                if (AProjectile* NewProjectile = GetWorld()->SpawnActor<AProjectile>(ProjectileClass, SpawnLocation, SpawnRotation, GenerateSpawnParams()))
                {
                    PROJECTILE_LOG(Verbose, TEXT("(%i:%i.%i) (ID: N/A): Successfully spawned authoritative projectile (%s) on time for local server. No prediction performed."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), *GetNameSafe(NewProjectile));
                    
                    if (ShouldBroadcastAbilityTaskDelegates())
                    {
                        Success.Broadcast(NewProjectile);
                    }
                    
                    EndTask();
                    
                    return;
                }
            }

           // ...
{% endhighlight %}

Now, we should _finally_ have our finished task. We should now be seeing a fake projectile being spawned on clients, and an authoritative projectile being spawned on servers:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/projectile-prediction-2/final-task-demo.mp4" type="video/mp4">
    Video tag not supported.
</video>
<br>
We can even test our rejection handling by adding the following script to our ability's `CanActivate` function:

![Script to reject ability activation on the server only]({{ '/' | absolute_url }}/assets/images/per-post/projectile-prediction-2/rejection-test-script.png){: .align-center}

Now, since the server will always reject the ability, we should see our client's fake projectile being destroyed by the rejection:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/projectile-prediction-2/task-rejection-testing.mp4" type="video/mp4">
    Video tag not supported.
</video>
<br>
## What's Next

Both of our projectiles are now being properly spawned, but we still need to finish linking them together before we can start implementing any logic.

In the next section. we'll begin implementing the `Projectile` class (and finishing some missing code in this class) to link the fake and authoritative projectiles together once they're spawned.
