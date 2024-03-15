---
layout: post
title: "Syscall 301: A Deep Dive into the System Calls in Linux Kernel"
date: 2024-03-09 11:14:00-0400
description: Highly detailed walkthrough of system call mechanism
tags: Linux
categories: sample-posts
toc:
  beginning: true
---

In this artical, we are taking a deep dive into the system calls in Linux Kernel 5.4 on an `x86_64` architecture.

# Set up the syscall table

The Linux Kernel utilizes a lot of tricks in C in an impressive way and the syscall table initilization is one of my favourite. It is a perfect example of how coding in C can be elegant, flexible, and creative.

The definition of syscall table -- also known as `sys_call_table` in the Linux Kernel -- locates in `/arch/x86/entry/syscall_64.c` and is an array of function pointers. Below is the source code that performs syscall table setup:

```c
#define __SYSCALL_64(nr, sym, qual) extern asmlinkage long sym(const struct pt_regs *);
#include <asm/syscalls_64.h>
#undef __SYSCALL_64

#define __SYSCALL_64(nr, sym, qual) [nr] = sym,

asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &__x64_sys_ni_syscall,
#include <asm/syscalls_64.h>
};

#undef __SYSCALL_64
```

Remember we talked about the how unrealistic to manually initialize the syscall table in the previous artical? Here we are going to unveal the magic trick applied by the Linux Kernel, which is the use of `#define`, `#include`, and `#undef`.

> The first `__SYSCALL_64` generates the declaration of every syscall handler.
>
> The second `__SYSCALL_64` loads the address syscall handler into the `sys_call_table`, using their syscall number as the index within the syscall table array.

Such a creative way of using the feature of C during the preprocessing phase! By defining, undefining, and redefining only one macro, the preprocessor could automatically generate all declaration of every syscall handler and the whole syscall table! How seemingless yet highly efficient it is!

Now, if you are still curious about what is in this header, and how it is loaded when included, fasten your seat belt because we are going follow the preprocessor to see how miracle is performed. Allons-y!

## What is in `<asm/syscalls_64.h>`?

When I saw the above syscall initialization for the first time, I was shocked and confused: how and why does `<asm/syscalls_64.h>` get included here twice yet expanded into two different pieces of codes?

To understand why, we need to look into the content of this header, which locates under `/arch/x86/include/generated/asm/syscalls_64.h`. Below is a clip of it:

```c
__SYSCALL_64(0, __x64_sys_read, )
__SYSCALL_64(1, __x64_sys_write, )
__SYSCALL_64(2, __x64_sys_open, )
__SYSCALL_64(3, __x64_sys_close, )
...
```

For the sake of simplicity, I only show the what will be left after the preprocessing phase on the `x86_64` architecture. In the actual header, the Kernel uses other `#ifdef` directive to support more flexible Kernel configuration during preprocessing.

To make things easier to understand, let’s take `read` for an example to see how the two different `__SYSCALL_64` macros expand the above mappings separatedly.

```c
#define __SYSCALL_64(nr, sym, qual) extern asmlinkage long sym(const struct pt_regs *);
extern asmlinkage long __x86_sys_read(const struct pt_regs *);
...
#undef __SYSCALL_64

#define __SYSCALL_64(nr, sym, qual) [nr] = sym,

asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &__x64_sys_ni_syscall,
	[0] = __x86_sys_read,
	...
};

#undef __SYSCALL_64
```

As shown above, the first `__SYSCALL_64` expands the header into the declaration of syscall handlers, which will later be used to locate the handler entry during syscall table lookup. Notably, before expanding the second `__SYSCALL_64`, the Kernel first initializes all syscall handler to be `__x64_sys_ni_syscall `, just in case any unsupported syscall number triggers a segmentation fault and blow up the whole Kernel. After this, the second `__SYSCALL_64` associates the syscall handler with its syscall number in the table. In this way, the Kernel implements a general setup of syscall table, independent from the varied syscall mapping on different hardware platforms.

## Why does `/arch/x86/include/generated` seem weird?

My first impression when I looked at this path is: how could someone name the folder to be `generated`? The answer may _not_ surprise you: because this folder **is** generated! But how, why, and by whom? 

