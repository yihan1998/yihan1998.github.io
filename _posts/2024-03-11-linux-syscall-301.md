---
layout: post
title: "Syscall 301: A Deep Dive into the Syscall Mechanism in Linux Kernel"
date: 2024-03-09 11:14:00-0400
description: How does Linux actually implement systems calls?
tags: Linux
categories: sample-posts
toc:
  sidebar: left
mermaid:
  enabled: true
  zoomable: true
---

In this artical, we are taking a deep dive into the system call mechanism in Linux Kernel 5.4 running on `x86_64` architecture.

# Setting up the syscall table

The Linux Kernel utilizes a lot of tricks in C in an impressive way and the syscall table initilization is one of my favourite. It is a perfect example of how coding in C can be elegant, flexible, and creative.

The definition of syscall table -- also known as `sys_call_table` -- locates in `/arch/x86/entry/syscall_64.c` and is an array of function pointers. Below is the source code that performs syscall table setup:

```C
#define __SYSCALL_64(nr, sym, qual) extern asmlinkage long sym(const struct pt_regs *);
#include <asm/syscalls_64.h>
#undef __SYSCALL_64

#define __SYSCALL_64(nr, sym, qual) [nr] = sym,

asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	/*
	 * Smells like a compiler bug -- it doesn't work
	 * when the & below is removed.
	 */
	[0 ... __NR_syscall_max] = &__x64_sys_ni_syscall,
#include <asm/syscalls_64.h>
};

#undef __SYSCALL_64
```

Remember we talked about the how unrealistic to manually initialize the syscall table in the previous artical? Here we are going to unveal the magic trick applied by the Linux Kernel, which is the use of `#define`, `#include`, and `#undef`. In short, the first `__SYSCALL_64` generates the declaration of every syscall function, and the second `__SYSCALL_64` loads the address syscall function into the `sys_call_table` array, using their syscall number as the index within the syscall table.

Such a creative way of using the feature of C during the compiling phase! Now let's follow the compiler to see how on earth does it set up the whole syscall table for us.

When I saw this header for the first time, I was shocked and confused: how and why does `<asm/syscalls_64.h>` get included twice yet expanded into two different pieces of codes? To understand why, we need to look into the content of this header, which locates under `/arcg/x86/include/generated/asm/syscalls_64.h`. Below is a clip of it:

```C
__SYSCALL_64(0, __x64_sys_read, )
__SYSCALL_64(1, __x64_sys_write, )
__SYSCALL_64(2, __x64_sys_open, )
__SYSCALL_64(3, __x64_sys_close, )
...
```

For the sake of simplicity, I only show the what will be compiled on `x86_64`.

```Makefile
out := arch/$(SRCARCH)/include/generated/asm

syscall64 := $(srctree)/$(src)/syscall_64.tbl

systbl := $(srctree)/$(src)/syscalltbl.sh

$(out)/syscalls_64.h: $(syscall64) $(systbl)
	$(call if_changed,systbl)
```

