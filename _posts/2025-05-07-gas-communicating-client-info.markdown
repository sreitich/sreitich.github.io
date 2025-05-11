---
layout: single
title: "Communicating Client-Side Data with GAS"
excerpt: How to script gameplay abilities to use data only available on clients, or to use a client-side value instead of the server's with GAS.
header:
    teaser: /assets/images/per-post/gas-communicating-client-info/teaser.png
author: Meta
last_modified_at: 2025-05-11
---

This is a short tutorial explaining how to script gameplay abilities when you need data that's only available on clients, or when you want to use the client-side version of data instead of the server's. I see _a lot_ of people encounter this problem when working with GAS, and I can't find anything online that explains how to properly do it.

This is an in-depth explanation of this script and its networking technicalities. If you just want the script and a summary, it can be found on [Unreal Engine's Learning site](https://dev.epicgames.com/community/learning/tutorials/Gj56/unreal-engine-using-client-side-data-and-mitigating-latency-with-the-gameplay-abilities-system).
{: .notice--info}

## The Problem

![Teaser]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/teaser.png){: .align-center}

### A Short Explanation of Client-Side Prediction

Unreal Engine's Gameplay Abilities System has built-in client-side prediction. This means that when you activate an ability (whose net execution policy is `Local Predicted`), it will be executed on both the local client and the server. The client will "predict" what will happen (so the game feels more responsive and accurate to the player), but the server will determine what _actually_ happens.

An issue arises when the client predicts something different from the server. For example, if the client tried to activate an ability, but the server didn't let them (maybe they got stunned), then the server has to send a message to the client to "correct" them. This is done by rewinding the client's missed prediction (e.g. undoing any logic their rejected ability performed), and resimulating their game to align with the server's game (e.g. putting them into the "stunned" state). This is called "server rollback."

This is a very rudimentary explanation of a very complex topic, since I'm hoping you're familiar with it already if you're using GAS for a multiplayer game.
{: .notice--info}

### Client vs. Server Data

GAS is server-authoritative by default. When executing logic inside abilities, if the local client and server perform different actions, GAS will always treat the server as the "correct" one. This is usually what we want to do (players are notoriously untrustworthy).

There are times, however, when we actually want to use the _client's_ version instead. There are two common situations when this happens.

#### 1. The Data Only Exists on the Client

Sometimes, you want to use some data in a gameplay ability, but it doesn't exist on the server.

The most common example of this that I see is when people want to create some kind of "Dash" ability that moves the player in the direction of their input. E.g., if the player is holding down `A` on their keyboard, they should dash left. This is an incredibly common ability in games: Tracer's "Blink" in _Overwatch_, Jett's "Tailwind" in _Valorant_, directional sliding in _Call of Duty_, etc.

<video width="100%" height="100%" muted autoplay loop>
   <source src="/assets/videos/per-post/gas-communicating-client-info/jett-dash-vid.mp4" type="video/mp4">
    Video tag not supported.
</video><br>
But in Unreal Engine, you'll very quickly encounter a problem: the variable that tracks the player's directional input, `APawn::LastControlInputVector`, isn't replicated; it only exists on the client. So if you try to set up an ability like so:

![Dash ability using "GetLastMovementInputVector" on the server]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/intuitive-dash-setup.png){: .align-center}

... it won't work. The client will correctly predict the movement, but the server will rubber-band us back, because calling `GetLastMovementInputVector` on the server returns `(0, 0, 0)`.

#### 2. The Client's Data is More Accurate

Another common situation where we want to use the client's data is when it's more accurate than the server's.

If we, for example, want to fire a weapon in the direction the player is aiming, their version of `GetPlayerViewPoint` is going to be more accurate than the server's, since they're the one that's actually reading the mouse data to know where the player is aiming.

