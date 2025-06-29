---
layout: single
title: "Projectile Prediction: Part 1"
excerpt: An exploration of the theory behind projectile prediction.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-06-28
---

Part 1 of my series exploring projectile prediction in multiplayer games. This part breaks down the theory behind projectile prediction, some approaches to implementing it, and a short overview of the version _we'll_ create in (starting in part 2) using Unreal Engine.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...). However, this series is more focused on the theory behind projectile prediction, and the different approaches that can be taken to implement it, since, rather than just the code itself, since the approach you choose should be based on _your_ project's specific needs. 
{: .notice--info}

## Introduction

Real-time multiplayer games have to deal with something very tricky: latency—the time it takes for messages to travel between the server and clients. Whenever a player wants to do something, like fire a weapon, they need to tell the server, "_Hey, I want to fire my weapon._" 10, 30, or even 100 milliseconds later, the server receives the message, says "_Okay!_," and (usually) does what the client asked. Then, it takes _another_ 10+ ms for that "_Okay!_" to reach the client (along with every other connected client), so they can finally see their weapon fire.

This is the basic framework of client-server communication. And it means that everything the client sees and does is always noticeably behind what's _actually_ happening on the server. It may not seem like a lot, but even just 20ms (10ms each way) makes a noticeable difference in how the game looks and feels.

Game developers use all kinds of tricks to make latency as unnoticeable as possible, to make sure the game feels responsive. Chief among these is _client-side prediction_: instead of waiting to receive the server's "_Okay!_" when requesting some kind of action, clients will often just go ahead and do it on their own, with the assumption that the server will allow them to and eventually send that "_Okay_." If they're wrong (maybe they got stunned, so they _can't_ fire that weapon), the server will instead senc a message informing them what _actually_ happened, so the client can rewind their game state and correct themselves.