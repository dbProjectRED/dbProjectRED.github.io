---
title: Implementing Redis Sets on DynamoDB
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

**We're going to be using the operations and concepts referred to in the [DynamoDB Foundation](/redimo/dynamodb-foundation/), so please read that first.**

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

### [SCARD](https://redis.io/commands/scard) key
Counting a set, also known as checking its *cardinality*, in our case is counting the number of items inside a particular partition key `pk`. We do this with a modified form of the [`Query`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html) API operation.

```yaml
# Query
KeyConditionExpression: pk = :pk
ExpressionAttributeValues:
  pk:
    S: key
Select: COUNT
```

This query runs over all the items inside partition `pk`, which represents all the members of our set, but instead of returning them it returns the count of the items instead. 

If there are too many items to count in a single query, the response will contain a `LastEvaluatedKey`, which can be passed back into the `Query` operation again as [`ExclusiveStartKey`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html#DDB-Query-request-ExclusiveStartKey). This allows a continuation of the count until a `LastEvaluatedKey` is no longer being returned, at which point we know the entire set has been counted. 

The simplest way to finish all query operations that return a *cursor* like the `LastEvaluatedKey` would be a `do...while` loop with the presence of the `LastEvaluatedKey` as the condition for the `while`. The `do` portion of the loop ensures that it runs at least once, and then paginates until there are no more pages available. 

### [SISMEMBER](https://redis.io/commands/sismember) key member
Checking whether a member is part of a set is as simple as trying to get it and seeing what we get. We can call [`GetItem`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_GetItem.html) with the same arguments as passed into the `PutItem` when adding the member.  

```yaml
# GetItem
Item:
  pk:
    S: key
  sk:
    S: member
```

If the member isn't present in the set, this will return an empty result, which allows us to return false for the operation. 

### [SMEMBERS](https://redis.io/commands/smembers) key
Fetching all the members of a set is very similar to counting them, which is what we don in [`SCARD`](#scard-key). We'll use the [`Query`](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_Query.html) API operation:

```yaml
# Query
KeyConditionExpression: pk = :pk
ExpressionAttributeValues:
  pk:
    S: key
```

 This is will return the first batch of members, along with a `LastEvaluatedKey` if there are any more. If this key is present, we can make another call to query and pass it in as `ExclusiveStartKey`. We keep doing this until we no longer see the `LastEvaluatedKey`, which tells us that all the members have been fetched. It makes sense to run `Query` in a `do...while` loop similar to [`SCARD`](#scard-key).
 
### [SINTER](https://redis.io/commands/sinter), [SUNION](https://redis.io/commands/sunion), [SDIFF](https://redis.io/commands/sdiff)
These are really just application level computation, from the DynamoDB point of view. We'll need to load each key we want to work with into application memory using `SMEMBERS` and perform the intersection, union or diff operations there. 

### [SINTERSTORE](https://redis.io/commands/sinterstore), [SUNIONSTORE](https://redis.io/commands/sunionstore), [SDIFFSTORE](https://redis.io/commands/sdiffstore)
Similar to `SINTER`, `SUNION` and `SDIFF`, these are also application level executions of those operations followed by `SADD` for the resulting members. It is possible to optimise this by using [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html) but this won't tell you if the members you're inserting already exist or not, so it may or may not work for you.

### [SRANDMEMBER](https://redis.io/commands/srandmember) and [SPOP](https://redis.io/commands/spop)
The `SRANDMEMBER` fetches a random member of the set — there's no comparable operation in DynamoDB, so depending on your application you might want to just fetch the first or last member using a `Query`. It is possible to use a random number as well, stored in a secondary index. Or to add an offset to the query that skips the first random `N` members. This all depends on your application, though. 

`SPOP` is pretty much the same as `SRANDMEMBER`, except that it also deletes the member from the set, for which we use `SREM`.