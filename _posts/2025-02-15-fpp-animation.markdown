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

### Locomotion Blend Space and State Machine

Like Matt says, we need to start with a base pose for our character. Then, we'll play our animations on top of that (we'll use Unreal Engine's [animation slots](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-slots-in-unreal-engine) instead of [Maya Layers](https://help.autodesk.com/view/MAYAUL/2024/ENU/?guid=GUID-5C202CB8-EB3C-4ADE-B203-5F93A9FD9104)), and apply additive poses on top of the resulting animation.

To get the base pose, we're going to use a simple locomotion blend space. First-person locomotion animations are significantly less complex than third-person: we don't need to account for turns or bother with state machines; we just need to blend between an `Idle` animation and a `Walking` animation.

Let's create a new animation blueprint based on our animation instance class, and start by playing a blend space. Since we want to re-use this animation blueprint with each character, we'll create a new `Idle/Walk BS` blend space variable and bind it to the player.

![Blend space player without inputs]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-bs-01.png){: .align-center}

This is how we'll define all of our animation assets. This way, to make each character's animation blueprint, all we have to do is subclass this animation blueprint and set each variable to use that character's unique animation assets and settings.
{: .notice--info}

This blend space will work perfectly for our grounded movement, but we also need to account for when we jump or fall. So, before we go any further, let's create a new state machine that will switch between our grounded and airborne locomotion:

![Locomotion state machine]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-sm-01.png){: .align-center}

Inside, we only need two states: one for when we're on the ground and one for when we're in the air. For the grounded state, we'll use the blend space we just created (you can just copy/paste it).

For our airborne state, we'll simply loop a new `Falling` animation sequence, which we can bind to a new variable (in _Cloud Crashers_, we just re-use the idle animation):

![Locomotion state machine states]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-sm-02.png){: .align-center}

![Airborne state]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-sm-03.png){: .align-center}

State machines for jumping are usually extremely complex, in order to account for each state of the jump (`Jump`, `Falling Up`, `Apex`, etc.). But we're actually going to create our jump animations procedurally with our additives later, so we just need this one state.
{: .notice--info}

Next, we'll calculate the parameters we need to drive our blend space and transition between our locomotion states.

### Calculating Velocity Data

To blend the animations in our blend space, we need our character's speed. Specifically, because we're using a 2D blend space that can define animations for walking in each direction, we want to know how fast the character is moving forwards or backwards, and right or left.

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

We want to update these variables (and most of the variables we'll use) every frame. To do this with multithreading, we implement the `NativeThreadSafeUpdateAnimation` function:
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

Back in our animation blueprint, we can bind our local, normalized velocity to our blend space player:

![Blend space player final]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-locomotion-bs-02.png){: .align-center}

Now, we have a base locomotive pose based on our directional movement speed. But we still need to transition between our `Grounded` and `Airborne` states. If we don't, we'll keep running when we jump or fall, which isn't what we want.

To transition between states, we can simply check the character's current movement mode inside the transition rules:

![Grounded to airborne transition rule]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-sm-rule-01.png){: .align-center}

![Airborne to grounded transition rule]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-sm-rule-02.png){: .align-center}

Remember to use the `Property Access` node to keep our blueprint thread-safe!
{: .notice--info}

Finally, we have our base pose, based on our directional movement speed _and_ our current movement state:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/fpp-animation/fpp-anim-locomotion-final-vid.mp4" type="video/mp4">
    Video tag not supported.
</video>

## Additives

Now things get more interesting. To add that extra level of personality to our animations, we're going to use additive poses to offset the base pose. We want to apply three different additives:

- **Movement Sway**: Make the character lean toward or lag behind the direction they're moving.
- **Aim Sway**: Make the character's weapon lead or lag behind the player's aim when they turn.
- **Falling Offset**: Blend to a `Jumping` or `Landing` pose based on the character's vertical velocity. This creates a procedural "Jump" animation that is more flexible and, more importantly, looks much nicer than one that uses a state machine (i.e. `Grounded` -> `Jump` -> `Falling Up` -> `Apex` -> `Falling Down` -> `Landing` -> `Grounded`).

