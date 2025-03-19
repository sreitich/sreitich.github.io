---
layout: single
title: "Cone-Shaped Tracing"
excerpt: Learn to perform cone traces in Unreal Engine!
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-sweep-01.png
author: Meta
last_modified_at: 2025-03-15
---

Learn to perform cone traces in Unreal Engine!

## Introduction

![Teaser]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/#TODO){: .align-center}

If you've used Unreal Engine before, you may know that there are only four different trace shapes built into the engine: **line**, **box**, **sphere**, and **capsule**.

This is because traces, sweeps, and overlaps use a structure called `FCollisionShape`, which can only represent these four shapes. And the reason why simply has to do with performance.

Important vocab! A "trace" is a raycast query that traces a line through the world. A "sweep" is a different type of query that moves a geometry object along a line. In other words, a "trace" is a trace with a line and a "sweep" is a trace with any other shape. In Unreal, this might be confusing because in the Kismet library, both of these queries are referred to as "traces." However, under the hood, these shape "traces" (e.g. `UKismetSystemLibrary::SphereTraceSingle`) are actually performing sweeps. So, really, we'll be making a "cone _sweep_" function, but we'll still call it a trace to align with Unreal's conventions.
{: .notice--info}

Unreal Engine used to use PhysX to drive its physics simulations. In PhysX, these shapes are _primitives_ because they can be represented using pure mathematics, which makes them extremely efficient to compute compared to more complex shapes (which would require you to perform computations on individual triangles).

Now, Unreal Engine uses Chaos, which is a proprietary physics solution. But Chaos still uses these same shapes as primitives for same reason PhysX does.

You'll notice this pattern in most physics engines for the same reason. Havok, for example, supports cubes, capsules, and spheres (and "quads").
{: .notice--info}

There may be times, however, when none of these shapes fit the needs of your project. So let's learn how to make a pretty common one: cones! Cones can be a really useful shape when programming gameplay: you can use them for melee attacks, flamethrowers, magical spells, or whatever other conical machinations you may have.

## Solution

### Sphere Sweep

To perform a sweep in the shape of a cone (without simply sweeping a cone-shaped mesh, which would be incredibly inefficient), **we'll perform a sphere sweep encompassing an "imaginary" cone, and then use some math** (yay) **to get rid of any hits that would be outside of that cone.**

Visually, our encompassing sphere sweep will look like this.

![Sphere Sweep Visual 1]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-sweep-01.png){: .align-center}

You can see how this initial sweep will essentially give us the same results we would get from performing an actual cone-shaped sweep. The only difference is that, since the sphere sweep is bigger than the cone, we'll end up with a few extraneous results that we'll get rid of after.

Here's a clearer look at what our sweep actually looks like, since, in practice, the distance between each sphere is incredibly small:

![Sphere Sweep Visual 2]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-sweep-02.png){: .align-center}

Let's start with a signature for our function. I usually put this in a blueprint function library named something like `UCollisionLibrary`:

{% highlight c++ %}
UFUNCTION(BlueprintCallable, Category = "Collision", Meta = (WorldContext = "WorldContextObject", AutoCreateRefTerm = "ActorsToIgnore", DisplayName = "Multi Cone Trace By Channel", AdvancedDisplay = "TraceColor, TraceHitColor, DrawTime", Keywords = "sweep"))
static bool ConeTraceMulti(const UObject* WorldContextObject, const FVector Start, const FRotator Direction, float ConeHeight, float ConeHalfAngle, ETraceTypeQuery TraceChannel, bool bTraceComplex, const TArray<AActor*>& ActorsToIgnore, EDrawDebugTrace::Type DrawDebugType, TArray<FHitResult>& OutHits, FLinearColor TraceColor = FLinearColor::Red, FLinearColor TraceHitColor = FLinearColor::Green, float DrawTime = 5.0f);
{% endhighlight %}

This function performs a multi-sweep using a collision channel. You could easily refactor this into a helper function to re-use it for a `ConeTraceSingle` function, and additional functions for `TraceByProfile` and `TraceForObjects`, like Unreal does for all of its built-in trace shapes.
{: .notice--info}

