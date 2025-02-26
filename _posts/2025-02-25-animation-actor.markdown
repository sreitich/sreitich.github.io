---
layout: single
title: "Animation Actors"
excerpt: Learn to create an animation notify that lets you add actors to your animations.
header:
  teaser: /assets/images/per-post/anim-actors/anim-actors-teaser.png
author: Meta
---

Learn to create an animation notify that lets you add actors to your animations.

![Locomotion state machine]({{ '/' | absolute_url }}/assets/images/per-post/anim-actors/anim-actors-teaser.png){: .align-center}

**_This page is still a work-in-progress!_**
{: .notice--info}

## Introduction

In a lot of ability-centric video games (fighting games, hero shooters, so on), characters often use actors in their ability animations:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/anim-actors/anim-actors-example-multiversus-bugs-bunny.mp4" type="video/mp4">
    Video tag not supported.
</video>

_Bugs Bunny spawns a hammer for his basic attacks in **MultiVersus**._
{: .notice--info}

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/anim-actors/anim-actors-example-overwatch-jq.mp4" type="video/mp4">
    Video tag not supported.
</video>

_Junker Queen spawns an axe for her **Carnage** ability in **Overwatch**._
{: .notice--info}

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/anim-actors/anim-actors-example-smash-mario.mp4" type="video/mp4">
    Video tag not supported.
</video>

_Mario spawns a cape for his side special in **Super Smash Bros.**._
{: .notice--info}

Unlike most of the actors we deal with when making games, which usually compose or interact with the game's framework or gameplay systems (pawns, controllers, environment actors, etc.), these actors (which I'll be calling **animation actors**) serve no gameplay purpose and have no side effects. They function more like visual effects: transient and purely cosmetic.

### Possible Approaches

Trying to manage these actors inside our ability logic — e.g. calling `Play Animation`, then calling `Spawn Actor` in a blueprint script — wouldn't be practical. It would be unintuitive for our animators, make the animations difficult to iterate on, and could result in visual synchronization issues between the animation and the actor if we didn't batch these calls properly in a multiplayer game.

Since these actors are effectively a part of the animation, we should manage them from within the animation itself. The solution for this in Unreal Engine is the `Animation Notify`: a timed event you can add to animations directly in the timeline.

