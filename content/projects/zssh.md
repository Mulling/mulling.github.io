---
author: "Mülling"
title: "zssh(1)"
date: 2025-04-26T20:20:02-03:00
draft: false
description: Freestanding implementation of the ssh(1) protocol.
---

Zig module that implements the ssh protocol. Not including implementations for cryptographic algorithms such as ed25519, or sntrup761x25519-sha512, etc... This allows for reuse of other established implementations, or use of non-spec defined algorithms.

Parsing of ssh primitives (i.e. keys, certkeys, sshsigs, agent messages) is done entirely in place.

You can read more about it [here](https://github.com/say-fish/zssh?tab=readme-ov-file#zssh).
