---
layout: single
title: "Projectile Prediction: Part 3"
excerpt: Linking fake and authoritative projectile actors.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-07-11
---

The third part of a series exploring and implementing projectile prediction for multiplayer games. This part walks through initializing the base `AProjectile` actor class, which can be subclassed into projectiles that can be spawned by our `SpawnPredictedProjectile` task.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...).
{: .notice--info}

## Introduction

In the previous section, we implemented an ability task to predictively spawn projectile actors. In this section, we'll finish initializing and linking our projectiles together, which I intentionally left unfinished in the last section.

This is the last step-by-step coding walkthrough in this series because, once the projectiles have been initialized and linked, you can implement the class itself however you want, in whichever way best suits your project.

In the next and last part of this series, I'll show you my own implementation, which is extremely configurable, and should be suitable for most games. However, given the scale and complexity of the class, a step-by-step walkthrough of the code isn't practical. So, instead, I'll simply be going through the class's final code, providing a breakdown and explanation of each of its features: movement, damage, visual effects, server reconciliation, debugging, etc..

But first, let's get our projectiles linked.

## Initialization

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
    void InitFakeProjectile(AMyPlayerController* OwningPlayer, uint32 InProjectileId);

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

To link the two projectiles (which we'll do in the authoritative projectile's `BeginPlay`), we just have each actor cache a reference to the other.

{% highlight c++ %}
// Projectile.cpp

void AProjectile::InitFakeProjectile(AMyPlayerController* OwningPlayer, uint32 InProjectileId)
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

Finally, we'll link the projectiles together in `BeginPlay`. Since the authoritative projectile has to be spawned by and replicated from the server, we know it will _always_ be spawned _after_ the fake projectile. Since we need both projectiles to link them, we wait until the authoritative projectile is replicated to the local client, and calls `BeginPlay` to try to link them, since we know that this is the earliest point at which both projectiles will exist.

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

    AMyPlayerController* PC = GetInstigator() ? GetInstigatorController<AMyPlayerController>() : nullptr;
    if (!ensureAlwaysMsgf(IsValid(PC), TEXT("Instigating player controller could not be found for predicted projectile projectile (%s). Failed to find player controller with instigator (%s). Predicted projectiles must be spawned with an instigator with a valid player controller."), *GetName(), *GetNameSafe(GetInstigator())))
    {
        Destroy();
        return;
    }

    // Initialize the replicated authoritative projectile once it's replicated to the owning client.
    if (!HasAuthority() && (ProjectileId != NULL_PROJECTILE_ID))
    {
        PROJECTILE_LOG(Verbose, TEXT("(%i:%i:%i) (ID: %i): Successfully replicated authoritative projectile (%s) to client."), FDateTime::UtcNow().GetMinute(), FDateTime::UtcNow().GetSecond(), FDateTime::UtcNow().GetMillisecond(), ProjectileId, *GetName());

        // Link to the associated fake projectile.
        if (ensureAlwaysMsgf(PC->FakeProjectiles.Contains(ProjectileId), TEXT("Client-side authoritative projectile (%s) failed to find corresponding fake projectile with ID (%i)."), *GetNameSafe(this), ProjectileId))
        {
            LinkFakeProjectile(PC->FakeProjectiles[ProjectileId]);
            PC->FakeProjectiles.Remove(ProjectileId);
        }
    }
}
{% endhighlight %}

Since we're going to call `InitFakeProjectile` during construction, `ProjectileId` will always be set before `BeginPlay`. And because we've set it to only replicate to the owning machine, we can easily identify whether we're the authoritative projectile on the owning client (that's what I mean when I say the "owning client's version" of the authoritative projectile): if we have authority, that means we're either the fake projectile (since we spawned it locally) or the authoritative projectile on the _server_; if we _don't_ have a projectile ID, that means we're the authoritative projectile on a _non-owning_ client (if we were on the owning client, the ID would have been replicated to us).
{: .notice--info}

Now that our linking is set up, we just need to call `InitProjectileId` and `InitFakeProjectile`. To do that, we're going to go back to our `SpawnPredictedProjectile` task class to fill in some missing code.

{% highlight c++ %}
// AbilityTask_SpawnPredictedProjectile.cpp

FActorSpawnParameters UAbilityTask_SpawnPredictedProjectile::GenerateSpawnParamsForFake(const uint32 ProjectileId) const
{
    FActorSpawnParameters Params = GenerateSpawnParams();
    AMyPlayerController* PC = Cast<AMyPlayerController>(Ability->GetCurrentActorInfo()->PlayerController);
    Params.CustomPreSpawnInitalization = [ProjectileId, PC](AActor* Actor)
    {
        if (AProjectile* Projectile = Cast<AProjectile>(Actor))
        {
            Projectile->InitProjectileId(ProjectileId);
            Projectile->InitFakeProjectile(PC, ProjectileId);
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

Remember to raise the `LogProjectiles` channel's verbosity level to `Verbose` or `VeryVerbose`.
{: .notice--info}

### What's Next

