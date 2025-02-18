---
layout: single
title: "Procedural First-Person Animation System"
excerpt: A breakdown of the first-person animation framework used in Cloud Crashers, and guide to building a similar system.
header:
  teaser: /assets/images/per-post/atomicity/thumb.jpg
author: Meta
---

A breakdown of the first-person animation framework used in _Cloud Crashers_, and guide to building a similar system.

# Introduction

**TODO: Teaser**

_Cloud Crashers_ is a hero-based fighting game. Each playable character has a unique weapon, set of abilities, and overall aesthetic that feels distinct. 

When designing the game's first-person animation system, we needed a robust framework that could streamline building large numbers of complex animation sets. But we also wanted a way to make each character feel unique, with their own sense of personality.

As I was researching solutions for animation systems, I came across this brilliant GDC talk by Blizzard Entertainment's Matt Boehm:

<iframe width="560" height="315" src="https://www.youtube.com/embed/7t0hLZd_8Z4?si=M6_tnPrCOSfHf0jU&amp;start=1192" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
<br>
In this presentation, Matt breaks down how _Overwatch_ uses animation layers, additives, and spring models—among other tricks—to convey each hero's unique personality through procedural first-person animations.

Even though it isn't a technical talk, Matt's high-level explanation of _Overwatch's_ animation framework provided great insights and inspiration for building a similar system for _Cloud Crashers_ in Unreal Engine.

On release, _Overwatch_ garnered critical acclaim for how it expressed character through first-person animations. [Check out this fantastic video by New Frame Plus that breaks down the animations themselves.](https://www.youtube.com/watch?v=7Dga-UqdBR8)
{: .notice--info}

In this article, I'll show how to create . By the end, we'll have an extremely powerful animation blueprint which can be used to create robust animation sets like this:

**TODO: Final result**

If you want to skip over the tutorial and just steal the code (you're more than welcome to!), check out the [CharacterAnimInstanceBase](https://github.com/ChangeStudios/ProjectCrash/blob/release/Source/ProjectCrash/Animation/CharacterAnimInstanceBase.h) and [FirstPersonCharacterAnimInstance](https://github.com/ChangeStudios/ProjectCrash/blob/release/Source/ProjectCrash/Animation/FirstPersonCharacterAnimInstance.h) classes on _Cloud Crashers'_ public source code.
<br>
<br>
As you can see, _Cloud Crashers_ actually uses two animation instance classes: a base class and a first-person subclass. This is because _Cloud Crashers_ also supports third-person, and the third-person class re-uses the code in the base animation instance class. For the sake of simplicity, in this tutorial, I've rewritten the base class and first-person class into a single class.
{: .notice--info}



![Default delta serialization]({{ '/' | absolute_url }}/assets/images/per-post/atomicity/deltaserialization.jpg){: .align-center}

To implement a custom atomic net serializer for our replicated struct we need to do the following:

{% highlight c++ %}
USTRUCT()
struct FExample
{
GENERATED_BODY()

	UPROPERTY()
	float PropertyA;

	UPROPERTY()
	float PropertyB;
 
	bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess);
}

template<>
struct TStructOpsTypeTraits<FExample> : public TStructOpsTypeTraitsBase2<FExample>
{
enum
{
WithNetSerializer = true
};
};
{% endhighlight %}

Our `FExample` struct should define `NetSerialize` and we should enable on its `TStructOpsTypeTraits` the property `WithNetSerializer`, which will make the struct replication to go through the implementation of `NetSerialize`, provided below:

{% highlight c++ %}
bool FExample::NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
{
Ar << PropertyA;
Ar << PropertyB;
bOutSuccess = true;
return true;
}
{% endhighlight %}

In this function, we are writing to the Archive `Ar` everything we want to replicate/serialize every time the struct changes. In this case we are writing both properties in the Archive, therefore we can guarantee that these two will be replicated always together. Meaning that, we can ensure atomicity.

`NetSerialize` will be called on the server for serialization, and on the client, for deserialization. We can also detect when we are serializing or deserializing using [`Ar.IsLoading()`](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Core/Serialization/FArchiveState/IsLoading/). Therefore If we wish to reduce the network bandwidth produce by our `NetSerialize`implementation, we can apply lossless and lossy compression techniques.

Note that references to other objects can never be atomic unless those objects are always resolvable on the client.
{: .notice--info}

# Conclusion

Today we learned the basics of replication atomicity and network serializers, but this is just the beginnning.

If you'd like to learn more about the matter I strongly recommend to take a look at [Giuseppe Portelli's](https://twitter.com/gportelli) [blog post about network serialization](http://www.aclockworkberry.com/custom-struct-serialization-for-networking-in-unreal-engine/), which provides a more elaborated explanation about how `TStructOpsTypeTraits` works and all the available properties for it, followed by some examples about compression techniques.

In addition, I also recommend taking a good read to the [`NetSerialization.h`](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Source/Runtime/Engine/Classes/Engine/NetSerialization.h) header from the Engine, as it contains very valuable information about how serialization works.

Thank you so much for reading, remember that all the feedback is welcomed, I deeply appreciate corrections and contributions, so feel free!

Enjoy, vori.