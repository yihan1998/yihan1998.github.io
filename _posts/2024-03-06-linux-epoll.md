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

# Structures and functionalities in a nutshell

|     Term     | Functionalities |
| :----------: | :-------------: |
|  eventpoll   |                 |
|    epitem    |                 |
| eppoll_entry |                 |
|  ep_pqueue   |                 |
| epoll_filefd |                 |