This is true for any data that's client-driven (basically anything involving input). Since this data is being calculated on the client and continuously replicated to the server (which is the opposite of how most replicated data is treated in server-authoritative games), the server's version may be behind—especially at high latencies, when it takes a long time for the client to update the server.

It's important to note that when you do this, you're _trusting_ the client. The method I'm going to show you is very simple, doesn't involve any kind of validation, and should only be used when clients can _not_ take advantage of manipulating the data they send.
<br>
<br>
For example, when scripting a "Dash" ability, there's little risk to trusting the client when determining the direction they should dash. But trusting the direction in which a client fires their weapon is a lot more dangerous.
<br>
<br>
For these kinds of situations, you should use a more robust prediction system, like GAS's [target actors](https://github.com/tranek/GASDocumentation?tab=readme-ov-file#4112-target-actors). Alternatively, see how [Lyra implements its own method for predicting weapon traces](https://github.com/EpicGames/UnrealEngine/blob/ue5-main/Samples/Games/Lyra/Source/LyraGame/Weapons/LyraGameplayAbility_RangedWeapon.h) with GAS.
{: .notice--info}

## The Solution

The solution to both of these situations is actually pretty simple. All we need to do is calculate the value on the client, then send that value to the server with an RPC. Here's what that looks like for the "Dash" example I mentioned earlier:

![Dash ability using a "Send Info" server RPC]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/final-setup.png){: .align-center}

Your ability's `Replication Policy` has to be set to `Replicate` for this to work. Otherwise, the ability won't be able to send the RPC. Also note that `SendInfo` is a reliable server RPC.
{: .notice--info}

The rest of this post is an explanation of the above. If you understand how this works, you're good to go.
{: .notice--info}

Small design note: If the player isn't pressing any buttons, the input vector will be `(0, 0, 0)`, so the player won't move at all. If you still want the player to move when they aren't pressing anything, you can do something like this, which will default to moving the player forward. _Overwatch_, _Valorant_, and _Call of Duty_ all do this:<br><br>
![Making direction the player's aim by default]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/setup-with-forward-default.png){: .align-center}
{: .notice--info}

This might look weirdly complicated, but that's only because we have to account for a few different scenarios, depending on who's activating the ability and where the ability is being executed (a remote client, a listen server, or a dedicated server). Let's go through each situation:

### Client Activation with Client and Server Execution

The most common situation is when a remote client activates an ability that runs on both the local client and the server; i.e. playing on a remote client when `Net Execution Policy` is `Local Predicted`.

On the local client, this code will cache its current value of `LastMovementInputVector` in `Direction`. Then, it will send an RPC with that value to the server. Finally, it will predicatively perform the dash with our cached value, which we know will be correct, since the server will use the same one once it receives it:

![Path when activated on client and executed on client]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/client-activated-client-server-executed-client.png){: .align-center}

On the server, `IsLocallyControlled` will be false, so nothing happens when the ability is initially activated. Instead, the server will wait to receive `SendInfo` from the client, and _then_ it will start its ability logic using the value sent by the client:

![Path when activated on client and executed on server]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/client-activated-client-server-executed-server.png){: .align-center}

You might think that this will cause a delay on the server, since it has to wait to receive the RPC, but it actually won't.
<br>
<br>
When the player activates an ability, they send an RPC to the server saying, "please activate this ability" (well, this isn't technically what happens, but it's close enough for this explanation), then the client activates the ability locally. What we're doing now is the same thing, but we're now sending _two_ RPCs: we're first saying, "please activate this ability," and then, after we activate the ability locally, we say, "and here's the value you should use for it," and we're telling the server to wait until it receives the _second_ RPC to start executing the ability's logic.
<br>
<br>
It does take time to activate the ability and fire the second RPC, but that time is so short (nanoseconds) that both RPCs will almost _always_ be fired in the same tick.
<br>
<br>
This means that both RPCs are likely reaching the server at the same time (depending on how Unreal sends the packets). So, even though we're telling the server to wait for the second RPC, there won't be any delay, since that RPC is arriving at the exact same time as the first RPC.
{: .notice--info}

