---
layout: post
title: "I/O Multiplexing 301: Deep into the Heart of Eventpoll in Linux"
date: 2024-03-16 11:14:00-0400
description:
tags: Linux
categories: sample-posts
toc:
  beginning: true
---

In this artical, we are digging deep into the `Epoll` in Linux Kernel 5.4 on an `x86_64` architecture.

# Core concepts in `Epoll`

## Epoll instance

## File descriptors

## Events

<!-- |     Term     | Functionalities |
| :----------: | :-------------: |
|  eventpoll   |  Heart of `epoll` mechanism. Wait, collect and report events happened on files that are added to this object. |
|    epitem    |  Represent each individual file descriptor that has been added to an eventpoll instance.  |
| eppoll_entry | Structure to help adding epitem into the wait queue in eventpoll. | -->

# Workflow of Epoll