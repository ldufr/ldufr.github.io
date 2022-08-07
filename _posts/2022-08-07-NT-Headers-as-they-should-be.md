---
layout: default
title: NT Headers as they should be
---

PE Headers parsing is a common practice with a large set of applications. The
format is fairly complex and has a lot of sharp corners which can make it a
risky endeavors when parsing potentially malicious files. In fact, Microsoft,
the creator of the format and an company with large resources keep having
issue with the parsing of it, both in the Kernel space and in the User space.

I'm unfortunately not going to give you a solution for that, probably because
there is no easy solutions, but during my time parsing the format, I always
found the definition of the headers rather strange. The goal of this short
article is to present a slightly different way to think about the NT headers,
which, I think, make much more sense.

The definition of the NT headers found in `winnt.h` build "0073" is
the following.
```c
typedef struct _IMAGE_NT_HEADERS {
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER32 OptionalHeader;
} IMAGE_NT_HEADERS32, *PIMAGE_NT_HEADERS32;

typedef struct _IMAGE_NT_HEADERS64 {
    DWORD Signature;
    IMAGE_FILE_HEADER FileHeader;
    IMAGE_OPTIONAL_HEADER64 OptionalHeader;
} IMAGE_NT_HEADERS64, *PIMAGE_NT_HEADERS64;
```

Regarding the signature, nothing special. You first read the first 4 bytes
and check if it matches an hard-coded value. Of course, the structure only
make sense if the first 4 bytes are matching a value, so the value is not
necessary, and first reading the structure then checking for the signature
can create worst error message. Regardless, this is a common approach and
nothing much to say here.

The second observation is that there is two different structures `IMAGE_NT_HEADERS`
and `IMAGE_NT_HEADERS64` for 32 bits and 64 bits resp. Yet, the definition of
which one to used is in `IMAGE_FILE_HEADER` (i.e., `Machine`) which is in fact
exactly the same layout regardless of the architecture. This make it both very
easy and very common to first get a variable of a type, which may be incorrect,
and then hopefully not forgetting to check the values.

In addition. The size of `IMAGE_OPTIONAL_HEADER(32|64)` is not generally known.
It's also specified through a field in `IMAGE_FILE_HEADER`, (i.e., SizeOfOptionalHeader)
which you need to consider to be forward compatible and which could be smaller
than the definition you have. You can of course refuse to deal with optional
headers smaller than your definition, but it's not invalid. Ideally, your
application would distinguish between invalid and unsupported file.

So, I found re-defining the types can make it drastically easier to reason
about what the format actually is, and arguably less error prone to parse it.
The definition is as follow:
```c
// Remove IMAGE_NT_HEADERS and IMAGE_NT_HEADERS64 definitions.

typedef struct _IMAGE_FILE_HEADER {
    DWORD   Signature;
    WORD    Machine;
    WORD    NumberOfSections;
    DWORD   TimeDateStamp;
    DWORD   PointerToSymbolTable;
    DWORD   NumberOfSymbols;
    WORD    SizeOfOptionalHeader;
    WORD    Characteristics;
} IMAGE_FILE_HEADER, *PIMAGE_FILE_HEADER;

typedef struct _IMAGE_OPTIONAL_HEADER32 {
    // Doesn't change
} IMAGE_OPTIONAL_HEADER32, *PIMAGE_OPTIONAL_HEADER32;

typedef struct _IMAGE_OPTIONAL_HEADER64 {
    // Doesn't change
} IMAGE_OPTIONAL_HEADER64, *PIMAGE_OPTIONAL_HEADER64;
```

To parse the NT headers, you simply:
- Do I have enough bytes for `IMAGE_FILE_HEADER`? If not, it's not a valid NT
  file, you can exit.
- Does the `Signature` match? If not, it's not a valid NT file, you can exit.
- Check which optional header should be used, using `Machine`.
- Check the `SizeOfOptionalHeader` match what you expect. The easy way to deal
  with it is to simply check that's at least the size of your optional header
  structure. If it's not, you can exit with "unsupported file".

The comment I did about the `Signature` field in the structure also apply here,
which means you could optionally remove the field from `IMAGE_FILE_HEADER` and
you would be back to the original definitions. Well, without `IMAGE_NT_HEADERS`
and `IMAGE_NT_HEADERS64` anymore which is the main win for the sanity of the
definitions.

While the re-definition won't fix the parsing, since the most difficult bugs
are not really about reading the headers, I do believe the definition I gave
are drastically more sane.

If there is one thing to learn from that is, don't make the type of your
structure depend on a field inside the structure.
