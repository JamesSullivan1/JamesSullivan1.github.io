---
layout: post
title: "executable -> readable"
description: "Reading a program's contents without read access"
category: 
tags: []
---
{% include JB/setup %}

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
