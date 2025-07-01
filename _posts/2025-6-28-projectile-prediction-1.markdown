---
layout: single
title: "Projectile Prediction: Part 1"
excerpt: An exploration of the theory behind projectile prediction.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-07-01
---

Part 1 of my series exploring and implementing projectile prediction for multiplayer games. This part breaks down the theory behind projectile prediction, some approaches to implementing it, and a short overview of the version _we'll_ create in (starting in part 2) using Unreal Engine.

This tutorial uses Unreal Engine for its code, but the theory and techniques are applicable for any game engine.
{: .notice--info}

If you just want the final code, it can be found on [Unreal Engine's Learning site](...). However, this series is more focused on the theory behind projectile prediction, and the different approaches that can be taken to implement it, since, rather than just the code itself, since the approach you choose should be based on _your_ project's specific needs. 
{: .notice--info}

## Introduction

Client-side prediction is a crucial component of making real-time online games feel responsive. It's commonly used for things like character movement, abilities, and visuals to conceal the effects of latency, and provide a more fair experience for players with volatile network conditions.

I'm assuming you know how client-side prediction works if you're reading this. If not, [this video provides a good overview of the topic.](https://www.youtube.com/watch?v=2Xl0oaTKBXo)
{: .notice--info}

One core feature of client-side prediction, present in most modern multiplayer games, is _projectile prediction_ (despite the fact that, for some reason, it's _really_ hard to find good resources on this topic, hence why I'm writing this).

Projectile prediction is the client-side prediction performed when a client fires a projectile (a rocket launcher, a grenade, etc.). When the player presses the "_fire_" input, we want to instantly spawn and simulate the projectile for them, to keep the game feeling responsive.

For disambiguation, the term "projectile prediction" can also refer to the indicators that appear when players are preparing to throw or shoot something, showing them the trajectory in which their projectile will travel. This is a separate, unrelated topic that we aren't covering here.
{: .notice--info}

However, this ends up being a lot more complex than predicting simple actions (like ray-tracing a gunshot or triggering a particle effect): projectiles are actual _actors_; they have complex hit detection, physics simulations, and a myriad of potential side effects that can be triggered during their lifespan (like an explosion when landing). If we were to simply spawn the client's version of the projectile instantly, we would quickly discover synchronization issues and visual discrepancies (which we'll see later on).

Like most things in game development, there's no universal solution to this problem. Different games implement client-side prediction in a different way, specifically suited to the needs of the project. So before we design and implement our own, let's start by looking at some possible approaches.

## Possible Approaches

### No Prediction

Just to get a baseline, let's look at what our projectile would look like without any prediction at all.

![Prediction diagram: no prediction]({{ '/' | absolute_url }}/assets/images/per-post/projectile-prediction-1/visualization-no-prediction.png){: .align-center}

In this situation, when the player presses the "fire" button, they send a message to the server, asking it to spawn the projectile. Once the projectile is spawned on the server, it's replicated back to clients.

In this diagram, the distance between the mannequin and the first projectile represents where the projectile appears locally, relative to its actual spawn location. On the server, the projectile appears at its proper location, right in front of the player, as soon as it's spawned. But because it takes time to replicate the projectile back to clients, the projectile will appear _ahead_ of its spawn location, since it's been traveling in the time it takes to replicate (the exact distance will be $$({client \: ping} / 2) \cdot {projectile \: velocity}$$).

Already, we can see two big problems. First: the projectile is spawned a considerable amount of time _after_ the player presses the input. For clients playing with `60ms` of ping, it will take `30ms` for a projectile to spawn on the server, and _another_ `30ms` for that projectile to appear on the client. Second: projectiles appear a noticeable distance ahead of where they're supposed to be, on both the local and remote clients. If a projectile is traveling `100m/s`, it will appear `3m` ahead of where it should on clients with `60ms` of ping.

When I say "local" and "remote," I'm referring to the perspective of the projectile, not the server. So the "local client" is the client that fired and owns the projectile, while the "remote clients" are any of the other clients connected to the server. 
{: .notice--info}

### Fake Projectile

To solve these problems, a good place to start is conventional client-side prediction methods: performing the action instantly locally, and reconciling later on if necessary. To do this, when we press our input, we can spawn a "fake" projectile locally that instantly starts traveling.



This presents a new issue, however: because we're spawning the fake projectile instantly, it is now _ahead_ of the real projectile. This desynchronization can result in jarring visual discrepancies: the client's fake projectile may hit something that the real projectile missed, or vice versa, especially at high latencies.

### Fake Projectile with Synchronization

To fix our synchronization issues, after spawning the fake projectile, we could try to synchronize it with the real one once it's been replicated. There are actually two different ways to implement this particular solution.

The first solution is to simply switch to the real projectile once it's replicated, destroying the fake projectile in the process.



This is how _Unreal Tournament_ handles projectile prediction. You can see how they implement it [here](https://github.com/JimmieKJ/unrealTournament/blob/clean-master/UnrealTournament/Source/UnrealTournament/Public/UTProjectile.h).
{: .notice--info}

The downside of this is that the projectile will visibly "jump" backwards in time, since we're switching between projectiles that are in two different locations. However, projectiles are usually so small and travel at such high speeds that this jump isn't noticeable—especially amidst the action of a fast-paced game.

An alternative solution is the smoothly synchronize the projectiles over time by lerping the fake projectile towards the real one.



This creates a smoother visual, but the projectile hits something shortly after being fired, it may not have had enough time to fully synchronize yet. Though, in practice, it's highly likely that both the fake and real projectile will end up hitting that same target, even if they haven't fully synchronized yet.

Both of these solutions are perfectly viable (we'll opt to use the latter when we implement our own), and they help solve both of our problems (at least, for the local client; we'll get to fixing remote clients later). However, there's another major issue that might be difficult to notice just from these diagrams, and it has to do with both responsiveness _and_ fairness.

We mentioned that our predicted projectile is a "fake": it doesn't actually damage enemies or have any effect on gameplay; that's still the responsibility of the server's projectile.

What that means is that, since the server's projectile is the one actually performing hit detection, players with lower latency will still have an advantage, because their projectiles will be spawned on the server faster and be closer to their _desired_ shot (i.e. which is represented by their _fake_ projectile). This is another issue that we need to account for.

### Fast-Forwarding

Ideally, for maximum responsiveness _and_ fairness, the real projectile should be as accurate to the fake projectile as possible, since the fake projectile represents what the client actually wanted to fire. To do this, we can actually "fast-forward" (a.k.a. "forward-predict") the server's projectile, so that it appears where it _would_ be if it had been fired instantly by the client.



This essentially mitigates latency from the equation _completely_, which is amazing. However, this presents yet another problem. 

### 

Partially fast-forwarding the server's projectile

###

Partially fast-forwarding the server's projectile and resimulating the remote clients' projectile