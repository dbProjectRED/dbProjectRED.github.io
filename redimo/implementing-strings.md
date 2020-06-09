---
title: Implementing Key-Values or “Strings”
layout: default
description: How to implement the Redis key-value or “strings” operations on DynamoDB.

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
    
The Strings operations are the foundation of the Redis API, and are the simplest to implement. The word “strings” isn't very applicable when working with DynamoDB, though, because we can really store any type we want to – DynamoDB is strongly typed, and schemaless at the same time.

Because `GET`, `SET`, `INCR` and their variants operate on just a key (with no sub-fields, members or elements), we technically only need to use the `pk` for them. Since we've decided to use a compound key for our table (we need sk for the other data structures) we'll just use a constant default sk value for all of these operations – something like `'/'`, `'–'` or `'.'` will do. We'll store our value in an attribute named `val`.

We'll use YAML in the examples to concisely specify the important parts of the request, and leave out the boilerplate options like the table name, etc. You can add these in or use your language's SDK to automate these options.

### [SET](https://redis.io/commands/set) key value [EX|NX]

This is a simple `PutItem` operation. If we're trying to SET the key foo to the value bar, we could call `PutItem` with the following request data:

```yaml
# PutItem
Item:
  pk:
    S: key
  sk:
    S: "."
  val:
    S: value
```
We can also handle the `EX` flag, which modifies `SET` to work only if the key already exists, using a `ConditionExpression`:

```yaml
# PutItem
ConditionExpression: attribute_exists(pk)
Item:
  pk:
    S: foo
  sk:
    S: "."
  val:
    S: bar
```

This allows the PutItem operation to succeed only if the given attribute already exists. The `NX` flag, or the separate `SETNX` operation, needs the reverse condition of `attribute_not_exists(pk)` to make sure that the item is *not* set if it already exists.

Since we're just checking to see if the item itself exists or not, it doesn't matter too much which attribute's existence we check for. The pk attribute is guaranteed to be available in every single item, so we'll usually just go with that as a convention.

### [GET](https://redis.io/commands/get) key

Getting an item by key is also really simple - we just call `GetItem` with the following request:

```yaml
# GetItem
ConsistentRead: false
Key:
  pk:
    S: foo
  sk:
    S: "."
```    

We've passed in the two components of our key, `pk`, and `sk`, along with `true / false` option to specify whether we want a `ConsistentRead`. We generally want avoid a `ConsistentRead`, because it forces DynamoDB to give us *strong consistency*. This forces it to read from the same master node that handles writes, which is slower, more expensive and puts more pressure on the node. One of the only times this is useful is when we're trying to write something and then read it back immediately, but in any eventually consistent system like DDB we want to do that as rarely as possible. We want to use `ConsistentRead: false` as much as we can, because *eventual consistency* is faster and cheaper and is usually enough for most applications. 

### [INCR](https://redis.io/commands/incr), [INCRBY](https://redis.io/commands/incrby), [DECR](https://redis.io/commands/decr), [DECRBY](https://redis.io/commands/decrby) & [INCRBYFLOAT](https://redis.io/commands/incrbyfloat)

All of the increment and decrement methods collapse into the same DynamoDB operation, because (a) decrementing is just incrementing with a negative number and (b) floats and integers in DDB are both the same numeric type. For these operations we'll use the `UpdateItem` API call, which allows us to send in the delta and atomically increment the number for us, without us having to load it into our code, modify it and save it again. Being able to do this prevents the classic race condition of two processes trying to do this at the same time and having one overwrite the other.

The `UpdateItem` API also has an option to return the new value after the update using the `ReturnValues = UPDATED_NEW` setting, which is exactly what we want. It also creates a new numeric attribute with a zero default value if it doesn't exist, which again is exactly what we want. Our inputs are the key in `pk` and the delta – the delta for `INCR` and `DECR` are 1 and -1 respectively, and is the user-supplied delta for the others. In the case of `DECRBY`, we'll need to remember to negate the delta (multiply it by -1), because we're just to send everything in with an `ADD` operation. You might remember that `x - y` is the same as `x + (y * -1)`.

```yaml
# UpdateItem
ExpressionAttributeValues:
  delta:
    N: '42'
Key:
  pk:
    S: foo
  sk:
    S: "."
ReturnValues: UPDATED_NEW
UpdateExpression: ADD val :delta
```

Handling all increment and decrement operations with a single `UpdateItem` call.

The main part of this API call is the `UpdateExpression`, which tells DDB to add the value of delta to the existing value stored at val. If this attribute or item does not exist, the existing value will be assumed to be zero. You'll also see the first use of the `ExpressionAttributeValues` input, which is just a way for us to parameterize our inputs without having to worry too much about string interpolation.

