---
layout: post
title: "async signal unsafe"
description: ""
category: concurrency
tags: [concurrency, threading, signals]
---
{% include JB/setup %}

A common feature of large applications is that they register a number of
signal handlers to deal with synchronous (e.g., raised by exceptions)
and asynchronous (delivered by other processes) signals. It can be
useful to know when a signal was delivered and to perform some logging,
but even with such a simple task, code that would otherwise be perfect
can become incorrect.

The issue with signal handlers (in particular, asynchronous signal
handlers) is that they could be executed at (almost) any point in an
application's execution. Barring any signal masking that the
application performs, an asynchronous signal will be delivered and
handled as soon as the kernel receives the signal request. The fact that
these signal handlers can start running almost anywhere in the code is
where people run into issues with them.

Consider a simple handler that logs the occurrence of a signal.

    void handler(int sig)
    {
        printf("Signal %d was caught\n", sig);
    }

    int main(void)
    {
        signal(SIGUSR1, handler);
        for (;;)
        {
            // do some work
            printf("Innocuous message\n");
            // do some more work
        }
    }

This looks benign enough-- all this is doing is reporting the signal and
exiting. As it turns out, this program can be very easily coerced into
deadlock. Try running the below code:

    /* deadlock.c */

    #include <signal.h>
    #include <stdio.h>
    #include <unistd.h>

    void handler(int sig)
    {
        printf("Signal %d was caught\n", sig);
    }

    int signal_parent(void)
    {
        pid_t ppid = getppid();
        for (;;)
        {
            kill(ppid, SIGUSR1);
        }
    }

    int main(void)
    {
        pid_t pid;
        signal(SIGUSR1, handler);

        pid = fork();
        if (pid == 0) {
            return signal_parent();
        }

        for (;;)
        {
            // do some work
            printf("Innocuous message\n");
            // do some more work
        }
    }

Given about 10 seconds, this hangs. This doesn't quite seem right-- all
that the signal handler is doing is calling printf! Surely that function
is safe, right?

As it turns out, printf isn't as simple or as safe as it seems on the
surface. Take a look at the call stack for this program after it hangs:

    #0  0x00007fb5f4fca0eb in __lll_lock_wait_private () from /usr/lib/libc.so.6
    #1  0x00007fb5f4f1a60b in vfprintf () from /usr/lib/libc.so.6
    #2  <signal handler called>
    #3  0x6567617373656d20 in ?? ()

The top stack frame is the interesting one here. `lock' should be a huge
red flag for any signal handler, and rightfully so. Locks are extremely
unsafe to use in any signal handler and are a very common way to
deadlock a program. The interesting part is that even if you use the
locks `correctly' and always acquire/release them correctly, you can
still cause deadlock. Consider this case:

    void handler(int sig)
    {
        lock();
        ++counter;
        unlock();
    }

    int main(void)
    {
        ...
        lock();
        printf("%d signals recieved\n", counter);
        unlock(): 
        ...
    }

Consider what happens when a signal is received by `main' after the lock
is acquired. The handler will call lock() again, and deadlock with
itself. The interesting thing here is that we didn't even need to make
any extra threads to lock up the program, and a glance at the source of
`main' would tell us that this is infallible.

The easy answer often given to this issue is to never use locks in
signal handlers. But as we saw in the simple case of printf,
there are many useful functions which use internal locks! Thankfully,
POSIX has been kind enough to define a list of functions that we can
always use in a signal handler. We call these `async-signal-safe', in
that they are safe to use within an asyncrhonous signal handler. The
list is as follows:

_Exit() _exit() abort() accept() access() aio_error() aio_return()
aio_suspend() alarm() bind() cfgetispeed() cfgetospeed() cfsetispeed()
cfsetospeed() chdir() chmod() chown() clock_gettime() close() connect()
creat() dup() dup2() execle() execve() fchmod() fchown() fcntl()
fdatasync() fork() fpathconf() fstat() fsync() ftruncate() getegid()
geteuid() getgid() getgroups() getpeername() getpgrp() getpid()
getppid() getsockname() getsockopt() getuid() kill() link() listen()
lseek() lstat() mkdir() mkfifo() open() pathconf() pause() pipe() poll()
posix_trace_event() pselect() raise() read() readlink() recv()
recvfrom() recvmsg() rename() rmdir() select() sem_post() send()
sendmsg() sendto() setgid() setpgid() setsid() setsockopt() setuid()
shutdown() sigaction() sigaddset() sigdelset() sigemptyset()
sigfillset() sigismember() sleep() signal() sigpause() sigpending()
sigprocmask() sigqueue() sigset() sigsuspend() sockatmark() socket()
socketpair() stat() symlink() sysconf() tcdrain() tcflow() tcflush()
tcgetattr() tcgetpgrp() tcsendbreak() tcsetattr() tcsetpgrp() time()
timer_getoverrun() timer_gettime() timer_settime() times() umask()
uname() unlink() utime() wait() waitpid() write()

Which seems like quite the list, but is actually a pretty small piece of
the list of POSIX functions. Note the lack of some of the major ones,
such as our friend `printf', or _any_ of the pthreads functions.
(pthreads and signals are known to play very poorly together). Perhaps
the best advice, then, is to keep signal handlers simple and make sure
that you don't call anything that isn't async-signal-unsafe.

There is, however, one dirty secret that can help you log to your
heart's content in a signal handler. Note that the list above contains
the `sigprocmask' function. From the man page of this function:

       sigprocmask — examine and change blocked signals

Interesting. What happens if we block signals in a `critical' section?
It turns out that this is enough to prevent all of the nasty
interactions that usually happen with async-signal-unsafe functions.
Seems great, right? With this, we could define a signal-safe lock as
follows:

    int signal_lock(lock_t *l, sigset_t *save)
    {
        sigset_t set;
        sigfillset(&set); /* Create a full signal set */
        sigprocmask(SIG_BLOCK, &set, &save); /* No more signals */
        _lock(l); /* The actual lock */
        return 0;        
    }

    int signal_unlock(lock_t *l, sigset_t *save)
    {
        _unlock(l);
        sigprocmask(SIG_SETMASK, &save, NULL); /* Enable again */
        return 0;
    }

This works, but it's got a nasty side effect-- if this is overused, it
tends to make it so that your program never receives signals. This is
okay for very simple programs, but as I mentioned earlier, many large
programs use signals heavily. This sort of thing isn't acceptable in
these contexts.

At the end of the day, it's probably `better' to not use
async-signal-unsafe functions or locks at all in your code. (Instead,
try to just use the limited set of functions, use lock-free data
structures, etc). If this isn't feasible (and sometimes it isn't), it's
good to know that this is an option, but it's also important to be
careful not to overuse it and be aware of the consequences.

 
