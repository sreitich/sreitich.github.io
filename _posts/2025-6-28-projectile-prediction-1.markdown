---
layout: single
title: "Projectile Prediction: Part 1"
excerpt: An exploration of the theory behind projectile prediction.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-06-29
---

Part 1 of my series exploring and implementing projectile prediction for multiplayer games. This part breaks down the theory behind projectile prediction, some approaches to implementing it, and a short overview of the version _we'll_ create in (starting in part 2) using Unreal Engine.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...). However, this series is more focused on the theory behind projectile prediction, and the different approaches that can be taken to implement it, since, rather than just the code itself, since the approach you choose should be based on _your_ project's specific needs. 
{: .notice--info}

## Introduction

Real-time multiplayer games have to deal with something very tricky: latency—the time it takes for messages to travel between the server and clients. Whenever a player wants to do something, like fire a weapon, they need to send a request to the server; the server processes their request, updates the server game state, then sends a message to each client, telling them what happened, so everyone else can update their local game state to match. This results in a delay between every input players perform and the corresponding action.

This delay is a fundamental problem in multiplayer games—especially ones that need to be responsive. Without any compensation, controls will feel unresponsive and slow, and players with lower latency will have an advantage over others.

In game development, the primary method for making latency as unnoticeable as possible, and for helping to mitigate unfairness for players with different network conditions, is client-side prediction. With client-side prediction, instead of waiting for the server to acknowledge our input requests, we perform them instantaneously on our local game state, with the assumption that the server will approve our request. If it does, we predicted successfully, and the client and server can synchronize their game states (since the client will now be a little _ahead_ of the server). If we predicted _incorrectly_ (e.g. we tried to fire a weapon, but we were stunned on the server, so we failed), the server will tell us what _actually_ happened, so we correct our game state to match the server's (this is called server reconciliation).

This is an extremely simplistic overview of how client-server models and client-side prediction work, because I'm assuming you have some experience with these if you're reading this. If not, [this video provides a more comprehensive overview of this topic.](https://www.youtube.com/watch?v=2Xl0oaTKBXo)
{: .notice--info}

Unreal Engine handles 