The delta attribute was also specified under the key of `N` – which tells DDB that it's a numeric type. The `ADD` instruction that we use in the `UpdateExpression` is also only applicable on numeric types. The `pk` and `sk` attributes, on the other hand, are defined under `S` for string. The actual encoding of `N` itself is still a string – this is because we'd lose precision with floating point numbers if we used the native JSON number representation. DDB will parse the string under `N` as a number before using it.

The last important part is the `ReturnValues` input, which tells DDB what outputs we want back from the operation – and here we want the new number after our update, so we ask for `UPDATED_NEW`. This will populate the response JSON with the updated attribute under val.

### [MSET](https://redis.io/commands/mset) key1:value1, key2:value2...

The `MSET` operation requires a transactional DynamoDB API because it offers the guarantee that all keys will be set atomically – i.e. there will never be a case where readers using `MGET` will see some keys pointing to older values and some to newer ones. To do this, we'll use the `TransactWriteItems` API. This API takes a list of operations, each of which are similar to the PutItem or `UpdateItem` APIs – but the difference is that all operations are applied atomically, so either they'll all succeed of they'll all fail.

```yaml
# TransactWriteItems
TransactItems:
- Put:
    Item:
      pk:
        S: key1
      sk:
        S: "."
      val:
        S: value1
- Put:
    Item:
      pk:
        S: key2
      sk:
        S: "."
      val:
        S: value2
```

The API allows mixing any combination of `Put`, `Update`, `Delete` and `ConditionCheck` operations, but for `MSET` we're just going to keep it simple. The table name is also required inside each operation but has been omitted from the example for brevity.

One important thing to note is that because transactions span partitions and even tables, internally the DynamoDB nodes that are responsible for all of this data must coordinate with each other and perform locking to enable a transactional write. Because of this the operation is relatively slow, expensive and limited to a total of 25 items.

### [MGET](https://redis.io/commands/mget) key1, key2...

The `MGET` operation is the inverse of the `MSET` – it fetches a set of keys, but guarantees to do so in an atomic fashion, so you can be sure that no partial set of changes been applied to these keys. This requires the TransactGetItems API on DynamoDB, which makes that exact guarantee. If we use the TransactWriteItems that we saw in `MSET` to set multiple keys at the same time, we can use the `TransactGetItems` call to make sure that we never get a list of keys where a transaction has been partially applied. We call the API with a list of item keys:

```yaml
# TransactGetItems
TransactItems:
- Get:
    Key:
      pk:
        S: key1
      sk:
        S: "."
- Get:
    Key:
      pk:
        S: key2
      sk:
        S: "."
```

Both `MSET` and `MGET` are operations that invite failure by their very nature, so it's worth remembering the special error that DynamoDB gives out when a transaction on `TransactGetItems` or `TransactWriteItems` has failed because other transactions are happening on those keys at the same time. This error is the `TransactionCanceledException`, which has an internal `TransactionConflict` code for exactly this problem. There are other codes that might accompany this error as well, like `ItemCollectionSizeLimitExceeded` to indicate that there are too many (or too large) items in this transaction.

### [GETSET](https://redis.io/commands/getset) key

The `GETSET` operation is a special way to both set a value and get the old one out at the same time. The best way to do this is similar to what we did for the increment and decrement operations – use an `UpdateItem` call, but this time we want to use the `ReturnValues = UPDATED_OLD` parameter instead. We're also not doing any arithmetic, we we'll use the `SET` expression instead:

```yaml
# UpdateItem
ExpressionAttributeValues:
  newVal:
    S: newness
Key:
  pk:
    S: foo
  sk:
    S: "."
ReturnValues: UPDATED_OLD
UpdateExpression: SET val = :newVal
```

That's about all the operations we can reasonably build on DynamoDB. Redis has more operations in this category that do bit-fiddling, but this doesn't make much sense in Dynamo. It works on Redis because the operations happen quickly with a lock on a single memory location, but with DynamoDB we'd have to the bits into our application, change them, and write them back. We can still do all that with the basic operations we've listed here, maybe using a long string or by representing the bits as an integer – it's just that it doesn't make sense to treat them as special natively supported operations.

## Limitations

It's worth noting the differences form Redis, with these operations – Redis stores all data, including keys and values as internal strings, so they're subject to a 512MB size limit (you wouldn't want to go anywhere near this limit anyway, but that's besides the point). In DynamoDB the keys are limited to a kilobyte, so about a thousand characters. The value sizes are limited to 400KB, whatever their types are. These limits may be upgraded in the future, but it generally makes sense to use DynamoDB as a fast index for small bits of data or identifiers to larger pieces of data. If you're storing a large object you're better off putting the bytes in S3 and using DynamoDB to index the metadata. If you're storing photos, for example, you could store the photo itself in S3 with a UUID and add the metadata and indexing attributes into DynamoDB.

Redimo for Go is out now, and I'm working on other languages as well. Let me know [@sudhirj](https://twitter.com/sudhirj) if you want the library for a particular language or if you need help modeling your application data as Redis data structures.
