---
layout: post
title: "Function Name Resolution in Stack Traces"
description: ""
category: 
tags: [debugging]
---
{% include JB/setup %}

Lately I've been writing an [OS](https://www.github.com/JamesSullivan1/os)
for fun. One of the first things I wanted was a decent stack trace
routine which would help debugging any kernel panics and bugs.

Writing a stack tracer is very simple and requires only a basic
understanding of the stack layout for the architecture you care about.
For example, on x86:

    void
    backtrace(unsigned int max_frames)
    {
        unsigned int *ebp = &max_frames - 2;
        unsigned int eip;
        unsigned int fr;

        for (fr = 0;
             (fr < max_frames) && ebp;
             fr++, ebp = (unsigned int *)(*ebp))
        {
            eip = (ebp ? ebp[1] : 0);
            kprintf("0x%08x\n", eip);
        }
    }

Of course, this doesn't give the most useful output, unless you're
particularly good at converting hex strings to code addresses.  We can
certainly determine where the program was in each frame by using
something like `addr2line`, but we can save a lot of time and effort by
having the program itself determine this. In this post I'll discuss one
way in which you can do function name resolution on both hosted and
unhosted (i.e. an OS) platforms.

Our goal is to write a function

    const char *lookup_func(unsigned long addr);

Which returns a string of the form:

    0xdeadbeef (foo+0xabcd)  // On success
    ????????                 // On lookup failure

Indicating that the address `0xdeadbeef` is in function `foo` at offset
`0xabcd`.

## Prerequisites

We need a couple of things in place for this to work:

* A binary that is not stripped of symbols, and
* A means to access the binary's symbol and string tables.

Tackling the first problem boils down to how you are compiling your
application. There's really no convincing reason to strip symbols out of
your application before it is shipped, so this should not be an issue.
For a production application, you may or may not want to expose the
function names to the user, so this may be something that you want to
entirely disable in production.

The second prerequisite can be trickier. Most ELF files that are spit
out by your standard tool chain will *not* mark the segment containing
the symbol and string tables as `PT_LOAD` (i.e. the segment will not
exist in program memory), because generally most programs have no need
for this data at run time. This means that we need a way for our program
to access its own ELF data.

On a hosted application, this is made easy: We can simply open and read
the binary file that corresponds to our program. There's a handy symlink
in the `/proc` filesystem called `/proc/self/exe` which points to the
binary which the calling process is executing, and so there is no
problem here.

The tricky bit comes with an unhosted application, such as a kernel. If
a kernel gets to the point where it can initialize a filesystem and load
its own image from a file, then we can do something like the above.
However, it would be nice to have access to our debugging tool before
this point, since there are many, many things in the kernel that can go
wrong before and during filesystem initialization.

If the kernel in question is loaded by a multiboot-compliant bootloader,
then rejoice! The multiboot specification ensures that our ELF's section
headers and their data are all mapped into physical memory at known
locations. From the multiboot header:

    typedef struct elf_section_header_table
    {
        unsigned long num; /* Number of section headers */
        unsigned long size; /* Size per section header */
        unsigned long addr; /* *physical* addr of section header table */
        unsigned long shndx; /* Index of the section header string table */
    } elf_section_header_table_t;

    typedef struct multiboot_info
    {
        ...
        union
        {
            aout_symbol_table_t aout_sym;
            elf_section_header_table_t elf_sec;
        } u;
        ...
    } multiboot_info_t;

The `multiboot_info_t` struct is passed to the kernel's initialization
routines by the bootloader. From there, (assuming your kernel has
paging), we simply map the physical addresses of the ELF data to virtual
addresses and access the data as we would anything else.

## Loading the symbol and string tables

The first step in either case is to load the symbol and string tables of
the ELF. Let's assume that we have implemented the following:

    typedef struct elf {
        Elf_Shdr *shtab; // Easy access to shtab
        Elf_Phdr *phtab; // Easy access to phtab
        const char *shstrtab; // Easy access to section header names
        union {
            Elf_Ehdr ehdr;
            char     data[0]; 
        } u;
    } Elf;

    Elf *load_self_image(void);

Our first goal is to set up references to the string and symbol tables
of the ELF, since we will be looking up addresses in the symbol table
and finding their corresponding string entry in the string table. This
basically comes down to searching the section headers for sections with
the name ".symtab" and ".strtab" (which presupposes that our symbol and
string tables will actually be called this; a safe assumption for all C
compilers I am aware of).

    const char *
    section_name(Elf *e, Elf_Shdr *sh)
    {
        return e->shstrtab + sh->sh_name;
    }

    Elf_Shdr *
    find_section(Elf *e, const char *name)
    {
        unsigned i = 0;
        for (; i < e->u.ehdr.e_shnum; i++)
        {
            Elf_Shdr *sh = e->shtab + i;
            if (!strcmp(section_name(e, sh), name))
                return sh;
        }
        return NULL;
    }

    void
    foo(Elf *e)
    {
        Elf_Shdr *symtab, *strtab;
        symtab = find_section(e, ".symtab"); // Check for NULL
        strtab = find_section(e, ".strtab"); // Check for NULL
    }

