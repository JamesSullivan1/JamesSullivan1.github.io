---
layout: post
title: "executable -> readable"
description: "Reading a program's contents without read access"
category: 
tags: []
---
{% include JB/setup %}

Today, I encountered an interesting exercise inspired by the Utumno Wargame by [OverTheWire](http://overthewire.org). An executable ELF file is given to you that you must read the process address space of in order to find a secret challenge- the caveat being that the file *only* has execute permission, and not read or write.

Immediately, this takes away the easier solutions, such as strings(1), objdump(1), and hexdump(1). However, this ends up being a bigger problem than I anticipated, when I attempted to open up gdb...

And was greeted with a firm *Permission Denied*. It seems likely to me that because gdb not only executes the program but also reads its symbols, it cannot be used in this case. Nevertheless, I continued to try other tools in my toolkit. Another tool that came to mind was strace(1)...

With which I had much greater luck. I could not yet read arbitrary memory, but this gave me two key pieces of information I could work with.

1. I now have the address in memory of a variety of system calls.
2. *ptrace(2)*, the underlying system call that strace(1) uses, is fair game.

What is ptrace(2)?
---------------
ptrace(2) is a UNIX system call that enables a process to *trace execution* of another process, and *examine its memory and registers*. This is an extremely powerful (and interesting) system call that enables many of the tools that we use for binary inspection and debugging (gdb, anyone?).