With a simple searching in the repository, we can easily locate the creator of the `generated/` folder — the `Makefile` under `/arch/x86/syscalls/`. Below is the part of it that is associated with the generation of syscall related files.

```makefile
out := arch/$(SRCARCH)/include/generated/asm

syscall64 := $(srctree)/$(src)/syscall_64.tbl

systbl := $(srctree)/$(src)/syscalltbl.sh

$(out)/syscalls_64.h: $(syscall64) $(systbl)
	$(call if_changed,systbl)
```

Basically, what it does is to call `syscalltbl.sh` that takes `syscall_64.tbl` as input and generates `syscalls_64.h`. How the bash script work is too detailed for this article, but in one sentence, it maps every single syscall number to its handler entry in a format of `__SYSCALL_64/X32(*index*, *handler_entry*)` in `syscalls_64.h`.  The header file shall be regenerated if there is any changes in `syscalltbl.sh` or `syscall_64.tbl`. In fact, this is a universal approach for every hardware architecture that supports the Linux Kernel to automatically create the syscall header. If you look into the source code of version 5.4, there is some `Makefile` in each subfolder under `/arch/` that serves the same purpose like the commands above.

Now let’s see how syscall numbers and syscall handler are mapped in `syscall_64.tbl`:

```sh
# The format is:
# <number> <abi> <name> <entry point>
# The abi is "common", "64", or "x32" for this file
0	common	read		__x64_sys_read
1	common	write	__x64_sys_write
2	common	open		__x64_sys_open
3	common	close	__x64_sys_close
...
19	64		readv	__x64_sys_readv
20	64		writev	__x64_sys_writev
...
512	x32		readv	__x32_compat_sys_readv
513	x32		writev	__x32_compat_sys_writev
...
```

It's clear from the clip above that this file supports three types of ABI: `common` (for both 32-bit and 64-bit), `64` (for 64-bit), and `x32` (for 32-bit). You can also easily distinguish their type based on the prefix of the handler entry. 

# Define a system call handler

If you try to search for the syscall handler we've mentioned above (e.g., `__x64_sys_read`), you might be confused when you fail to see any function with that name in the Kernel source code. Why and how could it be possible? The answer lies in another header file -- `/include/linux/syscalls.h` -- and we are about to be amazed by the wisdom of system designers again.

## How is a syscall hander be defined?

Let's take `read` as an exmaple again. The actual implementation locates in `/fs/read_write.c` and is defined as follows:

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
```

This doesn't seem like an ordinary defination of a function, so what is `SYSCALL_DEFINE3` macro and how does the Linux Kernel turn the above definition into `__x64_sys_read` in the syscall table? 

Let's begin with `SYSCALL_DEFINE<#>` macro in `/include/linux/syscalls.h`. The purpose of this macro is to define a format of syscall handler within the Kernel. As we mentioned in the previous article, the `<#>` here stands for the number of arguments that this syscall handler takes. Although there is no restriction on the number of arguments for a C function, a caller on the `x86_64` architecture can use at most 6 registers for argument passing to the callee, which are `%rdi, %rsi, %rdx, %rcx, %r8, %r9` in left-to-right order. Therefore, the `<#>` in `SYSCALL_DEFINE<#>` macro ranges from 0 to 6 on `x86_64`. Here is the definition of the series of `SYSCALL_DEFINE<#>` macro:

```c
#define SYSCALL_DEFINE_MAXARGS	6

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)
```

What worth noticing is the `SYSCALL_DEFINE0` macro, which might be defined differently across hardware architecture. On `x86_64`, it is defined in `/arch/x86/include/asm/syscall_wrapper.h` as follows:

```c
/*
 * As the generic SYSCALL_DEFINE0() macro does not decode any parameters for
 * obvious reasons, and passing struct pt_regs *regs to it in %rdi does not
 * hurt, we only need to re-define it here to keep the naming congruent to
 * SYSCALL_DEFINEx()
 */
#ifndef SYSCALL_DEFINE0
#define SYSCALL_DEFINE0(sname)						\
	asmlinkage long __x64_sys_##sname(const struct pt_regs *__unused);\
	asmlinkage long __x64_sys_##sname(const struct pt_regs *__unused)
#endif
```

