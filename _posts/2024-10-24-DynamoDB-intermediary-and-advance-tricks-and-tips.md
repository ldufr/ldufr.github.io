---
layout: default
title: Intermediary and advance tricks and tips for DynamoDB
---

# Introduction

My first experience with DynamoDB was in a code base that had number of issue
with the technology. This interested me and I set for myself the goal to learn
and fix our usage. I became very impressed by how elegant and powerful the
technology is when correctly used. I also was stuck few times on problem that
didn't has an obvious solution or for which I had difficulty finding useful
information online. The goal of this blog post is to present some of the tricks
and aha moments I had.

Before diving in, I want to emphasize that the blog post targets persons that
already have a fair amount of knowledge about DynamoDB. There is great resources
that do a much better job than I would, in presenting the tech and giving you
the tools necessary to start being productive. If you aren't, you can start by
looking the following links.

- [Single-Table Design with DynamoDB - Alex DeBrie](https://youtu.be/BnDKD_Zv0og?si=wb9Jx2o7PY5Ex3Fn)
- [Why data modeling is critical for success](https://www.youtube.com/live/YI67mWmjbZ4?si=WKFgklhPvrkAt0uY&t=15376)
- [Advanced Design Patterns for DynamoDB](https://youtu.be/HaEPXoXVf2k?si=VVVMvUBUA95Rre8p)
- [How to model one-to-many relationships in DynamoDB](https://www.alexdebrie.com/posts/dynamodb-one-to-many/)
- [The What, Why, and When of Single-Table Design with DynamoDB](https://www.alexdebrie.com/posts/dynamodb-single-table/)
- [Understanding DynamoDB Condition Expressions](https://www.alexdebrie.com/posts/dynamodb-condition-expressions/)
- [When to use (and when not to use) DynamoDB Filter Expressions](https://www.alexdebrie.com/posts/dynamodb-filter-expressions/)

# Include the item type as an attribute

One of the simplest improvement you can do is to add an attribute containing
the type of item. This will save you a ton of gymnastic on the primary key,
and simply allow you to switch/if on the type of item in order to deserialize
it correctly. For instance,

| pk     | sk        | type           |
|--------|-----------|----------------|
| USER#1 | #METADATA | User           |
| TEAM#1 | #METADATA | Team           |
| USER#1 | TEAM#1    | TeamMembership |

It might seems more intuitive to use the prefixes in the primary key, but it
gets quite convoluted and isn't worth the hassle.

# Supporting multiple kind of login methods

Initially, it wasn't so clear to me how to solve this problem. Taking a simple
example where the user is uniquely identified by an arbitrary id (e.g., a uuid)
and can connect with an email, or with a username. Naively, you might create
the following structure:

| pk     | sk        |                            |                  |                               |
|--------|-----------|----------------------------|------------------|-------------------------------|
| USER#1 | #METADATA | email: "user1@example.com" | username: "John" | password: bcrypt("Password!") |

You can create an index with partition key set to `email`, and similarly
create an other index with partition key set to `username`. This isn't a
scalable design, because, you're limited to 20 global secondary indexes
(by default), and there is many more login methods? Facebook, Google,
Steam, etc.

Instead, you can structure your data in the following way:

| pk     | sk                      |                               |                  |
|--------|-------------------------|-------------------------------|------------------|
| USER#1 | #METADATA               | email: "user1@example.com"    | username: "John" |
| USER#1 | EMAIL#user1@example.com | password: bcrypt("Passw0rd!") | confirmed: true  |
| USER#1 | USERNAME#John           | password: bcrypt("Passw0rd!") |                  |

If you combine that with the so-called "inverted index", that is the partition
key is `sk` and the sort key is `pk`, you can easily login a user based on the
email with the following query `sk = EMAIL#{email}`. Similarly, a user can
login with a username with the query: `sk = USERNAME#{username}`.

# Don't store data in the primary key (pk & sk)

One of the mistake I initially did was to use the primary key to store some of
the data. Consider the following:

| pk     | sk        |                            |                  |
|--------|-----------|----------------------------|------------------|
| USER#1 | #METADATA | email: "user1@example.com" | username: "John" |
| TEAM#1 | #METADATA | name: "John Team"          |                  |
| USER#1 | TEAM#1    | role: Admin                |                  |

If I wanted to query all the teams the user 1 is in, I would query for
`pk = "USER#1" and begins_with(sk, "TEAM#")`. This would give me all the
team memberships. I could extract the team id from `sk` by stripping the
prefix `TEAM#`. This adds a ton of complexity and isn't worth the effort.
Instead, duplicate the data in the following way.

| pk     | sk        |             |                            |                  |
|--------|-----------|-------------|----------------------------|------------------|
| USER#1 | #METADATA | userId: "1" | email: "user1@example.com" | username: "John" |
| TEAM#1 | #METADATA | teamId: "1" | name: "John Inc."          |                  |
| USER#1 | TEAM#1    | userId: "1" | teamId: "1"                | role: Admin      |

It's easy to build the pk and sk in order to do a query, but by duplicating the
data, you don't need to care about parsing the format. You can throw those
values out. This because especially apparent when using clever sk such as:

| pk     | sk                            | chatId | msgId | timestamp              |
|--------|-------------------------------|--------|-------|------------------------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1  | 1      | 1     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2  | 1      | 2     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3  | 1      | 3     | 2024-10-14T01:01:02.0Z |

This approach is especially great when you have the ability to automatically
deserialize the DynamoDB result in an structure. Think Rust serde, Go
marshaling, Java annotation, or any other. You may in fact, not even want to
store the `pk` and `sk`, but simply have a function to build them from the
other fields.

# Don't use an inverted index

When you learn about structuring many-to-many relationships, one of the first
"aha moment" should be the introduction of the inverted index. It seems that
you should always have one. Don't do that, simply set the primary key of the
index to a new field `gs1pk` and the sort key to `gs1sk`.

If we take back the chat example, with the following structure:

| pk     | sk                            | chatId | msgId | timestamp              |
|--------|-------------------------------|--------|-------|------------------------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1  | 1      | 1     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2  | 1      | 2     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3  | 1      | 3     | 2024-10-14T01:01:02.0Z |

the inverted index would be somewhat awkward to query. Instead, you could have
the following (simplified):

| pk     | sk                           | gs1pk  | gs1sk     |
|--------|------------------------------|--------|-----------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1 | MSG#1  | #METADATA |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2 | MSG#2  | #METADATA |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3 | MSG#3  | #METADATA |
| USER#1 | #METADATA                    |        |           |
| TEAM#1 | #METADATA                    |        |           |
| USER#1 | TEAM#1                       | TEAM#1 | USER#1    |

You can notice, that:

- For chat message, we have a `gs1pk`, but don't really use the `gs1sk`
- For user and team metadata, we simply don't include them in the secondary
  index. Having `#METADATA` as `gs1pk` would not serve much purpose other
  than listing all users.
- For the team membership, we do have the equivalent to the inverted index.

In this very simple example, it might seems useless, but this can avoid having
to create too many indexes. Indeed, in more complex scenario, we may want to
use the `gs1pk` and `gs1sk` for an access pattern that isn't covered by the
inverted index.

# Many-to-many vs many-to-many vs many-to-many

When using a SQL database, you never distinguish how many items are typically
in a many-to-many relationship. For instance, in Discord, a user can be in many
servers, but the number is certainly much lower than how many users can be in
a single server. In this relationship, you have "some servers"-to-"a lot of clients".

This can make a pretty big difference if you need to de-normalize your data.
Let's start with this very simple example.

| pk       | sk        |                  |
|----------|-----------|------------------|
| USER#1   | #METADATA | username: "John" |
| SERVER#1 | #METADATA | name: "Gaming"   |
| SERVER#1 | USER#1    | role: Member     |

You can easily list all server ids a given user is in, but that's probably not
what you want. When you query all the server the user is in, you probably also
want the name of the server. Similarly, querying all user ids in a server won't
allow you to show a nice list. You will need to follow-up with N queries in
order to get the username for every users.

This problem is particularly annoying, because if you duplicate the data, e.g.,

| pk       | sk        |                  |                  |                |
|----------|-----------|------------------|------------------|----------------|
| USER#1   | #METADATA | username: "John" |                  |                |
| SERVER#1 | #METADATA | name: "Gaming"   |                  |                |
| SERVER#1 | USER#1    | role: Member     | username: "John" | name: "Gaming" |

and you modify the server name, you may need to update hundreds, or thousands,
or maybe even millions of items. On the other hand, if you have thousands of
users in a server, you don't want to query thousands of usernames in order to
list them. Paging may seem to help, but only partially.

But knowing it's a "some servers"-to-"a lot of clients" let you structure it
this way:

| pk       | sk        |                  |                  |
|----------|-----------|------------------|------------------|
| USER#1   | #METADATA | username: "John" |                  |
| SERVER#1 | #METADATA | name: "Gaming"   |                  |
| SERVER#1 | USER#1    | role: Member     | username: "John" |

You can list all users in a server with a single query, including their
usernames. You can query the id of all server the user is in, and follow
up with multiple queries to get all server ids. But, since the assumption
is that a user is never in that many servers, usually 5-10, maybe sometime
50, it's not that high of a cost, especially with [BatchGetItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchGetItem.html).

If you edit the username of a user, you also need to update the server
memberships, but similarly, it's a small number. Especially with [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html)
or even [TransactWriteItems](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html).

On the other hand, changing the name of a server could have required updating
thousands or more items.

This is one of the consideration you should have when deciding which data to
de-normalize in a many-to-many relationships.

# Beware of querying multiple different type of items

One of idea I had, which I'm still not sure whether it make sense, but tend to
think not nowadays is querying many different item type in a single query.
Consider the following:

| pk     | sk                       |                          |
|--------|--------------------------|--------------------------|
| USER#1 | #METADATA                |                          |
| USER#1 | #EMAIL#user1@example.com | email: user1@example.com |
| USER#1 | #USERNAME#John           | username: John           |
| USER#1 | #GOOGLE#DEAD             | google: DEAD             |
| USER#1 | #META#F00D               | meta: F00D               |
| USER#1 | TEAM#1                   |

In this example, the idea is that every items starting with `#` are just an
"extension" of the metadata. The query `pk = USER#1 and begins_with(sk, '#')`
would return all items, except the team membership, because a user can be in
multiple teams.

Now, this might seems optimized, and probably save some places, but it's
a bit of a mess to deal with from code. You end-up building up a bigger
structure that merge all those fields, and especially if you have automated
serializing/deserializing, it's a pain to deal with. Instead, I now prefer:

| pk     | sk                       |                          |                |              |            |
|--------|--------------------------|--------------------------|----------------|--------------|------------|
| USER#1 | #METADATA                | email: user1@example.com | username: John | google: DEAD | meta: F00D |
| USER#1 | #EMAIL#user1@example.com | email: user1@example.com |                |              |            |
| USER#1 | #USERNAME#John           | username: John           |                |              |            |
| USER#1 | #GOOGLE#DEAD             | google: DEAD             |                |              |            |
| USER#1 | #META#F00D               | meta: F00D               |                |              |            |
| USER#1 | ORG#1                    |                          |                |              |            |

With this setup, I can query the single item I care about, in the code, I
deserialize it automatically in a struct/class containing annotations, and
updating those field do require to update more 2 items, but this is really
not a problem with [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html)
or even [TransactWriteItems](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html).

# Exception to the previous point

I mentioned in the previous point that I usually prefer to duplicate the data
instead of querying multiple different item types. There can be exception to
this rule, for instance, consider the following:

| pk     | sk              |                       |              |
|--------|-----------------|-----------------------|--------------|
| USER#1 | #METADATA       | state: <4.3 KBs blob> | click: 24600 |

Imagine a cookie clicker kind of game, where every time a user click a cookie,
you increase the attribute count. Since updating the item cost 1 WCU per KB,
rounded up, you would pay 5 WCUs for every click. Now, this further increase
if you have GSIs. If you have two where the user is replicated, you now pay
15 WCUs per clicks.

Instead, organize your data like this.

| pk     | sk              |                     |
|--------|-----------------|---------------------|
| USER#1 | #METADATA       | state: <large blob> |
| USER#1 | #METADATA#STATS | click: 0            |

Now, on every click, you can update the relevant item, which now cost 1 WCU
and this item is very unlikely to be replicated in any GSI. This example is
15 times cheaper.

# Conclusion

DynamoDB is a very impressive technology which is extremely cheap to use when
looking at AWS pricing. It's also extremely rigid, because changing access
patterns may require you to move your data around instead of updating a query
in the code. I don't think I would ever recommend it to a startup, but I also
think that most CRUD app have fairly stable and obvious access patterns. The
risk of drastically changing those access patterns is overdone in my opinion.
