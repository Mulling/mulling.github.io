---
author: "Mülling"
title: "static_libs(2)"
date: 2024-02-20T23:31:05-03:00
draft: false
description: "ldd: not a dynamic executable."
---

The current fragmentation of Linux desktop environments (and servers to some extend) is further aggravated by the fact that dynamic linkage is, **today**, the norm. A relic from the times where disk space, memory, and instructions where a sparse commodity. Back in the day, there were only static binaries, everything you needed to run your program was there, binary file formats where practically non-existent and one would simply map the program into memory, set the start address and the program would run. This, however, caused a problem when time sharing became a thing; suddenly, programs would compete for the precious machine resources. One of those was space. It did not take long for people to realize that code was duplicated -- people where reusing the same routines and functions -- and could be made reusable. And when virtual memory became a thing, shared between the processes. The concept of dynamic shared libraries was then born.

Dynamic shared libraries solved the problem of code reuse in the resource constraint environments of the time. Today, however, the constraints have shifted; we are no longer bound by limited memory, disk space or how many instructions the CPU can chug in a second. We have, however, high heterogeneity between platforms, distributions, architectures, and features.

> <b>The Problem: How to use ones binary on another computer</b>?

This problem has been discussed [several times](https://drewdevault.com/dynlib.html), by several people, and at this point also proves the cave-allegory. It also serves to show a growing trend in the today's software industry, of solving problems with bigger problems. Let's start with the more idiotic solutions, going down to the more sensible ones.

* Snap Packages: One of the sewage spill's from Canonical (the guys that make Ubuntu and [a bunch of annoying questions](https://ubuntu.com/blog/written-interviews)). Snap packages require a lot of infrastructure to be present in the host system to even work. That includes a **daemon(3)**, xdg-desktop extensions, kernel modules, and more. Snaps are also tightly coupled to the Ubuntu ecosystem. The masses, however seem to dislike snap for other reasons, mainly, the size of snap packages, they however are not bothered by the gigabytes of debug-info cargo just dumped in their home directories. This furthers eludes to the point above. Snap packages are not a terrible idea because they bundle all of the necessary moving parts of the program, but, because of the other problems they introduce. [^1]
* Flatpaks: Self entitled <i>"the future of application distribution"</i>. Flatpaks are similar to Snaps, but, less bad, i.e., they use the Linux namespaces for sandboxing, opposed to reinventing the wheel like Snaps do with AppArmor.
* AppImages: Is a self-sufficient runtime that bundles the program dependencies and does not depend on any host machinery to work (they kinda do since the runtime is dynamically linked against glibc, but let's ignore that for now).
* Package everything yourself (also known as static linking with extra steps): This one is similar to the idea of AppImages, but, instead, you do the hard work.
* Static linking: Just static link everything. Will work everywhere, most of the time (we will talk more about this in the future).

In all cases we see that all dependencies need to be in the final bunble -- the package. If that is the case, I'll raise the argument: Why we don't just statically link everything?

> <b>The future is the past, now!</b>

There are two reasons that I can think static linkage is not more common today. The first one is that most applications depend on glibc, which is a pain to statically link. The second one is misconception on how statically linked libraries work, if compiled with position independent code, then libraries can be mapped into memory and pages shared between processes, if not, text relocation takes place and that does not hold true; the problem is however, that the program will end-up using only a few symbols from the hole mapped page. A third argument could be made that updating existent tooling and build systems to support static linkage or new binary file formats is too costly, but I do think it could be done gradually.

As a practical exercise let's compare a dynamic and a static version of the same application. The application we are going to compare is **st(1)**, a simple terminal that is complex enough to have a decent amount of dynamic dependencies. To do the comparison we are going to use **ldd(1)**, **pmap(1)**, and **top(1)**. Mainly looking at memory mappings and memory usage.

Source code and instructions on how to compile each version can be found [here](https://github.com/Mulling/st). I further encourage the reader to do so and share the static version with their friends, just like the 60's.

Lets analyse the first version, dynamic linked st -- st-dyn:

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

We see a decent amont of dynamic dependencies. Next, let's run **pmap(1)**. **pmap(1)** is basically a pretty printer for /proc/\<pid\>/smaps which shows memory cosumption of each memory mapping and flags.

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
00007182277bb000       8 r---- 0000000000000000 103:00002 libXfixes.so.3.1.0
00007182277bd000      12 r-x-- 0000000000002000 103:00002 libXfixes.so.3.1.0
00007182277c0000       4 r---- 0000000000005000 103:00002 libXfixes.so.3.1.0
00007182277c1000       4 r---- 0000000000006000 103:00002 libXfixes.so.3.1.0
00007182277c2000       4 rw--- 0000000000007000 103:00002 libXfixes.so.3.1.0
00007182277c3000      12 r---- 0000000000000000 103:00002 libXcursor.so.1.0.2
00007182277c6000      20 r-x-- 0000000000003000 103:00002 libXcursor.so.1.0.2
00007182277cb000       8 r---- 0000000000008000 103:00002 libXcursor.so.1.0.2
00007182277cd000       4 r---- 0000000000009000 103:00002 libXcursor.so.1.0.2
00007182277ce000       4 rw--- 000000000000a000 103:00002 libXcursor.so.1.0.2
00007182277cf000     600 r---- 0000000000000000 103:00002 NotoSansMono-Bold.ttf
0000718227865000     584 r---- 0000000000000000 103:00002 NotoSansMono-Regular.ttf
00007182278f7000    1060 r--s- 0000000000000000 103:00002 923e285e415b1073c8df160bee08820f-le64.cache-9
0000718227a00000    2988 r---- 0000000000000000 103:00002 locale-archive
0000718227d03000     356 r---- 0000000000000000 103:00002 LC_CTYPE
0000718227d5c000      20 rw--- 0000000000000000 000:00000   [ anon ]
0000718227d61000      12 r---- 0000000000000000 103:00002 libpcre2-8.so.0.11.2
0000718227d64000     428 r-x-- 0000000000003000 103:00002 libpcre2-8.so.0.11.2
0000718227dcf000     172 r---- 000000000006e000 103:00002 libpcre2-8.so.0.11.2
0000718227dfa000       4 r---- 0000000000099000 103:00002 libpcre2-8.so.0.11.2
0000718227dfb000       4 rw--- 000000000009a000 103:00002 libpcre2-8.so.0.11.2
0000718227dfc000       4 r---- 0000000000000000 103:00002 libbrotlicommon.so.1.1.0
0000718227dfd000       4 r-x-- 0000000000001000 103:00002 libbrotlicommon.so.1.1.0
0000718227dfe000     124 r---- 0000000000002000 103:00002 libbrotlicommon.so.1.1.0
0000718227e1d000       4 r---- 0000000000021000 103:00002 libbrotlicommon.so.1.1.0
0000718227e1e000       4 rw--- 0000000000022000 103:00002 libbrotlicommon.so.1.1.0
0000718227e1f000      12 r---- 0000000000000000 103:00002 libgraphite2.so.3.2.1
0000718227e22000      96 r-x-- 0000000000003000 103:00002 libgraphite2.so.3.2.1
0000718227e3a000      20 r---- 000000000001b000 103:00002 libgraphite2.so.3.2.1
0000718227e3f000       8 r---- 000000000001f000 103:00002 libgraphite2.so.3.2.1
0000718227e41000       4 rw--- 0000000000021000 103:00002 libgraphite2.so.3.2.1
0000718227e42000       8 rw--- 0000000000000000 000:00000   [ anon ]
0000718227e44000     116 r---- 0000000000000000 103:00002 libglib-2.0.so.0.7800.4
0000718227e61000     632 r-x-- 000000000001d000 103:00002 libglib-2.0.so.0.7800.4
0000718227eff000     564 r---- 00000000000bb000 103:00002 libglib-2.0.so.0.7800.4
0000718227f8c000       4 r---- 0000000000148000 103:00002 libglib-2.0.so.0.7800.4
0000718227f8d000       4 rw--- 0000000000149000 103:00002 libglib-2.0.so.0.7800.4
0000718227f8e000       4 rw--- 0000000000000000 000:00000   [ anon ]
0000718227f8f000      56 r---- 0000000000000000 103:00002 libm.so.6
0000718227f9d000     512 r-x-- 000000000000e000 103:00002 libm.so.6
000071822801d000     368 r---- 000000000008e000 103:00002 libm.so.6
0000718228079000       4 r---- 00000000000e9000 103:00002 libm.so.6
000071822807a000       4 rw--- 00000000000ea000 103:00002 libm.so.6
000071822807b000       8 r---- 0000000000000000 103:00002 libXdmcp.so.6.0.0
000071822807d000       8 r-x-- 0000000000002000 103:00002 libXdmcp.so.6.0.0
000071822807f000       8 r---- 0000000000004000 103:00002 libXdmcp.so.6.0.0
0000718228081000       4 r---- 0000000000005000 103:00002 libXdmcp.so.6.0.0
0000718228082000       4 rw--- 0000000000006000 103:00002 libXdmcp.so.6.0.0
0000718228083000       4 r---- 0000000000000000 103:00002 libXau.so.6.0.0
0000718228084000       4 r-x-- 0000000000001000 103:00002 libXau.so.6.0.0
0000718228085000       4 r---- 0000000000002000 103:00002 libXau.so.6.0.0
0000718228086000       4 r---- 0000000000002000 103:00002 libXau.so.6.0.0
0000718228087000       4 rw--- 0000000000003000 103:00002 libXau.so.6.0.0
0000718228088000       4 r---- 0000000000000000 103:00002 libbrotlidec.so.1.1.0
0000718228089000      36 r-x-- 0000000000001000 103:00002 libbrotlidec.so.1.1.0
0000718228092000      12 r---- 000000000000a000 103:00002 libbrotlidec.so.1.1.0
0000718228095000       4 r---- 000000000000c000 103:00002 libbrotlidec.so.1.1.0
0000718228096000       4 rw--- 000000000000d000 103:00002 libbrotlidec.so.1.1.0
0000718228097000       8 rw--- 0000000000000000 000:00000   [ anon ]
0000718228099000      48 r---- 0000000000000000 103:00002 libharfbuzz.so.0.60830.0
00007182280a5000     808 r-x-- 000000000000c000 103:00002 libharfbuzz.so.0.60830.0
000071822816f000     216 r---- 00000000000d6000 103:00002 libharfbuzz.so.0.60830.0
00007182281a5000       4 r---- 000000000010c000 103:00002 libharfbuzz.so.0.60830.0
00007182281a6000       4 rw--- 000000000010d000 103:00002 libharfbuzz.so.0.60830.0
00007182281a7000      24 r---- 0000000000000000 103:00002 libpng16.so.16.41.0
00007182281ad000     160 r-x-- 0000000000006000 103:00002 libpng16.so.16.41.0
00007182281d5000      40 r---- 000000000002e000 103:00002 libpng16.so.16.41.0
00007182281df000       4 r---- 0000000000038000 103:00002 libpng16.so.16.41.0
00007182281e0000       4 rw--- 0000000000039000 103:00002 libpng16.so.16.41.0
00007182281e1000       8 r---- 0000000000000000 103:00002 libbz2.so.1.0.8
00007182281e3000      52 r-x-- 0000000000002000 103:00002 libbz2.so.1.0.8
00007182281f0000       8 r---- 000000000000f000 103:00002 libbz2.so.1.0.8
00007182281f2000       4 r---- 0000000000010000 103:00002 libbz2.so.1.0.8
00007182281f3000       4 rw--- 0000000000011000 103:00002 libbz2.so.1.0.8
00007182281f4000      12 r---- 0000000000000000 103:00002 libz.so.1.3.1
00007182281f7000      56 r-x-- 0000000000003000 103:00002 libz.so.1.3.1
0000718228205000      28 r---- 0000000000011000 103:00002 libz.so.1.3.1
000071822820c000       4 r---- 0000000000017000 103:00002 libz.so.1.3.1
000071822820d000       4 rw--- 0000000000018000 103:00002 libz.so.1.3.1
000071822820e000       8 r---- 0000000000000000 103:00002 libexpat.so.1.9.0
0000718228210000     112 r-x-- 0000000000002000 103:00002 libexpat.so.1.9.0
000071822822c000      32 r---- 000000000001e000 103:00002 libexpat.so.1.9.0
0000718228234000       8 r---- 0000000000026000 103:00002 libexpat.so.1.9.0
0000718228236000       4 rw--- 0000000000028000 103:00002 libexpat.so.1.9.0
0000718228237000       8 r---- 0000000000000000 103:00002 libXrender.so.1.3.0
0000718228239000      28 r-x-- 0000000000002000 103:00002 libXrender.so.1.3.0
0000718228240000       8 r---- 0000000000009000 103:00002 libXrender.so.1.3.0
0000718228242000       4 r---- 000000000000a000 103:00002 libXrender.so.1.3.0
0000718228243000       4 rw--- 000000000000b000 103:00002 libXrender.so.1.3.0
0000718228244000       8 rw--- 0000000000000000 000:00000   [ anon ]
0000718228246000      48 r---- 0000000000000000 103:00002 libxcb.so.1.1.0
0000718228252000      80 r-x-- 000000000000c000 103:00002 libxcb.so.1.1.0
0000718228266000      36 r---- 0000000000020000 103:00002 libxcb.so.1.1.0
000071822826f000       4 r---- 0000000000028000 103:00002 libxcb.so.1.1.0
0000718228270000       4 rw--- 0000000000029000 103:00002 libxcb.so.1.1.0
0000718228271000     144 r---- 0000000000000000 103:00002 libc.so.6
0000718228295000    1388 r-x-- 0000000000024000 103:00002 libc.so.6
00007182283f0000     340 r---- 000000000017f000 103:00002 libc.so.6
0000718228445000      16 r---- 00000000001d3000 103:00002 libc.so.6
0000718228449000       8 rw--- 00000000001d7000 103:00002 libc.so.6
000071822844b000      32 rw--- 0000000000000000 000:00000   [ anon ]
0000718228453000      56 r---- 0000000000000000 103:00002 libfreetype.so.6.20.1
0000718228461000     564 r-x-- 000000000000e000 103:00002 libfreetype.so.6.20.1
00007182284ee000     168 r---- 000000000009b000 103:00002 libfreetype.so.6.20.1
0000718228518000      32 r---- 00000000000c4000 103:00002 libfreetype.so.6.20.1
0000718228520000       4 rw--- 00000000000cc000 103:00002 libfreetype.so.6.20.1
0000718228521000      32 r---- 0000000000000000 103:00002 libfontconfig.so.1.14.0
0000718228529000     180 r-x-- 0000000000008000 103:00002 libfontconfig.so.1.14.0
0000718228556000      96 r---- 0000000000035000 103:00002 libfontconfig.so.1.14.0
000071822856e000       8 r---- 000000000004c000 103:00002 libfontconfig.so.1.14.0
0000718228570000       4 rw--- 000000000004e000 103:00002 libfontconfig.so.1.14.0
0000718228571000      16 r---- 0000000000000000 103:00002 libXft.so.2.3.8
0000718228575000      64 r-x-- 0000000000004000 103:00002 libXft.so.2.3.8
0000718228585000      16 r---- 0000000000014000 103:00002 libXft.so.2.3.8
0000718228589000       4 r---- 0000000000017000 103:00002 libXft.so.2.3.8
000071822858a000       4 rw--- 0000000000018000 103:00002 libXft.so.2.3.8
000071822858b000     112 r---- 0000000000000000 103:00002 libX11.so.6.4.0
00007182285a7000     556 r-x-- 000000000001c000 103:00002 libX11.so.6.4.0
0000718228632000     596 r---- 00000000000a7000 103:00002 libX11.so.6.4.0
00007182286c7000      12 r---- 000000000013b000 103:00002 libX11.so.6.4.0
00007182286ca000      16 rw--- 000000000013e000 103:00002 libX11.so.6.4.0
00007182286ce000       8 rw--- 0000000000000000 000:00000   [ anon ]
00007182286d0000      12 r--s- 0000000000000000 103:00002 6ba42ae0000f58711b5caaf10d690066-le64.cache-9
00007182286d3000      56 r--s- 0000000000000000 103:00002 210c0516121708a580e22e6b1f9a103a-le64.cache-9
00007182286e1000       4 r--s- 0000000000000000 103:00002 11b490a251c73e9885ef8a4f43e7c08f-le64.cache-9
00007182286e2000       4 r---- 0000000000000000 103:00002 ld-linux-x86-64.so.2
00007182286e3000     156 r-x-- 0000000000001000 103:00002 ld-linux-x86-64.so.2
000071822870a000      44 r---- 0000000000028000 103:00002 ld-linux-x86-64.so.2
0000718228715000       8 r---- 0000000000032000 103:00002 ld-linux-x86-64.so.2
0000718228717000       8 rw--- 0000000000034000 103:00002 ld-linux-x86-64.so.2
00007ffd9ddf0000     132 rw--- 0000000000000000 000:00000   [ stack ]
00007ffd9de26000      16 r---- 0000000000000000 000:00000   [ anon ]
00007ffd9de2a000       8 r-x-- 0000000000000000 000:00000   [ anon ]
ffffffffff600000       4 --x-- 0000000000000000 000:00000   [ anon ]
mapped: 17940K    writeable/private: 2380K    shared: 1132K
```

What we can take form this is that our dynamic version of st mapped 17940K (17Mb) of which 2380K is private data -- memory that the terminal is using, and only 1132K is shared data.

Now, let's look at the static version -- st-static:
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

We see a decrease in mapped memory usage, 7364K. Private data remain mostly the same and shared data follows this as well. This however does not paint the hole picture.

Let's look at the output of **top(1)**, first for st-dyn.

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

We can see a significant decrease in resident memory usage -- RES, about half. Summing things up in the table below (take the values shown here for what they are). I'll leave the conclusions as an exercise to the reader, there are trade-offs and we are only looking at a single binary, but the benefits are there. There are also arguments that can be made about the security of DSO's and how to deploy security patches to static linked code, but they fall short of the scope of this article.

```shell {linenos=false}
            +--------------------------+------- +-----------+
            |                          | st-dyn | st-static |
            +--------------------------+------- +-----------+
            | size of the binary       |   113K |     3236K |
            | VIRT (top)               | 17936K |     7360K |
            | RES (top)                | 10368K |     5048K |
            | SHR (top)                |  8064K |     3328K |
            | mapped (pmap)            | 17940K |     7364K |
            | writeable/private (pmap) |  2380K |     2308K |
            | shared (pmap)            |  1132K |     1068K |
            +--------------------------+------- +-----------+
```

[^1]: Take for instance the communication between browser extensions and a native modules (you have likely used this feature if you have installed a gnome shell extension), normally, communication between browser and the native module happens using a simple **stdin(3)** to **stdout(3)** **pipe(3)**. With snaps, however, communication has to go trough the xdg-desktop-portal -- dbus(1), meaning, in the end, they are not portable. If your system is older and has an older version of xdg-desktop-portal that does not support the protocol which handles browser to native module communication, it simply won't work. Snaps are broken, and they break what they package. If you want to see this for yourself, try to use a PKCS #11 token in Snap Firefox.
