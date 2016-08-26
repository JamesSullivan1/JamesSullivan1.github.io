---
layout: post
title: "typedefs and type safety"
description: ""
tags: [C, programming]
---
{% include JB/setup %}

*tl;dr: typedefs on primitives don't give you type safety. Wrap your
primitive in a struct, and typedef that!*

I just spend the last few hours tearing my hair out over a bug in my
kernel's memory allocator. For whatever reason, the memory that I was
getting back from the allocator was mapped as readable only, no matter
what flags I passed in!

This strange bug popped up during a major refactoring of my code. During
this refactoring, I made the completely arbitrary decision to change a
method's parameter ordering; `slab_getpages(flags, page_order)` became
`slab_getpages(page_order, flags)`.

For context, this particular method is to allocate a set of
`1<<page_order` pages and map them into linear memory with the given
flags. You might already see where this is going.

Turns out that I forgot to change one of the invocations of
`slab_getpages`, and so I was making a call with utter nonsense. Give me
`1<<$SOME_RANDOM_BITMASK` pages and map them with `$RANDOM_INTEGER`
flags, please! Simple mistake, simple fix, but it was surprisingly hard
to find out where I made the mistake.

Why was it so easy to make this mistake? Well, because I made the
mistake of using typedefs for type safety, and I want the world to know
about this and stop using typedefs like this!

For the unfamiliar, the `typedef` keyword allows the programmer to
specify an alias for another data type. For example, you can put a
`typedef unsigned int uint;` in your source and can henceforth just type
`uint` instead of `unsigned int`.

This is useful for a few reasons. For example, it's nice to be able to
type `foo` instead of `struct foo` for every `foo` you have (`typedef
struct foo foo;` will do the trick). Typedefs are also used by C
libraries extensively, giving us convenient names such as `uint64_t`
that we can be sure to have some behaviour or property (here, being
unsigned and 64 bits wide).

What typedefs cannot do is enforce type safety, because *all typedefs
are is a text alias*. When you declare `typedef unsigned int flags_t;`
Your compiler does not know the difference between `flags_t` and
`unsigned int`. So while your typedef may *visually* make it clear that
`flag_t` variables are distinct from `unsigned ints`, your compiler will
happily use the two interchangeably.

What's the solution? The best way to handle this is to limit your
typedefs to *structs only* (except in a few obvious cases, like if
you're porting `stdint.h` and need a new `uint32_t`).

    typedef unsigned int flags_t; // BAD BAD BAD
    typedef struct { unsigned int flags; } flags_t; // GOOD!

Sure, you have to do a bit of extra work when you want to change flags,
since you have to access the internal variable. But it's absolutely
worth it because your compiler can now tell you when you have
accidentally switched your `flag_t` and your `unsigned int`.

I hope that after reading this, you won't waste hours of your life on
silly things like this. typedef is a useful tool, but know its
limitations before applying it!
