---
layout: single
title: "Procedural First-Person Animation System"
excerpt: A breakdown of the first-person animation framework used in Cloud Crashers, and guide to building a similar system.
header:
  teaser: /assets/images/per-post/atomicity/thumb.jpg
author: Meta
---

A breakdown of the first-person animation framework used in _Cloud Crashers_, and guide to building a similar system.

## Introduction

**TODO: Teaser**

[_Cloud Crashers_](https://store.steampowered.com/app/2995940/Cloud_Crashers/) is a hero-based fighting game. Each playable character has a unique weapon, set of abilities, and overall aesthetic that feels distinct.

When designing the game's first-person animation system, we needed a robust framework that could streamline building large numbers of complex animation sets. But we also wanted a way to make each character feel unique, with their own sense of personality.

As I was researching solutions for animation systems, I came across this brilliant GDC talk by Blizzard Entertainment's Matt Boehm:

<iframe width="560" height="315" src="https://www.youtube.com/embed/7t0hLZd_8Z4?si=M6_tnPrCOSfHf0jU&amp;start=1192" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
<br>
In this presentation, Matt breaks down how _Overwatch_ uses animation layers, additives, and spring models—among other tricks—to convey each hero's unique personality through procedural first-person animations.

Even though it isn't a technical talk, Matt's high-level explanation of _Overwatch's_ animation framework provided great insights and inspiration for building a similar system for _Cloud Crashers_ in Unreal Engine.

In this article, I'll show how to implement a flexible first-person animation system from scratch. By the end, we'll have an extremely powerful animation blueprint which can be used to create robust animation sets like this:

**TODO: Final result**

If you want to skip over the tutorial and just steal the code (you're completely welcome to), check out the [CharacterAnimInstanceBase](https://github.com/ChangeStudios/ProjectCrash/blob/release/Source/ProjectCrash/Animation/CharacterAnimInstanceBase.h) and [FirstPersonCharacterAnimInstance](https://github.com/ChangeStudios/ProjectCrash/blob/release/Source/ProjectCrash/Animation/FirstPersonCharacterAnimInstance.h) classes on _Cloud Crashers'_ public source code.
<br>
<br>
As you can see, _Cloud Crashers_ actually uses two animation instance classes: a base class and a first-person subclass. This is because _Cloud Crashers_ also supports third-person, and the third-person class re-uses the code in the base animation instance class. For the sake of simplicity, in this tutorial, I've rewritten the base class and first-person class into a single class.
{: .notice--info}

## Base Pose

### Creating an Animation Instance Class

Let's start by creating a new C++ class called `FirstPersonCharacterAnimInstance`. This will be a subclass of Unreal's `AnimInstance` class, and will serve as the base class for our animation blueprint.

We'll be collecting a lot of data, and performing a lot of calculations for our character. Doing this in a C++ class will be a lot easier, and will help keep our animation blueprint clean. We'll cache all of our data so our animators (us) have access to it in the animation graph.

In our constructor, let's enable [multithreading](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-optimization-in-unreal-engine) to avoid bottlenecks when animating multiple characters.

{% highlight c++ %}
// FirstPersonCharacterAnimInstance.h

UClass(Abstract)
class PROJECTCRASH_API UFirstPersonCharacterAnimInstance : public UAnimInstance
{
    GENERATED_BODY()

public:

    UFirstPersonCharacterAnimInstance();
}
{% endhighlight %}

{% highlight c++ %}
// FirstPersonCharacterAnimInstance.cpp

UFirstPersonCharacterAnimInstance::UFirstPersonCharacterAnimInstance()
{
    bUseMultiThreadedAnimationUpdate = true;
}
{% endhighlight %}

Remember to replace `PROJECTCRASH_API` with your game's API name. Unreal does this automatically if you use the "New C++ Class..." option.
{: .notice--info}

### Locomotion Blend Space

Like Matt says, we need to start with a base pose for our character. Then, we'll play our animations on top of that (we'll use Unreal Engine's [animation slots](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-slots-in-unreal-engine) instead of [Maya Layers](https://help.autodesk.com/view/MAYAUL/2024/ENU/?guid=GUID-5C202CB8-EB3C-4ADE-B203-5F93A9FD9104)), and apply additive poses on top of the resulting animation.

To get the base pose, we're going to use a simple locomotion blend space. First-person locomotion animations are significantly less complex than third-person: we don't need to account for turns or bother with state machines; we just need to blend between an `Idle` animation and a `Walking` animation.

Let's create a new animation blueprint based on our animation instance class, and start by playing a blend space. Since we want to re-use this animation blueprint with each character, we'll create a new `Idle/Walk BS` blend space variable and bind it to the player.

![Blend space player without inputs]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-bs-01.png){: .align-center}

This is how we'll define all of our animation assets. This way, to make each character's animation blueprint, all we have to do is subclass this animation blueprint and set each variable to use that character's unique animation assets and settings.
{: .notice--info}

### Calculating Velocity Data

To blend our animations, we need our character's speed. Specifically, because we're using a 2D blend space that can define animations for walking in each direction, we want to know how fast the character is moving forwards or backwards, and right or left.

_Cloud Crashers_ uses the same walking animation regardless of the direction the character is moving, but we have the option to use directional animations. Either way, we'll need these variables to calculate our additives later.
{: .notice--info}

To get these values, we want to calculate the character's velocity along their local x-axis (forward/backward) and local y-axis (right/left).

In our animation instance, let's define some variables in our header file:

{% highlight c++ %}
protected:

	// This character's current velocity, relative to its world rotation.
	UPROPERTY(BlueprintReadOnly, Category = "Velocity Data")
	FVector LocalVelocity;

	// This character's current local velocity with vertical velocity (Z) masked out.
	UPROPERTY(BlueprintReadOnly, Category = "Velocity Data")
	FVector LocalVelocity2D;

	/* This character's current local velocity, normalized to its maximum movement speed. Vertical velocity (Z) is
	 * masked out. */
	UPROPERTY(BlueprintReadOnly, Category = "Velocity Data")
	FVector LocalVelocity2DNormalized;
{% endhighlight %}

We want to update these variables (and most of the variables we'll use) every frame. To do this with multithreading, we implement the `NativeThreadSafeUpdateAnimation`:
{% highlight c++ %}
public:

	virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds) override;
{% endhighlight %}

Now, in our implementation (.cpp) file, we'll start with a couple checks to make sure we have what we need to calculate these variables. To normalize our velocity to our maximum speed, we need our [character movement component](https://dev.epicgames.com/documentation/en-us/unreal-engine/movement-components-in-unreal-engine#charactermovementcomponent). So let's make sure we have one:

{% highlight c++ %}
void UFirstPersonCharacterAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    Super::NativeThreadSafeUpdateAnimation(DeltaSeconds);

	APawn* PawnOwner = TryGetPawnOwner();
	if (!PawnOwner)
	{
		return;
	}

	UCharacterMovementComponent* CharMovementComp = Cast<UCharacterMovementComponent>(PawnOwner->GetMovementComponent());
	if (!CharMovementComp || (CharMovementComp->MovementMode == MOVE_None))
	{
		return;
	}

	// ...
}
{% endhighlight %}

