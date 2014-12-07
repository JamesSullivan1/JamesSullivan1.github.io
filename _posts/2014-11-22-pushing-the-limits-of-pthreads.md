---
layout: post
title: "Pushing the limits of pthreads"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Pthreads (or POSIX Threads) is the POSIX standard for the creation and
manipulation of threads. This API is a set of C types and functions that
enable a high-level interface for application developers to the
underlying threading functionality of the Operating System. For example,
in Linux, pthreads provides a front-end for the creation of anonymous
Lightweight Processes (a fancy word for threads), which have their own
stack, but share a virtual address space with their parent and siblings.

On Linux, the pthreads library is supported by the clone(2) system call.
From the man page:


    This page describes both the glibc  clone()  wrapper  function and  the
    underlying  system  call  on which it is based.  The main text describes
    the wrapper function; the  differences  for  the  raw  system call  are
    described toward the end of this page.

    Unlike  fork(2),  clone() allows the child process to share parts of its
    execution context with the calling process, such as  the  memory space,
    the  table of file descriptors, and the table of signal handlers.  (Note
    that on this manual page,  "calling  process"  normally corresponds  to
    "parent process".  But see the description of CLONE_PARENT below.)

    The  main  use  of  clone() is to implement threads: multiple threads of
    control in a program that run concurrently in a shared memory space.


