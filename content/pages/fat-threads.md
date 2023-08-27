---
author: "Mülling"
title: "fat_threads(3)"
date: 2023-08-27T17:27:19-03:00
draft: false
description: Sandboxing of dinamicaly bound symbols.
---

The POSIX standard lacks in functionality for run-time isolation of dynamically bound symbols, i.e., **dlopen** and **dlsym** can, and will, crash your programs. Being from symbol collision from statically (and dynamic) compiled libraries with other dynamic object symbols, to crashes from execution of dynamically loaded code. Meaning, programs that are extendable through plugins or use cryptographic API's such as PKCS #11 are at mercy of third-party developers.

Library interoperability of dynamic shared objects (DSO's) which link against anything that uses **pthreads** or **glibc** is delicate at best. **glibc** developers tried to fix this with **dlmopen(2)** and previously with **RTLD_DEEPBIND**, which most likely will also crash your programs (this will make more sense later on).

Take for instance the program bellow, function {{<highlight c "linenos=inline,hl_inline=true">}}foo(){{</highlight>}} calls an illegal user-space instruction, terminating the program:
```c
void foo() {
    asm("hlt");
}
```
Let's turn our program into a shared library:
```shell {linenos=false}
$ gcc -shared l.c -o l.so
```
Now, we can load the DSO with {{< highlight c "linenos=inline,hl_inline=true" >}}dlopen("./lib.so", RTLD_NOW){{< / highlight >}} and bind the symbol to a local function pointer with {{< highlight c "linenos=inline,hl_inline=true" >}}void (*foo)() = dlsym(handle, "foo"){{< / highlight >}}. See below:
```c
#include <dlfcn.h>
#include <stdio.h>

int main() {
    void* handle = dlopen("./lib.so", RTLD_NOW);

    if (!handle) {
        perror(dlerror());
        return 1;
    }

    void (*foo)() = dlsym(handle, "foo");

    char* e = dlerror();
    if (e) {
        perror(dlerror());
        return 1;
    }

    foo();

    return 0;
}
```
Building and running:
```shell {linenos=false}
$ gcc -o main main.c
$ ./main
Segmentation fault
```
As hinted before, the question here is, how can we safely call {{<highlight c "linenos=inline,hl_inline=true">}}foo(){{</highlight>}}, without it terminating our program? There's also the problem of unloading the shared library, non-static objects lifetimes (i.e. a thread created by the DSO) also makes impossible for one to call **dlclose(3)** on unknown shared objects, that road leads to undefined behaviour.

The common paradigm is to **fork(2)** **exec(3)**, perform the dynamic binding work in the child, and subsequently, kill the child. Evidently, two binaries are necessary, this can be achieved by means of embedding one binary in another and using **memfd_create(2)** + **fexecve(2)**; or by loading a binary form a file and **fexecve(2)** it (this is necessary to verify the hash). Of course, this is also not portable.

Let's explore what happens if we try to deviate from this (we won't dive into the problems with multithreaded programs and **fork(2)**). To **fork(2)**, without **exec(3)**, and load the DSO. Bellow is an assert of that (for brevity, from now on, I've omitted all the error handling code):
```c
#include <dlfcn.h>
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    pid_t pid;

    if ((pid = fork()) == 0) {
        void* h = dlopen("./lib.so", RTLD_NOW);

        void (*foo)() = (void (*)())dlsym(h, "foo");

        foo();

        return 0;
    }

    int status;
    waitpid(pid, &status, 0);

    printf("child status %d\n", status);

    return 0;
}
```
Ok, we've got what we wanted, but, our program now resides at the limbo between **fork(2)** and **exec(3)**, essentially, both processes now have the same image, the same virtual memory and the same state. It is also not safe to do anything that is not **async-signal safe**. The DSO's symbols we bind will have access to every file descriptor the parent has open at forking time, to quote the man page:

> The child inherits copies of the parent's set of open file descriptors.  Each file descriptor in the child refers to the same open file description   (see open(2)) as the corresponding file descriptor in the parent. This means that the two file descriptors share open file status flags, file offset, and signal-driven I/O attributes [...]

This also happens if files are not marked with **FD_CLOEXEC** prior to **exec(3)**. Yes, we can do anything we want with the parents file descriptors, and if we are clever, structures in memory also ;). This is also true even if we didn't fork at all. Making some changes to our shared library to test the former.
```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

void foo() {
    for (int i = 0; i < 256; i++) {
        char c;

        lseek(i, 0, SEEK_SET);

        fcntl(i, F_SETFL, O_NONBLOCK);

        if (read(i, &c, sizeof c) > 0) {
            putchar(c);

            while (read(i, &c, sizeof c) != 0) putchar(c);

            puts("");

            return;
        }
    }
}
```
We loop through the first 256 file descriptors testing if they are readable, (recall that **fork(2)** copies the state of each file descriptors offset, we need to reset it by using {{<highlight c "linenos=inline,hl_inline=true">}}lseek(i, 0, SEEK_SET){{</highlight>}}). We set **O_NONBLOCK** with {{<highlight c "linenos=inline,hl_inline=true">}}fcntl(i, F_SETFL, O_NONBLOCK){{</highlight>}}, as to not block on a empty pipe. Below is our main block, updated to create a file that holds some data for testing purposes:
```c
#include <dlfcn.h>
#include <stdio.h>
#include <sys/wait.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    int fd = open("./file.txt", O_CREAT | O_RDWR );

    char secret[] = "parent data";

    write(fd, secret, sizeof secret);

    pid_t pid;

    if ((pid = fork()) == 0) {
        void* h = dlopen("./lib.so", RTLD_NOW);

        void (*foo)() = (void (*)())dlsym(h, "foo");

        foo();

        return 0;
    }

    int status;

    waitpid(pid, &status, 0);
    printf("child status %d\n", status);

    return 0;
}
```
Running the code:
```
$ gcc -shared lib.c -o lib.so
$ gcc -o main main.c
$ ./main
parent data
child status 0
```
We can remediate this by requesting a new namespace with **dlmopen(3)**. To quote to man pages for **dlmopen(3)**:
> The dlmopen() function differs from dlopen() primarily in that it accepts an additional argument, lmid, that specifies the link-map list (also referred to as a namespace) in which the shared object should be loaded.

