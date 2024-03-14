---
layout: post
title: "Syscall 201: The Lifecycle of System Calls in Operaing System"
date: 2024-03-09 11:14:00-0400
description: Initialization, invocation, and execution of system calls
tags: Linux
categories: sample-posts
toc:
  sidebar: left
---

In this article, we are getting a general overview of the life cycle of system calls — how they are defined, declared, organized, and more importantly, invoked inside the Kernel.

# Quick Recap on Syscall

|     Terms      |                                 In a Nutshell                                 |
| :------------: | :---------------------------------------------------------------------------: |
|  System call   |         Interface between userspace programs and the Kernel services          |
| Syscall table  |  An array of system calls where the location of syscall functions are stored  |
| Syscall number | The offset used to look up the corresponding syscall in the system call table |

# The Journey before the Kernel even exists

## Define a System Call Function

In Linux Kernel, every system call function is defined via a macro called `SYSCALL_DEFINEx`, where `x` stands for the number of arguments this system call function takes. We shall take a closer look at this macro in the next blog.

## Establishment of System Call Table

The system call table is a globally shared variable inside the Kernel. Therefore, every system call made by any process is actually accessing the same memory region where the table is stored. In short, the system call table is an array of function pointers, where the index indicates the system call number of the corresponding syscall function.

In the actual Kernel, there could be hundreds of system calls and manually filling the table is obviously both unrealistic and not extendable. If you want to know more about the initialization trick here, do check out the following articles of system calls.

# Call of the adventure: triggering a system call

Typically, there are two ways to invoke a system call — though this is not exactly accurate because they are fundamentally the same — direct invocation by specifying the system call number and trapping straight into the Kernel, while indirect invocation by using certain library function which does the former invocation on your behalf.

It is worth noticing that system calls are one of the few ways that incur mode switch between the user and kernel space. The others are interrupt (e.g., from keyboard) and exception (e.g., dividing by 0), which we will cover in other chapters. The exact instruction that cause mode switching in system call differs from underlying instruction set architecture. For those who have learned assembly before, you might come across the instruction `int 0x80`. This is commonly used in `x86` architecture to invoke a system call. While in `x86_64`, such role is replaced by a faster`syscall` instruction.

# Dive into the Kernel world

# Execute the system call

# Return back to the userspace
