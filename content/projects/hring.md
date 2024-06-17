---
author: "Mülling"
title: "hring(3)"
date: 2024-05-27T23:10:30-03:00
draft: false
description: Io-Uring based inter process communication
---

Using IORING_OP_NOP it's possible to send arbitrary data (up to 64 bits) to another process. We can use this to share memory-pool entries, meaning, we can send data to another process. All the synchronization machinery is already provided -- for free -- by io_uring. You only need a shared memory allocator.

You can read more about it [here](https://github.com/Mulling/io-uring-ipc?tab=readme-ov-file#io_uring-ipc).