Calling **dlmopen(3)** with **LM_ID_NEWLM** will bind the DSO symbols to a new, empty, link-map list. Changing line 16 of our main block of code to: {{<highlight c "linenos=inline,hl_inline=true">}}void* handle = dlmopen(LM_ID_NEWLM, "./lib.so", RTLD_NOW){{</highlight>}}, now, the child's file descriptor table is not accessible in the new namespace, cool. **dlmopen(3)** comes however with hole new set of problems with symbols that are shared across namespaces. https://sourceware.org/bugzilla/show_bug.cgi?id=24776

Tangentially, **RTLD_DEEPBIND** can be used to place the DSO symbols in another link-map list, with priority over the global symbols table, but, parent data can still be read. We still need to exec...

Let's now take a look at **clone(2)**. A quick glance at the man page reveals some useful information:

> By contrast with fork(2), these system calls provide more precise control over what pieces of execution context are shared between the calling process and the child process. For example, using these system calls, the caller  can control whether or not the two processes share the virtual address space, the table of file descriptors, and the table of signal handlers.  These system calls also allow the new child process to be placed in separate namespaces(7). [...] When the child process is created with the clone() wrapper function, it commences execution by calling the function pointed to by the argument fn. (This differs from fork(2), where execution continues in the child from the point of the fork(2) call.) The arg argument is passed as the argument of the function fn.

Updating your main block:
```c
#define _GNU_SOURCE
#include <dlfcn.h>
#include <sched.h>
#include <stdint.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/wait.h>
#include <unistd.h>

static int fn(void* a) {
    void* handle = dlopen("./lib.so", RTLD_NOW | RTLD_DEEPBIND);

    void (*foo)() = dlsym(handle, "foo");

    foo();

    return 0;
}

int main() {
    uint8_t* t;

    uint8_t* s = mmap(NULL, 1024 << 10, PROT_READ | PROT_WRITE,
                      MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);

    pid_t pid = clone(fn, (t = s + (1024 << 10)), SIGCHLD, NULL);

    int status;

    waitpid(pid, &status, 0);
    printf("child status %d\n", status);

    return 0;
}
```
The first step is allocate a stack for your fat thread, this is done with {{<highlight c "linenos=inline,hl_inline=true">}}mmap(NULL, 1024 << 10, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0){{</highlight>}}. The important bits are **PROT_READ**, **PROT_WRITE**, **MAP_PRIVATE**, and **MAP_ANONYMOUS**. Contrary of common sense, **MAP_STACK** does nothing. Now, calling {{<highlight c "linenos=inline,hl_inline=true">}}clone(fn, (t = s + (1024 << 10)), SIGCHLD, NULL){{</highlight>}}. The important bits for clone are **SIGCHLD**, so the parent can know when the fat thread terminates. This seems to solve our problems, but, in the end, we are essentially forking... And, again, we still need to exec... There is no way for the current Linux linker/loader to pull only the needed symbols when cloning. The whole image is copied over. For a hack, the binary could be made into a executable DSO, loaded lazily with **dlmopen(3)** + **RTLD_LAZY**, then DSO binding performed inside the empty namespace.

Ideally we need fork + exec isolation capabilities that only binds the needed dependencies of the entry point, I think the eBPF loader does this, we shall investigate this soon, stay tuned.
