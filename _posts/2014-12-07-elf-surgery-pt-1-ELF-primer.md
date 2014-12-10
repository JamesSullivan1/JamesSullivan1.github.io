---
layout: post
title: "ELF Surgery, Pt. 1: A Primer on ELF"
description: ""
category: ELF
tags: [ELF, reverse engineering, exploitation, linux]
---
{% include JB/setup %}

Executable file formats (such as ELF or PE) can seem dauntingly opaque
at a first glance. It's easy to assume that these files are an
unintelligible sequence of bytes which can be understood only by
complicated linking and loading software. The reality is that the ELF
format is a well-designed and well-defined structure, and its rich
structure is far from undecipherable. 

<!--break-->

Earlier this year, I had led myself to believe that executable formats
were undecipherable black boxes of machine code. The thought of
modifying them while keeping their format as valid was a pipe dream. Of
course, this is far from the truth- and so to prove myself wrong, I
created a program that did exactly this.
[inject](https://github.com/JamesSullivan1/inject) is a program that
will (mostly) reliably inject a new section into an ELF and redirect
control to this new section's payload. This careful modification leaves
the ELF in an otherwise pristine state, and can transfer control back to
the original entry point of the ELF for covert operation. In this post I
outline how this was accomplished, and some of the inherent challenges
with modifying a complex binary artifact.

### A quick primer on ELF

Before we start, it's useful to have a reference for the ELF format.
ELF stands for Executable and Linkable Format, and is the lingua franca
of UNIX executables. A number of other executable formats exist and are
supported by UNIX-like systems, but for the most part, all programs are
ELFs. The format is quite beautiful, and for the most part well
documented. The ELF is also unique in its flexibility- there are many
ways to put together an ELF. This diversity, in fact, is a challenge to
overcome for a task such as this.

The ELF consists of a number of headers, which records the metadata of
the file. The ELF Header describe the file itself, and is always the
first set of bytes in an ELF. The Program Header Table usually follows
closely after this, and documents the way in which the ELF should be
loaded into memory. The actual bytes of the ELF follow this table, and
at the end (or close to it), is the Section header, which describes the
familiar sections of the ELF such as `.data` and `.text`. 

#### The ELF header

    #define EI_NIDENT 16
     
    typedef struct {
        unsigned char e_ident[EI_NIDENT]; /* \x7fELF, bits, OS, etc. */
        uint16_t      e_type;    /* Executable/Reloadable/Core/Dyn */
        uint16_t      e_machine; /* EM_386, for example */
        uint32_t      e_version; /* CURRENT or INVALID */
        ElfN_Addr     e_entry;   /* Virtual Entry Point to executable */
        ElfN_Off      e_phoff;   /* Program Header Table offset */
        ElfN_Off      e_shoff;   /* Section header Table offset */
        uint32_t      e_flags;   /* Not currently used or checked */
        uint16_t      e_ehsize;  /* Header size */
        uint16_t      e_phentsize; /* Size of a program header entry */
        uint16_t      e_phnum;     /* Number of program headers */
        uint16_t      e_shentsize; /* Size of a section header entry */
        uint16_t      e_shnum;     /* Number of section headers */
        uint16_t      e_shstrndx; /* Index of the shdr string table */
    } ElfN_Ehdr;

This relatively simple data structure describes the metadata of the ELF,
including its type, bits, and some other identification. More pertinent
for our task is the descriptors of the program and section header
tables- we'll come back to this later, but for now note that this is
where all of the metadata for these tables (their location and their
size) is held.

#### Program Headers

    typedef struct {
        uint32_t   p_type;   /* A number of pre-defined segment types */
        Elf32_Off  p_offset; /* Offset in the file of the segment */
        Elf32_Addr p_vaddr;  /* Where in memory the segment will go */ 
        Elf32_Addr p_paddr;  /* Used for systems which use phy addressing */
        uint32_t   p_filesz; /* The size in the file */
        uint32_t   p_memsz;  /* The size in memory */
        uint32_t   p_flags;  /* Permissions of the segment (RWX) */
        uint32_t   p_align;  /* The alignment in file and memory */
    } Elf32_Phdr;

Each program header is a description of a segment, a region of the file
that contains one or more sections. Each program header describes how
its segment's range of bytes in the file ought to be mapped into memory
when we load the executable. Generally there is a 1-1 mapping of the
byte offset of the file to the byte offset in memory, but this is not
always the case. Some segments need not be loaded into memory at all- in
fact we only load segments of type PT\_LOAD into memory. Of course,
we'll also need to pay attention to the other segment types- for
instance, PT\_DYNAMIC, which is used to specify a region that will
affect the dynamic linking of the program.

#### Section Headers

    typedef struct {
        uint32_t   sh_name;      /* Index into a string table */
        uint32_t   sh_type;      /* Pre-defined section types */
        uint32_t   sh_flags;     /* Some section attributes */
        Elf32_Addr sh_addr;      /* Address of the section in memory */
        Elf32_Off  sh_offset;    /* Offset of the section in file */
        uint32_t   sh_size;      /* Size of the section */
        uint32_t   sh_link;      /* More on this later */
        uint32_t   sh_info;      /* Type specific metadata */
        uint32_t   sh_addralign; /* Alignment in memory */
        uint32_t   sh_entsize;   /* Some sections are tables; this is 
                                    the size of their entries */
    } Elf32_Shdr;

Section headers describe (surprisingly enough) the sections of the ELF
file, which are perhaps the most familiar artifacts of an ELF. Sections
collectively contain every last byte of data in the ELF (excluding the
headers), and the section that a byte lives in describes its semantics.
Most modern ELF files have upwards of 30 sections, each of which has
slightly unique properties. The most familiar sections include `.text`
which contains executable instructions, and `.data` which contains
initialized global data.

Here is an example of a simple ELF file- the 64-bit version of `true`, a
program which only returns 0. We capture this output with the extremely
useful `readelf` utility.

This is the ELF Header at the start of the file, describing some basic
metadata about the ELF.

    ELF Header:
      Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
      Class:                             ELF64
      Data:                              2's complement, little endian
      Version:                           1 (current)
      OS/ABI:                            UNIX - System V
      ABI Version:                       0
      Type:                              EXEC (Executable file)
      Machine:                           Advanced Micro Devices X86-64
      Version:                           0x1
      Entry point address:               0x401402
      Start of program headers:          64 (bytes into file)
      Start of section headers:          25440 (bytes into file)
      Flags:                             0x0
      Size of this header:               64 (bytes)
      Size of program headers:           56 (bytes)
      Number of program headers:         9
      Size of section headers:           64 (bytes)
      Number of section headers:         28
      Section header string table index: 27

After this, we find the section header table. You can spot a few
recognizable sections, but a surprising number of them are completely
foreign. In reality, we mostly care about a select few of these for our
task (and a number of them have almost no function), but will need to
consider almost all of them separately in order to make a 'clean'
modification.

    Section Headers:
      [Nr] Name              Type             Address           Offset
           Size              EntSize          Flags  Link  Info  Align
      [ 0]                   NULL             0000000000000000  00000000
           0000000000000000  0000000000000000           0     0     0
      [ 1] .interp           PROGBITS         0000000000400238  00000238
           000000000000001c  0000000000000000   A       0     0     1
      [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
           0000000000000020  0000000000000000   A       0     0     4
      [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
           0000000000000024  0000000000000000   A       0     0     4
      [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
           000000000000005c  0000000000000000   A       5     0     8
      [ 5] .dynsym           DYNSYM           00000000004002f8  000002f8
           0000000000000528  0000000000000018   A       6     1     8
      [ 6] .dynstr           STRTAB           0000000000400820  00000820
           000000000000024a  0000000000000000   A       0     0     1
      [ 7] .gnu.version      VERSYM           0000000000400a6a  00000a6a
           000000000000006e  0000000000000002   A       5     0     2
      [ 8] .gnu.version_r    VERNEED          0000000000400ad8  00000ad8
           0000000000000060  0000000000000000   A       6     1     8
      [ 9] .rela.dyn         RELA             0000000000400b38  00000b38
           0000000000000078  0000000000000018   A       5     0     8
      [10] .rela.plt         RELA             0000000000400bb0  00000bb0
           0000000000000480  0000000000000018   A       5    12     8
      [11] .init             PROGBITS         0000000000401030  00001030
           000000000000001a  0000000000000000  AX       0     0     4
      [12] .plt              PROGBITS         0000000000401050  00001050
           0000000000000310  0000000000000010  AX       0     0     16
      [13] .text             PROGBITS         0000000000401360  00001360
           0000000000002a39  0000000000000000  AX       0     0     16
      [14] .fini             PROGBITS         0000000000403d9c  00003d9c
           0000000000000009  0000000000000000  AX       0     0     4
      [15] .rodata           PROGBITS         0000000000403dc0  00003dc0
           0000000000000c09  0000000000000000   A       0     0     64
      [16] .eh_frame_hdr     PROGBITS         00000000004049cc  000049cc
           000000000000024c  0000000000000000   A       0     0     4
      [17] .eh_frame         PROGBITS         0000000000404c18  00004c18
           0000000000000b5c  0000000000000000   A       0     0     8
      [18] .init_array       INIT_ARRAY       0000000000605e10  00005e10
           0000000000000008  0000000000000000  WA       0     0     8
      [19] .fini_array       FINI_ARRAY       0000000000605e18  00005e18
           0000000000000008  0000000000000000  WA       0     0     8
      [20] .jcr              PROGBITS         0000000000605e20  00005e20
           0000000000000008  0000000000000000  WA       0     0     8
      [21] .dynamic          DYNAMIC          0000000000605e28  00005e28
           00000000000001d0  0000000000000010  WA       6     0     8
      [22] .got              PROGBITS         0000000000605ff8  00005ff8
           0000000000000008  0000000000000008  WA       0     0     8
      [23] .got.plt          PROGBITS         0000000000606000  00006000
           0000000000000198  0000000000000008  WA       0     0     8
      [24] .data             PROGBITS         00000000006061a0  000061a0
           0000000000000074  0000000000000000  WA       0     0     32
      [25] .bss              NOBITS           0000000000606240  00006214
           00000000000001c0  0000000000000000  WA       0     0     64
      [26] .comment          PROGBITS         0000000000000000  00006214
           000000000000004e  0000000000000001  MS       0     0     1
      [27] .shstrtab         STRTAB           0000000000000000  00006262
           00000000000000f8  0000000000000000           0     0     1
    Key to Flags:
      W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
      I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
      O (extra OS processing required) o (OS specific), p (processor
    specific)

Now we see the program header table, describing the segments. Note that
the file offset and the virtual address are almost in correspondence,
with the virtual address being offset by one of two fixed values.  This
is, of course, not a requirement, but is a common feature of an ELF.

    Program Headers:
      Type           Offset             VirtAddr           PhysAddr
                     FileSiz            MemSiz              Flags  Align
      PHDR           0x0000000000000040 0x0000000000400040
    0x0000000000400040
                     0x00000000000001f8 0x00000000000001f8  R E    8
      INTERP         0x0000000000000238 0x0000000000400238
    0x0000000000400238
                     0x000000000000001c 0x000000000000001c  R      1
          [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
      LOAD           0x0000000000000000 0x0000000000400000
    0x0000000000400000
                     0x0000000000005774 0x0000000000005774  R E    200000
      LOAD           0x0000000000005e10 0x0000000000605e10
    0x0000000000605e10
                     0x0000000000000404 0x00000000000005f0  RW     200000
      DYNAMIC        0x0000000000005e28 0x0000000000605e28
    0x0000000000605e28
                     0x00000000000001d0 0x00000000000001d0  RW     8
      NOTE           0x0000000000000254 0x0000000000400254
    0x0000000000400254
                     0x0000000000000044 0x0000000000000044  R      4
      GNU_EH_FRAME   0x00000000000049cc 0x00000000004049cc
    0x00000000004049cc
                     0x000000000000024c 0x000000000000024c  R      4
      GNU_STACK      0x0000000000000000 0x0000000000000000
    0x0000000000000000
                     0x0000000000000000 0x0000000000000000  RW     10
      GNU_RELRO      0x0000000000005e10 0x0000000000605e10
    0x0000000000605e10
                     0x00000000000001f0 0x00000000000001f0  R      1

This following section is a simple mapping of sections into segments.
Every section is contained in one or more segments. This is purely
calculated by whether the file offsets of the section and segment
overlap. 

Since segments describe the semantics of the data at load time, we need
to preserve these mappings in our modification. Note, for instance, that
the .text section is in segment 02, which is (from the above table) a
LOAD segment, which means that the loader will actually map this into
memory. Messing with this will break the ELF.

     Section to Segment mapping:
      Segment Sections...
       00     
       01     .interp 
       02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym
              .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt 
              .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
       03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
       04     .dynamic 
       05     .note.ABI-tag .note.gnu.build-id 
       06     .eh_frame_hdr 
       07     
       08     .init_array .fini_array .jcr .dynamic .got 

It's useful to have a good idea of what a 'nice' ELF looks like when
you're attempting to dissect one. Though not all ELFs will conform to
such a standard (it's not very hard to craft an ELF of your own), most
ELFs are produced by a fairly similar or consistent toolchain which
results in many of the standard binaries on a Linux system conforming to
this format.

### The Plan

What we would like to do is to insert a new section into the ELF, with
executable content, and redirect control at execution time into this
section. The simplest way to do this is to add an entirely new LOAD
segment at the end of the file, but I've opted to expand the segment
containing the .text section for two reasons- one, it's a more covert
insertion which leaves the ELF in a fairly 'normal' state, and two, it's
more interesting.

1. Find a suitable location to insert the new section, preferably within
the existing LOAD segment. I've opted to place my new section before the
`.text` section, which any executable ELF will have. 
2. Fix up the section header table to make room for this new section,
and add our new entry. This mostly involves incrementing offsets (we
may also have to re-align). There's a bit of a gotcha here involving the
location of the table itself, which I'll come back to later.
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

Easy enough, right? In the next post we'll delve into each of these
steps in greater detail.

-James