### Listen Server Activation with Client and Server Execution

Another situation we account for is when a listen server activates an ability that runs on both the local client and the server; i.e. playing on a listen server when `Net Execution Policy` is `Local Predicted`.

Here, the server has access to `LastMovementInputVector`, since it's also acting as the local client, so we can skip the RPC entirely. Instead, we simply use the local value:

![Path when activated on a listen server]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/listen-server-activated-client-server-executed.png){: .align-center}

Since the local client and server are the same machine, this is the only possible flow of execution.

### Server-Only Execution

If your ability only runs on the server, you'll have no way to receive client data within the gameplay ability; the client never runs the ability, so it never gets the chance to call the RPC. It's rare you'd need client data in a server-only ability. But if you do, you could change the policy to `Local Predicted` or `Server Initiated` (instead of `Server Only`) so the client can send the RPC, and put any actual ability logic behind a `Has Authority` check:

![Path when activated on a server and executed on server]({{ '/' | absolute_url }}/assets/images/per-post/gas-communicating-client-info/server-activated-server-executed.png){: .align-center}

### Client-Only Execution

Finally, if your ability only runs on the local client, you'll obviously have access to any client data you want, so you'll never encounter this problem.

Note that our "Dash" ability is a terrible example for this. Character movement is always replicated and is server-authoritative by default, so you'll just get rubber-banded back by the server if you try to move in an ability that only runs on the client.

## Other Solutions

Here are a couple other potential solutions that might seem enticing, but I'll explain why you _shouldn't_ do any of these, and should use the above method instead.

### Make the Data Replicated

If we want to know `LastControlInputVector` on the server, we could just make another variable in our player character named something like `ReplicatedControlInputVector`. We could make this variable a replicated property and, each frame, set it to the value of `LastControlInputVector`.

Technically, this works, but it's an unnecessary waste of bandwidth. If we only need this variable when we activate an ability, it's wasteful to continuously replicate it to the server, when we could just replicate it once when we actually need it.

Replicating a single variable (especially something as small as a Vector3) doesn't take a ton of bandwidth. But `ACharacter` has a default `NetUpdateFrequency` of `100.0`. If we activate this ability once every 10 seconds, we could be sending **1,000** replication updates (depending on the tick rate) instead of **1**, which would be a little silly.

### Just Use Server-Known Values

If we wanted to know the direction of the player's input, we could just use their acceleration instead. The movement system works by reading the player's input vector and applying acceleration in that direction, and, unlike the input vector, acceleration is known on the server. Using the player's normalized acceleration direction would give us a good approximation of their input vector.

However, this is more of a hack than a solution; there's a lot of ways this setup could break. For example, if we had _any_ kind of knockback system in our game (a lot of multiplayer games do), this setup won't work, since the player's acceleration will instead be `InputVector + CurrentKnockback`.

Even we _could_ guarantee that our acceleration always matched our input vector, this solution—using the server's value—would still be suboptimal, since the server's version of a client-driven value won't be as accurate (as explained earlier in this post).

## Conclusion

Gameplay abilities hook into Unreal's replication system, so this setup works for _any_ data that can be replicated. For example, in the _Lyra_ project, Epic uses this exact script to replicate which animation montage asset should be used for their "Dash" ability.

Lastly, I want to note that this script is a great way to mitigate latency issues when using client-driven values in abilities. For example, if you have an ability that moves the player in the direction they're looking, you'll get a lot of rubber-banding at higher latencies. The player's `ControlRotation` is known on the server, so the traditional "predict with the client's value, use the server's value" ability script will work. But, because of the latency to the server that I explained earlier, the server's `ControlRotation` may not match the client's, resulting in rubber-banding when the client uses its own value in its prediction. Sending the client's `ControlRotation` to the server instead will mitigate this issue and prevent rubber-banding.
