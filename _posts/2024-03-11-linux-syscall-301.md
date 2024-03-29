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

Basically, what it does is to call `syscalltbl.sh` that takes `syscall_64.tbl` as input and generates `syscalls_64.h`. How the bash script work is too detailed for this article, but in one sentence, it maps every single syscall number to its handler entry in a format of `__SYSCALL_64/X32(*index*, *handler_entry*)` in `syscalls_64.h`. The header file shall be regenerated if there is any changes in `syscalltbl.sh` or `syscall_64.tbl`. In fact, this is a universal approach for every hardware architecture that supports the Linux Kernel to automatically create the syscall header. If you look into the source code of version 5.4, there is some `Makefile` in each subfolder under `/arch/` that serves the same purpose like the commands above.

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

If we go back to the definition of syscall table, we shall find that it is defined as an array of `sys_call_ptr_t`, which based on its name, we know it should be a function pointer pointing to syscall handler. It is defined in `/arch/x86/include/asm/syscall.h` as follows:

```c
typedef asmlinkage long (*sys_call_ptr_t)(const struct pt_regs *);
```

According to this definition, each syscall handler should take `const struct pt_regs *` -- which saves the register context -- as the single argument and return a long variable. But why didn't we see `const struct pt_regs *` in the argument list of `SYSCALL_DEFINE3`? What steps are performed between the Kernel passes `struct pt_regs` to the syscall handler entry and the ultimate handler is invoked?

Let's begin with `SYSCALL_DEFINE<n>` macro in `/include/linux/syscalls.h`. The purpose of this macro is to define a format of syscall handler within the Kernel. As we mentioned in the previous article, the `<n>` here stands for the number of arguments that this syscall handler takes. Although there is no restriction on the number of arguments for a C function, a caller on the `x86_64` architecture can use at most 6 registers for argument passing to the callee, which are `%rdi, %rsi, %rdx, %rcx, %r8, %r9` in left-to-right order. Therefore, the `<n>` in `SYSCALL_DEFINE<n>` macro ranges from 0 to 6 on `x86_64`. Here is the definition of the series of `SYSCALL_DEFINE<n>` macro:

```c
#define SYSCALL_DEFINE_MAXARGS	6

#define SYSCALL_DEFINE1(name, ...) SYSCALL_DEFINEx(1, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE2(name, ...) SYSCALL_DEFINEx(2, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE5(name, ...) SYSCALL_DEFINEx(5, _##name, __VA_ARGS__)
#define SYSCALL_DEFINE6(name, ...) SYSCALL_DEFINEx(6, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)				\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
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

Now we have reached the end of syscalls that takes no argument (e.g., `fork` and `getpid`). Let’s see how `SYSCALL_DEFINEx` helps to implement an uniform interface for various syscalls that may have different number of arguments. On `x86_64`, `__SYSCALL_DEFINEx` is defined in `/arch/x86/include/asm/syscall_wrapper.h` as follows:

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

In short, this macro involves multiple declaration and definition to hide the process of extracting arguments from register context. But let’s start with the invocation chain first. Using `read` as an example, the invocation chain is `__x64_sys_read` -> `__se_sys_read` -> `__do_sys_read`. Therefore, the code for `read` is within a local function named `__do_sys_read`. Now we will move on to the more complicated process -- the declaration and extraction of syscall arguments.

## How to deal with different number of arguments?

As shown above, the definition of `__SYSCALL_DEFINEx` consist of three functions. Before we go through them one by one, we are going to introduce some important macros that appears in all three of them.

### Definition of argument lists via `__MAP`

Now let’s see how `__MAP` macro deals with handler arguments. The definition is within `/include/linux/syscalls.h` as follows:

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

The `n` in `__MAP` is the number of type and name pairs to be concatenated together, and the corresponding `__MAP<n>` will be selected. As shown above, `__MAP<n>` series is a recursive definition, and it keeps appending type `t` with name `a` in the format of `m` until all pairs are adjoined. The format `m` makes `__MAP` macro extremely useful in both declaration, definition, and invocation. Now let’s take a close look at macros that are used as the concatenation format.

### Adjoin the type with variable name via `__SC_*` 

```c
#define __SC_DECL(t, a)	t a
#define __TYPE_AS(t, v)	__same_type((__force t)0, v)
#define __TYPE_IS_L(t)	(__TYPE_AS(t, 0L))
#define __TYPE_IS_UL(t)	(__TYPE_AS(t, 0UL))
#define __TYPE_IS_LL(t) (__TYPE_AS(t, 0LL) || __TYPE_AS(t, 0ULL))
#define __SC_LONG(t, a) __typeof(__builtin_choose_expr(__TYPE_IS_LL(t), 0LL, 0L)) a
#define __SC_CAST(t, a)	(__force t) a
#define __SC_TYPE(t, a)	t
#define __SC_ARGS(t, a)	a
#define __SC_TEST(t, a) (void)BUILD_BUG_ON_ZERO(!__TYPE_IS_LL(t) && sizeof(t) > sizeof(long))
```

The commonly used formats are also defined in `/include/linux/syscalls.h`, and their jobs are as follows:

* `__SC_DECL`: define a variable named `a` as type `t`
* `__SC_LONG`: define a variable named `a` as type `long` or `long long`
* `__SC_CAST`: cast a variable named `a` as type `t` by force
* `__SC_TYPE`: only keep the variable type `t`
* `__SC_ARGS` : only keep the variable name `a`
* `__SC_TEST`: report error if type `t` is not `long long` or `unsigned long long` but is still longer than a `long` type

### Extract the maximum number of arguments from register context

```c
asmlinkage long __x64_sys##name(const struct pt_regs *regs)
{
	return __se_sys##name(SC_X86_64_REGS_TO_ARGS(x,__VA_ARGS__));
}
```

The first function `__x64_sys##name()` is the exposed syscall entry. The argument `struct pt_regs` is where values of all registers are saved and `x` is the number of arguments for this syscall. As we mentioned before, a syscall takes at most 6 arguments, passing via `%rdi, %rsi, %rdx, %rcx, %r8, %r9` in left-to-right order. Therefore, the first step is to extract these 6 arguments from the full register context to avoid any unnecessary exposure of register information. This is performed by the following`SC_X86_64_REGS_TO_ARGS` macro. 

```c
/* Mapping of registers to parameters for syscalls on x86-64 and x32 */
#define SC_X86_64_REGS_TO_ARGS(x, ...)					\
	__MAP(x,__SC_ARGS						\
		,,regs->di,,regs->si,,regs->dx				\
		,,regs->r10,,regs->r8,,regs->r9)			\
```

The details of `__MAP` will be discussed later, but in short, this macro concatenate the variable and its type together in a certain format. Here is what we get after expanding `__MAP`.

```c
asmlinkage long __x64_sys##name(const struct pt_regs *regs)
{
	return __se_sys##name(regs->di,regs->si,regs->dx,
				regs->r10,regs->r8,regs->r9);
}
```

### Cast the register content into the required type

Now we move on to the second function `__se_sys##name()`. 

```c
static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))
{
	long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));
	__MAP(x,__SC_TEST,__VA_ARGS__);
	return ret;
}
```

### Invocation of the ultimate syscall handler

```c
static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```