## Setting up a data structure for address lookup

Okay, so we have access to the symbol and string tables of the ELF. The
naive way to proceed now is to scan the entire symbol table and return
the greatest address that is less than the target (recall that we might
be at some positive offset into the function). Of course, this is not
the most efficient way to do this, since we have to scan *every* symbol,
and since not every symbol is even a function!

There are two optimizations we can do here:

* Filter out entries in the symbol table which are not functions, and
* Sort the entries in the symbol table to accelerate the search from
  O(n) to O(lg n).

We will do both of these at the same time, creating a sorted list of
function symbols which we can easily search. For simplicity of use, we
will also have this function symbol list contain an actual pointer to
the function name, rather than an offset into the string table.

    typedef struct {
        unsigned long addr;
        const char   *name;
    } FuncSymbol;

    /* Some routines used for sorting a FuncSymbol array */

    // Returns -1 if a->addr < b->addr, 1 if a->addr > b->addr, 0 otherwise
    int func_cmp(const FuncSymbol *a, const FuncSymbol *b);
    // Swaps a, b
    void func_swap(FuncSymbol *a, FuncSymbol *b);

    FuncSymbol *
    setup_symbol_table(Elf_Shdr *symtab, Elf_Shdr *shtab, int *num_ents)
    {
        /* section_data just retrieves shdr->addr from the shdr,
           adjusting it from a physical to virtual address on an
           unhosted implementation. */
        Elf_Sym *sym_start = section_data(symtab);
        Elf_Sym *sym_end   = sym_start + symtab->sh_size;
        const char *strtab_start = section_data(strtab);
        const char *strtab_end = strtab_start + strtab->sh_size;

        unsigned long max = 1024, num = 0;
        FuncSymbol *ret = malloc(max * sizeof(FuncSymbol));
        // Check for NULL

        Elf_Sym *sym = sym_start;
        for (; sym < sym_end; sym++)
        {
            /* Skip non-function symbols */
            if (ELF_ST_TYPE(sym->st_info) != STT_FUNC)
                continue;

            /* Expand our output array as needed */
            if (num + 1 >= max) {
                void *tmp = realloc(ret, (max *= 2));
                // Check for NULL
                ret = tmp;
            }
            /* Add the symbol */
            ret[num].addr = sym->st_value;
            ret[num].name = strtab_start + sym->st_name;
            num++;
        }
        /* Now sort the list by address. We could have inserted the
           elements in sort order, but this is simpler to implement and
           is a one-time cost. */
        sort(ret, num, sizeof(FuncSymbol), func_cmp, func_swap);
        *num_ents = num;
        return ret;
    } 

Okay, great! Now we have a list of function symbols, sorted by their
address, which we can easily search through.

## Searching our data structure

Finally we get to implement the lookup routine itself. We are interested
in finding the function with the *greatest address less than the input*, 
and returning a string describing the function and an offset into it.

Since we have a sorted array, we can use a basic binary search here,
with the slight modification that we don't care about exact matches.
Here is one way to implement that.

    /* Assume that we have already set these up */
    unsigned long num_symbols;
    FuncSymbol *symbols;

    static FuncSymbol *
    find_func(unsigned long addr)
    {
        FuncSymbol *ent = NULL;
        unsigned long bot = 0;
        unsigned long top = num_symbols;

        while (bot < top)
        {
            unsigned long ind = (top + bot / 2);
            ent = symbols + ind;

            if (addr < ent->addr) {
                /* Search the lower half */
                top = ind;
            } else if (addr > ent->addr) {
                /* Check to see if we're done. There are two cases:
                    1) The entry is the biggest in the list, or
                    2) The next entry up is after the input address */
                if ((ind == num_symbols - 1) ||
                        (symbols[ind+1].addr > addr)) {
                    break;
                } else {
                    /* Search top half */
                    bot = ind;
                }
            } else {
                /* Exact matches are OK too! */
                break;
            }
        }
        return (bot < top ? ent : NULL);
    }

    /* Reusable return buffer */
    char buf[256];

    const char *
    lookup_func(unsigned long addr)
    {
        FuncSymbol *p = find_func(addr);
        if (!p)
            return "????????"; 
        snprintf(buf, sizeof(buf)-1, "0x%08x (%s+0x%x)", addr, p->name,
                 addr - p->addr);
        buf[255] = '\0';
        return (const char *)buf;
    }

## The end product

Once we've got the above written, all we need to do is drop in our
function into our stack tracer to get some lovely, useful traces.

    void
    new_backtrace(unsigned int max_frames)
    {
        unsigned int *ebp = &max_frames - 2;
        unsigned int eip;
        unsigned int fr;

        for (fr = 0;
             (fr < max_frames) && ebp;
             fr++, ebp = (unsigned int *)(*ebp))
        {
            eip = (ebp ? ebp[1] : 0);
            kprintf("%s\n", lookup_func(eip));
        }
    }


    0xdeadbeef (foo+0x12)
    0xfeedbad0 (bar+0x40)
    0x00000000 (????????)
