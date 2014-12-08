---
layout: post
title: "ELF Surgery, Pt. 1: Scrubs On"
description: ""
category: ELF
tags: [elf, RE, reverse engineering, exploitation, linux]
---
{% include JB/setup %}

In this series, we develop a technique to reliably modify an ELF
executable, inserting a new executable section into it and transferring
control to the section. The last post detailed the ELF format, and we
can see that there is significant, rich structure in this format. If we
want our slice-and-diced ELF to be a good imposter, we need to have a
clear understanding of this structure. In this post, we'll go over the
dirty details of dissecting an ELF.

<!--break-->

### The Plan

Let's recap what the plan was, from last time.

1. Find a suitable location to insert the new section, preferably within
the existing LOAD segment. I find that the best place for this is
directly before the '.text' section, but after the '.plt' section (more
on this later).
2. Fix up the section header table to make room for this new section,
and add our new entry. This mostly involves incrementing offsets (we
may also have to re-align). There's a bit of a gotcha here, which I'll
come back to later.
3. Fix up and extend the segment descriptors so that the original
mappings are preserved, but our new section is in a LOAD segment.
This is also done by incrementing offsets and sizes, but again we need
to be cautious of alignment.
4. Inject the new section's data into its location in the buffer.
Optionally, inject an instruction at the end of the payload which
transfers control to the 'right' place; this makes your payload more
covert if it is desirable.
5. Fix up the dynamic linking and symbolic entries references to account
for the new offsets. The same must also be done for offset tables, which
are unfortunately hard-coded with virtual offsets.

At a high level, this doesn't look to be terribly challenging- and the
truth is, it's not. The hard part of this task is the book-keeping- the
ELF has a lot of expectations as to where the bytes are and what the
bytes say, so we have to consider almost every section of the ELF in
isolation to ensure the correctness.

### 1: Find the Injection Point

One absolute requirement of this task is that your new section must be
in a LOAD segment, if you want to execute its contents. This is for a
fairly simple reason- if it's not a LOAD segment, it won't be mapped
into the virtual memory. 

The simplest way to go about this is to extend the segment that the
program text lives in. This includes 'init', 'plt', and, of course,
'text'. For reliability reasons, the best place to stick your code in is
right where '.text' starts. Any executable binary has this section, and
so you know for a fact that it's somewhere in the ELF. The 'plt'
section, on the other hand, may be omitted. 

The PLT, or the Procedure Linkage Table, is nothing but a table of
stubs at a known location. These stubs direct control to an entry in the
GOT (Global Offset Table), which in turn transfers control to some
corresponding shared library function. This double-indirection accounts
for the ability to load a shared object almost anywhere in memory,
giving us a fixed location to call no matter where the actual function
is. We can dynamically fix up the GOT to point to the actual function's
location, much easier than fixing up every call to the function.

However, the PLT is only necessary for ELFs which do dynamic relocation,
and so in the general case it's best to not assume that there is such a
section.

### 2: Fix up the Section Header Table




