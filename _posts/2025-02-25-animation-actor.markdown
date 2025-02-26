---
layout: single
title: "Animation Actors"
excerpt: Learn to create an animation notify that lets you add actors to your animations!
header:
  teaser: /assets/images/per-post/
author: Meta
---

Learn to create an animation notify that lets you add actors to your animations!

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

As of UE 5.5, Unreal doesn't have a built-in notify for spawning actors, so I'll be showing you how to create one. This notify will be able to spawn actors within an animation (we'll be spawning mesh actors, but you could modify this notify to allow any actor to be spawned). It will support both static and skeletal meshes, allow you to play animations on spawned skeletal mesh actors, and allow you to override the mesh's materials.

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
// AnimNotifyState_SpawnAnimActor.h

UCLASS(Const, HideCategories = Object, CollapseCategories, DisplayName = "Spawn Animation Actor")
{% endhighlight %}

- `Const`: Prevents our notify from changing during runtime. We can still edit our notifies in the editor, but we wouldn't want them somehow changing while playing the game.
- `HideCategories`: Notifies are technically `UObject` classes that we edit _inside_ the animation asset editor. Hiding the object's properties allows us to just focus on the notify's properties. This just makes it easier to work with; it will appear more like an event or node with parameters, rather than a class.
- `CollapseCategories`: This formats all the notify's properties into one category to, again, make it nicer to work with.
- `DisplayName`: This is the name that will appear in the `Add Notify State...` pop-up when adding notifies to your animations.

This isn't just a feature; it's a _tool_ for our animators. So we want to make it as intuitive and streamlined as possible to work with!
{: .notice--info}

Before we start implementing the notify itself, we can add one more feature to improve this our workflow:

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
