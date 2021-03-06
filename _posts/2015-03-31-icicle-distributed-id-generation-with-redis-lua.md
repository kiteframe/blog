---
layout: post
title: Distributed ID generation with Redis and Lua
date: 2015-03-31 15:00:00
excerpt: 'Introducing Icicle: a distributed, k-sortable unique ID generation system using Redis and Lua.'
image: /assets/images/id-generation.jpg
thumbnail: /assets/images/id-generation-thumbnail.jpg
categories: engineering
author: Nathan Kleyn
---

**TL;DR: We are open-sourcing our distributed _k_-sortable ID generation project called "Icicle", which generates IDs using Lua scripting within distributed Redis hosts. You can [find more information in the GitHub repository](https://github.com/intenthq/icicle)!**

At Intent HQ, we generate a lot of data. Part of the task of capturing this data is to package it up into a common format and store it in a way it can be retrieved. Commonly, labelling this data in a way that facilitates retrieval is a task handled by your database. However, in a big data environment, we need a way to label these pieces of data with an ID in a distributed and scalable way independent of a database.

Over the course of this article, we'll walk you through how we arrived at a solution for generating IDs in a distributed fashion using the Redis instances we already had deployed and in a format that allows _k_-ordering. We'll share with you the code for doing this so you can use this ID generation in your own projects!

## What is an ID?

An ID can be many things, but surely its most important definition is that of "uniqueness". Of all the properties, this is the only one that truly matters. If you label a piece of data with an ID, you want to be sure you don't later have ambiguities when looking up that data again by the ID you had.

IDs may have some other useful properties, however:

* _Ordered:_ They may be ordered to allow comparison of items stamped with IDs.

* _Meaningful:_ They may embed certain information about the point at which the ID was generated, for example the time (however inexact).

* _Distributed:_ They may be generated by a machine without consulting any other machine (ie. they must be truly distributed).

* _Compact:_ They may be represented in a specific format for space or performance reasons. Normally, you'd want to represent this in as compact a format as possible but you often trade-off longevity to do this (UNIX timestamps are a great example of compact albeit finite time representation in action [^1]).

These all sound like useful properties to have in an ID, but existing methods don't really work to preserve these properties in a distributed environment. Traditionally, in software, we have generated IDs using methods that do not scale. Either we let an application like a database generate IDs for us, in which case we generally have scalability issues, or we use standards like the UUID which do not guarantee uniqueness in a distributed environment without significant efforts at coordination [^2].

As a turns out, primarily, the reasons for doing this are deeply rooted in our inability to keep time across a distributed environment.

## Time is hard

Keeping time is actually a really hard problem to solve [^3]. At the core of the issue is the fact that computers suffer from "clock drift", where the clock is constantly skewing forwards or backwards away from the actual time. Perhaps even worse, every computer skews at a different rate [^4].

There are potential solutions for this problem, but each has its own set of pros and cons.

So called "time oracles" are dedicated servers normally equipped with an atomic clock or GPS device whose sole purpose is for keeping time in as precise a manner as possible. Time oracle servers are used frequently in environments where super accurate time is important (such as trading platforms and financial applications), but they are prohibitively expensive and not a viable solution for those working in an environment where deploying custom hardware is impossible (think cloud environments).

To combat this, software such as NTP is available, specifically written to ameliorate this problem for those who cannot access a dedicated time oracle. The idea is basically that large organisations donate access to their time oracle servers by adding them to a global pool of NTP servers which the masses can use to adjust their clock against. This is, unfortunately, not without its problems. Its an error prone process, as it involves network hops and jitter, and is thus susceptible to all kinds of inaccuracies that the underlying algorithms do their best to abate but cannot avoid. It's also really important to note that NTP is, by default, allowed to move time backwards to compensate for clock drift, something you must disable if you want to ensure any ID generation based on time can never generate duplicates.

![Distributed time is very difficult!]( /assets/images/id-generation-time.png)

Thus, you're left with one immutable fact: time is never constant in a distributed environment. You can can never _solely_ rely on time for ordering or identification, as we can simply not guarantee these properties over multiple hosts without a dedicated time oracle, a luxury not afforded by most.

## Ordering

The first property of IDs that breaks down in a distributed environment is that of ordering. In traditional SQL databases, we use things like auto incrementing primary keys despite knowing it has serious limitations in a replicated environment because ordering is so useful [^5]. Coordination techniques like global locking or single master setups to ensure sure IDs don't get duplicated or out-of-order can be a recipe for future scalability and performance woes.

Using time for ordering would be convenient because we don't need to coordinate on this value, but as we know time isn't a precise thing in a distributed environment, we know we can't use it on its own. We'll want to generate more than a single ID in a millisecond, and switching to more granular time would cause more problems due to the distributed inaccuracies aforementioned.

Chances are, in most applications you actually care more about uniqueness than ordering. What if we were to relax our requirements for ordering so that we could use time as a backing for our ID in a way that could be distributed?

This is where _k_-sorted comes in. The term "_k_-sorted" can be applied to anything which is "loosely sorted". That is, if I gave you the following numbers:

> 1, 5, 100, 102, 2, 101

A _k_-sorted copy of these numbers might look something like this:

> 1, 5, 2, 100, 102, 101

They are mostly, but not exactly, in order. Notice that really, all we're doing here is loosly ordering on the first part of the number: numbers less than 100 before numbers greater than 100. If we apply this to an ID, we may be able to sort on just a small portion of the ID to sort them. Perhaps the first portion of our ID could be the time, so as to allow ordering, and we could put put some stuff at the end that guarantees uniqueness?

## Space and Performance

How you choose to represent an ID has a big impact on performance in a distributed, big data application.

Atomically incrementing primary key columns in traditional SQL databases are difficult to scale because they have to lock while incrementing, but they can generate really compact IDs because of this. On the other hand, UUIDs try to circumvent the contention problem by adding entropy to the IDs until the chance of generating a duplicate ID becomes infinitesimally small; however the chance still exists and the IDs are huge.

We know that numbers are one of the fastest and most compact data types we can use on a computer, so a format like a hex string based UUID is out if we really care about these properties. In big data applications we need to care about data compactness as well, so ideally we would limit the size of the ID as much as possible.

As CPUs are primarily 64-bit these days, it makes logical sense to choose to pack an ID in something like a 64-bit long. It's the ideal data structure for our needs: the ordering operations we do with them will use bare-metal CPU instructions, and the space it takes up is minimal. In addition, we can hold many of these IDs in memory, which is a big performance win when dealing with big data.

The trade-off we'll have to make by trying to pack our IDs into just 64-bits is that, if we want time to be a part of it, they won't last forever (remember, time has to be just a small part of our ID, so we have only some portion of the 64-bits to represent it with). Some alternative solutions in this space have opted for using much larger numerical IDs [^6], up to and over 128-bits, but we think 64-bit integers represent the ideal sweet-spot between memory efficiency, performance and longevity for big data applications.

## The structure of an ID

We have the seeds for an ID: we know it will have time in it, we know it needs to have something that will make it unique across many nodes, and we'd preferably like it to be as small as possible for performance reasons.

We chose to pack our IDs in a 64-bit long, structured as follows:

![The ID structure, in the format ABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBCCCCCCCCCCDDDDDDDDDDDD](/assets/images/id-structure.png)

* The first bit is the reserved signed bit (which we chose not to use because some platforms make it difficult to get at and it messes with ordering).

* Secondly, the timestamp in milliseconds since a custom epoch, 41 bits in total.

* Next is the logical shard ID, 10 bits in total.

* Finally, the sequence, 12 bits in total.

We have represented the time as a 41-bit integer by creating a custom epoch: this simply involves picking an arbitrary point in time, calling it "0", and starting to count the milliseconds from there. As we're representing the time as a 41-bit number, we can generate IDs for ~69 years.

We have decided to generate IDs using Redis, but Redis is not distributed by default, so we assign each node an ID, called the "logical shard ID". This is simply a number between 0 and 1023 to distinguish one node from another. This ensures that if two nodes are asked for an ID during the same millisecond, they'll return different IDs.

What if a single node is asked for an ID again in the same millisecond though? Well, we have the "sequence number" to take care of that. This is simply a 12 bit number that starts at 0 and increments up to 4095 before looping back to 0 and starting again. This effectively means we can generate a maximum of 4096 IDs, per millisecond, per node. We have a maximum of 1024 nodes, we we can generate a theoretical maximum of 4,194,304 IDs in a single millisecond without duplicates. Should you exceed this amount, we should refuse to generate another ID until the next millisecond.

With all this together, once a list of IDs is sorted, they'll be grouped together by time and sorted by the (relatively) meaningless shard ID and sequence number after that. That is, they'll be _k_-sorted by time. Remember that time is not always consistent across these servers, so we only guarantee a roughly sorted list.

## Redis and Lua

The final piece of the puzzle is in why and how we use Redis to do this.

We primarily chose Redis for the task because it's a service we already had. We preferred not to have to deploy and manage another service for this task, opting instead to use something we already know how to monitor and we understand the behaviour of at scale.

Redis fitted our needs well as it allowed us to run arbitrary scripts using Lua; however, you could use any service that allows this [^7]. The ability  to run Lua scripts is important in the case of Redis, as it otherwise lacks the ability to conditionally run commands in a transaction.

We ran into one limitation with Redis: at the time of coding, the Lua bitops library provided by Redis didn't provide appropriate bit shift operations, something we needed to compose the 64-bit ID in pure Lua. As such, we currently call to Redis with the Lua script to get the components which make up the ID (time, logical shard ID, sequence) and bit shift them all together in the client-library (which is currently written in Java).

```java
long id = ((timestamp - CUSTOM_EPOCH) << TIMESTAMP_SHIFT)
       | (logicalShardId << LOGICAL_SHARD_ID_SHIFT)
       | sequence;
```

One final interesting aspect of Redis is that it does not actually support any form of distributed mode (although that will change when Redis cluster is released [^8]). As a result, we implement a naïve round-robin algorithm to cycle between a large, pool of Redis servers. This allows us to have full redundancy, and a relatively evenly distributed load across the servers.

## Open-source goodness

Now you know the theory and limitations behind generating distributed IDs, [we've open-sourced our implementation of these ideas on GitHub as "Icicle"](https://github.com/intenthq/icicle). The reference implementation is in Java, but really you could reimplement this easily in any language (remember, the primary part of it is [the Lua script that gets passed to Redis](https://github.com/intenthq/icicle/blob/master/icicle-core/src/main/resources/id-generation.lua), which is already implemented for you!)

Contributions to the source-code are welcome via GitHub pull requests!

**p.s:** The name of our project was inspired by a (sadly) now defunct project from Twitter called "Snowflake", from which we drew much inspiration [^9].

---

_[Image of fingerprint](http://en.wikipedia.org/wiki/Fingerprint#/media/File:Fingerprint_detail_on_male_finger.jpg) by [Frettie](http://commons.wikimedia.org/wiki/User:Frettie) is licensed under [CC BY 3.0](http://creativecommons.org/licenses/by/3.0/)_

[^1]: [The year 2038 problem](https://en.wikipedia.org/wiki/Year_2038_problem) is a great example of the trade-off between longevity and brevity and the eventual problems it can cause.
[^2]: [UUIDs require a great deal of manual configuration](http://en.wikipedia.org/wiki/Universally_unique_identifier#Random_UUID_probability_of_duplicates) to ensure IDs will generate properly in a distributed environment.
[^3]: A [recent article entitled "There is No Now"](http://queue.acm.org/detail.cfm?id=2745385) by Justin Sheehy, published on ACM Queue, is a great introduction to the difficulties of distributed time keeping.
[^4]: The [reasons for clock skew](https://en.wikipedia.org/wiki/Clock_skew) are numerous and varied.
[^5]: The Flickr engineering team blogged about how they [worked around the problems of replicated auto-incremented primary keys](http://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/).
[^6]: [Flake](http://www.boundary.com/blog/2012/01/flake-a-decentralized-k-ordered-unique-id-generator-in-erlang/) pitches itself as a distributed ID generation service written in Erlang. It generates 128-bit IDs.
[^7]:  Instagram have written about [their distributed ID solution which uses stored procedures in PostgreSQL](http://instagram-engineering.tumblr.com/post/10853187575/sharding-ids-at-instagram).
[^8]: [Redis version 3 will implement Redis cluster](http://antirez.com/news/79), a distributed mode for Redis.
[^9]: Twitter retired their [Snowflake](https://github.com/twitter/snowflake) project in 2014, but are planning on open-sourcing their current internal rewrite which is based on [Twitter-server](https://twitter.github.io/twitter-server/).