We'll be calculating a lot of variables; I don't want to put them all into `NativeThreadSafeUpdateAnimation`. Instead, we'll separate them into different functions. To update our velocity variables, let's create a new function called `UpdateVelocityData`. Inside, we'll calculate our character's local velocity, normalize with their maximum movement speed:

{% highlight c++ %}
protected:

	// Calculate velocity data this frame.
	void UpdateVelocityData();
{% endhighlight %}

{% highlight c++ %}
void UFirstPersonCharacterAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    // ... 

    UpdateVelocityData();
}
{% endhighlight %}

{% highlight c++ %}
void UFirstPersonCharacterAnimInstance::UpdateVelocityData()
{
    APawn* PawnOwner = TryGetPawnOwner();
    UCharacterMovementComponent* CharMovementComp = Cast<UCharacterMovementComponent>(PawnOwner->GetMovementComponent());

    const FVector WorldVelocity = PawnOwner->GetVelocity();
    const FRotator WorldRotation = PawnOwner->GetActorRotation();
    
    // The character's "local" velocity is their world velocity relative to their world rotation.
    LocalVelocity = WorldRotation.UnrotateVector(WorldVelocity);
    LocalVelocity2D = LocalVelocity * FVector(1.0f, 1.0f, 0.0f);
    
    // Normalize the character's local velocity to their maximum movement speed.
    const float MaxMovementSpeed = CharMovementComp->GetMaxSpeed();
    const float NormalizedX = FMath::Clamp(UKismetMathLibrary::NormalizeToRange(LocalVelocity2D.X, 0.0f, MaxMovementSpeed), -1.0f, 1.0f);
    const float NormalizedY = FMath::Clamp(UKismetMathLibrary::NormalizeToRange(LocalVelocity2D.Y, 0.0f, MaxMovementSpeed), -1.0f, 1.0f);
    LocalVelocity2DNormalized = FVector(NormalizedX, NormalizedY, 0.0f);
}
{% endhighlight %}

Back in our animation blueprint, we can bind our local, normalized velocity to our blend space player (I've given the blend space asset a default value here, so we can see a preview):

![Blend space player final]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-bs-02.png){: .align-center}

Now, we have a base locomotive pose based on our directional movement speed:

<iframe width="560" height="315" src="https://www.youtube.com/embed/_DeKiJuUris?si=LAouhGJdwWhosxQ8?autoplay=1&loop=1&mute=1&controls=0&showinfo=0&rel=0&modestbranding=1" frameborder="0" allow="autoplay" allowfullscreen></iframe>
<br>

## Additives

Now things get more interesting. To add that extra level of personality to our animations, we're going to use additive poses to offset the base pose. We want to apply four different additives:

- **Movement Sway**: Make the character lean in the direction they're moving or lag behind.
- **Aim Sway**: Make the character's weapon lead the player's aim or lag behind when they turn.
- **Falling Offset**: Blend to a `Jumping` or `Landing` pose based on the character's vertical velocity. This results in a procedural "Jump" animation that is more flexible and, more importantly, looks much nicer than one using a state machine (i.e. `Grounded` -> `Jumping` -> `Apex` -> `Falling` -> `Landing` -> `Grounded`).
- **Pitch Offset**: Adjust the character's pose based on whether they're aiming up or down.