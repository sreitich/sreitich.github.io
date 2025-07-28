---
layout: single
title: "Projectile Prediction: Part 3"
excerpt: Linking fake and authoritative projectile actors.
header:
    teaser: /assets/images/per-post/projectile-prediction-3/projectile-prediction-3-teaser.png
author: Meta
last_modified_at: 2025-07-27
---

Part 3 of a series exploring and implementing projectile prediction for multiplayer games. This part walks through initializing the base `Projectile` actor class, which can be subclassed into projectiles that can be spawned by our `SpawnPredictedProjectile` task.

The code used for this series can be found on [Unreal Engine's Learning site](https://dev.epicgames.com/community/learning/tutorials/LZ66/projectile-prediction-in-unreal-engine). This section is a step-by-step walkthrough to finishing the `UAbilityTask_SpawnPredictedProjectile` class. If you don't want a detailed walkthrough, you can copy the code directly.
{: .notice--info}

This code was written for a game called [_Cloud Crashers_](https://store.steampowered.com/app/2995940/Cloud_Crashers/), and uses project-specific classes named as such. For your game, you'll need to replace the `ACrashPlayerController` class with your game's player controller class.
{: .notice--info}

## Introduction

In the previous section, we implemented an ability task to predictively spawn projectile actors. In this section, we'll finish initializing and linking our projectiles together, which we left unfinished.

This is the last coding walkthrough in this series because, once the projectiles have been initialized and linked, you can implement the class itself however you want, in whichever way best suits your project.

In the subsequent and final section of this series, we'll examine the implementation we use for _Cloud Crashers_, which is highly configurable, and should be suitable for most games. Given the length and complexity of the class, a step-by-step walkthrough wouldn't be practical. So we'll instead be going through the class's key features and techniques (movement, damage, visual effects, reconciliation, etc.), breaking down how each one works.

But first, let's get our projectiles linked.

## Linking

We set up a "projectile ID" system in the last section, but we never actually assigned or used any of the IDs we generated. Let's start by giving our projectile class a member to store an ID, and a setter to initialize it.

{% highlight c++ %}
// Projectile.h

public:

    /** Default constructor. */
    AProjectile(const FObjectInitializer& ObjectInitializer);

    /** Initialize this projectile's ID. */
    FORCEINLINE void InitProjectileId(uint32 InProjectileId) { ProjectileId = InProjectileId; }

protected:

    /** This projectile's ID. Used to link fake and authoritative projectiles. Only valid on the owning client (i.e. the
     * client with the fake projectile). NULL_PROJECTILE_ID on other machines. Set before BeginPlay. */
    UPROPERTY(Replicated)
    uint32 ProjectileId;
{% endhighlight %}

And let's make sure we enable replication so we can replicate this variable.

We'll be calling `InitProjectileId` on construction, before the first replication tick, and since a projectile's ID should never change, we don't need to worry about marking `ProjectileId` as dirty.
{: .notice--info}

{% highlight c++ %}
// Projectile.cpp

AProjectile::AProjectile(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    bReplicates = true;
    NetPriority = 2.0f;
    SetMinNetUpdateFrequency(100.0f);
    ProjectileId = NULL_PROJECTILE_ID;
}

void AProjectile::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    constexpr bool bUsePushModel = true;

    FDoRepLifetimeParams OwnerOnlyParams{COND_OwnerOnly, REPNOTIFY_Always, bUsePushModel};
    DOREPLIFETIME_WITH_PARAMS_FAST(AProjectile, ProjectileId, OwnerOnlyParams);
}
{% endhighlight %}

We'll also need to know whether this actor is a fake projectile or the authoritative projectile, so let's add another variable to track that.

{% highlight c++ %}
// Projectile.h

public:

    /** Initialize this projectile as the fake projectile. */
    void InitFakeProjectile(ACrashPlayerController* OwningPlayer, uint32 InProjectileId);

protected:

    /** Whether this projectile is a fake client-side projectile. Set before BeginPlay. */
    bool bIsFakeProjectile;
{% endhighlight %}

Lastly, let's add the actual function we'll call to link the fake and authoritative projectiles, and a way for them to reference one-another once they're linked.

{% highlight c++ %}
protected:

    /** The fake projectile that is representing this actor on the owning client, if this actor is the authoritative
     * server-side projectile. */
    UPROPERTY()
    TObjectPtr<AProjectile> LinkedFakeProjectile;

    /** The authoritative server-side projectile that is representing this actor on the server and remote clients, if
     * this actor is the fake client-side projectile. */
    UPROPERTY()
    TObjectPtr<AProjectile> LinkedAuthProjectile;

    /** Link this authoritative projectile with its corresponding fake projectile. */
    void LinkFakeProjectile(AProjectile* InFakeProjectile);
{% endhighlight %}

When we initialize a projectile as the fake one, we'll add it to the player controller's list of unlinked fake projectiles for the authoritative projectile to link to once it's been spawned. (We don't need to worry about replication, since the fake projectile only exists locally, so we only need to link it locally.)

To link the two projectiles (which we'll do in the authoritative projectile's `BeginPlay`), we'll just have each actor cache a reference to the other.

{% highlight c++ %}
// Projectile.cpp