Now you might be wondering, what is the `SYSCALL_DEFINEx`?

```c
#define SYSCALL_DEFINEx(x, sname, ...)				\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

## How to deal with different number of syscall arguments? 

If we go back to the definition of syscall table, we shall find that it is defined as an array of `sys_call_ptr_t`, which based on its name, we know it should be a function pointer pointing to syscall handler. It is defined in `/arch/x86/include/asm/syscall.h` as follows:

```c
typedef asmlinkage long (*sys_call_ptr_t)(const struct pt_regs *);
```

### From `struct pt_regs` to argument list

```c
/*
 * Instead of the generic __SYSCALL_DEFINEx() definition, this macro takes
 * struct pt_regs *regs as the only argument of the syscall stub named
 * __x64_sys_*(). It decodes just the registers it needs and passes them on to
 * the __se_sys_*() wrapper performing sign extension and then to the
 * __do_sys_*() function doing the actual job. These wrappers and functions
 * are inlined (at least in very most cases), meaning that the assembly looks
 * as follows (slightly re-ordered for better readability):
 *
 * <__x64_sys_recv>:		<-- syscall with 4 parameters
 *	callq	<__fentry__>
 *
 *	mov	0x70(%rdi),%rdi	<-- decode regs->di
 *	mov	0x68(%rdi),%rsi	<-- decode regs->si
 *	mov	0x60(%rdi),%rdx	<-- decode regs->dx
 *	mov	0x38(%rdi),%rcx	<-- decode regs->r10
 *
 *	xor	%r9d,%r9d	<-- clear %r9
 *	xor	%r8d,%r8d	<-- clear %r8
 *
 *	callq	__sys_recvfrom	<-- do the actual work in __sys_recvfrom()
 *				    which takes 6 arguments
 *
 *	cltq			<-- extend return value to 64-bit
 *	retq			<-- return
 *
 * This approach avoids leaking random user-provided register content down
 * the call chain.
 */
#define __SYSCALL_DEFINEx(x, name, ...)					\
	asmlinkage long __x64_sys##name(const struct pt_regs *regs);	\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));	\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));\
	asmlinkage long __x64_sys##name(const struct pt_regs *regs)	\
	{								\
		return __se_sys##name(SC_X86_64_REGS_TO_ARGS(x,__VA_ARGS__));\
	}								\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))	\
	{								\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));\
		__MAP(x,__SC_TEST,__VA_ARGS__);				\
		return ret;						\
	}								\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

```c
/*
 * __MAP - apply a macro to syscall arguments
 * __MAP(n, m, t1, a1, t2, a2, ..., tn, an) will expand to
 *    m(t1, a1), m(t2, a2), ..., m(tn, an)
 * The first argument must be equal to the amount of type/name
 * pairs given.  Note that this list of pairs (i.e. the arguments
 * of __MAP starting at the third one) is in the same format as
 * for SYSCALL_DEFINE<n>/COMPAT_SYSCALL_DEFINE<n>
 */
#define __MAP0(m,...)
#define __MAP1(m,t,a,...) m(t,a)
#define __MAP2(m,t,a,...) m(t,a), __MAP1(m,__VA_ARGS__)
#define __MAP3(m,t,a,...) m(t,a), __MAP2(m,__VA_ARGS__)
#define __MAP4(m,t,a,...) m(t,a), __MAP3(m,__VA_ARGS__)
#define __MAP5(m,t,a,...) m(t,a), __MAP4(m,__VA_ARGS__)
#define __MAP6(m,t,a,...) m(t,a), __MAP5(m,__VA_ARGS__)
#define __MAP(n,...) __MAP##n(__VA_ARGS__)
```

```c
/* Mapping of registers to parameters for syscalls on x86-64 and x32 */
#define SC_X86_64_REGS_TO_ARGS(x, ...)					\
	__MAP(x,__SC_ARGS						\
		,,regs->di,,regs->si,,regs->dx				\
		,,regs->r10,,regs->r8,,regs->r9)			\
```