As of UE 5.5, Unreal doesn't have a built-in notify for spawning actors, so I'll be showing you how to create one. This notify will be able to spawn actors within an animation (we'll be spawning mesh actors, but you could modify this notify to allow any actor to be spawned). It will support both static and skeletal meshes, allow you to play animations on spawned skeletal mesh actors, and allow you to override the mesh's materials, and we'll be able to preview it in real-time in the animation editor.

If you want to skip the tutorial and just see the code, you can find the source code for this class [here](https://github.com/ChangeStudios/ProjectCrash/blob/release/Source/ProjectCrash/Animation/AnimNotifies/AnimNotifyState_SpawnAnimationActor.h) But note that it has some features specific to _[Cloud Crashers](https://store.steampowered.com/app/2995940/Cloud_Crashers/)_ (the game I created this for), like support for first- and third-person animations.
{: .notice--info}

## Implementation

### Creating a Notify Class

Animation notifies come in two flavors: `AnimNotify` and `AnimNotifyState`. The former is a simple "fire-and-forget" event that executes its code and disappears. But the latter instantiates an object that remains alive for a specified duration within its animation, maintaining a persistent state while active.

We want our spawned actors to be destroyed automatically—either at the end of the animation, or at a specified time within our animation. Normal animation notifies don't maintain a state after being triggered, so they have no way to store a reference to the spawned actor to destroy it later.

Instead, we'll use a notify state that spawns our animation actor when it starts, and destroys the actor when it ends. Since notify states are stateful, we'll be able to cache a reference to the spawned actor at the start of the notify, so we can destroy it at the end.

Let's start by creating the new animation notify class. The parent class should be `AnimNotifyState`, and we'll name it `AnimNotifyState_SpawnAnimActor`.

`AnimNotify_` and `AnimNotifyState_` are the standard naming conventions for animation notifies in Unreal.
{: .notice--info}

![Unreal Engine C++ class wizard]({{ '/' | absolute_url }}/assets/images/per-post/anim-actors/anim-actors-new-class.png){: .align-center}

The first thing we want to do is add some important [class specifiers](https://dev.epicgames.com/documentation/en-us/unreal-engine/class-specifiers) to our new UClass:

{% highlight c++ %}
UCLASS(Const, HideCategories = Object, CollapseCategories, DisplayName = "Spawn Animation Actor")
class GAME_API UAnimNotifyState_SpawnAnimActor : public UAnimNotifyState
{
    // ...
{% endhighlight %}

- `Const`: Prevents our notify from changing during runtime. We can still edit our notifies in the editor, but we wouldn't want them somehow changing while playing the game.
- `HideCategories`: Notifies are technically `UObject` classes that we edit _inside_ the animation asset editor. Hiding the object's properties allows us to just focus on the notify's properties. This just makes it easier to work with; it will appear more like an event or node with parameters, rather than a class.
- `CollapseCategories`: This formats all the notify's properties into one category to, again, make it nicer to work with.
- `DisplayName`: This is the name that will appear in the `Add Notify State...` pop-up when adding notifies to your animations.

This isn't just a feature; it's a _tool_ for our animators. So we want to make it as intuitive and streamlined as possible to work with!
{: .notice--info}

Before we start implementing the notify itself, we can add one more feature to improve our workflow:

{% highlight c++ %}
// AnimNotifyState_SpawnAnimActor.h

public:

	// Default constructor.
	UAnimNotifyState_SpawnAnimActor();
{% endhighlight %}

{% highlight c++ %}
// AnimNotifyState_SpawnAnimActor.cpp

UAnimNotifyState_SpawnAnimActor::UAnimNotifyState_SpawnAnimActor()
{
#if WITH_EDITORONLY_DATA
    NotifyColor = FColor(255, 25, 150);
#endif
}
{% endhighlight %}

This changes the color (in RGB) of the notify in the editor so it's easier to distinguish. I chose a nice coral color.

If you compile this, open up an animation sequence or montage, and right-click in the timeline, you should see our new notify!

![New notify in Add Notify State menu]({{ '/' | absolute_url }}/assets/images/per-post/anim-actors/anim-actors-add-notify-menu.png){: .align-center}

### Spawning the Actor

When our notify state starts, we want to spawn our actor, and when it ends, we want to destroy it. Luckily for us, the `AnimNotifyState` class has virtual functions for both of these events. Let's start with the first:

{% highlight c++ %}
public:

    // Spawns the actor defined by this notify.
    virtual void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference) override;
{% endhighlight %}

`AnimNotifyState` has two virtual functions with this name, but one of them is deprecated. The correct one has an `EventReference` parameter.
{: .notice--info}

{% highlight c++ %}
void UAnimNotifyState_SpawnAnimActor::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);
}
{% endhighlight %}

Before we can spawn our actor, we have to know what we actually want to spawn.

For most use-cases, we'll want to spawn some kind of mesh: Bugs Bunny's hammer, Junker Queen's axe, Mario's cape, etc. If we let our notify spawn _any_ actor with a reference to its class, we'd have to create a different actor class for every single mesh we spawn. Not only does that waste time and memory, but it's also really unintuitive to force our animators to create another actor class every time they want to spawn something in an animation.

Instead, our notify will spawn a mesh actor, and its parameters will define which mesh to use for that actor. This provides a significantly better workflow, and it still covers most—if not all—use-cases. We'll also see that restricting our spawned actors to meshes actually gives us _more_ flexibility, because we'll be able to add mesh-specific options, like playing animations and customizing materials.

Now that we know that we're spawning a mesh actor, let's define the parameters we'll use to spawn the mesh:

{% highlight c++ %}
protected:

    // Static or skeletal mesh asset to spawn. Must be a skeletal mesh to use ActorAnimation.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", Meta = (AllowedClasses = "/Script/Engine.SkeletalMesh /Script/Engine.StaticMesh"))
    TObjectPtr<UStreamableRenderAsset> MeshToSpawn;
    
    // Optional socket at which to attach the animation actor. "None" to not attach the spawned actor.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify")
    FName AttachSocket;
    
    // Transform with which to spawn the animation actor, relative to AttachSocket if it's set.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", DisplayName = "Relative Transform")
    FTransform SpawnTransform;
    
    // Whether to override the spawned mesh's materials.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", DisplayName = "Enable Material Overrides")
    uint32 bOverrideMaterials : 1;
    
    /* The materials to use for each of the spawned mesh's material indices. Overrides the mesh's default materials.
     * Indices without a valid material will be skipped, and revert to the mesh's default material for that index. */
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", Meta = (EditCondition = "bOverrideMaterials", EditConditionHides = "true", ScriptName = "MaterialOverrides"))
    TArray<TObjectPtr<UMaterialInterface>> OverrideMaterials;
    
    // The overlay material to apply to the mesh.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", Meta = (EditCondition = "bOverrideMaterials", EditConditionHides = "true"))
    TObjectPtr<UMaterialInterface> OverlayMaterial;
    
    // Optional animation to play on the animation actor when spawned, if it's a skeletal mesh.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", DisplayName = "Actor Animation to Play", Meta = (EditConditionHides = "true"))
    TObjectPtr<UAnimationAsset> ActorAnimation;
    
    // Whether the actor animation should loop.
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AnimNotify", DisplayName = "Loop Actor Animation?", Meta = (EditConditionHides = "true"))
    bool ActorAnimationLoops;
{% endhighlight %}

The `AllowedClasses` specifier on `MeshToSpawn` restricts this property to assets of type `SkeletalMesh` or `StaticMesh`, since their common parent, `StreamableRenderAsset`, has other subclasses.
{: .notice--info}

The `ScriptName` specifier on `OverrideMaterials` is required to differentiate this variable from `bOverrideMaterials`, since their names will appear the same in some places (e.g. any scripts using Python).
{: .notice--info}

Hopefully these parameters are self-explanatory: when our notify begins, we want to spawn a mesh actor at `SpawnTransform`, attach it to `AttachSocket`, set its mesh to `MeshToSpawn`, override any specified materials, and play (or loop) an optional animation on it. If that sounds like a lot, don't worry; these steps are all fairly concise.

First, we want to make sure we actually have a mesh to spawn and a world to spawn it in:

{% highlight c++ %}
void UAnimNotifyState_SpawnAnimActor::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration, const FAnimNotifyEventReference& EventReference)
{
    Super::NotifyBegin(MeshComp, Animation, TotalDuration, EventReference);

    // Retrieve the world to spawn our actor.
    UWorld* World = MeshComp->GetWorld();
    if (!World)
    {
        return;
    }

    // Make sure we have a mesh to spawn and a mesh comp to attach it to.
    if (!MeshToSpawn || !MeshComp)
    {
        if (!World->IsPreviewWorld())
        {
            UE_LOG(LogAnimation, Error, TEXT("Attempted to spawn animation actor in animation (%s), but no mesh to spawn was specified."), *GetNameSafe(Animation));
        }

        return;
    }
}
{% endhighlight %}

We check if we're in a preview world to avoid spamming error messages when we've added the notify to an animation, but haven't selected a mesh yet.
{: .notice--info}