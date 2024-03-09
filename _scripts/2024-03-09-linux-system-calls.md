---
layout: post
title: System Calls 101
date: 2024-03-09 14:14:00-0400
description: Gatekeepers of the Kernel world
tags: Linux
categories: sample-posts
toc:
  sidebar: left
---

# TL;DR

# What is system calls and why do we need them

System calls are the *interface* exposed by the Kernel for communication between the user programs and certain Kernel services. Such services include but are not limited to time, file, network, which are the underlying infrastructure for user programs and are essential to all programs in the whole system. System calls exist because of two main reasons: (a) the user program *relies on* these Kernel services for correct execution, and (b) these services should be used *safely*, especially in a potentially malicious environment. Contamination of them could lead to the fall of the whole system. 

To understand why we need system calls, let’s take a closer look into time subsystem. The best example to demonstrate why clock should be properly protected is probably Evil Under the Sun by Agatha Christie (highly recommended but I’m not spoiling anything). The clock in computer is protected by the Kernel to provide any malicious program from overwriting the time and jeopardize the correct execution of other programs. Therefore, the only way a program can know about the time is by asking the Kernel to read the clock for it. And that is when system calls take place, posting the clock reading request to the Kernel and returning the time back to the program.

# How do system calls work

I always explain how system calls work to people as trying to order food in a foreign country. Imagine you walk into a restaurant, starving, but unfortunately your foreign language skill is limited to the beginner level on Duolingo. What do you do? Obviously it is too late for you to learn how to speak another language because you are starving to death. You pick up the menu, and thank God that there are numbers and pictures of the items on it. You decide to order #5 — whatever that is — and signal the waiter. He approaches and you say “#5 please” using your limited language skill and pray that you get it right. After some time he comes back with your meal — luckily — and hopefully it fits your appetite. 

This is basically what happens when a program tries to invoke a system call, where the program is the nervous customer and the Kernel is the waiter taking orders. 