void AProjectile::InitFakeProjectile(ACrashPlayerController* OwningPlayer, uint32 InProjectileId)
{
    if (InProjectileId == NULL_PROJECTILE_ID)
    {
        return;
    }

    bIsFakeProjectile = true;
    if (OwningPlayer)
    {
        OwningPlayer->FakeProjectiles.Add(InProjectileId, this);
    }
}

void AProjectile::LinkFakeProjectile(AProjectile* InFakeProjectile)
{
    LinkedFakeProjectile = InFakeProjectile;
    InFakeProjectile->LinkedAuthProjectile = this;
}
{% endhighlight %}

Finally, we'll link the projectiles together in `BeginPlay`. Since the authoritative projectile has to be spawned by and replicated from the server, we know it will _always_ be spawned _after_ the fake projectile. That means that the earliest point at which both projectiles will exist (since we need both to link them together) will be the authoritative projectile's `BeginPlay`, which is called as soon as the authoritative projectile is replicated back to the local client.

{% highlight c++ %}
// Projectile.h

    /** Tries to link the client's version of the authoritative projectile to its corresponding fake projectile. */
    virtual void BeginPlay() override;
{% endhighlight %}

{% highlight c++ %}
// Projectile.cpp

void AProjectile::BeginPlay()
{
    Super::BeginPlay();

    ACrashPlayerController* CrashPC = GetInstigator() ? GetInstigatorController<ACrashPlayerController>() : nullptr;
    if (!ensureAlwaysMsgf(IsValid(CrashPC), TEXT("Instigating player controller could not be found for predicted projectile projectile (%s). Failed to find player controller with instigator (%s). Predicted projectiles must be spawned with an instigator with a valid player controller."), *GetName(), *GetNameSafe(GetInstigator())))
    {
        Destroy();
        return;
    }

    // Initialize the replicated authoritative projectile once it's replicated to the owning client.
    if (!HasAuthority() && (ProjectileId != NULL_PROJECTILE_ID))
    {
        PROJECTILE_LOG(Verbose, TEXT("(%i:%i:%i) (ID: %i): Successfully replicated authoritative projectile (%s) to client."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), ProjectileId, *GetName());

        // Link to the associated fake projectile.
        if (ensureAlwaysMsgf(CrashPC->FakeProjectiles.Contains(ProjectileId), TEXT("Client-side authoritative projectile (%s) failed to find corresponding fake projectile with ID (%i)."), *GetNameSafe(this), ProjectileId))
        {
            LinkFakeProjectile(CrashPC->FakeProjectiles[ProjectileId]);
            CrashPC->FakeProjectiles.Remove(ProjectileId);
        }
    }
}
{% endhighlight %}

Since we're going to call `InitFakeProjectile` during construction, `ProjectileId` will always be set before `BeginPlay`. And because we've set it to only replicate to the owning machine (with `COND_OwnerOnly`), we can easily identify whether we're the authoritative projectile on the owning client: if we have a projectile ID, that means we're either the fake projectile on the owning client (the only place it exists), the authoritative projectile on the server, or the authoritative projectile on the _owning_ client (again, the only place `ProjectileId` is replicated to); and if we have _authority_, we're either the fake projectile (since it's spawned locally) or the authoritative projectile on the server. Therefore, if we _do_ have a projectile ID and _don't_ have authority, we must be the authoritative projectile on the owning client.
<br>
<br>
In case it's not clear, the reason we only want to link our projectiles on the owning client is because that's the only place where the fake projectile is. The whole point of the fake projectile is to make _their_ game feel responsive. Everyone else (including the server) just sees the authoritative projectile.
{: .notice--info}

Now that our linking is set up, we just need to call `InitProjectileId` and `InitFakeProjectile`. To do that, we're going to go back to our `SpawnPredictedProjectile` task class to fill in some missing code.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParamsForFake(const uint32 ProjectileId) const
{
    FActorSpawnParameters Params = GenerateSpawnParams();
    ACrashPlayerController* CrashPC = Cast<ACrashPlayerController>(Ability->GetCurrentActorInfo()->PlayerController);
    Params.CustomPreSpawnInitalization = [ProjectileId, CrashPC](AActor* Actor)
    {
        if (AProjectile* Projectile = Cast<AProjectile>(Actor))
        {
            Projectile->InitProjectileId(ProjectileId);
            Projectile->InitFakeProjectile(CrashPC, ProjectileId);
        }
    };
    return Params;
}

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParamsForAuth(const uint32 ProjectileId) const
{
    FActorSpawnParameters Params = GenerateSpawnParams();
    Params.CustomPreSpawnInitalization = [ProjectileId](AActor* Actor)
    {
        if (AProjectile* Projectile = Cast<AProjectile>(Actor))
        {
            Projectile->InitProjectileId(ProjectileId);
        }
    };
    return Params;
}
{% endhighlight %}

Now, our projectiles should both be getting spawned _and_ linked together. We'll know that they are if we get the `Successfully replicated...` debug message on the server, and don't get an `ensure` error.

Remember to raise the `LogProjectiles` channel's verbosity level to `Verbose` or `VeryVerbose` with the `log LogProjectiles Verbose` command.
{: .notice--info}

## What's Next

Now that we have our projectiles spawned and linked, all we need to do is implement the `Projectile` class itself. You can do this however you want, but in the final section of this series, we'll be breaking down the implementation we use for all of the projectiles in _Cloud Crashers_, which includes features like projectile movement, hit detection, effects, and reconciliation.