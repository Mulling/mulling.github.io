---
author: "Mülling"
title: "make-less-noise(1)"
date: 2025-04-26T19:34:00-03:00
draft: false
description: Reducing the noise of the make(1)
---

If you ever compiled the Linux Kernel, you will notice the distinct make output:

```
  GEN     usr/initramfs_data.cpio
  COPY    usr/initramfs_inc_data
  AS      usr/initramfs_data.o
  AR      usr/built-in.a
  CC      kernel/exec_domain.o
  AS      arch/x86/coco/tdx/tdcall.o
  CC      arch/x86/coco/tdx/tdx.o
  HOSTCC  certs/extract-cert
  GENKEY  certs/signing_key.pem
-----
  CC      certs/blacklist.o
  CC      ipc/util.o
  CC      security/keys/key.o
  CC      crypto/cipher.o
  CC      io_uring/io_uring.o
  CC      block/fops.o
  GEN     certs/blacklist_hash_list
  CERT    certs/x509_revocation_list
  CERT    certs/x509_certificate_list
  CERT    certs/signing_key.x509
  CC      certs/blacklist_hashes.o
```

This is quite useful, as you can clearly see what is going on. The kernel achieves this by using custom variables, see the ones for CC in file tools/build/Makefile.build:

```make
# Compile command
quiet_cmd_cc_o_c = CC      $@
      cmd_cc_o_c = $(CC) $(c_flags) -c -o $@ $<

quiet_cmd_host_cc_o_c = HOSTCC  $@
      cmd_host_cc_o_c = $(HOSTCC) $(host_c_flags) -c -o $@ $<

quiet_cmd_cxx_o_c = CXX     $@
      cmd_cxx_o_c = $(CXX) $(cxx_flags) -c -o $@ $<

quiet_cmd_cpp_i_c = CPP     $@
      cmd_cpp_i_c = $(CC) $(c_flags) -E -o $@ $<

quiet_cmd_cc_s_c = AS      $@
      cmd_cc_s_c = $(CC) $(c_flags) -S -o $@ $<
```

We can achieve something similar by using this simple setup (see: tools/scripts/Makefile.include if you want to understand how the kernel does it):

```make
ifndef VERBOSE
.SILENT:

QUIET_CC	    = @echo 'CC     ' $@;
QUIET_LINK	    = @echo 'LINK   ' $@;
QUIET_AR	    = @echo 'AR     ' $@;
endif
```

And those are used in the makefile like this:

```make
main: main.c
    $(QUIET_CC) gcc -o main main.c
```

If you need to debug, simply set `VERBOSE=1`.
