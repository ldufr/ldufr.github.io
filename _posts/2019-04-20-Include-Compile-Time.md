---
layout: default
title: #include <compile_time.h>
---
The increasing use of templates, increasing numbers of translation units
and increasing amount of code in headers has tremendously affected the
compile-time of C++ code bases. Indeed, it's hard to justify the compile-time
that we typically have in C++ when comparing with code bases of similar
size in languages such as C# or Go. This article has for objective to
explore the reasons that cause slower compile-time in C++ and with that
understanding introduce trade-off that can improve the quality and compile
time of a C or C++ code bases. I won't discuss about techniques such as
pre-compiled headers, but they can be added to further improve compile-time.

### Translation units
The C and C++ language is typically compiled by compiling translation units
(producing object files) that are later assembled (linked) by the linker to
create an executable binary. Moreover, the compilation of the translation
units is done independently of each others. For instance, you can
independently compile every translation units (with as many command as
translation unit) and then invoke the linker to produce the executable.
See compiler flag `-c` on gcc or clang) The translation unit generally has
a one-to-one relationship with a source file (i.e. `.c`, `.cc`, `.cpp`), but the standard doesn't require it. For instance, the [unity build programming
technique](https://en.wikipedia.org/wiki/Single_Compilation_Unit) doesn't has this one-to-one relationship.

This concept of translation units had and has several benefits, but comes
with drawbacks. In my opinion, the main drawback is that it doesn't work
as most people expect. For this reason, it's not used efficiently by most
programmers. Indeed, `#include` isn't an inclusion of dependency, but an
inclusion of text. One way to illustrate that is with the following code
which compile correctly.

```c
// header.h
void print()
{
printf("Te"

// main.c
#include <stdio.h>
#include "test.h"
"st\n");
}
int main(void)
{
    print();
    return 0;
}
```
This last example also imply that an header generally can't be pre-compiled
independently of the translation units in which it will be included.

Example that illustrate this independence:
```c
// header.h
struct foo {
#ifdef DEBUG
    const char *name;
#endif
    int number;
};

void print(struct foo *f);

// file.c (clang -c file.c -DDEBUG)
#include <stdio.h>
#include "header.h"
void print(struct foo *f)
{
#ifdef DEBUG
    printf("[%s] %d\n", f->name, f->number);
#else
    printf("%d\n", f->number);
#endif
}

// main.c (clang -c file.c)
#include "header.h"

int main(void)
{
    struct foo f = {10};
    print(&f);
    return 0;
}
```

When the following command are used to compile, the program will very likely
segfault, because the memory layout of `struct foo` isn't the same in the
two different translation units.

Command used to compiles the executable:
```c
clang -c main.c
clang -c file.c -DDEBUG
clang -o main main.o file.o
./main
```

It is not surprising, because the code in the header only make sense in
the context of a translation units, because `#include` is an inclusion
of text, not an inclusion of dependency. The header "vector" of Microsoft
libc++ change the layout of `std::vector` depending if `_SECURE_SCL` is
defined and that can cause similar problems when crossing the boundaries
of two translation units.

### Include Guards
[Learn C the Hard Way](https://learncodethehardway.org/), a reference in
C programming tutorial, introduce their first header, dbg.h, with a series
of logging macros. Those macros uses `fprintf`, `errno` and `strerror`, so
dbg.h includes stdio.h, errno.h and string.h. Additionally, this is explained
with the sentence "Includes for the functions that these macros need".

The approach of including dependencies of the header in the header is
the dominating practice in C and C++. This lead to an obvious problem,
recursive and multiple inclusion, but it's easily circumvented with [include
guards](https://en.wikipedia.org/wiki/Include_guard). Moreover, C and C++
compilers can generally not assume that an including header is [idempotent]
(https://en.wikipedia.org/wiki/Idempotence) and the following example
illustrate that.
```
// file1.h
#ifndef FILE1_H
# define FILE1_H
# pragma message("file1.h included")
#else
# undef FILE1_H
#endif
```

This could be an important factor in slow compilation time, because
compilers would need to re-interpret the header in the context of where
it's included, every time it's included. [#pragma once](
https://en.wikipedia.org/wiki/Pragma_once) was a tentative to simplify
this problem, but today the difference in compile time between guarding
with pragma once and guarding with classic include guards is negligible.
Indeed, compilers may optimize the include guards if it semantically
equivalent to re-interpreting the header in the new context every time.
GCC and Clang have shown that those optimizations are possible.

The typical include guard is the scheme 1 in the following snippet. I will
also present a different kind of guard, that will be further explained.

Scheme 1: (Classic Include Guards)
```c
// file1.h
#ifndef FILE_1_H
#define FILE_1_H
#include <stdio.h>
#include <string.h>

#define log_error(msg) fprintf(stderr, msg)
#endif

// main.c
#include "file1.h"
// ...
```

Scheme 2: (Single depth inclusion)
```c
// file1.h
#ifdef FILE_1_H
#error "file1.h included more than once"
#endif
#define FILE_1_H

#define log_error(msg) fprintf(stderr, msg)

// main.c
#include <stdio.h>
#include <string.h>
#include "file1.h"
// ...
```

### Header's dependencies
Before going over the advantages and drawbacks of the two include guard
schemes, I will formalize how I will use the word "dependency" from now
on. Indeed, in C and C++, headers don't have dependency in itself. The
translation unit will have dependencies and adding an header may creates
more of them. Moreover, if you are not using all functionality of the
header some dependency may only need a declaration. For instance, the
following code will compile, but if you were to use the function `func_bar`
you would need the definition of `struct bar`.
```c
// file1.h
struct foo;
struct bar;

void func_foo(struct foo *f);
struct bar func_bar(void);
// main.c
#include <stdio.h>
#include "file1.h"
int main(void)
{
    printf("Hello World!\n");
    return 0;
}
```

In this case we could say that the declaration of `struct foo` and
`struct bar` are dependencies when including file1.h, but their
definitions are not. For instance, adding the following code to file1.h,
would not compile, because in the context of the translation unit
`sizeof(struct foo)` is unknown.
```c
static void print(void)
{
    printf("%d\n", sizeof(struct foo));
}
```

With that philosophy we can define (informally) a more granular definition
of dependency.
1. A macro definition creates no dependency.
2. A pointer to a type `T` creates a dependency on the declaration of `T`.
3. A structure definition creates dependencies on the definition of all
the nested types.
4. A function declaration creates dependencies on the declaration of all
the types involved.
5. Using a type creates a dependency on the definition of the type.
6. Using a macro recursively creates dependencies on the generated code.

We can apply this definition of dependency to analyze the header "dbg.h"
from Learn C The Hard Way. If you were to use the `debug` macro which use
`fprintf` and `stderr`, you would create a dependency on the declaration
of `fprintf` and the declaration of `stderr`.

### Scheme 1 vs Scheme 2
The method used in the first scheme creates a lot of extra works for the
compiler. The compiler will generally include all levels of dependencies
for a given headers. In returns, a user can include the an header and have
a guarantee that all functionalities are completely accessible. This method
also creates a fairly random include order that can be hard to control,
especially when the header offers "#define" to customize its functionalities.
(e.g. `WIN32_LEAN_MEAN`, `NOMINMAX`, ...)

The second scheme offer the most granular control. That is, an header
doesn't includes any other headers. It forces you to manually include
the dependencies in every translation units. This can be quite interesting,
because by looking at the source file, it's very easy to understand the
include order and see design flaws. Typically, it will forces you to create
a oriented tree of dependency. In practice, you will group your header in
**dependency groups** where "Group 0" is the least dependent and that
increment. Header, should never creates dependency on the same level or
on higher level groups. This means, that the include order doesn't mater
within a group, but groups should be included in ascending order.
```c
// Group 0
#include <stddef.h> // no dependency
#include <stdint.h> // no dependency

// Group 1
#include <MyProgram/array.h> // needs size_t (stddef.h)
#include <MyProgram/string.h> // needs uint8_t (stdint.h)

// Group 2
#include <MyProgram/path.h> // needs "MyProgram/string.h"
#include <MyProgram/unicode.h> // needs "MyProgram/string.h"

// Group 3
#include <MyProgram/file.h> // needs "MyProgram/path.h" & "MyProgram/unicode.h"
```

### .h = .def + .inl + .decl
This section is particularly relevant to C++, but can be applied to C.
The rule 2 & 4 of dependency were about creating dependency on declaration.
Declaring used type in the header (e.g. forward declaration) can avoid
includes and this can tremendously improve the compile time. In C, it's
generally easy, because in most cases declaring a type `T` will be writing
`struct T;` at the top of the header. In C++ this isn't necessary the case.
For instance, the following code was taken from a [json](
https://github.com/nlohmann/json) library.
```cpp
template <
    template<typename U, typename V, typename... Args> class ObjectType = std::map,
    template<typename U, typename... Args> class ArrayType = std::vector,
    class StringType = std::string,
    class BooleanType = bool,
    class NumberIntegerType = std::int64_t,
    class NumberUnsignedType = std::uint64_t,
    class NumberFloatType = double,
    template<typename U> class AllocatorType = std::allocator,
    template<typename T, typename SFINAE = void> class JSONSerializer = adl_serializer
    >
class basic_json
{
    // ...
};

using json = basic_json<>;
```

Now assume that you have a class Foo (header foo.h) with a private
helper method which has a parameter of type `json`. This function needs
to be declared in the header with the class definition so, from rule 4
of dependency, foo.h creates a dependency on the declaration of `json`.
But, you can't forward declare a typedef so you will need to forward
declare `basic_json` and write the typedef yourself. This is not practical,
because the declaration of `basic_json` is very convoluted and may
changes with updates. Moreover, it creates dependency on the declaration
of `std::map`, `std::vector`, etc. which have the same problem.
Practically, "json.hpp" will be included before every include of "foo.h".
Similarly, \<vector>, \<map>, etc. are included and not forward declared.
This header has 14k lines of highly templated code that includes
\<algorithm>, \<array>, \<cassert>, \<ciso646>, \<clocale>, \<cmath>,
\<cstddef>, \<cstdint>, \<cstdlib>, \<cstring>, \<forward_list>,
\<functional>, \<initializer_list>, \<iostream>, \<iterator>, \<limits>,
\<locale>, \<map>, \<memory>, \<numeric>, \<sstream>, \<string>,
\<type_traits>, \<utility> and \<vector>.

Compiling a empty program with `clang version 6.0.0 (tags/RELEASE_600/final)`
take me 0 second. Including this header (without using it) took 0.75 seconds
on the first try and stabilized at 0.525 seconds. Of course, this example is
extreme, but libraries such as libc++ and boost are in this extreme case.

I compared those compile time with [stb.h](
https://github.com/nothings/stb/blob/master/stb.h) which is a so-called
"single-file library". This library offer the option to declare or/and
to define the external symbols. "stb.h" always declares the symbols and
defines them if `STB_DEFINE` is defined. In addition, it's a file of 14.5k
lines of code. With the same conditions as "json.h" the first compilation
with `STB_DEFINE` took 0.5 seconds and it stabilized at 0.23 seconds.
Without `STB_DEFINE`, it compiled in 0.0938 seconds and stabilize at
0.0938 seconds. This header includes \<intrin.h>, \<stdlib.h>, \<stdio.h>,
\<string.h>, \<time.h> and if `STB_DEFINE` is defined it includes
\<assert.h>, \<stdarg.h>, \<stddef.h>, \<ctype.h>, \<math.h>, \<io.h>,
\<direct.h>, \<stdlib.h>, \<string.h>, \<malloc.h>, \<io.h> and
\<process.h> as well.

This problem could be fixed, but it would requires to split the headers
in three different files, namely the declarations (.decl), the definition
(.def) and the inlined functions (.inl). The ".inl" depends on the ".def"
which depends on the ".decl". In practice, ".def" doesn't depend on ".decl",
because a definition is also a declaration in C and C++.

### Amalgamation of Sources
Let's start by looking at the compilation time of [SQLite](
https://www.sqlite.org/), more precisely the compilation time of the
sqlite-amalgamation version 3.27.1. This version concatenate all the
code in a single file of 221 850 lines (137 320 lines of code). With
clang version 6.0.0, the first compilation took 2.2969 seconds and then
stabilized at ~2.1 seconds. I use this example to show that C and C++
compilers can process an important amount of code in a short amount of
time. This example, is ideal for compile time, because there is no
redundant work by the compiler. The reason for that, is the single
translation unit. I emphasize the "single translation unit" over the
"single source file", because one could split the code in several source
files that would then be compiled in the same translation unit. Indeed,
nothing forbid you the include a source file and there is no reason to
believe it's bad. In fact, I will elaborate over why, when and how you
may want to include different source files in the same translation unit.

Let's remind ourself, that the goal of this article is to explore how we
can improve our iterating time by improving the compilation speed of C
and C++ projects. Although, the numbers I have computed don't follow a
very scientific approach, they are only their to give a rough estimate
of the problem. Indeed, when a project of less than 250k lines takes more
than a minute to compile, it's obvious that the potential iterating time
is vastly affected. Extrapolating from sqlite-amalgamation 3.27.1, you
could compile 6.3 million lines (linear scaling) and 1.336 million lines
(quadratic scaling) in 60 seconds.

Moreover, having different translation units will necessary impact the
compile time of a "rebuild". Indeed, every translation unit will need
to go over all the included headers and interpret them in the context
of the translation unit. In returns, having different translation unit
allow more parallelism and partial recompilation. So, the first question
to ask yourself is how mature your project is. This question can be
answered by looking at how often a large portion of the project is
recompiled. Typically, how often are there modifications to public headers.

Finally, to reduce the friction if you ever need to split a translation
unit in different translation units I recommend to include the sources
starting with the highest group and going down. e.g.
```c
// header includes
#include <Group0/file1.h>
#include <Group0/file2.h>
#include <Group1/file3.h>
#include <Group1/file4.h>
#include <Group2/file5.h>

// source includes
#include <Group2/file5.c>
#include <Group1/file4.c>
#include <Group1/file3.c>
#include <Group0/file2.c>
#include <Group0/file1.c>
```
Theoretically speaking, the include order of the sources shouldn't mater,
because the sources should only rely on the api defined in the headers.
Practically speaking, a source could rely on internal api defined in an
other source file included before. But, this include order is useful,
because it forces higher dependency group to only rely on the content of
the headers. Moreover, tests can be done by removing source includes from
top to down while ensuring that the code still compile at every step.

I will finish this section by adding that you don't need to restrict
yourself to a single amalgamation. Indeed, you can compile different
amalgamation and this is particularly relevant when several project
works together. Interestingly enough, that was the original purpose of
a translation units. It was lost with time, especially when the Java OOP
started to impose itself with the one-class-one-file principle.

### Conclusion
In conclusion, I think one needs to be able to combined those different methods in a project if compile time is a problem. Forward declaring object
is a very easy option for several structs or classes. A project that is
very well separated can compile few translation units (e.g. network.cpp,
graphic.cpp) that includes the relevant source files themselves. Finally,
I really recommend trying to include files by dependency groups to expose
the design flaws and potentially improves compile-time.
