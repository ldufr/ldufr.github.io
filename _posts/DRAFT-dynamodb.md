---
layout: default
title: DynamoDB tips and learning
---

# Introduction

My first experience with DynamoDB was somewhat negative. It didn't support join and doing all the necessary queries manually would incur multiple database round-trips. But, like MongoDB's meme goes, "you're using it wrong". The thing is, DynamoDB can't used incorrectly, at least, not in the same way as MongoDB. I will try to present the main "
aha moment" I had while learning about DynamoDB, but especially include examples that takes real life inspired problems.

Before diving in, here are some of the resources which are a must in order to familiarize yourself on the basic of DynamoDB. For this blog-post, I will assume you're already familiar with concepts presented in those videos.

- [Single-Table Design with DynamoDB - Alex DeBrie](https://youtu.be/BnDKD_Zv0og?si=wb9Jx2o7PY5Ex3Fn)

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
| ORG#1  | #METADATA | name: "Mike Inc."          |                  |
| USER#1 | ORG#1     | role: Admin                |                  |

Now, if I wanted to query all the org the user 1 is in, I would query for `pk = "USER#1" and begins_with(sk, "ORG#")`. This would give me all the org membership, and I could use `sk` to find the org id. I would simply need to strip `ORG#`. This adds a ton of complexity and isn't worth the effort. Instead, duplicate the data in the following way.

| pk     | sk        |             |                            |                  |
|--------|-----------|-------------|----------------------------|------------------|
| USER#1 | #METADATA | userId: "1" | email: "user1@example.com" | username: "Mike" |
| ORG#1  | #METADATA | orgId: "1"  | name: "Mike Inc."          |                  |
| USER#1 | ORG#1     | userId: "1" | orgId: "1"                 | role: Admin      |

It's easy to build the pk and sk in order to do a query, but by duplicating the data, you don't need to care about parsing the format. You can throw those values out. This because especially apparent when using clever sk such as:

| pk     | sk                            | chatId | msgId | timestamp              |
|--------|-------------------------------|--------|-------|------------------------|
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#10 | 1      | 10    | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:01.0Z#20 | 1      | 20    | 2024-10-14T01:01:01.0Z |
| CHAT#1 | MSG#2024-10-14T01:01:02.0Z#3  | 1      | 3     | 2024-10-14T01:01:02.0Z |

# Don't use an inverted index

When you learn about many-to-many relationships, one of the first "aha moment" should be the introduction of the inverted index. It seems that you should always have one. Don't do that, simply set the primary key to a new field `gs1pk` and the sort key to `gs1sk`.

When you start to have a lot of different kind of items in your table, you probably will have many for which, the inverted index is the most logical. But probably, not all of them. You can then easily structure your code such that, every item type has one required function `PrimaryKey getPrimaryKey()`, and for each global security index, an other that may return no primary key if the item doesn't have the need. I re-iterate, you don't need to store the pk/sk/gs1pk/gs1sk/etc. in your item, just build it on insertion.

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
