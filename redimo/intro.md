---
title: Intro to REDIMO
layout: default
description: Redimo is a suite of libraries that apply the Redis API over DynamoDB. Currently availalbe for Go.

# Micro navigation
micro_nav: false

# Page navigation
page_nav:
    prev:
        content: REDIMO
        url: /redimo/
    next:
        content: DynamoDB Foundation
        url: /redimo/dynamodb-foundation

---

When AWS built DynamoDB they created a monster. Not necessarily the bad kind, just the kind that's incredibly powerful but ugly has hell – the kind that needs arcane incantations to make it do what you want it to do, but can destroy worlds if you tame it.

The API for DynamoDB is a reflection of what it is. It acts as a key-value store in that it allows you to put items into DynamoDB and get items out of it, but it also acts as a sorted B-Tree index that lets you sort items, but only if you specify a sort key along with the partition key from the key-value store paradigm. And query only by that one sort key, but you could set up other indexes that work on different keys, but you can't query two of them together. But if you index with a local secondary index you're doing it wrong and limiting your data size, but then a global secondary index will cause you not to be able to read items you just wrote. And you can do transactions, but only with 25 items, and if you do you can only reference each key attribute in your item once, except that you can also add a condition for other items, but only if you're not transacting on them. This goes on for [450 pages in the book that says DynamoDB doesn't have to be complicated](https://www.dynamodbbook.com/), so there you go.

The Redis project, on the other hand, hit a sweet spot. It also has a large API, but every operation is a read from or a modification to a data structure like a key-value map, set, sorted set, list, hash-fields, geospatial sorted set, and so on.

Redis doesn't follow the RDBMS approach of asking you to model your data as a reflection of your application objects' relationships – by having your data completely disjoint in the system, and requiring you to JOIN things LEFT, RIGHT and INNER to do any work at all, an RDBMS forces all your data to be in the same place to work with any measure of effectiveness.

Nor does Redis follow the DynamoDB approach, trying to make you fit your data into the primitives that AWS engineers currently think is implementable in a large scale distributed system.

Redis asks you to model your data into well-known and well-understood data structures that you can find in any textbook, and provides fast & efficient implementations for these data structures. This idea of doing the work to represent your data in well-chosen data structures is something that smart people have been recommending since before I was born, so let's assume its a good idea for now. Is there a way we can apply the Redis API principles to DynamoDB?

### Why not just stick to Redis?

Redis is laser-focused on speed and efficiency, so it went down the path of restricting itself to a single computer (and a single core), saving all its data in RAM and writing to disk asynchronously. This is fantastic for a particular set of use cases, but sometimes you don't need microsecond response times with one operation running at a time – you're okay with millisecond responses if your workload can be spread across tens or hundreds of servers, so multiple operations run simultaneously without blocking each other.

Redis does have a cluster mode, but if you compare it to DynamoDB you can [almost hear Dynamo say](https://www.youtube.com/watch?v=bpmNgPzklmQ) “Haawww, You think the cloud is your ally. You merely adopted the cloud. I was born in it. Moulded by it.” before proceeding to kick some Redass. There's really no comparing the two, becuase DynamoDB has the whole distributed system game locked down so hard AWS runs AWS on it, and once Amazon moved their shopping cart to DynamoDB they run Prime Days without having to look at the DB dashboards the way concert pianists play without having to look at their fingers.

That's not to say DynamoDB is perfect – if you do all your work on a single key you'll be sending all your DynamoDB operations to one single server while a hundred thousand more sit idly laughing at you with that droning staccato their fans make – and then that one server will throttle you; crashing your app and causing you to write angry blog posts on Hacker News about how Dynamo totally failed you and your snowflake use-case.

Being a distributed system allows DynamoDB do things like make sure your data is safe on disk in multiple data centers before acknowledging your writes, automatically copying your data to multiple nodes so your response times actually get faster under higher load, give you near-infinite storage, billing that can work on a per-request basis (so you pay nothing when not using it), scale up and down automatically within minutes to handle load spikes and a bunch of other serverless advantages.

But what can we do about that horrible and overly-demanding API? We could just overlay the Redis API on top of it. DynamoDB has key-value and B-Tree operations, and you can go a long way in data-structure land with those. The Redimo project implements all the current Redis structures, including streams and geospatial sets, into efficient DynamoDB operations. Some operations are perfectly efficient (as fast as they can logically be), some better than Redis, and some worse.

The documentation on each Redimo method will tell you what the differences are from Redis, both in terms of the interface and the time/cost complexity, but for the most part there's parity between the two systems. Some operations like counting become O(N) on DynamoDB (where we don't want to do distributed locking and book-keeping) instead of O(1) on Redis (where every operation locks by design), but others become faster because we can do things in parallel, and some others like sorted sets become more functional because we can have two indexes instead of one.

### But what's the catch?

There's no way to build an abstraction that doesn't leak  – as you push against the limits of DynamoDB you will start to see bits of it peek through, but 1) most applications will not push that hard and 2) the DynamoDB team is constantly working to make it faster and better, so this is less likely to happen as time goes by. There are basic principles that need to be followed, the most important of which is to **spread your workload across your keys as evenly as possible**. If you can handle that, Redimo will do the right thing, call out gotchas in the documentation, and handle your data just the way you expect.

### What's Available Now

The [Go Redimo library is now in beta](https://github.com/dbProjectRED/redimo.go). It will remain in beta while documentation is added and it's tested in my own production applications, after which v1 will be released – v1 will guarantee that the API and data format will be backwards compatible with new minor versions of the library. 