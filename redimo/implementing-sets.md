---
title: Implementing Sets
layout: default
description: How to implement the Redis set operations on DynamoDB.

# Micro navigation
micro_nav: false

# Page navigation
page_nav:
    prev:
        content: REDIMO
        url: /redimo/
    # next:
    #     content: Implementing Strings
    #     url: '#'

---

The set operations from the Redis API are pretty simple to implement on DynamoDB, because having a partition and range key means we can just have the partition key as the set key, and the member itself as the range key. This gives us all the behaviour that we'd expect with a set. Repeatedly adding members will not duplicate them. The DynamoDB range is sorted, but we don't care about the sorting behaviour for the set operations anyway. 

### [SADD](https://redis.io/commands/sadd) key member
Remember that we've created our DynamoDB with a string partition key `pk` and a string sort key `sk`. The first operation is `SADD` which a member into a set. We can use a `PutItem` for this, with the set key on `pk` and the member value itself on `sk` — this will give us set behaviour where we don't duplicate members and just overwrite them if they already exist.

```yaml
# PutItem
Item:
  pk:
    S: key
  sk:
    S: member
ReturnValues: ALL_OLD
```

We don't send a value — we don't actually need or use one here. We also use the `ReturnValues: ALL_OLD` to check if the member was already present before the `PutItem` call. Redis counts the number of items actually inserted into the set in the call, so this allows us to do that.

### [SREM](https://redis.io/commands/srem) key member
Removing a member from a set is the similar to adding it. We call the `DeleteItem` operation with the same `pk` and `sk` we used for the `PutItem`. 

```yaml
# DeleteItem
Item:
  pk:
    S: key
  sk:
    S: member
ReturnValues: ALL_OLD
```

The `ReturnValues: ALL_OLD` helps again with knowing if the member was actually present in the set before we deleted it.
