---
layout: default
title: DynamoDB tips and learning
---

# Introduction

My first experience with DynamoDB was somewhat negative. It didn't support join and doing all the necessary queries manually would incur multiple database round-trips. But, like MongoDB's meme goes, "you're using it wrong". The thing is, DynamoDB can't used incorrectly, at least, not in the same way as MongoDB. I will try to present the main "
aha moment" I had while learning about DynamoDB, but especially include examples that takes real life inspired problems.

Before diving in, here are some of the resources which are a must in order to familiarize yourself on the basic of DynamoDB. For this blog-post, I will assume you're already familiar with concepts presented in those videos.

- [Single-Table Design with DynamoDB - Alex DeBrie](https://youtu.be/BnDKD_Zv0og?si=wb9Jx2o7PY5Ex3Fn)

# Include the type of item as an attribute

One of the simplest improvements you can do is add an attribute to all item which contains the type of the elements. This will save you a ton of gymnastic on the primary key, and simply allow you to switch/if/other on the type of item in order to deserialize it. For instance,

| pk     | sk        | type           |
|--------|-----------|----------------|
| USER#1 | #METADATA | User           |
| TEAM#1 | #METADATA | Team           |
| USER#1 | TEAM#1    | TeamMembership |

# Supporting multiple kind of login methods

Let's start with an example where the user uniquely identified by an arbitrary id can connect with an email, or with a username. Naively, you might create the following structure:

| pk     | sk                      |                               |                  |                               |
|--------|-------------------------|-------------------------------|------------------|-------------------------------|
| USER#1 | #METADATA               | email: "user1@example.com"    | username: "Mike" | password: bcrypt("Password!") |

Then, you can create an index with primary key set to `email`, and similarly create an other index with primary key set to `username`. This isn't a scalable design, because, you're limited to 20 Global Secondary Index by default, and what happen when you want to support more way to connect? Facebook, Google, Steam, etc.

Instead, you can structure your data in the following way:

| pk     | sk                      |                               |                  |
|--------|-------------------------|-------------------------------|------------------|
| USER#1 | #METADATA               | email: "user1@example.com"    | username: "Mike" |
| USER#1 | EMAIL#user1@example.com | password: bcrypt("Passw0rd!") | confirmed: true  |
| USER#1 | USERNAME#Mike           | password: bcrypt("Passw0rd!") |                  |

If you combine that with the so-called "inverted index", that is the primary key is `sk` and the secondary key is `pk`, you can easily login a user based on the email by simply querying the inverted index for `sk = EMAIL#{input}`. Similarly, you support querying by username by simple querying the inverted index for `sk = USERNAME#{input}`.

# Don't store data in the pk and the sk

One of the mistake I initially did was to user the pk & the sk to store some of the data. Consider the following:

| pk     | sk        |                            |                  |
|--------|-----------|----------------------------|------------------|
| USER#1 | #METADATA | email: "user1@example.com" | username: "Mike" |
| TEAM#1 | #METADATA | name: "Mike Team"          |                  |
| USER#1 | TEAM#1    | role: Admin                |                  |

Now, if I wanted to query all the teams the user 1 is in, I would query for `pk = "USER#1" and begins_with(sk, "TEAM#")`. This would give me all the team membership, and I could use `sk` to find the team id. I would simply need to strip `TEAM#`. This adds a ton of complexity and isn't worth the effort. Instead, duplicate the data in the following way.

| pk     | sk        |             |                            |                  |
|--------|-----------|-------------|----------------------------|------------------|
| USER#1 | #METADATA | userId: "1" | email: "user1@example.com" | username: "Mike" |
| TEAM#1 | #METADATA | teamId: "1" | name: "Mike Inc."          |                  |
| USER#1 | TEAM#1    | userId: "1" | teamId: "1"                | role: Admin      |

It's easy to build the pk and sk in order to do a query, but by duplicating the data, you don't need to care about parsing the format. You can throw those values out. This because especially apparent when using clever sk such as:

| pk     | sk                            | chatId | msgId | timestamp              |
|--------|-------------------------------|--------|-------|------------------------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1  | 1      | 1     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2  | 1      | 2     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3  | 1      | 3     | 2024-10-14T01:01:02.0Z |

# Don't use an inverted index

When you learn about many-to-many relationships, one of the first "aha moment" should be the introduction of the inverted index. It seems that you should always have one. Don't do that, simply set the primary key of the index to a new field `gs1pk` and the sort key to `gs1sk`.

If we take back the chat example, with the following structure:

| pk     | sk                            | chatId | msgId | timestamp              |
|--------|-------------------------------|--------|-------|------------------------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1  | 1      | 1     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2  | 1      | 2     | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3  | 1      | 3     | 2024-10-14T01:01:02.0Z |

the inverted index would be somewhat awkward to query. Instead, you could have the following (simplified):

| pk     | sk                           | gs1pk  | gs1sk  |
|--------|------------------------------|--------|--------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#1 | MSG#1  |        |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#2 | MSG#2  |        |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3 | MSG#3  |        |
| USER#1 | #METADATA                    |        |        |
| TEAM#1 | #METADATA                    |        |        |
| USER#1 | TEAM#1                       | TEAM#1 | USER#1 |

You can notice, that:

- For chat message, we have a `gs1pk`, but not a `gs1sk`, as we have no such need.
- For user and team metadata, we simply don't include them in the secondary index. Having `#METADATA` as `gs1pk` would not serve much purpose other than listing all users.
- For the team membership, we do have the typical inverted index, because it's a typical many-to-many relationship.


# Many-to-many vs many-to-many vs many-to-many

One distinction that is never done in SQL when you have a many-to-many relationship is how many item are you expecting to have? For instance, in Discord, a user can be in many server, but the number is certainly much lower than how many users can be in a single server. In this relationship, you have "some servers"-to-"a lot of clients".

This can make a pretty big difference if you need to de-normalize your data. Let's start with this very simple example.

| pk       | sk        |                  |
|----------|-----------|------------------|
| USER#1   | #METADATA | username: "John" |
| SERVER#1 | #METADATA | name: "Gaming"   |
| SERVER#1 | USER#1    | role: Member     |

You can easily list all server, well all server id, a given user is in, but that's probably not what you want. When you query all the server the user is in, you probably also want the name of the server. Similarly, querying all user id in a server won't allow you to show a nice list. You will need to follow-up by N queries in order to get the username for every users.

This problem is particularly annoying, because if you duplicate this data, i.e.,

| pk       | sk        |                  |                  |                |
|----------|-----------|------------------|------------------|----------------|
| USER#1   | #METADATA | username: "John" |                  |                |
| SERVER#1 | #METADATA | name: "Gaming"   |                  |                |
| SERVER#1 | USER#1    | role: Member     | username: "John" | name: "Gaming" |

and you modify the server name, you may need to update 100s, or 1000s, or maybe even millions of items. On the other hand, if you 1000s of user in a server, you don't want to follow up with 1000s of query to get the username. Paging may seem to help, but only partially. Once you have all the user id, you still need to query all of them individually.

This is where knowing the number of items on both sides of the many-to-many is very useful. Let's instead structure it this way:

| pk       | sk        |                  |                  |
|----------|-----------|------------------|------------------|
| USER#1   | #METADATA | username: "John" |                  |
| SERVER#1 | #METADATA | name: "Gaming"   |                  |
| SERVER#1 | USER#1    | role: Member     | username: "John" |

Now, you can with a single query list all the user in a server, including their username. If you want to query all the server a user is in, you will need to query the inverted index and this will give you all server ids. You will need to follow by an appropriate amount of queries. But, since the assumption is that a user is never in that many servers, usually 5-10, maybe sometime 50, it's not that high of a cost, especially with [BatchGetItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchGetItem.html).

If you edit the username of a user, you also need to update the server memberships, but similarly, it's a small number. Especially with [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html) or even [TransactWriteItems](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html).

On the other hand, changing the name of a server may have required updating 1000s or more items.

This is one of the consideration you should have when deciding to de-normalize your data and looking at what happen when this data is changed.

# Beware of querying multiple different type of items

One of idea I had, which I'm still not sure whether it make sense, but tend to think not nowadays is querying many different item type in a single query. Consider the following:

| pk     | sk                       |
|--------|--------------------------|
| USER#1 | #METADATA                |
| USER#1 | #EMAIL#user1@example.com |
| USER#1 | #USERNAME#Mike           |
| USER#1 | #GOOGLE#DEAD             |
| USER#1 | #META#F00D               |
| USER#1 | TEAM#1                   |

In this example, the idea would be that every item that are just an "extension" of the metadata, or exist at most 1 per user, would start with a `#` allowing to write the query `pk = USER#1 and begins_with(sk, '#')`. This would return all items, except the team membership, because a user can be in multiple teams.

Now, this might seems optimize, and probably save some places, but it's a bit of a mess to deal with from code. You end-up building up a bigger structure that merge all those fields, and especially if you have automated serializing/deserializing, it's a pain to deal with. Instead, I now prefer:

| pk     | sk                       |                          |                |              |            |
|--------|--------------------------|--------------------------|----------------|--------------|------------|
| USER#1 | #METADATA                | email: user1@example.com | username: Mike | google: DEAD | meta: F00D |
| USER#1 | #EMAIL#user1@example.com |                          |                |              |            |
| USER#1 | #USERNAME#Mike           |                          |                |              |            |
| USER#1 | #GOOGLE#DEAD             |                          |                |              |            |
| USER#1 | #META#F00D               |                          |                |              |            |
| USER#1 | ORG#1                    |                          |                |              |            |

With this setup, I can query the single item I care about, in the code, I deserialize it automatically in a struct/class containing annotations, and updating those field do require to update more 2 items, but this is really not a problem with [BatchWriteItem](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_BatchWriteItem.html) or even [TransactWriteItems](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_TransactWriteItems.html).