The important parameters here are `ConeHeight` and `ConeHalfAngle`. I'm choosing to define the cone by these parameters because this is what designers usually care about: the range of the trace (the cone's height) and the maximum angle of the trace (which is really the cone's _half_-angle).

With these parameters, we can calculate the length and the radius of a sphere sweep that encompasses our cone:

{% highlight c++ %}
bool UCollisionLibrary::ConeTraceMulti(
    const UObject* WorldContextObject,
    const FVector Start,
    const FRotator Direction,
    float ConeHeight,
    float ConeHalfAngle,
    ETraceTypeQuery TraceChannel,
    bool bTraceComplex,
    const TArray<AActor*>& ActorsToIgnore,
    EDrawDebugTrace::Type DrawDebugType,
    TArray<FHitResult>& OutHits,
    FLinearColor TraceColor,
    FLinearColor TraceHitColor,
    float DrawTime)
{
    OutHits.Reset();
    
    ECollisionChannel CollisionChannel = UEngineTypes::ConvertToCollisionChannel(TraceChannel);
    FCollisionQueryParams Params(SCENE_QUERY_STAT(ConeTraceMulti), bTraceComplex);
    Params.bReturnPhysicalMaterial = true;
    Params.AddIgnoredActors(ActorsToIgnore);
    
    UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
    if (!World)
    {
        return false;
    }
   
    TArray<FHitResult> TempHitResults;
    const FVector End = Start + (Direction.Vector() * ConeHeight);
    const double ConeHalfAngleRad = FMath::DegreesToRadians(ConeHalfAngle);
    // r = h * tan(theta / 2)
    const double ConeBaseRadius = ConeHeight * tan(ConeHalfAngleRad);
    const FCollisionShape SphereSweep = FCollisionShape::MakeSphere(ConeBaseRadius);

    // Perform a sweep encompassing an imaginary cone.
    World->SweepMultiByChannel(TempHitResults, Start, End, Direction.Quaternion(), CollisionChannel, SphereSweep, Params);
}
{% endhighlight %}

The distance of the sweep should be the height of the cone, which is given. The starting location and direction are given, so we can multiply that distance by the trace's direction to get the point where the trace should end.

The radius of the sweep should be the radius of the _base_ of the cone, so it can fully encompass the entire shape, but we didn't make this a parameter.

A cone's height, angle, and radius are all related, so it only takes two of these values to define it. I chose to use the height and angle because it's more intuitive to define a cone by these attributes, rather than its radius. However, because these values are tied together, we can calculate the radius we want using the height and angle of the cone:

![Radius formula]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-radius-formula-high-def.png){: .align-center}

$${r} = {h} \cdot \tan({θ})$$

### Filtering

Now that we've generated a collection of hits from this sphere sweep, all we have to do is filter out extraneous results—that is, any results that were inside our sphere sweep, but would be _outside_ our imaginary cone.

There are two situations where this can occur...

1. A hit is outside the angle of the cone: 
    ![Space outside angle]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-outside-angle.png){: .align-center}
2. A hit is beyond the cone, within the ending cap of the sphere sweep:
    ![Space beyond cone]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-beyond-end.png){: .align-center}

Fortunately, we can do both of these pretty easily with a few calculations:

{% highlight c++ %}
    // ...

    // Filter for hits that would be inside the cone.
    for (FHitResult& HitResult : TempHitResults)
    {
        const FVector HitDirection = (HitResult.ImpactPoint - Start).GetSafeNormal();
        const double Dot = FVector::DotProduct(Direction.Vector(), HitDirection);
        // theta = arccos((A • B) / (|A|*|B|)). |A|*|B| = 1 because A and B are unit vectors.
        const double DeltaAngle = FMath::Acos(Dot);

        // Hit is outside the angle of the cone.
        if (DeltaAngle > ConeHalfAngleRad)
        {
            continue;
        }

        const double Distance = (HitResult.ImpactPoint - Start).Length();
        // Hypotenuse = adjacent / cos(theta)
        const double LengthAtAngle = ConeHeight / cos(DeltaAngle);

        // Hit is beyond the cone. This can happen because we sweep with spheres, which results in a cap at the end of the sweep.
        if (Distance > LengthAtAngle)
        {
            continue;
        }

        OutHits.Add(HitResult);
    }
}
{% endhighlight %}

To determine whether a hit is outside the angle of our cone, we can calculate the angle of the hit, and check if it's greater than our cone's angle. We can do that with the following formula:

$${θ} = \arccos{(\frac{ {A} \dotproduct {B} }{\abs{A} \cdot \abs{B}})}$$

It can be hard to tell in LaTeX: the top equation is a dot product; the bottom is a multiplication.
{: .notice--info}

In this equation, $${A}$$ and $${B}$$ are the 