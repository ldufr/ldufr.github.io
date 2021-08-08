---
layout: default
title: Leveraging multi-threading to build unreliable services
---

## The Goal
The goal of this article is to explore a pattern that I see too often and to
demonstrate why it's not optimal. The problem in question is present in several
server implementation and usually the default for those so-called high
performances servers.

My experience is different though. This approach is complex and more
inefficient than the alternative approach I will propose.

## Embarrassingly parallel problem
An embarrassingly parallel problem is a problem where little to no effort is
required to parallelize it. For instance, if you have a server that accept TCP
connections and simply send back "Hello World!", it is one where there is no
sharing and can be parallelize easily.

### Microservices is an embarrassingly parallel problem
First of all, for this article I'll define microservices as a finite list of
service types which all leverage horizontal scaling. So, any services can be
deployed on an arbitrary amount of machines to deal with more load.

This problem is embarrassingly parallel, because by building an infrastructure
that can be parallelize on multiple machines, it follows that it can be
parallelize on multiple cores trivially. In fact, it will be common to create
a amount of worker threads based on the number of core available.

I do stretch the definition of embarrassingly parallel in this case, but this
article address people that need to build a microservices. Obviously, if you
can avoid it, then avoid it!

Microservices is a pretty robust design to build highly available services that
can handle huge amount of clients. There is added complexity, because  you can
never assume that a request to an other service succeed and when it does, it
can have big delay. This forces programmer to build the infrastructures
resistant to those issues.

## Leveraging multi-threading

### async/await
Network frameworks build to do this kind of work will usually try to leverage
all threads on a machine through OS features to efficiently deal with a large
amount of sockets at the time.
* [IOCP on Windows](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports)
* [epoll on Linux](https://man7.org/linux/man-pages/man7/epoll.7.html)

FreeBSD and Mac also have equivalents features. The next step is to use the 
asynchronous operations to accept clients, read to sockets and writes to the
sockets.

You can see that with [tokio-rs](https://tokio.rs/) which is a runtime for
Rust, but the implementation are similar for a lot of different frameworks
spamming over multiple languages. Note that specifically for tokio the
underlying low level implementation is the much smaller and underrated
[mio](https://github.com/tokio-rs/mio) library.

Now, this make sense. It's even fair to assume that connection don't interact
between each others and if they did it may be difficult to ensure the different
connections are on the same machines forcing to use more complex technique to
enable such interactions. Within those assumptions, processing connections
is an embarrassingly parallel problem.

### My experience
The problem with the previous approach is that it doesn't consider that in most
cases there need to be some state that are shared between the connections. Very
simple examples of such state are service statistics and service configurations.
More complex examples may be connection to other service(s) potentially
including connections to database(s).

When this happen, you observe a diamond shaped flow where an single service
spawn N-threads to process multiple clients. Clients are processed on any of
the worker thread, but the state they need to share must be written in a thread
safe way. This forces programmers to re-implement all the thread safety
guarantee for every components that are shared which usually mean:
* Thread safe queuing. (Including async processing)
* Serialization and deserialization.
The thread safe queuing can be particularly annoying and slow when the
processing simply writes the content to an other queue. The initial work
wouldn't have been costly to do in the worker thread, but couldn't while
guaranteeing thread safety.

Moreover, when talking about connection to an other service, an other
problematic appears. For fast processing, it's not rare to see multiple
connections open to every services. For instance, database will often do that
to process requests in parallel.

## Use one "service" per thread!
Interestingly enough, if you split the concept of listener and the concept of
service, it become easy to create one service object per thread and write the
port listener to assign clients to those different services. Normally, the
infrastructure should easily this flow support this flow as we previously
explored. Indeed, if the clients need to talk to each other, the infrastructure
needs to ensure the clients are on the same service or it needs to write the
feature to so without being on the same service. Whether the service is lock
to a thread and whether there can be several services in the same process
shouldn't affect that at all.

When doing so, you guarantee that all connections within a service can never
run in parallel which make the code trivial to write, you can use freely all
resources owned by the service (caches, config, statistics, db connections,
etc.) without worrying about multithreading issues and finally you also scale
easily with the number of threads.

### How to deal with global statistics
It is very likely that you want global statistics about your infrastructure.
In the proposed solution, you can freely save them in the service object, but
this is not aggregated by the machine. That said, it's easy to once per frame
have a thread that simply "move" (C++ meaning of move) the data out of the
service in a thread-safe way. It should be extremely cheap!

### Blocking a worker thread
One argument against locking client to a service which also lock it to a worker
thread is that if any clients block the worker thread (deadlock or other) it
will impact all clients of the same service. Indeed, if the clients could be
processed on an other thread, he could fallback on it. Obviously, while it's
not ideal to build a software to be robust of deadlocks, I can appreciate the
value of it. Where there is value on both side, the decision become a
trade-offs problems and it will depend on your specific taste and experience.

My opinion is that this problem is not big enough and that in my experience,
even in the case where clients could roam on any worker threads, blocking it
for an extended amount of time was always a very big issue. Moreover, the
chance of deadlocking a far fewer, because of the reduced amount of
multithreading.

### Dealing with non-async 3rd party
I've experienced this problem with the [mongo-c-driver](https://github.com/mongodb/mongo-c-driver)
where request to the database where synchronous. Offering an asynchronous API
was asked and the reply can be found [here](https://jira.mongodb.org/browse/CDRIVER-27?focusedCommentId=1349325&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-1349325).
While I don't disagree with the analysis, the most important value of async API
is the non-blocking part of it.

Nonetheless, I think the problem is still nicer to solve with one service per
thread. Indeed, a second thread can be spawned (per service) and the
communication can be done with a single-producer-single-consumer thread-safe
queue. In fact, you would need to do the same when you allow clients to roam
between worker threads, but using a multiple-producers-single-consume
thread-safe queue.

## Conclusion

While thread has allowed us to reach new height in term of performances, I've
rarely seen, even very skillful, programmer use them well over the lifetime
of a project. On the other hand, a lot of work has been done to enable
application to be multi-threaded that can be leverage in single-threaded
applications. If your problem is embarrassingly parallel, you should be able
to write it single-threaded and spawn in N times to leverage N cores.

Try that with microservices implementation!
