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

Important vocab! A "trace" is a raycast query that traces a line through the world. A "sweep" is a different type of query that moves a geometry object along a line. In other words, a "trace" is a trace with a line and a "sweep" is a trace with any other shape. In Unreal, this might be confusing because in the Kismet library, both of these queries are referred to as "traces." However, under the hood, these shape "traces" (e.g. `UKismetSystemLibrary::SphereTraceSingle`) are actually performing sweeps. So, really, we'll be making a "cone _sweep_" function. I just named this article "cone tracing" because that's probably what you Googled.
{: .notice--info}

Unreal Engine used to use PhysX to drive its physics simulations. In PhysX, these shapes are _primitives_ because they can be represented using pure mathematics, which makes them extremely efficient to compute compared to more complex shapes (which would require you to perform computations on individual triangles).

Now, Unreal Engine uses Chaos, which is a proprietary physics solution. But Chaos still uses these same shapes as primitives for same reason PhysX does.

You'll notice this pattern in most physics engines for the same reason. Havok, for example, supports cubes, capsules, and spheres (and "quads").
{: .notice--info}

There may be times, however, when none of these shapes fit the needs of your project. So let's learn how to make a pretty common one: cones! Cones can be a really useful shape when programming gameplay: you can use them for melee attacks, flamethrowers, magical spells, or whatever other conical machinations you may have.

## Solution

To perform a sweep in the shape of a cone (without simply sweeping a cone-shaped mesh, which would be incredibly inefficient), **we'll perform a sphere sweep encompassing an "imaginary" cone, and then use some math** (yay) **to get rid of any hits that would be outside of that cone.**

Visually, our encompassing sphere sweep will look like this.

![Sphere Sweep Visual 1]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-sweep-01.png){: .align-center}

You can see how this initial sweep will essentially give us the same results we would get from performing an actual cone-shaped sweep. The only difference is that, since the sphere sweep is bigger than the cone, we'll end up with a few extraneous results that we'll get rid of after.

Here's a clearer look at what our sweep actually looks like, since, in practice, the distance between each sphere is incredibly small:

![Sphere Sweep Visual 2]({{ '/' | absolute_url }}/assets/images/per-post/cone-trace/cone-trace-sweep-02.png){: .align-center}