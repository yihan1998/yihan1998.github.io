---
layout: post
title: System Calls 101
date: 2024-03-09 11:14:00-0400
description: Gatekeepers of the Kernel world
tags: Linux
categories: sample-posts
toc:
  sidebar: left
---

# TL;DR

# What are system calls and why do we need them

# How do system calls work

I always explain how system calls work to people as trying to order food in a foreign country. Imagine you walk into a restaurant, starving, but unfortunately your foreign language skill is limited to the beginner level on Duolingo. What do you do? Obviously it is too late for you to learn how to speak another language because you are starving to death. You pick up the menu, and thank God that there are numbers and pictures of the items on it. You decide to order #5 — whatever that is — and signal the waiter. He approaches and you say “#5 please” using your limited language skill and pray that you get it right. After some time he comes back with your meal — luckily — and hopefully it fits your appetite. 

This is basically what happens when a program tries to invoke a system call, where the program is the nervous customer and the Kernel is the waiter taking orders. 