You might hear some people say "rolls," "aim roll," or "turning sway" instead of "aim sway."
{: .notice--info}

Just so it's clear what we're trying to achieve, here's an example of what these poses may look like. This is the set of additive poses for the **Knight** character:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/fpp-animation/fpp-anim-additive-poses-vid.mp4" type="video/mp4">
    Video tag not supported.
</video>

If you watched the GDC talk linked at the beginning of this page, this is what Matt referred to as the "aim suite."
{: .notice--info}

Here's what these animation assets actually look like in _Cloud Crashers_. Notice how they're simply an animation sequence that's one frame-long (yours don't need to be exactly one frame, but we'll only ever use one frame of the animation):

![Additive animation asset]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-additive-settings-01.png){: .align-center}

Applying these sways and offsets as additive animations allows us to apply them on top of whatever the character animation is currently playing, similar to an [aim offset](https://dev.epicgames.com/documentation/en-us/unreal-engine/aim-offset-in-unreal-engine). Whether we're idling, walking, reloading, or doing anything else, our sways and offsets will still be applied.

For a short explanation of how additive animations actually work, check out the documentation on aim offsets, linked above.
{: .notice--info}

### Applying Additives

To apply these additives, we'll take our base pose and layer them on top. Since we want to apply _different_ additives depending on the direction of the driving variable (e.g. `Fall Up` with a positive velocity vs. `Fall Down` with a negative velocity), we can use more blend spaces to determine which additives to play.

Now, we _could_ create another blend space and use an `Apply Additive` node to apply each one. But a better solution would actually be to use an aforementioned aim offset, because an aim offset is essentially an additive blend space: aim offsets evaluate a blend space and additively apply the result on top of a base pose, which is exactly what we want to do.

Instead of creating an aim offset asset, I'm actually going to create the aim offsets _inside_ the animation blueprint with the `Aim Offset Blend Space` node. This way, we don't need to create an aim offset asset for every additive set, for every character; we can just change the additive animation assets in each character's animation blueprint.

![Additive aim offset nodes in animation graph]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-aim-offsets-no-params-01.png){: .align-center}

Our `Alpha` should always be 1.0, so I've unchecked `Expose as Pin` from the `Alpha` binding. I've also left the horizontal axis of the `Falling Offset` aim offset as "None" and unchecked `Expose as Pin` on its binding to hide it, since we only need one axis for this offset.
{: .notice--info}

Inside, each aim offset has samples evaluating a bound animation sequence at the extrema of each axis. Since we're treating our additive animations as poses, we can skip the overhead of actually playing them, and instead just evaluate the pose at their first (and only) frame (specified by the `Explicit Time` parameter). On the left, you'll also see the new variables we've created to bind our animation assets.

![Aim offset graph]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-additive-aim-offsets-no-params-01.png){: .align-center}

![Aim offset sample]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fppanim-additive-aim-offsets-no-params-02.png){: .align-center}

