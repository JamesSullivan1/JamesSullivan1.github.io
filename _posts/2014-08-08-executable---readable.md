---
layout: post
title: "executable -> readable"
description: "Reading a program's contents without read access"
category: reverse engineering 
tags: [reverse engineering, RE, ptrace, linux, ELF]
---
{% include JB/setup %}

_note: this is still a work in progress which I will 
update as time and interest permit._

Today, I encountered an interesting exercise inspired by the 
Utumno Wargame by [OverTheWire](http://overthewire.org). An 
executable ELF file is given to you that you must read the 
process address space of in order to find a secret message- 
the caveat being that the file *only* has execute permission, 
and not read or write.

Immediately, this takes away the easier solutions, such as 
strings(1), objdump(1), and hexdump(1). However, this ends 
up being a bigger problem than I anticipated, when I attempted 
to open up gdb...

And was greeted with a firm *Permission Denied*. It seems likely 
to me that because gdb not only executes the program but also 
reads its symbols, it cannot be used in this case. Nevertheless, 
I continued to try other tools in my toolkit. Another tool that 
came to mind was strace(1)...

With which I had much greater luck. I could not yet read arbitrary 
memory, but this gave me two key pieces of information I could work 
with.

1. I now have the address in memory of a variety of system calls.
2. *ptrace(2)*, the underlying system call that strace(1) uses, 
is fair game.

### What is ptrace?
ptrace(2) is a Linux system call that enables a process to *trace 
execution* of another process, and *examine its memory and registers*. 
This is an extremely powerful (and interesting) system call that 
enables many of the tools that we use for binary inspection and 
debugging (gdb, anyone?). Of course, this tool isn't infinitely 
powerful- it cannot attach to a process to which its own 
execution context does not have execution privelege over.

    long ptrace(enum __ptrace_request request, pid_t pid,
                    void *addr, void *data);

ptrace is quite an interesting system call, because it does damn-near
*everything*. Not only can you read any byte in the tracee address
space, you can also *write* any byte. A typical ptrace call will
have a particular request (ie, PTRACE_PEEKDATA), the pid of the
tracee, a pointer to the target address, and the data that will
be transferred (the latter two fields are ignored for many requests). 

While a program is being traced, it will halt execution whenever
a signal is delivered (except SIGKILL which will have the usual effect).
The tracer is notified when it next calls waitpid(2) or any other related
wait calls. 

Let's look at some interesting requests:

    PTRACE_TRACEME : 
        Indicates that the process is to be traced by
        its parent. Raises a SIGSTOP when the execl(2) system call
        is made (which enables a useful pattern we will see below.)

    PTRACE_ATTACH :
        Attach to a given process and raises a SIGSTOP in the tracee.

    PTRACE_GETREGS :
        Copies the tracee's general purpose registers to the address
        data in the tracer. This returns a pointer to a struct
        user_regs_struct.

    PTRACE_{PEEK,POKE}DATA :
        Reads/Sets a value in the tracee's memory at address to/from
        memory at address data.

Immediately we can see a workflow for extracting the ELF from a process'
address space.

0. Attach to the target process via ptrace,
1. Obtain the address of the ELF Header,
2. Determine the size of the ELF Header and its sections, and
3. Read the necessary memory via ptrace, and write it to a new binary.

### Step 1: Attaching to a process

There are two ways in which ptrace can attach a tracer to a tracee.
The tracer can either attach themself explicitly by calling 
ptrace(PTRACE_ATTACH, pid, ...) or by ptrace(PTRACE_SEIZE, pid, ...).
PTRACE_ATTACH will attach to the process PID, and raise a SIGSTOP in this
process to halt execution. PTRACE_SEIZE does the same thing but it does not
raise the SIGSTOP- not as useful for our purpose, as we need to halt execution.

    pid_t tracee_pid = 12345;
    ptrace(PTRACE_ATTACH, tracee_pid, NULL, NULL);
    int status;
    wait(&status)   // Wait until we are informed that proc PID is
                    // ready to be traced    
    foo(pid);



The second way in which a process can be attached to with ptrace is via
PTRACE_TRACEME. This will inform the parent of the process that it wants to
be traced, and raises a SIGSTOP on any system calls (including execl) to 
halt its own execution.

    pid_t tracee_pid = fork();
    if(!tracee_pid) {
        // Executed by the CHILD        
        ptrace(PTRACE_TRACEME, NULL, NULL, NULL);
    } else {
        // Executed by the PARENT
        int status;
        wait(&status);  // Wait until we are informed that process PID is ready to 
                        // be traced
        foo(pid);
    }
        
### Step 2: Obtaining the address of the ELF Header

The next step is to determine the location of the ELF 
header in the process address space. A caveat, of course,
is that our process stops as soon as execve(2) is called.
If we examine the output of strace, we see that this is
actually not far enough into execution; since *we have not
yet mapped the ELF into the address space*.

Below is the strace output for the `echo` program.

    // Here is where execution has stopped.
    execve("/usr/bin/echo", ["echo"], [/* 6 vars */]) = 0

    // Note that we haven't even hit `brk` yet, which
    // is used to modify the location of the program
    // break.
    // However, this is calling brk with an invalid
    // address, which will return the current break.
    brk(0)                                  = 0x1086000

    // Getting the shared libraries... Not concerning to us.
    access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
    open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 4
    fstat(4, {st_mode=S_IFREG|0644, st_size=186704, ...}) = 0
    mmap(NULL, 186704, PROT_READ, MAP_PRIVATE, 4, 0) = 0x7ff5fa8c3000

            Some lines omitted

    // Here is where we actually load the ELF file.
    open("/usr/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 4
read(4, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\20\1\2\0\0\0\0\0"..., 832) = 832

            Some lines omitted

    // A second call to brk, which once again
    // just returns the current break.
    // This is probably a good spot to be.
    brk(0)                                  = 0x1086000

            Some lines omitted

    // Here is the system calls that are actually
    // done by the program and not the loader.
    // Echo is a simple one, just a few calls.
    write(1, "\n", 1
    )                       = 1
    close(1)                                = 0

    // And this is some post-process cleanup.
    munmap(0x7ff5fa8f0000, 4096)            = 0
    close(2)                                = 0
    exit_group(0)                           = ?
    +++ exited with 0 +++

We need to make sure that we can continue execution until
we have at least loaded the ELF header, but not so far
that the program terminates. A good place to do this is
a few `brk` system calls in. This is a fairly trivial
task:

    int status;
    int ret;
    int brks_left = 2;
    struct user_regs_struct regs;
    do {
        // Step to the next system call entry/exit
        if(ptrace(PTRACE_SYSCALL, pid, 0, 0)) {
            return 1;
        }
        wait(&status);   

        errno = 0; 
        // Get the registers of the program,
        // and check orig_eax == SYS_brk.
        ptrace(PTRACE_GETREGS, pid, 0, &regs);
        if(errno) {
            return 1;
        }  
        ret = regs.orig_rax;
        if(ret == SYS_brk)
            brks_left--;

    } while(brks_left > 0);
    // Step once more to exit BRK
    ptrace(PTRACE_SYSCALL, pid, 0, 0);
    wait(&status);

    return 0; 

Note that we want to check the value of `orig_eax` at
each system call. This is because we stop execution 
as soon as the system call is entered with SIGINT,
and at this point the register `orig_eax` contains
the value of eax before the system call was made- 
which is the code for the system call.

Now that the ELF header is loaded, we need to search
for it. However, the ELF spec does not require that
the ELF header is loaded at some constant address,
and in fact a non-standard program could load it
at any page-aligned address. A typical range of values
on an x86 architecture are [0x08048000, 0x08080000]
and on x86_64, [0x400000, 0x8000000] (and these values
are not chosen at random, but are attributable to the
way that linux creates a process address space).

Despite a fairly large search space, this is not
a difficult task, for two reasons.

1. The ELF header must be page aligned, and
2. The ELF header _always_ starts with '.ELF'.

The constant '.ELF' (alternatively, \x7f\x45\x4c\x46)
is what is called the magic field, and it is essentially
an unambiguous way of marking an address as the start
of an ELF header.

Of course, it's entirely possible that by some chance,
we have multiple points in memory where the string
'\x7f\x45\x4c\x46' occurs, and so it is best to 
list all of the possible addresses and then individually
determine which is the actual ELF header.
