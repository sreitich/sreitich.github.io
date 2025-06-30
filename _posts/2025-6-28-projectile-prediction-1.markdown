---
layout: single
title: "Projectile Prediction: Part 1"
excerpt: An exploration of the theory behind projectile prediction.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-06-30
---

Part 1 of my series exploring and implementing projectile prediction for multiplayer games. This part breaks down the theory behind projectile prediction, some approaches to implementing it, and a short overview of the version _we'll_ create in (starting in part 2) using Unreal Engine.

This tutorial uses Unreal Engine for its code, but the theory and techniques are applicable for any game engine.
{: .notice--info}

If you just want the final code, it can be found on [Unreal Engine's Learning site](...). However, this series is more focused on the theory behind projectile prediction, and the different approaches that can be taken to implement it, since, rather than just the code itself, since the approach you choose should be based on _your_ project's specific needs. 
{: .notice--info}

## Overview of Client-Side Prediction

Real-time multiplayer games have to deal with something very tricky: latency—the time it takes for messages to travel between the server and clients. Whenever a player wants to do something, like fire a weapon, they need to send a request to the server; the server processes their request, updates the server game state, then sends a message to each client, telling them what happened, so everyone else can update their local game state to match. This results in a delay between every input players perform and the corresponding action.

This delay is a fundamental problem in multiplayer games—especially ones that need to be responsive. Without any compensation, controls will feel unresponsive and slow, and players with lower latency will have an advantage over others.

In game development, the primary method for making latency as unnoticeable as possible, and for helping to mitigate unfairness for players with different network conditions, is client-side prediction. With client-side prediction, instead of waiting for the server to acknowledge our input requests, we perform them instantaneously on our local game state, with the assumption that the server will approve our request. If it does, we predicted successfully, and the client and server can synchronize their game states (since the client will now be a little _ahead_ of the server). If we predicted _incorrectly_ (e.g. we tried to fire a weapon, but we were stunned on the server, so we failed), the server will tell us what _actually_ happened, so we correct our game state to match the server's (this is called server reconciliation).

This is an extremely simplistic overview of how client-server models and client-side prediction work, because I'm assuming you have some experience with these if you're reading this. If not, [this video provides a more comprehensive overview of this topic.](https://www.youtube.com/watch?v=2Xl0oaTKBXo)
{: .notice--info}

Between the character movement system and the Gameplay Ability System, Unreal Engine actually comes with most of the client-side prediction you'll need built-in. However, there's one important feature that Unreal, and most other engines, won't help you with: **projectile prediction**.

For disambiguation, the term "projectile prediction" can also refer to the indicators that appear when players are preparing to throw or shoot something, showing them the trajectory in which their projectile will travel. This is a separate, unrelated topic that we aren't covering here.
{: .notice--info}

## What is Projectile Prediction?

Projectile prediction is the client-side prediction performed when a client fires a projectile (a rocket launcher, a grenade, etc.). When the player presses the "_fire_" input, we want to instantly spawn the projectile for them, to keep the game feeling responsive.

However, this ends up being a lot more complex than predicting simple actions (like ray-tracing a gunshot or triggering visual effects): projectiles are actual _actors_; they have complex hit detection, physics simulations, and a myriad of other potential effects that can be triggered during their lifespan (like an explosion when they land). If we were to simply spawn the client's version of the projectile instantly, we would quickly discover synchronization issues and visual discrepancies (which we'll see later on).

Just like most complex issues in game development, there's no universal solution to this problem. Every game implements client-side prediction in a unique way (which is also why it's so difficult to find good resources on this topic online), specifically suited to the needs of the project. So before we design and implement our own, let's start by looking at some possible approaches.

### No Prediction

Just to get a baseline, let's look at what our projectile would look like without any prediction at all.
