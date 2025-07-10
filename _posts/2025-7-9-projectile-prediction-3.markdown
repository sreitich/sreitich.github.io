---
layout: single
title: "Projectile Prediction: Part 3"
excerpt: Implementing a base projectile actor class.
header:
    teaser: /assets/images/per-post/cone-trace/cone-trace-teaser.png
author: Meta
last_modified_at: 2025-07-09
---

The third and final part of a series exploring and implementing projectile prediction for multiplayer games. This part walks through the implementation of the base `AProjectile` actor class, which can be subclassed into projectiles that can be spawned by our `SpawnPredictedProjectile` task.

If you just want the final code, it can be found on [Unreal Engine's Learning site](...).
{: .notice--info}

## Introduction

In the previous section, we implemented an ability task to predictively spawn projectile actors. In this section, we'll implement a highly configurable base `AProjectile` class.

We'll start by finishing initializing and linking our projectiles together, which I intentionally left unfinished in the last section. From there, you can implement the class however you want to suit the needs of your project. But I'll show you my own implementation—including logic for movement, damage, visual effects, server reconciliation, and debugging—which should be suitable for most games.

There's around 2000 lines of code for this class, so a step-by-step walkthrough (like I usually do for these posts) probably isn't something either of us want. Instead, after we finish the initialization and linking step from the previous post, I'll just skip to the final code and go over each of the class's features, giving a brief explanation of how each works.

## Initialization

construction & linking

### Synchronization

lerp sync

## Debugging

dev settings class

debugging utilities

## Movement

## Hit Detection

## Effects

## Detonation & Reconciliation