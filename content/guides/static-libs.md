---
author: "mulling"
title: "static-libs(2)"
date: 2024-02-20T23:31:05-03:00
draft: false
description: "ldd: not a dynamic executable."
---

**The Problem: How to use ones binary on another computer?**

This problem has been discussed several times, by several people, and at this point also proves the [cave-allegory](https://en.wikipedia.org/wiki/Allegory_of_the_cave). It also serves to show a growing trend in the today's software industry -- of solving problems with bigger problems. Let's start with the more idiotic solutions, going down to the more sensible ones.

* [Snap Packages](https://snapcraft.io/): One of the sewage spill's from Canonical (the guys that make Ubuntu and [a bunch of annoying questions](https://ubuntu.com/blog/written-interviews)). Described as:

    > Snaps are a secure and scalable way to embed applications on Linux devices. A snap is an application containerised with all its dependencies. A snap can be installed using a single command on any device running Linux. With snaps, software updates are automatic and resilient. Applications run fully isolated in their own sandbox, thus minimising security risks.

    Snap packages require a lot of infrastructure to be present in the host system to even work. That includes a **daemon(3)**, xdg-desktop extensions, kernel modules, and more. Snaps are also tightly coupled to the Ubuntu ecosystem. Snaps also break what they package[^1].
* [Flatpaks](https://www.flatpak.org/): Self-entitled:
    > The future of apps on Linux

    Flatpaks are similar to Snaps, but, less bad, i.e., they use the Linux namespaces for sandboxing, opposed to reinventing the wheel like Snaps do with AppArmor. Still a terrible idea for package distribution.
* [AppImages](https://appimage.org/): Is a self-sufficient runtime that bundles the program dependencies and does not depend on any host machinery to work (they kinda do since the runtime is dynamically linked against glibc, but let's ignore that for now). It's a valid solution for the problem, but, limited in some ways.
* Package everything yourself (also known as static linking with extra steps): This one is similar to the idea of AppImages, but, instead, you do the hard work.
* Static linking: Just static link everything. Will work everywhere, most of the time.

In all cases we see that all dependencies need to be in the final bundle -- in the final package. If that is the case, I'll raise the argument: Why we don't just statically link everything?

**The future is the past, now!**

There are two reasons that I can think static linkage is not more common today. The first one is that most applications depend on glibc, which is a pain to statically link. The second one is a misconception tha static linked programs will use more disk space and memory. A third argument could be made that updating existent tooling and build systems to support static linkage or new binary file format is too costly.

As a practical exercise let's compare a dynamic and a static version of the same application. The application we are going to compare is **st(1)**, a simple terminal that is complex enough to have a decent amount of dynamic dependencies (listed bellow). For the comparison we are going to use **ldd(1)**, **pmap(1)**, and **top(1)**, and are mainly going to be looking at memory mappings and memory usage.

Source code and instructions on how to compile each version can be found [here](https://github.com/Mulling/st). I further encourage the reader to do so and share the static version with their friends.

Below is the output of **ldd(1)**, which displays all the dynamic dependencies of a program:

```shell {linenos=false}
$ ldd st-dyn
linux-vdso.so.1 (0x00007ffc23982000)
libX11.so.6 => /usr/lib/libX11.so.6 (0x00007f2ce6644000)
libXft.so.2 => /usr/lib/libXft.so.2 (0x00007f2ce662a000)
libfontconfig.so.1 => /usr/lib/libfontconfig.so.1 (0x00007f2ce65da000)
libfreetype.so.6 => /usr/lib/libfreetype.so.6 (0x00007f2ce650c000)
libc.so.6 => /usr/lib/libc.so.6 (0x00007f2ce632a000)
libxcb.so.1 => /usr/lib/libxcb.so.1 (0x00007f2ce62ff000)
libXrender.so.1 => /usr/lib/libXrender.so.1 (0x00007f2ce62f0000)
libexpat.so.1 => /usr/lib/libexpat.so.1 (0x00007f2ce62c7000)
libz.so.1 => /usr/lib/libz.so.1 (0x00007f2ce62ad000)
libbz2.so.1.0 => /usr/lib/libbz2.so.1.0 (0x00007f2ce629a000)
libpng16.so.16 => /usr/lib/libpng16.so.16 (0x00007f2ce6260000)
libharfbuzz.so.0 => /usr/lib/libharfbuzz.so.0 (0x00007f2ce6152000)
libbrotlidec.so.1 => /usr/lib/libbrotlidec.so.1 (0x00007f2ce6141000)
/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f2ce67b7000)
libXau.so.6 => /usr/lib/libXau.so.6 (0x00007f2ce613c000)
libXdmcp.so.6 => /usr/lib/libXdmcp.so.6 (0x00007f2ce6134000)
libm.so.6 => /usr/lib/libm.so.6 (0x00007f2ce6048000)
libglib-2.0.so.0 => /usr/lib/libglib-2.0.so.0 (0x00007f2ce5efd000)
libgraphite2.so.3 => /usr/lib/libgraphite2.so.3 (0x00007f2ce5ed8000)
libbrotlicommon.so.1 => /usr/lib/libbrotlicommon.so.1 (0x00007f2ce5eb5000)
libpcre2-8.so.0 => /usr/lib/libpcre2-8.so.0 (0x00007f2ce5e1a000)
```

We see a decent amount of dynamic dependencies. Next, let's run **pmap(1)**. **pmap(1)** is basically a pretty printer for /proc/\<pid\>/smaps which shows memory consumption of each memory mapping and flags. I've omitted a few things for brevity.
```shell {linenos=false}
$ pmap -d $(pidof st-dyn)
24430:   ./st-dyn
Address           Kbytes Mode  Offset           Device    Mapping
0000648c6a2a2000      20 r---- 0000000000000000 103:00002 st-dyn
0000648c6a2a7000      56 r-x-- 0000000000005000 103:00002 st-dyn
0000648c6a2b5000      12 r---- 0000000000013000 103:00002 st-dyn
0000648c6a2b8000       4 r---- 0000000000016000 103:00002 st-dyn
0000648c6a2b9000      12 rw--- 0000000000017000 103:00002 st-dyn
0000648c6a2bc000       8 rw--- 0000000000000000 000:00000   [ anon ]
0000648c6c1b4000    1720 rw--- 0000000000000000 000:00000   [ anon ]
0000718227712000     308 rw--- 0000000000000000 000:00000   [ anon ]
[...]
00007182277cf000     600 r---- 0000000000000000 103:00002 NotoSansMono-Bold.ttf
0000718227865000     584 r---- 0000000000000000 103:00002 NotoSansMono-Regular.ttf
[...]
00007ffd9ddf0000     132 rw--- 0000000000000000 000:00000   [ stack ]
00007ffd9de26000      16 r---- 0000000000000000 000:00000   [ anon ]
00007ffd9de2a000       8 r-x-- 0000000000000000 000:00000   [ anon ]
ffffffffff600000       4 --x-- 0000000000000000 000:00000   [ anon ]
mapped: 17940K    writeable/private: 2380K    shared: 1132K
```

What we can take form this is that our dynamic version of st mapped 17940K (17Mb) of which 2380K is private data -- memory that the terminal is using -- and only 1132K is shared data. It is also interesting to see what pages are code, marked `r-x--`; which ones are data, marked `r----` or `rw---`. We also see a few fonts that Xft mapped for us to use.

Now, let's look at the static version -- st-static, first running **pmap(1)**:
```shell {linenos=false}
$ pmap -d $(pidof st-static)
8327:   ./st-static
Address           Kbytes Mode  Offset           Device    Mapping
0000000000400000       4 r---- 0000000000000000 103:00002 st-static
0000000000401000    1696 r-x-- 0000000000001000 103:00002 st-static
00000000005a9000    1076 r---- 00000000001a9000 103:00002 st-static
00000000006b6000      68 rw--- 00000000002b6000 103:00002 st-static
00000000006c7000      12 rw--- 0000000000000000 000:00000   [ anon ]
00000000012c9000    1788 rw--- 0000000000000000 000:00000   [ anon ]
000071ed8597d000     308 rw--- 0000000000000000 000:00000   [ anon ]
000071ed85a3a000     600 r---- 0000000000000000 103:00002 NotoSansMono-Bold.ttf
000071ed85ad0000     584 r---- 0000000000000000 103:00002 NotoSansMono-Regular.ttf
000071ed85b62000    1000 r--s- 0000000000000000 103:00002 923e285e415b1073c8df160bee08820f-le64.cache-8
000071ed85c5c000      12 r--s- 0000000000000000 103:00002 6ba42ae0000f58711b5caaf10d690066-le64.cache-8
000071ed85c5f000      52 r--s- 0000000000000000 103:00002 210c0516121708a580e22e6b1f9a103a-le64.cache-8
000071ed85c6c000       4 r--s- 0000000000000000 103:00002 11b490a251c73e9885ef8a4f43e7c08f-le64.cache-8
00007ffc677ce000     132 rw--- 0000000000000000 000:00000   [ stack ]
00007ffc677f2000      16 r---- 0000000000000000 000:00000   [ anon ]
00007ffc677f6000       8 r-x-- 0000000000000000 000:00000   [ anon ]
ffffffffff600000       4 --x-- 0000000000000000 000:00000   [ anon ]
mapped: 7364K    writeable/private: 2308K    shared: 1068K
```

We can see a LOT fewer pages mapped, of which only 3 are `r-x--` (executable). We can also see the same mapped fonts as the dynamic version. There's also a decrease in mapped memory usage, 7364K. Private data remain mostly the same and shared data follows this as well. This however does not paint the hole picture.

Let's look at the output of **top(1)**, first for st-dyn. **top(1)** will give us a different metrics, since it accounts memory usage differently. It also will show us resident memory use -- which more closely matches what you would expect for the program memory usage.

```shell {linenos=false}
$ top -b | grep -e "PID|st-dyn"
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
32068 mlso      20   0   17936  10368   8064 S   0.0   0.1   0:00.06 st-dyn
```

Now looking at the static version:
```shell {linenos=false}
$ top -b | grep -e "PID|st-static"
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 3338 mlso      20   0    7360   5048   3328 S   0.0   0.0   0:00.03 st-static
```

We can see a significant decrease in resident memory usage (RES), about half.

Summing things up in the table below (take the values shown here for what they are). I'll leave the conclusions as an exercise to the reader, there are trade-offs and we are only looking at a single binary, but the benefits are there. There are also arguments that can be made about the security of DSO's and how to deploy security patches to static linked code, but they fall short of the scope of this article.

```shell {linenos=false,class=table}
+--------------------------+------- +-----------+
|                          | st-dyn | st-static |
+--------------------------+------- +-----------+
| size of the binary       |   113K |     3236K |
| VIRT               (top) | 17936K |     7360K |
| RES                (top) | 10368K |     5048K |
| SHR                (top) |  8064K |     3328K |
| mapped            (pmap) | 17940K |     7364K |
| writeable/private (pmap) |  2380K |     2308K |
| shared            (pmap) |  1132K |     1068K |
+--------------------------+------- +-----------+
```

[^1]: In snap packages, the communication between browser extensions and a native modules happens through the xdg-desktop-portal. Normally, communication between the browser and the native module happens using a simple **stdin(3)** to **stdout(3)** **pipe(3)**. If your system is older and has an older version of xdg-desktop-portal that does not support the protocol which handles browser to native module communication, it simply won't work.