For the samples at the center of our graph (`(0, 0)`, where we don't want to apply any additives), we're evaluating whatever pose our additives are defined relative to. This ensures that no additives are applied when the additive value is `0.0`. Otherwise, our aim offsets will try to create a base pose by averaging each additive, instead of just leaving the underlying animation alone when we don't want any additives applied to it.

For _Cloud Crashers_, our additives are defined relative to the first frame of the `Idle` animation, which we export as an asset called `Aim Forward` for convenience. You can see this in the image of the additive animation asset above.
{: .notice--info}

### Calculating Additives

Here's where things get tricky: our blend spaces need parameters to determine how to apply each additive. Let's consider what values we want to bind to each additive type:

- **Movement Sway**: Horizontal velocity (how fast we're moving forwards/backwards and right/left)
- **Aim Sway**: Rotational velocity (how fast we're turning up/down and right/left)
- **Falling Offset**: Vertical velocity (how fast we're jumping up/falling down)

Logically, if we normalize these values and bind them to our blend spaces, like we did with our locomotion, we should get what we're looking for. So let's see what happens when we try this:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/fpp-animation/fpp-anim-interped-additives-no-bs-smoothing-vid.mp4" type="video/mp4">
    Video tag not supported.
</video>

Well, that looks... odd. If you looked closely at the `Blend Space` settings in our aim offsets, you might realize that this is because we aren't smoothing between our additives.

Characters in _Cloud Crashers_ have an acceleration speed of `16384.0 cm/s`, so whenever we start moving in one direction, we reach our maximum velocity very quickly, and when we stop, we return to being idle very quickly. The same issue occurs with our other additives when we turn, jump, or falls.

By adding `Smoothing Time` to our aim offsets, we'll blend between additives more slowly, creating a smoother transition. Let's try using the `Ease In/Out` smoothing type:

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/fpp-animation/fpp-anim-interped-additives-with-bs-smoothing-vid.mp4" type="video/mp4">
    Video tag not supported.
</video>

Okay, some of those look a _little_ better. Maybe one of the other smoothing types will look better?

I'll save you the time: they don't. So what's wrong?

Let's take a second to think about what effect we actually want to achieve.

We want to realistically simulate how our body organically reacts to movement. Our current method is linearly interpolating between different poses depending on the direction we're moving. We're essentially just "snapping" between different poses depending on the direction we're moving, turning, or falling. Mathematically, blending poses like that (like we did in the first video above) looks like this:

![Linear interpolation graph]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fpp-anim-linear-interpolation-graph-01.png){: .align-center}

The problem is that our bodies don't move that mechanically. When our muscles move, they don't "snap" into place. They take time to start moving, might overshoot their destination, and take time to stop and settle into place. Visually, our muscles move between positions more like this:

![Spring interpolation graph]({{ '/' | absolute_url }}/assets/images/per-post/fpp-animation/fpp-anim-spring-interpolation-graph-01.png){: .align-center}

Well, fortunately for us, there's a mathematical model that does exactly this, and it's what a lot of first-person shooter games use to create natural-looking sways: **springs**.

## Springs

Springs (or, more technically, "[_oscillating systems_](https://en.wikipedia.org/wiki/Oscillation)") provide a perfect way to simulate how our bodies move because, from a visual perspective, they move very similarly. Springs have tension, so they take time to start and stop moving, and their bounciness causes them to oscillate back and forth before settling back into place.

The graph above is a simple equation called **[damped oscillation](https://www.geeksforgeeks.org/damped-oscillation-definition-equation-types-examples/)**. We'll be using a more robust model that's already built into Unreal Engine.
{: .notice--info}

So, how can we leverage spring models to apply additives more naturally?

The current magnitude of each of our additives (i.e. how heavily they're applied) will be determined by a scalar variable called the `Current Value`. Each frame, we'll calculate a `Target Value` using the data from that frame. For example, if we're moving forward very fast, our target value for our forward/backward movement sway will be a large positive number (positive for forward, negative for backward). But if we suddenly stop moving, the target value will be `0`.

Next, we'll plug our `Current Value` and `Target Value` into a spring model. Our spring model will give us a new `Current Value` by stepping towards the `Target Value`, depending on how much time passed this frame. Finally, we update `Current Value`, and repeat this process the next frame, and so on.

By continuously blending towards whichever pose is desired by our additives' dependent values (horizontal, rotational, and vertical velocity, respectively), this method not only achieves more natural-looking blending, but _also_ fixes our smoothing issue. Linear interpolation isn't great at handling sharp changes (like quickly turning back and forth in the video above), but springs are _great_ at it. This is because of how [springs _damp_ oscillations](https://en.wikipedia.org/wiki/Damping): they're able to handle these dramatic changes, and can smoothly interpolate between rapidly changing targets without breaking.

If that's confusing, skip ahead to our final results, and compare how quickly turning right and left looks compared to the videos above. This will more clearly demonstrate the effects of spring damping.
{: .notice--info}

### Calculating Aim Data

We'll be using the same method of calculation for all three of our additives. We already have the data we need for our movement sway and falling offset (which we collected in our `UpdateVelocityData` function). But we still need to calculate some data for our aim sway.

Our aim sway is determined by how quickly our character is turning right or left and up or down. Since this is a first-person game, our character's rotation is determined by our camera. So all we need to do is calculate how much our camera rotates each frame along each axis: yaw (right/left) and pitch (up/down).

We'll actually use our pawn's `BaseAimRotation`, which gives us the controller's aim rotation, instead of wasting time trying to find the player's camera.
{: .notice--info}

Let's create a new function to calculate this data, with a float parameter called `DeltaSeconds`. This will be given by `NativeThreadSafeUpdateAnimation`, and it tells us how much time has passed this frame (e.g. at 60 frames/second: `1.0 seconds ÷ 60.0 frames ≈ 0.0167 seconds/frame`). We didn't need this for our movement data because actors already track their velocity, but we'll need to calculate our camera's rotation speed ourselves.

{% highlight c++ %}
protected:
    
    // Calculate aim data this frame.
    void UpdateAimData(float DeltaSeconds);
{% endhighlight %}

{% highlight c++ %}
void UFirstPersonCharacterAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    // ...
    
    UpdateAimData(DeltaSeconds);
}
{% endhighlight %}

Let's also add the variables we'll be calculating:

{% highlight c++ %}
protected:

    // This character's current base aim rotation.
    UPROPERTY(BlueprintReadOnly, Category = "Aim Data")
    FRotator AimRotation;
    
    // The normalized rate at which the owning character's aim yaw is changing.
    UPROPERTY(BlueprintReadOnly, Category = "Aim Data", DisplayName = "Aim Speed (Right/Left)")
    float AimSpeedRightLeft;
    
    // The normalized rate at which the owning character's aim pitch is changing.
    UPROPERTY(BlueprintReadOnly, Category = "Aim Data", DisplayName = "Aim Speed (Up/Down)")
    float AimSpeedUpDown;
{% endhighlight %}

We want to calculate our rotation speed in `Degrees/Second`. We can do this with the following formula:

$$\frac{Degrees}{Second} = \frac{Degrees}{Frame} \cdot \frac{Frames}{Second}$$

`Degrees/Frame` is the amount we've rotated this frame, and we can calculate `Frames/Second` by taking the inverse of `DeltaSeconds` (since `DeltaSeconds` represents `Seconds/Frame`):

{% highlight c++ %}
void UFirstPersonCharacterAnimInstance::UpdateAimData(float DeltaSeconds)
{
    const FRotator PreviousAimRotation = AimRotation;
    
    AimRotation = TryGetPawnOwner()->GetBaseAimRotation();
    AimRotation.Pitch = FRotator::NormalizeAxis(AimRotation.Pitch); // Fix for a problem with how UE replicates aim rotation.
    
    // Use a normalized delta to account for winding (e.g. 359.0 -> 1.0 should be 2.0, not -358.0).
    const FRotator RotationDelta = UKismetMathLibrary::NormalizedDeltaRotator(AimRotation, PreviousAimRotation);
    
    const float InverseDeltaSeconds = ((DeltaSeconds > 0.0f) ? (1.0f / DeltaSeconds) : 0.0f); // Avoid dividing by 0.
    
    AimSpeedRightLeft = RotationDelta.Yaw * InverseDeltaSeconds;
    AimSpeedUpDown = RotationDelta.Pitch * InverseDeltaSeconds;
}
{% endhighlight %}

In _Cloud Crashers_, we skip the aim speed calculation the first frame. If our character is spawned, for example, with a rotation of `(0, 0, 180)`, our initial aim speed will be 180 degrees/second, because `AimRotation` is initialized to `(0, 0, 0)`. On the first frame, we just update `AimRotation`, to properly initialize it, and leave our aim speeds at `0.0` to avoid this.
{: .notice--info}

