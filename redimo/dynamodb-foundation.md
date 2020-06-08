---
title: DynamoDB Foundation
layout: default
description: A quick primer on the DynamoDB concepts and operations used in the Redimo libraries.

# Micro navigation
micro_nav: false

# Page navigation
page_nav:
    prev:
        content: Introducing Redimo
        url: /redimo/intro
    # next:
    #     content: Implementing Strings
    #     url: '#'

---

The Redimo set of libraries apply the approachable [Redis API](https://redis.io/) and data structures over the powerful DynamoDB system, letting you use all that power with a high-level API that hits the sweet spot between the domain reflection of SQL / RDBMS systems and the very low level operations of DynamoDB. This post describes the basics of DynamoDB required to implement Redimo.

First off, all our data in DynamoDB will live under a single *table*. Unlike some other database systems, a DynamoDB table isn't built to represent a single type of entity in our application. It's more a logical grouping that allows simpler scaling and management of our application as a whole. This is because DynamoDB is scaled and distributed using partitions, not tables. Each piece of data is a key-value map called an *item* – every item has a special key called the partition key that DynamoDB hashes to find which internal node that item's data lives on. The table is effectively just a prefix to the partition that aids with organization, billing and scaling properties.

The other points to note about items are that their attributes are strongly typed, so the API requires us to specify a type for each value. The items themselves are *schemaless*, though – we can have any set of attributes on any item.

Every one of our items is indexed – to do this DDB asks us to specify one main attribute called the partition key, that needs to be present on every item. This attribute is hashed by DynamoDB to determine which internal node the attribute lives on, and allows for very fast `O(1)` lookup. In any non-trivial system, there are going to be multiple kinds of data living on the same table, so instead of naming this indexing attributes with a domain / application specific name, we'll just call it `pk`, short for *partition key*.

DDB also optionally allows a second level of indexing inside a partition – as a sorted array based on the value of a second attribute called a sort key. This allows us to do fast queries using range/between,  starts-with, greater/less than, equal-to queries on the values of this attribute. Depending on the internal implementation, this might be a B-Tree index, which would result in these queries running in `O(log N)` time, but more likely they're a different data structure that's customized for the internal workings of DynamoDB. Either way, the cost complexity is `O(1)` – DDB charges the same for these queries irrespective of the amount of data you have inside that partition key, so we have to assume that they're managed to make it work with `O(1)-ish` economics. The important point to note is that these queries can only happen inside the scope of a partition, so the partition key is always necessary when querying on this sort key as well. We'll designate one attribute to be the *sort key*, and we'll call it `sk`.

DynamoDB also allows designating multiple other sort keys to index on, under the same partition, and these are called *secondary indexes*. We'll be using one for Redimo, specifically in the sorted-set operations, where a set member is queryable both by a string name and a numerical score. In cases like this, we'll designate another sort key, just called `skN` for the numeric sort key.

The secondary index will need to be either a *local* or *global* secondary index. The difference is that a local secondary index lives on the same partition as the original data, so there are some restrictions on total size of the data inside that partition – as of now it needs to be less than 10GB. Having the local index allows us to read data with *strong consistency* – we can choose to read from the same internal node as the master data, so if we write something we're guaranteed to see it immediately on a strongly consistent read. The alternative read type is *eventual consistency*, where the read can possibly happen from other nodes that have a copy of the data – but they're not guaranteed to be up-to-date immediately after a write. Using the *global* secondary index allows us to have only eventual consistency, because global indexes are spread over many nodes in the DynamoDB system, but removes any limitation on data size. Redimo works fine in both cases, but you'll want to choose one based on your application's needs. In Redimo this choice of consistency model only impacts sorted-set queries on score, because that's the only secondary index we use.

To sum up, we'll use the following index configuration to implement the Redis data structures:

* `pk [string]` will be the main partition key. This will correspond to a Redis key.
* `sk [string]` will be the main sort key. This will help us with sets, sorted set members, hash fields, geo members, stream IDs, etc. This also means that every item in the table is uniquely identified by the pk, sk compound primary key.
* `skN [numeric]` will our secondary index, either LSI or GSI. This will hold the sorted set scores and geo cells. Indexed as `pk -> skN`.

The main functions in the DynamoDB API that we'll use are:

* [`UpdateItem`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_UpdateItem.html) is the most common API we'll use call because it works for both creating and updating items, and for doing both conditionally. It takes in a compound key (`pk & sk`), a map of attributes and values to write on the item, and conditions to check. Despite its name, UpdateItem also allows writing items that don't exist yet. `UpdateItem` also allows incrementing or decrementing numeric attributes atomically – we can just send in a `SET val = val + 42` instruction; as opposed to loading val into the application, modifying it and writing it back, which causes a lot of race conditions. It also allows us to change some attributes of an item without having to rewrite (or even know of) all of them.
* [`PutItem`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html) creates or completely overwrites the item at the given key with the given input attributes. The key is the `pk & sk` combination, and the inputs a map to string names to typed values. If the item does not exist it is created, and if it does all existing attributes are removed and the new ones are written. `PutItem` can be conditional as well, which helps in cases like “set only if item doesn't exist” or `SETNX`.
* [`GetItem`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_GetItem.html) gets items. That's about it. Very useful. Has an option for strong or eventual consistency. We'll be using this with `pk & sk` compound key.
* [`Query`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html) runs queries against our table, returning all items that match certain conditions. In our case all queries will be inside the context of a single `pk`, so `pk` equality will always be the first condition. Depending the operation we might use `BETWEEN`, `< <= >= >` or `=`on the `sk` as well. Query also lets us count items instead of return them, which is useful in all the length check operations. We can also run queries against the other indexes, so sorted set score queries will run on pk and `skN` instead, against the local or global secondary index created with those attributes.
* [`TransactWriteItems`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html) is a complex API that acts as a wrapper for multiple (up to 25) `UpdateItem`, `PutItem` or `DeleteItem` operations. It gives us an atomic transaction over these operations, ensuring that they all succeed or fail atomically (there's no way half the updates will succeed while the other half fails). This allows us to build more complex data structures like lists – in a double linked list the pointers on both sides of an element must be updated simultaneously, along with the addition or deletion of the element itself. We also use it to make sure we never insert items older than existing items in streams. If we didn't have an operation like this, writing to the same parts of our data from multiple sources at the same time could easily corrupt it.

Each of the Redis data structures – key-values(strings), sets, sorted-sets, geo-sets, lists, hashes and streams – will be covered in a separate post and linked here. For now, check out the [Go library](https://github.com/dbProjectRED/redimo.go).
