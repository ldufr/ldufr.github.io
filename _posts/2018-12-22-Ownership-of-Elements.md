---
layout: default
title: Ownership of Elements in C++
---

> The world would be a better place if the signature of free was
  `void free(void *ptr, size_t size)`.

One of the central problem in software design is the lifetime management of the
resources. Some language (Java, C#, etc.) chooses to have a garbage collector.
In C++, the garbage collector is not an acceptable solution and the programmers
have to solve this problem manually. It can be very simple (e.g. stack variable)
or very messy in the case of heap objects. To help with the management of those
resources, the C++ standard offer the so-called smart pointers. (i.e. shared_ptr
, unique_ptr, ...)

Andre Weissflog talks in more details about `std::shared_ptr` in an article that
you can find [here](https://floooh.github.io/2018/06/17/handles-vs-pointers.html).

In this article I will go in more details over the very common use cases of a
structure that interact with many elements that share a common interface.

### Manager & Inheritance
Let's start by defining few classes.

```cpp
struct Parent {}
struct Child1 : Parent {}
struct Child2 : Parent {}
...
struct ChildN : Parent {}

struct Manager {
    Container<Parent *> elems;
};
```

I use the term "Manager" in this context to indicate that the `Manager`
structure deal with the lifetime of the elements. Furthermore, inheritance can
be replaced with composition of structures (i.e. `struct Child1 { Parent base; }`)
if you prefer.

A more explicit example is: `Parent` is `Connection`, `Child1` is `HttpConnection`,
`Child2` is `FtpConnection`, etc.

It's important to notice that the structure `Manager` only knows about the common
interface `Parent`. It doesn't know about the different types of child, it doesn't
know the size of those types and it doesn't care. We are assuming that, `Manager`
code is compiled without dependencies to any of the child definitions. To reuse
the `Connection` example, the Manager could be a `Server` that dispatch the
messages received from the different endpoints.

Let's dive in how the memory can be handled with this design. By our previous
assumption, `Manager` doesn't have any mean to allocate or construct the elements.
This problem can be dodged by different solutions:

Solution 1:
```cpp
template<typename T, typename... Args>
bool Manager::Add(Args&&... args) {
    Parent *elem = new T(std::forward<Args>(args)...);
    if (!elem) return false;
    elems.Add(elem);
    return true;
}
```

Solution 2:
```cpp
// The ownership of `elem` is transfered to the `Manager`
void Manager::Add(Parent *elem) {
    elems.Add(elem);
}
```

In fact the second solution can be written with `std::unique_ptr` if you prefer.

```cpp
void Manager::Add(std::unique_ptr<Parent>&& elem) {
    elems.Add(elem.release());
}
```

The two solutions boils down the same thing, which is the `Manager` taking
ownership of the elements added. This create friction, because the elements
added not only need to implement the interface of `Parent`, but also need to
respect the expected lifetime setted up by the `Manager`. The `Manager` has a
lack of context about the type of the element and the consequence is more friction.

The problem in this scenario is very subtle. When using inheritance, the lifetime
of the `Child` is the same as the lifetime of the `Parent`. The `Manager` can't
"delete" the `Parent` without "deleting" the Child so it will "delete" them when
he is done with the `Parent`. The `Parent` is the structure with the less context.
The simplicity and flexibility of the `Manager` is affected by this restriction.
Moreover, it also imposes a common memory allocator that adds even more friction
and affect the performances. Indeed, the most obvious case is when the "trivial
allocator" is used. I use "trivial allocator" to describe the allocation that
happened when a structure is on the stack or composition is used.

### Utilizer
A much more flexible solution when using inheritance would be a `Utilizer`. The
`Utilizer` is a structure that doesn't take the responsibility of handling the
lifetime of the elements, but only keep a reference for usage. This can be a very
elegant solution, because in some cases, the lifetime of a structure is very
obvious from the application context.

Example:
Let's say `Manager` is `IOCompletionPort`, `Parent` is `Connection` and there is
three child classes. Namely, `AuthConnection`, `GameConnection` & `HttpConnection`.
From the `IOCompletionPort` point of view, it's not clear how to handle the
lifetime of the connections, but from a client context it's obvious. Indeed,
a client always have a single copy of each connection types.

```cpp
struct Connection {
    virtual void OnRecv(size_t buffer_id) = 0;
    virtual void OnSend(size_t buffer_id) = 0;
    virtual void OnDisconnect() = 0;

    void Connect(struct sockaddr *host);

    msec_t          t0;
    socket_t        fd;
    struct sockaddr host;
};

struct AuthConnection : Connection {
    AuthConnection();
    // Implement the interface

    // Function specific to the AuthConnection
    void HandleAuthToken();
};

struct GameConnection : Connection {
    GameConnection();
    // Implement the interface

    // Function specific to the GameConnection
    void HandleTradeRequest();
};

struct HttpConnection : Connection {
    HttpConnection(const char *host);
    // Implement the interface

    // Function specific to the HttpConnection
    void HandlePostRequest();
};

struct IOCompletionPort {
    ~IOCompletionPort() {
        // We don't delete the connection !!
    }

    void Add(Connection *c) {
        connections.Add(c);
    }

    Container<Connection *> connections;
};

struct ClientContext {
    IOCompletionPort iocp;
    AuthConnection auth_conn;
    GameConnection game_conn;
    HttpConnection http_conn("https://my_game.com");

    void Init() {
        iocp.Add(&auth_conn);
        iocp.Add(&http_conn);
    }

    void OnJoinGame() {
        iocp.Add(&game_conn);
    }
};
```

Notice that you get the allocation and the deallocation of `ClientContext` for
almost free, there is no risk for a connection allocation to fail and there
can't be any memory leaks.

### Manager
Having a `Manager` that deal with lifetime of the elements is not necessary bad.
Inheritance is not adapted to this use cases, because of the shared lifetime
between the interface and the specific object.

The `Manager` scheme can be a very well suited solution, but the implementation
needs to differ a little bit. In [nginx](https://github.com/nginx/nginx), the
`ngx_connection_t` are owned by the manager, but a module can still use it.
The only difference is that the lifetime of a `ngx_connection_t` is not shared
with the child connections. The code looks like the following:

```cpp
struct ngx_connection_t {};
struct ngx_http_connection_t {
    ngx_connection_t *base;
};
struct ngx_mail_session_t {
    ngx_connection_t *base;
};
...
struct ngx_custom_connection_t {
    ngx_connection_t *base;
};

struct Manager {
    Container<ngx_connection_t> elements;

    virtual void OnConnection(ngx_connection_t *c) = 0;
};
```

### What to use and when ?

The two approaches can be summarized to the following C-style structures.
```c
struct Parent {};
struct Type1 {
    Parent base;
};
struct Type2 {
    Parent *base;
};
```

First of all it's very important to understand that doing a `base = new Parent`
in the `Type2` constructor would still fall in the `Type1` category. Moreover,
you can define `Type1` with inheritance. (i.e. `struct Type1 : Parent {};`)

Secondly, `Type1` and `Type2` are not only different approaches. They carry
different ideas and different levels of involvement.

- `Type1`: The advantage of the first type is to not deal with the lifetime and
  the ownership of the elements. It's done by forwarding the responsibility to
  someone that has more context. Indeed, doing an `Utilizer` is generally easier,
  because you simply assume that the elements are valid as long as you use it.
  We demonstrated that this can be well suited to certain situation. The disadvantage
  of this approach is that it's much harder to make assumption over the elements
  structure. You can't move them, you can't optimize the allocation and the handling
  of the elements is decentralized. Moreover, it's much harder to enforce a valid
  state over the elements since you don't control them.

- `Type2`: In the second approach, `Parent` is an unmanaged resource from the
  user point of view, similar to an `HANDLE` in Window. In contrast, the `Type1`,
  `Parent` need not be an abstract structure. This approach is generally observed
  in mature code bases where the problem solved is well understood. Indeed, the
  main characteristics of the `Type2` is that it's not an abstract structure.
  It doesn't require any particular handling from the child to work correctly.

### Conclusion

I started the article by stating that the world would be a better place if `free`
required the size of the memory region to be freed. The reason I made this
statement was that, if you don't have enough context to allocate resources,
it's unlikely that you have enough context to understand its lifetime and hence
freeing it creates an implicit interface over an unknown object. This extra
friction affect the simplicity, the understanding, the robustness, the flexibility
and the performances of the application. I referred to the article "Handles are
the better pointers" from Andre Weissflog that goes over some problems related
to smart pointers. It particularly focuses on `std::shared_ptr` and my
understanding is that a `std::shared_ptr` is meant to share ownership of resources
which in my opinion is very good way of shooting yourself.