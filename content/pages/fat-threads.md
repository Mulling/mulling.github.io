---
author: "Mülling"
title: "fat_threads(3)"
date: 2023-08-17T01:32:52-03:00
draft: true
description: Seeking portability without clone(2).
---

The POSIX standard often lacks in functionality for run-time isolation of dynamically bound symbols, i.e., `dlopen` and `dysym` can, and will, `SEGFAULT` your program. This means that programs that are extendable through plugins or use cryptographic APIs such as PKCS #11 are at mercy of library developers.

Take for instance this innocent program:
```c
#include <stdio.h>

void l() {
    asm("hlt");
}
```

When the symbol `l` is loaded and executed, like bellow, your program will just SEGFAULT. One solution is to handle the signal, this case an illegal instruction, the downsides are: a) it may mask other problems; b) you often don't know which signals to handle, meaning you have to handle all the signals, which may cause other problems and limitations.

```c
#include <dlfcn.h>
#include <stdio.h>

int main() {
    void* h = dlopen("./l.so", RTLD_NOW);

    if (!h) {
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }

    void (*l)() = dlsym(h, "l");

    char* e = dlerror();
    if (e) {
        fprintf(stderr, "%s\n", e);
        return 1;
    }

    puts("calling void l()");
    l();
    puts("called void l()");

    return 0;
}
```

Fortunately, the Linux kernel has a syscall that can help, `clone(2)`. A quick look in the man page reveals some useful information:

> By  contrast with fork(2), these system calls provide more precise control over what pieces of execution context are shared between the calling process and the child process.  For  example, using these system calls, the caller  can control whether or not the two processes share the virtual address space, the table of file descriptors, and the table of signal handlers.  These system calls  also  allow  the  new child process to be placed in separate namespaces(7). [...] When the child process is created with the clone() wrapper function, it commences execution by calling the function pointed to by the argument fn.  (This differs from fork(2), where execution continues in the child from the point of the fork(2) call.)  The arg argument is passed as the argument of the function fn.


Let's fix our program:
```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <sched.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/wait.h>

static int fn(void* a) {
    void* h = dlopen("./l.so", RTLD_NOW);

    if (!h) {
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }

    void (*l)() = dlsym(h, "l");

    char* e = dlerror();
    if (e) {
        fprintf(stderr, "%s\n", e);
        return 1;
    }

    puts("calling void l()");
    l();
    puts("called void l()");
    return 0;
}

int main() {
    uint8_t* t;

    uint8_t* s = mmap(NULL, 1024 << 10, PROT_READ | PROT_WRITE,
             MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);

    if (s == MAP_FAILED) {
        fprintf(stderr, "fail to allocate stack");
        return 1;
    }

    pid_t pid = clone(fn, (t = s + (1024 << 10)), SIGCHLD, NULL);

    if (pid == -1) {
        fprintf(stderr, "fail to clone");
        return 1;
    }

    int status;

    waitpid(pid, &status, 0);
    printf("child status %d\n", status);

    return 0;
}
```

The first step is allocate a stack for your "thread" -- `fn` -- we do this with `mmap`. The important bits here are `PROT_READ`, `PROT_WRITE`, `MAP_PRIVATE`, and `MAP_ANONYMOUS`. Contrary of common sense, `MAP_STACK` does nothing.

```c
    uint8_t* s = mmap(NULL, 1024 << 10, PROT_READ | PROT_WRITE,
             MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
```

Now we can call `clone`. The important bits here are `SIGCHLD`, so we can know when your process terminates.

```c
    pid_t pid = clone(fn, (t = s + (1024 << 10)), SIGCHLD, NULL);
```

After setting IPC channels (which goes beyond the scope of this), our program
can safely load third party dynamic libraries. The problem is, this only works on Linux.

...
